---
title: "Capacity Planning — Headroom, Forecasts, and Failure-Mode Capacity"
date: 2026-05-03
updated: 2026-05-03
tags: [performance, capacity-planning, headroom, autoscaling, n+1, forecasting]
---

# Capacity Planning — Headroom, Forecasts, and Failure-Mode Capacity

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `performance` `capacity-planning` `headroom` `autoscaling` `n+1` `forecasting`

---

## Table of Contents

- [Summary](#summary)
- [1. The Two Questions Capacity Planning Answers](#1-the-two-questions-capacity-planning-answers)
- [2. Back-of-Envelope: From Load to Boxes](#2-back-of-envelope-from-load-to-boxes)
  - [2.1 Throughput-Based Sizing](#21-throughput-based-sizing)
  - [2.2 Resource-Based Sizing](#22-resource-based-sizing)
  - [2.3 Storage and IOPS Sizing](#23-storage-and-iops-sizing)
- [3. Peak-to-Average Ratio](#3-peak-to-average-ratio)
- [4. The Headroom Rule](#4-the-headroom-rule)
  - [4.1 Why 50%](#41-why-50)
  - [4.2 Headroom Across the Stack](#42-headroom-across-the-stack)
  - [4.3 Headroom and Cost](#43-headroom-and-cost)
- [5. Forecasting Growth](#5-forecasting-growth)
  - [5.1 Linear and Exponential Models](#51-linear-and-exponential-models)
  - [5.2 Seasonal Patterns](#52-seasonal-patterns)
  - [5.3 Step-Function Events](#53-step-function-events)
- [6. Scaling Triggers and Autoscaling Lag](#6-scaling-triggers-and-autoscaling-lag)
  - [6.1 What to Scale On](#61-what-to-scale-on)
  - [6.2 Autoscaling Lag in Real Numbers](#62-autoscaling-lag-in-real-numbers)
  - [6.3 Why Static Floors Exist](#63-why-static-floors-exist)
- [7. Failure-Mode Capacity — N+1 and N+2](#7-failure-mode-capacity--n1-and-n2)
  - [7.1 The Concept](#71-the-concept)
  - [7.2 N+1 Math](#72-n1-math)
  - [7.3 Regional Capacity](#73-regional-capacity)
  - [7.4 Cell-Based Architectures](#74-cell-based-architectures)
- [8. Cost vs Reliability Curves](#8-cost-vs-reliability-curves)
- [9. Capacity Review Cadence](#9-capacity-review-cadence)
- [Related](#related)
- [References](#references)

---

## Summary

Capacity planning is the discipline of having enough infrastructure to serve traffic — at peak, with reliability, when something has failed — without bankrupting the company. Done well it is a quiet, repeatable monthly review using load test numbers and forecasts. Done badly it is a firefighting exercise that begins when a launch breaks production. The math is mostly back-of-envelope arithmetic plus a handful of rules that compound: 50% headroom against the latency knee, peak-to-average ratios that capture daily and weekly patterns, autoscaling lag that means the floor matters more than the ceiling, and N+1/N+2 capacity so a single failure does not consume your headroom. This doc covers each rule with concrete numbers, the forecasting methods that work, the autoscaling pitfalls that don't show up in tutorials, and the cost-vs-reliability curve every team eventually has to argue about with finance.

---

## 1. The Two Questions Capacity Planning Answers

Capacity planning exists to answer:

1. **Do I have enough capacity right now to serve peak load with my SLOs intact?**
2. **Do I have a plan to have enough capacity in 30/90/180 days?**

The first is operational — measure current load, current latency under load, identify the constrained resource, decide whether to add capacity. The second is strategic — forecast growth, schedule provisioning lead times, budget cost.

Both questions need a baseline (load test numbers, peak production utilization), a target (SLO + headroom rule), and a cadence (monthly or quarterly review).

---

## 2. Back-of-Envelope: From Load to Boxes

The first capacity question is "how many machines do I need to serve X requests per second". Two complementary approaches: throughput-based and resource-based.

For the broader back-of-envelope toolkit, see [System Design — Back-of-Envelope Estimation](../../system-design/foundations/back-of-envelope-estimation.md).

### 2.1 Throughput-Based Sizing

Measure (via load test) the maximum sustained throughput a single instance can handle while staying within latency SLO. Call this `λ_max_per_box`. Required instances:

```
N = ⌈ peak_load / (λ_max_per_box × (1 − headroom)) ⌉
```

If load test says one box does 500 req/s at 80% CPU and you want 50% headroom (target 50% CPU at peak):

```
λ_max_per_box at 50% CPU ≈ 500 × (50/80) = 312.5 req/s
N for 5,000 peak rps = ⌈ 5000 / 312.5 ⌉ = 16 boxes
```

The headroom factor protects against the latency knee (Section 4) and provides slack for failure-mode capacity (Section 7).

### 2.2 Resource-Based Sizing

Compute per-request resource cost:

- CPU per request (CPU-seconds per request × peak rps) ≤ available CPU.
- Memory per request × concurrent requests ≤ available RAM.
- Bytes/sec ingress + egress ≤ NIC bandwidth.
- Disk IOPS per request × peak rps ≤ disk IOPS limit.

Whichever resource saturates first is the bottleneck. Capacity is sized to that resource.

Example: Spring Boot service. Per request: 5 ms CPU, 2 KB heap allocation, 1 KB ingress/egress, 0 DB-direct disk IOPS (the DB is separate).

At 5,000 rps:
- CPU demand = 5000 × 0.005 = 25 CPU-seconds/sec = 25 cores.
- With 50% headroom: 50 cores total.
- 4-vCPU instances: 13 boxes.
- 8-vCPU instances: 7 boxes.

### 2.3 Storage and IOPS Sizing

Database capacity has different math than service capacity. Consider:

- **Storage size:** rows × per-row size × replication factor + index overhead (typically 30-100% of base size). Project growth.
- **Read IOPS:** dominant during peak load. Cache hit rate matters; at 95% hit rate, 1,000 read rps becomes 50 IOPS to disk.
- **Write IOPS:** more constant; depends on write path (WAL, commit groups).
- **Connection capacity:** Postgres has per-connection memory cost; max ~200-500 connections per modest instance.

For Postgres on AWS RDS, the storage IOPS limit (default: 3 IOPS per GB on gp3, max 16,000) is often the wall, not CPU. Verify before sizing instance class.

See [Database — Connection Management](../../database/operations/connection-management.md) and [Database — Monitoring](../../database/operations/monitoring.md).

---

## 3. Peak-to-Average Ratio

The peak-to-average ratio (PAR) is the ratio of peak hour load to average hour load. Typical patterns:

| Service Type | Daily PAR | Weekly PAR (peak-day to off-day) |
|--------------|-----------|----------------------------------|
| Internal tools | 4-6× | 5× (zero on weekends) |
| Consumer e-commerce | 3-5× | 2× (weekend higher) |
| Streaming media | 2-3× | 1.2× |
| B2B SaaS (global) | 2× | 1.5× |
| Background batch | 1× | 1× |

Provisioning at the average will fail at peak. Provisioning at peak wastes money during the trough.

Two strategies:

1. **Provision for peak.** Simple, expensive at scale. Acceptable when peak/avg is low or when you have hard SLOs that punish autoscaling lag.
2. **Autoscale across the curve.** Set a floor that handles 1.5× the average and a ceiling that handles peak with headroom. Save 30-60% on infrastructure cost. Pay the autoscaling-lag tax (Section 6).

Hybrid: provision the **base** floor for steady-state, autoscale the **delta** to peak. This is the standard pattern in Kubernetes (HPA with `minReplicas` floor and `maxReplicas` ceiling).

For services with infrequent but very high spikes (e.g., a news event drives 20× normal), maintain a **warm pool** of instances that can be promoted to active capacity in seconds without going through cold-start.

---

## 4. The Headroom Rule

### 4.1 Why 50%

The "50% headroom rule" (target steady-state utilization at 50% of measured saturation point) compounds three concerns:

1. **The latency knee.** As shown in [Little's Law](02-littles-law-and-queueing-theory.md), latency is `~ρ/(1-ρ) × E[S]`. At ρ=0.5, queue wait ≈ 1× service time. At ρ=0.8, queue wait ≈ 4× service time. The 50% target keeps latency below the knee.
2. **Burst absorption.** Real arrivals are bursty. A 2× short burst at ρ=0.5 baseline pushes the system to ρ=1.0 momentarily — but for a few seconds, that's tolerable. The same burst at ρ=0.8 baseline is fatal (the system goes deeply unstable).
3. **Failure-mode capacity.** If you have N boxes and one fails, the remaining (N-1) need to absorb the full load. At 50% baseline utilization, losing one of two boxes leaves a single box at 100% — survivable. At 80% baseline, losing one box overloads the other.

For services with strict latency SLOs, use 50%. For services with lax SLOs (batch, eventual consistency), 70-80% is acceptable.

### 4.2 Headroom Across the Stack

Headroom must be applied at each layer:

| Layer | Resource | Target Utilization at Peak |
|-------|----------|----------------------------|
| Application | CPU | 50-60% |
| Application | Heap | 70-80% (post-GC) |
| Application | Thread/connection pool | 60-70% |
| Database | CPU | 60-70% |
| Database | IOPS | 50-70% |
| Database | Connections | 70-80% |
| Cache | Memory | 80-90% (eviction is OK) |
| Network | Bandwidth | 50-60% (for cross-AZ) |
| Disk | Free space | < 80% used |

Disk free space is non-negotiable: filesystems often degrade or fail past 90% full. Set hard alerts at 80%.

### 4.3 Headroom and Cost

Headroom costs money. The argument for it is:

- Lost-revenue cost of a saturation event > excess-infrastructure cost.
- Most outages are capacity-related, and capacity-related outages are entirely preventable.
- Engineering time spent firefighting saturation > the savings of running tighter.

For a finance-team conversation: convert the SLO into expected cost of violation (per Google SRE: error budget × revenue impact). Headroom is the price of staying inside the error budget.

---

## 5. Forecasting Growth

### 5.1 Linear and Exponential Models

For a stable product, traffic grows roughly linearly or with a slow exponential. Fit a regression to the last 90 days of daily peak rps:

- **Linear:** `peak_rps(t) = a + b·t`. Suitable for mature products.
- **Exponential:** `peak_rps(t) = a · e^(bt)`. Common for early-growth products.
- **Compound exponential with cap:** logistic growth (ceiling at market saturation). Mature SaaS.

Pick the model that fits R² > 0.9. Project 30/60/90 days forward. Reorder capacity now to be ready when traffic gets there, accounting for **provisioning lead time** (cloud: minutes; physical: weeks; new region: months).

### 5.2 Seasonal Patterns

Most services have weekly seasonality (peak weekdays vs trough weekends, or vice versa for consumer products) and yearly seasonality (Black Friday for commerce, end-of-quarter for B2B SaaS, fiscal year for some).

Use a model that decomposes trend + season + residual (e.g., STL, Prophet, ETS). Forecast peak-of-peak (the maximum hourly rps inside the maximum day inside the maximum week).

For Black Friday-class events: pre-provision in advance based on prior years' growth × this year's run-rate growth. Don't rely on autoscaling — the ramp is too fast and the cost of getting it wrong is too high.

### 5.3 Step-Function Events

Some traffic changes are not smooth. Plan capacity for:

- **Marketing campaigns / paid ads.** Get the campaign schedule from marketing.
- **Mobile push notifications.** A push to 1M users sends 1M users to your app within minutes.
- **Press / launches.** Coordinate with the launch team.
- **Geographic expansion.** New region = new capacity, new latency profile.
- **Feature flag rollouts.** Especially when the new feature changes per-request cost (e.g., a new ML inference path).

Each is a known-unknown — *that* it will happen is known, *when* and *how big* require coordination. Capacity reviews should explicitly enumerate them for the next 90 days.

---

## 6. Scaling Triggers and Autoscaling Lag

### 6.1 What to Scale On

Common autoscaling triggers, ranked by quality:

| Trigger | Quality | Notes |
|---------|---------|-------|
| **Per-pod request rate** | Best | Direct measure of demand. Requires custom metric. |
| **CPU utilization** | Decent | Lags request rate; works for CPU-bound services. |
| **Custom application metric** (queue depth, in-flight requests) | Best | If you can correlate it with saturation. |
| **Memory utilization** | Bad | JVM/V8 won't release memory; signal is unreliable. |
| **Latency** | Bad | Lagging indicator; by the time latency spikes, it's too late. |

Kubernetes HPA can scale on CPU (default), memory, or custom metrics via Prometheus Adapter / KEDA.

### 6.2 Autoscaling Lag in Real Numbers

Total time from "load increased" to "new capacity is serving traffic":

| Step | Typical Time |
|------|--------------|
| Metric collection interval | 15-60 sec |
| Metric scrape + aggregation | 30 sec |
| HPA evaluation interval | 15-30 sec |
| Pod scheduling | 5-30 sec |
| Image pull | 0-60 sec (cached) - 5+ min (cold) |
| Container start | 1-10 sec |
| **JVM startup** | 30-60 sec (Spring Boot), 200ms (GraalVM native) |
| **Node.js startup** | 1-3 sec |
| Health check warmup | 10-30 sec |
| LB picks up new pod | 5-30 sec |
| **Total** | **2-10 minutes** |

In a fast burst (load jumps in 30 seconds), autoscaling lag means existing capacity must absorb the spike alone for 2-10 minutes. Headroom matters more than autoscaling speed.

For JVM services, the biggest lever is **startup time**. Spring Boot AOT compilation, GraalVM native images, and CDS archives can reduce startup from 60s to under 5s. See [Java — Spring Startup Lifecycle](../../java/spring-startup-lifecycle.md).

### 6.3 Why Static Floors Exist

Setting `minReplicas: 1` saves money but means the first burst hits a single instance. Most production deployments use `minReplicas` ≥ 3 (one per AZ for AZ-redundancy and burst absorption) and let the HPA scale up from there.

**The floor handles the spike. The ceiling handles the peak.** Set both deliberately.

---

## 7. Failure-Mode Capacity — N+1 and N+2

### 7.1 The Concept

If your peak load requires N boxes, and you provision exactly N, then losing one box puts you at 100/(N) % overload — the system saturates. Capacity planning has to account for failures.

- **N+1:** provision one extra. Survives a single failure (instance crash, AZ outage of one zone, planned maintenance).
- **N+2:** survives two simultaneous failures. Used for higher-tier services or services in regions with frequent maintenance.

Failure-mode capacity is **not** the same as the headroom rule (Section 4). Both apply:

- Headroom keeps you below the latency knee in steady state.
- N+1 ensures that even after a failure, you remain below saturation.

For a service that needs N=10 boxes at the latency knee, with 50% headroom that becomes 20 boxes for steady state, plus N+1 = 22, plus N+2 = 24. The math compounds.

### 7.2 N+1 Math

Required instances under a failure model with `f` allowed simultaneous failures:

```
N_provision = ⌈ N_required × (1 + headroom_factor) × (replicas / (replicas - f)) ⌉
```

Example: peak demand = 100 cores. 50% headroom → 200 cores. Want N+1 across 4 AZs (each AZ ~25% of capacity).

Per-AZ: 50 cores. Lose one AZ: 150 cores survive. To still serve 100 cores of peak, 150 ≥ 100 ✓ (33% headroom remains). If steady-state baseline is 200, loss of one AZ makes survivors run at 100/150 = 67% of capacity — within the headroom rule.

If the deployment has 3 AZs instead of 4, each AZ holds 33%; losing one leaves 67% of capacity = 134 cores, vs 100 needed → 34% headroom. Tighter but acceptable.

### 7.3 Regional Capacity

For multi-region deployments, the failure unit is a region. Active-active deployment with N regions and traffic split N ways: losing one region pushes 1/N more load onto each survivor.

Active-active with two regions: each region must hold 100% of traffic for the active-passive case. That doubles your capacity. If full-region failover is rare (annual), some teams accept reduced SLO during failover and provision each region for ~70% of total traffic. Trade-off, document it.

For single-region with multiple AZs: provision so that any AZ can fail and the surviving AZs hold full capacity at the headroom target.

### 7.4 Cell-Based Architectures

Cell-based architecture (used by AWS, Google) partitions the workload into "cells", each handling a fraction of users. Failure of one cell affects only its users. Capacity within each cell follows the rules above.

Benefits: blast radius is bounded; failure of one cell doesn't cascade. Cost: provisioning headroom must be replicated per cell, which is more expensive than centralized headroom.

For most teams, AZ-redundancy with N+1 is the right level. Cell-based architectures are appropriate at hyperscale or when blast-radius concerns dominate (large multi-tenant SaaS, banking).

---

## 8. Cost vs Reliability Curves

A useful exercise: plot expected cost (infrastructure + downtime cost) vs availability target.

```text
total cost
  ^
  |\
  | \
  |  \         infrastructure cost
  |   \________________________________
  |                        |
  |    downtime cost       |     /
  |                        |    /
  |    \                   |   /
  |     \                  |  /
  |      \_________________|_/
  |                        |
  +─────────────────────────────────> availability target
   90%  99%  99.9%  99.99%   99.999%
                       ^
              minimum total-cost point
              (your real target)
```

Infrastructure cost grows superlinearly past ~99.9% (more replicas, more regions, more headroom, more redundancy). Downtime cost (lost revenue, SLA penalties) drops superlinearly with each "nine".

The crossover point is roughly where you should set your SLO. For most consumer SaaS that's 99.9-99.95%. For payments/banking it's 99.99%+. For internal tools, 99% or less.

Cargo-culting "five nines" without doing this math is a way to spend infrastructure money that produces no business value. So is racing to 99% and discovering that downtime cost dominates.

See [System Design — SLA/SLO/SLI](../../system-design/foundations/sla-slo-sli-and-availability.md).

---

## 9. Capacity Review Cadence

A workable rhythm:

- **Weekly:** quick check on per-service capacity utilization and forecast vs actual. Adjust autoscaling parameters.
- **Monthly:** full capacity review. For each service: peak load (last 30 days), peak load (forecast next 30/60/90 days), current capacity, headroom, failure-mode capacity. Identify services with diminishing headroom.
- **Quarterly:** infrastructure budget review. Reconcile forecast vs spend. Plan for events (launches, seasonal peaks).
- **Annually:** strategic. Region expansion, hardware refresh, multi-cloud, capacity locality (for new markets).

The artifact is a capacity dashboard or doc per service with: current peak, capacity ceiling, projected peak in 90 days, headroom, failure-mode capacity, and identified risks. Review it the same way you review SLO compliance.

The cost of capacity planning is a few engineer-hours per month. The cost of *not* doing it is the next saturation incident at 3am.

---

## Related

- [Latency, Throughput, Percentiles](01-latency-throughput-percentiles.md)
- [Little's Law and Queueing Theory](02-littles-law-and-queueing-theory.md)
- [Load Testing Methodology](06-load-testing-methodology.md)
- [System Design — Back-of-Envelope Estimation](../../system-design/foundations/back-of-envelope-estimation.md)
- [System Design — Capacity Planning and Load Testing](../../system-design/performance-observability/capacity-planning-and-load-testing.md)
- [System Design — SLA/SLO/SLI and Availability](../../system-design/foundations/sla-slo-sli-and-availability.md)
- [System Design — Horizontal vs Vertical and Stateless](../../system-design/scalability/horizontal-vs-vertical-and-stateless.md)
- [Java — Spring Startup Lifecycle](../../java/spring-startup-lifecycle.md)
- [Database — Connection Management](../../database/operations/connection-management.md)

---

## References

- **Google SRE Book — "Capacity Planning" (Chapter 18).** https://sre.google/sre-book/software-engineering-in-sre/ — practical capacity planning at Google scale.
- **Google SRE Workbook — "Non-abstract Large System Design" (Chapter 12).** https://sre.google/workbook/non-abstract-design/ — capacity-driven system design.
- **Neil J. Gunther — "Guerrilla Capacity Planning", Springer, 2007.** Hands-on capacity planning with queueing theory.
- **Mor Harchol-Balter — "Performance Modeling and Design of Computer Systems", Cambridge University Press, 2013.** Chapters on M/M/c queues and the capacity implications.
- **Brendan Gregg — "Systems Performance: Enterprise and the Cloud", 2nd ed., Pearson, 2020.** Chapter 2 (Methodologies) covers the capacity-planning method.
- **AWS Well-Architected Framework — Reliability Pillar.** https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html — N+1, multi-AZ, multi-region.
- **Google SRE Workbook — "Managing Load" (Chapter 11).** https://sre.google/workbook/managing-load/ — autoscaling, load shedding, and capacity coupling.
- **Kubernetes Horizontal Pod Autoscaler.** https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- **KEDA (Kubernetes Event-driven Autoscaling).** https://keda.sh/ — for queue-depth-based autoscaling.
- **AWS — Cell-based architectures.** https://docs.aws.amazon.com/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/reducing-scope-of-impact-with-cell-based-architecture.html
- **Werner Vogels — "Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service", USENIX ATC 2022.** https://www.usenix.org/conference/atc22/presentation/elhemali — capacity in production at hyperscale.
