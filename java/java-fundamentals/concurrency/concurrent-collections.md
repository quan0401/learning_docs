---
title: "Concurrent Collections — ConcurrentHashMap, LongAdder, CAS, ABA Problem"
date: 2026-04-24
updated: 2026-04-24
tags: [java, concurrency, concurrent-collections, cas, lock-free, atomics]
---

# Concurrent Collections — ConcurrentHashMap, LongAdder, CAS, ABA Problem

**Date:** 2026-04-24 | **Updated:** 2026-04-24
**Tags:** `java` `concurrency` `concurrent-collections` `cas` `lock-free` `atomics`

## Table of Contents

- [Summary](#summary)
- [Why Not `Collections.synchronizedMap`](#why-not-collectionssynchronizedmap)
- [ConcurrentHashMap](#concurrenthashmap)
  - [Atomic Compound Ops](#atomic-compound-ops)
  - [size() and Weakly Consistent Iterators](#size-and-weakly-consistent-iterators)
- [LongAdder vs AtomicLong](#longadder-vs-atomiclong)
- [CopyOnWriteArrayList](#copyonwritearraylist)
- [ConcurrentSkipListMap](#concurrentskiplistmap)
- [CAS Patterns and VarHandle](#cas-patterns-and-varhandle)
- [The ABA Problem](#the-aba-problem)
- [Memory Ordering Modes](#memory-ordering-modes)
- [Performance Pitfalls — False Sharing, Boxing](#performance-pitfalls--false-sharing-boxing)
- [Picking the Right Structure](#picking-the-right-structure)
- [Related](#related)
- [References](#references)

---

## Summary

`java.util.concurrent` provides thread-safe collections and atomic primitives that outperform synchronized wrappers by orders of magnitude in contended workloads. [`ConcurrentHashMap`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html) is lock-striped and mostly lock-free on reads; [`LongAdder`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/LongAdder.html) shards an atomic counter across cells to eliminate CAS contention; [`VarHandle`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/VarHandle.html) exposes the memory-ordering modes that the JMM formalized in JEP 193 as a safe replacement for `Unsafe`. This doc covers the primitives every production concurrent system uses, the ABA trap when doing CAS on pointers, and the rule-of-thumb for picking the right concurrent collection.

---

## Why Not `Collections.synchronizedMap`

The JDK's oldest solution — wrap a `HashMap` with a global lock — scales terribly:

- **Every operation** (read or write) takes the same lock.
- Under concurrency, readers block on writers and each other.
- Iteration requires holding the lock externally (or you get `ConcurrentModificationException`).

`ConcurrentHashMap` replaces this with:
- Non-blocking reads (volatile + version check).
- Lock-striped writes (per-bucket lock in modern JDKs; not segment-level).
- Weakly consistent iterators — never throw CME.

Rule: in new code, never use `Collections.synchronizedMap/List/Set`. Prefer the concurrent variants or immutable snapshots.

---

## ConcurrentHashMap

The workhorse concurrent collection in Java.

```java
ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();
counts.put("apple", 5);
counts.put("banana", 3);
```

Internals (JDK 8+):
- Bucket array; each bucket either holds a linked list or a red-black tree (for heavy hash collisions).
- Reads are lock-free — volatile loads plus a `tabAt` read.
- Writes synchronize on the bucket head only.
- Resize is concurrent (multiple threads help grow the table).

### Atomic Compound Ops

The methods that replace racy read-modify-write patterns:

```java
// WRONG — check-then-act is not atomic
if (!counts.containsKey(key)) counts.put(key, 1);   // two threads can both enter

// RIGHT — atomic compound ops
counts.putIfAbsent(key, 1);
counts.computeIfAbsent(key, k -> loadFromDb(k));    // lazy init (famous cache idiom)
counts.compute(key, (k, v) -> v == null ? 1 : v + 1);
counts.merge(key, 1, Integer::sum);                 // increment-or-init
```

Rules:
- Use `merge` for counters — concise and atomic.
- Use `computeIfAbsent` for lazy init / cache-miss loading — the function runs at most once per missing key.
- **Never** call methods of the same `ConcurrentHashMap` inside the compute/merge lambda — can deadlock.

### size() and Weakly Consistent Iterators

`size()` is O(1) but approximate — it doesn't lock the whole table. For an exact count, use `mappingCount()` (which returns `long`).

Iteration is **weakly consistent**: you'll see entries that existed when the iterator was created, and may or may not see concurrent updates. Never throws `ConcurrentModificationException`. Good enough for analytics; not for "exactly this snapshot".

---

## LongAdder vs AtomicLong

[`AtomicLong`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/AtomicLong.html) uses a single CAS loop — under heavy contention, every thread retries its CAS. Throughput collapses.

```java
AtomicLong counter = new AtomicLong();
counter.incrementAndGet();   // CAS-retry loop
```

`LongAdder` shards the counter into multiple cells. Each thread hashes to a cell and updates it independently. Reads sum all cells.

```java
LongAdder counter = new LongAdder();
counter.increment();         // near lock-free at any contention
long total = counter.sum();  // read: sum across cells (not atomic snapshot)
```

| Scenario | `AtomicLong` | `LongAdder` |
|----------|--------------|-------------|
| Single writer | Fast | Fast |
| 4–8 contending writers | Slow (CAS retries) | Fast |
| Need compare-and-set semantics | Yes | No |
| Need an atomic snapshot of count | Yes (`get()`) | No (`sum()` is approximate) |

Rule: for monotonic counters (request counts, bytes transferred, errors), use `LongAdder`. For anything needing CAS (optimistic locking, single-value guards), use `AtomicLong`. [`LongAccumulator`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/LongAccumulator.html) generalizes this to any binary op.

---

## CopyOnWriteArrayList

Every write allocates a new array; reads are lock-free on the current array.

```java
List<Listener> listeners = new CopyOnWriteArrayList<>();

// reads — no lock, no snapshot drift
for (Listener l : listeners) l.fire(event);

// writes — expensive (full array copy)
listeners.add(newListener);
```

Use when:
- Writes are **rare** compared to reads (e.g., event listeners registered at startup).
- You need iteration without external locking.

Don't use when:
- Writes are frequent — O(N) per write, memory pressure.
- Large lists — every mutation copies the whole array.

Similar: `CopyOnWriteArraySet`. Built on top of COWAL.

---

## ConcurrentSkipListMap

Sorted, concurrent map with O(log n) operations. Based on [skip lists](https://en.wikipedia.org/wiki/Skip_list).

```java
ConcurrentSkipListMap<Instant, Event> timeline = new ConcurrentSkipListMap<>();

// Range query
NavigableMap<Instant, Event> lastHour = timeline
    .tailMap(Instant.now().minus(Duration.ofHours(1)));
```

Use when you need:
- Concurrent **sorted** map (ConcurrentHashMap is unsorted).
- Range queries.
- Ceiling/floor/higher/lower keys.

Slower than ConcurrentHashMap for point lookups. Similar: `ConcurrentSkipListSet`.

---

## CAS Patterns and VarHandle

Compare-and-set is the primitive behind every non-blocking algorithm:

```java
AtomicReference<State> state = new AtomicReference<>(initial);

State current, next;
do {
    current = state.get();
    next = transition(current);
} while (!state.compareAndSet(current, next));
```

[`VarHandle`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/VarHandle.html) (Java 9+) generalizes this to any field without requiring an `AtomicXxx` wrapper:

```java
public class Counter {
    private static final VarHandle VALUE;
    static {
        try {
            VALUE = MethodHandles.lookup()
                .findVarHandle(Counter.class, "value", long.class);
        } catch (Exception e) { throw new ExceptionInInitializerError(e); }
    }

    private volatile long value;

    public void increment() {
        long c;
        do { c = (long) VALUE.getAcquire(this); }
        while (!VALUE.compareAndSet(this, c, c + 1));
    }
}
```

VarHandle is the official replacement for `sun.misc.Unsafe`. Any lock-free code written in the last five years should use it.

---

## The ABA Problem

A subtle CAS bug: Thread 1 reads `A`, Thread 2 changes `A` → `B` → `A`, Thread 1's CAS succeeds even though the underlying state changed twice.

```java
// Node A → B → C (linked list)
AtomicReference<Node> head = new AtomicReference<>(nodeA);

// Thread 1: reads head = nodeA, plans to pop A (leaving B → C)
Node oldHead = head.get();        // nodeA
Node newHead = oldHead.next;      // nodeB

// Thread 2 pops A, pops B, pushes A back
// Now list is: A → C (B is gone)

// Thread 1's CAS succeeds because head == nodeA, but newHead (nodeB) is now orphaned!
head.compareAndSet(oldHead, newHead);   // returns true — list is now B → C but B was freed
```

Fix: [`AtomicStampedReference<T>`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/AtomicStampedReference.html) includes a version counter:

```java
AtomicStampedReference<Node> head = new AtomicStampedReference<>(nodeA, 0);

int[] stamp = new int[1];
Node current = head.get(stamp);
Node next = transition(current);
head.compareAndSet(current, next, stamp[0], stamp[0] + 1);   // version bump
```

The version counter catches the A→B→A sequence because the stamp changed.

Garbage-collected environments (Java) hit this less often than manual-memory-management (C++), because freed nodes don't get reused immediately — but it still happens with recycling pools and custom allocators. Any time you do CAS on a pointer and freedom of the pointee matters, consider stamping.

---

## Memory Ordering Modes

The JMM (Java Memory Model) used to have just `volatile`. Java 9 introduced nuanced access modes via VarHandle:

| Mode | Guarantee | Cost |
|------|-----------|------|
| `getPlain` / `setPlain` | No ordering — plain field access | Cheapest |
| `getOpaque` / `setOpaque` | Same-thread ordering, weak cross-thread | Very cheap |
| `getAcquire` / `setRelease` | Acquire-release pairing (happens-before) | Cheap |
| `getVolatile` / `setVolatile` | Full volatile semantics, sequential consistency | Most expensive |

In practice:
- Default: `volatile` semantics. Don't optimize unless profiling shows it's hot.
- Acquire-release is enough for most lock-free algorithms (publication of data, reading a recent-enough value).
- Opaque/Plain are for expert scenarios (benchmarking, lock-free data structures with explicit fences).

This parallels C++ `std::memory_order`. See [JEP 193](https://openjdk.org/jeps/193) for the formal spec.

---

## Performance Pitfalls — False Sharing, Boxing

**Boxing in atomics**: `AtomicReference<Integer>` incurs `Integer` boxing on every update. Use `AtomicInteger` for primitive int.

**False sharing**: two threads updating adjacent longs in the same cache line (64 bytes on x86) each invalidate the other's cache:

```java
public class PaddedCounter {
    private volatile long value;
    @jdk.internal.vm.annotation.Contended  // force padding to cache-line boundary
    private volatile long counter;
}
```

`@Contended` is JDK-internal; for user code, use `LongAdder` (which pads internally) or manually pad with dummy fields.

**Cache line sizing** matters for high-throughput concurrent code — profile with async-profiler's `-e LLC_MISSES`. See [concurrency-debugging.md](concurrency-debugging.md).

---

## Picking the Right Structure

| Need | Use |
|------|-----|
| Thread-safe map, high throughput | `ConcurrentHashMap` |
| Sorted concurrent map | `ConcurrentSkipListMap` |
| Monotonic counter under contention | `LongAdder` |
| Compare-and-set on a long | `AtomicLong` |
| Compare-and-set on a pointer | `AtomicReference` (watch for ABA) |
| Read-heavy event listener list | `CopyOnWriteArrayList` |
| Producer-consumer queue | `BlockingQueue` family — see [producer-consumer-patterns.md](producer-consumer-patterns.md) |
| Priority queue across threads | `PriorityBlockingQueue` |
| Cache with concurrent load | `ConcurrentHashMap.computeIfAbsent` or Caffeine |
| Read-only snapshot | `Map.copyOf(...)` returns an immutable snapshot |
| Set of enums | `EnumSet` + external sync, or `Collections.newSetFromMap(new ConcurrentHashMap<>())` |

Rule of thumb: start with `ConcurrentHashMap` / `LongAdder` / `CopyOnWriteArrayList` for 90% of cases. Reach for `VarHandle` / `AtomicStampedReference` only when profiling or correctness requires it.

---

## Related

- [Concurrency Basics](concurrency-basics.md) — `synchronized`, `volatile`, `AtomicInteger`.
- [Multithreading Deep Dive](multithreading-deep-dive.md) — JMM, happens-before, locks, synchronizers.
- [ForkJoinPool and Parallel Streams](forkjoinpool-and-parallel-streams.md) — work-stealing queues use these primitives internally.
- [ThreadLocal and Context Propagation](threadlocal-and-context.md) — when concurrent state doesn't need to be shared.
- [Concurrency Debugging](concurrency-debugging.md) — profiling cache-line contention, CAS retries.
- [Caching Deep Dive](../../data-repositories/caching-deep-dive.md) — Caffeine's use of `ConcurrentHashMap` patterns.

---

## References

- [`java.util.concurrent.atomic` Javadoc (JDK 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/package-summary.html)
- [`ConcurrentHashMap` Javadoc](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
- [JEP 193: Variable Handles](https://openjdk.org/jeps/193)
- [Brian Goetz et al. — *Java Concurrency in Practice*](https://jcip.net/) — Chapter 15 ("Atomic Variables and Nonblocking Synchronization").
- [Doug Lea — `java.util.concurrent` design papers](https://gee.cs.oswego.edu/dl/cpj/)
- [Aleksey Shipilëv — JVM Anatomy Quarks (False Sharing, Atomics)](https://shipilev.net/jvm/anatomy-quarks/)
- [Aleksey Shipilëv — VarHandles deep dive](https://shipilev.net/blog/2016/close-encounters-of-jmm-kind/)
- [The ABA Problem — Wikipedia](https://en.wikipedia.org/wiki/ABA_problem)
