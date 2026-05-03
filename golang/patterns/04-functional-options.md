---
title: "The Functional Options Pattern"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, patterns, functional-options, api-design]
---

# The Functional Options Pattern

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `patterns` `functional-options` `api-design`

---

## Table of Contents

1. [The problem: no defaults, no overloading](#1-the-problem-no-defaults-no-overloading)
2. [The pattern](#2-the-pattern)
3. [Worked example: an HTTP client constructor](#3-worked-example-an-http-client-constructor)
4. [Required vs optional split](#4-required-vs-optional-split)
5. [Defaults applied before options](#5-defaults-applied-before-options)
6. [Variations: typed `Option` interface for error handling](#6-variations-typed-option-interface-for-error-handling)
7. [Comparison with the builder pattern](#7-comparison-with-the-builder-pattern)
8. [Real examples in the wild](#8-real-examples-in-the-wild)
9. [When NOT to use functional options](#9-when-not-to-use-functional-options)

## Summary

Go has no default arguments and no function overloading. A function declared `NewClient(addr string)` cannot also be called as `NewClient(addr, timeout)` — the language won't allow it. Adding a configuration parameter is a breaking change to the call signature. The community answer is the **functional options pattern**: take a variadic of "option functions" that mutate a configuration struct, and apply each one inside the constructor.

The pattern was crystallized by Rob Pike's 2014 blog post on self-referential functions and Dave Cheney's "Functional options for friendly APIs" later the same year. It is now used throughout the Go ecosystem — gRPC, OpenTelemetry, AWS SDK v2, prominent client libraries — as the default constructor shape for any type with more than two or three optional knobs.

## 1. The problem: no defaults, no overloading

In TypeScript:

```ts
function newClient(addr: string, options?: { timeout?: number; retries?: number }) { ... }
```

In Java:

```java
public Client(String addr) { this(addr, Duration.ofSeconds(30), 3); }
public Client(String addr, Duration timeout) { ... }
public Client(String addr, Duration timeout, int retries) { ... }
```

Or in Java with the builder pattern:

```java
Client.newBuilder().addr("foo").timeout(Duration.ofSeconds(30)).retries(3).build();
```

Go has none of these. Every parameter is required and named at the call site:

```go
// Hypothetical — bad
func NewClient(addr string, timeout time.Duration, retries int, ...) *Client
```

This puts you in a corner the moment you want to add a new optional knob. Three real-world responses, in roughly increasing quality:

**1. Long parameter list.** Adds a parameter every time. Every existing call site breaks.

**2. Config struct passed by value.**
```go
type ClientConfig struct {
    Timeout time.Duration
    Retries int
}
func NewClient(addr string, cfg ClientConfig) *Client
```
Better — adding a field doesn't break callers. But every caller must construct a `ClientConfig`, even if they only want the defaults: `NewClient(addr, ClientConfig{})`. The empty struct call is awkward and zero-values may be wrong defaults (a `Timeout: 0` that means "no timeout" is a sharp edge).

**3. Functional options.** What this doc is about.

Dave Cheney's framing of why option structs are insufficient:

> ... the ambiguity around defaults [is a real problem]. Callers must pass *something*, often nil, even for basic usage. ... The real subtle problem is that callers might retain a reference to that configuration struct, and modify it after construction.
>
> — https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis (paraphrased)

## 2. The pattern

The structural shape:

```go
type Client struct {
    addr    string
    timeout time.Duration
    retries int
    logger  *slog.Logger
}

// Option is a function that configures a Client.
type Option func(*Client)

func WithTimeout(d time.Duration) Option {
    return func(c *Client) { c.timeout = d }
}

func WithRetries(n int) Option {
    return func(c *Client) { c.retries = n }
}

func WithLogger(l *slog.Logger) Option {
    return func(c *Client) { c.logger = l }
}

// Constructor: required args positional, options variadic.
func NewClient(addr string, opts ...Option) *Client {
    // 1. Start with defaults.
    c := &Client{
        addr:    addr,
        timeout: 30 * time.Second,
        retries: 3,
        logger:  slog.Default(),
    }
    // 2. Apply each option in order.
    for _, opt := range opts {
        opt(c)
    }
    return c
}
```

Call sites:

```go
// Default everything
c := NewClient("https://api.example.com")

// One override
c := NewClient("https://api.example.com", WithTimeout(5*time.Second))

// Several
c := NewClient("https://api.example.com",
    WithTimeout(5*time.Second),
    WithRetries(0),
    WithLogger(myLogger),
)
```

The shape gives you:

- **Backward compatibility.** Adding `WithCircuitBreaker(...)` doesn't break any existing call.
- **Clarity at the call site.** `WithTimeout(5*time.Second)` says exactly what's happening.
- **Compile-time safety.** No string keys, no runtime "unknown option" errors.
- **Documentation per option.** Each `With...` function gets its own doc comment, surfaces in `go doc`, IDE hover.
- **Composability.** Options can call each other or be conditionally added: `if debug { opts = append(opts, WithLogger(devLogger)) }`.

## 3. Worked example: an HTTP client constructor

A more realistic shape:

```go
package httpclient

import (
    "log/slog"
    "net/http"
    "time"
)

type Client struct {
    base       string
    httpClient *http.Client
    timeout    time.Duration
    retries    int
    headers    http.Header
    logger     *slog.Logger
}

type Option func(*Client)

// WithHTTPClient overrides the underlying *http.Client.
// Useful for injecting custom transport (e.g., for tests, for mTLS).
func WithHTTPClient(h *http.Client) Option {
    return func(c *Client) { c.httpClient = h }
}

// WithTimeout sets the per-request timeout.
// Default: 30 seconds.
func WithTimeout(d time.Duration) Option {
    return func(c *Client) { c.timeout = d }
}

// WithRetries configures the number of retry attempts on transient errors.
// Default: 3. Pass 0 to disable retries.
func WithRetries(n int) Option {
    return func(c *Client) { c.retries = n }
}

// WithHeader adds a default header sent on every request.
// May be called multiple times to add multiple headers.
func WithHeader(key, value string) Option {
    return func(c *Client) {
        if c.headers == nil { c.headers = http.Header{} }
        c.headers.Add(key, value)
    }
}

// WithLogger sets the structured logger.
// Default: slog.Default().
func WithLogger(l *slog.Logger) Option {
    return func(c *Client) { c.logger = l }
}

func New(base string, opts ...Option) *Client {
    c := &Client{
        base:    base,
        timeout: 30 * time.Second,
        retries: 3,
        logger:  slog.Default(),
    }
    for _, opt := range opts {
        opt(c)
    }
    if c.httpClient == nil {
        c.httpClient = &http.Client{Timeout: c.timeout}
    }
    return c
}
```

Note `WithHeader` is callable repeatedly — that's idiomatic for additive options. The `WithHTTPClient` option wins over `WithTimeout` if both are passed (the timeout was used to build the default client, so an explicit override replaces it). These ordering rules deserve doc comments — write them.

## 4. Required vs optional split

Required parameters stay positional; optional ones go through options.

```go
// addr is required — no default makes sense.
func NewClient(addr string, opts ...Option) *Client

// db is required — a Repo with no DB is meaningless.
func NewRepo(db *sql.DB, opts ...Option) *Repo

// Multiple required args are still positional.
func NewProcessor(in <-chan Job, out chan<- Result, opts ...Option) *Processor
```

The rule of thumb: if there is no sensible default, the parameter is required and positional. If there is a sensible default — even one chosen for the 90% case — it's an option.

The trap to avoid: making a "required" thing optional via `WithAddr(...)`. If a `Client` cannot work without an address, do not let the caller forget to set it. Take it positionally, fail fast.

## 5. Defaults applied before options

The order matters:

```go
func NewClient(addr string, opts ...Option) *Client {
    // 1. Construct with defaults.
    c := &Client{addr: addr, timeout: 30 * time.Second, retries: 3}
    // 2. Apply each option, in order.
    for _, opt := range opts {
        opt(c)
    }
    // 3. Final validation / derivation.
    if c.httpClient == nil {
        c.httpClient = &http.Client{Timeout: c.timeout}
    }
    return c
}
```

Why this order:

- Defaults first means the option function sees a coherent struct, not a zero-value one. `WithLogger` can decorate the existing logger if needed.
- Options run in **call order**, so later options win for the same field. `WithTimeout(5*time.Second)` followed by `WithTimeout(10*time.Second)` results in 10 seconds. If you want first-wins, document it.
- Final derivation runs after options so that derived fields (the `*http.Client` built from `timeout`) reflect the user's customization.

If an option needs to validate and fail (e.g., `WithRetries(-1)` should be rejected), use the typed-Option variant in §6.

## 6. Variations: typed `Option` interface for error handling

The function-form `Option = func(*T)` cannot return an error. For options that can fail validation, two variations exist:

**Option as an interface with `apply(*T) error`.**

```go
type Option interface {
    apply(*Client) error
}

type optionFunc func(*Client) error
func (f optionFunc) apply(c *Client) error { return f(c) }

func WithTimeout(d time.Duration) Option {
    return optionFunc(func(c *Client) error {
        if d < 0 {
            return fmt.Errorf("timeout must be non-negative, got %v", d)
        }
        c.timeout = d
        return nil
    })
}

func New(addr string, opts ...Option) (*Client, error) {
    c := &Client{addr: addr, timeout: 30 * time.Second}
    for _, opt := range opts {
        if err := opt.apply(c); err != nil {
            return nil, err
        }
    }
    return c, nil
}
```

Now `New` returns `(*Client, error)` and bad options surface at construction time.

**Option as a function returning error directly.**

```go
type Option func(*Client) error
```

Same idea, less ceremony. Many real codebases use this.

The trade-off: typed/error-returning options give you validation; plain `func(*Client)` options are simpler. For most types the simple form is enough — bad arguments tend to manifest at usage time anyway, and you can `panic` on truly programmer-error inputs (negative timeout) just as the standard library does.

A third minor variation worth knowing: **self-referential options** (Rob Pike's original). The option function returns its inverse — the option that would undo what it just did. This is rare but appears in the standard library `expvar` and in some testing utilities.

## 7. Comparison with the builder pattern

| | Builder (Java/Lombok) | Functional options (Go) |
|---|---|---|
| Define a knob | `setTimeout(d)` method on builder | `WithTimeout(d) Option` function |
| Call site | `.timeout(d).retries(n).build()` | `WithTimeout(d), WithRetries(n)` |
| Construction step | `.build()` produces the object | constructor takes options directly |
| Validation | typically inside `build()` | inside option functions or constructor |
| Mutability after construction | usually no (builder is the mutable thing) | no (options applied during construction only) |
| Backward compat when adding a knob | yes | yes |
| Default-only construction | `Client.newBuilder().build()` (verbose) | `New(addr)` (clean) |
| Doc per knob | yes | yes |

For Go specifically, functional options have one ergonomic edge: the default-only path is a single normal function call, not a builder ritual. `New(addr)` is shorter than `Client.newBuilder().addr(addr).build()`.

For some constructors — when there's a strong "construct then verify" semantics (e.g., a SQL query builder) — the builder pattern fits better. For "create configured object", functional options usually wins in Go.

## 8. Real examples in the wild

**gRPC `ServerOption`.**

```go
server := grpc.NewServer(
    grpc.UnaryInterceptor(myUnaryInterceptor),
    grpc.MaxConcurrentStreams(100),
    grpc.Creds(creds),
)
```

`grpc.ServerOption` is an interface (the typed-option variant from §6) — gRPC needs this because some options are mutually exclusive or must run in a specific order, and the interface form gives them flexibility.

**OpenTelemetry SDK.**

```go
tp := sdktrace.NewTracerProvider(
    sdktrace.WithSampler(sdktrace.AlwaysSample()),
    sdktrace.WithBatcher(exporter),
    sdktrace.WithResource(resource),
)
```

Each OTel SDK constructor (`TracerProvider`, `MeterProvider`, batchers, exporters) uses functional options. The Trace SDK alone exposes 8+ option functions on `TracerProvider`.

**AWS SDK v2.**

```go
cfg, err := config.LoadDefaultConfig(ctx,
    config.WithRegion("us-west-2"),
    config.WithRetryer(func() aws.Retryer { return retry.NewStandard() }),
)
```

The whole v2 SDK was rewritten around functional options — v1 used struct configs and was much harder to extend without breakage.

**Counter-example: `net/http.Server`.** The standard library's `http.Server` uses public struct fields, not options:

```go
s := &http.Server{
    Addr:           ":8080",
    Handler:        router,
    ReadTimeout:    10 * time.Second,
    WriteTimeout:   10 * time.Second,
    MaxHeaderBytes: 1 << 20,
}
```

This is the older Go idiom — predates the functional options pattern's popularization. It works fine because:

- `http.Server` was designed when the patterns hadn't crystallized yet.
- Public fields are simple and the type is small enough that exposing them isn't painful.
- Setting `Server{}` to the zero value gives sensible-ish defaults.

You'll still see this style in stdlib types (`tls.Config`, `tar.Header`). It's not wrong; it just wouldn't be the choice today for a new library.

## 9. When NOT to use functional options

Functional options are not free — every option is a public function in your API, and every option is one more thing readers have to look at. Don't reach for them when:

**The struct has only required fields.** A `type Point struct { X, Y float64 }` should be constructed with a struct literal `Point{1, 2}` or a simple `NewPoint(1, 2)`. No options needed.

**There are only one or two optional knobs and they're cheap to express.** A `NewLogger(level Level)` where `Level` has a sensible default (`InfoLevel`) might just expose two constructors: `NewLogger()` and `NewLoggerWithLevel(level)`. Or take the level as a positional argument with `LevelInfo` as a documented common value. Functional options for one knob is over-engineering.

**The configuration is genuinely complex and needs a builder.** Query builders, expression trees, and chained-call DSLs (`gorm`, `squirrel`) read better as fluent builders.

**You're modeling immutable value types.** A `time.Duration` doesn't need options — it's just a value. A `Money{Amount, Currency}` is the same.

**The struct is for reading data, not configuration.** A response DTO with public fields is fine; don't build options for it.

The mental check: *would adding a new optional knob require me to break callers?* If yes, functional options solve a real problem. If no, you're decorating something that didn't need it.

## Related

- [Idiomatic Project Layout](01-idiomatic-project-layout.md) — package design, where constructors live
- [Hexagonal Architecture in Go](03-hexagonal-architecture.md) — adapter constructors as a heavy use case for options
- [Errors & Domain Modeling](02-errors-and-domain-modeling.md) — error-returning option patterns
- [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) — `Option` as an interface
- [Pointers, Methods & Receivers](../fundamentals/07-pointers-methods-receivers.md) — why options take `*T`
- [Low-Level Design Index](../../low-level-design/INDEX.md) — pattern landscape

## References

- Rob Pike, "Self-referential functions and the design of options" (2014) — https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html
- Dave Cheney, "Functional options for friendly APIs" (2014) — https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis
- gRPC `ServerOption` (worked example) — https://pkg.go.dev/google.golang.org/grpc#ServerOption
- OpenTelemetry Go SDK options (worked example) — https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace
- `net/http.Server` (counter-example with public fields) — https://pkg.go.dev/net/http#Server
- Effective Go — https://go.dev/doc/effective_go
