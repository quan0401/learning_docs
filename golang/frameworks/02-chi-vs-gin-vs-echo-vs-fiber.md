---
title: "chi vs gin vs echo vs fiber"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, frameworks, chi, gin, echo, fiber]
---

# chi vs gin vs echo vs fiber

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `frameworks` `chi` `gin` `echo` `fiber`

---

## Table of Contents

1. [The Go HTTP framework landscape](#1-the-go-http-framework-landscape)
2. [chi — the stdlib-aligned router](#2-chi--the-stdlib-aligned-router)
3. [gin — fastest popular net/http framework](#3-gin--fastest-popular-nethttp-framework)
4. [echo — a sibling of gin with its own taste](#4-echo--a-sibling-of-gin-with-its-own-taste)
5. [fiber — fasthttp speed at the cost of ecosystem](#5-fiber--fasthttp-speed-at-the-cost-of-ecosystem)
6. [Routing benchmarks: why "fastest" rarely matters](#6-routing-benchmarks-why-fastest-rarely-matters)
7. [Decision matrix — what to pick when](#7-decision-matrix--what-to-pick-when)
8. [Comparison with Express, Fastify, NestJS, Spring Boot](#8-comparison-with-express-fastify-nestjs-spring-boot)

## Summary

Four routers dominate Go web development outside the standard library: **chi**, **gin**, **echo**, and **fiber**. The first three are thin layers over `net/http`. The fourth, fiber, is built on `fasthttp` — a deliberately incompatible HTTP implementation that trades the entire `net/http` middleware ecosystem for raw throughput.

The community advice that has solidified over the past five years: **default to chi unless you have a strong reason to pick something else**, and treat the routing speed gap as effectively irrelevant compared to your database, network, and serialization costs. Since Go 1.22 added method patterns and wildcards to the standard `ServeMux`, even chi is no longer mandatory for many APIs — see [`net/http` Deep Dive §3](01-net-http-deep-dive.md#3-go-122-servemux-method-patterns-and-wildcards).

This doc walks each option, the tradeoffs you actually feel in production, and a decision matrix tied to the kind of service you're building.

## 1. The Go HTTP framework landscape

Three things shape the landscape and explain why it looks different from Node or Java:

- **`net/http` is good.** The standard library handles connection management, TLS, HTTP/2, graceful shutdown, and (since 1.22) reasonable routing. Most "frameworks" in Go are routers + a small handful of helpers.
- **Go has no decorators.** The NestJS / Spring / FastAPI style of declaring routes, validation, and middleware via metadata cannot be ported one-to-one. What you get instead is explicit chaining and struct tags.
- **The community is allergic to magic.** "Where does this middleware run?" should be answerable by reading source. Frameworks that hide too much get rejected.

The four contenders here all live in that space — small, opinionated wrappers — and split cleanly into two camps:

| Camp | Frameworks | Trade |
|---|---|---|
| **net/http compatible** | chi, gin, echo | Use any `http.Handler` middleware in the Go ecosystem; deploy behind any `net/http`-compatible TLS / observability tooling |
| **fasthttp-based** | fiber | Higher requests/sec on micro-benchmarks; cannot use `http.Handler` middleware directly; some observability tools (anything that sniffs `*http.Request`) won't work |

The split matters more than any individual API choice between gin and echo.

## 2. chi — the stdlib-aligned router

`go-chi/chi` is the smallest of the four. The README states the position outright:

> 100% compatible with net/http - use any http or middleware pkg in the ecosystem that is also compatible with net/http.

API shape:

```go
import "github.com/go-chi/chi/v5"

r := chi.NewRouter()
r.Use(middleware.Logger)
r.Use(middleware.Recoverer)

r.Get("/health", healthHandler)
r.Route("/api/v1", func(r chi.Router) {
    r.Use(authMiddleware) // applied to this subtree only
    r.Get("/users/{id}", showUser)
    r.Post("/users", createUser)
})

http.ListenAndServe(":8080", r)
```

What chi gives you over plain `net/http`:
- Route groups with their own middleware stacks (`r.Route`, `r.Group`).
- `chi.URLParam(r, "id")` for path values (analogous to Go 1.22's `r.PathValue`).
- A curated `middleware` subpackage: request ID, real IP, logger, recoverer, timeout, throttle, basic CORS.
- Sub-router mounting (`r.Mount("/admin", adminRouter)`).

What it deliberately does not give you:
- No JSON binding helpers. You decode `r.Body` with `encoding/json` yourself.
- No validation. Use `go-playground/validator` directly on your decoded struct.
- No custom Context type. Handlers stay `func(http.ResponseWriter, *http.Request)`, so any `http.Handler`-shaped middleware composes.

The reason this is the recommended default: **the seams with the rest of Go are zero**. Your handlers are still `http.Handler`. Your middleware is still `http.Handler`-shaped. When you swap chi for something else, you rewrite routing, not handlers.

Used in production by Cloudflare, Heroku, Pressly (its origin), and many others. Stable, low churn, ~1k LOC core.

## 3. gin — fastest popular net/http framework

`gin-gonic/gin` is the most popular Go web framework by GitHub stars. It runs on `net/http` (your handlers ultimately reach `srv.ListenAndServe`) but exposes its own `*gin.Context` rather than the stdlib `http.ResponseWriter` + `*http.Request`.

```go
import "github.com/gin-gonic/gin"

r := gin.Default() // includes Logger and Recovery middleware

r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")
    c.JSON(http.StatusOK, gin.H{"id": id})
})

r.POST("/users", func(c *gin.Context) {
    var u CreateUser
    if err := c.ShouldBindJSON(&u); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    // ...
})

r.Run(":8080")
```

What gin gives you that chi does not:
- **Built-in JSON binding and validation.** `c.ShouldBindJSON(&u)` decodes the body and runs `validator` rules from struct tags (`binding:"required,email"`).
- **Path parameters with `:` and `*` syntax** (`/users/:id`, `/files/*path`).
- **One-line response helpers** (`c.JSON`, `c.XML`, `c.String`, `c.File`, `c.Redirect`).
- A `c.Next()` / `c.Abort()` middleware model that matches Express's mental shape.

What you trade:
- **Handlers are not `http.Handler`.** Generic middleware that takes `http.Handler` cannot drop in; you adapt with `gin.WrapH` / `gin.WrapF`. Most ecosystem libraries (OTel, Prometheus, sentry-go) ship gin-specific adapters, so this is rarely a blocker — but it is a real ongoing tax.
- **`gin.Context` is its own world.** Logging, tracing, validation errors all flow through it. Tests need to construct a `*gin.Context` instead of a `*httptest.ResponseRecorder` + `*http.Request` directly.

Performance: the README claims "up to 40 times faster" than older frameworks like Martini, leveraging `httprouter`-style trie matching with zero allocations on the routing path. Real-world numbers (see §6) put gin within a couple of percent of any other `net/http`-compatible router for realistic workloads.

Pick gin when: you want one framework that handles routing + binding + validation + standard middleware in a coherent style, and you don't mind the gin-shaped middleware ecosystem.

## 4. echo — a sibling of gin with its own taste

`labstack/echo` is gin's closest peer. Same architectural choice (custom `echo.Context`, built-in binding, curated middleware), different API surface, different style decisions.

```go
import "github.com/labstack/echo/v4"

e := echo.New()
e.Use(middleware.Logger())
e.Use(middleware.Recover())

e.GET("/users/:id", func(c echo.Context) error {
    id := c.Param("id")
    return c.JSON(http.StatusOK, map[string]string{"id": id})
})

e.POST("/users", func(c echo.Context) error {
    var u CreateUser
    if err := c.Bind(&u); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }
    // ...
    return c.JSON(http.StatusCreated, u)
})

e.Logger.Fatal(e.Start(":8080"))
```

Distinctive choices vs gin:
- **Handlers return `error`.** Errors flow through a centralized `HTTPErrorHandler` you can override globally — you do not write `if err != nil { c.JSON(...); return }` in every handler. This is closer to Spring Boot's `@ControllerAdvice` than to Express.
- **Built-in TLS via Let's Encrypt** (`e.AutoTLSManager`, `e.StartAutoTLS`).
- **Built-in middleware for JWT, basic auth, CSRF, rate limiting, gzip, body limit, request ID.**
- HTTP/2 support without ceremony.
- Latest major version is **v5** (as of January 2026); v4 receives security/bug fixes only until December 2026 per the README.

What you trade vs gin: smaller community, fewer third-party gin-style adapters in some ecosystems, but the framework itself is feature-richer for security/TLS concerns out of the box.

The choice between gin and echo is mostly aesthetic and ecosystem-dependent. If your shop already uses one, stay there. The handler-returns-error model in echo is genuinely nicer for centralized error handling; gin's `c.Next()` / `c.Abort()` is genuinely nicer for the Express-style mindset.

## 5. fiber — fasthttp speed at the cost of ecosystem

`gofiber/fiber` is the outlier. Per the README:

> Fiber is an Express inspired web framework built on top of Fasthttp, the fastest HTTP engine for Go.

Two things follow from "built on top of Fasthttp":

**API style** — explicitly Express-like:

```go
import "github.com/gofiber/fiber/v2"

app := fiber.New()
app.Use(logger.New())

app.Get("/users/:id", func(c *fiber.Ctx) error {
    return c.JSON(fiber.Map{"id": c.Params("id")})
})

app.Post("/users", func(c *fiber.Ctx) error {
    u := new(CreateUser)
    if err := c.BodyParser(u); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": err.Error()})
    }
    // ...
    return nil
})

app.Listen(":8080")
```

**The compatibility tax** — fasthttp uses its own `RequestCtx`. It is **not** an `http.Handler`. From the fasthttp README:

> fasthttp might not be for you! fasthttp was designed for some high performance edge cases. Unless your server/client needs to handle thousands of small to medium requests per second and needs a consistent low millisecond response time fasthttp might not be for you. For most cases `net/http` is much better as it's easier to use and can handle more cases.

What this costs you in practice:
- **Generic Go HTTP middleware does not compose.** Anything written for `http.Handler` (the entire `net/http` ecosystem) needs a fiber-specific port or an adapter shim.
- **Some libraries silently misbehave.** Anything that introspects `*http.Request` — observability tools, profiling tooling, certain auth libraries, parts of OpenTelemetry's HTTP semconv — was written against `net/http` and may not work or may need fiber-specific instrumentation.
- **HTTP/2 story is weaker.** fasthttp's HTTP/2 support has historically lagged `net/http`'s.
- **Request reuse pitfall.** fasthttp pools `RequestCtx` objects across requests and reuses headers / body slices. You **cannot** hold a reference to header values past the handler return, or you will read another request's data. This is documented but trips people coming from `net/http` semantics.

What you get: throughput. On synthetic benchmarks (small JSON payloads, no real backend) fiber comes out 2–5× ahead of gin/echo/chi. On realistic workloads (database query, downstream HTTP call, JSON serialization) the gap collapses — your DB round-trip dwarfs the framework cost.

Pick fiber only when you have **measured** that HTTP framework overhead is the bottleneck — gateways, sidecars, ultra-thin proxy services with thousands of small requests per second per core. For ordinary application services, the ecosystem cost is not worth it.

The fiber team has added shims since v2 — per the README, "Fiber automatically adapts common net/http handler shapes when you register them on the router, and you can still use the adaptor middleware when you need to bridge entire apps or net/http middleware." This makes the bridge less painful but does not undo the core architectural split: the runtime is fasthttp, not `net/http`.

## 6. Routing benchmarks: why "fastest" rarely matters

Every Go HTTP framework's README leads with a benchmark chart. Take them with a grain of salt:

- **They measure routing alone.** Match a path, dispatch to a no-op handler, return a fixed response. Real services serialize JSON, talk to databases, call other services — work that takes 100 µs to 10 ms.
- **Microbenchmarks favor whoever wrote them.** Workloads, hardware, Go versions, GC configuration all shift the ranking.
- **Allocation-per-request differences are measurable but rarely meaningful** at the scale most services run.

A useful rule of thumb: if your p99 request handler time is over 5 ms (anything that touches a database), the framework's contribution to it is in the noise. Going from chi's ~150 ns of routing overhead to fiber's ~30 ns saves 120 ns out of 5 000 000 ns. The choice should be made on ergonomics, ecosystem, and operational fit — not benchmark deltas.

The exceptions, where routing throughput genuinely matters:
- Public-facing API gateways doing minimal work per request.
- Edge proxies, reverse proxies, ad-tech bidders.
- Services where each request really does have a sub-millisecond budget.

For those, benchmark *your* workload on *your* hardware before committing.

## 7. Decision matrix — what to pick when

| Situation | Pick | Why |
|---|---|---|
| New CRUD service, mostly REST, normal scale | **chi** (or stdlib if Go 1.22+ is fine) | Smallest seam, full ecosystem, low churn |
| You want JSON binding + validation in the framework | **gin** or **echo** | Built-in `Bind` + struct-tag validation |
| Centralized error handling preference (Spring `@ControllerAdvice` muscle memory) | **echo** | Handlers return `error`, single error handler |
| Express muscle memory, team is JS-heavy | **gin** or **fiber** | Both feel like Express; gin is the safer pick |
| You measured the framework as the bottleneck (gateway / proxy) | **fiber** | Fasthttp earns its keep here, and only here |
| Existing house standard | Stay there | Switching costs > marginal gains |

A common arc: start with chi or stdlib. If the team finds themselves writing the same `decode + validate + render` boilerplate every endpoint, graduate to gin or echo. If the team finds themselves blocked on raw HTTP overhead — measured, not assumed — evaluate fiber.

A second axis: **OpenAPI**. None of these four generate OpenAPI from code first-class. If schema-first is a hard requirement, look at `huma`, `oapi-codegen`, or `goa`, which sit on top of (or beside) the routers above.

## 8. Comparison with Express, Fastify, NestJS, Spring Boot

| Concern | Express | Fastify | NestJS | Spring Boot | net/http | chi | gin/echo | fiber |
|---|---|---|---|---|---|---|---|---|
| Built-in router | yes | yes | yes (decorators) | yes (`@RequestMapping`) | yes (1.22+) | yes | yes | yes |
| Path parameters | yes (`:id`) | yes (`:id`) | yes (`@Param`) | yes (`@PathVariable`) | yes (`{id}`, 1.22+) | yes | yes | yes |
| Body validation built-in | no (use `joi`/`zod`) | yes (JSON Schema) | yes (`class-validator`) | yes (`@Valid`/JSR 380) | no | no | yes (validator tags) | partial |
| DI container | no | no | yes | yes | no | no | no | no |
| Decorator-driven config | no | no | yes | yes | n/a (no decorators in Go) | n/a | n/a | n/a |
| Compatible with stdlib server | n/a | n/a | n/a | n/a | yes | yes | yes (custom Context wraps it) | **no** (fasthttp) |
| Hot reload / starters | community | community | yes | yes (Spring Initializr) | community | community | community | community |
| Ecosystem of "use any middleware" | yes (`(req,res,next)`) | partial | NestJS-specific | Spring-specific | yes | yes | mostly (some adapters) | gated |

Closest analogues:
- **Express → chi** in spirit (small core, you wire your stack), but more honestly **Express → plain `net/http` with a few helpers**. Both expose the raw req/res and trust you.
- **Fastify → gin/echo** — opinionated speed-leaning frameworks with built-in validation.
- **NestJS / Spring Boot** — no real Go counterpart in this list. The ergonomic gap is intentional. The closest "decorators with reflection" approach in Go is `huma`, which generates the entire OpenAPI surface from struct tags but is much less central to the ecosystem than NestJS is to TS.
- **fasthttp / fiber** — there is no Node analogue. Imagine if Node shipped a totally separate event loop you could opt into for 5× speed but lose the npm ecosystem. The Go community has largely declined the trade.

For deeper context on the TypeScript side, see [Express vs alternatives](../../typescript/frameworks/express-vs-alternatives.md) and [NestJS for Spring devs](../../typescript/frameworks/nestjs-for-spring-devs.md).

## Related

- [`net/http` Deep Dive](01-net-http-deep-dive.md) — what every framework here builds on
- [gRPC in Go](03-grpc-in-go.md) — when you should pick a different protocol entirely
- [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) — why `http.Handler` middleware composes so cleanly
- [Errors as Values](../fundamentals/06-errors-as-values.md) — echo's handler-returns-error model maps naturally to Go errors
- [Context Propagation](../concurrency/04-context-propagation.md) — every framework above threads `context.Context`
- [Express vs alternatives (TS)](../../typescript/frameworks/express-vs-alternatives.md)
- [NestJS for Spring devs (TS)](../../typescript/frameworks/nestjs-for-spring-devs.md)

## References

- chi — https://github.com/go-chi/chi
- chi documentation — https://go-chi.io/
- gin — https://github.com/gin-gonic/gin
- echo — https://github.com/labstack/echo
- echo documentation — https://echo.labstack.com/
- fiber — https://github.com/gofiber/fiber
- fasthttp (note about incompatibility with `net/http`) — https://github.com/valyala/fasthttp
- Go 1.22 release notes — https://go.dev/doc/go1.22
- "Routing Enhancements for Go 1.22" — https://go.dev/blog/routing-enhancements
- `httprouter` (the trie-style router gin and many others descend from) — https://github.com/julienschmidt/httprouter
- `go-playground/validator` (used by gin and echo for struct-tag validation) — https://github.com/go-playground/validator
