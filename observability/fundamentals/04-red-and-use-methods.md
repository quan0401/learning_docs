---
title: "RED, USE, and the Four Golden Signals"
date: 2026-05-03
updated: 2026-05-03
tags: [observability, red, use, golden-signals, dashboards, monitoring]
---

# RED, USE, and the Four Golden Signals

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `observability` `red` `use` `golden-signals` `dashboards` `monitoring`

---

## Table of Contents

- [Summary](#summary)
- [1. Three Methods, Three Lenses](#1-three-methods-three-lenses)
- [2. The RED Method](#2-the-red-method)
  - [2.1 Origins](#21-origins)
  - [2.2 The Three Signals](#22-the-three-signals)
  - [2.3 What RED Is For](#23-what-red-is-for)
  - [2.4 Implementing RED with Histograms](#24-implementing-red-with-histograms)
- [3. The USE Method](#3-the-use-method)
  - [3.1 Origins](#31-origins)
  - [3.2 The Three Signals](#32-the-three-signals-1)
  - [3.3 What USE Is For](#33-what-use-is-for)
  - [3.4 Per-Resource Walkthrough](#34-per-resource-walkthrough)
  - [3.5 USE Checklist Pattern](#35-use-checklist-pattern)
- [4. The Four Golden Signals](#4-the-four-golden-signals)
  - [4.1 Origin](#41-origin)
  - [4.2 The Four Signals](#42-the-four-signals)
  - [4.3 Saturation Specifically](#43-saturation-specifically)
- [5. How They Overlap](#5-how-they-overlap)
- [6. Picking the Right Method](#6-picking-the-right-method)
- [7. Concrete Dashboard: RED for an HTTP Service](#7-concrete-dashboard-red-for-an-http-service)
- [8. Concrete Dashboard: USE for a Linux Host](#8-concrete-dashboard-use-for-a-linux-host)
- [9. Common Mistakes](#9-common-mistakes)
- [Related](#related)
- [References](#references)

---

## Summary

RED, USE, and the Four Golden Signals are checklists for "what should be on the dashboard." Each was popularized by a specific person in a specific context: USE by Brendan Gregg for resource-level performance analysis on systems, RED by Tom Wilkie at Weaveworks for microservices, and the Four Golden Signals by Rob Ewaschuk in the Google SRE Book. They are not competing — they target different layers of the stack and combining them is the norm. This doc states each method precisely, shows where they overlap, and walks through a concrete dashboard layout for both an HTTP service (RED) and a Linux host (USE).

---

## 1. Three Methods, Three Lenses

| Method | Best lens for… | Coined by |
|--------|----------------|-----------|
| **RED** — Rate, Errors, Duration | Request-driven services (HTTP, gRPC, queue consumers) | Tom Wilkie (Weaveworks / Grafana) |
| **USE** — Utilization, Saturation, Errors | Resources (CPU, memory, disks, NICs, IO devices) | Brendan Gregg |
| **Four Golden Signals** — Latency, Traffic, Errors, Saturation | User-facing service health from an SRE's perspective | Rob Ewaschuk (Google SRE Book) |

Treat them as orthogonal checklists. RED tells you whether the service *has a problem*. USE tells you which underlying resource is *the constraint*. The Golden Signals overlap with both.

---

## 2. The RED Method

### 2.1 Origins

Tom Wilkie introduced RED in a 2015 KubeCon talk and a Weaveworks blog post titled *The RED Method*. The premise: every microservice has the same shape of dashboard, regardless of what it does internally. If you instrument three numbers, you can compare every service in the fleet on equal footing.

### 2.2 The Three Signals

| Signal | Meaning | Typical metric type |
|--------|---------|---------------------|
| **R**ate | Requests per second served | Counter → `rate()` |
| **E**rrors | Failed requests per second (or fraction) | Counter (filtered to error statuses) |
| **D**uration | Time taken to serve a request, by percentile | Histogram |

These three numbers, **per service** and **per route/endpoint**, are sufficient for first-look triage of any request-driven service.

### 2.3 What RED Is For

- HTTP services
- gRPC services
- Message consumers (rate = messages handled / sec, errors = poison/failures / sec, duration = end-to-end processing time)
- Anything that processes a stream of discrete units of work

It is **not** a fit for resource-level monitoring (use USE) or for batch jobs that don't have a per-request shape (track makespan and success of the job instead).

### 2.4 Implementing RED with Histograms

OpenTelemetry semantic conventions emit a histogram named `http.server.request.duration` per HTTP server request, with attributes like `http.request.method`, `http.route`, `http.response.status_code`, and `server.address`. Prometheus scrapes the bucket counters.

```promql
# Rate (per route)
sum by (route) (
  rate(http_server_request_duration_seconds_count{service="checkout"}[5m])
)

# Errors (5xx fraction per route)
sum by (route) (
  rate(http_server_request_duration_seconds_count{
    service="checkout", status=~"5.."}[5m])
)
/
sum by (route) (
  rate(http_server_request_duration_seconds_count{service="checkout"}[5m])
)

# Duration p99 (per route)
histogram_quantile(
  0.99,
  sum by (le, route) (
    rate(http_server_request_duration_seconds_bucket{service="checkout"}[5m])
  )
)
```

See [Prometheus and PromQL](07-prometheus-and-promql.md) for `histogram_quantile` mechanics.

---

## 3. The USE Method

### 3.1 Origins

Brendan Gregg published the USE Method in 2012 (*"Thinking Methodically about Performance"*, ACM Queue, and his website [brendangregg.com/usemethod.html](https://www.brendangregg.com/usemethod.html)). Coming from a kernel/systems background, Gregg argued that resource-level performance bottlenecks recur across operating systems, and that a fixed checklist beats ad-hoc poking.

### 3.2 The Three Signals

For every resource:

| Signal | Definition |
|--------|------------|
| **U**tilization | The average time the resource was busy servicing work (e.g., 80% CPU busy) |
| **S**aturation | The degree to which the resource has extra work it can't service, often queued (run-queue length, swap, TCP retransmits) |
| **E**rrors | Count of error events (NIC drops, ECC errors, disk read errors) |

Saturation is the part most teams skip and the one that catches problems earliest. Utilization at 90% is fine if the queue is empty; utilization at 70% with a queue depth of 50 is on fire.

### 3.3 What USE Is For

- CPU
- Memory (capacity, page-in/out rate)
- Disks (IOPS, throughput, queue, retries)
- Network interfaces (bandwidth, drops, errors)
- File descriptors / handle limits
- Database connection pools
- Thread pools, semaphores, buffer pools

Anything that has a **fixed (or rate-limited) capacity** is a USE candidate.

### 3.4 Per-Resource Walkthrough

| Resource | Utilization | Saturation | Errors |
|----------|-------------|------------|--------|
| CPU | `1 - rate(node_cpu_seconds_total{mode="idle"}[5m])` | `node_load5 / cpu_count` (or run-queue length from `/proc/loadavg`) | machine-check / NMI counters |
| Memory | `1 - (MemAvailable / MemTotal)` | swap activity (`vmstat si/so`), OOM kill count | ECC error counters |
| Disk | `node_disk_io_time_seconds_total` rate | `node_disk_io_time_weighted_seconds_total` (queue × time) | `node_disk_read_errors_total` |
| Network | tx/rx bytes / link speed | `node_network_transmit_drop_total` rate, TCP `retrans/sec` | NIC error counters, malformed-frame counts |
| Connection pool | `active / max` | `pending acquisitions`, wait time histogram | acquisition-failed count |
| Thread pool | active / max | queued task count | rejected-execution count |

### 3.5 USE Checklist Pattern

Gregg recommends generating a static checklist per host class: list every resource the OS exposes, then for each, name the metric for U, S, and E. Walk it top-to-bottom during incidents. This is duller than ad-hoc investigation and significantly faster.

---

## 4. The Four Golden Signals

### 4.1 Origin

Rob Ewaschuk's chapter "Monitoring Distributed Systems" in *Site Reliability Engineering* (Beyer et al., 2016) names them.

> If you can only measure four metrics of your user-facing system, focus on these four.

### 4.2 The Four Signals

| Signal | What it is |
|--------|------------|
| **Latency** | Time to service a request — for both successes and failures separately |
| **Traffic** | Demand on the system (requests/sec, transactions/sec, network bytes/sec) |
| **Errors** | Rate of requests that fail — explicitly (5xx), implicitly (wrong content), or by policy (too slow) |
| **Saturation** | How "full" the service is — typically the most-utilized constrained resource |

### 4.3 Saturation Specifically

Saturation in the Golden Signals leans toward the USE meaning: how close are you to running out of headroom? The SRE Book gives the example of a service whose memory is leaking — utilization is the snapshot, saturation tells you when you will hit the wall.

A pragmatic saturation indicator is **utilization-of-the-bottleneck-resource projected forward** — if memory will exceed limit in 4 hours at the current rate, that's a meaningful saturation signal.

---

## 5. How They Overlap

Mapping the methods onto each other:

| Concept | RED | USE | Golden Signals |
|---------|-----|-----|----------------|
| Demand | Rate | — | Traffic |
| Failure | Errors | Errors | Errors |
| Time-to-serve | Duration | (kind of) | Latency |
| Resource pressure | — | Utilization + Saturation | Saturation |

Practical synthesis:

- **Service dashboards**: RED + a saturation panel for the dominant bottleneck.
- **Host / infrastructure dashboards**: USE.
- **SLO dashboards**: derived from RED (success ratio, latency-objective ratio).
- **Combined "service summary"**: RED on top, with linked drill-downs to per-resource USE panels for the host/pod the service runs on.

---

## 6. Picking the Right Method

| Layer | Method |
|-------|--------|
| HTTP / gRPC / RPC service | RED |
| Message-driven worker | RED (rate = consumed/sec, errors = DLQ rate, duration = handler latency) |
| Queue itself | USE (utilization = depth/capacity, saturation = consumer lag) |
| Database | RED on queries + USE on connection pool, buffers, disk |
| Linux / Windows host | USE |
| Kubernetes node | USE (CPU, memory, disk, network) |
| Pod | RED (if it's a service) + USE on its limits |
| Cache (Redis, Memcached) | RED on commands + USE on memory and eviction rate |
| User-facing top-level | Four Golden Signals + SLO |

---

## 7. Concrete Dashboard: RED for an HTTP Service

A 5-row, 3-column dashboard layout — works for any HTTP service, regardless of business logic.

```
┌─────────────────────────────────────────────────────────────────┐
│  Service: checkout    env: production    region: us-east-1      │
├──────────────────┬──────────────────┬───────────────────────────┤
│  Row 1 — Top-level RED                                           │
├──────────────────┼──────────────────┼───────────────────────────┤
│  Total req/s     │  Error rate (5xx)│  p50 / p95 / p99 latency  │
│  (gauge + line)  │  (gauge + line)  │  (multi-line)             │
├──────────────────┴──────────────────┴───────────────────────────┤
│  Row 2 — RED per route (top 10 by traffic)                       │
│  table:                                                          │
│  route | rps | err% | p99 ms | trend                             │
├──────────────────┬──────────────────┬───────────────────────────┤
│  Row 3 — Saturation                                               │
│  CPU on pods     │  Heap used (Java)│  Event-loop lag (Node)    │
│                  │  / RSS           │  / GC pause time          │
├──────────────────┼──────────────────┼───────────────────────────┤
│  Row 4 — Downstream RED                                           │
│  DB query p99    │  Cache hit ratio │  Outbound HTTP error rate │
├──────────────────┴──────────────────┴───────────────────────────┤
│  Row 5 — Errors by class                                          │
│  stacked bar by status_code, exception.type, error_code           │
└──────────────────────────────────────────────────────────────────┘
```

Pin a `traceparent`/`trace_id` filter so engineers can scope to a specific incident's request set.

---

## 8. Concrete Dashboard: USE for a Linux Host

Driven by Prometheus `node_exporter` and `process_exporter`:

| Resource | Utilization | Saturation | Errors |
|----------|-------------|------------|--------|
| CPU | `1 - avg by (cpu)(rate(node_cpu_seconds_total{mode="idle"}[5m]))` | `node_load5 / cpu_count` | `rate(node_machine_check_event_total[5m])` |
| Memory | `1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)` | `rate(node_vmstat_pswpin[5m]) + rate(node_vmstat_pswpout[5m])`, `kube_pod_container_status_terminated_reason{reason="OOMKilled"}` | `node_memory_HardwareCorrupted_bytes` |
| Disk | `rate(node_disk_io_time_seconds_total[5m])` | `rate(node_disk_io_time_weighted_seconds_total[5m])` | `rate(node_disk_io_errors_total[5m])` |
| Network | `rate(node_network_receive_bytes_total[5m])` / link speed | `rate(node_network_receive_drop_total[5m])` + TCP retrans | `rate(node_network_receive_errs_total[5m])` |
| File descriptors | `process_open_fds / process_max_fds` | n/a | EMFILE-class errors in app logs |
| TCP sockets | `node_sockstat_TCP_inuse` | TIME_WAIT count, retrans rate | `node_netstat_Tcp_RetransSegs` rate |

A pod-level USE dashboard adds:

- `container_cpu_cfs_throttled_seconds_total` rate (saturation under cgroups limits)
- `container_memory_working_set_bytes` vs limit (utilization toward OOM kill)

---

## 9. Common Mistakes

- **Average latency on a Duration panel.** Use percentiles from a histogram. Means hide tails.
- **Errors as a count without normalizing by traffic.** A 500/min spike means different things at 1k RPS and 10 RPS.
- **No saturation panel on RED dashboards.** A service can pass RED while quietly running out of memory; saturation is the early-warning channel.
- **USE applied to services.** "Utilization of the checkout service" is meaningless. Use RED.
- **CPU at 100% treated as failure.** It is the goal for batch workers; for latency-sensitive services it's a problem only if saturation (queueing) is also rising.
- **One dashboard per metric.** Each layer (service, host, dependency) deserves its own panel set, linked by labels and trace IDs, not a single mega-dashboard with 80 panels nobody reads.
- **Missing per-route breakdowns.** Aggregate RED hides routes that are silently broken under high-volume routes that are healthy.

---

## Related

- [Three Pillars: Metrics, Logs, Traces](01-three-pillars-metrics-logs-traces.md)
- [SLI / SLO / Error Budgets](03-sli-slo-error-budgets.md)
- [Prometheus and PromQL](07-prometheus-and-promql.md)
- [Spring Boot Actuator Deep Dive](../../java/actuator-deep-dive.md)
- [Reactive Observability (Micrometer + Reactor)](../../java/reactive-observability.md)
- [Node.js Production Observability](../../typescript/production/observability.md)
- [System Design: RED/USE/Golden Signals](../../system-design/performance-observability/monitoring-red-use-golden-signals.md)
- [Network Observability](../../networking/advanced/network-observability.md)

---

## References

- **Tom Wilkie — "The RED Method: How to instrument your services"** (Weaveworks blog / Grafana). https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/
- **Tom Wilkie — "Monitoring and Observability with USE and RED"** (talk, Grafana Labs). https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/
- **Brendan Gregg — "The USE Method."** https://www.brendangregg.com/usemethod.html
- **Brendan Gregg — "Thinking Methodically About Performance,"** *ACM Queue*, 2012. https://queue.acm.org/detail.cfm?id=2413037
- **Beyer, Jones, Petoff, Murphy (eds.) — *Site Reliability Engineering***, Chapter 6 ("Monitoring Distributed Systems"). https://sre.google/sre-book/monitoring-distributed-systems/
- **Brendan Gregg — *Systems Performance: Enterprise and the Cloud*, 2nd ed.** (Addison-Wesley, 2020). USE method, Chapter 2.
- **OpenTelemetry HTTP semantic conventions** (latency / status / route attributes). https://opentelemetry.io/docs/specs/semconv/http/
- **Prometheus node_exporter — exposed metrics reference.** https://github.com/prometheus/node_exporter
