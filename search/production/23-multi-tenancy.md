---
title: "Multi-Tenancy in Search — Index-per-Tenant, Routing Keys, and DLS"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, multi-tenancy, routing, document-level-security]
---

# Multi-Tenancy in Search — Index-per-Tenant, Routing Keys, and DLS

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `multi-tenancy` `routing` `document-level-security`

---

## Table of Contents

- [Summary](#summary)
- [1. The Three Models](#1-the-three-models)
- [2. Model A — Index per Tenant](#2-model-a--index-per-tenant)
- [3. Model B — Single Index with Tenant Filter](#3-model-b--single-index-with-tenant-filter)
- [4. Model C — Single Index with Custom Routing](#4-model-c--single-index-with-custom-routing)
- [5. Filtered Aliases as Tenant-Scoped Read Views](#5-filtered-aliases-as-tenant-scoped-read-views)
- [6. Document-Level and Field-Level Security](#6-document-level-and-field-level-security)
- [7. Aggregation Isolation and Resource Limits](#7-aggregation-isolation-and-resource-limits)
- [8. Quota Enforcement Lives Outside the Engine](#8-quota-enforcement-lives-outside-the-engine)
- [9. Hybrid Patterns — Hot-Tenant Carve-Out and Per-Tenant ILM](#9-hybrid-patterns--hot-tenant-carve-out-and-per-tenant-ilm)
- [10. Practical Traps](#10-practical-traps)
- [Related](#related)
- [References](#references)

---

## Summary

A SaaS app with N customers ("tenants") and one Elasticsearch / OpenSearch cluster has to answer one question on every request: *whose documents is this query allowed to see?* Three architectural choices compose the entire design space. **Index per tenant** gives airtight isolation at the cost of cluster-state explosion (every index is at minimum N primary shards × 1+ replicas, each with its own mapping, FST, and segment files). **Single index with `tenant_id` filter** is dirt cheap and trivially scales to millions of tenants on the indexing side, but provides zero isolation: a noisy tenant's expensive aggregation eats heap shared with everyone else. **Single index with custom routing** sits between them — a tenant's documents land on exactly one shard, so a tenant-scoped query touches one shard instead of all of them, while the cluster still has only a normal number of shards.

The right choice almost always depends on tenant count and tenant size distribution: <100 tenants → index per tenant; thousands of small tenants → routing; whales-and-minnows distributions → routing for the long tail plus dedicated indices for the top 1%. On top of whichever physical layout you pick, **document-level security** (a paid Elastic Stack feature in Platinum or higher; free in OpenSearch's Apache-2.0 security plugin) lets you push the tenant filter into the engine so application code can't accidentally forget it. This doc walks each model, the routing mechanics, the DLS / FLS pieces, and the production traps that bite teams who pick wrong at 100 tenants and discover at 10,000.

---

## 1. The Three Models

| Model | Isolation | Shard count | Per-query shards | Best fit |
|-------|-----------|-------------|------------------|----------|
| Index per tenant | Strong (separate mapping, separate ILM, separate shards) | `N tenants × P primaries × (1+R replicas)` | `P × (1+R)` | <100 tenants, varied schemas, regulated data |
| Single index + filter | None (shared everything) | `P × (1+R)` total | All `P` shards | Internal apps, uniform schema, low isolation needs |
| Single index + routing | Logical (one shard per tenant) | `P × (1+R)` total | 1 shard per query | 100s–1000s of small tenants, uniform schema |

The rules of thumb that fall out:

- Each Lucene shard carries fixed overhead — open file handles, an FST per segment held in heap, and a slot in the cluster-state document the master ships to every node. Elastic's official guidance has, for years, recommended **20 shards per GB of JVM heap** as an upper bound on a data node, with per-shard sizes targeting roughly **10–50 GB**. An index-per-tenant design with 5,000 tenants × 1 primary × 1 replica = 10,000 shards already exceeds this on a 24 GB heap before you index a single document.
- A single-index search with `tenant_id` filter scatters to *every* primary shard in the index. A 30-shard index serving 30,000 tenants does 30 sub-queries per request even when one tenant has only 12 documents. Routing collapses this to 1 shard per request.
- Document-level security (Elastic) and security-plugin DLS (OpenSearch) live independently of physical layout — they enforce the filter inside the engine no matter which model you pick.

---

## 2. Model A — Index per Tenant

```http
PUT /tenant-acme-corp
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 },
  "mappings": { "properties": { "title": { "type": "text" }, ... } }
}

PUT /tenant-globex
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 },
  "mappings": { "properties": { "title": { "type": "text" }, ... } }
}
```

Reads and writes naturally go to the right index because the URL contains the tenant identifier:

```http
POST /tenant-acme-corp/_search
{ "query": { "match": { "title": "kafka" } } }
```

**What you get.** Strong isolation at every layer. A tenant's documents live in their own files on disk. A tenant can have a custom mapping (different analyzers, extra fields, a per-tenant synonym filter). Per-tenant ILM policies — hot-warm-cold transitions, retention windows, snapshot schedules — are trivial because the index is the unit ILM operates on. Deleting a tenant is `DELETE /tenant-acme-corp` and the data is gone in O(seconds).

**What it costs.** Cluster state. Every index, mapping, alias, and template is part of the cluster-state document the elected master replicates to every node on every change. Past a few thousand indices the cluster-state pings get large enough that master-eligible nodes spend significant CPU just gossiping the state, and a master failover can take tens of seconds while the new master rebuilds in-memory structures. Elastic's own multi-tenancy guidance is explicit: thousands of indices per cluster is a smell, tens of thousands a hard problem.

**Picking the primary count.** Per index, `1` is the default for tenant indices and usually the right answer. Two primaries per tenant doubles your shard count for no win unless individual tenants are large enough (>50 GB) to hit per-shard sizing limits.

---

## 3. Model B — Single Index with Tenant Filter

```http
PUT /documents
{
  "settings": { "number_of_shards": 30, "number_of_replicas": 1 },
  "mappings": {
    "properties": {
      "tenant_id": { "type": "keyword" },
      "title":     { "type": "text" }
    }
  }
}

POST /documents/_search
{
  "query": {
    "bool": {
      "filter": [{ "term": { "tenant_id": "acme-corp" } }],
      "must":   [{ "match": { "title": "kafka" } }]
    }
  }
}
```

`tenant_id` as a `keyword` filter is the cheapest possible isolation: a `term` filter on a keyword field is converted to a postings-list intersection, cached in the node's filter cache, and reused across queries. The query is fast on the data path.

The price is paid in three other places:

1. **Fan-out.** With 30 primary shards, every search executes 30 sub-queries even though a small tenant might have all its docs on one shard. The coordinating node pays a context-switch + network round-trip per shard.
2. **Aggregation noisy-neighbor.** A `terms` aggregation on `customer_email` for a 100M-doc tenant builds a per-shard buckets table proportional to that tenant's cardinality, on shards shared with everyone else. The fielddata limit and `search.max_buckets` (default 65,536, configurable since 7.x and tightened in OpenSearch defaults) is the only thing standing between a noisy tenant and `OutOfMemoryError` on the data nodes the rest of the tenants share.
3. **Forgetting the filter.** Every query path in the application code must remember to add the `tenant_id` filter. One missed code path is a cross-tenant data leak. DLS (§6) is the engine-level fix.

This model is reasonable for internal apps with low isolation requirements and tens of millions of total documents. For external SaaS it almost always pairs with DLS so that the filter is *not* the application's responsibility.

---

## 4. Model C — Single Index with Custom Routing

Routing pins each document to one shard based on a hash. By default Elasticsearch routes by `_id`, distributing docs uniformly. Setting the routing key to `tenant_id` makes the per-tenant doc set land on exactly one shard:

```http
PUT /documents
{
  "settings": { "number_of_shards": 30, "number_of_replicas": 1 },
  "mappings": {
    "_routing": { "required": true },
    "properties": {
      "tenant_id": { "type": "keyword" },
      "title":     { "type": "text" }
    }
  }
}

POST /documents/_doc?routing=acme-corp
{ "tenant_id": "acme-corp", "title": "Kafka design notes" }

POST /documents/_search?routing=acme-corp
{
  "query": {
    "bool": {
      "filter": [{ "term": { "tenant_id": "acme-corp" } }],
      "must":   [{ "match": { "title": "kafka" } }]
    }
  }
}
```

Two consequences:

- Indexing for tenant `acme-corp` always writes to `shard = hash("acme-corp") mod 30`. The shard is deterministic — there's no scatter on writes.
- Searches with `?routing=acme-corp` execute on **one shard only**, not all 30. This is the headline win: per-tenant query latency stops scaling with `number_of_shards`. Elastic's official routing documentation calls this out as the standard pattern for "user data" use cases.

You still keep the `term` filter on `tenant_id` as belt-and-braces — `routing` is a *physical placement hint*, not a security boundary, and a request without the routing parameter falls back to scattering to all shards. DLS or app-layer enforcement remains the security mechanism; routing is the performance mechanism.

**The skew trap.** Routing rebalances poorly when tenants are wildly uneven in size. If one tenant has 40% of all documents, the shard they land on is 40% of the index regardless of how many shards you provision. The mitigations are *partition routing* (`index.routing_partition_size`, which spreads a tenant across a *subset* of shards rather than one) for moderate skew, and the hot-tenant carve-out pattern (§9) for severe skew.

---

## 5. Filtered Aliases as Tenant-Scoped Read Views

A **filtered alias** is a virtual index whose name resolves to one or more concrete indices, with an attached query filter that gets `AND`-ed into every search. They are the easiest way to give one team a tenant-scoped read endpoint without rewriting the search code:

```http
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index":  "documents",
        "alias":  "documents-acme-corp",
        "filter": { "term": { "tenant_id": "acme-corp" } },
        "routing": "acme-corp"
      }
    }
  ]
}
```

Now `GET /documents-acme-corp/_search` automatically applies the tenant filter and uses the routing hint. The mapping is shared with the underlying index; you can add more aliases at zero per-document storage cost. Aliases pair naturally with the routed-single-index model (§4): one alias per tenant, each pinned to its routing key, gives a per-tenant URL surface without per-tenant indices.

The trap: filtered aliases are a *read* convenience, not a security boundary on writes. A `POST /documents-acme-corp/_doc` is allowed and will inherit the alias's routing, but the application can also write to `/documents/_doc` and bypass everything. Combine with role permissions (§6) if you need write enforcement.

---

## 6. Document-Level and Field-Level Security

Application-layer filtering is fine until someone forgets it. The engine-level fix is to attach the tenant filter to the *role* the user authenticates as, so it is applied to every query that role makes.

### 6.1 Elastic Stack — paid feature, verify your license

In the commercial Elastic Stack, **document-level security (DLS)** and **field-level security (FLS)** are part of Elasticsearch Security and require a **Platinum** or **Enterprise** subscription. They are not in the free Basic tier as of the published Elastic subscription matrix. Consult the current Elastic subscription page before designing around them — license tiers shift.

```http
POST /_security/role/acme_reader
{
  "indices": [
    {
      "names":  ["documents"],
      "privileges": ["read"],
      "query":  { "term": { "tenant_id": "acme-corp" } },
      "field_security": {
        "grant":  ["title", "body", "created_at"],
        "except": ["internal_score"]
      }
    }
  ]
}
```

A user assigned `acme_reader` cannot see other tenants' documents and cannot read `internal_score` regardless of how they query. The filter is composed into every search at the shard level.

### 6.2 OpenSearch — free, in the security plugin

OpenSearch's Security plugin (Apache 2.0 licensed, bundled with the distribution) provides DLS and FLS at no cost. The role definition lives in the security configuration and is shaped similarly:

```yaml
acme_reader:
  index_permissions:
    - index_patterns: ["documents"]
      allowed_actions: ["read"]
      dls: '{"term": {"tenant_id": "acme-corp"}}'
      fls: ["~internal_score"]
```

The `~` prefix in `fls` excludes a field; without `~` the list is a positive grant.

### 6.3 The performance cost

DLS adds a `bool` filter to every query the role executes — usually cheap, since `term` on a keyword is a postings intersection. The cost scales with the *complexity* of the role query. A DLS clause that includes a `terms` lookup against another index, or a scripted condition, runs that lookup on every search. Keep DLS clauses to simple `term` / `terms` / `range` filters where possible, and benchmark before shipping anything heavier.

FLS has near-zero query-time cost but disables the `_source` filter cache for affected roles; the engine has to re-filter the source per request.

---

## 7. Aggregation Isolation and Resource Limits

Aggregations are the hardest part of multi-tenancy in the single-index model. A `terms` aggregation walks `.dvd` doc values for every matched doc and builds a per-shard hashmap; a deeply nested aggregation multiplies shard memory by the cross-product of bucket counts.

The engine ships two main brakes:

- **`search.max_buckets`** — hard cap on the total number of buckets a single search request can produce. Default 65,536 in modern Elasticsearch and OpenSearch. Tripping it returns an error rather than running the cluster out of heap. Tune up only when you can prove a specific tenant needs it.
- **Field data circuit breaker** and **request circuit breaker** — JVM-level guards in the indices breaker hierarchy. The request breaker (default 60% of heap) trips when an in-flight search's tracked memory exceeds the limit. The fielddata breaker exists primarily to prevent text-field fielddata from eating heap; modern mappings should use `keyword` + doc values to avoid loading text fielddata at all.

What none of these solve: a noisy tenant whose legitimate query consumes most of the request budget on a shared shard, slowing the rest of the tenants on that node. The structural fix is to take the tenant out of the shared index — see §9.

---

## 8. Quota Enforcement Lives Outside the Engine

Elasticsearch and OpenSearch do not have native per-tenant rate limiting, request budgets, or QPS quotas. The engine has cluster-wide settings (search thread pool size, queue depth) and per-request guards (the circuit breakers in §7). Per-tenant quotas — "tenant `acme-corp` gets 200 search QPS, anyone above pays the 429" — must be implemented at the **API gateway or application layer**:

- Token-bucket per `tenant_id` at the gateway.
- A search-API service that owns all queries, attaches the routing key, applies the tenant filter, and decrements a per-tenant budget before forwarding to the cluster.
- For burst protection, a separate slow-tenant queue with bounded concurrency so a runaway tenant cannot starve the search thread pool.

This is not optional in serious SaaS deployments. The cluster will not protect itself from a tenant that decides to launch 50 simultaneous deep-pagination scrolls.

---

## 9. Hybrid Patterns — Hot-Tenant Carve-Out and Per-Tenant ILM

Real SaaS tenant distributions are power-law, not uniform. The common pattern that absorbs this:

### 9.1 Hot-tenant carve-out

The bottom 99% of tenants live in one routed index (`documents-shared`) using §4 mechanics. The top 1% — "whales" whose data dominates a shard or whose query volume blocks others — get their own dedicated index (`documents-acme-corp`) with whatever shard count and node-tier placement they need. The application's tenant lookup table maps each `tenant_id` to either the shared index alias or its dedicated index, and the search-API service routes accordingly.

This combines the cheapness of the single-index model for the long tail with the isolation of index-per-tenant for the few tenants that need it. Promotions (a small tenant grows into a whale) are a reindex operation behind an alias swap.

### 9.2 Per-tenant ILM in the index-per-tenant model

When tenants have different retention requirements (one is a 7-day SaaS, another a regulated industry with 7-year retention), index-per-tenant lets you attach a different ILM policy to each tenant's index template:

```http
PUT /_ilm/policy/short-retention
{ "policy": { "phases": { "hot": { ... }, "delete": { "min_age": "7d", "actions": { "delete": {} } } } } }

PUT /_ilm/policy/long-retention
{ "policy": { "phases": { "hot": { ... }, "warm": { "min_age": "30d", ... }, "cold": { "min_age": "180d", ... } } } }
```

In the single-index models you cannot do per-tenant retention without painful per-doc deletes; ILM operates per-index.

### 9.3 Shard allocation filtering

Tenants on different SLA tiers can be physically separated by tagging nodes and pinning indices to tiers:

```http
PUT /documents-enterprise/_settings
{ "index.routing.allocation.include._tier_preference": "data_hot" }

PUT /documents-trial/_settings
{ "index.routing.allocation.include._tier_preference": "data_warm" }
```

Enterprise tenants get the SSD-backed hot tier; trial tenants get cheaper warm-tier nodes. Combined with carve-out this is the foundation of any tiered-pricing search SaaS.

---

## 10. Practical Traps

- **Routing skew → hot shards.** A routing-key model with one whale tenant produces one shard 5–10× larger than the others, with proportionally more queries hitting it. Diagnose with the `_cat/shards?v` size column; mitigate with partition routing or carve-out.
- **Cluster-state explosion.** Index-per-tenant past a few thousand indices makes master operations (mapping updates, allocation decisions) slow. Watch master CPU and `pending_tasks` queue length. The cluster-state size is exposed via `GET /_cluster/state?filter_path=metadata.indices` — measure before scaling tenant count.
- **Forgotten routing parameter.** A search without `?routing=` falls back to scattering to all shards. The query still returns correct results because of the `tenant_id` filter, but the latency is 30× what it should be. Build the routing parameter into the search-API service so application code can't omit it.
- **DLS license drift.** Teams design around Elastic DLS during a free trial, then discover at production-launch time that DLS requires Platinum. Either confirm the license is funded or build the equivalent at the application layer / use OpenSearch.
- **Cross-tenant scroll cursors.** A scroll context outlives the request that created it. If a scroll ID leaks (logs, error reports), the next user who replays it inherits the original user's view — including the original DLS filter. Treat scroll IDs as sensitive, set short scroll TTLs, and prefer `search_after` for paginated reads.
- **Per-tenant analyzers in shared indices.** You cannot have per-tenant analyzers in a single shared index — the mapping is global. Per-tenant analyzer needs force you toward index-per-tenant for the tenants that need them.
- **Reindexing whale promotion.** Moving a tenant from the shared routed index to a dedicated one requires a reindex. Use the Reindex API with a `tenant_id` filter and an alias-swap to make the cutover atomic from the application's view.

---

## Related

- _Sharding Strategy and Capacity Planning (planned)_ — the sizing rules referenced in §1 and §2 (20 shards/GB heap, 10–50 GB/shard)
- _Index Lifecycle Management (planned)_ — per-tenant ILM patterns referenced in §9.2
- [Inverted Index Internals — Postings Lists, Skip Lists, and FSTs](../foundations/01-inverted-index-internals.md) — why the FST in `.tip` is the structure that makes thousands of indices a heap problem
- [System Design — Multi-Tenancy Patterns](../../system-design/INDEX.md) — broader multi-tenant architecture beyond search
- [Security — Authorization and Access Control](../../security/INDEX.md) — RBAC primitives that DLS / FLS roles plug into
- [Performance — Tail Latency and Noisy Neighbors](../../performance/INDEX.md) — the framework behind §7's aggregation-isolation discussion

## References

- [Elasticsearch Reference — Customize document routing](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-routing-field.html) — `_routing` mapping, the `?routing=` parameter, partition routing via `index.routing_partition_size`
- [Elasticsearch Reference — Aliases (filtered + routed)](https://www.elastic.co/guide/en/elasticsearch/reference/current/aliases.html) — filter and routing options on alias creation referenced in §5
- [Elasticsearch Reference — Document-level security](https://www.elastic.co/guide/en/elasticsearch/reference/current/document-level-security.html) — the role-attached `query` clause used in §6.1; DLS is part of Elasticsearch Security
- [Elasticsearch Reference — Field-level security](https://www.elastic.co/guide/en/elasticsearch/reference/current/field-level-security.html) — the `field_security` block (`grant` / `except`) used in §6.1
- [Elastic Subscriptions](https://www.elastic.co/subscriptions) — authoritative source for which security features are Basic vs Platinum vs Enterprise; verify the current matrix before designing
- [OpenSearch — Security plugin: Document-level security](https://opensearch.org/docs/latest/security/access-control/document-level-security/) — DLS in OpenSearch's Apache-2.0 security plugin (no separate license)
- [OpenSearch — Security plugin: Field-level security](https://opensearch.org/docs/latest/security/access-control/field-level-security/) — FLS configuration syntax referenced in §6.2
- [Elasticsearch Reference — Size your shards](https://www.elastic.co/guide/en/elasticsearch/reference/current/size-your-shards.html) — the "20 shards per GB heap" and "10–50 GB per shard" guidance cited in §1
- [Elasticsearch Reference — Circuit breaker settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html) — request and fielddata breakers cited in §7
- [Elasticsearch Reference — `search.max_buckets`](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket.html) — aggregation bucket cap default of 65,536 cited in §7
- [Elastic Blog — Multi-tenancy in Elasticsearch](https://www.elastic.co/blog/multi-tenancy-in-elasticsearch) — the engineering tradeoffs between index-per-customer and shared indices that frame §1–§3
