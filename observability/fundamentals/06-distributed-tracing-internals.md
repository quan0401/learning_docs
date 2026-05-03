---
title: "Distributed Tracing Internals — Spans, Context, Baggage, and Sampling"
date: 2026-05-03
updated: 2026-05-03
tags: [observability, distributed-tracing, dapper, w3c-trace-context, sampling, baggage, opentelemetry]
---

# Distributed Tracing Internals — Spans, Context, Baggage, and Sampling

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `observability` `distributed-tracing` `dapper` `w3c-trace-context` `sampling` `baggage` `opentelemetry`

---

## Table of Contents

- [Summary](#summary)
- [1. Origins: The Dapper Paper](#1-origins-the-dapper-paper)
- [2. Trace and Span Anatomy](#2-trace-and-span-anatomy)
  - [2.1 Identifiers](#21-identifiers)
  - [2.2 Parent / Child Relationships](#22-parent--child-relationships)
  - [2.3 Span Kinds](#23-span-kinds)
  - [2.4 Attributes, Events, Status, and Links](#24-attributes-events-status-and-links)
- [3. W3C Trace Context](#3-w3c-trace-context)
  - [3.1 The traceparent Header](#31-the-traceparent-header)
  - [3.2 The tracestate Header](#32-the-tracestate-header)
  - [3.3 Why W3C Replaced B3 / Jaeger Headers](#33-why-w3c-replaced-b3--jaeger-headers)
- [4. Baggage](#4-baggage)
  - [4.1 What Baggage Is](#41-what-baggage-is)
  - [4.2 The Baggage Header](#42-the-baggage-header)
  - [4.3 Caveats](#43-caveats)
- [5. Context Propagation Mechanics](#5-context-propagation-mechanics)
  - [5.1 In-Process Context](#51-in-process-context)
  - [5.2 Cross-Process Propagators](#52-cross-process-propagators)
  - [5.3 Across Async Boundaries](#53-across-async-boundaries)
  - [5.4 Across Messaging Systems](#54-across-messaging-systems)
- [6. Sampling Strategies](#6-sampling-strategies)
  - [6.1 Why Sample at All](#61-why-sample-at-all)
  - [6.2 Head-Based Sampling](#62-head-based-sampling)
  - [6.3 Probabilistic Sampling](#63-probabilistic-sampling)
  - [6.4 Rate-Limited Sampling](#64-rate-limited-sampling)
  - [6.5 Tail-Based Sampling](#65-tail-based-sampling)
  - [6.6 Choosing a Strategy](#66-choosing-a-strategy)
  - [6.7 Sampler Composition](#67-sampler-composition)
- [7. Common Failure Modes](#7-common-failure-modes)
- [8. Practical Patterns](#8-practical-patterns)
- [Related](#related)
- [References](#references)

---

## Summary

Distributed tracing exists because at a certain scale, you can no longer reason about request paths from logs alone. A single user click can fan out across 30 services, async queues, and shared databases; the only useful representation of "what happened" is a tree (or DAG) of timed work. The mechanics are simple in principle and subtle in practice: every service generates **spans** stamped with a shared **trace ID**, propagates context across the wire via standard headers, and decides via a **sampler** which traces to keep. This doc covers the data model, the W3C Trace Context propagation standard, baggage, the mechanics of context propagation across HTTP, gRPC, and messaging, and the trade-offs of head-based vs tail-based sampling.

---

## 1. Origins: The Dapper Paper

Modern distributed tracing descends from Sigelman et al.'s 2010 Google paper *Dapper, a Large-Scale Distributed Systems Tracing Infrastructure*. Dapper introduced:

- **Universally unique trace IDs** propagated through RPCs.
- **Spans** as the unit of timed work, with a parent/child tree.
- **Sampling** as a first-class concern: low overhead on the hot path, low storage downstream.
- **Annotations** (now "events" in OTel) for marking points in time inside a span.

Zipkin (Twitter, 2012) and HTrace (Cloudera) were direct open-source descendants. OpenZipkin's B3 propagation headers and Dapper-style spans seeded what became OpenTracing → OpenCensus → OpenTelemetry. Read the Dapper paper at least once — most of the design choices in modern tracing systems trace back to it.

---

## 2. Trace and Span Anatomy

### 2.1 Identifiers

| ID | Bits | Purpose |
|----|------|---------|
| `trace_id` | 128 (16 bytes, 32 hex) | Globally unique per request graph |
| `span_id` | 64 (8 bytes, 16 hex) | Unique per span within a trace |
| `parent_span_id` | 64 | Span ID of the immediate parent (omitted for root spans) |

Both IDs are random with sufficient entropy that collisions are negligible. The 128-bit trace ID matches W3C Trace Context; some older systems used 64-bit trace IDs (Zipkin originally), which is why you'll occasionally see 16-hex `trace_id` values in old data.

### 2.2 Parent / Child Relationships

A span has **at most one** parent. The set of spans sharing a `trace_id` forms a tree (or, with span links — see below — a DAG).

```
Trace 4bf92f3577b34da6a3ce929d0e0e4736

[checkout / POST /orders]                          span 00f067aa0ba902b7  (root)
   ├─[checkout / db.query users.findById]          span aae6b39ab4a9f3a1
   ├─[payments / POST /charge]                     span 73c2bf9d72e47a90  (kind=client)
   │    └─[payments-svc / POST /charge handler]    span 91ab... (kind=server, parent=73c2... in remote service)
   │         ├─[payments-svc / stripe.charges.create]  span ...
   │         └─[payments-svc / kafka.publish payments.charged]   span ... (kind=producer)
   └─[checkout / db.query orders.insert]           span ...
```

The root span has no parent. Each child span carries `parent_span_id` pointing at its parent.

### 2.3 Span Kinds

OpenTelemetry defines five kinds:

| Kind | When | Example |
|------|------|---------|
| `INTERNAL` | Default; work inside a single process | `computeTax()` |
| `SERVER` | Handling an inbound request | HTTP handler, gRPC service method |
| `CLIENT` | Making an outbound call | `fetch()`, JDBC, gRPC client |
| `PRODUCER` | Publishing to a queue/log | `kafka.publish` |
| `CONSUMER` | Receiving from a queue/log | Kafka consumer message processing |

The kind drives how UIs render arrows, how latencies are interpreted, and which semantic conventions apply.

### 2.4 Attributes, Events, Status, and Links

A span carries:

- **Attributes** — key/value pairs (`http.request.method`, `db.query.text`, custom keys). Use semantic conventions; don't invent names that already exist.
- **Events** (formerly "annotations") — timestamped key/value records inside a span: "cache miss", "circuit breaker opened", an exception with stack trace.
- **Status** — `UNSET`, `OK`, or `ERROR`. `ERROR` should carry an explanatory message.
- **Links** — references to other spans, in this trace or another, for cases where a single piece of work has multiple causes (batch processors fanning in N producer messages into one consumer span).

---

## 3. W3C Trace Context

### 3.1 The traceparent Header

The W3C Trace Context Recommendation defines a stable on-the-wire format. Every HTTP-aware OTel SDK injects `traceparent` by default:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ─┬ ─────────────┬───────────────── ────────┬───────── ─┬
              │              │                          │           │
              │              │                          │           └── trace-flags (8 bits, hex)
              │              │                          │                bit 0 = sampled (1) / not sampled (0)
              │              │                          └── parent-id (16 hex) — the calling span's span_id
              │              └── trace-id (32 hex)
              └── version (currently "00")
```

Receivers:

- Validate the version, trace-id ≠ all-zero, parent-id ≠ all-zero.
- Treat the parent-id as the parent of the next span they create.
- Honor the sampled flag where appropriate (see [Section 6](#6-sampling-strategies)).

### 3.2 The tracestate Header

`tracestate` is a sibling header that carries vendor-specific data along the trace, formatted as comma-separated key=value pairs, max 32 entries, max 512 bytes:

```
tracestate: rojo=00f067aa0ba902b7,congo=t61rcWkgMzE
```

Vendors prefix their keys (`congo=`, `dd=`, `aws=`) so multiple coexist without collisions. Common uses: vendor-specific sampling decisions, deployment IDs, internal correlation handles.

### 3.3 Why W3C Replaced B3 / Jaeger Headers

Before W3C Trace Context, the field was a Tower of Babel:

| Format | Headers | Owner |
|--------|---------|-------|
| **B3 (multi)** | `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-ParentSpanId`, `X-B3-Sampled` | Zipkin / OpenZipkin |
| **B3 (single)** | `b3: traceId-spanId-sampled-parentSpanId` | Zipkin |
| **Jaeger** | `uber-trace-id: traceId:spanId:parentId:flags` | Uber/Jaeger |
| **AWS X-Ray** | `X-Amzn-Trace-Id` | AWS |
| **Datadog** | `x-datadog-trace-id`, `x-datadog-parent-id`, `x-datadog-sampling-priority` | Datadog |

A multi-vendor system had to inject and extract all of them. W3C Trace Context (2020 recommendation) gives one canonical pair (`traceparent`, `tracestate`) every conforming system speaks. OTel SDKs default to W3C and let you also configure B3 / Jaeger for backward compatibility.

---

## 4. Baggage

### 4.1 What Baggage Is

**Baggage** is a separate W3C-standardized mechanism for propagating arbitrary key/value pairs alongside the trace context, intended for **business attributes** that should travel with the request:

- `tenant_id`
- `user_tier=premium`
- `experiment.checkout=v3`
- `region=eu-west-1`

Baggage is **not** part of the trace's identity — it's a sidecar dictionary that propagates with the request so any service in the chain can read it without re-deriving it.

### 4.2 The Baggage Header

```
baggage: tenant_id=acme,user_tier=premium,experiment.checkout=v3
```

Maximum total size around 8192 bytes per the spec; per-entry limits also apply.

### 4.3 Caveats

- **Baggage values cross trust boundaries.** A user-controlled service (or a partner) on the path can read and tamper with baggage. Never put secrets, never trust unsigned values, and consider stripping baggage at trust boundaries.
- **Baggage adds bytes to every downstream HTTP request.** Keep it small. Don't put a serialized object in there.
- **Baggage entries are not automatically span attributes.** If you want them on spans, attach them explicitly. OTel has a `BaggageSpanProcessor` that copies allow-listed baggage keys onto spans.

---

## 5. Context Propagation Mechanics

### 5.1 In-Process Context

Within a single process, the active span lives in a `Context` object that follows execution flow:

- **Java** — `io.opentelemetry.context.Context`, propagated through thread-locals, with explicit `try (Scope s = context.makeCurrent())` blocks.
- **Node.js** — `@opentelemetry/api` `context.active()`, propagated through `AsyncLocalStorage` (the default `AsyncHooksContextManager`).
- **Go** — explicit `context.Context` parameter on every function (idiomatic Go).
- **Python** — `contextvars` (PEP 567).

### 5.2 Cross-Process Propagators

A propagator knows how to **inject** the active context into a carrier (HTTP headers, gRPC metadata, message headers) and **extract** it on the other side.

```typescript
// inject (client side)
import { propagation, context } from '@opentelemetry/api';
const headers: Record<string, string> = {};
propagation.inject(context.active(), headers);
// headers now has { traceparent: "...", tracestate: "...", baggage: "..." }

// extract (server side)
const ctx = propagation.extract(context.active(), incomingHeaders);
// ctx is now the parent context for the next span
```

OTel's default global propagator is a composite: W3C Trace Context + W3C Baggage. You can swap it for B3, Jaeger, or composite-with-extras during a migration:

```bash
OTEL_PROPAGATORS=tracecontext,baggage,b3multi
```

### 5.3 Across Async Boundaries

Async work (`setTimeout`, `setImmediate`, thread pools, Reactor schedulers, executors) breaks naïve context propagation unless the runtime hooks it.

- **Node** — `AsyncLocalStorage` covers `await`, `setTimeout`, `setImmediate`, `process.nextTick`, `Promise` callbacks. Worker threads need explicit context capture.
- **Java** — `Context.current().wrap(runnable)` or `Context.taskWrapping(executor)` for `ExecutorService`. Reactor 3.5+ + Micrometer Context Propagation handle `Mono`/`Flux` automatically when configured.
- **Kotlin coroutines** — `withContext(currentCoroutineContext().asContextElement())` patterns; the `kotlinx-coroutines-context-propagation` integrations handle it.

If a span starts under a `trace_id` and its child spans somehow have a different `trace_id`, you have an async propagation gap. The fix is at the runtime layer, not in business code.

### 5.4 Across Messaging Systems

For Kafka/RabbitMQ/SQS/SNS, the propagator injects into **message headers**:

- Producer side: `propagation.inject(ctx, messageHeaders)` and emit a `PRODUCER` span.
- Consumer side: `propagation.extract(ctx, messageHeaders)` and emit a `CONSUMER` span as the child (or with a span link if processing is decoupled).

Span links matter here: a consumer that batches N producer messages into one logical operation should use links to each producer span, not pretend it has a single parent.

---

## 6. Sampling Strategies

### 6.1 Why Sample at All

Tracing every request at every span is unaffordable for any non-trivial service. A 1k-RPS service emitting 10 spans per request at 500 bytes per span is ~5 MB/s of trace data sustained — 13 TB/month, before retention multipliers. Sampling is the lever.

### 6.2 Head-Based Sampling

Decision is made **at trace start** (the root span). The decision is encoded into the `traceparent` `trace-flags` field (sampled bit) and propagates downstream. All services on the trace agree.

Pros: cheap, simple, predictable, works without infrastructure.

Cons: you sample blindly — the decision predates the outcome, so interesting traces (errors, slow requests) are sampled at the same rate as boring ones.

### 6.3 Probabilistic Sampling

A flavor of head-based sampling: sample with probability `p`. Compute deterministically from `trace_id` so all services in the trace agree without coordination.

```bash
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1   # 10% of new traces
```

`parentbased_*` means "if there's an inbound parent decision, respect it; otherwise sample at this rate." This is the recommended default — it preserves trace completeness across services.

### 6.4 Rate-Limited Sampling

Cap the absolute rate of sampled traces (e.g., max 100/sec). Cheap protection against runaway sampling cost during traffic spikes. Often combined with probabilistic sampling.

### 6.5 Tail-Based Sampling

Decision is made **after the trace completes**, in a buffering Collector. Examples of tail policies:

- Keep all traces with at least one ERROR span.
- Keep all traces with root duration above the 99th percentile.
- Keep all traces touching a specific service or route.
- Keep 1% of everything else.

Pros: signal-rich. The slow trace, the failed trace — you keep them all. The boring trace gets dropped.

Cons:

- Requires the OTel Collector (or vendor equivalent) with `tailsamplingprocessor`.
- Buffer time (typically 10–60 s) — the Collector waits for all spans of a trace before deciding. This pushes trace ingest latency from "near real-time" to "minute-ish".
- Memory pressure: the Collector must hold every in-flight trace in RAM.
- Multi-collector setups need **trace-id-aware load balancing** so all spans of one trace land on the same Collector instance.

### 6.6 Choosing a Strategy

| Situation | Recommended |
|-----------|-------------|
| Just getting started | Head-based, parent-based, 100% (or 10–25% if RPS is high) |
| Production cost control | Head-based at p, tune p to budget |
| Want all errors and slow traces | Tail-based (Collector) |
| Multi-cluster, mixed vendors | Combine: head-based at SDK + tail at Collector |
| Per-route sensitivity | Use a sampler per route (some SDKs allow this) or apply rules at Collector |

### 6.7 Sampler Composition

OTel allows composing samplers. A common pattern:

```
ParentBased(
  root = ConsistentRatio(0.05),               # 5% of new traces
  remoteParentSampled = AlwaysOn,             # respect upstream "yes"
  remoteParentNotSampled = AlwaysOff,         # respect upstream "no"
  localParent* = match parent decision
)
```

For force-sampling specific routes (e.g., "/admin", "/healthz NEVER, error replays ALWAYS"), use a `Jaeger remote sampler` config or a custom sampler.

---

## 7. Common Failure Modes

- **Broken traces (orphan spans).** A service logs spans with a `parent_span_id` that points at a span no other service emitted. Root cause: a hop dropped `traceparent` (legacy proxy), or async context was lost and a new trace was started.
- **Two trace IDs for one user request.** Async work or retries created a fresh trace. Fix the propagation in the worker or use span links.
- **Sampler disagreement.** SDK said "sampled", Collector tail-sampler dropped the trace. Confusion when only some spans show. Decide on one location for sampling decisions.
- **Cardinality explosion via attributes.** Setting `user.id` and `request.id` on every span is fine for traces, but if a vendor backend indexes them as label-style facets, costs balloon. Most trace backends are columnar and tolerate high cardinality, but check yours.
- **Deep span trees with no useful detail.** Hundreds of internal spans for one request, all `INTERNAL` kind, no semantic attributes. Spans should be meaningful units of work.
- **Missing async instrumentation.** Workers, schedulers, and queue consumers run with no trace context. Either explicitly propagate or accept that those code paths are dark.
- **Logging trace IDs that don't match.** Your log says `trace_id=abc` but the trace database has `trace_id=def`. Mismatched logger and tracer SDKs, or the logger is not bridged to OTel.

---

## 8. Practical Patterns

- **Span at every external call.** HTTP, gRPC, DB, cache, queue. Auto-instrumentation usually covers this; verify.
- **One business span per request handler.** Above the framework's auto-span, add a domain-named span (`checkout.processOrder`) with business attributes (`order.line_count`, `cart.total_usd`).
- **Always record exceptions.** `span.recordException(err)` + `span.setStatus(ERROR)`. Otherwise the trace looks fine but the request failed.
- **Use events for moments inside a span.** `span.addEvent("cache.miss", { key })` is cheaper than starting a child span and easier to read for "instantaneous" moments.
- **Tail-sample errors and slow traces.** Even if you head-sample at 1%, keep 100% of failures. The OTel Collector tail-sampling processor is the default tool.
- **Strip secrets in the Collector.** Do not rely on application code being perfect about not putting tokens in span attributes.
- **Test propagation.** Add an integration test that runs across two services and asserts a single `trace_id` across spans.

---

## Related

- [Three Pillars: Metrics, Logs, Traces](01-three-pillars-metrics-logs-traces.md)
- [OpenTelemetry Fundamentals](02-opentelemetry-fundamentals.md)
- [Structured Logging](05-structured-logging.md)
- [Java Distributed Tracing](../../java/observability/distributed-tracing.md)
- [Reactive Observability (Micrometer + Reactor)](../../java/reactive-observability.md)
- [Node.js Production Observability](../../typescript/production/observability.md)
- [System Design: Tracing, Metrics, Logs](../../system-design/performance-observability/tracing-metrics-logs.md)
- [Network Observability](../../networking/advanced/network-observability.md)

---

## References

- **Sigelman, Barroso, Burrows, Stephenson, Plakal, Beaver, Jaspan, Shanbhag — "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure"** (Google, 2010). https://research.google/pubs/dapper-a-large-scale-distributed-systems-tracing-infrastructure/
- **W3C — Trace Context (Recommendation, 2020).** https://www.w3.org/TR/trace-context/
- **W3C — Baggage (Working Draft).** https://www.w3.org/TR/baggage/
- **OpenTelemetry — Tracing specification.** https://opentelemetry.io/docs/specs/otel/trace/api/
- **OpenTelemetry — Sampling.** https://opentelemetry.io/docs/concepts/sampling/
- **OpenTelemetry Collector — `tailsamplingprocessor`.** https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor
- **OpenZipkin — B3 Propagation.** https://github.com/openzipkin/b3-propagation
- **Charity Majors / Honeycomb — "An Introduction to Distributed Tracing."** https://www.honeycomb.io/blog/distributed-tracing-a-modern-introduction
- **Cindy Sridharan — *Distributed Systems Observability*** (O'Reilly free e-book). https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/
- **Yuri Shkuro — *Mastering Distributed Tracing*** (Packt, 2019). The author of Jaeger.
