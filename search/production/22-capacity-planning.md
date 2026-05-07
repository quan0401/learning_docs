---
title: "Capacity Planning for Elasticsearch and OpenSearch — Heap, Page Cache, Shards, and Nodes"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, capacity, sizing, heap, jvm]
---

# Capacity Planning for Elasticsearch and OpenSearch — Heap, Page Cache, Shards, and Nodes

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `capacity` `sizing` `heap` `jvm`

---

## Table of Contents

- [Summary](#summary)
- [1. The Two-Pool Model — Heap and Page Cache](#1-the-two-pool-model--heap-and-page-cache)
- [2. Heap Sizing — The 50% Rule and the Compressed-Oops Cliff](#2-heap-sizing--the-50-rule-and-the-compressed-oops-cliff)
- [3. Shard Count — The 20-Shards-per-GB-Heap Ceiling](#3-shard-count--the-20-shards-per-gb-heap-ceiling)
- [4. Shard Size — 10–50 GB, Up to 200 GB for Time Series](#4-shard-size--1050-gb-up-to-200-gb-for-time-series)
- [5. Cluster Sizing — Data Nodes from Total Data](#5-cluster-sizing--data-nodes-from-total-data)
- [6. Vertical vs Horizontal Scaling](#6-vertical-vs-horizontal-scaling)
- [7. Disk and IOPS](#7-disk-and-iops)
- [8. Network and Cross-Zone Latency](#8-network-and-cross-zone-latency)
- [9. Master and Coordinating Tiers](#9-master-and-coordinating-tiers)
- [10. Hot/Warm/Cold Tier Sizing](#10-hotwarmcold-tier-sizing)
- [11. Practical Traps](#11-practical-traps)
- [Related](#related)
- [References](#references)

---

## Summary

Capacity planning for Elasticsearch and OpenSearch is fundamentally a story about **two memory pools** — the JVM heap that holds Lucene's per-segment FSTs, doc-values caches, and request buffers, and the OS page cache that holds the mmaped postings, doc values, and stored-field files. The 50% / 50% split between them is not a vibe; it is the direct consequence of how Lucene reads its data. Get this split wrong and either GC pauses dominate latency (heap too small or too large) or every query falls off a cliff into disk I/O (page cache starved).

The other axes that matter — shard count, shard size, node count, disk type, network topology — all interact with that two-pool model. A cluster with too many small shards burns heap on FSTs and segment metadata. A cluster with shards that are too large rebalances slowly when a node fails. A cluster sized for "current data" without headroom dies the first time a query plan changes. This doc walks through the rules of thumb Elastic and OpenSearch publish, explains the **why** behind each one, and calls out the traps that bite teams in production.

The audience is a TS/Node backend developer who has run an Elasticsearch cluster behind a Spring Boot or Express service and wants to understand the capacity math before the next on-call shift, not after.

---

## 1. The Two-Pool Model — Heap and Page Cache

A Lucene segment's hottest read structures are mmaped: `.doc` (postings), `.pos` (positions), `.dvd` (doc values), `.fdt` (stored fields). The kernel pages those files into the OS page cache lazily and keeps frequently touched pages resident. The JVM heap, in parallel, holds:

- The per-segment terms-index FST (`.tip`) loaded into off-heap memory in modern Lucene versions, but still with on-heap accounting structures.
- BM25 scorers, query-rewrite intermediate results, aggregation buckets, request and response buffers.
- The field-data cache (text-field aggregations) and the request cache.

Both pools live in the same physical RAM. The kernel cannot give the page cache memory the JVM has reserved, and vice versa. So sizing is a partitioning decision, not an "add more of each" decision.

The single most consequential consequence: **a query's worst case is "term not in page cache, must fault from disk."** Even a fast NVMe page-in is hundreds of microseconds versus a sub-microsecond memory read. A cluster with hot indexes that fit in page cache routinely runs at p99 < 50 ms; the same cluster with page cache evicted runs at p99 > 500 ms with no other change.

---

## 2. Heap Sizing — The 50% Rule and the Compressed-Oops Cliff

Elastic's official guidance is to set the JVM heap to **half of the node's RAM, capped at roughly 31 GB** to stay below the JVM's compressed-ordinary-object-pointers (compressed-oops) boundary. The exact cap depends on the JVM and operating system — Elastic's "Advanced configuration / Set the JVM heap size" page recommends `-Xms` and `-Xmx` of "no more than 50% of total memory" and "below the threshold that the JVM uses for compressed object pointers; the exact threshold varies but is near 32 GB."

### 2.1 Why 50%

The other 50% is for the OS page cache that holds Lucene's mmaped files (§1). Give Elasticsearch all the RAM and the page cache is starved; cold queries fault every block from disk.

### 2.2 Why the ~32 GB cliff

The HotSpot JVM uses compressed oops to store 64-bit object references in 32-bit fields, scaled by an object alignment factor (default 8 bytes) to address up to ~32 GB of heap. Crossing that boundary forces the JVM into uncompressed 64-bit pointers, which doubles the size of every reference in every Java object. The result is counterintuitive: a 33 GB heap holds **less** working memory than a 31 GB heap, *and* GC pauses get worse because there is more pointer data to scan.

Elastic recommends the JVM flag `-XX:+UseCompressedOops` is in effect (it is the default on 64-bit JVMs up to ~32 GB) and verifying via the JVM startup logs. The practical heap caps that ship safely below the cliff are:

- 30 GB on most Linux distributions with default 8-byte object alignment.
- 31 GB on tuned JVMs.
- Elastic's own guidance reads "no more than 31 GB" in current docs — verify against the version you run.

### 2.3 What goes wrong at each boundary

| Heap setting | Symptom |
|---|---|
| Too small (<2 GB) | OOM on aggregations; FST loading fails; bulk indexing rejected |
| Right-sized (~50% of RAM, ≤31 GB) | Page cache hot, GC pauses bounded |
| Too large (>~32 GB) | Compressed oops disabled; heap effectively shrinks, GC cost rises |
| 50% of a 256 GB box | Crosses the cliff; **prefer two JVM instances per host or cap at 31 GB and donate the rest to page cache** |

For very large nodes, the canonical workaround is to run multiple Elasticsearch instances per host, each with a 31 GB heap, with `cluster.routing.allocation.same_shard.host: true` to prevent two replicas of the same shard landing on the same physical machine.

---

## 3. Shard Count — The 20-Shards-per-GB-Heap Ceiling

Elastic's "Size your shards" page recommends keeping the number of shards per node **proportional to heap**, with a rule of thumb of "no more than 20 shards per GB of heap" as an upper bound. Verify against your version — the exact figure has shifted between Elasticsearch 7.x and 8.x, and OpenSearch's parallel guidance reads similarly.

The reason is per-shard overhead. Every shard is a Lucene index with N segments; every segment keeps metadata in heap (FST accounting, segment info, field info, mappings). A node with a 31 GB heap should target **at most ~620 shards total** under that rule; many production guides recommend staying well below it (300–400) to leave headroom for query workspace.

Cluster-wide, Elasticsearch defaults `cluster.max_shards_per_node` to 1,000 — a soft ceiling that prevents runaway shard creation. Hitting it is a smell, not a normal operating point.

### 3.1 The "many small shards" anti-pattern

Splitting a 50 GB index into 50 one-GB shards seems intuitive ("more parallelism!") but is backwards:

- Every shard pays fixed coordination cost on every query.
- FST and segment metadata multiply by shard count.
- Recovery and rebalancing churn 50 file sets instead of one.

The correct shape for most workloads is **fewer, larger shards** sized within the next section's targets.

---

## 4. Shard Size — 10–50 GB, Up to 200 GB for Time Series

Elastic's published shard-size guidance:

- **Search-heavy / general workloads:** target 10–50 GB per shard.
- **Time-series / log-style read-heavy workloads:** up to 200 GB per shard (since these are often searched as recent slices and cold shards rarely hit hot paths).

The lower bound exists because of fixed per-shard overhead; the upper bound exists because:

- Segment merges on huge shards take longer and use more disk.
- Recovery (replica resync after a node failure) ships the entire shard across the network.
- Snapshot and restore times scale with shard size.

A 50 GB shard with a 1 Gb/s replica recovery link rebuilds in roughly 7 minutes; a 200 GB shard takes ~30 minutes. For an SLA-bound search service, that recovery time is the floor on your "node failure → full health" budget.

### 4.1 Picking a primary count

Given a target shard size and an estimated dataset size:

```text
primaries = ceil(estimated_size_gb / target_shard_size_gb)
```

For a 500 GB log index targeting 50 GB shards: `ceil(500 / 50) = 10` primaries. Account for growth — overshoot the primary count modestly so shards do not balloon past 200 GB before the next reindex window.

For time-series data, use **rollover** with the data-stream pattern (`logs-2026.05.07-000001`) and let each backing index hold a bounded slice; primary count then drives only how parallel a single day's writes are, not lifetime data volume.

---

## 5. Cluster Sizing — Data Nodes from Total Data

A first-pass cluster size:

```text
nodes = ceil(total_data_gb × replication_factor × headroom / per_node_disk_gb)
```

Where:

- `replication_factor` is `1 + number_of_replicas` (default 2 for one replica).
- `headroom` is typically 1.2 — Elastic's watermark defaults reserve disk at 85% (low), 90% (high), and 95% (flood-stage); going past 85% throttles allocation, and going past 95% turns indexes read-only.

Worked example: 5 TB of primary data, one replica, 2 TB of usable disk per node:

```text
nodes = ceil(5,000 × 2 × 1.2 / 2,000) = ceil(6.0) = 6 data nodes
```

Add separately:

- 3 dedicated master-eligible nodes (§9).
- Optional coordinating-only nodes if query fan-out cost dominates (§9).

Always size for the **post-failure** state: if losing one node should not push the survivors past the 85% watermark, recompute with `(N-1)` denominator nodes. Six nodes at 80% pre-failure become five nodes at 96% post-failure — past the flood-stage line.

---

## 6. Vertical vs Horizontal Scaling

Use this table as a starting heuristic, not gospel:

| Bottleneck | Vertical (scale-up) | Horizontal (scale-out) |
|---|---|---|
| Heap GC pauses | No — capped at ~31 GB | Yes — split shards across more nodes |
| Page cache misses | Yes — more RAM per node | Yes — fewer hot shards per node |
| Indexing throughput | Yes (NVMe + CPU) up to a point | Yes — more primary shards parallelize writes |
| Query fan-out CPU | Yes (more cores) | Yes (more shards across nodes) |
| Disk capacity | Limited by node-class max disk | Yes — linear in node count |
| Network bandwidth | Limited per host | Yes — cross-node replication parallelizes |

The general principle: **scale up first until the heap cliff or per-host disk ceiling, then scale out.** A cluster of 8 well-sized nodes is easier to operate than 32 small ones, but a cluster pushed past 31 GB heap or past per-host disk limits has nowhere left to go vertically.

---

## 7. Disk and IOPS

Elastic and OpenSearch both recommend **SSD or NVMe** for any latency-sensitive workload. Spinning disks are viable only for cold tiers where queries are rare and acceptable latency is in the seconds.

Three usage patterns to size for:

- **IOPS** drives indexing throughput. Each refresh writes a new segment; merges read multiple segments and write one. A heavy bulk-index workload can saturate a 3,000-IOPS gp3 volume; 10,000+ IOPS (gp3 provisioned, io2, or local NVMe) is normal for indexing-heavy tiers.
- **Throughput** (MB/s) drives queries that scan large doc-value ranges and aggregations. NVMe local SSDs run at GB/s; cloud-attached SSDs are typically capped at 250–1,000 MB/s per volume.
- **Capacity** must hold: primary shard data + replica shard data + translog (default 512 MB per shard, capped) + segment merge headroom (a merge can temporarily double a shard's footprint).

Plan for translog + 2× the largest shard's worth of merge scratch space. Going past the 95% flood-stage watermark forces all indexes on the affected node into a read-only state — recovering from that requires manual intervention via the cluster settings API.

---

## 8. Network and Cross-Zone Latency

Replicas live on different nodes, and in a multi-AZ cluster they typically live in different zones. Every primary write traverses the zone boundary to its replicas; every search either fan-outs cross-zone or, with `preference=_local`/zone-aware routing, stays in zone at the cost of zone-affinity bias.

Practical numbers (cloud regions, intra-region inter-AZ):

- 1–2 ms RTT typical between AZs in the same AWS region.
- 50–500 µs RTT typical within a single AZ.
- A bulk indexing request to a primary plus two replicas pays two cross-zone hops in parallel before the client gets an ack with `wait_for_active_shards=all`.

The implication: cross-region clusters are an anti-pattern for latency-bound search. Use cross-cluster replication or cross-cluster search instead, with each cluster local to its AZs.

---

## 9. Master and Coordinating Tiers

### 9.1 Dedicated masters

Run **3 dedicated master-eligible nodes** (or 5 for very large clusters). They do not handle data, just cluster-state coordination — shard allocation decisions, mapping updates, node join/leave events. Modest spec is enough: 4–8 vCPU, 8–16 GB RAM, small disk. Three is the minimum for quorum (2 of 3 must be alive); 5 buys tolerance for two simultaneous failures.

The reason for dedicated masters is isolation: a data node under heavy GC must not delay cluster-state heartbeats. A delayed heartbeat is interpreted as a failed node and triggers re-election or shard reallocation — a stampede that compounds the original load problem.

### 9.2 Coordinating-only nodes

Coordinating nodes accept client requests, scatter them to the relevant data nodes, gather responses, merge results, and return. By default, every data node is also a coordinating node. Peeling off dedicated coordinating-only nodes makes sense when:

- Query fan-out aggregation (large `from`+`size` paginations, deep aggregations) consumes significant CPU on data nodes.
- A load balancer in front of the cluster cannot do client-aware routing and you want a stable set of "front door" hosts.
- You need to isolate heap pressure of large response merges away from the data tier.

For most clusters under ~10 data nodes, a load balancer in front of all data nodes is simpler and equivalent. Coordinating-only nodes are an optimization, not a default.

---

## 10. Hot/Warm/Cold Tier Sizing

Time-series workloads typically use Index Lifecycle Management (Elastic) or Index State Management (OpenSearch) to roll data through tiers:

| Tier | Storage | RAM:Disk ratio | Workload |
|---|---|---|---|
| Hot | Local NVMe | high (e.g. 1:30) | Active writes + recent searches |
| Warm | SSD (often cheaper) | lower (1:100) | Read-mostly, occasional searches |
| Cold | HDD or object-store mounted | very low (1:300+) | Rare searches, mostly archived |
| Frozen | S3 / object store, partial mount | minimal | Compliance, occasional access |

Hot tier sizing follows §1–§7. Warm and cold tiers can use much less RAM per disk because their indexes are not in hot read paths and page cache misses are tolerable. Force-merging shards as they roll from hot to warm (via ILM `forcemerge` action) consolidates segments, reduces FST count, and shrinks heap footprint per shard — pay the merge cost once at promotion rather than continuously.

---

## 11. Practical Traps

A non-exhaustive list of things that have eaten weekends:

1. **Heap above ~32 GB.** Loses compressed oops; GC pauses get *worse*. Cap at 31 GB and hand the remaining RAM to the page cache, or run two JVMs per host.
2. **Heap below ~50% of RAM with nothing using the rest.** Page cache is healthy but you are leaving query latency on the table — bumping heap up to the 50% line often shrinks p99 by 20–40% just by making aggregations stop spilling.
3. **Too many shards per node.** Watch `_cat/shards | wc -l` divided by node count; staying under ~20 per GB of heap is the published bound. Going over manifests as long cluster-state updates and slow recovery.
4. **One huge shard.** A 500 GB shard takes hours to recover and dominates rebalance time. Reindex with more primaries before it gets there.
5. **Disk past the 85% low watermark.** Allocation throttles, then stops; past 95% your indexes go read-only. Watermarks are a feature, not a bug — they prevent disk-full crashes.
6. **One AZ, one rack.** A correlated power or network failure takes the whole cluster. Spread across at least three AZs with `cluster.routing.allocation.awareness.attributes: zone`.
7. **Default refresh interval (1s) on heavy indexing.** Each refresh creates a segment; thousands of tiny segments per minute eat heap on FSTs and trigger merge storms. Bump to 30s or longer for ingest-heavy indexes that do not need second-level search visibility.
8. **No replicas in production.** A single node failure loses primary data with no failover. The default `number_of_replicas: 1` exists for a reason.
9. **Master-eligible data nodes under load.** Cluster-state delays under GC trigger spurious reallocation. Use dedicated masters at any non-trivial scale.
10. **Sizing for current data, not 12-month projection.** Reindexing to add shards is operationally expensive; overprovision primary count modestly at index creation.

---

## Related

- [Inverted Index Internals — Postings Lists, Skip Lists, and FSTs](../foundations/01-inverted-index-internals.md) — the §9 "filesystem cache matters more than heap" claim is what this doc operationalizes
- _Lucene Segments and Merge Policy (planned)_ — the merge-cost model that drives shard-size upper bounds
- [Operating Systems — Page Cache and Buffered I/O](../../operating-systems/operating-systems-tier1/03-page-cache-and-buffered-io.md) — the kernel side of why mmaped Lucene files want half the host's RAM
- [Performance — Latency, Throughput, Percentiles](../../performance/INDEX.md) — framework for reasoning about p99 regressions when the page cache cools
- [Kubernetes — Workloads and Stateful Services](../../kubernetes/INDEX.md) — sizing implications when running ES/OS as StatefulSets

## References

- [Elastic — Advanced configuration: Set the JVM heap size](https://www.elastic.co/guide/en/elasticsearch/reference/current/advanced-configuration.html#set-jvm-heap-size) — official 50%-of-RAM rule and the ≤~32 GB compressed-oops cap
- [Elastic — Size your shards](https://www.elastic.co/guide/en/elasticsearch/reference/current/size-your-shards.html) — 10–50 GB shard target, time-series up to 200 GB, "20 shards per GB of heap" rule of thumb
- [Elastic — Cluster-level shard allocation and routing settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html) — `cluster.max_shards_per_node`, allocation awareness, watermarks
- [Elastic — Disk-based shard allocation settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html#disk-based-shard-allocation) — 85% / 90% / 95% watermark defaults
- [OpenSearch — Sizing your OpenSearch cluster](https://opensearch.org/docs/latest/tuning-your-cluster/) — OpenSearch's parallel sizing guidance, including heap and shard rules
- [OpenSearch — Performance tuning](https://opensearch.org/docs/latest/tuning-your-cluster/performance/) — refresh interval, translog, merge policy guidance
- [OpenJDK HotSpot — Compressed Oops](https://wiki.openjdk.org/display/HotSpot/CompressedOops) — primary source for the ~32 GB compressed-oops boundary and object alignment scaling
- [Elastic — Index lifecycle management overview](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html) — ILM phases driving hot/warm/cold tier sizing in §10
- [Elastic — Designing for resilience](https://www.elastic.co/guide/en/elasticsearch/reference/current/high-availability-cluster-design.html) — dedicated master tier and quorum sizing referenced in §9
