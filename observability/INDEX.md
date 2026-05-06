# Observability Documentation Index — Learning Path

A progressive path from observability fundamentals (the three pillars, OpenTelemetry, SLOs) through instrumentation, backends, alerting, and continuous profiling. Practical and backend-oriented — connects observability theory to what you actually instrument, query, and respond to as a TypeScript/Node or Java/Spring Boot developer.

Cross-references to the [Java learning path](../java/INDEX.md) (Spring Boot Actuator, Micrometer, distributed tracing), the [TypeScript learning path](../typescript/INDEX.md) (Node profiling, production observability), the [Go learning path](../golang/INDEX.md) (`runtime/pprof`, `runtime/trace`, `slog`, OpenTelemetry-go), the [Networking learning path](../networking/INDEX.md) (network-level observability and eBPF), the [Kubernetes learning path](../kubernetes/INDEX.md) (cluster monitoring and logging), the [System Design learning path](../system-design/INDEX.md) (RED/USE, dashboards, runbooks), and the [Web Scraping learning path](../web-scraping/INDEX.md) (block-rate by ASN, freshness SLOs, scraper-fleet tracing) where topics overlap.

**Markers:** **★** = core must-learn (everyday backend instrumentation work, what you reach for during incidents, common in interviews and SRE conversations). **○** = supporting deep-dive (more SRE/ops territory or specialized advanced topics). Internalize all ★ before going deep on ○.

---

## Tier 1 — Foundations

The mental models and primitives every observability system rests on. Pillars, OTel, SLOs, structured logs, traces, and Prometheus.

1. [★ The Three Pillars: Metrics, Logs, and Traces — and Why That Model Is Imperfect](fundamentals/01-three-pillars-metrics-logs-traces.md) — what each signal is good and bad at, cardinality, sampling, cost, the case for events/wide structured logs, and correlation by `trace_id` _(2026-05-03)_
2. [★ OpenTelemetry Fundamentals — API, SDK, Collector, and OTLP](fundamentals/02-opentelemetry-fundamentals.md) — signals, resource semantic conventions, OTLP wire protocol, auto vs manual instrumentation, exporter patterns, vendor-neutral pipelines, Node and Spring Boot examples _(2026-05-03)_
3. [★ SLI / SLO / Error Budgets — Measuring What Users Care About](fundamentals/03-sli-slo-error-budgets.md) — choosing SLIs, SLO targets, error budget math, burn-rate alerts, multi-window multi-burn-rate, worked examples from the Google SRE Book and *Implementing SLOs* _(2026-05-03)_
4. [★ RED, USE, and the Four Golden Signals](fundamentals/04-red-and-use-methods.md) — Tom Wilkie's RED for request-driven services, Brendan Gregg's USE for resources, the SRE Book's Four Golden Signals, and how they overlap on real dashboards _(2026-05-03)_
5. [★ Structured Logging — JSON, Levels, Correlation, and Cost Control](fundamentals/05-structured-logging.md) — semantic fields, ECS, log levels, correlation IDs via `trace_id`/`span_id`, sampling, PII scrubbing, pino/winston/Logback/SLF4J examples _(2026-05-03)_
6. [★ Distributed Tracing Internals — Spans, Context, Baggage, and Sampling](fundamentals/06-distributed-tracing-internals.md) — span/trace anatomy, W3C Trace Context (`traceparent`/`tracestate`), baggage, head vs tail sampling, the Dapper paper, span attributes/events _(2026-05-03)_
7. [★ Prometheus and PromQL — Pull Model, Histograms, and Alert Rules](fundamentals/07-prometheus-and-promql.md) — scrape config, exposition format, counter/gauge/histogram/summary, `rate`/`irate`/`histogram_quantile`, recording and alerting rules, federation, remote write _(2026-05-03)_

---

## Tier 2 — Instrumentation in Practice _(planned)_

Hands-on instrumentation for Node and Spring Boot services: HTTP middleware, database tracing, Kafka consumers, async context propagation, custom metrics, and the trade-offs of auto-instrumentation.

## Tier 3 — Backends and Storage _(planned)_

How metrics/logs/traces are actually stored and queried: Prometheus TSDB internals, Mimir/Thanos for horizontal scale, Loki and ELK for logs, Tempo/Jaeger for traces, columnar stores like ClickHouse for wide events.

## Tier 4 — Alerting and On-Call _(planned)_

From signal to page: alert design, symptom-based alerting, alert fatigue, notification routing (Alertmanager, PagerDuty), runbooks, postmortems, and the on-call lifecycle.

## Tier 5 — Continuous Profiling and eBPF Observability _(planned)_

Always-on profilers (Pyroscope, Parca, Datadog Continuous Profiler), pprof for Go and async-profiler for the JVM, flame graphs at scale, and eBPF-based observability (Pixie, Cilium Hubble, parca-agent).

---

## Quick Reference by Topic

### Foundations and Mental Models

- [Three Pillars: Metrics, Logs, Traces](fundamentals/01-three-pillars-metrics-logs-traces.md)
- [OpenTelemetry Fundamentals](fundamentals/02-opentelemetry-fundamentals.md)
- [SLI / SLO / Error Budgets](fundamentals/03-sli-slo-error-budgets.md)
- [RED, USE, and Four Golden Signals](fundamentals/04-red-and-use-methods.md)

### Signals

- [Structured Logging](fundamentals/05-structured-logging.md)
- [Distributed Tracing Internals](fundamentals/06-distributed-tracing-internals.md)
- [Prometheus and PromQL](fundamentals/07-prometheus-and-promql.md)

---

## Bug Spotting

Active-recall practice docs. Each presents 22+ broken snippets organized by difficulty (Easy / Subtle / Senior trap), with one-line `<details>` hints inline and full root-cause + fix in a Solutions section. Every bug cites a real reference (RFC, CVE, official-doc gotcha, postmortem, library issue). Use these to pressure-test concept knowledge after working through the tiers above.

- [Distributed Tracing & Sampling — Bug Spotting](fundamentals/tracing-sampling-bug-spotting.md) ★ — _(2026-05-03)_

### Sibling docs in this repo

- [Spring Boot Actuator Deep Dive](../java/actuator-deep-dive.md)
- [Reactive Observability (Micrometer + Reactor)](../java/reactive-observability.md)
- [Java Distributed Tracing](../java/observability/distributed-tracing.md)
- [Node.js Production Observability](../typescript/production/observability.md)
- [Node.js Profiling and Debugging](../typescript/production/profiling-and-debugging.md)
- [Kubernetes Monitoring and Logging](../kubernetes/operations/monitoring-and-logging.md)
- [Network Observability (eBPF, RED)](../networking/advanced/network-observability.md)
- [System Design: Tracing, Metrics, Logs](../system-design/performance-observability/tracing-metrics-logs.md)
- [System Design: RED/USE/Golden Signals](../system-design/performance-observability/monitoring-red-use-golden-signals.md)
- [System Design: Log Aggregation and Structured Logging](../system-design/performance-observability/log-aggregation-and-structured-logging.md)
- [System Design: Dashboards, Runbooks, On-Call](../system-design/performance-observability/dashboards-runbooks-on-call.md)
