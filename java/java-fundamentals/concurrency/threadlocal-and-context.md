---
title: "ThreadLocal and Context Propagation — Leaks, ScopedValue, Reactor Context"
date: 2026-04-24
updated: 2026-04-24
tags: [java, concurrency, threadlocal, scopedvalue, context-propagation, mdc]
---

# ThreadLocal and Context Propagation — Leaks, ScopedValue, Reactor Context

**Date:** 2026-04-24 | **Updated:** 2026-04-24
**Tags:** `java` `concurrency` `threadlocal` `scopedvalue` `context-propagation` `mdc`

## Table of Contents

- [Summary](#summary)
- [Classic ThreadLocal](#classic-threadlocal)
- [The ThreadLocal Leak](#the-threadlocal-leak)
- [InheritableThreadLocal](#inheritablethreadlocal)
- [Where ThreadLocal Shows Up in Spring](#where-threadlocal-shows-up-in-spring)
- [Virtual Threads Break the ThreadLocal Assumption](#virtual-threads-break-the-threadlocal-assumption)
- [ScopedValue — The Modern Replacement](#scopedvalue--the-modern-replacement)
- [TransmittableThreadLocal — Pool-Crossing TL](#transmittablethreadlocal--pool-crossing-tl)
- [Reactor Context](#reactor-context)
- [OpenTelemetry Context](#opentelemetry-context)
- [Migration Guide](#migration-guide)
- [Related](#related)
- [References](#references)

---

## Summary

[`ThreadLocal<T>`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ThreadLocal.html) gives each thread its own copy of a value — used everywhere for request-scoped context (security principal, trace ID, MDC). It's load-bearing but legacy: in Spring, it's behind `SecurityContextHolder`, `RequestContextHolder`, and Logback MDC. In 2026, there are three better options depending on what you're doing: [`ScopedValue`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ScopedValue.html) (Java 25 finalized, JEP 506) for in-process context with virtual-thread-friendly semantics, [Reactor `Context`](https://projectreactor.io/docs/core/release/api/reactor/util/context/Context.html) for reactive pipelines, and [OpenTelemetry `Context`](https://opentelemetry.io/docs/specs/otel/context/) for cross-library observability. This doc covers all four, their trade-offs, and the migration path away from ThreadLocal.

---

## Classic ThreadLocal

```java
private static final ThreadLocal<String> TRACE_ID = ThreadLocal.withInitial(() -> "unknown");

// Inbound servlet filter
void doFilter(...) {
    TRACE_ID.set(request.getHeader("X-Trace-Id"));
    try {
        chain.doFilter(request, response);
    } finally {
        TRACE_ID.remove();   // CRITICAL — see "ThreadLocal leak" below
    }
}

// Anywhere in the request handling stack
String id = TRACE_ID.get();
log.info("[{}] processing", id);
```

API surface:
- `get()` — returns the current thread's value, or runs `initialValue()` if not set.
- `set(value)` — store a value for the current thread.
- `remove()` — clear the value. **Must be called in pooled-thread environments.**
- `withInitial(supplier)` — shorthand for providing a default.

Internally, `ThreadLocal` stores entries in a map attached to each `Thread` object. The map key is the `ThreadLocal` instance (via WeakReference); the value is strong-referenced. That asymmetry is the root of the leak.

---

## The ThreadLocal Leak

The classic bug: use a pooled thread, set a ThreadLocal, never `remove()` it. The thread returns to the pool. Next request gets the same thread — sees the previous request's value.

Worse, if the value holds a large object (a `ClassLoader`, a DB connection, a classloader reference), it's retained for the lifetime of the thread. In servlet containers with hot redeploy, this leaks entire old classloaders across redeploys — the infamous "PermGen / Metaspace leak".

```mermaid
flowchart LR
    A[Request 1] -->|set\(\)| B[Thread in pool]
    A -->|completes, no remove\(\)| C[Thread returns to pool]
    D[Request 2] --> C
    D -->|get\(\)| E[Reads stale value from Request 1!]
```

Rule: **always pair `set()` with `remove()` in a `try/finally`**. Frameworks (Spring's `RequestContextFilter`, Logback's filter) do this for you. Custom code doesn't get the same protection.

Modern fix: **don't use ThreadLocal for new code**. Use [`ScopedValue`](#scopedvalue--the-modern-replacement) or Reactor Context instead.

---

## InheritableThreadLocal

`InheritableThreadLocal<T>` copies parent thread's value to child threads at `Thread` creation time:

```java
private static final InheritableThreadLocal<String> TRACE_ID = new InheritableThreadLocal<>();

TRACE_ID.set("abc");
new Thread(() -> log.info(TRACE_ID.get())).start();   // prints "abc"
```

Caveats:
- Only copies **at thread creation**. Doesn't propagate to already-running threads.
- Only for `new Thread(...)`. Does NOT work with thread pools — the child thread was created when the pool was instantiated, not when the parent kicked off a task.
- Shares the leak risk of regular ThreadLocal.

In practice, InheritableThreadLocal solves the "spawn a child thread for this work" case but fails the "submit to an executor" case — which is the overwhelming majority. Usually superseded by more explicit context-passing.

---

## Where ThreadLocal Shows Up in Spring

Virtually every request-scoped Spring API is ThreadLocal-backed:

- `SecurityContextHolder` — the authenticated principal for the current request.
- `RequestContextHolder` — the current `HttpServletRequest`.
- `LocaleContextHolder` — locale for i18n.
- `TransactionSynchronizationManager` — the current transaction (JDBC connection, entity manager).
- Logback / Log4j2 MDC (`MDC.put("traceId", ...)`).

All suffer the same constraints:
- Must be cleaned up per request (Spring handles this via filters).
- Don't propagate across `@Async` or `ExecutorService` boundaries by default.
- Break on reactive (WebFlux) because requests hop threads.

Spring's `MODE_INHERITABLETHREADLOCAL` for `SecurityContextHolder` attempts to propagate to child threads — partial fix, pool-blind.

See [logging.md](../../logging.md) for MDC specifically.

---

## Virtual Threads Break the ThreadLocal Assumption

Each virtual thread has its own independent ThreadLocal storage. With 1M concurrent VTs, 10 ThreadLocal values per VT = 10M map entries, potentially GB of memory for nothing.

```java
// Anti-pattern at VT scale: every VT gets a heavy cached object
private static final ThreadLocal<SimpleDateFormat> DATE_FMT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
```

With platform threads (hundreds), this cache was free. With virtual threads (millions), each VT allocates its own `SimpleDateFormat` — that's millions of formatter instances.

Fix:
- Swap `SimpleDateFormat` for `DateTimeFormatter` (which is thread-safe, no TL needed).
- Use `ScopedValue` for request-scoped context (immutable, no per-VT state beyond the scope).

The general lesson: with VTs, **reconsider every ThreadLocal**. If it's caching a thread-safe value, drop it. If it's holding request state, use ScopedValue.

---

## ScopedValue — The Modern Replacement

[JEP 506](https://openjdk.org/jeps/506) finalized `ScopedValue` in Java 25 (after several preview rounds). It replaces ThreadLocal for the common case: pass immutable context down a call chain.

```java
private static final ScopedValue<String> TRACE_ID = ScopedValue.newInstance();

// Establish binding for a bounded scope
ScopedValue.where(TRACE_ID, "abc").run(() -> {
    processRequest();   // TRACE_ID.get() returns "abc" in this stack
});

// Read — anywhere in the call stack, from any method
String id = TRACE_ID.get();   // "abc" inside the scope, throws outside
```

Properties:
- **Immutable** — cannot be reassigned inside the scope.
- **Bounded lifetime** — cleared when the scope exits; no leak possible.
- **Cheap** — no per-thread map; uses a stack-walking lookup. O(1) for small scope depths.
- **VT-friendly** — no memory growth per VT.
- **Inherits in structured concurrency** — subtasks forked inside a scope see the value automatically.

Compare to ThreadLocal:

| | `ThreadLocal` | `ScopedValue` |
|---|---|---|
| Lifetime | Thread (pool reuses!) | Explicit scope |
| Mutability | Settable anywhere | Bound once, immutable |
| Cleanup | Manual `remove()` required | Automatic |
| VT memory | Per-VT map entry | None |
| Inheritance to child threads | `InheritableThreadLocal` only, limited | Automatic with `StructuredTaskScope` |

Migration rule: any ThreadLocal where you can bound the lifetime (request handler, job scope, transaction) → `ScopedValue`. Where you can't (framework-internal caches tied to thread lifetime) → reconsider whether you need TL at all.

See [structured-concurrency.md](structured-concurrency.md) for the broader story and [modern-java-features.md § Java 25](../modern-java-features.md#scoped-values-jep-506).

---

## TransmittableThreadLocal — Pool-Crossing TL

Alibaba's [TransmittableThreadLocal](https://github.com/alibaba/transmittable-thread-local) extends `InheritableThreadLocal` to propagate across thread-pool boundaries using a Java agent / executor wrapper:

```java
TransmittableThreadLocal<String> TRACE = new TransmittableThreadLocal<>();
TRACE.set("abc");

ExecutorService wrapped = TtlExecutors.getTtlExecutorService(realExecutor);
wrapped.submit(() -> TRACE.get());   // sees "abc" even in pool thread
```

Relevant when:
- You have extensive legacy ThreadLocal usage you can't migrate.
- You need to preserve context across `ExecutorService.submit()`.
- You can't upgrade to Java 21+ / ScopedValue.

On modern stacks, prefer ScopedValue (if VT-available) or explicit context passing.

---

## Reactor Context

Reactor's `Context` is the reactive counterpart — travels with the subscription, not the thread:

```java
Mono<Response> mono = webClient.get().uri("/api").retrieve().bodyToMono(Response.class)
    .flatMap(this::enrich);

// Inject context at subscribe time
mono.contextWrite(Context.of("traceId", "abc")).subscribe();

// Read inside the pipeline
Mono<String> fetch = Mono.deferContextual(ctx ->
    Mono.just(ctx.get("traceId"))
);
```

Key properties:
- Immutable — `contextWrite` returns a new Mono.
- Flows upstream in the subscription chain (downstream reads propagate up).
- Survives operator thread hops (`publishOn`, `subscribeOn`).
- Not ThreadLocal — different subscribers have different contexts on the same thread.

Why it exists: ThreadLocal is wrong for reactive. A reactive pipeline hops threads; ThreadLocal wouldn't follow. Context is tied to the subscription, which is the "logical request" in reactive land.

**Bridging TL and Context**: Spring's [Context Propagation](https://github.com/micrometer-metrics/context-propagation) library copies TL into Context at subscribe and back at `publishOn` boundaries. Used by Micrometer Tracing, Spring Security, Logback MDC in WebFlux.

See [reactive-programming-java.md](../../reactive-programming-java.md), [reactive-advanced-topics.md](../../reactive-advanced-topics.md), and [reactive-observability.md](../../reactive-observability.md).

---

## OpenTelemetry Context

OTel's [`io.opentelemetry.context.Context`](https://opentelemetry.io/docs/specs/otel/context/) is the standardized cross-library propagation mechanism:

```java
Span span = tracer.spanBuilder("op").startSpan();
try (Scope scope = span.makeCurrent()) {
    // In this scope, Context.current() has the span
    runWork();
} finally {
    span.end();
}
```

Under the hood, `Context.makeCurrent()` sets a ThreadLocal. But the API is context-propagation-agnostic — the OTel instrumentation libraries bridge it to Reactor Context, gRPC Metadata, HTTP headers (W3C Trace Context), etc.

Rule: if you're emitting traces, use OTel Context. For application-level context that doesn't cross the wire, use ScopedValue or Reactor Context directly.

See [distributed-tracing.md](../../observability/distributed-tracing.md).

---

## Migration Guide

| Current | Target | Notes |
|---------|--------|-------|
| `ThreadLocal` for request-scoped context in servlet app | `ScopedValue` (JDK 21+ pref, JDK 25+ LTS) | Set in filter, bound to request processing scope. |
| `ThreadLocal` for DateTimeFormatter / cache | Plain `static final DateTimeFormatter` | It's immutable and thread-safe. |
| `InheritableThreadLocal` for child threads | `ScopedValue` in structured concurrency | Auto-propagates. |
| `ThreadLocal` propagated via TtlExecutor | `ScopedValue` + structured concurrency | Cleaner. |
| `ThreadLocal` in reactive pipeline | Reactor `Context` | TL doesn't survive thread hops. |
| `MDC.put` in blocking code | Keep (Logback integrates with MDC everywhere) | Bridged to Reactor via Context Propagation. |
| `MDC.put` in WebFlux | Reactor `Context` + Context Propagation bridge | Already set up by Spring 6 / Boot 3. |
| Spring `SecurityContextHolder` TL | Keep; wire Context Propagation for reactive | Spring handles it. |

Bottom line: for new code, default to ScopedValue in blocking paths and Reactor Context in reactive paths. Treat ThreadLocal as legacy.

---

## Related

- [Concurrency Basics](concurrency-basics.md) — `ThreadLocal` introduced.
- [Multithreading Deep Dive](multithreading-deep-dive.md) — JMM and thread storage.
- [Virtual Threads](virtual-threads.md) — why ThreadLocal at VT scale is a memory issue.
- [Structured Concurrency](structured-concurrency.md) — `StructuredTaskScope` + `ScopedValue` together.
- [Modern Java Features](../modern-java-features.md) — Java 25 finalized ScopedValue.
- [Logging](../../logging.md) — MDC, the most common ThreadLocal in production code.
- [Reactive Programming in Java](../../reactive-programming-java.md) — Reactor Context basics.
- [Advanced Reactive Programming](../../reactive-advanced-topics.md) — Context deep dive.
- [Reactive Observability](../../reactive-observability.md) — Context Propagation for MDC + traces.
- [Distributed Tracing](../../observability/distributed-tracing.md) — OTel Context.

---

## References

- [`ThreadLocal` Javadoc (JDK 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ThreadLocal.html)
- [`ScopedValue` Javadoc (JDK 21+ preview, JDK 25 final)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ScopedValue.html)
- [JEP 506: Scoped Values (Java 25 final)](https://openjdk.org/jeps/506)
- [Reactor Context documentation](https://projectreactor.io/docs/core/release/reference/#context)
- [Micrometer Context Propagation](https://github.com/micrometer-metrics/context-propagation)
- [TransmittableThreadLocal](https://github.com/alibaba/transmittable-thread-local)
- [OpenTelemetry Context spec](https://opentelemetry.io/docs/specs/otel/context/)
- [Brian Goetz et al. — *Java Concurrency in Practice*](https://jcip.net/) — ThreadLocal chapter.
- [Spring — Context Propagation in Reactive](https://docs.spring.io/spring-framework/reference/core/spring-core.html#core.context-propagation)
