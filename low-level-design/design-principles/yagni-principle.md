---
title: "YAGNI — You Aren't Gonna Need It"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, yagni, simplicity, speculative-generality]
---

# YAGNI — You Aren't Gonna Need It

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `yagni` `simplicity` `speculative-generality`

## Summary

YAGNI, popularized by Ron Jeffries and Kent Beck in Extreme Programming, says do not implement features, abstractions, or hooks until you actually need them. The principle targets *speculative generality* — code written for futures that often never arrive — but it is not absolute, and ignoring it can be correct for irreversible decisions and public APIs.

## Table of Contents

- [What YAGNI Is and Isn't](#what-yagni-is-and-isnt)
- [Speculative Generality](#speculative-generality)
- [Hooks for Nothing](#hooks-for-nothing)
- [The Cost of Premature Features](#the-cost-of-premature-features)
- [When YAGNI Is Wrong](#when-yagni-is-wrong)
- [The Reversibility Test](#the-reversibility-test)
- [Practical Examples](#practical-examples)
- [Checklist](#checklist)
- [Related](#related)

## What YAGNI Is and Isn't

YAGNI says: *do not build what you do not currently need*. It does not say "ship low-quality code" or "ignore the future." It says the *best preparation for an unknown future is a small, well-factored present*. A clean, simple system is easier to extend later than a sprawling speculative one.

Martin Fowler's framing in his "yagni" article: presumptive features have four costs even before they are wrong:

1. **Cost of build** — time you spent on this rather than what users need.
2. **Cost of delay** — features users wanted that shipped later because of this.
3. **Cost of carry** — every bug fix, refactor, and review now drags this code along.
4. **Cost of repair** — when you finally do need the feature, your guess about what it should do was almost certainly wrong, so now you fix two designs.

## Speculative Generality

Fowler's *Refactoring* names this as a code smell: *abstractions added for future flexibility that no current code uses*. Symptoms:

- Interfaces with a single implementation, and no plan to add another
- Abstract base classes whose only override is the one concrete subclass
- Configuration knobs that nobody flips
- Strategy pattern wired up where there is one strategy
- Generic types where a concrete type would do

```java
// Speculative — built "in case we add more notification channels"
public interface NotificationChannel {
    void send(String to, String subject, String body);
}

public class EmailChannel implements NotificationChannel { /* the only impl */ }

public class NotificationService {
    private final NotificationChannel channel;
    public NotificationService(NotificationChannel channel) { this.channel = channel; }
    public void notifyUser(User user, String message) {
        channel.send(user.getEmail(), "Notification", message);
    }
}

// Concrete — what you actually need today
public class EmailNotifier {
    private final SmtpClient smtp;
    public EmailNotifier(SmtpClient smtp) { this.smtp = smtp; }
    public void notifyUser(User user, String message) {
        smtp.send(user.getEmail(), "Notification", message);
    }
}
```

When SMS is added later, you can introduce the interface *then* — the refactor is mechanical and informed by two real implementations rather than one and a guess.

## Hooks for Nothing

A *hook* is an extension point: a callback, a plugin slot, a strategy parameter, a `protected` method intended for override. Hooks have non-trivial cost:

- Every hook is part of the API surface and constrains your future refactors.
- Hooks invert control — you no longer fully understand what runs without checking what is plugged in.
- Hooks must be tested, documented, and maintained even when nothing uses them.

```typescript
// Hook for nothing — onBeforeSave, onAfterSave, validators[], formatters[]...
class UserService {
  private validators: Array<(u: User) => void> = [];
  private formatters: Array<(u: User) => User> = [];
  private beforeSave?: (u: User) => void;
  private afterSave?: (u: User) => void;

  registerValidator(v: (u: User) => void) { this.validators.push(v); }
  registerFormatter(f: (u: User) => User) { this.formatters.push(f); }

  save(user: User) {
    this.beforeSave?.(user);
    let processed = user;
    for (const f of this.formatters) processed = f(processed);
    for (const v of this.validators) v(processed);
    this.repo.save(processed);
    this.afterSave?.(processed);
  }
}

// What you actually use
class UserService {
  save(user: User) {
    validateUser(user);
    this.repo.save(user);
  }
}
```

If you ever truly need an after-save hook, you will know exactly what shape it should be. Today's hook is a guess.

## The Cost of Premature Features

A feature you don't need *also costs you the alternatives you didn't build*. The opportunity cost is real but invisible. Engineers tend to optimize against direct cost (lines of code) and ignore opportunity cost.

Premature features also tend to lock in wrong assumptions:

- A "user roles" system designed before any role conflict has appeared usually models roles wrong.
- A pluggable storage layer designed before any second storage backend existed usually has the wrong abstraction boundary.
- A multi-tenant design built before the second tenant usually entangles tenancy with the wrong layer.

Real second use cases reveal the seams. Imagined ones invent seams that don't fit.

## When YAGNI Is Wrong

YAGNI is a default, not an absolute. It is *wrong* when:

### Irreversible decisions

Some choices are difficult or impossible to change later. Examples:

- **Database schema for production data** — adding a column is cheap; backfilling a deeply embedded denormalization is not.
- **URL structure of a public website** — once external sites link to you, the URLs are forever.
- **Wire protocol versioning** — once clients ship, you cannot remove fields without coordination.
- **Data privacy / encryption boundaries** — adding encryption to historical data is painful; designing it in is cheap.

For these, *invest now even if you don't strictly need it today*. The cost-of-repair is asymmetric.

### Public APIs

Once a method, route, or library is published to consumers you do not control, removing or renaming it is a breaking change. For public APIs:

- Be conservative about what you expose
- Add versioning and deprecation paths upfront
- Mark unstable parts clearly (`@Beta`, `experimental`)
- Consider future evolution before locking the contract

### Security and reliability foundations

Authentication, authorization, audit logging, rate limiting, idempotency, and observability are easier to design in than retrofit. They are *infrastructure*, not features. Don't YAGNI them.

### Data model decisions with downstream cost

If you skip a `created_at` timestamp now, you will regret it the first time you debug a production issue. If you skip soft-delete and someone deletes a row, the data is gone. These are cheap up front and expensive to add later.

## The Reversibility Test

Ask one question for each potential YAGNI violation:

> **If I'm wrong, how expensive is it to add this later?**

| Cost to add later | YAGNI says |
|-------------------|-----------|
| A few lines of refactor | Don't add it now |
| A multi-day refactor | Maybe add it now |
| A migration on production data | Probably add it now |
| Impossible without breaking customers | Add it now |

This converts a binary debate into a calibrated risk decision.

## Practical Examples

### Skip — and add later

```typescript
// You don't need a strategy interface for one strategy
function calculatePrice(items: CartItem[]): number {
  return items.reduce((sum, i) => sum + i.price * i.qty, 0);
}

// Later, when a second pricing strategy actually exists, refactor:
type PricingStrategy = (items: CartItem[]) => number;
const standardPricing: PricingStrategy = items => /* ... */;
const memberPricing: PricingStrategy = items => /* ... */;
```

### Don't skip — design in

```java
// Idempotency keys on financial APIs — bake in from day one
@PostMapping("/payments")
public Payment createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest request) {
    return paymentService.createIdempotent(idempotencyKey, request);
}
```

Adding idempotency to a payment API after duplicates start corrupting books is far more expensive than designing it in.

### The middle case

```typescript
// Adding a `createdAt` column on a new table — yes, even if no UI uses it
interface OrderRow {
  id: string;
  customerId: string;
  total: number;
  createdAt: Date;     // cheap now, very useful later
  updatedAt: Date;
}
```

This is technically YAGNI-violating, but the field is so cheap and so often needed that it's a sensible default.

## Checklist

- [ ] Are you adding this for a real, current need or for a hypothetical future?
- [ ] If you skip it, how expensive is the future fix?
- [ ] Is this an interface or hook with only one implementation today?
- [ ] Is this a public API, irreversible decision, or security foundation?
- [ ] Could the present design be evolved cleanly when the need arrives?
- [ ] Have you written down the assumption you're betting against?

## Related

- [KISS — Keep It Simple, Stupid](kiss-principle.md)
- [DRY — Don't Repeat Yourself](dry-principle.md)
- [Code Smells and Refactoring Triggers](code-smells-and-refactoring-triggers.md)
- [Anti-Patterns in OO Design](anti-patterns-in-oo-design.md)
- [Open/Closed Principle](../solid/open-closed-principle.md)
- [Composing Objects Principle](composing-objects-principle.md)

## References

- Fowler, M. — "Yagni" essay, *Refactoring*
- Beck, K. — *Extreme Programming Explained*
- Jeffries, R. — early XP writings on simplicity
- Ousterhout, J. — *A Philosophy of Software Design*
