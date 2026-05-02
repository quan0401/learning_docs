---
title: "Handling 'Extend to Support X' Interview Follow-ups"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, interview-method, extensibility, ocp, yagni]
---

# Handling "Extend to Support X" Interview Follow-ups

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `interview-method` `extensibility` `ocp` `yagni`

## Summary

The first half of an OOD interview is "design X." The second half — and often the more revealing half — is "now extend it to support Y." The interviewer is checking whether your model has good seams or whether it has to be torn apart at every change. The candidate's job is to show *judgment*: when to extend in place, when to introduce a new abstraction, and when to push back with YAGNI without sounding lazy.

This document gives a decision framework for the most common extension follow-ups (multi-tenant, i18n, audit logging, soft delete, versioning, undo/redo, import/export, plugins) and walks a worked example: extending the parking-lot LLD to handle EVs, then reservations, then multi-location.

## Table of Contents

- [What the Follow-up Is Really Testing](#what-the-follow-up-is-really-testing)
- [The Three Responses](#the-three-responses)
- [Stretch vs Introduce](#stretch-vs-introduce)
- [When to Use Strategy](#when-to-use-strategy)
- [When to Use Visitor](#when-to-use-visitor)
- [When to Use Open-Closed Extension Points](#when-to-use-open-closed-extension-points)
- [When to Refuse with YAGNI](#when-to-refuse-with-yagni)
- [Common Follow-ups: Catalog](#common-follow-ups-catalog)
- [Worked Example: Parking Lot](#worked-example-parking-lot)
- [Interview Talking Points](#interview-talking-points)
- [Related](#related)
- [References](#references)

## What the Follow-up Is Really Testing

Extension questions probe four things at once:

1. **Did you anticipate change?** Are the right axes of variation already abstracted?
2. **Can you reason about cost?** Do you know the difference between a 10-minute change and a 2-week refactor, and can you defend the estimate?
3. **Do you over-engineer?** Will you build a plugin system to support one extra case?
4. **Can you negotiate scope?** "Yes but" and "No because" are both valid answers in real engineering.

A good answer is rarely "yes I built that already." A great answer is "here is how this maps onto the existing model, here is what changes, here is what I would push back on."

## The Three Responses

Every extension request has one of three correct responses:

### 1. "It already supports that."

If the existing abstraction handles it — a new `Strategy`, a new subclass, a new value in a config — say so and walk through the change in 30 seconds. This is the best outcome and confirms the original design was OCP-aligned.

### 2. "Here is the small change to support it."

The model needs an additive change: a new method on an interface, a new field on a value object, a new branch behind an abstraction that already exists. Quote the change, name the files touched, and move on.

### 3. "I would push back on that — here is why."

Sometimes the requested extension is speculative, costly, and unjustified. YAGNI applies. Articulate the cost, propose a smaller alternative, and ask what concrete need is driving the request.

The mistake to avoid: treating every follow-up as a chance to draw boxes. The interviewer is not paying you by the abstraction.

## Stretch vs Introduce

The central tactical decision is: **stretch the existing model**, or **introduce a new abstraction**.

### Stretch

- The new case differs only in *data* from the existing cases.
- An existing extension point (interface, strategy, config) accommodates it directly.
- Cost: minutes. Risk: low.

### Introduce

- The new case differs in *behavior* in ways the existing model cannot express.
- An existing extension point would be deformed by forcing the case through it.
- Cost: hours to days. Risk: medium. The new abstraction must justify its weight.

Heuristic: if accommodating the new case requires `if` statements scattered across multiple methods, you are stretching too far — introduce. If it requires only a new entry in a registry or a new subclass, stretch.

The smell of stretching too far is **type checks creeping into business logic**: `if (vehicle instanceof EVCar)` showing up in three different services. That is the model telling you a new abstraction is overdue.

## When to Use Strategy

Strategy is the right answer when **policy varies independently of the surrounding workflow**. Classic shapes:

- Pricing: `PricingStrategy.calculate(Ticket, Duration) -> Money` — flat rate, hourly, peak/off-peak, custom contract
- Routing: `RoutingStrategy.pickGate(Vehicle) -> Gate` — nearest, least-loaded, by type
- Selection: `SpotAllocationStrategy.allocate(Vehicle, List<Spot>) -> Optional<Spot>`

Signal: the interviewer says "we want to support a different way of doing X." If X is a single, well-bounded decision with a clear input/output contract, Strategy is the right hammer.

```java
public interface PricingStrategy {
    Money calculate(Ticket ticket, Instant exitTime);
}

public class HourlyPricing implements PricingStrategy { ... }
public class PeakOffPeakPricing implements PricingStrategy { ... }
public class ContractPricing implements PricingStrategy { ... }
```

The host class holds a `PricingStrategy` reference. Adding a new policy is one new class — no existing code changes.

## When to Use Visitor

Visitor is the right answer when **operations vary over a stable hierarchy**. The hierarchy of types rarely changes; the operations you want to perform on it grow over time.

Classic example: AST nodes. `Add`, `Multiply`, `Literal`, `Variable` rarely change. But you keep adding operations: pretty-printing, evaluating, type-checking, optimizing. Visitor lets you add operations without modifying the node classes.

In an interview, Visitor is over-applied. Bring it up only when:

- The type hierarchy is closed (you control all subclasses)
- Multiple operations need to dispatch on type
- Adding a new operation should not require touching every subclass

If the interviewer says "we want to add a new vehicle type," Visitor is *the wrong answer* — you would have to touch every visitor. If they say "we want to add a new report over existing vehicles," Visitor fits.

## When to Use Open-Closed Extension Points

Open-Closed Principle: open for extension, closed for modification. In practice, this means designing a small interface that *future* implementations can satisfy, with the host class depending only on the interface.

```java
public interface DiscountRule {
    Optional<Money> apply(Ticket t, Money base);
}

public class BillingService {
    private final List<DiscountRule> rules;
    public Money price(Ticket t, Money base) {
        Money current = base;
        for (DiscountRule r : rules) {
            current = r.apply(t, current).orElse(current);
        }
        return current;
    }
}
```

Adding a `LoyaltyDiscount` or `WeekendDiscount` is one new class registered into the list. `BillingService` is closed for modification but open to new rules. This is the OCP form of plug-in extension within a single process.

The trap: pre-building OCP scaffolding for variation that may never come. OCP is *retroactive insurance*, not *speculative architecture*. Build it when the second variation lands, not on the chance the second variation might land.

## When to Refuse with YAGNI

"You aren't gonna need it" — the rule that you do not implement until the need is concrete and present.

The interview version is more nuanced because the interviewer is *literally telling you* you will need it. But you can still refuse a *specific shape* of solution:

- "We may want to support multiple currencies later." → "I would not introduce currency conversion now. I would store an explicit `Currency` on every `Money`, fail loudly on cross-currency operations, and add conversion when there is a real second currency."
- "We may want to plug in third-party rules engines." → "I would design a clean `PricingStrategy` interface today. A plugin loader and DSL would come if and when a second engine is real."

The pattern: **acknowledge the direction, refuse the speculation, name the trigger that would change your mind**. That is YAGNI applied with judgment, not YAGNI used as an excuse.

Articulating it without sounding lazy:

- "I want this design to *not block* that extension, but I do not want to *pay for* it now."
- "The cost of adding it later is roughly equal to adding it now, so I would defer."
- "If we do this speculatively, we will get the abstraction wrong, because we do not yet know what the second case looks like."

## Common Follow-ups: Catalog

Quick reference for the extensions that come up most often.

### Multi-tenant

- Add a `tenantId` to every aggregate root and every query. Index it.
- Tenant scoping at the repository layer, not in business logic.
- Decision point: shared schema with `tenantId` column, schema-per-tenant, or DB-per-tenant. Cheapest to start: shared schema. Hardest to retrofit: physical isolation.
- Watch for: cross-tenant leaks in caches, in logs, in batch jobs.

### Internationalization

- Separate the string from where it is shown. Identifiers (`"errors.payment.declined"`) in code; translations in resource bundles.
- Money, dates, and numbers need locale-aware formatting — never `String.format` for user-facing output.
- Pluralization is its own problem (ICU MessageFormat handles it).
- Right-to-left layout affects UI, not domain — usually out of scope for OOD.

### Audit Logging

- A cross-cutting concern. Decorator on the use case, or domain event raised by the aggregate.
- Capture: who, what, when, before, after. Do not capture sensitive payloads in cleartext.
- Append-only store. Never mutate audit rows.
- Decision point: synchronous write (consistent but slow) vs async via outbox (consistent eventually, faster).

### Soft Delete

- Add `deletedAt: Instant?` (nullable) instead of removing rows.
- Filter `deletedAt IS NULL` at the repository boundary, by default.
- Provide an explicit "include deleted" path for restore and admin.
- Watch for: unique constraints (a deleted email should not block reuse), foreign keys (the parent is "deleted" but children still reference it).

### Versioning

- API versioning: `/v1/` in the URL, or content-negotiated. Pick one and commit.
- Schema versioning: every event/document carries a `schemaVersion` field. Consumers handle multiple versions during migration windows.
- Optimistic concurrency: a `version` column on the row, incremented on every update; updates fail if the expected version does not match.

### Undo/Redo

- Command pattern. Each user action is a `Command` with `execute()` and `undo()`.
- Stack of executed commands for undo; separate stack for redo.
- Hard parts: undo of operations that touched external state (sent email, charged card) — usually impossible, so treat them as un-undoable and surface that to the user.

### Import/Export

- Define an explicit DTO layer separate from the domain model. The wire format is *not* your domain model.
- Streaming, not load-everything-into-memory, for files of unknown size.
- Idempotency keys on import — running the same file twice should not create duplicates.

### Plugins

- A plugin is a third party (or first-party but separate) implementation of a known interface.
- Decisions: discovery (config file, classpath scan, ServiceLoader), isolation (own classloader, own process), trust boundary.
- A plugin system is a major architectural commitment. Refuse it until the second concrete plugin exists.

## Worked Example: Parking Lot

Walk through three escalating follow-ups on a parking-lot LLD. Assume the original design has:

- `Vehicle` (abstract), with `Car`, `Bike`, `Truck`
- `Spot` with a `VehicleType` it accepts
- `ParkingLot` with `park(Vehicle)` and `leave(Ticket)`
- `PricingStrategy` interface
- `Ticket` and `Receipt` value objects

### Follow-up 1: "Now support EVs."

**Stop and ask one question:** "Is the EV behavior different from a regular car, or do EVs need charging spots specifically?"

If the answer is "EVs need charging spots":

- *Stretch*. Add a `boolean hasCharger` to `Spot`. Add an `isElectric()` to `Vehicle` (or a subclass `ElectricCar extends Car`). Allocation logic: an EV may take a charging spot or a regular spot; a non-EV may not take a charging spot if any non-charging spot is free.
- The change touches `Spot`, `Vehicle`, and the allocation strategy. No new top-level abstraction needed.
- Pricing? Add an `ElectricSurcharge` `DiscountRule` (or `PriceAdjustmentRule`) — composable with the existing pricing chain.

If the answer is "EVs are just cars":

- *Stretch even less*. A flag on `Car`. No allocation change. Done.

The interviewer is watching whether you ask before designing.

### Follow-up 2: "Now support reservations."

This is a bigger change. A reservation is a *temporal commitment of a spot to a vehicle that has not yet arrived*.

The model now needs:

- A `Reservation` aggregate: `{ spotId, vehicleId or driverId, windowStart, windowEnd, status }`.
- A new state on `Spot`: `AVAILABLE`, `OCCUPIED`, `RESERVED` (with reservation ref).
- Allocation rule changes: a reserved spot is not available to walk-ins during its window, even though it is physically empty.
- Lifecycle: reservation can be CONFIRMED, FULFILLED (vehicle arrived), NO-SHOW (window expired), CANCELLED.

Key decisions to articulate:

- **Where does the reservation logic live?** A new `ReservationService` (use case), separate from `ParkingLot`. `ParkingLot` consults it during allocation.
- **Concurrency.** Two parallel reservation requests for the same spot must not both succeed. Optimistic locking on `Spot`, or a row-level DB lock.
- **No-show policy.** Configurable: hold the reservation for N minutes past start, then release. This is itself a `Strategy`.

What you should *not* do: add a `reservations` field to `Spot` and mix the workflow into `ParkingLot`. That conflates two responsibilities. Introduce a new abstraction (`ReservationService`) — a real OCP/SRP win.

### Follow-up 3: "Now multi-location."

A genuine architectural change. The original design assumed one `ParkingLot`. Multi-location means:

- A `Location` entity owning many `ParkingLot` instances (or `Floor`s).
- Cross-location queries: "find me a spot near my destination," "what is my reservation across all my visits?"
- Cross-location pricing: per-location rates, possibly tenant-style.
- Cross-location concurrency: a reservation system that spans physical sites is closer to a distributed system question than an OOD question.

Push back here if scope gets vague: "Are we still designing the in-memory model for one location with location-aware extensions, or are we now designing the distributed system?" The right OOD answer:

- Hoist `Location` as the new aggregate boundary.
- `ParkingLotService` becomes `LocationService`. Most existing classes survive unchanged but are now scoped by `LocationId`.
- Repositories carry `LocationId`. Queries scope to it.
- Pricing strategies are looked up per-location.
- Reservations are also scoped per-location; a future "find me a reservation anywhere" feature is a separate query layer.

The decision tree for the candidate:

```
"Extend X" arrives
  |
  +-- Does the existing model already support it? -- Yes --> walk through, done
  |
  +-- Is it a small additive change? -- Yes --> add field/subclass/strategy, done
  |
  +-- Does it require new behavior orthogonal to current model?
  |     Yes --> introduce a new abstraction, name it, name what it owns,
  |             explain what it touches and what stays the same
  |
  +-- Does it shift the aggregate boundary?
  |     Yes --> say so explicitly. Re-scope the model. Negotiate what
  |             stays in this design vs what becomes a separate concern.
  |
  +-- Is the request speculative?
        Yes --> articulate YAGNI, propose the smaller version, name the trigger
                that would justify the larger version.
```

## Interview Talking Points

- "Before I extend, let me check what the existing model already gives me." — establishes that you are reusing the design, not bypassing it.
- "I would stretch the existing `X` here because the variation is in data, not behavior." — names the heuristic.
- "I would introduce a new `Y` here because pushing this through `X` would scatter type checks across services." — names the smell.
- "I would push back on that for now — here is the smaller version that does not block it later." — refusing without sounding rigid.
- "What is the concrete business case driving this? That changes how I shape it." — buys time and scopes the answer.

## Related

- [Approach OOD Interviews](./approach-ood-interviews.md)
- [Choose Design Patterns](./choose-design-patterns.md)
- [Identify Entities and Model Relationships](./identify-entities-and-model-relationships.md)
- [Designing for Testability](./designing-for-testability.md)
- [Object Lifecycle and Resource Management](./object-lifecycle-and-resource-management.md)
- [Open-Closed Principle](../solid/open-closed-principle.md)
- [Single Responsibility Principle](../solid/single-responsibility-principle.md)
- [Strategy Pattern](../design-patterns/behavioral/)
- [Visitor Pattern](../design-patterns/behavioral/)
- [Command Pattern](../design-patterns/behavioral/)
- [YAGNI Principle](../design-principles/yagni-principle.md)
- [KISS Principle](../design-principles/kiss-principle.md)
- [Code Smells and Refactoring Triggers](../design-principles/code-smells-and-refactoring-triggers.md)
- [Anti-patterns in OO Design](../design-principles/anti-patterns-in-oo-design.md)

## References

- Robert C. Martin, *Clean Architecture* — open-closed, dependency rule, plugin architecture
- Robert C. Martin, *Clean Code* — small, focused changes; resist speculative generality
- Joshua Bloch, *Effective Java* — favor composition over inheritance, design for change
- Michael Feathers, *Working Effectively with Legacy Code* — extension via seams
- Erich Gamma et al., *Design Patterns* — Strategy, Visitor, Command, Decorator
- Eric Evans, *Domain-Driven Design* — aggregate boundaries; when extension forces a new aggregate
