---
title: "Tail Latency & Percentiles — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, performance, latency, percentiles, p99, tail-latency]
---

# Tail Latency & Percentiles — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `performance` `latency` `percentiles` `p99` `tail-latency`

---

## Table of Contents
1. [How to use this doc](#how-to-use-this-doc)
2. [Easy (warm-up traps)](#1-easy-warm-up-traps)
3. [Subtle (review-passers)](#2-subtle-review-passers)
4. [Senior trap (production-only failures)](#3-senior-trap-production-only-failures)
5. [Solutions](#4-solutions)
6. [Related](#related)
7. [References](#references)

---

## Summary

Tail-latency bugs are insidious because the data itself is lying. The load test under-reports because it sleeps when the server stalls; the dashboard under-reports because percentiles get averaged across instances; the alert never fires because buckets are too coarse. Then production fans out 20 ways and the user sees the slowest of every dependency. This doc collects 24 bugs covering coordinated omission, percentile aggregation, fan-out amplification, GC tail coupling, queueing math, retry/timeout pathologies, and infrastructure tail sources. Practice cadence: aim to spot the failure mode in under 60 seconds before opening the hint. Sources lean on Gil Tene's coordinated-omission writeups, Dean & Barroso's "Tail at Scale", Prometheus histogram docs, Marc Brooker's queueing posts, and real load-tester (wrk2 / k6) docs.

## How to use this doc

- Try to spot the bug before opening `<details>`.
- The hint inside `<details>` is one line; the full root-cause + fix is in §4 Solutions, keyed by bug number.
- Difficulty sections are cumulative: don't skip Easy unless you've already nailed those traps in code review.

---

## 1. Easy (warm-up traps)

### Bug 1 — `wrk` for latency benchmarks
```bash
# benchmark.sh — measuring p99 of /api/checkout under load
wrk -t8 -c200 -d60s --latency https://api.example.com/api/checkout
```
<details><summary>Hint</summary>
What does <code>wrk</code> do when the server takes 5 s to respond?
</details>

### Bug 2 — Averaging p99 across replicas
```promql
# alert: p99 latency too high
avg(http_request_duration_seconds{quantile="0.99"}) by (service) > 0.5
```
<details><summary>Hint</summary>
What math operation are percentiles closed under?
</details>

### Bug 3 — Reporting the mean
```go
// after a load test
fmt.Printf("avg latency = %v\n", totalLatency/time.Duration(numRequests))
```
<details><summary>Hint</summary>
A bimodal distribution and a unimodal distribution can share a mean.
</details>

### Bug 4 — Histogram buckets stop at 1 s
```go
var reqLatency = prometheus.NewHistogramVec(prometheus.HistogramOpts{
    Name:    "http_request_duration_seconds",
    Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1},
}, []string{"route"})
```
<details><summary>Hint</summary>
Where does <code>histogram_quantile</code> place every observation above the last finite bucket?
</details>

### Bug 5 — Synchronous logging on the hot path
```java
@GetMapping("/order/{id}")
public Order get(@PathVariable String id) {
    log.info("loading order id={} caller={} headers={}", id, caller(), allHeaders());
    return orderService.load(id);
}
```
<details><summary>Hint</summary>
Where does that log line go, and does it block when the disk is slow?
</details>

### Bug 6 — k6 default scenario
```js
// loadtest.js — k6 script
export const options = { vus: 200, duration: '5m' };
export default function () { http.get('https://api.example.com/feed'); }
```
<details><summary>Hint</summary>
Closed-model VUs back off when the server slows down.
</details>

### Bug 7 — Timing from receive on the server
```go
func handle(w http.ResponseWriter, r *http.Request) {
    start := time.Now()       // measured here
    process(r)
    metrics.Observe(time.Since(start))
}
```
<details><summary>Hint</summary>
What about the time the request spent in the accept queue?
</details>

---

## 2. Subtle (review-passers)

### Bug 8 — Connection pool sized to p50
```yaml
# hikari config — service handles 200 rps, mean query 10 ms
spring:
  datasource:
    hikari:
      maximum-pool-size: 4
      connection-timeout: 30000
```
<details><summary>Hint</summary>
Little's Law on a burst, not on the mean.
</details>

### Bug 9 — Fan-out without budget
```java
// gateway: assemble user dashboard
CompletableFuture<Profile>     a = profileClient.get(userId);
CompletableFuture<List<Order>> b = ordersClient.list(userId);
CompletableFuture<Feed>        c = feedClient.get(userId);
// ... 10 dependencies total
return CompletableFuture.allOf(a, b, c, /* ... */).thenApply(...);
```
<details><summary>Hint</summary>
If each backend has p99 = 10 ms, what is the gateway p99?
</details>

### Bug 10 — Retries on every 5xx
```java
@Retryable(maxAttempts = 4, backoff = @Backoff(delay = 100))
public Quote fetch(String sym) { return http.get("/quote/" + sym); }
```
<details><summary>Hint</summary>
What does the dependency see during a partial failure?
</details>

### Bug 11 — Child timeout exceeds parent
```java
// parent SLO: 200 ms
WebClient.create().get().uri("/inner")
    .httpRequest(rq -> rq.responseTimeout(Duration.ofSeconds(5)))
    .retrieve();
```
<details><summary>Hint</summary>
The parent will already have given up.
</details>

### Bug 12 — `histogram_quantile` on raw counters
```promql
# p99 over the last 5 minutes
histogram_quantile(0.99, http_request_duration_seconds_bucket)
```
<details><summary>Hint</summary>
Cumulative since process start versus rate over a window.
</details>

### Bug 13 — Alert on instance-level p99
```promql
max(histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket[5m]))) by (instance) > 0.5
```
<details><summary>Hint</summary>
One JVM doing a stop-the-world will fire this every deploy.
</details>

### Bug 14 — Hedge without cancellation
```java
CompletableFuture<Resp> primary = client.call(req);
CompletableFuture<Resp> hedge   = scheduler.delay(20, ms)
    .thenCompose(__ -> client.call(req));
return primary.applyToEither(hedge, Function.identity());
```
<details><summary>Hint</summary>
The server keeps doing what work, exactly?
</details>

### Bug 15 — Coordinated retry storm
```java
catch (ServerBusy e) {
    Thread.sleep(1000);                 // every client backs off the same amount
    return client.call(req);
}
```
<details><summary>Hint</summary>
A million clients re-converge on the same wall-clock instant.
</details>

### Bug 16 — Lock-protected hot map
```java
private final Map<String, Quote> cache = new HashMap<>();
public synchronized Quote get(String sym) {
    return cache.computeIfAbsent(sym, this::loadFromUpstream);
}
```
<details><summary>Hint</summary>
Holding a lock across an I/O call.
</details>

### Bug 17 — Load shedding only at the edge
```yaml
# nginx in front of api drops at 5k rps; api -> db has no shed
nginx:
  limit_req_zone: rate=5000r/s
api:
  db_pool: 50
```
<details><summary>Hint</summary>
Where does the queue actually live now?
</details>

### Bug 18 — TCP_NODELAY off + small writes
```go
conn.SetNoDelay(false)            // Nagle on
for _, chunk := range smallChunks {
    conn.Write(chunk)             // 40 byte each, RPC framed
}
```
<details><summary>Hint</summary>
Nagle plus delayed-ACK on the peer.
</details>

---

## 3. Senior trap (production-only failures)

### Bug 19 — GC pause coupled to allocation rate
```java
// hot path allocates a fresh DTO graph per request
public ResponseDTO map(Entity e) {
    return ResponseDTO.builder()
        .lines(e.getLines().stream().map(this::mapLine).collect(toList()))
        .meta(new Meta(e.getId(), Instant.now(), new HashMap<>(e.getAttrs())))
        .build();
}
```
<details><summary>Hint</summary>
G1 young-gen frequency scales with allocation rate; pause time scales with survivor copy.
</details>

### Bug 20 — Single-CPU saturation on multi-core
```bash
# pinned worker on a 16-vCPU box
taskset -c 0 ./reactor-server
```
<details><summary>Hint</summary>
Your CPU dashboard reads 6 % — and yet.
</details>

### Bug 21 — Background compaction stealing tail
```yaml
# rocksdb / cassandra defaults left alone
compaction_throughput_mb_per_sec: 0   # unthrottled
```
<details><summary>Hint</summary>
The tail goes saw-tooth on a 6 hour cycle.
</details>

### Bug 22 — Cold-cache p99 dominates after deploy
```yaml
# k8s rolling update
strategy:
  type: RollingUpdate
  rollingUpdate: { maxSurge: 25%, maxUnavailable: 0 }
readinessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 5
```
<details><summary>Hint</summary>
Healthy != warm.
</details>

### Bug 23 — `histogram_quantile` over coarse buckets at high traffic
```go
// SLO is p99 < 50 ms
Buckets: prometheus.ExponentialBuckets(0.001, 4, 6) // 1ms, 4, 16, 64, 256, 1024 ms
```
<details><summary>Hint</summary>
Linear interpolation inside the bucket containing p99.
</details>

### Bug 24 — TCP retransmit timeout dominates the tail
```bash
# default linux: tcp_retries2=15, initial RTO ~1s, exponential backoff
sysctl net.ipv4.tcp_retries2
# 15
```
<details><summary>Hint</summary>
A single dropped SYN-ACK to a dead peer.
</details>

---

## 4. Solutions

### Bug 1 — `wrk` for latency benchmarks
**Root cause:** `wrk` uses a closed model: each connection waits for the previous response before issuing the next request. When the server stalls, `wrk` stops generating load — the latency numbers are not what users would see, they are what `wrk` saw *after* coordinating with the server's slowness. Gil Tene named this "coordinated omission". Reported p99 can understate real p99 by 1000× or more.
**Fix:** use `wrk2` (or `k6` with arrival-rate scenarios, or `Gatling` with constant request rate) and pin a target rate independent of server response time.
```bash
wrk2 -t8 -c200 -d60s -R 5000 --latency https://api.example.com/api/checkout
```
**Reference:** Gil Tene, "How NOT to Measure Latency" (HdrHistogram talk); ScyllaDB writeup of coordinated omission noting YCSB measured p99 of ~240 µs vs corrected ~660 ms.

### Bug 2 — Averaging p99 across replicas
**Root cause:** percentiles are not linear; `avg(p99_a, p99_b)` is meaningless. If replica A's p99 is 50 ms and B's is 500 ms, the cluster p99 is not 275 ms.
**Fix:** aggregate the histogram, then compute the quantile.
```promql
histogram_quantile(0.99,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
```
**Reference:** Prometheus histograms doc — "aggregating the precomputed quantiles from a summary rarely makes sense"; same constraint applies to averaging classic-histogram quantiles.

### Bug 3 — Reporting the mean
**Root cause:** the mean hides bimodality. A service that serves 99 % of requests in 5 ms and 1 % in 5 s has a mean of ~55 ms — looks fine, but 1 % of users see 5 s.
**Fix:** report a distribution: p50, p90, p99, p99.9, max. Plot a histogram or HdrHistogram percentile distribution.
**Reference:** Gil Tene, HdrHistogram README ("Latency does not live on a bell curve"); Brendan Gregg, "Frequency Trails" technique for visualizing tails.

### Bug 4 — Histogram buckets stop at 1 s
**Root cause:** every request that took longer than 1 s is lumped into `+Inf`. `histogram_quantile` then linearly interpolates inside the last finite bucket and silently caps p99 reports at 1 s.
**Fix:** extend buckets past your timeout. Match buckets to your SLO and to the worst plausible latency.
```go
Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10, 30}
```
**Reference:** Prometheus histograms doc, bucket-choice guidance; native histograms remove this trap entirely.

### Bug 5 — Synchronous logging on the hot path
**Root cause:** the default Logback/Log4j2 file appender blocks on a stalled disk; one slow `fsync` on a noisy neighbor pegs your p99. JSON-rendering a header map allocates and stalls under GC.
**Fix:** use an async appender with a bounded queue and a defined drop policy; render lazily.
```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
  <queueSize>8192</queueSize>
  <discardingThreshold>0</discardingThreshold>
  <neverBlock>true</neverBlock>
</appender>
```
**Reference:** Logback `AsyncAppender` docs; Brendan Gregg, "Off-CPU analysis" — disk waits show up in off-CPU time, not CPU profiles.

### Bug 6 — k6 default scenario
**Root cause:** `vus + duration` is a closed model — same coordinated-omission failure as `wrk`. VUs back off when the server slows down, so observed latency is bounded by the speed at which the server can serve those VUs.
**Fix:** use an arrival-rate scenario with `preAllocatedVUs` large enough not to throttle.
```js
export const options = {
  scenarios: {
    contacts: {
      executor: 'constant-arrival-rate',
      rate: 5000, timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 1000,
    },
  },
};
```
**Reference:** k6 docs — "Open vs closed models" (grafana.com/docs/k6).

### Bug 7 — Timing from receive on the server
**Root cause:** server-side timer starts after the kernel's accept queue has handed you the request. Connection-establishment time, TLS handshake stalls, and accept-queue waits are invisible. Under load this is exactly where the tail lives.
**Fix:** measure on the client wall clock, or instrument the load balancer's request-start timestamp (e.g. envoy `start_time`, NGINX `$request_time` includes receive time but not pre-accept queueing — be aware of the boundary).
**Reference:** Gil Tene's coordinated-omission writeup — "service time is not response time"; envoy access-log fields documentation.

### Bug 8 — Connection pool sized to p50
**Root cause:** at 200 rps × 10 ms mean, Little's Law says you need 2 in-flight on average — but a 100 ms p99 burst pushes 20 in-flight. Pool size of 4 → requests block on `getConnection()`, and that wait isn't query time, it's pure queueing. p99 explodes the moment traffic deviates from the mean.
**Fix:** size for burst (peak rps × p99 latency, plus headroom) and watch `hikaricp_pending` and `hikaricp_active` ratio.
```yaml
maximum-pool-size: 30
connection-timeout: 2000
```
**Reference:** HikariCP pool-sizing wiki; Marc Brooker, "It's about queue length" (brooker.co.za) on queue-depth as the real signal.

### Bug 9 — Fan-out without budget
**Root cause:** if each of 10 backends has independent p99 = 10 ms (so each backend is below its SLO 99 % of the time), the probability all 10 stay below 10 ms is `0.99^10 ≈ 0.904` — meaning ~10 % of gateway requests have at least one slow dependency. The gateway p99 is closer to the *backend p99.9*. Dean & Barroso called this "tail at scale".
**Fix:** target backend p99.9, not p99; use hedged requests with cancellation; per-call deadlines that compose.
**Reference:** Dean & Barroso, "The Tail at Scale" (research.google).

### Bug 10 — Retries on every 5xx
**Root cause:** during a partial outage every client multiplies its load by `maxAttempts`. The dependency, already overloaded, sees 4× traffic and fails harder. Retries also extend the per-request latency: p99 becomes p99 of attempt-1 + attempt-2 + ...
**Fix:** retry budgets (token bucket capping retry rate to a fraction of base traffic), retry only on idempotent + transient codes, jittered exponential backoff.
**Reference:** Marc Brooker, "Exponential Backoff and Jitter" (AWS Architecture Blog); Google SRE book, ch. 22 "Addressing Cascading Failures".

### Bug 11 — Child timeout exceeds parent
**Root cause:** parent SLO is 200 ms; child timeout is 5 s. The parent's request will be cancelled (or its caller will time out) long before the child gives up — the child keeps doing work nobody is waiting for, burning CPU and connections, and amplifying queue depth.
**Fix:** propagate a deadline. Each hop subtracts elapsed time from a deadline header; child timeout = `min(local_max, remaining_budget)`.
```java
Duration budget = Deadline.fromHeaders(req).remaining();
WebClient.create().get().uri("/inner")
    .httpRequest(rq -> rq.responseTimeout(budget.minusMillis(20)))
    .retrieve();
```
**Reference:** gRPC deadline propagation docs; Cindy Sridharan, "Deadlines, context cancellation and request hedging" (medium @copyconstruct).

### Bug 12 — `histogram_quantile` on raw counters
**Root cause:** the `_bucket` series is a counter monotonic since process start. Without `rate()` or `increase()`, `histogram_quantile` sees lifetime distribution and never reflects current behavior.
**Fix:**
```promql
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```
**Reference:** Prometheus histograms doc, "Quantiles" section.

### Bug 13 — Alert on instance-level p99
**Root cause:** `by (instance)` produces one p99 per pod. A single GC pause on one pod → that pod's p99 spikes → alert fires. But user-visible p99 is the cluster aggregate. Worse, this also has the averaging problem if you `avg` across instances.
**Fix:** aggregate buckets first.
```promql
histogram_quantile(0.99,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))) > 0.5
```
**Reference:** Prometheus histograms doc, "Aggregating histograms"; Tyler Treat, "Latency Driven Performance" (bravenewgeek.com).

### Bug 14 — Hedge without cancellation
**Root cause:** primary and hedge both run to completion. Server-side work doubles for any request slower than 20 ms — which under load is a lot of them. The dependency oscillates between healthy and overloaded as hedging amplifies the load it tries to escape.
**Fix:** Dean & Barroso's *tied request*: send to two replicas with cross-cancellation; the server cancels its sibling when one starts execution.
**Reference:** Dean & Barroso, "The Tail at Scale" — tied requests section.

### Bug 15 — Coordinated retry storm
**Root cause:** every client sleeps the same amount and re-converges on the wall clock. The dependency sees a thundering-herd retry exactly `delay` after each failure. Fixed sleeps without jitter turn N independent clients into one synchronized client.
**Fix:** full jitter: `sleep = random(0, base * 2^attempt)`.
**Reference:** Marc Brooker, "Exponential Backoff and Jitter" (AWS Architecture Blog).

### Bug 16 — Lock-protected hot map
**Root cause:** `synchronized` plus `loadFromUpstream` (an HTTP call) means every miss serializes every other request behind the lock for the duration of an upstream call. A slow upstream → request pile-up → p99 of the cache becomes p99 of the upstream times queue depth.
**Fix:** use a non-blocking concurrent cache (Caffeine `AsyncLoadingCache`) so loads run off-thread and concurrent misses for the same key dedupe via a `CompletableFuture`.
**Reference:** Caffeine wiki, `AsyncLoadingCache`; Brian Goetz, *Java Concurrency in Practice* — "Don't hold locks during I/O".

### Bug 17 — Load shedding only at the edge
**Root cause:** NGINX caps inbound rps but the API has 50 db connections and the db has finite capacity. Once db is saturated, requests queue *inside* the API — heap grows, GC pauses lengthen, healthy edges keep arriving and all of them stall together.
**Fix:** shed at every tier. The API needs its own concurrency limit (Resilience4j bulkhead, semaphore, or adaptive concurrency à la Netflix concurrency-limits) and a fast-fail behavior when over budget.
**Reference:** Netflix Tech Blog, "Performance Under Load" (concurrency-limits library); Marc Brooker, "Fairness in queueing".

### Bug 18 — TCP_NODELAY off + small writes
**Root cause:** Nagle's algorithm (RFC 896) coalesces small sends; the receiver's delayed-ACK (RFC 1122) waits up to 200 ms before ACKing. Together they cause a deterministic ~40 ms (or up to 500 ms on some stacks) stall on every small RPC write that doesn't fill an MSS. Tail latency picks up the mode of the stall.
**Fix:** for RPC framing, set `TCP_NODELAY` and frame-coalesce in user space.
**Reference:** RFC 896 (Nagle); Heroku engineering blog historical post-mortems on Nagle/delayed-ACK interactions; gRPC sets `TCP_NODELAY` by default for this reason.

### Bug 19 — GC pause coupled to allocation rate
**Root cause:** every request allocates a fresh object graph. Higher rps → faster eden fill → more frequent young-gen pauses; survivor objects copied in the pause cause variable pause times. Tail latency tracks request rate, which looks like a scaling problem but is a GC problem.
**Fix:** reduce allocation on hot path (reuse mappers, avoid `Instant.now()` per request if redundant, prefer arrays over wrapper streams), tune G1 with `MaxGCPauseMillis`, or move to ZGC / Shenandoah for sub-ms pauses.
**Reference:** JEP 439 (Generational ZGC); JEP 379 (Shenandoah); Brendan Gregg, "Java in Flames" on GC tail diagnosis.

### Bug 20 — Single-CPU saturation on multi-core
**Root cause:** pinning to one CPU caps throughput at one core's worth of work. Once that core hits 100 % the run-queue grows and all in-flight requests share queue time — p99 climbs while CPU dashboards (averaged across 16 cores) read ~6 %, showing "stairstep" tail behavior under bursts.
**Fix:** drop the pin (or pin to a CPU set that matches the parallelism), watch `mpstat -P ALL` not `top`, alert on per-CPU saturation.
**Reference:** Brendan Gregg, USE method (brendangregg.com/usemethod.html); `mpstat` man page.

### Bug 21 — Background compaction stealing tail
**Root cause:** unthrottled compaction periodically saturates disk I/O and CPU; foreground reads/writes wait behind compaction's I/O. The tail develops a sawtooth: clean for hours, then a 10× spike for the duration of the compaction.
**Fix:** throttle background work to leave foreground headroom, schedule heavy compaction during low traffic, or use rate-limited compactors (RocksDB `rate_limiter_bytes_per_sec`, Cassandra `compaction_throughput_mb_per_sec` non-zero).
**Reference:** RocksDB tuning guide, "Rate Limiter" section (github.com/facebook/rocksdb/wiki); Discord engineering blog on Cassandra tail-latency tuning ("How Discord Stores Trillions of Messages").

### Bug 22 — Cold-cache p99 dominates after deploy
**Root cause:** readiness only checks `/healthz` — the new pod accepts traffic before its caches, JIT, and connection pools are warm. With 25 % surge during a rolling deploy, 25 % of traffic hits cold pods → p99 spikes for the duration of every deploy.
**Fix:** warmup phase before readiness goes green: pre-load caches, run a self-test that triggers JIT C2 compilation on hot paths, prime the connection pool. Use `startupProbe` separately from `readinessProbe`. Some teams ship synthetic warmup traffic via an init job.
**Reference:** Kubernetes docs, "Configure Liveness, Readiness and Startup Probes"; LinkedIn engineering, "Fast warmup of JVM services".

### Bug 23 — `histogram_quantile` over coarse buckets at high traffic
**Root cause:** with buckets at `[1ms, 4, 16, 64, 256, 1024 ms]` and an SLO of 50 ms, p99 always falls inside the `(16, 64]` bucket. `histogram_quantile` linearly interpolates within that bucket, so the reported p99 is essentially "somewhere between 16 and 64 ms" — too coarse to drive the SLO.
**Fix:** add buckets straddling the SLO, or use Prometheus native histograms which give exponential buckets with configurable resolution and bounded relative error.
```go
Buckets: []float64{.001, .005, .01, .025, .04, .05, .075, .1, .25, .5, 1, 2.5, 10}
```
**Reference:** Prometheus histograms doc, "Errors of quantile estimation"; native histograms documentation.

### Bug 24 — TCP retransmit timeout dominates the tail
**Root cause:** Linux `tcp_retries2` defaults to 15, with exponential backoff from initial RTO (~200 ms-1 s). When a peer dies mid-connection, the kernel retries the segment for ~15 minutes before reporting the socket dead. Application-layer timeouts that don't have their own connect/read deadlines inherit this. Tail looks like multi-second hangs that recover.
**Fix:** application-layer connect and read timeouts shorter than `tcp_retries2`'s effective duration; tune `tcp_user_timeout` (per-socket override) or rely on libraries that set it. For load balancers, configure aggressive idle and connect timeouts.
**Reference:** Linux `tcp(7)` man page (`TCP_USER_TIMEOUT`, `tcp_retries2`); Cloudflare blog, "When TCP sockets refuse to die" (blog.cloudflare.com).

---

## Related

- [Tail Latency and p99 — Why the 99th Percentile Owns Your User Experience](03-tail-latency-and-p99.md)
- [Latency, Throughput, and Percentiles](01-latency-throughput-percentiles.md)
- [Little's Law and Queueing Theory](02-littles-law-and-queueing-theory.md)
- [Load Testing Methodology](06-load-testing-methodology.md)
- [USE and RED — Applied](04-use-and-red-applied.md)
- [Tracing Sampling — Bug Spotting](../../observability/fundamentals/tracing-sampling-bug-spotting.md)

## References

- Gil Tene, "How NOT to Measure Latency" — talk + slides; HdrHistogram README (github.com/HdrHistogram/HdrHistogram).
- Gil Tene, "Coordinated Omission" — original mailing-list writeup; ScyllaDB blog summary (scylladb.com).
- Jeff Dean & Luiz Barroso, "The Tail at Scale", *Communications of the ACM*, 56(2), 2013 (research.google/pubs/the-tail-at-scale).
- Prometheus, "Histograms and summaries" (prometheus.io/docs/practices/histograms).
- Prometheus, `histogram_quantile` reference (prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile).
- Marc Brooker, blog (brooker.co.za) — queueing length, exponential backoff and jitter.
- Brendan Gregg, USE method and off-CPU analysis (brendangregg.com).
- Cindy Sridharan, "Deadlines, context cancellation and request hedging" (medium @copyconstruct).
- Tyler Treat, "Latency Driven Performance" (bravenewgeek.com).
- wrk2 README (github.com/giltene/wrk2); k6 "Open vs closed models" (grafana.com/docs/k6); Gatling open-injection profile docs.
- RFC 896 (Nagle's algorithm); RFC 1122 §4.2.3.2 (delayed ACK); RFC 9293 (TCP).
- JEP 439 — Generational ZGC; JEP 379 — Shenandoah (openjdk.org/jeps/).
- Linux `tcp(7)` man page — `TCP_USER_TIMEOUT`, `tcp_retries2`.
- Cloudflare blog, "When TCP sockets refuse to die" (blog.cloudflare.com).
- Discord engineering, "How Discord Stores Trillions of Messages" (discord.com/blog).
- Kubernetes docs, "Configure Liveness, Readiness and Startup Probes" (kubernetes.io/docs).
- Netflix Tech Blog, "Performance Under Load" / `concurrency-limits` library (github.com/Netflix/concurrency-limits).
- Caffeine wiki, `AsyncLoadingCache` (github.com/ben-manes/caffeine/wiki).
- HikariCP pool-sizing guide (github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing).
- RocksDB tuning guide, rate limiter (github.com/facebook/rocksdb/wiki).
