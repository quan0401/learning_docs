---
title: "Structured Logging — JSON, Levels, Correlation, and Cost Control"
date: 2026-05-03
updated: 2026-05-03
tags: [observability, logging, structured-logs, ecs, correlation, pii, sampling]
---

# Structured Logging — JSON, Levels, Correlation, and Cost Control

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `observability` `logging` `structured-logs` `ecs` `correlation` `pii` `sampling`

---

## Table of Contents

- [Summary](#summary)
- [1. Why Structured Logging](#1-why-structured-logging)
- [2. JSON vs Logfmt vs Free Text](#2-json-vs-logfmt-vs-free-text)
- [3. The Semantic Field Set](#3-the-semantic-field-set)
  - [3.1 Required-by-Default Fields](#31-required-by-default-fields)
  - [3.2 Elastic Common Schema (ECS)](#32-elastic-common-schema-ecs)
  - [3.3 OpenTelemetry Logs Data Model](#33-opentelemetry-logs-data-model)
- [4. Log Levels — Correctly Used](#4-log-levels--correctly-used)
- [5. Correlation IDs and Trace Context](#5-correlation-ids-and-trace-context)
  - [5.1 trace\_id and span\_id](#51-trace_id-and-span_id)
  - [5.2 Request IDs at the Edge](#52-request-ids-at-the-edge)
  - [5.3 Async Context Propagation](#53-async-context-propagation)
- [6. Sampling and Cost Control](#6-sampling-and-cost-control)
- [7. PII and Secret Scrubbing](#7-pii-and-secret-scrubbing)
- [8. Node.js: pino and winston](#8-nodejs-pino-and-winston)
- [9. Java: Logback / SLF4J for Spring Boot](#9-java-logback--slf4j-for-spring-boot)
- [10. Common Anti-Patterns](#10-common-anti-patterns)
- [Related](#related)
- [References](#references)

---

## Summary

A log line is only as valuable as the queries you can run against it later. Free-text logs require regex archaeology; structured logs let you filter, aggregate, and join across services with the same ergonomics as a SQL or KQL query. The structured-logging discipline has four parts: a consistent **schema** of well-known fields, **correlation** via trace context, **levels** used the way they were designed, and **cost control** through sampling and PII scrubbing. This doc defines each, points to the two community schemas worth knowing (Elastic Common Schema and the OpenTelemetry Logs Data Model), and shows pino/winston and Logback/SLF4J configurations that hold up in production.

---

## 1. Why Structured Logging

Compare:

```
2026-05-03 14:22:01 WARN payment retry attempt 2 for user u_8821 via stripe took 184ms
```

vs:

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

Same content, but the structured form lets you:

- `level=warn AND service=checkout AND attempt>=2` — find every retry across the fleet.
- `provider=stripe AND elapsed_ms>500` — carve out slow Stripe calls.
- `trace_id=4bf92f...` — pull every log line for a specific request, across services.
- aggregate: `count() by provider` — see which payment provider misbehaves most.

Structured logs unlock the same investigative power as wide events ([see Three Pillars §6.2](01-three-pillars-metrics-logs-traces.md#62-wide-structured-events)) without changing storage backends.

---

## 2. JSON vs Logfmt vs Free Text

| Format | Example | Pros | Cons |
|--------|---------|------|------|
| Free text | `payment retry attempt 2 for user u_8821` | Human-readable | Non-queryable without regex; brittle to format changes |
| Logfmt | `level=warn user_id=u_8821 attempt=2 msg="payment retry"` | Compact, still grep-friendly | Limited to flat key=value; no nested types; whitespace ambiguities |
| **JSON** | `{"level":"warn","user_id":"u_8821",...}` | Universally parseable, types preserved, nested objects | More bytes; harder to eyeball without a CLI prettifier |

JSON is the default in production. Use a CLI like `pino-pretty`, `jq`, `lnav`, or `humanlog` for local development — keep the wire format machine-readable.

For very high-volume services, consider **Logfmt over Loki** or **Protobuf over OTLP/logs** to cut bytes; for everything else, JSON over OTLP/HTTP is fine.

---

## 3. The Semantic Field Set

### 3.1 Required-by-Default Fields

Every log line should carry:

| Field | Purpose |
|-------|---------|
| `timestamp` (ISO-8601 with timezone, or epoch nanos) | When the event happened |
| `level` (`debug`/`info`/`warn`/`error`/`fatal`) | Severity |
| `service.name` | Which service emitted it |
| `service.version` | Build / commit / release tag |
| `deployment.environment.name` | `production` / `staging` / `dev` |
| `host.name` / `k8s.pod.name` | Where it ran |
| `trace_id` / `span_id` | Trace correlation |
| `msg` (or `message`) | Short, fixed-string description |

Variable context goes in **separate fields**, not interpolated into `msg`. `msg` should be a low-cardinality string you can group by.

```json
// Bad — high-cardinality message
{ "msg": "user u_8821 failed to charge $25.00 on stripe" }

// Good — low-cardinality message + structured fields
{ "msg": "charge failed", "user_id": "u_8821", "amount_usd": 25.00, "provider": "stripe" }
```

### 3.2 Elastic Common Schema (ECS)

ECS is a schema published by Elastic for log/event data. It nests fields under namespaces:

```json
{
  "@timestamp": "2026-05-03T14:22:01.482Z",
  "log": { "level": "warn" },
  "service": { "name": "checkout", "version": "2026.05.03-rc1" },
  "host": { "name": "checkout-7b9d6c-x9k2p" },
  "trace": { "id": "4bf92f3577b34da6a3ce929d0e0e4736" },
  "span": { "id": "00f067aa0ba902b7" },
  "user": { "id": "u_8821" },
  "http": { "request": { "method": "POST" }, "response": { "status_code": 502 } },
  "event": { "action": "payment_retry", "outcome": "failure" },
  "message": "payment retry"
}
```

ECS is mature, widely adopted in the Elastic ecosystem, and aligned where possible with OpenTelemetry semantic conventions. If you ship logs to Elastic / OpenSearch, follow ECS.

### 3.3 OpenTelemetry Logs Data Model

OTel defines a vendor-neutral log record:

```
timestamp        : nano-precision
observed_timestamp
severity_number  : 1..24
severity_text    : DEBUG / INFO / WARN / ERROR / FATAL
body             : the message (string or structured)
trace_id, span_id, trace_flags
attributes       : key/value (event-specific)
resource         : key/value (service, host, k8s, …)
```

OTel is converging on ECS-compatible attribute names where possible. For greenfield work, emit OTLP logs from your application (via the `opentelemetry-appender-*` bridges or directly), and let the Collector translate to your backend's format.

---

## 4. Log Levels — Correctly Used

| Level | When |
|-------|------|
| **DEBUG** | Verbose, off in production by default. Step-by-step decisions inside a hot function. Behind a flag if at all. |
| **INFO** | Routine business events worth a forensic record: "order created", "user signed up", "job started". |
| **WARN** | Something unusual happened that the system handled. Retries, degraded fallbacks, deprecated-API hits. **No human action required right now.** |
| **ERROR** | The system failed to do its job for a specific request/event. Likely needs investigation in aggregate; does not necessarily page. |
| **FATAL** / `panic` | Process cannot continue. Almost never appropriate in a long-running service — prefer ERROR + crash + restart. |

Two rules:

- **WARN ≠ "smaller error"**. WARN means handled. If a call failed and the request errored to the user, it's an ERROR.
- **Don't log at ERROR for expected business outcomes.** A user typing the wrong password is INFO at most. ERRORs trigger alarms and on-call paging — keep them meaningful.

Production default: ship INFO and above; sample INFO; collect WARN/ERROR fully.

---

## 5. Correlation IDs and Trace Context

### 5.1 trace\_id and span\_id

The most important fields after `level` and `service`. They are the join key across pillars (see [Three Pillars §7](01-three-pillars-metrics-logs-traces.md#7-correlation-across-signals)).

When using OpenTelemetry SDKs, the recommended pattern is to use a **logging bridge** that automatically reads the active span from the Context and stamps `trace_id`/`span_id` onto every log record:

- Node: `@opentelemetry/instrumentation-pino`, `@opentelemetry/instrumentation-winston`
- Java: `OpenTelemetryAppender` for Logback / Log4j2

### 5.2 Request IDs at the Edge

Even with traces, a stable per-request ID at your edge (gateway, load balancer) is useful:

- Set `X-Request-ID` at the gateway (Nginx `$request_id`, Envoy `x-request-id`, AWS ALB request ID).
- Propagate it through service-to-service calls (alongside `traceparent`).
- Log it as a top-level field. Customers can quote it in support tickets.

### 5.3 Async Context Propagation

Background work (queues, timers, thread pools) loses the ambient context unless you propagate it manually:

- **Node**: rely on `AsyncLocalStorage` (Node ≥ 16.4) or the OTel ContextManager (`@opentelemetry/context-async-hooks`). Check that your job framework supports context propagation; if not, capture `Context.current()` when enqueuing and `context.with()` it when handling.
- **Java**: use `Context.current().wrap(runnable)`, the OTel `ContextPropagators`, or Reactor's `Context` which Micrometer Tracing integrates with.

A log line emitted from a background worker without a `trace_id` is forensically useless — fix the propagation, do not paper over it with random correlation IDs.

---

## 6. Sampling and Cost Control

Logging cost grows linearly with traffic. Levers:

| Lever | Effect | Trade-off |
|-------|--------|-----------|
| Drop DEBUG in prod | Big | Lose detail when issues happen |
| Sample INFO at 10% | Linear cut | Statistics + biased toward common events |
| Aggregate repetitive logs | "27 retries to provider=stripe" instead of 27 lines | Loses individual instances |
| Send to cheap cold storage | 10×–100× cheaper | Slower queries, restricted retention |
| Tail-based: keep all logs for traces with errors or high latency | Best signal/cost | Requires Collector tail sampling and trace correlation |
| Drop high-cardinality fields | Cuts indexing cost | Can break queries |

A common high-volume setup: ship **all WARN/ERROR**, plus **logs whose trace ended in an error or latency outlier** (tail-correlated), plus **1% of normal INFO** as a baseline. Tail correlation requires the OTel Collector or a similar pipeline that links logs to trace decisions.

---

## 7. PII and Secret Scrubbing

A log pipeline that ingests user data is in scope for GDPR, CCPA, HIPAA (where applicable), PCI-DSS, and your own internal data-handling policy. Some non-negotiables:

- **Do not log secrets**: API keys, tokens, JWT bodies, password fields, full credit-card numbers, full SSNs. Reject them in tests with a static-analysis or unit-test rule.
- **Scrub PII fields**: full email, full name, IP (in some jurisdictions), home address, government IDs.
- **Minimum-viable identifier**: log a stable, internal `user_id`, not `email`. If you need email for support workflows, log a one-way hash and store the lookup in a separate, access-controlled system.

Two-layer defense:

1. **In the application**: typed loggers with allow-listed fields. Whatever your library, configure a redactor for known-sensitive keys.
2. **In the Collector**: an `attributes` or `transform` processor that hashes/strips known PII keys regardless of what the application emitted. This catches escapes.

Pino has a built-in `redact` config. Logback has masking layouts (e.g., `LogstashMaskingPatternLayout` from logstash-logback-encoder).

```json
// pino redact example output (key replaced with "[REDACTED]")
{ "user": { "email": "[REDACTED]", "id": "u_8821" } }
```

---

## 8. Node.js: pino and winston

**pino** is the high-performance default for new Node services — JSON-by-design, low overhead, async transports.

```typescript
// logger.ts
import pino from 'pino';
import { trace, context } from '@opentelemetry/api';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  base: {
    service: process.env.OTEL_SERVICE_NAME ?? 'checkout',
    env: process.env.DEPLOY_ENV ?? 'dev',
    version: process.env.GIT_SHA ?? 'dev',
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  redact: {
    paths: [
      'req.headers.authorization',
      'req.headers.cookie',
      'user.email',
      '*.password',
      '*.token',
    ],
    censor: '[REDACTED]',
  },
  // Stamp trace context on every log
  mixin() {
    const span = trace.getSpan(context.active());
    if (!span) return {};
    const ctx = span.spanContext();
    return { trace_id: ctx.traceId, span_id: ctx.spanId };
  },
});
```

For request logging in Express/Fastify, use `pino-http` which logs one structured line per request including method, route, status, duration, and (with the OTel instrumentation enabled) trace context.

**winston** is the older, more featureful option. Configure a JSON formatter and add a custom formatter that reads OTel trace context — or use `@opentelemetry/instrumentation-winston` which does it for you.

---

## 9. Java: Logback / SLF4J for Spring Boot

Spring Boot uses Logback by default through the SLF4J facade. The standard production setup:

```xml
<!-- logback-spring.xml -->
<configuration>
  <appender name="STDOUT_JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <customFields>{"service.name":"checkout","deployment.environment.name":"production"}</customFields>
      <fieldNames>
        <timestamp>@timestamp</timestamp>
        <message>message</message>
        <thread>thread</thread>
        <logger>logger</logger>
        <level>level</level>
      </fieldNames>
    </encoder>
  </appender>

  <!-- OTel appender bridges Logback → OTLP logs -->
  <appender name="OpenTelemetry"
            class="io.opentelemetry.instrumentation.logback.appender.v1_0.OpenTelemetryAppender">
  </appender>

  <root level="INFO">
    <appender-ref ref="STDOUT_JSON"/>
    <appender-ref ref="OpenTelemetry"/>
  </root>
</configuration>
```

```java
// In code:
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import net.logstash.logback.argument.StructuredArguments;

public class CheckoutService {
    private static final Logger log = LoggerFactory.getLogger(CheckoutService.class);

    public void chargePayment(Order order) {
        log.warn("payment retry",
            StructuredArguments.kv("user_id", order.getUserId()),
            StructuredArguments.kv("attempt", 2),
            StructuredArguments.kv("provider", "stripe"),
            StructuredArguments.kv("elapsed_ms", 184));
    }
}
```

The `logstash-logback-encoder` adds `kv()`, `array()`, and `entries()` for structured arguments; the OTel appender handles trace context automatically when the OpenTelemetry Java agent is loaded. See [Spring Boot Actuator Deep Dive](../../java/actuator-deep-dive.md) for end-to-end Actuator + Micrometer + Logback integration.

For Reactor / WebFlux services where the request thread isn't fixed, ensure you've enabled `Hooks.enableAutomaticContextPropagation()` (Reactor 3.5+) or use Micrometer Context Propagation so MDC and trace context survive scheduler hops.

---

## 10. Common Anti-Patterns

- **String-interpolated messages.** `log.info("user " + id + " did " + action)` defeats the entire schema. Use structured fields.
- **High-cardinality `msg`.** `msg = f"failed to charge {user_id}"` blows up text indexes. Keep `msg` static; put the variable in a field.
- **Logging exceptions and re-throwing them.** Each layer logs the same error → 4× the cost, 4× the noise. Log at the boundary that handles the error or attaches business context, not at every stack frame.
- **Logs as the primary metric source.** Counting log lines to compute a rate is fragile and expensive. Emit a metric.
- **Logs without trace context.** Forensically dead during incidents.
- **`console.log` in production.** Bypasses formatting, redaction, level filtering, and trace stamping. Lint it out.
- **Logging request/response bodies by default.** PII risk, cost risk. Behind a flag, sample, redact known fields.
- **One giant log file per host.** Use stdout in containers; let the runtime + log shipper handle rotation.

---

## Related

- [Three Pillars: Metrics, Logs, Traces](01-three-pillars-metrics-logs-traces.md)
- [OpenTelemetry Fundamentals](02-opentelemetry-fundamentals.md)
- [Distributed Tracing Internals](06-distributed-tracing-internals.md)
- [Spring Boot Actuator Deep Dive](../../java/actuator-deep-dive.md)
- [Node.js Production Observability](../../typescript/production/observability.md)
- [System Design: Log Aggregation and Structured Logging](../../system-design/performance-observability/log-aggregation-and-structured-logging.md)
- [Kubernetes Monitoring and Logging](../../kubernetes/operations/monitoring-and-logging.md)

---

## References

- **Elastic Common Schema (ECS) — reference.** https://www.elastic.co/guide/en/ecs/current/index.html
- **OpenTelemetry — Logs data model and specification.** https://opentelemetry.io/docs/specs/otel/logs/data-model/
- **OpenTelemetry — Logging in Java (with Logback bridge).** https://opentelemetry.io/docs/languages/java/instrumentation/#logs
- **pino — official documentation.** https://getpino.io/
- **winston — official documentation.** https://github.com/winstonjs/winston
- **logstash-logback-encoder.** https://github.com/logfellow/logstash-logback-encoder
- **Charity Majors — "Logs vs Structured Events"** (Honeycomb blog). https://www.honeycomb.io/blog/logs-vs-structured-events
- **OWASP — Logging Cheat Sheet.** https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- **W3C Trace Context.** https://www.w3.org/TR/trace-context/
- **Brendan Burns et al. — *Designing Distributed Systems*** chapter on logging and observability.
