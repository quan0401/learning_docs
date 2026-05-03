---
title: "Flame Graphs — On-CPU and Off-CPU Profiling"
date: 2026-05-03
updated: 2026-05-03
tags: [performance, profiling, flame-graphs, off-cpu, perf, async-profiler]
---

# Flame Graphs — On-CPU and Off-CPU Profiling

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `performance` `profiling` `flame-graphs` `off-cpu` `perf` `async-profiler`

---

## Table of Contents

- [Summary](#summary)
- [1. The Flame Graph Idea](#1-the-flame-graph-idea)
  - [1.1 What a Flame Graph Shows](#11-what-a-flame-graph-shows)
  - [1.2 How to Read One](#12-how-to-read-one)
  - [1.3 What a Flame Graph Does Not Show](#13-what-a-flame-graph-does-not-show)
- [2. Sampling Profilers Under the Hood](#2-sampling-profilers-under-the-hood)
  - [2.1 The Sampling Trick](#21-the-sampling-trick)
  - [2.2 Sample Rate Choice](#22-sample-rate-choice)
  - [2.3 Stack Unwinding](#23-stack-unwinding)
- [3. On-CPU Flame Graphs — Where Time Burns](#3-on-cpu-flame-graphs--where-time-burns)
  - [3.1 What They Catch](#31-what-they-catch)
  - [3.2 Linux `perf` Workflow](#32-linux-perf-workflow)
  - [3.3 JVM with async-profiler](#33-jvm-with-async-profiler)
  - [3.4 Node.js with 0x and clinic.js](#34-nodejs-with-0x-and-clinicjs)
- [4. Off-CPU Flame Graphs — Where Time Waits](#4-off-cpu-flame-graphs--where-time-waits)
  - [4.1 The Off-CPU Insight](#41-the-off-cpu-insight)
  - [4.2 Off-CPU on Linux](#42-off-cpu-on-linux)
  - [4.3 Off-CPU on the JVM (wall-clock mode)](#43-off-cpu-on-the-jvm-wall-clock-mode)
  - [4.4 Async I/O and Off-CPU](#44-async-io-and-off-cpu)
- [5. Differential Flame Graphs](#5-differential-flame-graphs)
- [6. Icicle Graphs](#6-icicle-graphs)
- [7. A Disciplined Profiling Workflow](#7-a-disciplined-profiling-workflow)
- [Related](#related)
- [References](#references)

---

## Summary

A flame graph is the most efficient visualization invented for stack-sample data. Brendan Gregg created it because reading hundreds of thousands of stack traces by hand is impossible and traditional `gprof`-style call graphs hide where the time actually goes. The flame graph aggregates samples by stack, sorts them, and renders the result as a single image where horizontal width = time and vertical depth = stack depth — letting you see "what is using CPU right now" in seconds. Two variants matter for backend work: **on-CPU flame graphs** show where threads are running (CPU-bound paths, hot loops); **off-CPU flame graphs** show where threads are *waiting* (locks, I/O, scheduling). Most real performance bugs — connection pool starvation, lock contention, slow downstream calls — are off-CPU, so reading only on-CPU flame graphs leaves half the bugs invisible. This doc covers how flame graphs work, how to generate them with `perf`, async-profiler, and 0x/clinic.js, the off-CPU technique, and differential / icicle variants for regression hunting.

---

## 1. The Flame Graph Idea

### 1.1 What a Flame Graph Shows

A flame graph is built from a list of stack samples. Each sample is a single observation of "the program is here right now" — a chain of function names from the bottom of the call stack to the top:

```
main → handleRequest → queryDb → executeStatement → networkSend → tcp_sendmsg
```

The flame graph plots:

- **X axis:** alphabetically sorted stack frames at each level (no time axis — the X axis is *cumulative samples*).
- **Y axis:** stack depth, with `main` at the bottom and the topmost (currently-executing) frame on top.
- **Width:** number of samples in which this frame appeared at this position. Wider = more time on CPU.

Two stack samples that share a common prefix are merged in the graph. So the flame graph shows you, at a glance, "which top-level paths through the code are using the most CPU".

```text
[                main                ]
[      handleRequest      ][admin]
[ parseRequest ][ executeQuery ]
                [ formatResponse ]
                [ jsonStringify ]
```

### 1.2 How to Read One

Width = time. Depth = call depth, but it's not the interesting axis — what matters is the wide rectangles at the top of each stack, because those are the leaf functions doing the work.

A common pattern: "the top of the flame graph is mostly `JSON.stringify`" — your serializer is your hot path. Another: "a wide rectangle of `pthread_cond_wait`" — your CPU is mostly *waiting on a condition variable* (lock contention; this would also need an off-CPU profile to confirm).

The colors in a classic Brendan Gregg flame graph are random and convey no semantic meaning — they exist to make the visual easier to scan. Differential flame graphs (Section 5) use color semantically.

### 1.3 What a Flame Graph Does Not Show

- **Time order.** The X axis is not time. A flame graph is an *aggregate* over the profiling window. To see how the profile changes over time, generate flame graphs for time slices and compare (or use a time-resolved variant like FlameScope).
- **Duration of any single stack.** A wide rectangle means many samples landed there; it does not mean "this function call took a long time" in any single invocation.
- **Cause of the time.** A wide `read()` rectangle could be hot disk I/O or a million tiny reads. Need additional context (counts vs duration) to disambiguate.
- **Off-CPU time.** A standard flame graph from a CPU profiler shows only on-CPU samples. Time spent waiting is invisible — see Section 4.

---

## 2. Sampling Profilers Under the Hood

### 2.1 The Sampling Trick

A sampling profiler interrupts the running process at fixed intervals — typically 99 Hz or 999 Hz — and captures the current stack trace. The frequency is intentionally non-round (99 instead of 100) so it doesn't synchronize with periodic events at round frequencies in the code.

After running for, say, 60 seconds at 99 Hz, the profiler has ~5,940 samples. The stack frame that appears in the most samples is the one the program spends the most CPU time in — within the statistical noise of the sampling.

Two big advantages over instrumented (deterministic) profilers:

- **Low overhead.** A sample-and-walk-the-stack costs single-digit microseconds; at 99 Hz overhead is well under 1% of CPU.
- **No code changes.** Most kernel-supported profilers (`perf`, `eBPF`, `async-profiler`) attach to a running process.

The trade-off: sampling cannot detect *infrequent* events. A function called once for 10 ms in a 60-second profile may produce zero samples and be invisible. Use deterministic profiling for rare-but-expensive events.

### 2.2 Sample Rate Choice

Higher rate = more resolution, more overhead. The sweet spots:

- **99 Hz** — cheap, good for steady-state production profiling. Aliasing-resistant.
- **999 Hz** — short bursts (5-30 seconds). Better resolution for short-duration profiling.
- **9,999 Hz** — research-grade, only viable for short bursts on idle systems.

For services running at high load, 99 Hz over 60-300 seconds is typical.

### 2.3 Stack Unwinding

Capturing a stack from an interrupt requires walking the chain of saved frame pointers. Three flavors:

- **Frame pointer unwinding** (`-fno-omit-frame-pointer`). Cheapest and most reliable on Linux. Requires the binary to be compiled with frame pointers (most distros now ship with FP enabled in glibc/system libs; many third-party libraries don't). On amd64, typical overhead is under 1%.
- **DWARF unwinding.** Walks DWARF debug info to reconstruct the stack. Slow per-sample (10-100×) and can be unreliable. Avoid for production profiling.
- **LBR (Last Branch Records).** Hardware-assisted on recent Intel/AMD; very fast but limited to the last 16-32 branches.

For Java, async-profiler uses **AsyncGetCallTrace** (an internal HotSpot API) to capture safe Java stacks without requiring safepoints, plus frame-pointer unwinding for native frames. This is the reason async-profiler can profile a JVM at 1 kHz with under 1% overhead.

For Node.js / V8, sampling captures both V8 JS stacks and native libuv stacks, with the V8 isolate cooperating to provide JIT-compiled frame names.

---

## 3. On-CPU Flame Graphs — Where Time Burns

### 3.1 What They Catch

On-CPU flame graphs show you *where the CPU is busy*. They detect:

- Hot loops in user code.
- Expensive serialization/deserialization (JSON, Protobuf).
- Cryptography (TLS handshakes, hashing).
- Regex backtracking pathology.
- Reflection-heavy code paths (Java).
- JIT compilation and deoptimization (visible as time in the JIT compiler thread or in deopt handlers).
- Inefficient `String` concatenation, boxing, allocations in hot paths.

### 3.2 Linux `perf` Workflow

```bash
# 1. Capture stacks at 99 Hz for 60 seconds across all CPUs
sudo perf record -F 99 -a -g --call-graph dwarf -- sleep 60

# 2. Convert to a flame-graph-friendly format
sudo perf script > out.perf

# 3. Use Brendan Gregg's FlameGraph scripts
git clone https://github.com/brendangregg/FlameGraph
./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
./FlameGraph/flamegraph.pl out.folded > flame.svg
```

Open `flame.svg` in a browser — the SVG is interactive, click to zoom into a frame, search to highlight matches.

For frame-pointer unwinding (preferred when available):

```bash
sudo perf record -F 99 -a -g --call-graph fp -- sleep 60
```

For per-PID profiling: replace `-a` with `-p <pid>`.

### 3.3 JVM with async-profiler

`async-profiler` is the de facto standard for JVM profiling. It produces flame graphs directly.

```bash
# Profile by attaching to a running PID for 30 seconds
./profiler.sh -d 30 -f profile.html <pid>

# Profile with allocation sampling (heap)
./profiler.sh -d 30 -e alloc -f alloc.html <pid>

# Wall-clock profiling (includes off-CPU time — see Section 4.3)
./profiler.sh -d 30 -e wall -f wall.html <pid>

# Flame graph in collapsed format for further processing
./profiler.sh -d 30 -o collapsed -f profile.collapsed <pid>
```

`async-profiler` integrates with JFR (Java Flight Recorder) and can write JFR files that IntelliJ, JMC, or `jfr` CLI can analyze.

For Spring Boot in containers, run async-profiler as an agent at JVM startup:

```bash
java -agentpath:/opt/async-profiler/libasyncProfiler.so=start,event=cpu,file=cpu.html ...
```

See [Java — observability docs](../../java/observability/distributed-tracing.md) and [Java — JVM GC pause diagnosis](../../java/jvm-gc/pause-diagnosis.md) for the broader JVM observability picture.

### 3.4 Node.js with 0x and clinic.js

[`0x`](https://github.com/davidmarkclements/0x) generates V8 sampling-profile flame graphs for Node:

```bash
npm install -g 0x
0x -- node server.js
# Generate load against the server, then Ctrl+C
# 0x writes flamegraph.html to a profile dir
```

Clinic.js provides higher-level analysis:

```bash
npm install -g clinic
# Flame graph
clinic flame --on-port 'autocannon -d 30 http://localhost:3000' -- node server.js

# Doctor — overall analysis (CPU, event loop, memory, GC)
clinic doctor --on-port 'autocannon -d 30 http://localhost:3000' -- node server.js

# Bubbleprof — async dependency profiling (timeline of async resources)
clinic bubbleprof --on-port 'autocannon -d 30 http://localhost:3000' -- node server.js
```

`bubbleprof` is the closest thing Node has to an off-CPU flame graph for async work — see Section 4.4.

For production, use `--inspect` and `--cpu-prof` flags to capture V8 CPU profiles, then convert with `flamegraph-convert`. See [TypeScript — Node Profiling and Debugging](../../typescript/production/profiling-and-debugging.md).

---

## 4. Off-CPU Flame Graphs — Where Time Waits

### 4.1 The Off-CPU Insight

Brendan Gregg's key insight: in a typical service, threads spend the **majority of wall-clock time waiting** — for I/O, for locks, for downstream services, for the scheduler. CPU profiling is blind to all of this. A service running at 5% CPU utilization is making 95% of its responses by waiting, and the on-CPU flame graph tells you nothing about why a request is slow.

Off-CPU profiling captures **scheduler events**: every time a thread is taken off-CPU (blocks on a syscall, waits on a lock, sleeps), capture its stack and the time it stays off-CPU. The result is a flame graph where width = wall-clock time spent waiting in this stack.

### 4.2 Off-CPU on Linux

The standard tool is `perf` with scheduler tracepoints, or `eBPF` with `bcc/offcputime`:

```bash
# bcc-tools offcputime (requires root; uses eBPF)
sudo offcputime -df -p $(pgrep -nx my-service) 60 > out.off

# Generate flame graph
./FlameGraph/flamegraph.pl --color=io --title="Off-CPU Time" out.off > offcpu.svg
```

The output shows where threads were **scheduled out**. Wide bars at `tcp_recvmsg` mean lots of time waiting for a network response. Wide bars at `futex_wait` mean lock contention.

`bpftrace` one-liner equivalent:

```bash
sudo bpftrace -e '
  kprobe:finish_task_switch /comm == "java"/ {
    @[kstack, ustack] = sum(nsecs - @start[tid]);
  }
  kprobe:finish_task_switch { @start[tid] = nsecs; }
  END { print(@); clear(@); clear(@start); }'
```

### 4.3 Off-CPU on the JVM (wall-clock mode)

`async-profiler -e wall` captures samples regardless of whether the thread is on or off CPU. The resulting flame graph shows wall-clock time, which is what you want for diagnosing slow requests.

```bash
./profiler.sh -d 30 -e wall -t -f wall.html <pid>
```

The `-t` flag includes thread names so you can see "the request thread spent 90% of its wall time in `Socket.read()` waiting for the downstream service".

This is the JVM equivalent of an off-CPU flame graph. Use it when:

- p99 latency is high but CPU utilization is low (classic blocking-I/O bottleneck).
- The on-CPU flame graph shows mostly framework code that can't possibly be the bottleneck.
- You suspect lock contention (look for wide `Object.wait`, `LockSupport.park`, `parkAndCheckInterrupt`).

### 4.4 Async I/O and Off-CPU

Async runtimes (Node.js, Java reactive, Go) make off-CPU profiling harder because there is no single thread waiting on the I/O — the work is split across an event loop.

For Node.js, **clinic.js bubbleprof** traces async resources (Promises, callbacks) over time, producing a visualization of "what async chain was waiting and on what". It's not a flame graph in the strict sense but solves the same problem.

For Java reactive (Project Reactor / WebFlux), wall-clock profiling with async-profiler is still useful, but the threads doing the waiting are scheduler threads (`reactor-http-nio-N`) shared across many requests. You'll see hot spots like `epoll_wait` (the event loop waiting) which is normal — the interesting question is **what concurrent operations were waiting at that moment** and that requires distributed tracing more than flame graphs.

See [Java — Reactive Programming](../../java/reactive-programming-java.md) and [TypeScript — Event Loop Internals](../../typescript/runtime/event-loop-internals.md).

---

## 5. Differential Flame Graphs

A differential flame graph compares two profiles — typically "before a change" and "after a change" — and color-codes:

- **Red** frames where the second profile spent *more* time.
- **Blue** frames where the second profile spent *less* time.
- White/neutral for unchanged.

Workflow:

```bash
# Capture two profiles
sudo perf record -F 99 -p <pid> -g -- sleep 30
sudo perf script | ./FlameGraph/stackcollapse-perf.pl > before.folded

# Deploy the change

sudo perf record -F 99 -p <pid> -g -- sleep 30
sudo perf script | ./FlameGraph/stackcollapse-perf.pl > after.folded

# Generate differential
./FlameGraph/difffolded.pl before.folded after.folded | ./FlameGraph/flamegraph.pl > diff.svg
```

Differential flame graphs are the right tool for **regression hunting**. Instead of staring at two flame graphs side by side, you see exactly which stacks got hotter or colder.

For perf-CI pipelines, automate before/after differential comparison on each PR.

---

## 6. Icicle Graphs

An **icicle graph** is a flame graph rendered upside-down — top-level functions at the top, leaves at the bottom. Some tools (Java Flight Recorder, perfetto) prefer icicle orientation because it matches the way call graphs are usually drawn (caller above callee).

The information content is identical to a flame graph. Use whichever the team already reads.

A related variant: **flame charts** (used by Chrome DevTools, perfetto). These plot stacks against the real time axis, so you can see *when* a hot function was running. Useful for understanding bursty workloads but harder to read for steady-state aggregate profiling.

---

## 7. A Disciplined Profiling Workflow

The mistake most engineers make is profiling first, hypothesizing second. The disciplined order:

1. **Identify a specific symptom.** "p99 of /api/checkout is 800 ms over the 200 ms SLO." Without this, you have no signal to compare against.

2. **Confirm USE saturation.** Is CPU saturated? Memory? DB pool? Disk? The answer determines whether on-CPU or off-CPU profiling is the right starting point.
   - CPU saturated → on-CPU flame graph.
   - DB pool saturated → wall-clock / off-CPU profile to confirm threads waiting on pool, then DB-side EXPLAIN.
   - Nothing saturated but high latency → off-CPU profile to find what's *waiting*.

3. **Capture under realistic load.** Profiling at idle shows you idle behavior. Profile during the actual problem window or under reproduction load.

4. **Compare against baseline.** A flame graph in isolation shows what's hot; what you want to know is what changed. Differential flame graph against last week or against canary.

5. **Validate the hypothesis with code/data.** A wide rectangle suggests an answer; the answer must be confirmed by code review or by changing the code and re-profiling. "It looks like it's slow because of X" is a hypothesis, not a finding.

6. **Profile again after the fix.** Confirm the hot spot moved. If it didn't, the diagnosis was wrong.

Common anti-patterns:

- Profiling without a baseline. The flame graph "looks busy" — yes, every running program does. Without a comparison you cannot tell what's normal.
- Reading only on-CPU graphs for a service that is mostly waiting. The flame graph will show framework code (Tomcat, Netty, libuv) consuming CPU, but the bottleneck is invisible.
- Assuming wide = wrong. A wide rectangle in `JSON.parse` is expected if your service is a JSON-shuffling API. Wide is bad only when it's wider *than it should be*.

---

## Related

- [Latency, Throughput, Percentiles](01-latency-throughput-percentiles.md)
- [Tail Latency and p99](03-tail-latency-and-p99.md)
- [USE and RED Applied](04-use-and-red-applied.md)
- [Load Testing Methodology](06-load-testing-methodology.md)
- [Java — JVM GC Pause Diagnosis](../../java/jvm-gc/pause-diagnosis.md)
- [TypeScript — Node Profiling and Debugging](../../typescript/production/profiling-and-debugging.md)
- [TypeScript — V8 Engine Pipeline](../../typescript/runtime/v8-engine-pipeline.md)
- [Networking — Async I/O Models](../../networking/network-programming/async-io-models.md)

---

## References

- **Brendan Gregg — "Flame Graphs", Communications of the ACM, 59(6), 2016.** https://queue.acm.org/detail.cfm?id=2927301 — the canonical paper.
- **Brendan Gregg — "Off-CPU Analysis".** https://www.brendangregg.com/offcpuanalysis.html — the off-CPU technique.
- **Brendan Gregg — "Flame Graphs" (FlameGraph repo).** https://github.com/brendangregg/FlameGraph — `flamegraph.pl`, `difffolded.pl`, `stackcollapse-*.pl`.
- **Brendan Gregg — "Systems Performance: Enterprise and the Cloud", 2nd ed., Pearson, 2020.** Chapter 6 (CPU profiling) and Chapter 13 (perf).
- **async-profiler.** https://github.com/async-profiler/async-profiler — JVM low-overhead sampling profiler with flame-graph output.
- **0x — Single-command flame graphs for Node.js.** https://github.com/davidmarkclements/0x
- **clinic.js — Doctor, Flame, Bubbleprof for Node.js.** https://clinicjs.org/
- **bcc-tools — `offcputime`, `profile`, `cpudist`.** https://github.com/iovisor/bcc — eBPF-based profiling tools.
- **Linux `perf` documentation.** https://perf.wiki.kernel.org/ — the canonical reference for `perf record/script/report`.
- **Brendan Gregg — "FlameScope".** https://github.com/Netflix/flamescope — time-windowed flame graphs to find bursts.
- **Java Flight Recorder + JDK Mission Control.** https://docs.oracle.com/en/java/javase/21/jfapi/ — built-in JVM profiling that integrates with async-profiler.
