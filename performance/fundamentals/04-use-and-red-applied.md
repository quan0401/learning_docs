---
title: "USE and RED Applied — Dashboards, Alerts, and Drill-Down Workflows"
date: 2026-05-03
updated: 2026-05-03
tags: [performance, observability, use-method, red-method, dashboards, alerting]
---

# USE and RED Applied — Dashboards, Alerts, and Drill-Down Workflows

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `performance` `observability` `use-method` `red-method` `dashboards` `alerting`

---

## Table of Contents

- [Summary](#summary)
- [1. Two Methods, One Goal](#1-two-methods-one-goal)
  - [1.1 RED for Services (Request-Centric)](#11-red-for-services-request-centric)
  - [1.2 USE for Resources (Resource-Centric)](#12-use-for-resources-resource-centric)
  - [1.3 Why You Need Both](#13-why-you-need-both)
- [2. RED in Practice](#2-red-in-practice)
  - [2.1 What to Measure](#21-what-to-measure)
  - [2.2 Per-Endpoint vs Per-Service](#22-per-endpoint-vs-per-service)
  - [2.3 RED Dashboard Layout](#23-red-dashboard-layout)
  - [2.4 RED Alerting](#24-red-alerting)
- [3. USE in Practice](#3-use-in-practice)
  - [3.1 Resources to Cover](#31-resources-to-cover)
  - [3.2 Utilization, Saturation, Errors — Definitions](#32-utilization-saturation-errors--definitions)
  - [3.3 USE Dashboard Layout](#33-use-dashboard-layout)
  - [3.4 USE Alerting Pitfalls](#34-use-alerting-pitfalls)
- [4. The Joined Workflow — Incident Drill-Down](#4-the-joined-workflow--incident-drill-down)
  - [4.1 The Decision Tree](#41-the-decision-tree)
  - [4.2 A Concrete Example](#42-a-concrete-example)
- [5. Golden Signals — Google's Synthesis](#5-golden-signals--googles-synthesis)
- [6. Cardinality Discipline](#6-cardinality-discipline)
- [Related](#related)
- [References](#references)

---

## Summary

USE and RED are the two simplest, most useful checklists in performance observability. RED (Requests, Errors, Duration) tells you what your **service** looks like to its callers; USE (Utilization, Saturation, Errors) tells you what each **resource** is doing under the hood. The two views are complementary — RED detects user-visible degradation; USE explains why it's happening. Used together they form a tight drill-down workflow: an alert fires on RED, you drill into USE on the constrained resource, and the resource that's saturating points you at the fix. This doc translates the two methods into concrete dashboards, alert thresholds, and the question-driven workflow that incident responders actually use. RED is from Tom Wilkie (Weaveworks); USE is from Brendan Gregg. Google's "Four Golden Signals" combines them.

---

## 1. Two Methods, One Goal

### 1.1 RED for Services (Request-Centric)

For every service, track:

| Metric | What it tells you |
|--------|------|
| **Rate** | Requests per second. The traffic the service is receiving. |
| **Errors** | Per-second count or rate of failed requests (HTTP 5xx, gRPC non-OK, exceptions). |
| **Duration** | Distribution of request durations — emit as a histogram, view as p50/p95/p99/p99.9. |

RED gives you the **outside-in** view. A service is healthy from the perspective of its callers when its rate is in expected range, error rate is below SLO, and duration percentiles are below latency SLO.

### 1.2 USE for Resources (Resource-Centric)

For every **resource** (CPU, memory, disk, network, database, thread pool, connection pool, queue), track:

| Metric | What it tells you |
|--------|------|
| **Utilization** | Fraction of time the resource is busy (CPU %), or fraction of capacity used (queue depth / queue size). |
| **Saturation** | Degree to which work is *queued* on this resource — work that can't be served immediately. Run-queue depth, queue length, swap usage, throttling time. |
| **Errors** | Errors *attributable to the resource* — disk read errors, NIC drops, memory allocation failures, lock acquisition timeouts. |

USE gives you the **inside-out** view. The resources are healthy when utilization is reasonable, no resource is saturated, and the resource isn't reporting errors.

### 1.3 Why You Need Both

RED alone tells you *that* something is slow. It does not tell you *why*. Two services with identical RED dashboards (same rate, same errors, same p99 latency) can have completely different USE profiles — one may be CPU-bound, the other waiting on a saturated DB connection pool.

USE alone tells you that a resource is saturated. It does not tell you whether users are suffering. CPU at 95% on a batch worker is a feature, not a bug.

The pair is what gives you a drill-down workflow: alert on RED (user-visible), diagnose with USE (resource-attributable cause), fix the constrained resource.

---

## 2. RED in Practice

### 2.1 What to Measure

For every HTTP/gRPC service:

```text
http_requests_total{service, route, method, status_class}     # counter
http_request_duration_seconds_bucket{service, route, method}  # histogram
```

- `route` should be a templated path (`/users/:id`), not the raw URL — otherwise cardinality explodes.
- `status_class` (`2xx`, `3xx`, `4xx`, `5xx`) is much cheaper than full status codes for alerting.
- Histogram buckets should bracket your SLO. If your p99 SLO is 200 ms, buckets at `[0.005, 0.01, 0.025, 0.05, 0.1, 0.2, 0.5, 1, 2.5, +Inf]` give you the resolution to compute p99 accurately. Default buckets often miss the band where your tail lives.

For Spring Boot, this is provided by `spring-boot-starter-actuator` + Micrometer with the `http.server.requests` timer. For Node.js, libraries like `prom-client` and OpenTelemetry's HTTP instrumentation emit equivalent metrics.

### 2.2 Per-Endpoint vs Per-Service

Per-service RED is too coarse for diagnosis. A regression in one endpoint that handles 5% of traffic is invisible in service-wide p99 if other endpoints are fast and high-traffic.

Per-endpoint RED is what you want. The alert fires per endpoint or per route; the dashboard breaks out latency by endpoint. The cost is metric cardinality (one histogram per route × method), so be ruthless about route templating.

### 2.3 RED Dashboard Layout

A useful per-service RED dashboard has three rows × N columns (one column per endpoint or per traffic class):

```
+----------------+----------------+----------------+
|  Rate          |  Rate          |  Rate          |
|  (req/s)       |  (req/s)       |  (req/s)       |
+----------------+----------------+----------------+
|  Errors        |  Errors        |  Errors        |
|  (% of traffic)|  (% of traffic)|  (% of traffic)|
+----------------+----------------+----------------+
|  Duration      |  Duration      |  Duration      |
|  p50/p95/p99   |  p50/p95/p99   |  p50/p95/p99   |
+----------------+----------------+----------------+
   /api/search    /api/checkout    /api/profile
```

Plus an overlay row showing all endpoints' p99 stacked, so a regression in one endpoint stands out.

### 2.4 RED Alerting

Concrete alert rules using Prometheus-style PromQL:

```yaml
# Error rate spike (per endpoint, 5-minute window)
- alert: HighErrorRate
  expr: |
    sum by (service, route) (rate(http_requests_total{status_class="5xx"}[5m]))
    /
    sum by (service, route) (rate(http_requests_total[5m]))
    > 0.01
  for: 5m

# p99 latency above SLO
- alert: P99LatencyAboveSLO
  expr: |
    histogram_quantile(0.99,
      sum by (le, service, route) (rate(http_request_duration_seconds_bucket[5m]))
    ) > 0.2   # 200 ms SLO
  for: 10m
```

Notes:

- Use `sum by (le, ...)` *inside* `histogram_quantile` so percentiles are computed on aggregated buckets, not pod-local pre-percentiles. See [Tail Latency](03-tail-latency-and-p99.md#why-you-cannot-average-percentiles).
- A 5-minute window suppresses single-minute blips. A 10-minute window for `for:` reduces flapping.
- Burn-rate alerts (multi-window, multi-burn-rate) are more refined and detect both fast burns and slow burns of error budget. See Google SRE workbook chapter on alerting on SLOs.

---

## 3. USE in Practice

### 3.1 Resources to Cover

Brendan Gregg's USE checklist for a Linux server:

| Resource | Utilization | Saturation | Errors |
|----------|-------------|------------|--------|
| **CPU** | `%CPU` per core | run queue depth (`vmstat r`), `pressure_cpu`, scheduler latency | machine check exceptions (rare) |
| **Memory** | RSS / total | swap in-rate, page-out rate, oom-kills, `pressure_memory` | allocation failures, OOM events |
| **Disk** | per-disk %busy (`iostat`) | `aqu-sz` (avg queue size), `pressure_io` | I/O errors, smart counters |
| **Network** | bandwidth used / capacity | NIC TX/RX queue overflow, dropped packets | NIC errors, CRC errors |
| **DB connections** | in-use / pool size | pending acquisitions in pool | connection acquisition timeouts |
| **Thread pool** | active / max threads | queued tasks, rejection count | task rejections |

Each application-level resource (pool, queue, cache) deserves its own row. The list above is a starting point, not exhaustive.

### 3.2 Utilization, Saturation, Errors — Definitions

A common mistake is to merge utilization and saturation. They are different things and they cross different lines:

- **Utilization** = fraction of capacity in use. Bounded by definition (0-100%). High utilization is not necessarily bad — see "running at 60% to leave headroom" in [Capacity Planning](07-capacity-planning.md).
- **Saturation** = work that can't be served right now and is *queued or blocked*. Utilization can be 100% with saturation = 0 (the resource is busy but no extra work is waiting). Saturation can be > 0 with utilization < 100% if the resource is bursty.
- **Errors** = countable error events attributable to the resource.

A CPU running at 100% with zero run-queue depth is fully utilized but not saturated. A CPU running at 70% with run-queue depth 5 is *saturated* — there's queued work despite headroom — usually meaning a single core is hot while others are idle (poor scheduling, sticky threads).

### 3.3 USE Dashboard Layout

For each host or workload, one USE panel per resource:

```
+--------------------+--------------------+--------------------+
| CPU Utilization %  | CPU Run Queue Depth| CPU Throttle Events|
| (per core stacked) | (sum)              | (cgroup throttling)|
+--------------------+--------------------+--------------------+
| Memory RSS / Limit | Swap I/O           | OOM Kills          |
+--------------------+--------------------+--------------------+
| Disk Util          | Disk Queue Depth   | Disk I/O Errors    |
+--------------------+--------------------+--------------------+
| Network bps        | NIC Drops          | NIC Errors         |
+--------------------+--------------------+--------------------+
| DB Pool In-Use     | DB Pool Pending    | DB Connection Errors|
+--------------------+--------------------+--------------------+
```

Linux's PSI (Pressure Stall Information) — `pressure_cpu`, `pressure_io`, `pressure_memory` — is an excellent saturation signal: it reports the fraction of time *some* task was stalled waiting on the resource, regardless of utilization. PSI is available in `/proc/pressure/*` since kernel 4.20.

### 3.4 USE Alerting Pitfalls

Common bad alerts:

- **"CPU > 80%"** — meaningless on its own. A batch worker should sit at 95%; an interactive service should fire at 60%. Set thresholds per workload class.
- **"Memory > 90%"** — modern Linux uses unused memory for the page cache; high memory utilization is normal and good. Alert on **available memory below threshold** or on **PSI memory pressure rising**, not on % used.
- **"Disk > 80%"** — same issue as memory. Alert on free space below threshold, IOPS approaching limit, or queue depth rising.

Better alerts target saturation and errors:

- Run-queue depth > 1 per core for 10 minutes (saturation).
- PSI cpu pressure > 0.5 for 10 minutes (a process is stalling on CPU half the time).
- DB pool pending acquisitions > 0 for 5 minutes (the pool is the bottleneck).
- NIC dropped packets > 0 in 5-minute window (something is wrong; you should never drop in a healthy network).

---

## 4. The Joined Workflow — Incident Drill-Down

### 4.1 The Decision Tree

When a RED alert fires, the workflow is:

```text
RED alert: p99 latency above SLO on /api/search
                       │
                       ▼
         Is the rate also up significantly?
                       │
            ┌──────────┴──────────┐
           Yes                    No
            │                     │
            ▼                     ▼
   Load-driven regression   Workload-shape change
   (USE check first)        (look at endpoint mix,
                             external dependencies,
                             upstream changes)
            │
            ▼
   Which resource is saturated?
            │
   ┌────────┼────────┬────────────┬─────────────┐
   ▼        ▼        ▼            ▼             ▼
  CPU    Memory   Disk          Network     DB pool / Cache
   │        │       │              │             │
   ▼        ▼       ▼              ▼             ▼
  Profile  Heap    iostat /     packet drops, Pool size,
  CPU      dump,   IOPS limit,  NIC bw,       slow queries,
  flame    GC log  PSI io       PSI           cache hit rate
  graph
```

### 4.2 A Concrete Example

**Alert:** `/api/checkout` p99 latency > 500ms for 10 minutes.

**Step 1 — RED sweep.** Rate is normal (4,000 req/s, baseline). Error rate is normal (0.1%). Duration p50 went from 30 ms to 80 ms (also up). p99 went from 200 ms to 700 ms.

**Step 2 — USE on the host.** CPU 75% (was 60% baseline). Memory normal. Disk idle. Network idle. DB pool at 95% utilization (was 60%), with non-zero pending acquisitions.

**Step 3 — Drill into DB.** Postgres dashboard shows query duration p99 spiked. Check `pg_stat_statements`: a query with `WHERE created_at > NOW() - INTERVAL '7 days'` is now scanning sequentially instead of using the index. A bulk insert overnight pushed the table size past the planner's threshold for index use.

**Step 4 — Fix.** `ANALYZE` the table; add a partial index; or rewrite the query. p99 returns to 200 ms.

The point is the **path**: RED → USE → resource-specific dashboard → root cause. Without USE, you'd be guessing. Without RED, you'd be staring at "CPU is at 75%" and not knowing whether it matters.

See [System Design — RED/USE/Golden Signals](../../system-design/performance-observability/monitoring-red-use-golden-signals.md) for the broader observability picture and [Database — EXPLAIN ANALYZE Guide](../../database/query-optimization/explain-analyze-guide.md) for the DB-side drill-down.

---

## 5. Golden Signals — Google's Synthesis

Google's SRE book proposes the **Four Golden Signals** as a higher-level synthesis:

| Signal | Definition |
|--------|-----------|
| **Latency** | Time to serve a request — both successful and failed. Measure them separately; failed requests can be artificially fast (timeout to error). |
| **Traffic** | Demand on the system. RPS for HTTP, transactions/sec for a DB, sessions for WebSockets. |
| **Errors** | Rate of failed requests. Includes failures, wrong-content responses, and SLO violations. |
| **Saturation** | How "full" the most constrained resource is. The single most useful resource metric. |

Latency + Traffic + Errors ≈ RED. Saturation ≈ the most important USE component. The Golden Signals deliberately fold Utilization into Saturation because, in their experience, the alert should fire on saturation (work queueing) not on utilization alone.

For a service, instrument all four. For a resource, USE gives you the granular view you'll need when the Golden-Signal alert fires.

---

## 6. Cardinality Discipline

USE and RED both produce metrics with labels. Labels are cardinality, and Prometheus (and most TSDBs) have hard limits on cardinality before they collapse.

Bad labels (high cardinality):

- `user_id` — millions of values.
- `request_id` — unique per request.
- `path` (raw URL with query string) — unbounded.

Good labels (bounded cardinality):

- `service` — tens.
- `route` (templated path) — hundreds.
- `method`, `status_class` — fixed small sets.
- `region`, `availability_zone`, `pod_role` — bounded.

A general rule: a label's cardinality should be O(small number), and the *product* of cardinalities for a metric should be bounded. `service × route × method × status_class` for `http_requests_total` is fine if each is bounded; adding `user_id` blows it up.

For high-cardinality data (per-user, per-trace, per-request), use **logs** or **traces**, not metrics. RED metrics summarize; logs and traces preserve the detail.

---

## Related

- [Latency, Throughput, Percentiles](01-latency-throughput-percentiles.md)
- [Tail Latency and p99](03-tail-latency-and-p99.md)
- [Flame Graphs (CPU and Off-CPU)](05-flame-graphs-cpu-and-off-cpu.md)
- [System Design — RED/USE/Golden Signals](../../system-design/performance-observability/monitoring-red-use-golden-signals.md)
- [System Design — Dashboards / Runbooks / On-Call](../../system-design/performance-observability/dashboards-runbooks-on-call.md)
- [Java — Distributed Tracing](../../java/observability/distributed-tracing.md)
- [TypeScript — Node Observability](../../typescript/production/observability.md)
- [Database — Monitoring](../../database/operations/monitoring.md)

---

## References

- **Brendan Gregg — "The USE Method".** https://www.brendangregg.com/USEmethod/use-method.html — the original USE checklist.
- **Brendan Gregg — "USE Method: Linux Performance Checklist".** https://www.brendangregg.com/USEmethod/use-linux.html — concrete tools (`vmstat`, `iostat`, `mpstat`, `sar`, `ss`, `nicstat`).
- **Tom Wilkie — "The RED Method: How to Instrument Your Services".** https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/ — the original RED writeup.
- **Google SRE Book — "Monitoring Distributed Systems" (Chapter 6).** https://sre.google/sre-book/monitoring-distributed-systems/ — the Four Golden Signals.
- **Google SRE Workbook — "Alerting on SLOs" (Chapter 5).** https://sre.google/workbook/alerting-on-slos/ — burn-rate alerting.
- **Linux Pressure Stall Information (PSI).** https://www.kernel.org/doc/html/latest/accounting/psi.html — saturation signal.
- **Prometheus — Histograms and Summaries.** https://prometheus.io/docs/practices/histograms/ — bucket selection and `histogram_quantile`.
- **Cindy Sridharan — "Distributed Systems Observability", O'Reilly, 2018.** https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/ — broader observability framework around RED/USE.
- **Charity Majors et al. — "Observability Engineering", O'Reilly, 2022.** https://www.oreilly.com/library/view/observability-engineering/9781492076438/ — high-cardinality observability beyond metrics.
