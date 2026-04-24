---
title: "Structured Concurrency Before Project Loom — Pre-Java-21 Approaches and Cross-Language Precedents"
date: 2026-04-17
updated: 2026-04-24
tags: [java, concurrency, structured-concurrency, history, cross-language]
---

# Structured Concurrency Before Project Loom — Pre-Java-21 Approaches and Cross-Language Precedents

**Date:** 2026-04-17 | **Updated:** 2026-04-24
**Tags:** `java` `concurrency` `structured-concurrency` `history` `cross-language`

## Table of Contents

- [Summary](#summary)
- [Why This History Matters](#why-this-history-matters)
- [The Origin: "Go Statement Considered Harmful" (2018)](#the-origin-go-statement-considered-harmful-2018)
- [Python's Trio and the Nursery Pattern](#pythons-trio-and-the-nursery-pattern)
- [Kotlin Coroutines and CoroutineScope](#kotlin-coroutines-and-coroutinescope)
- [Swift's TaskGroup and async let](#swifts-taskgroup-and-async-let)
- [Go — The Counter-Example](#go--the-counter-example)
- [JavaScript and TypeScript — AbortController as a Partial Answer](#javascript-and-typescript--abortcontroller-as-a-partial-answer)
- [Java Pre-Loom Approaches](#java-pre-loom-approaches)
  - [Manual ExecutorService + Future + Tracking](#manual-executorservice--future--tracking)
  - [Quasar Fibers](#quasar-fibers)
  - [Project Reactor — Disposable as a Scope](#project-reactor--disposable-as-a-scope)
  - [Vavr Future Composition](#vavr-future-composition)
  - [Kotlin Coroutines Running on the JVM](#kotlin-coroutines-running-on-the-jvm)
- [Why Java Was Late](#why-java-was-late)
- [What JEP 453 Borrowed from Each Precedent](#what-jep-453-borrowed-from-each-precedent)
- [Side-by-Side Comparison](#side-by-side-comparison)
- [Related](#related)
- [References](#references)

---

## Summary

Before Java 21 brought [`StructuredTaskScope`](structured-concurrency.md), "structured concurrency" was a concept pioneered elsewhere: Nathaniel J. Smith coined the term and built [Trio](https://trio.readthedocs.io/) in Python in 2017, Kotlin shipped it as the default behavior of coroutines in 2018, and Swift followed in 2021. Java developers who needed the same discipline had to compose it manually on top of `ExecutorService` / `CompletableFuture`, or reach for third-party tools like [Quasar](https://docs.paralleluniverse.co/quasar/) or [Project Reactor](https://projectreactor.io/). This doc traces that history so you understand *why* [JEP 453](https://openjdk.org/jeps/453) looks the way it does — almost every feature of the Java API has a direct ancestor in one of these systems.

---

## Why This History Matters

Reading [`StructuredTaskScope`](structured-concurrency.md) in isolation, it can feel like Java invented the idea. It didn't. Every design decision in the JEP — lexical lifetimes, fork/join-as-unit, cooperative cancellation, parent/child hierarchy — was prototyped in another language or library first, often five or more years earlier. Understanding the lineage helps you:

- Recognize equivalent patterns when you jump between Java, Kotlin, Swift, or Python.
- Understand *why* Java chose the shapes it did (factory methods in [JEP 505](https://openjdk.org/jeps/505), `Subtask<T>` as an observable handle, `ShutdownOnFailure`/`ShutdownOnSuccess` policies).
- Know when to use an existing library instead of the preview API — if you're on Java 11/17 and can't wait for Java 21+, you have options.

---

## The Origin: "Go Statement Considered Harmful" (2018)

The term "structured concurrency" was popularized by [Nathaniel J. Smith's 2018 essay "Notes on structured concurrency, or: Go statement considered harmful"](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/). The piece argued that `go` / `spawn` / "launch a goroutine or thread and forget about it" is as dangerous as the classic `GOTO` statement — it creates control-flow that exits the enclosing function without a way to observe completion or propagate errors.

The manifesto's core claim:

> Control flow in a single thread has a beautiful structure where every function call eventually returns. Concurrency should preserve that structure — when a function returns, all the tasks it spawned should be finished.

Smith's alternative is the **nursery**: a block that owns a set of child tasks. You can spawn children into a nursery, but the nursery itself refuses to exit until all its children have exited. Exceptions in children propagate to the nursery, which can cancel siblings. If you've squinted at Java's `try (var scope = new StructuredTaskScope.ShutdownOnFailure()) { ... }` and thought "this is a really weird try-with-resources," the answer is: it's a nursery dressed in Java syntax.

```python
# The Trio nursery — the canonical example Smith's essay sold
import trio

async def parent():
    async with trio.open_nursery() as nursery:
        nursery.start_soon(fetch_user, user_id)
        nursery.start_soon(fetch_orders, user_id)
    # Execution only reaches here after BOTH children complete.
    # If either raised, both are cancelled and the exception propagates.
```

Smith's essay remains the best single document for understanding *why* structured concurrency matters, independent of any language.

---

## Python's Trio and the Nursery Pattern

Trio, released in 2017, was Smith's reference implementation. It deliberately broke with the existing Python async ecosystem (`asyncio`) to bake structured concurrency in as a core invariant rather than an optional pattern.

```python
async with trio.open_nursery() as nursery:
    nursery.start_soon(task_a)
    nursery.start_soon(task_b)
    # Block here until both finish. If either raises, both are cancelled.
```

Key design decisions that Java eventually copied:

- **The nursery is an object with a lexical lifetime** (`async with` in Python ≈ try-with-resources in Java).
- **Spawn-point is explicit** (`start_soon` in Trio ≈ `fork` in `StructuredTaskScope`).
- **Cancellation is cooperative** and propagates down through the tree automatically.
- **Exceptions propagate up** from child to parent nursery.

The Python community later backported many of these ideas into `asyncio.TaskGroup` (Python 3.11+), though Trio retains the cleanest implementation.

---

## Kotlin Coroutines and CoroutineScope

JetBrains shipped [Kotlin coroutines with structured concurrency as a core feature in 2018](https://elizarov.medium.com/structured-concurrency-722d765aa952), almost five years before Java got it. Roman Elizarov's blog post announcing the change is a direct companion piece to Smith's. Every coroutine in Kotlin *must* run inside a [`CoroutineScope`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/) — there is no top-level `launch` that escapes the tree.

```kotlin
suspend fun loadProfile(userId: String): Profile = coroutineScope {
    val user = async { fetchUser(userId) }
    val orders = async { fetchOrders(userId) }

    Profile(user.await(), orders.await())
    // coroutineScope guarantees both async blocks complete before returning.
    // If either throws, the sibling is cancelled and the exception propagates.
}
```

Kotlin enforces the discipline at the type level: `launch`, `async`, and `withContext` are all extension functions on `CoroutineScope`, so you can't spawn a coroutine without being inside one. The scope also carries a `CoroutineContext` (dispatcher, exception handler, scoped values) that children inherit — this is the direct precedent for Java's `ScopedValue` inheriting across `StructuredTaskScope.fork`.

Cancellation semantics map 1:1 to what `StructuredTaskScope` later adopted: cancel a parent, all children die; one child throws, all siblings are cancelled. Kotlin coroutines running on the JVM remain a viable alternative for structured concurrency today if you don't want to wait for Java's preview API to stabilize.

---

## Swift's TaskGroup and async let

Swift 5.5 (2021) introduced [`async/await` together with structured concurrency from day one](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/). Two primary constructs:

**`async let` — static, known-at-compile-time fan-out:**

```swift
func loadProfile(id: String) async throws -> Profile {
    async let user = fetchUser(id)
    async let orders = fetchOrders(id)

    return try await Profile(user: user, orders: orders)
    // Both async lets finish before the function returns.
}
```

**`TaskGroup` — dynamic, run-time fan-out:**

```swift
func fetchAll(ids: [String]) async throws -> [User] {
    try await withThrowingTaskGroup(of: User.self) { group in
        for id in ids {
            group.addTask { try await fetchUser(id) }
        }
        var users: [User] = []
        for try await user in group {
            users.append(user)
        }
        return users
    }
}
```

`async let` is the direct analogue of "fork two subtasks" in Java, while `TaskGroup` corresponds to Java's general `StructuredTaskScope` — fork in a loop, accumulate results. Swift's design is particularly important because Apple's concurrency proposal [SE-0304](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) explicitly cites Smith's essay and Kotlin's coroutines as prior art.

---

## Go — The Counter-Example

Go's `go func() { ... }()` is exactly the "go statement" Smith's essay considered harmful. Spawning a goroutine doesn't return a handle you can `join` on, doesn't link the goroutine's lifetime to the caller's, and offers no built-in cancellation.

The Go community's workaround is the [`context` package](https://go.dev/blog/context) — you thread a `context.Context` through every function signature so receivers can cooperatively check for cancellation:

```go
func loadProfile(ctx context.Context, id string) (*Profile, error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    userCh := make(chan *User)
    orderCh := make(chan []*Order)
    errCh := make(chan error, 2)

    go func() {
        u, err := fetchUser(ctx, id)
        if err != nil { errCh <- err; return }
        userCh <- u
    }()
    go func() {
        o, err := fetchOrders(ctx, id)
        if err != nil { errCh <- err; return }
        orderCh <- o
    }()

    // Now compose the results… this is where leaks happen
}
```

**Goroutine leaks** — forgotten background goroutines — are one of the most common production bugs in Go code. The language offers no mechanism to prevent them; discipline is the only safeguard. Libraries like [`errgroup`](https://pkg.go.dev/golang.org/x/sync/errgroup) add structured-concurrency-like patterns on top of `context`, but they're not the default.

Go is the cautionary tale that Java, Kotlin, Swift, and Python explicitly designed against. When Java's docs talk about "eliminating thread leaks by construction," this is the leak story they mean.

---

## JavaScript and TypeScript — AbortController as a Partial Answer

JavaScript has no native structured concurrency. The ecosystem has partial building blocks. If the event loop and worker model are not fresh in your head, review [Event Loop Internals](../../typescript/runtime/event-loop-internals.md) and [Worker Threads & Concurrency](../../typescript/runtime/worker-threads.md) alongside this section:

- **`Promise.all([p1, p2])`** — waits for all promises. Rejects on the first failure — but the siblings keep running in the background. No automatic cancellation.
- **`Promise.race([p1, p2])`** — resolves on the first settlement. Same issue: the loser keeps running.
- **`AbortController` / `AbortSignal`** — the closest thing JS has to propagating cancellation. A controller emits an abort signal that consumers can observe and act on.

```ts
const controller = new AbortController();
const signal = controller.signal;

try {
    const [user, orders] = await Promise.all([
        fetchUser(id, { signal }),
        fetchOrders(id, { signal }),
    ]);
    return { user, orders };
} catch (err) {
    controller.abort();  // manually cancel siblings
    throw err;
}
```

This works but requires every API to opt into the `AbortSignal` protocol. Coverage across the ecosystem is patchy. There are proposals (e.g., [`Promise.withResolvers`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/withResolvers), the nascent `AsyncContext`) that move JS closer to structured concurrency, but nothing at the language level yet.

---

## Java Pre-Loom Approaches

Before Java 21, JVM developers had these options — each with trade-offs.

### Manual ExecutorService + Future + Tracking

The DIY approach. Submit to a thread pool, collect futures, handle cancellation yourself:

```java
ExecutorService exec = Executors.newFixedThreadPool(4);
try {
    Future<User> userF = exec.submit(() -> fetchUser(id));
    Future<List<Order>> ordersF = exec.submit(() -> fetchOrders(id));

    User user;
    List<Order> orders;
    try {
        user = userF.get(5, TimeUnit.SECONDS);
    } catch (Exception e) {
        ordersF.cancel(true);   // Remember to cancel sibling
        throw e;
    }
    try {
        orders = ordersF.get(5, TimeUnit.SECONDS);
    } catch (Exception e) {
        // user already completed — nothing to cancel
        throw e;
    }
    return new Profile(user, orders);
} finally {
    exec.shutdown();
}
```

Problems: verbose, easy to forget cancellation on one side, no automatic propagation of child failure to siblings, no lexical scoping. Every experienced Java engineer has debugged a variant of this pattern where a background task outlived its caller.

### Quasar Fibers

[Quasar](https://docs.paralleluniverse.co/quasar/), developed by Ron Pressler (who now leads Project Loom at Oracle), was the most ambitious pre-Loom attempt. Quasar used bytecode instrumentation to implement **fibers** — lightweight green threads that behaved like `Thread` but scaled to millions. It predated Loom by nearly a decade.

```java
@Suspendable
public Profile loadProfile(String id) throws SuspendExecution {
    Fiber<User> userFiber = new Fiber<>(() -> fetchUser(id)).start();
    Fiber<List<Order>> ordersFiber = new Fiber<>(() -> fetchOrders(id)).start();

    return new Profile(userFiber.get(), ordersFiber.get());
}
```

Quasar offered fiber-based primitives that resembled modern virtual threads, but never had the full structured-concurrency surface area. It required the `-javaagent` flag, had compatibility caveats with third-party libraries, and — crucially — became the direct intellectual foundation for Project Loom. Pressler moved from Quasar to Oracle and brought the ideas with him. Quasar is no longer actively maintained; if you were using it, migrate to virtual threads.

### Project Reactor — Disposable as a Scope

[Project Reactor](https://projectreactor.io/) and RxJava gave JVM developers a functional, composable concurrency model years before Loom. Reactor's [`Disposable`](https://projectreactor.io/docs/core/release/api/reactor/core/Disposable.html) is a cancellation handle: when you `.subscribe()` to a `Flux` or `Mono`, you get back a `Disposable` whose `.dispose()` call cancels the subscription chain.

```java
public Mono<Profile> loadProfile(String id) {
    Mono<User> userMono = userRepo.findById(id);
    Mono<List<Order>> ordersMono = orderRepo.findByUser(id);

    return Mono.zip(userMono, ordersMono)
        .map(tuple -> new Profile(tuple.getT1(), tuple.getT2()))
        .timeout(Duration.ofSeconds(5));
}
```

Reactor's `Mono.zip` and `Flux.merge` compose operations structurally — if one upstream cancels, the whole pipeline disposes. [`disposeGracefully()`](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Scheduler.html#disposeGracefully--) adds the graceful-shutdown semantics that `StructuredTaskScope.close()` later adopted. The shape isn't identical — Reactor is pull-based and back-pressured, structured concurrency is imperative — but the lifetime-binding property is shared.

If you're on WebFlux today, you're already using a structured-concurrency-like system. See [this project's reactive programming guide](../reactive-programming-java.md) for the full picture.

### Vavr Future Composition

[Vavr](https://www.vavr.io/) added functional-style futures with richer composition than `CompletableFuture`:

```java
Future<User> userF = Future.of(() -> fetchUser(id));
Future<List<Order>> ordersF = Future.of(() -> fetchOrders(id));

Future<Profile> profileF = userF.flatMap(user ->
    ordersF.map(orders -> new Profile(user, orders)));
```

Vavr's `Future` supports cancellation and is composable via `flatMap`, but shares `CompletableFuture`'s fundamental limitation: no enforced lexical scope for the children's lifetimes. You can leak. Vavr is a productivity win over raw `CompletableFuture` but not a structural solution.

### Kotlin Coroutines Running on the JVM

Perhaps the pragmatic best answer before Java 21: **just use Kotlin coroutines**. You can mix Kotlin and Java in the same Gradle project, and Kotlin's structured concurrency works identically on any JVM that Kotlin supports (Java 8+). Many shops that wanted structured concurrency before Loom simply adopted Kotlin for the service-layer code and left the rest in Java.

---

## Why Java Was Late

Java's delay wasn't intellectual — the ideas were well-understood by 2018. The reasons were ecosystem constraints:

1. **Platform threads are 1:1 with OS threads.** Without virtual threads, creating many child subtasks was prohibitively expensive. Structured concurrency is only practical when spawning tasks is nearly free.
2. **Backward compatibility.** Any new API has to coexist with decades of `ExecutorService`, `CompletableFuture`, and `Thread` code. The JDK team refused to introduce a new concurrency primitive that wasn't compatible with `Thread`.
3. **Project Loom had to ship first.** Virtual threads (JEP 444) and structured concurrency (JEP 453) are intentionally paired. Delivering structured concurrency alone without cheap threads would have been an attractive nuisance.
4. **Preview process.** Java's JEP preview mechanism (introduced in JDK 12) allows years of community feedback before features go final. Structured concurrency has been in preview since JDK 21 (2023) and is still preview as of the JDK 27 plans ([JEP 533](https://openjdk.org/jeps/533)).

The delay produced a better API. `StructuredTaskScope` integrates cleanly with `Thread`, `ThreadLocal`, `InterruptedException`, and existing executor semantics — something none of the earlier Java libraries managed.

---

## What JEP 453 Borrowed from Each Precedent

| Idea | Origin | How it shows up in Java |
|------|--------|-------------------------|
| "Nursery" as a lexically-scoped container | Trio (Python, 2017) | `StructuredTaskScope` opened in try-with-resources |
| Fork/join-as-unit semantics | Trio, Kotlin | `fork()` + `join()` + auto-close |
| Parent/child task hierarchy | Kotlin `CoroutineScope` | Scope owns forked subtasks |
| Cooperative cancellation through parent | Kotlin coroutines, Trio | Scope's `shutdown()` interrupts children |
| Immutable context inheritance | Kotlin's `CoroutineContext`, Swift's task-local values | [`ScopedValue`](https://openjdk.org/jeps/506) |
| Dynamic task group | Swift `TaskGroup` | `StructuredTaskScope` + loop of `fork()` calls |
| Policy-driven shutdown (fail-fast, first-success) | Custom patterns across all | `ShutdownOnFailure`, `ShutdownOnSuccess`, `Joiner` |
| Result handle type | Swift's `Task<T>`, Kotlin's `Deferred<T>` | `Subtask<T>` |
| Virtual threads for cheap forks | Quasar fibers | JEP 444 virtual threads |
| Graceful disposal | Reactor's `disposeGracefully()` | `StructuredTaskScope.close()` timeout behaviour |

Every good idea in `StructuredTaskScope` has a lineage. Java is the integrator, not the inventor — but that integration is where the value lies.

---

## Side-by-Side Comparison

How the same "fetch user + orders, fail if either fails" code looks across languages:

```python
# Python (Trio)
async with trio.open_nursery() as nursery:
    nursery.start_soon(fetch_user, id)
    nursery.start_soon(fetch_orders, id)
```

```kotlin
// Kotlin Coroutines
coroutineScope {
    val user = async { fetchUser(id) }
    val orders = async { fetchOrders(id) }
    Profile(user.await(), orders.await())
}
```

```swift
// Swift async let
async let user = fetchUser(id)
async let orders = fetchOrders(id)
return try await Profile(user: user, orders: orders)
```

```java
// Java 21 StructuredTaskScope
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var userTask = scope.fork(() -> fetchUser(id));
    var ordersTask = scope.fork(() -> fetchOrders(id));
    scope.join().throwIfFailed();
    return new Profile(userTask.get(), ordersTask.get());
}
```

```go
// Go with errgroup (the closest Go gets)
g, ctx := errgroup.WithContext(ctx)
var user *User
var orders []*Order
g.Go(func() error { u, err := fetchUser(ctx, id); user = u; return err })
g.Go(func() error { o, err := fetchOrders(ctx, id); orders = o; return err })
if err := g.Wait(); err != nil { return nil, err }
return &Profile{User: user, Orders: orders}, nil
```

```js
// JavaScript (best approximation — not truly structured)
const controller = new AbortController();
try {
    const [user, orders] = await Promise.all([
        fetchUser(id, { signal: controller.signal }),
        fetchOrders(id, { signal: controller.signal }),
    ]);
    return { user, orders };
} catch (err) {
    controller.abort();
    throw err;
}
```

The semantic core — "start N concurrent tasks bounded by this block; cancel siblings on failure" — is the same across all of them. The syntactic weight varies dramatically. Java's version is the verbosest, but it's also the most explicit about the lifetime boundary (the `try` braces).

---

## Related

- [Structured Concurrency in Java](structured-concurrency.md) — the modern `StructuredTaskScope` API this doc provides context for
- [Virtual Threads in Java](virtual-threads.md) — the runtime primitive that made structured concurrency practical on the JVM
- [Concurrency Basics](concurrency-basics.md) — `ExecutorService`, `CompletableFuture`, and the pre-Loom tools covered in detail
- [Reactive Programming in Java](../reactive-programming-java.md) — Project Reactor's approach, another pre-Loom structured model
- [Modern Java Features](modern-java-features.md) — Java 17–21 features including sealed types and pattern matching that pair with the new concurrency API
- [Event Loop Internals](../../typescript/runtime/event-loop-internals.md) — how promise continuations and callback scheduling actually work in Node.
- [Worker Threads & Concurrency](../../typescript/runtime/worker-threads.md) — Node's real parallelism model, shared memory, and atomics.

## References

- [Notes on structured concurrency, or: Go statement considered harmful — Nathaniel J. Smith (2018)](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/) — the founding essay
- [Structured concurrency — Roman Elizarov (2018)](https://elizarov.medium.com/structured-concurrency-722d765aa952) — Kotlin's announcement and rationale
- [Coroutines basics — Kotlin Documentation](https://kotlinlang.org/docs/coroutines-basics.html) — official Kotlin coroutines guide
- [CoroutineScope — kotlinx.coroutines API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/) — Kotlin scope javadoc
- [Concurrency — The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/) — Swift's concurrency guide
- [SE-0304 Structured Concurrency — Swift Evolution](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md) — Swift's `async let` proposal, cites Smith and Kotlin as prior art
- [Trio — structured concurrency for Python](https://trio.readthedocs.io/) — Smith's reference implementation
- [Go Concurrency Patterns: Context — The Go Programming Language Blog](https://go.dev/blog/context) — Go's context package, the unstructured-concurrency baseline
- [Quasar — lightweight threads and Erlang-like actors for the JVM](https://docs.paralleluniverse.co/quasar/) — historical pre-Loom fibers library
- [Project Reactor Disposable Javadoc](https://projectreactor.io/docs/core/release/api/reactor/core/Disposable.html) — Reactor's cancellation handle, a structured pattern in disguise
- [JEP 453: Structured Concurrency (Preview)](https://openjdk.org/jeps/453) — Java's first stable preview of structured concurrency
- [Project Loom: Understand the new Java concurrency model — InfoWorld](https://www.infoworld.com/article/2334607/project-loom-understand-the-new-java-concurrency-model.html) — background on how Loom unifies virtual threads and structured concurrency
