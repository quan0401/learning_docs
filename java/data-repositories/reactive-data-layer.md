---
title: "Building a Reactive Data Layer — Mongo, R2DBC, and Cross-Store Patterns"
date: 2026-04-18
updated: 2026-04-18
tags: [reactive, data-layer, mongo, r2dbc, webflux, architecture]
---

# Building a Reactive Data Layer — Mongo, R2DBC, and Cross-Store Patterns

Date: 2026-04-18
Tags: reactive, data-layer, mongo, r2dbc, webflux, architecture

## Table of Contents

1. [Summary](#summary)
2. [The reactive data layer goal](#the-reactive-data-layer-goal)
3. [Picking your primary store](#picking-your-primary-store)
4. [Mixing stores](#mixing-stores)
5. [Reactive Mongo specifics](#reactive-mongo-specifics)
6. [Auditing reactively](#auditing-reactively)
7. [Pagination — the hard part](#pagination--the-hard-part)
8. [Event publishing from repositories](#event-publishing-from-repositories)
9. [Transactions in a reactive data layer](#transactions-in-a-reactive-data-layer)
10. [Blocking escape hatch](#blocking-escape-hatch)
11. [Virtual threads as an alternative](#virtual-threads-as-an-alternative)
12. [Migration patterns](#migration-patterns)
13. [Testing reactive data layers](#testing-reactive-data-layers)
14. [Performance observations](#performance-observations)
15. [Common architecture anti-patterns](#common-architecture-anti-patterns)
16. [Decision guide](#decision-guide)
17. [Related](#related)
18. [References](#references)

---

## Summary

A reactive data layer is the set of decisions that make every database call return `Mono` or `Flux` without blocking a thread. It is not a single library choice — it is an architectural stance that propagates from the HTTP boundary all the way to the driver.

This doc is an architectural guide. It pulls together the concrete pieces (reactive Mongo, R2DBC, reactive Redis, reactive Elasticsearch) into a coherent data tier, and covers the cross-cutting concerns that only become interesting once you have more than one store: cross-store queries, auditing, pagination without `Page<T>`, event publishing, transactions (and why they mostly do not work across stores), and when the right call is to bail out to blocking JDBC running on virtual threads instead.

If you are new to R2DBC itself, start with `r2dbc-deep-dive.md`. If you are new to repository interfaces, start with `repository-interfaces.md`. This document assumes you are past both and are making system-level decisions.

---

## The reactive data layer goal

The single goal of a reactive data layer is to **preserve backpressure end-to-end** from the HTTP request all the way to the database driver. Any blocking call anywhere in that chain breaks the invariant — Netty's event loop stalls, threads pile up, and the whole "non-blocking" story collapses to "non-blocking with a blocking tail".

```mermaid
flowchart LR
    A[HTTP request] -->|Netty event loop| B[WebFlux controller]
    B -->|Mono Flux| C[Service layer]
    C -->|Mono Flux| D[Reactive repository]
    D -->|Reactive driver| E[R2DBC or Mongo driver]
    E -->|Non-blocking IO| F[(Database)]
    F -->|Backpressure| E
    E -->|Backpressure| D
    D -->|Backpressure| C
    C -->|Backpressure| B
    B -->|Streamed response| A
```

The arrows in both directions matter. Backpressure is not a one-way signal. A slow consumer upstream (a slow HTTP client) must be able to slow the database query, and a slow database must be able to slow the response stream. Anywhere a blocking `.block()` or a `CompletableFuture.get()` sits, that bidirectional signal is cut.

This is the invariant every architectural decision below is trying to protect.

---

## Picking your primary store

Most apps still have one primary store. Pick it based on the data, not on reactive availability — but know which drivers give you real reactive support.

### Reactive Mongo (most mature)

- `ReactiveMongoRepository<T, ID>` for Spring Data derived queries.
- `ReactiveMongoTemplate` for imperative-style programmatic queries with the `Criteria` builder.
- Tailable cursors for capped collections via `@Tailable`.
- Change streams via `ReactiveChangeStreamOperation` for event-driven patterns.
- Configuration: `spring.data.mongodb.uri=mongodb://...`.

Mongo's reactive driver has been production-grade longer than anything else on the JVM. If you are already on Mongo, the reactive path is the obvious one.

### R2DBC (for SQL)

- Postgres, MySQL, MSSQL, Oracle, H2, MariaDB.
- Full cross-reference: `r2dbc-deep-dive.md`.
- Do not confuse R2DBC with async JDBC — it is a different protocol-level driver family.

### Reactive Redis

- `ReactiveRedisTemplate` for caching, pub/sub, distributed locks, rate limiting.
- Lettuce is the reactive-native client under the hood; Jedis is not.
- Useful as a second-tier cache in front of a slower primary store.

### Reactive Elasticsearch

- `ReactiveElasticsearchOperations` (or the newer `ReactiveElasticsearchClient`).
- Full-text search, aggregations, autocomplete.
- Treat it as a read model fed from your primary store, not as a source of truth.

### Reactive Cassandra

- `ReactiveCassandraRepository` for wide-column workloads.
- Native DataStax driver is already non-blocking, so the reactive wrapper is thin.
- Best for time-series and very-high-write workloads.

---

## Mixing stores

Real apps rarely stay on one store. Reactive data layers compose cleanly across stores because every operation returns a `Mono` or `Flux` — fan-out and join become `Mono.zip(...)` calls.

```mermaid
flowchart TD
    A[Service loadDashboard] -->|parallel| B[Mongo findById]
    A -->|parallel| C[R2DBC findByUserId]
    A -->|parallel| D[Redis get prefs]
    B -->|Mono User| E[Mono.zip combine]
    C -->|Flux Order collectList| E
    D -->|Mono Preferences| E
    E -->|Mono UserDashboard| F[Controller]
```

A minimal example:

```java
public Mono<UserDashboard> loadDashboard(String userId) {
    Mono<User> userMono = userRepo.findById(userId);                    // Mongo
    Flux<Order> ordersFlux = orderRepo.findByUserId(userId);            // R2DBC
    Mono<Preferences> prefsMono = prefsCache.get(userId);               // Redis

    return Mono.zip(userMono, ordersFlux.collectList(), prefsMono)
        .map(t -> new UserDashboard(t.getT1(), t.getT2(), t.getT3()));
}
```

Three stores, three drivers, three concurrent I/O operations, one combined response. No thread blocks at any point. The latency floor is `max(mongo, r2dbc, redis)`, not the sum.

Common fan-out mistakes:

- Calling `.block()` on any of the three — instant event loop stall.
- Doing them sequentially (`flatMap` chain) when they are independent — you pay the sum of latencies.
- Fanning out to a dozen stores — each store has its own connection pool, and you can saturate the slowest one.

---

## Reactive Mongo specifics

Mongo's reactive support is the most mature in the Spring Data family. A quick tour of the pieces you actually use:

**`ReactiveMongoRepository<T, ID>`** extends `ReactiveCrudRepository<T, ID>` and adds Mongo-specific sugar (`findAllByExample`, etc.). Derived query methods work the same way they do in blocking Mongo — `findByStatusAndCreatedAtAfter(...)` generates the query at startup.

**`ReactiveMongoTemplate`** is the programmatic escape hatch when derived queries stop being expressive enough:

```java
Query q = new Query(Criteria.where("status").is("ACTIVE")
    .and("score").gte(0.8));
Flux<Document> results = template.find(q, Document.class, "documents");
```

**Tailable cursors** for capped collections — the cursor stays open and emits new documents as they are inserted:

```java
@Tailable
@Query("{ 'status': 'PENDING' }")
Flux<Job> watchPending();
```

This is a `Flux` that never completes on its own. Perfect for work queues backed by Mongo.

**Change streams** via `ReactiveChangeStreamOperation` give you a real CDC feed on replica sets — inserts, updates, deletes flow in as a `Flux<ChangeStreamEvent<T>>`. Use this for event-driven patterns rather than polling.

**Configuration:**

```properties
spring.data.mongodb.uri=mongodb://user:pass@host:27017/mydb?replicaSet=rs0
spring.data.mongodb.auto-index-creation=false
```

Leave `auto-index-creation` off in production. Manage indexes via migrations, not via framework magic at startup.

---

## Auditing reactively

Auditing (who created this, who last modified it, when) needs a different wiring in reactive apps because the "current user" is not a thread-local — it lives in the `Mono`'s context.

For Mongo:

```java
@Configuration
@EnableReactiveMongoAuditing
class AuditingConfig {

    @Bean
    public ReactiveAuditorAware<String> auditorProvider() {
        return () -> ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication().getName())
            .switchIfEmpty(Mono.just("system"));
    }
}
```

The `ReactiveAuditorAware<T>` returns a `Mono<T>` instead of a `T`. Spring Data Mongo subscribes to it at save time and threads the result into `@CreatedBy` / `@LastModifiedBy` fields.

The same pattern works for R2DBC with `@EnableR2dbcAuditing` and the same `ReactiveAuditorAware` bean — both modules pick it up.

Two gotchas:

- Blocking `SecurityContextHolder.getContext()` will return empty in a reactive pipeline. Always use `ReactiveSecurityContextHolder`.
- The `switchIfEmpty(Mono.just("system"))` matters. If the chain emits empty, Spring Data will save without an auditor and your `@CreatedBy` field will be null.

---

## Pagination — the hard part

Reactive repositories do not usually return `Spring Data Page<T>` directly. The awkward part is not that the count is inherently blocking, but that `Page` needs both the content and total-count metadata, which usually means running two queries and coordinating them into one response.

You have three options.

### Slice-style pagination

Accept a `Pageable`, return a `Flux<T>`. This maps to `LIMIT / OFFSET` or skip/take. No total count, no `Page` wrapper, just the slice.

```java
public interface OrderRepo extends ReactiveCrudRepository<Order, Long> {
    Flux<Order> findAllBy(Pageable pageable);
}
```

Use this when the UI does "next / previous" without a total page count.

### Custom `Page` assembly

When you genuinely need a total count, assemble it yourself by running the content query and the count query in parallel:

```java
public Mono<Page<Order>> paged(Pageable p) {
    return orderRepo.findAllBy(p).collectList()
        .zipWith(orderRepo.count())
        .map(t -> new PageImpl<>(t.getT1(), p, t.getT2()));
}
```

This is two round trips but in parallel. Fine for small-to-medium tables; painful on tables where `COUNT(*)` is expensive.

### Cursor-based pagination (best for large datasets)

Stop paginating by offset. Paginate by the last seen id:

```sql
SELECT * FROM orders
WHERE id > :lastSeenId
ORDER BY id ASC
LIMIT :size
```

No `COUNT(*)`, stable under insertion, scales to huge tables. The UI gives you the cursor of the last row it saw, and the next request continues from there.

Full treatment in `queries-and-pagination.md`.

---

## Event publishing from repositories

Spring Data's `@DomainEvents` annotation publishes events synchronously via `ApplicationEventPublisher` — which is blocking. That is awkward inside a reactive pipeline.

Options:

**Manual publish with offload.** Keep `ApplicationEventPublisher`, but push the call off the reactive thread:

```java
return repo.save(entity)
    .flatMap(saved -> Mono.fromRunnable(() -> publisher.publishEvent(new Saved(saved)))
        .subscribeOn(Schedulers.boundedElastic())
        .thenReturn(saved));
```

Works, but the event consumer is still blocking. Only use this for in-process listeners that do trivial work.

**Reactive sink.** Publish to a `Sinks.Many<Event>` that downstream services subscribe to as a `Flux`:

```java
private final Sinks.Many<Saved> sink = Sinks.many().multicast().onBackpressureBuffer();

public Flux<Saved> events() { return sink.asFlux(); }

// in save path
sink.tryEmitNext(new Saved(saved));
```

Fully reactive, but in-process only and lost on restart.

**Outbox pattern.** Write the event row to the same DB transaction as the entity; a separate relay reads the outbox and ships events to Kafka / RabbitMQ / whatever. This is the only option that survives restart and gives at-least-once delivery across services. Full treatment in `../messaging/event-driven-patterns.md`.

---

## Transactions in a reactive data layer

A short section because the full treatment lives in `reactive-transactions.md`.

- **Single-store reactive transactions**: `@Transactional` on a method that returns `Mono` or `Flux` works on Spring 5.2+. The framework binds the transaction to the reactive context, not to a thread-local.
- **Cross-store**: there is no reactive XA. Cross-store atomicity is out. Use sagas (with compensating actions) or the outbox pattern.
- **Reactive Mongo transactions** require a replica set. Standalone Mongo cannot do transactions, reactive or otherwise.

Do not reach for `@Transactional` on a method that writes to both Mongo and R2DBC and expect atomicity. It will not give you that.

---

## Blocking escape hatch

Sometimes you must call blocking code: a legacy JDBC driver, a vendor SDK that only has a synchronous client, a file system call. The escape hatch is `Schedulers.boundedElastic()`:

```java
return Mono.fromCallable(() -> legacyJdbcCall(id))
    .subscribeOn(Schedulers.boundedElastic());
```

Rules:

- `boundedElastic()` is sized roughly at 10x CPU count by default. Do not overwhelm it.
- Never call blocking code directly in a reactive method without this wrap. The type system will let you, but the event loop will hang.
- Put a timeout on it. Blocking code that blocks forever will exhaust the scheduler.

Full treatment of this anti-pattern-but-sometimes-necessary-pattern lives in `../reactive-blocking-jpa-pattern.md`.

---

## Virtual threads as an alternative

Java 21+ introduced virtual threads, which make blocking calls cheap. A blocking JDBC call on a virtual thread costs a few hundred bytes of stack and parks instead of consuming an OS thread.

For greenfield apps, this changes the calculus:

- If you do not **need** backpressure all the way to the DB (most CRUD apps do not), virtual threads + JDBC + JPA is simpler than a reactive data layer.
- **Hybrid pattern**: Reactive at the web layer (WebFlux or a thin reactive controller over a blocking service) with data access on virtual threads. The reactive surface handles fan-in from many clients; the virtual-thread pool handles the blocking DB calls without starving.
- The reactive data layer still wins for **streaming** workloads (Server-Sent Events over a Mongo tailable cursor, a WebSocket fed from a change stream) and for genuinely high concurrency (10k+ concurrent in-flight operations per node).

Full treatment in `../spring-virtual-threads.md`.

---

## Migration patterns

Moving from a JPA app to a reactive data layer is a real project. A realistic path:

1. **Find the blocking-in-reactive chains first.** These are buggy regardless of your migration plan. Anywhere a reactive service calls a blocking repository is a latent production incident.
2. **Decide per-module.** Some modules benefit from a reactive data layer (streaming, high concurrency, cross-store fan-out). Some do not (admin screens, reporting). Do not migrate uniformly.
3. **Pick the replacement per module:**
   - Full reactive: replace JPA with R2DBC. Expect to rewrite entity classes (no lazy loading, simpler relationships).
   - Virtual-thread isolation: keep JPA, run the calls inside a virtual-thread executor. Faster migration, retains JPA features.
4. **Watch for JPA features that do not translate:**
   - No lazy loading in R2DBC — you load the graph explicitly.
   - Simpler relationship support — R2DBC does not do `@OneToMany` joins for you; you do the join or a second query.
   - No first-level cache — no identity map across a transaction.
   - No cascading — cascades are application-level in R2DBC.
5. **Data migration is usually unnecessary.** Same database, different driver. Your schema does not change because you switched from JDBC to R2DBC.

---

## Testing reactive data layers

- **`@DataMongoTest`** — slice test for reactive Mongo repositories. Spins up the Mongo client only.
- **`@DataR2dbcTest`** — slice test for R2DBC. Uses Spring's reactive `DatabaseClient` and repositories.
- **Testcontainers** — real Postgres, real Mongo, in Docker. Mandatory for anything non-trivial — H2 in R2DBC mode does not cover enough of Postgres's SQL.
- **`StepVerifier`** from Reactor Test — the idiomatic way to assert on a `Mono` or `Flux`:

```java
StepVerifier.create(repo.findByStatus("ACTIVE"))
    .expectNextMatches(o -> o.getId() != null)
    .expectNextCount(4)
    .verifyComplete();
```

Cross-refs: `../testing/testcontainers.md`, `../testing/spring-boot-test-basics.md`.

---

## Performance observations

A few notes on what reactive data layers actually buy you in the real world.

- **High-concurrency I/O-bound workloads** are where reactive wins. 10k concurrent in-flight DB calls on a handful of Netty threads is not hard. The same workload on JDBC + thread-per-request requires a huge thread pool and runs into context-switch overhead.
- **For simple CRUD at low-to-moderate load**, a well-tuned JPA + HikariCP setup has comparable throughput to R2DBC. The reactive data layer buys you nothing here.
- **Connection pool sizing still matters.** R2DBC does not magically give you unlimited database capacity. Your DB can handle N concurrent queries; beyond that, queries queue. Reactive is better at queuing without wasting threads, but the queue is real.
- **Once the data layer is reactive, the DB becomes the bottleneck.** That is the correct bottleneck. Before, the bottleneck was usually your thread pool, which is a lie — the DB was sitting idle while threads were blocked. Reactive exposes the real constraint.
- **Latency distribution matters more than p50.** Reactive apps tend to have better tail latency (p99, p99.9) because threads are not competing for a limited pool. Measure p99, not the average.

---

## Common architecture anti-patterns

A field guide to things that look fine and are not:

- **`.block()` inside a reactive service.** Defeats the entire point of the stack. Will work in dev, explode under load.
- **Reactive controller calling a blocking repository wrapped in `Mono.just(...)`.** The type system now says `Mono<Foo>` — but the blocking call happened on the Netty event loop before the `Mono` was constructed. Silent stall.
- **Over-fanned-out queries.** `Mono.zip(a, b, c, d, e, f, g, h)` saturating the DB connection pool. Each parallel call takes a connection. The pool is finite.
- **`Flux` of individual lookups.** The N+1 problem in reactive form. `flatMap(id -> repo.findById(id))` sends N queries. Use batch queries (`findAllById(Collection)`) or a single join.
- **Reactive data layer with synchronous business logic everywhere.** If every controller ends in `.block()` to build a response, you added complexity and got nothing in return. Either commit to reactive end-to-end or go back to blocking with virtual threads.
- **Sharing a blocking service across reactive and blocking callers.** The blocking caller does not care; the reactive caller blocks the event loop. Split the service, or make it reactive.

---

## Decision guide

When is a reactive data layer actually worth it?

| Scenario | Recommendation |
|---|---|
| Mostly CRUD, under 1000 rps | JPA + virtual threads is simpler |
| Streaming APIs reading from a DB, 10k+ rps | Reactive data layer + WebFlux |
| SSE or WebSocket feeds backed by DB change streams | Reactive data layer |
| Heavy cross-store fan-out (Mongo + SQL + Redis + search) | Reactive data layer |
| Mixed — some endpoints reactive, some blocking | Hybrid: reactive web, virtual-thread blocking data access |
| Existing JPA app, no performance problems | Do not migrate for migration's sake |
| Greenfield app, team has no reactive experience | JPA + virtual threads first; migrate later if needed |
| Long-running connections (tailable cursors, change streams) | Reactive data layer — it is what those APIs are designed for |

Rule of thumb: a reactive data layer is infrastructure, not a feature. If it solves a problem you have, it pays for itself. If it does not, it is a tax on every future developer who touches the code.

---

## Related

- `r2dbc-deep-dive.md`
- `reactive-transactions.md`
- `repository-interfaces.md`
- `queries-and-pagination.md`
- `../reactive-blocking-jpa-pattern.md`
- `../spring-virtual-threads.md`
- `../configurations/database-config.md`
- `../reactive-programming-java.md`

---

## References

- Spring Data Reactive Repositories overview — https://docs.spring.io/spring-data/commons/reference/repositories/reactive.html
- Spring Data MongoDB Reactive — https://docs.spring.io/spring-data/mongodb/reference/mongodb/reactive-mongo-template.html
- Spring Data R2DBC reference — https://docs.spring.io/spring-data/relational/reference/r2dbc.html
- Reactive Streams specification — https://www.reactive-streams.org/
- Project Reactor documentation — https://projectreactor.io/docs/core/release/reference/
- Spring Framework reactive transactions — https://docs.spring.io/spring-framework/reference/data-access/transaction/reactive.html
- R2DBC specification — https://r2dbc.io/spec/
