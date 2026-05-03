# System Design Documentation Index — Learning Path

A progressive path from mental models and building blocks through scalability, data consistency, reliability, and real-world design problems. System design is where the other learning paths compose — databases, protocols, caches, queues, and services become architectures that have to survive traffic, failure, and change.

Cross-references to the [Low-Level Design learning path](../low-level-design/INDEX.md) (class-level OOD, the layer below this one), the [TypeScript learning path](../typescript/INDEX.md), the [Java learning path](../java/INDEX.md), the [Database learning path](../database/INDEX.md), the [Networking learning path](../networking/INDEX.md), the [Kubernetes learning path](../kubernetes/INDEX.md), the [Operating Systems learning path](../operating-systems/INDEX.md) (Linux primitives behind every service), the [Observability learning path](../observability/INDEX.md) (the curriculum behind the observability tier), the [Performance Engineering learning path](../performance/INDEX.md) (latency/throughput/capacity grounding for back-of-envelope), and the [Security learning path](../security/INDEX.md) (the curriculum behind the security tier) where topics overlap.

If you already know how to build and deploy a service, **start at Tier 1 (Foundations & Mental Models)** to install the reasoning vocabulary you'll use in every design discussion, then work forward. Tier 10 (case studies) exercises everything from Tiers 1–9 against real problems. Tiers 12–13 cover the algorithms and data-engineering surface most backend engineers eventually touch.

**Markers:** **★** = core must-learn (asked in nearly every senior system-design interview, used in everyday backend work). **○** = supporting deep-dive (specialized, advanced, or optional). Aim to internalize all ★ entries before going deep on ○ entries.

---

## Tier 1 — Foundations & Mental Models

Groundwork: what system design is, how to reason about trade-offs, and the numbers every designer should memorize before drawing a single box.

1. [★ Back-of-Envelope Estimation — Latency Numbers, QPS Math, and Capacity Planning](foundations/back-of-envelope-estimation.md) — memorized latency table, QPS → storage → bandwidth conversions, Fermi estimates, sizing servers, caches, and queues _(2026-04-24)_
2. [★ CAP, PACELC, and Consistency Models — Strong, Eventual, Causal, Linearizable](foundations/cap-and-consistency-models.md) — CAP theorem in practice, PACELC extension, consistency hierarchy, read-your-writes, monotonic reads, session guarantees _(2026-04-24)_
3. [★ SLA, SLO, SLI, and the Math of Availability — Error Budgets and Composition](foundations/sla-slo-sli-and-availability.md) — nines vs minutes, serial vs parallel availability composition, error budgets, burn rate alerts, SLO-driven engineering _(2026-04-24)_
4. [★ Non-Functional Requirements Checklist — Scalability, Availability, Durability, Cost](foundations/non-functional-requirements.md) — the NFR taxonomy, how to elicit requirements, trade-off articulation, what "scale" actually means per dimension _(2026-04-24)_
5. [★ Core Trade-offs Catalog — The Axes Every Design Must Resolve](foundations/core-tradeoffs-catalog.md) — concurrency vs parallelism, push vs pull, stateful vs stateless, strong vs eventual, sync vs async, latency vs throughput; "when you see X, articulate the trade-off as Y" reference _(2026-04-26)_

---

## Tier 2 — Core Building Blocks

The named components every design reuses. One doc per block, deep enough to reason about trade-offs instead of just naming them.

6. [★ Load Balancers in System Design — L4 vs L7, Algorithms, and Health](building-blocks/load-balancers.md) — where an LB fits in the design, stickiness, graceful drain, global vs regional vs local LBs, LB as an SPOF, cross-ref to networking _(2026-04-24)_
7. [★ Caching Layers — Client, CDN, Reverse-Proxy, Application, Distributed](building-blocks/caching-layers.md) — the caching hierarchy, cache-aside vs read/write-through, invalidation strategies, TTLs, stampede protection, Redis vs Memcached _(2026-04-24)_
8. [★ Databases as a Component — SQL, NoSQL, NewSQL, and Picking One](building-blocks/databases-as-a-component.md) — OLTP taxonomy, document vs key-value vs wide-column vs graph, NewSQL trade-offs, polyglot persistence, decision framework _(2026-04-24)_
9. [★ Message Queues & Brokers — Kafka, RabbitMQ, SQS, NATS](building-blocks/message-queues-and-brokers.md) — log vs queue semantics, delivery guarantees, partitioning, consumer groups, when to choose which broker _(2026-04-24)_
10. [★ Object and Blob Storage — S3-Style Systems, Chunking, Presigned URLs](building-blocks/object-and-blob-storage.md) — flat namespace, eventual consistency (now strong on S3), multipart upload, lifecycle policies, presigned URLs, cold tiers _(2026-04-24)_
11. [○ Search Systems — Inverted Index, Elasticsearch / OpenSearch](building-blocks/search-systems.md) — what a search engine really is, index pipeline, relevance scoring, facets, when search belongs next to vs on top of the DB _(2026-04-24)_
12. [★ API Gateways and BFFs — Routing, Auth Offload, Composition](building-blocks/api-gateways-and-bff.md) — gateway responsibilities, BFF pattern, gateway vs service mesh, common anti-patterns, Kong/Envoy/AWS API Gateway/Apollo _(2026-04-24)_
13. [★ Rate Limiters — Token Bucket, Leaky Bucket, Sliding Window](building-blocks/rate-limiters.md) — algorithms with pseudocode, distributed rate limiting with Redis, per-user vs per-IP vs per-key, graceful degradation _(2026-04-24)_
14. [★ API Design Styles — REST, GraphQL, gRPC, and Data Formats](building-blocks/api-design-styles.md) — REST vs GraphQL vs gRPC trade-offs, data formats (JSON, Protobuf, Avro, MessagePack), versioning models, when each style wins _(2026-04-26)_
15. [★ REST API Design in Depth — Resources, Pagination, Versioning, Errors](building-blocks/rest-api-design-in-depth.md) — resource naming, pagination patterns (offset vs cursor vs keyset), versioning approaches, error envelopes, idempotency at the API layer, HATEOAS reality check _(2026-04-26)_
16. [○ Distributed File Systems & Erasure Coding](building-blocks/distributed-file-systems-and-erasure-coding.md) — HDFS/GFS lineage, Ceph, MinIO erasure modes, erasure coding vs replication cost/durability math, when DFS replaces or supplements object storage _(2026-04-26)_

---

## Tier 3 — Scalability Patterns

How you grow a system past a single box. These patterns compose on top of the Tier 2 building blocks.

17. [★ Horizontal vs Vertical Scaling & Stateless Services](scalability/horizontal-vs-vertical-and-stateless.md) — scale-up vs scale-out economics, the stateless invariant, session externalization, autoscaling signals, warm pools _(2026-04-24)_
18. [★ Sharding Strategies — Range, Hash, Directory, Geo, Consistent Hashing](scalability/sharding-strategies.md) — shard key selection, hot spots, rebalancing, consistent hashing with virtual nodes, cross-shard queries, resharding playbook _(2026-04-24)_
19. [★ Replication Patterns — Primary-Replica, Multi-Primary, Quorum](scalability/replication-patterns.md) — sync vs async replication, replication lag, failover, split-brain, multi-primary conflict resolution, quorum reads/writes intro _(2026-04-24)_
20. [★ Read/Write Splitting & Cache Strategies — Through, Back, Around, Aside](scalability/read-write-splitting-and-cache-strategies.md) — write-through/back/around, cache-aside in depth, TTL vs event invalidation, stampede locks, negative caching, thundering herd _(2026-04-24)_
21. [○ CQRS and Event Sourcing — Commands, Queries, and the Event Log](scalability/cqrs-and-event-sourcing.md) — why split read and write models, event sourcing mechanics, projections, snapshotting, trade-offs vs classic CRUD _(2026-04-24)_
22. [★ Backpressure, Bulkhead, and Circuit Breakers — Failing Loudly Under Load](scalability/backpressure-bulkhead-circuit-breaker.md) — reactive streams backpressure, bulkhead isolation, three-state circuit breaker, timeouts vs deadlines, load shedding _(2026-04-24)_
23. [○ Read-Path Optimizations — Denormalization, Materialized Views, Compression](scalability/read-path-optimizations.md) — denormalization patterns, materialized views (pull and pre-aggregated), covering and projection indexes, row vs column storage, compression (Zstd/LZ4/Snappy) trade-offs _(2026-04-26)_

---

## Tier 4 — Data & Consistency

Distributed data is where most designs fail. This tier goes deeper than Tier 2's database overview.

24. [★ ACID vs BASE and Isolation Levels in Practice](data-consistency/acid-vs-base-and-isolation-levels.md) — isolation level anomalies (dirty/non-repeatable/phantom), snapshot isolation, serializable, BASE trade-offs _(2026-04-25)_
25. [★ Distributed Transactions — 2PC, 3PC, Sagas, Outbox](data-consistency/distributed-transactions.md) — why 2PC is rare, 3PC, saga orchestration vs choreography, the transactional outbox pattern, idempotent consumers _(2026-04-25)_
26. [★ Consensus — Raft and Paxos at a Conceptual Level](data-consistency/consensus-raft-and-paxos.md) — the problem consensus solves, Raft state machine, log replication, leader election, Paxos contrast _(2026-04-25)_
27. [○ Leader Election and Coordination Services — ZooKeeper, etcd](data-consistency/leader-election-and-coordination.md) — ephemeral nodes, watches, leases, fencing tokens, when to use ZK/etcd vs rolling your own _(2026-04-25)_
28. [★ Quorum Reads/Writes (NWR) and Tunable Consistency](data-consistency/quorum-and-tunable-consistency.md) — N/W/R math, read/write repair, hinted handoff, Dynamo-style systems, Cassandra consistency levels _(2026-04-25)_
29. [○ Time and Ordering in Distributed Systems — Logical Clocks, Lamport, Vector, HLC](data-consistency/time-and-ordering.md) — clock synchronization problem, NTP, Lamport timestamps, vector clocks, hybrid logical clocks, why ordering is the prerequisite for consistency _(2026-04-25)_
30. [★ Change Data Capture (CDC) and Dual Writes](data-consistency/change-data-capture.md) — why dual writes break, CDC via logical replication, Debezium architecture, CDC → Kafka → consumers _(2026-04-25)_
31. [★ OLTP vs OLAP, Data Lakes, and Lakehouses](data-consistency/oltp-vs-olap-and-lakehouses.md) — row vs columnar storage, warehouse vs lake vs lakehouse, Iceberg/Delta/Hudi, the analytics side of system design _(2026-04-25)_

---

## Tier 5 — Communication & Messaging

How services talk. Sync vs async, delivery guarantees, streaming.

32. [★ Sync vs Async Communication — REST, gRPC, Messaging](communication/sync-vs-async-communication.md) — latency coupling, failure coupling, when async wins, request-response over queues, saga handoff _(2026-04-25)_
33. [★ Real-Time Channels — Long Polling, WebSockets, SSE, Webhooks, WebRTC](communication/real-time-channels.md) — decision matrix for each, scaling characteristics, fallback ladders, when to choose which channel and why _(2026-04-25)_
34. [○ Push vs Pull Architecture](communication/push-vs-pull-architecture.md) — feed generation push vs pull, polling overhead and patterns, hybrid push-pull, fan-out implications _(2026-04-25)_
35. [★ Event-Driven Architecture — Pub/Sub, Choreography vs Orchestration](communication/event-driven-architecture.md) — event types, event vs command, choreography trade-offs, orchestration with workflow engines _(2026-04-25)_
36. [★ Idempotency and Exactly-Once Semantics](communication/idempotency-and-exactly-once.md) — idempotency keys, dedup windows, effectively-once via idempotent consumers, transactional outbox intro _(2026-04-25)_
37. [★ Dead-Letter Queues, Retries, and Poison Messages](communication/dead-letter-queues-and-retries.md) — retry topologies, DLQ patterns, poison message handling, retry budgets, observability on DLQs _(2026-04-25)_
38. [○ Stream Processing — Kafka Streams, Flink, Windowing](communication/stream-processing.md) — event-time vs processing-time, tumbling/sliding/session windows, exactly-once in streams, state stores _(2026-04-25)_
39. [○ IoT & Edge Computing Ingest](communication/iot-and-edge-ingest.md) — MQTT vs HTTP, massive connection multiplexing, AWS IoT Device Shadows _(2026-05-02)_

---

## Tier 6 — Reliability & Resilience

Designing for failure instead of against it.

40. [★ Failure Modes and Fault Tolerance Taxonomy](reliability/failure-modes-and-fault-tolerance.md) — crash-stop, omission, timing, byzantine, gray failures, blast radius, failure domains _(2026-04-25)_
41. [○ Failure Detection — Heartbeats, Phi-Accrual, and Gossip (SWIM)](reliability/failure-detection.md) — heartbeat protocols, phi-accrual detector, SWIM gossip-based failure detection, fencing tokens revisited _(2026-04-25)_
42. [○ Network Partitions and Split Brain](reliability/network-partitions-and-split-brain.md) — how partitions actually manifest (asymmetric, gray, flaky), detection strategies, the choices split-brain forces, real outage examples _(2026-04-25)_
43. [★ Single Point of Failure Analysis](reliability/single-point-of-failure-analysis.md) — finding SPOFs in a design, "what breaks if this dies" drills, HA patterns by failure class, SPOF audit checklist _(2026-04-25)_
44. [★ Retry Strategies — Exponential Backoff, Jitter, Budgets](reliability/retry-strategies.md) — why naive retries make outages worse, full jitter, retry budgets, circuit-breaker integration _(2026-04-25)_
45. [○ Chaos Engineering and Game Days](reliability/chaos-engineering-and-game-days.md) — chaos principles, blast-radius containment, game day structure, what to simulate first _(2026-04-25)_
46. [★ Disaster Recovery — RPO, RTO, Backup Strategies](reliability/disaster-recovery.md) — RPO vs RTO, backup tiers, pilot light vs warm standby vs active-active, DR drills _(2026-04-25)_
47. [★ Multi-Region Architectures — Active-Active, Active-Passive, Geo-Routing](reliability/multi-region-architectures.md) — regional failure domains, replication topologies, conflict resolution, geo-DNS vs anycast vs global LB _(2026-04-25)_
48. [★ Graceful Degradation and Feature Flags](reliability/graceful-degradation-and-feature-flags.md) — degradation tiers, kill switches, circuit breaker + flag integration, flag lifecycle _(2026-04-25)_

---

## Tier 7 — Performance & Observability

Making it fast and knowing when it isn't.

49. [★ Performance Budgets and Latency Analysis — Tail Latency and Coordinated Omission](performance-observability/performance-budgets-and-latency.md) — averages lie, p99/p99.9, coordinated omission, hedged requests, latency SLO sizing _(2026-04-26)_
50. [★ Distributed Tracing, Metrics, and Logs — The Three Pillars](performance-observability/tracing-metrics-logs.md) — OpenTelemetry model, trace propagation, cardinality traps, structured logging, correlation IDs _(2026-04-26)_
51. [○ Log Aggregation and Structured Logging](performance-observability/log-aggregation-and-structured-logging.md) — ELK/Loki/Splunk topologies, cardinality limits, log-to-metric conversion, correlation IDs across services _(2026-04-26)_
52. [★ Monitoring and Alerting — RED, USE, and the Four Golden Signals](performance-observability/monitoring-red-use-golden-signals.md) — RED for services, USE for resources, golden signals, SLO-based alerting, alert fatigue _(2026-04-26)_
53. [○ Dashboards, Runbooks, and On-Call](performance-observability/dashboards-runbooks-on-call.md) — dashboard design principles, runbook structure, auto-remediation, SLO burn-down dashboards, on-call ergonomics _(2026-04-26)_
54. [★ Capacity Planning and Load Testing](performance-observability/capacity-planning-and-load-testing.md) — headroom planning, load test shapes (ramp/soak/spike), k6/Gatling/Locust, production shadowing _(2026-04-26)_

---

## Tier 8 — Security in System Design

Design-level security decisions — not code-level hardening.

55. [★ Authentication — Sessions, Tokens, JWT, and Refresh Flows](security/authentication.md) — session vs token, JWT structure and pitfalls, refresh tokens, sliding sessions, multi-device sign-in _(2026-04-26)_
56. [★ Authorization — RBAC, ABAC, ReBAC, and Policy Engines](security/authorization.md) — role-based, attribute-based, relationship-based authorization, OPA/Cedar/Zanzibar-style systems, central vs distributed authz _(2026-04-26)_
57. [○ Single Sign-On — SAML, OIDC, and Federation](security/single-sign-on.md) — SAML vs OIDC vs OAuth2, federation patterns, IdP-initiated vs SP-initiated, SCIM for provisioning _(2026-04-26)_
58. [○ Secrets Management and Key Rotation](security/secrets-management-and-key-rotation.md) — secret storage (Vault, SM, KMS), rotation strategies, envelope encryption, workload identity _(2026-04-26)_
59. [★ Encryption at Rest and in Transit](security/encryption-at-rest-and-in-transit.md) — at-rest: disk vs DB vs app; in-transit: TLS, mTLS, E2EE; KMS hierarchies, BYOK _(2026-04-26)_
60. [★ Defense in Depth and Threat Modeling](security/defense-in-depth-and-threat-modeling.md) — layered defense, STRIDE, trust boundaries, attack surface, threat model in the design review _(2026-04-26)_
61. [○ Zero Trust Architecture — BeyondCorp, Identity-Aware Proxies](security/zero-trust-architecture.md) — identity-centric model, microsegmentation, IAP, policy engines, what zero trust does NOT mean _(2026-04-26)_

---

## Tier 9 — Architectural Styles

How to slice the system.

62. [★ Monolith, Modular Monolith, SOA, Microservices — When Each Wins](architectural-styles/monolith-to-microservices.md) — tech vs org trade-offs, modular monolith as the default, microservices prerequisites, "distributed monolith" anti-pattern _(2026-04-26)_
63. [○ Event-Driven Architecture as a Style](architectural-styles/event-driven-architecture-style.md) — EDA beyond a pattern, event backbone, topology choices, when EDA is wrong _(2026-04-26)_
64. [○ Serverless and FaaS Trade-offs](architectural-styles/serverless-and-faas.md) — cold starts, state, concurrency limits, vendor lock-in, cost curves, good vs bad fits _(2026-04-26)_
65. [★ Hexagonal, Clean, and DDD — Ports, Adapters, Bounded Contexts](architectural-styles/hexagonal-clean-ddd.md) — domain-first design, ports/adapters, aggregates, bounded contexts, context maps _(2026-04-26)_
66. [○ Service Discovery — Client-Side, Server-Side, DNS-Based](architectural-styles/service-discovery.md) — client-side (Ribbon/Eureka) vs server-side (LB-fronted) vs DNS-based, Consul/etcd, mesh-integrated discovery _(2026-04-26)_
67. [○ Sidecar Pattern — Beyond Service Mesh](architectural-styles/sidecar-pattern.md) — sidecars for logging, config, crypto, traffic; Istio/Envoy/Linkerd comparison; resource overhead, ambient mesh alternative _(2026-04-26)_
68. [★ Strangler Fig Pattern — Incremental Migration](architectural-styles/strangler-fig-pattern.md) — routing splits, dual-run, cutover mechanics, anti-corruption layer, real-world migration timelines _(2026-04-26)_
69. [○ Service Mesh as an Architectural Decision](architectural-styles/service-mesh-as-architectural-decision.md) — what a mesh buys you, sidecar overhead, ambient mesh, when mesh is overkill _(2026-04-26)_
70. [○ Peer-to-Peer Architecture — DHTs, Gossip, BitTorrent, IPFS](architectural-styles/peer-to-peer-architecture.md) — Kademlia and Chord DHTs, gossip overlays, content-addressed storage, when P2P is the right answer _(2026-04-26)_
71. [○ Cell-Based Architecture](architectural-styles/cell-based-architecture.md) — containing blast radius, pods/cells, stateless routing tiers _(2026-05-02)_
72. [○ Control Plane vs Data Plane](architectural-styles/control-plane-vs-data-plane.md) — configuration push vs traffic flow, gateway/mesh mental models _(2026-05-02)_

---

## Tier 10 — Case Studies: Designing Real Systems

The interview-style design problems. Each doc follows a consistent template: requirements → estimates → API → data model → high-level design → deep dives → bottlenecks → trade-offs. Categories and difficulty markers track the [AlgoMaster system-design problem set](https://algomaster.io/learn/system-design).

> Some problems also exist in the [Low-Level Design path](../low-level-design/INDEX.md) (URL Shortener, Rate Limiter, Search Autocomplete, Chat, Notifications, Online Auction, Movie Booking, Stock Exchange, etc.). The HLD entry below focuses on capacity, sharding, replication, infrastructure topology; the LLD twin focuses on classes, patterns, and concurrency primitives.

### 10.A Basic Designs

73. [★ Design a URL Shortener](case-studies/basic/design-url-shortener.md) — counter vs hash vs base62, redirect infra, click analytics, custom aliases, abuse protection _(2026-04-25, easy)_
    - 73.1 [★ Code Generation Strategies](case-studies/basic/url-shortener/code-generation-strategies.md) — base62, Snowflake, hash+truncate, random+collision, allocator architecture _(2026-04-27)_
    - 73.2 [○ Cache Strategy](case-studies/basic/url-shortener/cache-strategy.md) — multi-tier caching, hot keys, thundering herd, negative caching, edge-native pattern _(2026-04-27)_
    - 73.3 [○ Custom Aliases](case-studies/basic/url-shortener/custom-aliases.md) — namespace design, uniqueness, banned words, homoglyphs, squatting, i18n _(2026-04-27)_
    - 73.4 [○ Click Analytics Pipeline](case-studies/basic/url-shortener/click-analytics-pipeline.md) — Kafka, ClickHouse, HyperLogLog, S3 archive, late events, bot filtering _(2026-04-27)_
    - 73.5 [○ 301 vs 302 Redirects](case-studies/basic/url-shortener/redirect-301-vs-302.md) — RFC 9110/9111 semantics, browser/CDN behavior, analytics impact, SEO, retirement _(2026-04-27)_
74. [○ Design Pastebin](case-studies/basic/design-pastebin.md) — paste object model, expiry policies, key generation, read-vs-write skew, abuse / DMCA flow _(2026-04-25, easy)_
    - 74.1 [○ ID Generation](case-studies/basic/pastebin/id-generation.md) — random base62 vs UUID, content-addressed IDs, salting, visibility tiers, crawler resistance _(2026-04-27)_
    - 74.2 [○ Storage Choice](case-studies/basic/pastebin/storage-choice.md) — DB vs S3 split, hot/cold tiering, compression, encryption, lifecycle _(2026-04-27)_
    - 74.3 [○ Expiry Handling](case-studies/basic/pastebin/expiry-handling.md) — TTL semantics, sweepers, time-partitioned tables, burn-after-reading, GDPR _(2026-04-27)_
    - 74.4 [○ Abuse Defense](case-studies/basic/pastebin/abuse-defense.md) — content scanning, secret detection, DMCA, CSAM, ban lists, transparency _(2026-04-27)_
    - 74.5 [○ Caching and CDN](case-studies/basic/pastebin/caching-and-cdn.md) — immutable Cache-Control, private/password caching, edge compression, multi-region _(2026-04-27)_
    - 74.6 [○ Optional Features](case-studies/basic/pastebin/optional-features.md) — passwords, zero-knowledge encryption, syntax highlighting, oEmbed, webhooks _(2026-04-27)_
75. [★ Design a Rate Limiter (Service)](case-studies/basic/design-rate-limiter.md) — centralized vs distributed, Redis-backed sliding window, policy engine, client SDKs, observability _(2026-04-25, medium)_
    - 75.1 [○ Algorithm Choice](case-studies/basic/rate-limiter/algorithm-choice.md) — fixed/sliding/token/leaky/GCRA/concurrency/adaptive, decision matrix _(2026-04-27)_
    - 75.2 [○ Centralized vs Distributed](case-studies/basic/rate-limiter/centralized-vs-distributed.md) — single Redis, two-tier Envoy, gossip, CRDT, multi-region trade-offs _(2026-04-27)_
    - 75.3 [○ Race Conditions and Atomicity](case-studies/basic/rate-limiter/race-conditions-and-atomicity.md) — INCR+EXPIRE race, Lua atomicity, Redlock debate, DynamoDB, SQL alternatives _(2026-04-27)_
    - 75.4 [○ Hot Keys and Sharding](case-studies/basic/rate-limiter/hot-keys-and-sharding.md) — random/hash suffix sharding, local pre-aggregation, CMS detection, hash tags _(2026-04-27)_
    - 75.5 [○ Failure Modes](case-studies/basic/rate-limiter/failure-modes.md) — fail open/closed/local, watchdog, cascading failure, chaos engineering _(2026-04-27)_
    - 75.6 [○ Multi-Tier Limits](case-studies/basic/rate-limiter/multi-tier-limits.md) — nested vs orthogonal quotas, dry-run-then-commit, Stripe 4-tier, refunds _(2026-04-27)_
    - 75.7 [○ Distributed Synchronization](case-studies/basic/rate-limiter/distributed-synchronization.md) — gossip, periodic flush, CRDTs, drift bounds, clock sync _(2026-04-27)_

### 10.B Real-Time Communication

76. [★ Design WhatsApp / Chat System](case-studies/real-time/design-whatsapp.md) — 1:1 vs group, presence, delivery receipts, message ordering, media handling, offline delivery, e2e encryption _(2026-04-25, medium)_
    - 76.1 [★ Per-Conversation Ordering](case-studies/real-time/whatsapp/per-conversation-ordering.md) — server-assigned IDs, Lamport/HLC, reorder buffers, idempotent inserts _(2026-04-27)_
    - 76.2 [○ Group Chat Fanout](case-studies/real-time/whatsapp/group-chat-fanout.md) — per-user mailbox, group size tiers, Sender Keys, mention priority _(2026-04-27)_
    - 76.3 [○ Presence Service](case-studies/real-time/whatsapp/presence-service.md) — subscription model, TTL heartbeat, privacy controls, battery impact _(2026-04-27)_
    - 76.4 [○ Read Receipts and Typing](case-studies/real-time/whatsapp/read-receipts-and-typing.md) — sent/delivered/read tiers, group aggregation, privacy controls _(2026-04-27)_
    - 76.5 [○ Offline Delivery and Push](case-studies/real-time/whatsapp/offline-delivery-and-push.md) — APNs/FCM, mailbox retention, push-as-hint, VoIP push _(2026-04-27)_
    - 76.6 [○ Media Handling](case-studies/real-time/whatsapp/media-handling.md) — direct upload, E2E media encryption, TUS chunking, EXIF stripping _(2026-04-27)_
    - 76.7 [○ End-to-End Encryption](case-studies/real-time/whatsapp/end-to-end-encryption.md) — Signal Protocol, X3DH, Double Ratchet, Sender Keys, PQXDH _(2026-04-27)_
    - 76.8 [○ Multi-Device Sync](case-studies/real-time/whatsapp/multi-device-sync.md) — companion model, Sesame, key transparency, CRDT read state _(2026-04-27)_
    - 76.9 [○ Connection Scaling](case-studies/real-time/whatsapp/connection-scaling.md) — Erlang/BEAM, sticky LB, reconnect storm, TLS 1.3 0-RTT, QUIC _(2026-04-27)_
    - 76.1 [○ Presence Service](case-studies/real-time/whatsapp/presence-service.md) — online/offline state, heartbeats, fan-out of presence changes, last-seen semantics, privacy controls _(2026-04-27)_
    - 76.2 [★ Per-Conversation Ordering](case-studies/real-time/whatsapp/per-conversation-ordering.md) — sequence assignment, causal expectations, retries, duplicate suppression, ordering across reconnects _(2026-04-27)_
    - 76.3 [○ Offline Delivery and Push](case-studies/real-time/whatsapp/offline-delivery-and-push.md) — durable inbox, reconnect catch-up, APNs/FCM wake-up flow, battery constraints, device fan-in _(2026-04-27)_
    - 76.4 [○ Read Receipts and Typing Indicators](case-studies/real-time/whatsapp/read-receipts-and-typing.md) — ephemeral signals vs durable state, idempotent updates, privacy toggles, fan-out control _(2026-04-27)_
    - 76.5 [○ Group Chat Fanout](case-studies/real-time/whatsapp/group-chat-fanout.md) — sender-side vs server-side fanout, large-group amplification, membership snapshots, ack aggregation _(2026-04-27)_
77. [○ Design Slack](case-studies/real-time/design-slack.md) — workspace/channel hierarchy, threads, search, integrations, large-team fanout, presence at workspace scale _(2026-04-25, medium)_
78. [○ Design Live Comments](case-studies/real-time/design-live-comments.md) — fan-out at thousands of viewers per second, ordering, moderation hooks, push channel selection _(2026-04-25, medium)_
79. [★ Design Google Docs / Collaborative Editor](case-studies/real-time/design-google-docs.md) — OT vs CRDT, presence, cursor broadcast, offline merges, history, large-doc partitioning _(2026-04-25, hard)_
80. [○ Design Zoom](case-studies/real-time/design-zoom.md) — selective forwarding unit (SFU) vs MCU vs P2P mesh, signaling, simulcast, recording, geographic routing _(2026-04-25, hard)_
81. [○ Design Discord](case-studies/real-time/design-discord.md) — guild architecture, massive voice/text concurrency, Erlang presence, fan-out at scale _(planned, hard)_
82. [○ Design Figma](case-studies/real-time/design-figma.md) — WebGL/Canvas state sync, multiplayer CRDTs for graphical objects, memory constraints _(planned, hard)_

### 10.C Social Media

83. [○ Design Instagram](case-studies/social-media/design-instagram.md) — photo upload pipeline, thumbnails, feed, stories, follow graph, hashtag/explore feed _(2026-04-25, medium)_
    - 83.1 [○ Photo Upload Pipeline](case-studies/social-media/instagram/photo-upload-pipeline.md) — presigned PUT, derivatives, perceptual dedup, format choice _(2026-04-29)_
    - 83.2 [○ Feed Generation](case-studies/social-media/instagram/feed-generation.md) — pull/push/hybrid, celebrity threshold, ranking layer _(2026-04-29)_
    - 83.3 [○ Stories](case-studies/social-media/instagram/stories.md) — 24h TTL, ephemeral reads, viewer tracking, Highlights archival _(2026-04-29)_
    - 83.4 [○ Hashtag and Location Search](case-studies/social-media/instagram/hashtag-and-location-search.md) — inverted index, hot tags, geohash/H3, trending CMS _(2026-04-29)_
    - 83.5 [○ Explore](case-studies/social-media/instagram/explore.md) — two-tower retrieval, account suggestions, cold start, safety filters _(2026-04-29)_
    - 83.6 [○ Like and Comment Counters](case-studies/social-media/instagram/like-and-comment-counters.md) — sharded counters, write buffer, HLL, recent likers _(2026-04-29)_
    - 83.7 [○ Direct Messages](case-studies/social-media/instagram/direct-messages.md) — cross-app messaging, E2E rollout, vanish mode, request inbox _(2026-04-29)_
    - 83.8 [○ CDN Strategy](case-studies/social-media/instagram/cdn-strategy.md) — immutable derivatives, multi-CDN, origin shield, format negotiation _(2026-04-29)_
84. [★ Design Facebook News Feed](case-studies/social-media/design-facebook-news-feed.md) — fan-out on write vs read, celebrity problem, timeline generation, ranking, freshness vs personalization _(2026-04-25, medium)_
85. [○ Design TikTok](case-studies/social-media/design-tiktok.md) — short-video pipeline, ranking with implicit feedback, content moderation at scale, recommendation as the product _(2026-04-25, medium)_
86. [○ Design Reddit](case-studies/social-media/design-reddit.md) — subreddits, threaded comments, voting + ranking (hot/new/top/controversial), moderation tooling _(2026-04-25, medium)_
87. [○ Design Tinder](case-studies/social-media/design-tinder.md) — geo + preference matching, swipe storage, deck pre-loading, mutual-match flow, anti-bot defenses _(2026-04-25, medium)_
88. [○ Design a Likes Counting System](case-studies/social-media/design-likes-counting-system.md) — counter sharding, eventual aggregation, Redis HLL/Count-Min, debouncing, accuracy vs cost _(2026-04-25, medium)_

### 10.D Media Streaming & Delivery

89. [○ Design Spotify](case-studies/media-streaming/design-spotify.md) — track CDN distribution, playlist + library service, recommendation intro, offline mode, royalty-counting precision _(2026-04-25, medium)_
90. [★ Design YouTube](case-studies/media-streaming/design-youtube.md) — upload pipeline, transcoding fan-out, ABR, CDN strategy, comments + recommendations, creator analytics _(2026-04-25, medium)_
91. [★ Design Netflix](case-studies/media-streaming/design-netflix.md) — content prep pipeline, Open Connect CDN, viewing-session tracking, regional licensing constraints, recommendation system _(2026-04-25, medium)_
    - 91.1 [○ Open Connect CDN](case-studies/media-streaming/netflix/open-connect.md) — purpose-built ISP-deployed CDN, OCAs, pre-fill, peering economics _(2026-04-29)_
    - 91.2 [○ Encoding Pipeline](case-studies/media-streaming/netflix/encoding-pipeline.md) — bitrate ladder, per-title/per-shot, AV1, HDR/Dolby Vision _(2026-04-29)_
    - 91.3 [○ Viewing Session and ABR](case-studies/media-streaming/netflix/viewing-session-and-abr.md) — HLS/DASH, BOLA/MPC, stall avoidance, resume tokens _(2026-04-29)_
    - 91.4 [○ Recommendation System](case-studies/media-streaming/netflix/recommendation-system.md) — recall+ranking, personalized rows/artwork, contextual bandits _(2026-04-29)_
    - 91.5 [○ Microservices on AWS](case-studies/media-streaming/netflix/microservices-on-aws.md) — 8-year migration, Hystrix/Eureka/Zuul, Spinnaker, deploy cadence _(2026-04-29)_
    - 91.6 [○ Chaos Engineering](case-studies/media-streaming/netflix/chaos-engineering.md) — Chaos Monkey, Simian Army, FIT, ChAP, steady-state hypothesis _(2026-04-29)_
    - 91.7 [○ Multi-Region Active-Active](case-studies/media-streaming/netflix/multi-region-active-active.md) — 3 AWS regions, Cassandra, Chaos Kong, regional evacuation _(2026-04-29)_
    - 91.8 [○ DRM](case-studies/media-streaming/netflix/drm.md) — CENC, Widevine/FairPlay/PlayReady, EME, key ladder, per-tier security _(2026-04-29)_
92. [★ Design Google Drive / Dropbox](case-studies/media-streaming/design-google-drive.md) — chunking, dedup, delta sync, metadata DB, block storage, sharing/permission, mobile constraints _(2026-04-25, medium)_
93. [○ Design Gmail](case-studies/media-streaming/design-gmail.md) — IMAP/SMTP boundary, search index over mail, label/folder model, anti-spam pipeline, threading, attachment handling _(2026-04-25, hard)_
94. [○ Design Twitch](case-studies/media-streaming/design-twitch.md) — live ingest, low-latency delivery (HLS-LL/CMAF), chat at viewer scale, VOD pipeline, subscription/monetization _(2026-04-25, hard)_

### 10.E Location-Based Services

95. [○ Design Airbnb](case-studies/location-based/design-airbnb.md) — listing search with geospatial filter, availability + booking concurrency, pricing/promo, reviews, payments _(2026-04-25, medium)_
96. [○ Design DoorDash / Swiggy](case-studies/location-based/design-doordash.md) — restaurant index, dispatch + driver matching, ETAs, multi-leg pickup, surge pricing _(2026-04-25, medium)_
97. [★ Design Uber / Ride-Hailing](case-studies/location-based/design-uber.md) — geo indexing (geohash/S2/H3), dispatch matching, trip lifecycle state machine, surge pricing, dual-write concerns _(2026-04-25, hard)_
    - 97.1 [★ H3 Geo-Indexing](case-studies/location-based/uber/h3-geo-indexing.md) — hexagonal grid, 16 resolutions, k-ring queries, pentagon distortion _(2026-04-29)_
    - 97.2 [○ Driver Location Ingestion](case-studies/location-based/uber/driver-location-ingestion.md) — wire protocol, regional sharding, Kalman filtering, time-series compression _(2026-04-29)_
    - 97.3 [○ Matching and Dispatch](case-studies/location-based/uber/matching-and-dispatch.md) — DISCO geosearch, batched assignment, Hungarian, fairness _(2026-04-29)_
    - 97.4 [○ Trip State Machine](case-studies/location-based/uber/trip-state-machine.md) — lifecycle states, idempotent transitions, saga compensation _(2026-04-29)_
    - 97.5 [○ Surge Pricing](case-studies/location-based/uber/surge-pricing.md) — demand-supply ratio, smoothing, honored surge, regulatory caps _(2026-04-29)_
    - 97.6 [○ ETA Prediction](case-studies/location-based/uber/eta-prediction.md) — routing engine + ML correction, DeepETA, confidence intervals _(2026-04-29)_
    - 97.7 [○ Real-Time Tracking](case-studies/location-based/uber/real-time-tracking.md) — WebSocket protocol, snap-to-road, smoothing, geofences _(2026-04-29)_
    - 97.8 [○ Payment and Receipts](case-studies/location-based/uber/payment-and-receipts.md) — pre-auth, settlement, double-entry ledger, multi-currency _(2026-04-29)_
98. [○ Design Google Maps](case-studies/location-based/design-google-maps.md) — tile pyramid, road graph, routing (A*/contraction hierarchies), real-time traffic, address geocoding _(2026-04-25, hard)_

### 10.F Search & Aggregation

99. [★ Design Search Autocomplete](case-studies/search-aggregation/design-search-autocomplete.md) — trie vs prefix index, ranking, typo tolerance, freshness, personalization _(2026-04-25, easy)_
100. [○ Design a News Aggregator](case-studies/search-aggregation/design-news-aggregator.md) — crawl-vs-RSS ingest, deduplication (near-duplicate detection), ranking, topic clustering, personalization _(2026-04-25, medium)_
101. [○ Design a Web Crawler](case-studies/search-aggregation/design-web-crawler.md) — frontier scheduling, politeness/robots, dedup, content-addressed storage, recrawl strategy, distributed coordination _(2026-04-25, medium)_
102. [○ Design Google Search](case-studies/search-aggregation/design-google-search.md) — index sharding, query serving, ranking signals, freshness vs depth, query understanding _(2026-04-25, hard)_
    - 102.1 [○ Web Crawling at Scale](case-studies/search-aggregation/google-search/web-crawling.md) — Googlebot, URL frontier, politeness, dedup (SimHash), JS rendering _(2026-04-30)_
    - 102.2 [○ Inverted Index Sharding](case-studies/search-aggregation/google-search/inverted-index-sharding.md) — doc-partitioned, scatter-gather, compression, tiered indexes _(2026-04-30)_
    - 102.3 [○ Query Serving and Latency](case-studies/search-aggregation/google-search/query-serving-latency.md) — Tail at Scale, hedged requests, cache layers, per-stage math _(2026-04-30)_
    - 102.4 [○ Ranking Signals](case-studies/search-aggregation/google-search/ranking-signals.md) — PageRank, BM25, BERT/MUM, RankBrain, learn-to-rank _(2026-04-30)_
    - 102.5 [○ Freshness vs Depth](case-studies/search-aggregation/google-search/freshness-vs-depth.md) — Caffeine, instant indexing, sitemap hints, QDF _(2026-04-30)_
    - 102.6 [○ Autocomplete and Query Rewriting](case-studies/search-aggregation/google-search/autocomplete-and-rewriting.md) — trie, spell correction, query expansion _(2026-04-30)_
    - 102.7 [○ Federated SERP Composition](case-studies/search-aggregation/google-search/federated-serp.md) — verticals, OneBox, Knowledge Panel, AI Overviews _(2026-04-30)_
103. [○ Design an Ad Click Aggregator](case-studies/search-aggregation/design-ad-click-aggregator.md) — click capture at scale, fraud filtering, attribution windows, real-time aggregates, billing correctness _(2026-04-25, hard)_

### 10.G E-Commerce & Marketplace

104. [○ Design Amazon E-Commerce](case-studies/e-commerce/design-amazon-ecommerce.md) — catalog, search, cart, order, payment, fulfillment graph, recommendations — the HLD survey _(2026-04-25, medium)_
    - 104.1 [○ Catalog at Planet Scale](case-studies/e-commerce/amazon-ecommerce/catalog-service.md) — ASIN, marketplace vs direct, variations, regional differences, taxonomy, A+ content, SKU dedup _(2026-04-30)_
    - 104.2 [○ Search and Browse](case-studies/e-commerce/amazon-ecommerce/search-and-browse.md) — A9, attribute index, facets, ranking layer, semantic search, sponsored ads, OOS handling _(2026-04-30)_
    - 104.3 [○ Inventory Consistency](case-studies/e-commerce/amazon-ecommerce/inventory-consistency.md) — per-FC counts, reservation flow, over-sell protection, FBA vs MFN, reconciliation, flash sale concurrency _(2026-04-30)_
    - 104.4 [○ Cart Service](case-studies/e-commerce/amazon-ecommerce/cart-service.md) — Dynamo origin, conflict resolution, cross-device merge, guest cart, abandonment, share cart _(2026-04-30)_
    - 104.5 [○ Checkout Saga](case-studies/e-commerce/amazon-ecommerce/checkout-saga.md) — orchestration vs choreography, auth vs capture, order ID, idempotency, retry semantics, failure recovery _(2026-04-30)_
    - 104.6 [○ Payment Integration](case-studies/e-commerce/amazon-ecommerce/payment-integration.md) — multi-processor abstraction, pre-auth, capture at ship-time, 3DS2/SCA, PCI scope, fraud, regional methods _(2026-04-30)_
    - 104.7 [○ Fulfillment Graph](case-studies/e-commerce/amazon-ecommerce/fulfillment-graph.md) — FC topology, sourcing routing, SLA tiers, carrier assignment, zone skip, returns, lockers _(2026-04-30)_
    - 104.8 [○ Recommendation Pipeline](case-studies/e-commerce/amazon-ecommerce/recommendation-pipeline.md) — item-to-item CF, FBT, two-tower DL, multi-objective, vs Netflix, cold start _(2026-04-30)_
105. [○ Design Shopify](case-studies/e-commerce/design-shopify.md) — multi-tenant store platform, plugin/app marketplace, checkout extensibility, multi-region payments _(2026-04-25, medium)_
106. [○ Design a Flash Sale System](case-studies/e-commerce/design-flash-sale.md) — extreme write contention, queue-based admission, inventory reservation, anti-bot, graceful degradation _(2026-04-25, hard)_
107. [○ Design an Online Auction System (HLD)](case-studies/e-commerce/design-online-auction.md) — bid storage, sniping protection, real-time price broadcast, payments + escrow, anti-fraud _(2026-04-25, hard)_
108. [○ Design a Movie Booking System (HLD)](case-studies/e-commerce/design-movie-booking-system.md) — show / screen / seat, seat-hold concurrency, payment integration, refund flow, multi-cinema sharding _(2026-04-25, hard)_

### 10.H Payment & Financial Systems

109. [★ Design a Payment System](case-studies/payment/design-payment-system.md) — idempotency, double-entry ledgers, reconciliation, fraud gates, async settlement, processor adapters _(2026-04-25, medium)_
110. [○ Design a Digital Wallet](case-studies/payment/design-digital-wallet.md) — balance ledger, top-up flow, transfers, KYC, transaction limits, regulatory + audit constraints _(2026-04-25, hard)_
111. [○ Design an Online Stock Exchange](case-studies/payment/design-stock-exchange.md) — order book, matching engine, market vs limit, T+N settlement, market data fan-out, regulatory feeds _(2026-04-25, hard)_
    - 111.1 [○ Order Book Data Structure](case-studies/payment/stock-exchange/order-book-data-structure.md) — price levels, FIFO at price, cache locality, intrusive lists _(2026-04-30)_
    - 111.2 [○ Matching Engine Determinism](case-studies/payment/stock-exchange/matching-engine-determinism.md) — single-threaded design, mechanical sympathy, bit-identical replay _(2026-04-30)_
    - 111.3 [○ Sequencer Pattern](case-studies/payment/stock-exchange/sequencer-pattern.md) — total ordering, gap detection, sequencer failover _(2026-04-30)_
    - 111.4 [○ Market Data Fan-Out](case-studies/payment/stock-exchange/market-data-fan-out.md) — UDP multicast, A/B feeds, snapshot recovery, ITCH/MDP _(2026-04-30)_
    - 111.5 [○ Pre-Trade Risk and Circuit Breakers](case-studies/payment/stock-exchange/pre-trade-risk-and-circuit-breakers.md) — fat-finger, buying power, kill switch, market-wide halts _(2026-04-30)_
    - 111.6 [○ T+1 Settlement Pipeline](case-studies/payment/stock-exchange/settlement-pipeline.md) — CCP, multilateral netting, DvP, default management _(2026-04-30)_
    - 111.7 [○ Audit, Replay, and Regulatory Feeds](case-studies/payment/stock-exchange/audit-replay-regulatory-feeds.md) — append-only journals, CAT, MiFID II, PTP clocks _(2026-04-30)_
    - 111.8 [○ Order Type Semantics](case-studies/payment/stock-exchange/order-type-semantics.md) — market/limit/stop/IOC/FOK, iceberg, TIF, auctions _(2026-04-30)_

### 10.I Distributed Infrastructure

112. [○ Design a Load Balancer (Service)](case-studies/distributed-infra/design-load-balancer.md) — L4/L7 pipeline, health probing, connection draining, hash-based affinity, autoscaling targets _(2026-04-25, medium)_
113. [○ Design an API Gateway (Service)](case-studies/distributed-infra/design-api-gateway.md) — route table, auth offload, rate limiting, request transformation, plugin model, multi-tenant isolation _(2026-04-25, medium)_
114. [★ Design a Notification System](case-studies/distributed-infra/design-notification-system.md) — fan-out, channel routing (push/email/SMS), preferences, rate limiting, DLQ handling _(2026-04-25, medium)_
115. [★ Design a Key-Value Store](case-studies/distributed-infra/design-key-value-store.md) — Dynamo-style design, NWR, consistent hashing, replication, anti-entropy, conflict resolution _(2026-04-25, hard)_
116. [★ Design a Distributed Cache](case-studies/distributed-infra/design-distributed-cache.md) — partitioning, replication, eviction, consistent hashing ring, client topology awareness _(2026-04-25, hard)_
    - 116.1 [★ Partitioning and Hash Slots](case-studies/distributed-infra/distributed-cache/partitioning-and-hash-slots.md) — consistent hashing, virtual nodes, jump hash, Maglev, slot migration _(2026-05-01)_
    - 116.2 [○ Replication per Partition](case-studies/distributed-infra/distributed-cache/replication-per-partition.md) — primary-replica, async streams, Sentinel, multi-AZ/region trade-offs _(2026-05-01)_
    - 116.3 [★ Eviction Policies](case-studies/distributed-infra/distributed-cache/eviction-policies.md) — LRU, sampling-LRU, LFU, TinyLFU, TTL active vs lazy expiry _(2026-05-01)_
    - 116.4 [○ Topology Awareness](case-studies/distributed-infra/distributed-cache/topology-awareness.md) — smart clients vs proxies, CLUSTER SLOTS, slot map drift _(2026-05-01)_
    - 116.5 [★ Cache Stampede Protection](case-studies/distributed-infra/distributed-cache/cache-stampede-protection.md) — singleflight, distributed locks, probabilistic early expiration _(2026-05-01)_
    - 116.6 [★ Hot Key Handling](case-studies/distributed-infra/distributed-cache/hot-key-handling.md) — fan-out replicas, local L1 tier, sharded counters, hot pool promotion _(2026-05-01)_
    - 116.7 [★ Write Strategies](case-studies/distributed-infra/distributed-cache/write-strategies.md) — cache-aside, write-through, write-behind, invalidation patterns _(2026-05-01)_
    - 116.8 [○ Multi-Tier Caching](case-studies/distributed-infra/distributed-cache/multi-tier-caching.md) — L1 in-process + L2 distributed, coherence via invalidation broadcast _(2026-05-01)_
117. [○ Design a CDN](case-studies/distributed-infra/design-cdn.md) — edge POPs, anycast, cache hierarchies, purge propagation, origin shielding, edge compute _(2026-04-25, hard)_
118. [○ Design Object Storage (S3-Style)](case-studies/distributed-infra/design-object-storage.md) — flat namespace, durability via erasure coding, multipart upload, lifecycle, presigned URLs, eventual → strong consistency story _(2026-04-25, hard)_
119. [○ Design a Message Queue (Service)](case-studies/distributed-infra/design-message-queue.md) — log vs queue semantics, partitioning, consumer groups, durability, ordering guarantees _(2026-04-25, hard)_
120. [○ Design a Time-Series Database](case-studies/distributed-infra/design-time-series-database.md) — write-optimized columnar storage, downsampling, retention, range query optimization, cardinality budget _(2026-04-25, hard)_
121. [○ Design a Distributed Locking Service](case-studies/distributed-infra/design-distributed-locking.md) — lease-based locks, fencing tokens, ZooKeeper/etcd patterns, deadlock prevention, watchdog renewal _(2026-04-25, hard)_

### 10.J Counting & Ranking Systems

122. [○ Design a Real-Time Leaderboard](case-studies/counting-ranking/design-realtime-leaderboard.md) — sorted set semantics, sharded counters, periodic snapshots, top-K queries, percentile rankings _(2026-04-25, medium)_
    - 122.1 [○ Redis Sorted Set Internals](case-studies/counting-ranking/realtime-leaderboard/redis-sorted-set-internals.md) — skip lists, span counters, ZADD/ZRANGE/ZREVRANK semantics _(2026-05-01)_
    - 122.2 [○ Sharded Score Aggregation](case-studies/counting-ranking/realtime-leaderboard/sharded-score-aggregation.md) — hot-write sharding, batching, stream aggregator _(2026-05-01)_
    - 122.3 [○ Snapshots to Durable Storage](case-studies/counting-ranking/realtime-leaderboard/snapshots-to-durable-storage.md) — event-log replay, RDB/AOF, hybrid recovery _(2026-05-01)_
    - 122.4 [○ Top-K Queries](case-studies/counting-ranking/realtime-leaderboard/top-k-queries.md) — global/friends/region, pre-materialized vs live merge, streaming top-K _(2026-05-01)_
    - 122.5 [○ Percentile Rankings via t-digest](case-studies/counting-ranking/realtime-leaderboard/percentile-rankings-t-digest.md) — t-digest, HDR, KLL, mergeable sketches _(2026-05-01)_
    - 122.6 [○ Rolling Windows](case-studies/counting-ranking/realtime-leaderboard/rolling-windows.md) — tumbling/sliding, bucketed mini-ZSETs, watermarks _(2026-05-01)_
    - 122.7 [○ Tie-Breaking](case-studies/counting-ranking/realtime-leaderboard/tie-breaking.md) — composite double encoding, IEEE 754 budget _(2026-05-01)_
    - 122.8 [○ Tournament Mode](case-studies/counting-ranking/realtime-leaderboard/tournament-mode.md) — lifecycle, brackets, idempotent prize-grant saga _(2026-05-01)_
123. [○ Design a Top-K System](case-studies/counting-ranking/design-top-k-system.md) — count-min sketch + heap, sliding-window top-K, multi-region merge, accuracy vs memory _(2026-04-25, hard)_

### 10.K Asynchronous Systems

124. [○ Design a Job Scheduler](case-studies/async/design-job-scheduler.md) — cron + one-shot triggers, distributed lock per job, retry + backoff, persistent queue, multi-tenant fairness _(2026-04-25, medium)_
    - 124.1 [○ Distributed Lock per Job](case-studies/async/job-scheduler/distributed-lock-per-job.md) — lease + TTL, fencing tokens, ZK/etcd/Postgres/Redis backends _(2026-05-01)_
    - 124.2 [○ Retry and Idempotency](case-studies/async/job-scheduler/retry-and-idempotency.md) — backoff/jitter, retry budgets, inbox pattern, DLQ _(2026-05-01)_
    - 124.3 [○ Multi-Tenant Fairness](case-studies/async/job-scheduler/multi-tenant-fairness.md) — WFQ, DRR, stride/lottery, hierarchical fairness _(2026-05-01)_
    - 124.4 [○ Time-Zone Correctness](case-studies/async/job-scheduler/time-zone-correctness.md) — IANA tzdata, DST gap/overlap policies, RRULE _(2026-05-01)_
    - 124.5 [○ Missed-Fire Policies](case-studies/async/job-scheduler/missed-fire-policies.md) — catch-up vs skip vs coalesce, watermark detection _(2026-05-01)_
    - 124.6 [○ Dependency DAG](case-studies/async/job-scheduler/dependency-dag.md) — topological scheduling, trigger rules, fan-out/fan-in _(2026-05-01)_
    - 124.7 [○ Leader Election for Scheduler](case-studies/async/job-scheduler/leader-election-for-scheduler.md) — single-decider HA, lease TTL, sharded schedulers _(2026-05-01)_
125. [○ Design a CI/CD Pipeline](case-studies/async/design-cicd-pipeline.md) — pipeline DAG, runner pool, artifact store, secrets, environment promotion, failure isolation _(2026-04-25, medium)_
126. [○ Design a Monitoring & Alerting System (Datadog-Style)](case-studies/async/design-monitoring-alerting.md) — ingestion, time-series storage, rollups, alerting engine, dashboard compute, alert fatigue mitigation _(2026-04-25, medium)_
127. [○ Design a Distributed Cron](case-studies/async/design-distributed-cron.md) — time-wheel algorithm, distributed execution, misfire handling _(planned, hard)_

### 10.L Specialized Systems

128. [○ Design LeetCode](case-studies/specialized/design-leetcode.md) — problem catalog, code execution sandbox, judging pipeline, contest mode, plagiarism detection _(2026-04-25, medium)_
129. [○ Design a Calendar System](case-studies/specialized/design-calendar-system.md) — events, recurrence (RRULE), free-busy, scheduling assistant, time-zone correctness, conflict detection _(2026-04-25, hard)_
    - 129.1 [○ RRULE Expansion](case-studies/specialized/calendar-system/rrule-expansion.md) — materialize vs compute on demand, EXDATE, RECURRENCE-ID overrides _(2026-05-01)_
    - 129.2 [○ Calendar Time-Zone Correctness](case-studies/specialized/calendar-system/time-zone-correctness.md) — IANA tzdata, DST gap/overlap, per-attendee display _(2026-05-01)_
    - 129.3 [○ Free-Busy Queries](case-studies/specialized/calendar-system/free-busy-queries.md) — interval merge, GiST range index, privacy projection _(2026-05-01)_
    - 129.4 [○ Scheduling Assistant](case-studies/specialized/calendar-system/scheduling-assistant.md) — multi-user intersection, working-hours mask, top-K slot ranking _(2026-05-01)_
    - 129.5 [○ Conflict Detection](case-studies/specialized/calendar-system/conflict-detection.md) — overlap predicate, interval trees, Postgres EXCLUDE constraints _(2026-05-01)_
    - 129.6 [○ Reminders and Notifications](case-studies/specialized/calendar-system/reminders-and-notifications.md) — delayed queues, DST re-anchoring, idempotent delivery _(2026-05-01)_
    - 129.7 [○ Sharing, Permissions, and Delegation](case-studies/specialized/calendar-system/sharing-permissions-delegation.md) — ACL tiers, per-event visibility, delegation, audit log _(2026-05-01)_
    - 129.8 [○ ICS and CalDAV Interop](case-studies/specialized/calendar-system/ics-and-caldav-interop.md) — iCalendar wire format, CalDAV REPORT, iTIP/iMIP, JMAP _(2026-05-01)_
130. [○ Design Online Chess](case-studies/specialized/design-online-chess.md) — matchmaking, anti-cheat, real-time move sync, ELO updates, time control state machine, spectator mode _(2026-04-25, hard)_
131. [○ Design GitHub](case-studies/specialized/design-github.md) — git operations at scale, delta compression, packfiles, Git LFS, code search _(2026-04-27, hard)_

---

## Tier 11 — The Interview Framework

How to actually run a system design discussion — useful even if you never interview again, because it mirrors how real architecture reviews should flow.

132. [★ The 6-Step Framework — Requirements → Estimates → API → Data Model → HLD → Deep Dives](interview-framework/six-step-framework.md) — the canonical structure, time allocation, where candidates stumble _(2026-04-26)_
133. [★ Trade-off Articulation and Bottleneck Analysis](interview-framework/tradeoff-articulation-and-bottlenecks.md) — naming trade-offs explicitly, identifying bottlenecks, the "what breaks at 10x?" drill _(2026-04-26)_
134. [★ Common Anti-Patterns in System Design Interviews](interview-framework/common-anti-patterns.md) — premature detail, resume-driven design, ignoring NFRs, hand-waving at scale _(2026-05-02)_

---

## Tier 12 — Data Structures for Scale

The algorithms and data structures that unlock system-design answers — what goes into the design diagram when "regular data structures won't scale".

135. [★ Bloom Filters & Cuckoo Filters — Probabilistic Membership Tests](data-structures/bloom-and-cuckoo-filters.md) — false-positive math, CDN caching, LSM read path, duplicate detection, when cuckoo beats bloom _(2026-04-26)_
136. [○ Merkle Trees — Efficient Diff at Scale](data-structures/merkle-trees.md) — anti-entropy in Dynamo/Cassandra, blockchain roots, rsync, S3 inventory comparisons, version trees _(2026-04-26)_
137. [○ HyperLogLog — Cardinality Estimation in Constant Memory](data-structures/hyperloglog.md) — accuracy vs memory math, Redis HLL, BigQuery APPROX_COUNT_DISTINCT, merging HLL sketches across shards _(2026-04-26)_
138. [○ Count-Min Sketch & Top-K — Frequency Estimation](data-structures/count-min-sketch-and-top-k.md) — heavy hitters, rate abuse detection, streaming aggregations, error bounds vs space _(2026-04-26)_
139. [○ Skip Lists — Probabilistic Balanced Order](data-structures/skip-lists.md) — Redis sorted sets internals, LSM memtables, lock-free variants, vs balanced trees in practice _(2026-04-26)_
140. [★ Consistent Hashing & Rendezvous Hashing — Beyond the Basics](data-structures/consistent-and-rendezvous-hashing.md) — virtual nodes math, jump hash, Maglev, rendezvous hashing comparison, hot-spot mitigation _(2026-04-26)_
141. [★ Geohash — Encoding 2D Coordinates as Strings](data-structures/geohash.md) — bit-interleaving, prefix-based proximity, neighbor calculation, geohash limits, S2 and H3 alternatives _(2026-04-26)_
142. [○ Quadtrees & R-Trees — Multi-Dimensional Indexing](data-structures/quadtrees-and-r-trees.md) — point quadtrees, region quadtrees, R-tree splits, ride-sharing dispatch, map rendering, k-NN queries _(2026-04-26)_

---

## Tier 13 — Batch & Stream Processing

Data engineering in system design. The analytics and pipeline side that most backend engineers eventually have to reason about.

143. [★ Batch vs Stream Processing](batch-and-stream/batch-vs-stream-processing.md) — when each wins, latency vs completeness trade-off, exactly-once in each model, hybrid patterns _(2026-04-26)_
144. [○ MapReduce and Its Descendants](batch-and-stream/mapreduce-and-descendants.md) — the model not just Hadoop, map/shuffle/reduce, Spark/Flink as evolutions, why MapReduce-as-API faded _(2026-04-26)_
145. [★ ETL, ELT, and Data Pipeline Architecture](batch-and-stream/etl-elt-and-pipelines.md) — Airflow, Dagster, Prefect, declarative vs imperative DAGs, data contracts, idempotent pipeline design _(2026-04-26)_
146. [○ Lambda vs Kappa Architecture](batch-and-stream/lambda-vs-kappa-architecture.md) — dual-pipeline vs stream-only, reprocessing strategies, when each still applies, modern alternatives _(2026-04-26)_
147. [○ Modern Streaming Engines — Flink, Spark Structured Streaming, Kafka Streams, Beam](batch-and-stream/modern-streaming-engines.md) — comparing engines, exactly-once across them, state backends, choosing for your workload _(2026-04-26)_

---

## Tier 14 — AI & Machine Learning Systems

The systems and infrastructure that power modern GenAI and ML applications.

148. [★ Vector Databases and Semantic Search](ai-ml/vector-databases-and-semantic-search.md) — HNSW index, IVF-PQ, quantization, pgvector vs Pinecone/Milvus, hybrid search _(2026-05-02)_
149. [★ RAG (Retrieval-Augmented Generation) Architecture](ai-ml/rag-architecture.md) — ingestion pipeline, chunking strategies, embedding generation, prompt construction _(2026-05-02)_
150. [○ LLM Inference Serving at Scale](ai-ml/llm-inference-serving.md) — vLLM, continuous batching, KV cache, token-based rate limiting, model parallelism _(2026-05-02)_
151. [○ Recommendation Systems Architecture](ai-ml/recommendation-systems.md) — candidate generation, scoring, re-ranking, two-tower models, feature stores _(2026-05-02)_

---

## Quick Reference by Topic

### Foundations

- [Back-of-Envelope Estimation](foundations/back-of-envelope-estimation.md)
- [CAP, PACELC, and Consistency Models](foundations/cap-and-consistency-models.md)
- [SLA, SLO, SLI, and Availability](foundations/sla-slo-sli-and-availability.md)
- [Non-Functional Requirements Checklist](foundations/non-functional-requirements.md)
- [Core Trade-offs Catalog](foundations/core-tradeoffs-catalog.md)

### Building Blocks

- [Load Balancers](building-blocks/load-balancers.md)
- [Caching Layers](building-blocks/caching-layers.md)
- [Databases as a Component](building-blocks/databases-as-a-component.md)
- [Message Queues & Brokers](building-blocks/message-queues-and-brokers.md)
- [Object and Blob Storage](building-blocks/object-and-blob-storage.md)
- [Search Systems](building-blocks/search-systems.md)
- [API Gateways and BFFs](building-blocks/api-gateways-and-bff.md)
- [Rate Limiters](building-blocks/rate-limiters.md)
- [API Design Styles](building-blocks/api-design-styles.md)
- [REST API Design in Depth](building-blocks/rest-api-design-in-depth.md)
- [Distributed File Systems & Erasure Coding](building-blocks/distributed-file-systems-and-erasure-coding.md)

### Scalability Patterns

- [Horizontal vs Vertical Scaling](scalability/horizontal-vs-vertical-and-stateless.md)
- [Sharding Strategies](scalability/sharding-strategies.md)
- [Replication Patterns](scalability/replication-patterns.md)
- [Read/Write Splitting & Cache Strategies](scalability/read-write-splitting-and-cache-strategies.md)
- [CQRS and Event Sourcing](scalability/cqrs-and-event-sourcing.md)
- [Backpressure, Bulkhead, Circuit Breaker](scalability/backpressure-bulkhead-circuit-breaker.md)
- [Read-Path Optimizations](scalability/read-path-optimizations.md)

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

### Reliability & Resilience

- [Failure Modes and Fault Tolerance](reliability/failure-modes-and-fault-tolerance.md)
- [Failure Detection (Heartbeats, Phi-Accrual, SWIM)](reliability/failure-detection.md)
- [Network Partitions and Split Brain](reliability/network-partitions-and-split-brain.md)
- [Single Point of Failure Analysis](reliability/single-point-of-failure-analysis.md)
- [Retry Strategies](reliability/retry-strategies.md)
- [Chaos Engineering and Game Days](reliability/chaos-engineering-and-game-days.md)
- [Disaster Recovery](reliability/disaster-recovery.md)
- [Multi-Region Architectures](reliability/multi-region-architectures.md)
- [Graceful Degradation and Feature Flags](reliability/graceful-degradation-and-feature-flags.md)

### Performance & Observability

- Performance Budgets and Latency
- Distributed Tracing, Metrics, Logs
- Log Aggregation and Structured Logging
- Monitoring (RED/USE/Golden Signals)
- Dashboards, Runbooks, On-Call
- Capacity Planning and Load Testing

### Security in System Design

- Authentication (Sessions, Tokens, JWT)
- Authorization (RBAC, ABAC, ReBAC)
- Single Sign-On (SAML, OIDC, Federation)
- Secrets Management and Key Rotation
- Encryption at Rest and in Transit
- Defense in Depth and Threat Modeling
- Zero Trust Architecture

### Architectural Styles

- Monolith → Microservices
- Event-Driven Architecture (style)
- Serverless and FaaS
- Hexagonal / Clean / DDD
- Cell-Based Architecture
- Control Plane vs Data Plane
- Service Discovery
- Sidecar Pattern
- Strangler Fig Pattern
- Service Mesh (architectural decision)
- Peer-to-Peer Architecture

### Case Studies

- **Basic:** URL Shortener, Pastebin, Rate Limiter
- **Real-Time Communication:** WhatsApp, Slack, Live Comments, Google Docs, Zoom, Discord, Figma
- **Social Media:** Instagram, Facebook News Feed, TikTok, Reddit, Tinder, Likes Counter
- **Media Streaming:** Spotify, YouTube, Netflix, Google Drive, Gmail, Twitch
- **Location-Based:** Airbnb, DoorDash, Uber, Google Maps
- **Search & Aggregation:** Search Autocomplete, News Aggregator, Web Crawler, Google Search, Ad Click Aggregator
- **E-commerce:** Amazon, Shopify, Flash Sale, Online Auction, Movie Booking
- **Payment & Financial:** Payment System, Digital Wallet, Stock Exchange
- **Distributed Infrastructure:** Load Balancer, API Gateway, Notification, Key-Value Store, Distributed Cache, CDN, Object Storage, Message Queue, Time-Series DB, Distributed Locking
- **Counting & Ranking:** Real-Time Leaderboard, Top-K System
- **Asynchronous:** Job Scheduler, CI/CD Pipeline, Monitoring & Alerting, Distributed Cron
- **Specialized:** LeetCode, Calendar System, Online Chess, GitHub

### Interview Framework

- 6-Step Framework
- Trade-off Articulation and Bottleneck Analysis
- Common Anti-Patterns

### Data Structures for Scale

- Bloom & Cuckoo Filters
- Merkle Trees
- HyperLogLog
- Count-Min Sketch & Top-K
- Skip Lists
- Consistent & Rendezvous Hashing
- Geohash
- Quadtrees & R-Trees
- Graph Databases and Index-Free Adjacency

### AI & Machine Learning Systems

- Vector Databases and Semantic Search
- RAG Architecture
- LLM Inference Serving
- Recommendation Systems Architecture

### Batch & Stream Processing

- Batch vs Stream Processing
- MapReduce and Its Descendants
- ETL, ELT, and Pipelines
- Lambda vs Kappa Architecture
- Modern Streaming Engines

---

## Cross-Path Reading

- [Low-Level Design Path](../low-level-design/INDEX.md) — class-level OOD, design patterns, SOLID, UML, and the canonical LLD interview problems. The layer below HLD; some case-study names overlap (URL Shortener, Rate Limiter, Chat) — different deliverable.
- [Networking Path](../networking/INDEX.md) — the wire-level detail behind load balancers, TLS, WebSocket, gRPC, CDN, service mesh.
- [Database Path](../database/INDEX.md) — the engine-internals (B-tree, LSM, MVCC, query planner) and query optimization layer underneath "databases as a component".
- [Kubernetes Path](../kubernetes/INDEX.md) — how the scalability and reliability patterns materialize as platform primitives.
- [Java Path](../java/INDEX.md) — Spring Boot / reactive / production guidance for the JVM side of the designs.
- [TypeScript Path](../typescript/INDEX.md) — Node/TS side of the same designs, plus compiler and V8 internals where performance matters.
