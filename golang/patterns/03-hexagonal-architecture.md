---
title: "Hexagonal Architecture in Go"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, patterns, hexagonal, ports-adapters, dependency-injection]
---

# Hexagonal Architecture in Go

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `patterns` `hexagonal` `ports-adapters` `dependency-injection`

---

## Table of Contents

1. [The hexagonal model briefly](#1-the-hexagonal-model-briefly)
2. [Why it lands cleanly in Go](#2-why-it-lands-cleanly-in-go)
3. [Domain at the center — pure Go types](#3-domain-at-the-center--pure-go-types)
4. [Ports as interfaces, defined where consumed](#4-ports-as-interfaces-defined-where-consumed)
5. [Adapters under `internal/<adapter>/`](#5-adapters-under-internaladapter)
6. [Manual DI via constructors — `main.go` is the wiring](#6-manual-di-via-constructors--maingo-is-the-wiring)
7. [Testing seams — fakes, integration adapters](#7-testing-seams--fakes-integration-adapters)
8. [When NOT to use hexagonal](#8-when-not-to-use-hexagonal)
9. [Comparison with Spring's `@Autowired` / NestJS providers](#9-comparison-with-springs-autowired--nestjs-providers)

## Summary

Hexagonal architecture (Cockburn's "ports and adapters") asks one design question: **what is your application, separately from the technology it talks to?** The application is the hexagon; ports are the interfaces it offers and depends on; adapters wire the ports to concrete technology — HTTP, gRPC, Postgres, Redis, Kafka, S3.

Go was not designed for this pattern, but it accidentally fits it well. Structural typing means a port is just an interface; "accept interfaces, return structs" puts the port definition where the consumer lives; `internal/` enforces module boundaries; and `cmd/<binary>/main.go` is a natural wiring location with no DI container required. Spring and NestJS need annotation-driven containers to express what Go expresses with a constructor call.

This doc shows what hexagonal looks like in idiomatic Go, when to bother (medium-and-up services), and when not to (small CLIs, tiny microservices).

## 1. The hexagonal model briefly

Alistair Cockburn published "Hexagonal Architecture" in 2005 (revised many times since). The original framing:

> Allow an application to equally be driven by users, programs, automated test or batch scripts, and to be developed and tested in isolation from its eventual run-time devices and databases.
>
> — https://alistair.cockburn.us/hexagonal-architecture/

The picture: a hexagon containing the application's logic. Each side of the hexagon is a **port** — an interface the application offers (driven side) or depends on (driving side). On the outside of each port is an **adapter** — a piece of code that translates between the port and a concrete technology.

The naming is cosmetic. "Hexagon" is just because Cockburn drew six sides; the pattern works with any number. The substance is:

- **Domain logic does not import frameworks.** The hexagon never `import`s the HTTP library, the SQL driver, or the Redis client.
- **Dependencies point inward.** Adapters depend on the domain (they implement domain interfaces); the domain does not depend on adapters.
- **Substituting one adapter for another should not require changes inside the hexagon.** Switching from Postgres to MySQL, or from REST to gRPC, touches adapters only.

This is the same shape as Onion Architecture (Jeffrey Palermo) and Clean Architecture (Robert Martin) — three names for the same dependency-rule pattern. Cockburn's version is the most language-neutral.

## 2. Why it lands cleanly in Go

Several Go conventions are already half of hexagonal:

**Interfaces are structural.** A port is just an interface. Any type that has the right methods satisfies it; the implementation doesn't `implements UserStore`. This means an adapter package doesn't need to know about the domain's interface declarations — it just needs to have the right method shapes. See [05-interfaces-and-structural-typing.md](../fundamentals/05-interfaces-and-structural-typing.md).

**Accept interfaces, return structs.** The community heuristic that constructors take interfaces and return concrete types is exactly the inversion-of-control half of hexagonal. The domain's `NewService(s UserStore)` takes a port; the caller (typically `main.go`) supplies the adapter.

**`internal/` is compiler-enforced.** Domain code goes under `internal/<domain>/` and physically cannot leak into other modules. This is the architectural seam Java/Spring projects achieve via Maven module separation, available with no build configuration.

**`cmd/<binary>/main.go` is the wiring point.** Go's convention of one main package per binary, with constructor calls in main, gives you the dependency-injection composition root for free.

Effective Go's interface guidance lines up with the same instinct:

> Interfaces with only one or two methods are common in Go code, and are usually given a name derived from the method, such as `io.Writer` for something that implements `Write`.
>
> — https://go.dev/doc/effective_go#interfaces

Small interfaces are exactly what good ports look like. When `UserStore` has 12 methods, it's a smell that the domain has an unfocused dependency on its persistence — and probably that you've inverted the control flow incompletely.

## 3. Domain at the center — pure Go types

The domain package owns the **types and rules**. It does not import:

- HTTP libraries (`net/http`, `chi`, `gin`).
- Database drivers (`database/sql`, `pgx`, `gorm`).
- Caching libraries (`go-redis`).
- Queue clients (`kafka-go`, `nats.go`).
- Any framework.

It may import:

- The standard library (`time`, `context`, `fmt`, `errors`, `unicode`).
- Other domain packages within the same module.
- Tiny utility libraries that don't tie you to a runtime technology (`uuid`, `decimal`).

```go
// internal/user/user.go
package user

import (
    "context"
    "errors"
    "time"
)

type User struct {
    ID        string
    Email     string
    Name      string
    CreatedAt time.Time
}

var (
    ErrNotFound     = errors.New("user not found")
    ErrEmailTaken   = errors.New("email already taken")
)

type ValidationError struct {
    Field, Message string
}

func (e *ValidationError) Error() string {
    return e.Field + ": " + e.Message
}

func (u *User) Validate() error {
    if u.Email == "" {
        return &ValidationError{"email", "required"}
    }
    return nil
}
```

```go
// internal/user/service.go
package user

import "context"

// Service is the domain entry point.
type Service struct {
    store  UserStore
    events EventPublisher
}

func NewService(store UserStore, events EventPublisher) *Service {
    return &Service{store: store, events: events}
}

func (s *Service) Register(ctx context.Context, email, name string) (*User, error) {
    u := &User{Email: email, Name: name, CreatedAt: time.Now()}
    if err := u.Validate(); err != nil {
        return nil, err
    }
    if err := s.store.SaveUser(ctx, u); err != nil {
        return nil, fmt.Errorf("user.Register: %w", err)
    }
    _ = s.events.PublishUserCreated(ctx, u)
    return u, nil
}
```

The service has no idea whether `UserStore` is Postgres, an in-memory map, or a fake. It just calls methods.

## 4. Ports as interfaces, defined where consumed

This is the part that surprises Java/Spring developers most. In Spring:

```java
// repository package
public interface UserRepository extends JpaRepository<User, String> {
    Optional<User> findByEmail(String email);
}

// service package
@Service
public class UserService {
    @Autowired
    private UserRepository repo;
}
```

The interface lives **with the implementation** (the JPA-generated repository). The service depends on the repository's package.

In Go, the interface lives **with the consumer**:

```go
// internal/user/ports.go
package user

import "context"

// UserStore is the port; defined where the domain consumes it.
type UserStore interface {
    GetUser(ctx context.Context, id string) (*User, error)
    GetUserByEmail(ctx context.Context, email string) (*User, error)
    SaveUser(ctx context.Context, u *User) error
}

// EventPublisher port for outbound events.
type EventPublisher interface {
    PublishUserCreated(ctx context.Context, u *User) error
}
```

```go
// internal/userpg/repo.go — adapter, implements UserStore
package userpg

import (
    "context"
    "database/sql"
    "myapp/internal/user"
)

type Repo struct{ db *sql.DB }

func New(db *sql.DB) *Repo { return &Repo{db: db} }

func (r *Repo) GetUser(ctx context.Context, id string) (*user.User, error) { ... }
func (r *Repo) GetUserByEmail(ctx context.Context, email string) (*user.User, error) { ... }
func (r *Repo) SaveUser(ctx context.Context, u *user.User) error { ... }
```

`userpg` imports `user` (for the `User` type), but `user` does **not** import `userpg`. The dependency points inward. There is no `implements UserStore` — Go's structural typing checks this at the call site (`NewService(repoInstance, ...)` succeeds only if `repoInstance`'s methods match `UserStore`).

A consequence: **don't make a separate `interfaces/` or `ports/` package**. The whole point of putting the interface where it's consumed is that the *consumer* declares its own dependency. Centralizing interfaces fights this.

## 5. Adapters under `internal/<adapter>/`

Each adapter is its own package, named after the technology:

```text
internal/
├── user/                   // domain
│   ├── ports.go            // UserStore, EventPublisher
│   ├── service.go          // *Service struct
│   ├── user.go             // *User struct, errors
│   └── service_test.go     // uses fake UserStore
├── userpg/                 // adapter: Postgres
│   └── repo.go
├── usermem/                // adapter: in-memory (for tests, dev mode)
│   └── repo.go
├── kafkaevents/            // adapter: Kafka EventPublisher
│   └── publisher.go
├── nopevents/              // adapter: no-op EventPublisher (tests, dev)
│   └── publisher.go
└── httpapi/                // adapter: HTTP transport (the "driving" side)
    ├── router.go
    └── userhandler.go
```

The naming convention is `<domain><technology>` (`userpg`, `userredis`) or `<technology>` for cross-cutting adapters (`kafkaevents`). Avoid `userrepo` — that doesn't say which storage; you'll regret it when you add a second storage backend.

The HTTP layer is also an adapter — it's a "driving" adapter (it drives the domain). Its job is to translate HTTP requests into domain method calls and translate domain errors back to HTTP status codes. See [Errors & Domain Modeling §4](02-errors-and-domain-modeling.md#4-mapping-at-the-boundary).

```go
// internal/httpapi/userhandler.go
package httpapi

type UserService interface {  // port re-declared from the transport's perspective
    Register(ctx context.Context, email, name string) (*user.User, error)
    Get(ctx context.Context, id string) (*user.User, error)
}

type userHandler struct{ svc UserService }

func (h *userHandler) register(w http.ResponseWriter, r *http.Request) {
    var req struct{ Email, Name string }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "bad json", http.StatusBadRequest)
        return
    }
    u, err := h.svc.Register(r.Context(), req.Email, req.Name)
    if err != nil { writeError(w, err); return }
    writeJSON(w, http.StatusCreated, u)
}
```

Notice the `httpapi` package declares its **own** `UserService` interface — exactly the methods it needs. The `*user.Service` struct from the domain happens to satisfy this. This keeps the transport layer testable in isolation (you can pass a fake `UserService` from `httpapi`'s tests).

## 6. Manual DI via constructors — `main.go` is the wiring

Go does not need a DI container. The `main.go` for each binary wires everything explicitly:

```go
// cmd/api/main.go
package main

import (
    "context"
    "log"
    "os/signal"
    "syscall"

    "myapp/internal/config"
    "myapp/internal/httpapi"
    "myapp/internal/kafkaevents"
    "myapp/internal/postgres"
    "myapp/internal/user"
    "myapp/internal/userpg"
)

func main() {
    cfg := config.MustLoad()

    // Infrastructure
    db := postgres.MustOpen(cfg.DB)
    defer db.Close()

    kafka := kafkaevents.MustOpen(cfg.Kafka)
    defer kafka.Close()

    // Adapters
    userRepo := userpg.New(db)
    eventPub := kafkaevents.New(kafka, "user-events")

    // Domain
    userSvc := user.NewService(userRepo, eventPub)

    // Transport
    router := httpapi.NewRouter(userSvc)
    srv := httpapi.NewServer(cfg.Listen, router)

    // Lifecycle
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    if err := srv.Run(ctx); err != nil {
        log.Fatal(err)
    }
}
```

Read top to bottom: config, infrastructure, adapters, domain, transport, run. The entire dependency graph is visible. There is no annotation magic, no XML, no runtime discovery, no autowiring failures at startup that you don't see until production. If you want a different storage backend in tests, you call `usermem.New()` instead of `userpg.New(db)` — the rest of the wiring is identical.

This is the composition root from Clean Architecture, and Go gets it for free because there's nothing else.

For larger services where the constructor sequence becomes long enough to be tedious, Google's `wire` tool generates the wiring at compile time from a small `wire.go` declaration. It produces a function that does what `main.go` does above, with no runtime cost. `wire` is the closest Go gets to a DI framework, and it's *still* compile-time and explicit.

`uber-go/dig` exists for runtime DI. It has the same behavior surprises as Spring's reflection-driven container. For backend services, the mainstream Go advice is: just write the constructor calls.

## 7. Testing seams — fakes, integration adapters

The payoff of hexagonal in Go shows up in tests. Three common patterns:

**Unit tests against a fake adapter.**

```go
// internal/user/service_test.go
type fakeStore struct {
    users map[string]*user.User
}
func (f *fakeStore) GetUser(_ context.Context, id string) (*user.User, error) {
    u, ok := f.users[id]
    if !ok { return nil, user.ErrNotFound }
    return u, nil
}
// ... other methods

func TestRegister(t *testing.T) {
    store := &fakeStore{users: map[string]*user.User{}}
    events := nopevents.New()
    svc := user.NewService(store, events)

    u, err := svc.Register(context.Background(), "a@b.com", "Alice")
    require.NoError(t, err)
    require.Equal(t, "a@b.com", u.Email)
}
```

The fake is just a struct with the right methods. No mocking framework needed. (Mocking libraries like `gomock` exist; for small interfaces, hand-written fakes are usually clearer.)

**Integration tests against a real adapter** using `testcontainers-go` for ephemeral Postgres / Redis / Kafka. The adapter package owns its integration tests; the domain package's tests stay fast and pure.

```go
// internal/userpg/repo_test.go
func TestRepo_GetUser_NotFound(t *testing.T) {
    db := setupTestDB(t)  // uses testcontainers
    repo := userpg.New(db)
    _, err := repo.GetUser(ctx, "nonexistent")
    require.ErrorIs(t, err, user.ErrNotFound)
}
```

**End-to-end tests** drive HTTP through the real `httpapi` layer with all real adapters wired in. There's typically only a handful of these — the unit and integration layers cover most of the surface.

The leverage: a 200-line domain change touches the domain package's tests only. A storage migration touches the adapter package's tests only. A new HTTP route adds an `httpapi` test. Each layer changes independently because the seams are real.

## 8. When NOT to use hexagonal

Hexagonal has a real cost: more packages, more interfaces, more constructors, more wiring in `main.go`. For some services it's overkill:

- **Single-purpose CLIs.** A 200-line CLI that wraps an API call doesn't benefit from a hexagon — there's nothing to swap.
- **Tiny microservices.** A 5-route service that proxies to one downstream and writes to one database can put everything in one or two packages and be done.
- **Throwaway code.** Hexagonal pays back when the code lives long enough to be modified by people who didn't write it. Throwaway code doesn't reach that horizon.
- **Glue services.** A service whose entire job is "receive from queue A, write to bucket B" doesn't have a domain; making one up adds packages without value.

Apply hexagonal when:

- The service has non-trivial business rules (validation, invariants, multi-step operations).
- You expect to add more transports (HTTP today, gRPC tomorrow) or more storage backends (Postgres now, S3 cold storage later).
- The domain is something you'd want to test in isolation from infra.
- The team is more than two people, or the code will outlive the team.

The Java/DDD-fundamentalist version of hexagonal — separate value-object packages, aggregate root packages, factory packages, repository abstractions, application services, domain services — is generally too much for Go. The Go version is thinner: domain types and one Service struct per domain, with ports as interfaces and adapters in `internal/<adapter>/`. That's enough.

## 9. Comparison with Spring's `@Autowired` / NestJS providers

| Concern | Spring Boot | NestJS | Go |
|---|---|---|---|
| Define a port | `interface UserRepository` (in repo package) | provider interface in module | `type UserStore interface` in consumer package |
| Implement port | `class JpaUserRepository implements UserRepository` | `@Injectable() class UserRepoImpl` | struct with right method shapes (no `implements`) |
| Wire dependency | `@Autowired` (annotation) or `@Bean` in `@Configuration` | `@Module({ providers: [UserService] })` + `@Inject()` | constructor call in `main.go` |
| Discover at startup | reflection-based scanning of `@Component` | reflection over decorators | none — explicit |
| Override for tests | `@MockBean` | `Test.createTestingModule().overrideProvider()` | pass a different struct to the constructor |
| Failure mode if wiring is wrong | runtime `NoSuchBeanDefinitionException` at startup | runtime `Nest can't resolve dependency` | compile error |
| Annotations on domain code | `@Service`, `@Component`, `@Transactional` | `@Injectable` | none |

The Go approach trades convenience for explicitness:

- No magic — what you read in `main.go` is what runs.
- No annotation-driven framework lock-in. The domain is plain Go that compiles without the framework.
- No reflection-based runtime resolution. Wiring failures are compile errors.
- Cost: you write 30 lines of constructor calls in `main.go` instead of three annotations.

For a backend service of any meaningful size, the explicit version is usually a net win — especially in long-lived codebases where developers are reading more code than writing it. The annotations save you typing once; the explicitness pays back every time someone reads the code to understand what the system does.

The TS analogue, [hexagonal-architecture.md](../../typescript/patterns/hexagonal-architecture.md), takes the same shape — TypeScript with structural typing is similarly well-suited.

## Related

- [Idiomatic Project Layout](01-idiomatic-project-layout.md) — `internal/`, `cmd/`, package design
- [Errors & Domain Modeling](02-errors-and-domain-modeling.md) — error mapping at adapter seams
- [The Functional Options Pattern](04-functional-options.md) — adapter constructors with options
- [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) — accept interfaces, return structs
- [Pointers, Methods & Receivers](../fundamentals/07-pointers-methods-receivers.md) — method sets and interfaces
- [TypeScript Hexagonal Architecture](../../typescript/patterns/hexagonal-architecture.md) — sibling doc
- [TypeScript DDD Tactical Patterns](../../typescript/patterns/ddd-tactical-patterns.md) — domain modeling
- [Low-Level Design Index](../../low-level-design/INDEX.md) — broader pattern landscape
- [System Design Index](../../system-design/INDEX.md) — service architecture context

## References

- Alistair Cockburn, "Hexagonal Architecture" — https://alistair.cockburn.us/hexagonal-architecture/
- Effective Go, "Interfaces" — https://go.dev/doc/effective_go#interfaces
- Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software* (Addison-Wesley, 2003) — origin of much of the domain-modeling vocabulary
- Robert C. Martin, *Clean Architecture* (Prentice Hall, 2017) — same dependency rule, different framing
- Google's `wire` (compile-time DI) — https://github.com/google/wire
- Uber's `dig` (runtime DI) — https://github.com/uber-go/dig
- `testcontainers-go` (integration test infrastructure) — https://golang.testcontainers.org/
