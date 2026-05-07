---
title: "Index Lifecycle Management and the Hot/Warm/Cold/Frozen Tier Model"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, ilm, ism, tiers]
---

# Index Lifecycle Management and the Hot/Warm/Cold/Frozen Tier Model

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `ilm` `ism` `tiers`

---

## Table of Contents

- [Summary](#summary)
- [1. Why ILM Exists — Time-Series Outgrows a Single Index](#1-why-ilm-exists--time-series-outgrows-a-single-index)
- [2. Data Tiers and Node Roles](#2-data-tiers-and-node-roles)
- [3. ILM Phases and Allowed Actions](#3-ilm-phases-and-allowed-actions)
- [4. Rollover and the `is_write_index` Alias Pattern](#4-rollover-and-the-is_write_index-alias-pattern)
- [5. Index Templates and Policy Attachment](#5-index-templates-and-policy-attachment)
- [6. Searchable Snapshots — What Frozen Is Doing Underneath](#6-searchable-snapshots--what-frozen-is-doing-underneath)
- [7. OpenSearch ISM — The Equivalent and Where It Diverges](#7-opensearch-ism--the-equivalent-and-where-it-diverges)
- [8. Practical Patterns — Logs, Metrics, Sizing](#8-practical-patterns--logs-metrics-sizing)
- [9. Cost Model — Hot Is Expensive, Frozen Is Cheap](#9-cost-model--hot-is-expensive-frozen-is-cheap)
- [Related](#related)
- [References](#references)

---

## Summary

Time-series data — application logs, infra metrics, audit events, click streams — does not fit the "one index forever" mental model that Elasticsearch's getting-started examples show. A single index that ingests 100 GB/day for a year accumulates 36 TB on disk, ten thousand-plus segments per shard, and a query surface that no single hot node can keep paged in. **Index Lifecycle Management (ILM)** is Elastic's answer: a policy DSL that rolls a write-target index over when it hits a size or age threshold, then progressively migrates the now-read-only older indices through cheaper hardware tiers — `hot` (NVMe + RAM, accepts writes), `warm` (slower SSD, read-only), `cold` (spinning disk or fewer replicas), `frozen` (S3-backed via searchable snapshots, mostly-on-demand) — before finally deleting them. **OpenSearch's Index State Management (ISM)** is the same idea with different names and a JSON state-machine shape.

The single most useful framing for backend engineers coming from a relational database background: **an Elasticsearch "index" in a logs cluster is closer to a table partition than a table.** You do not write into one giant index; you write into the *current* partition (the write-target alias), and ILM handles partition rotation, compaction (force_merge), demotion to cheaper storage (allocate), and retention (delete). This doc walks the tier model, the phase/action grammar, and the production patterns that actually ship.

---

## 1. Why ILM Exists — Time-Series Outgrows a Single Index

Three operational realities push you toward ILM the moment a cluster starts ingesting time-series:

- **Shards have a sweet spot.** Elastic's sizing guidance puts primary shard size in the 10–50 GB range for most search workloads (Elastic's "Size your shards" guide is the canonical source). A single index that grows unboundedly violates this within weeks; you either start with too many shards (over-shards small days, wasting heap on FSTs) or too few (over-shards large days, hitting per-shard memory ceilings).
- **Read patterns differ across age.** A 5-minute-old log line is queried by an alerting rule 50 times per minute; a 5-month-old log line is queried by an annual audit, maybe twice. Paying NVMe + replicated-RAM prices for both is wasteful.
- **Retention is a separate concern from ingestion.** Compliance ("delete after 90 days"), cost ("we cannot afford to keep this hot"), and product ("show last 7 days in the UI") all want different cutoffs. Hard-coding these into application code couples retention policy to deploys.

ILM separates these concerns. The application writes to a single alias (`logs-app`); ILM rotates the underlying indices, demotes them across tiers, and deletes them on schedule. This is the same conceptual move as PostgreSQL declarative partitioning with a retention worker, but built into the search engine rather than bolted on.

---

## 2. Data Tiers and Node Roles

Elastic's data tiers are an explicit cluster-level concept since 7.10. A node can hold one or more tier roles via its `node.roles` setting:

| Tier | Node role | Storage profile | Workload |
|------|-----------|-----------------|----------|
| Hot | `data_hot` | Fastest disks (NVMe SSD), most RAM, replicated | Accepts indexing; serves recent reads |
| Warm | `data_warm` | SSD or fast spinning disk, less RAM, replicated | Read-only; less-frequent reads |
| Cold | `data_cold` | Cheaper storage; often searchable-snapshot-backed with one replica or fewer | Rarely accessed |
| Frozen | `data_frozen` | Object storage (S3, GCS, Azure Blob) via searchable snapshots; tiny on-disk footprint | Search-on-demand only |

The legacy generic `data` role implicitly grants all four tier roles, which is fine for small clusters but explicit roles let you scale tiers independently — buy six high-RAM hot nodes, twelve cheap warm nodes, three frozen nodes whose disks only hold the searchable-snapshot cache.

The **content tier** (`data_content`) is a fifth role for non-time-series indices (product catalogs, user data) where the hot/warm/cold rotation does not apply; ILM phases simply do not move data off it. The four tiers above are the time-series rotation; content is the everything-else bucket.

Allocation between tiers is driven by index settings — `index.routing.allocation.include._tier_preference: data_warm,data_hot` tells the cluster "prefer warm, fall back to hot." ILM's `allocate` action is what writes that setting during a phase transition.

---

## 3. ILM Phases and Allowed Actions

An ILM policy is an ordered sequence of phases, each of which fires when the index ages into it. The phases, from Elastic's ILM reference:

1. **Hot** — index is actively written and read. Allowed actions: `rollover`, `set_priority`, `unfollow`, `readonly`, `shrink`, `forcemerge`, `searchable_snapshot`, `downsample`.
2. **Warm** — index is read-only and recent. Allowed actions: `set_priority`, `unfollow`, `readonly`, `allocate`, `migrate`, `shrink`, `forcemerge`, `downsample`.
3. **Cold** — index is rarely read. Allowed actions: `set_priority`, `unfollow`, `readonly`, `allocate`, `migrate`, `searchable_snapshot`, `downsample`.
4. **Frozen** — index is searchable only via snapshot. Allowed actions: `searchable_snapshot`.
5. **Delete** — index is removed. Allowed actions: `wait_for_snapshot`, `delete`.

The grammar is strict: you cannot, for example, run `rollover` outside hot, because rollover only makes sense for the write-target index. The actions themselves are the building blocks:

- **`rollover`** — when a trigger fires (`max_primary_shard_size`, `max_age`, `max_docs`, `max_primary_shard_docs`), creates a new index with an incremented suffix and points the write alias at it. This is hot-only.
- **`shrink`** — copies the index into a new index with fewer primary shards (must divide evenly into the original). Used when you over-sharded the hot phase but want fewer shards in warm/cold to cut heap and file-handle pressure.
- **`forcemerge`** — merges segments down to a target count (typically 1) for read-only segments. Smaller segment count = smaller terms-dictionary FSTs in heap = faster aggregations. Expensive — only run on indices that won't be written to again.
- **`readonly`** — sets `index.blocks.write: true`. Required before some actions.
- **`allocate`** — changes shard placement and replica count. The `migrate` shorthand action sets `_tier_preference` to the current phase's tier; `allocate` is the manual equivalent that also lets you change replica count.
- **`searchable_snapshot`** — takes a snapshot to a configured repository and (in cold) mounts a partial cache of it, or (in frozen) mounts only a tiny on-disk cache backed by the snapshot.
- **`downsample`** — Time Series Data Stream (TSDS) action that aggregates documents to a coarser time bucket (e.g., 10s metrics → 1m metrics), trading resolution for size. Available since 8.5 for TSDS indices.
- **`delete`** — removes the index. The `wait_for_snapshot` companion action ensures a snapshot has captured the index before deletion.

Each action has a `min_age` (relative to rollover, by default — see the docs for the exact origin point) gating when the phase fires.

---

## 4. Rollover and the `is_write_index` Alias Pattern

Rollover is the lynchpin. The pattern: the application writes to a stable alias (`logs-app`); the alias points at a concrete index (`logs-app-000001`); when the index hits a rollover trigger, ILM creates `logs-app-000002`, points the alias at it as the new write target, and leaves the old index reachable for reads.

Two flavors:

- **Aliases with `is_write_index`.** A single alias points at multiple indices; one is marked `is_write_index: true`. Indexing requests go only to the write index; search requests fan out to all indices the alias names. This is the older, lower-level mechanism.
- **Data streams.** Since 7.9, data streams wrap the alias-and-rollover dance into a first-class abstraction. You write to `logs-app` (a data stream); reads fan out across the backing indices `.ds-logs-app-2026.05.07-000001` etc. ILM owns rollover entirely. For new builds, use data streams unless you have a reason not to. Data streams are append-only — you cannot update or delete-by-id, only delete the whole backing index (which ILM does for you).

A minimal rollover trigger:

```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "7d",
            "max_primary_shard_docs": 200000000
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

`max_primary_shard_size` is the recommended primary trigger because it directly enforces the shard sweet spot; `max_age` is the safety net for slow-write indices; `max_primary_shard_docs` guards against single-shard pathologies. Whichever fires first triggers rollover.

---

## 5. Index Templates and Policy Attachment

ILM policies attach to indices via index templates (or, for data streams, via the data stream's backing template). The template specifies `index.lifecycle.name` (the policy) and, for non-data-stream rollover, `index.lifecycle.rollover_alias` (the write alias).

```json
PUT _index_template/logs-app-template
{
  "index_patterns": ["logs-app-*"],
  "data_stream": {},
  "template": {
    "settings": {
      "index.lifecycle.name": "logs-30d",
      "index.number_of_shards": 2,
      "index.number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message":    { "type": "text" },
        "level":      { "type": "keyword" }
      }
    }
  }
}
```

When a new backing index is created, the template's settings — including the lifecycle policy — are baked into it. Changing the policy after the fact updates *all* indices currently using it on the next phase evaluation; this is how you extend retention from 30 to 60 days without reindexing.

---

## 6. Searchable Snapshots — What Frozen Is Doing Underneath

The frozen tier's pitch — "10× cheaper, search still works" — depends on **searchable snapshots**, a feature where an index's segments live in object storage (S3, GCS, Azure Blob) and the cluster mounts only a small local cache.

Two mount modes from the searchable snapshots reference:

- **Fully cached (`full_copy`)** — the entire snapshot is cached on local disk. Used by the cold tier. Disk cost is the same as a normal index, but you can drop replicas because the snapshot itself is the durability layer.
- **Partially cached (`shared_cache`)** — only a small disk cache (default a few hundred GB cluster-wide) is used; segment data is fetched on-demand from object storage. Used by the frozen tier. Disk cost is tiny, but query latency is whatever S3 round-trips take for a cache miss.

Frozen tier queries are slower — first-touch latency for a cold range can be seconds rather than milliseconds — but the storage cost difference is dramatic. A 30 TB warm index might cost thousands of dollars per month in EBS or local SSD; the same data as a frozen searchable snapshot is essentially S3 storage cost plus the small frozen-node footprint.

The `searchable_snapshot` ILM action takes a snapshot to a configured repository, deletes the local index, and mounts the snapshot in the appropriate cache mode for the phase. This is destructive — once an index is converted to a searchable snapshot, the original local segments are gone; the snapshot repository is the source of truth.

---

## 7. OpenSearch ISM — The Equivalent and Where It Diverges

OpenSearch (the AWS fork after the 2021 license change) ships **Index State Management** in place of ILM. Conceptually identical: a policy moves an index through states, each with actions and transitions, with rollover, force_merge, replica counts, allocation, snapshot, and delete all available.

Differences worth knowing:

- **State machine, not phase list.** ISM policies are an explicit state graph — you define states and transitions between them. ILM is a fixed phase sequence. ISM is more flexible (skip phases, branch on conditions); ILM is more opinionated and gives you data-stream-aware tooling out of the box.
- **No first-class `frozen` tier with searchable snapshots in mainline OpenSearch.** OpenSearch supports remote-backed indices and searchable snapshots, but the tier vocabulary and ILM-style automation around them are evolving differently from Elastic. AWS-managed OpenSearch Service offers the UltraWarm and cold-storage features as managed equivalents.
- **Action names differ.** ISM uses `rollover`, `force_merge`, `read_only`, `replica_count`, `allocation`, `notification`, `snapshot`, `delete` — overlapping but not identical with ILM's vocabulary.
- **Aliases vs data streams.** OpenSearch supports data streams as of 2.x; older clusters still rely on alias-and-rollover patterns.

For application code, the abstraction is the same — write to an alias or data stream, let the management layer rotate underneath. The policy JSON is what you swap.

---

## 8. Practical Patterns — Logs, Metrics, Sizing

Three patterns cover the bulk of real-world deployments.

### 8.1 Application logs — hot 7d, warm 30d, delete

```text
hot   (writes + recent reads)
  rollover at 50 GB or 7 days
  ↓ at 7 days
warm  (read-only, force_merge to 1 segment, reduce replicas)
  ↓ at 30 days
delete
```

Indexing is concentrated in hot; the warm phase trims storage cost and segment-count heap pressure; deletion fires at the compliance cutoff. No cold/frozen tier — the data isn't queried often enough past 30 days to justify keeping it.

### 8.2 Metrics — hot 1d, cold 90d, frozen 1y, delete

```text
hot    (writes + dashboards last 24h)
  rollover at 50 GB or 1 day
  ↓ at 1 day
cold   (searchable snapshot, fully cached, replica=0)
  ↓ at 90 days
frozen (searchable snapshot, partially cached)
  ↓ at 1 year
delete
```

Metrics queries skew very recent (Grafana refreshes the last 15 min on every render); historical metrics are queried by humans investigating an incident or by capacity planners. Cold + frozen pays off heavily because the bytes-per-query of historical metrics is tiny.

### 8.3 Sizing rules of thumb

- Aim for **10–50 GB primary shard size** in the hot phase. Below 10 GB you waste heap on FSTs; above 50 GB recovery time and merge cost balloon.
- Aim for **20 shards per GB of JVM heap** as an upper ceiling for open shards per node. Push past this and you start losing search threads to per-shard overhead.
- Force-merge to **1 segment per shard** in warm/cold; do not force-merge in hot. Force-merge writes a single huge segment that, until it ages out, blocks future merges.
- Set **`index.codec: best_compression`** on warm indices. The default codec optimizes for indexing speed; warm indices are read-only so the slower-but-smaller `DEFLATE`-based codec is a free win.
- **Drop replicas in cold/frozen** when backed by searchable snapshots. The snapshot repository is the durability layer, not the cluster's replicas.

---

## 9. Cost Model — Hot Is Expensive, Frozen Is Cheap

The order-of-magnitude cost model that makes ILM worth doing:

| Tier | Storage cost driver | Relative bytes-per-month cost |
|------|---------------------|-------------------------------|
| Hot | NVMe SSD, replicated, RAM-resident segments | ~10× |
| Warm | SSD or fast spinning, replicated | ~3–5× |
| Cold | Local disk, searchable snapshot fully cached, replica=0 | ~1–2× |
| Frozen | Object storage + tiny local cache | ~0.1–0.3× |

Numbers are illustrative — actual ratios depend on cloud provider, instance family, and snapshot repository pricing — but the shape is real and well-documented in Elastic's tier-cost guidance. A retention policy that keeps everything hot for a year costs roughly 10–100× a tiered policy that demotes after a week. The query cost model goes the opposite direction (hot queries milliseconds, frozen queries seconds), so the engineering question is always "what's the read pattern?", not "how cheap can I get the storage?".

The lever ILM gives you is matching tier cost to query frequency, automatically. Tier 6 (Production Patterns) of this learning path will revisit capacity planning with specific cluster sizing examples.

---

## Related

- [Lucene Segments and Merge Policy (planned)](./05-segments-and-merges.md) — what `forcemerge` is actually doing under the hood
- [Cluster Topology — Master, Data, Coordinator, Ingest Roles (planned)](../operations/02-cluster-roles.md) — node-role configuration that the tier model relies on
- [Snapshot and Restore (planned)](../operations/05-snapshot-and-restore.md) — the repository layer that searchable snapshots build on
- [Database — Partitioning and Retention](../../database/INDEX.md) — relational analog of the rollover/delete pattern
- [Observability — Log Retention Patterns](../../observability/INDEX.md) — application-side decisions that drive ILM policy choices
- [Performance — Capacity Planning](../../performance/INDEX.md) — framework for matching tier cost to query frequency

## References

- [Elasticsearch Reference — Index lifecycle management overview](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html) — authoritative source for phases, actions, and trigger semantics
- [Elasticsearch Reference — Index lifecycle actions](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-actions.html) — the canonical action list including `rollover`, `shrink`, `forcemerge`, `allocate`, `searchable_snapshot`, `downsample`, `delete`
- [Elasticsearch Reference — Data tiers](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-tiers.html) — `data_hot`, `data_warm`, `data_cold`, `data_frozen`, `data_content` node roles and tier preference allocation
- [Elasticsearch Reference — Searchable snapshots](https://www.elastic.co/guide/en/elasticsearch/reference/current/searchable-snapshots.html) — fully cached vs partially cached mount modes; the underlying mechanism for cold and frozen tiers
- [Elasticsearch Reference — Size your shards](https://www.elastic.co/guide/en/elasticsearch/reference/current/size-your-shards.html) — 10–50 GB primary shard guidance and shards-per-heap rules cited in §8
- [Elasticsearch Reference — Rollover](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-rollover.html) — `max_primary_shard_size`, `max_age`, `max_docs`, `max_primary_shard_docs` triggers
- [Elasticsearch Reference — Data streams](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-streams.html) — append-only abstraction over the alias-and-rollover pattern
- [Elasticsearch Reference — Downsampling a time series data stream](https://www.elastic.co/guide/en/elasticsearch/reference/current/downsampling.html) — the `downsample` action for TSDS
- [OpenSearch Documentation — Index State Management](https://opensearch.org/docs/latest/im-plugin/ism/index/) — OpenSearch's state-machine equivalent of ILM
- [OpenSearch Documentation — ISM policies](https://opensearch.org/docs/latest/im-plugin/ism/policies/) — state, action, and transition syntax
