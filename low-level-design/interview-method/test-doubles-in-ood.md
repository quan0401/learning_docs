---
title: "Test Doubles — Mocks, Stubs, Fakes, Spies"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, interview-method, testing, test-doubles, dip]
---

# Test Doubles — Mocks, Stubs, Fakes, Spies

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `interview-method` `testing` `test-doubles` `dip`

## Summary

"Mock" is overused as a catch-all. Gerard Meszaros's *xUnit Test Patterns* gives a precise taxonomy: dummy, stub, fake, spy, mock. Each serves a different testing need. The ability to use them at all is a design-quality signal — code that admits test doubles is code that depends on abstractions (Dependency Inversion Principle), not concretions. In an interview, naming the right double for the job demonstrates both testing instinct and architectural discipline.

## Meszaros's Taxonomy

All five types are *test doubles*: stand-ins for real collaborators in tests. They differ along two axes:

- **Does it have behavior?** (returns canned values, runs real logic, or does nothing)
- **Does it record what happened?** (used to make assertions about interactions)

| Type | Has behavior? | Records calls? | Typical use |
|---|---|---|---|
| Dummy | No | No | Fills a parameter slot; never used in the test |
| Stub | Canned | No | Provides indirect inputs |
| Fake | Working (light) | No | Real-ish implementation, unsuitable for prod |
| Spy | Real or stub | Yes | Need to verify interactions after the fact |
| Mock | Pre-programmed expectations | Yes | Strict verification of calls |

Knowing the distinction matters because the wrong type produces fragile or vague tests.

## Each Type in Detail

### Dummy

An object passed in to satisfy a parameter list, never actually used by the code under test.

```java
public class DummyClock implements Clock {
    public Instant instant() {
        throw new UnsupportedOperationException("Dummy should not be called");
    }
}
```

If the test never exercises a path that touches the clock, a dummy is fine. The throw is intentional — if the test *does* touch it, you want a loud failure, not silent zeros.

### Stub

Returns canned data. Provides indirect inputs to the code under test. No verification.

```java
public class StubBillingService implements BillingService {
    public Money calculate(Ticket t, Instant exitTime) {
        return Money.of(100, Currency.USD);
    }
}
```

```typescript
class StubBillingService implements BillingService {
  calculate(_t: Ticket, _exitTime: Date): Money {
    return new Money(100, "USD");
  }
}
```

Use when the test needs to drive a code path that depends on the result.

### Fake

A working implementation, simplified — typically in-memory.

```java
public class InMemorySlotRepo implements SlotRepository {
    private final Map<UUID, Slot> slots = new HashMap<>();

    public Optional<Slot> findFirstAvailable(VehicleType type) {
        return slots.values().stream()
                .filter(s -> s.isAvailable() && s.canFit(type))
                .findFirst();
    }

    public void save(Slot s) { slots.put(s.id(), s); }
}
```

A fake is the most useful test double in machine-coding interviews — write it once, use it everywhere, and it doubles as your initial production implementation while the real database stays out of scope.

### Spy

Wraps a real or stub implementation and records calls, so the test can assert what happened.

```java
public class SpyEmailSender implements EmailSender {
    private final List<Email> sent = new ArrayList<>();

    public void send(Email e) {
        sent.add(e);
    }

    public List<Email> sentEmails() { return List.copyOf(sent); }
}
```

```typescript
class SpyEmailSender implements EmailSender {
  readonly sent: Email[] = [];
  send(e: Email): void {
    this.sent.push(e);
  }
}
```

Test then asserts: `assertThat(spy.sentEmails()).hasSize(1);`

### Mock

A pre-programmed expectation. The test specifies in advance: "this method must be called with these arguments, this many times." A mocking framework verifies on teardown. Failing the verification fails the test.

```java
@Test
void sendsReceiptOnExit() {
    EmailSender mockSender = mock(EmailSender.class);

    Receipt receipt = parkingService.exit(ticketId);

    verify(mockSender, times(1)).send(argThat(e -> e.to().equals(member.email())));
}
```

Mocks are strict. They couple tests to call structure. Use sparingly — they make refactors painful.

## Stub vs Mock — The Common Confusion

Both can return canned values. The difference is intent:

- **Stub** is for **state-based** verification: did the system end up in the right state?
- **Mock** is for **interaction-based** verification: was the right call made?

If you would equally accept any implementation that produces the same outcome, use a stub. If "the email was sent" is the outcome, use a mock or spy.

## Test Doubles as a Design-Quality Signal

The ability to substitute a double exists only when the code depends on **an abstraction**, not a concrete class. This is the Dependency Inversion Principle in action.

Code that resists doubles:

```java
public class ParkingService {
    public Ticket enter(Vehicle v) {
        Slot slot = new SqlSlotRepository().findFirstAvailable(v.type()); // hardcoded
        Instant now = Instant.now(); // hardcoded
        ...
    }
}
```

You cannot test this without a database and a real clock. The design is wrong.

Code that admits doubles:

```java
public class ParkingService {
    private final SlotRepository slots;
    private final Clock clock;

    public ParkingService(SlotRepository slots, Clock clock) {
        this.slots = slots;
        this.clock = clock;
    }

    public Ticket enter(Vehicle v) {
        Slot slot = slots.findFirstAvailable(v.type()).orElseThrow();
        Instant now = clock.instant();
        ...
    }
}
```

Now any double can be plugged in. The test runs in milliseconds with no infrastructure.

**The rule:** if you cannot test it without standing up real infrastructure, your design has a missing seam. Add a constructor parameter, an interface, or a factory.

## When to Use Which Double

| Need | Use |
|---|---|
| Fill a parameter you don't use | Dummy |
| Drive a code path with canned data | Stub |
| Replace infrastructure with a working in-memory version | Fake |
| Verify a side effect happened | Spy |
| Verify a specific call signature was made | Mock |

A common interview-friendly mix:

- `Clock` -> stub returning a fixed `Instant`
- `Repository` -> fake (in-memory map)
- `EmailSender` / `EventPublisher` -> spy
- `IdGenerator` -> stub returning predictable IDs

## Worked Example — Parking Service Test Suite

```java
class ParkingServiceTest {
    private InMemorySlotRepo slots;       // fake
    private InMemoryTicketRepo tickets;   // fake
    private StubClock clock;              // stub
    private SpyEventPublisher events;     // spy
    private ParkingService service;

    @BeforeEach
    void setup() {
        slots = new InMemorySlotRepo();
        slots.add(new Slot("A1", VehicleType.CAR));
        tickets = new InMemoryTicketRepo();
        clock = new StubClock(Instant.parse("2026-05-02T10:00:00Z"));
        events = new SpyEventPublisher();
        service = new ParkingService(slots, tickets, new FlatBilling(), clock, events);
    }

    @Test
    void issuesTicketOnEntry() {
        Vehicle car = new Vehicle("KA-01-AA-1234", VehicleType.CAR);
        Ticket t = service.enter(car);

        assertThat(t.entryTime()).isEqualTo(clock.now());
        assertThat(slots.findById("A1").orElseThrow().isAvailable()).isFalse();
    }

    @Test
    void publishesExitEvent() {
        Vehicle car = new Vehicle("KA-01-AA-1234", VehicleType.CAR);
        Ticket t = service.enter(car);
        clock.advanceBy(Duration.ofHours(2));

        service.exit(t.id());

        assertThat(events.published()).hasSize(2); // entered, exited
        assertThat(events.published().get(1)).isInstanceOf(VehicleExitedEvent.class);
    }
}
```

```typescript
describe("ParkingService", () => {
  let slots: InMemorySlotRepo;
  let tickets: InMemoryTicketRepo;
  let clock: StubClock;
  let events: SpyEventPublisher;
  let service: ParkingService;

  beforeEach(() => {
    slots = new InMemorySlotRepo();
    slots.add(new Slot("A1", "CAR"));
    tickets = new InMemoryTicketRepo();
    clock = new StubClock(new Date("2026-05-02T10:00:00Z"));
    events = new SpyEventPublisher();
    service = new ParkingService(slots, tickets, new FlatBilling(), clock, events);
  });

  it("issues ticket on entry", () => {
    const t = service.enter(new Vehicle("KA-01-AA-1234", "CAR"));
    expect(t.entryTime).toEqual(clock.now());
    expect(slots.findById("A1")?.isAvailable).toBe(false);
  });

  it("publishes exit event", () => {
    const t = service.enter(new Vehicle("KA-01-AA-1234", "CAR"));
    clock.advanceByMs(2 * 60 * 60 * 1000);
    service.exit(t.id);
    expect(events.published).toHaveLength(2);
  });
});
```

Mix and match: fakes for state, stub for time, spy for outbound effects. No mocking framework needed.

## When Mocks Earn Their Keep

- Verifying a remote call was made with the right parameters when the response is irrelevant
- Verifying ordering or call count when those are the contract
- Edge cases where building a fake is overkill (third-party SDK with 50 methods, you use 1)

The cost: tests now know about the call structure of the implementation. Refactoring the implementation refactors every test.

## Common Anti-Patterns

### Mock-The-World Tests

Every collaborator mocked, every interaction verified. Tests pass, then real wiring breaks in prod. Prefer fakes + state-based assertions where possible.

### "Mock" That Is Actually a Stub

Calling everything a "mock" loses the distinction. Be precise. It clarifies what the test is actually checking.

### Verifying Implementation Detail

`verify(repo).findById(id)` — does the *behavior* require this exact lookup? If you change the implementation to look up by a different field, do you want every test to fail? If not, assert on outcome, not interaction.

### Static `Mockito.mockStatic` Ad Nauseam

If you need to mock static methods constantly, your design has hidden dependencies. Inject them.

### No Doubles At All — Real DB In Unit Tests

If your unit tests connect to PostgreSQL, they are integration tests. Different scope, slower feedback, often abandoned. Carve out a fake repository tier and unit-test against it.

## In an LLD Interview

You rarely write a full mocking framework call. You will:

- Show that your services accept their collaborators by interface (DIP)
- Sketch one fake (in-memory repository) in your code
- Mention that "in real tests I'd inject a `Clock` stub and a fake repository"
- If asked about a side effect (event, email), mention "I'd use a spy to verify it was published"

Bonus credit: explicitly say "the testability of this design comes from the constructor injection — I could swap any of these for an in-memory fake."

## Verbal Patterns That Score Well

- "I'll use a fake in-memory repository — it doubles as the production stub for now."
- "A stub clock keeps tests deterministic."
- "I'd use a spy on the event publisher rather than a mock — I want to assert on outcome, not call structure."
- "The fact that I can plug in a fake here is the DIP at work."

## Verbal Patterns That Lose Points

- "I'd mock everything." (too coupling-prone)
- "I'd test against the real database." (slow, flaky)
- Calling every double a "mock"
- Skipping testing entirely with "we don't have time"

## Quick Reference

- **Need to fill a parameter?** Dummy
- **Need a canned return value?** Stub
- **Need a working substitute for infrastructure?** Fake
- **Need to assert a side effect happened?** Spy
- **Need to enforce a precise call signature?** Mock

## Related

- [approach-ood-interviews.md](approach-ood-interviews.md)
- [approach-machine-coding-interviews.md](approach-machine-coding-interviews.md)
- [identify-entities-and-model-relationships.md](identify-entities-and-model-relationships.md)
- [write-clean-code-in-interview.md](write-clean-code-in-interview.md)
- [choose-design-patterns.md](choose-design-patterns.md)
- [handle-concurrency-scenarios.md](handle-concurrency-scenarios.md)
- [../solid/](../solid/)
- [../design-patterns/](../design-patterns/)
- [../../system-design/INDEX.md](../../system-design/INDEX.md)

## References

- Meszaros, "xUnit Test Patterns" — the canonical taxonomy
- "Cracking the Coding Interview" — testing chapter
