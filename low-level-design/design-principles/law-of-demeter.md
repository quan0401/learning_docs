---
title: "Law of Demeter — Talk Only to Friends"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, law-of-demeter, coupling, encapsulation]
---

# Law of Demeter — Talk Only to Friends

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `law-of-demeter` `coupling` `encapsulation`

## Summary

The Law of Demeter (LoD), formulated at Northeastern University in the 1980s, says a method should only call methods on a small set of "friends" — itself, its parameters, its fields, and objects it locally creates — and not chase chains of references through the object graph. The visible symptom of violation is the *train wreck*: `a.getB().getC().getD().doSomething()`. Following LoD reduces coupling, but applied dogmatically it can produce paranoid wrapper classes and worse APIs.

## Table of Contents

- [The Rule](#the-rule)
- [Train Wreck Calls](#train-wreck-calls)
- [What Counts as a Violation](#what-counts-as-a-violation)
- [Why It Matters](#why-it-matters)
- [Refactoring Out a Violation](#refactoring-out-a-violation)
- [When LoD Makes the API Worse](#when-lod-makes-the-api-worse)
- [Data Structures Are Different](#data-structures-are-different)
- [Checklist](#checklist)
- [Related](#related)

## The Rule

A method `m` of an object `O` may only invoke methods of:

1. `O` itself (i.e., `this`)
2. Parameters of `m`
3. Objects created locally inside `m`
4. Direct component objects of `O` (its fields)
5. Global objects accessible to `O` (typically singletons or DI-injected services)

Crucially, `m` may **not** invoke methods on objects *returned* by methods on those friends — that is reaching through one friend to talk to a stranger.

A folksy phrasing: "talk only to your immediate friends, not to strangers your friends know." Or: "don't talk to strangers."

## Train Wreck Calls

The visual sign of LoD violation is a long chain of method calls:

```java
// Train wreck
String city = order.getCustomer().getAddress().getCity().getName();

// Or with side effects
order.getCustomer().getWallet().getPrimaryCard().charge(amount);
```

Each `.` traverses the object graph. The caller is now coupled to:

- `Order`, its `Customer` accessor, its return type
- `Customer`, its `Address` accessor, its return type
- `Address`, its `City` accessor
- `City`, its `Name` accessor

A change anywhere in that chain — `Customer` becoming `Optional<Customer>`, `Address` being split into `BillingAddress` and `ShippingAddress`, `City` becoming a value object — breaks the caller.

```typescript
// TypeScript train wreck
const total = invoice.lineItems[0].product.pricing.tiers[2].amount;
```

## What Counts as a Violation

Not every chained call is a violation. The distinction is **whether you're talking to your data, or your friends' friends**.

### Violation: chaining methods on returned objects

```java
// Each .get returns an object you then send a message to
order.getCustomer().getAddress().getCity().getName();
```

You're invoking `getAddress()` on a *stranger* (`Customer`, returned by `order.getCustomer()`), then `getCity()` on another stranger, etc.

### Not a violation: fluent / builder pattern

```java
// Each method returns the same builder; you're talking to one friend repeatedly
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://example.com"))
    .header("Content-Type", "application/json")
    .POST(BodyPublishers.ofString(payload))
    .build();
```

The builder returns `this` (or a related builder). You are not navigating an object graph — you are configuring one object.

### Not a violation: stream pipelines on collections

```typescript
const totals = orders
  .filter(o => o.isPaid)
  .map(o => o.total)
  .reduce((a, b) => a + b, 0);
```

You're operating on a single data collection, not chasing references through unrelated domain objects.

### Violation: reaching through structure

```typescript
// Bad — coupled to invoice's internal layout
function chargeFirstItem(invoice: Invoice, amount: number) {
  invoice.lineItems[0].product.pricing.charge(amount);
}
```

The function is coupled to `Invoice` having `lineItems`, items having `product`, products having `pricing`. Move any of those and the function breaks.

## Why It Matters

LoD reduces *transitive coupling*. With LoD-compliant code:

- A class only knows about its direct neighbors.
- Refactoring a distant class doesn't ripple through unrelated code.
- Mocking for tests only requires faking direct collaborators, not whole object graphs.

Without it:

- Changing `Address` to require a `Country` parameter forces you to find every `getCustomer().getAddress()` chain.
- Tests must construct three or four levels of fake objects to call one method.
- Encapsulation leaks: `Order` exposed `Customer`'s structure to its callers.

The rule of thumb: **a method should ask its collaborators to do work, not pull data out of them and do the work itself**. This is the classic "Tell, Don't Ask" guidance.

## Refactoring Out a Violation

```java
// Before — train wreck in the caller
public class ShippingCalculator {
    public BigDecimal calculate(Order order) {
        String country = order.getCustomer().getAddress().getCountry().getCode();
        return rates.lookup(country);
    }
}
```

The caller knows about `Customer`, `Address`, and `Country`. Push the knowledge down:

```java
public class Order {
    private final Customer customer;
    public String getShippingCountryCode() {
        return customer.getShippingCountryCode();
    }
}

public class Customer {
    private final Address address;
    public String getShippingCountryCode() {
        return address.getCountryCode();
    }
}

public class Address {
    private final Country country;
    public String getCountryCode() {
        return country.getCode();
    }
}

// Caller now talks only to its friend
public class ShippingCalculator {
    public BigDecimal calculate(Order order) {
        return rates.lookup(order.getShippingCountryCode());
    }
}
```

The chain is gone. Each class exposes only what callers need, and internal structure can change without rippling.

A better refactor in many cases is to have the order *do the calculation*:

```java
public class Order {
    private final Customer customer;
    public BigDecimal calculateShipping(RatesTable rates) {
        return customer.calculateShipping(rates);
    }
}
```

Now the rates lookup is encapsulated alongside the data it needs.

## When LoD Makes the API Worse

Strict LoD can produce delegation explosions:

```java
public class Order {
    public String getCustomerName() { return customer.getName(); }
    public String getCustomerEmail() { return customer.getEmail(); }
    public String getCustomerPhone() { return customer.getPhone(); }
    public LocalDate getCustomerBirthday() { return customer.getBirthday(); }
    public String getCustomerAddressStreet() { return customer.getAddress().getStreet(); }
    public String getCustomerAddressCity() { return customer.getAddress().getCity(); }
    public String getCustomerAddressCountry() { return customer.getAddress().getCountry(); }
    // ...
}
```

`Order` becomes a giant pass-through. The principle was meant to reduce coupling, but here every method on `Customer` is duplicated with `getCustomerXxx`, doubling the API surface and *increasing* coupling between `Order` and `Customer`.

When the right answer is *the caller really does need to know about `Customer`*, just expose `Customer` directly:

```java
order.getCustomer().getName();
```

This is fine if `Customer` is a stable, deliberately-public collaborator and the caller has a legitimate domain reason to know about it. The Law of Demeter is a *guideline*, not a rule that overrides good API design.

## Data Structures Are Different

A widely-cited distinction (Robert Martin, *Clean Code*): LoD applies to *objects with behavior*, not to *data transfer objects*. A DTO is intentionally a bag of data — chaining through its fields is exactly what it's for:

```typescript
// Fine — DTO/value object access
const street = response.data.user.address.street;
```

If `response` is a JSON DTO, you're not violating encapsulation by accessing nested fields — there is no encapsulation to violate. The danger appears when you accidentally treat a *behavior-bearing* domain object as a DTO.

Heuristic:

- If the methods are mostly `getX()` / `setX()` and there's no real behavior — it's a data structure, chaining is fine.
- If the methods do meaningful work — chaining through it leaks design decisions you'll regret.

## Checklist

- [ ] Does this expression chain method calls through more than one returned object?
- [ ] Is the caller coupled to the internal structure of objects it doesn't directly hold?
- [ ] Could you push the work into the object that owns the data ("Tell, Don't Ask")?
- [ ] Are you adding pass-through delegate methods for every nested field? Reconsider — exposing the collaborator might be cleaner.
- [ ] Is the chain on a builder, fluent API, stream, or DTO? Then it's probably fine.
- [ ] Will mocking this in a test require a four-level deep fake graph?

## Related

- [Coupling and Cohesion](coupling-and-cohesion.md)
- [Separation of Concerns](separation-of-concerns.md)
- [Composing Objects Principle](composing-objects-principle.md)
- [Code Smells and Refactoring Triggers](code-smells-and-refactoring-triggers.md)
- [Encapsulation](../oop-fundamentals/encapsulation.md)
- [Single Responsibility Principle](../solid/single-responsibility-principle.md)
- [Dependency Inversion Principle](../solid/dependency-inversion-principle.md)

## References

- Lieberherr, K. & Holland, I. — original LoD paper, "Assuring Good Style for Object-Oriented Programs"
- Martin, R. — *Clean Code*, chapter on objects vs data structures
- Hunt, A. & Thomas, D. — *The Pragmatic Programmer*
- Fowler, M. — *Refactoring*, "Message Chains" smell
