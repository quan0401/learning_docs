---
title: "CompletableFuture Deep Dive — Composition, Errors, Timeouts, Interop"
date: 2026-04-24
updated: 2026-04-24
tags: [java, concurrency, completablefuture, completionstage, async]
---

# CompletableFuture Deep Dive — Composition, Errors, Timeouts, Interop

**Date:** 2026-04-24 | **Updated:** 2026-04-24
**Tags:** `java` `concurrency` `completablefuture` `completionstage` `async`

## Table of Contents

- [Summary](#summary)
- [CompletionStage vs CompletableFuture](#completionstage-vs-completablefuture)
- [Creation](#creation)
- [Composition — Apply vs Compose vs Combine](#composition--apply-vs-compose-vs-combine)
- [Fan-Out / Fan-In — allOf and anyOf](#fan-out--fan-in--allof-and-anyof)
- [Exception Handling — exceptionally, handle, whenComplete](#exception-handling--exceptionally-handle-whencomplete)
- [Timeouts (Java 9+)](#timeouts-java-9)
- [Async Variants and Executor Selection](#async-variants-and-executor-selection)
- [The Blocking-Inside-a-Chain Trap](#the-blocking-inside-a-chain-trap)
- [Interop with Reactor Mono](#interop-with-reactor-mono)
- [Virtual Threads and CompletableFuture](#virtual-threads-and-completablefuture)
- [Testing CompletableFuture](#testing-completablefuture)
- [Related](#related)
- [References](#references)

---

## Summary

[`CompletableFuture<T>`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html) is Java's general-purpose asynchronous composition API — closer in spirit to a JavaScript `Promise` than to the earlier `Future<T>` interface it extends. It arrived in Java 8 and became the backbone of every async library that isn't reactive. Where JS `Promise` has `.then` / `.catch` / `Promise.all`, Java has `thenApply` / `thenCompose` / `thenCombine` / `exceptionally` / `handle` / `allOf` / `anyOf` — more granular, more type-safe, also more error-prone. This doc covers the parts of the API that matter in practice: composition, error handling, timeouts, executor selection, and interop with Reactor and virtual threads.

---

## CompletionStage vs CompletableFuture

`CompletableFuture` implements two interfaces:

- [`Future<T>`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/Future.html) — the pre-Java-8 "I'll have a value later" interface. `get()`, `cancel()`, `isDone()`. Blocking API.
- [`CompletionStage<T>`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletionStage.html) — the Java 8 "compose me" interface. All the `thenApply`/`thenCompose`/`thenCombine` methods. Non-blocking.

When you return `CompletableFuture<T>` from a library method, **prefer declaring `CompletionStage<T>` as the return type** — it hides the mutating methods (`complete()`, `cancel()`) that callers shouldn't touch.

```java
public CompletionStage<User> loadUser(String id) {           // read-only view
    return CompletableFuture.supplyAsync(() -> ...);
}
```

---

## Creation

```java
// Result known now
CompletableFuture<String> done = CompletableFuture.completedFuture("hi");
CompletableFuture<String> failed = CompletableFuture.failedFuture(new RuntimeException());

// Run on an executor
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> loadFromDb());
CompletableFuture<Void>   cf2 = CompletableFuture.runAsync(() -> sendWebhook());

// Custom executor
CompletableFuture<String> cf3 = CompletableFuture.supplyAsync(() -> loadFromDb(), ioExecutor);

// Manual — you complete it yourself (signal-style)
CompletableFuture<Result> promise = new CompletableFuture<>();
eventBus.on(id, event -> promise.complete(new Result(event)));   // resolve
eventBus.onError(id, err -> promise.completeExceptionally(err)); // reject

// From another CompletionStage
CompletableFuture<T> cf4 = CompletableFuture.fromCompletionStage(stage);
```

The manual-completion pattern (`new CompletableFuture<>()` + `complete()` elsewhere) is the Java equivalent of `new Promise((resolve, reject) => ...)` in JS — extremely useful for bridging callback-style APIs into async composition.

---

## Composition — Apply vs Compose vs Combine

The three composition methods map to functional concepts:

| Method | Signature | Equivalent | Use when |
|--------|-----------|------------|----------|
| `thenApply` | `CF<T> → Function<T,U> → CF<U>` | `map` | The function returns a plain value. |
| `thenCompose` | `CF<T> → Function<T,CF<U>> → CF<U>` | `flatMap` | The function returns another `CompletableFuture` — avoids `CF<CF<U>>`. |
| `thenCombine` | `CF<T> × CF<U> → BiFunction<T,U,V> → CF<V>` | `zip` | Two independent CFs, combine their results. |

```java
CompletableFuture<User> user = loadUser(userId);

// map — function returns plain value
CompletableFuture<String> email = user.thenApply(User::email);

// flatMap — function returns another CF (chained async call)
CompletableFuture<List<Order>> orders = user.thenCompose(u -> loadOrders(u.id()));

// zip — combine two independent CFs
CompletableFuture<Profile> profile = user.thenCombine(loadPrefs(userId), Profile::new);
```

**Rule**: if the callback returns `CompletableFuture`, use `thenCompose`, not `thenApply`. Using `thenApply` gives you `CompletableFuture<CompletableFuture<Orders>>` — usually a bug.

---

## Fan-Out / Fan-In — allOf and anyOf

```java
CompletableFuture<User>    userCf    = loadUser(id);
CompletableFuture<Orders>  ordersCf  = loadOrders(id);
CompletableFuture<Prefs>   prefsCf   = loadPrefs(id);

CompletableFuture<Void> allDone = CompletableFuture.allOf(userCf, ordersCf, prefsCf);

// Join the results (caller decides when)
CompletableFuture<Profile> profile = allDone.thenApply(v ->
    new Profile(userCf.join(), ordersCf.join(), prefsCf.join()));
```

**Traps**:

- `allOf` returns `CompletableFuture<Void>` — not a list of results. You must `join()` each original CF (safe because `allOf` has already completed).
- `allOf` only completes once **all** have completed, success *or* failure. If you want fast-fail on first error, you need more work.
- `anyOf` returns `CompletableFuture<Object>` (not typed). If your CFs all return the same type, cast is safe but ugly. Consider `CompletableFuture.anyOf(cf1, cf2).thenApply(o -> (User) o)`.

Fast-fail fan-out — use `applyToEither` or manual cancellation:

```java
CompletableFuture<User> raceToFail = userCf.applyToEither(ordersCf.thenApply(x -> null),
    x -> { throw new IllegalStateException("any failed"); });
```

In practice, for complex parallel loading, **use Reactor's `Mono.zip`** — the error semantics are better. See [reactive-programming-java.md](../../reactive-programming-java.md).

---

## Exception Handling — exceptionally, handle, whenComplete

Three methods, different behavior:

| Method | Runs on | Can transform? | Suppresses error? |
|--------|---------|----------------|-------------------|
| `exceptionally(fn)` | Error only | Yes — returns replacement value | Yes |
| `handle(fn)` | Always | Yes — sees both success/error | Yes |
| `whenComplete(fn)` | Always | No — returns original | No (side effect only) |

```java
// exceptionally — catch and recover
CompletableFuture<User> user = loadUser(id)
    .exceptionally(ex -> new User("anonymous", ""));

// handle — unified success+error, with transform
CompletableFuture<String> msg = loadUser(id)
    .handle((u, ex) -> ex != null ? "failed: " + ex.getMessage() : "ok: " + u.name());

// whenComplete — side effect only (logging, metrics)
CompletableFuture<User> logged = loadUser(id)
    .whenComplete((u, ex) -> {
        if (ex != null) log.error("user load failed", ex);
        else log.info("loaded user {}", u.id());
    });
```

**The wrapped exception gotcha**: inside these callbacks, a `CompletionException` wraps the original. Unwrap with `ex.getCause()` if you need the underlying type.

```java
.exceptionally(ex -> {
    Throwable cause = ex instanceof CompletionException ? ex.getCause() : ex;
    if (cause instanceof NotFoundException) return User.anonymous();
    throw new CompletionException(cause);  // rethrow if not handled
})
```

---

## Timeouts (Java 9+)

Pre-Java-9, CF had no timeout — you had to roll your own with a `ScheduledExecutorService`. Java 9 added:

```java
loadUser(id)
    .orTimeout(2, TimeUnit.SECONDS)        // completes with TimeoutException
    .exceptionally(ex -> User.anonymous());

loadUser(id)
    .completeOnTimeout(User.anonymous(), 2, TimeUnit.SECONDS);  // completes with fallback
```

`orTimeout` is the right default — combined with `exceptionally`, you get a fail-fast + recover pattern. `completeOnTimeout` is fine when you genuinely want a silent fallback.

**Cleanup caveat**: neither method *cancels* the underlying work — the inner task keeps running. If that work holds resources (DB connection, HTTP client), you'll leak them. For real cancellation, use [structured concurrency](structured-concurrency.md) or [`Future.cancel(true)`](interruption-and-cancellation.md#futurecancelmayinterruptifrunning-and-its-caveats).

---

## Async Variants and Executor Selection

Every composition method has an `Async` variant:

```java
cf.thenApply(fn)                         // runs callback on whatever thread completed the CF
cf.thenApplyAsync(fn)                    // submits callback to ForkJoinPool.commonPool()
cf.thenApplyAsync(fn, myExecutor)        // submits callback to your executor
```

Rules:

- **`thenApply` without `Async`** — the callback runs on whatever thread completed the CF. If that's an I/O pool or a Netty event loop, you might block it.
- **`thenApplyAsync` default** — uses `ForkJoinPool.commonPool()`, which is shared across the JVM. Blocking anything inside a commonPool task starves parallel streams and every other library using the pool.
- **Always specify a custom executor** for any CF doing blocking I/O.

```java
Executor io = Executors.newFixedThreadPool(50, r -> {
    Thread t = new Thread(r, "io-" + t.getId()); t.setDaemon(true); return t;
});

loadUser(id)                             // returns CF on DB pool
    .thenApplyAsync(u -> expensive(u), cpuBoundPool)
    .thenComposeAsync(r -> persist(r), io);
```

With virtual threads (Java 21+), a single `Executors.newVirtualThreadPerTaskExecutor()` obviates most of this — see below.

---

## The Blocking-Inside-a-Chain Trap

The #1 CF bug:

```java
// WRONG — blocks the thread that's supposed to be composing
loadUser(id)
    .thenApply(user -> loadOrders(user.id()).join())  // join() blocks!
    .thenApply(...);
```

`.join()` (or `.get()`) inside a `thenApply` callback blocks the calling thread. If that thread is a small pool (commonPool, WebFlux event loop, Netty), you've just pinned it and may deadlock.

Fix: use `thenCompose` so the result stays asynchronous:

```java
loadUser(id)
    .thenCompose(user -> loadOrders(user.id()))       // no blocking
    .thenApply(...);
```

Scan every `.join()` / `.get()` in your code. If it's inside a callback, it's almost certainly a bug.

---

## Interop with Reactor Mono

Bridge between `CompletableFuture` and Reactor's `Mono`:

```java
// CF → Mono
Mono<User> mono = Mono.fromFuture(loadUser(id));

// Mono → CF
CompletableFuture<User> cf = userMono.toFuture();

// Reactor Mono.fromCompletionStage preserves cancellation better
Mono<User> mono2 = Mono.fromCompletionStage(loadUser(id));
```

`Mono.fromFuture` subscribes once; if the Mono has multiple subscribers, each triggers a separate completion — not usually what you want. For multicasting, use `.cache()`.

See [reactive-programming-java.md](../../reactive-programming-java.md) and [reactive-advanced-topics.md](../../reactive-advanced-topics.md) for Reactor patterns.

---

## Virtual Threads and CompletableFuture

Java 21 [virtual threads](virtual-threads.md) make much of the async-composition ceremony unnecessary. When every thread is cheap:

```java
// With virtual threads, plain blocking code is fine
var executor = Executors.newVirtualThreadPerTaskExecutor();
User user = executor.submit(() -> loadUser(id)).get();
List<Order> orders = executor.submit(() -> loadOrders(user.id())).get();
```

No CF chain needed — the blocking calls don't tie up platform threads. Millions of VTs is fine.

When CF is still the right tool:

- Parallel composition with clean result merging (`thenCombine` reads well).
- Bridging callback-style APIs that don't return `Future`.
- Library APIs that must stay pre-Java-21 compatible.
- Timeout semantics (`orTimeout` is clean).

Rule: on JDK 21+, reach for VT + structured concurrency *first*; reach for CF only when the composition is genuinely multi-stage and parallel.

---

## Testing CompletableFuture

Synchronous tests:

```java
@Test
void loadsUser() {
    User user = loadUser("u1").orTimeout(5, SECONDS).join();
    assertThat(user.email()).isEqualTo("alice@example.com");
}
```

With [Awaitility](https://github.com/awaitility/awaitility) for events:

```java
await().atMost(5, SECONDS).until(cf::isDone);
assertThat(cf.join()).isEqualTo(expected);
```

Reactor's `StepVerifier` works too — convert to Mono first:

```java
StepVerifier.create(Mono.fromFuture(loadUser(id)))
    .expectNextMatches(u -> u.email().contains("@"))
    .verifyComplete();
```

For unit-testing the composition logic, inject a test executor so execution is synchronous:

```java
Executor inline = Runnable::run;
loadUser(id, inline).thenApply(...).thenApply(...).join();  // all on test thread
```

---

## Related

- [Concurrency Basics](concurrency-basics.md) — `ExecutorService`, `Future`, threads.
- [Multithreading Deep Dive](multithreading-deep-dive.md) — `ThreadPoolExecutor`, JMM, happens-before that CF callbacks rely on.
- [Virtual Threads](virtual-threads.md) — the post-Java-21 alternative to deep CF chains.
- [Structured Concurrency](structured-concurrency.md) — `StructuredTaskScope` as the next-gen fan-out primitive.
- [Interruption and Cancellation](interruption-and-cancellation.md) — `Future.cancel(true)` semantics.
- [ForkJoinPool and Parallel Streams](forkjoinpool-and-parallel-streams.md) — why `commonPool` is a shared resource.
- [Reactive Programming in Java](../../reactive-programming-java.md) — Reactor as the alternative async model.
- [Advanced Reactive Programming](../../reactive-advanced-topics.md) — Reactor Context, error operators.
- [Async Processing in Spring](../../events-async/async-processing.md) — `@Async` returning `CompletableFuture<T>`.

---

## References

- [CompletableFuture Javadoc (JDK 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html)
- [CompletionStage Javadoc (JDK 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletionStage.html)
- [JEP 266: More Concurrency Updates (Java 9)](https://openjdk.org/jeps/266) — added `orTimeout`, `completeOnTimeout`, `copy`, `newIncompleteFuture`.
- [Tomasz Nurkiewicz — CompletableFuture tutorial](https://www.nurkiewicz.com/2013/05/java-8-definitive-guide-to.html)
- [Brian Goetz et al. — *Java Concurrency in Practice*](https://jcip.net/) — chapters 5–6 on `Future` fundamentals.
- [Awaitility](https://github.com/awaitility/awaitility)
