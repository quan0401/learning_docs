---
title: "Lucene Segments and Merge Policy in Production"
date: 2026-05-07
updated: 2026-05-07
tags: [search, lucene, elasticsearch, opensearch, segments, merge, refresh, translog]
---

# Lucene Segments and Merge Policy in Production

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `lucene` `elasticsearch` `opensearch` `segments` `merge` `refresh` `translog`

---

## Table of Contents

- [Summary](#summary)
- [1. The Indexing Lifecycle — Buffer to Durable Segment](#1-the-indexing-lifecycle--buffer-to-durable-segment)
- [2. The Translog — Durability Without Paying Flush Cost Per Write](#2-the-translog--durability-without-paying-flush-cost-per-write)
- [3. Refresh Interval — The Visibility Knob](#3-refresh-interval--the-visibility-knob)
- [4. TieredMergePolicy — How Segments Get Compacted](#4-tieredmergepolicy--how-segments-get-compacted)
- [5. Force-Merge — When Justified, When Dangerous](#5-force-merge--when-justified-when-dangerous)
- [6. Segment Count vs Heap — Why Tiny Segments Hurt](#6-segment-count-vs-heap--why-tiny-segments-hurt)
- [7. Deletes as Tombstones — The `.liv` File](#7-deletes-as-tombstones--the-liv-file)
- [8. Read-After-Write Semantics — The Tax for Hybrid Workflows](#8-read-after-write-semantics--the-tax-for-hybrid-workflows)
- [9. Practical Patterns](#9-practical-patterns)
- [Related](#related)
- [References](#references)

---

## Summary

A Lucene index is an append-only stack of immutable segments plus a write-ahead log called the **translog** (in Elasticsearch/OpenSearch terminology). Documents arriving from a bulk request first land in an in-memory buffer and the on-disk translog; a periodic **refresh** (default 1s) turns the buffer into a new in-memory segment that is searchable but not yet fsync'd; a periodic **flush** writes the segment to disk, fsyncs it, and truncates the translog. Three timers control everything you care about in production: `refresh_interval` (search visibility), `translog.sync_interval` / `translog.durability` (durability), and the merge policy (segment count). Tune them in the wrong direction and you get either lost writes or a search cluster that runs at 10× the cost it should.

For the TS/Node backend dev: think of refresh as "commit visibility" in a relational read-replica setup — the writer has the data, but readers don't see it until the next refresh tick. Think of the translog as the WAL in PostgreSQL — it's how you survive a node crash between flushes. The merge policy is what compacts old WAL-equivalent state; force-merging it manually has the same "looks helpful, often harmful" character as `VACUUM FULL`.

---

## 1. The Indexing Lifecycle — Buffer to Durable Segment

A single document indexed via `POST /my-index/_doc` traverses six stages on its way to durable, searchable storage:

```text
1. Request arrives → IndexWriter
2. Doc analyzed → in-memory indexing buffer (per shard)
3. Doc appended to translog (durability point — see §2)
4. REFRESH (every refresh_interval): buffer → in-memory segment, searchable
5. FLUSH (when translog hits flush_threshold_size, or periodically):
   in-memory segment → on-disk segment, fsync'd, translog truncated
6. MERGE (background): N small segments → 1 larger segment
```

Stages 4 and 5 are decoupled. **Refresh** makes a doc *searchable*. **Flush** makes it *durable across a process restart without translog replay*. Between refresh and flush, the doc is searchable but, if the JVM dies, will be recovered from the translog at startup. Between the request returning and the next refresh, the doc is durable (assuming `translog.durability=request`) but not yet visible to search — this is the read-after-write surprise covered in §8.

The `IndexWriter` itself is single-writer per shard. Concurrency comes from the per-shard write thread pool; there is no merging multiple writers into one segment from a single shard's perspective.

---

## 2. The Translog — Durability Without Paying Flush Cost Per Write

A Lucene `commit()` (the operation that fsyncs a segment to disk) is expensive — it forces the OS page cache, updates the segment manifest, and is by design infrequent. Doing it per request would cap throughput at single-digit thousands of writes per second. The translog is the workaround: every indexing operation is appended to a per-shard log file *before* the request returns, and that log is replayed on startup to reconstruct anything that hadn't been flushed.

### 2.1 The two durability knobs

`index.translog.durability` controls *when* the translog is fsync'd:

| Value | Behaviour | Default? |
|-------|-----------|----------|
| `request` | fsync and commit translog after every index/bulk/delete request before acknowledging | **yes** |
| `async` | fsync in the background every `index.translog.sync_interval` (default `5s`) |  |

`index.translog.sync_interval` controls the async tick (default `5s`). It is ignored when `durability=request`.

`index.translog.flush_threshold_size` controls when the translog forces a Lucene flush (writing in-memory segments to disk so the translog can be truncated). Default **`10gb`**. Hit this and Elasticsearch flushes immediately regardless of timers.

### 2.2 The throughput-vs-durability tradeoff

```text
durability=request    : ~1 fsync per bulk; safe across crash; bulk throughput dominates
durability=async + 5s : up to 5s of writes can be lost on crash; ~5s of writes amortize
                        into a single fsync; 2-5× throughput improvement on write-heavy
                        log/metric ingestion
```

For OLTP-adjacent workloads (orders, payments, anything you'd put in Postgres), keep `request`. For log/metric ingestion where 5 seconds of loss on a node crash is acceptable, `async` is the well-trodden path — it is what most Elastic-stack logging deployments run.

### 2.3 What the translog actually contains

It contains the *operations* (index/update/delete with the source doc), not the segment files. Recovery is a replay: read the translog from the last successful checkpoint, re-apply each op against the IndexWriter, then the shard is consistent with the pre-crash state.

---

## 3. Refresh Interval — The Visibility Knob

`index.refresh_interval` defaults to **`1s`** in self-managed Elasticsearch and **`5s`** in Elastic Cloud Serverless. Every tick, the in-memory indexing buffer becomes a new searchable segment.

### 3.1 Why raise it

A refresh creates a new segment. Each new segment is one more `.tip` FST in heap, one more set of postings to consult on the next query, and eventually one more candidate for merging. At 1s default with steady write traffic you generate **3,600 segments per shard per hour** before merges intervene. Most of those will be tiny. For write-heavy indices that don't need sub-second visibility (logs, metrics, analytics), raising the interval to `30s` cuts the segment churn by 30× and frees both heap and merge IO.

### 3.2 `refresh_interval=-1` for bulk loads

The well-known reindex pattern:

```http
PUT /my-index/_settings
{ "index": { "refresh_interval": "-1", "number_of_replicas": 0 } }

# ... run the bulk reindex ...

PUT /my-index/_settings
{ "index": { "refresh_interval": "1s", "number_of_replicas": 1 } }
POST /my-index/_refresh
POST /my-index/_forcemerge?max_num_segments=1
```

Disabling refresh entirely during bulk loads means the indexing buffer flushes only when full, producing a small number of large segments instead of thousands of tiny ones. This is one of the most consequential performance moves in the platform — order-of-magnitude faster reindex, and the resulting segment shape is nearly optimal.

---

## 4. TieredMergePolicy — How Segments Get Compacted

Lucene's `TieredMergePolicy` is the default since Lucene 4.x and is what every Elasticsearch/OpenSearch index uses unless overridden. It groups segments into roughly logarithmic tiers (size buckets) and prefers merging within a tier.

Defaults from the Lucene `TieredMergePolicy` Javadoc (verified against Lucene 9.10):

| Setting | Default | Meaning |
|---------|---------|---------|
| `segmentsPerTier` | **10.0** | Target segments per size tier; lower = fewer segments + more aggressive merging + higher IO |
| `maxMergeAtOnce` | **10** | Max segments combined in one merge (regular merge) |
| `maxMergedSegmentMB` | **5 GB** | Cap on segment size produced by background merges; segments at this cap are not eligible for further merges |
| `floorSegmentMB` | **2 MB** | Segments below this size are treated as if equal to this size for tier-grouping purposes |
| `deletesPctAllowed` | **20** | If percentage of deleted docs in the index exceeds this, merges are scheduled aggressively to reclaim |

The 5 GB cap is intentional: a single 50 GB segment would block the merge thread for hours, balloon recovery time, and prevent fine-grained replica shipping. By capping at 5 GB, Lucene keeps merge cost predictable.

### 4.1 What you actually tune

In practice the useful knobs are:
- **`max_merged_segment`** — lower (e.g. 2 GB) on indices where shard relocation must be quick; raise (e.g. 10 GB) for read-heavy indices where fewer segments matter more than relocation speed.
- **`segments_per_tier`** — raise (e.g. 30) if you can tolerate more segments to reduce merge IO; lower (e.g. 4) if heap pressure from FSTs is the bottleneck.

Most production deployments leave defaults. Bad merge tuning is a top-three cause of cluster pathologies on the Elastic forums — the defaults are tuned by people whose full-time job is tuning them.

---

## 5. Force-Merge — When Justified, When Dangerous

`POST /my-index/_forcemerge?max_num_segments=1` instructs Lucene to merge all segments in each shard down to N. It is the manual override of the policy in §4.

### 5.1 The legitimate use case

**Read-only indices.** Time-based indices (logs, metrics, audit trails) where yesterday's index will never receive another write. After the rollover boundary, force-merge yesterday's index to one segment per shard. You get:
- Maximum compression efficiency (one large segment compresses better than many small ones)
- Minimum heap (one FST per shard instead of dozens)
- Maximum query speed (one term lookup per shard instead of dozens)

This is the happy case and is documented as such by Elastic.

### 5.2 The dangerous cases

**Force-merging an index that still receives writes.** The single huge segment created by force-merge is now the floor: future tiny segments cannot merge with it (it is above `max_merged_segment`), so deletes against the huge segment will not be reclaimed and the index degrades over time. The Elastic docs explicitly warn against this.

**Force-merging during peak hours.** A force-merge to one segment on a 50 GB shard rewrites 50 GB of disk, evicts that much from the page cache, and runs the merge thread hot for the duration. Search latency suffers across the cluster.

**Force-merging without `max_num_segments`.** Without the parameter, the call merges any segments larger than the floor — usually not what you want.

The rule of thumb: force-merge is for indices that have crossed a write boundary, run during low-traffic windows, and never on a hot index.

---

## 6. Segment Count vs Heap — Why Tiny Segments Hurt

Each open segment keeps its terms-index FST (`.tip` — see [01-inverted-index-internals.md](./01-inverted-index-internals.md) §3) resident in heap. The FST is small per segment, but multiplied across thousands of segments per shard and dozens of shards per node, it dominates JVM heap usage.

A pathology pattern from the field:

```text
Index with refresh_interval=1s, low write rate, never bulk-loaded
  → 100s of tiny segments per shard
  → 100s of FSTs in heap per shard
  → JVM heap pressure, GC pauses, slow queries
  → operator force-merges, now back to 1 segment per shard
  → heap drops 70%, query latency drops 5×
```

The other axis: **filesystem cache competition**. Each segment's `.doc`/`.pos`/`.pay` files need to live in OS page cache for queries to be fast. More segments means more files, less sharing, worse cache locality. This is why operators monitor `_cat/segments` and treat segment-count spikes as a leading indicator of heap exhaustion.

---

## 7. Deletes as Tombstones — The `.liv` File

A delete in Lucene does not remove anything from the postings lists. It writes a bit into the segment's `.liv` (live-docs) bitmap marking that doc ID as deleted; queries walk the postings as before and skip docs whose `.liv` bit is unset.

Reclamation happens **only on merge**. When TieredMergePolicy merges segments S1 and S2 into S3, the deleted docs from S1 and S2 are dropped — they never enter S3. This is why `deletesPctAllowed` (default 20%) exists: above that threshold, the merge policy schedules merges aggressively specifically to reclaim deleted-doc space.

Operational consequence: a delete-heavy workload (e.g. updating documents frequently — every update is delete + insert) will keep growing on disk unless merges keep up. The `_stats` API exposes `deleted_docs` per shard; if it's drifting upward over hours, you have a merge-throughput problem.

---

## 8. Read-After-Write Semantics — The Tax for Hybrid Workflows

The most counterintuitive thing about Elasticsearch/OpenSearch for someone arriving from Postgres or MySQL: **a successful index response does not mean the document is searchable**. The doc is in the buffer + translog (durable), but search will not find it until the next refresh.

### 8.1 The three escape hatches

```http
# 1. ?refresh=true  — force an immediate refresh (expensive, do not use on hot path)
POST /my-index/_doc?refresh=true { ... }

# 2. ?refresh=wait_for  — block the request until the next scheduled refresh
POST /my-index/_doc?refresh=wait_for { ... }

# 3. GET /my-index/_doc/{id}  — realtime GET reads the in-memory buffer + translog
#    NOTE: this works for _doc by ID only, not for _search queries
```

`refresh=wait_for` is the right choice in tests and most user-facing flows: no extra IO, just latency up to one refresh interval. `refresh=true` forces a new segment creation and triggers all the segment-count costs in §6 — never use it under load.

### 8.2 Implications for tests

A common failing test pattern:

```ts
await client.index({ index: 'orders', id: '1', body: order });
const result = await client.search({ index: 'orders', body: { query: { match: { id: '1' } } } });
// FAILS — search has not seen the doc yet
```

Either use `refresh: 'wait_for'` on the index call or `await client.indices.refresh({ index: 'orders' })` after writes. Adding sleeps is the wrong fix.

### 8.3 Implications for hybrid OLTP+search

A common architecture: write-of-record in Postgres, denormalized projection into Elasticsearch via CDC. The search index is **eventually consistent** with Postgres. UI flows that "save then search" need to either: read-back from Postgres for the immediate confirmation, use `refresh=wait_for` on the projection write, or accept the visibility gap. Designing the UI to assume sub-second-but-not-instant search visibility is the durable answer.

---

## 9. Practical Patterns

### 9.1 Bulk index → refresh → search

```ts
await client.indices.putSettings({
  index: 'products',
  body: { index: { refresh_interval: '-1', number_of_replicas: 0 } },
});

for (const batch of chunks(allProducts, 5000)) {
  await client.bulk({ index: 'products', body: bulkBody(batch) });
}

await client.indices.putSettings({
  index: 'products',
  body: { index: { refresh_interval: '1s', number_of_replicas: 1 } },
});
await client.indices.refresh({ index: 'products' });
await client.indices.forcemerge({ index: 'products', max_num_segments: 1 });
```

This is the canonical reindex/seed pattern. Refresh disabled during ingest, replicas off (recreated from primary at the end), force-merge once at the very end.

### 9.2 Freeze old time-based indices

For daily indices `logs-2026.05.06`, `logs-2026.05.05`, …:

```http
POST /logs-2026.05.05/_forcemerge?max_num_segments=1
PUT  /logs-2026.05.05/_settings { "index": { "blocks": { "write": true } } }
```

Once force-merged and write-blocked, the old index is at minimum heap and disk cost. ILM (Index Lifecycle Management) automates this — the manual API calls above are what ILM runs underneath. Deeper coverage belongs in a Tier 4 Production Patterns doc.

### 9.3 Pick durability per workload

```yaml
# OLTP-adjacent (orders, accounts, payments projections)
index.translog.durability: request
index.refresh_interval: 1s

# Log/metric ingestion
index.translog.durability: async
index.translog.sync_interval: 5s
index.refresh_interval: 30s
```

The first set is the safe default. The second is the high-throughput ingestion preset that explicitly accepts up to 5s of write loss on a node crash in exchange for several-fold throughput.

---

## Related

- [01 — Inverted Index Internals](./01-inverted-index-internals.md) — segment file family (`.tim`, `.tip`, `.doc`, `.liv`) and the FST in `.tip` whose heap cost §6 quantifies
- _02 — BM25 and TF-IDF (planned)_ — relevance scoring on top of the segment-resident postings
- _03 — Analyzers (planned)_ — the write-path stage that produces tokens fed into the buffer in §1
- _05 — Index Aliases and Rollover (planned)_ — how time-based indices use the freeze-and-force-merge pattern from §9.2 in production
- [Database — Polyglot Persistence](../../database/INDEX.md) — the broader context for hybrid Postgres + Elasticsearch architectures touched on in §8.3
- [Operating Systems — Page Cache](../../operating-systems/operating-systems-tier1/03-page-cache-and-buffered-io.md) — the OS-level reason §6's segment-count argument also affects cache hit rate
- [Performance — Tail Latency](../../performance/INDEX.md) — frame for reasoning about merge-induced latency spikes in §5

## References

- [Apache Lucene 9.10 — `TieredMergePolicy` Javadoc](https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/index/TieredMergePolicy.html) — authoritative source for the §4 defaults: `segmentsPerTier=10.0`, `maxMergeAtOnce=10`, `maxMergedSegmentMB=5 GB`, `floorSegmentMB=2 MB`, `deletesPctAllowed=20`
- [Elasticsearch Reference — Translog](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-translog.html) — confirms `index.translog.durability` default `request`, `sync_interval` default `5s`, `flush_threshold_size` default `10gb`
- [Elasticsearch Reference — Index modules](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html) — confirms `index.refresh_interval` default `1s` (self-managed) / `5s` (Serverless), and `-1` to disable
- [Elasticsearch Reference — Force merge API](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-forcemerge.html) — official guidance on read-only-only force merge, the warning behind §5.2
- [Elasticsearch Reference — Refresh API and `?refresh` parameter](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-refresh.html) — defines `true` / `wait_for` / `false` semantics used in §8.1
- [Apache Lucene 9.10 — Index File Format](https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/codecs/lucene99/package-summary.html) — `.liv` (live docs) referenced in §7
- [Elastic Blog — Elasticsearch from the Bottom Up (segments and merging)](https://www.elastic.co/blog/found-elasticsearch-from-the-bottom-up) — the foundational walkthrough of the buffer → refresh → flush lifecycle in §1
- [OpenSearch Documentation — Index settings](https://docs.opensearch.org/latest/install-and-configure/configuring-opensearch/index-settings/) — OpenSearch inherits the same translog/refresh/merge semantics from its Lucene-7-era Elasticsearch fork
