---
title: "Load Testing Methodology — Open vs Closed Workloads, Trustworthy Numbers"
date: 2026-05-03
updated: 2026-05-03
tags: [performance, load-testing, open-loop, closed-loop, k6, wrk2, gatling]
---

# Load Testing Methodology — Open vs Closed Workloads, Trustworthy Numbers

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `performance` `load-testing` `open-loop` `closed-loop` `k6` `wrk2` `gatling`

---

## Table of Contents

- [Summary](#summary)
- [1. Why Most Load-Test Numbers Are Wrong](#1-why-most-load-test-numbers-are-wrong)
- [2. The Open vs Closed Workload Model](#2-the-open-vs-closed-workload-model)
  - [2.1 Closed Workload (Fixed Concurrency)](#21-closed-workload-fixed-concurrency)
  - [2.2 Open Workload (Fixed Arrival Rate)](#22-open-workload-fixed-arrival-rate)
  - [2.3 Why Real Users Are Open](#23-why-real-users-are-open)
  - [2.4 The Schroeder/Wierman/Harchol-Balter Result](#24-the-schroederwiermanharchol-balter-result)
- [3. Coordinated Omission in the Tool Layer](#3-coordinated-omission-in-the-tool-layer)
- [4. Tools and What They Get Right](#4-tools-and-what-they-get-right)
  - [4.1 wrk2](#41-wrk2)
  - [4.2 k6](#42-k6)
  - [4.3 Gatling](#43-gatling)
  - [4.4 JMeter](#44-jmeter)
  - [4.5 Vegeta](#45-vegeta)
  - [4.6 Apache Bench (`ab`)](#46-apache-bench-ab)
- [5. Load Profiles That Actually Test Something](#5-load-profiles-that-actually-test-something)
  - [5.1 Steady State (Baseline)](#51-steady-state-baseline)
  - [5.2 Ramp Test](#52-ramp-test)
  - [5.3 Soak Test](#53-soak-test)
  - [5.4 Spike Test](#54-spike-test)
  - [5.5 Stress Test](#55-stress-test)
- [6. Realistic Workload Construction](#6-realistic-workload-construction)
  - [6.1 Mix of Endpoints](#61-mix-of-endpoints)
  - [6.2 Cardinality of Inputs](#62-cardinality-of-inputs)
  - [6.3 Think Time](#63-think-time)
  - [6.4 Authentication and Session State](#64-authentication-and-session-state)
- [7. Where to Run the Test](#7-where-to-run-the-test)
- [8. What to Measure](#8-what-to-measure)
- [9. Common Mistakes Catalogue](#9-common-mistakes-catalogue)
- [Related](#related)
- [References](#references)

---

## Summary

Most load tests in the wild produce numbers that look plausible and are quietly wrong. The errors come from two systemic mistakes: using a **closed-loop** workload generator (a fixed pool of "virtual users" each issuing requests serially) when real users behave **open-loop** (arrive on a schedule, regardless of whether previous users got served), and falling into **coordinated omission** at the tool layer (the tool's measured latencies don't include time the tool *should have* been sending requests but couldn't because the system stalled). Both errors systematically *under-estimate* tail latency, so the tool reports a healthy p99 while the system would fall over in production. This doc covers the open vs closed model formally (Schroeder, Wierman, Harchol-Balter, "Open Versus Closed: A Cautionary Tale", NSDI 2006), what the right tools do and don't do, the load-profile shapes (ramp, soak, spike, stress) and what each tells you, and a checklist of common mistakes that turn load tests into expensive ways to lie to yourself.

---

## 1. Why Most Load-Test Numbers Are Wrong

A typical industry load test:

> "We ran 100 virtual users against the API for 10 minutes. p99 latency was 80 ms. We're good for production."

Three things are wrong with this statement before you've heard the system specs:

1. **"100 virtual users" is a closed-loop description.** Each virtual user is a thread that sends a request, waits for the response, then sends the next. Total in-flight requests is bounded by the pool size. Real users do not behave this way.
2. **The tool's reported p99 is suspect.** Most closed-loop tools (JMeter default, Gatling default, ApacheBench, `wrk`) suffer coordinated omission and under-report tail latency.
3. **No mention of arrival rate.** If "100 virtual users" produced 50 req/s in the test, and production peak is 500 req/s, the test underloaded the system by 10×. The reported p99 is for a workload the system will never see.

A trustworthy load test specifies an **arrival rate** (requests per second), uses an **open-loop** generator, and reports **uncoordinated** latencies — meaning latency measured from the time a request *should have been sent* under the schedule, not when the tool actually got around to sending it.

---

## 2. The Open vs Closed Workload Model

### 2.1 Closed Workload (Fixed Concurrency)

Closed workload model: a fixed number `N` of virtual users circulate through the system. Each user issues a request, waits for the response, optionally "thinks" for some time, then issues the next request. The number of in-flight requests is bounded by `N`.

```text
think → request → wait response → think → request → ...
   (one user's lifecycle, repeated forever)
```

Used to model: workloads with a bounded user population that has feedback (think: a fixed-size batch processing pipeline, or a UI with a constrained number of operators).

### 2.2 Open Workload (Fixed Arrival Rate)

Open workload model: requests arrive according to an external schedule (often Poisson at rate `λ`). The number of in-flight requests is *not bounded* by the generator — it is whatever ends up in the system as arrivals minus completions.

```text
arrivals at λ = 500 req/s, regardless of system response
   ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓
                  [system queues + processes]
```

Used to model: real user-facing services. Users arrive on the internet's schedule, not on the schedule of "I just got my response and now I'll think for 3 seconds".

### 2.3 Why Real Users Are Open

Web traffic is a superposition of many independent users, each issuing requests independent of the system's response time *to other users*. From the system's perspective, the arrival process is open: bursts happen, and the system has to absorb them. Closed-loop testing artificially throttles the arrival rate when the system slows down — exactly the opposite of what real load does.

This matters most at the **knee** of the latency-vs-load curve. In a closed-loop test, when the system slows down, the virtual users' next requests are delayed too — the offered load decreases automatically. The system never actually saturates from the test's perspective. In production, when the system slows down, users keep arriving — load does not decrease — and the system does saturate.

Closed-loop tests are blind to saturation behavior. They report graceful degradation that does not exist.

### 2.4 The Schroeder/Wierman/Harchol-Balter Result

Schroeder, Wierman, and Harchol-Balter, "Open Versus Closed: A Cautionary Tale", NSDI 2006, formalized this. Their key result: **closed-loop testing systematically underestimates the tail latency of open-loop workloads by an arbitrary factor**. There is no fixing this with longer tests or more virtual users — the model is wrong.

The paper showed concrete examples where closed-loop measurements yielded p99 latencies of 5 ms while equivalent open-loop measurements (same throughput, same service-time distribution) yielded p99 of 5 *seconds*. The system was the same. The model and the measurement gave fundamentally different answers, and the open-loop one matched what production would experience.

If you take one thing from the paper: **specify your test by arrival rate, not by concurrency.** "We sustained 1,000 req/s arrivals" is a meaningful claim. "We had 100 virtual users" is not.

---

## 3. Coordinated Omission in the Tool Layer

Already covered in [Latency / Percentiles § Coordinated Omission](01-latency-throughput-percentiles.md#6-coordinated-omission--the-bug-that-hides-tails). The summary as it applies to load tools:

A tool that intends to send 1 request every 1 ms (1,000 rps) but actually issues each request *only after the previous one returned* will, when the system stalls for 100 ms:

- Send 1 request, wait 100 ms for it to return.
- Record latency = 100 ms for that request.
- *Skip 99 requests that should have been sent during the stall.*

A correct tool either (a) maintains the schedule independent of completions (open-loop), or (b) records latencies relative to **intended** start time (which captures the queue wait the request would have suffered), or (c) uses HdrHistogram's `recordValueWithExpectedInterval()` to backfill the omitted samples retroactively.

`wrk` does (c) only partially. `wrk2` does it correctly. `k6` with its arrival-rate executor and `Gatling` with `constantUsersPerSec` injection profile do it correctly. `JMeter` default does *none* of the above and reports systematically rosy tail percentiles.

---

## 4. Tools and What They Get Right

### 4.1 wrk2

[`wrk2`](https://github.com/giltene/wrk2) is Gil Tene's fork of `wrk` with constant-throughput open-loop generation and HdrHistogram-based coordinated-omission correction. It is the lowest-overhead correct tool for HTTP perf testing.

```bash
# 1,000 requests/sec for 60 seconds, 4 threads, 100 connections, latency CSV output
wrk2 -t4 -c100 -R 1000 -d 60s --latency http://target:8080/api/checkout

# Output includes p50, p90, p99, p99.9, p99.99 with corrected coordinated omission
```

The `-R` flag specifies arrival rate; `wrk2` schedules requests on that rate regardless of completions, and records latency relative to intended start.

Limitations: HTTP/1.1 only (no HTTP/2, no gRPC, no WebSocket). Lua scripting available for custom request generation.

### 4.2 k6

[`k6`](https://k6.io/) (Grafana Labs) is the most ergonomic modern tool. JavaScript-defined scripts, multiple "executors" including arrival-rate, ramping arrival-rate, and constant VU (closed-loop, for comparison).

```javascript
// k6 script: 500 req/s for 5 minutes
export const options = {
  scenarios: {
    constant_arrival: {
      executor: 'constant-arrival-rate',
      rate: 500,
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 200,
      maxVUs: 500,
    },
  },
  thresholds: {
    'http_req_duration{status:200}': ['p(99) < 200'],  // p99 < 200ms
    'http_req_failed': ['rate < 0.01'],
  },
};

import http from 'k6/http';
export default function () {
  http.get('http://target:8080/api/checkout');
}
```

`constant-arrival-rate` and `ramping-arrival-rate` executors are open-loop. The `constant-vus` executor is closed-loop (use for explicit closed-loop modeling, not as the default).

k6 supports HTTP/1, HTTP/2, gRPC, WebSocket, and ships with built-in HdrHistogram. Output via Prometheus, Cloud, or InfluxDB.

### 4.3 Gatling

[Gatling](https://gatling.io/) supports both injection profiles. For open-loop testing, use the `constantUsersPerSec` or `rampUsersPerSec` injection:

```scala
setUp(
  scn.inject(
    constantUsersPerSec(500) during (5 minutes)
  )
).protocols(http.baseUrl("http://target:8080"))
```

`constantUsersPerSec` schedules new virtual users at the given rate, *not* a fixed pool of users. Each "user" runs the scenario once. This produces an open-loop workload at the given arrival rate.

The closed-loop equivalent is `atOnceUsers` or `nothingFor`/`incrementUsersPerSec` — useful for comparison but not the right default for HTTP perf testing.

### 4.4 JMeter

[Apache JMeter](https://jmeter.apache.org/) is the most widely deployed load testing tool and the most widely *misused*. The default "Thread Group" is a closed-loop generator (`N` threads, each running a scenario in a loop). To get open-loop behavior, use the **Concurrency Thread Group** plugin or the **Throughput Shaping Timer** plugin to schedule arrival rates explicitly.

JMeter's reported percentiles are computed from raw measurements and do not correct coordinated omission. For trustworthy results, post-process JMeter raw samples through HdrHistogram.

If your team standardizes on JMeter, the discipline is:

1. Use a Concurrency-based Thread Group with arrival-rate scheduling (not Thread Group's default).
2. Export raw `.jtl` samples and aggregate them externally (HdrHistogram, k6 cloud, custom script).
3. Do not trust JMeter's built-in percentile reports.

### 4.5 Vegeta

[`vegeta`](https://github.com/tsenart/vegeta) is a Go HTTP load tester with constant-rate (open-loop) generation by default:

```bash
echo "GET http://target:8080/api/checkout" | \
  vegeta attack -rate=500 -duration=5m | \
  vegeta report -type=json
```

`vegeta`'s percentile output is honest because it's open-loop and records intended-vs-actual times. It also has a nice latency CSV export for flame charts.

Lightweight and pipe-friendly; great for CI integration.

### 4.6 Apache Bench (`ab`)

`ab` is closed-loop, single-threaded per connection, no coordinated omission correction. Its output is unsuitable for tail-latency analysis. It's still useful for quick smoke tests and small-scale testing where the absolute numbers don't matter (e.g., "does the endpoint return 200?"), but never quote `ab` numbers as performance characteristics.

---

## 5. Load Profiles That Actually Test Something

Each profile answers a different question.

### 5.1 Steady State (Baseline)

Run at the expected production peak rate for 10-30 minutes. The point is to characterize p50, p95, p99, p99.9 at production load with full warm caches and JIT-stable code.

This is the test you compare every other test against. Run it after every release; flag regressions.

### 5.2 Ramp Test

Increase arrival rate gradually (e.g., 100 → 1,000 req/s over 15 minutes) and observe where the latency-vs-load curve bends. The bend point is your **measured knee**.

The knee is the most operationally useful number a load test produces. Plan capacity for steady-state at 50-70% of the knee, with peak at 80%.

```javascript
// k6 ramping arrival rate
scenarios: {
  ramp: {
    executor: 'ramping-arrival-rate',
    startRate: 100,
    timeUnit: '1s',
    preAllocatedVUs: 500,
    maxVUs: 2000,
    stages: [
      { target: 200, duration: '5m' },
      { target: 500, duration: '5m' },
      { target: 1000, duration: '5m' },
      { target: 2000, duration: '5m' },
    ],
  },
},
```

### 5.3 Soak Test

Steady moderate load (e.g., 50% of peak) for 8-24 hours. Detects:

- Memory leaks (RSS climbing over time).
- Connection pool leaks (in-use connections climbing without recovery).
- File descriptor leaks.
- Cache poisoning (slow degradation of hit rate).
- Background job interference (compaction, vacuum, log rotation).
- Disk fill (logs, temp files).

Soak tests are boring to run and the most likely to find bugs that would otherwise hit in week 2 of production. Schedule them weekly.

### 5.4 Spike Test

Sudden 5-10× arrival-rate increase for a short window (1-5 minutes), then back to baseline. Tests:

- Autoscaling response (does the autoscaler add capacity in time?).
- Backpressure (does the service shed load gracefully or collapse?).
- Recovery (does latency return to baseline after the spike, or does the system stay degraded?).
- Cold-cache behavior (the spike often hits cold paths).

Spike profiles in real services come from launches, news events, retry storms, and traffic-routing changes. Test for them.

### 5.5 Stress Test

Push beyond the knee until something breaks. Goal: characterize *failure mode*, not throughput.

You want to discover whether the service:

- Sheds load gracefully (returns 503 with `Retry-After`, latency stays bounded).
- Locks up (latency climbs unbounded; OOMKilled; deadlock).
- Cascades (failure propagates to downstream or upstream services).

Stress tests are best run in a representative environment (production-like infrastructure) and **never** against production. Use staging or a parallel preproduction fleet.

---

## 6. Realistic Workload Construction

### 6.1 Mix of Endpoints

Real services have many endpoints with different cost profiles. A test that hits only `/health` will report excellent numbers and tell you nothing. The test mix should reflect production traffic mix, ideally driven by sampling production access logs.

```javascript
// k6 weighted scenarios
scenarios: {
  search: { executor: 'constant-arrival-rate', rate: 300, exec: 'search' },
  product: { executor: 'constant-arrival-rate', rate: 150, exec: 'product' },
  checkout: { executor: 'constant-arrival-rate', rate: 50, exec: 'checkout' },
}
```

### 6.2 Cardinality of Inputs

A test hammering `GET /products/1` over and over will stay in cache. Real users hit thousands of distinct products, blowing through the cache. Use a sufficiently diverse input set (sampled from production) so cache hit rates match production.

For Postgres, this also means realistic query plan diversity — a single repeated query gets cached query plans; varied queries hit the planner.

### 6.3 Think Time

If you are *modeling* user behavior with closed-loop generators, add think time between requests so virtual users don't issue an unrealistic firehose. For open-loop generators with arrival-rate scheduling, think time is irrelevant — you're scheduling requests directly.

### 6.4 Authentication and Session State

Authenticated endpoints need realistic session handling. Pre-generate a pool of session tokens and cycle through them. Avoid using a single session for the entire test — auth caches will mask realistic hot-key behavior.

For services that perform per-user rate limiting, you must rotate users; otherwise you'll just measure your own rate limiter.

---

## 7. Where to Run the Test

The load generator must be:

- **Close enough** to the system under test that network latency from generator to target is small relative to the service's expected latency. Tests run from a different region add multi-ten-millisecond noise to every measurement.
- **Far enough** from the system that the generator is not stealing CPU/network from the target. Run on a separate machine, ideally in the same VPC, not on the target host.
- **Adequately sized.** A 1,000-rps test from a 1-CPU laptop saturates the laptop, not the target.
- **Not in production.** Always.

For cloud testing, a separate VPC subnet or a managed service (k6 Cloud, Loader.io, Gatling Frontline, Datadog Synthetics) avoids the most common topology errors.

---

## 8. What to Measure

Per test:

| Metric | Source |
|--------|--------|
| Arrival rate (intended vs actual) | Load generator |
| Latency p50, p90, p99, p99.9 | HdrHistogram from generator |
| Error rate (broken down by status class) | Load generator |
| CPU, memory, GC, disk, network on the target | USE dashboard on target |
| Downstream call rates and latencies | RED on target's dependencies |
| Connection pool / thread pool saturation | Application metrics |
| Database query rate and EXPLAIN snapshots | DB-side monitoring |

The point of the load test is not a single p99 number — it is a *correlated* picture: as load increases, what utilizations climb, what tails widen, where does the knee bend, and where does the system fail.

Without target-side metrics, the load test tells you "p99 was 200 ms"; with them, it tells you "p99 was 200 ms and the bottleneck was the DB pool — the fix is bigger pool or faster queries".

---

## 9. Common Mistakes Catalogue

- **Closed-loop generator reporting tail latencies.** Numbers are systematically rosy. Switch to open-loop or use coordinated-omission-corrected tooling.
- **Single-endpoint test claiming whole-service performance.** The endpoint with the lowest latency is not your performance.
- **Cold cache during the test.** First minute of every test is unrepresentative. Discard it or warm the cache explicitly.
- **Measuring through a CDN or LB cache.** You're measuring the cache. Hit the origin directly or invalidate aggressively.
- **No baseline.** A single number means nothing without comparison to last week's number on the same workload.
- **Generator on the same host as target.** Load generation steals CPU, distorts metrics. Use a separate machine.
- **Tiny target (1 CPU) and assuming it scales linearly.** Per-process overhead dominates at small sizes; behavior at 1 CPU is not predictive of behavior at 16 CPUs.
- **Ignoring downstream warm-up.** Database connection establishment, JIT warmup, auth-cache warmup — all real and all skew the first minutes.
- **Reporting only mean throughput.** Throughput at saturation is *higher* than throughput at the knee, but latency is unusable. Always report throughput + latency together.
- **No spike or soak test.** Steady-state numbers don't capture failure modes that only appear in production.
- **Hard-coded inputs.** Tests one cache hot path, misses cold-path performance.
- **No production traffic shape.** A 100% read test on a 60/40 read-write service measures the wrong thing.

---

## Related

- [Latency, Throughput, Percentiles](01-latency-throughput-percentiles.md)
- [Little's Law and Queueing Theory](02-littles-law-and-queueing-theory.md)
- [Tail Latency and p99](03-tail-latency-and-p99.md)
- [Capacity Planning](07-capacity-planning.md)
- [System Design — Capacity Planning and Load Testing](../../system-design/performance-observability/capacity-planning-and-load-testing.md)
- [Database — EXPLAIN ANALYZE Guide](../../database/query-optimization/explain-analyze-guide.md)

---

## References

- **Bianca Schroeder, Adam Wierman, Mor Harchol-Balter — "Open Versus Closed: A Cautionary Tale", NSDI 2006.** https://www.usenix.org/legacy/event/nsdi06/tech/schroeder/schroeder.pdf — the formal result on why closed-loop testing misleads.
- **Mor Harchol-Balter — "Performance Modeling and Design of Computer Systems", Cambridge University Press, 2013.** Chapter 14 covers open vs closed workload models.
- **Gil Tene — "How NOT to Measure Latency".** https://www.youtube.com/watch?v=lJ8ydIuPFeU — coordinated omission and load tooling.
- **`wrk2`.** https://github.com/giltene/wrk2 — the reference for correct HTTP load generation.
- **`k6` — Test types and executors.** https://k6.io/docs/test-types/ and https://k6.io/docs/using-k6/scenarios/executors/
- **Gatling — Injection Profiles.** https://docs.gatling.io/reference/script/core/injection/
- **JMeter — Concurrency Thread Group plugin.** https://jmeter-plugins.org/wiki/ConcurrencyThreadGroup/
- **`vegeta`.** https://github.com/tsenart/vegeta
- **Brendan Gregg — "Systems Performance", 2nd ed., Pearson, 2020.** Chapter 12 (Benchmarking).
- **HdrHistogram.** https://hdrhistogram.org/ — `recordValueWithExpectedInterval` for retroactive coordinated-omission correction.
- **Google SRE Workbook — "Non-abstract Large System Design".** https://sre.google/workbook/non-abstract-design/ — capacity testing in distributed systems.
