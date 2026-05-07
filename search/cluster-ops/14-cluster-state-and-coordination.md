---
title: "Cluster State and Coordination — Master Election, Voting Config, and the Search Coordinator Phase"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, cluster-state, master, coordination]
---

# Cluster State and Coordination — Master Election, Voting Config, and the Search Coordinator Phase

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `cluster-state` `master` `coordination`

---

## Table of Contents

- [Summary](#summary)
- [1. Two Different "Coordinators" — Don't Confuse Them](#1-two-different-coordinators--dont-confuse-them)
- [2. The Master Node and the Cluster State](#2-the-master-node-and-the-cluster-state)
- [3. Discovery and Master Election — The Modern Coordination Layer](#3-discovery-and-master-election--the-modern-coordination-layer)
- [4. Voting Configuration, Quorum, and Bootstrap](#4-voting-configuration-quorum-and-bootstrap)
- [5. Split-Brain — What Pre-7.0 Got Wrong and What 7.0+ Fixed](#5-split-brain--what-pre-70-got-wrong-and-what-70-fixed)
- [6. Dedicated Master Tier — Sizing and Topology](#6-dedicated-master-tier--sizing-and-topology)
- [7. The Coordinator Phase of Search — Query, Reduce, Fetch](#7-the-coordinator-phase-of-search--query-reduce-fetch)
- [8. Search Fan-Out Cost and the Caches That Buffer It](#8-search-fan-out-cost-and-the-caches-that-buffer-it)
- [9. Practical Patterns and Diagnostics](#9-practical-patterns-and-diagnostics)
- [Related](#related)
- [References](#references)

---

## Summary

An Elasticsearch or OpenSearch cluster has two control-plane structures a backend engineer must keep distinct in their head. The first is the **master node** (OpenSearch calls it the **cluster manager**) — a single elected node that owns the cluster state: index mappings, the routing table that names which shards live on which data nodes, templates, ILM policies, persistent settings. The master is *not* on the read or write data path; it publishes state changes to all nodes and gets out of the way. The second is the **coordinating node** for a search — any node that receives a `_search` request, fans it out across shards, reduces partial results, fetches `_source`, and returns the response. Every data node coordinates by default; some clusters add dedicated coordinator-only nodes as a load-balancer tier.

The election layer that keeps the master alive is the cluster coordination subsystem introduced in Elasticsearch 7.0 (community-named "Zen2"; Elastic's own docs simply call it "cluster coordination"). It replaced the older Zen Discovery and the manually-tuned `discovery.zen.minimum_master_nodes` setting that caused most pre-7.0 split-brain incidents. The new layer auto-manages a **voting configuration** of master-eligible nodes, requires a strict majority quorum for any cluster-state commit, and bootstraps a new cluster via `cluster.initial_master_nodes` — a setting you must remove the moment the cluster forms or you risk accidentally creating a second cluster on top of the first.

This doc walks the master role, election, voting config, the coordinator phase of a search, fan-out cost, and the caches that buffer it. It closes with the diagnostic incantations (`_cluster/health`, `_cluster/state`, `_cat/nodes`) you reach for when a cluster is misbehaving.

---

## 1. Two Different "Coordinators" — Don't Confuse Them

The word "coordination" overloads in this stack. Both meanings show up in the same operational discussion and getting them mixed up will derail a postmortem.

| Concept | What it does | Who plays the role |
|---------|--------------|--------------------|
| Cluster coordination | Maintains cluster state, elects master, propagates metadata | Master-eligible nodes (Zen2 layer) |
| Search coordination | Receives a `_search`, fans out to shards, merges, returns | Any node that received the HTTP request |

The first is a control-plane concern (configs, mappings, who is master). The second is a data-plane concern (how a single query gets answered). The same node can do both, but they are separate machinery and separate failure modes. §2–§6 cover the first. §7–§9 cover the second.

---

## 2. The Master Node and the Cluster State

The master holds the **cluster state** — a single in-memory document, versioned monotonically, that contains:

- **Metadata** — every index's mappings, settings, aliases, templates, ILM policy bindings.
- **Routing table** — for every shard (primary and replica), which data node currently hosts it and whether it is `STARTED`, `INITIALIZING`, `RELOCATING`, or `UNASSIGNED`.
- **Discovered nodes** — node IDs, roles (`master`, `data`, `ingest`, `coordinator`), addresses.
- **Persistent and transient cluster settings**.
- **Snapshot state, repository registrations, security role mappings** (when those features are enabled).

When anything changes — a node joins, a shard relocates, a mapping is updated, an index is created — the master computes a new state version and **publishes** it to every node in the cluster via a two-phase commit: the master sends the new state to the voting configuration, waits for a quorum of master-eligible nodes to acknowledge it, then commits. Only after commit is the new state visible cluster-wide.

Crucially, the master is **not on the data path**. A `_search` does not pass through the master. A `_bulk` index request does not pass through the master. The master's job is to know where things live and tell everyone else; the data nodes do the actual work. This is why a busy master often looks idle in CPU graphs while the data tier is on fire.

A useful Spring-Boot-shaped analogy: the master is your service registry (Eureka, Consul) plus your config server (Spring Cloud Config). It tells every other service where to find each other and what config they should run with — but it doesn't sit in the request path of any business call.

---

## 3. Discovery and Master Election — The Modern Coordination Layer

In Elasticsearch 7.0, the cluster coordination subsystem replaced the older Zen Discovery. Elastic's announcement post calls it "a new cluster coordination subsystem" and does not officially brand it "Zen2," though the community and many third-party docs use that name. OpenSearch inherited the same machinery from its 1.x fork of Elasticsearch 7.10 and continues to evolve it; OpenSearch additionally renamed the role from "master" to "cluster manager" starting in 2.0, while keeping the old setting names as aliases.

### Discovery — finding peers

When a node starts, it needs to locate the other nodes. The seed list is the entry point:

```yaml
# elasticsearch.yml
discovery.seed_hosts:
  - es-master-1.internal:9300
  - es-master-2.internal:9300
  - es-master-3.internal:9300
```

A node connects to the seed hosts, asks them for the current cluster state, and uses the returned node list to discover the rest. Seed hosts can be DNS names, static IPs, or — on cloud platforms — the output of a discovery plugin that queries the cloud's instance metadata.

### Election

Any master-eligible node can call an election. The first node whose proposal is acknowledged by a quorum of the voting configuration wins. To reduce the odds of two nodes calling elections at the same instant, the coordination layer schedules election attempts at randomized offsets with exponential backoff after failed rounds. In a healthy 3-master cluster, the elected master typically wins on the first round; in a degraded cluster (network partitions, slow nodes), elections can take seconds and the cluster is effectively read-only for state changes during that window.

---

## 4. Voting Configuration, Quorum, and Bootstrap

The **voting configuration** is the set of master-eligible nodes whose votes count for cluster-state decisions. It is *not* identical to "all master-eligible nodes" — the coordination layer auto-manages it, adding nodes when they join and removing nodes that have been gone long enough. A decision (master election, cluster-state commit) requires a strict majority of the voting configuration to acknowledge.

The cardinal rule from Elastic's quorum docs: **never stop half or more voting nodes simultaneously**. With three masters in the voting config, you can lose one and stay healthy; lose two and the cluster freezes until one returns. With five, you can lose two. Even numbers buy you nothing — a 4-node voting config still tolerates only one loss because a strict majority of 4 is 3.

### Bootstrap vs. join

When a brand-new cluster forms, nobody is master yet, so there is no existing voting config to vote with. Elasticsearch resolves this with a one-time bootstrap step: every master-eligible node lists the same set of nodes in `cluster.initial_master_nodes`. Once those nodes can see each other and agree, they form the initial voting config and elect a master.

```yaml
# Only on the master-eligible nodes, only on first cluster formation
cluster.initial_master_nodes:
  - es-master-1
  - es-master-2
  - es-master-3
```

Two warnings that the official reference is emphatic about:

1. **Use the same list on every node.** Different lists can bootstrap two separate clusters that then pretend to be one.
2. **Remove the setting after the cluster forms.** Leaving it in place means a future restart can re-bootstrap a *new* cluster on top of the existing data directory — a documented path to unrecoverable data loss.

Nodes joining an *existing* cluster do not set `cluster.initial_master_nodes`. They configure `discovery.seed_hosts`, contact the cluster, and the elected master adds them to the voting config if they are master-eligible.

---

## 5. Split-Brain — What Pre-7.0 Got Wrong and What 7.0+ Fixed

Pre-7.0 Zen Discovery required the operator to set `discovery.zen.minimum_master_nodes` to `(N/2) + 1` where `N` is the count of master-eligible nodes. Two problems followed:

- **It was per-node manual configuration.** When you scaled the master tier from 3 to 5, you had to update the setting on every node in lockstep. People forgot.
- **The system could not detect a wrong setting.** A cluster with `minimum_master_nodes = 1` and two master-eligibles would happily run; on a network partition, both sides could elect themselves master, accept writes, and silently diverge — classic split-brain.

The 7.0 coordination subsystem removed the setting entirely. Quorum is now derived automatically from the voting configuration, which the cluster owns and updates. There is no number for an operator to get wrong. Elastic's announcement post and the discovery reference both spell this out as the central design improvement of the rewrite.

The lesson generalises beyond Elasticsearch: **manual quorum configuration is an antipattern.** Etcd, ZooKeeper ensembles, and Raft-based systems all converged on the same answer — derive quorum from the membership, don't ask humans to type it.

---

## 6. Dedicated Master Tier — Sizing and Topology

Roles are configurable per node. The four common ones:

```yaml
# Dedicated master
node.roles: [ master ]

# Dedicated data
node.roles: [ data, data_content, data_hot ]

# Dedicated coordinator (no data, no master)
node.roles: [ ]

# Combined (small cluster)
node.roles: [ master, data, ingest ]
```

For anything beyond a development cluster, the recommended topology is **three dedicated master-eligible nodes**. Three is the smallest count that tolerates a single failure and a single rolling restart. Five buys an extra failure tolerance and is appropriate for very large clusters (hundreds of data nodes), where master workload — cluster-state publication, mapping updates, shard allocation decisions — grows with cluster size.

Why dedicated, rather than combined with data?

- **Stability.** A data node under search load can GC-pause for hundreds of milliseconds. If that node is also master, the cluster perceives the master as unresponsive and may call an election. Dedicated masters with smaller heaps and quieter workloads pause less.
- **Predictable resource ceilings.** Cluster-state publication for a 1,000-index cluster involves serialising and shipping a large state document on every change. You want headroom on the master and you want it untouched by query traffic.
- **Clear blast radius.** A bad query OOMs a data node, not the master tier.

Master nodes are typically modest: 4–8 GB heap, 2–4 vCPU, modest disk. Their job is bookkeeping, not heavy lifting.

---

## 7. The Coordinator Phase of Search — Query, Reduce, Fetch

When a `_search` lands on any node, that node becomes the **coordinating node** for that request. The classical execution model is **query then fetch** with three logical phases:

```text
                ┌──── coordinating node ────┐
HTTP /_search ─►│  parse, route, fan out    │
                └────┬─────┬─────┬──────────┘
                     │     │     │
              ┌──────▼┐ ┌──▼──┐ ┌▼─────┐
              │shard 0│ │shard│ │shard │   query phase
              │  P/R  │ │ 1   │ │ N    │   (per shard)
              └──────┬┘ └──┬──┘ └┬─────┘
                     │     │     │
                ┌────▼─────▼─────▼──────────┐
                │   coordinator: reduce      │   merge top-K
                │   global top-K by score    │   from each shard
                └────┬───────────────────────┘
                     │
                ┌────▼───────────────────────┐
                │   fetch phase (round 2)    │   _source for the
                │   ask the right shards     │   final K hits
                │   for the final docs       │
                └────┬───────────────────────┘
                     │
                HTTP response
```

### Query phase (per shard)

The coordinator routes a copy of the query to one copy (primary or replica) of every shard the index spans. Each shard runs the query against its local Lucene index and returns a small payload: the top-K doc IDs and their scores, plus aggregation partial state if the query has aggs. The shard does not send `_source` yet — only IDs and scores.

### Reduce on the coordinator

The coordinator merges the per-shard top-K lists into a global top-K by score, picks the actual `from + size` document IDs, and combines aggregation partials into final aggregation results. This step is pure CPU on the coordinator and scales with `shard_count × size`.

### Fetch phase

For the documents that actually made the global top-K, the coordinator goes back to the specific shards that own them and asks for `_source`, stored fields, and highlights. This is the second network round. It only touches the shards that have winning docs — typically a small fraction of the total fan-out.

For aggregation-only queries (`size: 0`), the fetch phase is skipped entirely.

For DFS variants (`search_type=dfs_query_then_fetch`), there is an extra preliminary round to gather global term statistics before scoring; it improves relevance accuracy across shards at the cost of an extra round-trip and is rarely worth it in practice.

### Coordinating nodes — combined or dedicated

Any data node can coordinate. In small clusters, every data node is also a coordinator and the HTTP load balancer (an ALB, an Nginx, an nginx-ingress in Kubernetes) round-robins client requests across them. In larger clusters, you can run **coordinator-only nodes** (`node.roles: [ ]`) — no data, no master role, just CPU and network for fan-out and reduce. These act as a query-fronting tier and absorb the per-request memory pressure of merging large top-K lists, keeping data-node heaps freer for postings and doc values.

---

## 8. Search Fan-Out Cost and the Caches That Buffer It

Every shard the index spans gets touched by every query that does not specify a routing key. This is the dominant scaling cost of a search cluster, and it is where most over-sharded clusters quietly fall over.

**The arithmetic:** an index with 30 primary shards, queried at 200 QPS, generates `30 × 200 = 6,000` per-shard query operations per second, plus a fetch round for the matching subset. Double the shard count and you double the fan-out load on every single query — even ones that match nothing.

The mitigation knobs:

- **Fewer, larger shards.** A few tens of GB per shard is the conventional target. Don't shard for the sake of sharding.
- **Routing.** If queries always carry a tenant ID, route by tenant so each query hits one shard, not all of them.
- **Caches** — three layers, each caching a different thing:

| Cache | What it caches | Scope | When it's invalidated |
|-------|----------------|-------|----------------------|
| Shard request cache | The full coordinator-visible result (hits + aggs) for an entire shard-level request, but only when `size: 0` | Per shard, per query body | On any refresh that changes the shard |
| Query cache (filter cache) | Per-segment bitsets for `filter` clauses (e.g., `term`, `range` on dates) | Per shard, per segment, per filter | When the segment is merged away |
| Fielddata cache | Per-field uninverted in-memory representation of `text` field values for sort/agg | Per shard, per field | On segment merge or eviction |

A few rules of thumb that follow from those scopes:

- The shard request cache helps dashboards (lots of `size: 0` aggregation queries with stable bodies). It does nothing for free-text search returning hits.
- The query cache helps filters that repeat across queries (`status: active`, `created_at: [now-7d TO now]`). It does nothing for ad-hoc full-text queries because the cache key is the filter itself.
- The fielddata cache is a footgun on `text` fields. Sort or aggregate on `text` and the engine will uninvert the entire field into heap. Use `keyword` fields and doc values for sorting/aggregations.

---

## 9. Practical Patterns and Diagnostics

### Topology cheat sheet

- **3 dedicated master-eligible nodes** for any cluster you care about. Five for very large clusters.
- **N data nodes**, each sized for hot-shard working set in OS page cache (see [01-inverted-index-internals.md](../foundations/01-inverted-index-internals.md) §9).
- **0 to M coordinator-only nodes** as a query-fronting tier; rule of thumb is one coordinator per ~5–10 data nodes when fan-out reduce starts hurting data-node heaps. Skip them entirely for small clusters.
- **Ingest nodes** (`node.roles: [ ingest ]`) are a separate concern; treat them like coordinators but for the write path.

### Diagnostics — what to run when the cluster is unhappy

```bash
# Cluster status — green/yellow/red, unassigned shard count
curl -s localhost:9200/_cluster/health?pretty

# Per-index health — find which index is yellow/red
curl -s 'localhost:9200/_cluster/health?level=indices&pretty'

# Why is a specific shard unassigned?
curl -s localhost:9200/_cluster/allocation/explain?pretty

# Who is master right now?
curl -s 'localhost:9200/_cat/master?v'

# Node roles, heap, load
curl -s 'localhost:9200/_cat/nodes?v&h=name,node.role,heap.percent,load_1m'

# Full cluster state — large; filter by metric
curl -s 'localhost:9200/_cluster/state/master_node,nodes,routing_table?pretty'

# Pending cluster-state tasks — backed-up master is a smell
curl -s 'localhost:9200/_cluster/pending_tasks?pretty'
```

A few read-the-tea-leaves heuristics:

- `_cluster/health` reports `status: red` with a non-zero `unassigned_shards`: a primary shard is missing. Run `_cluster/allocation/explain` for the why.
- `pending_tasks` is consistently non-empty: master is overloaded or underprovisioned. Cluster-state mutations are queueing.
- The elected master changes more than once a day under normal load: investigate master-tier GC pauses or network flakiness between master nodes.
- Search latency p99 spikes correlate with `refresh_interval` boundaries: the refresh is rebuilding readers and trashing caches. Lengthen the refresh interval if the workload tolerates it.

---

## Related

- [01 — Inverted Index Internals](../foundations/01-inverted-index-internals.md) — the per-shard data structure that the query phase walks
- _Sharding and Replicas — Allocation, Awareness, and Recovery (planned)_ — how the master decides where shards live
- _Snapshots, Restore, and Repository State (planned)_ — another cluster-state-bound subsystem
- [Database — Replication and Consensus](../../database/INDEX.md) — Raft/Paxos in relational replication; the coordination layer here is conceptually adjacent
- [Networking — TCP and Connection Lifecycle](../../networking/INDEX.md) — fan-out cost is fundamentally a network amplification story
- [Observability — RED/USE Method](../../observability/INDEX.md) — the diagnostic framing that `_cluster/health` and `_cat/nodes` slot into

## References

- [Elasticsearch Reference — Discovery and cluster formation](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery.html) — authoritative overview of the coordination subsystem, seed hosts, voting config
- [Elasticsearch Reference — Quorum-based decision making](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-quorums.html) — voting configuration semantics, never-stop-half-the-voters rule, election retries
- [Elasticsearch Reference — Bootstrapping a cluster](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-bootstrap-cluster.html) — `cluster.initial_master_nodes`, the bootstrap-vs-join distinction, the warning about leaving the setting in place
- [Elastic Blog — A New Era for Cluster Coordination in Elasticsearch](https://www.elastic.co/blog/a-new-era-for-cluster-coordination-in-elasticsearch) — the 7.0 coordination rewrite, why `discovery.zen.minimum_master_nodes` was removed
- [Elasticsearch Reference — Designing for resilience](https://www.elastic.co/guide/en/elasticsearch/reference/current/high-availability-cluster-design.html) — official guidance on master tier sizing and zone-aware topologies
- [Elasticsearch Reference — Node roles](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html) — `master`, `data`, `ingest`, coordinator-only role definitions
- [OpenSearch Documentation — Cluster formation](https://docs.opensearch.org/latest/tuning-your-cluster/cluster/) — OpenSearch's discovery/voting docs and the "cluster manager" terminology rename from 2.0
- [Elasticsearch Reference — Search your data](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-your-data.html) — entry point for the distributed-search docs covering query/fetch phases
- [Elasticsearch Reference — Shard request cache, query cache, fielddata cache](https://www.elastic.co/guide/en/elasticsearch/reference/current/shard-request-cache.html) — the three caches in §8
