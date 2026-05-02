---
title: "Anti-Patterns in OO Design"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, anti-patterns, oop-fundamentals, refactoring]
---

# Anti-Patterns in OO Design

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `anti-patterns` `oop-fundamentals` `refactoring`
## Summary

Anti-patterns are recurring "solutions" that look like patterns but consistently produce worse outcomes than naive code. They have names because we keep seeing them in real codebases — recognizing the shape lets you call out the problem and refactor toward a known good alternative. This document covers the most common OO anti-patterns: god object, anemic domain model, spaghetti, lasagna, golden hammer, premature optimization, copy-paste programming, magic numbers, base bean, and circular dependency.

## Table of Contents

- [What Makes an Anti-Pattern](#what-makes-an-anti-pattern)
- [God Object](#god-object)
- [Anemic Domain Model](#anemic-domain-model)
- [Spaghetti Code](#spaghetti-code)
- [Lasagna Code](#lasagna-code)
- [Golden Hammer](#golden-hammer)
- [Premature Optimization](#premature-optimization)
- [Copy-Paste Programming](#copy-paste-programming)
- [Magic Numbers](#magic-numbers)
- [Base Bean](#base-bean)
- [Circular Dependency](#circular-dependency)
- [Checklist](#checklist)
- [Related](#related)

## What Makes an Anti-Pattern

A pattern becomes an *anti-pattern* when it has all of:

1. A repeated, recognizable shape across many codebases.
2. Looks like a reasonable approach in the moment.
3. Reliably produces worse outcomes than alternatives.
4. Has a known refactoring path out of it.

Naming anti-patterns matters because it makes review feedback short and unambiguous: "this is a god object" carries shared meaning. Refactoring is easier when the destination has a name too.

## God Object

A class that knows or does too much — every responsibility, every dependency, every entry point eventually flows through it.

```typescript
class ApplicationManager {
  // database
  db: Database;
  // sessions
  sessions: Map<string, User>;
  // routing
  routes: Route[];
  // permissions
  permissions: PermissionMatrix;
  // billing, email, logging, caching, feature flags, audit, analytics...

  login() {} logout() {} createUser() {} deleteUser() {}
  charge() {} refund() {} sendEmail() {} log() {}
  cacheGet() {} cacheSet() {} flagOn() {}
  // 200+ methods
}
```

**Why it appears:** organic growth without resistance. Everyone adds "just one more thing" to the central class. Easier than introducing a new collaborator.

**Why it's bad:** every change risks unrelated regressions; tests require enormous setup; dependencies are unmanageable; merge conflicts are constant.

**Refactor toward:** Single Responsibility — extract focused services along business or technical seams. Composition over a central manager.

## Anemic Domain Model

Domain objects with no behavior — only fields and getters/setters. All logic lives in "service" classes that operate on the bag of data.

```java
public class Order {
    private OrderId id;
    private List<Item> items;
    private OrderStatus status;
    // getters and setters only — no behavior
}

public class OrderService {
    public BigDecimal calculateTotal(Order order) { /* ... */ }
    public void cancel(Order order) { /* mutates fields directly */ }
    public boolean canBeShipped(Order order) { /* logic about Order, not on Order */ }
}
```

Martin Fowler called this an anti-pattern because it inverts what objects are *for*: an object packages data with the operations on it. Anemic models are procedural code wearing OO clothing.

**Why it appears:** misunderstood DDD; layered architecture taken too literally ("entities are dumb data; services do the work"); ORM frameworks that nudge you toward bare data classes.

**Why it's bad:** invariants are not enforced — anyone can put `Order` in an inconsistent state by setting fields. Logic about an entity is scattered across services. Encapsulation is broken.

**Refactor toward:** rich domain models. Behavior moves onto the entity (`order.calculateTotal()`, `order.cancel()`, `order.canBeShipped()`). State changes go through methods that enforce invariants. Services orchestrate, they don't reach into entities.

## Spaghetti Code

Control flow tangled across files, methods, and conditions, with no clear entry or exit. Reading it requires jumping around without ever feeling oriented.

Symptoms:

- Deep nesting (5+ levels) of `if`/`while`/`try`.
- `goto`-equivalent flow control: deeply nested `break`/`continue`, exceptions used as flow.
- Side effects scattered through functions that look pure.
- Global state mutated from many places.

```typescript
// Spaghetti
function process(input: any): any {
  if (input.type === "A") {
    for (const x of input.items) {
      if (x.flag) {
        try {
          if (someGlobalCache[x.id]) {
            x.processed = true;
            globalQueue.push(x);
            return;
          }
          // ...
        } catch (e) { /* ... */ }
      } else if (x.other) {
        // ... another deep branch
      }
    }
  } else if (input.type === "B") { /* ... */ }
  // ...
}
```

**Refactor toward:** early returns, named extracted functions, polymorphism in place of nested type-switches, pure functions where possible, replacing global state with explicit dependencies.

## Lasagna Code

The opposite of spaghetti: too many *layers*. Each operation passes through six abstractions before doing anything. Every concept has a controller, service, façade, mapper, DTO, repository, entity, and value object — even when the operation is trivial.

```
HTTP request
  → Controller
    → ApplicationService
      → DomainService
        → DomainFacade
          → Repository
            → DataMapper
              → DAO
                → ORM
                  → SQL
```

**Why it appears:** dogmatic application of Clean Architecture, Hexagonal, DDD, or layered patterns to projects that don't need that depth. "Best practices" cargo-cult.

**Why it's bad:** every change requires editing multiple layers; debugging requires tracing through indirection; the cost-of-change is multiplied by layer count.

**Refactor toward:** collapse layers that don't earn their keep. A simple CRUD endpoint may not need separate application, domain, and infrastructure services. Add layers when complexity demands them, not preemptively.

## Golden Hammer

Using a familiar tool, framework, or pattern for every problem, even when it's a poor fit.

> "If all you have is a hammer, everything looks like a nail." — Maslow's hammer

Examples:

- Using Spring Beans, dependency injection, and AOP for a 100-line script.
- Modeling every problem as a state machine because the team just learned about XState.
- Forcing every domain into a graph database because graphs are interesting.
- Putting Kafka in front of every internal call.

**Why it appears:** human bias toward what we know; resume-driven development; team momentum.

**Why it's bad:** the tool's complexity is paid even when its benefits don't apply. Worse, alternatives never get evaluated.

**Refactor toward:** evaluate tools per problem. The smallest tool that solves the problem reliably is usually the right one. Resist adopting frameworks until you've felt the pain they solve.

## Premature Optimization

Donald Knuth: *"Premature optimization is the root of all evil (or at least most of it) in programming."*

Optimization without measurement, applied to code that isn't on the hot path, at the cost of clarity. Symptoms:

- Inlining everything "for performance" before profiling.
- Hand-rolling caches around code that runs once per day.
- Replacing readable list operations with mutable arrays "to avoid allocations" in non-critical code.
- Choosing a complex data structure for "scalability" when the input size is bounded.

```java
// Premature — micro-optimizing a startup function
public List<String> loadConfigKeys() {
    final int initialCapacity = 16;
    final ArrayList<String> result = new ArrayList<>(initialCapacity);
    final char[] buffer = new char[1024];
    // ... reads a config file *once* at boot
}

// Plain version is fine
public List<String> loadConfigKeys() {
    return Files.readAllLines(configPath);
}
```

**Why it's bad:** optimized code is harder to read, debug, and change — and the optimization usually doesn't matter. The 95% of code that isn't on a hot path should be optimized for the *reader*.

**Refactor toward:** measure first. Profile in realistic conditions. Optimize the few percent that actually matters. Comment optimizations clearly so they survive future "simplification" passes.

## Copy-Paste Programming

Solving a new problem by copying code from another place and tweaking it. Each copy diverges. When the underlying logic needs a fix, only some copies get the fix.

This is the inverse of DRY done badly: the duplication isn't recognized as duplication.

**Why it appears:** convenient, fast, makes it easy to "ship now and refactor later" — and "later" never comes.

**Why it's bad:** bugs survive in some copies after being fixed in others; security patches are missed; behavior drifts.

**Refactor toward:** when you're tempted to copy-paste, pause. If the third copy is about to happen (rule of three), extract. If you must paste now, leave a clear marker — `// TODO: extract shared helper for X` — and *actually do it*.

## Magic Numbers

Unexplained numeric or string literals scattered through the code.

```typescript
// Magic numbers
if (user.loginAttempts > 5) { lock(user); }
setTimeout(() => poll(), 30000);
if (order.total > 10000) { requireApproval(order); }
```

What is `5`? Why `30000`? Why `10000`?

**Refactor toward:** named constants that capture the *meaning*, not just the value.

```typescript
const MAX_LOGIN_ATTEMPTS = 5;
const POLL_INTERVAL_MS = 30_000;
const HIGH_VALUE_ORDER_THRESHOLD = 10_000;

if (user.loginAttempts > MAX_LOGIN_ATTEMPTS) { lock(user); }
setTimeout(() => poll(), POLL_INTERVAL_MS);
if (order.total > HIGH_VALUE_ORDER_THRESHOLD) { requireApproval(order); }
```

The same applies to magic strings: use enums or constants. Magic numbers are also a frequent source of duplicate knowledge — the same threshold encoded in three places, with three different values, drifting silently.

## Base Bean

Inheriting from a "useful" utility base class — `BaseService`, `BaseEntity`, `AbstractController` — to get its features, instead of using composition.

```java
// Base bean — every entity inherits from BaseEntity
public abstract class BaseEntity {
    protected Long id;
    protected Instant createdAt;
    protected Instant updatedAt;
    protected String createdBy;
    protected Long version;
    public void touch() { updatedAt = Instant.now(); }
    // 30 protected helpers
}

public class Order extends BaseEntity { /* ... */ }
public class Customer extends BaseEntity { /* ... */ }
public class Product extends BaseEntity { /* ... */ }
```

**Why it's bad:** locks every entity into a single inheritance chain; couples them to a class that will grow over time; breaks when you need a non-`BaseEntity` (e.g., a value object that doesn't have `id` semantics); the base class becomes a god object by accretion.

**Refactor toward:** composition. Embed the audit fields as a value object (`AuditMetadata`), or use a JPA `@Embeddable`, or wrap with a generic `Auditable<T>`. Reserve inheritance for genuine "is-a" relationships.

## Circular Dependency

Two or more modules depend on each other, directly or transitively. `A` imports `B`; `B` imports `A`. Often hidden behind a third module: `A → B → C → A`.

**Why it's bad:**

- Compile / build order becomes ambiguous.
- Modules can't be reasoned about, tested, or deployed independently.
- Changes in one trigger changes in the other; the cycle defeats modularity.
- Unit testing requires loading the entire cycle.

**Refactor toward:**

- **Extract a third module** that both depend on (often a shared interface or value type).
- **Apply Dependency Inversion:** the high-level module defines an interface; the low-level module implements it. The cycle becomes a one-way arrow.
- **Move shared logic up or down** the layer stack so the cycle disappears.

```java
// Cycle: Order knows Customer (for customer.applyDiscount), Customer knows Order (for order.totalSpent)

// Break with an interface
public interface DiscountPolicy { BigDecimal apply(BigDecimal subtotal); }
public class Customer implements DiscountPolicy { /* ... */ }

public class Order {
    public BigDecimal calculateTotal(DiscountPolicy policy) {
        return policy.apply(this.subtotal());
    }
}
// Now Order doesn't depend on Customer at all.
```

Tools like dependency-cruiser (TypeScript), JDepend (Java), and ESLint's `import/no-cycle` flag cycles automatically — make them part of CI.

## Checklist

- [ ] Is any one class doing far too much? (god object)
- [ ] Are domain objects mere data bags with logic in services? (anemic domain model)
- [ ] Is control flow nested 5+ levels deep with shared mutable state? (spaghetti)
- [ ] Are there too many layers passing data through with no decisions? (lasagna)
- [ ] Are you reaching for the same tool regardless of fit? (golden hammer)
- [ ] Are you optimizing without profiling? (premature optimization)
- [ ] Have you copy-pasted the same logic three or more times? (copy-paste)
- [ ] Are there unexplained literals in the code? (magic numbers)
- [ ] Are entities all inheriting from a growing base class? (base bean)
- [ ] Does the dep graph contain any cycles? (circular dependency)

## Related

- [DRY — Don't Repeat Yourself](dry-principle.md)
- [KISS — Keep It Simple, Stupid](kiss-principle.md)
- [YAGNI — You Aren't Gonna Need It](yagni-principle.md)
- [Law of Demeter](law-of-demeter.md)
- [Separation of Concerns](separation-of-concerns.md)
- [Coupling and Cohesion](coupling-and-cohesion.md)
- [Composing Objects Principle](composing-objects-principle.md)
- [Code Smells and Refactoring Triggers](code-smells-and-refactoring-triggers.md)
- [Single Responsibility Principle](../solid/single-responsibility-principle.md)
- [Liskov Substitution Principle](../solid/liskov-substitution-principle.md)
- [Dependency Inversion Principle](../solid/dependency-inversion-principle.md)
- [Encapsulation](../oop-fundamentals/encapsulation.md)
- [Inheritance](../oop-fundamentals/inheritance.md)

## References

- Brown, W. et al. — *AntiPatterns: Refactoring Software, Architectures, and Projects in Crisis*
- Fowler, M. — *Refactoring*, "AnemicDomainModel" essay
- Knuth, D. — "Structured Programming with go to Statements" (origin of premature-optimization quote)
- Martin, R. — *Clean Code*, *Clean Architecture*
- Bloch, J. — *Effective Java*
