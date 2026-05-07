---
title: "Cross-Cluster Search and Cross-Cluster Replication — Federation, DR, and Geo-Distribution"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, ccs, ccr, replication, cluster-ops, disaster-recovery]
---

# Cross-Cluster Search and Cross-Cluster Replication — Federation, DR, and Geo-Distribution

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `ccs` `ccr` `replication` `cluster-ops` `disaster-recovery`

---

## Table of Contents

- [Summary](#summary)
- [1. Why Two Clusters in the First Place](#1-why-two-clusters-in-the-first-place)
- [2. Cross-Cluster Search — Connection Model](#2-cross-cluster-search--connection-model)
- [3. CCS Query Syntax and Resilience Flags](#3-ccs-query-syntax-and-resilience-flags)
- [4. `ccs_minimize_roundtrips` — What the Flag Actually Does](#4-ccs_minimize_roundtrips--what-the-flag-actually-does)
- [5. Cross-Cluster Replication — Elastic CCR](#5-cross-cluster-replication--elastic-ccr)
- [6. OpenSearch Cross-Cluster Replication](#6-opensearch-cross-cluster-replication)
- [7. Conflict Handling, Lag, and `_ccr/stats`](#7-conflict-handling-lag-and-_ccrstats)
- [8. Use Cases — DR, Geo-Distribution, Hot/Cold Split](#8-use-cases--dr-geo-distribution-hotcold-split)
- [9. Practical Traps](#9-practical-traps)
- [Related](#related)
- [References](#references)

---

## Summary

A single Elasticsearch or OpenSearch cluster works fine until it doesn't. The forcing functions are usually one of three: **data sovereignty** (EU data must stay in EU, US data in US), **disaster recovery** (a regional outage cannot take search down), or **scale-out separation** (hot tier in one cluster, archival/cold tier in another so a runaway aggregation against six years of logs cannot starve the live ingestion path). The two features that span clusters answer different halves of the problem: **Cross-Cluster Search (CCS)** lets a single query fan out to indices that live on remote clusters, and **Cross-Cluster Replication (CCR)** asynchronously copies indices from a leader cluster to a read-only follower on another cluster.

CCS is built into both Elasticsearch and OpenSearch and free at every license tier. CCR in Elastic is a **paid Enterprise-tier feature** in self-managed and cloud subscriptions — a fact you must confirm against your contract before designing a DR plan around it. OpenSearch ships its own Apache-2.0 cross-cluster replication plugin with similar leader/follower semantics. Both implementations are asynchronous, both use a pull-based follower model, and both require operations to be retained on the leader long enough for the follower to catch up — the soft-deletes retention period in Elastic, an analogous translog/operation history in OpenSearch.

---

## 1. Why Two Clusters in the First Place

Within a single cluster, Lucene already handles replicas, shard failover, and node-loss recovery. That covers single-node and rack-level failures, not regional outages, not data residency, and not blast-radius isolation between workloads. The questions CCS and CCR answer:

- *"Search legal-hold archives in `eu-west-1` and live tickets in `us-east-1` from one query."* → CCS.
- *"Lose `us-east-1` entirely and keep serving search from `us-west-2` within minutes."* → CCR.
- *"Stop the analytics team's six-month aggregations from blowing up the cluster that takes live writes."* → CCR or cluster-per-tier with CCS on top.
- *"Ship every European tenant's data to a separate cluster, but let admins still search across tenants."* → CCS.

The two features compose. A common production pattern is CCR replicating leader → follower for DR, with CCS configured on a third "search-aggregator" cluster that fans out to both regions for global queries.

---

## 2. Cross-Cluster Search — Connection Model

A cluster that wants to search remote clusters registers them in its own cluster settings under the `cluster.remote.<alias>` namespace. The alias becomes the prefix used in queries.

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "logs_eu": {
          "mode": "sniff",
          "seeds": ["es-eu-1.example.com:9300", "es-eu-2.example.com:9300"],
          "skip_unavailable": true
        },
        "logs_us": {
          "mode": "proxy",
          "proxy_address": "es-us-proxy.example.com:9400",
          "skip_unavailable": true
        }
      }
    }
  }
}
```

Two connection modes exist:

- **Sniff mode** (default) — the local coordinating node connects to the listed seed nodes, fetches the remote cluster's node list, and opens connections directly to a subset of remote *gateway nodes*. Requires direct TCP reachability to every gateway node, which makes it awkward across VPCs or through L7 load balancers.
- **Proxy mode** — the local coordinator opens connections only to a single configured proxy address. The remote side terminates the connection and routes internally. This is what you use when there is a load balancer or service mesh between clusters, and it is the standard mode for Elastic Cloud and most managed deployments.

Both modes use the **transport protocol** on port 9300 (Elasticsearch) or 9300 (OpenSearch) by default — *not* the REST/HTTP port 9200. The transport port must be reachable from the searching cluster to the remote cluster. mTLS on the transport layer is the production norm and is described in §9.

---

## 3. CCS Query Syntax and Resilience Flags

Once a remote cluster is registered, querying it is a matter of prefixing the index name with the alias:

```http
GET logs_eu:nginx-2026-05-07/_search
{ "query": { "match": { "url.path": "/checkout" } } }
```

Local and remote indices freely mix in a single request — the parser splits each comma-separated index expression on the colon and routes accordingly:

```http
GET nginx-local-*,logs_eu:nginx-*,logs_us:nginx-*/_search
{ "query": { "range": { "@timestamp": { "gte": "now-1h" } } } }
```

Wildcards work on both sides of the colon, so `*:logs-*` fans out to every registered remote cluster's `logs-*` indices.

**`skip_unavailable`** (per-cluster setting) controls what happens when a remote cluster is down or unreachable. When `true`, the search continues with whatever clusters did respond and reports the failed cluster in `_clusters.skipped`. When `false`, the entire request fails. In Elasticsearch 8.15 the default flipped from `false` to `true` — search failing the whole query because one of three remote regions was being patched was the wrong default. For DR-grade availability you want this `true` on every remote and explicit handling of `_clusters.skipped` in the client.

The response carries cluster-level status in the `_clusters` block:

```json
{
  "_clusters": {
    "total": 3,
    "successful": 2,
    "skipped": 1,
    "details": {
      "logs_eu": { "status": "successful", "indices": "nginx-*", "took": 142 },
      "logs_us": { "status": "successful", "indices": "nginx-*", "took": 187 },
      "(local)": { "status": "successful", "indices": "nginx-local-*", "took": 31 }
    }
  },
  "hits": { ... }
}
```

A Spring Boot or Node client should always inspect `_clusters.skipped` and surface partial-result warnings to the caller. Treating a `skipped: 1` response as "all good" silently hides regional outages.

---

## 4. `ccs_minimize_roundtrips` — What the Flag Actually Does

CCS supports two execution strategies, controlled by the `ccs_minimize_roundtrips` query parameter.

**Minimized (default for synchronous search, `true`)** — the coordinating node sends a *single* search request per remote cluster. The remote cluster runs the full query/fetch locally and returns final hits. The local coordinator then merges per-cluster responses. Network: one round trip per remote.

**Not minimized (`false`)** — the coordinating node treats each remote shard as if it were local, opening a per-shard request through the transport connection. Network: one round trip per shard.

```http
GET logs_eu:nginx-*/_search?ccs_minimize_roundtrips=false
{ "query": { ... } }
```

Minimized is dramatically better for high-latency links (intercontinental, mTLS-heavy) because it collapses N shard requests into 1. It is the documented default for sync search. The not-minimized mode survives because it allows shard-level features (early termination, certain async-search code paths) that the minimized path could not always express. For async search the default is `false`.

The mental model: the cost of CCS is dominated by inter-cluster RTT, not by shard count, so the engine fights to make at most one RTT per cluster.

---

## 5. Cross-Cluster Replication — Elastic CCR

Elastic CCR is a **paid Enterprise-subscription feature** in self-managed Elasticsearch and Elastic Cloud. The Elastic subscriptions matrix explicitly lists "Cross-cluster replication" under the Enterprise tier only, with a footnote that all participating clusters must be on the same tier. Verify this against your contract before architecting DR around it — this is the single most consequential licensing fact in the cluster-ops space.

The model is **leader/follower**, asynchronous, pull-based:

- Writes go to a **leader index** on the leader cluster — exactly like a normal index.
- A **follower index** is created on the follower cluster pointing at a leader index.
- Each follower shard polls the corresponding leader shard for new operations and applies them locally.
- The follower index is read-only — search works, indexing requests are rejected.

The leader index must have **soft deletes enabled** (`index.soft_deletes.enabled: true`, the default for indices created in Elasticsearch 7.0+). Soft deletes are how Lucene keeps a recent history of operations available for the follower to pull. Operations are retained on the leader for `index.soft_deletes.retention_lease.period` (default 12 hours). If a follower is offline longer than that lease, the leader can purge the operations the follower needs and the follower must be re-bootstrapped from a fresh restore. Re-bootstrap is a full file-level copy, not a replay, so a 24-hour outage of a multi-TB follower is a several-hour recovery, not a millisecond catch-up.

```http
PUT /nginx-follower/_ccr/follow
{
  "remote_cluster": "logs_eu",
  "leader_index": "nginx-2026-05-07",
  "max_read_request_operation_count": 5120,
  "max_outstanding_read_requests": 12
}
```

An **auto-follow pattern** subscribes a follower cluster to leader index name patterns (`nginx-*`) so that as new daily indices are created on the leader, matching follower indices are created and started automatically. This is what makes CCR practical for time-series workloads with daily index rollover.

---

## 6. OpenSearch Cross-Cluster Replication

OpenSearch ships its own `opensearch-cross-cluster-replication` plugin (Apache 2.0, included in the default distribution). The model deliberately mirrors Elastic CCR's: leader/follower indices, asynchronous pull-based replication, read-only followers, auto-follow patterns. The OpenSearch documentation describes it as "active-passive replication where you index to a leader index, and the data is replicated to one or more read-only follower indices," with each follower shard pulling changes from its corresponding leader shard.

What **carries over**:

- Leader/follower terminology and mechanics
- Read-only follower semantics
- Auto-follow patterns for time-series indices
- Operation-history retention requirement on the leader (12-hour default in OpenSearch as well)
- A `_replication/status` API analogous to `_ccr/stats`

What **differs** in practice:

- The OpenSearch plugin is open-source and bundled — no separate license tier.
- API surface uses `_replication/*` endpoints rather than Elastic's `_ccr/*`.
- Security integration uses OpenSearch's security plugin (roles, role mappings, internal users, mTLS configured per the security plugin's conventions) rather than Elastic's stack-wide security.
- Settings names diverge — always read the version-specific doc rather than translating from Elastic CCR by reflex.

The two implementations are not wire-compatible. You cannot follow an Elasticsearch leader from an OpenSearch follower or vice versa; replication endpoints, auth, and operation formats differ even though the concepts line up.

---

## 7. Conflict Handling, Lag, and `_ccr/stats`

Conflict handling is the easy part: there is no conflict resolution because there are no conflicting writers. The follower is read-only; all writes go to the leader. If application code accidentally points at the follower for writes, the request is rejected at the coordinating node with a clear error. This is the single biggest correctness advantage of leader/follower over multi-master: no last-write-wins, no vector clocks, no application-level reconciliation.

Lag is the hard part. Replication is asynchronous, so the follower trails the leader by some amount of time and operation count. The lag is observable per-shard via:

```http
GET /_ccr/stats
GET /<follower-index>/_ccr/stats
```

Key fields per shard:

- `leader_global_checkpoint` — the latest sequence number on the leader.
- `follower_global_checkpoint` — the latest applied sequence number on the follower.
- The difference is the operation-level lag.
- `time_since_last_read_millis` — when the follower last successfully fetched.
- `failed_read_requests`, `failed_write_requests` — error counters that go up when the leader is unreachable, when the retention lease lapses, or when mapping drift breaks an apply (see §9).

OpenSearch exposes equivalent fields via `GET _plugins/_replication/<follower-index>/_status`. A production deployment should scrape these into Prometheus and alert on (a) global-checkpoint gap exceeding an SLO and (b) `failed_read_requests` non-zero. The lag SLO is a business decision — for DR, sub-minute is often the target; for analytics offload, an hour is fine.

---

## 8. Use Cases — DR, Geo-Distribution, Hot/Cold Split

**Disaster recovery.** Leader in `us-east-1`, follower in `us-west-2`. On regional failure, application traffic fails over to the DR region; the follower indices are unfollowed (`POST /<index>/_ccr/unfollow`) which makes them writable, and the application starts indexing into the former follower. Failback later requires re-establishing replication in the opposite direction — non-trivial and worth rehearsing as a game day.

**Geo-distributed search.** Each region runs a leader for its own users' data. A central "global search" cluster runs CCS against all regional leaders. Local users get low-latency search from their region; admins get a global view through CCS. This composes cleanly with data residency: data never *leaves* its region, but global queries can fan out to it.

**Hot/cold separation across clusters.** A "hot" cluster takes live ingestion and serves recent queries. CCR replicates each rolled-over daily index to a "cold" cluster sized for storage rather than write throughput. The cold cluster runs heavy aggregations and historical search without contending for hot-cluster resources. CCS on the application side (or on a small router cluster) presents a unified index pattern across both.

**Tenant isolation.** One cluster per tenant or tenant-tier. CCS aggregates for cross-tenant admin queries. Blast radius is one cluster, not one shared monolith.

---

## 9. Practical Traps

**Mapping drift between leader and follower.** The follower receives operations *and* mapping updates from the leader, but only mappings that arrive through the replication stream. If someone manually adds a field mapping on the follower, or if the follower was restored from a different snapshot, replicated operations may reference fields that don't exist or have a different type — the follower will fail with `IllegalArgumentException` on apply and `failed_write_requests` will climb. Treat the follower as fully managed by replication; never write to it, never alter its mappings.

**Retention lease expiration.** The leader keeps operation history for 12 hours by default. A follower that goes offline longer than that loses its place and must be reseeded from a snapshot or re-followed (which triggers a file-level copy). For DR designs that assume long maintenance windows on the follower, raise `index.soft_deletes.retention_lease.period` consciously — at the cost of more disk on the leader.

**Security and trust between clusters.** Both Elastic and OpenSearch require the leader and follower to trust each other's transport-layer certificates. mTLS is the production answer: both clusters present X.509 certs from a CA both sides trust, and roles on the leader cluster grant the follower-cluster service account `read_ccr` (Elastic) or the equivalent replication permissions (OpenSearch). Skipping mTLS — especially across the public internet or across cloud VPCs — is the most common security finding in CCR/CCS deployments.

**`skip_unavailable` defaulting matters for client correctness.** Pre-Elasticsearch-8.15 deployments defaulted to `false`, which meant a single dead remote took down search. Post-8.15 defaults to `true`, which silently degrades unless the client inspects `_clusters.skipped`. Pick one default behavior, document it, and make it explicit in the cluster settings rather than relying on the version default.

**Transport port reachability.** Sniff mode opens many TCP connections to remote gateway nodes on port 9300/9300. Cloud security groups, on-prem firewalls, and load balancers that only forward 9200 will silently break sniff mode while the registration call succeeds. Proxy mode is the safer default in any deployment with non-trivial network topology.

**CCR is asynchronous — never confused with synchronous replication.** A leader-side ack does not mean the follower has the operation. RPO (recovery-point objective) is bounded only by replication lag, not zero. Applications that need synchronous cross-region durability need a different architecture (multi-region datastore with synchronous quorum, with search downstream of it).

---

## Related

- _Snapshot, Restore, and Searchable Snapshots (planned)_ — the other half of DR; complements CCR rather than replacing it
- _Cluster State and Master Election (planned)_ — within-cluster coordination that CCR sits on top of
- [Networking — TLS and Mutual TLS](../../networking/INDEX.md) — the transport-layer trust model that CCR/CCS depend on
- [System Design — Multi-Region Architectures](../../system-design/INDEX.md) — broader RPO/RTO and active-passive vs active-active framing
- [Database — Replication Models](../../database/INDEX.md) — leader/follower vs multi-master, applied to relational stores; the conceptual parallel to CCR
- [Observability — SLI/SLO Design](../../observability/INDEX.md) — how to express replication-lag SLOs for an alerting policy

## References

- [Elasticsearch Reference — Search across clusters](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cross-cluster-search.html) — CCS configuration, sniff vs proxy mode, `skip_unavailable`, `ccs_minimize_roundtrips`. Explicitly notes the 8.15 default change for `skip_unavailable`.
- [Elasticsearch Reference — Cross-cluster replication](https://www.elastic.co/guide/en/elasticsearch/reference/current/xpack-ccr.html) — leader/follower model, soft-deletes requirement, retention leases, follower task mechanics.
- [Elasticsearch Reference — Get follower stats API (`_ccr/stats`)](https://www.elastic.co/guide/en/elasticsearch/reference/current/ccr-get-follow-stats.html) — global checkpoint fields and lag observability.
- [Elastic Subscriptions](https://www.elastic.co/subscriptions) — confirms Cross-cluster replication is included only in the **Enterprise** self-managed/cloud tier, with a footnote that all participating clusters must share the same subscription tier.
- [OpenSearch Documentation — Cross-cluster replication](https://docs.opensearch.org/latest/tuning-your-cluster/replication-plugin/index/) — leader/follower model and active-passive description for the OpenSearch plugin.
- [OpenSearch — Replication plugin API](https://docs.opensearch.org/latest/tuning-your-cluster/replication-plugin/api/) — `_replication/*` endpoints including status/stats analogous to `_ccr/stats`.
- [Elasticsearch Reference — Remote clusters](https://www.elastic.co/guide/en/elasticsearch/reference/current/remote-clusters.html) — sniff vs proxy connection modes and transport-layer requirements; mTLS guidance referenced in §9.
