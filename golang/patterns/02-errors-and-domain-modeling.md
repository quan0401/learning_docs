---
title: "Errors & Domain Modeling"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, patterns, errors, domain, error-wrapping]
---

# Errors & Domain Modeling

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `patterns` `errors` `domain` `error-wrapping`

---

## Table of Contents

1. [The layered error strategy](#1-the-layered-error-strategy)
2. [Sentinel vs typed errors per layer](#2-sentinel-vs-typed-errors-per-layer)
3. [Wrapping with `%w` — when to wrap, when to bare-return, when to swap](#3-wrapping-with-w--when-to-wrap-when-to-bare-return-when-to-swap)
4. [Mapping at the boundary](#4-mapping-at-the-boundary)
5. [`errors.Join` and multi-errors](#5-errorsjoin-and-multi-errors)
6. [`context.Canceled` and `context.DeadlineExceeded` as cross-cutting sentinels](#6-contextcanceled-and-contextdeadlineexceeded-as-cross-cutting-sentinels)
7. [Logging vs returning — handle once](#7-logging-vs-returning--handle-once)
8. [Stack traces — stdlib doesn't give them](#8-stack-traces--stdlib-doesnt-give-them)
9. [OpenTelemetry conventions for errors](#9-opentelemetry-conventions-for-errors)

## Summary

[Errors as Values](../fundamentals/06-errors-as-values.md) covers the language mechanics: `error` is a one-method interface, `errors.Is`, `errors.As`, `%w` wrapping, `panic`/`recover`. This doc moves up a level to **architecture**: how to organize errors across the domain / infrastructure / transport boundaries of a backend service so the same business condition produces a clean HTTP 404, a useful log line, a recognizable trace span, and a typed return value to other Go callers — without losing information at each translation.

The Spring/NestJS instinct is to throw a `UserNotFoundException` and let a global exception handler map it to HTTP. The Go approach is essentially the same idea, just expressed as values: domain packages declare typed/sentinel errors, infrastructure packages wrap their failures with `%w`, and a single boundary layer (HTTP handler, gRPC interceptor) maps domain errors to transport status. Each layer decides what's a sentinel, what's a typed error, and what's just a bare wrap-and-return.

## 1. The layered error strategy

Most Go services have three error-relevant layers:

| Layer | Cares about | Error shape |
|---|---|---|
| Domain (`internal/user`, `internal/billing`) | Business invariants ("user not found", "insufficient credit") | Sentinels and typed errors with semantic names |
| Infrastructure (`internal/userpg`, `internal/redisCache`) | Driver, network, file system failures | Wrapped underlying errors plus per-adapter sentinels (`ErrConnRefused`, `ErrSerialization`) |
| Transport (`internal/httpapi`, `internal/grpcapi`) | Mapping to status codes / gRPC codes / response shapes | Catches domain & infra errors, maps once to wire format |

The contract between them is roughly:

- Infrastructure returns errors that wrap the driver error with context, occasionally translating common cases into adapter-level sentinels.
- Domain catches infrastructure errors with `errors.Is` / `errors.As` and either translates them into a domain error (`ErrUserNotFound`) or wraps with context for higher-level inspection.
- Transport catches domain errors and writes the wire-format response. The transport layer **never** returns errors to its callers — the buck stops here.

This is the same shape as Spring's `@ControllerAdvice` global exception handlers, just made explicit with returned values instead of thrown exceptions.

## 2. Sentinel vs typed errors per layer

[Errors as Values §2-3](../fundamentals/06-errors-as-values.md#2-sentinel-errors-and-errorsis) covers the mechanics. The architectural choice per layer:

**Domain layer — prefer typed errors when they carry data; sentinels for clean true/false conditions.**

```go
package user

// Sentinel: the caller just needs to know "yes, not found"
var ErrNotFound = errors.New("user not found")

// Typed: the caller wants the field name and value
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("user.%s: %s", e.Field, e.Message)
}
```

Callers downstream:

```go
err := userSvc.Get(ctx, id)
switch {
case errors.Is(err, user.ErrNotFound):
    return http.StatusNotFound, "user not found"
case errors.As(err, &validationErr):
    return http.StatusBadRequest, validationErr.Message
}
```

**Infrastructure layer — wrap the driver error and lift only the cases the domain cares about.**

```go
package userpg

func (r *Repo) Get(ctx context.Context, id string) (*user.User, error) {
    var u user.User
    err := r.db.QueryRowContext(ctx, q, id).Scan(&u.ID, &u.Name)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, user.ErrNotFound  // translate to domain sentinel
    }
    if err != nil {
        return nil, fmt.Errorf("userpg.Get id=%s: %w", id, err)
    }
    return &u, nil
}
```

The infrastructure layer is where translation happens. `sql.ErrNoRows` is a database concept — it should not leak into the HTTP handler. The repo translates it into `user.ErrNotFound`, which the domain layer recognizes.

**Transport layer — never define new error types. Map and return.**

```go
package httpapi

func (h *userHandler) get(w http.ResponseWriter, r *http.Request) {
    u, err := h.svc.Get(r.Context(), chi.URLParam(r, "id"))
    if err != nil {
        writeError(w, err)
        return
    }
    writeJSON(w, http.StatusOK, u)
}

func writeError(w http.ResponseWriter, err error) {
    switch {
    case errors.Is(err, user.ErrNotFound):
        http.Error(w, "not found", http.StatusNotFound)
    case errors.Is(err, context.Canceled):
        http.Error(w, "canceled", 499)  // nginx convention
    case errors.Is(err, context.DeadlineExceeded):
        http.Error(w, "timeout", http.StatusGatewayTimeout)
    default:
        log.Error("internal error", "err", err)
        http.Error(w, "internal error", http.StatusInternalServerError)
    }
}
```

Dave Cheney's "Don't just check errors, handle them gracefully" makes the case more strongly: callers should depend on **behavior**, not **identity**, where possible. In practice for backend services, sentinel-and-typed-errors with `errors.Is`/`errors.As` is the dominant style — but resist over-exporting sentinels. Each one is a public API commitment.

## 3. Wrapping with `%w` — when to wrap, when to bare-return, when to swap

The Go 1.13 release introduced `%w` and `errors.Is`/`errors.As`. The blog post is explicit about when *not* to wrap:

> When adding additional context to an error, either with `fmt.Errorf` or by implementing a custom type, you need to decide whether the new error should wrap the original. There is no single answer to this question; it depends on the context in which the new error is created. Wrap an error to expose it to callers. Do not wrap an error when doing so would expose implementation details.
>
> — https://go.dev/blog/go1.13-errors

Three patterns, each with its own use case:

**Wrap (`%w`) — the caller may want to inspect the original.**

```go
return fmt.Errorf("loadConfig: %w", err)
```

Use when the underlying error is meaningful — the caller might `errors.Is(err, os.ErrNotExist)` or `errors.As(err, &netErr)`. This is the default for crossing layer boundaries and for any error where the chain is informative.

**Bare-return — the layer adds no context; just propagate.**

```go
if err != nil {
    return err
}
```

Use when you're inside the same logical operation and wrapping would only repeat the function name. The caller already knows where they called from. Don't over-decorate.

**Swap (`%v` or new error) — hide the underlying detail.**

```go
return fmt.Errorf("invalid token: %v", err)
```

Use when the underlying error is an implementation detail you don't want to expose. Once you swap, callers can no longer `errors.Is`/`errors.As` against the inner error — that's the point. Common use case: an auth library that doesn't want callers branching on the parsing-library error type. The user just sees "invalid token".

A practical heuristic: **wrap by default, bare-return when nothing is added, swap when the inner error type is leaking**. The Go 1.13 blog post says the same thing in different words.

## 4. Mapping at the boundary

The transport layer is the **error mapping boundary**. By convention, exactly one place in the codebase decides "domain error X → HTTP status Y / gRPC code Z". Centralizing this:

- Lets you add a new error case in one place.
- Avoids divergence between routes that should return the same status for the same condition.
- Gives you a single hook for logging, metrics, and span attribute setting.

```go
// internal/httpapi/errmap.go
func httpStatus(err error) (int, string) {
    switch {
    case errors.Is(err, context.Canceled):
        return 499, "canceled"
    case errors.Is(err, context.DeadlineExceeded):
        return http.StatusGatewayTimeout, "timeout"
    case errors.Is(err, user.ErrNotFound), errors.Is(err, order.ErrNotFound):
        return http.StatusNotFound, "not found"
    case errors.Is(err, user.ErrUnauthorized):
        return http.StatusUnauthorized, "unauthorized"
    }
    var v *user.ValidationError
    if errors.As(err, &v) {
        return http.StatusBadRequest, v.Error()
    }
    return http.StatusInternalServerError, "internal error"
}
```

The gRPC equivalent uses the `status` package and `codes.NotFound` / `codes.InvalidArgument` / etc. The structure is identical.

This is the analogue of Spring's `@ControllerAdvice` + `@ExceptionHandler(UserNotFoundException.class)` mappings. The Go version is shorter because `switch errors.Is` is just code, not annotations and runtime reflection.

## 5. `errors.Join` and multi-errors

Go 1.20 introduced `errors.Join` for representing multiple errors as one. This matters in three places:

**Validation results.**

```go
func (u *User) Validate() error {
    var errs []error
    if u.Email == "" {
        errs = append(errs, &ValidationError{Field: "email", Message: "required"})
    }
    if len(u.Name) > 200 {
        errs = append(errs, &ValidationError{Field: "name", Message: "too long"})
    }
    return errors.Join(errs...)
}
```

`errors.Is` and `errors.As` walk the joined chain. Callers can extract the first matching `*ValidationError` or iterate (since Go 1.20's multi-Unwrap, `errors.Unwrap` returns nil for joined errors — the iteration goes through `Unwrap() []error` directly via reflection or via packages like `errors.Walk`).

**Cleanup chains.**

```go
func (s *Server) Shutdown(ctx context.Context) error {
    err1 := s.httpServer.Shutdown(ctx)
    err2 := s.queueConsumer.Stop(ctx)
    err3 := s.db.Close()
    return errors.Join(err1, err2, err3)
}
```

If any subsystem fails to close cleanly, you get all of them, not just the first.

**Concurrent operations** — though `golang.org/x/sync/errgroup` and the Go 1.22+ `Group.Wait()` typically returns the first error, not all of them. For "I want all the errors", explicit `errors.Join` after a `WaitGroup` is the pattern.

The Go 1.20 release notes also note that `fmt.Errorf` now accepts multiple `%w` verbs, which produces a multi-wrapped error.

## 6. `context.Canceled` and `context.DeadlineExceeded` as cross-cutting sentinels

Two sentinels are special because they describe the *requestor's* state, not the *operation's* state: `context.Canceled` (the caller went away) and `context.DeadlineExceeded` (the deadline elapsed). Both are returned via the context `ctx.Err()`, but most stdlib operations that take a `context.Context` will also return one of these as the operation's error when the context fires.

These should be **mapped specially at the transport boundary**:

- `context.Canceled` typically means the client closed the connection. There's no point logging at error level, and there's no point returning a 5xx (the client isn't going to see it). The nginx convention `499 Client Closed Request` is widespread.
- `context.DeadlineExceeded` means *your* deadline expired. Log it; emit a metric; return 504 (Gateway Timeout) or the gRPC equivalent (`codes.DeadlineExceeded`).

Both should be checked at the *top* of the error map, before any domain-specific cases, because the caller doesn't care that "user not found" failed — they care that they're gone.

A common mistake: writing `if err == context.Canceled` instead of `errors.Is(err, context.Canceled)`. Driver libraries (database/sql, http.Client) wrap these errors with their own framing. Always use `errors.Is`.

The peer doc [04-context-propagation.md](../concurrency/04-context-propagation.md) covers context plumbing in depth.

## 7. Logging vs returning — handle once

Dave Cheney's principle: **handle each error once**. If you log and return, the caller logs and re-returns, and the next caller logs and returns to the transport boundary, you'll see the same error three or four times in your logs with each line giving slightly different context — and no clean stack of the chain.

The rule:

- If you're going to **return** an error, do not log it. The caller will decide whether to log.
- If you've **logged** an error, do not return it. You've already handled it; pretending to return is dishonest.
- The **transport boundary** is the natural place to log. By the time the request handler is about to write a 500, the entire context chain is available.

```go
// BAD: logs at every level
func (s *Service) DoThing(ctx context.Context) error {
    if err := s.repo.Save(ctx); err != nil {
        log.Error("save failed", "err", err)  // first log
        return fmt.Errorf("DoThing: %w", err)
    }
    return nil
}

// In the handler
if err := svc.DoThing(ctx); err != nil {
    log.Error("DoThing failed", "err", err)  // duplicate log
    http.Error(w, "internal error", 500)
}
```

```go
// GOOD: only the transport boundary logs
func (s *Service) DoThing(ctx context.Context) error {
    if err := s.repo.Save(ctx); err != nil {
        return fmt.Errorf("DoThing: %w", err)
    }
    return nil
}

// In the handler
if err := svc.DoThing(ctx); err != nil {
    log.Error("request failed",
        "path", r.URL.Path,
        "err", err)  // single log, full chain
    writeError(w, err)
}
```

The exception is goroutines you spawn that aren't tied to a request — those have no caller, so they must log themselves before exiting (and ideally also report a metric).

## 8. Stack traces — stdlib doesn't give them

Go's stdlib `errors` package does **not** capture stack traces. `errors.New("foo")` records the message; `fmt.Errorf("%w: ...", err)` records the chain. Neither remembers the file and line where the error was created.

This is deliberate — stack capture is expensive (the JVM and V8 both pay it on every exception construction), and Go's "errors are just values" philosophy says the caller's `if err != nil` should already give you the location. The `runtime/debug.Stack()` function gives you a stack trace from where you call it.

For services where the chain alone isn't enough — typically anywhere a single error message can be triggered from many call sites — there are two community options:

**`pkg/errors` (deprecated but widespread).**
Old, simple, attaches a stack trace via `errors.Wrap`. The repo is in maintenance mode but still imported by huge swathes of the ecosystem. Predates Go 1.13's `%w`.

**`github.com/cockroachdb/errors`.**
Active, drop-in replacement for `pkg/errors` and stdlib `errors`. Beyond stack traces, it adds:

- Network portability (errors can be encoded/decoded via protobuf for transmission across services and recognized identically on the other side).
- PII-safe redaction for error reporting.
- Native Sentry integration.
- Hints, details, issue links, and telemetry keys attached to errors.

For a typical Go backend service, the call is:

| Need | Choice |
|---|---|
| Lightweight, monolithic service, errors live and die in one process | Stdlib + good wrap messages; skip stack traces |
| Distributed system, errors cross gRPC boundaries, want consistent identity downstream | `cockroachdb/errors` |
| Existing codebase already on `pkg/errors` | Migrate to stdlib if the chain is enough; otherwise migrate to `cockroachdb/errors` |

You can also implement your own minimal stack-capture wrapper in 30 lines using `runtime.Callers` if you want full control.

## 9. OpenTelemetry conventions for errors

When errors cross the transport boundary, the trace span should know about them. The OpenTelemetry semantic conventions for exceptions specify:

- The span event name is `exception`.
- Required attribute: at least one of `exception.type` (e.g., `"*user.ValidationError"`) or `exception.message` (the error string).
- Recommended: `exception.stacktrace` if available, in the language's natural format.

The convention has been migrating from span events to log records — newer guidance recommends recording exceptions on **logs** rather than spans, with an opt-in environment variable (`OTEL_SEMCONV_EXCEPTION_SIGNAL_OPT_IN`) for transitional dual emission.

For Go, the practical wiring at the transport boundary:

```go
import "go.opentelemetry.io/otel/codes"

func (h *userHandler) get(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    span := trace.SpanFromContext(ctx)

    u, err := h.svc.Get(ctx, chi.URLParam(r, "id"))
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        writeError(w, err)
        return
    }
    writeJSON(w, http.StatusOK, u)
}
```

`span.RecordError(err)` adds the exception event with `exception.type` and `exception.message` derived from the error; `span.SetStatus(codes.Error, ...)` marks the span itself as failed (visible in trace UIs as a red span).

Note: don't record `context.Canceled` as a span error. It's not really an error — it's the client leaving — and tagging it red pollutes your error rate metrics. Filter at the same point where you map to HTTP 499.

## Related

- [Errors as Values](../fundamentals/06-errors-as-values.md) — language-level mechanics
- [Idiomatic Project Layout](01-idiomatic-project-layout.md) — where domain/infra/transport packages live
- [Hexagonal Architecture in Go](03-hexagonal-architecture.md) — error mapping at adapter seams
- [Context Propagation](../concurrency/04-context-propagation.md) — `ctx.Err()` plumbing
- [TypeScript Error Handling Architecture](../../typescript/patterns/error-handling-architecture.md) — Result types, error mapping in TS
- [System Design Index](../../system-design/INDEX.md) — error budgets, SLOs, retries

## References

- "Error handling and Go" (2011, original) — https://go.dev/blog/error-handling-and-go
- "Working with Errors in Go 1.13" — https://go.dev/blog/go1.13-errors
- Go 1.20 release notes (`errors.Join`, multi-Unwrap, multi-`%w`) — https://go.dev/doc/go1.20
- `errors` package — https://pkg.go.dev/errors
- `context` package, `Canceled` / `DeadlineExceeded` — https://pkg.go.dev/context
- `cockroachdb/errors` — https://github.com/cockroachdb/errors
- Dave Cheney, "Don't just check errors, handle them gracefully" — https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully
- OpenTelemetry semantic conventions for exceptions on spans — https://opentelemetry.io/docs/specs/semconv/exceptions/exceptions-spans/
