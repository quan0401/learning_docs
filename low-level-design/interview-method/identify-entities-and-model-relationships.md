---
title: "How to Identify Entities and Model Relationships"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, interview-method, ood, modeling, entities]
---

# How to Identify Entities and Model Relationships

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `interview-method` `ood` `modeling` `entities`

## Summary

Identifying entities is the bridge between fuzzy requirements and a class diagram. The standard technique is noun extraction: list every noun phrase in the agreed requirements, classify each as entity, value, role, or service, then test the model by walking it through sample flows. Multiplicity and lifecycle decisions are made on each pair, not in isolation.

## The Goal

Produce a small, defensible set of classes where:

- Each one represents a single concept
- Names match the language the domain expert would use
- Relationships have explicit multiplicity and direction
- The model can simulate every required use case

## Step 1 — Noun Extraction

Read the agreed requirements aloud. Underline every noun phrase. Do not filter yet.

**Example prompt:** "Design a library management system. Members can borrow books. A book has multiple copies. The librarian processes returns and applies late fees."

**Underlined nouns:** library, management system, member, book, copy, librarian, return, late fee.

This raw list will contain garbage. That is fine.

## Step 2 — Classify Each Candidate

For each noun, decide which bucket it belongs in.

### Entity

Has identity (an ID), a lifecycle, and may change state over time.

- `Member`, `Book`, `Copy`, `Loan`, `Reservation`

### Value Object

Equal by value, immutable, no identity.

- `ISBN`, `Money`, `LoanPeriod`, `Address`, `EmailAddress`

A `Money(10, USD)` is interchangeable with any other `Money(10, USD)`. A `Member` with the same name as another is still a different person.

### Role / Actor

A way an entity participates in a use case. Often modeled as a capability rather than a class hierarchy.

- `Borrower`, `Librarian`, `Admin`

If a single `User` plays multiple roles, prefer composition: `User` has `roles: Set<Role>`.

### Service

Coordinates entities to fulfill a use case. No persistent state of its own.

- `LoanService`, `LateFeeCalculator`, `ReservationService`

### Process / Event

A nounified verb. Often becomes a method or a domain event.

- `Return` -> a method `Loan.complete()` or an event `LoanReturned`
- `Borrowing` -> `LoanService.borrow(...)`

### Reject

- "Library management system" — this is the application, not a class
- "Late fee" — value object on a `Loan`, not a class of its own unless billing has its own lifecycle

## Step 3 — Test the Filter

Apply three sanity checks to each surviving candidate:

1. **Identity check**: Does it need an ID? If yes, entity.
2. **Equality check**: Are two with the same fields interchangeable? If yes, value object.
3. **Behavior check**: Does it have meaningful methods or just data? If only data, it might collapse into a value object.

A class that fails all three is probably a misclassified attribute.

## Step 4 — Identify Relationships

For every pair of entities that interact, decide:

- **Type** of relationship
- **Multiplicity** on each side
- **Directionality**
- **Lifecycle ownership**

### Relationship Types

| Type | Symbol | Lifecycle | Example |
|---|---|---|---|
| Association | plain line | Independent | `Member - Loan` |
| Aggregation | open diamond | Independent | `Library - Book` |
| Composition | filled diamond | Owned | `Book - Copy` |
| Inheritance | hollow arrow | Substitutable | `Vehicle <- Car` |
| Dependency | dashed arrow | Transient | `LoanService ..> Clock` |

See [../class-relationships/](../class-relationships/) for the deeper reference on each type.

### Multiplicity Decisions

Ask: "How many X can a Y have, at minimum and maximum?"

- A `Loan` belongs to exactly one `Member` -> `1..1`
- A `Member` can have many active `Loan`s -> `0..*`
- A `Book` has at least one `Copy` -> `1..*`
- A `Copy` is on at most one active `Loan` at a time -> `0..1`

The phrase "at a time" matters — temporal constraints often live in invariants, not multiplicities.

### Directionality

Default to unidirectional. Add the back-reference only if a use case requires it.

- `Loan -> Member` (every loan has a borrower) — needed for receipts
- `Member -> Loan` (loans by a member) — needed for "my loans" view

If both are needed, add both, but accept the bookkeeping burden.

### Lifecycle Ownership

If deleting parent should delete child, use composition.

- `ParkingLot` deletion deletes its `Floor`s — composition
- `Library` deletion does not delete `Book`s, just unowns them — aggregation
- `Member` deletion does not delete past `Loan`s (audit trail) — association

## Step 5 — Validate With Sample Flows

Walk every headline use case through the model. If you cannot perform a step, the model is wrong.

**Use case:** Member returns a book and gets charged a late fee.

```
loan = loanRepo.find(loanId)
copy = loan.copy()
returnDate = clock.now()
loan.complete(returnDate)        // sets state to RETURNED
copy.markAvailable()             // releases the copy
fee = lateFeeCalculator.compute(loan)
if (fee.greaterThan(Money.ZERO)) {
    member.account().charge(fee)
}
```

If `loan` does not know its `copy`, you have a missing edge. Add it.
If `member.account()` is not modeled but the flow needs charging, add `Account` as an entity.

This is where models get fixed — under flow pressure, not under abstract scrutiny.

## Worked Example — Library

**Entities:** Member, Book, Copy, Loan, Reservation, Account
**Value objects:** ISBN, Money, LoanPeriod
**Services:** LoanService, ReservationService, LateFeeCalculator

```java
public final class Loan {
    private final UUID id;
    private final Member member;
    private final Copy copy;
    private final Instant startedAt;
    private final LoanPeriod period;
    private LoanStatus status;
    private Instant returnedAt;

    public Loan(Member member, Copy copy, Instant startedAt, LoanPeriod period) {
        this.id = UUID.randomUUID();
        this.member = Objects.requireNonNull(member);
        this.copy = Objects.requireNonNull(copy);
        this.startedAt = Objects.requireNonNull(startedAt);
        this.period = Objects.requireNonNull(period);
        this.status = LoanStatus.ACTIVE;
    }

    public void complete(Instant returnedAt) {
        if (status != LoanStatus.ACTIVE) {
            throw new IllegalStateException("Loan already completed");
        }
        this.status = LoanStatus.RETURNED;
        this.returnedAt = returnedAt;
    }

    public boolean isOverdue(Instant now) {
        return now.isAfter(startedAt.plus(period.duration()));
    }

    public Member member() { return member; }
    public Copy copy() { return copy; }
    public Instant startedAt() { return startedAt; }
    public Optional<Instant> returnedAt() { return Optional.ofNullable(returnedAt); }
}
```

```typescript
export class Loan {
  readonly id: string;
  private status: LoanStatus = "ACTIVE";
  private returnedAt: Date | null = null;

  constructor(
    readonly member: Member,
    readonly copy: Copy,
    readonly startedAt: Date,
    readonly period: LoanPeriod,
  ) {
    this.id = crypto.randomUUID();
  }

  complete(returnedAt: Date): void {
    if (this.status !== "ACTIVE") {
      throw new Error("Loan already completed");
    }
    this.status = "RETURNED";
    this.returnedAt = returnedAt;
  }

  isOverdue(now: Date): boolean {
    const due = new Date(this.startedAt.getTime() + this.period.durationMs);
    return now > due;
  }
}
```

Notice `Loan` knows both `member` and `copy` directly — flows need both immediately, so we paid the bookkeeping cost.

## Common Modeling Mistakes

### Modeling Verbs as Classes

`BorrowAction`, `ReturnEvent` are usually methods on the relevant entity, not classes — unless your domain explicitly cares about an audit trail of actions.

### Confusing Roles With Classes

`Borrower`, `Librarian` are roles a `User` plays. Modeling them as separate hierarchies leads to duplication and misclassified state when one user does both.

### Missing the Aggregate Root

When several entities cluster (e.g., `Order`, `OrderLine`, `Payment`), one is the root through which others are accessed. Without a root, invariants drift across services.

### Multiplicities Without Time Awareness

"A copy can be on multiple loans" is true historically, false at any instant. Encode the temporal version in invariants, not in `*..*`.

### Symmetric Bidirectional Everywhere

Bidirectional links are expensive to maintain in code. Add the back-edge only when a use case requires it.

### "Manager" / "Handler" / "Helper" Classes

Names ending in `Manager` are usually a sign you skipped Step 2. Either it is a service with a domain-aligned name (`LoanService`) or it should be merged into the entity it manipulates.

## Heuristics That Speed You Up

- If two classes share most fields and differ only by a flag, they are probably one class with state
- If a class has no methods, it might be a value object — or you forgot the behavior
- If a service touches more than 4 entities, the model is probably under-decomposed
- If you cannot draw a use case end-to-end, the missing arrow is the answer

## Verbal Patterns That Score Well

- "I'll model `Loan` as the aggregate root because it owns the lifecycle of the borrowing event."
- "`Money` is a value object — equality by amount and currency."
- "I'm using composition between `Book` and `Copy` because deleting the book removes its physical copies."
- "I'll skip the back-reference from `Member` to `Loan` until a use case needs it."

## Related

- [approach-ood-interviews.md](approach-ood-interviews.md)
- [approach-machine-coding-interviews.md](approach-machine-coding-interviews.md)
- [write-clean-code-in-interview.md](write-clean-code-in-interview.md)
- [choose-design-patterns.md](choose-design-patterns.md)
- [handle-concurrency-scenarios.md](handle-concurrency-scenarios.md)
- [test-doubles-in-ood.md](test-doubles-in-ood.md)
- [../class-relationships/](../class-relationships/)
- [../solid/](../solid/)
- [../design-patterns/](../design-patterns/)
- [../../system-design/INDEX.md](../../system-design/INDEX.md)

## References

- "Cracking the Coding Interview" — OOD section
- "Designing Data-Intensive Applications" — for aggregate and identity thinking
