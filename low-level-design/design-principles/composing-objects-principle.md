---
title: "Composing Objects Principle — Composition Over Inheritance"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, composition, inheritance, gof]
---

# Composing Objects Principle — Composition Over Inheritance

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `composition` `inheritance` `gof`

## Summary

The Composing Objects Principle, captured in the *Design Patterns* (GoF) book as **"favor object composition over class inheritance,"** says new behavior should normally be assembled by combining objects rather than by extending a class. Inheritance creates a tight, compile-time, single-axis "is-a" relationship that becomes brittle as requirements grow; composition creates flexible, runtime-replaceable, multi-axis "has-a" relationships through interfaces.

## Table of Contents

- [The Two Reuse Mechanisms](#the-two-reuse-mechanisms)
- [Why Inheritance Gets Brittle](#why-inheritance-gets-brittle)
- [Composition + Interfaces as the Default](#composition--interfaces-as-the-default)
- [GoF Guidance](#gof-guidance)
- [When Inheritance Is Right](#when-inheritance-is-right)
- [Concrete Refactor](#concrete-refactor)
- [Tooling: Mixins, Traits, Default Methods](#tooling-mixins-traits-default-methods)
- [Checklist](#checklist)
- [Related](#related)

## The Two Reuse Mechanisms

Object-oriented systems have two main ways to reuse behavior:

- **Class inheritance** — a subclass extends a superclass, inheriting its fields and methods. Relationship is "is-a." Resolved at compile time. Single inheritance in Java; multiple inheritance in C++. The relationship is fixed when the class is declared.

- **Object composition** — an object holds references to other objects and delegates work to them. Relationship is "has-a" or "uses-a." Resolved at runtime. The composed parts can be swapped without recompiling the holder.

Both are forms of reuse. The principle says: **default to composition**.

## Why Inheritance Gets Brittle

### Inheritance breaks encapsulation

A subclass is intimate with its superclass — it sees protected fields, can override methods, and depends on internal call sequences ("template methods"). Changes to the superclass that are perfectly valid as far as its public contract is concerned can break subclasses. *Effective Java* (Joshua Bloch, Item 18: "Favor composition over inheritance") names this problem precisely.

### Inheritance is single-axis

A class can be one thing in a hierarchy. If you need behavior from two orthogonal axes — e.g., a `LoggedAuditedReplayableHttpClient` — single inheritance forces you to pick one axis as the parent and bake the other in. With composition, you can decorate independently along multiple axes.

### Inheritance is static

Inheritance relationships are decided at class declaration. You cannot change them at runtime. With composition, an object can swap out its strategy or collaborator at any time.

### The "yo-yo problem"

In deep hierarchies, understanding what a method does requires bouncing up and down the chain — superclass calls subclass override calls superclass helper, and so on. Reading a method becomes navigating an invisible call graph.

### Fragile base class

If a base class changes (renames a method, adds a parameter, changes call ordering), every subclass — including ones in third-party code — may break. The blast radius is unbounded.

```java
// Classic fragile base class
public class Counter {
    protected int count = 0;
    public void add(int x) { count += x; }
    public void addAll(Collection<Integer> xs) { for (int x : xs) add(x); }
}

public class LoggingCounter extends Counter {
    @Override public void add(int x) { System.out.println(x); super.add(x); }
    @Override public void addAll(Collection<Integer> xs) {
        for (int x : xs) System.out.println(x);
        super.addAll(xs);
    }
}
```

`LoggingCounter` logs each item *twice* because `addAll` calls `add`. The override depends on the *internal* implementation of the superclass. If `Counter` later changes `addAll` to not call `add`, log output silently changes too.

## Composition + Interfaces as the Default

The composition pattern: **define behavior as an interface; have a class that needs that behavior hold a reference to an implementation**. The class delegates rather than inherits.

```java
// Interface for the behavior
public interface Logger {
    void log(String message);
}

// Implementations
public class ConsoleLogger implements Logger {
    public void log(String message) { System.out.println(message); }
}

public class FileLogger implements Logger {
    public void log(String message) { /* write to file */ }
}

// Class that needs logging — composes, doesn't inherit
public class OrderService {
    private final Logger logger;
    public OrderService(Logger logger) { this.logger = logger; }
    public void placeOrder(Order o) {
        logger.log("placing order " + o.getId());
        // ...
    }
}
```

Benefits:

- **Swap at runtime:** `new OrderService(testLogger)` for tests, `new OrderService(prodLogger)` in production.
- **Multiple axes:** add caching by introducing a `Cache` collaborator without affecting logging.
- **Encapsulation preserved:** `OrderService` only sees `Logger`'s public interface.
- **No fragile base class:** `Logger` implementations evolve independently.

```typescript
// Same pattern in TypeScript
interface Notifier {
  send(to: string, message: string): Promise<void>;
}

class EmailNotifier implements Notifier {
  async send(to: string, msg: string) { /* SMTP */ }
}
class SlackNotifier implements Notifier {
  async send(to: string, msg: string) { /* Slack API */ }
}

class OrderService {
  constructor(private notifier: Notifier) {}
  async placeOrder(order: Order) {
    // ...
    await this.notifier.send(order.customerEmail, "Order placed");
  }
}
```

## GoF Guidance

The original *Design Patterns* book by Gamma, Helm, Johnson, and Vlissides codifies the principle. Quoting their summary:

> Favor object composition over class inheritance.

The book's patterns are largely applications of this idea:

- **Strategy** — replace inheritance hierarchies that vary one behavior with a composed strategy interface.
- **State** — represent state-dependent behavior as composed state objects rather than nested conditionals or class-per-state hierarchies.
- **Decorator** — wrap composed objects to add behavior, instead of subclassing.
- **Composite** — build tree structures by composing the same interface, not by inheriting from concrete tree nodes.
- **Bridge** — separate abstraction from implementation so they can vary independently, via composition.

Whenever you see a `*Strategy`, `*State`, `*Handler`, `*Notifier` interface with multiple implementations and a class that holds one as a field — that is the principle in action.

## When Inheritance Is Right

The principle is "favor composition," not "never inherit." Inheritance is the right tool when:

### True is-a relationship within a single domain

If `Square` is genuinely a kind of `Rectangle` (mathematically) — but note this is also the classic Liskov example where naive inheritance breaks. True Liskov-substitutable subtypes are rarer than they seem.

### Framework template methods designed for extension

When a base class is *explicitly designed for inheritance* — documented extension points, sealed-otherwise non-overridable methods, javadoc/jsdoc indicating the intended overrides — inheritance is the cleanest mechanism.

```java
public abstract class HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse res) { /* default */ }
    protected void doPost(HttpServletRequest req, HttpServletResponse res) { /* default */ }
    // ...
}
```

Servlet authors override `doGet` / `doPost`. The base is designed for it.

### Type hierarchy modeling

Algebraic-style hierarchies — `Shape` with `Circle`, `Square`, `Triangle`; `Result` with `Success` and `Failure` — express a closed sum type. In Java, `sealed` classes (since Java 17) make the closed nature explicit.

```java
public sealed interface Shape permits Circle, Square, Triangle { }
public record Circle(double radius) implements Shape { }
public record Square(double side) implements Shape { }
public record Triangle(double a, double b, double c) implements Shape { }
```

This is closer to algebraic data types than to classical OO inheritance and is generally fine.

### Sharing common state inside a tightly-controlled hierarchy

Within one team, one module, one closed hierarchy — inheritance can be a clean way to share state and template behavior. The danger is when subclasses live across module / team / package boundaries.

The general guidance: **inherit only when you control both the parent and the child, the relationship is genuinely "is-a," and the parent was designed for extension**. Otherwise, compose.

## Concrete Refactor

A typical inheritance-gone-wrong scenario:

```java
// Before — inheritance hierarchy modeling employee types
public abstract class Employee {
    protected String name;
    public abstract BigDecimal calculateBonus();
    public abstract void grantTimeOff(int days);
}

public class FullTimeEmployee extends Employee { /* ... */ }
public class PartTimeEmployee extends Employee { /* ... */ }
public class Contractor extends Employee { /* ... */ }
public class Intern extends Employee { /* ... */ }

// And then a contractor who is *also* an intern? A part-time who switches to full-time?
// You're stuck.
```

Refactor to composition:

```java
public class Employee {
    private final String name;
    private final CompensationPolicy compensation;
    private final TimeOffPolicy timeOff;

    public Employee(String name, CompensationPolicy comp, TimeOffPolicy off) {
        this.name = name;
        this.compensation = comp;
        this.timeOff = off;
    }

    public BigDecimal calculateBonus() { return compensation.bonus(this); }
    public void grantTimeOff(int days) { timeOff.grant(this, days); }
    // policies can change without changing Employee's class
}

public interface CompensationPolicy { BigDecimal bonus(Employee e); }
public interface TimeOffPolicy { void grant(Employee e, int days); }
```

Now an employee can switch from contractor to full-time by replacing one collaborator. Two orthogonal axes (compensation, time off) are independently configurable.

## Tooling: Mixins, Traits, Default Methods

Some languages provide intermediate forms:

- **Java default methods on interfaces** — let interfaces carry default implementations. Useful for backwards-compatible interface evolution. Don't abuse them as multiple inheritance.

- **Kotlin / Scala traits** — composable units of behavior. Closer to mixins than to traditional inheritance.

- **TypeScript mixins** — function-based class augmentation. Powerful but obscure; prefer plain composition.

These tools blur the line, but the guidance remains: when in doubt, hold a collaborator instead of becoming a subtype.

## Checklist

- [ ] Are you about to extend a class to add behavior? Could you hold a collaborator instead?
- [ ] Does the parent class advertise that it was designed for extension (sealed, abstract template methods, documentation)?
- [ ] Will subclasses live across module/team/library boundaries? Then inheritance is risky.
- [ ] Is the "is-a" relationship genuine, or is it really "uses" or "has"?
- [ ] Will requirements add a second orthogonal axis later? Composition handles it; inheritance doesn't.
- [ ] If swapping the behavior at runtime is plausible, you need composition.
- [ ] Are you fighting the "fragile base class" problem? Convert to composition.

## Related

- [Coupling and Cohesion](coupling-and-cohesion.md)
- [Separation of Concerns](separation-of-concerns.md)
- [Law of Demeter](law-of-demeter.md)
- [Code Smells and Refactoring Triggers](code-smells-and-refactoring-triggers.md)
- [Anti-Patterns in OO Design](anti-patterns-in-oo-design.md)
- [Liskov Substitution Principle](../solid/liskov-substitution-principle.md)
- [Open/Closed Principle](../solid/open-closed-principle.md)
- [Interface Segregation Principle](../solid/interface-segregation-principle.md)
- [Dependency Inversion Principle](../solid/dependency-inversion-principle.md)
- [Inheritance](../oop-fundamentals/inheritance.md)
- [Polymorphism](../oop-fundamentals/polymorphism.md)
- [Composition](../class-relationships/composition.md)

## References

- Gamma, E., Helm, R., Johnson, R., Vlissides, J. — *Design Patterns: Elements of Reusable Object-Oriented Software* (GoF)
- Bloch, J. — *Effective Java*, Item 18 ("Favor composition over inheritance")
- Martin, R. — *Clean Code*, *Agile Software Development*
- Fowler, M. — *Refactoring*
