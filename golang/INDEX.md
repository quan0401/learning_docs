# Documentation Index — Go Learning Path

A progressive path from Go language foundations through concurrency, runtime internals, testing, idiomatic architecture, frameworks, data access, and production. Backend-services oriented. Starting from zero — written for someone who already builds backends in TypeScript/Node and is partway through Java/Spring Boot, so every Tier 1 doc opens with a contrast to those mental models rather than a syntax tutorial.

Cross-references to the [TypeScript learning path](../typescript/INDEX.md) (the closest peer language with a different concurrency model and zero-cost abstractions), the [Java learning path](../java/INDEX.md) (interfaces vs implicit satisfaction, exceptions vs errors-as-values, JVM vs Go runtime), the [Database learning path](../database/INDEX.md) (Go's `database/sql` and connection pooling), the [Networking learning path](../networking/INDEX.md) (Go's `net/http` and `net` package as a hands-on lens), the [Low-Level Design learning path](../low-level-design/INDEX.md) (composition over inheritance), the [System Design learning path](../system-design/INDEX.md) (Go services in HLD case studies), the [Kubernetes learning path](../kubernetes/INDEX.md) (Go is the language of the cloud-native ecosystem), the [Operating Systems learning path](../operating-systems/INDEX.md) (G-M-P scheduler over OS threads), the [Observability learning path](../observability/INDEX.md) (`runtime/pprof`, `runtime/trace`, `slog`), the [Performance Engineering learning path](../performance/INDEX.md) (pprof and the GC pacer), the [Security learning path](../security/INDEX.md) (`crypto/tls`, `crypto/subtle`), and the [Web Scraping learning path](../web-scraping/INDEX.md) (Colly, JA3 fingerprinting via `crypto/tls`, Go's standout role in recon tooling — subfinder, amass, httpx, nuclei).

**Markers:** **★** = core must-learn (everyday Go backend work, common in interviews and production debugging). **○** = supporting deep-dive (advanced runtime topics or specialized areas). Internalize all ★ before going deep on ○.

---

## Tier 1 — Language Foundations

The "make it click" set for someone arriving with a TS+Java background. Concept-driven, not syntax-driven — every doc opens with the model that makes Go feel different from what you already know.

1. [★ Go Tour for TS + Java Developers](fundamentals/01-go-tour-for-ts-java-devs.md) — packages & imports, `var` / `:=`, basic types, the single `for` loop, defer, multiple returns, `gofmt` is law, side-by-side TS/Java/Go _(2026-05-03)_
2. [★ Module System & Project Layout](fundamentals/02-module-system-and-layout.md) — `go.mod` / `go.sum`, semantic import versioning, `internal/`, `cmd/` vs `pkg/`, vendoring, replace directives, contrast with `package.json` and Maven _(2026-05-03)_
3. [★ Types, Zero Values & Composite Literals](fundamentals/03-types-zero-values-composites.md) — value vs reference types, the zero-value rule, struct literals, embedded fields, type definitions vs aliases, `iota` enum idiom _(2026-05-03)_
4. [★ Slices, Arrays & Maps](fundamentals/04-slices-arrays-maps.md) — underlying-array model, `len` vs `cap`, append's realloc semantics, the slice aliasing bug, nil-map writes panic, when to preallocate _(2026-05-03)_
5. [★ Interfaces & Structural Typing](fundamentals/05-interfaces-and-structural-typing.md) — implicit satisfaction, `any`, type assertions vs type switches, accept-interfaces-return-structs, typed-nil-interface pitfall _(2026-05-03)_
6. [★ Errors as Values](fundamentals/06-errors-as-values.md) — `error` is just an interface, sentinel vs typed errors, `errors.Is` / `errors.As`, `%w` wrapping, when to use `panic`/`recover` _(2026-05-03)_
7. [★ Pointers, Methods & Receivers](fundamentals/07-pointers-methods-receivers.md) — when to take `&`, value vs pointer receivers, method sets, escape analysis primer, nil pointer panics _(2026-05-03)_

---

## Tier 2 — Concurrency

The killer feature. Goroutines and the G-M-P scheduler, channels in practice, the `sync` package and the Go memory model, and `context.Context` propagation.

8. [★ Goroutines & the Scheduler (G-M-P)](concurrency/01-goroutines-and-scheduler.md) — goroutines vs OS threads vs Java virtual threads, G-M-P model, run queues & work stealing, netpoller, syscall handoff, asynchronous preemption (Go 1.14+), `sysmon`, `runtime.LockOSThread` _(2026-05-03)_
9. [★ Channels in Practice](concurrency/02-channels-in-practice.md) — typed conduits, unbuffered vs buffered, the four channel axioms, `select` non-determinism, pipeline / fan-in / fan-out / worker pool / done-channel patterns, goroutine leaks, when to use a mutex instead _(2026-05-03)_
10. [★ The `sync` Package & the Go Memory Model](concurrency/03-sync-package-and-memory-model.md) — happens-before, `sync.Mutex` / `RWMutex` / `WaitGroup` / `Once` / `Pool`, `sync/atomic` typed wrappers (Go 1.19+), the `-race` detector, contrast with the Java memory model _(2026-05-03)_
11. [★ Context Propagation](concurrency/04-context-propagation.md) — `context.Context`, `Background` vs `TODO`, `WithCancel` / `WithTimeout` / `WithDeadline` / `WithValue`, ctx-first convention, `AfterFunc` / `WithoutCancel` (Go 1.21+), anti-patterns, contrast with `AbortController` and `InterruptedException` _(2026-05-03)_

---

## Tier 3 — Runtime Internals

What actually happens under your goroutines. The concurrent tri-color mark-sweep garbage collector and `GOGC` pacing, escape analysis and stack vs heap allocation, generics, and reflection / `unsafe`.

12. [★ Go's Garbage Collector](runtime/01-garbage-collector.md) — concurrent tri-color mark-sweep design, why no generations and no compaction, the four-phase cycle, `GOGC` and `GOMEMLIMIT` (Go 1.19+), STW pause times, observability via `gctrace` / `runtime/metrics`, contrast with JVM G1/ZGC and V8 _(2026-05-03)_
13. [★ Escape Analysis & Allocation](runtime/02-escape-analysis-and-allocation.md) — stack vs heap, the basic escape rules, reading `-gcflags='-m'` output, common allocation sources, `sync.Pool` reuse, profiling with `pprof -alloc_space` vs `-inuse_space` _(2026-05-03)_
14. [★ Generics in Go](runtime/03-generics.md) — type parameters and constraints, `comparable` and type sets, the `slices`/`maps`/`cmp` packages (Go 1.21+), GC shape stenciling, what generics cannot do, contrast with Java type erasure and TS generics _(2026-05-03)_
15. [○ Reflection & `unsafe`](runtime/04-reflection-and-unsafe.md) — the `reflect` package and the three Laws of Reflection, real-world reflection users, the four `unsafe.Pointer` conversion rules, `unsafe.Slice` / `unsafe.String` (Go 1.17+ / 1.20+), why `unsafe` is excluded from the Go 1 compatibility promise _(2026-05-03)_

---

## Tier 4 — Testing

Building a testing practice in idiomatic Go. The `testing` package fundamentals, mocking & test doubles, and integration & contract testing.

16. [★ The `testing` Package — Fundamentals](testing/01-testing-package-fundamentals.md) — `*_test.go` discovery, table-driven tests, `t.Run` / `t.Parallel` / `t.Helper` / `t.Cleanup` / `t.TempDir`, benchmarks, examples, fuzzing (Go 1.18+), coverage, contrast with Vitest and JUnit 5 _(2026-05-03)_
17. [★ Mocking & Test Doubles in Go](testing/02-mocking-and-test-doubles.md) — interfaces enable testing, hand-rolled fakes, `httptest`, `uber-go/mock` (the maintained gomock fork), `testify/mock`, `go-sqlmock`, injecting clocks and seeded RNG, over-mocking pitfalls _(2026-05-03)_
18. [★ Integration & Contract Testing](testing/03-integration-and-contract-testing.md) — `testcontainers-go`, `httptest.Server` for end-to-end routes, three DB-isolation patterns, golden files, Pact contract testing, CI considerations, `//go:build integration` tags _(2026-05-03)_

---

## Tier 5 — Idiomatic Architecture Patterns

Idiomatic project layout, errors & domain modeling, hexagonal architecture in Go, and the functional options pattern.

19. [★ Idiomatic Project Layout](patterns/01-idiomatic-project-layout.md) — package as the unit of cohesion, naming rules, `internal/` vs `pkg/` vs root, `cmd/` for binaries, splitting by domain not technical layer, what NOT to mirror from Java enterprise structure _(2026-05-03)_
20. [★ Errors & Domain Modeling](patterns/02-errors-and-domain-modeling.md) — three-layer error strategy (domain / infra / transport), sentinel vs typed at each layer, wrapping vs swapping, mapping at boundaries, `errors.Join` (Go 1.20+), `cockroachdb/errors` for stack traces, OTel exception spans _(2026-05-03)_
21. [○ Hexagonal Architecture in Go](patterns/03-hexagonal-architecture.md) — ports as interfaces defined where consumed, adapters in `/internal/<adapter>/`, manual DI via constructors, testing seams, when NOT to bother, contrast with Spring `@Autowired` and NestJS providers _(2026-05-03)_
22. [★ The Functional Options Pattern](patterns/04-functional-options.md) — `func(*T)` variadic options on a constructor, required vs optional split, defaulting before applying, typed `Option` interfaces, gRPC / OTel / AWS SDK v2 in the wild, contrast with the builder pattern _(2026-05-03)_

---

## Tier 6 — HTTP & gRPC Frameworks

`net/http` deep dive, comparison of chi / gin / echo / fiber, and gRPC in Go.

23. [★ `net/http` Deep Dive](frameworks/01-net-http-deep-dive.md) — `Handler` as a one-method interface, `ServeMux` (Go 1.22 method patterns, wildcards), middleware chains, the five `Server` timeout fields (`ReadHeaderTimeout` is security-critical), graceful shutdown, `MaxBytesReader`, `ResponseWriter` optional interfaces _(2026-05-03)_
24. [★ chi vs gin vs echo vs fiber](frameworks/02-chi-vs-gin-vs-echo-vs-fiber.md) — net/http-compatible camp (chi, gin, echo) vs the fasthttp camp (fiber), routing-benchmark caveats, decision matrix, contrast with Express / Fastify / NestJS / Spring Boot _(2026-05-03)_
25. [★ gRPC in Go](frameworks/03-grpc-in-go.md) — protobuf toolchain, `grpc.NewClient` and the `Dial` deprecation, server / client / bidi streaming, interceptors, deadline propagation via context, the 17 status codes, TLS / mTLS, gRPC-Gateway and connect-go _(2026-05-03)_

---

## Tier 7 — Data Access

`database/sql` and the Go data-access stack, plus the sqlc / GORM / Ent landscape.

26. [★ `database/sql` and the Go Data-Access Stack](data-access/01-database-sql.md) — driver registration, connection pool tuning (`MaxOpenConns` / `MaxIdleConns` / `ConnMaxLifetime`), `QueryContext` family, NULL handling, transactions and isolation levels, `sql.ErrNoRows` sentinel, `sqlx`, `pgx`, migrations (golang-migrate / goose / Atlas) _(2026-05-03)_
27. [★ sqlc vs GORM vs Ent](data-access/02-sqlc-gorm-ent.md) — SQL-first codegen vs full-feature ORM vs graph-based code-first ORM, decision matrix, migration story per option, contrast with Prisma and JPA/Hibernate _(2026-05-03)_

---

## Tier 8 — Production Go

Profiling with `pprof`, observability with `slog` and OpenTelemetry, and Go in Kubernetes.

28. [★ Profiling Go Services with pprof](production/01-profiling-with-pprof.md) — `runtime/pprof` (programmatic) vs `net/http/pprof` (live HTTP) vs `go tool pprof` (CLI), the seven profile types, flame graphs, `runtime/trace` for scheduler timelines, production-safe practices, contrast with V8 inspector and JFR _(2026-05-03)_
29. [★ Observability for Go Services](production/02-observability.md) — `log/slog` (Go 1.21+) handlers and attribute groups, OpenTelemetry-Go SDK setup, `otelhttp` / `otelsql` / `otelgrpc` auto-instrumentation, W3C Trace Context propagation, semantic conventions, contrast with pino + opentelemetry-js and Spring Boot + Micrometer _(2026-05-03)_
30. [★ Go in Kubernetes](production/03-go-in-kubernetes.md) — multi-stage `CGO_ENABLED=0` Dockerfile, scratch / distroless / alpine tradeoffs, hardened pod `securityContext`, `GOMAXPROCS` and `automaxprocs`, `GOMEMLIMIT` for cgroup memory ceilings, graceful shutdown via `signal.NotifyContext` and `Server.Shutdown`, liveness vs readiness vs startup probes _(2026-05-03)_

---

## Quick Reference by Topic

### Language Foundations

- [Go Tour for TS + Java Developers](fundamentals/01-go-tour-for-ts-java-devs.md)
- [Module System & Project Layout](fundamentals/02-module-system-and-layout.md)
- [Types, Zero Values & Composite Literals](fundamentals/03-types-zero-values-composites.md)
- [Slices, Arrays & Maps](fundamentals/04-slices-arrays-maps.md)
- [Interfaces & Structural Typing](fundamentals/05-interfaces-and-structural-typing.md)
- [Errors as Values](fundamentals/06-errors-as-values.md)
- [Pointers, Methods & Receivers](fundamentals/07-pointers-methods-receivers.md)

### Concurrency

- [Goroutines & the Scheduler (G-M-P)](concurrency/01-goroutines-and-scheduler.md)
- [Channels in Practice](concurrency/02-channels-in-practice.md)
- [The `sync` Package & the Go Memory Model](concurrency/03-sync-package-and-memory-model.md)
- [Context Propagation](concurrency/04-context-propagation.md)

### Runtime Internals

- [Go's Garbage Collector](runtime/01-garbage-collector.md)
- [Escape Analysis & Allocation](runtime/02-escape-analysis-and-allocation.md)
- [Generics in Go](runtime/03-generics.md)
- [Reflection & `unsafe`](runtime/04-reflection-and-unsafe.md)

### Testing

- [The `testing` Package — Fundamentals](testing/01-testing-package-fundamentals.md)
- [Mocking & Test Doubles in Go](testing/02-mocking-and-test-doubles.md)
- [Integration & Contract Testing](testing/03-integration-and-contract-testing.md)

### Architecture

- [Idiomatic Project Layout](patterns/01-idiomatic-project-layout.md)
- [Errors & Domain Modeling](patterns/02-errors-and-domain-modeling.md)
- [Hexagonal Architecture in Go](patterns/03-hexagonal-architecture.md)
- [The Functional Options Pattern](patterns/04-functional-options.md)

### Frameworks

- [`net/http` Deep Dive](frameworks/01-net-http-deep-dive.md)
- [chi vs gin vs echo vs fiber](frameworks/02-chi-vs-gin-vs-echo-vs-fiber.md)
- [gRPC in Go](frameworks/03-grpc-in-go.md)

### Data Access

- [`database/sql` and the Go Data-Access Stack](data-access/01-database-sql.md)
- [sqlc vs GORM vs Ent](data-access/02-sqlc-gorm-ent.md)

### Production

- [Profiling Go Services with pprof](production/01-profiling-with-pprof.md)
- [Observability for Go Services](production/02-observability.md)
- [Go in Kubernetes](production/03-go-in-kubernetes.md)

---

## Bug Spotting

Active-recall practice docs. Each presents 22+ broken snippets organized by difficulty (Easy / Subtle / Senior trap), with one-line `<details>` hints inline and full root-cause + fix in a Solutions section. Every bug cites a real reference (RFC, CVE, official-doc gotcha, postmortem, library issue). Use these to pressure-test concept knowledge after working through the tiers above.

- _(planned — slices/maps and concurrency are the natural first targets)_
