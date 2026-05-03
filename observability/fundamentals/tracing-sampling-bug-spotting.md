---
title: "Distributed Tracing & Sampling — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, tracing, opentelemetry, sampling, observability]
---

# Distributed Tracing & Sampling — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `tracing` `opentelemetry` `sampling` `observability`

---

## Table of Contents
1. [How to use this doc](#how-to-use-this-doc)
2. [Easy (warm-up traps)](#1-easy-warm-up-traps)
3. [Subtle (review-passers)](#2-subtle-review-passers)
4. [Senior trap (production-only failures)](#3-senior-trap-production-only-failures)
5. [Solutions](#4-solutions)
6. [Related](#related)
7. [References](#references)

## Summary

This doc presents 24 broken tracing setups across context propagation, head/tail sampling, baggage, span attributes (PII and cardinality), span lifecycle, OpenTelemetry SDK pitfalls, and W3C Trace Context edge cases. Code is mostly OpenTelemetry SDKs (TS/Java/Go/Python) plus collector YAML. Suggested cadence: 3–4 bugs per session, then re-derive the W3C Trace Context propagation rules from memory.

## How to use this doc
- Try to spot the bug before opening `<details>`.
- The hint is one line; full root-cause + fix is in §4 Solutions, keyed by bug number.
- Difficulty is cumulative: don't skip Easy unless you've already nailed those traps in code review.

---

## 1. Easy (warm-up traps)

### Bug 1 — Console exporter in prod
```ts
// app.ts (production)
import { NodeSDK } from '@opentelemetry/sdk-node';
import { ConsoleSpanExporter } from '@opentelemetry/sdk-trace-base';

const sdk = new NodeSDK({
  traceExporter: new ConsoleSpanExporter(),
});
sdk.start();
```
<details><summary>Hint</summary>
What does this do to stdout volume and your log bill?
</details>

### Bug 2 — Span status set as attribute string
```java
Span span = tracer.spanBuilder("checkout").startSpan();
try {
  doWork();
} catch (Exception e) {
  span.setAttribute("status", "ERROR");
  span.setAttribute("error.message", e.getMessage());
  throw e;
} finally {
  span.end();
}
```
<details><summary>Hint</summary>
There is a dedicated API for this; backends key off it for error coloring.
</details>

### Bug 3 — `http.url` attribute
```python
span.set_attribute("http.url", request.url)
span.set_attribute("http.method", request.method)
```
<details><summary>Hint</summary>
These attribute names were renamed in the stable HTTP semantic conventions.
</details>

### Bug 4 — Manual `fetch` without inject
```ts
import { trace } from '@opentelemetry/api';

const span = trace.getTracer('orders').startSpan('call-payments');
const res = await fetch('https://payments/charge', {
  method: 'POST',
  body: JSON.stringify(payload),
});
span.end();
```
<details><summary>Hint</summary>
The downstream service has no idea this call is part of a trace.
</details>

### Bug 5 — `traceparent` not honored
```go
// HTTP server — root-only sampler
tp := sdktrace.NewTracerProvider(
    sdktrace.WithSampler(sdktrace.TraceIDRatioBased(0.01)),
)
```
<details><summary>Hint</summary>
What happens to spans whose remote parent already had `sampled=1`?
</details>

### Bug 6 — Span ended in `finally`, error not recorded
```python
span = tracer.start_span("db.query")
try:
    cursor.execute(sql)
finally:
    span.end()
```
<details><summary>Hint</summary>
The span will end fine on exception, but what does the trace UI show?
</details>

### Bug 7 — User email as span attribute
```ts
span.setAttribute('user.email', user.email);
span.setAttribute('user.ip', req.ip);
span.setAttribute('auth.bearer', req.headers.authorization);
```
<details><summary>Hint</summary>
Tracing backends are not designed to be a PII vault.
</details>

---

## 2. Subtle (review-passers)

### Bug 8 — Sampler decides per-span with `Math.random`
```ts
class HomemadeSampler implements Sampler {
  shouldSample(): SamplingResult {
    return {
      decision: Math.random() < 0.05
        ? SamplingDecision.RECORD_AND_SAMPLED
        : SamplingDecision.NOT_RECORD,
    };
  }
}
```
<details><summary>Hint</summary>
What does the resulting trace look like when 5 services each independently roll the dice?
</details>

### Bug 9 — Async work continues after parent ends
```ts
async function handler(req, res) {
  const span = tracer.startSpan('handler');
  res.send('ok');
  span.end();
  // fire-and-forget audit
  void auditLog(req).then(() => {
    const child = tracer.startSpan('audit', { parent: span });
    child.end();
  });
}
```
<details><summary>Hint</summary>
A span's parent must still be open when its child starts.
</details>

### Bug 10 — Context lost across thread boundary
```java
@Async
public CompletableFuture<Result> chargeAsync(Order o) {
  // No context propagation through @Async / ForkJoinPool
  return CompletableFuture.completedFuture(charge(o));
}
```
<details><summary>Hint</summary>
OTel's Java context is thread-local by default.
</details>

### Bug 11 — Baggage cardinality explosion
```ts
// every service writes user_id into baggage AND copies it to every span
const baggage = propagation.getBaggage(ctx) ?? propagation.createBaggage({});
const next = baggage.setEntry('user_id', { value: req.user.id });
// later, on EVERY span:
span.setAttribute('user_id', next.getEntry('user_id')!.value);
```
<details><summary>Hint</summary>
What does this do to your backend's per-attribute cardinality budget?
</details>

### Bug 12 — gRPC interceptor order
```go
srv := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        authInterceptor,        // reads context
        otelgrpc.UnaryServerInterceptor(), // creates span here
    ),
)
```
<details><summary>Hint</summary>
The auth interceptor logs an error span — except it has no span yet.
</details>

### Bug 13 — High-cardinality span name
```python
with tracer.start_as_current_span(f"GET /users/{user_id}/orders/{order_id}"):
    ...
```
<details><summary>Hint</summary>
Span names are an aggregation key; treat them like a metric label.
</details>

### Bug 14 — Resource attribute with pod hash
```yaml
# k8s deployment env
env:
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "service.name=orders,host.name=$(HOSTNAME)"
# HOSTNAME = orders-7c9d8f4b6c-xkj2p
```
<details><summary>Hint</summary>
Every rollout doubles your `host.name` cardinality in the backend.
</details>

### Bug 15 — `tracestate` exceeds vendor budget
```ts
// each service appends its own vendor entry without trimming
const ts = incoming.tracestate
  + `,acme=${JSON.stringify(largeDebugBlob)}`;
res.setHeader('tracestate', ts);
```
<details><summary>Hint</summary>
W3C says vendors "should propagate at least 512 characters" — what about your 2 KB?
</details>

### Bug 16 — Span events on a hot path
```go
for _, row := range rows { // 200k iterations
    span.AddEvent("row.processed", trace.WithAttributes(
        attribute.String("row.id", row.ID),
    ))
}
```
<details><summary>Hint</summary>
Each event allocates and is held until span.End().
</details>

### Bug 17 — BatchSpanProcessor timeout too generous
```ts
new BatchSpanProcessor(exporter, {
  scheduledDelayMillis: 30_000,
  maxExportBatchSize: 10_000,
  maxQueueSize: 50_000,
});
// pod terminationGracePeriodSeconds: 5
```
<details><summary>Hint</summary>
What happens to the in-memory queue when the pod gets SIGTERM?
</details>

### Bug 18 — Sampler runs after redaction
```yaml
# otel-collector pipeline
processors:
  - attributes/redact      # strips trace_id-looking strings from attrs
  - tail_sampling          # uses trace_id for hash-based decisions
exporters: [otlp]
```
<details><summary>Hint</summary>
What does the tail sampler hash on if redaction nuked the IDs?
</details>

---

## 3. Senior trap (production-only failures)

### Bug 19 — Head sampling at the edge
```yaml
# api-gateway
sampling:
  type: probabilistic
  rate: 0.001  # 0.1%
# every downstream service uses ParentBased — so they sample 0.1% too
```
<details><summary>Hint</summary>
A 500 in service E is invisible 99.9% of the time. What sampler shape captures errors?
</details>

### Bug 20 — Single-instance tail sampler
```yaml
# one collector pod, otelcol-tailsampling, no load balancer in front
receivers: [otlp]
processors:
  - tail_sampling:
      decision_wait: 10s
      policies: [{name: errors, type: status_code, status_code: {status_codes: [ERROR]}}]
exporters: [otlp/backend]
# upstream: 50 app pods round-robin to "otelcol:4317"
```
<details><summary>Hint</summary>
For tail sampling to work, every span of a trace must reach the *same* decider.
</details>

### Bug 21 — Missing instrumentation across a queue
```ts
// producer
await kafka.send({ topic: 'orders', messages: [{ value: payload }] });
// consumer (different service) — no header extraction
consumer.run({
  eachMessage: async ({ message }) => {
    const span = tracer.startSpan('process-order'); // new root!
    handle(message);
    span.end();
  },
});
```
<details><summary>Hint</summary>
The trace dies at the broker. Where does `traceparent` need to live for queues?
</details>

### Bug 22 — Exporter blocks app thread on flush
```java
// shutdown hook
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
  tracerProvider.forceFlush().join(60, TimeUnit.SECONDS);
  tracerProvider.shutdown().join(60, TimeUnit.SECONDS);
}));
// SIGTERM → grace period 10s
```
<details><summary>Hint</summary>
What does k8s do at second 10 if your hook is still blocked on a slow OTLP endpoint?
</details>

### Bug 23 — Recording high-cardinality attribute as a metric label
```go
meter.Int64Counter("http.server.requests").Add(ctx, 1,
    metric.WithAttributes(
        attribute.String("http.target", req.URL.Path), // /users/123/orders/456
        attribute.String("user.id", userID),
    ))
```
<details><summary>Hint</summary>
Spans tolerate cardinality. Metrics don't — and your bill won't either.
</details>

### Bug 24 — Probabilistic sampler ignores parent
```ts
// service B
const sdk = new NodeSDK({
  sampler: new TraceIdRatioBasedSampler(0.1), // NOT wrapped in ParentBased
});
// service A samples at 100% and sends sampled=1
// service B independently rolls 10% → drops 90% of traces mid-flight
```
<details><summary>Hint</summary>
Inconsistent inclusion across services breaks every trace view that joins on trace_id.
</details>

---

## 4. Solutions

### Bug 1 — Console exporter in prod
**Root cause:** `ConsoleSpanExporter` writes every sampled span to stdout. Under load this drowns the application logs, blows up log-ingest costs, and slows the process via blocking I/O.
**Fix:**
```ts
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
const sdk = new NodeSDK({ traceExporter: new OTLPTraceExporter({ url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT }) });
```
**Reference:** [OpenTelemetry JS exporters](https://opentelemetry.io/docs/languages/js/exporters/) — `ConsoleSpanExporter` is documented as a debugging tool.

### Bug 2 — Span status set as attribute string
**Root cause:** Backends key error visualization off the OTel `Status` field, not arbitrary attributes. Setting `status="ERROR"` as a string is invisible to error-rate aggregations and `RECORD_AND_SAMPLE` filters.
**Fix:**
```java
span.setStatus(StatusCode.ERROR, e.getMessage());
span.recordException(e);
```
**Reference:** [OTel Trace API — Set Status](https://opentelemetry.io/docs/specs/otel/trace/api/#set-status); attempts to set `Unset` are ignored, and `Ok` is final.

### Bug 3 — `http.url` attribute
**Root cause:** The stable HTTP semantic conventions deprecated `http.url`/`http.method`/`http.target` in favor of `url.full`/`http.request.method`/`url.path`. Old names break dashboards built against the stable schema.
**Fix:**
```python
span.set_attribute("url.full", str(request.url))
span.set_attribute("http.request.method", request.method)
```
**Reference:** [OTel HTTP semantic conventions (stable)](https://opentelemetry.io/docs/specs/semconv/http/http-spans/).

### Bug 4 — Manual `fetch` without inject
**Root cause:** Plain `fetch` does not run OTel's HTTP instrumentation, so `traceparent` is never injected. The downstream service starts a new root trace and the call chain is broken.
**Fix:**
```ts
import { propagation, context } from '@opentelemetry/api';
const headers: Record<string, string> = {};
propagation.inject(context.active(), headers);
await fetch(url, { method: 'POST', body, headers });
```
**Reference:** [W3C Trace Context — propagation requirements](https://www.w3.org/TR/trace-context/#mutating-the-traceparent-field).

### Bug 5 — `traceparent` not honored
**Root cause:** A bare `TraceIDRatioBased(0.01)` ignores incoming sampled flags, so 99% of traces that upstream marked sampled get dropped here, breaking trace continuity.
**Fix:**
```go
tp := sdktrace.NewTracerProvider(
    sdktrace.WithSampler(sdktrace.ParentBased(
        sdktrace.TraceIDRatioBased(0.01),
    )),
)
```
**Reference:** [OTel Trace SDK — `ParentBased` sampler](https://opentelemetry.io/docs/specs/otel/trace/sdk/#parentbased).

### Bug 6 — Span ended in `finally`, error not recorded
**Root cause:** Without `set_status(StatusCode.ERROR)` or `record_exception`, the span ends with `Unset` and looks identical to a successful call in the trace UI.
**Fix:**
```python
span = tracer.start_span("db.query")
try:
    cursor.execute(sql)
except Exception as e:
    span.set_status(Status(StatusCode.ERROR, str(e)))
    span.record_exception(e)
    raise
finally:
    span.end()
```
**Reference:** [OTel Trace API — RecordException](https://opentelemetry.io/docs/specs/otel/trace/api/#record-exception).

### Bug 7 — User email as span attribute
**Root cause:** Span attributes are exported to a tracing backend that is rarely access-controlled like a PII store, often retained 7–30 days, and visible in many UI surfaces. Bearer tokens here also let anyone with trace read access impersonate users.
**Fix:** Hash or omit. Use a stable opaque ID (`user.id`) and never put credentials in spans.
```ts
span.setAttribute('enduser.id', sha256(user.id));
```
**Reference:** [OTel semantic conventions — `enduser.*`](https://opentelemetry.io/docs/specs/semconv/general/attributes/) flags PII handling explicitly. Honeycomb's [PII guidance](https://www.honeycomb.io/blog/sensitive-data-observability-pipelines) covers scrubbing patterns.

### Bug 8 — Sampler decides per-span with `Math.random`
**Root cause:** Sampling must be a function of `trace_id` so that all services in a trace make the same decision. Per-span coin flips produce partial traces (services A and C kept, B dropped).
**Fix:** Use the SDK's `TraceIdRatioBasedSampler` wrapped in `ParentBased`, which hashes `trace_id` deterministically.
**Reference:** [OTel SDK — `TraceIdRatioBased`](https://opentelemetry.io/docs/specs/otel/trace/sdk/#traceidratiobased): "consistent sampling of a trace requires that all spans in a trace make the same decision."

### Bug 9 — Async work continues after parent ends
**Root cause:** Once `span.end()` is called, that span is immutable and exported; creating a child afterwards produces an "orphan" span pointing at a parent that the backend already finalized. The order is wrong: end children first.
**Fix:** Either keep the parent alive until async work completes, or detach with a `Link` instead of a parent reference.
```ts
const link = { context: span.spanContext() };
const child = tracer.startSpan('audit', { links: [link] });
```
**Reference:** [OTel Trace API — span lifecycle](https://opentelemetry.io/docs/specs/otel/trace/api/#end); ending a span makes future updates a no-op.

### Bug 10 — Context lost across thread boundary
**Root cause:** OTel's `Context` is propagated via thread-locals (`ThreadLocal` in Java). `@Async` and `ForkJoinPool` jump threads, so the active span is lost and the work appears as a new root trace.
**Fix:** Use `Context.taskWrapping(executor)` or the OTel Java auto-instrumentation's `@WithSpan` plus `Context.current().wrap(runnable)`.
```java
Executor wrapped = Context.taskWrapping(executor);
```
**Reference:** [OTel Java — context propagation across threads](https://opentelemetry.io/docs/languages/java/instrumentation/#context-propagation).

### Bug 11 — Baggage cardinality explosion
**Root cause:** Baggage is for *propagation*; it is not automatically attached to spans. Manually copying `user_id` onto every span burns the backend's per-key cardinality budget (Honeycomb, Lightstep, Tempo all charge or degrade on this).
**Fix:** Keep `user_id` in baggage if downstream needs it; add it only to the *root* span, or to the spans where it's actually queried.
**Reference:** [W3C Baggage spec](https://www.w3.org/TR/baggage/) — baggage is independent from tracing. Honeycomb's [high-cardinality guidance](https://www.honeycomb.io/blog/observability-101-cardinality) explains the cost model.

### Bug 12 — gRPC interceptor order
**Root cause:** `ChainUnaryInterceptor` runs interceptors left-to-right wrapping the handler, so `authInterceptor` executes *before* `otelgrpc`'s span starts. Any error logging in auth has no active span to attach to, and propagation extraction hasn't run yet.
**Fix:** Put OTel first.
```go
grpc.ChainUnaryInterceptor(
    otelgrpc.UnaryServerInterceptor(),
    authInterceptor,
)
```
**Reference:** [otelgrpc README — recommended interceptor placement](https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation/google.golang.org/grpc/otelgrpc).

### Bug 13 — High-cardinality span name
**Root cause:** Span names are aggregation keys in every backend. Baking `user_id` and `order_id` into the name produces unbounded distinct names, breaking percentile rollups and "top N slow operations" views.
**Fix:** Use the route template as the name; put IDs in attributes.
```python
with tracer.start_as_current_span("GET /users/{user_id}/orders/{order_id}") as span:
    span.set_attribute("user.id", user_id)
    span.set_attribute("order.id", order_id)
```
**Reference:** [OTel HTTP semantic conventions — span name](https://opentelemetry.io/docs/specs/semconv/http/http-spans/#name) — use the route template, not the raw path.

### Bug 14 — Resource attribute with pod hash
**Root cause:** `host.name` containing the rolling pod hash means every deployment creates a fresh `host.name` value, exploding cardinality on backends that index resource attributes (e.g. Tempo's `service.name` + `host.name` index).
**Fix:** Use stable resource attributes; put the pod-specific value in `k8s.pod.name` (where backends expect ephemeral cardinality).
```yaml
value: "service.name=orders,k8s.pod.name=$(HOSTNAME),k8s.namespace.name=$(NAMESPACE)"
```
**Reference:** [OTel resource semantic conventions for k8s](https://opentelemetry.io/docs/specs/semconv/resource/k8s/).

### Bug 15 — `tracestate` exceeds vendor budget
**Root cause:** W3C says vendors *should propagate at least 512 characters* and *should drop entries over 128 characters first* when truncating. A 2 KB `tracestate` will be silently truncated by intermediaries (CDNs, proxies), and you'll lose your own vendor's data first if it's the largest.
**Fix:** Keep vendor-specific debug data out of `tracestate`. It exists for vendor *correlation IDs*, not blobs.
**Reference:** [W3C Trace Context §3.3.1.2 tracestate length limits](https://www.w3.org/TR/trace-context/#tracestate-limits).

### Bug 16 — Span events on a hot path
**Root cause:** Each `AddEvent` allocates and the events live in the span until `End()`. 200k events per span produces multi-megabyte spans, GC pressure, and exporter rejections (OTLP collector default `max_recv_msg_size_mib` is 4).
**Fix:** Aggregate (a counter attribute), sample events, or split the work into child spans.
```go
span.SetAttributes(attribute.Int("rows.processed", len(rows)))
```
**Reference:** [OTel SDK span limits — `event_count_limit`](https://opentelemetry.io/docs/specs/otel/trace/sdk/#span-limits).

### Bug 17 — BatchSpanProcessor timeout too generous
**Root cause:** `scheduledDelayMillis: 30_000` plus `maxQueueSize: 50_000` keeps up to 30 seconds of spans in memory. With a 5-second termination grace period, every restart silently drops a queue's worth of spans. Worse on OOM-kills (no grace at all).
**Fix:** Tune to your grace period. A typical pairing:
```ts
new BatchSpanProcessor(exporter, {
  scheduledDelayMillis: 1_000,
  maxExportBatchSize: 512,
  maxQueueSize: 2_048,
  exportTimeoutMillis: 4_000,
});
```
Add a `SIGTERM` hook calling `forceFlush()` with a budget shorter than `terminationGracePeriodSeconds`.
**Reference:** [OTel SDK — BatchSpanProcessor defaults](https://opentelemetry.io/docs/specs/otel/trace/sdk/#batching-processor); see also [grafana.com — collector loss postmortem patterns](https://grafana.com/blog/2023/12/12/opentelemetry-best-practices-a-users-guide-to-getting-started-with-opentelemetry/).

### Bug 18 — Sampler runs after redaction
**Root cause:** `tail_sampling` decisions hash on `trace_id`. If the redaction processor mangles trace IDs (because they look like UUIDs/hex strings), the tail sampler sees a different ID per span and partitions the trace across decisions.
**Fix:** Order matters. Put `tail_sampling` *before* attribute redaction in the pipeline. Better, never redact trace IDs.
```yaml
processors:
  - tail_sampling
  - attributes/redact
```
**Reference:** [OTel collector pipeline order](https://opentelemetry.io/docs/collector/configuration/#service); [tail_sampling processor docs](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor).

### Bug 19 — Head sampling at the edge
**Root cause:** Head sampling at 0.1% means 99.9% of error traces are dropped before the error happens. The downstream service has no way to "rescue" a trace its parent already discarded — the sampling bit propagates.
**Fix:** Either (a) sample at 100% at the edge and use *tail sampling* in the collector to keep errors + a baseline; or (b) use an SDK that supports consistent probability sampling with rules (e.g. always-keep on `http.status_code >= 500`).
**Reference:** [Honeycomb — sampling strategies](https://www.honeycomb.io/blog/dynamic-sampling-by-example); [OTel tail_sampling processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor).

### Bug 20 — Single-instance tail sampler
**Root cause:** Tail sampling needs *all spans of a trace* on the same decider. A round-robin DNS to a fleet of collectors splits a trace's spans across pods; each pod sees a partial trace and makes the wrong decision (e.g. drops a trace that did contain an error in another pod).
**Fix:** Two-tier collectors: a "load balancing exporter" upfront keyed by `trace_id` routes all spans of a trace to the same backend collector instance.
```yaml
exporters:
  loadbalancing:
    routing_key: traceID
    protocol: { otlp: { tls: { insecure: true } } }
    resolver: { dns: { hostname: otelcol-tailsampling.observability.svc.cluster.local } }
```
**Reference:** [OTel collector — load balancing exporter for tail sampling](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/loadbalancingexporter).

### Bug 21 — Missing instrumentation across a queue
**Root cause:** Kafka, SQS, RabbitMQ, etc. don't propagate HTTP headers; `traceparent` must travel as a *message header* (Kafka headers, SQS message attributes). Without inject/extract on the queue boundary, every consumer span is a new root.
**Fix:**
```ts
// producer
const headers: Record<string, string> = {};
propagation.inject(context.active(), headers, {
  set: (carrier, k, v) => (carrier[k] = v),
});
await kafka.send({ topic: 'orders', messages: [{ value: payload, headers }] });
// consumer
const parentCtx = propagation.extract(context.active(), message.headers, {
  get: (carrier, k) => carrier[k]?.toString(),
  keys: (carrier) => Object.keys(carrier),
});
const span = tracer.startSpan('process-order', undefined, parentCtx);
```
**Reference:** [OTel messaging semantic conventions](https://opentelemetry.io/docs/specs/semconv/messaging/messaging-spans/).

### Bug 22 — Exporter blocks app thread on flush
**Root cause:** A 60-second flush inside a 10-second grace period guarantees `SIGKILL` halfway through. Worse, JVM shutdown hooks run synchronously; if your hook blocks on a slow OTLP endpoint, you lose *all* spans queued during that window plus delay process exit.
**Fix:** Bound the flush to the grace period minus a safety margin, and fail open:
```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
  tracerProvider.forceFlush().join(5, TimeUnit.SECONDS);
  tracerProvider.shutdown().join(2, TimeUnit.SECONDS);
}));
```
**Reference:** [OTel SDK — Shutdown / ForceFlush semantics](https://opentelemetry.io/docs/specs/otel/trace/sdk/#shutdown).

### Bug 23 — Recording high-cardinality attribute as a metric label
**Root cause:** Metrics are stored as one time-series per unique label combination. `http.target` with raw paths and `user.id` produces millions of series, costing both DB storage and per-series billing on hosted backends.
**Fix:** Use the route template (`http.route`) and drop `user.id` from metric labels; keep it on the trace.
```go
meter.Int64Counter("http.server.requests").Add(ctx, 1,
    metric.WithAttributes(
        attribute.String("http.route", route), // /users/{id}/orders/{id}
    ))
```
**Reference:** [OTel HTTP metrics conventions — `http.route` not raw target](https://opentelemetry.io/docs/specs/semconv/http/http-metrics/).

### Bug 24 — Probabilistic sampler ignores parent
**Root cause:** Without `ParentBased` wrapping, service B re-rolls per-trace at 10%, so 90% of traces marked sampled by service A get dropped at B. Trace views show truncated spans; correlation breaks.
**Fix:**
```ts
const sdk = new NodeSDK({
  sampler: new ParentBasedSampler({
    root: new TraceIdRatioBasedSampler(0.1),
  }),
});
```
**Reference:** [OTel SDK — ParentBased sampler delegation](https://opentelemetry.io/docs/specs/otel/trace/sdk/#parentbased); the root sampler only fires when there is no parent context.

---

## Related
- [OpenTelemetry Fundamentals](02-opentelemetry-fundamentals.md)
- [Distributed Tracing Internals](06-distributed-tracing-internals.md)
- [Three Pillars: Metrics, Logs, Traces](01-three-pillars-metrics-logs-traces.md)
- [Structured Logging](05-structured-logging.md)

## References
- OpenTelemetry Trace SDK specification — <https://opentelemetry.io/docs/specs/otel/trace/sdk/>
- OpenTelemetry Trace API specification — <https://opentelemetry.io/docs/specs/otel/trace/api/>
- OpenTelemetry HTTP semantic conventions (stable) — <https://opentelemetry.io/docs/specs/semconv/http/http-spans/>
- OpenTelemetry HTTP metrics conventions — <https://opentelemetry.io/docs/specs/semconv/http/http-metrics/>
- OpenTelemetry messaging conventions — <https://opentelemetry.io/docs/specs/semconv/messaging/messaging-spans/>
- OpenTelemetry resource k8s conventions — <https://opentelemetry.io/docs/specs/semconv/resource/k8s/>
- W3C Trace Context — <https://www.w3.org/TR/trace-context/>
- W3C Baggage — <https://www.w3.org/TR/baggage/>
- OTel collector contrib — tail_sampling processor — <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor>
- OTel collector contrib — load balancing exporter — <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/loadbalancingexporter>
- otelgrpc instrumentation — <https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation/google.golang.org/grpc/otelgrpc>
- Honeycomb — Dynamic sampling by example — <https://www.honeycomb.io/blog/dynamic-sampling-by-example>
- Honeycomb — Cardinality 101 — <https://www.honeycomb.io/blog/observability-101-cardinality>
- Honeycomb — Sensitive data in observability pipelines — <https://www.honeycomb.io/blog/sensitive-data-observability-pipelines>
- Grafana — OpenTelemetry best practices — <https://grafana.com/blog/2023/12/12/opentelemetry-best-practices-a-users-guide-to-getting-started-with-opentelemetry/>
