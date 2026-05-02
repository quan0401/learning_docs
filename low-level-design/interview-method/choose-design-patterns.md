---
title: "How to Choose Design Patterns Without Forcing Them"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, interview-method, design-patterns, trade-offs]
---

# How to Choose Design Patterns Without Forcing Them

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `interview-method` `design-patterns` `trade-offs`

## Summary

Patterns are tools to communicate intent, not items on a checklist. Strong candidates choose a pattern when the **intent** matches the problem (variation, decoupling, lifecycle), not when the **shape** looks familiar. The "have I named the trade-off" rule keeps you honest: if you cannot say what you are trading away, you are reaching for ceremony.

## Pattern Intent vs Pattern Shape

Every pattern has two faces:

- **Intent** — the problem it solves. *Why* you would reach for it.
- **Shape** — the structural code (interfaces, classes, delegation).

Junior engineers recognize shape. Senior engineers select on intent.

### Example: Strategy

- **Shape:** an interface, multiple implementations, a context that holds one.
- **Intent:** the algorithm varies independently from the caller, and you need to swap or extend at runtime.

If you have one implementation today and no realistic second one, the shape exists but the intent does not. You are adding ceremony.

### Example: Observer

- **Shape:** a subject holding a list of listeners with `subscribe`/`notify`.
- **Intent:** decouple producers and consumers of an event when you do not control or do not know the consumers.

Three callbacks all owned by the same module are not Observer — they are just function calls.

## The "Have I Named the Trade-Off" Rule

Before introducing a pattern, finish this sentence out loud:

> "I am using **[pattern]** because **[problem in domain language]**, and I am accepting **[concrete cost]** in exchange."

Examples:

- "I am using **Strategy** because billing rules differ per vehicle type and we want to add types without recompiling the service. I am accepting **one extra interface** and the **constructor wiring cost**."
- "I am using **Factory** because instantiation requires resolving config that is not yet known at compile time. I am accepting that **callers no longer see the concrete type**."
- "I am using **Decorator** because we need to layer logging and rate-limiting onto the same `LoanService` interface without modifying it. I am accepting **a deeper call stack** and **harder breakpoints**."

If you cannot complete the sentence, you do not need the pattern.

## Pattern Cheat Sheet — Intent First

For each pattern below, the intent is what to scan for in the problem.

### Creational

| Pattern | Intent — pick when |
|---|---|
| Factory Method | Subclasses decide which concrete type to instantiate |
| Abstract Factory | A family of related products must vary together |
| Builder | Many optional parameters; immutable result; readable construction |
| Singleton | Genuinely one instance must exist (rare; usually a code smell) |
| Prototype | Cloning is cheaper than constructing from scratch |

### Structural

| Pattern | Intent — pick when |
|---|---|
| Adapter | Existing class has wrong interface; you cannot change it |
| Bridge | Two dimensions of variation must vary independently |
| Composite | Tree-shaped structure must be treated uniformly with leaves |
| Decorator | Add behavior dynamically without subclassing every variation |
| Facade | Hide complex subsystem behind one entry point |
| Proxy | Control access, defer cost, or add a transparent layer |
| Flyweight | Many instances share intrinsic state; memory pressure is real |

### Behavioral

| Pattern | Intent — pick when |
|---|---|
| Strategy | Algorithm varies independently from caller |
| Observer | Producer must notify unknown or many consumers |
| Command | Operations must be queued, undone, or logged |
| State | Behavior depends on state; explicit transitions matter |
| Template Method | Skeleton stays fixed; specific steps vary by subclass |
| Iterator | Traverse without exposing internal structure |
| Mediator | Many-to-many coupling becomes a hairball; centralize coordination |
| Chain of Responsibility | Multiple handlers; the right one is decided at runtime |
| Visitor | Operations over a stable type hierarchy must vary |

## A Worked Example — Parking Lot Billing

Naive approach:

```java
public Money calculate(Ticket t, Instant exitTime) {
    Duration d = Duration.between(t.entryTime(), exitTime);
    long hours = d.toHours() + 1;
    if (t.slot().vehicle() instanceof Motorcycle) return Money.of(20 * hours);
    if (t.slot().vehicle() instanceof Car) return Money.of(40 * hours);
    if (t.slot().vehicle() instanceof Truck) return Money.of(80 * hours);
    throw new IllegalStateException();
}
```

Three problems: every new vehicle type touches this method; rates are buried; testing requires real vehicle subclasses.

**Strategy fits because:**

- The algorithm (per-hour rate) varies independently per vehicle type
- New vehicle types are likely (EVs, scooters)
- The variation axis is clear: vehicle type -> rate

```java
public interface BillingStrategy {
    Money calculate(Duration parked);
}

public final class FlatHourlyRate implements BillingStrategy {
    private final Money rate;
    public FlatHourlyRate(Money rate) { this.rate = rate; }
    public Money calculate(Duration parked) {
        long hours = parked.toHours() + 1;
        return rate.times(hours);
    }
}

public final class BillingService {
    private final Map<VehicleType, BillingStrategy> strategies;

    public BillingService(Map<VehicleType, BillingStrategy> strategies) {
        this.strategies = Map.copyOf(strategies);
    }

    public Money calculate(Ticket ticket, Instant exitTime) {
        BillingStrategy strategy = strategies.get(ticket.vehicleType());
        if (strategy == null) {
            throw new UnknownVehicleTypeException(ticket.vehicleType());
        }
        Duration parked = Duration.between(ticket.entryTime(), exitTime);
        return strategy.calculate(parked);
    }
}
```

```typescript
export interface BillingStrategy {
  calculate(parkedMs: number): Money;
}

export class FlatHourlyRate implements BillingStrategy {
  constructor(private readonly rate: Money) {}
  calculate(parkedMs: number): Money {
    const hours = Math.floor(parkedMs / 3_600_000) + 1;
    return this.rate.times(hours);
  }
}

export class BillingService {
  constructor(
    private readonly strategies: Map<VehicleType, BillingStrategy>,
  ) {}

  calculate(ticket: Ticket, exitTime: Date): Money {
    const strategy = this.strategies.get(ticket.vehicleType);
    if (!strategy) throw new UnknownVehicleTypeError(ticket.vehicleType);
    const parkedMs = exitTime.getTime() - ticket.entryTime.getTime();
    return strategy.calculate(parkedMs);
  }
}
```

**Trade-off named:** Strategy because billing rules vary per type and we expect more types. We accept one interface + a wiring map.

## When the Same Shape Is Not the Same Pattern

State and Strategy share shape (interface + implementations + holder) but not intent.

- **Strategy** — the holder picks the algorithm by *external* criteria (config, vehicle type)
- **State** — the holder transitions the algorithm by *internal* state changes (`Active -> Cancelled -> Refunded`)

If you would describe the variation as "depending on the type of input," it is Strategy. If "depending on what happened so far," it is State.

## Anti-Patterns to Avoid

### Pattern Stuffing

Wrapping every two-line function in Command, every list in Visitor, every method in Template Method. The interviewer will ask "why" — if you cannot answer with a domain reason, you lose more than you gained.

### Singleton by Default

Singletons hide global state. They make tests order-dependent and parallelism unsafe. Prefer dependency injection. If you genuinely need uniqueness, document why.

### Factory for One Concrete Class

A factory that always returns the same concrete type is just a constructor with extra steps. Skip it.

### Observer for Three In-Module Callbacks

If the producer and all consumers live in the same module and you control them, a method call list is fine. Observer earns its keep when consumers are unknown or vary.

### Decorator When Composition Suffices

Three nested decorators to add logging, metrics, and validation can often be replaced by one method that calls each in sequence — clearer and easier to debug.

### Strategy with Only One Strategy

A single strategy implementation behind an interface "for the future" is YAGNI. Inline the logic and add the interface when the second strategy actually appears.

## Choosing Quickly Under Time Pressure

In a 60-minute round, default to the simplest path:

1. Write the code straight, no patterns.
2. If the code clearly varies along an axis, name the axis aloud.
3. Match the axis to a pattern intent — usually Strategy, State, or Factory.
4. Name the trade-off.
5. Refactor in one focused pass.

If step 4 is not crisp, go back to step 1.

## Verbal Patterns That Score Well

- "I'm avoiding a pattern here because the variation does not exist yet."
- "I'm choosing Strategy over inheritance because the variation is per-instance, not per-type."
- "If we needed to add an EV type, the only file that changes is the strategy registration."
- "The trade-off is one extra interface and one map; I think that is worth it."

## Verbal Patterns That Lose Points

- "I'd use Visitor here." (without saying why)
- "Let me make this a Singleton."
- "We could use Command, Observer, and Decorator together..."
- "It's good practice to wrap everything in interfaces."

## A Quick Decision Heuristic

Ask yourself, in order:

1. Is there real variation today, or am I imagining it?
2. Is the variation axis stable enough to name?
3. Will the next change be along that axis, or orthogonal?
4. Can I express the trade-off in one sentence?

If any answer is "no" or "not sure," skip the pattern and write straight code.

## Related

- [approach-ood-interviews.md](approach-ood-interviews.md)
- [approach-machine-coding-interviews.md](approach-machine-coding-interviews.md)
- [identify-entities-and-model-relationships.md](identify-entities-and-model-relationships.md)
- [write-clean-code-in-interview.md](write-clean-code-in-interview.md)
- [handle-concurrency-scenarios.md](handle-concurrency-scenarios.md)
- [test-doubles-in-ood.md](test-doubles-in-ood.md)
- [../design-patterns/](../design-patterns/)
- [../solid/](../solid/)
- [../../system-design/INDEX.md](../../system-design/INDEX.md)

## References

- "Cracking the Coding Interview" — design pattern overview
- "Designing Data-Intensive Applications" — for trade-off reasoning vocabulary
