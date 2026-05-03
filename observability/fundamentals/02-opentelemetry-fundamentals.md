---
title: "OpenTelemetry Fundamentals — API, SDK, Collector, and OTLP"
date: 2026-05-03
updated: 2026-05-03
tags: [observability, opentelemetry, otlp, instrumentation, collector, semantic-conventions]
---

# OpenTelemetry Fundamentals — API, SDK, Collector, and OTLP

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `observability` `opentelemetry` `otlp` `instrumentation` `collector` `semantic-conventions`

---

## Table of Contents

- [Summary](#summary)
- [1. Why OpenTelemetry Exists](#1-why-opentelemetry-exists)
- [2. Architecture: API, SDK, Collector](#2-architecture-api-sdk-collector)
  - [2.1 The API / SDK Split](#21-the-api--sdk-split)
  - [2.2 The Collector](#22-the-collector)
  - [2.3 Reference Topology](#23-reference-topology)
- [3. Signals](#3-signals)
  - [3.1 Traces](#31-traces)
  - [3.2 Metrics](#32-metrics)
  - [3.3 Logs](#33-logs)
  - [3.4 Profiles (Stable / Experimental Status)](#34-profiles-stable--experimental-status)
- [4. Resource and Semantic Conventions](#4-resource-and-semantic-conventions)
  - [4.1 Resource](#41-resource)
  - [4.2 Semantic Conventions](#42-semantic-conventions)
  - [4.3 Stability Levels](#43-stability-levels)
- [5. OTLP — The Wire Protocol](#5-otlp--the-wire-protocol)
  - [5.1 OTLP/gRPC vs OTLP/HTTP](#51-otlpgrpc-vs-otlphttp)
  - [5.2 Endpoints and Encoding](#52-endpoints-and-encoding)
  - [5.3 Why OTLP Replaced Format-Per-Vendor](#53-why-otlp-replaced-format-per-vendor)
- [6. Auto-Instrumentation vs Manual Instrumentation](#6-auto-instrumentation-vs-manual-instrumentation)
- [7. Exporter Patterns](#7-exporter-patterns)
- [8. Code: Node.js / TypeScript](#8-code-nodejs--typescript)
  - [8.1 Auto-Instrumentation](#81-auto-instrumentation)
  - [8.2 Manual SDK Setup](#82-manual-sdk-setup)
  - [8.3 Custom Spans and Metrics](#83-custom-spans-and-metrics)
- [9. Code: Java / Spring Boot](#9-code-java--spring-boot)
  - [9.1 Java Agent (Zero-Code)](#91-java-agent-zero-code)
  - [9.2 Spring Boot + Micrometer Bridge](#92-spring-boot--micrometer-bridge)
  - [9.3 Custom Spans with the OTel API](#93-custom-spans-with-the-otel-api)
- [10. Collector Configuration Walkthrough](#10-collector-configuration-walkthrough)
- [11. Common Pitfalls](#11-common-pitfalls)
- [Related](#related)
- [References](#references)

---

## Summary

OpenTelemetry (OTel) is a CNCF project that standardizes how applications produce telemetry — traces, metrics, and logs — and how that telemetry is shipped to backends. The win is vendor neutrality: instrument once with the OTel API, then point the SDK at any compatible backend (Jaeger, Tempo, Prometheus, Honeycomb, Datadog, New Relic) without changing application code. The pieces you need to internalize are the **API/SDK split**, the **Collector** as a sidecar/gateway, **OTLP** as the wire protocol, **resource attributes** as the service-identity envelope, and **semantic conventions** as the agreed vocabulary for span and metric names. This doc covers all of those, then shows working setups for Node.js and Spring Boot.

---

## 1. Why OpenTelemetry Exists

Before OTel there was a fragmented landscape:

- **OpenTracing** (2016) defined a tracing API but had no metrics story and no reference SDK.
- **OpenCensus** (2018, Google) covered traces and metrics with a reference SDK but had its own wire format.
- Each vendor (Datadog, New Relic, Honeycomb, Jaeger, Zipkin) had its own agent, library, and protocol.

OpenTracing and OpenCensus merged in 2019 to form OpenTelemetry. The project now graduated CNCF status for traces, metrics, and the OTLP protocol, with logs reaching stability in 2023 and profiling at experimental status. The mandate is explicit: **the API is permanent, the SDK is replaceable, the wire format is open, and instrumentation is portable across vendors**.

---

## 2. Architecture: API, SDK, Collector

### 2.1 The API / SDK Split

OpenTelemetry separates the instrumentation API your code calls from the SDK that actually exports data:

```
┌─────────────────────────┐
│   Application code      │  imports otel API
│   tracer.start_span()   │  ── stable, never breaks
│   counter.add(1, attrs) │
└────────────┬────────────┘
             │ uses
             ▼
┌─────────────────────────┐
│       OTel API          │  ABI-stable, no-op when no SDK
│   (interfaces only)     │
└────────────┬────────────┘
             │ delegates to
             ▼
┌─────────────────────────┐
│       OTel SDK          │  implements: sampling, batching, exporters
│   (replaceable impl)    │
└────────────┬────────────┘
             │ exports via
             ▼
┌─────────────────────────┐
│         OTLP            │  → Collector or backend
└─────────────────────────┘
```

Two consequences worth noting:

- **Library authors** can call the OTel API without forcing their users to ship an SDK. If no SDK is registered, calls become no-ops.
- **Application owners** install the SDK once at process startup; library spans automatically light up.

### 2.2 The Collector

The OpenTelemetry Collector is a separate binary that:

- **Receives** telemetry from SDKs (OTLP, Jaeger, Zipkin, Prometheus scrape, Fluent Forward, etc.).
- **Processes** it: batching, attribute filtering, redaction, sampling, transformation.
- **Exports** to one or more backends (Tempo, Mimir, Loki, Honeycomb, Datadog, S3, Kafka, …).

Three deployment modes:

| Mode | Where it runs | Use case |
|------|---------------|----------|
| **Agent** (sidecar / DaemonSet) | One per host or pod | Fast collection, local buffering, host-level enrichment |
| **Gateway** (cluster service) | Centralized fleet | Tail sampling, fan-out to many backends, egress control |
| **Both** | Agent → Gateway → backends | Production default for medium+ scale |

### 2.3 Reference Topology

```
┌───────────────┐    OTLP/gRPC     ┌──────────────┐    OTLP/gRPC    ┌────────┐
│  Service A    ├─────────────────▶│ Agent        ├────────────────▶│ Gateway│
│ (OTel SDK)    │                  │ Collector    │                 │        │
└───────────────┘                  │ (sidecar /   │                 │        │
┌───────────────┐                  │  DaemonSet)  │                 │        │
│  Service B    ├─────────────────▶│              │                 │        │
└───────────────┘                  └──────────────┘                 │        │
                                                                    │        │
                            tail-sampling, redaction, fan-out  ────▶│        │
                                                                    └───┬────┘
                                                                        │
                              ┌─────────────────┬───────────────────────┼───────────────┐
                              ▼                 ▼                       ▼               ▼
                         Tempo / Jaeger     Mimir / Prom            Loki / ELK      vendor SaaS
                         (traces)           (metrics)               (logs)          (Datadog, …)
```

---

## 3. Signals

A "signal" in OTel terminology is a category of telemetry with its own data model, API surface, and SDK pipeline.

### 3.1 Traces

A `Tracer` produces `Span`s. Each span has a name, kind (server, client, internal, producer, consumer), start/end time, attributes, status, and an optional parent reference. Spans share a `trace_id`. Context propagation crosses process boundaries via the W3C `traceparent` header (default in OTel) or B3 / Jaeger formats. See [Distributed Tracing Internals](06-distributed-tracing-internals.md).

### 3.2 Metrics

A `Meter` produces instruments:

| Instrument | OTel name | Behavior |
|------------|-----------|----------|
| Counter | `Counter` | Monotonic, additive |
| UpDownCounter | `UpDownCounter` | Non-monotonic additive |
| Gauge | `Gauge` (sync) / observable | Point-in-time value |
| Histogram | `Histogram` | Bucketed distribution |
| ObservableCounter | callback-based | Pulled at collection time |

Aggregation is configurable via **Views**. The default histogram aggregation uses fixed boundaries; an "exponential histogram" aggregation gives bounded-error quantiles with a small fixed memory footprint.

### 3.3 Logs

The OTel logs data model wraps a log record with `timestamp`, `severity_number`, `severity_text`, `body`, `attributes`, and an inherited trace context. The recommended pattern is **bridge** existing loggers (Logback, SLF4J, pino, Winston) into OTel rather than calling an OTel logger API directly — log libraries are mature and OTel just needs the records on the wire in OTLP format.

### 3.4 Profiles (Stable / Experimental Status)

A profiles signal is in development as of the spec's 2024–2025 work, with pprof-compatible payloads carried over OTLP. Treat as experimental; check `opentelemetry.io/status` before relying on it in production.

---

## 4. Resource and Semantic Conventions

### 4.1 Resource

A **Resource** is a set of attributes that identify the entity producing telemetry — typically the service plus its host/container/cloud context. It is attached **once** to each exported batch, not to every span.

```yaml
service.name: checkout
service.namespace: shop
service.version: 2026.05.03-rc1
service.instance.id: checkout-7b9d6c-x9k2p
deployment.environment.name: production
host.name: ip-10-0-3-42
k8s.namespace.name: shop
k8s.pod.name: checkout-7b9d6c-x9k2p
cloud.provider: aws
cloud.region: us-east-1
```

`service.name` is the only **required** resource attribute; without it, the SDK substitutes `unknown_service` and your data ends up in a junk drawer in every backend.

### 4.2 Semantic Conventions

Semantic conventions standardize attribute keys and values so that dashboards work across services and across vendors. Examples:

| Domain | Key attributes |
|--------|----------------|
| HTTP server | `http.request.method`, `url.path`, `http.response.status_code`, `server.address`, `server.port` |
| Database | `db.system.name`, `db.namespace`, `db.query.text`, `db.operation.name` |
| Messaging | `messaging.system`, `messaging.destination.name`, `messaging.operation.type` |
| RPC | `rpc.system`, `rpc.service`, `rpc.method` |
| Exception | `exception.type`, `exception.message`, `exception.stacktrace` |

Use the published conventions verbatim — bespoke key names defeat the entire vendor-neutral premise.

### 4.3 Stability Levels

Each convention is marked **stable**, **experimental**, or **development**. Stable means breaking changes require a major version bump. HTTP and database conventions are stable; some messaging conventions and all profiling conventions are still experimental as of mid-2025.

---

## 5. OTLP — The Wire Protocol

### 5.1 OTLP/gRPC vs OTLP/HTTP

| Variant | Transport | Encoding | Compression | Default port |
|---------|-----------|----------|-------------|--------------|
| OTLP/gRPC | HTTP/2 + gRPC | Protobuf | gzip | 4317 |
| OTLP/HTTP (binary) | HTTP/1.1 or 2 | Protobuf | gzip | 4318 |
| OTLP/HTTP (JSON) | HTTP/1.1 or 2 | JSON | gzip | 4318 |

gRPC has lower overhead and built-in streaming; HTTP is friendlier to firewalls and easier for browser/mobile SDKs.

### 5.2 Endpoints and Encoding

```
POST /v1/traces       application/x-protobuf  (or application/json)
POST /v1/metrics
POST /v1/logs
```

Standard environment variables (read by every official SDK):

```bash
OTEL_SERVICE_NAME=checkout
OTEL_RESOURCE_ATTRIBUTES="service.namespace=shop,deployment.environment.name=production"
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1
```

### 5.3 Why OTLP Replaced Format-Per-Vendor

Before OTLP, supporting N backends meant writing N exporters in M languages — an N × M matrix. OTLP collapses that to N × 1 (each language ships an OTLP exporter; each backend ships an OTLP receiver). Vendors compete on storage and UX, not on lock-in via wire format.

---

## 6. Auto-Instrumentation vs Manual Instrumentation

| Aspect | Auto | Manual |
|--------|------|--------|
| Setup | Java agent, `@opentelemetry/auto-instrumentations-node`, etc. | `import` and write code |
| Coverage | Common libraries (HTTP, gRPC, DB drivers, Kafka, Redis) | Only what you instrument |
| Granularity | Generic spans, generic attributes | Domain-specific |
| Risk | Conflicts with patches, perf overhead, surprise spans | None beyond what you wrote |
| Use case | Fast time-to-value, polyglot fleet | Critical paths, custom business logic |

**Realistic stance**: start with auto-instrumentation, add manual spans for business-critical operations (e.g., "checkout.computeTax", "feature_flag.evaluation"). Don't ship 100% manual — you will miss spans the autoloader catches for free.

---

## 7. Exporter Patterns

| Pattern | When |
|---------|------|
| **Direct to backend** | Small services, low-volume, single backend |
| **Direct to gateway Collector** | Standard production |
| **Sidecar agent → gateway** | Multi-tenant clusters, per-pod buffering |
| **DaemonSet agent → gateway** | Kubernetes, one collector per node |
| **Batch + retry built into SDK** | Always — never use the SimpleSpanProcessor in prod |

The SDK provides a `BatchSpanProcessor` and `BatchLogRecordProcessor` with a maximum queue size and an export timeout. Drops are reported via internal metrics — instrument **the collector** itself with metrics or you will not know when telemetry is being lost.

---

## 8. Code: Node.js / TypeScript

### 8.1 Auto-Instrumentation

```bash
npm i @opentelemetry/api \
      @opentelemetry/sdk-node \
      @opentelemetry/auto-instrumentations-node \
      @opentelemetry/exporter-trace-otlp-http \
      @opentelemetry/exporter-metrics-otlp-http
```

Run with:

```bash
node --require ./otel.js dist/server.js
```

```typescript
// otel.js — loaded before app code
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';
import { resourceFromAttributes } from '@opentelemetry/resources';
import {
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
} from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: resourceFromAttributes({
    [ATTR_SERVICE_NAME]: 'checkout',
    [ATTR_SERVICE_VERSION]: process.env.GIT_SHA ?? 'dev',
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT
      ? `${process.env.OTEL_EXPORTER_OTLP_ENDPOINT}/v1/traces`
      : 'http://otel-collector:4318/v1/traces',
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter(),
    exportIntervalMillis: 15_000,
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
process.on('SIGTERM', () => sdk.shutdown());
```

### 8.2 Manual SDK Setup

You only need this when auto-instrumentation is too broad (e.g., a CLI that should not patch every HTTP module). In a server, prefer auto.

### 8.3 Custom Spans and Metrics

```typescript
import { trace, metrics, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('checkout');
const meter = metrics.getMeter('checkout');

const taxLatency = meter.createHistogram('checkout.tax.duration', {
  unit: 'ms',
  description: 'Tax computation latency',
});

export async function computeTax(order: Order) {
  return tracer.startActiveSpan('checkout.computeTax', async (span) => {
    span.setAttribute('order.line_count', order.lines.length);
    const start = performance.now();
    try {
      const tax = await taxService.compute(order);
      span.setStatus({ code: SpanStatusCode.OK });
      return tax;
    } catch (err) {
      span.recordException(err as Error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: (err as Error).message });
      throw err;
    } finally {
      taxLatency.record(performance.now() - start, { provider: order.taxProvider });
      span.end();
    }
  });
}
```

---

## 9. Code: Java / Spring Boot

### 9.1 Java Agent (Zero-Code)

```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=checkout \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4318 \
     -Dotel.exporter.otlp.protocol=http/protobuf \
     -jar app.jar
```

This single agent flag instruments servlets, JDBC, HTTP clients, Kafka, Redis, gRPC, RabbitMQ, and the major Spring stacks — no code changes.

### 9.2 Spring Boot + Micrometer Bridge

Spring Boot 3.x ships with **Micrometer Tracing** that can use OTel as the underlying tracer while keeping the Micrometer API for metrics. This is the path most Spring teams take.

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-otlp</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 0.1
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
    metrics:
      export:
        url: http://otel-collector:4318/v1/metrics
        step: 15s
```

See [Spring Boot Actuator Deep Dive](../../java/actuator-deep-dive.md) for Actuator configuration that complements this.

### 9.3 Custom Spans with the OTel API

```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;

@Service
public class TaxService {

    private final Tracer tracer;

    public TaxService(OpenTelemetry openTelemetry) {
        this.tracer = openTelemetry.getTracer("checkout");
    }

    public BigDecimal computeTax(Order order) {
        Span span = tracer.spanBuilder("checkout.computeTax")
            .setAttribute("order.line_count", order.getLines().size())
            .startSpan();
        try (var scope = span.makeCurrent()) {
            return externalTaxApi.compute(order);
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

---

## 10. Collector Configuration Walkthrough

Minimal production-ish gateway config:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 8192

  memory_limiter:
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 25

  resource:
    attributes:
      - key: deployment.environment.name
        value: production
        action: upsert

  attributes/redact:
    actions:
      - key: http.request.header.authorization
        action: delete
      - key: user.email
        action: hash

  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: slow
        type: latency
        latency: { threshold_ms: 1000 }
      - name: random-1pct
        type: probabilistic
        probabilistic: { sampling_percentage: 1 }

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls: { insecure: true }
  prometheusremotewrite:
    endpoint: http://mimir:9009/api/v1/push
  otlphttp/loki:
    endpoint: http://loki:3100/otlp

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resource, attributes/redact, tail_sampling, batch]
      exporters: [otlp/tempo]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [prometheusremotewrite]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, resource, attributes/redact, batch]
      exporters: [otlphttp/loki]
```

Key processors to know: `batch`, `memory_limiter` (always include), `tail_sampling`, `attributes` / `transform` (for redaction), `k8sattributes` (auto-enriches with pod/namespace/cluster).

---

## 11. Common Pitfalls

- **Missing `service.name`.** Telemetry shows up as `unknown_service:node` and gets ignored.
- **Auto + manual instrumenting the same library.** Double spans. Use one or the other for a given concern.
- **Exporting straight to a backend with no Collector.** Fine for dev; in prod you lose buffering, redaction, and the ability to swap backends.
- **Synchronous span exporters.** `SimpleSpanProcessor` blocks — never use in hot paths.
- **Custom attribute names that diverge from semantic conventions.** Vendor dashboards stop working.
- **Forgetting context propagation in async work.** `setTimeout`, queues, thread pools, and `worker_threads` need explicit context attach. `@opentelemetry/context-async-hooks` handles Node async; use `@WithSpan` or manual `Context.current().wrap()` in Java.
- **PII in span attributes.** `user.email`, request bodies, JWT contents — strip in the Collector.
- **Sampling configured in two places.** SDK head-sampling and Collector tail-sampling can conflict; prefer SDK = always-on, Collector = the sampler.

---

## Related

- [Three Pillars: Metrics, Logs, Traces](01-three-pillars-metrics-logs-traces.md)
- [Distributed Tracing Internals](06-distributed-tracing-internals.md)
- [Structured Logging](05-structured-logging.md)
- [Prometheus and PromQL](07-prometheus-and-promql.md)
- [Spring Boot Actuator Deep Dive](../../java/actuator-deep-dive.md)
- [Java Distributed Tracing](../../java/observability/distributed-tracing.md)
- [Node.js Production Observability](../../typescript/production/observability.md)

---

## References

- **OpenTelemetry — main spec and docs.** https://opentelemetry.io/docs/
- **OTLP Specification (v1.5+).** https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/protocol/otlp.md
- **OpenTelemetry Semantic Conventions.** https://opentelemetry.io/docs/specs/semconv/
- **OpenTelemetry Collector documentation.** https://opentelemetry.io/docs/collector/
- **OpenTelemetry Java agent.** https://github.com/open-telemetry/opentelemetry-java-instrumentation
- **OpenTelemetry JavaScript / Node.js SDK.** https://opentelemetry.io/docs/languages/js/
- **W3C Trace Context (Recommendation).** https://www.w3.org/TR/trace-context/
- **Spring Boot — Observability documentation.** https://docs.spring.io/spring-boot/reference/actuator/observability.html
- **Micrometer Tracing.** https://docs.micrometer.io/tracing/reference/
- **CNCF — OpenTelemetry project page.** https://www.cncf.io/projects/opentelemetry/
