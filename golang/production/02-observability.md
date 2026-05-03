---
title: "Observability for Go Services"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, production, observability, slog, opentelemetry, tracing]
---

# Observability for Go Services

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `production` `observability` `slog` `opentelemetry` `tracing`

---

## Table of Contents

1. [The three pillars in Go](#1-the-three-pillars-in-go)
2. [`log/slog` — structured logging in the standard library](#2-logslog--structured-logging-in-the-standard-library)
3. [Replacing third-party loggers (zap, zerolog, logrus)](#3-replacing-third-party-loggers-zap-zerolog-logrus)
4. [OpenTelemetry-go: SDK setup and OTLP export](#4-opentelemetry-go-sdk-setup-and-otlp-export)
5. [Auto-instrumentation: `otelhttp`, `otelsql`, `otelgrpc`](#5-auto-instrumentation-otelhttp-otelsql-otelgrpc)
6. [Trace context propagation through `context.Context`](#6-trace-context-propagation-through-contextcontext)
7. [Metrics with OpenTelemetry Go](#7-metrics-with-opentelemetry-go)
8. [Semantic Conventions](#8-semantic-conventions)
9. [Correlating logs with traces](#9-correlating-logs-with-traces)
10. [Comparison: pino + OpenTelemetry-JS, Spring Boot + Micrometer](#10-comparison-pino--opentelemetry-js-spring-boot--micrometer)

## Summary

Observability for a Go service has three signals — logs, metrics, traces — and as of Go 1.21 the standard library covers one of them well (`log/slog` for logs) and leaves the other two to OpenTelemetry. The OpenTelemetry-Go SDK is the de facto choice for traces and metrics; it implements the W3C Trace Context spec for distributed propagation and exports via OTLP to any compatible backend (Jaeger, Tempo, Honeycomb, Datadog, SigNoz, the OpenTelemetry Collector).

The wire-up effort is roughly: pick a `slog.Handler` (probably `JSONHandler` for production), set up an OTLP tracer/meter provider in `main`, wrap your HTTP handler with `otelhttp.NewHandler`, wrap your `*sql.DB` with `otelsql`, and pass `context.Context` through every layer so trace IDs propagate. Once that scaffold is in place, every request gets a span, every log line that includes the request context can carry the trace ID, and every metric is dimensional (with attributes) instead of pre-aggregated.

This doc walks through that stack from the bottom up, with the V8/Node and Spring Boot equivalents flagged where useful.

## 1. The three pillars in Go

A common mental model — logs, metrics, traces — and where the Go ecosystem lands on each:

| Signal | Stdlib | De facto library | Wire format |
|---|---|---|---|
| Logs | `log/slog` (Go 1.21+) | slog handlers; specific backends via [zap](https://github.com/uber-go/zap), [zerolog](https://github.com/rs/zerolog), [logrus](https://github.com/sirupsen/logrus) | JSON line per record; sometimes OTLP |
| Metrics | none (Go 1.21+ has `runtime/metrics`, but it's read-side only) | OpenTelemetry-Go, Prometheus client_golang | OTLP, Prometheus exposition |
| Traces | none | OpenTelemetry-Go | OTLP (W3C Trace Context propagation) |

The `runtime/metrics` package is worth a quick note: it exposes the runtime's own metrics (GC pause time, goroutine count, heap size) in a stable, structured way. It is **not** an application metrics library — you cannot define your own counters with it — but it is the right way to feed runtime metrics into Prometheus or OpenTelemetry. See [`go.opentelemetry.io/contrib/instrumentation/runtime`](https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/runtime) for an OTel adapter that does this.

## 2. `log/slog` — structured logging in the standard library

`log/slog` shipped in Go 1.21 (August 2023) and is now the default for new code. The model:

- A **Logger** is the call surface: `slog.Info`, `slog.Error`, etc.
- A **Handler** is the backend: it receives a **Record** (timestamp, level, message, attributes) and decides how to write it. Two ship in the stdlib: `TextHandler` (human-readable `key=value`) and `JSONHandler` (one JSON object per line).
- An **Attr** is a typed key-value pair: `slog.String("user_id", id)`, `slog.Int("count", n)`, `slog.Duration("elapsed", d)`.

A typical production setup:

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level:     slog.LevelInfo,
        AddSource: false,  // include file:line; off for prod (perf cost)
    })
    logger := slog.New(handler)
    slog.SetDefault(logger)

    slog.Info("server starting", "port", 8080, "env", "prod")
}
```

Output:

```json
{"time":"2026-05-03T12:00:00Z","level":"INFO","msg":"server starting","port":8080,"env":"prod"}
```

Three idioms worth knowing:

**Variadic key-value vs typed `Attr`.** Both work; typed is faster (no reflection on values) and type-safe.

```go
slog.Info("processed", "count", 42, "duration_ms", 17)            // variadic
slog.Info("processed", slog.Int("count", 42), slog.Int64("duration_ms", 17))  // typed
slog.LogAttrs(ctx, slog.LevelInfo, "processed",                   // zero-allocation form
    slog.Int("count", 42), slog.Int64("duration_ms", 17))
```

For hot logging paths, `LogAttrs` is the form to use. The slog blog post mentions that 95% of log calls in surveyed code used five or fewer attributes — slog's design optimizes that case.

**`With` for shared context.** Build a derived logger that always emits a set of attributes:

```go
reqLogger := slog.With("request_id", id, "user_id", userID)
reqLogger.Info("started")
reqLogger.Info("validated input")
reqLogger.Error("db failure", "err", err)
```

Each line gets the request_id and user_id automatically — the equivalent of `child` loggers in pino or `MDC` in Java's logging frameworks.

**Dynamic level control via `LevelVar`.** Useful for flipping a service to debug without restarting it:

```go
var logLevel = new(slog.LevelVar)  // defaults to Info
handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: logLevel})

// later, e.g., from an admin endpoint:
logLevel.Set(slog.LevelDebug)
```

## 3. Replacing third-party loggers (zap, zerolog, logrus)

For most new services, **slog is enough**. The stdlib handler is fast (the design notes in the slog blog post explicitly target zap-class performance), the API is stable, and the fact that it's in the stdlib means every library can import it without forcing a dependency on a third-party logger.

When to reach for a third-party logger anyway:

| Library | When it matters |
|---|---|
| [zap](https://github.com/uber-go/zap) | You need the absolute fastest possible logging hot path; `zap.SugaredLogger` is a fraction faster than slog in microbenchmarks. Its sampler (drop log lines after N per second per stack) is built in. |
| [zerolog](https://github.com/rs/zerolog) | Similar performance niche to zap, with a chained API. |
| [logrus](https://github.com/sirupsen/logrus) | Legacy code. New code should not pick logrus; the maintainer has [marked it in maintenance mode](https://github.com/sirupsen/logrus#logrus). |

Both zap and zerolog ship slog **handlers** (`zapslog`, `zerolog/sloghandler`), so you can use the slog API as your call surface and zap as the backend, getting the best of both: stdlib-stable surface, third-party performance underneath. This is the modern recommendation when you specifically need zap's speed.

For most applications, the slog `JSONHandler` is fast enough and avoids a dependency. Profile (see [pprof doc](01-profiling-with-pprof.md)) before optimizing here.

## 4. OpenTelemetry-go: SDK setup and OTLP export

[OpenTelemetry-Go](https://github.com/open-telemetry/opentelemetry-go) is the canonical implementation of the OpenTelemetry spec for Go. The SDK has three layers:

- **API** (`go.opentelemetry.io/otel`) — what your application code imports. Defines `Tracer`, `Meter`, `Span`, etc.
- **SDK** (`go.opentelemetry.io/otel/sdk`) — implements the API. You wire it up once in `main`.
- **Exporters** (`go.opentelemetry.io/otel/exporters/...`) — send data to a backend. OTLP (the OpenTelemetry Protocol) is the default; vendors often have their own exporters too.

Minimum viable setup for traces:

```go
package main

import (
    "context"
    "log"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

func initTracer(ctx context.Context) (func(context.Context) error, error) {
    exporter, err := otlptracegrpc.New(ctx)  // reads OTEL_EXPORTER_OTLP_ENDPOINT from env
    if err != nil {
        return nil, err
    }

    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("billing-api"),
            semconv.ServiceVersion("v1.4.2"),
        ),
    )
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.05))),  // 5% sampling
    )
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.TraceContext{})  // W3C Trace Context

    return tp.Shutdown, nil
}

func main() {
    ctx := context.Background()
    shutdown, err := initTracer(ctx)
    if err != nil {
        log.Fatal(err)
    }
    defer shutdown(ctx)

    // ... your service ...
}
```

A few notes:

- **`WithBatcher` vs `WithSyncer`.** Batcher buffers spans and exports them in batches; production should always use the batcher. Syncer is for tests where you need to assert on exported spans synchronously.
- **Sampling decisions are made at span creation time** and propagate via the trace flags in the W3C `traceparent` header (see §6). `ParentBased` says "if my parent was sampled, I am too; if there's no parent, decide via the inner sampler." This keeps sampling decisions consistent across services in a trace.
- **`Shutdown` flushes pending spans.** If you don't call it on graceful shutdown, the batcher may drop the last few seconds of telemetry. Pair with `signal.NotifyContext` (see [Go in Kubernetes doc](03-go-in-kubernetes.md)).

## 5. Auto-instrumentation: `otelhttp`, `otelsql`, `otelgrpc`

The OpenTelemetry Go contrib repository ships **wrappers** for common stdlib packages. Wrap once and every operation through them produces spans automatically.

**`net/http`** — wrap your handler:

```go
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

mux := http.NewServeMux()
mux.HandleFunc("/orders", handleOrders)
http.ListenAndServe(":8080", otelhttp.NewHandler(mux, "billing-api"))
```

The wrapper:
- Extracts incoming `traceparent` / `tracestate` headers.
- Starts a server span for each request.
- Records HTTP attributes (`http.method`, `http.route`, `http.status_code`) per the OpenTelemetry HTTP semantic conventions.
- Closes the span when the response is written.

For outgoing calls, wrap `http.RoundTripper`:

```go
client := &http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
}
// outgoing requests now inject traceparent and emit client spans
```

**`database/sql`** — wrap the driver:

```go
import (
    "github.com/XSAM/otelsql"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

db, err := otelsql.Open("postgres", dsn,
    otelsql.WithAttributes(semconv.DBSystemPostgreSQL))
```

Each `db.QueryContext` / `db.ExecContext` call now produces a span, with attributes for the operation type and (optionally, off by default for privacy) the query text.

**gRPC** — interceptors:

```go
import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

server := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)
conn, err := grpc.Dial(addr,
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
)
```

The pattern is the same for every protocol: a wrapper that creates spans, propagates context through `context.Context`, and serializes trace state into protocol headers.

## 6. Trace context propagation through `context.Context`

The mechanism that ties spans across services together is **W3C Trace Context** — two HTTP headers defined by the [W3C spec](https://www.w3.org/TR/trace-context/):

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: vendor1=value1,vendor2=value2
```

The `traceparent` header has four fields separated by hyphens:

- `00` — version (always 00 today)
- `4bf92f3577b34da6a3ce929d0e0e4736` — 16-byte trace ID
- `00f067aa0ba902b7` — 8-byte span ID (the parent of the receiver's span)
- `01` — flags; bit 0 is the sampled flag

The `tracestate` header carries vendor-specific extensions as comma-separated key=value pairs.

In Go, propagation flows through `context.Context`. The OpenTelemetry SDK injects and extracts trace state via the configured propagator:

```go
otel.SetTextMapPropagator(propagation.TraceContext{})
// or, to also support the older Jaeger/B3 formats:
otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
    propagation.TraceContext{},
    propagation.Baggage{},
))
```

Application code never touches the headers directly — `otelhttp` extracts them on the server side, attaches the resulting `SpanContext` to the request's `context.Context`, and `otelhttp.NewTransport` injects them on the client side from a context with an active span. Your code's job is just to **pass `ctx` through every function call**:

```go
func handleOrder(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    user, err := loadUser(ctx, r.URL.Query().Get("id"))
    if err != nil { /* ... */ }
    if err := chargeCard(ctx, user); err != nil { /* ... */ }
    // ...
}

func loadUser(ctx context.Context, id string) (*User, error) {
    ctx, span := tracer.Start(ctx, "loadUser")
    defer span.End()
    // ... db query with ctx; otelsql sees the parent span and links the child
}
```

The discipline here is the same `ctx context.Context` first-argument convention that Go's standard library has used since Go 1.7. If a function does not accept `ctx`, it cannot participate in tracing or cancellation. This is why most Go style guides — and `golangci-lint` rules — flag any blocking function that doesn't take `ctx` as the first parameter.

## 7. Metrics with OpenTelemetry Go

The metrics API is the meter-and-instrument model:

```go
import "go.opentelemetry.io/otel/metric"

meter := otel.Meter("billing-api")

requestCount, _ := meter.Int64Counter("http.server.request.count",
    metric.WithDescription("Total HTTP requests"),
    metric.WithUnit("{request}"))

requestDuration, _ := meter.Float64Histogram("http.server.request.duration",
    metric.WithDescription("HTTP request duration"),
    metric.WithUnit("s"))

// in a request handler:
start := time.Now()
defer func() {
    requestDuration.Record(r.Context(), time.Since(start).Seconds(),
        metric.WithAttributes(
            attribute.String("http.method", r.Method),
            attribute.Int("http.status_code", status),
        ))
    requestCount.Add(r.Context(), 1,
        metric.WithAttributes(
            attribute.String("http.method", r.Method),
        ))
}()
```

Instrument types:

| Instrument | Use for |
|---|---|
| `Counter` | Monotonic counters: requests, errors, bytes processed |
| `UpDownCounter` | Non-monotonic gauges-via-deltas: active connections, queue depth |
| `Histogram` | Distributions: request duration, payload size |
| `Gauge` (Go 1.22 SDK+) | Snapshot values: current memory usage, current temperature |
| `ObservableCounter` / `ObservableUpDownCounter` / `ObservableGauge` | Async — invoked by the SDK during collection (e.g., reading `runtime/metrics`) |

Metrics export via the same OTLP exporter as traces (use `otlpmetricgrpc` instead of `otlptracegrpc`), or — if your backend is Prometheus — via the Prometheus exporter, which exposes a `/metrics` HTTP endpoint in the standard Prometheus exposition format.

The auto-instrumented `otelhttp.NewHandler` already records the standard HTTP server metrics, so for many services you do not need to define HTTP metrics manually — only your business metrics.

## 8. Semantic Conventions

[OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/) are the vendor-neutral attribute names you should use. Examples:

| Domain | Attributes |
|---|---|
| HTTP | `http.request.method`, `http.response.status_code`, `http.route`, `url.path`, `url.scheme` |
| Database | `db.system`, `db.namespace`, `db.operation.name`, `db.collection.name` |
| RPC | `rpc.system`, `rpc.service`, `rpc.method` |
| Messaging | `messaging.system`, `messaging.destination.name`, `messaging.operation.type` |
| Resource | `service.name`, `service.version`, `deployment.environment.name`, `host.name` |

Why this matters: backends like Tempo, Jaeger, Datadog, and Honeycomb all build their query languages around these conventions. Using `service.name` (the convention) instead of `app` (something you made up) means dashboards, default views, and alerting templates work out of the box. The Go SDK ships generated constants (`semconv.ServiceName(...)`, `semconv.HTTPRequestMethodKey.String(...)`) so you don't have to remember the strings.

The conventions are versioned (currently v1.26.0 at the time of writing). Pin a version in your imports and upgrade deliberately — attribute renames have happened across versions (`http.method` → `http.request.method` was a famous one).

## 9. Correlating logs with traces

The bridge between the logs pillar and the traces pillar is including the trace ID and span ID on every log line. With slog and OpenTelemetry, the idiomatic pattern is a small handler wrapper:

```go
import (
    "context"
    "log/slog"

    "go.opentelemetry.io/otel/trace"
)

type traceContextHandler struct {
    slog.Handler
}

func (h traceContextHandler) Handle(ctx context.Context, r slog.Record) error {
    if span := trace.SpanFromContext(ctx); span.SpanContext().IsValid() {
        sc := span.SpanContext()
        r.AddAttrs(
            slog.String("trace_id", sc.TraceID().String()),
            slog.String("span_id", sc.SpanID().String()),
        )
    }
    return h.Handler.Handle(ctx, r)
}

logger := slog.New(traceContextHandler{
    Handler: slog.NewJSONHandler(os.Stdout, nil),
})
slog.SetDefault(logger)
```

Now any code that uses `slog.InfoContext(ctx, ...)` (the context-aware variants) inside an active span emits log lines with `trace_id` and `span_id`. From there the pattern is:

- Click a slow span in your tracing UI.
- Copy its trace ID.
- Search your log backend (Loki, Cloud Logging, Elasticsearch) for that trace ID.
- See exactly the log lines emitted during that request, in order.

This is the single highest-leverage observability move, and it costs about 20 lines of glue code. The OpenTelemetry-Go contrib repository also ships an experimental `otelslog` handler that does this and bridges slog records into OpenTelemetry log signals — keep an eye on it as the OTel logs pillar matures.

## 10. Comparison: pino + OpenTelemetry-JS, Spring Boot + Micrometer

The ergonomic comparison for a TS/Java dev:

**TypeScript / Node:**

| Concern | Node ecosystem | Go equivalent |
|---|---|---|
| Logger | [pino](https://github.com/pinojs/pino) (JSON, fast) | `slog.NewJSONHandler` |
| Child logger | `logger.child({ requestId })` | `slog.With("request_id", id)` |
| HTTP auto-instrumentation | `@opentelemetry/instrumentation-http` | `otelhttp.NewHandler` |
| Tracer setup | `NodeTracerProvider` + auto-instrumentation registration | `sdktrace.NewTracerProvider` + manual wrapping |
| Async-context propagation | `AsyncLocalStorage` (used by OpenTelemetry Node) | `context.Context` (passed explicitly) |

The biggest cultural difference: in Node, OpenTelemetry uses `AsyncLocalStorage` to propagate context implicitly through async/await chains. In Go, you pass `ctx` explicitly as the first argument to every function. The Go approach is more verbose but has zero magic — there's no equivalent of "the context was lost across a `setImmediate`" debugging session.

**Java / Spring Boot:**

| Concern | Spring ecosystem | Go equivalent |
|---|---|---|
| Logger | Logback / Log4j2 with Spring Boot autoconfig, structured layouts | `slog.NewJSONHandler` |
| MDC | `MDC.put("requestId", id)` (thread-local) | `slog.With(...)` returning a derived logger |
| Metrics | Micrometer (with Prometheus or OTLP registry) | OpenTelemetry-Go meter |
| Tracing | Spring Cloud Sleuth / Micrometer Tracing + OTel bridge | OpenTelemetry-Go directly |
| Auto-instrumentation | Spring Boot Actuator + OTel Java agent (bytecode rewriting) | Explicit wrappers (`otelhttp`, `otelsql`) |
| Health endpoints | `/actuator/health`, `/actuator/metrics` | hand-rolled `/healthz` + Prometheus exporter |

The headline difference is auto-instrumentation. The OpenTelemetry Java agent uses bytecode instrumentation to wrap Spring, JDBC, gRPC, Kafka, etc. transparently — drop the agent JAR onto your JVM start command and you get a fully-traced application without code changes. Go has no equivalent: there is no bytecode rewriter, no agent. Every layer gets explicit wrapping (`otelhttp.NewHandler`, `otelsql.Open`, `otelgrpc` interceptors). This is a real ergonomic cost, but it's also a clarity benefit — the instrumentation is in your code, you can read it, and there's no surprise behavior from an agent.

For the corresponding Node and Java production guides, see [TypeScript — Observability](../../typescript/production/observability.md). For dashboards and SLI/SLO design that the metrics feed into, see [Observability — INDEX](../../observability/INDEX.md).

## Related

- [Profiling Go Services with pprof](01-profiling-with-pprof.md) — the fourth pillar, profiling, and how it complements logs/metrics/traces
- [Goroutines & the Scheduler (G-M-P)](../concurrency/01-goroutines-and-scheduler.md) — `context.Context` propagation and goroutine lifecycle
- [Go in Kubernetes](03-go-in-kubernetes.md) — graceful shutdown so the OTLP exporter flushes before the pod dies
- [Observability — INDEX](../../observability/INDEX.md) — the three pillars at a vendor-neutral level
- [Performance — INDEX](../../performance/INDEX.md) — what to do with the histograms once you're emitting them
- [TypeScript — Observability](../../typescript/production/observability.md) — pino, OpenTelemetry-JS, AsyncLocalStorage

## References

- `log/slog` package documentation — https://pkg.go.dev/log/slog
- "Structured Logging with slog" (Go blog) — https://go.dev/blog/slog
- Go 1.21 release notes (slog GA) — https://go.dev/doc/go1.21
- OpenTelemetry-Go repository — https://github.com/open-telemetry/opentelemetry-go
- OpenTelemetry Go documentation — https://opentelemetry.io/docs/languages/go/
- `otelhttp` instrumentation — https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
- `otelgrpc` instrumentation — https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc
- `otelsql` (community-maintained `database/sql` instrumentation) — https://github.com/XSAM/otelsql
- W3C Trace Context Recommendation — https://www.w3.org/TR/trace-context/
- OpenTelemetry Semantic Conventions — https://opentelemetry.io/docs/specs/semconv/
- `runtime/metrics` package — https://pkg.go.dev/runtime/metrics
- OpenTelemetry runtime instrumentation for Go — https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/runtime
