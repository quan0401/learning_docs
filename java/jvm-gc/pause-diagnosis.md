---
title: "GC Pause Diagnosis Playbook — Logs, JFR, async-profiler, Heap Dumps"
date: 2026-04-18
updated: 2026-04-18
tags: [java, jvm, gc, profiling, observability, performance]
---

# GC Pause Diagnosis Playbook — Logs, JFR, async-profiler, Heap Dumps

**Date:** 2026-04-18 | **Updated:** 2026-04-18
**Tags:** `java` `jvm` `gc` `profiling` `observability` `performance`

## Table of Contents

- [Summary](#summary)
- [Framing — Pause Rate vs Pause Time](#framing--pause-rate-vs-pause-time)
- [Enable Unified GC Logging](#enable-unified-gc-logging)
- [Reading a G1 Log](#reading-a-g1-log)
- [Reading a ZGC Log](#reading-a-zgc-log)
- [Tools of the Trade](#tools-of-the-trade)
  - [GCViewer and GCeasy](#gcviewer-and-gceasy)
  - [JDK Flight Recorder (JFR)](#jdk-flight-recorder-jfr)
  - [async-profiler](#async-profiler)
  - [jcmd and jstat](#jcmd-and-jstat)
- [Heap Dumps](#heap-dumps)
- [Pathologies Playbook](#pathologies-playbook)
  - [Humongous Allocations](#humongous-allocations)
  - [To-Space Exhausted / Evacuation Failure](#to-space-exhausted--evacuation-failure)
  - [Metaspace Leak](#metaspace-leak)
  - [Allocation Rate Spikes](#allocation-rate-spikes)
  - [Direct Memory OOM](#direct-memory-oom)
  - [Finalizer / Cleaner Backlog](#finalizer--cleaner-backlog)
  - [Reference Processing Stalls](#reference-processing-stalls)
  - [Long Time-to-Safepoint (TTSP)](#long-time-to-safepoint-ttsp)
- [Production Rollout Checklist](#production-rollout-checklist)
- [CI — Catching Regressions with JMH](#ci--catching-regressions-with-jmh)
- [Related](#related)
- [References](#references)

---

## Summary

Diagnosing GC pauses is a three-layer problem: collect the right data (unified GC logs, JFR, async-profiler), find the right signal (pause rate vs pause time vs allocation rate), and match it to a known pathology (humongous objects, evacuation failure, metaspace leaks, long TTSP). Almost every production GC incident fits one of ~eight well-known patterns — this doc is a cookbook for matching symptoms to fixes. The golden rule: always start from `gc.log` + `safepoint` log. Tools like GCeasy and GCViewer parse those logs and give you 80% of the answer in under a minute.

---

## Framing — Pause Rate vs Pause Time

Before diving in: what are you actually trying to minimize?

- **Pause time (max)** — longest single STW pause. Drives p99+ latency.
- **Pause time (sum)** — total time spent in GC. Drives throughput.
- **Pause frequency** — how often pauses happen. Drives p50 and p95 latency.
- **Allocation rate** — MB/s allocated. Drives young-GC frequency.
- **Live-set size** — MB retained across cycles. Drives old-gen pressure and cycle length.

These are linked but distinct. A service with 200 ms pauses every 10 minutes has a *max pause* problem. A service with 20 ms pauses every 2 seconds has a *frequency* problem. Different fixes. Know which one you are chasing before you change anything.

---

## Enable Unified GC Logging

JDK 9 replaced the old `-XX:+PrintGCDetails` family with **unified JVM logging** ([JEP 158](https://openjdk.org/jeps/158), [JEP 271](https://openjdk.org/jeps/271)). The minimum-viable configuration for any production service:

```bash
-Xlog:gc*,gc+heap=debug,gc+age=trace,safepoint:file=/var/log/gc.log:time,uptime,level,tags:filecount=10,filesize=50M
```

Breaking that down:

| Component | What it does |
|-----------|--------------|
| `gc*` | All GC-related output at info level. |
| `gc+heap=debug` | Heap region details per collection. |
| `gc+age=trace` | Object age distribution (tenuring) — useful for young-gen tuning. |
| `safepoint` | Time-to-safepoint per operation. Surfaces non-GC pauses. |
| `file=...` | Rotating file. `filecount=10, filesize=50M` caps total disk at ~500 MB. |
| `time,uptime,level,tags` | Decorators on each line. `time` = wall clock, `uptime` = since JVM start. |

Turn logging on **before** you need it. Trying to enable it after a production incident means you have already lost the data.

---

## Reading a G1 Log

A young GC entry in G1 looks like this:

```text
[2026-04-18T13:25:04.123+0000][45.234s][info][gc,start      ] GC(42) Pause Young (Normal) (G1 Evacuation Pause)
[2026-04-18T13:25:04.125+0000][45.236s][info][gc,task       ] GC(42) Using 8 workers of 8 for evacuation
[2026-04-18T13:25:04.143+0000][45.254s][info][gc,heap       ] GC(42) Eden regions: 200->0(200)
[2026-04-18T13:25:04.143+0000][45.254s][info][gc,heap       ] GC(42) Survivor regions: 25->25(25)
[2026-04-18T13:25:04.143+0000][45.254s][info][gc,heap       ] GC(42) Old regions: 87->92
[2026-04-18T13:25:04.143+0000][45.254s][info][gc,heap       ] GC(42) Humongous regions: 0->0
[2026-04-18T13:25:04.143+0000][45.254s][info][gc,metaspace  ] GC(42) Metaspace: 85234K(87040K)->85234K(87040K)
[2026-04-18T13:25:04.143+0000][45.254s][info][gc            ] GC(42) Pause Young (Normal) (G1 Evacuation Pause) 2245M->1876M(4096M) 20.123ms
```

What each field tells you:

- **`GC(42) Pause Young (Normal)`** — 42nd GC since JVM start. "Normal" young GC, not mixed or concurrent. Other reasons: `(Concurrent Start)`, `(Mixed)`, `(Prepare Mixed)`, `(System.gc())`.
- **`Using 8 workers`** — parallel GC thread count. If this is much less than your CPU budget, check `-XX:ParallelGCThreads`.
- **`Eden regions: 200->0(200)`** — went from 200 regions in use to 0, with 200 regions allocated after the pause. Always 0 after a young GC (Eden fully evacuated).
- **`Survivor regions: 25->25(25)`** — 25 → 25 means some objects survived and were copied to the "to" survivor.
- **`Old regions: 87->92`** — **this grew by 5 regions**. Those are promotions. Consistent growth over many young GCs signals the old gen will fill up.
- **`Humongous regions: 0->0`** — any non-zero number here is a red flag; see [Humongous allocations](#humongous-allocations).
- **`Metaspace`** — grows independent of heap. A slow monotonic climb over days = metaspace leak.
- **`2245M->1876M(4096M) 20.123ms`** — before → after (max). Pause was 20 ms.

### Concurrent cycle

When G1 starts a concurrent marking cycle, you see:

```text
GC(43) Pause Young (Concurrent Start) (G1 Humongous Allocation)
GC(43) Concurrent Mark Cycle
GC(43)   Concurrent Clear Claimed Marks
GC(43)   Concurrent Scan Root Regions
GC(43)   Concurrent Mark (45.3s)
GC(43) Pause Remark 5.123ms
GC(43)   Concurrent Rebuild Remembered Sets
GC(43) Pause Cleanup 1.234ms
```

Two STW pauses (Remark, Cleanup) plus several concurrent phases. If Remark pauses grow large (> 100 ms), check the **ref processing** subphase — reference processing can serialize on a single thread.

### Full GC

```text
GC(99) Pause Full (G1 Evacuation Pause) 4096M->3812M(4096M) 3456.789ms
```

**This is bad.** G1 shouldn't Full GC in normal operation. A Full GC means evacuation failed (no free regions to copy into), metaspace filled, or someone called `System.gc()`. See [To-space exhausted](#to-space-exhausted--evacuation-failure).

---

## Reading a ZGC Log

ZGC splits a cycle into phases, each of which may be STW or concurrent. A generational ZGC cycle:

```text
[info][gc,start] GC(17) Minor Collection (Allocation Rate)
[info][gc,phases] GC(17) Pause Mark Start 0.234ms
[info][gc,phases] GC(17) Concurrent Mark 12.345ms
[info][gc,phases] GC(17) Pause Mark End 0.876ms
[info][gc,phases] GC(17) Concurrent Relocate 15.123ms
[info][gc] GC(17) Minor Collection (Allocation Rate) 2048M(50%)->1024M(25%) 28.578ms
```

Notable fields:

- **`Pause Mark Start`, `Pause Mark End`** — STW. These are the pauses that count against your p99. Target is sub-millisecond.
- **`Concurrent Mark`, `Concurrent Relocate`** — runs while the app is live. Uses CPU but doesn't pause.
- **`(Allocation Rate)`** — trigger reason. Others: `(Proactive)`, `(Warmup)`, `(Timer)`.
- **`2048M(50%)->1024M(25%)`** — heap before/after the cycle. Percentages are of `SoftMaxHeapSize`.
- **Total `28.578ms`** — wall-clock cycle time. The user-visible pause is just the STW phases, which should sum to < 1 ms.

An **Allocation Stall** in ZGC means the allocator outran the concurrent collector. This prints:

```text
[warning][gc,alloc] Allocation Stall (main) 123.456ms
```

Stalls are real pauses — threads waited. If you see these, allocation rate is too high for ZGC to keep up. Either lower allocation rate (see [Allocation rate spikes](#allocation-rate-spikes)) or give ZGC more CPU (`-XX:ConcGCThreads`).

---

## Tools of the Trade

### GCViewer and GCeasy

Drag-and-drop `gc.log` parsers that give you:
- Throughput % (app time / total time).
- Pause histograms.
- Allocation rate over time.
- Promotion rate.
- Concurrent vs STW time.

[GCViewer](https://github.com/chewiebug/GCViewer) is local, open-source, offline. [GCeasy](https://gceasy.io/) is web-based and has slightly prettier visualizations — but it requires uploading your log to a third-party service, which is often not acceptable for production logs.

Start here every time. Most questions are answered within a minute of loading the log.

### JDK Flight Recorder (JFR)

[JFR](https://docs.oracle.com/en/java/javase/21/jfapi/flight-recorder-api-guide.html) is a low-overhead profiler baked into the JVM. It captures CPU samples, allocation samples, lock contention, I/O, GC events — everything with < 1% overhead on a reasonable configuration.

Enable it always-on in production:

```bash
-XX:StartFlightRecording=filename=/var/log/recording.jfr,maxage=24h,maxsize=1g,settings=profile
```

Or dump on demand via `jcmd`:

```bash
jcmd <pid> JFR.start name=inc duration=60s filename=/tmp/inc.jfr settings=profile
```

Open `.jfr` files with [JDK Mission Control](https://adoptium.net/jmc/) or the `jfr` CLI. Key views for GC work:

- **GC → Garbage Collections** — per-cycle breakdown, reason, pause time.
- **Memory → Allocation in New TLAB** — which call stacks allocate most.
- **Memory → Reference Statistics** — soft/weak/phantom reference counts.
- **JVM → Safepoints** — TTSP over time.

The "allocation in new TLAB" view is gold — it points to the *code* causing allocation pressure, which is usually what you actually want to fix.

### async-profiler

[async-profiler](https://github.com/async-profiler/async-profiler) is a sampling profiler that works where JFR has blind spots. Its killer feature for GC work is **allocation profiling with flame graphs**:

```bash
java -jar async-profiler.jar -e alloc -d 60 -f /tmp/alloc.html <pid>
```

The resulting flame graph shows allocation sites by bytes allocated. This identifies `StringBuilder.append`, `JSONObject.put`, or Reactor operator chains that are silently allocating GB per minute. Fixing a single hot allocation can cut young GC frequency in half.

### jcmd and jstat

`jcmd` is the Swiss Army knife for live JVM introspection:

```bash
jcmd <pid> GC.class_histogram      # live object counts by class
jcmd <pid> GC.heap_dump /tmp/d.hprof
jcmd <pid> VM.native_memory summary  # needs -XX:NativeMemoryTracking=summary
jcmd <pid> VM.flags                  # current flag values
jcmd <pid> Thread.print              # thread dump
jcmd <pid> JFR.start
jcmd <pid> JFR.dump filename=...
```

`jstat -gcutil <pid> 1s` gives you a live table of utilization per generation. Useful for quick sanity checks — "is the old gen growing?".

---

## Heap Dumps

When you need to answer "why is this heap full", you take a heap dump and analyze it in [Eclipse MAT](https://www.eclipse.org/mat/) (Memory Analyzer Tool).

```bash
jcmd <pid> GC.heap_dump /var/log/heap.hprof
# or automatically on OOM:
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/
```

Heap dumps are huge (the size of your live heap + overhead) and taking one STW-pauses the JVM for seconds. Don't do it casually in production.

Key MAT views:

- **Dominator Tree** — the objects retaining the most memory. Top 10 is usually where your leak is.
- **Leak Suspects Report** — MAT's heuristic analysis. Usually identifies the real problem.
- **Histogram** — class counts, sorted by shallow or retained size.
- **OQL (Object Query Language)** — SQL-like queries against the dump.

Common finds:
- A static `ConcurrentHashMap` holding every request ever seen (classic "in-memory cache without eviction").
- A classloader referenced by a ThreadLocal, preventing Tomcat hot-redeploy from freeing old classes.
- Log4j `MDC` with 10M entries from a leaked thread-local.

---

## Pathologies Playbook

Match symptom → diagnosis → fix. Each subsection is a recipe for one failure mode.

### Humongous Allocations

**Symptom:** G1 log shows `Humongous regions: 0->N` with N > 0 repeatedly, or `(G1 Humongous Allocation)` as a GC cause. Pauses are fine but concurrent cycles happen way more often than expected.

**Diagnosis:** Objects > 50% of G1 region size are allocated directly to old-gen regions. With default 1–2 MB region size, arrays of ~1 MB+ are humongous. Large byte buffers, JSON payloads, protobuf messages, JDBC `ResultSet` buffers are typical.

**Fix:**

1. Size up regions: `-XX:G1HeapRegionSize=16m` (powers of 2, 1–32 MB). Bigger regions = fewer humongous allocations.
2. Fix the allocation: batch it differently, stream it, or use off-heap buffers.
3. On JDK 11+, humongous regions are reclaimed during young GC. Pre-JDK-8u60, they were not — if you're on an ancient JDK, this alone was catastrophic.

### To-Space Exhausted / Evacuation Failure

**Symptom:** G1 log shows `to-space exhausted` or a Full GC with cause "Evacuation Failure". Pause time spikes from 50 ms to several seconds.

**Diagnosis:** During young GC, G1 tried to copy surviving objects into new regions but ran out. Either the old gen is full, or the reserve wasn't enough. Once evacuation fails, G1 falls back to a **serial full GC** — the worst pause.

**Fix:**

1. Larger heap — more room to evacuate into.
2. Increase `-XX:G1ReservePercent` (default 10). Try 20.
3. Lower `-XX:InitiatingHeapOccupancyPercent` (default 45). Earlier concurrent cycles reclaim old-gen sooner.
4. If the root cause is a leak, no tuning will save you — find the leak with a heap dump.

### Metaspace Leak

**Symptom:** `Metaspace` value in GC log grows monotonically over hours or days. Eventually: `java.lang.OutOfMemoryError: Metaspace`. Heap itself is healthy.

**Diagnosis:** Classes are being loaded faster than they are unloaded. Usually a **classloader leak** — a long-lived object (thread-local, cache) retains a reference to a `ClassLoader`, preventing it and all its classes from being GC'd. Common causes:

- Tomcat or similar containers doing hot redeploys while your app leaks the old classloader.
- Libraries that load classes dynamically (Groovy, JRuby, scripting engines) without cleanup.
- Spring devtools in production (don't).
- Byte-code instrumentation agents that create classes per request.

**Fix:**

1. `-XX:NativeMemoryTracking=summary` then `jcmd <pid> VM.native_memory summary`. Metaspace subcategory shows growth.
2. Heap dump + MAT → find the leaking ClassLoader. "Leak Suspects Report" usually nails it.
3. Cap metaspace: `-XX:MaxMetaspaceSize=512m` turns a slow leak into a fast failure — better for k8s autoheal.

### Allocation Rate Spikes

**Symptom:** Young GC frequency goes from once/minute to once/second. Pauses themselves are fine (short), but throughput plummets because the service spends too much total time in GC.

**Diagnosis:** Something started allocating a lot. In reactive pipelines this is often a fanout operator, a JSON serializer, or a `String.format` in a logger call. The allocation rate (MB/s) in the GC log or JFR tells you the magnitude.

**Fix:**

1. async-profiler `-e alloc` — flame graph of allocation sites. Fix the top one.
2. If the allocation is legitimate and necessary, give the young gen more room: `-XX:G1NewSizePercent=30 -XX:G1MaxNewSizePercent=60` (defaults are 5 and 60).
3. Consider ZGC — its concurrent nature means allocation spikes translate to CPU usage, not pause time.

### Direct Memory OOM

**Symptom:** `java.lang.OutOfMemoryError: Direct buffer memory` even though heap usage is healthy. RSS grows unboundedly, eventually k8s OOM-kills the pod.

**Diagnosis:** Netty, NIO, or `Unsafe` allocating direct `ByteBuffer`s that aren't being released. Netty has a **leak detector** that samples a % of buffers and prints warnings when they're GC'd without being released:

```text
-Dio.netty.leakDetection.level=paranoid   # 100% sampling, expensive
-Dio.netty.leakDetection.level=advanced   # includes recent access stacks
```

**Fix:**

1. Turn leak detection to `advanced`, then `paranoid` temporarily. Find the operator or handler that forgets `release()`.
2. In Reactor Netty: make sure custom `ChannelHandler`s call `ReferenceCountUtil.release(msg)` on dropped messages.
3. Cap direct memory: `-XX:MaxDirectMemorySize=1g`. Same rationale as capping metaspace — fail fast.
4. See [Reactive impact on GC — Netty direct memory](reactive-impact.md#netty-direct-memory) for the Reactor-specific patterns.

### Finalizer / Cleaner Backlog

**Symptom:** Old-gen usage creeps up across concurrent cycles even though you don't seem to have a leak. `java.lang.ref.Finalizer` shows up high in a heap histogram.

**Diagnosis:** Objects with `finalize()` methods, or that register with `java.lang.ref.Cleaner`, cannot be collected in a single GC cycle. They need two: one to detect unreachability, one to actually reclaim after the cleanup runs. If the finalizer/cleaner thread can't keep up, memory accumulates.

**Fix:**

1. Stop writing `finalize()` methods (they were [deprecated in JDK 9](https://openjdk.org/jeps/421)).
2. If a library uses them, open an issue. `Cleaner` is strictly better.
3. As a stopgap: increase finalizer thread priority or switch to ZGC (which processes references concurrently).

### Reference Processing Stalls

**Symptom:** G1 remark pauses unexpectedly long (500 ms+). G1 log with `gc+phases=debug` shows "Reference Processing" taking most of the pause.

**Diagnosis:** You have huge numbers of `WeakReference`, `SoftReference`, or `PhantomReference` — typically a cache using weak keys, or an overly-clever library.

**Fix:**

- Enable parallel reference processing: `-XX:+ParallelRefProcEnabled` (used to be off by default; default in modern JDKs).
- Reduce reference count: is that weak-reference cache really necessary? A bounded Caffeine cache is usually better.

### Long Time-to-Safepoint (TTSP)

**Symptom:** Safepoint log shows TTSP > 10 ms. App latency spikes don't correlate with GC pauses but do correlate with *something*.

**Diagnosis:** A thread is taking too long to reach a safepoint. The collector waits for it before starting STW work. Causes:

- **Counted loops with `long` index** — pre-JDK 10 these had no safepoint poll in the back-edge. Modern JDKs fix this.
- **Blocking native calls** — a JNI call that doesn't return polls for safepoints. If it blocks for seconds, the whole JVM waits.
- **Very long methods** — only entry/exit have safepoints.
- **`Arrays.fill`, `System.arraycopy`, large intrinsic operations** — run in native code, no polls.

**Fix:**

1. Enable `safepoint` logging; identify which operation is slow.
2. `-XX:+SafepointTimeout -XX:SafepointTimeoutDelay=5000` — the JVM will dump threads if a safepoint takes > 5s.
3. Fix the offending code: break up the big operation, use async JNI, etc.

---

## Production Rollout Checklist

Before changing the GC in production:

- [ ] GC logs enabled and rotated for the last 2 weeks of baseline.
- [ ] Baseline p50/p95/p99 latency captured.
- [ ] Baseline allocation rate, promotion rate, live-set size recorded.
- [ ] Heap size and pause targets are reasonable for the SLO (see [collectors.md](collectors.md#collector-decision-tree)).
- [ ] JFR running continuously with `maxage=24h`.
- [ ] `HeapDumpOnOutOfMemoryError` set.
- [ ] Rollout is gradual: one canary pod, then one zone, then region-wide.
- [ ] Rollback plan is one env-var flip, not a code change.

---

## CI — Catching Regressions with JMH

[JMH](https://github.com/openjdk/jmh) (Java Microbenchmark Harness) is the only legitimate way to write JVM microbenchmarks. For GC regression detection, use the `-prof gc` profiler:

```bash
mvn clean install
java -jar target/benchmarks.jar -prof gc MyBench
```

Output includes per-invocation allocation rate (`gc.alloc.rate.norm` — bytes per op), and GC-churn time. If a refactor doubles `gc.alloc.rate.norm`, you have a regression before it hits production. Wire this into CI for performance-sensitive hot paths (parsers, serializers, request handlers).

---

## Related

- [GC Concepts and Mental Model](concepts.md) — the vocabulary used throughout this doc.
- [JVM Collectors — Serial, Parallel, CMS, G1, ZGC, Shenandoah](collectors.md) — choose the right collector *before* trying to diagnose pauses.
- [Reactive / WebFlux / VT impact on GC](reactive-impact.md) — how reactive code shapes the log you will be reading.
- [Logging](../logging.md) — application logs that pair well with GC logs during incidents.
- [Reactive observability](../reactive-observability.md) — Micrometer JVM metrics.

---

## References

- [OpenJDK GC Tuning Guide (JDK 21)](https://docs.oracle.com/en/java/javase/21/gctuning/)
- [JEP 158: Unified JVM Logging](https://openjdk.org/jeps/158)
- [JEP 271: Unified GC Logging](https://openjdk.org/jeps/271)
- [GCViewer (chewiebug)](https://github.com/chewiebug/GCViewer)
- [GCeasy — hosted GC log analyzer](https://gceasy.io/)
- [async-profiler](https://github.com/async-profiler/async-profiler)
- [Eclipse Memory Analyzer Tool (MAT)](https://www.eclipse.org/mat/)
- [JDK Mission Control](https://adoptium.net/jmc/)
- [JMH — Java Microbenchmark Harness](https://github.com/openjdk/jmh)
- [Netty Reference Counted Objects](https://netty.io/wiki/reference-counted-objects.html)
- [Aleksey Shipilëv — "JVM Anatomy Quarks"](https://shipilev.net/jvm/anatomy-quarks/) — includes TTSP, card tables, TLABs.
- [JEP 421: Deprecate Finalization for Removal](https://openjdk.org/jeps/421)
