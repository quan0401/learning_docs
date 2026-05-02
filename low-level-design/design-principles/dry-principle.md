---
title: "DRY — Don't Repeat Yourself"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, dry, refactoring, abstraction]
---

# DRY — Don't Repeat Yourself

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `dry` `refactoring` `abstraction`

## Summary

DRY, coined by Andy Hunt and Dave Thomas in *The Pragmatic Programmer*, says every piece of knowledge should have a single, unambiguous, authoritative representation in a system. It is a principle about *knowledge* duplication, not surface-level code similarity — and applying it blindly creates premature abstractions that are harder to fix than the duplication they replaced.

## Table of Contents

- [What DRY Actually Means](#what-dry-actually-means)
- [Knowledge Duplication vs Code Similarity](#knowledge-duplication-vs-code-similarity)
- [The Rule of Three](#the-rule-of-three)
- [When DRY Makes Things Worse](#when-dry-makes-things-worse)
- [Recognizing Real Duplication](#recognizing-real-duplication)
- [Practical Examples](#practical-examples)
- [Checklist](#checklist)
- [Related](#related)

## What DRY Actually Means

The original wording from *The Pragmatic Programmer*:

> Every piece of knowledge must have a single, unambiguous, authoritative representation within a system.

The keyword is **knowledge**. DRY targets duplication of business rules, validation logic, schema definitions, and domain concepts — not duplication of token sequences. Two functions that happen to look alike but encode different rules are *not* a DRY violation. Two functions that encode the same rule in two places *are*, even if the code looks different.

## Knowledge Duplication vs Code Similarity

A common mistake is to deduplicate based on visual pattern matching:

```typescript
// Looks like duplication...
function calculateInvoiceTax(amount: number): number {
  return amount * 0.1;
}

function calculateShippingFee(amount: number): number {
  return amount * 0.1;
}
```

These functions share *syntax*, not *knowledge*. Tax law and shipping policy change independently. Merging them into a single `multiplyByTenPercent(amount)` function couples two unrelated concepts and produces a worse design when one rate changes.

By contrast, this *is* a DRY violation:

```java
// Same business rule encoded twice
public class OrderService {
    public boolean isEligibleForDiscount(Order order) {
        return order.getTotal().compareTo(new BigDecimal("100")) >= 0
            && order.getCustomer().isReturning();
    }
}

public class CheckoutController {
    public String renderDiscountBanner(Order order) {
        if (order.getTotal().compareTo(new BigDecimal("100")) >= 0
                && order.getCustomer().isReturning()) {
            return "You qualify for a discount!";
        }
        return "";
    }
}
```

The eligibility rule lives in two places. Change one without the other and the system contradicts itself. Extract into a single method.

## The Rule of Three

Martin Fowler's heuristic in *Refactoring*:

> The first time you do something, you just do it. The second time you do something similar, you wince at the duplication, but you do the duplicative thing anyway. The third time you do something similar, you refactor.

Why three? Two data points are not enough to see the shape of the abstraction. Pulling out a "common" function from two call sites usually fits both — but that abstraction is shaped by an accident of those two cases, not by the underlying concept. The third call site reveals what is actually general and what was situational.

```typescript
// First occurrence — write it inline
function sendWelcomeEmail(user: User) {
  const subject = `Welcome, ${user.name}!`;
  smtp.send(user.email, subject, renderTemplate("welcome", user));
}

// Second — wince, leave it
function sendPasswordReset(user: User, token: string) {
  const subject = "Reset your password";
  smtp.send(user.email, subject, renderTemplate("reset", { user, token }));
}

// Third — now extract
function sendTransactional(
  user: User,
  template: string,
  subject: string,
  context: Record<string, unknown>
) {
  smtp.send(user.email, subject, renderTemplate(template, context));
}
```

By the third use the parameter list and abstraction are grounded in three real cases instead of speculation.

## When DRY Makes Things Worse

### Premature abstraction

The cost of wrong abstraction is high: every consumer is now coupled to a shape that does not fit. Fixing it requires either inlining (un-DRYing) or layering more parameters and conditionals on top.

Sandi Metz: *"Duplication is far cheaper than the wrong abstraction."*

### Coincidental coupling

When two pieces of code happen to be similar today but evolve in different directions, forcing them through one abstraction creates change-amplification: a fix needed for one call site requires verifying every other.

```java
// Two endpoints look the same today
public void createUser(UserDto dto) {
    validate(dto);
    repository.save(toEntity(dto));
    eventBus.publish(new UserCreated(dto.getId()));
}

public void createOrder(OrderDto dto) {
    validate(dto);
    repository.save(toEntity(dto));
    eventBus.publish(new OrderCreated(dto.getId()));
}
```

A `genericCreate(dto, eventFactory)` helper would compile, but user creation and order creation will diverge — order creation will need inventory checks, payment authorization, fraud scoring, idempotency keys. The "shared" helper becomes a flag-laden monster.

### DRY across boundaries

Sharing types, validation, or logic between services that own different lifecycles is a frequent source of pain. Microservices intentionally duplicate DTOs across the boundary so each service can evolve independently. The cost of duplication here is less than the cost of cross-service coupling.

### Tests

Test code is *intentionally* repetitive. A test should read top-to-bottom without the reader chasing helpers. Light duplication in setup and assertions is usually clearer than over-engineered fixtures.

## Recognizing Real Duplication

Ask:

1. **Does this represent the same business rule?** If yes, deduplicate.
2. **Will these change together for the same reason?** If yes, deduplicate. (This is essentially the Single Responsibility Principle.)
3. **If one needs to change, will I forget to change the other?** If yes, deduplicate — the duplication is a bug waiting to happen.
4. **Are these similar by coincidence?** If yes, leave them alone.

The phrase *"these change for the same reasons"* is the strongest test. It is the inverse of SRP: SRP says one module, one reason to change. DRY says one reason to change, one module.

## Practical Examples

### Configuration as the single source

```typescript
// Bad — magic numbers scattered
function isEligibleForFreeShipping(orderTotal: number) {
  return orderTotal >= 50;
}
function showShippingBanner(orderTotal: number) {
  return orderTotal < 50 ? `Add $${50 - orderTotal} for free shipping` : "";
}

// Good — one source of truth
const FREE_SHIPPING_THRESHOLD = 50;

function isEligibleForFreeShipping(orderTotal: number) {
  return orderTotal >= FREE_SHIPPING_THRESHOLD;
}
function showShippingBanner(orderTotal: number) {
  const remaining = FREE_SHIPPING_THRESHOLD - orderTotal;
  return remaining > 0 ? `Add $${remaining} for free shipping` : "";
}
```

The threshold is a single piece of knowledge. It belongs in one place.

### Schema duplication

A common Java pattern: the same field validation lives on the JPA entity, the DTO, the form, and the API contract.

```java
// Centralize via Bean Validation annotations on a shared interface or DTO
public interface EmailConstraint {
    @Email
    @NotBlank
    @Size(max = 255)
    String getEmail();
}

public class UserCreateDto implements EmailConstraint { /* ... */ }
public class UserUpdateDto implements EmailConstraint { /* ... */ }
```

The validation rules — the *knowledge* — live once.

### Database migrations and code

If a column is `VARCHAR(255)` in the database but `String` (unbounded) in the application, the constraint is duplicated implicitly: any change to the column length must be reflected in validation. Driving validation from the schema (or generating one from the other) collapses the duplication.

## Checklist

- [ ] Does duplicated code encode the same *rule*, or just look similar?
- [ ] Have you seen the third occurrence before extracting?
- [ ] Will both copies change together for the same reason?
- [ ] Does extraction cross a boundary (service, team, deployment) where duplication is cheaper than coupling?
- [ ] Are you deduplicating tests where readability matters more?
- [ ] Have you named the abstraction after the *concept*, not the shape of the code?

## Related

- [KISS — Keep It Simple, Stupid](kiss-principle.md)
- [YAGNI — You Aren't Gonna Need It](yagni-principle.md)
- [Coupling and Cohesion](coupling-and-cohesion.md)
- [Code Smells and Refactoring Triggers](code-smells-and-refactoring-triggers.md)
- [Anti-Patterns in OO Design](anti-patterns-in-oo-design.md)
- [Single Responsibility Principle](../solid/single-responsibility-principle.md)
- [Encapsulation](../oop-fundamentals/encapsulation.md)

## References

- Hunt, A. & Thomas, D. — *The Pragmatic Programmer*
- Fowler, M. — *Refactoring: Improving the Design of Existing Code*
- Metz, S. — *Practical Object-Oriented Design in Ruby*
- Martin, R. — *Clean Code*
