---
title: "Idiomatic Project Layout"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, patterns, project-layout, package-design]
---

# Idiomatic Project Layout

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `patterns` `project-layout` `package-design`

---

## Table of Contents

1. [The package as the unit of cohesion](#1-the-package-as-the-unit-of-cohesion)
2. [Naming packages](#2-naming-packages)
3. [Accept interfaces, return structs — and what it does to layout](#3-accept-interfaces-return-structs--and-what-it-does-to-layout)
4. [`internal/`, `pkg/`, and the module root](#4-internal-pkg-and-the-module-root)
5. [`cmd/` for binaries](#5-cmd-for-binaries)
6. [Splitting by domain, not by technical layer](#6-splitting-by-domain-not-by-technical-layer)
7. [Avoiding circular imports — the package design forcing function](#7-avoiding-circular-imports--the-package-design-forcing-function)
8. ["Standard Go Project Layout" — useful, not blessed](#8-standard-go-project-layout--useful-not-blessed)
9. [What not to do (mirroring a Java enterprise layout)](#9-what-not-to-do-mirroring-a-java-enterprise-layout)

## Summary

Go's smallest unit of organization is the **package**, not the class. A package is a directory of `.go` files compiled as one translation unit, with shared visibility (lowercase = package-private, uppercase = exported). The Java instinct of "one type per file, one file per concept, group files by layer" produces friction in Go because Go expects the directory itself to be the cohesive unit. Layout decisions in Go are therefore mostly package-design decisions: what belongs in the same directory, what doesn't, and where the seams sit.

This doc covers naming conventions (short, lowercase, no `util`/`common`/`models`), the `cmd/` and `internal/` layout idioms, why splitting by domain beats splitting by technical layer in Go, and why the much-cited `golang-standards/project-layout` repo is a useful reference but explicitly disclaims being an official standard.

## 1. The package as the unit of cohesion

In Java, the unit of cohesion is the **class**. Files almost always contain one public type. Packages exist mostly to group related classes and to provide visibility tiers (`public`, `protected`, package-private, `private`). Spring further amplifies the per-class focus: a `UserService` class, a `UserController` class, a `UserRepository` interface — each in its own file, each annotated, each wired by the container.

In Go, the unit of cohesion is the **package**. A package is a directory; the files inside it share scope and visibility. There is no `private`/`protected` keyword — the only access control is "lowercase identifier = unexported (visible only inside the package)". This pushes you to think about what a *package* exposes, not what a *type* exposes.

```text
auth/
├── auth.go        // type Authenticator interface, public functions
├── token.go       // token parsing, lowercase helpers
├── token_test.go
└── session.go     // session tracking
```

All four files share scope. `token.go` can call any unexported function in `auth.go` without ceremony. The unit you reason about — and the unit you put in the directory listing — is the package, not the type.

The first practical implication: **don't reflexively split a file just because the type count rose**. If three small structs participate in one logical operation, it is fine for them to live in the same file. Go's standard library does this constantly — `net/http/request.go` contains `Request`, several helpers, and a dozen small types.

## 2. Naming packages

Go is strict about package naming. Effective Go and the official Go blog post "Package names" both push the same set of rules:

- **Short, single word, lowercase.** Not `mixedCase`, not `snake_case`. The package name appears at every call site (`bufio.NewReader`), so length costs you readability.
- **No plurals.** It's `bytes`, not `byteutils`; `time`, not `times`.
- **Don't repeat the name in exported identifiers.** Clients write `http.Server`, not `http.HTTPServer`. Inside `package http`, the type is `Server`.

Rakyll/Jaana Dogan (a former Go team member) and the official Go blog both list the names you should not pick:

| Anti-name | Why it's bad |
|---|---|
| `util` | Tells the caller nothing about what's inside. |
| `common` | Same — a junk drawer. |
| `helpers` | Same. |
| `misc` | Literally "miscellaneous". |
| `base` | OO inheritance vocabulary; Go has no inheritance. |
| `models` | A Java/Rails-ism. Anemic types separate from behavior is an anti-pattern in Go. |
| `api` | Too vague — every package is "an API". |
| `types` | Same. |
| `interfaces` | Same; also pushes you toward the wrong place to define interfaces (see §3). |

The Go blog states the rule explicitly:

> Packages named `util`, `common`, or `misc` provide clients with no sense of what the package contains. This makes it harder for clients to use the package and makes it harder for maintainers to keep the package focused. Over time, they accumulate dependencies that can make compilation significantly and unnecessarily slower, especially in large programs.
>
> — https://go.dev/blog/package-names

When you reach for `util`, the real package is hiding. If a function is about parsing strings, it belongs in a package called `strparse` or in the existing `strings` consumer. If it's about retrying RPCs, it belongs in `retry`. The exercise of naming forces the design.

## 3. Accept interfaces, return structs — and what it does to layout

The Go community heuristic is "accept interfaces, return structs": a function should accept the smallest interface its work requires, and return a concrete struct so callers can use any method without re-asserting.

```go
// Bad: forces every caller to know about *userpg.Repo
func NewService(r *userpg.Repo) *Service { ... }

// Good: any type with these two methods works
type UserStore interface {
    GetUser(ctx context.Context, id string) (*User, error)
    SaveUser(ctx context.Context, u *User) error
}
func NewService(s UserStore) *Service { ... }
```

A consequence the Java/NestJS world finds counterintuitive: **interfaces are defined where they are *consumed*, not where they are *implemented*.** In Java/Spring, the `UserRepository` interface lives in the same package as the implementation (often the same file). In Go, the consumer (the `service` package) declares the interface it needs; the implementation (the `userpg` package) is just a struct with methods that happen to satisfy it.

This shapes the layout:

```text
internal/
├── user/                     // domain
│   ├── service.go            // type Service struct; interface UserStore declared here
│   └── user.go               // type User struct
└── userpg/                   // adapter
    └── repo.go               // type Repo struct with GetUser/SaveUser methods
```

`userpg` doesn't import `user` for the interface — there is no shared interface package to import. The implementation just satisfies the consumer's interface structurally. There is no `interfaces/` directory in idiomatic Go for this reason: it would push interfaces *away* from the consumer that defines them.

See [05-interfaces-and-structural-typing.md](../fundamentals/05-interfaces-and-structural-typing.md) for the full argument.

## 4. `internal/`, `pkg/`, and the module root

Go has exactly one **compiler-enforced** layout rule:

> An import of a path containing the element "internal" is disallowed if the importing code is outside the tree rooted at the parent of the "internal" directory.
>
> — Go Modules Reference (https://go.dev/ref/mod)

This means `myapp/internal/auth` is importable from anywhere under `myapp/` but not from any external module. It is the Go equivalent of `internal` in C# or package-private in Java, with a directory-tree shape instead of a keyword.

Where this lands in practice:

```text
myapp/
├── go.mod
├── server.go              // package myapp — public, importable
├── internal/              // private to this module
│   ├── auth/
│   ├── billing/
│   └── repo/
└── cmd/
    └── api/main.go
```

**Default to `internal/` for everything that isn't explicitly meant to be a public library.** It's the closest Go gets to enforced encapsulation. A type imported under `internal/` can never accidentally become someone else's public dependency.

`pkg/` is **not** compiler-enforced. The `golang-standards/project-layout` README describes it as a convention for "library code that's ok to use by external applications" — but the same README states it is "not universally accepted in the Go community," and the Go standard library itself does not use `pkg/`. Many large projects (Kubernetes, Prometheus) have moved away from it. The current advice from most Go practitioners:

- For an app/service: don't bother with `pkg/`. Put everything in `internal/`. If a few things need to be public, put them at the module root or in a clearly named directory.
- For a library: put public code at the module root or in well-named subdirectories. `pkg/` adds path noise without compiler benefit.

See [02-module-system-and-layout.md](../fundamentals/02-module-system-and-layout.md) §5 for the full breakdown of layout directories.

## 5. `cmd/` for binaries

`cmd/<name>/main.go` is the convention for binary entry points. Each subdirectory under `cmd/` produces one binary:

```text
myapp/
├── cmd/
│   ├── api/main.go          // builds to ./api
│   ├── worker/main.go       // builds to ./worker
│   └── migrate/main.go      // builds to ./migrate
└── internal/
    └── ...
```

`go build ./cmd/api` produces a binary named `api`. `main.go` is intentionally thin — it parses flags, reads config, wires dependencies via constructor calls, and starts the server. **It is the only place in the program where the wiring happens.** No DI container, no annotations, no autowiring magic. A typical `main.go` is 50–200 lines.

```go
func main() {
    cfg := config.MustLoad()
    db := postgres.MustOpen(cfg.DB)
    cache := redis.MustOpen(cfg.Redis)

    repo := userpg.New(db)
    svc := user.NewService(repo, cache)
    srv := httpadapter.New(svc, cfg.Listen)

    log.Fatal(srv.Run(ctx))
}
```

This is the "manual DI" half of hexagonal architecture in Go, covered in [03-hexagonal-architecture.md](03-hexagonal-architecture.md). The contrast with Spring's `@SpringBootApplication` autowiring is sharp: in Go, you can read the entire dependency graph by reading `main.go` top-to-bottom.

## 6. Splitting by domain, not by technical layer

The default Java/Spring layout splits by **technical layer**:

```text
src/main/java/com/example/myapp/
├── controller/
│   ├── UserController.java
│   ├── OrderController.java
│   └── BillingController.java
├── service/
│   ├── UserService.java
│   └── ...
├── repository/
└── model/
```

Translating this directly to Go produces an awkward shape:

```text
internal/
├── controller/
│   ├── user.go
│   ├── order.go
│   └── billing.go
├── service/
│   ├── user.go
│   └── ...
└── repository/
```

Two problems land immediately:

1. **Cross-cutting noise.** Touching the `User` domain means editing files in `controller/`, `service/`, and `repository/` — three directories where each has a single user-related file alongside files about other domains.
2. **Anemic packages.** `service/` ends up with no shared abstractions of its own — every type in there belongs to a different domain and just happens to live next to the others.

The idiomatic Go layout splits by **domain**:

```text
internal/
├── user/                     // owns User type, UserService, UserStore interface
│   ├── service.go
│   ├── user.go
│   └── service_test.go
├── order/
│   ├── service.go
│   └── order.go
├── billing/
│   └── ...
├── userpg/                   // postgres adapter for user
├── orderpg/                  // postgres adapter for order
├── httpapi/                  // HTTP transport: routes everything
└── platform/                 // genuinely cross-cutting (logger, metrics)
```

Now the user domain is a single directory. Tests live with their code. The `httpapi` package handles transport concerns for all domains in one place (because routing is genuinely cross-cutting in HTTP — middleware, auth, content negotiation), while business logic stays per-domain.

Even Spring projects moving to "package by feature" land here for the same reason. Go just makes the cost of the wrong shape immediate via package boundaries and circular imports.

## 7. Avoiding circular imports — the package design forcing function

Go forbids circular imports at the package level. If `user` imports `order` and `order` imports `user`, the compiler refuses. There is no equivalent to Java's tolerated cyclic-class-dependency-within-a-package, because in Go that would be a single package and the cycle would not exist; or it would be two packages and the cycle would be illegal.

This sounds annoying. In practice it forces good design. Cycles between domains almost always indicate one of:

- **A shared concept that wants its own package.** If `user` and `order` both refer to a `Customer`, lift `customer` into its own package they both import.
- **A direction you got wrong.** If `order` refers to `user.User`, that's normal — orders belong to users. If `user` refers to `order.Order`, that's suspect — does `User` really need to know about its orders, or does the orders package fetch them given a user ID?
- **A cross-cutting interface that should be defined where it's consumed.** If `user` needs to publish events and `eventbus` needs to know about user events, the answer isn't a cycle — it's an `EventPublisher` interface declared in the `user` package and an implementation in `eventbus`.

The forcing function works in your favor. Every cycle is a design problem the compiler is naming for you.

## 8. "Standard Go Project Layout" — useful, not blessed

The repository at `github.com/golang-standards/project-layout` is the most-cited Go layout reference on the internet. Its README opens with:

> This is `NOT` an official standard defined by the core Go dev team. ... This is a set of common historical and emerging project layout patterns in the Go ecosystem.

Treat it as **descriptive, not prescriptive**. It documents what large Go projects have done, including some choices that the community has since soured on (`pkg/`, deep `vendor/` discipline, separate `api/` for OpenAPI specs). Read it for vocabulary and to understand what other repos have done; don't treat it as a checklist to copy.

The shapes that have stuck:

- `cmd/<name>/main.go` for binaries — universal.
- `internal/` for private packages — universal because the compiler enforces it.
- `go.mod`, `go.sum` at the root — required.

The shapes that are optional:

- `pkg/` — controversial, often skipped.
- `api/`, `web/`, `configs/`, `scripts/`, `deployments/` — useful when applicable, optional otherwise.
- `test/` — many projects keep tests beside the code (`*_test.go`) and reserve `test/` for integration/e2e fixtures only.

## 9. What not to do (mirroring a Java enterprise layout)

A short list of layout decisions that are technically legal but fight Go's grain:

**A `models/` package full of anemic structs.**
This is the strongest Java-ism to unlearn. Go structs that own their behavior live with the code that operates on them. A `User` struct's validation logic, its JSON marshalling tweaks, and its database scanner all belong in the `user` package, not split across `models/`, `dto/`, and `entity/`.

**Per-class files when the class has nothing else.**
A 30-line file containing only `type UserDTO struct { ... }` and nothing else is a sign you're translating from Java rather than designing for Go. Keep small types together; split when a file actually grows.

**Dependency injection containers.**
`uber/dig` and `google/wire` exist; they are not the default. `wire` does compile-time DI codegen and is reasonable when wiring becomes truly tedious. `dig` does runtime DI and has the same surprises as Spring's reflection-driven container. For most Go services, the manual-wiring `main.go` is the right answer — and it's much shorter than the Spring `@Configuration` equivalent because there are far fewer types to wire.

**`interfaces/` packages.**
Defining all interfaces in one shared package re-creates the problem `internal/` and "interfaces defined where consumed" were designed to avoid. Interfaces belong with their consumer.

**Mirroring `controller/service/repository/model/` from Spring.**
Already covered in §6. The shape produces cross-cutting churn and anemic packages. Domain-first is the idiomatic answer.

**`base.go` or `Base*` types.**
Go has no inheritance. There is no abstract base class. Embedding gives you composition, not inheritance, and a base type with embed-me semantics tends to leak abstraction. Prefer interfaces and composition.

A useful smell test for any new directory: *if I imagine reading the package name aloud at a code review, does it tell me what's inside?* `auth/`, `billing/`, `userpg/`, `httpapi/` — yes. `util/`, `common/`, `models/` — no.

## Related

- [Go Module System & Project Layout](../fundamentals/02-module-system-and-layout.md) — `cmd/`, `internal/`, `pkg/` mechanics
- [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) — accept interfaces, return structs
- [Pointers, Methods & Receivers](../fundamentals/07-pointers-methods-receivers.md) — methods and the package they live in
- [Hexagonal Architecture in Go](03-hexagonal-architecture.md) — how layout maps to ports & adapters
- [Errors & Domain Modeling](02-errors-and-domain-modeling.md) — error types per domain package
- [TypeScript DDD Tactical Patterns](../../typescript/patterns/ddd-tactical-patterns.md) — domain modeling cross-reference
- [Low-Level Design Index](../../low-level-design/INDEX.md) — package design vs class design

## References

- Effective Go, "Names" / "Package names" — https://go.dev/doc/effective_go#names
- "Package names" (official Go blog, Sameer Ajmani) — https://go.dev/blog/package-names
- Go Code Review Comments, "Package Names" — https://go.dev/wiki/CodeReviewComments
- Go FAQ, "How do I create a multifile package?" — https://go.dev/doc/faq
- Go Modules Reference, `internal` rule — https://go.dev/ref/mod
- `golang-standards/project-layout` (community, explicitly not official) — https://github.com/golang-standards/project-layout
- "Style guideline for Go packages" (Jaana Dogan; rakyll.org, archived) — referenced via Go Code Review Comments link list
