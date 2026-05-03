---
title: "Context Propagation"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, concurrency, context, cancellation, deadlines, request-scoped]
---

# Context Propagation

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `concurrency` `context` `cancellation` `deadlines` `request-scoped`

---

## Table of Contents

1. [What `context.Context` is and the problem it solves](#1-what-contextcontext-is-and-the-problem-it-solves)
2. [`Background`, `TODO`, and the root of the tree](#2-background-todo-and-the-root-of-the-tree)
3. [`WithCancel`, `WithTimeout`, `WithDeadline` — and the `defer cancel()` rule](#3-withcancel-withtimeout-withdeadline--and-the-defer-cancel-rule)
4. [`WithValue`: the most-misused constructor](#4-withvalue-the-most-misused-constructor)
5. [The `ctx` parameter convention](#5-the-ctx-parameter-convention)
6. [Context in `net/http`, `database/sql`, gRPC, `os/exec`](#6-context-in-nethttp-databasesql-grpc-osexec)
7. [`AfterFunc`, `WithoutCancel`, `WithDeadlineCause` (Go 1.21)](#7-afterfunc-withoutcancel-withdeadlinecause-go-121)
8. [Anti-patterns](#8-anti-patterns)
9. [Comparison: TS `AbortController`, Java `InterruptedException`](#9-comparison-ts-abortcontroller-java-interruptedexception)

## Summary

`context.Context` is Go's standard way to carry cancellation, deadlines, and request-scoped values across API boundaries — the connective tissue between an HTTP handler, the database calls and RPC fan-outs it triggers, and the goroutines those spawn. The contract is explicit: every function that may run for a non-trivial amount of time, do I/O, or start a goroutine takes a `Context` as its first parameter, named `ctx`. When the request that started everything is canceled or times out, every leaf of the tree learns about it through the same `ctx.Done()` channel.

The package is small (a handful of constructors and one interface) but it is opinionated, and most production Go bugs around it are about misusing `WithValue`, leaking the `cancel` function, swallowing `ctx.Err()`, or forgetting to thread the context through.

## 1. What `context.Context` is and the problem it solves

A `Context` is an interface:

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

The four methods do four jobs. `Done()` returns a channel that closes when the context is canceled or expires — this is what every goroutine `select`s on. `Err()` tells you *why* it was canceled (`context.Canceled`, `context.DeadlineExceeded`, or as of Go 1.21, a custom cause via `context.Cause`). `Deadline()` lets a callee adjust its own behavior (e.g., a database driver setting a query timeout from the remaining time). `Value()` pulls a request-scoped value out of the context.

The problem context solves is summarized in the [Go blog post that introduced it](https://go.dev/blog/context):

> When an incoming request arrives, the server typically spawns multiple goroutines (for database access, RPC calls, etc.). When the request is canceled or times out, all goroutines working on that request should exit quickly so the system can reclaim any resources they are using.

Before context, every package invented its own cancellation channel, every API needed a per-package "stop me" mechanism, and there was no standard way to propagate a deadline. Context unifies the wire.

The package was promoted from `golang.org/x/net/context` into the standard library in Go 1.7 — the [Go 1.7 release notes](https://go.dev/doc/go1.7) note that the move "allows the use of contexts for cancellation, timeouts, and passing request-scoped data in other standard library packages, including `net`, `net/http`, and `os/exec`."

## 2. `Background`, `TODO`, and the root of the tree

Every context tree has a root. Two roots exist; they differ only in intent.

```go
context.Background()  // top-level, never canceled, never has a deadline or values
context.TODO()        // a placeholder for "I don't have a context yet" — same behavior, different signal to humans and tools
```

Use `Background` in `main`, in initialization, in tests, and in long-running daemon-like goroutines that have no enclosing request. Use `TODO` when you are mid-refactor and have not yet plumbed the right context through — `TODO` documents the intent, and static analysis tools (e.g., the [`contextcheck` linter](https://pkg.go.dev/github.com/sylvia7788/contextcheck)) can flag it.

Practical rule: in real request-handling code paths, you almost never write `context.Background()`. The framework or transport layer (`net/http`, gRPC) hands you a context derived from the incoming request; you derive children from that.

## 3. `WithCancel`, `WithTimeout`, `WithDeadline` — and the `defer cancel()` rule

Three constructors derive a child context. Each returns a `(ctx, cancel)` pair, and each requires that you call `cancel`.

```go
ctx, cancel := context.WithCancel(parent)
ctx, cancel := context.WithTimeout(parent, 5 * time.Second)
ctx, cancel := context.WithDeadline(parent, time.Now().Add(5 * time.Second))
```

The pattern at every call site is:

```go
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()

doWork(ctx)
```

`defer cancel()` is not optional. The `cancel` function tears down the context's internal state and removes it from the parent's children list; if you do not call it, the context lingers until the parent is canceled — which, for a long-lived parent like an HTTP server's base context, may be never. The package doc puts this directly: failing to call `cancel` "leaks" the context, and `go vet`'s `lostcancel` analyzer will flag the omission. (The cost of calling `cancel()` after the timeout has already fired is zero; cancel is idempotent.)

A clean cascade for a request:

```go
func (s *Server) handle(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    user, err := s.users.LookupContext(ctx, r.URL.Query().Get("id"))
    if err != nil { /* ... */ }

    items, err := s.catalog.SearchContext(ctx, user.SearchPrefs)
    if err != nil { /* ... */ }

    writeJSON(w, items)
}
```

`r.Context()` is automatically canceled when the client disconnects (see §6). The timeout further bounds total work to 2s. Both child operations see `ctx.Done()` fire if the deadline expires *or* the client hangs up.

A goroutine that respects context looks like this:

```go
func (s *Service) bgPoll(ctx context.Context) {
    t := time.NewTicker(time.Minute)
    defer t.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-t.C:
            s.tick(ctx)
        }
    }
}
```

Every blocking operation pairs with `ctx.Done()` in a `select`. This is the universal shape.

## 4. `WithValue`: the most-misused constructor

`context.WithValue(parent, key, val)` returns a child that carries a key/value. The package doc spells out the discipline:

> WithValue should be used only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions.

Translation: a `Context` is not a free-form bag for "anything I might want to thread through the call stack." Putting your logger, your database handle, or your config in the context bypasses the type system, hides dependencies, and makes refactoring miserable.

What does qualify:

- **Request ID / trace ID / span context** — propagated by middleware, read by logging and downstream RPC.
- **Authenticated principal / tenant ID** — set by an auth middleware, read by authorization checks deep in the call stack.
- **Locale / preferred language** — for response formatting.

What does not qualify:

- A database connection or transaction. (Pass it explicitly. If you have a `Tx`, every function that uses it should declare that in its signature.)
- A logger. (Pass it explicitly, or use a package-level logger and add request fields with a slog `Handler`.)
- "Optional" config. (Optional config goes in function arguments, struct fields, or a real config object.)

Two correctness rules:

**Use unexported key types.** Two packages that both used `string` keys could collide. The convention is a per-package unexported type:

```go
package middleware

type ctxKey int
const userIDKey ctxKey = 0

func WithUserID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, userIDKey, id)
}

func UserID(ctx context.Context) (string, bool) {
    s, ok := ctx.Value(userIDKey).(string)
    return s, ok
}
```

Callers cannot construct a `middleware.ctxKey`, so they cannot collide.

**`Value` is O(depth).** Lookups walk up the chain of derived contexts. This is fine in practice — request contexts rarely nest more than a handful of levels — but if you find yourself reading the same value tens of thousands of times in a hot loop, hoist it to a local variable.

## 5. The `ctx` parameter convention

The convention is older than the package itself, and it appears verbatim in the package doc:

> Do not store Contexts inside a struct type; instead, pass a Context explicitly to each function that needs it. The Context should be the first parameter, typically named ctx.

```go
func DoSomething(ctx context.Context, arg Arg) error {
    // ... use ctx ...
}
```

Three sub-rules unpack this:

**`ctx` first, always.** Even before `self`-equivalents like a method receiver — well, the receiver still comes first because Go syntax requires it, but `ctx` is the first regular parameter. This is universal in the standard library and the broader ecosystem.

**Don't store `ctx` in a struct.** A `Context` is request-scoped. Stuffing it in a `*Service` field makes one request's context leak across requests, makes cancellation semantics incoherent, and confuses every reader. The exception is genuinely request-scoped struct types (e.g., a per-request handler builder), where storing the context for the lifetime of that one request is fine; for service-wide types, never.

**Don't pass `nil`.** Use `context.TODO()` if you have nothing better. The standard library will not panic on a nil context, but a lot of third-party code will, and `nil` defeats every linter.

## 6. Context in `net/http`, `database/sql`, gRPC, `os/exec`

The standard library and the major ecosystem libraries are wired to context end-to-end. You almost never call `Background()` in real request paths — you start from a context the framework hands you and derive children.

**`net/http`.** `(*http.Request).Context()` returns a `context.Context` that is canceled when the client closes the connection, when the server's `Shutdown` is called, or when the response is finished. To propagate context through to the next hop, use `req = req.WithContext(ctx)` (or build the request with `http.NewRequestWithContext`):

```go
func (c *Client) Get(ctx context.Context, url string) (*Response, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil { return nil, err }
    return c.do(req)
}
```

The `http.Client` will cancel the underlying transport request when the context is done — the package doc notes "the Client cancels requests to the underlying Transport as if the Request's Context ended." You get socket-level cancellation for free.

**`database/sql`.** Every method has a `Context` variant: `QueryContext`, `ExecContext`, `BeginTx` (which takes a context), `PingContext`. Use them. Without context, a slow query has no way to be canceled when the client times out — your goroutine sits in the driver waiting for the server to finish, and you've leaked a goroutine plus held a connection from the pool.

```go
rows, err := db.QueryContext(ctx, "SELECT id FROM users WHERE tenant=$1", tenantID)
```

If `ctx` expires, the driver issues the database-specific cancel (`pq_cancel` for Postgres, etc.) and the goroutine returns promptly.

**gRPC.** Every generated client method takes a `context.Context` as its first argument. The grpc-go library propagates the context to the server side, including its deadline — the server sees a `Context` whose `Deadline()` reflects the caller's remaining budget. Trace and tenancy metadata travel through gRPC interceptors and are typically attached via `metadata.NewOutgoingContext`.

**`os/exec`.** `exec.CommandContext(ctx, name, args...)` returns a `*exec.Cmd` that, when the context is canceled, kills the process. This is invaluable for bounded subprocess work — without it, a hanging child process leaks until something else reaps it.

## 7. `AfterFunc`, `WithoutCancel`, `WithDeadlineCause` (Go 1.21)

Go 1.21 added several context utilities documented in the [Go 1.21 release notes](https://go.dev/doc/go1.21):

**`context.AfterFunc(ctx, f)`** registers `f` to run in its own goroutine when `ctx` is canceled. It returns a `stop` function that, if called before `f` runs, prevents the run. This compresses a common pattern:

```go
// before:
go func() {
    <-ctx.Done()
    cleanup()
}()

// after:
stop := context.AfterFunc(ctx, cleanup)
defer stop()
```

The new form does not start a goroutine eagerly; the runtime arranges things internally. It is the right primitive for "do X when this context dies" — for example, closing a watcher, releasing a lease, or logging a deadline-exceeded event.

**`context.WithoutCancel(parent)`** returns a context that copies `parent`'s values but is *not* canceled when `parent` is. Use it sparingly. The legitimate case is a fire-and-forget background job that should outlive the request that started it — e.g., asynchronously sending an audit log entry — but where you still want the request's trace ID to flow into the log:

```go
func handle(w http.ResponseWriter, r *http.Request) {
    bgCtx := context.WithoutCancel(r.Context())
    go writeAuditLog(bgCtx, /* ... */)  // request can return; audit log keeps trace ID
    /* ... */
}
```

You are explicitly opting out of cancellation. That is a strong signal in code review and worth a comment.

**`context.WithDeadlineCause(parent, d, cause)` and `context.WithTimeoutCause(parent, t, cause)`** let you attach a custom error that will be returned by `context.Cause(ctx)` when the deadline fires. `ctx.Err()` still returns `context.DeadlineExceeded` (for compatibility); `context.Cause(ctx)` returns your richer error. This is the cleanest way to distinguish "timed out because the upstream RPC budget hit zero" from "timed out because the user-facing handler budget hit zero" in observability.

## 8. Anti-patterns

A short tour of mistakes that cause real production incidents.

**Storing `ctx` on a struct.** Already covered in §5; common enough to repeat. `s.ctx = ctx` in a constructor is wrong unless `s` is request-scoped. For service-scoped types, take `ctx` as the first argument of every method that does work.

**Passing `nil`.** Always pass a real context. If unsure, use `context.TODO()`.

**Ignoring `ctx.Err()`.** When a select case fires on `ctx.Done()`, return `ctx.Err()` (or wrap it with `fmt.Errorf("doing thing: %w", ctx.Err())`). Returning a generic `errors.New("canceled")` loses the distinction between `Canceled` and `DeadlineExceeded`; loses the `context.Cause` if one was set; and breaks `errors.Is(err, context.DeadlineExceeded)` checks higher up. (See [Errors as Values](../fundamentals/06-errors-as-values.md) for the wrapping rules.)

**Spawning a goroutine that doesn't take `ctx`.** A long-running goroutine that ignores context is a leak waiting to happen. Every `go func()` in a request path should accept the request's context (or a derived one) and `select` on `ctx.Done()`.

**Putting business state in `WithValue`.** Already covered in §4. The smell: any value passed via `WithValue` whose absence would be a logic bug rather than a missing piece of request metadata.

**Reusing a canceled context.** A canceled context cannot be "uncanceled." If you genuinely need a fresh budget for a follow-up operation that should outlive the original, you want either `WithoutCancel` (Go 1.21+) or a fresh `context.Background()` plus explicit propagation of the trace IDs you care about.

**Not respecting cancellation in CPU-bound loops.** A tight loop processing items has no I/O blocking on `ctx.Done()`, but should still check periodically:

```go
for i, item := range items {
    if i%128 == 0 {
        select {
        case <-ctx.Done(): return ctx.Err()
        default:
        }
    }
    process(item)
}
```

`ctx.Done()` is cheap to read, but doing it every iteration in a hot CPU loop is measurable; sample.

## 9. Comparison: TS `AbortController`, Java `InterruptedException`

**TypeScript / web.** The closest analog is `AbortController` and the `AbortSignal` it produces. `fetch` accepts `{ signal }`; downstream APIs accept `signal` and propagate it. The shape — derive a controller, pass the signal across API boundaries, abort to cancel — is identical to Go's pattern. Two real differences:

- `AbortSignal` is a single-purpose cancellation token. It does not carry deadlines (you set `setTimeout` and call `abort` yourself, or use `AbortSignal.timeout()`), and it does not carry request-scoped values. Go's `Context` bundles all three.
- `AbortSignal` propagates via parameters; there is no implicit attachment to a request, so frameworks like Express tend to invent their own equivalents (`req` itself, plus middleware-set fields). Go's `r.Context()` is the unified entry point.

If you are already comfortable threading `AbortSignal` through a Node or browser codebase, threading `ctx` in Go feels familiar.

**Java.** The closest analog is `Thread.interrupt()` and `InterruptedException`. A blocking method (`Thread.sleep`, `Object.wait`, `Future.get`, `BlockingQueue.take`) throws `InterruptedException` when interrupted; non-blocking code is expected to check `Thread.currentThread().isInterrupted()` periodically. Spring's `WebClient` and CompletableFutures push toward `Mono`/`Flux`-style cancellation, and modern Java is moving toward structured concurrency with `StructuredTaskScope` (preview).

The Java-Go gap is bigger than the TS-Go gap. Java cancellation is bound to threads; Go cancellation is bound to a `Context` value that flows orthogonally to whichever goroutine is actually executing. Spring Boot developers who have used Reactor's `Context` (the reactive-streams one) already have the right mental model: a value that travels with the reactive chain rather than with the executing thread.

## Related

- [Channels in Practice](02-channels-in-practice.md) — `ctx.Done()` is a `<-chan struct{}`; the patterns from §4–6 apply directly
- [Goroutines & the Scheduler (G-M-P)](01-goroutines-and-scheduler.md) — what's getting canceled when `ctx.Done()` fires
- [The `sync` Package & the Go Memory Model](03-sync-package-and-memory-model.md) — what synchronization edges the context creates
- [Errors as Values](../fundamentals/06-errors-as-values.md) — wrapping `context.Canceled` / `context.DeadlineExceeded`
- [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) — `Context` is the canonical small interface
- [TypeScript event loop internals](../../typescript/runtime/event-loop-internals.md) — `AbortSignal`'s home
- [CPU scheduling (CFS)](../../operating-systems/fundamentals/04-cpu-scheduling-cfs.md) — what the kernel does to your goroutines under load

## References

- `context` package — https://pkg.go.dev/context
- "Go Concurrency Patterns: Context" (Sameer Ajmani, blog) — https://go.dev/blog/context
- Go 1.7 release notes (context promoted to stdlib, integrated into `net`, `net/http`, `os/exec`) — https://go.dev/doc/go1.7
- Go 1.21 release notes (`AfterFunc`, `WithoutCancel`, `WithDeadlineCause`, `WithTimeoutCause`, `Cause`) — https://go.dev/doc/go1.21
- `net/http.Request.Context` — https://pkg.go.dev/net/http#Request.Context
- `net/http.NewRequestWithContext` — https://pkg.go.dev/net/http#NewRequestWithContext
- `database/sql.QueryContext` — https://pkg.go.dev/database/sql#DB.QueryContext
- `os/exec.CommandContext` — https://pkg.go.dev/os/exec#CommandContext
- `cmd/vet` `lostcancel` analyzer — https://pkg.go.dev/cmd/vet
