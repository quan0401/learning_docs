---
title: "Monitoring — RED, USE, and Golden Signals"
date: 2026-04-26
updated: 2026-04-26
tags: [system-design, observability, monitoring, slo]
---

# Monitoring — RED, USE, and Golden Signals

**Date:** 2026-04-26 | **Updated:** 2026-04-26
**Tags:** `system-design` `observability` `monitoring` `slo`

## Table of Contents

- [Summary](#summary)
- [Overview — When Each Method Applies](#overview--when-each-method-applies)
- [Key Concepts](#key-concepts)
  - [RED — Rate, Errors, Duration](#red--rate-errors-duration)
  - [USE — Utilization, Saturation, Errors](#use--utilization-saturation-errors)
  - [The Four Golden Signals](#the-four-golden-signals)
  - [Symptom vs Cause Alerting](#symptom-vs-cause-alerting)
  - [SLO-Based Alerting and Burn Rate](#slo-based-alerting-and-burn-rate)
  - [Multi-Window Multi-Burn-Rate Alerts](#multi-window-multi-burn-rate-alerts)
  - [Error Budget Mechanics](#error-budget-mechanics)
  - [Alert Fatigue Mitigation](#alert-fatigue-mitigation)
- [Trade-offs](#trade-offs)
- [Code Examples](#code-examples)
  - [PromQL — RED Metrics for an HTTP Service](#promql--red-metrics-for-an-http-service)
  - [PromQL — USE Metrics for a Host](#promql--use-metrics-for-a-host)
  - [PromQL — Golden Signals on a Gateway](#promql--golden-signals-on-a-gateway)
  - [PromQL — Multi-Burn-Rate Alert Rule](#promql--multi-burn-rate-alert-rule)
- [Real-World Uses](#real-world-uses)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

You cannot meaningfully operate a distributed system without a small, opinionated set of metrics and a discipline for alerting on them. Three frameworks dominate: **RED** (Rate, Errors, Duration — Tom Wilkie) for request-driven services, **USE** (Utilization, Saturation, Errors — Brendan Gregg) for hardware and resources, and the **Four Golden Signals** (Latency, Traffic, Errors, Saturation — Google SRE Book) for user-facing systems. They overlap deliberately. Once metrics exist, the next problem is *alerting on the right thing*: symptoms users feel, not causes operators imagine. The mature pattern is **SLO-based, burn-rate alerting** — page only when you are consuming the error budget faster than the SLO allows, and only with enough confidence (multi-window, multi-burn-rate). This doc covers all three monitoring methods, when to use each, the math of error budgets, the multi-window multi-burn-rate alert pattern from the Google SRE Workbook, and the anti-patterns that produce alert fatigue.

## Overview — When Each Method Applies

The three frameworks are not competitors. They cover different layers of the stack and answer different questions.

```text
                       USER
                         |
            +------------+------------+
            |   Golden Signals on the |
            |   user-facing entrypoint|
            +------------+------------+
                         |
                +--------+---------+
                |   service A      |   <-- RED on each service
                +--------+---------+
                         |
                +--------+---------+
                |   service B      |   <-- RED
                +--------+---------+
                         |
              +----------+----------+
              |  database / cache   |   <-- USE on the box,
              |  / kafka broker     |       RED on the protocol
              +---------------------+
                         |
                  CPU, mem, disk,    <-- USE
                  NIC, PSI, fd
```

| Method | Origin | Subject | Best for |
|--------|--------|---------|----------|
| **RED** | Tom Wilkie (Weaveworks, ~2015) | Request-driven services | Any HTTP/gRPC/RPC microservice — what callers experience |
| **USE** | Brendan Gregg (Sun/Netflix, 2012) | Resources (CPU, memory, disk, NIC, mutex) | Capacity, hardware saturation, kernel-level performance |
| **Golden Signals** | Google SRE Book ch. 6 (Beyer et al., 2016) | User-facing systems | The smallest set you would put on a single overview dashboard |

**Practical heuristic.** Apply Golden Signals to the **edge** (where users are). Apply RED to **every service** between the edge and storage. Apply USE to **every resource** the services run on. The three together let you answer in this order: *Are users hurting?* (Golden Signals) → *Which service is misbehaving?* (RED) → *Why is that service struggling?* (USE).

## Key Concepts

### RED — Rate, Errors, Duration

Tom Wilkie distilled a useful service-level pattern from Google's golden-signals work and Weaveworks production experience:

- **Rate** — requests per second the service is handling.
- **Errors** — how many of those requests are failing (5xx, gRPC `Internal`, timeouts, client-visible failures).
- **Duration** — distribution of how long requests take, *as percentiles*, not averages.

RED is **per-service** and **per-route** when possible. It is what an SRE on call needs to know about a service in five seconds. The choice of *Duration* over *latency* is intentional: duration is observed at the server boundary; client-side latency includes network and is the remit of the caller's RED.

```text
service-foo
  rate     :  ████████████████░░░░░░  1,250 req/s
  errors   :  █░░░░░░░░░░░░░░░░░░░░░  0.3% (3.7 req/s)
  duration :  p50=12ms  p95=84ms  p99=410ms  p99.9=1.2s
```

**Why averages are a trap.** Duration distributions in real systems are heavy-tailed and bimodal. A 50 ms average can hide a p99 of 5 s — and the p99 is what users with 100 requests on a page actually experience. Always alert on **percentiles**, computed from histograms (Prometheus `histogram_quantile`, OpenTelemetry exponential histograms). Never alert on a moving average alone.

### USE — Utilization, Saturation, Errors

Brendan Gregg formulated USE for **resources** — anything finite that the workload competes for:

- **Utilization** — average fraction of time the resource was busy (or, for memory, the fraction in use).
- **Saturation** — degree to which extra work is queued up because the resource is full (run queue length, swap usage, PSI pressure, TCP accept-queue depth).
- **Errors** — error events on the resource itself (ECC corrections, disk read errors, NIC drops, OOM kills).

The killer insight is **saturation**. A CPU at 80% utilization with empty run queues is healthy; a CPU at 80% utilization with a run-queue length of 30 is not. Linux Pressure Stall Information (PSI), `vmstat`'s `r` column, NIC `tx_dropped`, and storage queue depth all measure saturation directly.

| Resource | Utilization | Saturation | Errors |
|----------|-------------|------------|--------|
| CPU | `1 - rate(node_cpu_seconds_total{mode="idle"}[1m])` | run-queue length, PSI `cpu/some` | thermal throttles |
| Memory | `1 - MemAvailable / MemTotal` | swap-in rate, PSI `memory/full`, OOM kills | ECC corrections |
| Disk | `rate(node_disk_io_time_seconds_total[1m])` | average queue length, PSI `io/full` | read/write errors, SMART |
| NIC | `rate(node_network_*_bytes_total) / link_speed` | `tx_dropped`, retransmits, `recv-Q` | rx/tx errors |
| File descriptors | `process_open_fds / process_max_fds` | accept-queue overflow | `EMFILE` |
| DB connection pool | active / max | wait queue length, wait time | acquisition timeouts |

**USE is for capacity questions.** When RED tells you the service is slow and the cause is not in the code path, USE on the resources it consumes will usually tell you why.

### The Four Golden Signals

Google's SRE Book chapter 6 ("Monitoring Distributed Systems") proposes the smallest set of metrics worth putting on the overview dashboard for a user-facing service:

1. **Latency** — time to serve a request, segmented by success and failure. Failed-request latency matters: a 50 ms 500 is different from a 30 s timeout.
2. **Traffic** — demand on the system in domain-appropriate units: HTTP req/s, transactions/s, MB/s of streamed video, concurrent sessions.
3. **Errors** — failure rate, including silent failures (wrong content, policy violations, content-mismatch checksums).
4. **Saturation** — how full the service is — closest-to-the-bottleneck utilization. For a thread-pool-bound service it is queue depth; for a memory-bound service it is heap headroom.

Golden Signals overlap RED almost perfectly (Latency ≈ Duration, Traffic ≈ Rate, Errors ≈ Errors). The added dimension is **Saturation**, which RED leaves out and which is essential for predicting *future* failure rather than just describing the present one. Saturation is the leading indicator; Latency, Traffic, and Errors are coincident.

### Symptom vs Cause Alerting

Rob Ewaschuk's "My Philosophy on Alerting" (Google) articulates the rule that should govern every page:

> Alert on **symptoms**, not causes. A symptom is a thing a user can experience. A cause is a thing an operator suspects.

- **Symptom alert (good).** "p99 checkout latency > 800 ms for 10 minutes." Users feel this.
- **Cause alert (bad, usually).** "Cache hit rate dropped below 90%." Maybe nothing is wrong; maybe the symptom is that everything is fine because of an unrelated traffic shift.

Causes belong on **dashboards** and in **runbooks**. They help you diagnose during an incident; they are not what wakes you up. Many on-call rotations are noisy because every team has bolted a cause-alert on every metric they ever found interesting.

The right exception: page on causes that **always** lead to user-visible symptoms with **lead time** the symptom does not give you. "Disk will fill in 2 hours at the current write rate" is a cause alert worth keeping — you need lead time before the symptom appears.

### SLO-Based Alerting and Burn Rate

A **Service Level Objective (SLO)** is a target for the proportion of "good events" over a window — for example, *99.9% of HTTP requests return non-5xx within 500 ms over a rolling 28-day window*. The complement of the SLO is the **error budget**: 0.1% of requests, or about 40 minutes per 28 days at 100% availability equivalence.

The error budget reframes alerting. Instead of asking "is the error rate high right now?" you ask "are we burning the budget faster than 28 days will permit?"

**Burn rate** = (current error rate) / (error rate the SLO permits over the window).

| Burn rate | Time to exhaust budget at this rate (28-day SLO) |
|-----------|---------------------------------------------------|
| 1× | 28 days (the SLO target itself) |
| 2× | 14 days |
| 6× | ~4.7 days |
| 14.4× | 2 days (alert threshold for 1-hour window in Google's table) |
| 36× | ~18.6 hours |
| 1000× | ~40 minutes |

Burn-rate alerts are **proportional**: a service running 0.5% errors for an hour burns the same fraction of a 99% SLO that the same service running 0.05% errors burns of a 99.9% SLO. The alert fires when burning is fast enough to matter.

### Multi-Window Multi-Burn-Rate Alerts

A single threshold alert (e.g. "page if error rate > 0.5%") has two failure modes:

- **Slow detection of large outages.** A 10-minute averaging window may take 5+ minutes to cross the threshold during a 50% outage.
- **Noise on small spikes.** A 1-minute averaging window will fire on every brief blip.

Google's SRE Workbook (chapter 5, "Alerting on SLOs") proposes **multi-window, multi-burn-rate** alerting:

- A **page** fires when the burn rate is high over **both** a long window (an hour) and a short window (5 minutes). The long window provides confidence; the short window provides recency (so you do not alert on a problem that already ended).
- A **lower-urgency ticket** fires when a slower burn is sustained over much longer windows (6 hours and 30 minutes).

A common configuration for a 30-day SLO:

| Severity | Long window | Short window | Burn rate | Budget consumed if sustained |
|----------|-------------|--------------|-----------|------------------------------|
| Page | 1 hour | 5 minutes | 14.4× | 2% of 30-day budget in 1 hour |
| Page | 6 hours | 30 minutes | 6× | 5% in 6 hours |
| Ticket | 24 hours | 2 hours | 3× | 10% in 24 hours |
| Ticket | 72 hours | 6 hours | 1× | 10% in 72 hours |

Both the long and short window must exceed the burn-rate threshold for the alert to fire. This gives fast detection on large incidents (the short window crosses quickly), filters noise (the long window does not move on transient blips), and auto-resolves when the incident ends (the short window drops back below threshold).

### Error Budget Mechanics

Error budgets are the SRE invention that aligns reliability with development velocity. The mechanics:

- **Budget = 1 − SLO**, applied to the same window. 99.9% over 28 days = 40 min 19 s of unavailability or 0.1% of requests.
- **Budget consumption** is tracked continuously. Any failed request, any minute over latency target, any policy violation, eats into it.
- **Budget remaining** drives policy: when budget is healthy, ship features; when budget is exhausted, freeze risky launches and pay down reliability debt.

A common policy: if the rolling 28-day error budget is below 0%, releases of the affected service require a senior approver, or are frozen until budget recovers. This puts reliability decisions on the same incentive structure as feature work — both are scarce resources allocated by the team.

A subtle point: not every error eats the budget. **Excluded events** (planned maintenance, dependency outages outside the SLO scope, requests classified as bot traffic) are filtered out by your SLI definition. Get the SLI definition right before you negotiate the SLO; the SLI is what your alerts and budgets are computed from.

### Alert Fatigue Mitigation

Alert fatigue is the silent killer of on-call rotations. The mitigations stack:

1. **Page only on symptoms.** Move cause alerts to dashboards. (See [Symptom vs Cause Alerting](#symptom-vs-cause-alerting).)
2. **Use SLO burn-rate alerts**, not raw threshold alerts. Burn rate is proportional and self-resolving.
3. **Require both windows** in multi-window multi-burn-rate. Cuts false positives from short spikes.
4. **One alert per symptom**, not one per metric that contributes to it. Group with Alertmanager `group_by`, `group_wait`, `group_interval`.
5. **Inhibit downstream alerts.** If the API gateway is down, do not also page on every backend's error rate. Use Alertmanager `inhibit_rules` or equivalent.
6. **Tier severity correctly.** Pages wake people up. Tickets do not. If something does not need response within the hour, it should not be a page.
7. **Audit alert quality monthly.** For every page in the last 30 days, ask: *Was action required? Did we know what to do? Was the runbook accurate?* Delete or downgrade alerts that fail.
8. **Escalation policies that route, not flood.** Page primary; if no ack in 5 min, secondary; if no ack in 10 min, manager. Do not page everyone at once.
9. **Quiet hours and rotation health.** Track pages-per-shift; redesign alerts when it exceeds two per shift sustained.
10. **Runbooks attached to every alert.** An alert without a runbook entry is a half-finished alert.

The single biggest win is **deleting alerts**. A team with 200 alert rules where 180 fire occasionally is producing more noise than signal; a team with 20 alert rules that *always* mean something is responsive and well-rested.

## Trade-offs

| Choice | Upside | Downside |
|--------|--------|----------|
| RED everywhere | Universal language across services; works for any request-driven thing | Misses non-request work (cron, streaming consumers, queue drainers) |
| USE on every host | Catches capacity issues before symptoms | Volume of metrics; cardinality explodes if naive |
| Golden Signals on the edge only | One overview dashboard the whole org reads | Hides intra-service problems until they reach the edge |
| Burn-rate alerts | Proportional, self-resolving, fewer pages | Requires good SLI definitions and historical data |
| Multi-window multi-burn-rate | Best detection-vs-noise trade-off published | More PromQL; more rules to maintain; harder to explain |
| Error budgets as policy | Aligns reliability and velocity | Requires real organizational buy-in; pointless if budget exhaustion has no consequence |
| Symptom-only alerting | Wakes you up only when users hurt | You miss leading-indicator capacity problems unless paired with USE-based ticket alerts |
| Histograms for percentiles | Accurate p99/p99.9 across aggregations | High cardinality; expensive at scale; needs bucket tuning |
| Pre-aggregated summaries | Cheap | Cannot aggregate across instances correctly; lies at fleet scale |

## Code Examples

### PromQL — RED Metrics for an HTTP Service

Assuming the standard `http_requests_total` counter and `http_request_duration_seconds` histogram emitted by an instrumented service:

```promql
# Rate — requests per second over the last 5 minutes
sum by (service, route) (
  rate(http_requests_total{job="checkout"}[5m])
)

# Errors — 5xx ratio
sum by (service, route) (
  rate(http_requests_total{job="checkout", status=~"5.."}[5m])
)
/
sum by (service, route) (
  rate(http_requests_total{job="checkout"}[5m])
)

# Duration — p50, p95, p99 from a histogram
histogram_quantile(0.50,
  sum by (le, service, route) (
    rate(http_request_duration_seconds_bucket{job="checkout"}[5m])
  )
)

histogram_quantile(0.99,
  sum by (le, service, route) (
    rate(http_request_duration_seconds_bucket{job="checkout"}[5m])
  )
)
```

Note the order of operations on percentiles: `rate()` over `_bucket` series first, then `sum` over `le`, then `histogram_quantile`. Aggregating percentiles directly across instances is mathematically wrong; aggregating histograms first is correct.

### PromQL — USE Metrics for a Host

Using the `node_exporter` standard metrics:

```promql
# Utilization — CPU busy fraction across cores
1 - avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle"}[1m])
)

# Saturation — Pressure Stall Information (kernel >= 4.20)
# fraction of time at least one task was stalled on CPU
rate(node_pressure_cpu_waiting_seconds_total[1m])

# Saturation — run-queue length proxy: load1 / cores
node_load1 / count by (instance) (
  node_cpu_seconds_total{mode="idle"}
)

# Memory utilization
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Memory saturation — swap activity
rate(node_vmstat_pswpin[1m]) + rate(node_vmstat_pswpout[1m])

# Disk utilization — fraction of time the device was busy
rate(node_disk_io_time_seconds_total[1m])

# Disk saturation — average queue length
rate(node_disk_io_time_weighted_seconds_total[1m])

# NIC errors
rate(node_network_receive_errs_total[1m])
  + rate(node_network_transmit_errs_total[1m])

# NIC drops (saturation symptom)
rate(node_network_receive_drop_total[1m])
  + rate(node_network_transmit_drop_total[1m])
```

### PromQL — Golden Signals on a Gateway

For an edge-facing API gateway, you would put these four panels on the overview dashboard:

```promql
# 1. Latency — p99 of successful requests
histogram_quantile(0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{
      job="api-gateway", status!~"5.."
    }[5m])
  )
)

# 2. Traffic — total request rate
sum(rate(http_requests_total{job="api-gateway"}[1m]))

# 3. Errors — 5xx ratio
sum(rate(http_requests_total{job="api-gateway", status=~"5.."}[5m]))
/
sum(rate(http_requests_total{job="api-gateway"}[5m]))

# 4. Saturation — in-flight requests vs configured concurrency
sum(http_in_flight_requests{job="api-gateway"})
/
sum(http_max_concurrency{job="api-gateway"})
```

### PromQL — Multi-Burn-Rate Alert Rule

A two-window pair of alerts for a 99.9% availability SLO over a 30-day window. The burn-rate ratios (14.4× and 6×) come from the Google SRE Workbook table. The alert fires only when both the long and short windows exceed the threshold.

```yaml
groups:
- name: checkout-slo-burn-rate
  rules:

  # SLI: success ratio over multiple windows, recorded for reuse
  - record: job:slo_errors_per_request:ratio_rate5m
    expr: |
      sum(rate(http_requests_total{job="checkout", status=~"5.."}[5m]))
      /
      sum(rate(http_requests_total{job="checkout"}[5m]))

  - record: job:slo_errors_per_request:ratio_rate1h
    expr: |
      sum(rate(http_requests_total{job="checkout", status=~"5.."}[1h]))
      /
      sum(rate(http_requests_total{job="checkout"}[1h]))

  - record: job:slo_errors_per_request:ratio_rate30m
    expr: |
      sum(rate(http_requests_total{job="checkout", status=~"5.."}[30m]))
      /
      sum(rate(http_requests_total{job="checkout"}[30m]))

  - record: job:slo_errors_per_request:ratio_rate6h
    expr: |
      sum(rate(http_requests_total{job="checkout", status=~"5.."}[6h]))
      /
      sum(rate(http_requests_total{job="checkout"}[6h]))

  # Fast burn — page. 14.4x burn over 1h, confirmed by 5m short window.
  # Threshold = 14.4 * (1 - 0.999) = 0.0144
  - alert: CheckoutErrorBudgetFastBurn
    expr: |
      job:slo_errors_per_request:ratio_rate1h  > (14.4 * 0.001)
      and
      job:slo_errors_per_request:ratio_rate5m  > (14.4 * 0.001)
    for: 2m
    labels:
      severity: page
      slo: checkout-availability
    annotations:
      summary: "Checkout error budget burning at 14.4x (1h)"
      runbook: "https://runbooks.example.com/checkout/error-budget-fast-burn"

  # Slow burn — page. 6x burn over 6h, confirmed by 30m short window.
  # Threshold = 6 * (1 - 0.999) = 0.006
  - alert: CheckoutErrorBudgetSlowBurn
    expr: |
      job:slo_errors_per_request:ratio_rate6h  > (6 * 0.001)
      and
      job:slo_errors_per_request:ratio_rate30m > (6 * 0.001)
    for: 15m
    labels:
      severity: page
      slo: checkout-availability
    annotations:
      summary: "Checkout error budget burning at 6x (6h)"
      runbook: "https://runbooks.example.com/checkout/error-budget-slow-burn"
```

The pattern generalizes to latency SLOs by changing the SLI to *fraction of requests slower than the latency target*. The burn-rate ratios stay the same.

## Real-World Uses

| System | What they monitor and how |
|--------|---------------------------|
| **Google** | SRE-Book Golden Signals on every user-facing service; multi-window multi-burn-rate alerts; error budgets drive launch decisions |
| **Netflix** | RED on every microservice via Atlas/Mantis; USE on every instance; chaos-engineering paired with monitoring |
| **GitHub** | RED on Rails endpoints and gRPC services; SLO dashboards public via the [GitHub status page](https://www.githubstatus.com/) |
| **Cloudflare** | Per-data-center Golden Signals; multi-window burn-rate alerts on edge availability |
| **Shopify** | RED + USE; error budgets enforced as deploy gates per service |
| **Grafana Labs / Weaveworks** | Tom Wilkie's home turf — RED is the default in Mimir/Cortex dashboards |
| **Datadog / New Relic / Honeycomb** | All offer pre-canned RED, USE, and Golden Signals dashboards on autodetected services |
| **Kubernetes** | `kubelet` exposes USE metrics; pod-level RED is added by service meshes (Linkerd, Istio) without instrumentation changes |
| **Nginx / Envoy** | Built-in counters and histograms map directly to Golden Signals at the edge |

## Anti-Patterns

- **Alerting on causes that may not produce symptoms.** "CPU > 90%" with no user impact wakes you up for nothing. Move to a dashboard.
- **Threshold-only alerts on raw error rates.** Either too noisy (5-minute window) or too slow (1-hour window). Use multi-window multi-burn-rate.
- **No error budget, or a budget with no consequence.** A budget that the team blows past month after month without a feature freeze is theatre.
- **Averages instead of percentiles.** A 99.9% SLO measured against an average latency does not mean what you think. Use histograms.
- **Aggregating percentiles directly.** `avg(p99)` across instances is wrong. Aggregate histograms, then take percentiles.
- **One alert per metric.** Collapse correlated alerts into one symptom alert; route diagnostic detail through the runbook.
- **Symptom-only with no leading indicators.** Pure symptom alerting catches outages but misses capacity creep. Pair with low-urgency USE-based tickets.
- **Different SLI between dashboard and alert.** If the dashboard shows requests-per-second on `status=200|201|204` and the alert fires on `status!~"5.."`, you will fight every incident debate over which is "right."
- **Excluding too much from the SLI.** "Bot traffic excluded" can hide real degradation. Be conservative about exclusions; document each one.
- **Pages with no runbook.** If you cannot point to a step-by-step page that resolved the issue last time, the alert is incomplete.
- **Notifying everyone on every alert.** Use escalation chains. Page primary, then secondary, then manager — not all three at once.
- **Alert rules in code review purgatory.** Treat alerts as code, but make the path to add or modify them smooth. Bureaucracy here breeds duct-tape ad-hoc alerts elsewhere.
- **No quarterly alert audit.** Alerts rot. Review fire frequency, action-required ratio, and runbook accuracy at least quarterly.

## Related

- [Tracing, Metrics, and Logs — The Three Pillars](tracing-metrics-logs.md) _(planned)_ — where these monitoring methods sit in the broader observability stack; OpenTelemetry, exemplars, log/metric correlation
- [Performance Budgets and Latency](performance-budgets-and-latency.md) _(planned)_ — how latency SLOs feed performance budgets and capacity planning
- [Dashboards, Runbooks, and On-Call](dashboards-runbooks-on-call.md) _(planned)_ — turning monitoring data into operable rotations; runbook structure; postmortems
- [SLA, SLO, SLI, and Availability](../foundations/sla-slo-sli-and-availability.md) — definitions, measurement windows, and how SLOs become the input to everything in this doc

## References

- Beyer, Jones, Petoff, Murphy (eds.), *Site Reliability Engineering* — chapter 6, ["Monitoring Distributed Systems"](https://sre.google/sre-book/monitoring-distributed-systems/) (Google, 2016) — original Four Golden Signals
- Beyer, Murphy, Rensin, Kawahara, Thorne (eds.), *The Site Reliability Workbook* — chapter 5, ["Alerting on SLOs"](https://sre.google/workbook/alerting-on-slos/) (Google, 2018) — multi-window multi-burn-rate pattern, with the canonical burn-rate table
- Tom Wilkie, ["The RED Method: How To Instrument Your Services"](https://thenewstack.io/monitoring-microservices-red-method/) (Weaveworks / The New Stack, 2018) and the [conference talk on YouTube](https://www.youtube.com/watch?v=TJLpYXbnfQ4)
- Brendan Gregg, ["The USE Method"](https://www.brendangregg.com/usemethod.html) — canonical reference page with per-resource checklists for Linux and other OSes
- Rob Ewaschuk, ["My Philosophy on Alerting"](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q/edit) (Google, 2014) — symptom-vs-cause alerting and the philosophy behind it
- Björn Rabenstein and Tom Wilkie, ["Anatomy of a Production-Ready Service"](https://www.youtube.com/watch?v=h4Sl21AKiDg) — RED in production with Prometheus, by the Prometheus and Cortex maintainers
- [Prometheus Alerting Rules — `histogram_quantile` and recording rules](https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile) — official docs for the percentile and rate functions used above
- [Google Cloud SLO Monitoring docs](https://cloud.google.com/stackdriver/docs/solutions/slo-monitoring) — production implementation of multi-window multi-burn-rate as a managed service
- Charity Majors, Liz Fong-Jones, George Miranda, *Observability Engineering* (O'Reilly, 2022) — broader treatment of the "metrics + traces + logs" stack that surrounds these methods
