---
title: "Single Responsibility Principle (SRP)"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, solid, srp, oop, design-principles]
---

# Single Responsibility Principle (SRP)

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `solid` `srp` `oop` `design-principles`

## Summary

The Single Responsibility Principle states that a class (or module) should have one, and only one, reason to change. Robert C. Martin's modern phrasing reframes "responsibility" as "axis of change driven by a single actor" — a department, a stakeholder group, or a role. SRP is less about counting methods and more about asking *whose* request would force this code to change.

## Table of Contents

- [Origin and the "Reason to Change" Definition](#origin-and-the-reason-to-change-definition)
- [The Actor Interpretation](#the-actor-interpretation)
- [God Class Smell and How It Forms](#god-class-smell-and-how-it-forms)
- [Splitting a God Class](#splitting-a-god-class)
- [SRP at the Module Level](#srp-at-the-module-level)
- [Common Misreadings](#common-misreadings)
- [Related](#related)

## Origin and the "Reason to Change" Definition

SRP was popularized by Robert C. Martin in *Agile Software Development, Principles, Patterns, and Practices* (2002) and refined in *Clean Architecture* (2017). The classic formulation:

> A class should have only one reason to change.

The trap is that "reason" is fuzzy. A naive read leads people to chop classes into useless one-method shells. The principle is about *coupling to sources of change*, not method count.

## The Actor Interpretation

In *Clean Architecture*, Martin sharpens the rule:

> A module should be responsible to one, and only one, actor.

An **actor** is a stakeholder or group whose needs drive change: the finance team, the security officer, the database administrator, the marketing team. Code that serves two actors becomes a battleground when their requirements conflict.

The canonical example is `Employee` with three methods: `calculatePay()`, `reportHours()`, `save()`. Each method serves a different actor:

- `calculatePay()` — the **CFO / accounting** team
- `reportHours()` — the **COO / operations** team
- `save()` — the **CTO / DBAs**

If `calculatePay()` and `reportHours()` both call a private helper `regularHours()`, and accounting changes the rules for regular hours, operations' report silently breaks. Two actors, one class, hidden coupling.

```java
// Violates SRP: serves three actors
public class Employee {
    public Money calculatePay() { /* CFO logic */ }
    public String reportHours() { /* COO logic */ }
    public void save(DbConnection db) { /* DBA logic */ }
}
```

## God Class Smell and How It Forms

A "god class" knows too much and does too much. They rarely start that way — they accrete:

1. A `User` class begins with identity fields.
2. Someone adds password hashing — "it's about users, right?"
3. Someone adds permission checks.
4. Someone adds session token generation.
5. Someone adds notification preferences and email sending.
6. Six months later, every change to the auth flow, the notification system, or the database schema touches `User`.

Symptoms:

- The file exceeds several hundred lines.
- Tests for one feature require mocking dependencies for unrelated features.
- Multiple teams open conflicting pull requests on the same class.
- The class imports from many unrelated packages.

## Splitting a God Class

The fix is to identify actors and group methods that share an actor.

```java
// After SRP refactor
public final class Employee {
    private final EmployeeId id;
    private final String name;
    private final Money baseSalary;
    // value object — no behavior tied to specific actors
}

public class PayCalculator {                  // CFO actor
    public Money calculatePay(Employee e, TimeCard tc) { /* ... */ }
}

public class HourReporter {                   // COO actor
    public String reportHours(Employee e, TimeCard tc) { /* ... */ }
}

public class EmployeeRepository {             // DBA actor
    public void save(Employee e) { /* ... */ }
    public Optional<Employee> findById(EmployeeId id) { /* ... */ }
}
```

Shared logic that genuinely belongs to all three (e.g. how regular hours are computed) becomes its own collaborator with a clear owner — typically the actor who actually defines that policy.

```typescript
// TypeScript equivalent
interface Employee {
    readonly id: string;
    readonly name: string;
    readonly baseSalary: number;
}

class PayCalculator {                         // CFO actor
    calculatePay(employee: Employee, timeCard: TimeCard): number { /* ... */ }
}

class HourReporter {                          // COO actor
    reportHours(employee: Employee, timeCard: TimeCard): string { /* ... */ }
}

class EmployeeRepository {                    // DBA actor
    async save(employee: Employee): Promise<void> { /* ... */ }
    async findById(id: string): Promise<Employee | null> { /* ... */ }
}
```

A façade can re-expose the original API surface if external callers depended on it:

```java
public class EmployeeFacade {
    private final PayCalculator pay;
    private final HourReporter reporter;
    private final EmployeeRepository repo;

    public EmployeeFacade(PayCalculator pay, HourReporter reporter, EmployeeRepository repo) {
        this.pay = pay;
        this.reporter = reporter;
        this.repo = repo;
    }

    public Money calculatePay(Employee e, TimeCard tc) { return pay.calculatePay(e, tc); }
    public String reportHours(Employee e, TimeCard tc) { return reporter.reportHours(e, tc); }
    public void save(Employee e) { repo.save(e); }
}
```

## SRP at the Module Level

SRP applies recursively — to functions, classes, modules, and services.

- **Function level:** a function should do one thing at one level of abstraction. `processOrder()` that validates, charges the card, writes the database, and emails the customer is a function-scale SRP violation.
- **Class level:** the actor-based reading above.
- **Module / package level:** packages should group classes that change together. A `billing` package whose contents change for billing reasons; an `inventory` package whose contents change for inventory reasons. This is the **Common Closure Principle** — a sibling rule from Martin's package-level principles.
- **Service level:** a microservice should own one bounded context. A service that owns both customer profiles and shipping logistics will fight itself across deployments.

```typescript
// Function-level SRP
// Bad: one function, multiple concerns
async function processOrder(order: Order): Promise<void> {
    if (!order.items.length) throw new Error("empty order");
    const total = order.items.reduce((s, i) => s + i.price * i.qty, 0);
    await chargeCard(order.cardToken, total);
    await db.orders.insert(order);
    await emailService.send(order.email, "Thanks for your order");
}

// Better: each step is its own function and can change independently
async function processOrder(order: Order): Promise<void> {
    validateOrder(order);
    const total = totalFor(order);
    await charge(order, total);
    await persist(order);
    await notify(order);
}
```

## Common Misreadings

**"One method per class."** No. A class can have many methods if they all serve the same actor and change together. SRP is about *cohesion of reasons*, not headcount.

**"Split by noun."** Splitting `User` into `UserName`, `UserEmail`, `UserPassword` is decomposition theater. The right question is *which actor owns each behavior*, not which noun it touches.

**"SRP means small files."** Small files are a byproduct, not the goal. A 600-line class that genuinely serves one actor is fine. A 50-line class that serves three actors is broken.

**"SRP forbids helper methods."** Private helpers are how a single-responsibility class stays readable. The unit of responsibility is the public contract, not every internal method.

A useful test: imagine the next change request. If you can confidently name *one* stakeholder it would come from, the class probably has a single responsibility. If you find yourself saying "depends on whether finance or ops asked," you've got two responsibilities sharing a class.

A second test, sometimes called the "elevator pitch," asks you to describe what the class does in a single sentence without using the word "and." If the description naturally contains "and" — "this class validates orders *and* sends emails *and* writes audit logs" — you have multiple responsibilities. The conjunction is a tell.

## Related

- [Open/Closed Principle (OCP)](./open-closed-principle.md)
- [Liskov Substitution Principle (LSP)](./liskov-substitution-principle.md)
- [Interface Segregation Principle (ISP)](./interface-segregation-principle.md)
- [Dependency Inversion Principle (DIP)](./dependency-inversion-principle.md)
- [../oop-fundamentals/encapsulation.md](../oop-fundamentals/encapsulation.md)
- [../design-principles/coupling-and-cohesion.md](../design-principles/coupling-and-cohesion.md)
- [../design-principles/separation-of-concerns.md](../design-principles/separation-of-concerns.md)
