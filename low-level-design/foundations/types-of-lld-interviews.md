---
title: "Types of LLD Interviews — OOD, Machine Coding, Live Pairing"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, foundations, interview-method, ood, machine-coding]
---

# Types of LLD Interviews — OOD, Machine Coding, Live Pairing

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `foundations` `interview-method` `ood` `machine-coding`
## Summary

LLD interviews come in three main shapes — the OOD whiteboard round (concept-first, ~45–60 min), the machine-coding round (code-first, ~60–120 min), and the live pairing or take-home (collaboration-first, hours to days). Each evaluates a distinct skill: the whiteboard tests *conceptual clarity*, the machine-coding test rewards *working code with tests*, and pairing rewards *communication and incremental delivery*. Knowing which format you're in changes how you should spend the first ten minutes.

## Table of Contents

- [Overview](#overview)
- [Format 1: OOD Whiteboard Interview](#format-1-ood-whiteboard-interview)
- [Format 2: Machine-Coding Interview](#format-2-machine-coding-interview)
- [Format 3: Live Pairing and Take-Home](#format-3-live-pairing-and-take-home)
- [Comparing the Three Formats](#comparing-the-three-formats)
- [Common Question Bank by Format](#common-question-bank-by-format)
- [Cross-Format Skills](#cross-format-skills)
- [How to Tell Which Format You're In](#how-to-tell-which-format-youre-in)
- [Related](#related)

## Overview

All three formats are nominally about "design a system that does X". The difference is what artifact the interviewer wants by the end:

- **OOD whiteboard** — diagrams, class lists, pattern callouts.
- **Machine coding** — running code with passing tests.
- **Live pairing / take-home** — code plus narrated trade-off discussion plus iteration.

A candidate who can do all three well has a meaningfully wider option set in the job market than one who only practices LeetCode or only practices system design.

## Format 1: OOD Whiteboard Interview

### Shape

- Duration: 45–60 minutes.
- Surface: physical whiteboard, virtual whiteboard (Excalidraw, Miro), or shared text doc.
- Output: class diagram + brief sequence narration + identification of patterns.

### What it evaluates

- Can you decompose a fuzzy requirement into entities and responsibilities?
- Do you understand class relationships (composition vs inheritance, aggregation, association)?
- Can you name and apply the right design patterns (Strategy, Factory, Observer, State, Command, Decorator)?
- Can you reason about extensibility — "what if next quarter we need to add X?"
- Can you handle SOLID-style follow-ups without panicking?

### Typical question types

- "Design a parking lot."
- "Design a chess game."
- "Design a vending machine."
- "Design a library management system."
- "Design Splitwise."
- "Design an elevator system."
- "Design a card game (Poker, BlackJack)."
- "Design a restaurant booking system."

These problems are deliberately bounded so a clean class graph can fit on one whiteboard.

### How time is usually spent

| Phase | Time | Activity |
|---|---|---|
| Clarify requirements | 5–8 min | Ask about scope, actors, must-have features |
| Identify entities | 5–10 min | List nouns; promote to classes; mark obvious value objects |
| Define relationships | 10–15 min | Draw the class diagram; pick composition vs inheritance |
| Walk a flow | 5–10 min | Sequence diagram or pseudocode for the main use case |
| Apply patterns | 5–10 min | "Pricing varies by user type → Strategy"; "states transition → State pattern" |
| Extension follow-ups | 5–10 min | "What if we add X?" — show your design accommodates it |

### Common failure modes

- Diving into code before the entity list is stable.
- Inheritance everywhere — modeling every variation as a subclass.
- Inventing classes the requirements didn't ask for.
- Forgetting to call out *why* a pattern was chosen.
- Treating the whiteboard like an HLD round (services, queues, sharding).

## Format 2: Machine-Coding Interview

### Shape

- Duration: 60–120 minutes (Atlassian, Flipkart, Razorpay, Uber, many late-stage startups).
- Surface: candidate's own IDE, screen-shared. Sometimes a CoderPad-style web IDE.
- Output: working code that compiles, runs, and ideally has tests.

### What it evaluates

- Can you actually write code under time pressure?
- Do you naturally apply LLD discipline — interfaces, separation of concerns, dependency injection — when you code, or do you collapse into a single 400-line `Main.java`?
- Are your tests meaningful (not just "happy path returns 200")?
- Is your code readable — naming, formatting, file organization?
- Do you handle edge cases (null inputs, empty collections, concurrency)?

### Typical question types

Often the same problem space as OOD whiteboard, but with a deliverable bar:

- "Build a working Splitwise — must support add expense, settle up, show balances."
- "Build an in-memory cache with LRU eviction and TTL support."
- "Build a snake-and-ladders game playable from the CLI."
- "Build a parking lot system with multi-floor support, ticketing, and pricing."
- "Build a rate limiter (token bucket / sliding window) with unit tests."
- "Build a logger with multiple sinks (console, file) and log levels."
- "Build a job scheduler that runs tasks at given intervals."

### How time is usually spent

| Phase | Time | Activity |
|---|---|---|
| Clarify requirements | 5–10 min | Same as OOD round, but tighter — the deliverable is code |
| Sketch design | 5–10 min | Quick notes — entities, interfaces, file structure |
| Implement core | 30–60 min | Build vertically: pick one user flow, make it work end-to-end |
| Add tests | 10–20 min | At least the main happy path + 2–3 edge cases |
| Demo | 5–10 min | Run it; walk through code; explain trade-offs |

### Recommended structure

```
src/
├── model/          # Entities and value objects
├── service/        # Business logic, depends on interfaces
├── repository/     # Data access (in-memory implementations are fine)
├── strategy/       # Strategy pattern implementations (pricing, eviction, etc.)
└── Main.java       # Wiring + small demo
test/
└── service/        # Unit tests for service layer
```

### Common failure modes

- Spending 40 minutes on design, leaving 20 minutes for code → nothing runs.
- One giant `Main` file with no separation → fails the LLD-discipline check.
- No tests, or tests that only assert `notNull` → fails the engineering-rigor check.
- Skipping clarification → solving the wrong problem perfectly.
- Premature optimization (concurrency, caching) before the basic flow works.

### A small example (pricing strategy)

```java
public interface PricingStrategy {
    Money calculate(Vehicle vehicle, Duration parked);
}

public final class FlatRatePricing implements PricingStrategy {
    private final Money hourlyRate;
    public FlatRatePricing(Money hourlyRate) { this.hourlyRate = hourlyRate; }

    @Override
    public Money calculate(Vehicle vehicle, Duration parked) {
        long hours = Math.max(1, parked.toHours());
        return hourlyRate.multiply(hours);
    }
}

public final class TieredPricing implements PricingStrategy {
    // first hour: $5, each additional: $3
    @Override
    public Money calculate(Vehicle vehicle, Duration parked) {
        long hours = Math.max(1, parked.toHours());
        Money total = Money.usd(5);
        if (hours > 1) total = total.add(Money.usd(3).multiply(hours - 1));
        return total;
    }
}
```

The interviewer wants to see this *during* the interview, not on a whiteboard.

## Format 3: Live Pairing and Take-Home

### Shape

- **Live pairing**: 60–120 min, candidate and engineer share an editor, code together. Common at engineering-culture-first companies (Stripe, Shopify, smaller series-B/C startups).
- **Take-home**: candidate gets a problem (often 2–8 hours of expected work) and submits a repo. Often followed by an in-person review session.

### What it evaluates

- Communication while coding ("I'm thinking about it this way because...").
- Reaction to feedback ("Good point, let me refactor.").
- Honest scope management ("This won't fit in our time box, here's what I'd skip.").
- Long-form judgment: file structure, naming, README quality, test coverage, commit hygiene (for take-home).
- Handling unfamiliar APIs or libraries gracefully.

### Typical question types

**Live pairing**:
- Extending an existing small codebase with a new feature.
- Debugging a failing test suite.
- Implementing a small feature in a stack the candidate already knows.

**Take-home**:
- "Build a CLI that parses our log format and reports stats."
- "Build a small REST API for a TODO list with persistence."
- "Implement a grep-like tool with these flags."
- "Build a tic-tac-toe game with a pluggable AI."

### How time is usually spent (live pairing)

| Phase | Time | Activity |
|---|---|---|
| Read existing code | 5–10 min | Get oriented; ask questions; don't pretend to understand |
| Plan together | 5 min | "Here's how I'd approach it; sound right?" |
| Implement incrementally | 30–60 min | Make small changes, run the suite often |
| Refactor / polish | 10–15 min | Improve names, extract methods, add a missing test |
| Discuss trade-offs | 5–10 min | "If I had more time I'd..." |

### Take-home anti-patterns

- Over-engineering the solution because "I have time".
- Leaving no README — no setup instructions, no design rationale.
- Skipping tests because "it's just a take-home".
- Padding with generated boilerplate (extensive `.gitignore`s, dotfiles) that obscures real work.
- Submitting code that doesn't run on a clean machine.

### Take-home good practices

- Short README: how to run, how to test, design choices, what was skipped and why.
- Clear commit history (small, intent-revealing commits beat one giant "submission" commit).
- Working tests that actually run with a single command.
- Honest trade-off notes ("I chose in-memory storage for brevity; a real version would use SQLite").

## Comparing the Three Formats

| Dimension | OOD Whiteboard | Machine Coding | Live Pairing / Take-Home |
|---|---|---|---|
| Duration | 45–60 min | 60–120 min | 60–120 min live; hours to days take-home |
| Output | Diagram + narration | Working code + tests | Working code + discussion |
| Primary skill | Conceptual decomposition | Engineering rigor under time pressure | Collaboration + judgment |
| Code required? | Pseudocode at most | Yes, must run | Yes, must run |
| Tests required? | Sometimes mentioned | Strongly expected | Strongly expected |
| Where it dominates | FAANG-style product loops, India tech ecosystem | Atlassian, Flipkart, Razorpay, Uber, Indian unicorns | Stripe, Shopify, smaller startups, dev-tools companies |
| Practicable on LeetCode? | Partially | Partially | Not really |

## Common Question Bank by Format

### OOD whiteboard staples

- Parking Lot, Elevator, Vending Machine
- Splitwise, Snake & Ladders, Chess, Tic-Tac-Toe
- Library Management, Hotel Booking, Restaurant Booking
- ATM, Stock Exchange order book
- Logger, Cache (LRU), Rate Limiter

### Machine-coding staples

- All of the above, but as runnable code with tests
- In-memory KV store with TTL
- Job scheduler / cron-like system
- BookMyShow / movie booking
- File system (in-memory)
- Concurrent task runner

### Live pairing / take-home staples

- Extend or fix an existing small codebase
- Build a small CLI tool (grep-like, log-parser, file-watcher)
- Build a tiny REST API
- Implement an algorithm with good code quality (not just correctness)

## Cross-Format Skills

Skills that pay off in *all three* formats:

- **Clarifying requirements** — the first 5 minutes is universally about scoping.
- **Naming things well** — `PaymentProcessor` beats `Helper`, `Manager`, `Util` everywhere.
- **Thinking in interfaces** — separating "what" from "how" makes both diagrams and code cleaner.
- **Using design patterns by name** — naming a pattern signals you know the literature.
- **Writing focused tests** — even on a whiteboard, naming a test ("should reject a coin that doesn't match the price") is a signal of rigor.
- **Acknowledging trade-offs** — every reasonable choice has a downside; saying so out loud builds trust.

## How to Tell Which Format You're In

Recruiter emails often blur formats. Use these signals:

- **"Whiteboard / virtual whiteboard / OOD round / design round"** → OOD whiteboard.
- **"Machine coding round / coding round (90 min) / build a working X"** → Machine coding.
- **"Pair programming / collaborative coding / IDE round"** → Live pairing.
- **"Take-home / project / submit a PR / 4-hour exercise"** → Take-home.
- **"Architecture round / system design"** without "OOD" → likely HLD, not LLD.

When unsure, ask the recruiter explicitly: "Will I be expected to produce running code, or is a class diagram the deliverable?" The answer determines whether you spend the morning in an IDE or on a whiteboard tool.

## Related

- [What Is Low-Level Design?](./what-is-lld.md)
- [LLD vs HLD — Where the Boundary Sits](./lld-vs-hld.md)
- [System Design Index](../../system-design/INDEX.md)
- [Java Index](../../java/INDEX.md) — most LLD interviews are conducted in Java or a Java-shaped language
