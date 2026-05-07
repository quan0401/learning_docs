---
title: "Aliases and Zero-Downtime Reindexing — Atomic Swaps, Filtered Views, and Blue/Green for Indices"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, aliases, reindex, zero-downtime]
---

# Aliases and Zero-Downtime Reindexing — Atomic Swaps, Filtered Views, and Blue/Green for Indices

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `aliases` `reindex` `zero-downtime`

---

## Table of Contents

- [Summary](#summary)
- [1. Why Aliases Exist — The Indirection Layer](#1-why-aliases-exist--the-indirection-layer)
- [2. Read Aliases vs Write Aliases](#2-read-aliases-vs-write-aliases)
- [3. The Aliases API — `_aliases` and `_alias`](#3-the-aliases-api--_aliases-and-_alias)
- [4. Filtered Aliases — Tenant-Scoped Views](#4-filtered-aliases--tenant-scoped-views)
- [5. Routing on Aliases](#5-routing-on-aliases)
- [6. The Blue/Green Reindex Pattern](#6-the-bluegreen-reindex-pattern)
- [7. Mapping/Analyzer/Sharding Changes That Force a Reindex](#7-mappinganalyzersharding-changes-that-force-a-reindex)
- [8. Catch-Up Indexing — Closing the Reindex Delta](#8-catch-up-indexing--closing-the-reindex-delta)
- [9. Versioned Write Alias and Rollback Strategy](#9-versioned-write-alias-and-rollback-strategy)
- [10. Practical Traps](#10-practical-traps)
- [11. ILM Rollover — Aliases Under the Hood](#11-ilm-rollover--aliases-under-the-hood)
- [Related](#related)
- [References](#references)

---

## Summary

An **alias** is a pointer name that the application talks to instead of the underlying physical index. Elasticsearch and OpenSearch both implement aliases as a thin metadata layer over indices, and the operation that swaps an alias from one index to another is **atomic** — there is no observable instant where the alias points to nothing or to both. That single atomic-swap guarantee is the entire foundation of zero-downtime reindexing: you build a new index in the background, swap the alias in one cluster-state update, and the application keeps issuing requests against `orders` while the bytes underneath flip from `orders-v1` to `orders-v2`.

The piece that trips up engineers coming from "just change the table" relational thinking: **mappings are mostly immutable**. Changing a field's type, an analyzer, the number of primary shards, or a tokenizer requires a brand-new index and a reindex into it — there is no `ALTER COLUMN` for a mapped field. Aliases are how you make that reindex invisible to the application. This doc walks the alias model, the atomic actions API, the blue/green reindex workflow, the catch-up problem during long reindexes, and the traps that bite in production: in-flight bulk requests during the swap, version conflicts on copy, and replication lag against a moving source.

---

## 1. Why Aliases Exist — The Indirection Layer

Without aliases, application code references an index by name (`client.search({ index: 'orders' })`). The day you need a different shape of `orders` — new mapping, new shard count, new analyzer pipeline — you are stuck: you cannot edit those properties on a live index, and renaming the index requires a synchronized application deploy.

Aliases break that coupling. The application talks to the alias `orders`. It points at `orders-v1` today and `orders-v2` tomorrow, invisibly to application code — the same indirection benefit that a database view or a load-balancer DNS name gives you. The Elasticsearch [aliases reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/aliases.html) and OpenSearch [alias documentation](https://docs.opensearch.org/docs/latest/im-plugin/index-alias/) describe the same model. Every index your application touches in production should be behind one.

---

## 2. Read Aliases vs Write Aliases

Aliases come in two flavors that share a surface but have different rules.

**Read aliases** — one alias maps to *one or many* indices. A search against the alias fans out to every backing index. This is how time-series indices work: `logs` is a read alias over `logs-2026.05.05`, `logs-2026.05.06`, `logs-2026.05.07`, and a query against `logs` searches all three.

**Write aliases** — one alias accepts writes and routes them to *exactly one* underlying index. If the alias points at multiple indices, you must mark exactly one of them as `is_write_index: true`. Indexing into an alias whose write-index is ambiguous is a 400 error.

```json
POST /_aliases
{
  "actions": [
    { "add": { "index": "logs-2026.05.07", "alias": "logs", "is_write_index": true } },
    { "add": { "index": "logs-2026.05.06", "alias": "logs", "is_write_index": false } },
    { "add": { "index": "logs-2026.05.05", "alias": "logs", "is_write_index": false } }
  ]
}
```

A search against `logs` reads all three. An index request against `logs` lands in `logs-2026.05.07`. This split is what makes ILM rollover possible (§11).

---

## 3. The Aliases API — `_aliases` and `_alias`

There are two endpoints, with different atomicity guarantees.

`POST /_aliases` accepts a list of actions (`add`, `remove`, `remove_index`) and applies them **atomically**. Either every action succeeds or none does. This is the endpoint you use for the swap.

```json
POST /_aliases
{
  "actions": [
    { "remove": { "index": "orders-v1", "alias": "orders" } },
    { "add":    { "index": "orders-v2", "alias": "orders" } }
  ]
}
```

Between the moment you submit this and the moment the cluster state propagates, no client ever sees `orders` pointing at both `v1` and `v2`, nor at neither. That is the load-bearing property.

`PUT /<index>/_alias/<name>` and `DELETE /<index>/_alias/<name>` are convenience endpoints for single-action changes. They are **not** atomic across multiple operations — if you `DELETE` then `PUT` as separate calls, there is a window where the alias points at nothing. Always use `_aliases` for swaps.

A TS/Node example using the official client:

```ts
await client.indices.updateAliases({
  body: {
    actions: [
      { remove: { index: 'orders-v1', alias: 'orders' } },
      { add:    { index: 'orders-v2', alias: 'orders' } },
    ],
  },
});
```

The Java high-level client exposes the same shape via `IndicesAliasesRequest` with a list of `AliasActions`.

---

## 4. Filtered Aliases — Tenant-Scoped Views

An alias can carry a `filter` clause. Queries through the alias have that filter AND-ed onto every search.

```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "orders-v2",
        "alias": "orders-tenant-acme",
        "filter": { "term": { "tenant_id": "acme" } },
        "index_routing": "acme",
        "search_routing": "acme"
      }
    }
  ]
}
```

A search against `orders-tenant-acme` returns only documents matching `tenant_id: acme` — the application never has to add the filter itself. Filtered aliases are not a security boundary on their own: a caller who can talk to the cluster directly can still bypass the alias by querying the underlying index. Pair them with [document-level security](https://www.elastic.co/guide/en/elasticsearch/reference/current/document-level-security.html) (Elastic) or [fine-grained access control](https://docs.opensearch.org/docs/latest/security/access-control/document-level-security/) (OpenSearch) when the boundary must be enforced.

---

## 5. Routing on Aliases

Aliases can also pin shard routing. `index_routing` is applied on writes through the alias; `search_routing` is applied on searches. With both set to `"acme"` (as in §4), every write lands on the shard chosen by `hash("acme") % num_primary_shards` and every search hits only that shard. For a multi-tenant index where shard-per-tenant routing keeps query fan-out tight, the alias is the natural place to encode the routing rule — the application stays oblivious.

---

## 6. The Blue/Green Reindex Pattern

Assume the application talks to alias `orders`, which currently points at `orders-v1`.

1. **Create the target.** `PUT /orders-v2` with the new mapping/settings.
2. **Copy the data.** Use the [Reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html) to stream documents. For large indices, run it asynchronously and track the task via `GET /_tasks/<task_id>`.

   ```json
   POST /_reindex?wait_for_completion=false&slices=auto
   {
     "source": { "index": "orders-v1" },
     "dest":   { "index": "orders-v2" }
   }
   ```

   `slices=auto` parallelizes the copy across the source's shards.

3. **Atomic swap.** When the reindex finishes and the catch-up is closed (§8), flip the alias with the `_aliases` action set from §3.
4. **Keep `orders-v1`** for a cooling-off period, then delete it once you trust `v2`.

The application code never changed. No deploy was needed. The cutover happened in one cluster-state update.

---

## 7. Mapping/Analyzer/Sharding Changes That Force a Reindex

You cannot change these properties in place. Each of them forces the blue/green dance:

- **Field type changes** (e.g. `text` → `keyword`, `long` → `double`).
- **New analyzer or tokenizer** on an existing field — Lucene's terms dictionary was built with the old analyzer; existing postings will not be re-tokenized.
- **A `.keyword` or other multi-field added after the fact** — applies only to documents indexed after the change.
- **Number of primary shards** — routing is `hash(_id) % num_primary_shards`; changing the divisor invalidates every existing routing decision. (Split and Shrink APIs handle the doubling/halving cases.)
- **Custom analyzer settings** that rewrite tokens (stemming, index-time synonyms, n-grams).

Mental model: any change that would alter what bytes Lucene wrote to `.tim`/`.doc`/`.pos` (covered in [foundations/01](../foundations/01-inverted-index-internals.md)) requires a fresh index. Reindex + alias swap is the only safe path.

---

## 8. Catch-Up Indexing — Closing the Reindex Delta

The Reindex API is a snapshot operation. It walks `orders-v1` documents that exist *now* and writes them to `orders-v2`. If the application is still indexing into `orders-v1` while the reindex runs, those new writes do not appear in `orders-v2` automatically.

Two strategies:

### 8.1 Dual writes during reindex

Before starting the reindex, change the application to write to *both* `orders-v1` and `orders-v2` (idempotent, by `_id`). Run reindex over the snapshot with `version_type: external` and `conflicts: proceed` — reindex respects the source's `_version` and skips any doc the dual write already populated:

```json
POST /_reindex
{
  "conflicts": "proceed",
  "source": { "index": "orders-v1" },
  "dest":   { "index": "orders-v2", "version_type": "external" }
}
```

### 8.2 Delta reindex by timestamp

If documents have an `updated_at` field, run the bulk reindex, then a delta reindex with a range query over `updated_at >= T0` (where `T0` is the start of the bulk reindex). Repeat until the delta is empty, then swap. No dual writes required, but you need reliable timestamps and one final tight catch-up window before the swap.

---

## 9. Versioned Write Alias and Rollback Strategy

The pattern that bakes rollback into the alias scheme is **two aliases, not one**: `orders-read` (the read alias, may span both old and new during cutover) and `orders-write` (points at exactly one index at a time via `is_write_index: true`).

Cutover sequence:

1. Reindex `orders-v1` → `orders-v2` with dual writes via the application.
2. Confirm catch-up is clean.
3. Atomic swap: `orders-write` flips from `v1` to `v2`.
4. Atomic swap: `orders-read` removes `v1` and adds `v2`.
5. Keep `orders-v1`. On regression, swap both aliases back — the application is unchanged.
6. After the cooling-off window, delete `orders-v1`.

The split also enables shadow reads (read from `v2` for a percentage of traffic while writes still go to `v1`) by adding `v2` to `orders-read` early.

---

## 10. Practical Traps

### 10.1 In-flight bulk requests during the swap

A bulk request that started against `orders-v1` finishes against `orders-v1` even after the alias swap, because the request resolved the alias at submit time. Bulk requests that arrive after the swap go to `v2`. There is no cross-index torn write per-document. The requirement is that the dual-write window stays open until the swap is acknowledged across the cluster.

### 10.2 Replication lag and stale reads

Aliases are cluster-state, propagated via the master node. A search on a replica that has not yet seen the new cluster state can still resolve the alias to the old index for a brief window — usually milliseconds, but during master failover or cluster-state pressure it can be longer. Do not delete `orders-v1` immediately after the swap.

### 10.3 Version conflicts on reindex

Without `version_type: external`, reindex assigns fresh `_version` numbers in `orders-v2`, overwriting any dual-written newer document with the older snapshot. **Always pass `version_type: external`** when reindexing against a live source. Pair with `conflicts: proceed` so the reindex tolerates documents the dual write got to first.

### 10.4 `is_write_index` ambiguity

If two indices carry `is_write_index: true` under the same alias, every write is a 400. If you remove the only one without setting another, writes break. Always express both sides of the swap in one `_aliases` action set.

### 10.5 Refresh and replica settings during reindex

Set `index.refresh_interval: -1` and `number_of_replicas: 0` on `orders-v2` for the duration of the bulk reindex, then restore both before the swap — the Elastic [tune for indexing speed](https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-indexing-speed.html) doc covers this. Also note: reindex copies `_source`, so an index created with `_source: { enabled: false }` cannot be reindexed without rederiving documents from your system of record. Discover this *before* the cutover plan.

---

## 11. ILM Rollover — Aliases Under the Hood

Elasticsearch's [Index Lifecycle Management rollover](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-rollover.html) (and OpenSearch's [ISM rollover](https://docs.opensearch.org/docs/latest/im-plugin/ism/policies/)) is built directly on the write-alias model. Create `logs-000001` with the alias `logs` and `is_write_index: true`. When the write-index hits a size, age, or doc-count threshold, ILM creates `logs-000002`, marks it as the new write index, and demotes `logs-000001` to read-only under the same alias. The application keeps writing to `logs`; old indices age through warm/cold/delete phases.

This is the same atomic-swap mechanism from §3, run on a schedule by the cluster instead of by your reindex script. Understanding manual alias swaps first makes ILM straightforward — it is alias swaps with extra phases.

---

## Related

- [Inverted Index Internals — Postings Lists, Skip Lists, and FSTs](../foundations/01-inverted-index-internals.md) — why the on-disk structure cannot be edited in place, which is the whole reason aliases and reindex exist
- _Mappings, Dynamic Templates, and the Cost of Schema Drift (planned)_ — the upstream decisions that determine when you owe yourself a reindex
- _Bulk API and Indexing Throughput Tuning (planned)_ — the write path that the reindex uses and the settings to relax during a backfill
- [Database — Migrations and Zero-Downtime Schema Changes](../../database/INDEX.md) — adjacent pattern in the relational world (expand-contract, dual-write, backfill); the alias swap is the search-engine analogue of a view rename
- [Kubernetes — Rolling Updates and Blue/Green Deployments](../../kubernetes/INDEX.md) — the same atomic-cutover thinking applied to pods instead of indices

## References

- [Elasticsearch Reference — Aliases](https://www.elastic.co/guide/en/elasticsearch/reference/current/aliases.html) — authoritative source for `_aliases`, `_alias`, `is_write_index`, filtered aliases, and routing
- [Elasticsearch Reference — Reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html) — `slices`, `version_type`, `conflicts: proceed`, async tasks
- [Elasticsearch Reference — Index Lifecycle Management Rollover](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-rollover.html) — how rollover uses aliases
- [Elasticsearch Reference — Tune for Indexing Speed](https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-indexing-speed.html) — `refresh_interval`, replica count, bulk size during reindex
- [Elasticsearch Reference — Document Level Security](https://www.elastic.co/guide/en/elasticsearch/reference/current/document-level-security.html) — when filtered aliases are not enough
- [OpenSearch Documentation — Index Aliases](https://docs.opensearch.org/docs/latest/im-plugin/index-alias/) — OpenSearch's equivalent semantics; same atomic-swap guarantee
- [OpenSearch Documentation — Reindex Document](https://docs.opensearch.org/docs/latest/api-reference/document-apis/reindex/) — OpenSearch reindex API surface
- [OpenSearch Documentation — Index State Management](https://docs.opensearch.org/docs/latest/im-plugin/ism/index/) — rollover and lifecycle in the OpenSearch fork
- [Elastic Blog — Changing Mapping with Zero Downtime](https://www.elastic.co/blog/changing-mapping-with-zero-downtime) — the canonical write-up of the alias-swap reindex pattern
