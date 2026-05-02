---
title: "How to Write Clean Code in an Interview"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, interview-method, clean-code, naming, code-quality]
---

# How to Write Clean Code in an Interview

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `interview-method` `clean-code` `naming` `code-quality`

## Summary

Clean code in an interview is not maximum polish — it is intentional polish on the surfaces a reviewer actually inspects. Names carry the most signal, function size signals discipline, comments justify rather than describe, and explicit error handling separates seniors from juniors. Invest reviewer time on the public API of your services and on the headline domain entities.

## What Reviewers Read First

In a 10-minute review of your submission, an experienced reviewer looks at:

1. **The README** — does it tell me how to run and what was skipped?
2. **The headline service class** — is the API obvious?
3. **One domain entity** — is it well-bounded, immutable where possible?
4. **One test** — does it pass for the right reason?
5. **The longest function** — is it actually that long, or could it be split?

Polish *those* surfaces. Everything else is supporting cast.

## Naming

Naming is where seniority shows fastest.

### Class Names

- Match domain language. `Loan`, not `BookBorrowingRecord`.
- Avoid `Manager`, `Handler`, `Util` unless you genuinely mean it. Prefer the verb-noun like `LoanService`, `LateFeeCalculator`.
- Suffix services with `Service`, repositories with `Repository`, factories with `Factory`. Be predictable.

### Method Names

- Verb phrases that read like English. `member.borrow(book)`, not `member.processLoan(book)`.
- Boolean methods start with `is`, `has`, `can`, `should`. `loan.isOverdue(now)`.
- Avoid `getX` for behavior. `getCurrentTime()` is fine for a value, but `getNextValidSlot()` should be `findNextValidSlot()` or `nextValidSlot()`.

### Variable Names

- One-letter names only in tight scopes (`for (int i = 0; ...)` is fine; `int i` for a top-level field is not).
- Boolean variables should read as questions: `isAvailable`, `hasReservation`.
- Prefer concrete over abstract: `vehicleType` over `type`, `entryTime` over `t1`.

### Java Example

```java
// Weak
public class LM {
    public R proc(M m, B b) { ... }
}

// Strong
public class LoanService {
    public Loan borrow(Member member, Book book) { ... }
}
```

### TypeScript Example

```typescript
// Weak
class LM {
  proc(m: M, b: B): R { ... }
}

// Strong
class LoanService {
  borrow(member: Member, book: Book): Loan { ... }
}
```

## Function Size

A function's job is to do **one** thing at one level of abstraction.

- Target under 20 lines. Hard cap at 50.
- If you find yourself adding a comment to label a section, extract that section into a method.
- Early returns beat nested conditionals. Bail on invalid input first, then write the happy path linearly.

```java
// Nested - hard to follow
public Receipt exit(UUID ticketId) {
    Optional<Ticket> ticket = tickets.find(ticketId);
    if (ticket.isPresent()) {
        if (ticket.get().isActive()) {
            Money fee = billing.calculate(ticket.get(), clock.instant());
            ticket.get().slot().release();
            return new Receipt(ticket.get(), fee);
        } else {
            throw new IllegalStateException("Ticket already closed");
        }
    } else {
        throw new UnknownTicketException(ticketId);
    }
}

// Early returns - reads like prose
public Receipt exit(UUID ticketId) {
    Ticket ticket = tickets.find(ticketId)
            .orElseThrow(() -> new UnknownTicketException(ticketId));
    if (!ticket.isActive()) {
        throw new IllegalStateException("Ticket already closed");
    }
    Money fee = billing.calculate(ticket, clock.instant());
    ticket.slot().release();
    return new Receipt(ticket, fee);
}
```

## Comments

Good comments explain **why**, not **what**.

```java
// Bad - describes the code
// Increment counter by one
counter++;

// Good - explains the why
// Reservation extends the implicit hold by 5 minutes to account for clock drift
// between gateway and gate hardware.
expiresAt = expiresAt.plusMinutes(5);
```

Rules of thumb:

- If the code needs a comment to be readable, the code is wrong.
- Comment **invariants** and **non-obvious decisions**.
- Do not narrate. `// Now we calculate the fee` is noise.
- Public method JavaDoc/TSDoc is acceptable when the contract is non-trivial — preconditions, postconditions, throws.

## Error Handling

Junior code crashes or silently swallows. Senior code makes the failure mode explicit.

### Java

- Custom exception types per domain failure: `NoSlotAvailableException`, `UnknownTicketException`. Avoid `RuntimeException` everywhere.
- Validate at constructors using `Objects.requireNonNull` or fail-fast guards.
- Use `Optional` for absence, not `null`.
- Catch only what you can handle. Re-throw or wrap otherwise.

### TypeScript

- Custom error classes: `class NoSlotAvailableError extends Error`.
- `Result<T, E>` or discriminated unions when error paths are part of the API.
- `null` is acceptable for "not found", but be consistent.
- Never `catch (e) {}` empty.

```java
public Slot assign(Vehicle vehicle) {
    if (this.vehicle != null) {
        throw new SlotOccupiedException(this.id);
    }
    if (!this.canFit(vehicle.type())) {
        throw new VehicleTypeMismatchException(this.id, vehicle.type());
    }
    this.vehicle = vehicle;
    return this;
}
```

```typescript
assign(vehicle: Vehicle): Slot {
  if (this.vehicle !== null) throw new SlotOccupiedError(this.id);
  if (!this.canFit(vehicle.type)) {
    throw new VehicleTypeMismatchError(this.id, vehicle.type);
  }
  this.vehicle = vehicle;
  return this;
}
```

## Immutability

Mutation is the source of most subtle bugs. Default to immutable.

- Java: prefer `final` fields, return new collections via `List.copyOf`, expose `Optional` rather than nullable.
- TypeScript: prefer `readonly` fields, never push into shared arrays, return spread copies.
- Reserve mutation for entities with a real lifecycle (`Loan.complete()`).

When you do mutate, make the method name a clear verb that signals state change: `markCompleted()`, `release()`, `cancel()`.

## Where to Invest Reviewer Time

| Surface | Polish Level | Why |
|---|---|---|
| Service public methods | High | First read; defines the API |
| Domain entities | High | Reveals modeling choices |
| Headline test | High | Proves the model works |
| Repository interface | Medium | Shows seam thinking |
| Custom exceptions | Medium | Shows error discipline |
| `main()` / driver | Low | Read last, if at all |
| Build config | Low | Only matters if it breaks |
| Comment density | Variable | Justify decisions, not narrate |

## Tactical Cleanups Before You Submit

Run a 5-minute pass at the end:

- [ ] Delete commented-out code
- [ ] Delete `console.log` / `System.out.println` debug lines
- [ ] Rename any `tmp`, `data`, `result` variables
- [ ] Extract any function over 30 lines
- [ ] Remove unused imports
- [ ] Check that one test actually fails when you break the implementation
- [ ] Re-read your README — does it match what you built?

## Common Anti-Patterns to Avoid

### The God Service

`ParkingService` with 30 methods that does everything. Split by use case axis: `EntryService`, `ExitService`, `BillingService`.

### Public Setters Everywhere

If you can mutate any field from outside, you have no encapsulation. Validate on construction; expose only intentional mutation methods.

### Defensive `null` Checks Everywhere

`if (member != null && member.getName() != null && ...)` is a smell. Validate at the boundary, then trust your invariants.

### Magic Numbers

```java
// Bad
if (loan.daysOpen() > 14) { ... }

// Good
private static final int LOAN_PERIOD_DAYS = 14;
if (loan.daysOpen() > LOAN_PERIOD_DAYS) { ... }
```

Better still — pass it as configuration so it is testable.

### Catch and Rethrow as RuntimeException

If the only thing you do with an exception is wrap it, you have lost information. Either let it propagate, handle it meaningfully, or wrap it in a domain-specific exception that adds context.

### One-Letter Local Variables Outside Loops

`int x = service.getCount();` — read this back in 6 months and you will hate yourself.

## Verbal Patterns That Earn Trust

- "Let me extract that into a method — it is doing two things."
- "I'll make this immutable; only `complete()` mutates state."
- "I'm throwing a domain exception here so the controller can decide the HTTP code later."
- "Let me rename — `proc` is too cute."
- "I'll inject `Clock` so this is testable without `Thread.sleep`."

## Pre-Submission Checklist

Before declaring done:

- [ ] Every public method has a name that reads like the domain
- [ ] No function exceeds 50 lines
- [ ] No file exceeds 400 lines
- [ ] Errors are explicit, never silent
- [ ] At least one test passes; at least one error path is tested
- [ ] No mutation without a verb method
- [ ] README explains run, design, and skipped items

## Related

- [approach-ood-interviews.md](approach-ood-interviews.md)
- [approach-machine-coding-interviews.md](approach-machine-coding-interviews.md)
- [identify-entities-and-model-relationships.md](identify-entities-and-model-relationships.md)
- [choose-design-patterns.md](choose-design-patterns.md)
- [handle-concurrency-scenarios.md](handle-concurrency-scenarios.md)
- [test-doubles-in-ood.md](test-doubles-in-ood.md)
- [../solid/](../solid/)
- [../design-patterns/](../design-patterns/)
- [../../system-design/INDEX.md](../../system-design/INDEX.md)

## References

- "Cracking the Coding Interview"
- Meszaros, "xUnit Test Patterns" — for the test-readability lens
