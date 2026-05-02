---
title: "Code Smells and Refactoring Triggers"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, code-smells, refactoring, fowler]
---

# Code Smells and Refactoring Triggers

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `code-smells` `refactoring` `fowler`

## Summary

Code smells, named by Kent Beck and catalogued by Martin Fowler in *Refactoring*, are surface-level indicators of deeper design problems. They are not bugs and not always wrong — but each one is a question worth asking. This document covers the core Fowler catalog (long method, large class, feature envy, shotgun surgery, primitive obsession, data clumps, divergent change, parallel inheritance, refused bequest) as a design checklist for spotting where refactoring will pay off.

## Table of Contents

- [What a Code Smell Is](#what-a-code-smell-is)
- [Long Method](#long-method)
- [Large Class](#large-class)
- [Feature Envy](#feature-envy)
- [Shotgun Surgery](#shotgun-surgery)
- [Primitive Obsession](#primitive-obsession)
- [Data Clumps](#data-clumps)
- [Divergent Change](#divergent-change)
- [Parallel Inheritance Hierarchies](#parallel-inheritance-hierarchies)
- [Refused Bequest](#refused-bequest)
- [Using the Catalog as a Checklist](#using-the-catalog-as-a-checklist)
- [Related](#related)

## What a Code Smell Is

A code smell is a *symptom* — something the code tells you that hints at a structural problem. The smell is not the disease; the smell is the signal. Treating only the smell ("this method is too long, let me extract some private helpers") without diagnosing the underlying problem usually just hides the issue.

Each smell maps to one or more refactorings in Fowler's catalog. The workflow:

1. **Notice** the smell.
2. **Diagnose** the design problem behind it.
3. **Apply** the refactoring that fixes the underlying problem.
4. **Verify** with tests that behavior is unchanged.

## Long Method

A method that is too long to read at a glance — typically more than 20–30 lines. Long methods are usually doing several things, mixing levels of abstraction.

```java
// Long method — multiple responsibilities tangled together
public Invoice generateInvoice(Order order) {
    // 10 lines of validation
    if (order == null) throw new IllegalArgumentException();
    if (order.getItems().isEmpty()) throw new IllegalStateException("empty");
    // ... more checks

    // 15 lines of total / tax / discount calculation
    BigDecimal subtotal = BigDecimal.ZERO;
    for (var item : order.getItems()) { /* ... */ }
    BigDecimal tax = subtotal.multiply(new BigDecimal("0.1"));
    // ... discount calculation

    // 20 lines of building the invoice DOM / PDF / email body
    StringBuilder html = new StringBuilder();
    // ...

    // 10 lines of persistence
    invoiceRepo.save(invoice);
    // ...

    return invoice;
}
```

**Refactoring:** *Extract Method*. Pull each chunk into a private method whose name describes intent (`validate(order)`, `calculateTotals(order)`, `renderHtml(invoice)`, `persist(invoice)`). The top-level method becomes a high-level outline that reads like prose.

The deeper diagnosis is usually that the method is *the wrong abstraction altogether* — pulling out helpers may not be enough; the right answer might be splitting across multiple classes (each chunk becoming a method on a focused collaborator).

## Large Class

A class with too many fields, methods, or responsibilities — typically a sign of low cohesion. The class is doing several things that should be separate.

```typescript
// Large class — God-class tendencies
class UserManager {
  // 30+ fields
  // methods for authentication
  login() {} logout() {} resetPassword() {}
  // methods for profile management
  updateProfile() {} uploadAvatar() {}
  // methods for preferences
  setTheme() {} setLanguage() {}
  // methods for billing
  addPaymentMethod() {} chargeSubscription() {}
  // methods for notifications
  enableEmail() {} sendNotification() {}
  // methods for admin
  banUser() {} grantRole() {}
}
```

**Refactoring:** *Extract Class*. Pull cohesive groups into focused classes (`AuthService`, `ProfileService`, `BillingService`, `NotificationService`, `AdminService`). Often a class signals the absence of a domain concept that, once named, organizes its own fields and methods.

## Feature Envy

A method that seems more interested in a class *other than* the one it lives in — it pulls fields from another object and operates on them.

```java
// PriceCalculator is enviously inspecting Order's fields
public class PriceCalculator {
    public BigDecimal calculate(Order order) {
        BigDecimal subtotal = BigDecimal.ZERO;
        for (var item : order.getItems()) {
            subtotal = subtotal.add(item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())));
        }
        if (order.getCustomer().isVip()) {
            subtotal = subtotal.multiply(new BigDecimal("0.9"));
        }
        return subtotal;
    }
}
```

**Refactoring:** *Move Method*. The work belongs on `Order` (or on `Customer`/`Item`). The smell is that data and behavior have been separated.

```java
public class Order {
    public BigDecimal calculateTotal() {
        BigDecimal subtotal = items.stream()
            .map(Item::getLineTotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        return customer.applyDiscount(subtotal);
    }
}
```

The method now operates on `this` and asks collaborators to do their part — much closer to "Tell, Don't Ask" and the Law of Demeter.

## Shotgun Surgery

The opposite of *Divergent Change*: every time you make a single conceptual change, you have to edit many classes. Adding a new field to a `User` requires editing five DTOs, three validators, four mappers, and two tests.

The underlying issue is *scattering* — one concept is spread across many places, instead of being concentrated.

**Refactoring:** *Move Method* / *Move Field* to bring the scattered pieces together. Or introduce a class that *is* the missing concept (e.g., a `UserCreationCommand` that owns its validation, mapping, and DTO conversion).

```typescript
// Symptom: adding a "phone" field meant changes here, here, here, here, and here
//   - User entity
//   - UserCreateDto
//   - UserUpdateDto
//   - UserResponseDto
//   - UserMapper.toEntity
//   - UserMapper.toResponse
//   - UserValidator
//   - tests for all of the above
```

If a single intent requires a wide diff, the boundaries are wrong.

## Primitive Obsession

Using bare primitives (string, number, boolean) for things that have a richer concept — money, currency, IDs, email addresses, dates with time zones, durations.

```java
// Primitive obsession
public void transfer(String fromAccountId, String toAccountId, double amount, String currency) { /* ... */ }
```

Several problems:

- `amount` is a `double` — floating-point error in money calculations.
- `currency` is a string — typos compile.
- `fromAccountId` and `toAccountId` are both `String` — easy to swap.
- The signature does not encode that "amount + currency" is one concept.

**Refactoring:** *Replace Data Value with Object* or *Replace Primitive with Object*. Introduce value objects:

```java
public record AccountId(String value) { }
public record Money(BigDecimal amount, Currency currency) {
    public Money add(Money other) { /* ... */ }
}

public void transfer(AccountId from, AccountId to, Money amount) { /* ... */ }
```

Now `transfer(amount, fromId)` won't compile if you put arguments in the wrong order, money has correct arithmetic, and currency mismatches can be checked.

## Data Clumps

The same group of fields appears together in multiple places — function signatures, class fields, DTO bodies. The clump is a missing abstraction.

```typescript
// `street`, `city`, `postalCode`, `country` always travel together
function shipTo(orderId: string, street: string, city: string, postalCode: string, country: string) { }
function billTo(orderId: string, street: string, city: string, postalCode: string, country: string) { }
class Customer {
  street!: string; city!: string; postalCode!: string; country!: string;
}
```

**Refactoring:** *Extract Class* or introduce a value type. The clump is `Address`:

```typescript
class Address {
  constructor(readonly street: string, readonly city: string, readonly postalCode: string, readonly country: string) {}
}

function shipTo(orderId: string, address: Address) { }
function billTo(orderId: string, address: Address) { }
class Customer { address!: Address; }
```

When data clumps cluster, you've found a concept that wasn't named.

## Divergent Change

A class is modified for many *different* reasons — every kind of change ends up in the same file. Symptoms: a class file with weekly edits from three different teams, each editing for a different concern.

This violates SRP and SoC.

**Refactoring:** *Extract Class* along the change-axis. If the same `OrderService` is edited for pricing, fulfillment, and notification reasons, split into `PricingService`, `FulfillmentService`, `NotificationService` so each has one reason to change.

Compare with *Shotgun Surgery* (one change touches many classes); both are problems, but they're inverses:

- **Divergent Change:** one class, many reasons → split the class.
- **Shotgun Surgery:** one reason, many classes → consolidate the concept.

## Parallel Inheritance Hierarchies

Every time you create a new subclass of A, you also have to create a corresponding subclass of B. Two hierarchies that mirror each other.

```java
// Hierarchy 1: shapes
class Shape { }
class Circle extends Shape { }
class Square extends Shape { }
class Triangle extends Shape { }

// Hierarchy 2: shape renderers — one per shape
class ShapeRenderer { }
class CircleRenderer extends ShapeRenderer { }
class SquareRenderer extends ShapeRenderer { }
class TriangleRenderer extends ShapeRenderer { }

// Add a new shape `Hexagon` → must also add `HexagonRenderer`. Forever.
```

**Refactoring:** Use the *Visitor* pattern, or move the rendering logic into the shapes (composition / strategy), or have a single renderer that dispatches via type or pattern matching. The duplication of structure is the smell — the two hierarchies should be one, or one of them should disappear.

## Refused Bequest

A subclass inherits methods or fields from its superclass that it doesn't actually want — it overrides them to throw exceptions, no-ops, or returns defaults.

```java
public class Bird {
    public void fly() { /* ... */ }
    public void layEggs() { /* ... */ }
}

public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("penguins don't fly");
    }
    public void layEggs() { /* ... */ }
}
```

`Penguin` is refusing the `fly` bequest — a sign that the inheritance relationship is wrong, or that the superclass conflates two concerns.

**Refactoring:** *Replace Inheritance with Delegation*, or restructure the hierarchy. Often, splitting `Bird` into `Bird` and `FlyingBird` (or pulling `fly()` into a `Flyer` interface that only flying birds implement) restores Liskov substitutability.

This smell is also a violation of LSP and the Interface Segregation Principle.

## Using the Catalog as a Checklist

When reviewing code or working in unfamiliar territory, walk the catalog:

- Is any method too long to read in one screen? (Long Method)
- Does any class do too many things? (Large Class)
- Is any method more interested in another class's data? (Feature Envy)
- Does adding one feature require editing many classes? (Shotgun Surgery)
- Are primitives standing in for richer concepts? (Primitive Obsession)
- Do the same fields travel together everywhere? (Data Clumps)
- Is one class edited for unrelated reasons? (Divergent Change)
- Do two hierarchies grow in lockstep? (Parallel Inheritance)
- Does any subclass refuse what it inherited? (Refused Bequest)

A "yes" on any of these is not automatic justification for a refactor — but it is a *prompt to look closer*. The corresponding refactoring in Fowler's catalog usually has the right shape.

Other smells worth knowing (briefly): *Switch Statements* (often missing polymorphism), *Lazy Class* (does nothing useful), *Speculative Generality* (YAGNI violation), *Comments* (often a sign that the code itself isn't clear), *Dead Code*, *Long Parameter List*, *Message Chains* (Law of Demeter violation), *Middle Man* (excessive delegation).

## Related

- [DRY — Don't Repeat Yourself](dry-principle.md)
- [KISS — Keep It Simple, Stupid](kiss-principle.md)
- [YAGNI — You Aren't Gonna Need It](yagni-principle.md)
- [Law of Demeter](law-of-demeter.md)
- [Separation of Concerns](separation-of-concerns.md)
- [Coupling and Cohesion](coupling-and-cohesion.md)
- [Composing Objects Principle](composing-objects-principle.md)
- [Anti-Patterns in OO Design](anti-patterns-in-oo-design.md)
- [Single Responsibility Principle](../solid/single-responsibility-principle.md)
- [Liskov Substitution Principle](../solid/liskov-substitution-principle.md)
- [Interface Segregation Principle](../solid/interface-segregation-principle.md)
- [Encapsulation](../oop-fundamentals/encapsulation.md)
- [Inheritance](../oop-fundamentals/inheritance.md)

## References

- Fowler, M. — *Refactoring: Improving the Design of Existing Code*
- Beck, K. — coined the term "code smell"
- Martin, R. — *Clean Code*
- Kerievsky, J. — *Refactoring to Patterns*
