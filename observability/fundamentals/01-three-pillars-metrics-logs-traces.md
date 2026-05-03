---
title: "The Three Pillars: Metrics, Logs, and Traces — and Why That Model Is Imperfect"
date: 2026-05-03
updated: 2026-05-03
tags: [observability, metrics, logs, traces, events, cardinality, sampling]
---

# The Three Pillars: Metrics, Logs, and Traces — and Why That Model Is Imperfect

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `observability` `metrics` `logs` `traces` `events` `cardinality` `sampling`

---

## Table of Contents

- [Summary](#summary)
- [1. Where the "Three Pillars" Come From](#1-where-the-three-pillars-come-from)
- [2. Metrics](#2-metrics)
  - [2.1 What a Metric Is](#21-what-a-metric-is)
  - [2.2 Strengths and Weaknesses](#22-strengths-and-weaknesses)
  - [2.3 Cardinality, Bluntly](#23-cardinality-bluntly)
  - [2.4 Cost Model](#24-cost-model)
- [3. Logs](#3-logs)
  - [3.1 What a Log Is](#31-what-a-log-is)
  - [3.2 Strengths and Weaknesses](#32-strengths-and-weaknesses)
  - [3.3 Sampling and Cost Control](#33-sampling-and-cost-control)
- [4. Traces](#4-traces)
  - [4.1 What a Trace Is](#41-what-a-trace-is)
  - [4.2 Strengths and Weaknesses](#42-strengths-and-weaknesses)
  - [4.3 Head- vs Tail-Based Sampling](#43-head--vs-tail-based-sampling)
- [5. Side-by-Side Comparison](#5-side-by-side-comparison)
- [6. Why the Three-Pillar Model Is Imperfect](#6-why-the-three-pillar-model-is-imperfect)
  - [6.1 Charity Majors' Critique](#61-charity-majors-critique)
  - [6.2 Wide Structured Events](#62-wide-structured-events)
  - [6.3 The "Fourth Pillar": Continuous Profiling](#63-the-fourth-pillar-continuous-profiling)
- [7. Correlation Across Signals](#7-correlation-across-signals)
  - [7.1 The trace\_id Backbone](#71-the-trace_id-backbone)
  - [7.2 Exemplars: Linking Metrics to Traces](#72-exemplars-linking-metrics-to-traces)
  - [7.3 Practical Correlation Pattern](#73-practical-correlation-pattern)
- [8. Choosing the Right Signal for the Question](#8-choosing-the-right-signal-for-the-question)
- [Related](#related)
- [References](#references)

---

## Summary

"Metrics, logs, and traces" is the canonical pitch deck for observability tools, and it is a useful first taxonomy. Each signal has a sharp edge and a soft side: metrics are cheap and fast but lossy, logs are expressive but expensive and structurally weak at scale, traces are excellent for causality but biased by sampling. The model is also incomplete and partially wrong: many practitioners (notably Charity Majors at Honeycomb) argue that observability is really about **wide, high-cardinality structured events** and that "three pillars" mistakes data formats for an ontology. Continuous profiling is sometimes described as a fourth pillar. This document defines each pillar precisely, names the failure modes, and shows how `trace_id` is the connective tissue that ties them together during incidents.

---

## 1. Where the "Three Pillars" Come From

The phrasing was popularized around 2017 by Peter Bourgon's blog post *Metrics, tracing, and logging* and became standard shorthand once vendors adopted it. It maps cleanly onto the three families of open-source projects most teams reach for:

| Pillar | Canonical OSS stack |
|--------|---------------------|
| Metrics | Prometheus, Graphite, StatsD, OpenTelemetry Metrics |
| Logs | syslog, ELK / OpenSearch, Loki, Fluent Bit |
| Traces | Jaeger, Zipkin, Tempo, OpenTelemetry Traces |

The model is useful for shopping, less useful as an engineering ontology — see [Section 6](#6-why-the-three-pillar-model-is-imperfect).

---

## 2. Metrics

### 2.1 What a Metric Is

A **metric** is a numerical measurement aggregated over time, identified by a name and a set of dimensions (labels). The wire-level shape is roughly:

```
http_request_duration_seconds_bucket{
  service="checkout",
  method="POST",
  route="/orders",
  status="200",
  le="0.1"
} 14823 @ 1714694400
```

Common metric types (Prometheus / OpenTelemetry):

- **Counter** — monotonically increasing (requests served, bytes sent).
- **Gauge** — point-in-time value that goes up and down (queue depth, connection pool size, heap usage).
- **Histogram** — pre-bucketed distribution; supports quantile estimation server-side.
- **Summary** — quantiles computed client-side; fixed quantiles, hard to aggregate.

### 2.2 Strengths and Weaknesses

**Strengths.**

- Constant cost per series regardless of traffic — a counter ticking 1 M times costs the same as ticking 1 K times.
- Cheap to scrape, cheap to store as a time series.
- Predictable latency for queries; suitable for fast dashboards and alerting.
- Aggregation is mathematically sound across infinite time windows.

**Weaknesses.**

- **Pre-aggregation is lossy.** Once you commit to "p99 latency by route × status," you cannot ask "what about user_id=42 specifically" after the fact.
- **No per-request context.** A spike in p99 tells you something is slow; it does not tell you which request, which user, which downstream call.
- **High-cardinality labels destroy them.** See [2.3](#23-cardinality-bluntly).

### 2.3 Cardinality, Bluntly

Each unique combination of label values is a separate **time series**. If a metric has labels `service` (10 values) × `route` (50) × `status` (5) × `region` (5), that is 12,500 series. Add `user_id` (1 M) and you have 12.5 B series. Prometheus and most TSDBs will fall over.

Rules of thumb (Prometheus docs):

- Keep label cardinality bounded and **known in advance**.
- Avoid putting unbounded values in labels: `user_id`, `request_id`, `email`, `path` (with path params), full URLs, error messages.
- For unbounded contextual data, use **logs or traces** (or [exemplars](#72-exemplars-linking-metrics-to-traces)).

### 2.4 Cost Model

Cost is roughly proportional to **active series count × scrape interval × retention**, plus query time. SaaS vendors charge per series-month or per ingested data point. A metric is cheap **only if** its cardinality is controlled.

---

## 3. Logs

### 3.1 What a Log Is

A **log** is an immutable, timestamped record of an event. Free-text logs are the legacy form; modern systems emit **structured logs** as JSON or key-value pairs:

```json
{
  "ts": "2026-05-03T14:22:01.482Z",
  "level": "warn",
  "service": "checkout",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "user_id": "u_8821",
  "msg": "payment retry",
  "attempt": 2,
  "provider": "stripe",
  "elapsed_ms": 184
}
```

See [Structured Logging](05-structured-logging.md) for the full pattern.

### 3.2 Strengths and Weaknesses

**Strengths.**

- Highest expressive power: arbitrary fields, full context, free-form messages.
- Naturally captures rare, unique events (one-off errors with full stack traces).
- Easy to emit from anywhere — every language, every framework.

**Weaknesses.**

- **Cost scales with traffic.** 10x the requests means roughly 10x the log volume, every byte ingested, indexed, stored, queried.
- **Indexing is expensive.** Full-text search engines (Elasticsearch / OpenSearch) need substantial RAM and disk to keep indexes hot.
- **Schema drift is the norm.** Without discipline, fields creep, types collide, dashboards break.
- **Cardinality has no natural cap** — which is both a feature and the source of the cost.

### 3.3 Sampling and Cost Control

At scale, ingesting every log line is uneconomical. Common controls:

| Technique | Description | Trade-off |
|-----------|-------------|-----------|
| **Level filtering** | Drop DEBUG/TRACE in production | Loses detail when you most need it |
| **Probabilistic sampling** | Keep N% of INFO logs | Statistical, biased toward common events |
| **Adaptive sampling** | Sample less when error rate is low | Complex, requires feedback loop |
| **Tail-based** (rarely) | Decide after request completes | Requires buffering |
| **Aggregation at the edge** | Convert repetitive logs to a counter | Loses individual instances |
| **Structured + columnar storage** | Loki, ClickHouse | Cheaper but different query model |

A pragmatic rule: **always keep WARN and ERROR; sample INFO; only emit DEBUG behind a flag**.

---

## 4. Traces

### 4.1 What a Trace Is

A **trace** is a directed acyclic graph of **spans**. Each span records a unit of work — an HTTP request handler, a SQL query, a Kafka publish — with start/end timestamps, attributes, and a parent reference. A trace is the transitive closure of all spans sharing a `trace_id`.

```
trace_id = 4bf92f3577b34da6a3ce929d0e0e4736

[checkout: POST /orders                              420 ms ]
   ├─[checkout: validate-cart                          5 ms ]
   ├─[checkout: db.query users.findById              22 ms ]
   ├─[payments: POST /charge                         310 ms ]
   │    ├─[payments: stripe.charges.create          290 ms ]
   │    └─[payments: kafka.publish payments.charged  6 ms ]
   └─[checkout: db.query orders.insert               18 ms ]
```

Full mechanics in [Distributed Tracing Internals](06-distributed-tracing-internals.md).

### 4.2 Strengths and Weaknesses

**Strengths.**

- The only signal that **shows causality** across service boundaries.
- Pinpoints where in a request graph latency or errors actually occur.
- High-cardinality by design — every span carries arbitrary attributes.

**Weaknesses.**

- Volume is enormous if unsampled (every request × every span × every attribute).
- **Sampling bias**: the rare, slow, broken request is exactly the one you want to see, and it may have been sampled out.
- Instrumentation gaps cause **broken traces** (missing parent, orphan spans).
- Async work (queues, scheduled jobs, background workers) breaks naive context propagation.

### 4.3 Head- vs Tail-Based Sampling

- **Head-based** (probabilistic, decided at trace start): cheap, simple, but throws away interesting outliers along with normal traffic.
- **Tail-based** (decided after the trace completes): keep all errors and slow traces; drop most fast successes. Requires a buffering layer (the OpenTelemetry Collector's `tailsamplingprocessor`).

The OpenTelemetry Collector documentation describes the trade-offs and a typical tail-sampling policy: keep 100% of traces with errors, 100% of traces above a latency threshold, and 1% of remaining traffic. See [Distributed Tracing Internals](06-distributed-tracing-internals.md) for details.

---

## 5. Side-by-Side Comparison

| Property | Metrics | Logs | Traces |
|----------|---------|------|--------|
| **Shape** | Numerical, aggregated | Discrete events with text + fields | DAG of spans |
| **Cardinality** | Must be bounded | Unbounded | Unbounded per trace |
| **Cost vs traffic** | Constant per series | Linear | Linear (mitigated by sampling) |
| **Best at** | Fast aggregate questions, alerting, dashboards | Detailed event records, post-hoc forensics | Cross-service causality |
| **Worst at** | Per-request detail | Aggregate questions over high volumes | Aggregate questions, full coverage |
| **Typical retention** | 13–30 days raw, downsampled longer | 7–30 days hot | 3–14 days |
| **Sampling needed?** | No | Often | Almost always |
| **Query latency target** | ms–seconds | seconds | seconds |
| **Schema** | Strict (name + labels) | Loose, drift-prone | Loose, but standard semantic conventions |

---

## 6. Why the Three-Pillar Model Is Imperfect

### 6.1 Charity Majors' Critique

Charity Majors (co-founder of Honeycomb) has argued repeatedly that the three-pillar framing confuses **storage formats** with **observability properties**. Her core points (paraphrased from Honeycomb's blog and her O'Reilly book *Observability Engineering*, co-authored with George Miller and Liz Fong-Jones):

- **"Pillars" suggests three separate systems with three separate budgets and three separate workflows.** In practice you spend your incident time hopping between tabs, copy-pasting `trace_id`s, and re-asking the same question in three query languages.
- The thing you actually need is the ability to ask **arbitrary, high-cardinality, high-dimensionality questions** against your production data, in a tight feedback loop. That is a property, not a pillar.
- Aggregated metrics are intrinsically lossy. Logs, when structured wide enough, can answer most metric-style questions.
- The "unknown unknowns" of distributed systems require exploration, not pre-canned dashboards.

### 6.2 Wide Structured Events

The alternative framing: emit **one wide, structured event per unit of work** (per HTTP request, per job, per message processed) with as many fields as relevant — request size, user tier, region, feature flags on, downstream call counts and latencies, error class. Metrics, traces, and logs become **derivations** of this richer underlying event stream.

Storage technologies built for wide events (Honeycomb, Datadog Wide Events, ClickHouse-backed setups) treat cardinality as a first-class concern instead of a budget hazard.

This does not invalidate the three-pillar tools — Prometheus is still excellent for what it does — but it reframes the goal: **rich events first, derived signals second**.

### 6.3 The "Fourth Pillar": Continuous Profiling

Continuous profiling — always-on CPU and memory profiles sampled at low overhead, attached to a service identity over time — has been called a fourth pillar by Grafana (Pyroscope), Polar Signals (Parca), and Datadog. It answers questions metrics, logs, and traces cannot: *which lines of code are burning CPU on this pod right now*. See planned Tier 5 of this index.

---

## 7. Correlation Across Signals

### 7.1 The trace\_id Backbone

`trace_id` is the universal join key. Once a request enters a system and gets a trace, every downstream signal — every span, every structured log, every exemplar attached to a metric — should carry that ID. This is what makes metric → log → trace navigation possible during an incident.

The propagation mechanics are defined by the W3C Trace Context spec (`traceparent` HTTP header), covered in [Distributed Tracing Internals](06-distributed-tracing-internals.md).

### 7.2 Exemplars: Linking Metrics to Traces

An **exemplar** is a sampled trace ID attached to a histogram bucket. When a request takes 1.3 seconds and falls into the `le=2` bucket, Prometheus can store one example trace ID along with that observation. Then Grafana can render exemplar dots on a histogram chart, click-through to Tempo or Jaeger.

OpenMetrics specifies the exemplar format. Prometheus supports it natively; the OpenTelemetry SDK emits exemplars with trace context attached.

```
http_request_duration_seconds_bucket{route="/orders",le="2"} 5821 # {trace_id="4bf92f3577b34da6a3ce929d0e0e4736"} 1.32 1714694400.482
```

### 7.3 Practical Correlation Pattern

A working incident loop:

1. Alert fires on a metric (e.g., RED error-rate SLO burn).
2. Click the exemplar on the offending histogram bar → open the trace in Tempo/Jaeger.
3. Find the failing span → copy its `trace_id`.
4. Pivot to logs: `service="checkout" AND trace_id="..."` → see the full structured log context for that request.
5. Pivot to a profile: `service="checkout"` over the same window → check CPU/heap.

Each pillar answers a slightly different shape of question; the join key is what makes the loop fast.

---

## 8. Choosing the Right Signal for the Question

| Question | Best signal |
|----------|-------------|
| "What is current p99 latency by route?" | Metric (histogram) |
| "Has error rate breached the SLO budget?" | Metric + alert |
| "Why is *this specific* checkout slow right now?" | Trace |
| "What did the system log when user u_8821 paid yesterday?" | Log (filtered by user_id + trace_id) |
| "Which service is the bottleneck in this request graph?" | Trace |
| "Is my heap leaking?" | Metric (gauge) → confirm with profile |
| "Which function is burning CPU on pod-7?" | Continuous profile |
| "Did any request retry the payment provider three times?" | Wide event / log query |
| "How many of my paid-tier users hit a 5xx in the last 30 minutes?" | Wide event (logs with structured fields) — metrics can't, because `user_tier × user_id` cardinality is too high |

When a question lives in the seam between two signals, that is usually a sign you need a wide-event store or better correlation, not a fourth tool.

---

## Related

- [OpenTelemetry Fundamentals](02-opentelemetry-fundamentals.md)
- [Structured Logging](05-structured-logging.md)
- [Distributed Tracing Internals](06-distributed-tracing-internals.md)
- [Prometheus and PromQL](07-prometheus-and-promql.md)
- [Spring Boot Actuator Deep Dive](../../java/actuator-deep-dive.md)
- [Node.js Production Observability](../../typescript/production/observability.md)
- [System Design: Tracing, Metrics, Logs](../../system-design/performance-observability/tracing-metrics-logs.md)
- [Network Observability (eBPF, RED)](../../networking/advanced/network-observability.md)

---

## References

- **Charity Majors, Liz Fong-Jones, George Miller — *Observability Engineering*** (O'Reilly, 2022). Chapter 1 ("What Is Observability?") and Chapter 8 ("Analyzing Events to Achieve Observability") make the case for wide events. https://www.oreilly.com/library/view/observability-engineering/9781492076438/
- **Charity Majors — "Observability — A 3-Year Retrospective"** (Honeycomb blog). https://www.honeycomb.io/blog/observability-3-year-retrospective
- **Peter Bourgon — "Metrics, tracing, and logging"** (peter.bourgon.org, 2017). The post that crystallized the three-pillar phrasing. https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html
- **OpenTelemetry — Signals overview** (opentelemetry.io). https://opentelemetry.io/docs/concepts/signals/
- **Prometheus — "Naming and labels" / cardinality guidance** (prometheus.io). https://prometheus.io/docs/practices/naming/
- **OpenMetrics — Exemplars specification.** https://github.com/OpenObservability/OpenMetrics/blob/main/specification/OpenMetrics.md#exemplars
- **Grafana Labs — "Exemplars in Grafana."** https://grafana.com/docs/grafana/latest/fundamentals/exemplars/
- **Sigelman et al. — "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure"** (Google, 2010). The foundational distributed-tracing paper. https://research.google/pubs/dapper-a-large-scale-distributed-systems-tracing-infrastructure/
- **Google SRE Book — Beyer, Jones, Petoff, Murphy (eds.), *Site Reliability Engineering*** (O'Reilly, 2016), Chapter 6 ("Monitoring Distributed Systems"). https://sre.google/sre-book/monitoring-distributed-systems/
- **Brendan Gregg — "Continuous Profiling and Observability."** https://www.brendangregg.com/blog/2024-03-17/continuous-profiling-and-observability.html
