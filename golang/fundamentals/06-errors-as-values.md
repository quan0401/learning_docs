---
title: "Errors as Values"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, fundamentals, errors, panic, recover, error-wrapping]
---

# Errors as Values

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `fundamentals` `errors` `panic` `recover` `error-wrapping`

---

## Table of Contents

1. [`error` is just an interface](#1-error-is-just-an-interface)
2. [Sentinel errors and `errors.Is`](#2-sentinel-errors-and-errorsis)
3. [Typed errors and `errors.As`](#3-typed-errors-and-errorsas)
4. [Wrapping with `%w` and unwrapping](#4-wrapping-with-w-and-unwrapping)
5. [`panic` and `recover`: not exceptions](#5-panic-and-recover-not-exceptions)
6. [Recovery boundaries in real services](#6-recovery-boundaries-in-real-services)
7. [Comparison with TS try/catch and Java exceptions](#7-comparison-with-ts-trycatch-and-java-exceptions)

## Summary

Go has no exceptions. Errors are ordinary values, returned alongside the success value, and the caller is required to handle (or explicitly ignore) them. The `error` type is a one-method interface. The standard library gives you three building blocks: `errors.New` and `fmt.Errorf` for creating errors, `errors.Is` and `errors.As` for inspecting them, and the `%w` verb for wrapping a cause inside a higher-level error.

`panic` and `recover` exist but are not the analogue of exceptions. They are reserved for unrecoverable program states (nil dereference, index out of range, contract violations) and one specific recoverable boundary use case: keeping a long-running process alive when a single request handler panics.

## 1. `error` is just an interface

The `error` type is defined in the builtin package as:

```go
type error interface {
    Error() string
}
```

Any type with an `Error() string` method satisfies the interface. The simplest construction:

```go
import "errors"

func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("divide by zero")
    }
    return a / b, nil
}
```

`errors.New` returns a value of an unexported type `*errorString` that holds the message. For most one-shot errors, `fmt.Errorf` is more useful because it does formatting:

```go
return 0, fmt.Errorf("divide by zero (a=%v)", a)
```

Calling convention: errors are the **last** return value. By convention, when `err != nil`, the other return values are zero-valued and should not be used. There are exceptions — `io.Reader.Read` legitimately returns `n > 0` with a non-nil error when partial data is followed by EOF — but those exceptions are documented per-function.

## 2. Sentinel errors and `errors.Is`

A **sentinel error** is a singleton error value exported by a package, used by callers to distinguish a specific failure mode:

```go
// Standard library examples
var (
    io.EOF                = errors.New("EOF")
    sql.ErrNoRows         = errors.New("sql: no rows in result set")
    os.ErrNotExist        = errors.New("file does not exist")
    context.Canceled      = errors.New("context canceled")
)
```

Callers compare with `errors.Is`:

```go
n, err := r.Read(buf)
if errors.Is(err, io.EOF) {
    // end of stream — not really an error in the application sense
}
```

`errors.Is` walks the wrapped-error chain (see §4) so that wrapped sentinels are still recognized:

```go
err := fmt.Errorf("read config: %w", io.EOF)
errors.Is(err, io.EOF)  // true — even though the message is "read config: EOF"
```

Compared with `==`:
- `err == io.EOF` works only for **un-wrapped** sentinels. As soon as anyone in the chain calls `fmt.Errorf("...: %w", err)`, the equality fails.
- `errors.Is(err, io.EOF)` works for both wrapped and un-wrapped.

The convention since Go 1.13 (which introduced `errors.Is` / `errors.As` / `%w`) is: **always use `errors.Is`, never `==`** for sentinel comparison. The lone exception is `errors.Is(err, nil)`, which is just `err == nil`.

A sentinel error is the simplest form of "typed" error in Go. Use one when callers need to recognize a specific condition by identity but don't need to extract data from the failure.

## 3. Typed errors and `errors.As`

When the failure carries data — a status code, a missing field name, a row ID — define a struct type that implements `error`:

```go
type NotFoundError struct {
    Resource string
    ID       string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s not found: %s", e.Resource, e.ID)
}
```

Callers extract with `errors.As`, which walks the wrapped-error chain looking for a value assignable to the target type:

```go
err := userService.Get(ctx, "u_123")

var nf *NotFoundError
if errors.As(err, &nf) {
    log.Printf("missing %s id=%s", nf.Resource, nf.ID)
    return http.StatusNotFound
}
```

`errors.As` modifies `nf` to point at the matching error in the chain, and returns `true`. If no match, `nf` is unchanged and the function returns `false`.

A common pattern is to expose a constructor and have the caller use a sentinel-style check or an `errors.As` extraction depending on whether they need the data:

```go
package userrepo

type ErrNotFound struct { ID string }
func (e *ErrNotFound) Error() string { return "user not found: " + e.ID }

// Sometimes both are useful
var ErrUnauthorized = errors.New("unauthorized")
```

The TypeScript and Java analogues — `instanceof` checks, exception class hierarchies — are mechanically similar. The difference is that in Go, the check is **at the call site** rather than in a `catch` block, and the failure path is a normal control flow with `return`.

## 4. Wrapping with `%w` and unwrapping

`fmt.Errorf` with the `%w` verb produces an error that wraps another error as its **cause**:

```go
func loadConfig(path string) (*Config, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, fmt.Errorf("loadConfig: open %s: %w", path, err)
    }
    defer f.Close()
    // ...
}
```

The result is an error whose `Error()` method returns the full formatted message:

```text
loadConfig: open /etc/myapp.toml: open /etc/myapp.toml: no such file or directory
```

…and whose **chain** is walked by `errors.Is` and `errors.As`:

```go
err := loadConfig("/etc/myapp.toml")
errors.Is(err, os.ErrNotExist)  // true — os.PathError wraps a fs.ErrNotExist sentinel
```

Two rules of thumb:

**Wrap when crossing a layer boundary.** If your service layer catches an error from the repository layer, wrap with context (`"loadUser: %w"`) so the trace tells the caller where it failed. If you're inside the same layer and just bubbling up, `return err` directly is fine.

**Don't wrap if you're going to log and stop.** Wrapping is for callers who will continue inspecting the error. If you're at the top of a request handler about to return a 500, just log the error — wrapping further serves no one.

To unwrap manually: `errors.Unwrap(err)` returns the wrapped error or `nil`. You almost never need this directly — `errors.Is` and `errors.As` do their own walking.

A type can participate in unwrapping by implementing `Unwrap() error` (single-cause) or `Unwrap() []error` (multi-cause, since Go 1.20):

```go
type MultiError struct { errs []error }
func (m *MultiError) Error() string { /* ... */ }
func (m *MultiError) Unwrap() []error { return m.errs }
```

`errors.Join(errs...)` is the standard-library helper for combining errors; the result satisfies `Unwrap() []error` automatically.

## 5. `panic` and `recover`: not exceptions

`panic(v)` aborts normal execution. The runtime unwinds the stack, running `defer`red functions on the way up, until either the program crashes (printing the panic value and a stack trace) or a `defer`red function calls `recover()`.

`recover()` is valid only inside a deferred function. It returns the panic value if a panic is in flight, or `nil` otherwise.

```go
func safeCall(f func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    f()
    return nil
}
```

What panics:
- Index out of range, nil dereference, divide by zero — runtime panics.
- Closing a closed channel, sending on a closed channel — runtime panics.
- Explicit `panic(...)` calls.

What does **not** panic in idiomatic Go:
- Anything you'd expect to be an exception in Java: file not found, network errors, malformed input, business-rule violations. Those are returned errors.

`panic` is for genuinely unrecoverable states — situations where continuing execution would be wrong, not just inconvenient. The standard library uses `panic` extensively for programmer-error conditions (`regexp.MustCompile` panics on invalid regexes — the assumption is you wrote the regex literal, you control it, an invalid regex is a bug not a runtime condition).

The dual `Must<X>` convention exists across the standard library: a normal version returns an error, a `Must` version panics on failure for use during initialization.

```go
re := regexp.MustCompile(`^\d{4}-\d{2}-\d{2}$`)  // panics if pattern is bad
re, err := regexp.Compile(userPattern)            // returns error
```

## 6. Recovery boundaries in real services

The one production-relevant use of `recover` is the **request boundary** in long-running servers. If a single request handler panics (because of a bug, or a malformed input you didn't validate), you want to:

1. Return a 500 to the client, not crash the process.
2. Log the panic with a stack trace.
3. Increment a metric so you notice.

The standard library's `net/http` does this for you — every HTTP handler runs inside a `recover` boundary in `(*conn).serve`, so a panic in your handler returns a 500 instead of crashing the server. Your application code doesn't need to add its own.

Where you **do** need it:
- Goroutines you spawn that aren't tied to a request (background workers, queue consumers). A panic in a goroutine that no one is recovering crashes the process. Add a `defer recover()` at the top of the goroutine.
- Plugins or untrusted code (rare).

```go
func startWorker(ctx context.Context, jobs <-chan Job) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("worker panic: %v\n%s", r, debug.Stack())
                metrics.PanicCount.Inc()
                // restart the worker (recursive call or send a signal)
            }
        }()
        for {
            select {
            case <-ctx.Done():
                return
            case j := <-jobs:
                process(j)
            }
        }
    }()
}
```

The Go FAQ explicitly warns against using panic for control flow:

> The convention in the Go libraries is that even when a package uses panic internally, its external API still presents explicit error return values.
>
> — https://go.dev/doc/faq#exceptions

## 7. Comparison with TS try/catch and Java exceptions

| | TypeScript / Node | Java | Go |
|---|---|---|---|
| Failure mechanism | thrown exceptions (`throw`) | thrown exceptions (`throw new ...`) | returned values |
| Unhandled failures | `unhandledRejection` / process crash | uncaught exception → thread death | unchecked → silently dropped (you must check) |
| Forced handling | no | checked exceptions (`throws`) — controversial | linter / convention; `errcheck` and `go vet` catch many cases |
| Stack trace | yes, attached to exception | yes | no, by default — wrap with libraries like `pkg/errors` or use `runtime/debug.Stack()` |
| Cost | exception construction is expensive (V8 stack capture) | exception is expensive (JVM stack capture) | error construction is cheap (just a struct) |
| Unwinding | exception unwinds the stack | exception unwinds the stack | no unwinding — explicit return at every level |

The Go model has costs: you write `if err != nil` a lot. Estimates from large Go codebases put error-handling lines at 20–30% of the total. The Go team has explored alternatives (notably the `try` proposal, withdrawn in 2019) and concluded that the verbosity is the price of explicitness — every `if err != nil` is a place where a reader can see exactly what is being checked and where the failure goes.

The benefit: there is no surprise control flow. Every place that can return an error is visible. There is no equivalent of catching `RuntimeException` in Java and finding that you've been silencing real bugs for a year.

## Related

- [Interfaces & Structural Typing](05-interfaces-and-structural-typing.md) — `error` is the canonical small interface
- [Go Tour for TS + Java Developers](01-go-tour-for-ts-java-devs.md) — multiple return values, `defer`
- [Pointers, Methods & Receivers](07-pointers-methods-receivers.md) — pointer receivers on error types (typed-nil pitfall)
- [TypeScript Error Handling Architecture](../../typescript/patterns/error-handling-architecture.md) — Result types and error mapping in TS
- [Java Coding Style](../../java/) — exception conventions on the Java side

## References

- The Go Programming Language Specification, "Errors" — https://go.dev/ref/spec#Errors
- "Error handling and Go" (the original 2011 post) — https://go.dev/blog/error-handling-and-go
- "Working with Errors in Go 1.13" (introduced `errors.Is`, `errors.As`, `%w`) — https://go.dev/blog/go1.13-errors
- Go FAQ, "Why does Go not have exceptions?" — https://go.dev/doc/faq#exceptions
- Go FAQ, "When should I use panic?" — https://go.dev/doc/faq#use_panic
- Go 1.20 release notes — multi-error `errors.Join` and `Unwrap() []error` — https://go.dev/doc/go1.20#errors
- The withdrawn `try` proposal — https://github.com/golang/go/issues/32437 (background reading on why Go's verbose error handling stays verbose)
- `errcheck` linter — https://github.com/kisielk/errcheck
- `runtime/debug.Stack` — https://pkg.go.dev/runtime/debug#Stack
