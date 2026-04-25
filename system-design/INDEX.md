# System Design Documentation Index — Learning Path

A progressive path from mental models and building blocks through scalability, data consistency, reliability, and real-world design problems. System design is where the other learning paths compose — databases, protocols, caches, queues, and services become architectures that have to survive traffic, failure, and change.

Cross-references to the [TypeScript learning path](../typescript/INDEX.md), the [Java learning path](../java/INDEX.md), the [Database learning path](../database/INDEX.md), the [Networking learning path](../networking/INDEX.md), and the [Kubernetes learning path](../kubernetes/INDEX.md) where topics overlap.

If you already know how to build and deploy a service, **start at Tier 1 (Foundations & Mental Models)** to install the reasoning vocabulary you'll use in every design discussion, then work forward. Tier 10 (case studies) exercises everything from Tiers 1–9 against real problems. Tiers 12–13 cover the algorithms and data-engineering surface most backend engineers eventually touch.

---

## Tier 1 — Foundations & Mental Models

Groundwork: what system design is, how to reason about trade-offs, and the numbers every designer should memorize before drawing a single box.

1. [Back-of-Envelope Estimation — Latency Numbers, QPS Math, and Capacity Planning](foundations/back-of-envelope-estimation.md) — memorized latency table, QPS → storage → bandwidth conversions, Fermi estimates, sizing servers, caches, and queues _(2026-04-24)_
2. [CAP, PACELC, and Consistency Models — Strong, Eventual, Causal, Linearizable](foundations/cap-and-consistency-models.md) — CAP theorem in practice, PACELC extension, consistency hierarchy, read-your-writes, monotonic reads, session guarantees _(2026-04-24)_
3. [SLA, SLO, SLI, and the Math of Availability — Error Budgets and Composition](foundations/sla-slo-sli-and-availability.md) — nines vs minutes, serial vs parallel availability composition, error budgets, burn rate alerts, SLO-driven engineering _(2026-04-24)_
4. [Non-Functional Requirements Checklist — Scalability, Availability, Durability, Cost](foundations/non-functional-requirements.md) — the NFR taxonomy, how to elicit requirements, trade-off articulation, what "scale" actually means per dimension _(2026-04-24)_
5. [Core Trade-offs Catalog — The Axes Every Design Must Resolve](foundations/core-tradeoffs-catalog.md) — concurrency vs parallelism, push vs pull, stateful vs stateless, strong vs eventual, sync vs async, latency vs throughput; "when you see X, articulate the trade-off as Y" reference _(planned)_

---

## Tier 2 — Core Building Blocks

The named components every design reuses. One doc per block, deep enough to reason about trade-offs instead of just naming them.

6. [Load Balancers in System Design — L4 vs L7, Algorithms, and Health](building-blocks/load-balancers.md) — where an LB fits in the design, stickiness, graceful drain, global vs regional vs local LBs, LB as an SPOF, cross-ref to networking _(2026-04-24)_
7. [Caching Layers — Client, CDN, Reverse-Proxy, Application, Distributed](building-blocks/caching-layers.md) — the caching hierarchy, cache-aside vs read/write-through, invalidation strategies, TTLs, stampede protection, Redis vs Memcached _(2026-04-24)_
8. [Databases as a Component — SQL, NoSQL, NewSQL, and Picking One](building-blocks/databases-as-a-component.md) — OLTP taxonomy, document vs key-value vs wide-column vs graph, NewSQL trade-offs, polyglot persistence, decision framework _(2026-04-24)_
9. [Message Queues & Brokers — Kafka, RabbitMQ, SQS, NATS](building-blocks/message-queues-and-brokers.md) — log vs queue semantics, delivery guarantees, partitioning, consumer groups, when to choose which broker _(2026-04-24)_
10. [Object and Blob Storage — S3-Style Systems, Chunking, Presigned URLs](building-blocks/object-and-blob-storage.md) — flat namespace, eventual consistency (now strong on S3), multipart upload, lifecycle policies, presigned URLs, cold tiers _(2026-04-24)_
11. [Search Systems — Inverted Index, Elasticsearch / OpenSearch](building-blocks/search-systems.md) — what a search engine really is, index pipeline, relevance scoring, facets, when search belongs next to vs on top of the DB _(2026-04-24)_
12. [API Gateways and BFFs — Routing, Auth Offload, Composition](building-blocks/api-gateways-and-bff.md) — gateway responsibilities, BFF pattern, gateway vs service mesh, common anti-patterns, Kong/Envoy/AWS API Gateway/Apollo _(2026-04-24)_
13. [Rate Limiters — Token Bucket, Leaky Bucket, Sliding Window](building-blocks/rate-limiters.md) — algorithms with pseudocode, distributed rate limiting with Redis, per-user vs per-IP vs per-key, graceful degradation _(2026-04-24)_
14. [API Design Styles — REST, GraphQL, gRPC, and Data Formats](building-blocks/api-design-styles.md) — REST vs GraphQL vs gRPC trade-offs, data formats (JSON, Protobuf, Avro, MessagePack), versioning models, when each style wins _(planned)_
15. [REST API Design in Depth — Resources, Pagination, Versioning, Errors](building-blocks/rest-api-design-in-depth.md) — resource naming, pagination patterns (offset vs cursor vs keyset), versioning approaches, error envelopes, idempotency at the API layer, HATEOAS reality check _(planned)_
16. [Distributed File Systems & Erasure Coding](building-blocks/distributed-file-systems-and-erasure-coding.md) — HDFS/GFS lineage, Ceph, MinIO erasure modes, erasure coding vs replication cost/durability math, when DFS replaces or supplements object storage _(planned)_

---

## Tier 3 — Scalability Patterns

How you grow a system past a single box. These patterns compose on top of the Tier 2 building blocks.

17. [Horizontal vs Vertical Scaling & Stateless Services](scalability/horizontal-vs-vertical-and-stateless.md) — scale-up vs scale-out economics, the stateless invariant, session externalization, autoscaling signals, warm pools _(2026-04-24)_
18. [Sharding Strategies — Range, Hash, Directory, Geo, Consistent Hashing](scalability/sharding-strategies.md) — shard key selection, hot spots, rebalancing, consistent hashing with virtual nodes, cross-shard queries, resharding playbook _(2026-04-24)_
19. [Replication Patterns — Primary-Replica, Multi-Primary, Quorum](scalability/replication-patterns.md) — sync vs async replication, replication lag, failover, split-brain, multi-primary conflict resolution, quorum reads/writes intro _(2026-04-24)_
20. [Read/Write Splitting & Cache Strategies — Through, Back, Around, Aside](scalability/read-write-splitting-and-cache-strategies.md) — write-through/back/around, cache-aside in depth, TTL vs event invalidation, stampede locks, negative caching, thundering herd _(2026-04-24)_
21. [CQRS and Event Sourcing — Commands, Queries, and the Event Log](scalability/cqrs-and-event-sourcing.md) — why split read and write models, event sourcing mechanics, projections, snapshotting, trade-offs vs classic CRUD _(2026-04-24)_
22. [Backpressure, Bulkhead, and Circuit Breakers — Failing Loudly Under Load](scalability/backpressure-bulkhead-circuit-breaker.md) — reactive streams backpressure, bulkhead isolation, three-state circuit breaker, timeouts vs deadlines, load shedding _(2026-04-24)_
23. [Read-Path Optimizations — Denormalization, Materialized Views, Compression](scalability/read-path-optimizations.md) — denormalization patterns, materialized views (pull and pre-aggregated), covering and projection indexes, row vs column storage, compression (Zstd/LZ4/Snappy) trade-offs _(planned)_

---

## Tier 4 — Data & Consistency

Distributed data is where most designs fail. This tier goes deeper than Tier 2's database overview.

24. [ACID vs BASE and Isolation Levels in Practice](data-consistency/acid-vs-base-and-isolation-levels.md) — isolation level anomalies (dirty/non-repeatable/phantom), snapshot isolation, serializable, BASE trade-offs _(2026-04-25)_
25. [Distributed Transactions — 2PC, 3PC, Sagas, Outbox](data-consistency/distributed-transactions.md) — why 2PC is rare, 3PC, saga orchestration vs choreography, the transactional outbox pattern, idempotent consumers _(2026-04-25)_
26. [Consensus — Raft and Paxos at a Conceptual Level](data-consistency/consensus-raft-and-paxos.md) — the problem consensus solves, Raft state machine, log replication, leader election, Paxos contrast _(2026-04-25)_
27. [Leader Election and Coordination Services — ZooKeeper, etcd](data-consistency/leader-election-and-coordination.md) — ephemeral nodes, watches, leases, fencing tokens, when to use ZK/etcd vs rolling your own _(2026-04-25)_
28. [Quorum Reads/Writes (NWR) and Tunable Consistency](data-consistency/quorum-and-tunable-consistency.md) — N/W/R math, read/write repair, hinted handoff, Dynamo-style systems, Cassandra consistency levels _(2026-04-25)_
29. [Time and Ordering in Distributed Systems — Logical Clocks, Lamport, Vector, HLC](data-consistency/time-and-ordering.md) — clock synchronization problem, NTP, Lamport timestamps, vector clocks, hybrid logical clocks, why ordering is the prerequisite for consistency _(2026-04-25)_
30. [Change Data Capture (CDC) and Dual Writes](data-consistency/change-data-capture.md) — why dual writes break, CDC via logical replication, Debezium architecture, CDC → Kafka → consumers _(2026-04-25)_
31. [OLTP vs OLAP, Data Lakes, and Lakehouses](data-consistency/oltp-vs-olap-and-lakehouses.md) — row vs columnar storage, warehouse vs lake vs lakehouse, Iceberg/Delta/Hudi, the analytics side of system design _(2026-04-25)_

---

## Tier 5 — Communication & Messaging

How services talk. Sync vs async, delivery guarantees, streaming.

32. [Sync vs Async Communication — REST, gRPC, Messaging](communication/sync-vs-async-communication.md) — latency coupling, failure coupling, when async wins, request-response over queues, saga handoff _(2026-04-25)_
33. [Real-Time Channels — Long Polling, WebSockets, SSE, Webhooks, WebRTC](communication/real-time-channels.md) — decision matrix for each, scaling characteristics, fallback ladders, when to choose which channel and why _(2026-04-25)_
34. [Push vs Pull Architecture](communication/push-vs-pull-architecture.md) — feed generation push vs pull, polling overhead and patterns, hybrid push-pull, fan-out implications _(2026-04-25)_
35. [Event-Driven Architecture — Pub/Sub, Choreography vs Orchestration](communication/event-driven-architecture.md) — event types, event vs command, choreography trade-offs, orchestration with workflow engines _(2026-04-25)_
36. [Idempotency and Exactly-Once Semantics](communication/idempotency-and-exactly-once.md) — idempotency keys, dedup windows, effectively-once via idempotent consumers, transactional outbox intro _(2026-04-25)_
37. [Dead-Letter Queues, Retries, and Poison Messages](communication/dead-letter-queues-and-retries.md) — retry topologies, DLQ patterns, poison message handling, retry budgets, observability on DLQs _(2026-04-25)_
38. [Stream Processing — Kafka Streams, Flink, Windowing](communication/stream-processing.md) — event-time vs processing-time, tumbling/sliding/session windows, exactly-once in streams, state stores _(2026-04-25)_

---

## Tier 6 — Reliability & Resilience

Designing for failure instead of against it.

39. [Failure Modes and Fault Tolerance Taxonomy](reliability/failure-modes-and-fault-tolerance.md) — crash-stop, omission, timing, byzantine, gray failures, blast radius, failure domains _(planned)_
40. [Failure Detection — Heartbeats, Phi-Accrual, and Gossip (SWIM)](reliability/failure-detection.md) — heartbeat protocols, phi-accrual detector, SWIM gossip-based failure detection, fencing tokens revisited _(planned)_
41. [Network Partitions and Split Brain](reliability/network-partitions-and-split-brain.md) — how partitions actually manifest (asymmetric, gray, flaky), detection strategies, the choices split-brain forces, real outage examples _(planned)_
42. [Single Point of Failure Analysis](reliability/single-point-of-failure-analysis.md) — finding SPOFs in a design, "what breaks if this dies" drills, HA patterns by failure class, SPOF audit checklist _(planned)_
43. [Retry Strategies — Exponential Backoff, Jitter, Budgets](reliability/retry-strategies.md) — why naive retries make outages worse, full jitter, retry budgets, circuit-breaker integration _(planned)_
44. [Chaos Engineering and Game Days](reliability/chaos-engineering-and-game-days.md) — chaos principles, blast-radius containment, game day structure, what to simulate first _(planned)_
45. [Disaster Recovery — RPO, RTO, Backup Strategies](reliability/disaster-recovery.md) — RPO vs RTO, backup tiers, pilot light vs warm standby vs active-active, DR drills _(planned)_
46. [Multi-Region Architectures — Active-Active, Active-Passive, Geo-Routing](reliability/multi-region-architectures.md) — regional failure domains, replication topologies, conflict resolution, geo-DNS vs anycast vs global LB _(planned)_
47. [Graceful Degradation and Feature Flags](reliability/graceful-degradation-and-feature-flags.md) — degradation tiers, kill switches, circuit breaker + flag integration, flag lifecycle _(planned)_

---

## Tier 7 — Performance & Observability

Making it fast and knowing when it isn't.

48. [Performance Budgets and Latency Analysis — Tail Latency and Coordinated Omission](performance-observability/performance-budgets-and-latency.md) — averages lie, p99/p99.9, coordinated omission, hedged requests, latency SLO sizing _(planned)_
49. [Distributed Tracing, Metrics, and Logs — The Three Pillars](performance-observability/tracing-metrics-logs.md) — OpenTelemetry model, trace propagation, cardinality traps, structured logging, correlation IDs _(planned)_
50. [Log Aggregation and Structured Logging](performance-observability/log-aggregation-and-structured-logging.md) — ELK/Loki/Splunk topologies, cardinality limits, log-to-metric conversion, correlation IDs across services _(planned)_
51. [Monitoring and Alerting — RED, USE, and the Four Golden Signals](performance-observability/monitoring-red-use-golden-signals.md) — RED for services, USE for resources, golden signals, SLO-based alerting, alert fatigue _(planned)_
52. [Dashboards, Runbooks, and On-Call](performance-observability/dashboards-runbooks-on-call.md) — dashboard design principles, runbook structure, auto-remediation, SLO burn-down dashboards, on-call ergonomics _(planned)_
53. [Capacity Planning and Load Testing](performance-observability/capacity-planning-and-load-testing.md) — headroom planning, load test shapes (ramp/soak/spike), k6/Gatling/Locust, production shadowing _(planned)_

---

## Tier 8 — Security in System Design

Design-level security decisions — not code-level hardening.

54. [Authentication — Sessions, Tokens, JWT, and Refresh Flows](security/authentication.md) — session vs token, JWT structure and pitfalls, refresh tokens, sliding sessions, multi-device sign-in _(planned)_
55. [Authorization — RBAC, ABAC, ReBAC, and Policy Engines](security/authorization.md) — role-based, attribute-based, relationship-based authorization, OPA/Cedar/Zanzibar-style systems, central vs distributed authz _(planned)_
56. [Single Sign-On — SAML, OIDC, and Federation](security/single-sign-on.md) — SAML vs OIDC vs OAuth2, federation patterns, IdP-initiated vs SP-initiated, SCIM for provisioning _(planned)_
57. [Secrets Management and Key Rotation](security/secrets-management-and-key-rotation.md) — secret storage (Vault, SM, KMS), rotation strategies, envelope encryption, workload identity _(planned)_
58. [Encryption at Rest and in Transit](security/encryption-at-rest-and-in-transit.md) — at-rest: disk vs DB vs app; in-transit: TLS, mTLS, E2EE; KMS hierarchies, BYOK _(planned)_
59. [Defense in Depth and Threat Modeling](security/defense-in-depth-and-threat-modeling.md) — layered defense, STRIDE, trust boundaries, attack surface, threat model in the design review _(planned)_
60. [Zero Trust Architecture — BeyondCorp, Identity-Aware Proxies](security/zero-trust-architecture.md) — identity-centric model, microsegmentation, IAP, policy engines, what zero trust does NOT mean _(planned)_

---

## Tier 9 — Architectural Styles

How to slice the system.

61. [Monolith, Modular Monolith, SOA, Microservices — When Each Wins](architectural-styles/monolith-to-microservices.md) — tech vs org trade-offs, modular monolith as the default, microservices prerequisites, "distributed monolith" anti-pattern _(planned)_
62. [Event-Driven Architecture as a Style](architectural-styles/event-driven-architecture-style.md) — EDA beyond a pattern, event backbone, topology choices, when EDA is wrong _(planned)_
63. [Serverless and FaaS Trade-offs](architectural-styles/serverless-and-faas.md) — cold starts, state, concurrency limits, vendor lock-in, cost curves, good vs bad fits _(planned)_
64. [Hexagonal, Clean, and DDD — Ports, Adapters, Bounded Contexts](architectural-styles/hexagonal-clean-ddd.md) — domain-first design, ports/adapters, aggregates, bounded contexts, context maps _(planned)_
65. [Service Discovery — Client-Side, Server-Side, DNS-Based](architectural-styles/service-discovery.md) — client-side (Ribbon/Eureka) vs server-side (LB-fronted) vs DNS-based, Consul/etcd, mesh-integrated discovery _(planned)_
66. [Sidecar Pattern — Beyond Service Mesh](architectural-styles/sidecar-pattern.md) — sidecars for logging, config, crypto, traffic; Istio/Envoy/Linkerd comparison; resource overhead, ambient mesh alternative _(planned)_
67. [Strangler Fig Pattern — Incremental Migration](architectural-styles/strangler-fig-pattern.md) — routing splits, dual-run, cutover mechanics, anti-corruption layer, real-world migration timelines _(planned)_
68. [Service Mesh as an Architectural Decision](architectural-styles/service-mesh-as-architectural-decision.md) — what a mesh buys you, sidecar overhead, ambient mesh, when mesh is overkill _(planned)_
69. [Peer-to-Peer Architecture — DHTs, Gossip, BitTorrent, IPFS](architectural-styles/peer-to-peer-architecture.md) — Kademlia and Chord DHTs, gossip overlays, content-addressed storage, when P2P is the right answer _(planned)_

---

## Tier 10 — Case Studies: Designing Real Systems

The interview-style design problems. Each doc follows a consistent template: requirements → estimates → API → data model → high-level design → deep dives → bottlenecks → trade-offs.

70. [Design a URL Shortener](case-studies/design-url-shortener.md) — counter vs hash vs base62, redirect infra, click analytics, custom aliases, abuse protection _(planned)_
71. [Design a Rate Limiter (Service)](case-studies/design-rate-limiter.md) — centralized vs distributed, Redis-backed sliding window, policy engine, client SDKs, observability _(planned)_
72. [Design a Distributed Cache](case-studies/design-distributed-cache.md) — partitioning, replication, eviction, consistent hashing ring, client topology awareness _(planned)_
73. [Design a News Feed / Timeline (Twitter-Style)](case-studies/design-news-feed.md) — fan-out on write vs read, celebrity problem, timeline generation, ranking intro _(planned)_
74. [Design a Chat System (WhatsApp-Style)](case-studies/design-chat-system.md) — 1:1 vs group, presence, delivery receipts, message ordering, media handling, offline delivery _(planned)_
75. [Design a Notification System](case-studies/design-notification-system.md) — fan-out, channel routing (push/email/SMS), preferences, rate limiting, DLQ handling _(planned)_
76. [Design a Video Streaming Service (YouTube/Netflix-Style)](case-studies/design-video-streaming.md) — upload pipeline, transcoding, CDN distribution, ABR, recommendation intro _(planned)_
77. [Design a Ride-Sharing Service (Uber-Style)](case-studies/design-ride-sharing.md) — geo indexing (geohash/S2), dispatch matching, trip lifecycle, surge pricing, dual write concerns _(planned)_
78. [Design a File Storage & Sync Service (Dropbox-Style)](case-studies/design-file-storage-and-sync.md) — chunking, dedup, delta sync, metadata DB, block storage, mobile constraints _(planned)_
79. [Design a Collaborative Editor (Google Docs-Style)](case-studies/design-collaborative-editor.md) — OT vs CRDT, presence, cursor broadcast, offline merges, history _(planned)_
80. [Design a Search Autocomplete](case-studies/design-search-autocomplete.md) — trie vs prefix index, ranking, typo tolerance, freshness, personalization _(planned)_
81. [Design a Payment System](case-studies/design-payment-system.md) — idempotency, double-entry ledgers, reconciliation, fraud gates, async settlement _(planned)_
82. [Design a Metrics & Monitoring System (Datadog-Style)](case-studies/design-metrics-monitoring-system.md) — ingestion, time-series storage, rollups, alerting engine, dashboard compute _(planned)_
83. [Design an Ad Click Tracking Pipeline](case-studies/design-ad-click-tracking.md) — click-through capture, fraud filtering, attribution windows, real-time aggregates, billing correctness _(planned)_

---

## Tier 11 — The Interview Framework

How to actually run a system design discussion — useful even if you never interview again, because it mirrors how real architecture reviews should flow.

84. [The 6-Step Framework — Requirements → Estimates → API → Data Model → HLD → Deep Dives](interview-framework/six-step-framework.md) — the canonical structure, time allocation, where candidates stumble _(planned)_
85. [Trade-off Articulation and Bottleneck Analysis](interview-framework/tradeoff-articulation-and-bottlenecks.md) — naming trade-offs explicitly, identifying bottlenecks, the "what breaks at 10x?" drill _(planned)_
86. [Common Anti-Patterns in System Design Interviews](interview-framework/common-anti-patterns.md) — premature detail, resume-driven design, ignoring NFRs, hand-waving at scale _(planned)_

---

## Tier 12 — Data Structures for Scale

The algorithms and data structures that unlock system-design answers — what goes into the design diagram when "regular data structures won't scale".

87. [Bloom Filters & Cuckoo Filters — Probabilistic Membership Tests](data-structures/bloom-and-cuckoo-filters.md) — false-positive math, CDN caching, LSM read path, duplicate detection, when cuckoo beats bloom _(planned)_
88. [Merkle Trees — Efficient Diff at Scale](data-structures/merkle-trees.md) — anti-entropy in Dynamo/Cassandra, blockchain roots, rsync, S3 inventory comparisons, version trees _(planned)_
89. [HyperLogLog — Cardinality Estimation in Constant Memory](data-structures/hyperloglog.md) — accuracy vs memory math, Redis HLL, BigQuery APPROX_COUNT_DISTINCT, merging HLL sketches across shards _(planned)_
90. [Count-Min Sketch & Top-K — Frequency Estimation](data-structures/count-min-sketch-and-top-k.md) — heavy hitters, rate abuse detection, streaming aggregations, error bounds vs space _(planned)_
91. [Skip Lists — Probabilistic Balanced Order](data-structures/skip-lists.md) — Redis sorted sets internals, LSM memtables, lock-free variants, vs balanced trees in practice _(planned)_
92. [Consistent Hashing & Rendezvous Hashing — Beyond the Basics](data-structures/consistent-and-rendezvous-hashing.md) — virtual nodes math, jump hash, Maglev, rendezvous hashing comparison, hot-spot mitigation _(planned)_
93. [Geohash — Encoding 2D Coordinates as Strings](data-structures/geohash.md) — bit-interleaving, prefix-based proximity, neighbor calculation, geohash limits, S2 and H3 alternatives _(planned)_
94. [Quadtrees & R-Trees — Multi-Dimensional Indexing](data-structures/quadtrees-and-r-trees.md) — point quadtrees, region quadtrees, R-tree splits, ride-sharing dispatch, map rendering, k-NN queries _(planned)_

---

## Tier 13 — Batch & Stream Processing

Data engineering in system design. The analytics and pipeline side that most backend engineers eventually have to reason about.

95. [Batch vs Stream Processing](batch-and-stream/batch-vs-stream-processing.md) — when each wins, latency vs completeness trade-off, exactly-once in each model, hybrid patterns _(planned)_
96. [MapReduce and Its Descendants](batch-and-stream/mapreduce-and-descendants.md) — the model not just Hadoop, map/shuffle/reduce, Spark/Flink as evolutions, why MapReduce-as-API faded _(planned)_
97. [ETL, ELT, and Data Pipeline Architecture](batch-and-stream/etl-elt-and-pipelines.md) — Airflow, Dagster, Prefect, declarative vs imperative DAGs, data contracts, idempotent pipeline design _(planned)_
98. [Lambda vs Kappa Architecture](batch-and-stream/lambda-vs-kappa-architecture.md) — dual-pipeline vs stream-only, reprocessing strategies, when each still applies, modern alternatives _(planned)_
99. [Modern Streaming Engines — Flink, Spark Structured Streaming, Kafka Streams, Beam](batch-and-stream/modern-streaming-engines.md) — comparing engines, exactly-once across them, state backends, choosing for your workload _(planned)_

---

## Quick Reference by Topic

### Foundations

- [Back-of-Envelope Estimation](foundations/back-of-envelope-estimation.md)
- [CAP, PACELC, and Consistency Models](foundations/cap-and-consistency-models.md)
- [SLA, SLO, SLI, and Availability](foundations/sla-slo-sli-and-availability.md)
- [Non-Functional Requirements Checklist](foundations/non-functional-requirements.md)
- [Core Trade-offs Catalog](foundations/core-tradeoffs-catalog.md) _(planned)_

### Building Blocks

- [Load Balancers](building-blocks/load-balancers.md)
- [Caching Layers](building-blocks/caching-layers.md)
- [Databases as a Component](building-blocks/databases-as-a-component.md)
- [Message Queues & Brokers](building-blocks/message-queues-and-brokers.md)
- [Object and Blob Storage](building-blocks/object-and-blob-storage.md)
- [Search Systems](building-blocks/search-systems.md)
- [API Gateways and BFFs](building-blocks/api-gateways-and-bff.md)
- [Rate Limiters](building-blocks/rate-limiters.md)
- [API Design Styles](building-blocks/api-design-styles.md) _(planned)_
- [REST API Design in Depth](building-blocks/rest-api-design-in-depth.md) _(planned)_
- [Distributed File Systems & Erasure Coding](building-blocks/distributed-file-systems-and-erasure-coding.md) _(planned)_

### Scalability Patterns

- [Horizontal vs Vertical Scaling](scalability/horizontal-vs-vertical-and-stateless.md)
- [Sharding Strategies](scalability/sharding-strategies.md)
- [Replication Patterns](scalability/replication-patterns.md)
- [Read/Write Splitting & Cache Strategies](scalability/read-write-splitting-and-cache-strategies.md)
- [CQRS and Event Sourcing](scalability/cqrs-and-event-sourcing.md)
- [Backpressure, Bulkhead, Circuit Breaker](scalability/backpressure-bulkhead-circuit-breaker.md)
- [Read-Path Optimizations](scalability/read-path-optimizations.md) _(planned)_

### Data & Consistency

- [ACID vs BASE and Isolation Levels](data-consistency/acid-vs-base-and-isolation-levels.md)
- [Distributed Transactions (2PC, 3PC, Sagas, Outbox)](data-consistency/distributed-transactions.md)
- [Consensus (Raft, Paxos)](data-consistency/consensus-raft-and-paxos.md)
- [Leader Election and Coordination](data-consistency/leader-election-and-coordination.md)
- [Quorum and Tunable Consistency](data-consistency/quorum-and-tunable-consistency.md)
- [Time and Ordering (Lamport, Vector, HLC)](data-consistency/time-and-ordering.md)
- [Change Data Capture](data-consistency/change-data-capture.md)
- [OLTP vs OLAP, Lakehouses](data-consistency/oltp-vs-olap-and-lakehouses.md)

### Communication & Messaging

- [Sync vs Async Communication](communication/sync-vs-async-communication.md)
- [Real-Time Channels (Long Polling, WebSockets, SSE, Webhooks, WebRTC)](communication/real-time-channels.md)
- [Push vs Pull Architecture](communication/push-vs-pull-architecture.md)
- [Event-Driven Architecture](communication/event-driven-architecture.md)
- [Idempotency and Exactly-Once](communication/idempotency-and-exactly-once.md)
- [Dead-Letter Queues and Retries](communication/dead-letter-queues-and-retries.md)
- [Stream Processing](communication/stream-processing.md)

### Reliability & Resilience _(planned)_

- Failure Modes and Fault Tolerance
- Failure Detection (Heartbeats, Phi-Accrual, SWIM)
- Network Partitions and Split Brain
- Single Point of Failure Analysis
- Retry Strategies
- Chaos Engineering and Game Days
- Disaster Recovery
- Multi-Region Architectures
- Graceful Degradation and Feature Flags

### Performance & Observability _(planned)_

- Performance Budgets and Latency
- Distributed Tracing, Metrics, Logs
- Log Aggregation and Structured Logging
- Monitoring (RED/USE/Golden Signals)
- Dashboards, Runbooks, On-Call
- Capacity Planning and Load Testing

### Security in System Design _(planned)_

- Authentication (Sessions, Tokens, JWT)
- Authorization (RBAC, ABAC, ReBAC)
- Single Sign-On (SAML, OIDC, Federation)
- Secrets Management and Key Rotation
- Encryption at Rest and in Transit
- Defense in Depth and Threat Modeling
- Zero Trust Architecture

### Architectural Styles _(planned)_

- Monolith → Microservices
- Event-Driven Architecture (style)
- Serverless and FaaS
- Hexagonal / Clean / DDD
- Service Discovery
- Sidecar Pattern
- Strangler Fig Pattern
- Service Mesh (architectural decision)
- Peer-to-Peer Architecture

### Case Studies _(planned)_

- URL Shortener, Rate Limiter, Distributed Cache
- News Feed, Chat, Notifications
- Video Streaming, Ride-Sharing, File Sync
- Collaborative Editor, Search Autocomplete
- Payment System, Metrics/Monitoring, Ad Click Tracking

### Interview Framework _(planned)_

- 6-Step Framework
- Trade-off Articulation and Bottleneck Analysis
- Common Anti-Patterns

### Data Structures for Scale _(planned)_

- Bloom & Cuckoo Filters
- Merkle Trees
- HyperLogLog
- Count-Min Sketch & Top-K
- Skip Lists
- Consistent & Rendezvous Hashing
- Geohash
- Quadtrees & R-Trees

### Batch & Stream Processing _(planned)_

- Batch vs Stream Processing
- MapReduce and Its Descendants
- ETL, ELT, and Pipelines
- Lambda vs Kappa Architecture
- Modern Streaming Engines

---

## Cross-Path Reading

- [Networking Path](../networking/INDEX.md) — the wire-level detail behind load balancers, TLS, WebSocket, gRPC, CDN, service mesh.
- [Database Path](../database/INDEX.md) — the engine-internals (B-tree, LSM, MVCC, query planner) and query optimization layer underneath "databases as a component".
- [Kubernetes Path](../kubernetes/INDEX.md) — how the scalability and reliability patterns materialize as platform primitives.
- [Java Path](../java/INDEX.md) — Spring Boot / reactive / production guidance for the JVM side of the designs.
- [TypeScript Path](../typescript/INDEX.md) — Node/TS side of the same designs, plus compiler and V8 internals where performance matters.
