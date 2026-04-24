# System Design Documentation Index — Learning Path

A progressive path from mental models and building blocks through scalability, data consistency, reliability, and real-world design problems. System design is where the other learning paths compose — databases, protocols, caches, queues, and services become architectures that have to survive traffic, failure, and change.

Cross-references to the [TypeScript learning path](../typescript/INDEX.md), the [Java learning path](../java/INDEX.md), the [Database learning path](../database/INDEX.md), the [Networking learning path](../networking/INDEX.md), and the [Kubernetes learning path](../kubernetes/INDEX.md) where topics overlap.

If you already know how to build and deploy a service, **start at Tier 1 (Foundations & Mental Models)** to install the reasoning vocabulary you'll use in every design discussion, then work forward. Tier 10 (case studies) exercises everything from Tiers 1–9 against real problems.

---

## Tier 1 — Foundations & Mental Models

Groundwork: what system design is, how to reason about trade-offs, and the numbers every designer should memorize before drawing a single box.

1. [Back-of-Envelope Estimation — Latency Numbers, QPS Math, and Capacity Planning](foundations/back-of-envelope-estimation.md) — memorized latency table, QPS → storage → bandwidth conversions, Fermi estimates, sizing servers, caches, and queues _(2026-04-24)_
2. [CAP, PACELC, and Consistency Models — Strong, Eventual, Causal, Linearizable](foundations/cap-and-consistency-models.md) — CAP theorem in practice, PACELC extension, consistency hierarchy, read-your-writes, monotonic reads, session guarantees _(2026-04-24)_
3. [SLA, SLO, SLI, and the Math of Availability — Error Budgets and Composition](foundations/sla-slo-sli-and-availability.md) — nines vs minutes, serial vs parallel availability composition, error budgets, burn rate alerts, SLO-driven engineering _(2026-04-24)_
4. [Non-Functional Requirements Checklist — Scalability, Availability, Durability, Cost](foundations/non-functional-requirements.md) — the NFR taxonomy, how to elicit requirements, trade-off articulation, what "scale" actually means per dimension _(2026-04-24)_

---

## Tier 2 — Core Building Blocks

The named components every design reuses. One doc per block, deep enough to reason about trade-offs instead of just naming them.

5. [Load Balancers in System Design — L4 vs L7, Algorithms, and Health](building-blocks/load-balancers.md) — where an LB fits in the design, stickiness, graceful drain, global vs regional vs local LBs, LB as an SPOF, cross-ref to networking _(2026-04-24)_
6. [Caching Layers — Client, CDN, Reverse-Proxy, Application, Distributed](building-blocks/caching-layers.md) — the caching hierarchy, cache-aside vs read/write-through, invalidation strategies, TTLs, stampede protection, Redis vs Memcached _(2026-04-24)_
7. [Databases as a Component — SQL, NoSQL, NewSQL, and Picking One](building-blocks/databases-as-a-component.md) — OLTP taxonomy, document vs key-value vs wide-column vs graph, NewSQL trade-offs, polyglot persistence, decision framework _(2026-04-24)_
8. [Message Queues & Brokers — Kafka, RabbitMQ, SQS, NATS](building-blocks/message-queues-and-brokers.md) — log vs queue semantics, delivery guarantees, partitioning, consumer groups, when to choose which broker _(2026-04-24)_
9. [Object and Blob Storage — S3-Style Systems, Chunking, Presigned URLs](building-blocks/object-and-blob-storage.md) — flat namespace, eventual consistency (now strong on S3), multipart upload, lifecycle policies, presigned URLs, cold tiers _(2026-04-24)_
10. [Search Systems — Inverted Index, Elasticsearch / OpenSearch](building-blocks/search-systems.md) — what a search engine really is, index pipeline, relevance scoring, facets, when search belongs next to vs on top of the DB _(2026-04-24)_
11. [API Gateways and BFFs — Routing, Auth Offload, Composition](building-blocks/api-gateways-and-bff.md) — gateway responsibilities, BFF pattern, gateway vs service mesh, common anti-patterns, Kong/Envoy/AWS API Gateway/Apollo _(2026-04-24)_
12. [Rate Limiters — Token Bucket, Leaky Bucket, Sliding Window](building-blocks/rate-limiters.md) — algorithms with pseudocode, distributed rate limiting with Redis, per-user vs per-IP vs per-key, graceful degradation _(2026-04-24)_

---

## Tier 3 — Scalability Patterns

How you grow a system past a single box. These patterns compose on top of the Tier 2 building blocks.

13. [Horizontal vs Vertical Scaling & Stateless Services](scalability/horizontal-vs-vertical-and-stateless.md) — scale-up vs scale-out economics, the stateless invariant, session externalization, autoscaling signals, warm pools _(2026-04-24)_
14. [Sharding Strategies — Range, Hash, Directory, Geo, Consistent Hashing](scalability/sharding-strategies.md) — shard key selection, hot spots, rebalancing, consistent hashing with virtual nodes, cross-shard queries, resharding playbook _(2026-04-24)_
15. [Replication Patterns — Primary-Replica, Multi-Primary, Quorum](scalability/replication-patterns.md) — sync vs async replication, replication lag, failover, split-brain, multi-primary conflict resolution, quorum reads/writes intro _(2026-04-24)_
16. [Read/Write Splitting & Cache Strategies — Through, Back, Around, Aside](scalability/read-write-splitting-and-cache-strategies.md) — write-through/back/around, cache-aside in depth, TTL vs event invalidation, stampede locks, negative caching, thundering herd _(2026-04-24)_
17. [CQRS and Event Sourcing — Commands, Queries, and the Event Log](scalability/cqrs-and-event-sourcing.md) — why split read and write models, event sourcing mechanics, projections, snapshotting, trade-offs vs classic CRUD _(2026-04-24)_
18. [Backpressure, Bulkhead, and Circuit Breakers — Failing Loudly Under Load](scalability/backpressure-bulkhead-circuit-breaker.md) — reactive streams backpressure, bulkhead isolation, three-state circuit breaker, timeouts vs deadlines, load shedding _(2026-04-24)_

---

## Tier 4 — Data & Consistency

Distributed data is where most designs fail. This tier goes deeper than Tier 2's database overview.

19. [ACID vs BASE and Isolation Levels in Practice](data-consistency/acid-vs-base-and-isolation-levels.md) — isolation level anomalies (dirty/non-repeatable/phantom), snapshot isolation, serializable, BASE trade-offs _(planned)_
20. [Distributed Transactions — 2PC, Sagas, Outbox](data-consistency/distributed-transactions.md) — why 2PC is rare, saga orchestration vs choreography, the transactional outbox pattern, idempotent consumers _(planned)_
21. [Consensus — Raft and Paxos at a Conceptual Level](data-consistency/consensus-raft-and-paxos.md) — the problem consensus solves, Raft state machine, log replication, leader election, Paxos contrast _(planned)_
22. [Leader Election and Coordination Services — ZooKeeper, etcd](data-consistency/leader-election-and-coordination.md) — ephemeral nodes, watches, leases, fencing tokens, when to use ZK/etcd vs rolling your own _(planned)_
23. [Quorum Reads/Writes (NWR) and Tunable Consistency](data-consistency/quorum-and-tunable-consistency.md) — N/W/R math, read/write repair, hinted handoff, Dynamo-style systems, Cassandra consistency levels _(planned)_
24. [Change Data Capture (CDC) and Dual Writes](data-consistency/change-data-capture.md) — why dual writes break, CDC via logical replication, Debezium architecture, CDC → Kafka → consumers _(planned)_
25. [OLTP vs OLAP, Data Lakes, and Lakehouses](data-consistency/oltp-vs-olap-and-lakehouses.md) — row vs columnar storage, warehouse vs lake vs lakehouse, Iceberg/Delta/Hudi, the analytics side of system design _(planned)_

---

## Tier 5 — Communication & Messaging

How services talk. Sync vs async, delivery guarantees, streaming.

26. [Sync vs Async Communication — REST, gRPC, Messaging](communication/sync-vs-async-communication.md) — latency coupling, failure coupling, when async wins, request-response over queues, saga handoff _(planned)_
27. [Event-Driven Architecture — Pub/Sub, Choreography vs Orchestration](communication/event-driven-architecture.md) — event types, event vs command, choreography trade-offs, orchestration with workflow engines _(planned)_
28. [Idempotency and Exactly-Once Semantics](communication/idempotency-and-exactly-once.md) — idempotency keys, dedup windows, effectively-once via idempotent consumers, transactional outbox intro _(planned)_
29. [Dead-Letter Queues, Retries, and Poison Messages](communication/dead-letter-queues-and-retries.md) — retry topologies, DLQ patterns, poison message handling, retry budgets, observability on DLQs _(planned)_
30. [Stream Processing — Kafka Streams, Flink, Windowing](communication/stream-processing.md) — event-time vs processing-time, tumbling/sliding/session windows, exactly-once in streams, state stores _(planned)_

---

## Tier 6 — Reliability & Resilience

Designing for failure instead of against it.

31. [Failure Modes and Fault Tolerance Taxonomy](reliability/failure-modes-and-fault-tolerance.md) — crash-stop, omission, timing, byzantine, gray failures, blast radius, failure domains _(planned)_
32. [Retry Strategies — Exponential Backoff, Jitter, Budgets](reliability/retry-strategies.md) — why naive retries make outages worse, full jitter, retry budgets, circuit-breaker integration _(planned)_
33. [Chaos Engineering and Game Days](reliability/chaos-engineering-and-game-days.md) — chaos principles, blast-radius containment, game day structure, what to simulate first _(planned)_
34. [Disaster Recovery — RPO, RTO, Backup Strategies](reliability/disaster-recovery.md) — RPO vs RTO, backup tiers, pilot light vs warm standby vs active-active, DR drills _(planned)_
35. [Multi-Region Architectures — Active-Active, Active-Passive, Geo-Routing](reliability/multi-region-architectures.md) — regional failure domains, replication topologies, conflict resolution, geo-DNS vs anycast vs global LB _(planned)_
36. [Graceful Degradation and Feature Flags](reliability/graceful-degradation-and-feature-flags.md) — degradation tiers, kill switches, circuit breaker + flag integration, flag lifecycle _(planned)_

---

## Tier 7 — Performance & Observability

Making it fast and knowing when it isn't.

37. [Performance Budgets and Latency Analysis — Tail Latency and Coordinated Omission](performance-observability/performance-budgets-and-latency.md) — averages lie, p99/p99.9, coordinated omission, hedged requests, latency SLO sizing _(planned)_
38. [Distributed Tracing, Metrics, and Logs — The Three Pillars](performance-observability/tracing-metrics-logs.md) — OpenTelemetry model, trace propagation, cardinality traps, structured logging, correlation IDs _(planned)_
39. [Monitoring and Alerting — RED, USE, and the Four Golden Signals](performance-observability/monitoring-red-use-golden-signals.md) — RED for services, USE for resources, golden signals, SLO-based alerting, alert fatigue _(planned)_
40. [Capacity Planning and Load Testing](performance-observability/capacity-planning-and-load-testing.md) — headroom planning, load test shapes (ramp/soak/spike), k6/Gatling/Locust, production shadowing _(planned)_

---

## Tier 8 — Security in System Design

Design-level security decisions — not code-level hardening.

41. [AuthN vs AuthZ — OAuth2/OIDC, JWT, Sessions at Scale](security/authn-authz-oauth-oidc-jwt.md) — authentication vs authorization, OAuth2 flows, OIDC on top, JWT vs sessions trade-offs, central vs distributed authz _(planned)_
42. [Secrets Management and Key Rotation](security/secrets-management-and-key-rotation.md) — secret storage (Vault, SM, KMS), rotation strategies, envelope encryption, workload identity _(planned)_
43. [Encryption at Rest and in Transit](security/encryption-at-rest-and-in-transit.md) — at-rest: disk vs DB vs app; in-transit: TLS, mTLS, E2EE; KMS hierarchies, BYOK _(planned)_
44. [Defense in Depth and Threat Modeling](security/defense-in-depth-and-threat-modeling.md) — layered defense, STRIDE, trust boundaries, attack surface, threat model in the design review _(planned)_
45. [Zero Trust Architecture — BeyondCorp, Identity-Aware Proxies](security/zero-trust-architecture.md) — identity-centric model, microsegmentation, IAP, policy engines, what zero trust does NOT mean _(planned)_

---

## Tier 9 — Architectural Styles

How to slice the system.

46. [Monolith, Modular Monolith, SOA, Microservices — When Each Wins](architectural-styles/monolith-to-microservices.md) — tech vs org trade-offs, modular monolith as the default, microservices prerequisites, "distributed monolith" anti-pattern _(planned)_
47. [Event-Driven Architecture as a Style](architectural-styles/event-driven-architecture-style.md) — EDA beyond a pattern, event backbone, topology choices, when EDA is wrong _(planned)_
48. [Serverless and FaaS Trade-offs](architectural-styles/serverless-and-faas.md) — cold starts, state, concurrency limits, vendor lock-in, cost curves, good vs bad fits _(planned)_
49. [Hexagonal, Clean, and DDD — Ports, Adapters, Bounded Contexts](architectural-styles/hexagonal-clean-ddd.md) — domain-first design, ports/adapters, aggregates, bounded contexts, context maps _(planned)_
50. [Service Mesh as an Architectural Decision](architectural-styles/service-mesh-as-architectural-decision.md) — what a mesh buys you, sidecar overhead, ambient mesh, when mesh is overkill _(planned)_

---

## Tier 10 — Case Studies: Designing Real Systems

The interview-style design problems. Each doc follows a consistent template: requirements → estimates → API → data model → high-level design → deep dives → bottlenecks → trade-offs.

51. [Design a URL Shortener](case-studies/design-url-shortener.md) — counter vs hash vs base62, redirect infra, click analytics, custom aliases, abuse protection _(planned)_
52. [Design a Rate Limiter (Service)](case-studies/design-rate-limiter.md) — centralized vs distributed, Redis-backed sliding window, policy engine, client SDKs, observability _(planned)_
53. [Design a Distributed Cache](case-studies/design-distributed-cache.md) — partitioning, replication, eviction, consistent hashing ring, client topology awareness _(planned)_
54. [Design a News Feed / Timeline (Twitter-Style)](case-studies/design-news-feed.md) — fan-out on write vs read, celebrity problem, timeline generation, ranking intro _(planned)_
55. [Design a Chat System (WhatsApp-Style)](case-studies/design-chat-system.md) — 1:1 vs group, presence, delivery receipts, message ordering, media handling, offline delivery _(planned)_
56. [Design a Notification System](case-studies/design-notification-system.md) — fan-out, channel routing (push/email/SMS), preferences, rate limiting, DLQ handling _(planned)_
57. [Design a Video Streaming Service (YouTube/Netflix-Style)](case-studies/design-video-streaming.md) — upload pipeline, transcoding, CDN distribution, ABR, recommendation intro _(planned)_
58. [Design a Ride-Sharing Service (Uber-Style)](case-studies/design-ride-sharing.md) — geo indexing (geohash/S2), dispatch matching, trip lifecycle, surge pricing, dual write concerns _(planned)_
59. [Design a File Storage & Sync Service (Dropbox-Style)](case-studies/design-file-storage-and-sync.md) — chunking, dedup, delta sync, metadata DB, block storage, mobile constraints _(planned)_
60. [Design a Collaborative Editor (Google Docs-Style)](case-studies/design-collaborative-editor.md) — OT vs CRDT, presence, cursor broadcast, offline merges, history _(planned)_
61. [Design a Search Autocomplete](case-studies/design-search-autocomplete.md) — trie vs prefix index, ranking, typo tolerance, freshness, personalization _(planned)_
62. [Design a Payment System](case-studies/design-payment-system.md) — idempotency, double-entry ledgers, reconciliation, fraud gates, async settlement _(planned)_
63. [Design a Metrics & Monitoring System (Datadog-Style)](case-studies/design-metrics-monitoring-system.md) — ingestion, time-series storage, rollups, alerting engine, dashboard compute _(planned)_
64. [Design an Ad Click Tracking Pipeline](case-studies/design-ad-click-tracking.md) — click-through capture, fraud filtering, attribution windows, real-time aggregates, billing correctness _(planned)_

---

## Tier 11 — The Interview Framework

How to actually run a system design discussion — useful even if you never interview again, because it mirrors how real architecture reviews should flow.

65. [The 6-Step Framework — Requirements → Estimates → API → Data Model → HLD → Deep Dives](interview-framework/six-step-framework.md) — the canonical structure, time allocation, where candidates stumble _(planned)_
66. [Trade-off Articulation and Bottleneck Analysis](interview-framework/tradeoff-articulation-and-bottlenecks.md) — naming trade-offs explicitly, identifying bottlenecks, the "what breaks at 10x?" drill _(planned)_
67. [Common Anti-Patterns in System Design Interviews](interview-framework/common-anti-patterns.md) — premature detail, resume-driven design, ignoring NFRs, hand-waving at scale _(planned)_

---

## Quick Reference by Topic

### Foundations

- [Back-of-Envelope Estimation](foundations/back-of-envelope-estimation.md)
- [CAP, PACELC, and Consistency Models](foundations/cap-and-consistency-models.md)
- [SLA, SLO, SLI, and Availability](foundations/sla-slo-sli-and-availability.md)
- [Non-Functional Requirements Checklist](foundations/non-functional-requirements.md)

### Building Blocks

- [Load Balancers](building-blocks/load-balancers.md)
- [Caching Layers](building-blocks/caching-layers.md)
- [Databases as a Component](building-blocks/databases-as-a-component.md)
- [Message Queues & Brokers](building-blocks/message-queues-and-brokers.md)
- [Object and Blob Storage](building-blocks/object-and-blob-storage.md)
- [Search Systems](building-blocks/search-systems.md)
- [API Gateways and BFFs](building-blocks/api-gateways-and-bff.md)
- [Rate Limiters](building-blocks/rate-limiters.md)

### Scalability Patterns

- [Horizontal vs Vertical Scaling](scalability/horizontal-vs-vertical-and-stateless.md)
- [Sharding Strategies](scalability/sharding-strategies.md)
- [Replication Patterns](scalability/replication-patterns.md)
- [Read/Write Splitting & Cache Strategies](scalability/read-write-splitting-and-cache-strategies.md)
- [CQRS and Event Sourcing](scalability/cqrs-and-event-sourcing.md)
- [Backpressure, Bulkhead, Circuit Breaker](scalability/backpressure-bulkhead-circuit-breaker.md)

### Data & Consistency _(planned)_

- ACID vs BASE and Isolation Levels
- Distributed Transactions (2PC, Sagas, Outbox)
- Consensus (Raft, Paxos)
- Leader Election and Coordination
- Quorum and Tunable Consistency
- Change Data Capture
- OLTP vs OLAP, Lakehouses

### Communication & Messaging _(planned)_

- Sync vs Async Communication
- Event-Driven Architecture
- Idempotency and Exactly-Once
- Dead-Letter Queues and Retries
- Stream Processing

### Reliability & Resilience _(planned)_

- Failure Modes and Fault Tolerance
- Retry Strategies
- Chaos Engineering and Game Days
- Disaster Recovery
- Multi-Region Architectures
- Graceful Degradation and Feature Flags

### Performance & Observability _(planned)_

- Performance Budgets and Latency
- Distributed Tracing, Metrics, Logs
- Monitoring (RED/USE/Golden Signals)
- Capacity Planning and Load Testing

### Security in System Design _(planned)_

- AuthN vs AuthZ (OAuth2/OIDC/JWT)
- Secrets Management and Key Rotation
- Encryption at Rest and in Transit
- Defense in Depth and Threat Modeling
- Zero Trust Architecture

### Architectural Styles _(planned)_

- Monolith → Microservices
- Event-Driven Architecture (style)
- Serverless and FaaS
- Hexagonal / Clean / DDD
- Service Mesh (architectural decision)

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

---

## Cross-Path Reading

- [Networking Path](../networking/INDEX.md) — the wire-level detail behind load balancers, TLS, WebSocket, gRPC, CDN, service mesh.
- [Database Path](../database/INDEX.md) — the engine-internals and query optimization layer underneath "databases as a component".
- [Kubernetes Path](../kubernetes/INDEX.md) — how the scalability and reliability patterns materialize as platform primitives.
- [Java Path](../java/INDEX.md) — Spring Boot / reactive / production guidance for the JVM side of the designs.
- [TypeScript Path](../typescript/INDEX.md) — Node/TS side of the same designs, plus compiler and V8 internals where performance matters.
