---
title: "What Is Low-Level Design?"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, foundations, ood, sdlc, interviews]
---

# What Is Low-Level Design?

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `foundations` `ood` `sdlc` `interviews`

## Summary

Low-Level Design (LLD) is the practice of describing software at the granularity of classes, interfaces, methods, and the interactions between them, so that an engineer can implement the system without having to re-decide its internal shape. It sits one zoom level below system architecture and one zoom level above source code, and its primary deliverables are class diagrams, sequence diagrams, interface contracts, and code skeletons that compile.

## Table of Contents

- [Overview](#overview)
- [Class-Level OOD vs System-Level Architecture](#class-level-ood-vs-system-level-architecture)
- [Deliverables of LLD](#deliverables-of-lld)
- [Audience](#audience)
- [When LLD Matters](#when-lld-matters)
- [Where LLD Sits in the SDLC](#where-lld-sits-in-the-sdlc)
- [Common Misconceptions](#common-misconceptions)
- [Quality Bar for an LLD Document](#quality-bar-for-an-lld-document)
- [Related](#related)

## Overview

LLD answers the question: **"What classes, interfaces, and interactions implement this feature, and how do they collaborate?"**

It is concerned with:

- The object model (entities, value objects, services).
- Responsibilities and collaborations between classes.
- Interfaces and contracts (method signatures, pre/post conditions, exceptions).
- Concurrency and lifecycle (who creates what, who owns what, who locks what).
- Extension points (where the next requirement plugs in without rewriting).

It is *not* concerned with which servers run the code, how data is sharded across nodes, or how the network topology routes a request. Those are the job of High-Level Design (HLD).

## Class-Level OOD vs System-Level Architecture

Two activities people often blur into one:

**Object-Oriented Design (class-level)**
- Granularity: a single bounded context — e.g. "the parking lot module", "the order placement service".
- Output: classes, interfaces, relationships (composition, aggregation, inheritance), patterns applied (Strategy, Factory, Observer, etc.).
- Validated by: can a competent engineer implement this and have it compile?

**System Architecture (system-level)**
- Granularity: many services, many machines, many teams.
- Output: services, queues, datastores, gateways, load balancers, regions, failure domains.
- Validated by: does it meet capacity, latency, availability, and cost targets?

LLD lives in the first column. HLD lives in the second. A real design review usually moves between both, but the *artifacts* are different and the *failure modes* are different.

## Deliverables of LLD

A complete LLD package typically contains the following, in roughly this order of importance:

### 1. Class Diagram

A UML class diagram (or equivalent) showing:
- Each class with its key fields and methods.
- Relationships: inheritance (`extends`), implementation (`implements`), composition (filled diamond), aggregation (open diamond), association (line + arrow).
- Multiplicity on relationships (1, 0..1, 1..*, *).

### 2. Sequence Diagram

For each important flow ("place order", "park vehicle", "checkout cart"), a sequence diagram showing:
- Actor → entry point → collaborators → return path.
- Synchronous vs asynchronous calls.
- Where errors are thrown and where they are caught.

### 3. Interface Contracts

Method signatures, often expressed in code:

```java
public interface PaymentGateway {
    PaymentResult charge(Money amount, PaymentMethod method) throws PaymentException;
    RefundResult refund(TransactionId txn, Money amount) throws RefundException;
}
```

Contract documentation should cover:
- Inputs and validation expectations.
- Return shape and possible failure modes.
- Idempotency guarantees (if any).
- Thread-safety guarantees (if any).

### 4. Code Skeleton

Compileable shells of the major classes — constructors, fields, method signatures, key delegations — without business logic. The point is to prove the design *fits together* before anyone writes algorithms.

### 5. Concurrency & Lifecycle Notes

- Which objects are stateful vs stateless.
- Which objects are shared across threads.
- Lock ordering, if locks exist.
- Object lifetimes (request-scoped, session-scoped, singleton).

### 6. Trade-off Log (Optional but Valuable)

A short list of "we considered X, picked Y because Z". This is the document future-you will thank present-you for.

## Audience

LLD is written **for engineers who will implement the design**, not for executives, product managers, or sales.

The implications:

- It can assume programming literacy (interfaces, inheritance, exceptions are not explained).
- It must use precise vocabulary (don't say "manages users" — say which class owns the `User` entity, which class owns authentication, which class owns sessions).
- It can include code. In fact, it usually should.
- It does not need to justify *why the product exists* — that's the PRD's job.

Secondary audiences include:

- **Reviewers** — senior engineers approving the design before implementation.
- **Future maintainers** — engineers six months later trying to understand why the code looks the way it does.
- **Interviewers** — in an interview setting, the LLD *is* the deliverable.

## When LLD Matters

LLD is a tool, not a ritual. It pays off most clearly in these situations:

### Interviews

- OOD whiteboard rounds (45–60 min) at most product companies.
- Machine-coding rounds (60–120 min) at companies like Flipkart, Atlassian, Uber, and many late-stage startups.
- "Design X" follow-ups in standard system design interviews where the interviewer drills into one component.

### Design Reviews

When a feature involves more than one developer, or touches a module owned by another team, an LLD doc lets the team converge on a shape before anyone wastes a week implementing the wrong abstraction.

### Large Refactors

Replacing a 5-year-old subsystem is a high-stakes activity. LLD lets you propose the new shape, compare it to the old shape, and get sign-off before you start tearing things out.

### New Modules in Established Codebases

When you're adding a new bounded context (e.g. introducing a "loyalty" module to an e-commerce app), LLD helps you align the new module's conventions with the rest of the system.

### Skip LLD When

- Prototype / spike code with a known short lifespan.
- One-engineer features that fit in a single file.
- Bug fixes and minor enhancements to existing classes.

## Where LLD Sits in the SDLC

A simplified pipeline:

```
Product Requirements (PRD)
        │
        ▼
High-Level Design (HLD) — services, datastores, capacity
        │
        ▼
Low-Level Design (LLD) — classes, interfaces, sequences
        │
        ▼
Implementation (code + unit tests)
        │
        ▼
Code Review → Integration → Deployment
```

LLD is the bridge between "we agree on the system" and "we agree on the code". Without it, every implementer makes design decisions on the fly, and the resulting codebase reflects whoever happened to write each file first.

In agile practice, LLD often lives inside a design doc attached to an epic, or inline in a Jira/Linear ticket, rather than as a standalone Word document. The *form* matters less than the *artifacts* (class diagram, sequence diagram, interface contracts, skeleton).

## Common Misconceptions

- **"LLD is just UML."** UML is one notation. The artifacts (relationships, contracts, sequences) matter; the specific notation does not.
- **"LLD is the same as the code."** Code is the implementation. LLD is the shape the code is supposed to take. Good LLD often discards detail that good code must include (logging, retries, telemetry).
- **"LLD only matters for big systems."** It also matters for small modules with non-trivial collaboration — e.g. a notification subsystem with multiple channels and routing rules.
- **"LLD is a one-shot artifact."** Real LLDs evolve as the implementation reveals constraints. The doc should be updated, or explicitly marked stale.

## Quality Bar for an LLD Document

A reviewable LLD should let a reader answer:

- [ ] What are the entry points (public methods, API handlers, message consumers)?
- [ ] What classes exist, and what does each one own?
- [ ] How do the classes relate (composition, inheritance, association)?
- [ ] What are the sequence flows for the 3–5 most important operations?
- [ ] What are the failure modes, and who handles them?
- [ ] Where are the extension points for future requirements?
- [ ] What are the concurrency and lifetime assumptions?

If the reader can answer those without asking the author, the LLD is doing its job.

## Related

- [LLD vs HLD — Where the Boundary Sits](./lld-vs-hld.md)
- [Types of LLD Interviews — OOD, Machine Coding, Live Pairing](./types-of-lld-interviews.md)
- [System Design Index](../../system-design/INDEX.md)
- [Java Index](../../java/INDEX.md) — class structure, interfaces, exceptions
