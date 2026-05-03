---
title: "SLI / SLO / Error Budgets — Measuring What Users Care About"
date: 2026-05-03
updated: 2026-05-03
tags: [observability, sre, slo, sli, error-budget, alerting, burn-rate]
---

# SLI / SLO / Error Budgets — Measuring What Users Care About

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `observability` `sre` `slo` `sli` `error-budget` `alerting` `burn-rate`

---

## Table of Contents

- [Summary](#summary)
- [1. Definitions](#1-definitions)
  - [1.1 SLI — Service Level Indicator](#11-sli--service-level-indicator)
  - [1.2 SLO — Service Level Objective](#12-slo--service-level-objective)
  - [1.3 SLA — Service Level Agreement](#13-sla--service-level-agreement)
  - [1.4 Error Budget](#14-error-budget)
- [2. Choosing SLIs](#2-choosing-slis)
  - [2.1 Categories of SLI](#21-categories-of-sli)
  - [2.2 The Good-Events / Valid-Events Ratio](#22-the-good-events--valid-events-ratio)
  - [2.3 Avoiding Bad SLIs](#23-avoiding-bad-slis)
- [3. Choosing SLO Targets](#3-choosing-slo-targets)
  - [3.1 The Cost of Each Nine](#31-the-cost-of-each-nine)
  - [3.2 Time-Window Choices](#32-time-window-choices)
- [4. Error Budget Math](#4-error-budget-math)
  - [4.1 Budget From an SLO](#41-budget-from-an-slo)
  - [4.2 Budget Consumption Rate (Burn Rate)](#42-budget-consumption-rate-burn-rate)
- [5. Burn-Rate Alerting](#5-burn-rate-alerting)
  - [5.1 Why "Threshold on Error Rate" Is Wrong](#51-why-threshold-on-error-rate-is-wrong)
  - [5.2 Single-Window Burn-Rate Alert](#52-single-window-burn-rate-alert)
  - [5.3 Multi-Window, Multi-Burn-Rate (MWMBR)](#53-multi-window-multi-burn-rate-mwmbr)
- [6. Worked Example: Checkout API](#6-worked-example-checkout-api)
  - [6.1 SLIs](#61-slis)
  - [6.2 SLOs](#62-slos)
  - [6.3 PromQL for Availability](#63-promql-for-availability)
  - [6.4 Burn-Rate Alert Rules](#64-burn-rate-alert-rules)
- [7. Operating With Error Budgets](#7-operating-with-error-budgets)
- [8. Common Mistakes](#8-common-mistakes)
- [Related](#related)
- [References](#references)

---

## Summary

An SLO turns "the service should be reliable" into a measurable contract: *X% of well-defined good events over a defined window*. It is the most leveraged single concept in SRE because it converts reliability from a vibe into math you can alert on, plan against, and have product conversations about. The Google SRE Book and the follow-up book *The Site Reliability Workbook* (specifically the chapter "Implementing SLOs" by Hidalgo, Lewandowski, et al.) lay out the discipline. This doc covers the definitions, how to choose SLIs that actually correlate with user happiness, the math behind error budgets, and how to alert on burn rate with the multi-window multi-burn-rate (MWMBR) pattern that has become the industry standard.

---

## 1. Definitions

### 1.1 SLI — Service Level Indicator

> A carefully defined quantitative measure of some aspect of the level of service that is provided.
> — *Site Reliability Engineering*, Chapter 4

Examples: request success ratio, p99 latency, replication lag in seconds, end-to-end durability count.

An SLI is **a number** that goes up or down over time. It is not a Slack alert, it is not a dashboard, and it is not a feeling.

### 1.2 SLO — Service Level Objective

> A target value or range of values for a service level, measured by an SLI.

Form: `SLI ≥ target over rolling window` (or `≤` for things like latency).

Example: `availability_30d ≥ 99.9%` where `availability = good_requests / valid_requests`.

### 1.3 SLA — Service Level Agreement

A **contract** with consequences (refunds, credits, exit clauses). SLAs are commercial; SLOs are engineering. The SLA target is virtually always *looser* than the internal SLO so engineers have a margin before the lawyers get involved.

### 1.4 Error Budget

The complement of the SLO target over a window:

```
error_budget = (1 - SLO) × total_valid_events_in_window
```

For 99.9% over 30 days with 100 M requests:

```
error_budget = 0.001 × 100_000_000 = 100_000 errors
```

Spent budget is the sum of bad events seen so far in the window. Remaining budget is what you have left before breaching the SLO.

The error budget reframes reliability as a **finite resource**. You do not strive to be infinitely reliable — you strive to spend the budget on the right things (deploys, experiments, partial outages from migrations) and not on accidents.

---

## 2. Choosing SLIs

### 2.1 Categories of SLI

The Google SRE Book identifies four families of user-facing SLI:

| Category | Question it answers | Typical SLI |
|----------|--------------------|-------------|
| **Availability** | "Did the request succeed?" | `successful_requests / valid_requests` |
| **Latency** | "Was it fast enough?" | `requests faster than X / valid_requests` |
| **Throughput** / quality | "Did we serve enough?" | bytes/sec, items processed/sec |
| **Correctness** / freshness | "Was the answer right and current?" | data freshness lag, replication lag, fraction of correct results |

For data pipelines, replace availability/latency with **freshness, completeness, correctness, and durability** (per *Implementing SLOs*).

### 2.2 The Good-Events / Valid-Events Ratio

The cleanest SLI shape is a ratio of:

- **good events** — events that met the bar (status 2xx, latency below threshold, replica age below limit)
- **valid events** — events that we should count (excludes irrelevant traffic, scanner bots, health probes)

```
sli = good_events / valid_events
```

Why this shape:

- It is dimensionless (a ratio in [0, 1]).
- It composes cleanly across services if "valid" is defined consistently.
- It is what burn-rate alerting math expects.

### 2.3 Avoiding Bad SLIs

Symptoms vs causes — **always pick the symptom**:

| Bad (cause) | Better (symptom) |
|-------------|-------------------|
| CPU utilization on web tier | Request error rate, request latency |
| Pod restart count | Successful health checks served |
| Queue depth | Time to process a message end-to-end |
| Replication lag in bytes | Data freshness as observed by a downstream reader |

CPU at 90% does not bother any user; HTTP 500s do. Use cause-level signals on **dashboards**, not SLOs.

Other failure modes:

- **Picking metrics you already have rather than ones that matter** — vanity SLOs.
- **SLI that ignores the user's perspective** — measuring at the load balancer when the actual breakage is in the JS bundle that 3% of users get a stale CDN edge of.
- **SLI tied to a single host** — masks fleet-wide degradation.

---

## 3. Choosing SLO Targets

### 3.1 The Cost of Each Nine

| SLO target | Allowed downtime per 30 days | Allowed downtime per year |
|------------|------------------------------|---------------------------|
| 99% | 7.2 hours | 3.65 days |
| 99.5% | 3.6 hours | 1.83 days |
| 99.9% | 43.2 minutes | 8.76 hours |
| 99.95% | 21.6 minutes | 4.38 hours |
| 99.99% | 4.32 minutes | 52.6 minutes |
| 99.999% | 25.9 seconds | 5.26 minutes |

Each additional nine roughly **10× the cost** of the previous one in engineering effort, redundancy, and operational overhead. *Implementing SLOs* makes the case bluntly: most internal services do not need more than 99.9%, and many can run at 99.5% if downstream consumers tolerate it.

A useful constraint: your SLO should be **slightly worse than what your dependencies offer** because your availability is the product of theirs. If three downstreams each offer 99.9%, the best naive composition is 99.7% — and that is before any of your own bugs.

### 3.2 Time-Window Choices

**Rolling vs calendar:**

- **Rolling 30 days** — most common. Always reflects recent reality. Good for paging.
- **Calendar quarter** — used by some product/business teams. Resets on a fixed boundary; can lead to "we have plenty of budget left because the calendar reset" gaming.

Pick rolling for engineering-facing SLOs. Pick calendar windows only when there is a contractual reason.

---

## 4. Error Budget Math

### 4.1 Budget From an SLO

```
error_budget_fraction = 1 - SLO_target
```

For a 99.9% SLO, you may emit up to 0.1% bad events per window. Over 100 M requests in a 30-day window, that is 100,000 bad requests. Over 1,000 requests it is 1 bad request — small services are sample-poor and need careful interpretation.

### 4.2 Budget Consumption Rate (Burn Rate)

**Burn rate** = how fast you are using your budget, normalized to the rate that would exhaust the entire budget exactly at the end of the window.

```
burn_rate = current_error_rate / error_budget_fraction
```

For a 99.9% SLO (budget = 0.001):

| Current error rate | Burn rate | Time to burn full 30-day budget at this rate |
|--------------------|-----------|-----------------------------------------------|
| 0.001 (0.1%) | 1× | 30 days |
| 0.01 (1%) | 10× | 3 days |
| 0.05 (5%) | 50× | 14.4 hours |
| 0.14 (14%) | 144× | 5 minutes |

Burn rate is the better metric for alerting because it is **scale-invariant** — the same threshold works at 100 RPS and 100k RPS.

---

## 5. Burn-Rate Alerting

### 5.1 Why "Threshold on Error Rate" Is Wrong

A naive alert "page if error rate > 1% for 5 minutes":

- Pages on transient blips that consume an irrelevant slice of budget.
- Misses slow burns that, over a day, blow the entire budget.
- Cannot be tuned to be both sensitive and quiet.

### 5.2 Single-Window Burn-Rate Alert

Better: alert when the burn rate over a chosen window exceeds a chosen threshold. Choose the window such that, at that burn rate, you would **consume X% of the monthly budget** before the window ends. The classic table from *The Site Reliability Workbook*:

| Detection target | Burn rate threshold | Window length | Budget consumed at firing |
|------------------|---------------------|---------------|----------------------------|
| Catastrophic | 14.4× | 1 hour | 2% of 30-day budget |
| Severe | 6× | 6 hours | 5% |
| Slow | 3× | 1 day | 10% |
| Slow background | 1× | 3 days | 10% |

### 5.3 Multi-Window, Multi-Burn-Rate (MWMBR)

Single windows still false-page on transient noise. The solution from *Implementing SLOs* (Chapter 5 of *The Site Reliability Workbook*) and Google's `sre/sre-mwmbr` patterns: **fire only when both a long window and a short window confirm the burn**. The short window keeps the alert responsive; the long window avoids paging on a 30-second blip.

A standard config (page-class):

| Severity | Long window / threshold | Short window / threshold |
|----------|-------------------------|--------------------------|
| **Page (critical)** | `burn_rate(1h) > 14.4` | `burn_rate(5m) > 14.4` |
| **Page (medium)** | `burn_rate(6h) > 6` | `burn_rate(30m) > 6` |
| **Ticket** | `burn_rate(24h) > 3` | `burn_rate(2h) > 3` |
| **Ticket** | `burn_rate(72h) > 1` | `burn_rate(6h) > 1` |

Both conditions must be true simultaneously.

This pattern is now the default in tools like Sloth, Pyrra, OpenSLO, Grafana SLO, Datadog SLO Burn-Rate alerts, and Google Cloud SLO alerting.

---

## 6. Worked Example: Checkout API

### 6.1 SLIs

- **Availability SLI**: `valid` = all `POST /orders` requests excluding 4xx-due-to-client (400, 401, 403, 404, 422) and excluding health checks. `good` = same set with HTTP status `< 500`.
- **Latency SLI**: of `valid` requests, fraction completing within 500 ms.

### 6.2 SLOs

- Availability: 99.9% over rolling 30 days.
- Latency: 95% of valid requests complete within 500 ms over rolling 30 days.

Error budgets:

- Availability: 0.1% × ~100 M requests / month ≈ 100,000 5xx allowed.
- Latency: 5% × 100 M = 5 M slow requests allowed.

### 6.3 PromQL for Availability

Assuming a histogram counter `http_server_requests_seconds_count{route,status}`:

```promql
# total valid (exclude client errors and health checks)
sum(rate(
  http_server_requests_seconds_count{
    service="checkout",
    route="/orders",
    status!~"^(4..)$",
    method="POST"
  }[5m]
))

# good (server-side success: 2xx, 3xx)
sum(rate(
  http_server_requests_seconds_count{
    service="checkout",
    route="/orders",
    status=~"^(2..|3..)$",
    method="POST"
  }[5m]
))
```

Recording rules make burn-rate calculations cheap:

```yaml
groups:
- name: checkout-slo
  interval: 30s
  rules:
  - record: slo:checkout_orders_requests_total:rate5m
    expr: sum(rate(http_server_requests_seconds_count{
            service="checkout", route="/orders", method="POST",
            status!~"^(4..)$"}[5m]))

  - record: slo:checkout_orders_errors_total:rate5m
    expr: sum(rate(http_server_requests_seconds_count{
            service="checkout", route="/orders", method="POST",
            status=~"^5.."}[5m]))

  - record: slo:checkout_orders_error_rate:5m
    expr: slo:checkout_orders_errors_total:rate5m
          / slo:checkout_orders_requests_total:rate5m
```

Define the same recording rules for `1h`, `6h`, `1d`, `3d` windows.

### 6.4 Burn-Rate Alert Rules

```yaml
groups:
- name: checkout-slo-burn
  rules:
  # Critical: 2% of monthly budget burned in 1h
  - alert: CheckoutSLOFastBurn
    expr: |
      slo:checkout_orders_error_rate:1h > (14.4 * 0.001)
      and
      slo:checkout_orders_error_rate:5m > (14.4 * 0.001)
    for: 2m
    labels:
      severity: page
      slo: checkout-availability
    annotations:
      summary: "Checkout availability SLO burning at 14.4× (fast burn)"
      runbook: "https://runbooks.example.com/checkout-availability"

  # Medium: 5% of monthly budget over 6h
  - alert: CheckoutSLOMediumBurn
    expr: |
      slo:checkout_orders_error_rate:6h > (6 * 0.001)
      and
      slo:checkout_orders_error_rate:30m > (6 * 0.001)
    for: 15m
    labels:
      severity: page
      slo: checkout-availability

  # Slow ticket-class burns
  - alert: CheckoutSLOSlowBurn
    expr: |
      slo:checkout_orders_error_rate:1d > (3 * 0.001)
      and
      slo:checkout_orders_error_rate:2h > (3 * 0.001)
    for: 1h
    labels:
      severity: ticket
      slo: checkout-availability
```

The `0.001` is `1 - SLO = 1 - 0.999`. Replace per SLO target.

---

## 7. Operating With Error Budgets

The error budget is the lever for product / engineering negotiations:

- **Budget remaining** → ship features, do experiments, take controlled risk.
- **Budget exhausted** → freeze risky changes, focus on reliability, do postmortems.
- **Budget consistently unspent** → SLO is too loose, or the team is over-investing in reliability at the cost of features.

This last case is the main argument *against* the "more nines is always better" instinct. If you finish each month with 80% of the budget unspent, you are effectively taxing feature velocity for no user benefit — tighten the SLO or accept faster experimentation.

Postmortems should reference **how much of the budget the incident consumed**. That converts "the incident felt bad" into "the incident burned 60% of our quarterly budget."

---

## 8. Common Mistakes

- **Cause-level SLOs** ("CPU < 80%"). Page on user-facing symptoms, not infrastructure causes.
- **One SLO covering many product flows.** Checkout availability and search availability deserve separate SLOs.
- **Excluding too much from "valid".** If you exclude every cohort with errors you make the SLO trivial.
- **Treating SLAs as SLOs.** SLAs are looser; using SLA values as engineering targets means you are paging only when contracts are about to be breached — too late.
- **Latency SLO based on average.** Use a percentile (p95, p99) on a histogram. Averages hide tails.
- **No burn-rate alerts.** Threshold-on-error-rate alerts either page on noise or miss slow burns.
- **SLOs that nobody owns.** Every SLO must have a service owner who is accountable for it.

---

## Related

- [Three Pillars: Metrics, Logs, Traces](01-three-pillars-metrics-logs-traces.md)
- [RED, USE, and Four Golden Signals](04-red-and-use-methods.md)
- [Prometheus and PromQL](07-prometheus-and-promql.md)
- [Spring Boot Actuator Deep Dive](../../java/actuator-deep-dive.md)
- [System Design: Dashboards, Runbooks, On-Call](../../system-design/performance-observability/dashboards-runbooks-on-call.md)

---

## References

- **Beyer, Jones, Petoff, Murphy (eds.) — *Site Reliability Engineering*** (O'Reilly, 2016). Chapters 3 ("Embracing Risk") and 4 ("Service Level Objectives"). https://sre.google/sre-book/service-level-objectives/
- **Beyer, Murphy, Rensin, Kawahara, Thorne (eds.) — *The Site Reliability Workbook*** (O'Reilly, 2018). Chapter 2 ("Implementing SLOs") and Chapter 5 ("Alerting on SLOs"). https://sre.google/workbook/implementing-slos/
- **Alex Hidalgo — *Implementing Service Level Objectives*** (O'Reilly, 2020). Definitive operational treatment.
- **Google SRE — "Alerting on SLOs"** (workbook chapter). The MWMBR table originates here. https://sre.google/workbook/alerting-on-slos/
- **OpenSLO specification.** https://github.com/OpenSLO/OpenSLO
- **Sloth — Prometheus SLO generator.** https://sloth.dev/
- **Pyrra — open-source SLO controller.** https://github.com/pyrra-dev/pyrra
- **Datadog — "SLOs and burn rate alerts."** https://docs.datadoghq.com/service_management/service_level_objectives/burn_rate/
- **Google Cloud — "Alerting on SLO burn rate."** https://cloud.google.com/architecture/sre/sre-fundamentals
