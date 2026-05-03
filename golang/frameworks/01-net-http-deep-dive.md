---
title: "`net/http` Deep Dive"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, frameworks, net-http, server, middleware]
---

# `net/http` Deep Dive

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `frameworks` `net-http` `server` `middleware`

---

## Table of Contents

1. [`Handler` is a one-method interface](#1-handler-is-a-one-method-interface)
2. [`HandlerFunc`, `HandleFunc`, and `ServeMux`](#2-handlerfunc-handlefunc-and-servemux)
3. [Go 1.22 ServeMux: method patterns and wildcards](#3-go-122-servemux-method-patterns-and-wildcards)
4. [Middleware as Handler-wrapping functions](#4-middleware-as-handler-wrapping-functions)
5. [The `Server` struct and the timeouts that matter](#5-the-server-struct-and-the-timeouts-that-matter)
6. [Graceful shutdown — the SIGTERM dance](#6-graceful-shutdown--the-sigterm-dance)
7. [Request body limits and JSON decoding](#7-request-body-limits-and-json-decoding)
8. [`ResponseWriter`: WriteHeader, Hijacker, Flusher, Pusher](#8-responsewriter-writeheader-hijacker-flusher-pusher)
9. [Request context: `r.Context()` and request-scoped values](#9-request-context-rcontext-and-request-scoped-values)
10. [When you actually need a framework](#10-when-you-actually-need-a-framework)

## Summary

Go's `net/http` is a complete production HTTP server in the standard library. Unlike Node's `http` module — which is essentially a low-level building block — `net/http` ships with a router (`ServeMux`), a server with full lifecycle management (`http.Server`), and the conventions that make middleware composition trivial. Since Go 1.22 the built-in `ServeMux` supports method-aware patterns and path wildcards, narrowing the gap with third-party routers further.

The mental shift from Express or NestJS is that **everything is a `Handler`**. There is no special "controller" or "middleware" type — both are `Handler` values. A middleware is just a function that takes a `Handler` and returns a `Handler`. There is no DI container, no decorator metadata, no framework lifecycle. The whole server fits in your head.

The flip side is that production-readiness lives in defaults you must set yourself. `http.ListenAndServe(":8080", nil)` is convenient; it is also dangerous — no header timeout means a single Slowloris client can exhaust your file-descriptor budget. The deep dive below is half about what `net/http` gives you for free, and half about what you must configure on every real server.

## 1. `Handler` is a one-method interface

The whole package builds on this:

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

Any type with that one method is a handler. The router (`ServeMux`) is itself a `Handler` (it dispatches to other handlers based on the path). The server takes a `Handler` and calls `ServeHTTP` for every accepted request. There is nothing else.

```go
type pingHandler struct{}

func (pingHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("pong"))
}

http.ListenAndServe(":8080", pingHandler{})
```

This is the same one-method-interface pattern walked through in [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md). `pingHandler` satisfies `http.Handler` implicitly — no `implements` keyword.

A goroutine is spawned per request — see [Goroutines & the Scheduler](../concurrency/01-goroutines-and-scheduler.md). Inside `ServeHTTP` you are running on your own goroutine, you can block, and other requests are unaffected. This is the inverse of the Node single-threaded event loop: blocking is fine, parallelism is the default.

## 2. `HandlerFunc`, `HandleFunc`, and `ServeMux`

Defining a struct just to attach a method is verbose. The standard library provides an adapter:

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) { f(w, r) }
```

`HandlerFunc` is a function type with a `ServeHTTP` method on it. Wrapping a plain function in `http.HandlerFunc(fn)` lifts it into a `Handler` without writing a struct. This is the canonical Go trick for "interface with one method, but I want to write a function literal" — see also `sort.Slice` and `http.RoundTripper`.

`http.ServeMux` is the built-in router. It maps path patterns to handlers:

```go
mux := http.NewServeMux()
mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("ok"))
})
mux.Handle("/api/", apiHandler) // a Handler value, mounted as a subtree
```

`http.HandleFunc` (no receiver) registers on the package-level `DefaultServeMux`. Avoid it in production — global mutable state in tests is a footgun. Always make your own `mux`.

## 3. Go 1.22 ServeMux: method patterns and wildcards

Pre-1.22, the standard `ServeMux` was the reason most teams reached for `chi` or `gin`. It matched on path only — no method awareness, no path parameters, no precedence rules. Go 1.22 added all three.

**Method patterns** — bind a handler to one HTTP method:

```go
mux.HandleFunc("GET /posts/{id}", showPost)
mux.HandleFunc("POST /posts",     createPost)
mux.HandleFunc("DELETE /posts/{id}", deletePost)
```

A request with the wrong method on a registered path now gets `405 Method Not Allowed` automatically — no more handlers that switch on `r.Method` themselves.

**Single-segment wildcards** — bound path values you read with `r.PathValue`:

```go
mux.HandleFunc("GET /users/{user}/posts/{id}", func(w http.ResponseWriter, r *http.Request) {
    user := r.PathValue("user")
    id   := r.PathValue("id")
    // ...
})
```

**Multi-segment wildcards** — match the rest of the path:

```go
mux.HandleFunc("GET /files/{path...}", serveFile)
// /files/2024/q1/report.pdf → r.PathValue("path") == "2024/q1/report.pdf"
```

**End-of-path matcher** — make a trailing slash exact rather than a prefix:

```go
mux.HandleFunc("GET /posts/{$}", listPosts) // matches /posts/ but not /posts/123
```

**Precedence** — the more specific pattern wins. `/posts/latest` beats `/posts/{id}`. Two patterns that overlap with neither being more specific cause a panic at registration time, which surfaces routing ambiguity at startup rather than at request time.

For most CRUD APIs, the stdlib router is now sufficient. The case for chi/gin/echo is no longer "you need a real router" — it is ergonomic JSON binding, validation, and OpenAPI generation. See [chi vs gin vs echo vs fiber](02-chi-vs-gin-vs-echo-vs-fiber.md).

## 4. Middleware as Handler-wrapping functions

A middleware in Go is a function that takes a `Handler` and returns a new `Handler` that wraps it:

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %s", r.Method, r.URL.Path, time.Since(start))
    })
}

func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if !validToken(token) {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

Composition is just function calls:

```go
handler := loggingMiddleware(authMiddleware(mux))
http.ListenAndServe(":8080", handler)
```

This is the entire middleware system. There is no framework registry, no decorator order, no `app.use()` table. The composition is plain Go — you can read top-to-bottom what runs in what order. Compare to [Express middleware](../../typescript/frameworks/express-vs-alternatives.md) (registered as a list with implicit ordering) and [NestJS guards/interceptors](../../typescript/frameworks/nestjs-for-spring-devs.md) (declared via decorators with framework-controlled order).

To pass values from middleware to handler, attach them to the request context (§9), not to mutable globals.

A useful pattern for stacks of middleware is a tiny helper:

```go
func chain(h http.Handler, mws ...func(http.Handler) http.Handler) http.Handler {
    for i := len(mws) - 1; i >= 0; i-- {
        h = mws[i](h)
    }
    return h
}

handler := chain(mux,
    loggingMiddleware,
    authMiddleware,
    rateLimitMiddleware,
)
```

This is roughly all `chi.Use` does internally.

## 5. The `Server` struct and the timeouts that matter

`http.ListenAndServe(":8080", handler)` is a one-line shortcut. For anything beyond a sketch, build the `Server` explicitly so you can set timeouts:

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           handler,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       30 * time.Second,
    WriteTimeout:      30 * time.Second,
    IdleTimeout:       120 * time.Second,
    MaxHeaderBytes:    1 << 20, // 1 MiB
}
log.Fatal(srv.ListenAndServe())
```

The five fields, with the security-critical one first:

| Field | What it bounds | Default |
|---|---|---|
| **`ReadHeaderTimeout`** | Time to read the request line + headers | **0 (no limit)** |
| `ReadTimeout` | Time from connection accept to fully read body | 0 |
| `WriteTimeout` | Time from header read complete to response fully written | 0 |
| `IdleTimeout` | Keep-alive idle time between requests | 0 (falls back to `ReadTimeout`) |
| `MaxHeaderBytes` | Max bytes in request headers | 1 MiB (`DefaultMaxHeaderBytes`) |

**Set `ReadHeaderTimeout` on every internet-facing server.** With it unset, a Slowloris attacker — a client that opens many connections and sends headers one byte at a time — can pin every file descriptor on the box. Filippo Valsorda's "So you want to expose Go on the Internet" (linked in references) is the canonical write-up. Quoting Filippo:

> a single malicious client can just open as many connections as your server has file descriptors, hold them half-way through the headers, respond to the rare keep-alives, and effectively take down your service

`ReadTimeout` covers the whole body too, so for endpoints that legitimately accept large uploads you want a tight `ReadHeaderTimeout` (5–10s) and a longer `ReadTimeout` matching the upload budget. `WriteTimeout` should comfortably exceed your slowest legitimate response — long-poll endpoints break it.

`IdleTimeout` matters because `net/http` reuses keep-alive connections by default; setting `IdleTimeout: 120s` lets idle connections die without waiting for the next read attempt to fail.

## 6. Graceful shutdown — the SIGTERM dance

Production deployments (Kubernetes, ECS, Nomad) send `SIGTERM` to ask a process to shut down. The handler's job is to stop accepting new connections, let in-flight requests finish, then exit.

```go
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
defer stop()

srv := &http.Server{Addr: ":8080", Handler: handler /* + timeouts */}

go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatalf("listen: %v", err)
    }
}()

<-ctx.Done() // wait for SIGTERM

shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err := srv.Shutdown(shutdownCtx); err != nil {
    log.Printf("graceful shutdown failed: %v", err)
}
```

`srv.Shutdown(ctx)`:
1. Closes the listener — no new connections accepted.
2. Waits for active requests to finish, up to the context deadline.
3. Returns `nil` once everything is drained, or the context error on timeout.

The 30-second budget should be tighter than your orchestrator's `terminationGracePeriodSeconds` (default 30s in Kubernetes) so you fail before the platform `SIGKILL`s you. For long-poll or streaming endpoints, propagate `shutdownCtx` cancellation into them via [request context](#9-request-context-rcontext-and-request-scoped-values).

`context.NotifyContext` (Go 1.16+) removes the boilerplate of installing a signal channel. See [Context Propagation](../concurrency/04-context-propagation.md) for the full context model.

## 7. Request body limits and JSON decoding

`r.Body` is unbounded by default. A POST with a 10 GiB body will happily fill memory if you `io.ReadAll` it. Cap it explicitly:

```go
func createPost(w http.ResponseWriter, r *http.Request) {
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // 1 MiB

    var p Post
    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields()
    if err := dec.Decode(&p); err != nil {
        var maxErr *http.MaxBytesError
        if errors.As(err, &maxErr) {
            http.Error(w, "payload too large", http.StatusRequestEntityTooLarge)
            return
        }
        http.Error(w, "bad json: "+err.Error(), http.StatusBadRequest)
        return
    }
    // ...
}
```

Three things this snippet gets right:
- `MaxBytesReader` caps the read and signals the server to close the connection when the limit is hit (per the package docs).
- `json.NewDecoder(r.Body).Decode` streams; it does not buffer the whole body first like `json.Unmarshal(body)`.
- `dec.DisallowUnknownFields()` rejects unexpected fields — defense against client bugs and a soft form of input validation.

The error path uses [errors.As](../fundamentals/06-errors-as-values.md) on the typed `*http.MaxBytesError` rather than string matching.

For multipart uploads, `r.ParseMultipartForm(maxBytes)` enforces the same kind of cap on a per-form basis.

## 8. `ResponseWriter`: WriteHeader, Hijacker, Flusher, Pusher

The base interface is small:

```go
type ResponseWriter interface {
    Header() Header
    Write([]byte) (int, error)
    WriteHeader(statusCode int)
}
```

Order of operations matters:
1. Mutate `w.Header()` first.
2. Call `w.WriteHeader(status)` (or just call `w.Write` — the first `Write` implies `WriteHeader(200)`).
3. Once headers are sent, you cannot change them. Setting `Content-Type` after `Write` has no effect.

The standard library response writer implements three optional interfaces, recovered with type assertions:

```go
// Server-Sent Events / streaming response
if flusher, ok := w.(http.Flusher); ok {
    fmt.Fprintf(w, "data: %s\n\n", payload)
    flusher.Flush() // push bytes to client now
}

// WebSocket upgrade / protocol switch — take over the TCP connection
if hj, ok := w.(http.Hijacker); ok {
    conn, bufrw, err := hj.Hijack()
    // ... you now own conn; the http server stops touching it
}

// HTTP/2 server push (best-effort, deprecated by browsers)
if pusher, ok := w.(http.Pusher); ok {
    _ = pusher.Push("/static/app.css", nil)
}
```

The optional-interface-via-type-assertion pattern is the Go idiom for "extra capability if available." It is the same shape as `io.WriterTo` or `io.ReaderFrom`. See [Interfaces & Structural Typing §4](../fundamentals/05-interfaces-and-structural-typing.md#4-type-assertions-and-type-switches).

For modern streaming work, prefer `http.Flusher` for SSE and a real WebSocket library (`gorilla/websocket`, `nhooyr/websocket`, or `golang.org/x/net/websocket`) for WebSockets — they handle framing, pings, and subprotocol negotiation that you do not want to hand-roll on top of `Hijack`.

HTTP/2 support is automatic when you serve TLS (`srv.ListenAndServeTLS`); see [Networking — HTTP/2](../../networking/INDEX.md) and [Security — TLS](../../security/INDEX.md).

## 9. Request context: `r.Context()` and request-scoped values

Every request carries a `context.Context`. It is canceled when the client disconnects, when the server shuts down, or when a deadline you attach expires.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // pass ctx to anything that takes it: db.QueryContext, http.Client, etc.
    rows, err := db.QueryContext(ctx, "SELECT ...")
    // ... if the client hangs up, the query is canceled
}
```

Middleware can attach values to the context for downstream handlers. The conventional pattern uses an unexported key type to avoid collisions:

```go
type ctxKey int

const userKey ctxKey = iota

func withUser(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        u, err := authenticate(r)
        if err != nil {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        ctx := context.WithValue(r.Context(), userKey, u)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func currentUser(ctx context.Context) (*User, bool) {
    u, ok := ctx.Value(userKey).(*User)
    return u, ok
}
```

This is how the request user, request ID, tenant, trace span, and similar request-scoped data flow through a Go service. `context.WithValue` is **not** a general-purpose DI mechanism — Go style is to thread real dependencies through constructors and reserve context for things that are genuinely request-scoped.

The full context model — deadlines, cancellation, propagation, and the `context.Background` / `context.TODO` distinction — is covered in [Context Propagation](../concurrency/04-context-propagation.md).

## 10. When you actually need a framework

For straight CRUD with the Go 1.22 ServeMux, plain `net/http` plus a logging middleware and a JSON helper is enough. Reach for a framework when one of these is a daily friction:

| Friction | What a framework gives you |
|---|---|
| Path validation: "this id must be a UUID" | `chi`/`gin` route constraints, or a generic helper |
| Body validation rules and translated errors | `gin` and `echo` ship with `go-playground/validator` |
| OpenAPI / Swagger generation from code | `huma`, `goa`, or the `swag` annotations route |
| Per-route binding boilerplate (params + body + query) | `gin.Context.ShouldBind`, `echo.Context.Bind` |
| Versioned route groups with shared middleware | every framework supports this; stdlib needs a small helper |
| Built-in CORS, recovery, request ID, gzip middlewares | every framework ships these as one-liners |

What you should **not** pick a framework for:
- Routing performance — request matching is rarely the bottleneck. The benchmarks chi/gin/echo cite are real but typically dwarfed by your database round trip.
- "Frameworks are faster than stdlib" — Go 1.22's `ServeMux` matches in O(log n) with method patterns and is competitive enough that the gap rarely matters.

Compared to your TS / Java background:
- **Express** is closer to plain `net/http` than to gin: minimal core, build-your-own-stack. The Go translation is one-to-one.
- **NestJS** style (decorator-driven controllers, DI, guards/interceptors) does not have a direct Go equivalent — the language has no decorators. Frameworks like `huma` get part of the way via reflection and struct tags. See [NestJS for Spring devs](../../typescript/frameworks/nestjs-for-spring-devs.md).
- **Spring Boot's** auto-configuration and starter dependencies have no Go counterpart; the Go community is allergic to magic. You wire things in `main.go`, explicitly.

The follow-up doc surveys the four frameworks the community actually uses: [chi vs gin vs echo vs fiber](02-chi-vs-gin-vs-echo-vs-fiber.md). The third doc covers the gRPC ecosystem: [gRPC in Go](03-grpc-in-go.md).

## Related

- [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) — `Handler` is the canonical one-method interface
- [Errors as Values](../fundamentals/06-errors-as-values.md) — `errors.As` against `*http.MaxBytesError`
- [Goroutines & the Scheduler](../concurrency/01-goroutines-and-scheduler.md) — one goroutine per request
- [Context Propagation](../concurrency/04-context-propagation.md) — `r.Context()` and graceful cancellation
- [chi vs gin vs echo vs fiber](02-chi-vs-gin-vs-echo-vs-fiber.md) — framework selection
- [gRPC in Go](03-grpc-in-go.md) — when JSON-over-HTTP isn't the right call
- [Express vs alternatives (TS)](../../typescript/frameworks/express-vs-alternatives.md) — the closest TS analogue
- [NestJS for Spring devs](../../typescript/frameworks/nestjs-for-spring-devs.md) — what Go deliberately does not give you
- [Networking — HTTP/2, TLS](../../networking/INDEX.md)
- [Security — TLS, auth](../../security/INDEX.md)

## References

- `net/http` package — https://pkg.go.dev/net/http
- `http.Server` — https://pkg.go.dev/net/http#Server
- `http.MaxBytesReader` — https://pkg.go.dev/net/http#MaxBytesReader
- `http.Hijacker` — https://pkg.go.dev/net/http#Hijacker
- `http.Flusher` — https://pkg.go.dev/net/http#Flusher
- `http.Pusher` — https://pkg.go.dev/net/http#Pusher
- Go 1.22 release notes (ServeMux improvements) — https://go.dev/doc/go1.22
- "Routing Enhancements for Go 1.22" — https://go.dev/blog/routing-enhancements
- Filippo Valsorda, "So you want to expose Go on the Internet" — https://blog.cloudflare.com/exposing-go-on-the-internet/ (the canonical reference for HTTP server timeouts and Slowloris hardening)
- `signal.NotifyContext` — https://pkg.go.dev/os/signal#NotifyContext
- `context` package — https://pkg.go.dev/context
- Effective Go, "Web server" — https://go.dev/doc/effective_go#web_server
