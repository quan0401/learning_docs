---
title: "How to Approach Machine-Coding Interviews"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, interview-method, machine-coding, interview-prep, scaffolding]
---

# How to Approach Machine-Coding Interviews

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `interview-method` `machine-coding` `interview-prep` `scaffolding`

## Summary

Machine-coding interviews are 60-120 minute timed problems where you must produce runnable code that demonstrates a real domain model. Unlike OOD whiteboard rounds, the deliverable is compiled, executable, and often tested. Success comes from aggressive scaffolding choices — in-memory over database, command-line over REST, clarity over completeness — and from polishing the right surfaces while skipping cosmetics.

## What This Round Tests

The interviewer wants evidence that you can:

- Translate a domain model into idiomatic code under pressure
- Commit to clean abstractions before the second hour, not at the end
- Decide what to skip — and articulate that decision
- Write code that is reviewable, not just runnable
- Demonstrate testing instinct, even if not full coverage

## Format Variants

| Variant | Duration | Typical Constraint |
|---|---|---|
| Take-home | 4-24 hours | Must include README + tests |
| Onsite live | 60-120 min | Single sitting, share screen |
| Pair-programming | 60-90 min | Interviewer interrupts |
| Async + review | Variable | Submit code, then 30-min defense |

Live and pair are the most common at the senior IC level. Calibrate scaffolding aggressiveness by available time.

## The Scaffolding Decision Tree

The single biggest determinant of outcome is what you skip.

### Always Prefer

- **In-memory state** over database. Use `Map`, `List`, in-memory repository.
- **Command-line driver** or test harness over REST/HTTP server.
- **Synchronous** over async unless concurrency is the point.
- **Stubbed time and IDs** (`Clock` interface, `IdGenerator`) — makes tests trivial.
- **Simple package structure** — flat is fine for 90 minutes.
- **Standard library** over heavy frameworks. Plain Java with JUnit; plain TypeScript with Vitest.

### Skip Unless Asked

- Persistence layer
- Authentication
- Logging configuration
- Build tool flexibility
- Production-grade error handling
- Internationalization
- Pagination beyond a limit/offset

### Polish These — They Get Read

- Public API of your service classes
- Domain entities (immutability, equality, validation)
- One end-to-end test that exercises the headline use case
- README with how-to-run, design notes, what's skipped and why

## Time Allocation for a 90-Minute Slot

| Phase | Time | What you do |
|---|---|---|
| Read & clarify | 5-10 min | Re-read prompt, write open questions, ask them |
| Domain sketch | 10 min | Whiteboard or comment-block: entities, key methods |
| Skeleton + first test | 15-20 min | Folders, classes, one failing test for the core flow |
| Core implementation | 30-40 min | Make headline test pass, then expand |
| Edge cases | 10-15 min | Error paths, boundaries, second test |
| Cleanup & README | 5-10 min | Naming pass, remove dead code, write notes |

If you are at 60 minutes with no passing test, scope-cut immediately.

## Recommended Project Layout

For Java + JUnit:

```
src/
  main/java/lld/
    domain/
      Vehicle.java
      Slot.java
      Ticket.java
    service/
      ParkingService.java
      BillingService.java
    repo/
      InMemorySlotRepo.java
    Main.java
  test/java/lld/
    ParkingServiceTest.java
README.md
```

For TypeScript + Vitest:

```
src/
  domain/
    vehicle.ts
    slot.ts
    ticket.ts
  service/
    parkingService.ts
    billingService.ts
  repo/
    inMemorySlotRepo.ts
  index.ts
test/
  parkingService.test.ts
README.md
package.json
```

Flat enough to fit in a screen-share, structured enough to look intentional.

## A Minimal Service Skeleton

```java
public final class ParkingService {
    private final SlotRepository slots;
    private final TicketRepository tickets;
    private final BillingService billing;
    private final Clock clock;

    public ParkingService(SlotRepository slots,
                          TicketRepository tickets,
                          BillingService billing,
                          Clock clock) {
        this.slots = slots;
        this.tickets = tickets;
        this.billing = billing;
        this.clock = clock;
    }

    public Ticket enter(Vehicle vehicle) {
        Slot slot = slots.findFirstAvailable(vehicle.type())
                .orElseThrow(() -> new NoSlotAvailableException(vehicle.type()));
        slot.assign(vehicle);
        Ticket ticket = new Ticket(slot, clock.instant());
        tickets.save(ticket);
        return ticket;
    }

    public Receipt exit(UUID ticketId) {
        Ticket ticket = tickets.find(ticketId)
                .orElseThrow(() -> new UnknownTicketException(ticketId));
        Money fee = billing.calculate(ticket, clock.instant());
        ticket.slot().release();
        return new Receipt(ticket, fee);
    }
}
```

```typescript
export class ParkingService {
  constructor(
    private readonly slots: SlotRepository,
    private readonly tickets: TicketRepository,
    private readonly billing: BillingService,
    private readonly clock: Clock,
  ) {}

  enter(vehicle: Vehicle): Ticket {
    const slot = this.slots.findFirstAvailable(vehicle.type);
    if (!slot) throw new NoSlotAvailableError(vehicle.type);
    slot.assign(vehicle);
    const ticket = new Ticket(slot, this.clock.now());
    this.tickets.save(ticket);
    return ticket;
  }

  exit(ticketId: string): Receipt {
    const ticket = this.tickets.find(ticketId);
    if (!ticket) throw new UnknownTicketError(ticketId);
    const fee = this.billing.calculate(ticket, this.clock.now());
    ticket.slot.release();
    return new Receipt(ticket, fee);
  }
}
```

Both versions inject `Clock` so tests are deterministic — see [test-doubles-in-ood.md](test-doubles-in-ood.md).

## What "Done" Looks Like

A confident submission demonstrates:

1. Headline use case runs end-to-end via test or `main()`
2. At least one positive test and one error-path test
3. Clear domain entities with validation
4. README that lists what was skipped and why
5. No commented-out code, no `TODO` left in the hot path

## Common Stumbles

### Spending 30 Minutes on the Build Tool

Pick the simplest path that compiles. Maven/Gradle/Vite/npm scripts — whichever you can configure in 2 minutes blind.

### Drowning in Inheritance

Three vehicle subclasses are fine. Twelve abstract levels with overridden hooks for "extensibility" is theater. See [choose-design-patterns.md](choose-design-patterns.md).

### Premature Persistence

Implementing JDBC or an ORM in the first hour. Use a `Map<UUID, Ticket>` repository. Show the seam (`SlotRepository` interface) so a real DB could slot in.

### No Tests Until the End

Tests in the last 5 minutes are noise. Write the first test in the first 20 minutes — it forces clean APIs.

### Solving a Bigger Problem Than Asked

If the prompt does not mention payments, do not implement Stripe webhooks. The interviewer is watching for scope discipline.

### "It Works" but Unrunnable on Their Machine

Always include a one-line run command in the README. `./gradlew test`, `npm test`, `mvn test`. They will copy-paste it.

### Forgetting to Handle the Empty Case

Empty parking lot, no tickets, expired ticket. These are quick wins that signal seniority.

## What to Polish vs Skip

| Polish | Skip |
|---|---|
| Domain class equality and validation | Pretty CLI output |
| One green end-to-end test | 100% line coverage |
| Service-layer error types | Custom logger configuration |
| README run instructions | Multi-environment config |
| Constructor injection seams | Spring/NestJS DI containers |
| Idempotent operations where natural | Distributed locking |

## Take-Home Differences

If you have hours, not minutes:

- Add a real test suite (not just one happy path)
- Include a short ARCHITECTURE.md explaining trade-offs
- Add concurrency notes if state is shared — see [handle-concurrency-scenarios.md](handle-concurrency-scenarios.md)
- Provide an extension point worked example ("If we added EVs, here is the diff")
- Do not over-engineer. The reviewer compares time-spent vs polish — gold-plating reads as inefficient

## During the Interview — Verbal Patterns

- "I'm going to skip persistence and use an in-memory repo behind an interface so it can be swapped later."
- "I'll inject a Clock so I can write deterministic tests for the billing logic."
- "Given my remaining time, I'll cut the multi-floor logic and document the extension point in the README."
- "Let me write a failing test for the headline flow before I implement the service."

## Final Checklist Before You Stop Typing

- [ ] Code compiles cleanly, no warnings
- [ ] Headline test passes
- [ ] One error path is exercised
- [ ] No commented-out blocks
- [ ] README explains run command and skipped items
- [ ] Class names and method names read like the domain
- [ ] No god classes, no dead abstractions
- [ ] Concurrency boundary mentioned if state is shared

## Related

- [approach-ood-interviews.md](approach-ood-interviews.md)
- [identify-entities-and-model-relationships.md](identify-entities-and-model-relationships.md)
- [write-clean-code-in-interview.md](write-clean-code-in-interview.md)
- [choose-design-patterns.md](choose-design-patterns.md)
- [handle-concurrency-scenarios.md](handle-concurrency-scenarios.md)
- [test-doubles-in-ood.md](test-doubles-in-ood.md)
- [../solid/](../solid/)
- [../design-patterns/](../design-patterns/)
- [../../system-design/INDEX.md](../../system-design/INDEX.md)

## References

- "Cracking the Coding Interview" — for the meta-prep approach
- Meszaros, "xUnit Test Patterns" — for test scaffolding decisions
