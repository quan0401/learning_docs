# Documentation Index — TypeScript Deep Learning Path

A progressive path from TypeScript compiler internals through Node.js runtime to production architecture.
Starting at an intermediate level — you already write TS backend daily. This path takes you from "I use TS" to "I truly understand TS."

Cross-references to the [Java learning path](../java/INDEX.md), the [Go learning path](../golang/INDEX.md) for another statically-typed backend language with a different concurrency model, the [Networking learning path](../networking/INDEX.md), the [Kubernetes learning path](../kubernetes/INDEX.md), the [Low-Level Design learning path](../low-level-design/INDEX.md) for class-level OOD and design patterns, the [System Design learning path](../system-design/INDEX.md) where concepts parallel, the [Operating Systems learning path](../operating-systems/INDEX.md) for the Linux primitives Node sits on (epoll, page cache, fds), the [Observability learning path](../observability/INDEX.md) for metrics/logs/traces and SLO design, the [Performance Engineering learning path](../performance/INDEX.md) for latency/throughput/capacity, the [Security learning path](../security/INDEX.md) for OWASP, OAuth/OIDC, JWT, TLS, and secrets, the [Search learning path](../search/INDEX.md), the [Linux learning path](../linux/INDEX.md), the [Git Internals learning path](../git-internals/INDEX.md), and the [Web Scraping learning path](../web-scraping/INDEX.md) (Crawlee/Playwright in Node, source-map archaeology against minified TS bundles).

**Markers:** **★** = core must-learn (everyday TS/Node backend work, common in interviews and production debugging). **○** = supporting deep-dive (advanced type-level programming or specialized topics). Internalize all ★ before going deep on ○.

---

## Tier 1 — TypeScript Compiler & Configuration

Understanding what happens between your code and runtime. No more copy-pasting `tsconfig.json`.

1. [★ tsconfig Demystified](tooling/tsconfig-demystified.md) — `strict` family, `module`/`moduleResolution`, `target`, project references, path aliases _(2026-04-19)_
2. [★ Module Resolution Deep Dive](tooling/module-resolution.md) — Node's import resolution, `.js` extensions in ESM, `exports` field, dual CJS/ESM publishing _(2026-04-19)_
3. [○ Declaration Files & Type Acquisition](tooling/declaration-files.md) — `.d.ts` files, `@types/*`, declaration merging, module augmentation, `declare global` _(2026-04-19)_

---

## Tier 2 — Type System Mastery

The type-level programming that separates "using TS" from "understanding TS." Problem-driven — each doc opens with a real scenario.

4. [★ Structural Typing & Type Compatibility](type-system/structural-typing.md) — structural vs nominal, excess property checks, freshness, variance, branded types, `satisfies` _(2026-04-23)_
5. [★ Generics & Constraints in Practice](type-system/generics-and-constraints.md) — generic functions, constraints with `extends`, inference, defaults, overloads vs generics, utility types, advanced patterns, Java comparison _(2026-04-23)_
6. [★ Conditional Types & `infer`](type-system/conditional-types-and-infer.md) — distributive conditionals, `infer` for extraction, `never` as empty union, practical utility types _(2026-04-23)_
7. [★ Mapped Types & Key Remapping](type-system/mapped-types.md) — mapped types, key remapping with `as`, template literal types, filtering by value type, built-in utility type internals _(2026-04-23)_
8. [★ Type Narrowing & Control Flow](type-system/type-narrowing.md) — `typeof`/`instanceof` guards, discriminated unions, exhaustive `never` checks, type predicates, assertion functions, `satisfies` _(2026-04-23)_
9. [○ Recursive & Advanced Types](type-system/recursive-and-advanced-types.md) — recursive types, variadic tuples, `const` assertions, branded/opaque types deep dive, template literal parsing, type-level programming _(2026-04-23)_

---

## Tier 3 — V8 & Node Runtime Internals

What actually happens when Node runs your code. The same depth your Java path gives to JVM internals.

10. [★ V8 Engine Pipeline](runtime/v8-engine-pipeline.md) — parsing → Ignition → TurboFan → deopt, hidden classes, inline caches _(2026-04-19)_
11. [★ Event Loop Internals](runtime/event-loop-internals.md) — the 6 libuv phases, microtask queue, `nextTick` vs `queueMicrotask`, starvation _(2026-04-19)_
12. [★ V8 Memory & Garbage Collection](runtime/v8-memory-and-gc.md) — new space/old space, Scavenger vs Mark-Sweep-Compact, `--max-old-space-size`, memory leaks _(2026-04-19)_
13. [○ Worker Threads & Concurrency](runtime/worker-threads.md) — `worker_threads`, `SharedArrayBuffer`, `Atomics`, cluster vs child_process _(2026-04-19)_

---

## Tier 4 — Testing with Vitest

Building a testing practice from zero. Vitest as the modern TS-first test runner.

14. [★ Vitest Fundamentals](testing/vitest-fundamentals.md) — Vitest setup, test structure, matchers, lifecycle, coverage, JUnit 5 mapping _(2026-04-23)_
15. [★ Mocking & Test Doubles in Vitest](testing/mocking-and-test-doubles.md) — test doubles taxonomy, vi.fn/vi.spyOn/vi.mock, Prisma mocking, external API mocking, DI for testability, Mockito parallel _(2026-04-23)_
16. [★ Integration & API Testing](testing/integration-and-api-testing.md) — Supertest, database testing (Testcontainers/SQLite), fixtures, middleware testing, CI/CD, Spring Boot parallel _(2026-04-23)_

---

## Tier 5 — Architecture Patterns in TypeScript

Framework-agnostic patterns implemented in raw TS. DDD, hexagonal, CQRS — the patterns first, frameworks second.

17. [★ DDD Tactical Patterns in TypeScript](patterns/ddd-tactical-patterns.md) — entities, value objects (branded types), aggregates, domain events, repository interface _(2026-04-19)_
18. [★ Hexagonal Architecture / Ports & Adapters](patterns/hexagonal-architecture.md) — core domain, port interfaces, adapter implementations, dependency rule _(2026-04-19)_
19. [○ CQRS & Event Patterns](patterns/cqrs-and-events.md) — command/query separation, event bus, eventual consistency in Node _(2026-04-19)_
20. [★ Error Handling Architecture](patterns/error-handling-architecture.md) — Result type, domain vs infrastructure errors, error mapping across layers _(2026-04-19)_

---

## Tier 6 — Frameworks

NestJS through the lens of a Spring Boot developer. Plus the modern Express alternatives.

21. [★ NestJS for Spring Boot Developers](frameworks/nestjs-for-spring-devs.md) — modules ↔ `@Configuration`, providers ↔ `@Bean`, guards ↔ security filters, pipes ↔ `@Valid` _(2026-04-19)_
22. [○ NestJS Advanced Patterns](frameworks/nestjs-advanced-patterns.md) — custom decorators, dynamic modules, request-scoped providers, microservices, CQRS module _(2026-04-19)_
23. [★ Express vs Fastify vs Hono vs Elysia](frameworks/express-vs-alternatives.md) — performance, middleware models, schema validation, type safety, migration paths _(2026-04-19)_

---

## Tier 7 — Data Access

Prisma mastery and the rising alternative.

24. [★ Prisma Deep Dive](data-access/prisma-deep-dive.md) — schema patterns, transactions, raw queries, connection pooling, query optimization, extension API _(2026-04-19)_
25. [○ Prisma vs Drizzle](data-access/prisma-vs-drizzle.md) — schema-first vs code-first, SQL proximity, migrations, performance, type safety comparison _(2026-04-19)_

---

## Tier 8 — Production Node.js

Running Node in production at scale. Full standalone reference, even where concepts overlap with the Java production tier.

26. [★ Node.js Profiling & Debugging](production/profiling-and-debugging.md) — CPU profiling, heap snapshots, event loop lag, memory leak hunting, flame graphs _(2026-04-19)_
27. [★ Node.js in Kubernetes](production/nodejs-in-kubernetes.md) — graceful shutdown, health checks, resource limits, horizontal scaling, 12-factor _(2026-04-19)_
28. [★ Observability for Node Services](production/observability.md) — OpenTelemetry, distributed tracing, structured logging (pino), metrics, correlation IDs _(2026-04-19)_

---

## Quick Reference by Topic

### Compiler & Tooling

- [tsconfig Demystified](tooling/tsconfig-demystified.md)
- [Module Resolution Deep Dive](tooling/module-resolution.md)
- [Declaration Files & Type Acquisition](tooling/declaration-files.md)

### Type System

- [Structural Typing & Type Compatibility](type-system/structural-typing.md)
- [Generics & Constraints in Practice](type-system/generics-and-constraints.md)
- [Conditional Types & `infer`](type-system/conditional-types-and-infer.md)
- [Mapped Types & Key Remapping](type-system/mapped-types.md)
- [Type Narrowing & Control Flow](type-system/type-narrowing.md)
- [Recursive & Advanced Types](type-system/recursive-and-advanced-types.md)

### Runtime Internals

- [V8 Engine Pipeline](runtime/v8-engine-pipeline.md)
- [Event Loop Internals](runtime/event-loop-internals.md)
- [V8 Memory & Garbage Collection](runtime/v8-memory-and-gc.md)
- [Worker Threads & Concurrency](runtime/worker-threads.md)

### Testing

- [Vitest Fundamentals](testing/vitest-fundamentals.md)
- [Mocking & Test Doubles in Vitest](testing/mocking-and-test-doubles.md)
- [Integration & API Testing](testing/integration-and-api-testing.md)

### Architecture

- [DDD Tactical Patterns in TypeScript](patterns/ddd-tactical-patterns.md)
- [Hexagonal Architecture / Ports & Adapters](patterns/hexagonal-architecture.md)
- [CQRS & Event Patterns](patterns/cqrs-and-events.md)
- [Error Handling Architecture](patterns/error-handling-architecture.md)

### Frameworks

- [NestJS for Spring Boot Developers](frameworks/nestjs-for-spring-devs.md)
- [NestJS Advanced Patterns](frameworks/nestjs-advanced-patterns.md)
- [Express vs Fastify vs Hono vs Elysia](frameworks/express-vs-alternatives.md)

### Data Access

- [Prisma Deep Dive](data-access/prisma-deep-dive.md)
- [Prisma vs Drizzle](data-access/prisma-vs-drizzle.md)

### Production

- [Node.js Profiling & Debugging](production/profiling-and-debugging.md)
- [Node.js in Kubernetes](production/nodejs-in-kubernetes.md)
- [Observability for Node Services](production/observability.md)

---

## Bug Spotting

Active-recall practice docs. Each presents 22+ broken snippets organized by difficulty (Easy / Subtle / Senior trap), with one-line `<details>` hints inline and full root-cause + fix in a Solutions section. Every bug cites a real reference (RFC, CVE, official-doc gotcha, postmortem, library issue). Use these to pressure-test concept knowledge after working through the tiers above.

- [Node.js Event Loop — Bug Spotting](runtime/event-loop-bug-spotting.md) ★ — _(2026-05-03)_
- [TypeScript Type System — Bug Spotting](type-system/type-system-bug-spotting.md) ★ — _(2026-05-03)_
