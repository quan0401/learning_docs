---
title: "How to Handle Concurrency Scenarios"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, interview-method, concurrency, locking, immutability]
---

# How to Handle Concurrency Scenarios

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `interview-method` `concurrency` `locking` `immutability`

## Summary

Most LLD interviews include a "what about concurrency" probe. The expectation is not a full lock-free implementation. It is that you can identify shared state, draw a boundary around it, name a primitive (synchronized, ReentrantLock, ReadWriteLock, CAS, immutable copy, async queue), and articulate the trade-off. Mention it; draw the boundary; pick a primitive.

## What Interviewers Actually Expect

There are three behaviors that score:

1. **Mention it.** You proactively name shared state and risks before being asked.
2. **Draw the boundary.** You identify which methods are critical sections and which are pure / read-only.
3. **Pick a primitive.** You commit to one strategy, name its cost, and stop.

What does *not* score:

- Implementing a lock-free queue from scratch
- Reciting the Java Memory Model
- Five-minute monologues on volatile vs synchronized
- Pretending concurrency does not exist when the prompt says "many gates"

## Step 1 — Identify Shared State

Walk the model and ask: "If two threads called this at once, what could go wrong?"

For a parking lot:

- `SlotRepository.findFirstAvailable(...)` followed by `slot.assign(...)` — classic check-then-act race
- `Ticket` repository writes
- `BillingService` if it caches rates

Pure value objects (`Money`, `Coordinate`, `LoanPeriod`) are race-free by construction.

## Step 2 — Pick a Strategy Per Boundary

Do not pick one global strategy. Match each boundary to its access pattern.

| Pattern | Use when | Cost |
|---|---|---|
| Immutability | Object never changes after construction | None at runtime; copy cost on update |
| `synchronized` block / method | Short critical section, low contention | Coarse; easy to reason about |
| `ReentrantLock` | Need `tryLock`, timed, or interruptible | Slightly more code; explicit unlock |
| `ReadWriteLock` | Many readers, few writers | Writer can starve under heavy reads |
| `StampedLock` (Java 8+) | Very read-heavy with optimistic reads | Tricky semantics; verify carefully |
| CAS (`AtomicX`) | Single-field updates, no compound invariant | Fast; not suitable for multi-field ops |
| Concurrent collections | Map/queue is the shared state | Drop-in; covers the structure only |
| Actor / async queue | Want to serialize work without blocking | Throughput limited by single consumer |
| Per-aggregate lock | Need parallelism across aggregates | Lock-acquisition order discipline |

## Step 3 — Worked Examples

### Example A — Slot Assignment with `synchronized`

The check-then-act race: two threads find the same available slot and both call `assign`.

```java
public final class SlotRepository {
    private final List<Slot> slots;
    private final Object lock = new Object();

    public SlotRepository(List<Slot> slots) {
        this.slots = new ArrayList<>(slots);
    }

    public Slot reserveFor(VehicleType type) {
        synchronized (lock) {
            for (Slot s : slots) {
                if (s.isAvailable() && s.canFit(type)) {
                    s.markReserved();
                    return s;
                }
            }
            throw new NoSlotAvailableException(type);
        }
    }
}
```

Trade-off: simple and correct, but reserves contention on a single lock for every gate. Acceptable for a small lot.

### Example B — `ReentrantLock` with `tryLock` for Timeouts

When a gate cannot wait forever:

```java
private final ReentrantLock lock = new ReentrantLock();

public Slot reserveFor(VehicleType type, Duration maxWait) throws InterruptedException {
    if (!lock.tryLock(maxWait.toMillis(), TimeUnit.MILLISECONDS)) {
        throw new GateBusyException();
    }
    try {
        // critical section
        return findAndReserve(type);
    } finally {
        lock.unlock();
    }
}
```

Trade-off: more code, but failure modes (timeout, interrupt) are first-class.

### Example C — `ReadWriteLock` for Mostly-Read Inventory

If you have a pricing table read by every billing call but updated rarely:

```java
private final ReadWriteLock rw = new ReentrantReadWriteLock();
private Map<VehicleType, Money> rates;

public Money rateFor(VehicleType type) {
    rw.readLock().lock();
    try {
        return rates.get(type);
    } finally {
        rw.readLock().unlock();
    }
}

public void updateRates(Map<VehicleType, Money> next) {
    rw.writeLock().lock();
    try {
        this.rates = Map.copyOf(next);
    } finally {
        rw.writeLock().unlock();
    }
}
```

Trade-off: writer starvation under heavy read load. Acceptable when updates are rare.

### Example D — Immutable Snapshot + CAS

For high read concurrency on small state, swap whole snapshots atomically.

```java
private final AtomicReference<Map<VehicleType, Money>> rates =
        new AtomicReference<>(Map.of());

public Money rateFor(VehicleType type) {
    return rates.get().get(type);
}

public void updateRates(Map<VehicleType, Money> next) {
    rates.set(Map.copyOf(next));
}
```

Trade-off: readers may see stale snapshots momentarily. Often fine.

### Example E — Concurrent Collection

If the only shared structure is a map of tickets:

```java
private final ConcurrentHashMap<UUID, Ticket> tickets = new ConcurrentHashMap<>();

public Ticket save(Ticket t) {
    Ticket prev = tickets.putIfAbsent(t.id(), t);
    if (prev != null) throw new DuplicateTicketException(t.id());
    return t;
}
```

Trade-off: covers the *structure*, not compound invariants across multiple keys.

### Example F — TypeScript Equivalent (Single-Threaded Event Loop)

In Node.js you do not have shared mutable memory across threads by default — but you do have **interleaved async operations** that cause logically the same races.

```typescript
class SlotRepository {
  private readonly slots: Slot[];
  private inFlight: Promise<void> = Promise.resolve();

  constructor(slots: Slot[]) {
    this.slots = [...slots];
  }

  async reserveFor(type: VehicleType): Promise<Slot> {
    const release = this.queue();
    try {
      await release.acquired;
      const slot = this.slots.find((s) => s.isAvailable && s.canFit(type));
      if (!slot) throw new NoSlotAvailableError(type);
      slot.markReserved();
      return slot;
    } finally {
      release.release();
    }
  }

  private queue(): { acquired: Promise<void>; release: () => void } {
    let release!: () => void;
    const acquired = this.inFlight.then(
      () => new Promise<void>((r) => (release = r)),
    );
    this.inFlight = acquired;
    return { acquired, release: () => release() };
  }
}
```

Trade-off: a chain of promises serializes work; throughput equals the slowest critical section.

## Immutability as a First Resort

The cheapest concurrency answer is "this never mutates." Default value objects to immutable, and many concurrency questions disappear.

```java
public record Money(long amount, Currency currency) {
    public Money {
        if (amount < 0) throw new IllegalArgumentException();
        Objects.requireNonNull(currency);
    }
    public Money plus(Money other) { ... }
}
```

```typescript
export class Money {
  constructor(
    readonly amount: number,
    readonly currency: Currency,
  ) {
    if (amount < 0) throw new Error("Amount cannot be negative");
  }
  plus(other: Money): Money {
    return new Money(this.amount + other.amount, this.currency);
  }
}
```

A `Money` can be passed across threads freely. So can `Ticket`, `Receipt`, and most domain values.

## Async Patterns — When Locking Is the Wrong Tool

Some problems are better expressed as queues or actors than as critical sections.

- **Single-writer queue.** Producers enqueue; one consumer drains. No locks needed; throughput limited by consumer.
- **Per-aggregate executor.** One executor per aggregate ID; commands for that aggregate run in order. Useful when state is partitioned naturally.
- **Event sourcing.** Append-only log + projections. Concurrency at the log boundary only.

These show up more often in HLD than LLD, but mentioning them earns credit.

## Deadlock — How to Not Fall In

If you take more than one lock:

- Always acquire in a fixed global order (e.g., by ID)
- Use timed `tryLock` to fail rather than block forever
- Keep critical sections short — no I/O, no callbacks
- Prefer one lock per aggregate, not nested locks across aggregates

## Common Stumbles

### "I'd use synchronized" — Without Specifying What

Lock something. `synchronized` on `this` for a service whose state lives in a repository protects nothing.

### Implementing a Lock-Free Algorithm From Scratch

Unless asked, do not. State the use of `AtomicReference` or `ConcurrentHashMap` and move on.

### Ignoring the Read Path

Two threads reading non-volatile fields can see torn or stale values. If a field is read concurrently, it must be guarded or `volatile` (Java) or behind a lock.

### Confusing Atomicity Per Field With Atomicity Across Fields

`AtomicInteger.incrementAndGet()` is atomic for that field. If your invariant spans two fields, you need a lock or a single immutable snapshot.

### Pretending Node.js Has No Concurrency

It does — interleaved async functions race over shared state. The concurrency primitive is a logical lock built from promises, or a queue, or an immutable swap.

### Holding a Lock Across I/O

If you call a database, network, or external API while holding a lock, you have invented a contention point that can stall the whole system. Release first; reacquire if needed.

## What to Say in the Interview

- "The shared state here is the slot inventory. I'll guard it with a single lock for now."
- "For a real system I'd partition by floor so each floor has its own lock — that's the next refactor."
- "Reads dominate writes in pricing, so I'd use a `ReadWriteLock` or atomic snapshot."
- "I'm choosing immutability for `Money` and `Ticket` so they cross thread boundaries safely."
- "I won't implement a lock-free queue here, but `ConcurrentHashMap` is enough for the ticket store."

## Quick Decision Tree

1. Is the state mutable? -> if no, you're done.
2. Is one field enough to express the invariant? -> CAS / atomic.
3. Is the structure the only thing shared? -> concurrent collection.
4. Is the contention low and the section short? -> `synchronized` block.
5. Need timeouts or interrupts? -> `ReentrantLock`.
6. Reads >> writes? -> `ReadWriteLock` or immutable snapshot.
7. Cross-aggregate operation? -> queue / actor / single-writer.

## Related

- [approach-ood-interviews.md](approach-ood-interviews.md)
- [approach-machine-coding-interviews.md](approach-machine-coding-interviews.md)
- [identify-entities-and-model-relationships.md](identify-entities-and-model-relationships.md)
- [write-clean-code-in-interview.md](write-clean-code-in-interview.md)
- [choose-design-patterns.md](choose-design-patterns.md)
- [test-doubles-in-ood.md](test-doubles-in-ood.md)
- [../solid/](../solid/)
- [../design-patterns/](../design-patterns/)
- [../../system-design/INDEX.md](../../system-design/INDEX.md)

## References

- "Designing Data-Intensive Applications" — concurrency and replication chapters
- "Cracking the Coding Interview" — concurrency section
