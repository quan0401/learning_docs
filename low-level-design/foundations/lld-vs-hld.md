---
title: "LLD vs HLD — Where the Boundary Sits"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, foundations, hld, system-design, interview-method]
---

# LLD vs HLD — Where the Boundary Sits

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `foundations` `hld` `system-design` `interview-method`
## Summary

High-Level Design (HLD) describes a system as a set of services, datastores, queues, and infrastructure pieces talking over a network; Low-Level Design (LLD) describes the inside of any one of those services as classes, interfaces, and method-level interactions. The boundary sits at the process edge: anything that crosses a wire belongs to HLD, and anything that lives inside a single deployable unit belongs to LLD.

## Table of Contents

- [The Two Levels at a Glance](#the-two-levels-at-a-glance)
- [What HLD Produces](#what-hld-produces)
- [What LLD Produces](#what-lld-produces)
- [The Hand-Off Contract](#the-hand-off-contract)
- [Common Confusion](#common-confusion)
- [Concrete Example: URL Shortener](#concrete-example-url-shortener)
- [Typical Interview Split](#typical-interview-split)
- [Heuristics for Drawing the Line](#heuristics-for-drawing-the-line)
- [Related](#related)

## The Two Levels at a Glance

| Aspect | HLD | LLD |
|---|---|---|
| Granularity | Services, datastores, queues, regions | Classes, interfaces, methods |
| Unit of thought | A box on a network diagram | A type in a programming language |
| Primary diagrams | Component diagram, deployment diagram, data flow | Class diagram, sequence diagram |
| Quantitative concerns | QPS, latency budget, storage size, replication factor | Time/space complexity, memory layout |
| Failure modes addressed | Node failure, region outage, network partition | Null pointer, race condition, unhandled exception |
| Validated by | Capacity model, SLA targets | Compileability, code review, unit tests |
| Typical audience | Architects, SREs, leads, PMs | Engineers implementing the service |

Think of HLD as the city plan (where roads, power, water go) and LLD as the floor plan of a building (where rooms, walls, doors go).

## What HLD Produces

A complete HLD typically delivers:

- **Component topology** — the set of services, what each service owns, and how they call each other (REST, gRPC, async events).
- **Capacity model** — read QPS, write QPS, peak/avg, storage in GB/TB, growth rate.
- **Data flow** — how a request enters the system, which services it touches, where state is written, where caches sit.
- **Datastore choices** — SQL vs document vs key-value vs columnar, sharding scheme, replication strategy.
- **Caching strategy** — what is cached, where, with what TTL and invalidation rules.
- **Async pipelines** — message brokers (Kafka, SQS, NATS), event schemas, consumer groups.
- **Infrastructure shape** — load balancers, CDN, regions, failure domains, deployment topology (Kubernetes, ECS, bare VMs).
- **Cross-cutting concerns** — auth, observability, rate limiting, multi-tenancy isolation.
- **SLA / SLO targets** — availability, latency p50/p99, durability.

Notice none of these talk about classes, methods, or programming language. HLD is *language-agnostic by design* — it's about the system's shape on the network and disk, not its shape inside a single process.

## What LLD Produces

A complete LLD typically delivers:

- **Entities and value objects** — domain types like `Order`, `LineItem`, `Money`, `Address`.
- **Service / class structure** — the runtime objects: `OrderService`, `PaymentProcessor`, `InventoryReserver`.
- **Interfaces and contracts** — what each class promises (signature, exceptions, idempotency).
- **Class relationships** — composition, aggregation, inheritance, association, with multiplicities.
- **Sequence diagrams** — how the classes collaborate during a flow ("place order", "cancel order", "refund").
- **Concurrency model** — what is thread-safe, what is request-scoped, what locks exist.
- **Patterns applied** — Strategy for pricing rules, Observer for inventory updates, Factory for parser construction, etc.
- **Extension points** — where new rules / channels / providers will plug in next quarter.

LLD is necessarily language-flavored. The same HLD can yield very different LLDs depending on whether you're writing Java, TypeScript, Go, or Rust — because the language's idioms (interfaces vs structural types, inheritance vs composition, exceptions vs Result types) reshape the class graph.

## The Hand-Off Contract

The HLD → LLD hand-off should give the LLD author the following without requiring further questions:

1. **Which service am I designing the inside of?** (The bounded context.)
2. **What are this service's external interfaces?** (REST endpoints, gRPC methods, consumed events, produced events.)
3. **What datastores does this service own?** (Schema may be sketched in HLD or finalized in LLD — pick a convention and stick to it.)
4. **What are this service's SLAs?** (Drives concurrency model and caching choices in LLD.)
5. **What's outside this service?** (So LLD knows where the boundaries are — anything outside is a client of an interface, not a class to design.)

A clean hand-off lets LLD focus narrowly. A muddy hand-off forces LLD to reinvent decisions that should have been made at the HLD level (or vice versa).

## Common Confusion

These mix-ups show up constantly in interviews and design docs:

### Mixing infra into LLD

> "And then `OrderService` will write to a Postgres replica in `us-east-2` and the read replica in `us-west-2` will lag by..."

Stop. Replication topology is HLD. The LLD just says: `OrderService` depends on `OrderRepository`, an interface. Whether `OrderRepository` writes to one Postgres or three is invisible at this level.

### Mixing classes into HLD

> "We'll have a `User` class and a `UserService` class and an `AuthInterceptor`..."

Stop. At HLD level, the system has an "auth service" and a "user service" — boxes on the diagram. The class structure inside each box is LLD's job. Naming classes during HLD wastes the room's time and prematurely commits to one OO shape.

### Drawing class diagrams across services

A class diagram that crosses service boundaries is almost always wrong. Cross-service relationships are *protocol* relationships (HTTP calls, message contracts), not *class* relationships. Classes inside Service A do not "compose" classes inside Service B.

### Capacity numbers in LLD

LLD doesn't usually say "we expect 10K QPS". It says "this method must be O(log n) and thread-safe", because the QPS target is given by HLD and constrains the algorithmic choice — but the number itself doesn't belong in the class diagram.

### Patterns in HLD

GoF patterns (Strategy, Factory, Decorator, etc.) belong to LLD. Architectural patterns (CQRS, Event Sourcing, Saga, API Gateway) belong to HLD. The vocabulary overlap ("pattern") tricks people. They live at different zoom levels.

## Concrete Example: URL Shortener

To make the boundary tangible, here's how the same problem splits.

### HLD answers

- One service for write-path (`POST /shorten`), one service for read-path (`GET /:code`).
- Code generation: base62 over a monotonically increasing 64-bit counter, sharded.
- Storage: key-value store (DynamoDB / Cassandra) keyed on short code.
- Cache: read-side CDN + Redis L1, ~95% hit rate target.
- Read QPS: 50K avg / 200K peak. Write QPS: 500 avg / 2K peak.
- Multi-region active-active with eventual consistency for non-canonical regions.

### LLD answers (for the read service)

- `RedirectController` exposes `getOriginalUrl(code: String): RedirectResponse`.
- `RedirectController` depends on `RedirectService` (interface).
- `RedirectService` depends on `UrlCache` (interface) and `UrlRepository` (interface).
- `CachingRedirectService` implements `RedirectService`, checks `UrlCache` first, falls back to `UrlRepository`, populates cache on miss.
- `RedirectResponse` is a value object: `(originalUrl: String, statusCode: Int, ttlSeconds: Long)`.
- Concurrency: `CachingRedirectService` is stateless and safe to share across request threads. `UrlCache` implementations are responsible for their own thread-safety.
- Failure mode: `UrlRepository` throws `UrlNotFoundException` → controller maps to 404.

Notice that the LLD doesn't care that the cache is Redis or that the repository is DynamoDB. Those are infrastructure details bound at startup via DI configuration. Swapping them in tests is a plain mock against the interface.

## Typical Interview Split

At most major product companies, system-design loops have separated rounds:

| Round | Format | Focus | Time |
|---|---|---|---|
| HLD | Whiteboard / virtual board | Service topology, capacity, data, scaling | 45–60 min |
| LLD / OOD | Whiteboard / shared doc | Class diagram, interfaces, patterns | 45–60 min |
| Machine Coding | IDE / shared editor | Working code with tests for an OOD problem | 60–120 min |

The HLD round rewards thinking about *the wire* (network calls, replication, failure modes). The LLD round rewards thinking about *the program* (class graph, interfaces, extension).

A common candidate failure is treating the LLD round like an HLD round — drawing service diagrams, talking about Kafka, dodging the question of what classes exist. The interviewer is usually waiting for a UML-shaped answer, not an architecture diagram.

The reverse failure also happens: in an HLD round, candidates dive into class names and method signatures, leaving capacity, sharding, and failure modes on the floor.

## Heuristics for Drawing the Line

When unsure which level a topic belongs to, ask:

- **Does it cross a process boundary?** → HLD.
- **Does it require a programming language to express?** → LLD.
- **Does it have a QPS / GB / latency number attached?** → HLD.
- **Does it have a method signature attached?** → LLD.
- **Could this decision change without breaking any class diagram?** → HLD.
- **Could this decision change without changing any service topology?** → LLD.

These rules are heuristics, not laws. There are gray areas (e.g. database schema design straddles both). The point is to default to a level and only cross over with intention.

## Related

- [What Is Low-Level Design?](./what-is-lld.md)
- [Types of LLD Interviews — OOD, Machine Coding, Live Pairing](./types-of-lld-interviews.md)
- [System Design Index](../../system-design/INDEX.md) — HLD-side companion
- [Java Index](../../java/INDEX.md)
