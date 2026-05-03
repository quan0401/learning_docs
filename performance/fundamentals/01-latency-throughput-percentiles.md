---
title: "Latency, Throughput, Percentiles — How to Measure What Users Actually Feel"
date: 2026-05-03
updated: 2026-05-03
tags: [performance, latency, throughput, percentiles, hdr-histogram, coordinated-omission]
---

# Latency, Throughput, Percentiles — How to Measure What Users Actually Feel

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `performance` `latency` `throughput` `percentiles` `hdr-histogram` `coordinated-omission`

---

## Table of Contents

- [Summary](#summary)
- [1. Definitions That Get Confused on Purpose](#1-definitions-that-get-confused-on-purpose)
  - [1.1 Latency vs Response Time vs Service Time](#11-latency-vs-response-time-vs-service-time)
  - [1.2 Throughput vs Goodput vs Bandwidth](#12-throughput-vs-goodput-vs-bandwidth)
  - [1.3 Concurrency vs Parallelism vs Load](#13-concurrency-vs-parallelism-vs-load)
- [2. Why Averages Lie](#2-why-averages-lie)
  - [2.1 The Bimodal Reality](#21-the-bimodal-reality)
  - [2.2 The Mean Hides the Worst Experience](#22-the-mean-hides-the-worst-experience)
- [3. Percentiles — The Vocabulary You Need](#3-percentiles--the-vocabulary-you-need)
  - [3.1 p50, p90, p95, p99, p99.9](#31-p50-p90-p95-p99-p999)
  - [3.2 The Nine-Nines Slide](#32-the-nine-nines-slide)
  - [3.3 What Each Percentile Tells You](#33-what-each-percentile-tells-you)
- [4. Worked Example — A Realistic Distribution](#4-worked-example--a-realistic-distribution)
- [5. HDR Histogram — Recording Percentiles Honestly](#5-hdr-histogram--recording-percentiles-honestly)
  - [5.1 Why You Can't Just Store Every Sample](#51-why-you-cant-just-store-every-sample)
  - [5.2 How HDR Histogram Works](#52-how-hdr-histogram-works)
  - [5.3 Using HDR Histogram in Code](#53-using-hdr-histogram-in-code)
- [6. Coordinated Omission — The Bug That Hides Tails](#6-coordinated-omission--the-bug-that-hides-tails)
  - [6.1 What Coordinated Omission Is](#61-what-coordinated-omission-is)
  - [6.2 How Tools Cause It](#62-how-tools-cause-it)
  - [6.3 The Fix — Intended Start Time](#63-the-fix--intended-start-time)
- [7. Throughput Curves and the Knee](#7-throughput-curves-and-the-knee)
- [8. What to Track in Production](#8-what-to-track-in-production)
- [Related](#related)
- [References](#references)

---

## Summary

Performance numbers are political. The average latency in your dashboard exists because it makes the system look good, not because it describes user experience. Real systems have multimodal, heavy-tailed latency distributions — fast paths, slow paths, and rare disaster paths driven by GC pauses, lock contention, retries, queue saturation, and cold caches. Reasoning about averages is reasoning about a number nobody actually saw. This doc establishes the terminology you need (latency vs response time vs service time, throughput vs goodput), the percentile vocabulary that supersedes the average (p50/p90/p95/p99/p99.9), the HDR Histogram data structure that records percentiles without lying, and the coordinated-omission bug that makes most load-test results worthless. Get this right and the rest of performance engineering becomes possible. Get it wrong and your dashboards are decoration.

---

## 1. Definitions That Get Confused on Purpose

### 1.1 Latency vs Response Time vs Service Time

These three are routinely conflated. The distinction matters when you reason about queueing.

| Term | Meaning |
|------|---------|
| **Service time** | Time the server spends actively processing the request (CPU + IO directly attributable to this request). |
| **Wait time** (queueing time) | Time the request spent in queue before the server picked it up. |
| **Response time** | Wall-clock time the *client* observes from sending the request to receiving the full response. Equals service time + wait time + network round-trip. |
| **Latency** | Most rigorously: time spent waiting (queue + network). Colloquially used as a synonym for response time. Always check what the speaker means. |

In Gil Tene's vocabulary and in Brendan Gregg's writing, **latency = response time** as observed by the client. That's the convention used in the rest of this path.

```text
                                                  ┌── service time ──┐
client sends ──── network ──── queue ──── server processes ──── network ──── client receives
              ─────────────── response time (client-observed) ───────────────
```

A common mistake: a server that reports "average service time of 5ms" can have client-observed p99 latency of 5 *seconds* if its queue depth blows up under load. Service time is what the server *does*. Response time is what the user *suffers*.

### 1.2 Throughput vs Goodput vs Bandwidth

| Term | Meaning |
|------|---------|
| **Throughput** | Useful work completed per unit time. Measured in requests/sec, transactions/sec, bytes/sec, etc. |
| **Goodput** | Throughput excluding retries, errors, and non-useful traffic. The throughput your business actually got paid for. |
| **Bandwidth** | The capacity of the link. An upper bound on throughput, not a measurement of it. |

A 10 Gbps NIC has 10 Gbps of bandwidth. If half your traffic is retransmits and another 20% is failed requests, your goodput is much lower than your throughput, and your throughput is much lower than your bandwidth. All three numbers are real and useful, but they answer different questions.

### 1.3 Concurrency vs Parallelism vs Load

- **Concurrency** = number of in-flight requests at a given instant. Same as `L` in [Little's Law](02-littles-law-and-queueing-theory.md).
- **Parallelism** = number of physically simultaneous executions (cores, machines). Bounded by hardware.
- **Load** = arrival rate (`λ`) — requests per unit time entering the system.

Concurrency is what the system holds open. Load is what the world pushes in. Parallelism is what the hardware can chew on. They are independent dials.

---

## 2. Why Averages Lie

### 2.1 The Bimodal Reality

Real systems have at least two latency populations:

- **Fast path** — request is served from cache, hot CPU path, no GC, no lock contention. Microseconds to single-digit milliseconds.
- **Slow path** — cache miss, cold object, GC pause, lock contention, retry, or backend that itself was slow. Tens to thousands of milliseconds.

Often a third tail population from rare events: TCP retransmissions, leader-election stalls, JIT deoptimization, allocation failures.

The latency histogram looks like:

```text
count
 ^
 |  ##########
 |  ##########
 |  ##########                <-- fast path (cache hit)
 |  ##########
 |
 |          ###
 |          ###               <-- slow path (cache miss, lock contention)
 |          ###
 |
 |                       #    <-- tail (GC pause, retry)
 |                       #
 +--+--+--+--+--+--+--+--+--+-->  latency
   1ms     10ms    100ms    1s
```

This is **not** a bell curve. It is **not** a Gaussian. Computing a mean and standard deviation on it produces numbers that describe nothing real.

### 2.2 The Mean Hides the Worst Experience

Concrete: 1,000 requests measured at the application boundary.

- 990 requests served from cache: 5 ms each
- 10 requests hit a 2-second pause: 2,000 ms each

| Statistic | Value |
|-----------|-------|
| Mean | (990·5 + 10·2000) / 1000 = **24.95 ms** |
| Median (p50) | 5 ms |
| p99 | 5 ms (the 10th-worst is still in the fast bucket) |
| p99.1 | 2,000 ms |
| Max | 2,000 ms |

The mean of 25 ms describes neither the 990 users who saw 5 ms nor the 10 users who saw 2 seconds. Nobody experienced 25 ms. Worse, the mean is moved upward by the slow population enough to make the fast path look slow, while still hiding the disaster from anyone who only watches the average.

> If you remember one thing: **never alert on, never report, and never tune against the mean of a latency distribution.** Use percentiles.

---

## 3. Percentiles — The Vocabulary You Need

### 3.1 p50, p90, p95, p99, p99.9

A percentile pX is the value below which X% of samples fall. Concretely, sort all samples ascending; pX is the sample at index `ceil(X/100 · N)`.

| Percentile | Meaning |
|-----------|---------|
| **p50** (median) | Half of requests are faster, half slower. Describes the typical user experience but says nothing about tail. |
| **p90** | 90% of requests are at most this latency. Useful for "most users" claims. Often gameable. |
| **p95** | The slow-but-not-disaster threshold. Common SLO target for non-critical services. |
| **p99** | The user-experience threshold. 1 in 100 requests is worse than this. At 100 req/s per user that is once per second. |
| **p99.9** | The reliability threshold. Where queueing, GC, and infrastructure tails dominate. |
| **p99.99** | Where one rogue dependency in a fan-out tree manifests. See [Tail Latency](03-tail-latency-and-p99.md). |
| **max** | The single worst observation. Useful for outlier hunting, useless as an SLO. |

### 3.2 The Nine-Nines Slide

A useful frame from Gil Tene: each "nine" of the percentile is a different *user experience*, not a smooth gradation.

```text
 p50  → "the median user"
 p90  → "the slightly unlucky user"
 p99  → "the unlucky user"
 p99.9 → "the very unlucky user"
 p99.99 → "the user who hit a rare bug"
```

If your service handles 1,000 requests per second, p99.9 happens once a second, not once a day. The further into the tail you go, the more those rare events compound when fan-out enters the picture (see [Tail Latency](03-tail-latency-and-p99.md)).

### 3.3 What Each Percentile Tells You

- **p50 changes** → the typical path is slow. Could be the algorithm, the DB query plan, or the runtime baseline.
- **p99 changes but p50 stable** → tail-only regression. Likely GC, lock contention, retry behavior, or a slow dependency on a small fraction of requests.
- **p99.9 changes but p99 stable** → infrastructure event (TCP retransmits, scheduler stalls, hypervisor noise) or an extremely-rare-but-real bug.
- **p50 and p99 both rise together** → broad regression. Look at saturation (USE method).

You need **all** of these to reason. Reporting only p50 is reporting only half the system. Reporting only p99 hides the fact that the typical user got slower.

---

## 4. Worked Example — A Realistic Distribution

Let's measure a Spring Boot service hitting Postgres under steady load. The distribution is multimodal:

| Population | Share | Latency |
|-----------|-------|---------|
| Cache hit (Caffeine) | 70% | 0.5 ms |
| Cache miss → DB hot path | 28% | 12 ms |
| DB cold path (rare query plan) | 1.5% | 80 ms |
| GC pause coincidence | 0.4% | 250 ms |
| Connection pool wait | 0.1% | 1,500 ms |

Computed percentiles (round numbers):

| Percentile | Latency | Why |
|-----------|---------|-----|
| p50 | 0.5 ms | Median is in the cache-hit cluster |
| p90 | 12 ms | 90th percentile lands in the DB hot path |
| p95 | 12 ms | Still in DB hot path (28% population is wide) |
| p99 | 80 ms | First time the cold-path population dominates |
| p99.5 | ~250 ms | GC tail starts to register |
| p99.9 | ~1,500 ms | Connection pool starvation tail |
| Mean | ~7 ms | Useless number — describes no actual user |

Two takeaways:

1. The mean (~7 ms) is *between* every cluster and is not a value any individual saw.
2. The p99 (80 ms) is **160× worse than p50** (0.5 ms). That ratio — `p99/p50` — is a quick proxy for how heavy-tailed the system is. A ratio above ~10 means tail amplification will dominate any fan-out service built on top of you.

---

## 5. HDR Histogram — Recording Percentiles Honestly

### 5.1 Why You Can't Just Store Every Sample

Storing every latency sample is technically the most honest approach. At 10,000 req/s × 24 h that is 864M samples per day per service. Sortable but absurd. Most production systems aggregate.

The naive aggregation is to bucket into fixed-width bins (e.g., 1 ms each). Two failures:

- **Loss of resolution at the low end.** Below 1 ms everything collapses to the first bucket — fine for p99 reporting, terrible for any sub-millisecond service.
- **Loss of resolution at the high end.** A 10-second outlier sits in a 1-ms bucket — accurate, but uses 10,000 buckets to record one event.

You want **constant relative precision** across many orders of magnitude.

### 5.2 How HDR Histogram Works

Gil Tene's HdrHistogram (High Dynamic Range Histogram) is a bucketing scheme with constant *relative* precision: every value is recorded with at most a configurable percentage error (e.g., 0.1% or 1%) regardless of magnitude.

Mechanics:

- Configure a range (e.g., 1 µs to 1 hour) and a precision in significant digits (typically 2 or 3).
- The histogram uses a **logarithmic-like bucket structure**: a sequence of linear sub-buckets, each twice the range of the previous, so that any value within the range falls into a bucket with bounded relative error.
- Memory is small (tens of KB) for ranges spanning 9+ orders of magnitude at 3 significant digits.
- Two histograms can be **merged** by adding their bucket counts — essential for distributed aggregation.
- Percentile queries are O(buckets), not O(samples).

Critically, HDR Histogram has **no quantile estimation error** for the recorded distribution — only the configured relative precision per sample. Unlike t-digest or DDSketch (which trade exactness for streaming bounds), HDR is exact within precision. That's why it's the default in JMH, wrk2, Cassandra metrics, Aeron, and many JVM observability stacks.

### 5.3 Using HDR Histogram in Code

**Java (HdrHistogram library):**

```java
import org.HdrHistogram.Histogram;

// Range: 1 microsecond to 1 hour, 3 significant digits
Histogram histogram = new Histogram(1L, 3_600_000_000L, 3);

long start = System.nanoTime();
processRequest();
long elapsedMicros = (System.nanoTime() - start) / 1_000;
histogram.recordValue(elapsedMicros);

// Query
System.out.println("p50:   " + histogram.getValueAtPercentile(50.0));
System.out.println("p99:   " + histogram.getValueAtPercentile(99.0));
System.out.println("p99.9: " + histogram.getValueAtPercentile(99.9));
System.out.println("max:   " + histogram.getMaxValue());
```

**TypeScript (`hdr-histogram-js`):**

```typescript
import { build } from 'hdr-histogram-js';

const histogram = build({
  lowestDiscernibleValue: 1,
  highestTrackableValue: 60_000_000, // 60 seconds in microseconds
  numberOfSignificantValueDigits: 3,
});

const start = process.hrtime.bigint();
await handleRequest();
const elapsedMicros = Number((process.hrtime.bigint() - start) / 1000n);
histogram.recordValue(elapsedMicros);

console.log({
  p50: histogram.getValueAtPercentile(50),
  p99: histogram.getValueAtPercentile(99),
  p999: histogram.getValueAtPercentile(99.9),
  max: histogram.maxValue,
});
```

For Prometheus exposition specifically: **use native histograms (Prometheus 2.40+) where available**, or pre-compute percentiles client-side from an HDR Histogram and expose them as gauges. Do **not** rely on Prometheus' classic `histogram_quantile()` over coarse buckets for latency SLO reporting — its accuracy at p99/p99.9 is poor unless buckets are densely placed at the tail.

---

## 6. Coordinated Omission — The Bug That Hides Tails

### 6.1 What Coordinated Omission Is

Coordinated omission (Gil Tene's term, from his "How NOT to Measure Latency" talk) is a measurement bug where the load generator stops sending requests *while the system under test is stuck*, then resumes once it recovers — and only records the response times of the requests it actually sent. The hung period is invisible in the histogram.

Concrete scenario:

- Load tool intends 1,000 req/s (one request every 1 ms).
- Under steady state, response times are 1 ms.
- The server stalls for 100 ms (a GC pause).
- A naive load tool sends one request, waits for the response (which takes 100 ms because the server is paused), records 100 ms, then sends the next request.
- The 99 requests that *would have been sent* during the stall are silently dropped from the schedule.

The recorded distribution shows one 100 ms outlier and 999 1-ms responses. Percentiles say p99 = 1 ms. Reality: 99 requests would have suffered cumulative 1-99 ms additional wait time, and the *user* with the unfortunate request placement saw far worse.

### 6.2 How Tools Cause It

Coordinated omission is the default behavior of **closed-loop** load generators (a fixed pool of "virtual users" each issuing requests serially). JMeter, Gatling in default mode, ApacheBench (`ab`), and many homegrown scripts behave this way.

A closed-loop client by construction throttles its own send rate when the server slows down, because each "user" is blocked waiting for its response before issuing the next request. The arrival rate at the server is no longer Poisson — it is a feedback loop that hides the very stalls you're trying to measure.

See [Load Testing Methodology](06-load-testing-methodology.md) for the open vs closed model distinction in detail. The short version: load testing tools that report p99 from a closed-loop run are reporting a number that systematically under-states tail latency.

### 6.3 The Fix — Intended Start Time

The fix is to record latency relative to the **intended** request start time, not the actual one. Two approaches:

1. **Open-loop generator.** Send requests on a schedule (e.g., Poisson arrivals at λ requests/sec) regardless of whether prior requests have completed. Record per-request `response_time = completion_time − scheduled_start_time`. Tools: `wrk2`, `k6` with arrival-rate executor, Gatling open-injection profile.

2. **HDR Histogram with `recordValueWithExpectedInterval()`.** When the application records a slow event, HdrHistogram retroactively backfills missing samples that *would have* been issued during the stall, using the expected interval. This corrects an already-collected closed-loop trace.

```java
// Server-side: record request latency, but if it took longer than 1 ms (the
// expected interarrival), backfill the omitted samples for honest percentiles.
histogram.recordValueWithExpectedInterval(elapsedMicros, /* expectedIntervalMicros */ 1000);
```

`wrk2` does this internally and is the reason its tail percentiles match production behavior far better than `wrk` or `ab`.

---

## 7. Throughput Curves and the Knee

A typical throughput-vs-load curve looks like:

```text
throughput
   ^
   |                     ____________
   |                ___/              ─── ─── ─── (saturation)
   |             _/                            ─── (degradation)
   |          _/
   |        /
   |      /
   |    / <-- linear region
   |   /
   |  /
   | /
   +─────────────────────────────────> offered load (req/s)
                                  ^
                              the knee
```

- **Linear region.** Throughput tracks load 1:1. Latency is dominated by service time. This is where you want to operate.
- **Knee.** Resource saturation begins. Latency starts to rise even though throughput is still climbing.
- **Saturation plateau.** Throughput stops climbing. Latency rises sharply. Queues fill.
- **Degradation cliff.** Beyond saturation, throughput often *drops* due to context-switch overhead, lock contention, and GC death spirals. The system serves *fewer* requests as load increases.

The knee is not a fixed offered-load number — it depends on workload mix, cache state, time of day, and who else is running on the box. Capacity planning ([Tier 1 doc 7](07-capacity-planning.md)) treats the knee as a moving target and budgets headroom accordingly.

The relevant operational insight: **throughput at the knee is not your design capacity.** Plan for saturation at 50–70% of measured peak throughput so that p99 stays stable and you have headroom for failure-mode capacity.

---

## 8. What to Track in Production

Minimum metrics, per service, per endpoint:

| Metric | Why |
|--------|-----|
| Requests per second (RPS) | Throughput; correlate with latency to find the knee |
| Errors per second / error rate | Goodput. A latency drop with rising errors is not good news |
| p50 latency | Typical user experience |
| p95 latency | "Most users" SLO threshold |
| p99 latency | The number your alerting fires on |
| p99.9 latency | Tail health; sensitive to GC and infrastructure |
| Active concurrency (in-flight requests) | `L` in Little's Law; correlate with utilization |

Plus per-resource (USE method, [Doc 4](04-use-and-red-applied.md)): CPU utilization, memory pressure, disk queue depth, network saturation.

Avoid these traps:

- **Reporting only the mean** (or worse, only the p99 mean across pods). The mean of percentiles is not a percentile.
- **Computing percentiles by averaging time-bucketed percentiles.** You cannot average percentiles. See [Tail Latency](03-tail-latency-and-p99.md#why-you-cannot-average-percentiles) for why.
- **Bucket configurations that put p99 in the open-ended tail bucket.** Prometheus default buckets often place 200ms and 1s in the same bucket; your p99 is then between 200ms and 1s with no further resolution.
- **Histogram aggregation that re-quantizes.** If you have HDR data, ship it merged, not re-bucketed.

---

## Related

- [Little's Law and Queueing Theory](02-littles-law-and-queueing-theory.md)
- [Tail Latency and p99](03-tail-latency-and-p99.md)
- [USE and RED Applied](04-use-and-red-applied.md)
- [Load Testing Methodology](06-load-testing-methodology.md)
- [System Design — Performance Budgets and Latency](../../system-design/performance-observability/performance-budgets-and-latency.md)
- [System Design — SLA/SLO/SLI](../../system-design/foundations/sla-slo-sli-and-availability.md)
- [TypeScript — Node Profiling and Debugging](../../typescript/production/profiling-and-debugging.md)
- [Java — JVM GC Pause Diagnosis](../../java/jvm-gc/pause-diagnosis.md)

---

## References

- **Gil Tene — "How NOT to Measure Latency"** — the canonical talk on coordinated omission. https://www.youtube.com/watch?v=lJ8ydIuPFeU and slides at https://www.azul.com/presentation/how-not-to-measure-latency/
- **HdrHistogram** — Gil Tene's reference implementation and rationale. https://hdrhistogram.org/ and https://github.com/HdrHistogram/HdrHistogram
- **Brendan Gregg — "Systems Performance: Enterprise and the Cloud", 2nd ed., Pearson, 2020.** Chapter 2 (Methodologies) and Chapter 12 (Benchmarking) are the rigor floor. https://www.brendangregg.com/systems-performance-2nd-edition-book.html
- **Brendan Gregg — "Latency Heat Maps"** — visualizing distributions over time. https://www.brendangregg.com/HeatMaps/latency.html
- **Jeffrey Dean and Luiz André Barroso — "The Tail at Scale", CACM 56(2), 2013.** https://research.google/pubs/the-tail-at-scale/ — sets up why tail percentiles dominate fan-out systems.
- **Prometheus — Histograms and Summaries.** https://prometheus.io/docs/practices/histograms/ — and native histograms: https://prometheus.io/docs/concepts/metric_types/#histogram
- **Theo Schlossnagle — "The Problem with Percentiles".** https://www.circonus.com/2015/02/problem-percentiles-aggregating/
- **DDSketch (Datadog) and t-digest (Ted Dunning)** — alternative streaming quantile sketches with bounded relative error. https://www.datadoghq.com/blog/engineering/computing-accurate-percentiles-with-ddsketch/ and https://github.com/tdunning/t-digest
- **Martin Thompson — Mechanical Sympathy blog.** https://mechanical-sympathy.blogspot.com/ — practical latency measurement on the JVM.
- **wrk2** — Gil Tene's fork of wrk with corrected coordinated-omission handling. https://github.com/giltene/wrk2
