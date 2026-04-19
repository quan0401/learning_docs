# Documentation Index — TypeScript Deep Learning Path

A progressive path from TypeScript compiler internals through Node.js runtime to production architecture.
Starting at an intermediate level — you already write TS backend daily. This path takes you from "I use TS" to "I truly understand TS."

Cross-references to the [Java learning path](../java/INDEX.md) where concepts parallel.

---

## Tier 1 — TypeScript Compiler & Configuration

Understanding what happens between your code and runtime. No more copy-pasting `tsconfig.json`.

1. [tsconfig Demystified](tooling/tsconfig-demystified.md) — `strict` family, `module`/`moduleResolution`, `target`, project references, path aliases _(2026-04-19)_
2. [Module Resolution Deep Dive](tooling/module-resolution.md) — Node's import resolution, `.js` extensions in ESM, `exports` field, dual CJS/ESM publishing _(2026-04-19)_
3. [Declaration Files & Type Acquisition](tooling/declaration-files.md) — `.d.ts` files, `@types/*`, declaration merging, module augmentation, `declare global` _(2026-04-19)_

---

## Tier 2 — Type System Mastery _(planned)_

The type-level programming that separates "using TS" from "understanding TS." Problem-driven — each doc opens with a real scenario.

4. Structural Typing & Type Compatibility _(planned)_
5. Generics & Constraints in Practice _(planned)_
6. Conditional Types & `infer` _(planned)_
7. Mapped Types & Key Remapping _(planned)_
8. Type Narrowing & Control Flow _(planned)_
9. Recursive & Advanced Types _(planned)_

---

## Tier 3 — V8 & Node Runtime Internals

What actually happens when Node runs your code. The same depth your Java path gives to JVM internals.

10. [V8 Engine Pipeline](runtime/v8-engine-pipeline.md) — parsing → Ignition → TurboFan → deopt, hidden classes, inline caches _(2026-04-19)_
11. [Event Loop Internals](runtime/event-loop-internals.md) — the 6 libuv phases, microtask queue, `nextTick` vs `queueMicrotask`, starvation _(2026-04-19)_
12. [V8 Memory & Garbage Collection](runtime/v8-memory-and-gc.md) — new space/old space, Scavenger vs Mark-Sweep-Compact, `--max-old-space-size`, memory leaks _(2026-04-19)_
13. [Worker Threads & Concurrency](runtime/worker-threads.md) — `worker_threads`, `SharedArrayBuffer`, `Atomics`, cluster vs child_process _(2026-04-19)_

---

## Tier 4 — Testing with Vitest _(planned)_

Building a testing practice from zero. Vitest as the modern TS-first test runner.

14. Vitest Fundamentals _(planned)_
15. Mocking & Test Doubles in Vitest _(planned)_
16. Integration & API Testing _(planned)_

---

## Tier 5 — Architecture Patterns in TypeScript

Framework-agnostic patterns implemented in raw TS. DDD, hexagonal, CQRS — the patterns first, frameworks second.

17. [DDD Tactical Patterns in TypeScript](patterns/ddd-tactical-patterns.md) — entities, value objects (branded types), aggregates, domain events, repository interface _(2026-04-19)_
18. [Hexagonal Architecture / Ports & Adapters](patterns/hexagonal-architecture.md) — core domain, port interfaces, adapter implementations, dependency rule _(2026-04-19)_
19. [CQRS & Event Patterns](patterns/cqrs-and-events.md) — command/query separation, event bus, eventual consistency in Node _(2026-04-19)_
20. [Error Handling Architecture](patterns/error-handling-architecture.md) — Result type, domain vs infrastructure errors, error mapping across layers _(2026-04-19)_

---

## Tier 6 — Frameworks

NestJS through the lens of a Spring Boot developer. Plus the modern Express alternatives.

21. [NestJS for Spring Boot Developers](frameworks/nestjs-for-spring-devs.md) — modules ↔ `@Configuration`, providers ↔ `@Bean`, guards ↔ security filters, pipes ↔ `@Valid` _(2026-04-19)_
22. [NestJS Advanced Patterns](frameworks/nestjs-advanced-patterns.md) — custom decorators, dynamic modules, request-scoped providers, microservices, CQRS module _(2026-04-19)_
23. [Express vs Fastify vs Hono vs Elysia](frameworks/express-vs-alternatives.md) — performance, middleware models, schema validation, type safety, migration paths _(2026-04-19)_

---

## Tier 7 — Data Access

Prisma mastery and the rising alternative.

24. [Prisma Deep Dive](data-access/prisma-deep-dive.md) — schema patterns, transactions, raw queries, connection pooling, query optimization, extension API _(2026-04-19)_
25. [Prisma vs Drizzle](data-access/prisma-vs-drizzle.md) — schema-first vs code-first, SQL proximity, migrations, performance, type safety comparison _(2026-04-19)_

---

## Tier 8 — Production Node.js

Running Node in production at scale. Full standalone reference, even where concepts overlap with the Java production tier.

26. [Node.js Profiling & Debugging](production/profiling-and-debugging.md) — CPU profiling, heap snapshots, event loop lag, memory leak hunting, flame graphs _(2026-04-19)_
27. [Node.js in Kubernetes](production/nodejs-in-kubernetes.md) — graceful shutdown, health checks, resource limits, horizontal scaling, 12-factor _(2026-04-19)_
28. [Observability for Node Services](production/observability.md) — OpenTelemetry, distributed tracing, structured logging (pino), metrics, correlation IDs _(2026-04-19)_

---

## Quick Reference by Topic

### Compiler & Tooling

- [tsconfig Demystified](tooling/tsconfig-demystified.md)
- [Module Resolution Deep Dive](tooling/module-resolution.md)
- [Declaration Files & Type Acquisition](tooling/declaration-files.md)

### Runtime Internals

- [V8 Engine Pipeline](runtime/v8-engine-pipeline.md)
- [Event Loop Internals](runtime/event-loop-internals.md)
- [V8 Memory & Garbage Collection](runtime/v8-memory-and-gc.md)
- [Worker Threads & Concurrency](runtime/worker-threads.md)

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
