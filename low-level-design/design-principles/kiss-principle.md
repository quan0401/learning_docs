---
title: "KISS — Keep It Simple, Stupid"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, kiss, simplicity, complexity]
---

# KISS — Keep It Simple, Stupid

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `kiss` `simplicity` `complexity`

## Summary

KISS, attributed to Lockheed engineer Kelly Johnson, says systems work best when kept simple, and complexity must be justified rather than assumed. In code this means treating simplicity as a feature, spending a finite "complexity budget" deliberately, and recognizing that "smart" or clever code is usually a symptom rather than an achievement.

## Table of Contents

- [The Origin and Real Meaning](#the-origin-and-real-meaning)
- [Simplicity as a Feature](#simplicity-as-a-feature)
- [The Complexity Budget](#the-complexity-budget)
- [Smart Code Is a Code Smell](#smart-code-is-a-code-smell)
- [Two Kinds of Complexity](#two-kinds-of-complexity)
- [Practical Tactics](#practical-tactics)
- [When Simple Is Wrong](#when-simple-is-wrong)
- [Checklist](#checklist)
- [Related](#related)

## The Origin and Real Meaning

Kelly Johnson designed aircraft (the U-2, SR-71) where field-repairable parts could be the difference between mission success and failure. KISS was not "dumb it down" — it was "design so that an average mechanic with average tools in average conditions can keep this thing flying." The audience and operating environment dictate what *simple* means.

In software, your audience is the next developer (often you, six months from now) and the operating environment is production at 3 AM. *Simple* means:

- Reads top-to-bottom without surprises
- Behaves predictably under failure
- Can be debugged with `grep`, a stack trace, and reading
- Does not require holding the whole system in your head

## Simplicity as a Feature

Simplicity is a *deliverable*, not a side effect. Like performance or security, it costs effort to produce and erodes if not maintained. A senior engineer's job is partly to *remove* code, not just to add it.

Tony Hoare:

> There are two ways of constructing a software design: one way is to make it so simple that there are obviously no deficiencies; the other is to make it so complicated that there are no obvious deficiencies. The first method is far more difficult.

Designing the simple version requires understanding the problem more deeply than designing the elaborate one. Complex code often reflects unfinished thinking, not advanced thinking.

## The Complexity Budget

Every system has a *complexity budget* — a finite tolerance for cognitive load before bugs accelerate, onboarding stalls, and changes start producing unrelated regressions. Spending it badly bankrupts the project.

Where complexity actually pays for itself:

| Worth spending budget on | Not worth it |
|--------------------------|--------------|
| Core domain logic | Generic frameworks for one-time problems |
| Hot performance paths (proven by profiling) | "What if we need to swap the database later" |
| Concurrency and consistency | Plugin systems for nothing |
| Public API ergonomics | Inheritance hierarchies "for extensibility" |
| Failure handling on boundaries | Configuration knobs nobody uses |

Treat each abstraction layer, each indirection, each generic, each design pattern as a withdrawal. If the withdrawal does not buy you something concrete and present, refund it.

## Smart Code Is a Code Smell

"Smart" code is code that is *cleverer than the problem requires*. Symptoms:

- Three-line solution where a fifteen-line one would be obvious
- Heavy use of metaprogramming, decorators, or reflection for simple flow
- Operator overloading or DSLs to make code "read naturally" at the cost of debuggability
- Streams or chained higher-order functions that hide a simple loop

```typescript
// Smart — looks elegant, debugs poorly
const result = orders
  .reduce((acc, o) => ({ ...acc, [o.customerId]: [...(acc[o.customerId] ?? []), o] }), {} as Record<string, Order[]>);

// Simple — reads in one pass, easy to step through
const result: Record<string, Order[]> = {};
for (const order of orders) {
  if (!result[order.customerId]) {
    result[order.customerId] = [];
  }
  result[order.customerId].push(order);
}
```

The second version is longer, but a junior engineer can step through it under a debugger and understand it. The first requires holding three concepts simultaneously and breaks if `acc` is mutated by a future "improvement."

```java
// Smart — generic, reflective, surprising
public <T> T loadEntity(Class<T> type, Object id) {
    var query = em.createQuery(
        "SELECT e FROM " + type.getSimpleName() + " e WHERE e.id = :id", type);
    query.setParameter("id", id);
    return query.getSingleResult();
}

// Simple — explicit, type-safe, won't surprise you
public User findUserById(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));
}
```

The generic version is "DRY" but every caller now pays in lost type safety, lost IDE navigation, and runtime surprises if the entity is renamed.

## Two Kinds of Complexity

Fred Brooks distinguished:

- **Essential complexity** — inherent to the problem. Tax calculation has rules. Distributed systems have partial failure. You cannot delete this complexity, only model it well.
- **Accidental complexity** — created by tools, frameworks, abstractions, and prior decisions. This is what KISS attacks.

A simple system minimizes accidental complexity and surfaces essential complexity *clearly* so it can be reasoned about. Hiding essential complexity behind a clean-looking facade often makes things worse: the complexity returns at the worst time, in the wrong place.

## Practical Tactics

### Prefer functions to classes when possible

A standalone function with no shared state is the simplest unit. Reach for a class when behavior and state belong together, not because OOP says so.

### Prefer flat over nested

```typescript
// Nested — three levels of branching
function processOrder(order: Order) {
  if (order.isValid) {
    if (order.customer.isActive) {
      if (inventory.has(order.items)) {
        // ... actual work
      }
    }
  }
}

// Flat with early returns
function processOrder(order: Order) {
  if (!order.isValid) return reject("invalid order");
  if (!order.customer.isActive) return reject("customer suspended");
  if (!inventory.has(order.items)) return reject("out of stock");

  // ... actual work
}
```

### Prefer data to control flow

Lookup tables and configuration are easier to read and modify than nested conditionals.

```java
// Nested switch
public BigDecimal taxRate(String region, String category) {
    switch (region) {
        case "EU":
            switch (category) {
                case "BOOK": return new BigDecimal("0.05");
                case "FOOD": return new BigDecimal("0.07");
                default: return new BigDecimal("0.20");
            }
        // ...
    }
}

// Data
private static final Map<String, Map<String, BigDecimal>> RATES = Map.of(
    "EU", Map.of("BOOK", bd("0.05"), "FOOD", bd("0.07"), "*", bd("0.20")),
    "US", Map.of("*", bd("0.00"))
);

public BigDecimal taxRate(String region, String category) {
    var regionRates = RATES.getOrDefault(region, Map.of("*", BigDecimal.ZERO));
    return regionRates.getOrDefault(category, regionRates.get("*"));
}
```

### Choose boring tools

For a CRUD service, "boring" Spring Boot + PostgreSQL beats hand-rolled reactive streams 95% of the time. Boring tools have answers on Stack Overflow, predictable failure modes, and people who understand them.

### Delete code

The simplest function is no function. Periodically search for unused code, dead branches, and abstractions that have only one implementation, and remove them.

## When Simple Is Wrong

KISS does not mean *naive*. Some problems demand sophistication:

- **Concurrency:** A "simple" `synchronized` everywhere produces deadlocks. Correct concurrency requires understanding memory models, lock ordering, and atomicity.
- **Security:** Hand-rolled crypto is "simple" and broken. Use vetted libraries.
- **Distributed systems:** Pretending the network is reliable is simple — and produces silent data loss.
- **Performance-critical paths:** A measured-and-justified optimization may be necessary even if it's harder to read. Comment why.

The principle is "as simple as possible, but no simpler" (Einstein, paraphrased). Essential complexity must be modeled honestly.

## Checklist

- [ ] Could a junior on the team read this code top-to-bottom and explain it?
- [ ] Is every abstraction earning its keep right now?
- [ ] Are you using a clever language feature when an obvious one would do?
- [ ] Is the file under 400 lines? Function under 50?
- [ ] Have you removed dead code, single-use abstractions, and unused parameters?
- [ ] If this fails at 3 AM, can someone debug it without you?

## Related

- [DRY — Don't Repeat Yourself](dry-principle.md)
- [YAGNI — You Aren't Gonna Need It](yagni-principle.md)
- [Code Smells and Refactoring Triggers](code-smells-and-refactoring-triggers.md)
- [Anti-Patterns in OO Design](anti-patterns-in-oo-design.md)
- [Coupling and Cohesion](coupling-and-cohesion.md)
- [Single Responsibility Principle](../solid/single-responsibility-principle.md)

## References

- Brooks, F. — *The Mythical Man-Month*, "No Silver Bullet"
- Hoare, C. A. R. — *The Emperor's Old Clothes* (Turing Award lecture)
- Martin, R. — *Clean Code*
- Ousterhout, J. — *A Philosophy of Software Design*
