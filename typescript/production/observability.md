# Observability for Node Services

> **Problem**: A user reports checkout is slow. The request touches API Gateway, Order Service, Payment Service, and Notification Service. Latency is 4 seconds, but each service's own metrics show p99 under 200ms. Where is the time going? Without distributed tracing and correlated logs, you are guessing.

---

## The Three Pillars

Observability rests on three signal types. Each answers a different question, and the real power comes from connecting them.

| Pillar | Question It Answers | Format | Example |
|--------|-------------------|--------|---------|
| **Logs** | What happened? | Structured JSON events | `{"level":"error","msg":"payment declined","orderId":"abc-123","traceId":"4bf92f..."}` |
| **Metrics** | How is the system behaving in aggregate? | Numeric time series | `http_request_duration_seconds{route="/orders",method="POST"} 0.245` |
| **Traces** | What path did this specific request take? | Span trees with timing | API Gateway (12ms) -> Order Service (45ms) -> Payment Service (3800ms) -> Notification (15ms) |

### How They Connect

A single request generates all three:
- **Trace ID** is generated at the edge and propagated through every service.
- Every **log line** for that request includes the trace ID.
- **Metrics** are tagged with route, status code, and service name.

When the p99 latency metric spikes, you filter traces by that time window. You find a slow trace, grab its trace ID, and search logs across all services for that ID. You now have the full story: aggregate behavior (metrics), request path (traces), and detailed context (logs).

---

## Structured Logging with Pino

### Why Pino

Pino is the fastest production logger for Node.js. It outputs newline-delimited JSON by default, which is exactly what log aggregators (Elasticsearch, Loki, CloudWatch) want. No parsing regex needed.

**Pino vs console.log**: `console.log` is synchronous and blocks the event loop. Pino writes to a worker thread by default.

### Setup

```typescript
import pino from "pino";

const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  // Redact sensitive fields globally
  redact: ["req.headers.authorization", "req.headers.cookie"],
  // Add service metadata to every log line
  base: {
    service: "order-service",
    version: process.env.APP_VERSION ?? "unknown",
  },
  // Pretty-print in dev, raw JSON in prod
  transport:
    process.env.NODE_ENV === "development"
      ? { target: "pino-pretty", options: { colorize: true } }
      : undefined,
});

export { logger };
```

### Child Loggers with Context

Child loggers inherit the parent's config and add context fields. Every log line from a child includes the parent's fields plus the child's.

```typescript
// Per-request logger with correlation ID
function createRequestLogger(traceId: string, userId?: string) {
  return logger.child({
    traceId,
    userId,
  });
}

// Usage in a handler
app.post("/orders", (req, res) => {
  const reqLogger = createRequestLogger(
    req.headers["x-trace-id"] as string,
    req.user?.id
  );

  reqLogger.info({ body: req.body }, "Creating order");
  // => {"level":30,"service":"order-service","traceId":"abc-123","userId":"u-42","msg":"Creating order","body":{...}}

  try {
    const order = createOrder(req.body);
    reqLogger.info({ orderId: order.id }, "Order created");
    res.status(201).json(order);
  } catch (err) {
    reqLogger.error({ err }, "Order creation failed");
    res.status(500).json({ error: "Internal server error" });
  }
});
```

### Request Logging Middleware

Pattern: generate or extract `traceId` from `x-trace-id` header, record `performance.now()` at start, listen to `res.on("finish")`, then log `{ traceId, method, path, statusCode, durationMs }` at the appropriate level (error for 5xx, warn for 4xx, info otherwise). Set the trace ID on the response header for downstream propagation.

### Java Parallel: SLF4J + Logback

| Concern | Java/Spring Boot | Node.js |
|---------|-----------------|---------|
| Logger | SLF4J facade + Logback impl | pino |
| Structured output | Logback JSON encoder | JSON by default |
| Context propagation | MDC (ThreadLocal) | AsyncLocalStorage (see below) |
| Per-request fields | `MDC.put("traceId", id)` | `logger.child({ traceId })` |
| Log levels | TRACE, DEBUG, INFO, WARN, ERROR | trace, debug, info, warn, error, fatal |

---

## OpenTelemetry Integration

OpenTelemetry (OTel) is the vendor-neutral standard for traces, metrics, and logs. The Node.js SDK auto-instruments common libraries so you get spans without changing application code.

### Full Setup

Install dependencies:

```bash
npm install @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/exporter-metrics-otlp-http \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions
```

Create `tracing.ts` -- this file **must** be loaded before any other import:

```typescript
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { OTLPMetricExporter } from "@opentelemetry/exporter-metrics-otlp-http";
import { PeriodicExportingMetricReader } from "@opentelemetry/sdk-metrics";
import { Resource } from "@opentelemetry/resources";
import {
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
} from "@opentelemetry/semantic-conventions";

const resource = new Resource({
  [ATTR_SERVICE_NAME]: process.env.OTEL_SERVICE_NAME ?? "order-service",
  [ATTR_SERVICE_VERSION]: process.env.APP_VERSION ?? "0.0.0",
  "deployment.environment": process.env.NODE_ENV ?? "development",
});

const sdk = new NodeSDK({
  resource,
  traceExporter: new OTLPTraceExporter({
    url:
      process.env.OTEL_EXPORTER_OTLP_ENDPOINT ??
      "http://localhost:4318/v1/traces",
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url:
        process.env.OTEL_EXPORTER_OTLP_ENDPOINT ??
        "http://localhost:4318/v1/metrics",
    }),
    exportIntervalMillis: 15_000,
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      // Disable noisy fs instrumentation
      "@opentelemetry/instrumentation-fs": { enabled: false },
      // Configure specific instrumentations
      "@opentelemetry/instrumentation-http": {
        ignoreIncomingPaths: ["/healthz", "/ready"],
      },
    }),
  ],
});

sdk.start();

process.on("SIGTERM", async () => {
  await sdk.shutdown();
});
```

Load it first in your entrypoint:

```typescript
// server.ts
import "./tracing"; // MUST be first import
import { app } from "./app";

app.listen(3000);
```

### What Auto-Instrumentation Captures

The `@opentelemetry/auto-instrumentations-node` package patches libraries at import time. You get spans automatically for: `http`/`https` (inbound and outbound), `express` (route-level), `pg`/`prisma` (queries), `redis`/`ioredis` (cache ops), `grpc`, and `aws-sdk`.

### Manual Spans for Business Logic

Auto-instrumentation covers I/O. For business logic that matters, create manual spans:

```typescript
import { trace, SpanStatusCode } from "@opentelemetry/api";

const tracer = trace.getTracer("order-service");

async function processOrder(orderData: OrderInput): Promise<Order> {
  return tracer.startActiveSpan("processOrder", async (span) => {
    try {
      span.setAttribute("order.item_count", orderData.items.length);
      span.setAttribute("order.total", orderData.total);

      const validated = await tracer.startActiveSpan(
        "validateInventory",
        async (childSpan) => {
          const result = await checkInventory(orderData.items);
          childSpan.setAttribute(
            "inventory.all_available",
            result.allAvailable
          );
          childSpan.end();
          return result;
        }
      );

      if (!validated.allAvailable) {
        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: "Insufficient inventory",
        });
        throw new InsufficientInventoryError(validated.unavailable);
      }

      const order = await saveOrder(orderData);
      span.setAttribute("order.id", order.id);
      return order;
    } catch (err) {
      span.recordException(err as Error);
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw err;
    } finally {
      span.end();
    }
  });
}
```

---

## Distributed Tracing

### Trace Context Propagation

The W3C `traceparent` header carries the trace context between services. Format:

```
traceparent: 00-<trace-id>-<parent-span-id>-<flags>
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

OTel auto-instrumentation handles propagation automatically. When Service A calls Service B via HTTP, the SDK:
1. Reads the current span context.
2. Injects `traceparent` into the outgoing request headers.
3. Service B's SDK extracts it and creates a child span.

### Request Flow Example

```
User -> API Gateway -> Order Service -> Payment Service -> Notification Service

Trace: 4bf92f3577b34da6a3ce929d0e0e4736
|
|- [API Gateway]  POST /checkout         12ms
   |
   |- [Order Service]  processOrder      45ms
      |
      |- [Order Service]  validateInventory   8ms
      |
      |- [Payment Service]  POST /charge     3800ms  <-- HERE IS YOUR PROBLEM
      |  |
      |  |- [Payment Service]  stripeApi     3750ms  <-- Stripe timeout
      |
      |- [Notification Service]  POST /email  15ms
```

In Jaeger or Grafana Tempo, this renders as a waterfall timeline. You immediately see that Payment Service is the bottleneck, and specifically that the Stripe API call is slow.

### Cross-Service Setup

Every service runs the same OTel SDK setup from above. The only difference is `OTEL_SERVICE_NAME`. Set `OTEL_EXPORTER_OTLP_ENDPOINT` to point at your collector (Jaeger, Tempo, or an OTel Collector). The collector receives spans from all services and correlates them by trace ID. For local dev, `jaegertracing/all-in-one` with `COLLECTOR_OTLP_ENABLED=true` accepts OTLP on port 4318 and serves the UI on 16686.

---

## Metrics

### The RED Method

For request-driven services, instrument three things:

| Metric | What | Type | OTel Instrument |
|--------|------|------|----------------|
| **R**ate | Requests per second | Counter | `meter.createCounter()` |
| **E**rrors | Failed requests per second | Counter | `meter.createCounter()` |
| **D**uration | Latency distribution | Histogram | `meter.createHistogram()` |

### Custom Metrics with OTel API

```typescript
import { metrics } from "@opentelemetry/api";

const meter = metrics.getMeter("order-service");

// Request counter
const requestCounter = meter.createCounter("http_requests_total", {
  description: "Total HTTP requests",
});

// Error counter
const errorCounter = meter.createCounter("http_errors_total", {
  description: "Total HTTP errors (5xx)",
});

// Duration histogram
const durationHistogram = meter.createHistogram(
  "http_request_duration_seconds",
  {
    description: "HTTP request duration in seconds",
    unit: "s",
  }
);

// Event loop lag gauge
const eventLoopLag = meter.createObservableGauge(
  "nodejs_eventloop_lag_seconds",
  {
    description: "Event loop lag in seconds",
  }
);

// Observe event loop lag periodically
eventLoopLag.addCallback((result) => {
  const start = performance.now();
  setImmediate(() => {
    const lag = (performance.now() - start) / 1000;
    result.observe(lag);
  });
});
```

### Metrics Middleware

Same pattern as the request logger: measure duration in `res.on("finish")`, build labels `{ method, route, status_code }`, then `requestCounter.add(1, labels)`, `durationHistogram.record(duration, labels)`, and `errorCounter.add(1, labels)` for 5xx responses.

### Prometheus Scrape Endpoint

If your stack uses Prometheus pull-based collection instead of OTel push:

```bash
npm install prom-client
```

```typescript
import promClient from "prom-client";

// Collect default Node.js metrics (heap, event loop, GC)
promClient.collectDefaultMetrics();

app.get("/metrics", async (_req, res) => {
  res.set("Content-Type", promClient.register.contentType);
  res.end(await promClient.register.metrics());
});
```

```yaml
# Kubernetes annotation for Prometheus auto-discovery
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "3000"
    prometheus.io/path: "/metrics"
```

### Key Grafana Dashboard Panels

| Panel | Query (PromQL) | Purpose |
|-------|---------------|---------|
| Request rate | `rate(http_requests_total[5m])` | Traffic volume |
| Error rate | `rate(http_errors_total[5m]) / rate(http_requests_total[5m])` | Error percentage |
| p50/p95/p99 latency | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` | Latency distribution |
| Event loop lag | `nodejs_eventloop_lag_seconds` | Node.js health |
| Heap used | `nodejs_heap_size_used_bytes / nodejs_heap_size_total_bytes` | Memory pressure |

---

## Correlation IDs and AsyncLocalStorage

### The Problem

In Java, MDC (Mapped Diagnostic Context) stores per-request context in a `ThreadLocal`. Each thread handles one request, so context follows the request automatically.

Node.js is single-threaded with async operations. There is no thread-per-request. You need a way to carry context through `async`/`await` chains without passing it as a function argument through every layer.

### AsyncLocalStorage

`AsyncLocalStorage` is Node's answer to Java's `ThreadLocal`. It tracks context across async boundaries automatically.

```typescript
import { AsyncLocalStorage } from "node:async_hooks";

interface RequestContext {
  traceId: string;
  userId?: string;
  startTime: number;
}

const requestContext = new AsyncLocalStorage<RequestContext>();

export { requestContext };
```

### Middleware to Set Context

```typescript
import { randomUUID } from "node:crypto";

function contextMiddleware(req: Request, _res: Response, next: NextFunction) {
  const context: RequestContext = {
    traceId: (req.headers["x-trace-id"] as string) ?? randomUUID(),
    userId: req.user?.id,
    startTime: performance.now(),
  };

  // Everything inside this callback (including async work) has access to context
  requestContext.run(context, () => {
    next();
  });
}

// Must be registered before other middleware
app.use(contextMiddleware);
```

### Accessing Context Anywhere

```typescript
// In any file, at any depth in the call stack
import { requestContext } from "./context";
import { logger } from "./logger";

function getContextualLogger() {
  const ctx = requestContext.getStore();
  if (!ctx) return logger;
  return logger.child({ traceId: ctx.traceId, userId: ctx.userId });
}

// Deep in your business logic -- no need to pass traceId as argument
async function calculateShipping(items: CartItem[]): Promise<number> {
  const log = getContextualLogger();
  log.info({ itemCount: items.length }, "Calculating shipping");

  const weight = items.reduce((sum, item) => sum + item.weight, 0);
  const rate = await fetchShippingRate(weight);

  log.info({ weight, rate }, "Shipping calculated");
  return rate;
}
```

### Java Parallel: MDC vs AsyncLocalStorage

| Concern | Java (MDC) | Node.js (AsyncLocalStorage) |
|---------|-----------|---------------------------|
| Storage mechanism | `ThreadLocal` per thread | Async context tracking per async chain |
| Set context | `MDC.put("traceId", id)` | `asyncLocalStorage.run(ctx, fn)` |
| Get context | `MDC.get("traceId")` | `asyncLocalStorage.getStore()` |
| Cleanup | `MDC.clear()` in filter | Automatic when `run()` callback completes |
| Works with | Thread-per-request model | async/await, callbacks, promises |
| Framework support | SLF4J reads MDC automatically | Manual integration (pino, OTel) |

---

## Alerting Strategy

### SLIs and SLOs

Define what "healthy" means before you need to debug an incident.

| SLI (Indicator) | SLO (Objective) | Measurement |
|-----------------|-----------------|-------------|
| Availability | 99.9% of requests return non-5xx | `1 - (5xx_count / total_count)` over 30 days |
| Latency | p99 < 500ms | `histogram_quantile(0.99, ...)` |
| Error rate | < 0.1% of requests are errors | `error_count / total_count` over rolling window |

### Alert on Burn Rate, Not Threshold

Threshold alerts fire on momentary spikes and cause alert fatigue. Burn rate alerts fire when you are consuming your error budget faster than expected.

**Error budget**: If your SLO is 99.9% availability over 30 days, you can tolerate 43.2 minutes of downtime. That is your error budget.

**Burn rate**: How fast you are spending it. 1.0 = on pace to exhaust in 30 days. 14.4 = will exhaust in 2 days.

```yaml
# Prometheus alerting rule: fast burn rate
groups:
  - name: slo-alerts
    rules:
      - alert: HighErrorBurnRate
        # 14.4x burn rate over 1h AND 6x burn rate over 6h
        # This catches real incidents, not blips
        expr: |
          (
            sum(rate(http_errors_total[1h])) / sum(rate(http_requests_total[1h]))
          ) > (14.4 * 0.001)
          and
          (
            sum(rate(http_errors_total[6h])) / sum(rate(http_requests_total[6h]))
          ) > (6 * 0.001)
        labels:
          severity: critical
        annotations:
          summary: "High error burn rate - SLO at risk"
          description: "Error rate is consuming error budget 14x faster than sustainable."

      # Add a second rule for slow burn: 3x over 1d AND 1x over 3d -> severity: warning
```

### Incident Response Flow

Alert fires -> Grafana dashboard (error rate, latency, throughput) -> find slow traces in Jaeger/Tempo for the time window -> grab trace ID -> search logs by trace ID across all services -> full request context -> root cause.

---

## Java Parallel: Full Comparison

| Concern | Spring Boot Ecosystem | Node.js Ecosystem |
|---------|----------------------|------------------|
| Metrics library | Micrometer | OpenTelemetry Metrics / prom-client |
| Tracing library | Micrometer Tracing (ex-Sleuth) | @opentelemetry/sdk-node |
| Auto-instrumentation | Spring Boot Actuator + Micrometer auto-config | @opentelemetry/auto-instrumentations-node |
| Metrics endpoint | `/actuator/metrics`, `/actuator/prometheus` | Custom `/metrics` with prom-client |
| Log correlation | MDC (ThreadLocal) | AsyncLocalStorage |
| Structured logging | Logback JSON encoder | pino (JSON by default) |
| Logger API | SLF4J facade | pino directly |
| Health endpoint | `/actuator/health` (auto) | Custom `/healthz` endpoints |
| Trace propagation | Auto via Micrometer Tracing | Auto via OTel SDK |
| Grafana integration | Micrometer -> Prometheus -> Grafana | OTel/prom-client -> Prometheus -> Grafana |

The Spring Boot ecosystem gives you more out of the box. The Node.js ecosystem requires more wiring but the OTel standard means it is fully interoperable -- a trace can start in a Spring Boot service and continue through Node.js services with no gaps.

---

## References

- [OpenTelemetry JavaScript Documentation](https://opentelemetry.io/docs/languages/js/)
- [Pino Logger](https://getpino.io/)
- [Node.js AsyncLocalStorage](https://nodejs.org/api/async_context.html#class-asynclocalstorage)
- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/)
- [Google SRE Book -- Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [Google SRE Workbook -- Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Prometheus Alerting Best Practices](https://prometheus.io/docs/practices/alerting/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [Grafana Tempo Documentation](https://grafana.com/docs/tempo/latest/)
- [Micrometer Documentation](https://micrometer.io/docs/)
- [Spring Boot Actuator -- Production-Ready Features](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
