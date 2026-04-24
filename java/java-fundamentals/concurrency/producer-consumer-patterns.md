---
title: "Producer-Consumer Patterns — BlockingQueue, Backpressure, Disruptor"
date: 2026-04-24
updated: 2026-04-24
tags: [java, concurrency, producer-consumer, blockingqueue, backpressure, disruptor]
---

# Producer-Consumer Patterns — BlockingQueue, Backpressure, Disruptor

**Date:** 2026-04-24 | **Updated:** 2026-04-24
**Tags:** `java` `concurrency` `producer-consumer` `blockingqueue` `backpressure` `disruptor`

## Table of Contents

- [Summary](#summary)
- [BlockingQueue — The Canonical Primitive](#blockingqueue--the-canonical-primitive)
- [Queue Variants](#queue-variants)
- [Bounded vs Unbounded — Backpressure or Bust](#bounded-vs-unbounded--backpressure-or-bust)
- [Graceful Shutdown — Drain and Poison Pill](#graceful-shutdown--drain-and-poison-pill)
- [Multi-Producer Multi-Consumer](#multi-producer-multi-consumer)
- [TransferQueue — True Hand-Off](#transferqueue--true-hand-off)
- [Disruptor — Ring-Buffer High Throughput](#disruptor--ring-buffer-high-throughput)
- [Reactive Streams vs Manual Backpressure](#reactive-streams-vs-manual-backpressure)
- [Kafka as Distributed Producer-Consumer](#kafka-as-distributed-producer-consumer)
- [Testing Producer-Consumer](#testing-producer-consumer)
- [Related](#related)
- [References](#references)

---

## Summary

Producer-consumer is the canonical concurrency pattern: decouple who makes work from who does it via a queue. In Java, the primary API is [`BlockingQueue<T>`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/BlockingQueue.html) — `put` blocks when full, `take` blocks when empty, `offer`/`poll` take timeouts, and rejection policies handle overflow. Six JDK implementations span the design space (bounded array, unbounded linked, zero-capacity hand-off, priority, delay, transfer). Two advanced alternatives exist when BlockingQueue isn't fast enough: [LMAX Disruptor](https://lmax-exchange.github.io/disruptor/) (ring buffer, sub-microsecond) for same-JVM throughput, and Reactive Streams for backpressure-aware pipelines. Kafka is the distributed producer-consumer. This doc covers them all and the design decisions that separate correct implementations from broken ones.

---

## BlockingQueue — The Canonical Primitive

```java
BlockingQueue<Order> queue = new LinkedBlockingQueue<>(1000);   // bounded

// Producer
void produce() {
    while (running) {
        Order o = fetchNextOrder();
        queue.put(o);                   // blocks if full
    }
}

// Consumer
void consume() {
    while (running) {
        try {
            Order o = queue.take();     // blocks if empty
            process(o);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return;
        }
    }
}
```

The four method pairs:

| Method | Full queue | Empty queue |
|--------|-----------|-------------|
| `add(e)` / `remove()` | Throws `IllegalStateException` | Throws `NoSuchElementException` |
| `offer(e)` / `poll()` | Returns `false` / `null` | Returns `false` / `null` |
| `put(e)` / `take()` | Blocks until space | Blocks until item |
| `offer(e, t, unit)` / `poll(t, unit)` | Blocks up to timeout | Blocks up to timeout |

Rule: in producer-consumer, use `put`/`take` (block) for typical load or `offer`/`poll` with timeout (fail fast) for shedding.

---

## Queue Variants

| Implementation | Capacity | Order | Best for |
|----------------|----------|-------|----------|
| [`ArrayBlockingQueue`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ArrayBlockingQueue.html) | Bounded, fixed | FIFO | Predictable memory, moderate throughput. |
| [`LinkedBlockingQueue`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/LinkedBlockingQueue.html) | Optionally bounded | FIFO | High throughput (head/tail locks are separate). Default choice. |
| [`PriorityBlockingQueue`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/PriorityBlockingQueue.html) | Unbounded | Heap-ordered | Priority scheduling (urgent tasks jump the queue). |
| [`SynchronousQueue`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/SynchronousQueue.html) | Zero | Hand-off | Every `put` waits for a `take`. Used by `Executors.newCachedThreadPool`. |
| [`DelayQueue`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/DelayQueue.html) | Unbounded | Delay-based | Elements become available only after their delay expires. Schedulers. |
| [`LinkedTransferQueue`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/LinkedTransferQueue.html) | Unbounded | FIFO | Supports `transfer()` for true hand-off. Higher throughput than `LBQ`. |

Default choice: `LinkedBlockingQueue` with an explicit capacity. `ArrayBlockingQueue` when you want absolute memory determinism.

---

## Bounded vs Unbounded — Backpressure or Bust

An **unbounded** queue never blocks producers. Sounds nice — until producers outpace consumers. Memory fills, GC thrashes, OOM.

A **bounded** queue blocks producers when full — that's backpressure. The producer slows down naturally to the consumer's rate.

```java
// WRONG — unbounded queue, no backpressure
BlockingQueue<Event> q = new LinkedBlockingQueue<>();      // default = Integer.MAX_VALUE

// RIGHT — bounded, producer blocks when consumer is slow
BlockingQueue<Event> q = new LinkedBlockingQueue<>(10_000);
```

If the producer must not block (e.g., it's an event handler on a critical thread):

```java
// Shed load — drop events when queue is full
if (!q.offer(event)) {
    droppedCounter.increment();   // metric for shed events
}
```

The rejection policy decision is a design decision, not a default. Pick one explicitly:
- **Block** — producer waits. Easy, but the calling thread stalls.
- **Drop newest** — `offer` fails. Good for fresh-data streams where stale is worthless.
- **Drop oldest** — pop the head, add the new. Good for "latest state matters" (e.g., sensor readings).
- **Caller runs** — producer does the work itself. Natural throttling in executor pools.

---

## Graceful Shutdown — Drain and Poison Pill

Three shutdown strategies:

**1. Poison pill** — a sentinel value each consumer recognizes as "stop":

```java
private static final Order POISON = new Order("__STOP__", 0);

void shutdown(int consumerCount) throws InterruptedException {
    for (int i = 0; i < consumerCount; i++) queue.put(POISON);   // one per consumer
}

// Consumer
while (true) {
    Order o = queue.take();
    if (o == POISON) break;
    process(o);
}
```

Advantages: drains everything already queued, clean exit. Disadvantages: need consumer count upfront; can't combine well with dynamic scaling.

**2. Interrupt** — signal all consumer threads; they exit on `InterruptedException`:

```java
consumers.forEach(Thread::interrupt);
```

Advantages: works with unknown consumer count. Disadvantages: drops in-flight queue items unless you drain first.

**3. Flag + poll** — set a shared `volatile boolean running = false`; consumers check it:

```java
while (running) {
    Order o = queue.poll(1, SECONDS);      // don't block forever
    if (o != null) process(o);
}
```

Advantages: simple. Disadvantages: consumers may miss the flag for up to the poll timeout; wastes CPU on empty polls.

Typical production recipe: flag + interrupt + drain-then-poison:

```java
running = false;
// wait for producer to observe the flag and stop producing
Thread.sleep(500);
// now drain remaining items
while (!queue.isEmpty()) process(queue.poll());
// finally send poison to unblock any sleeping consumer
for (int i = 0; i < consumers; i++) queue.put(POISON);
```

See [interruption-and-cancellation.md](interruption-and-cancellation.md).

---

## Multi-Producer Multi-Consumer

All BlockingQueue implementations support multi-producer multi-consumer — the locks serialize concurrent `put`s and `take`s. No extra work needed for correctness.

For high throughput, sharding helps. Partition work by key across N queues, each with a dedicated consumer:

```java
int shards = 16;
List<BlockingQueue<Event>> queues = IntStream.range(0, shards)
    .mapToObj(i -> new LinkedBlockingQueue<Event>(1000))
    .toList();

void publish(Event e) {
    int shard = Math.floorMod(e.key().hashCode(), shards);
    queues.get(shard).put(e);
}
```

Preserves per-key order (same key → same consumer). Parallelizes across keys.

This is exactly how Kafka partitioning works — see [Kafka as distributed producer-consumer](#kafka-as-distributed-producer-consumer).

---

## TransferQueue — True Hand-Off

`LinkedTransferQueue.transfer(E)` waits until a consumer actually takes the element:

```java
LinkedTransferQueue<Task> q = new LinkedTransferQueue<>();

// Producer waits until a consumer takes
q.transfer(task);   // blocks until some thread calls q.take()
```

Use when you want the producer to pair with a consumer — no queuing, synchronous hand-off. `SynchronousQueue` is the zero-capacity ancestor; `LinkedTransferQueue` is the modern, faster variant that also supports traditional `put`/`take`.

Useful pattern for work distribution where you don't want buildup — a stalled consumer blocks the producer, which surfaces the problem immediately.

---

## Disruptor — Ring-Buffer High Throughput

[LMAX Disruptor](https://lmax-exchange.github.io/disruptor/) is a specialized structure for same-JVM, ultra-low-latency producer-consumer. Key ideas:

- Fixed-size **ring buffer** — no allocation per event.
- **Sequence numbers** (volatile longs) instead of locks.
- **Mechanical sympathy** — cache-line padding, memory pre-allocation, batching.

```java
Disruptor<Event> disruptor = new Disruptor<>(Event::new, 1024, Executors.defaultThreadFactory());
disruptor.handleEventsWith((event, seq, endOfBatch) -> process(event));
disruptor.start();

RingBuffer<Event> ring = disruptor.getRingBuffer();
long seq = ring.next();
try {
    Event e = ring.get(seq);
    e.set(...);                         // fill in-place
} finally {
    ring.publish(seq);
}
```

Benchmarks: Disruptor delivers ~10M events/sec per core. 100× faster than `ArrayBlockingQueue` for the same contract.

When to use:
- Sub-microsecond latency requirement.
- Known fixed-size event types (ring is pre-allocated).
- Low-GC need — zero allocation per event matters.
- Financial trading, telemetry pipelines, game engines.

When not:
- You'd bikeshed a `LinkedBlockingQueue` at < 100k/sec. Use that.
- You need persistence — Disruptor is memory-only.
- Over-process or network — use Kafka.

---

## Reactive Streams vs Manual Backpressure

Reactive Streams (Project Reactor, RxJava) builds backpressure into the protocol via `Subscription.request(n)`:

```java
Flux<Event> events = source()
    .onBackpressureBuffer(10_000, BufferOverflowStrategy.DROP_OLDEST)
    .publishOn(Schedulers.parallel(), 256);   // prefetch=256

events.subscribe(new BaseSubscriber<Event>() {
    @Override protected void hookOnSubscribe(Subscription s) { s.request(100); }
    @Override protected void hookOnNext(Event e) {
        process(e);
        request(1);                           // consumer-driven demand
    }
});
```

Difference from BlockingQueue:
- Demand flows *upstream* — consumer explicitly asks for N.
- Operator-level strategies (buffer, drop, latest) are first-class.
- No threads blocked — entire pipeline is non-blocking.

When manual BQ is simpler: a fixed pipeline of fixed operators in a single JVM. When reactive wins: composing many operators, variable fan-out, integration with reactive I/O (WebFlux, R2DBC).

See [reactive-programming-java.md](../../reactive-programming-java.md) and [reactive-advanced-topics.md § backpressure](../../reactive-advanced-topics.md).

---

## Kafka as Distributed Producer-Consumer

The producer-consumer pattern extended to a durable, distributed log. Key differences:

- Persistent — consumers can replay history.
- Partitioned — each partition is a producer-consumer queue with ordering guarantees per key.
- At-least-once delivery by default; exactly-once possible with transactions.
- Consumer groups — automatic work distribution among consumer instances.

See [reactive-kafka.md](../../messaging/reactive-kafka.md) for the reactive client and [spring-kafka.md](../../messaging/spring-kafka.md) for the imperative one. [Event-driven patterns](../../messaging/event-driven-patterns.md) covers outbox/saga on top of Kafka.

Rule: BlockingQueue for in-JVM, Disruptor for ultra-low-latency in-JVM, Kafka when you need durability, fan-out across services, or replay.

---

## Testing Producer-Consumer

Concurrent tests are notoriously flaky. Strategies:

**1. Bounded, deterministic tests**: use `CountDownLatch` to verify exact completion:

```java
@Test
void consumesAllProducedItems() throws Exception {
    int count = 1000;
    CountDownLatch done = new CountDownLatch(count);
    BlockingQueue<Integer> q = new LinkedBlockingQueue<>(100);

    // Consumer
    Thread consumer = Thread.startVirtualThread(() -> {
        while (true) {
            try {
                q.take();
                done.countDown();
            } catch (InterruptedException e) { return; }
        }
    });

    // Produce
    for (int i = 0; i < count; i++) q.put(i);

    assertTrue(done.await(5, SECONDS), "consumer did not drain queue");
    consumer.interrupt();
}
```

**2. [Awaitility](https://github.com/awaitility/awaitility)** for eventual assertions:

```java
await().atMost(5, SECONDS).untilAsserted(() ->
    assertThat(processedEvents.size()).isEqualTo(expected)
);
```

**3. [jcstress](concurrency-debugging.md#jcstress-for-lock-free-correctness)** for correctness of custom implementations.

Anti-patterns: `Thread.sleep` to "wait for it", single-run tests that happen to pass, `@Test(timeout = ...)` without assertion on final state.

---

## Related

- [Concurrency Basics](concurrency-basics.md) — thread fundamentals.
- [Multithreading Deep Dive](multithreading-deep-dive.md) — BlockingQueue section, executors.
- [Interruption and Cancellation](interruption-and-cancellation.md) — graceful shutdown.
- [Concurrent Collections](concurrent-collections.md) — non-blocking structures.
- [CompletableFuture Deep Dive](completablefuture-deep-dive.md) — async composition as an alternative.
- [Virtual Threads](virtual-threads.md) — VTs + BlockingQueue for simple concurrent consumers.
- [Reactive Kafka](../../messaging/reactive-kafka.md) — distributed producer-consumer.
- [Spring for Apache Kafka](../../messaging/spring-kafka.md) — imperative Kafka.
- [Event-Driven Patterns](../../messaging/event-driven-patterns.md) — broker patterns built on producer-consumer.
- [Reactive Programming](../../reactive-programming-java.md) — Flux backpressure.

---

## References

- [`BlockingQueue` Javadoc (JDK 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/BlockingQueue.html)
- [`LinkedTransferQueue` Javadoc](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/LinkedTransferQueue.html)
- [LMAX Disruptor documentation](https://lmax-exchange.github.io/disruptor/)
- [Disruptor paper — Martin Thompson et al.](https://lmax-exchange.github.io/disruptor/disruptor.html)
- [Brian Goetz et al. — *Java Concurrency in Practice*](https://jcip.net/) — Chapter 5 (BlockingQueue).
- [Reactive Streams specification](https://www.reactive-streams.org/)
- [Awaitility](https://github.com/awaitility/awaitility)
- [Aleksey Shipilëv — JVM Anatomy Quarks](https://shipilev.net/jvm/anatomy-quarks/)
