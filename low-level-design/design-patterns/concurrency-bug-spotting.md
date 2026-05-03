---
title: "Concurrency Patterns — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, concurrency, threading, low-level-design, java, cpp]
---

# Concurrency Patterns — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `concurrency` `threading` `low-level-design`

---

## Table of Contents
1. [How to use this doc](#how-to-use-this-doc)
2. [Easy (warm-up traps)](#1-easy-warm-up-traps)
3. [Subtle (review-passers)](#2-subtle-review-passers)
4. [Senior trap (production-only failures)](#3-senior-trap-production-only-failures)
5. [Solutions](#4-solutions)
6. [Related](#related)
7. [References](#references)

## Summary
Twenty-two broken concurrency snippets in Java and C++, sectioned by difficulty. Coverage spans publication races, double-checked locking without `volatile`/acquire-release, lock ordering, CAS livelock and ABA, executor lifecycle, memory ordering across the JMM and the C++ memory model, `wait`/`notify` correctness, and pool-thread leaks. Every bug carries a citation to JLS Ch. 17, the C++ memory model on cppreference, the Pugh "Double-Checked Locking is Broken" declaration, *Java Concurrency in Practice* (JCIP), or a Doug Lea note. Practice cadence: one section per session; redo Easy until it stops embarrassing you, then graduate.

## How to use this doc
- Try to spot the bug before opening `<details>`.
- The hint inside `<details>` is one line; the full root-cause and fix sit in §4 Solutions, keyed by bug number.
- Difficulty sections are cumulative — Easy bugs stay easy only after you have caught them in code review.

## 1. Easy (warm-up traps)

### Bug 1 — Shared HashMap
```java
class Counters {
    private final Map<String, Integer> counts = new HashMap<>();
    public void bump(String k) {
        Integer v = counts.get(k);
        counts.put(k, v == null ? 1 : v + 1);
    }
}
// Called from multiple threads.
```
<details><summary>Hint</summary>
HashMap is documented as not thread-safe; what's the read-modify-write story here?
</details>

### Bug 2 — Singleton without volatile
```java
class Holder {
    private static Holder INSTANCE;
    public static Holder get() {
        if (INSTANCE == null) {
            synchronized (Holder.class) {
                if (INSTANCE == null) INSTANCE = new Holder();
            }
        }
        return INSTANCE;
    }
}
```
<details><summary>Hint</summary>
Pugh et al. wrote a whole declaration about exactly this shape.
</details>

### Bug 3 — AtomicInteger misuse
```java
AtomicInteger n = new AtomicInteger();
void hit() { n.set(n.get() + 1); }
```
<details><summary>Hint</summary>
Two operations dressed up as one don't compose into atomicity.
</details>

### Bug 4 — Swallowed InterruptedException
```java
try {
    Thread.sleep(1_000);
} catch (InterruptedException e) {
    // ignore
}
```
<details><summary>Hint</summary>
Goetz devotes a chapter to why doing nothing here is wrong.
</details>

### Bug 5 — synchronized on Boolean
```java
class Gate {
    private Boolean open = Boolean.FALSE;
    public void enter() {
        synchronized (open) { /* critical section */ }
    }
}
```
<details><summary>Hint</summary>
What does `Boolean.FALSE` give you for identity across instances?
</details>

### Bug 6 — ExecutorService never shut down
```java
class Worker {
    private final ExecutorService pool = Executors.newFixedThreadPool(8);
    public void submit(Runnable r) { pool.submit(r); }
}
// No shutdown anywhere.
```
<details><summary>Hint</summary>
Non-daemon threads keep the JVM alive, and pools held in fields rarely die.
</details>

### Bug 7 — wait without while
```java
synchronized (lock) {
    if (!ready) lock.wait();
    consume();
}
```
<details><summary>Hint</summary>
The JLS explicitly permits a wakeup mode that this code does not handle.
</details>

## 2. Subtle (review-passers)

### Bug 8 — putIfAbsent done wrong
```java
ConcurrentMap<String, List<Job>> jobs = new ConcurrentHashMap<>();
void add(String k, Job j) {
    if (!jobs.containsKey(k)) {
        jobs.put(k, new ArrayList<>());
    }
    jobs.get(k).add(j);
}
```
<details><summary>Hint</summary>
Two concurrent threads can both lose the check-then-act race, and the inner list isn't safe either.
</details>

### Bug 9 — volatile long on a 32-bit JVM (or rather, non-volatile)
```java
class Stats {
    long total;            // not volatile
    void add(long n) { total += n; }
    long read() { return total; }
}
```
<details><summary>Hint</summary>
JLS §17.7 has something to say about 64-bit values that aren't `volatile`.
</details>

### Bug 10 — Volatile flag, non-volatile payload
```java
class Config {
    private Map<String, String> values;     // not volatile, not final
    private volatile boolean ready;          // volatile
    void load() {
        values = parse();    // (1)
        ready = true;        // (2)
    }
    String get(String k) {
        if (!ready) return null;
        return values.get(k); // (3)
    }
}
```
<details><summary>Hint</summary>
The volatile write is the *publication* point; everything written before it must hop the same fence.
</details>

### Bug 11 — Lock ordering deadlock
```java
class Account {
    final Object lock = new Object();
    long balance;
    static void transfer(Account from, Account to, long amt) {
        synchronized (from.lock) {
            synchronized (to.lock) {
                from.balance -= amt;
                to.balance += amt;
            }
        }
    }
}
```
<details><summary>Hint</summary>
Two threads, two accounts, two opposite directions.
</details>

### Bug 12 — DCL in C++ without acquire-release
```cpp
Helper* instance = nullptr;
std::mutex m;
Helper* get() {
    if (!instance) {                        // (1)
        std::lock_guard<std::mutex> g(m);
        if (!instance) instance = new Helper();
    }
    return instance;                        // (2)
}
```
<details><summary>Hint</summary>
The unlocked read at (1) and the unsynchronized read at (2) need explicit ordering against the publication.
</details>

### Bug 13 — Future.get with no timeout
```java
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<Result> f = pool.submit(() -> remoteCall());
Result r = f.get();   // unbounded
```
<details><summary>Hint</summary>
A hung remote call now hangs your worker thread forever, and your pool is finite.
</details>

### Bug 14 — Iterating while mutating ConcurrentHashMap and assuming snapshot
```java
ConcurrentHashMap<String, Long> m = ...;
long max = 0;
for (var e : m.entrySet()) {
    max = Math.max(max, e.getValue());
    if (m.size() > 10_000) m.remove(e.getKey());
}
return max;
```
<details><summary>Hint</summary>
The Javadoc word for this iterator is *weakly consistent*, not *snapshot*.
</details>

### Bug 15 — CAS loop with no backoff
```cpp
std::atomic<int> ctr{0};
void hit() {
    int cur = ctr.load(std::memory_order_relaxed);
    while (!ctr.compare_exchange_weak(cur, cur + 1,
                                      std::memory_order_relaxed)) {
        // tight retry under contention
    }
}
```
<details><summary>Hint</summary>
Many cores, one cache line — what does the bus look like?
</details>

### Bug 16 — Reentrant call into a different lock
```java
class Inventory {
    private final Object lock = new Object();
    private OrderService orders;
    public void receive(Item i) {
        synchronized (lock) {
            orders.notifyArrival(i);    // acquires its own lock
        }
    }
}
class OrderService {
    private final Object lock = new Object();
    public void notifyArrival(Item i) {
        synchronized (lock) { ... }
    }
    public void cancel(Order o) {
        synchronized (lock) {
            inventory.releaseHold(o);   // takes Inventory.lock
        }
    }
}
```
<details><summary>Hint</summary>
Two lock domains, reentered in opposite orders from two callers.
</details>

### Bug 17 — Non-final field read during construction leak
```java
class Listener {
    int threshold;
    Listener(EventBus bus) {
        bus.register(this);   // (1) leak before construction completes
        this.threshold = 10;  // (2)
    }
    public void onEvent(Event e) {
        if (e.size() >= threshold) ...   // can see 0
    }
}
```
<details><summary>Hint</summary>
JCIP §3.2 calls this "this-escape"; JLS §17.5 only protects `final` fields.
</details>

## 3. Senior trap (production-only failures)

### Bug 18 — ABA in lock-free stack
```cpp
struct Node { int v; Node* next; };
std::atomic<Node*> head{nullptr};

void push(int v) {
    auto n = new Node{v, head.load()};
    while (!head.compare_exchange_weak(n->next, n)) {}
}
int pop() {
    Node* h;
    do {
        h = head.load();
        if (!h) return -1;
    } while (!head.compare_exchange_weak(h, h->next));
    int v = h->v;
    delete h;
    return v;
}
```
<details><summary>Hint</summary>
Thread A reads `h` and `h->next`, sleeps. Thread B pops two and pushes the original head back.
</details>

### Bug 19 — ForkJoinPool.commonPool() for blocking I/O
```java
List<CompletableFuture<Resp>> fs = urls.stream()
    .map(u -> CompletableFuture.supplyAsync(() -> httpBlockingGet(u)))
    .toList();
fs.forEach(CompletableFuture::join);
```
<details><summary>Hint</summary>
Default executor is sized to CPU count and is shared with parallel streams across the JVM.
</details>

### Bug 20 — ThreadLocal leak in pool worker
```java
class TenantContext {
    private static final ThreadLocal<Tenant> CURRENT = new ThreadLocal<>();
    public static void set(Tenant t) { CURRENT.set(t); }
    public static Tenant get() { return CURRENT.get(); }
}
// Servlet filter calls set(...) but never remove(); request is handled on a pool thread.
```
<details><summary>Hint</summary>
Pool threads outlive requests; the next request on the same thread inherits state and pins classloaders.
</details>

### Bug 21 — Unbounded queue in ThreadPoolExecutor
```java
ExecutorService pool = new ThreadPoolExecutor(
    4, 4, 0L, TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<>()    // unbounded by default
);
// Producer outpaces consumer.
```
<details><summary>Hint</summary>
Where do unprocessed tasks accumulate, and what's the upper bound?
</details>

### Bug 22 — notify vs notifyAll losing wakeups
```java
class BoundedBuffer<T> {
    private final Queue<T> q = new ArrayDeque<>();
    private final int cap;
    public BoundedBuffer(int cap) { this.cap = cap; }
    public synchronized void put(T x) throws InterruptedException {
        while (q.size() == cap) wait();
        q.add(x);
        notify();   // wakes one
    }
    public synchronized T take() throws InterruptedException {
        while (q.isEmpty()) wait();
        T x = q.remove();
        notify();   // wakes one
        return x;
    }
}
```
<details><summary>Hint</summary>
Producers and consumers wait on the same monitor — a producer can wake another producer.
</details>

---

## 4. Solutions

### Bug 1 — Shared HashMap
**Root cause:** `HashMap` is not thread-safe; concurrent `put` can corrupt the internal table (historic Java 7 infinite-loop bug; in Java 8 you still get lost updates and `ConcurrentModificationException`). The `get`/`put` pair is also a non-atomic read-modify-write.
**Fix:**
```java
private final ConcurrentMap<String, AtomicLong> counts = new ConcurrentHashMap<>();
public void bump(String k) {
    counts.computeIfAbsent(k, __ -> new AtomicLong()).incrementAndGet();
}
```
**Reference:** Goetz, *Java Concurrency in Practice*, §4.4 (Client-side locking and `ConcurrentHashMap`); `java.util.HashMap` Javadoc — "If multiple threads access a hash map concurrently … it must be synchronized externally."

### Bug 2 — Singleton without volatile
**Root cause:** Without `volatile`, the write to `INSTANCE` and the writes initializing the new `Holder` can be reordered (or hoisted out of `synchronized`), so another thread sees a non-null reference to a partially constructed object.
**Fix:**
```java
private static volatile Holder INSTANCE;   // post-JSR-133 fix
```
Or use the lazy holder idiom (`static class Lazy { static final Holder I = new Holder(); }`).
**Reference:** Pugh et al., "The 'Double-Checked Locking is Broken' Declaration" (cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html); JLS §17.4.

### Bug 3 — AtomicInteger misuse
**Root cause:** `set(get()+1)` is a non-atomic read-modify-write — two threads can each read N and both write N+1. Using `AtomicInteger` here adds memory cost with no atomicity benefit.
**Fix:** `n.incrementAndGet();`
**Reference:** `java.util.concurrent.atomic` package Javadoc; JCIP §15.3 (Atomic variable classes).

### Bug 4 — Swallowed InterruptedException
**Root cause:** Eating the exception clears the interrupt status, so callers and library code (e.g., `Future`, `BlockingQueue`) cannot observe cancellation. Threads stuck in shutdown become "hung worker" alerts.
**Fix:**
```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // restore
    throw new MyAbortedException(e);      // or propagate
}
```
**Reference:** Goetz, JCIP §7.1.3 ("Responding to interruption"); `Thread.interrupt` Javadoc.

### Bug 5 — synchronized on Boolean
**Root cause:** `Boolean.FALSE` is the same JVM-wide singleton across all `Gate` instances, so unrelated code synchronizing on the same constant blocks each other. Same hazard for boxed `Integer` cache (`-128..127`) and interned strings.
**Fix:** `private final Object lock = new Object();` and `synchronized (lock) { ... }`.
**Reference:** JLS §5.1.7 (Boxing — cached values); JCIP §2.4 ("Guarding state with locks").

### Bug 6 — ExecutorService never shut down
**Root cause:** `Executors.newFixedThreadPool` creates non-daemon threads by default. Without `shutdown()`/`shutdownNow()`, threads keep the JVM alive on exit and `Worker` instances pin the pool through their fields, leaking threads on each construction.
**Fix:** Call `pool.shutdown()` in a lifecycle hook (`close()`, `@PreDestroy`, JVM shutdown hook), or inject a managed executor.
**Reference:** `ExecutorService` Javadoc; JCIP §7.2 ("Stopping a thread-based service").

### Bug 7 — wait without while
**Root cause:** `Object.wait()` may return without a matching `notify` — spurious wakeups are explicitly permitted. An `if` guard lets the thread proceed with `ready == false`.
**Fix:**
```java
synchronized (lock) {
    while (!ready) lock.wait();
    consume();
}
```
**Reference:** `Object.wait()` Javadoc — "A thread can also wake up without being notified, interrupted, or timing out, a so-called *spurious wakeup*"; JLS §17.2.

### Bug 8 — putIfAbsent done wrong
**Root cause:** Two threads can both see `containsKey == false` and both `put` a fresh list, losing one. Even if the list is created once, `ArrayList.add` from multiple threads is unsafe.
**Fix:**
```java
ConcurrentMap<String, Queue<Job>> jobs = new ConcurrentHashMap<>();
jobs.computeIfAbsent(k, __ -> new ConcurrentLinkedQueue<>()).add(j);
```
**Reference:** `ConcurrentHashMap.computeIfAbsent` Javadoc ("the entire method invocation is performed atomically"); JCIP §5.6.

### Bug 9 — non-volatile long
**Root cause:** Per JLS §17.7, a write to a non-`volatile` `long` or `double` may be split into two 32-bit writes, allowing a reader to observe a torn value composed of halves from two different writes. Independently, `total += n` is not atomic.
**Fix:** Use `LongAdder` or `AtomicLong`; `volatile long` would fix tearing but not the read-modify-write.
**Reference:** JLS §17.7 ("Non-Atomic Treatment of `double` and `long`"); JCIP §3.1.2.

### Bug 10 — Volatile flag, non-volatile payload (partial publication)
**Root cause:** Looks fine post-JSR-133 because the volatile write at (2) provides a release that publishes the prior write at (1). The hazard hides if a future refactor *reads* `values` in another path without first reading `ready` — the happens-before chain only kicks in via the volatile read. Worse, `parse()` may return a `HashMap` that another thread later mutates: publication is safe, but the object is not safe for concurrent use.
**Fix:** Make `values` `final` and assign once via the constructor, *or* use `Map.copyOf(parse())` and assign through a single volatile field, *or* use an `AtomicReference<Map<…>>`. Document that all reads must funnel through the volatile gate.
**Reference:** JSR-133 / JLS §17.4.4 (synchronization order, volatile actions); Goetz, JCIP §3.5 ("Safe publication") — "An improperly published object can appear to change its value spontaneously."

### Bug 11 — Lock ordering deadlock
**Root cause:** `transfer(A, B)` and `transfer(B, A)` running concurrently each grab one lock and wait on the other.
**Fix:** Impose a global lock order, e.g., by `System.identityHashCode` or a stable account id; tie-break with a third lock for hash collisions.
```java
Object first = System.identityHashCode(from) < System.identityHashCode(to) ? from.lock : to.lock;
Object second = first == from.lock ? to.lock : from.lock;
synchronized (first) { synchronized (second) { ... } }
```
**Reference:** Goetz, JCIP §10.1.2 ("Dynamic lock order deadlocks").

### Bug 12 — DCL in C++ without acquire-release
**Root cause:** Without `std::atomic`/memory ordering, the unlocked read of `instance` is a data race (UB), and even if not, the compiler/CPU may reorder the store of `instance` before the `Helper` constructor's writes.
**Fix:**
```cpp
std::atomic<Helper*> instance{nullptr};
std::mutex m;
Helper* get() {
    Helper* p = instance.load(std::memory_order_acquire);
    if (!p) {
        std::lock_guard<std::mutex> g(m);
        p = instance.load(std::memory_order_relaxed);
        if (!p) {
            p = new Helper();
            instance.store(p, std::memory_order_release);
        }
    }
    return p;
}
```
Or just use a function-local `static` (C++11 guarantees thread-safe initialization, [stmt.dcl]/4).
**Reference:** cppreference, *std::memory_order* (en.cppreference.com/w/cpp/atomic/memory_order); C++ standard [stmt.dcl]/4 ("Magic statics"); Meyers & Alexandrescu, "C++ and the Perils of Double-Checked Locking" (DDJ 2004).

### Bug 13 — Future.get with no timeout
**Root cause:** A blocking `get()` ties up the calling thread indefinitely; under load every pool worker can wedge on a downstream that never replies. Cascading hang.
**Fix:**
```java
try {
    Result r = f.get(2, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    f.cancel(true);
    throw new MyTimeoutException(e);
}
```
Pair with bounded queues, downstream timeouts, and circuit breakers.
**Reference:** `Future.get(long, TimeUnit)` Javadoc; JCIP §6.3.7 ("Placing time limits on tasks").

### Bug 14 — Weakly consistent iterator
**Root cause:** `ConcurrentHashMap`'s iterators reflect "the state of the hash table at some point at or since the creation of the iterator" — they do not throw `ConcurrentModificationException`, but they may or may not see updates made during iteration. `size()` is similarly an estimate; pruning during iteration is a logic bug, not safe pruning.
**Fix:** Snapshot if you need a consistent view (`new HashMap<>(m).entrySet()`), or use bulk operations like `m.reduceValues(...)` and `m.forEach(...)`.
**Reference:** `ConcurrentHashMap` Javadoc — "Iterators … are designed to be used by only one thread at a time" and "Bulk operations may complete abruptly … Bulk operations do not guarantee atomicity"; Doug Lea, *Concurrent Programming in Java*, §2.4.

### Bug 15 — CAS loop with no backoff
**Root cause:** Under contention, every core retries `compare_exchange_weak` in a tight loop, ping-ponging the cache line and starving forward progress (livelock-flavored throughput collapse). Especially bad on hyperthreaded cores sharing an L1.
**Fix:** Use `fetch_add(1, std::memory_order_relaxed)` for a counter (no CAS needed); for genuine CAS loops, add exponential backoff with `_mm_pause()` / `std::this_thread::yield()`, or partition the contention (e.g., per-thread counters).
**Reference:** cppreference, `std::atomic::compare_exchange_weak`; Herb Sutter, "Lock-Free Code: A False Sense of Security" / "Effective Concurrency" series (drdobbs.com / herbsutter.com).

### Bug 16 — Reentrant call into a different lock
**Root cause:** Two lock domains acquired in opposite orders by two call paths produce a classic deadlock; calling out to other locked services while holding a lock is the antipattern. JCIP calls these "alien methods".
**Fix:** Release locks before calling out; pass immutable snapshots; or invert the call so one side publishes events through a queue (actor / mailbox pattern).
**Reference:** Goetz, JCIP §10.1.3 ("Deadlocks between cooperating objects") and §2.4 ("Don't invoke alien methods with a lock held").

### Bug 17 — `this` escape during construction
**Root cause:** Registering `this` before the constructor finishes lets another thread call `onEvent` and observe `threshold == 0`. JLS §17.5 only guarantees safe publication for `final` fields when the reference *does not escape* during construction.
**Fix:** Use a static factory: construct fully, then publish.
```java
static Listener create(EventBus bus) {
    Listener l = new Listener(10);
    bus.register(l);
    return l;
}
```
And mark `threshold` `final`.
**Reference:** JLS §17.5 ("`final` Field Semantics"); JCIP §3.2 ("Publication and escape — Safe construction practices").

### Bug 18 — ABA in lock-free stack
**Root cause:** Thread A reads `h = N1, h->next = N2`. Thread B pops N1 then N2, pushes N1 back (perhaps from a free list / allocator reuse). A's CAS succeeds because `head == N1`, but the stack is now corrupt — `N2` has been freed, and A sets `head = N2`.
**Fix:** Use a tagged pointer / version counter (`std::atomic<TaggedPtr>` where each push bumps a 16-bit tag), or hazard pointers, or epoch-based reclamation. On x86_64 with 128-bit CAS (`cmpxchg16b`), a `(ptr, counter)` pair works.
**Reference:** Maged Michael, "Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects" (IEEE TPDS 2004); Doug Lea, "Concurrent Programming in Java" notes on ABA (gee.cs.oswego.edu/dl/).

### Bug 19 — `commonPool()` for blocking I/O
**Root cause:** `CompletableFuture.supplyAsync` with no executor uses `ForkJoinPool.commonPool()`, which defaults to `Runtime.availableProcessors() - 1` threads and is shared with `parallelStream()` and other library code. A handful of blocking HTTPs starves the entire JVM's parallel work.
**Fix:** Pass a dedicated bounded executor sized for I/O concurrency:
```java
ExecutorService io = Executors.newFixedThreadPool(64);
CompletableFuture.supplyAsync(() -> httpBlockingGet(u), io);
```
On Java 21+, prefer virtual threads (`Executors.newVirtualThreadPerTaskExecutor()`).
**Reference:** `ForkJoinPool.commonPool()` Javadoc; Doug Lea, "A Java Fork/Join Framework" (gee.cs.oswego.edu/dl/papers/fj.pdf); JEP 444 (Virtual Threads).

### Bug 20 — ThreadLocal leak in pool worker
**Root cause:** Pool threads are reused across requests. A `ThreadLocal` set on request N and never `remove()`d is visible on request N+1 — security disaster (cross-tenant data) and memory leak: the ThreadLocal map holds the value strongly, pinning the value's classloader for the life of the thread.
**Fix:**
```java
try {
    TenantContext.set(t);
    handle(req);
} finally {
    CURRENT.remove();
}
```
Servlet/Spring filters: clear in `finally` of the filter chain. For request-scoped state on virtual threads, prefer `ScopedValue` (Java 21 preview / 22+).
**Reference:** `ThreadLocal` Javadoc; Goetz, JCIP §3.3.3 ("ThreadLocal"); JEP 446 (Scoped Values).

### Bug 21 — Unbounded queue in ThreadPoolExecutor
**Root cause:** `LinkedBlockingQueue` with no capacity is unbounded; once producers outpace consumers, the queue grows until OOM. The pool's `maximumPoolSize` is also ignored because `LinkedBlockingQueue.offer` always succeeds, so the executor never grows past `corePoolSize`.
**Fix:** Bound the queue and pick a rejection policy.
```java
new ThreadPoolExecutor(4, 16, 30, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(1024),
    new ThreadPoolExecutor.CallerRunsPolicy());
```
`CallerRunsPolicy` produces natural backpressure on submitters.
**Reference:** `ThreadPoolExecutor` Javadoc ("Queuing — three general strategies … Unbounded queues"); JCIP §8.3.

### Bug 22 — `notify` vs `notifyAll`
**Root cause:** A producer waiting on full and a consumer waiting on empty share the same monitor. `notify()` wakes one *arbitrary* waiter — possibly another producer that goes right back to waiting, while the consumer that should have been woken keeps sleeping (lost wakeup). Result: stalls or full deadlock.
**Fix:** Use `notifyAll()` on the shared monitor, or split into two `Condition`s on a `ReentrantLock`:
```java
final ReentrantLock lk = new ReentrantLock();
final Condition notFull = lk.newCondition();
final Condition notEmpty = lk.newCondition();
// put: while(full) notFull.await(); ...; notEmpty.signal();
// take: while(empty) notEmpty.await(); ...; notFull.signal();
```
**Reference:** `Object.notify()` Javadoc; Goetz, JCIP §14.2.2 ("Condition queues — using `notifyAll`") and §14.3 (`Condition`).

---

## Related
- [Concurrency Patterns Survey — Read-Write Lock, Active Object, Monitor Object, DCL, Reactor](additional/concurrency-patterns.md)
- [Thread Pool Pattern](additional/thread-pool-pattern.md)
- [Producer-Consumer Pattern](additional/producer-consumer-pattern.md)
- [Handle Concurrency Scenarios](../interview-method/handle-concurrency-scenarios.md)

## References
- Pugh et al., "The 'Double-Checked Locking is Broken' Declaration" — https://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html
- *Java Language Specification*, Chapter 17 (Threads and Locks), specifically §17.4 Memory Model, §17.5 Final Field Semantics, §17.7 Non-Atomic Treatment of `double` and `long` — https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html
- JSR-133 (Java Memory Model and Thread Specification Revision) — https://www.cs.umd.edu/~pugh/java/memoryModel/
- cppreference, *std::memory_order* — https://en.cppreference.com/w/cpp/atomic/memory_order
- cppreference, `<thread>`, `<atomic>`, `<mutex>` — https://en.cppreference.com/w/cpp/thread
- Brian Goetz et al., *Java Concurrency in Practice* (Addison-Wesley, 2006) — chapters cited inline.
- Doug Lea, *Concurrent Programming in Java* and online notes — https://gee.cs.oswego.edu/dl/
- Doug Lea, "A Java Fork/Join Framework" — https://gee.cs.oswego.edu/dl/papers/fj.pdf
- Scott Meyers & Andrei Alexandrescu, "C++ and the Perils of Double-Checked Locking" (Dr. Dobb's, 2004) — https://www.aristeia.com/Papers/DDJ_Jul_Aug_2004_revised.pdf
- Herb Sutter, "Effective Concurrency" series — https://herbsutter.com/category/effective-concurrency/
- Maged M. Michael, "Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects", IEEE TPDS 15(6), 2004.
- JEP 444 (Virtual Threads) — https://openjdk.org/jeps/444
- JEP 446 (Scoped Values) — https://openjdk.org/jeps/446
