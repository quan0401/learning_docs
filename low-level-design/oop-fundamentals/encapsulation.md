---
title: "Encapsulation — Hiding the Right Things"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, oop-fundamentals, encapsulation, immutability]
---

# Encapsulation — Hiding the Right Things

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `oop-fundamentals` `encapsulation` `immutability`

## Summary

Encapsulation is the discipline of hiding internal state and exposing only the operations that preserve invariants. It is not "make every field private and add a getter" — that is a mechanical ritual that often produces the opposite of encapsulation. The point is that callers cannot reach inside and break your guarantees.

## Table of Contents

- [What encapsulation actually is](#what-encapsulation-actually-is)
- [Visibility modifiers](#visibility-modifiers)
- [Getters and setters reality check](#getters-and-setters-reality-check)
- [Immutability as encapsulation](#immutability-as-encapsulation)
- [Encapsulation vs naked records / DTOs](#encapsulation-vs-naked-records--dtos)
- [Common pitfalls](#common-pitfalls)
- [Related](#related)

## What encapsulation actually is

Encapsulation is a *contract*: "I, the class, guarantee these invariants. You, the caller, can only interact with me through these methods." Anything that lets the caller bypass those methods — public mutable fields, returning the internal list directly, exposing builders that mutate after build — is a leak.

```java
// Encapsulation done badly — fields are private but the invariant leaks anyway.
public class Cart {
    private final List<Item> items = new ArrayList<>();
    public List<Item> getItems() { return items; }   // returns the live list
}

Cart c = new Cart();
c.getItems().clear();     // caller just bypassed every invariant
```

```java
// Encapsulation done well — caller can read, but not mutate the internals.
public class Cart {
    private final List<Item> items = new ArrayList<>();

    public List<Item> getItems() { return List.copyOf(items); } // unmodifiable copy

    public void add(Item item) {
        if (items.size() >= 50) throw new IllegalStateException("cart full");
        items.add(item);
    }
}
```

The point isn't `private`. The point is that `add` is the *only* path that adds items, and it enforces "no more than 50."

## Visibility modifiers

| Java modifier | Same class | Same package | Subclass (any package) | Anywhere |
|---|---|---|---|---|
| `private` | Yes | No | No | No |
| _(no modifier — package-private)_ | Yes | Yes | No | No |
| `protected` | Yes | Yes | Yes | No |
| `public` | Yes | Yes | Yes | Yes |

TypeScript:

| TS modifier | Where reachable |
|---|---|
| `private` | Same class only (compile-time check) |
| `#field` | Same class only (true runtime privacy) |
| `protected` | Class and subclasses |
| `public` (default) | Anywhere |
| `readonly` | Modifies any of the above to forbid reassignment |

Two practical TS notes:

- `private` is a TypeScript-only check. At runtime the field is reachable. `#name` (the ECMAScript private field) is enforced by the runtime — pick `#` when you actually need privacy from JS code.
- `readonly` only forbids reassignment, not mutation of the referenced object. `readonly items: Item[]` lets callers `push` into the array.

```typescript
class Cart {
  readonly #items: Item[] = [];

  get items(): readonly Item[] { return this.#items; }   // truly private + readonly view

  add(item: Item): void {
    if (this.#items.length >= 50) throw new Error("cart full");
    this.#items.push(item);
  }
}
```

### Default visibility — pick the smallest that works

Start `private`. Widen only when you have a concrete consumer. Java's package-private is underused and excellent for "internal to this module" without exposing to the world.

## Getters and setters reality check

The "always make fields private and generate getters and setters for all of them" pattern is a Java-from-2003 caricature. It produces classes that are functionally identical to public-field DTOs while pretending to be encapsulated.

Three honest positions:

1. **Truly private state, no accessors.** The internal field is a hidden implementation detail. Best.
2. **Read-only accessor, no setter.** Callers can observe; they cannot change. Common for value objects.
3. **Public field on a record / DTO.** Honest about being just data. Better than fake encapsulation.

What is almost always wrong:

```java
public class Order {
    private OrderStatus status;
    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus s) { this.status = s; } // any state, any time
}
```

There is no contract here. Any caller can flip status from `DELIVERED` back to `PENDING`. Either:

- Replace `setStatus` with intentional transitions: `markPaid()`, `markShipped()`, `cancel()` — each enforcing valid transitions, or
- Make `Order` immutable and return a new `Order` for each transition (copy-on-write).

```java
public Order markShipped() {
    if (status != OrderStatus.PAID)
        throw new IllegalStateException("must be PAID before SHIPPED");
    return new Order(this.id, this.items, OrderStatus.SHIPPED);
}
```

Setters are not banned. They are appropriate for genuine "free to change" properties on entity objects (e.g. mutable form models). The test is: *does setting this field at any time break any invariant?* If yes, no setter.

## Immutability as encapsulation

Immutable objects are encapsulated by construction. There is no path to invalid state because there is no path to *any* state change at all. Once constructed, the value either is what it should be, or you threw in the constructor.

```java
public record Money(long cents, String currency) {
    public Money {
        if (cents < 0) throw new IllegalArgumentException("non-negative");
        if (currency == null || currency.length() != 3) throw new IllegalArgumentException("ISO 4217");
    }
    public Money add(Money other) {
        if (!currency.equals(other.currency)) throw new IllegalArgumentException("mixed currency");
        return new Money(cents + other.cents, currency);
    }
}
```

```typescript
class Money {
  constructor(readonly cents: number, readonly currency: string) {
    if (cents < 0) throw new Error("non-negative");
    if (currency.length !== 3) throw new Error("ISO 4217");
    Object.freeze(this);
  }
  add(other: Money): Money {
    if (other.currency !== this.currency) throw new Error("mixed currency");
    return new Money(this.cents + other.cents, this.currency);
  }
}
```

Properties of immutable objects:

- Trivially thread-safe (no shared mutable state to coordinate).
- Safe to share aliases across the codebase.
- Cacheable as map keys without surprise.
- Equality is just value equality.
- Easy to reason about — no temporal coupling.

Costs:

- Allocation pressure if you mutate hot loops with copy-on-write.
- Awkward for genuinely large mutable graphs (use persistent data structures or accept localized mutation behind a wall).

For most domain model code, the costs are negligible and the wins are large.

## Encapsulation vs naked records / DTOs

Records (Java), data classes (Kotlin), `interface`-shaped types (TS) intentionally expose all their fields. That is *not* an encapsulation failure — it is a deliberate "this is just data, not a behaviour-bearing object."

Use records / DTOs when:

- The object is a pure data carrier across a boundary (HTTP body, queue message, DB row).
- There are no invariants beyond "the fields exist and have valid types."
- Equality is value equality.

Use a real encapsulated class when:

- There is at least one invariant that ties fields together.
- There is meaningful behaviour beyond getters.
- The lifetime of the object includes legal state transitions.

A common architecture: rich, encapsulated *domain* objects in the core; flat *DTOs* at the edges (controllers, adapters); explicit mapping between them. The DTOs do not pretend to encapsulate anything; the domain objects do not pretend to be transport types.

```java
// DTO at the HTTP boundary — public-feeling, no invariants beyond shape
public record CreateOrderRequest(List<LineItemDto> items, String customerId) {}

// Domain object — encapsulated, enforces business rules
public class Order { /* private state, intentional methods */ }
```

## Common pitfalls

- **Returning mutable internals.** Defensive-copy on the way out, return unmodifiable views, or hold immutable collections internally.
- **Builders that mutate after `build()`.** A "built" object should be frozen. Mutable builder ➜ immutable result, and the builder should be unusable after build (or fresh per use).
- **Reflection and serialization sidesteps.** Jackson, JSON.stringify, Java reflection, and friends can punch through `private`. For domain objects you really want untouchable, design constructors to validate on every entry path — including deserialization.
- **`protected` as a slippery slope.** `protected` is a public API to subclasses. Treat it with the same care as `public`.
- **Assuming `private` in TypeScript means private at runtime.** It doesn't. Use `#field` for true privacy.
- **Treating "encapsulation" as "many small getters."** Encapsulation is about invariants, not the count of accessor methods. Tell, don't ask.

## Related

- [classes-and-objects.md](./classes-and-objects.md) — what state and behavior are
- [abstraction.md](./abstraction.md) — abstraction picks the *level*, encapsulation defends the *boundary*
- [interfaces.md](./interfaces.md) — the public contract of an encapsulated class
- [inheritance.md](./inheritance.md) — `protected` and the fragile base class problem
- [../../java/INDEX.md](../../java/INDEX.md) — records, immutability, defensive copies
- [../../typescript/INDEX.md](../../typescript/INDEX.md) — `#private`, `readonly`, structural escape hatches
