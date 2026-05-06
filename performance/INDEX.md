# Performance Engineering Documentation Index — Learning Path

A backend-engineer-focused path through the discipline of performance engineering: how to measure latency and throughput honestly, reason about queueing and tail behavior, profile what is actually slow, run load tests that produce trustworthy numbers, and plan capacity that survives peaks and failures. Practical and rigorous — built on the work of Brendan Gregg, Gil Tene, Martin Thompson, Dean & Barroso, and Mor Harchol-Balter rather than vendor blog hype.

Cross-references to the [System Design learning path](../system-design/INDEX.md), the [Java learning path](../java/INDEX.md), the [TypeScript learning path](../typescript/INDEX.md), the [Go learning path](../golang/INDEX.md) (pprof and the GC pacer as concrete tools), the [Database learning path](../database/INDEX.md), the [Networking learning path](../networking/INDEX.md), the [Observability learning path](../observability/) (in progress), the [Operating Systems learning path](../operating-systems/INDEX.md) (in progress), the [Behavioral Interviews learning path](../behavioral-interviews/INDEX.md) (the interview-stage counterpart — STAR, story banking, and the question-by-category bank), and the [Web Scraping learning path](../web-scraping/INDEX.md) (concurrency vs throughput in crawler design, Little's Law applied to crawler queues) where topics overlap.

**Markers:** **★** = core must-learn (required mental models for every backend engineer working on systems with latency or throughput SLOs). **○** = supporting deep-dive (specialized profiling, distributed-system perf). Internalize all ★ before going deep on ○.

---

## Tier 1 — Fundamentals: Mental Models You Cannot Skip

The vocabulary, math, and methods that underlie everything else. If you cannot explain percentiles, Little's Law, tail amplification, USE/RED, flame graphs, open vs closed workloads, and headroom out loud, no amount of profiler output will save you.

1. [★ Latency, Throughput, Percentiles — How to Measure What Users Actually Feel](fundamentals/01-latency-throughput-percentiles.md) — latency vs response time vs service time, throughput vs goodput, p50/p90/p95/p99/p99.9, why averages lie, HDR Histogram, coordinated omission _(2026-05-03)_
2. [★ Little's Law and Queueing Theory — Sizing Pools, Predicting Saturation](fundamentals/02-littles-law-and-queueing-theory.md) — L = λW, connection/thread pool sizing, Kingman's formula, the 80% utilization wall, M/M/1 vs M/M/c, head-of-line blocking _(2026-05-03)_
3. [★ Tail Latency and p99 — Why the 99th Percentile Owns Your User Experience](fundamentals/03-tail-latency-and-p99.md) — Dean & Barroso "Tail at Scale", fan-out amplification, hedged/tied requests, percentile aggregation pitfalls, GC/HOL/TCP as tail sources _(2026-05-03)_
4. [★ USE and RED Applied — Dashboards, Alerts, and Drill-Down Workflows](fundamentals/04-use-and-red-applied.md) — RED for services (rate/errors/duration), USE for resources (utilization/saturation/errors), joining them in incident response, real dashboards _(2026-05-03)_
5. [★ Flame Graphs — On-CPU and Off-CPU Profiling](fundamentals/05-flame-graphs-cpu-and-off-cpu.md) — sampling profilers, on-CPU flame graphs, off-CPU (lock/IO) flame graphs, differential and icicle graphs, async-profiler, perf, 0x/clinic flame _(2026-05-03)_
6. [★ Load Testing Methodology — Open vs Closed Workloads, Trustworthy Numbers](fundamentals/06-load-testing-methodology.md) — open vs closed models (Schroeder/Wierman/Harchol-Balter), why JMeter underestimates latency, k6/wrk2/Gatling, ramp/soak/spike profiles, coordinated omission in tools _(2026-05-03)_
7. [★ Capacity Planning — Headroom, Forecasts, and Failure-Mode Capacity](fundamentals/07-capacity-planning.md) — back-of-envelope, peak-to-average ratio, the 50% headroom rule, forecasting, autoscaling lag, N+1/N+2, regional capacity, cost vs reliability _(2026-05-03)_

---

## Tier 2 — Profiling Toolchains _(planned)_

Hands-on with the actual tools. How to drive each profiler, read its output, and combine outputs to triangulate a bottleneck.

- ○ Linux `perf` — sampling, tracepoints, dynamic probes, and `perf script` to flame graphs _(planned)_
- ○ async-profiler for the JVM — wall-clock vs CPU vs allocation profiling, JFR integration, native stacks _(planned)_
- ○ Node.js profiling with 0x and clinic.js — flame graphs, doctor, bubbleprof for async, heap snapshots _(planned)_
- ○ eBPF and bcc/bpftrace — kernel-level visibility without recompiling, latency histograms, off-CPU tracing _(planned)_
- ○ Continuous profiling in production — Pyroscope, Grafana Phlare, Parca, sampling cost vs signal _(planned)_

---

## Tier 3 — JVM and V8 Performance Deep Dives _(planned)_

Runtime-specific behavior that dominates application performance. Cross-references existing JVM and V8 docs and adds the perf-engineering lens.

- ○ JVM GC tuning in anger — G1, ZGC, Shenandoah, allocation rate vs promotion rate, safe-point analysis _(planned)_
- ○ JIT compilation and inlining — tiered compilation, deoptimization, `-XX:+PrintCompilation`, Graal vs C2 _(planned)_
- ○ V8 hidden classes, inline caches, and deopt — keeping objects monomorphic, why polymorphic call sites slow you down _(planned)_
- ○ Node.js event loop saturation — libuv thread pool, blocking syscalls, microtask starvation _(planned)_
- ○ Virtual threads (Project Loom) and reactive — when each wins, pinning hazards, scheduler interactions _(planned)_

---

## Tier 4 — System-Level Optimization _(planned)_

Below the runtime: where the OS, hardware, and network meet your code. Mostly Linux-flavoured.

- ○ CPU caches and mechanical sympathy — cache lines, false sharing, NUMA, branch prediction, prefetching _(planned)_
- ○ Memory hierarchies and allocator behavior — jemalloc/tcmalloc/glibc, fragmentation, RSS vs working set _(planned)_
- ○ Disk and filesystem perf — `fio`, queue depth, write amplification, page cache, `O_DIRECT` _(planned)_
- ○ Network stack tuning — sysctl knobs that matter, NIC offloads, RPS/RFS, AF_XDP, kernel bypass when justified _(planned)_
- ○ Container and cgroup performance — CPU throttling, memory pressure, the noisy-neighbor reality _(planned)_

---

## Tier 5 — Distributed System Performance _(planned)_

Performance engineering once a single box is no longer the right mental model. Tail at scale, backpressure, and capacity coupled to availability.

- ○ Backpressure and admission control — load shedding, rate limiting, the right place to reject _(planned)_
- ○ Hedging, tied requests, and request reissue in production — when they help, when they amplify load _(planned)_
- ○ Caching for latency, not just throughput — cache stampede, request coalescing, negative caching _(planned)_
- ○ Distributed tracing for latency analysis — critical-path extraction, span hygiene, cardinality budgets _(planned)_
- ○ Performance regression detection — canary analysis, statistical significance, perf CI gates _(planned)_

---

## Quick Reference by Topic

### Fundamentals

- [Latency, Throughput, Percentiles](fundamentals/01-latency-throughput-percentiles.md)
- [Little's Law and Queueing Theory](fundamentals/02-littles-law-and-queueing-theory.md)
- [Tail Latency and p99](fundamentals/03-tail-latency-and-p99.md)
- [USE and RED Applied](fundamentals/04-use-and-red-applied.md)
- [Flame Graphs (CPU and Off-CPU)](fundamentals/05-flame-graphs-cpu-and-off-cpu.md)
- [Load Testing Methodology](fundamentals/06-load-testing-methodology.md)
- [Capacity Planning](fundamentals/07-capacity-planning.md)

### Profiling Toolchains _(planned)_

- Linux perf, async-profiler (JVM), 0x / clinic.js (Node), eBPF, continuous profiling

### Runtime Deep Dives _(planned)_

- JVM GC, JIT, V8 hidden classes, event loop saturation, virtual threads vs reactive

### System-Level _(planned)_

- CPU caches, allocators, disk/fs, network tuning, cgroups

### Distributed _(planned)_

- Backpressure, hedging, caching, tracing, perf CI

---

## Bug Spotting

Active-recall practice docs. Each presents 22+ broken snippets organized by difficulty (Easy / Subtle / Senior trap), with one-line `<details>` hints inline and full root-cause + fix in a Solutions section. Every bug cites a real reference (RFC, CVE, official-doc gotcha, postmortem, library issue). Use these to pressure-test concept knowledge after working through the tiers above.

- [Tail Latency & Percentiles — Bug Spotting](fundamentals/tail-latency-bug-spotting.md) ★ — _(2026-05-03)_

---

## Cross-Path References

- **System Design — Foundations:** [Back-of-Envelope Estimation](../system-design/foundations/back-of-envelope-estimation.md), [SLA/SLO/SLI](../system-design/foundations/sla-slo-sli-and-availability.md), [Non-Functional Requirements](../system-design/foundations/non-functional-requirements.md)
- **System Design — Performance/Observability:** [Performance Budgets and Latency](../system-design/performance-observability/performance-budgets-and-latency.md), [Capacity Planning and Load Testing](../system-design/performance-observability/capacity-planning-and-load-testing.md), [RED/USE/Golden Signals](../system-design/performance-observability/monitoring-red-use-golden-signals.md)
- **Java:** [JVM GC Concepts](../java/jvm-gc/concepts.md), [GC Pause Diagnosis](../java/jvm-gc/pause-diagnosis.md), [Distributed Tracing](../java/observability/distributed-tracing.md)
- **TypeScript:** [Event Loop Internals](../typescript/runtime/event-loop-internals.md), [V8 Engine Pipeline](../typescript/runtime/v8-engine-pipeline.md), [V8 Memory and GC](../typescript/runtime/v8-memory-and-gc.md), [Node Profiling and Debugging](../typescript/production/profiling-and-debugging.md)
- **Database:** [EXPLAIN ANALYZE Guide](../database/query-optimization/explain-analyze-guide.md), [Connection Management](../database/operations/connection-management.md), [Monitoring](../database/operations/monitoring.md)
- **Networking:** [TCP Deep Dive](../networking/transport/tcp-deep-dive.md), [Connection Pooling and Keep-Alive](../networking/network-programming/connection-pooling.md), [Async I/O Models](../networking/network-programming/async-io-models.md)
