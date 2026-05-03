---
title: "Go's Garbage Collector"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, runtime, gc, gogc, gomemlimit, performance]
---

# Go's Garbage Collector

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `runtime` `gc` `gogc` `gomemlimit` `performance`

---

## Table of Contents

1. [The high-level design: concurrent, non-generational, non-compacting tri-color mark-sweep](#1-the-high-level-design-concurrent-non-generational-non-compacting-tri-color-mark-sweep)
2. [Why a different GC than the JVM](#2-why-a-different-gc-than-the-jvm)
3. [The collection cycle: STW phases and concurrent mark](#3-the-collection-cycle-stw-phases-and-concurrent-mark)
4. [The pacer and `GOGC`](#4-the-pacer-and-gogc)
5. [`GOMEMLIMIT`: the soft memory limit](#5-gomemlimit-the-soft-memory-limit)
6. [STW pause times in production](#6-stw-pause-times-in-production)
7. [Observing the GC](#7-observing-the-gc)
8. [When the GC affects tail latency, and what to do](#8-when-the-gc-affects-tail-latency-and-what-to-do)
9. [Contrast with JVM and V8 collectors](#9-contrast-with-jvm-and-v8-collectors)

## Summary

Go's garbage collector is a **concurrent, non-generational, non-compacting, tri-color mark-sweep** collector tuned overwhelmingly for **low pause times** rather than peak throughput. Most of the marking work runs in parallel with your goroutines; only two short stop-the-world (STW) phases bracket each cycle, and on a healthy production service those pauses are typically well under a millisecond. The pacer drives the cycle off a single knob, `GOGC` (default 100, meaning "run a cycle when the live heap has grown by 100%"), and since Go 1.19 a second knob, `GOMEMLIMIT`, lets the runtime treat a memory ceiling as the primary trigger — useful in containers with hard cgroup limits.

This is a very different design from the JVM's G1 or ZFG collectors and from V8's generational Scavenger + Mark-Sweep-Compact split. Go can afford to skip generations because escape analysis (covered in [the next doc](02-escape-analysis-and-allocation.md)) already routes most short-lived data onto the stack, where it costs the GC nothing. The cost of this latency-first design is throughput: Go's GC does more work per byte of garbage than a moving generational collector would, and it does not compact, so the heap can fragment.

## 1. The high-level design: concurrent, non-generational, non-compacting tri-color mark-sweep

Four design choices, each with consequences:

**Concurrent.** The mark phase runs alongside the application. Mutator goroutines keep allocating and mutating while marking is in progress. The Go [GC guide](https://tip.golang.org/doc/gc-guide) phrases it: this approach "avoids making the length of any global application pauses proportional to the size of the heap." Heaps can grow without making pauses grow.

**Tri-color mark-sweep.** Every heap object is conceptually one of three colors: white (unvisited, candidate garbage), grey (visited, but its children not yet scanned), or black (visited and its children scanned). The collector sweeps grey to black, scanning pointers and turning newly-reached whites grey. When no greys remain, all whites are unreachable. **Write barriers** fire whenever the application stores a pointer during the mark phase, ensuring no black-points-to-white edge can hide a still-reachable object.

**Non-generational.** There is no "young generation" / "old generation" split. Every cycle scans every reachable object. This is unusual — most modern collectors are generational because the "weak generational hypothesis" (most objects die young) is empirically true. Go skips generations because, as Rick Hudson explains in his ["Getting to Go" keynote](https://go.dev/blog/ismmkeynote), Go's escape analysis already pushes most short-lived allocations onto the stack, weakening the hypothesis: by the time you're allocating on the heap, the object probably is going to live a while. The savings from a young generation would not pay for the always-on write barriers it requires.

**Non-compacting.** Objects do not move once allocated. There is no copying collector underneath, no relocation, no read barriers. The price is that the heap can fragment: Go uses **size-segregated spans** (similar to TCMalloc) to mitigate this — small objects of similar sizes share spans, so internal fragmentation is bounded — but external fragmentation across spans is real. The benefit is that pointer values in your code are stable; you never need read barriers, and CGo and `unsafe.Pointer` interop is straightforward.

```text
                        ┌─────────────────────────────────┐
                        │     concurrent mark phase       │
                        │  (runs alongside goroutines)    │
                        └────────────┬────────────────────┘
                                     │
   STW: mark setup ──→ mark (concurrent) ──→ STW: mark term ──→ sweep (concurrent) ──→ off
       (~µs)                                       (~µs)
```

## 2. Why a different GC than the JVM

The JVM is a throughput-first runtime by tradition. Generational collectors (Parallel, CMS historically, G1 today, ZGC and Shenandoah for low-latency niches) thrive on the workload pattern where most objects die young — common in enterprise Java where allocations are heap-bound by default and short-lived request-scoped data dominates. Generational GCs save CPU by scanning only the young generation most of the time and promoting survivors lazily. The cost is the **card table** plus **write barriers always on** to track inter-generational pointers.

Go inverts the tradeoff. The runtime team explicitly framed the goal as latency over throughput. Hudson's keynote argues that even a 99th-percentile pause SLO doesn't scale: if you call enough services per request, you need 99.99th-percentile or better, which means STW pauses must be sub-millisecond, not tens of milliseconds. That target is incompatible with a stop-the-world young-gen collection sweeping millions of objects.

The other half of the answer is **escape analysis** (see §6 of the [pointers doc](../fundamentals/07-pointers-methods-receivers.md) and the [next doc](02-escape-analysis-and-allocation.md)). Go's compiler is aggressive about leaving values on the stack when their lifetime is provably bounded by the function frame. JVM heap allocation is the default for every object (modulo escape analysis in HotSpot, which exists but is more conservative because of class-loading and reflection). When most ephemeral allocations never reach the heap in the first place, the case for a young generation evaporates.

The result is two genuinely different collectors solving overlapping problems with opposite priorities:

| | Go (post-1.5) | JVM G1 (default since JDK 9) | JVM ZGC (since JDK 15+) |
|---|---|---|---|
| Generational | no | yes (regions, but partitioned by age) | no (since JDK 21; previously yes) |
| Compacting | no | yes | yes |
| Read barriers | no | no | yes (load barrier) |
| Write barriers | only during mark phase | always on | yes |
| Typical STW | sub-ms | tens of ms | sub-ms |
| Heap size sweet spot | small to medium (<32 GB) | small to large | very large (TB-class) |

## 3. The collection cycle: STW phases and concurrent mark

A cycle has four phases. Two are STW; two are concurrent.

**Phase 1 — Mark setup (STW).** The runtime stops all goroutines just long enough to enable write barriers, scan the roots of currently-running stacks, and set up the marking infrastructure. This is microseconds on a healthy program because stack scanning itself was made concurrent in Go 1.8.

**Phase 2 — Concurrent mark.** Goroutines resume. The collector runs marking work on the GC's own goroutines and may **assist** by drafting mutator goroutines that allocate quickly into helping with marking — this is the "GC assist" mechanism that prevents allocators from outrunning the collector. Write barriers ensure that any pointer the application stores during this phase is recorded so the collector cannot lose track of newly-reachable objects.

**Phase 3 — Mark termination (STW).** A second short STW finishes any residual marking, disables the write barrier, and computes the next cycle's heap goal.

**Phase 4 — Concurrent sweep.** The runtime returns memory of unmarked objects to the allocator's free lists. This happens lazily, interleaved with new allocations: when a goroutine asks for memory, the allocator may sweep a few spans before returning. There is no separate "off" phase in steady state — the tail of one cycle's sweep overlaps with the start of the next cycle's allocation.

The two STW phases are why Go's pause times are sub-ms in practice. They do **constant work** (or work proportional to the number of P's, not the heap size). The marking work, which does scale with the heap, runs concurrently.

## 4. The pacer and `GOGC`

The pacer's job is deciding **when** to start the next cycle. It has one user-visible knob.

```bash
GOGC=100 ./myserver   # the default
```

`GOGC` is a **percentage**. `GOGC=100` means: trigger the next GC when newly allocated memory reaches 100% of the live heap from the previous cycle. Concretely, if at the end of cycle N you had 200 MB live, the runtime aims to start cycle N+1 when total heap reaches 400 MB (200 MB live + 200 MB new).

From the [`runtime` package documentation](https://pkg.go.dev/runtime#hdr-Environment_Variables):

> **GOGC** sets the initial garbage collection target percentage. A collection is triggered when the ratio of freshly allocated data to live data remaining after the previous collection reaches this percentage. The default is GOGC=100. Setting GOGC=off disables the garbage collector entirely.

The [GC guide](https://tip.golang.org/doc/gc-guide) gives the practical intuition: "doubling GOGC will double heap memory overheads and roughly halve GC CPU cost." So:

| `GOGC` | Heap overhead | GC CPU cost | Use case |
|---|---|---|---|
| 50 | low (~1.5× live) | high | memory-constrained |
| 100 | balanced (~2× live) | balanced | default |
| 200 | high (~3× live) | low | latency-sensitive, RAM available |
| 500 | very high (~6× live) | very low | very large heaps, tail-latency-critical |
| `off` | unbounded | zero | benchmark or short-lived process |

The runtime actually adjusts the trigger dynamically — the pacer is a feedback controller, not a fixed threshold — but `GOGC` sets the steady-state target. You can change it at runtime with [`runtime/debug.SetGCPercent`](https://pkg.go.dev/runtime/debug#SetGCPercent).

## 5. `GOMEMLIMIT`: the soft memory limit

Before Go 1.19, `GOGC` was the only knob. That was a problem in containerized deployments: a service with 2 GB cgroup limit and a baseline live heap of 100 MB would, at `GOGC=100`, target 200 MB total — fine. But under load with a transient 800 MB live heap, the pacer targets 1.6 GB total, then a brief surge in allocation rate pushes it past the 2 GB cgroup limit and the kernel OOM-kills the process before the next cycle starts.

Go 1.19 introduced [`GOMEMLIMIT`](https://go.dev/doc/go1.19) as a **soft memory limit**. From the release notes:

> Go 1.19 introduces support for a soft memory limit that includes the Go heap and all other memory managed by the runtime, excluding external memory sources such as mappings of the binary itself, memory managed in other languages, and memory held by the operating system on behalf of the Go program. This limit may be managed via [`runtime/debug.SetMemoryLimit`](https://pkg.go.dev/runtime/debug#SetMemoryLimit) or the equivalent `GOMEMLIMIT` environment variable.

```bash
GOMEMLIMIT=1750MiB ./myserver
```

Two important properties:

**It's "soft."** The runtime makes a best effort but does not promise. The [GC guide](https://tip.golang.org/doc/gc-guide) is explicit that the runtime "makes no guarantees that it will maintain this memory limit under all circumstances; it only promises some reasonable amount of effort." If your live working set genuinely exceeds the limit, you'll still OOM — the runtime can't conjure memory.

**There's a GC CPU brake.** To prevent the runtime from spinning at 100% GC CPU as the live heap approaches the limit (a "GC death spiral"), Go 1.19 also added a CPU limiter that caps total GC CPU at **50%** (excluding idle). When that limiter kicks in, the runtime chooses memory growth over crippling the application — and reports it via the `/gc/limiter/last-enabled:gc-cycle` runtime metric.

The recommended container pattern is to set `GOMEMLIMIT` slightly below the cgroup limit (typical advice: ~90–95% of the container limit, leaving headroom for non-Go memory like CGo and the binary mapping), and **leave `GOGC=100`**. The pacer will use `GOGC` while there's headroom and tighten as the heap approaches `GOMEMLIMIT`. Setting `GOGC=off` is also safe with `GOMEMLIMIT` set — it tells the runtime "use as much memory as you're allowed; collect only when you must."

```bash
# Container with 2 GB hard limit
GOMEMLIMIT=1800MiB GOGC=100 ./myserver
```

The release notes warn that very small limits don't work well: "small memory limits, on the order of tens of megabytes or less, are less likely to be respected due to external latency factors. Larger memory limits, on the order of hundreds of megabytes or more, are stable and production-ready."

## 6. STW pause times in production

Go 1.5 (August 2015) was the release that brought concurrent GC. The [release notes](https://go.dev/doc/go1.5) stated: "the 'stop the world' phase of the collector will almost always be under 10 milliseconds and usually much less."

Hudson's [2018 ISMM keynote](https://go.dev/blog/ismmkeynote) traces the pause-time progression:

| Release | Year | Typical STW |
|---|---|---|
| Pre-1.5 | <2015 | 300–400 ms |
| 1.5 | 2015 | 30–40 ms |
| 1.6 | 2016 | 4–5 ms |
| 1.7 | 2016 | <10 ms (larger heaps now supported) |
| 1.8 | 2017 | <1 ms (concurrent stack scanning) |
| 1.9 | 2017 | 100–200 µs |
| 1.10+ | 2018+ | 500 µs SLO target |

In production today, on a healthy Go service with a few-GB heap and reasonable allocation rate, expect **STW pauses well under a millisecond** per phase, with two phases per cycle. Cycles run frequently — every few seconds under load — so this is what you'll see on an SLO chart.

When pauses spike beyond this, the cause is almost always one of:

1. **Goroutine preemption stalls.** A goroutine in a tight CPU loop with no function calls can resist preemption (pre-Go 1.14). Asynchronous preemption fixed this for most cases, but [tight loops over `unsafe`-style code or assembly stubs can still hold up an STW](../concurrency/01-goroutines-and-scheduler.md). The runtime won't enter an STW phase until every goroutine is preemptable.
2. **Very large stacks.** A single goroutine with a multi-megabyte stack takes proportionally longer to scan. This is rare unless you have deeply recursive code.
3. **CGo calls in flight.** A goroutine inside a long CGo call holds the M and counts as if it were in a syscall — STW waits for it to return.
4. **OS-level pauses** (page faults, context switches, kernel locks) that are misattributed to GC. Always cross-check pause-time spikes against system-level metrics — see [observability](../../performance/INDEX.md).

## 7. Observing the GC

Three layered tools, in order of overhead:

**`GODEBUG=gctrace=1`** prints one line per GC to stderr, free, with negligible runtime cost. From [the runtime docs](https://pkg.go.dev/runtime#hdr-Environment_Variables):

```bash
GODEBUG=gctrace=1 ./myserver
```

```text
gc 4 @0.456s 1%: 0.013+1.2+0.008 ms clock, 0.10+0.30/2.4/0.0+0.064 ms cpu, 8->9->5 MB, 9 MB goal, 0 MB stacks, 0 MB globals, 8 P
```

Reading this line:

| Field | Meaning |
|---|---|
| `gc 4` | the 4th GC cycle since program start |
| `@0.456s` | 456 ms after program start |
| `1%` | total GC CPU since program start is 1% |
| `0.013+1.2+0.008 ms clock` | wall-clock for [STW setup, concurrent mark, STW termination] |
| `0.10+0.30/2.4/0.0+0.064 ms cpu` | CPU for [setup / assist / dedicated mark / idle / termination] |
| `8->9->5 MB` | heap at start of cycle, end of cycle, live |
| `9 MB goal` | next cycle's trigger heap size |
| `8 P` | 8 logical processors |

A `(forced)` suffix means a manual `runtime.GC()` call.

**`runtime/metrics`** (Go 1.16+) is the supported, structured way to read everything `gctrace` reports plus much more — pause histograms, allocation rates, goal heap, GOGC and GOMEMLIMIT effective values. From the [package docs](https://pkg.go.dev/runtime/metrics):

```go
import "runtime/metrics"

samples := []metrics.Sample{
    {Name: "/gc/heap/live:bytes"},
    {Name: "/gc/heap/goal:bytes"},
    {Name: "/sched/pauses/total/gc:seconds"}, // Float64Histogram
    {Name: "/cpu/classes/gc/pause:cpu-seconds"},
}
metrics.Read(samples)
```

Export these to Prometheus (the `prometheus/client_golang` collector wraps them) and you have GC observability for free.

**`runtime.GC()`** runs a full GC synchronously, blocking until it completes. Useful in benchmarks (force a clean baseline before measuring) and in heap profiling (so the profile reflects steady state). Do not call it on the request path in production — you are deliberately stopping the world.

**`net/http/pprof`** exposes live profiles over HTTP. Mount it with a side-effect import:

```go
import _ "net/http/pprof"

go func() { _ = http.ListenAndServe("localhost:6060", nil) }()
```

Then:

```bash
go tool pprof http://localhost:6060/debug/pprof/heap
```

We'll use this extensively in the [next doc](02-escape-analysis-and-allocation.md).

## 8. When the GC affects tail latency, and what to do

Three symptoms point at the GC as the cause of latency anomalies:

1. **GC CPU above 25%.** Sustained `gctrace` percentages over 25% mean the program is allocating fast enough that the collector cannot keep up at `GOGC=100`. The pacer will throttle the application via assists.
2. **STW pauses correlated with p99 latency spikes.** Compare the timestamps of `gctrace` lines with your latency histogram. If pauses cluster at the same times as request slowdowns, the GC is involved.
3. **Memory pressure approaching `GOMEMLIMIT`.** When the limiter is enabled, GC CPU climbs and eventually hits the 50% cap. After that, memory grows; before, latency suffers.

The fixes, in rough order of impact:

**Reduce allocations.** Almost always the right answer. Profile with `pprof -alloc_space` (allocation count and bytes since program start, not just live) to see what's churning. Common targets: string concatenation in hot paths, `fmt.Sprintf` for log formatting, slice-of-pointers when slice-of-values would do, interface boxing (`any`/`interface{}`) where a concrete type would skip the heap. The next doc walks through the techniques in detail.

**Pool reusable objects.** `sync.Pool` is the standard answer for objects allocated and discarded at request rate (request structs, byte buffers, JSON decoders). It is *not* a general object cache — pooled items can be GC'd at any time, and the pool's value is amortizing allocator work, not retaining state.

**Increase the heap budget.** Bump `GOGC` to 200 or higher, or set `GOMEMLIMIT` higher if the container allows. More headroom means cycles run less often and assists fire less often. The cost is RAM. If you have RAM available and the workload is latency-sensitive, this is often the cheapest fix.

**Reduce escape.** When a value escapes to the heap unnecessarily (e.g., because of an interface assignment that didn't need to be one), fixing the escape removes the allocation entirely. See the [pointers doc, §6](../fundamentals/07-pointers-methods-receivers.md) and the [next doc](02-escape-analysis-and-allocation.md).

What **doesn't** typically help:

- Calling `runtime.GC()` manually — it just trades scheduled pauses for unscheduled ones.
- Setting `GOGC=off` without `GOMEMLIMIT` — you'll OOM under sustained load.
- Pinning specific objects with finalizers — finalizers run *after* the object is unreachable; they don't prevent collection, and they add overhead.

## 9. Contrast with JVM and V8 collectors

A backend engineer crossing between Go, the JVM, and Node.js runs into all three GCs. A short comparison:

**JVM G1** (default since JDK 9, mainline since JDK 9). Generational, regional (heap divided into ~2048 equal-size regions), incremental compacting. Pause-target-driven (`-XX:MaxGCPauseMillis=200` is a soft target). Excellent throughput, predictable pauses in the tens of milliseconds. Larger memory overhead than Go because it carries card tables and remembered sets.

**JVM ZGC** (production since JDK 15, generational since JDK 21). Concurrent, region-based, compacting via colored pointers and load barriers. Sub-ms pauses on heaps up to terabytes. The closest JVM analogue to Go's latency-first design, but uses read barriers (load barriers) to handle concurrent compaction — a price Go avoids by not compacting at all. Recommended for very large heaps where G1's pause times become unacceptable.

**V8 (Node.js, Chrome).** Generational. The young generation is collected by **Scavenger**, a stop-the-world copying collector running over a small (typically a few-MB) "new space"; this is fast because new space is small. The old generation uses **Mark-Sweep-Compact**, with concurrent and incremental marking added in recent versions to reduce pause times. V8 is aggressively tuned for the workload patterns of JavaScript browsers and short-lived Node servers. See the [TypeScript V8 memory and GC doc](../../typescript/runtime/v8-memory-and-gc.md) for the detail.

| | Go | JVM G1 | JVM ZGC | V8 |
|---|---|---|---|---|
| Generational | no | yes | yes (since JDK 21) | yes |
| Compacting | no | yes (incremental) | yes (concurrent) | yes (Mark-Compact) |
| Read barriers | no | no | yes | no |
| Write barriers | mark phase only | always | always | always (during marking) |
| Typical p99 STW | sub-ms | tens of ms | sub-ms | a few ms (young) / tens of ms (old) |
| Tuning surface | `GOGC`, `GOMEMLIMIT` | dozens of `-XX:` flags | a handful of flags | `--max-old-space-size`, a few more |
| Throughput cost | moderate | low | moderate–high | low–moderate |

The honest one-line summary: Go gets sub-ms pauses by giving up generations and giving up compaction, and pays for that by needing more heap headroom. The JVM gets throughput by paying for write barriers and a more complex tuning surface. V8 splits the difference for JavaScript's allocation patterns. None of them is wrong; they're designed for different workloads.

## Related

- [Escape Analysis & Allocation](02-escape-analysis-and-allocation.md) — the other half of why Go's GC works the way it does
- [Pointers, Methods & Receivers — escape analysis primer](../fundamentals/07-pointers-methods-receivers.md)
- [Slices, Arrays & Maps — slice retention bug, preallocation](../fundamentals/04-slices-arrays-maps.md)
- [Goroutines & the Scheduler — sysmon, preemption, why STW can stall](../concurrency/01-goroutines-and-scheduler.md)
- [TypeScript — V8 Memory & GC](../../typescript/runtime/v8-memory-and-gc.md) — Scavenger + Mark-Sweep-Compact
- [Operating Systems — Virtual Memory & Paging](../../operating-systems/fundamentals/02-virtual-memory-and-paging.md) — what "the heap" actually is
- [Performance INDEX](../../performance/INDEX.md) — tail latency, percentiles, and how GC pauses show up in SLOs

## References

- "A Guide to the Go Garbage Collector" — https://tip.golang.org/doc/gc-guide
- `runtime` package, "Environment Variables" — https://pkg.go.dev/runtime#hdr-Environment_Variables
- `runtime/metrics` package — https://pkg.go.dev/runtime/metrics
- `runtime/debug.SetMemoryLimit` — https://pkg.go.dev/runtime/debug#SetMemoryLimit
- `runtime/debug.SetGCPercent` — https://pkg.go.dev/runtime/debug#SetGCPercent
- Go 1.19 release notes (introduces `GOMEMLIMIT`) — https://go.dev/doc/go1.19
- Go 1.5 release notes (concurrent GC reaches production) — https://go.dev/doc/go1.5
- Rick Hudson, "Getting to Go: The Journey of Go's Garbage Collector" (ISMM 2018 keynote) — https://go.dev/blog/ismmkeynote
- `net/http/pprof` — https://pkg.go.dev/net/http/pprof
- OpenJDK G1 Garbage Collector documentation — https://docs.oracle.com/en/java/javase/21/gctuning/garbage-first-g1-garbage-collector1.html
- OpenJDK ZGC documentation — https://docs.oracle.com/en/java/javase/21/gctuning/z-garbage-collector.html
- V8 blog, "Trash talk: the Orinoco garbage collector" — https://v8.dev/blog/trash-talk
