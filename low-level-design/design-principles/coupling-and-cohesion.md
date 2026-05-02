---
title: "Coupling and Cohesion"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, coupling, cohesion, modularity]
---

# Coupling and Cohesion

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `coupling` `cohesion` `modularity`

## Summary

Coupling measures how strongly modules depend on each other; cohesion measures how strongly the elements inside one module belong together. The classic engineering rule, dating to Larry Constantine's structured design work in the 1970s, is **high cohesion, low coupling**: modules should be tightly focused inside and loosely connected outside. Coupling and cohesion together explain why a codebase is — or is not — easy to change.

## Table of Contents

- [Why These Two Metrics Matter](#why-these-two-metrics-matter)
- [Coupling: Types and Severity](#coupling-types-and-severity)
- [Afferent and Efferent Coupling](#afferent-and-efferent-coupling)
- [Cohesion: Types and Severity](#cohesion-types-and-severity)
- [The Combined Rule](#the-combined-rule)
- [Concrete Examples](#concrete-examples)
- [Measuring in Practice](#measuring-in-practice)
- [Checklist](#checklist)
- [Related](#related)

## Why These Two Metrics Matter

The cost of changing software is largely determined by how concentrated the change is. If you can change one module to deliver a feature, change is cheap. If you must touch ten modules across three layers, change is expensive.

- **Low cohesion** means a single module mixes unrelated responsibilities. Any change risks unrelated regressions.
- **High coupling** means a change to one module forces changes to many others. The blast radius is large.

Together, low cohesion and high coupling produce code where every change feels like surgery. High cohesion and low coupling produce code where most changes are local.

## Coupling: Types and Severity

Coupling is *not* binary. It comes in degrees, from worst to best (the standard structured-design taxonomy):

### Content coupling (worst)

One module reaches into another's internals — modifying its private state, jumping into its instructions. In modern OO this looks like reflection abuse:

```java
// Reaching past encapsulation
Field secret = customer.getClass().getDeclaredField("internalCreditScore");
secret.setAccessible(true);
secret.set(customer, 850);
```

### Common coupling

Multiple modules share global state. A change to the global affects every reader.

```typescript
// Shared mutable global
export const appState = { user: null as User | null, theme: "light" };
// Twenty files mutate appState directly
```

### External coupling

Modules depend on the same external format, device, or protocol. Format change cascades.

### Control coupling

A caller passes a flag that tells the callee what to do — meaning the caller knows the internal logic of the callee.

```typescript
// Caller controls callee's branching
function process(items: Item[], mode: "fast" | "thorough" | "debug") {
  if (mode === "fast") { /* ... */ }
  else if (mode === "thorough") { /* ... */ }
  else { /* ... */ }
}
```

Better: split into separate functions, or use polymorphism / strategy.

### Stamp coupling

Module A passes a whole record to module B, but B uses only one field. B is now coupled to fields it doesn't need.

```java
// validateEmail takes the whole user — coupled to User shape
public boolean validateEmail(User user) {
    return user.getEmail().contains("@");
}

// Better: pass only what you need
public boolean validateEmail(String email) { /* ... */ }
```

### Data coupling

Module A passes only the necessary primitives to B. Minimal coupling. This is the goal.

```typescript
function totalDiscount(amount: number, ratePct: number): number {
  return amount * (ratePct / 100);
}
```

### No coupling (loose ideal)

Modules don't know about each other at all. Achieved via events, message buses, or independent processes.

The progression: **content > common > external > control > stamp > data > none**, with later items being *better* (lower coupling).

## Afferent and Efferent Coupling

Robert Martin's metrics:

- **Afferent coupling (`Ca`)** — number of modules *that depend on* this module ("incoming" arrows). High Ca = many things rely on me; I am stable, hard to change.
- **Efferent coupling (`Ce`)** — number of modules *this module depends on* ("outgoing" arrows). High Ce = I rely on many things; I am unstable, easy to break.

From these:

- **Instability** `I = Ce / (Ca + Ce)` — 0 means maximally stable (no outgoing deps), 1 means maximally unstable (all outgoing).

The healthy pattern: foundational utilities and domain interfaces have *high Ca, low Ce* (stable). Outer layers like controllers have *low Ca, high Ce* (unstable, easy to change). When unstable modules are depended on by stable ones, you have an inverted dependency — a refactor smell.

```
┌──────────────┐
│ Controllers  │   high Ce, low Ca   ← changes often, depends on much
└──────┬───────┘
       │
┌──────▼───────┐
│ Use cases    │
└──────┬───────┘
       │
┌──────▼───────┐
│ Domain       │   low Ce, high Ca   ← changes rarely, depended on by much
└──────────────┘
```

## Cohesion: Types and Severity

Cohesion is also a spectrum, from worst to best:

### Coincidental cohesion (worst)

Elements grouped for no reason — `Utils.java` with random helpers. The module has no theme.

```java
public class Utils {
    public static String capitalize(String s) { /* ... */ }
    public static int parsePort(String s) { /* ... */ }
    public static boolean isWeekend(LocalDate d) { /* ... */ }
    public static byte[] hashSha256(byte[] in) { /* ... */ }
}
```

### Logical cohesion

Elements share a category but not behavior. Often parameterized by a flag.

```typescript
function inputHandler(kind: "keyboard" | "mouse" | "touch", e: Event) {
  if (kind === "keyboard") { /* ... */ }
  else if (kind === "mouse") { /* ... */ }
  else { /* ... */ }
}
```

### Temporal cohesion

Elements grouped because they happen at the same time — `initialize()` that sets up logging, opens a database, starts a scheduler, registers signal handlers. They run together but are otherwise unrelated.

### Procedural cohesion

Elements run in sequence as part of a procedure but operate on different data.

### Communicational cohesion

Elements operate on the same data — better, because data drives the grouping.

```java
public class OrderReport {
    public OrderReport(Order order) { /* ... */ }
    public BigDecimal calculateTotal() { /* ... */ }
    public BigDecimal calculateTax() { /* ... */ }
    public BigDecimal calculateShipping() { /* ... */ }
}
```

All operate on the same order.

### Sequential cohesion

Output of one element is input to the next, all toward one purpose. Natural pipelines.

### Functional cohesion (best)

All elements contribute to a single, well-defined task.

```typescript
class PriceCalculator {
  constructor(private taxTable: TaxTable, private discountTable: DiscountTable) {}
  calculate(items: Item[], customer: Customer): Price {
    const subtotal = this.subtotal(items);
    const discount = this.discount(subtotal, customer);
    const tax = this.tax(subtotal - discount, customer.region);
    return { subtotal, discount, tax, total: subtotal - discount + tax };
  }
  private subtotal(items: Item[]): number { /* ... */ }
  private discount(amount: number, c: Customer): number { /* ... */ }
  private tax(amount: number, region: string): number { /* ... */ }
}
```

Every method serves "calculate the price." Functional cohesion is the target.

The progression: **coincidental < logical < temporal < procedural < communicational < sequential < functional**, with later items being *better* (higher cohesion).

## The Combined Rule

The two principles work together:

> **High cohesion, low coupling.**

A module should:

- Do one well-defined thing (high cohesion)
- Need to know as little as possible about other modules (low coupling)
- Be known by as little as possible — exposing a minimal API (encapsulation, low afferent coupling on internals)

When you violate either, the symptoms compound:

| | Low coupling | High coupling |
|---|--------------|---------------|
| **High cohesion** | Ideal — easy to change, replace, reuse | Hard to extract — internally good but tangled with neighbors |
| **Low cohesion** | "Junk drawer" modules — internally messy but isolated | Worst case — messy and entangled, every change ripples |

## Concrete Examples

### Refactoring toward higher cohesion

```typescript
// Low cohesion — `OrderHelper` does unrelated things
class OrderHelper {
  validateOrder(o: Order) { /* ... */ }
  sendOrderEmail(o: Order) { /* ... */ }
  parseDateString(s: string) { /* ... */ }
  hashCustomerId(id: string) { /* ... */ }
}

// Split by cohesion theme
class OrderValidator { validate(o: Order) { /* ... */ } }
class OrderNotifier { sendConfirmation(o: Order) { /* ... */ } }
// `parseDateString` and `hashCustomerId` move to genuinely shared utils,
// each in a focused module.
```

### Refactoring toward lower coupling

```java
// High coupling — service knows concrete repository, concrete email client
public class OrderService {
    private final PostgresOrderRepository repo = new PostgresOrderRepository();
    private final SendgridClient email = new SendgridClient(API_KEY);
    public void placeOrder(Order o) {
        repo.save(o);
        email.send(o.getCustomerEmail(), "Order placed");
    }
}

// Lower coupling — depends on interfaces; injected
public class OrderService {
    private final OrderRepository repo;
    private final Notifier notifier;
    public OrderService(OrderRepository repo, Notifier notifier) {
        this.repo = repo;
        this.notifier = notifier;
    }
    public void placeOrder(Order o) {
        repo.save(o);
        notifier.orderPlaced(o);
    }
}
```

The service no longer cares whether the repo is Postgres or Mongo, or whether the notifier sends email or webhooks. It depends on data — the contract — not on implementations.

## Measuring in Practice

You don't usually compute coupling metrics by hand. Tools that help:

- **Java:** SonarQube, JDepend, ArchUnit (architectural rules in tests).
- **TypeScript / JavaScript:** ESLint with `import/no-cycle`, dependency-cruiser, madge.
- **General:** static analysis that produces module dependency graphs.

Useful checks even without tools:

- **Import lists:** if a class imports 30 other classes, efferent coupling is too high.
- **Cyclic dependencies:** indicate either wrong layering or wrong cohesion.
- **Files that change together in git history:** if `OrderService` and `InvoiceController` always change together, the boundary between them is wrong — they probably share a hidden concern.
- **Diff size per feature:** features that touch one or two files are healthy; features that touch fifteen suggest scattered concerns.

## Checklist

- [ ] Does each module do one well-defined thing? (cohesion)
- [ ] Are dependencies on interfaces or abstractions, not concretes? (coupling)
- [ ] Are imports minimal — does the module depend on only what it needs? (data coupling)
- [ ] Do dependencies point in one direction? No cycles?
- [ ] Are stable modules at the bottom and unstable at the top of the dep graph?
- [ ] Is shared state avoided in favor of explicit parameters? (no common coupling)
- [ ] Do callers avoid passing flags that control branching? (no control coupling)
- [ ] In git, do features tend to touch a focused set of files, not scatter widely?

## Related

- [Separation of Concerns](separation-of-concerns.md)
- [Law of Demeter](law-of-demeter.md)
- [Composing Objects Principle](composing-objects-principle.md)
- [DRY — Don't Repeat Yourself](dry-principle.md)
- [Code Smells and Refactoring Triggers](code-smells-and-refactoring-triggers.md)
- [Anti-Patterns in OO Design](anti-patterns-in-oo-design.md)
- [Single Responsibility Principle](../solid/single-responsibility-principle.md)
- [Dependency Inversion Principle](../solid/dependency-inversion-principle.md)
- [Encapsulation](../oop-fundamentals/encapsulation.md)

## References

- Constantine, L. & Yourdon, E. — *Structured Design*
- Stevens, W., Myers, G., Constantine, L. — "Structured Design" (IBM Systems Journal, 1974)
- Martin, R. — *Clean Architecture*, *Agile Software Development*
- Fowler, M. — *Refactoring*
