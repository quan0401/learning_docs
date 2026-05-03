---
title: "Prometheus and PromQL — Pull Model, Histograms, and Alert Rules"
date: 2026-05-03
updated: 2026-05-03
tags: [observability, prometheus, promql, metrics, histogram, alerting, recording-rules]
---

# Prometheus and PromQL — Pull Model, Histograms, and Alert Rules

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `observability` `prometheus` `promql` `metrics` `histogram` `alerting` `recording-rules`

---

## Table of Contents

- [Summary](#summary)
- [1. Architecture and the Pull Model](#1-architecture-and-the-pull-model)
  - [1.1 Components](#11-components)
  - [1.2 Pull vs Push](#12-pull-vs-push)
  - [1.3 Service Discovery](#13-service-discovery)
- [2. Scrape Configuration](#2-scrape-configuration)
- [3. Exposition Format](#3-exposition-format)
- [4. Metric Types](#4-metric-types)
  - [4.1 Counter](#41-counter)
  - [4.2 Gauge](#42-gauge)
  - [4.3 Histogram](#43-histogram)
  - [4.4 Summary](#44-summary)
  - [4.5 Native Histograms (Sparse)](#45-native-histograms-sparse)
- [5. Histogram Buckets and Quantile Estimation](#5-histogram-buckets-and-quantile-estimation)
  - [5.1 Choosing Buckets](#51-choosing-buckets)
  - [5.2 How histogram\_quantile Works](#52-how-histogram_quantile-works)
  - [5.3 Quantile Aggregation Pitfalls](#53-quantile-aggregation-pitfalls)
- [6. PromQL Essentials](#6-promql-essentials)
  - [6.1 Selectors](#61-selectors)
  - [6.2 Range Vectors and rate / irate / increase](#62-range-vectors-and-rate--irate--increase)
  - [6.3 Aggregation](#63-aggregation)
  - [6.4 Vector Matching](#64-vector-matching)
  - [6.5 Common PromQL Patterns](#65-common-promql-patterns)
- [7. Recording Rules](#7-recording-rules)
- [8. Alerting Rules and Alertmanager](#8-alerting-rules-and-alertmanager)
- [9. Federation and Remote Write](#9-federation-and-remote-write)
- [10. Common Mistakes](#10-common-mistakes)
- [Related](#related)
- [References](#references)

---

## Summary

Prometheus is the de facto open-source metrics system: a single-process server that scrapes HTTP endpoints, stores time series in a custom TSDB, and serves a query language (PromQL) over them. Its design choices — pull-based collection, dimensional labels, histogram-as-cumulative-buckets, and a well-defined exposition format — set the shape of the entire metrics ecosystem; OpenTelemetry, Mimir, Thanos, Cortex, VictoriaMetrics, and Datadog/New Relic all read or interoperate with it. This doc covers the architecture, the metric types, how histograms actually work (and how `histogram_quantile` lies subtly), the PromQL functions you reach for daily, and how to write recording and alerting rules that hold up.

---

## 1. Architecture and the Pull Model

### 1.1 Components

```
┌──────────────┐      scrape /metrics     ┌──────────────────┐
│  service A   │◀─────────────────────────│                  │
│ /metrics     │                          │   Prometheus     │
└──────────────┘                          │   server         │
┌──────────────┐                          │                  │
│  service B   │◀─────────────────────────│  ┌──────────┐    │   PromQL queries
│ /metrics     │                          │  │   TSDB   │    │◀──────────────────  Grafana / API
└──────────────┘                          │  └──────────┘    │
┌──────────────┐                          │  ┌──────────┐    │
│ node_exporter│◀─────────────────────────│  │  rules   │    │
└──────────────┘                          │  └──────────┘    │
                                          └────────┬─────────┘
                                                   │ alert notifications
                                                   ▼
                                            ┌─────────────┐
                                            │Alertmanager │ → PagerDuty, Slack, email
                                            └─────────────┘
```

### 1.2 Pull vs Push

Prometheus *pulls* metrics from targets it discovers. The arguments for pull (per Prometheus docs):

- **Health check side-effect**: if the scrape fails, the target is implicitly down.
- **No per-target push configuration**: targets don't need to know the metrics infra.
- **Easier multi-tenant**: any operator can spin up a Prometheus to scrape the same targets.
- **Backpressure handled by scrape interval**: targets aren't responsible for buffering when the metrics system is overloaded.

For workloads pull doesn't fit (batch jobs, AWS Lambda, ephemeral CLIs), the **Pushgateway** accepts pushes and exposes them for Prometheus to scrape. Use it sparingly — it is not a general-purpose ingest endpoint.

### 1.3 Service Discovery

Static targets are fine for a homelab. In production, use service discovery:

- `kubernetes_sd_configs` — the standard for K8s. Discovers pods, services, endpoints, ingresses.
- `ec2_sd_configs`, `gce_sd_configs`, `azure_sd_configs`.
- `consul_sd_configs`, `dns_sd_configs`.
- `file_sd_configs` — JSON file watched for changes; useful for ad-hoc setups.

---

## 2. Scrape Configuration

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
  external_labels:
    cluster: prod-us-east-1

scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with a "prometheus.io/scrape: true" annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels:
          - __address__
          - __meta_kubernetes_pod_annotation_prometheus_io_port
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: service
```

`relabel_configs` runs before the scrape (filtering targets, rewriting `__address__` and `__metrics_path__`). `metric_relabel_configs` runs on every scraped sample (filtering, label dropping). Both are powerful and easy to misuse — keep them small and readable.

---

## 3. Exposition Format

Prometheus targets expose metrics over HTTP at `/metrics` in a simple text format (OpenMetrics-compatible):

```
# HELP http_server_request_duration_seconds Duration of HTTP server requests
# TYPE http_server_request_duration_seconds histogram
http_server_request_duration_seconds_bucket{service="checkout",route="/orders",method="POST",status="200",le="0.005"} 0
http_server_request_duration_seconds_bucket{service="checkout",route="/orders",method="POST",status="200",le="0.01"} 14
http_server_request_duration_seconds_bucket{service="checkout",route="/orders",method="POST",status="200",le="0.025"} 138
http_server_request_duration_seconds_bucket{service="checkout",route="/orders",method="POST",status="200",le="0.05"} 1421
http_server_request_duration_seconds_bucket{service="checkout",route="/orders",method="POST",status="200",le="0.1"} 14823
http_server_request_duration_seconds_bucket{service="checkout",route="/orders",method="POST",status="200",le="+Inf"} 14999
http_server_request_duration_seconds_sum{service="checkout",route="/orders",method="POST",status="200"} 412.7
http_server_request_duration_seconds_count{service="checkout",route="/orders",method="POST",status="200"} 14999

# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 8721.43
```

Counters end in `_total` by convention. Histograms expose three series families: `_bucket{le=...}`, `_sum`, `_count`.

**OpenMetrics** is the standardized successor (RFC-style spec, exemplar support, native histograms). Prometheus parses both.

---

## 4. Metric Types

### 4.1 Counter

Monotonically non-decreasing. Resets to 0 on process restart.

```
http_requests_total{status="200"}  124581
```

Always query through `rate()` / `increase()` — raw counter values are meaningless.

### 4.2 Gauge

Point-in-time value. Goes up and down.

```
process_resident_memory_bytes  168432128
queue_depth{queue="orders"}    742
```

Query directly, or with `delta()` / `deriv()` for change.

### 4.3 Histogram

Pre-bucketed cumulative distribution. Each bucket records "count of observations ≤ `le`":

```
http_request_duration_seconds_bucket{le="0.1"}  14823   # ≤ 100ms
http_request_duration_seconds_bucket{le="0.25"} 14950   # ≤ 250ms
http_request_duration_seconds_bucket{le="+Inf"} 14999   # all
http_request_duration_seconds_sum               412.7   # sum of all observations
http_request_duration_seconds_count             14999   # total observations
```

Quantiles are computed with `histogram_quantile()` at query time — server-side aggregation across instances is mathematically valid (just sum the bucket counts).

### 4.4 Summary

Computes quantiles **client-side**, exposes them as labeled series:

```
rpc_duration_seconds{quantile="0.5"}  0.034
rpc_duration_seconds{quantile="0.9"}  0.082
rpc_duration_seconds{quantile="0.99"} 0.241
rpc_duration_seconds_sum  84211.4
rpc_duration_seconds_count 1000000
```

Pros: cheap to compute over a sliding window per process.

Cons: **summaries cannot be aggregated**. You cannot take the p99 across 100 instances by averaging or summing per-instance quantiles. For multi-instance services, prefer histograms.

### 4.5 Native Histograms (Sparse)

Native histograms are a Prometheus 2.40+ format: a single series per histogram with **adaptive, exponential bucket boundaries** plus a count and sum. They give bounded-error quantiles with far less storage than fixed-bucket histograms, and aggregate cleanly. OpenTelemetry's exponential histograms map to this format.

If your Prometheus and Grafana are recent enough, prefer native histograms for new instrumentation. Classic histograms remain the default in many libraries; check support.

---

## 5. Histogram Buckets and Quantile Estimation

### 5.1 Choosing Buckets

The accuracy of `histogram_quantile()` is bounded by your bucket boundaries. For an HTTP latency histogram, a typical boundary set:

```
0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10
```

Heuristics:

- **Cover the range you care about.** If your SLO is "p95 < 500 ms", you need granularity around 500 ms.
- **Use roughly geometric spacing.** Doubling or 2.5× spacing balances granularity vs series count.
- **Don't add 50 buckets "just in case".** Each bucket × each label combination = a separate time series.

OTel's default HTTP latency boundaries are reasonable; tune only when you have evidence.

### 5.2 How histogram\_quantile Works

`histogram_quantile(0.99, sum by (le)(rate(http_request_duration_seconds_bucket[5m])))` does:

1. Compute per-second rate per `le` bucket → counts of observations falling in each cumulative bucket per second.
2. Sum across instances/labels (still grouped by `le`) — valid because cumulative bucket counts are additive.
3. For the target quantile `q = 0.99`:
   - Find the bucket where cumulative count first crosses `q × total_count`.
   - **Linearly interpolate** within that bucket between its `le` and the previous bucket's `le`.

The interpolation is the source of error: if the true 99th percentile lies in a bucket spanning 1s–2.5s, `histogram_quantile` reports something between 1 and 2.5, with no further information. Buckets must be tight where you care.

### 5.3 Quantile Aggregation Pitfalls

- **Never average quantile values.** `(p99_a + p99_b) / 2 ≠ p99(a ∪ b)`. Use the histogram and aggregate buckets first.
- **Never sum summaries.** Summary quantiles are pre-computed and not aggregable.
- **Always group by `le`** in the inner aggregation; otherwise `histogram_quantile` has nothing to interpolate over.

---

## 6. PromQL Essentials

### 6.1 Selectors

```
http_requests_total                                 # all series
http_requests_total{service="checkout"}             # equality
http_requests_total{service!="checkout"}            # inequality
http_requests_total{service=~"checkout|payments"}   # regex match
http_requests_total{service!~"healthcheck.*"}       # negative regex
```

Returns an **instant vector**: one sample per matching series at the query time.

### 6.2 Range Vectors and rate / irate / increase

A range vector adds `[duration]` and returns a window of samples:

```
http_requests_total{service="checkout"}[5m]
```

You cannot graph a range vector directly; you apply functions to it.

| Function | Behavior |
|----------|----------|
| `rate(v[5m])` | Per-second average rate of increase, over the window. Smooths spikes. **Use for graphs and alerts.** |
| `irate(v[5m])` | Instantaneous rate using the last two samples. Spikier; best for high-resolution dashboards. |
| `increase(v[5m])` | Total counter delta over the window (= `rate * window_seconds`). |

`rate` handles counter resets (process restarts) by treating decreases as "reset, then continue counting from 0".

### 6.3 Aggregation

```promql
sum(rate(http_requests_total[5m]))                            # global sum
sum by (route)(rate(http_requests_total[5m]))                 # group by route
sum without (instance)(rate(http_requests_total[5m]))         # collapse instance label
avg by (route)(rate(http_requests_total[5m]))
max by (route)(...)
min, count, count_values, stddev, stdvar
topk(5, sum by (route)(rate(http_requests_total[5m])))        # top 5 routes by rate
bottomk(5, ...)
quantile(0.95, ...)                                            # quantile across series (NOT histogram_quantile)
```

### 6.4 Vector Matching

Binary operators between two vectors require matching labels. By default, all labels match; use `on(...)` and `ignoring(...)` to control:

```promql
# error rate as a fraction of total requests, per route
sum by (route)(rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (route)(rate(http_requests_total[5m]))
```

For one-to-many matches use `group_left` / `group_right`:

```promql
# join latency p99 with replica count per service
histogram_quantile(0.99, sum by (le, service)(rate(http_request_duration_seconds_bucket[5m])))
* on (service) group_left() service_replicas
```

### 6.5 Common PromQL Patterns

```promql
# RED: rate
sum by (service)(rate(http_server_request_duration_seconds_count[5m]))

# RED: error fraction
sum by (service)(rate(http_server_request_duration_seconds_count{status=~"5.."}[5m]))
/
sum by (service)(rate(http_server_request_duration_seconds_count[5m]))

# RED: p99 latency
histogram_quantile(
  0.99,
  sum by (le, service)(rate(http_server_request_duration_seconds_bucket[5m]))
)

# Apdex-like (success ratio meeting target)
sum(rate(http_server_request_duration_seconds_bucket{le="0.5"}[5m]))
/
sum(rate(http_server_request_duration_seconds_count[5m]))

# Kubernetes pod CPU saturation (cgroup throttling fraction)
sum by (pod)(rate(container_cpu_cfs_throttled_seconds_total[5m]))
/
sum by (pod)(rate(container_cpu_cfs_periods_total[5m]))

# Memory headroom toward OOM
1 - max by (pod)(container_memory_working_set_bytes
                 / on (pod, container) group_left
                 kube_pod_container_resource_limits{resource="memory"})

# Predict when disk fills (deriv = derivative; predict_linear extrapolates)
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 4*3600) < 0
```

---

## 7. Recording Rules

Recording rules pre-compute expensive queries on a schedule and store the result as a new time series. Two reasons:

1. **Speed** — dashboards loading in 200 ms vs 8 s.
2. **SLO math reuse** — once you've defined "good rate over 5m", every burn-rate alert reuses the same series.

```yaml
groups:
- name: checkout-red
  interval: 30s
  rules:
  - record: red:checkout_orders_request_rate:5m
    expr: |
      sum by (service, route, status) (
        rate(http_server_request_duration_seconds_count{
          service="checkout"}[5m])
      )

  - record: red:checkout_orders_p99_latency:5m
    expr: |
      histogram_quantile(0.99,
        sum by (le, route)(
          rate(http_server_request_duration_seconds_bucket{
            service="checkout"}[5m])
        )
      )
```

Naming convention (Prometheus docs): `level:metric:operations` — e.g., `instance:node_cpu:rate5m`. The `:` is permitted in metric names specifically for recording-rule output; production exporters should not use it.

---

## 8. Alerting Rules and Alertmanager

```yaml
groups:
- name: checkout-alerts
  rules:
  - alert: CheckoutErrorRateHigh
    expr: |
      sum(rate(http_server_request_duration_seconds_count{service="checkout",status=~"5.."}[5m]))
      / sum(rate(http_server_request_duration_seconds_count{service="checkout"}[5m]))
      > 0.02
    for: 10m
    labels:
      severity: page
      team: payments
    annotations:
      summary: "Checkout 5xx error rate above 2% for 10m"
      description: "Current error rate: {{ $value | humanizePercentage }}"
      runbook_url: "https://runbooks.example.com/checkout-error-rate"
```

Key fields:

- `expr` — must return an instant vector. Each sample becomes an alert instance.
- `for` — alert must be `Pending` for this duration before firing. Suppresses transient blips.
- `labels` — added to the alert; used for routing in Alertmanager.
- `annotations` — human-facing detail; templated with the alert's labels and `$value`.

**Alertmanager** handles routing, grouping, deduplication, inhibition, and silencing. Group alerts by team/service so a 50-pod outage doesn't 50-page on call.

For SLO burn-rate alerts (the better default), see [SLI / SLO / Error Budgets §5–6](03-sli-slo-error-budgets.md#5-burn-rate-alerting).

---

## 9. Federation and Remote Write

A single Prometheus instance scales to ~1–10 M active series before tuning gets uncomfortable. Two patterns for going further:

**Federation** (hierarchical Prometheus):

```yaml
# Top-level Prometheus pulls aggregated series from per-cluster Prometheuses
scrape_configs:
  - job_name: federate
    scrape_interval: 60s
    honor_labels: true
    metrics_path: /federate
    params:
      'match[]':
        - '{__name__=~"job:.*"}'           # only recording-rule outputs
        - '{__name__=~"slo:.*"}'
    static_configs:
      - targets: ['prom-us-east:9090', 'prom-eu-west:9090']
```

Federation is for **recording-rule output**, not raw data. Don't try to federate every series.

**Remote write** to a long-term store (Mimir, Thanos receive, VictoriaMetrics, Cortex):

```yaml
remote_write:
  - url: http://mimir:9009/api/v1/push
    queue_config:
      capacity: 10000
      max_samples_per_send: 2000
      batch_send_deadline: 5s
    write_relabel_configs:
      # Drop debug-only series before shipping
      - source_labels: [__name__]
        regex: 'debug_.*'
        action: drop
```

Remote write is the standard path for global aggregation, multi-cluster dashboards, and 13+ months of retention. Mimir/Thanos/VictoriaMetrics provide a horizontally scalable PromQL backend on top.

The OTel Collector's `prometheusremotewrite` exporter speaks the same protocol — you can ship OTel metrics into Prometheus-compatible long-term storage without running Prometheus itself.

---

## 10. Common Mistakes

- **High-cardinality labels.** `user_id`, `request_id`, full URL paths. See [Three Pillars §2.3](01-three-pillars-metrics-logs-traces.md#23-cardinality-bluntly).
- **Querying counters directly without `rate()`.** Returns the lifetime total, not a rate. Always wrap counters.
- **Using `irate` on `for:` alerts.** `irate` is too spiky for stable alerts; use `rate` over the alert window.
- **Averaging summary quantiles.** Mathematically wrong. Use histograms.
- **`histogram_quantile` without `by (le)`.** Function returns NaN or wrong values.
- **Buckets too sparse around the SLO threshold.** Quantile estimates are coarse where it matters.
- **Recording-rule output names colliding with exporter metric names.** Use the `level:metric:operations` convention to avoid this.
- **Alerts on causes, not symptoms.** Page on user-visible error rate or latency, not on CPU.
- **Forgetting `for:` on alerts.** Every transient flap pages.
- **One Alertmanager route for everything.** Routing-by-team-and-severity is cheap to set up and saves on-call sanity.
- **No remote write, no SLO recording rules, all dashboards rebuild from raw data.** Cheap fix that pays back in ms-saved per dashboard load.

---

## Related

- [Three Pillars: Metrics, Logs, Traces](01-three-pillars-metrics-logs-traces.md)
- [OpenTelemetry Fundamentals](02-opentelemetry-fundamentals.md)
- [SLI / SLO / Error Budgets](03-sli-slo-error-budgets.md)
- [RED, USE, and Four Golden Signals](04-red-and-use-methods.md)
- [Spring Boot Actuator Deep Dive](../../java/actuator-deep-dive.md)
- [Reactive Observability (Micrometer + Reactor)](../../java/reactive-observability.md)
- [Node.js Production Observability](../../typescript/production/observability.md)
- [Kubernetes Monitoring and Logging](../../kubernetes/operations/monitoring-and-logging.md)

---

## References

- **Prometheus — main documentation.** https://prometheus.io/docs/
- **Prometheus — Querying basics, functions, examples.** https://prometheus.io/docs/prometheus/latest/querying/basics/
- **Prometheus — Histograms and summaries.** https://prometheus.io/docs/practices/histograms/
- **Prometheus — Naming conventions and instrumentation.** https://prometheus.io/docs/practices/naming/
- **Prometheus — Recording rules.** https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/
- **Prometheus — Alerting rules.** https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- **Prometheus — Native histograms.** https://prometheus.io/docs/concepts/native_histograms/
- **Alertmanager.** https://prometheus.io/docs/alerting/latest/alertmanager/
- **OpenMetrics specification.** https://github.com/OpenObservability/OpenMetrics
- **Brian Brazil — *Prometheus: Up & Running*** (2nd ed., O'Reilly, 2023). The canonical book by a former Prometheus maintainer.
- **Grafana Mimir — Prometheus-compatible long-term storage.** https://grafana.com/docs/mimir/latest/
- **Thanos.** https://thanos.io/
- **VictoriaMetrics.** https://docs.victoriametrics.com/
