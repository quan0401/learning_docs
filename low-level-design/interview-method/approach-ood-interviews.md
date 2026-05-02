---
title: "How to Approach OOD Interviews"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, interview-method, ood, oop-fundamentals]
---

# How to Approach OOD Interviews

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `interview-method` `ood` `oop-fundamentals`
## Summary

Object-Oriented Design (OOD) interviews are typically 45-60 minute discussions where you must turn a fuzzy prompt ("Design a parking lot") into a concrete object model. The canonical structure is five steps: clarify requirements, identify entities, model relationships, define behaviors, and produce a diagram. The deliverable is not the "right" answer but a defensible model the interviewer can poke at.

## What an OOD Interview Actually Tests

Interviewers are not grading your knowledge of textbook patterns. They are assessing:

- Can you turn ambiguity into a written contract?
- Do you model the **domain**, or just shuffle nouns into classes?
- Do you understand encapsulation, polymorphism, and where each is the right tool?
- Can you defend a design choice by naming a trade-off?
- Will you collaborate, or argue?

A correct UML diagram with a poor justification loses to a slightly worse diagram with crisp reasoning.

## The Canonical 5-Step Structure

### Step 1 — Clarify Requirements (5-8 min)

The prompt is intentionally underspecified. Force scope before drawing anything.

**Functional questions to drive out:**

- Who are the actors? (customer, admin, system)
- What are the core use cases? (rank by priority, cap at 4-6)
- What is explicitly out of scope?
- What scale matters? (10 vehicles vs 10,000 changes the model)

**Non-functional questions:**

- Concurrency? (one user, many users, distributed?)
- Persistence? (in-memory, durable storage, both?)
- Extensibility hooks? ("we may add EV charging later")

Write the agreed scope on the board or in the editor. This becomes your verification checklist later.

### Step 2 — Identify Entities (5-10 min)

Extract nouns from the agreed requirements. Filter:

- **Real entities** — have identity and lifecycle (`Vehicle`, `Slot`, `Ticket`)
- **Value objects** — equal by value, immutable (`Money`, `Coordinate`, `LicensePlate`)
- **Roles / actors** — `Customer`, `Attendant`, `Admin`
- **Services** — orchestrate but do not represent data (`PaymentProcessor`, `SlotAllocator`)

Reject candidates that are merely attributes (`color` is a property of `Vehicle`, not a class).

See [identify-entities-and-model-relationships.md](identify-entities-and-model-relationships.md) for noun extraction technique.

### Step 3 — Model Relationships (10-15 min)

For each pair of related entities, decide:

- **Type** — association, aggregation, composition, inheritance, dependency
- **Multiplicity** — `1..1`, `1..*`, `0..*`, `0..1`
- **Direction** — bidirectional or unidirectional
- **Lifecycle** — does the parent own the child's lifetime?

Avoid the rookie move of using inheritance for everything. Composition over inheritance is the default. Reach for inheritance only when there is a true substitutability relationship (Liskov holds).

### Step 4 — Define Behaviors (10-15 min)

For each entity, list the public methods. Ask:

- What state does this method change?
- What invariants must hold before and after?
- Who is allowed to call it?
- What does it return on failure?

Sketch the most important interaction as a sequence. Example for parking lot:

```
Customer -> Gate.requestEntry(vehicle)
Gate -> SlotAllocator.findSlot(vehicle.type)
SlotAllocator -> SlotRepository.findFirstAvailable(type)
Gate -> TicketService.issue(slot, vehicle, now)
Gate -> Customer (ticket)
```

This is also where you surface concurrency — see [handle-concurrency-scenarios.md](handle-concurrency-scenarios.md).

### Step 5 — Draw the Diagram (5-10 min)

A class diagram with relationships, multiplicities, and key methods. Keep it readable, not exhaustive — 8-12 boxes is plenty. Add a short sequence diagram for the headline use case.

If time is tight, draw the class diagram first, sequence second, leave field-level detail last.

## Time Allocation Cheat Sheet

For a 60-minute round (subtract 5-10 min for intro and questions):

| Phase | Time | Output |
|---|---|---|
| Clarify | 5-8 min | Scope list, actors, non-functionals |
| Entities | 5-10 min | Class candidates with one-line purpose |
| Relationships | 10-15 min | Multiplicities, lifecycle, directionality |
| Behaviors | 10-15 min | Method signatures, key invariants |
| Diagram | 5-10 min | Class + sequence |
| Trade-offs / extensions | 5 min | "If we added X we'd change Y" |

For 45-minute rounds, compress relationships and behaviors into a single 20-minute pass. Cut the trade-off discussion last, not first.

## A Worked Mini Example

**Prompt:** "Design a parking lot."

**Clarify:**

- Multi-floor, mixed vehicle types (motorcycle, car, truck)
- Hourly billing, fixed rates per type
- One concurrent attendant per gate, multiple gates
- In-memory model is fine; persistence out of scope

**Entities:**

- `ParkingLot`, `Floor`, `Slot`, `Vehicle` (abstract), `Car/Truck/Motorcycle`, `Ticket`, `Gate`, `BillingService`

**Relationships:**

- `ParkingLot 1..* Floor` (composition)
- `Floor 1..* Slot` (composition)
- `Slot 0..1 Vehicle` (association, lifecycle independent)
- `Gate uses BillingService` (dependency)
- `Vehicle <|-- Car, Truck, Motorcycle` (inheritance for type-based slot fit)

**Behavior:**

- `Slot.canFit(vehicleType): boolean`
- `Gate.enter(vehicle): Ticket`
- `Gate.exit(ticket): Receipt`
- `BillingService.calculate(ticket, exitTime): Money`

```java
public final class Ticket {
    private final UUID id;
    private final Slot slot;
    private final Instant entryTime;

    public Ticket(Slot slot, Instant entryTime) {
        this.id = UUID.randomUUID();
        this.slot = Objects.requireNonNull(slot);
        this.entryTime = Objects.requireNonNull(entryTime);
    }

    public UUID id() { return id; }
    public Slot slot() { return slot; }
    public Instant entryTime() { return entryTime; }
}
```

```typescript
export class Ticket {
  readonly id: string;

  constructor(
    readonly slot: Slot,
    readonly entryTime: Date,
  ) {
    this.id = crypto.randomUUID();
  }
}
```

Notice both versions are immutable value-style holders — fewer concurrency landmines.

## Common Stumbles

### Skipping the Clarify Phase

Jumping straight to classes signals you cannot operate under ambiguity. Always restate scope, even if you "think you know."

### Drowning in CRUD Methods

`getX`, `setX` for every field is noise. List only behavior-bearing methods. Treat field access as implementation detail.

### Inheritance Abuse

`Manager extends Employee extends Person`. Almost always wrong. Prefer composition with `Role`, `EmploymentContract`, etc.

### One God Class

Putting all logic in `ParkingLot` because "it owns everything." Push behavior to the entity that owns the data the behavior touches (Information Expert).

### Ignoring Lifecycle

If `Floor` is composition under `ParkingLot`, deleting the lot deletes the floors. Be explicit; interviewers probe this.

### No Trade-Off Vocabulary

"I used Strategy here" is weaker than "I used Strategy because pricing rules vary by vehicle type and we want to add new types without recompiling the billing service." See [choose-design-patterns.md](choose-design-patterns.md).

### Diagram-First Thinking

Drawing UML before agreeing on scope produces beautiful but irrelevant pictures. Words first, boxes later.

### Refusing to Simplify

When time is short, cut features, not rigor. "Given remaining time, I'll model only entry/exit and stub billing" is a strong signal.

## Verbal Patterns That Score Well

- "Before I model anything, can I confirm..."
- "I'll use composition here because the lifecycle is owned."
- "If we needed to add electric vehicles, the change would be isolated to..."
- "The trade-off is X vs Y; I'm choosing X because..."
- "I'm aware this doesn't handle Z; if we have time I'll come back to it."

## Verbal Patterns That Lose Points

- "I'd just use a HashMap for everything."
- "Let me show you a Visitor pattern" (when it adds nothing)
- "It depends" without committing to one path
- Silence while drawing for 10 minutes

## Pre-Interview Practice Drill

Run this loop on at least 8-10 classic prompts (parking lot, elevator, library, vending machine, ride-share, chess, splitwise, online bookstore):

1. Set a 45-minute timer.
2. Talk out loud through all 5 steps.
3. Photograph or save the diagram.
4. Critique it the next day cold — would you accept this from a candidate?

The compounding gain from the next-day critique is larger than from another fresh attempt.

## Related

- [approach-machine-coding-interviews.md](approach-machine-coding-interviews.md)
- [identify-entities-and-model-relationships.md](identify-entities-and-model-relationships.md)
- [write-clean-code-in-interview.md](write-clean-code-in-interview.md)
- [choose-design-patterns.md](choose-design-patterns.md)
- [handle-concurrency-scenarios.md](handle-concurrency-scenarios.md)
- [test-doubles-in-ood.md](test-doubles-in-ood.md)
- [../solid/](../solid/)
- [../design-patterns/](../design-patterns/)
- [../../system-design/INDEX.md](../../system-design/INDEX.md) — for HLD interview cross-reference

## References

- "Cracking the Coding Interview" — OOD chapter
- "Designing Data-Intensive Applications" — for non-functional thinking
