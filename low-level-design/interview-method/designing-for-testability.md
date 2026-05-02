---
title: "Designing for Testability"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, interview-method, testability, dip, seams]
---

# Designing for Testability

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `interview-method` `testability` `dip` `seams`

## Summary

Testability is not a separate quality you bolt on after writing the production code. It is a *direct consequence of structural decisions* — where dependencies cross the boundary, where side effects live, whether time and randomness are injected or hard-wired. Code that is easy to test tends to be code that is easy to reason about, easy to extend, and easy to refactor. In interviews, a candidate who naturally pushes I/O to the edges, injects collaborators, and avoids hidden global state signals senior judgment without ever saying the word "test."

This document covers the structural patterns and smells that determine whether a class is testable, the seams concept from Michael Feathers, the Humble Object pattern, and the tension between "tell don't ask" and observability.

## Table of Contents

- [Why Testability Is a Design Property](#why-testability-is-a-design-property)
- [Inject the Edges (Clock, Random, Network, Filesystem)](#inject-the-edges-clock-random-network-filesystem)
- [Pure Functions and Side-Effect Isolation](#pure-functions-and-side-effect-isolation)
- [Avoid `static` and Singletons in Business Logic](#avoid-static-and-singletons-in-business-logic)
- [Hard-to-Test Smells](#hard-to-test-smells)
- [The Humble Object Pattern](#the-humble-object-pattern)
- [Seams (Michael Feathers)](#seams-michael-feathers)
- [Tell-Don't-Ask: Helps and Hurts](#tell-dont-ask-helps-and-hurts)
- [Interview Talking Points](#interview-talking-points)
- [Related](#related)
- [References](#references)

## Why Testability Is a Design Property

A class is testable when you can place it in a test harness, give it controlled inputs, and observe controlled outputs without spinning up the world. If you cannot, the obstacle is almost never "we did not write tests." The obstacle is one of:

- The class constructs its collaborators internally, so you cannot substitute them
- It reads global state (singletons, statics, environment) that the test cannot reset
- It performs I/O directly (clock, network, disk) that the test cannot intercept
- Its outputs are observed only via side effects to systems the test does not own

Each of those is a *design* defect. Tests are the canary; the disease is coupling.

This is the central thesis of hexagonal architecture and of clean architecture: push effects to the boundary, keep the core a pure transformation. Hexagonal calls these "ports and adapters." Bob Martin calls it the dependency rule. Both arrive at the same shape because both are responding to the same structural pressure.

## Inject the Edges (Clock, Random, Network, Filesystem)

The four most common testability killers are *implicit ambient dependencies*: things the code reaches out and grabs without asking. The cure is dependency injection (DIP applied at the boundary).

### Clock

```java
// Hard to test — uses ambient time
public class TokenIssuer {
    public Token issue(User u) {
        return new Token(u.id(), Instant.now().plus(Duration.ofMinutes(15)));
    }
}

// Testable — clock is injected
public class TokenIssuer {
    private final Clock clock;
    public TokenIssuer(Clock clock) { this.clock = clock; }
    public Token issue(User u) {
        return new Token(u.id(), clock.instant().plus(Duration.ofMinutes(15)));
    }
}
```

In tests, pass `Clock.fixed(...)`. Java's `java.time.Clock` is built for exactly this — it exists primarily as a testability seam. In TypeScript, the same idea:

```typescript
type Clock = { now(): Date };

class TokenIssuer {
  constructor(private clock: Clock) {}
  issue(user: User): Token {
    const exp = new Date(this.clock.now().getTime() + 15 * 60 * 1000);
    return { userId: user.id, exp };
  }
}
```

### Random

The same pattern. `Math.random()` and `new Random()` are ambient. Inject a `Supplier<UUID>` or a `Random` instance, and tests can pin the sequence.

### Network and Filesystem

Wrap them behind narrow interfaces (`UserRepository`, `BlobStore`, `EmailSender`). The implementation talks to Postgres or S3 or SMTP; the test passes an in-memory fake. The class under test never knows the difference.

This is DIP at work: the high-level policy depends on an abstraction; both the production adapter and the test fake depend on the same abstraction.

## Pure Functions and Side-Effect Isolation

A pure function — same input, same output, no side effects — is the most testable construct in software. You need no mocks, no setup, no teardown. You assert the return value.

The hexagonal architecture insight is that *most business logic can be pure if you arrange the code correctly*. Effects (DB writes, HTTP calls, log emission) live in a thin shell. The core takes domain inputs and returns domain outputs.

```java
// Impure — fetches, computes, writes, emits
public BillingResult charge(String customerId) {
    Customer c = customerRepo.find(customerId);     // I/O
    Money owed = c.outstandingBalance(LocalDate.now()); // ambient time
    paymentGateway.charge(c.cardToken(), owed);     // I/O
    auditLog.record(c.id(), owed);                  // I/O
    return new BillingResult(owed);
}

// Refactored — pure core, thin shell
public class BillingPolicy {
    // pure
    public ChargeRequest plan(Customer c, LocalDate today) {
        return new ChargeRequest(c.id(), c.outstandingBalance(today));
    }
}

public class BillingService {
    public BillingResult charge(String customerId) {
        Customer c = customerRepo.find(customerId);
        ChargeRequest req = policy.plan(c, clock.today());
        paymentGateway.charge(c.cardToken(), req.amount());
        auditLog.record(c.id(), req.amount());
        return new BillingResult(req.amount());
    }
}
```

`BillingPolicy.plan` is testable with no mocks. The `BillingService` shell is integration-tested or covered with simple stubs. Most bugs live in policy; most testing pain lives in the shell. Splitting them shrinks the painful surface.

## Avoid `static` and Singletons in Business Logic

Static methods and the singleton pattern are testability traps. They look convenient — `BillingPolicy.compute(...)` reads cleanly — but they fix the dependency at compile time. There is no seam to substitute a different implementation.

Static utilities are fine when they are *truly* pure with no policy: `Math.max`, `String.format`, `Collectors.toList`. They become harmful the moment they reach for ambient state — current time, current user, environment variables, a logger that writes to disk.

Singletons are the same trap with mutable state added. `Logger.getInstance().log(...)` couples every caller to one specific logger configured exactly one way, and tests inherit that coupling.

Prefer instance methods on injected collaborators. The "extra typing" is the cost of a seam.

## Hard-to-Test Smells

A handful of patterns reliably make code hard to test. Spotting them is fast feedback that the design is drifting.

### `new` Inside a Method

```java
public Receipt processOrder(Order o) {
    PaymentGateway gw = new StripeGateway(config);
    return gw.charge(o);
}
```

The test cannot substitute the gateway. Constructor-inject `PaymentGateway` instead.

### Hidden Global State

Reading `System.getenv`, `System.getProperty`, a static config holder, or a thread-local from inside business logic. Tests then have to reach in and mutate that global, which breaks isolation between tests.

### Time and Date in Logic

```java
if (LocalDate.now().getDayOfWeek() == DayOfWeek.FRIDAY) { ... }
```

Whether the test passes depends on what day you run it. Inject the clock.

### Long Constructors That Do Work

A constructor that opens files, connects to DBs, or starts threads cannot be cheaply instantiated in a test. Constructors should *wire*, not *act*. Move work to an explicit `start()` or a factory.

### Deep Inheritance Hierarchies

Subclass behavior that depends on superclass internals forces tests to set up the entire hierarchy. Composition is more substitutable.

### Methods That Return `void` and Mutate the World

You can only test such a method by inspecting the world afterward, which means coupling the test to whatever side-channel it uses. Prefer returning a value or an event the test can inspect directly.

## The Humble Object Pattern

Bob Martin's *Humble Object* pattern (sometimes credited to Gerard Meszaros's *xUnit Test Patterns* in the testing literature) addresses the boundary between testable logic and untestable infrastructure.

The rule: when you have a component that is hard to test (a UI view, a database driver, a hardware adapter), make it as *humble* as possible — strip its logic out and push it into a testable companion. The humble piece does the bare minimum of mechanical glue; the companion holds the policy.

Classic application: GUI code. The view is humble — it only renders fields and forwards events. A presenter holds all decision logic and is fully unit-testable without a UI framework.

Same shape elsewhere:

- **Database row mapper** humble; **query result interpreter** testable
- **HTTP controller** humble; **use case object** testable
- **Scheduler thread** humble; **work item dispatcher** testable

When you cannot make something testable, make it tiny and obviously correct, and put the brains somewhere else.

## Seams (Michael Feathers)

In *Working Effectively with Legacy Code*, Michael Feathers defines a **seam** as "a place where you can alter behavior in your program without editing in that place." Seams are how you get legacy code under test without rewriting it.

Feathers catalogs several seam types:

- **Object seam** — substitute a different object via polymorphism (constructor injection, setter injection)
- **Preprocessor seam** — `#ifdef` style substitution (C/C++ only)
- **Link seam** — substitute a different binary/library at link time
- **Build seam** — substitute via build configuration (test classpath vs prod classpath)

In modern Java/TS work, the object seam dominates. Wherever a class crosses a boundary it does not own (DB, network, time), introduce an interface and inject. That interface *is* the seam.

The interview-relevant insight: **designing testable code is designing intentional seams**. You decide ahead of time where substitution will be possible. Greenfield design has the luxury of placing them well; legacy work requires finding or carving them.

## Tell-Don't-Ask: Helps and Hurts

"Tell, don't ask" — instead of pulling state out of an object and deciding what to do, tell the object what you want and let it decide. Helps encapsulation, reduces coupling, often improves testability.

```java
// Ask
if (account.getBalance() >= amount) {
    account.setBalance(account.getBalance() - amount);
}

// Tell
account.withdraw(amount);
```

The "tell" form is more testable — `Account.withdraw` has a clear contract you can assert against, and you do not need to set up balance state in the caller's test.

But tell-don't-ask can hurt testability when carried too far. If a method has no observable return value and only mutates the receiver, the test must inspect internal state to verify the outcome — coupling the test to the implementation.

The balance: tell, but make the *outcome observable*. Either return a result object, raise an event, or expose enough query surface that the test can verify behavior without knowing about private fields.

```java
// Better — tell, but the result is observable
WithdrawalResult result = account.withdraw(amount);
assertThat(result.success()).isTrue();
assertThat(result.newBalance()).isEqualTo(Money.of(50));
```

## Interview Talking Points

When the interviewer pushes on testability, work through this checklist out loud:

1. **Where do effects live?** "Time, network, and DB are behind injected interfaces. The core is pure."
2. **Where are the seams?** "Anything I cannot test in-process — payment gateway, email, clock — is an interface I can substitute."
3. **What did I avoid?** "No statics for business logic, no singletons holding mutable state, no `new` inside methods I want to test."
4. **What is humble?** "The HTTP controller and the JDBC mapper are thin. The use case and the row interpreter hold the logic."
5. **How do I observe outcomes?** "Methods return result objects or emit domain events. I do not assert on private state."

Even if the interviewer never asks for tests explicitly, this vocabulary signals that you have thought past "make it work."

## Related

- [Test Doubles in OOD](./test-doubles-in-ood.md)
- [Approach OOD Interviews](./approach-ood-interviews.md)
- [Handle Concurrency Scenarios](./handle-concurrency-scenarios.md)
- [Write Clean Code in Interview](./write-clean-code-in-interview.md)
- [Dependency Inversion Principle](../solid/dependency-inversion-principle.md)
- [Single Responsibility Principle](../solid/single-responsibility-principle.md)
- [Coupling and Cohesion](../design-principles/coupling-and-cohesion.md)
- [Separation of Concerns](../design-principles/separation-of-concerns.md)
- [Code Smells and Refactoring Triggers](../design-principles/code-smells-and-refactoring-triggers.md)

## References

- Michael Feathers, *Working Effectively with Legacy Code* — the canonical text on seams
- Robert C. Martin, *Clean Architecture* — dependency rule, humble object
- Robert C. Martin, *Clean Code* — testability as a design quality
- Gerard Meszaros, *xUnit Test Patterns* — humble object in the testing context
- Joshua Bloch, *Effective Java* — favor composition over inheritance, minimize mutability
- Alistair Cockburn, original write-up on Hexagonal Architecture (ports and adapters)
- Java `java.time.Clock` — designed as an injectable seam for time
