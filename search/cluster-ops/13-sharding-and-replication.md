---
title: "Sharding and Replication — How an Index Becomes a Cluster"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, sharding, replication, cluster]
---

# Sharding and Replication — How an Index Becomes a Cluster

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `sharding` `replication` `cluster`

---

## Table of Contents

- [Summary](#summary)
- [1. Why Shards Exist](#1-why-shards-exist)
- [2. Primaries vs Replicas — What Each Is For](#2-primaries-vs-replicas--what-each-is-for)
- [3. Shard Sizing — The Numbers That Actually Matter](#3-shard-sizing--the-numbers-that-actually-matter)
- [4. Routing — How a Document Picks Its Shard](#4-routing--how-a-document-picks-its-shard)
- [5. Allocation — Placing Shards on Nodes](#5-allocation--placing-shards-on-nodes)
- [6. Hot Spots and How to Avoid Them](#6-hot-spots-and-how-to-avoid-them)
- [7. The Write Path — Primary First, Replicas in Parallel](#7-the-write-path--primary-first-replicas-in-parallel)
- [8. The Read Path — Adaptive Replica Selection](#8-the-read-path--adaptive-replica-selection)
- [9. Document-Based vs Segment-Based Replication](#9-document-based-vs-segment-based-replication)
- [10. Index Sorting](#10-index-sorting)
- [11. Practical Patterns](#11-practical-patterns)
- [Related](#related)
- [References](#references)

---

## Summary

A Lucene index sits on one node. An *Elasticsearch* (or OpenSearch) index is a collection of Lucene indexes — **shards** — distributed across nodes. Sharding is how you scale the index past a single machine's disk and CPU; replication is how you survive a node loss and absorb read load. The two concepts solve different problems and you size them with different rules.

The mental model that pays off: a primary shard is *the only place writes for a given document land*; a replica shard is a full copy of a primary, used for high availability and read scale. Adding replicas does **not** scale write throughput — it multiplies it by `(1 + replicas)` because every write still has to land on every copy. Adding primary shards does scale writes, but you cannot change the primary count of an existing index without a reindex (the routing math depends on it). This doc walks the distributed half of search — sizing, routing, allocation, hot spots, the write/read paths, and the segment-based replication evolution in modern Elasticsearch — for a backend engineer who already understands the per-shard inverted index from `01-inverted-index-internals.md`.

---

## 1. Why Shards Exist

A single Lucene index is bounded by one node's resources: disk for segments, RAM for FSTs and the OS page cache, CPU for query execution. Once your corpus exceeds what one node can comfortably hold or once query throughput exceeds what one node can serve, you need to split the index. Sharding is that split.

Two distinct scaling axes follow:

- **Storage and indexing throughput** — partition writes across N primaries so each Lucene index stays a manageable size, and indexing CPU is spread across nodes.
- **Query parallelism** — a query against a sharded index is fanned out to all shards in parallel; the coordinating node merges the per-shard results. More shards means more concurrent CPUs working on the query, *up to a point* (see §3).

The first axis is why you shard at all. The second is why "more shards is always better" is wrong: the coordinator-side merge cost and the per-shard overhead grow linearly with shard count.

---

## 2. Primaries vs Replicas — What Each Is For

Every document belongs to exactly one **primary shard**. A primary can have zero or more **replica shards** — full copies, kept on different nodes from the primary.

| Aspect | Primary | Replica |
|--------|---------|---------|
| Receives writes | Yes (first) | Yes (copied from primary) |
| Serves reads | Yes | Yes |
| Loss tolerated | Cluster red until replica promoted | Cluster yellow; rebuilt from primary |
| Scales write throughput | Yes (more primaries) | No (more replicas multiplies write cost) |
| Scales read throughput | n/a | Yes |

The replica-doesn't-scale-writes point catches people: setting `number_of_replicas: 2` on a busy index means every indexing request now does roughly three times the disk and network work. Replicas are for *availability* and *read fan-out*, not write capacity.

You can change `number_of_replicas` on a live index. You cannot change `number_of_shards` without reindexing into a new index — see §4 for why.

---

## 3. Shard Sizing — The Numbers That Actually Matter

Elastic's current guidance (verified against the official "Size your shards" page in the 8.x reference) is shaped by two opposing pressures: shards too small mean wasted overhead per shard; shards too large mean slow recovery and merge stalls. The published rules of thumb:

- **Aim for shard sizes between 10 GB and 50 GB** for typical search and time-series workloads. The Elastic docs list this as the recommended target range.
- **Avoid shards larger than ~200 GB.** The official guidance flags shards above this as harder to recover and operate, though it explicitly notes there is no hard cap and the right number depends on workload.
- **Aim for fewer than 20 shards per GB of JVM heap** on a data node. Each open shard consumes file handles, FST RAM, and small bookkeeping costs; the cluster-state size also grows with shard count.
- **Cap shard count growth.** Elasticsearch 7.0+ enforces a soft limit of 1,000 shards per data node by default (`cluster.max_shards_per_node`).

OpenSearch's "Sizing your domain" docs publish the same general framing — the 10-50 GB target band is consistent across both projects' current documentation, with OpenSearch additionally suggesting a starting point of `(source data size × replication factor × 1.45) / target shard size` to compute initial primary count.

These are *guidance*, not laws. Logging clusters routinely run 50 GB shards because the workload is write-heavy and rarely re-queries old data; small high-QPS search indexes often run 5 GB shards to maximize parallelism. The point is you size against the band, not pick numbers out of the air.

### 3.1 Why the primary count is fixed

The default routing function (next section) is `shard_num = hash(_id) % number_of_primaries`. If you change `number_of_primaries`, every existing document's shard assignment changes, so every document has to be moved. There is no incremental rebalance — you reindex.

Workarounds for "I sized wrong":

- **Reindex API** into a new index with the right shard count, then switch an alias.
- **Split index API** — only works if the new shard count is a multiple of the old, and only if the index was created with `index.number_of_routing_shards` set high enough.
- **Shrink index API** — reduce shard count to a divisor of the original.

Picking the right primary count up front matters because the cheap operations (split, shrink) are constrained, and the unconstrained operation (reindex) is a full data copy.

---

## 4. Routing — How a Document Picks Its Shard

When you index a document without a `routing` parameter, Elasticsearch computes:

```text
shard_num = hash(_id) % number_of_primary_shards
```

The hash is Murmur3 in current Lucene/Elasticsearch. The doc lands on `shard_num`'s primary, then is replicated.

You can override `_id` with a custom `routing` key in the index request:

```json
POST /orders/_doc?routing=tenant-42
{
  "tenant_id": "tenant-42",
  "amount": 19.99
}
```

Now `shard_num = hash("tenant-42") % primaries`. Every document for `tenant-42` lands on the same shard. Searches that filter on the tenant can include `?routing=tenant-42` to skip the fan-out and hit just that one shard — a real latency win when you have thousands of tenants and the per-tenant data is small.

The cost: routing keys with skewed distribution create hot shards. If 10% of your tenants generate 90% of the writes, two or three shards do almost all the work and the rest sit idle.

A related knob: `index.number_of_routing_shards`. This is the *virtual* shard count used for the modulo before the result is mapped down to the real `number_of_shards`. It exists so the Split API can later increase shard count by integer factors that are divisors of `number_of_routing_shards`. Set it high (the default in modern versions is generous) at index creation if you anticipate growth.

---

## 5. Allocation — Placing Shards on Nodes

Shard *routing* picks which shard a document goes to. Shard *allocation* picks which node a shard lives on. Allocation is the master node's job and is governed by allocation deciders. The ones that matter most in production:

### 5.1 Allocation awareness (zones)

Tell the cluster about its physical topology so it spreads replicas across failure domains:

```yaml
# elasticsearch.yml on each node
node.attr.zone: us-east-1a
cluster.routing.allocation.awareness.attributes: zone
```

With awareness enabled, the master tries to place the primary and its replicas in different zones. A single-AZ outage no longer takes a shard offline.

### 5.2 Forced awareness

Awareness alone is best-effort: if only one zone is up, the cluster will still allocate replicas to that zone. *Forced* awareness refuses to allocate replicas if their target zone is missing:

```yaml
cluster.routing.allocation.awareness.attributes: zone
cluster.routing.allocation.awareness.force.zone.values: us-east-1a,us-east-1b,us-east-1c
```

Now if `us-east-1c` is down, the replicas that would land there stay unassigned rather than overload the surviving zones. Reads go yellow, writes still succeed against the primaries, and cluster capacity stays predictable.

### 5.3 Allocation filtering

Pin shards to specific node attributes — useful for hot/warm tiers:

```json
PUT /old-logs/_settings
{ "index.routing.allocation.require.box_type": "warm" }
```

Combined with node attributes (`node.attr.box_type: hot|warm|cold`), this is the foundation of the data-tier feature in modern Elasticsearch.

---

## 6. Hot Spots and How to Avoid Them

A hot shard is a shard receiving disproportionate load. Three common causes:

- **Skewed routing keys.** Custom routing on a high-cardinality-but-imbalanced field (one giant tenant) concentrates writes on one shard. Mitigation: switch the giant tenant to a dedicated index, or fall back to default routing for them.
- **Time-based indices with all writes on "today".** A daily index pattern means every write hits today's index, and within that index, the modulo distribution means every shard takes its 1/N. That part is fine — the trap is when one node holds today's primary for several hot indices and gets pinned. The data-tier "hot" role plus enough hot nodes is the answer.
- **Hot key per shard.** Even with balanced routing, one document repeatedly updated (a counter, a session record) generates serialized writes and merge churn on whichever shard owns it. This is a modeling issue more than a sharding one — Elasticsearch is not a counter store.

The diagnostic is `GET _cat/shards?v` plus `GET _nodes/stats/indices` — look at per-shard `indexing.index_total` deltas over a window; if one shard is doing 10× the rest, you have a hot spot.

---

## 7. The Write Path — Primary First, Replicas in Parallel

An indexing request lands on any node (the *coordinator* for that request). The coordinator computes the shard via routing and forwards the request to the node holding that shard's primary. The primary then:

1. Validates and runs the document through ingest pipelines.
2. Writes to its in-memory Lucene buffer and to the translog.
3. **In parallel**, forwards the operation to every in-sync replica.
4. Waits for replica acknowledgements (or failure responses).
5. Returns to the coordinator, which returns to the client.

The wait-for-replicas step is governed by `wait_for_active_shards`. Default is `1` (only the primary needs to ack). Setting it to `all` waits for every replica — stronger durability, higher latency. Most production clusters leave it at the default and rely on the in-sync set plus translog for durability.

The translog is per-shard and is fsync'd by default on every request (`index.translog.durability: request`). This is the durability boundary: a successful write response means the translog has been fsync'd on the primary and on each in-sync replica. Refresh (which makes a doc searchable) and flush (which clears the translog by writing a Lucene commit) are separate, asynchronous events.

---

## 8. The Read Path — Adaptive Replica Selection

A search request lands on a coordinator. The coordinator picks one copy of each shard — primary or any replica, doesn't matter for read consistency since both are byte-identical at the segment level — and fans the query out. Each shard runs the query locally against its segments, returns its top-K, and the coordinator merges into the global top-K.

Picking *which* copy to query is the job of **adaptive replica selection** (ARS), introduced in Elasticsearch 6.1 and on by default. ARS scores each copy on response time, queue length, and service time, and prefers the copy with the lowest predicted latency. If a node is overloaded or in GC, its replicas drop out of the rotation until they recover. The official "search" reference describes the heuristic; the Elastic blog post "Improving response latency in Elasticsearch with adaptive replica selection" walks through the formula.

For most workloads ARS is a quiet win — you do not need to think about it. The case where it matters: heterogeneous hardware, where a slow node is dragging tail latency. ARS will route around it without operator intervention.

---

## 9. Document-Based vs Segment-Based Replication

Historically (Elasticsearch ≤ 7.x and OpenSearch ≤ 1.x), replication was **document-based**: the primary forwarded each indexing operation to its replicas, and each replica re-indexed the document into its own segments independently. Same input, same output, but the Lucene work happens N times.

OpenSearch 2.x introduced **segment replication** as an alternative: the primary does the Lucene work once, then ships the resulting segment files to replicas. Replicas pull segments from the primary on a schedule and refresh against those segments rather than building their own. The OpenSearch documentation reports significant CPU reduction on replicas and lower indexing throughput cost when replicas are added — at the cost of slightly higher refresh latency on replicas (they now lag the primary by the segment-shipping interval) and tighter coupling between primary and replica during recovery.

Elastic took a different path. Elasticsearch's stateless / serverless architecture (8.x cloud serverless tier) separates compute from storage entirely, with indexing and search nodes reading from a shared object store. Verify the current state of segment replication in the open-source Elasticsearch distribution before assuming parity with OpenSearch — at the time of writing, the open-source Elasticsearch project does not expose segment replication as a standard index setting the way OpenSearch does, and the comparison is not apples-to-apples.

The takeaway for an application engineer: if you operate OpenSearch and your replica CPU is your bottleneck, segment replication is worth evaluating. If you operate Elasticsearch, this is not a knob you turn.

---

## 10. Index Sorting

`index.sort.field` lets you tell Lucene to physically sort documents within each segment by a chosen field at index time:

```json
PUT /events
{
  "settings": {
    "index.sort.field": ["timestamp"],
    "index.sort.order": ["desc"]
  }
}
```

Two payoffs:

- **Early termination on sorted queries.** A query asking for "top 100 by timestamp desc" can stop scanning a segment once it has 100 hits, because the segment is already in that order. Elastic's "index sorting" docs describe and benchmark this.
- **Better compression.** Sorted documents tend to share more structure with their neighbors, so postings deltas are smaller and doc-values runs compress better.

The cost: indexing is slower because Lucene has to maintain the sort order during merges. Use it when you have a dominant query sort (timestamp on logs, score on something) and indexing throughput is not the bottleneck.

---

## 11. Practical Patterns

### 11.1 Time-based indices

For append-mostly logs and metrics: one index per day/week, a write alias pointing at "current", and ILM (Elastic) or ISM (OpenSearch) policies to roll over by size or age. Old indices become read-only, are force-merged down to one segment, and migrate to warm/cold tiers. Shard size targets the upper end of the recommended band (50 GB) because the data is rarely re-queried.

### 11.2 Index per tenant vs single index with routing

| | Index per tenant | Single index, routing key |
|---|---|---|
| Tenants | Tens to low hundreds | Thousands+ |
| Per-tenant search | Trivial | Use `?routing=` |
| Per-tenant deletion | Drop the index | `delete_by_query` (slow) |
| Mapping divergence | Allowed | Not allowed |
| Cluster-state cost | Grows with tenant count | Bounded |
| Shard balance | Skews if tenants vary in size | Single-shard hot spots |

The classic split: small number of large tenants → index per tenant; large number of small tenants → single index with routing. Mixed workloads sometimes use a hybrid (top tenants get their own index, the long tail shares a routed index).

### 11.3 Target shard size by data type

| Workload | Target shard size | Rationale |
|---|---|---|
| Search-heavy (catalog, knowledge base) | 10–25 GB | Smaller shards = more query parallelism |
| Time-series logs/metrics | 30–50 GB | Write-heavy, rarely re-queried |
| Mixed read/write | 20–40 GB | Middle ground |
| Vector / kNN-heavy | Smaller (workload-dependent) | HNSW graph fits in RAM per shard |

These are starting points anchored against the 10-50 GB band. Measure on your own workload; if your queries are dominated by aggregations on a single big shard, more parallelism (smaller shards) helps; if your queries are dominated by coordinator merge cost, fewer-larger shards help.

---

## Related

- [Search — Inverted Index Internals](../foundations/01-inverted-index-internals.md) — what each individual shard actually is on disk
- _Lucene Segments and Merge Policy (planned)_ — Tier 1 deep dive on the per-shard merge subsystem this doc only references
- _Refresh, Flush, and the Translog (planned)_ — durability and visibility timing referenced in §7
- [Database — Sharding and Replication Patterns](../../database/INDEX.md) — adjacent partitioning and replication problem in the relational world
- [System Design — Consistency and Availability](../../system-design/INDEX.md) — primary/replica tradeoffs across distributed systems
- [Operating Systems — Page Cache and Buffered I/O](../../operating-systems/operating-systems-tier1/03-page-cache-and-buffered-io.md) — why per-shard size interacts with node memory

## References

- [Elasticsearch Reference — Size your shards](https://www.elastic.co/guide/en/elasticsearch/reference/current/size-your-shards.html) — official source for the 10-50 GB target, ~200 GB cap guidance, and shards-per-heap-GB rule
- [Elasticsearch Reference — Cluster-level shard allocation and routing settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html) — allocation deciders, awareness, forced awareness
- [Elasticsearch Reference — Shard allocation awareness](https://www.elastic.co/guide/en/elasticsearch/reference/current/shard-allocation-awareness.html) — zones and forced awareness syntax used in §5
- [Elasticsearch Reference — Index sorting](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-index-sorting.html) — `index.sort.field` semantics and early-termination behavior
- [Elasticsearch Reference — Search your data / Adaptive replica selection](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-your-data.html) — ARS default-on behavior referenced in §8
- [Elastic blog — Improving response latency in Elasticsearch with adaptive replica selection](https://www.elastic.co/blog/improving-response-latency-in-elasticsearch-with-adaptive-replica-selection) — the heuristic behind ARS
- [OpenSearch Documentation — Sizing your domain](https://opensearch.org/docs/latest/tuning-your-cluster/size-your-cluster/) — OpenSearch's shard sizing and primary-count formula referenced in §3
- [OpenSearch Documentation — Segment replication](https://opensearch.org/docs/latest/tuning-your-cluster/availability-and-recovery/segment-replication/) — segment-based replication feature referenced in §9
- [Elastic blog — Found: Elasticsearch in Production](https://www.elastic.co/blog/found-elasticsearch-in-production) — operational guidance on cluster topology and the primary/replica model
