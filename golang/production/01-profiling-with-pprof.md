---
title: "Profiling Go Services with pprof"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, production, pprof, profiling, performance]
---

# Profiling Go Services with pprof

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `production` `pprof` `profiling` `performance`

---

## Table of Contents

1. [The pprof toolchain](#1-the-pprof-toolchain)
2. [Profile types](#2-profile-types)
3. [Enabling pprof in a server](#3-enabling-pprof-in-a-server)
4. [CPU profile workflow](#4-cpu-profile-workflow)
5. [Heap profile workflow](#5-heap-profile-workflow)
6. [Goroutine profile — diagnosing leaks](#6-goroutine-profile--diagnosing-leaks)
7. [Block and mutex profiles](#7-block-and-mutex-profiles)
8. [Flame graphs and the web UI](#8-flame-graphs-and-the-web-ui)
9. [`runtime/trace` — different from pprof](#9-runtimetrace--different-from-pprof)
10. [Production-safe practices](#10-production-safe-practices)
11. [Comparison with V8 inspector and JVM profilers](#11-comparison-with-v8-inspector-and-jvm-profilers)

## Summary

Go ships profiling as a first-class part of the standard library. Two packages do the heavy lifting: `runtime/pprof` (programmatic — start/stop a CPU profile, write a heap snapshot to a file) and `net/http/pprof` (registers `/debug/pprof/*` HTTP handlers on the default mux for live introspection). Both produce the same wire format; both are consumed by a single CLI, `go tool pprof`, which can render text, callgraph SVGs, source listings, and interactive flame graphs.

The model is **sampling**, not tracing. CPU profiles sample the call stack at 100 Hz by default; heap profiles sample one allocation per ~512 KB. This makes profiling cheap enough to leave on in production, but it also means you cannot rely on any single sample — only on aggregates over a few seconds of capture.

This doc covers the seven profile types, the CLI commands that matter, the production caveats (the `net/http/pprof` blank import is famous for accidentally being exposed publicly), and how `runtime/trace` is a different beast from pprof — a per-goroutine event timeline rather than a sampled aggregate.

## 1. The pprof toolchain

There are three layers, and confusing them is the most common source of "I can't find the right command" frustration:

| Layer | Package / binary | What it does |
|---|---|---|
| Programmatic API | `runtime/pprof` | `StartCPUProfile`, `Lookup("heap").WriteTo`, custom profiles |
| HTTP endpoint | `net/http/pprof` | Registers handlers under `/debug/pprof/` on the default `http.ServeMux` |
| CLI | `go tool pprof` | Reads a `.pb.gz` profile (from a file or HTTP), renders text/graph/web/flamegraph |

The wire format is `pprof.proto`, originally from Google's [pprof project](https://github.com/google/pprof). Anything that emits this format — Go's standard library, the `pprof` C++ library, perf converted via `perf_data_converter` — can be opened with `go tool pprof`.

The CLI binary is just `go tool pprof`; you do not need to install anything separately. For flame graphs and the web UI, the CLI invokes `dot` (Graphviz) under the hood, so install Graphviz (`brew install graphviz` / `apt install graphviz`) once and the rest works.

## 2. Profile types

Seven predefined profiles, exposed by both `runtime/pprof.Lookup(name)` and `/debug/pprof/<name>`:

| Profile | What it samples | Default rate | Typical question it answers |
|---|---|---|---|
| **cpu** | Stack traces during on-CPU time | 100 Hz | "Where is my service spending CPU?" |
| **heap** | Live allocations (in-use bytes/objects) | 1 per ~512 KB | "What's holding memory right now?" |
| **allocs** | All allocations since program start | 1 per ~512 KB | "What's churning the GC?" |
| **goroutine** | Stack of every live goroutine | exhaustive snapshot | "Do I have a goroutine leak?" |
| **block** | Stack at points where a goroutine blocked on sync | off by default | "Where am I waiting on channels/mutexes?" |
| **mutex** | Stack of holders of contended mutexes | off by default | "Who's holding the contended lock?" |
| **threadcreate** | Stack at OS thread creation | exhaustive snapshot | "Why is this process spawning threads?" |

`heap` vs `allocs` is the subtle one: both sample memory allocation, but `heap` shows what is **currently live**, while `allocs` shows the **cumulative history**. Use `heap inuse_space` to find what's pinning RSS now; use `allocs alloc_space` to find what's churning the GC.

Block and mutex profiles are off by default because each adds runtime overhead. Enable them programmatically:

```go
runtime.SetBlockProfileRate(1)  // 1 = sample every block event; larger int = sample 1 in N nanoseconds
runtime.SetMutexProfileFraction(1)  // 1 = sample every contended unlock; larger int = sample 1 in N
```

A common production pattern is `SetBlockProfileRate(10000)` (sample 1 per 10µs of blocking) and `SetMutexProfileFraction(100)` — frequent enough to catch real contention, rare enough not to tax the runtime.

## 3. Enabling pprof in a server

The blank import on `net/http/pprof` registers handlers as a side effect:

```go
import (
    "net/http"
    _ "net/http/pprof"   // registers /debug/pprof/* on http.DefaultServeMux
)

func main() {
    // Run pprof on a separate, internal-only listener
    go func() {
        log.Println(http.ListenAndServe("127.0.0.1:6060", nil))
    }()

    // Your real service on a different port
    http.ListenAndServe(":8080", appHandler())
}
```

**Two important details that the documentation does not warn loudly about:**

1. The blank import attaches handlers to `http.DefaultServeMux`. If you use the default mux for your real application, **`/debug/pprof/*` is exposed on your public listener too.** That has been the source of many [production data exfiltration incidents](https://www.veracode.com/blog/research/exposed-pprof-default-debugging-profiler-revealing-sensitive-services). The fix: bind pprof to a separate listener bound to `127.0.0.1` or to a debug port behind authentication, as in the snippet above.
2. The endpoints expose the **command line**, the **goroutine stacks** (which can include argument values), and arbitrary **execution traces**. Treat the pprof port as you would a database admin port.

If you do not want to use the default mux at all, use the explicit registration:

```go
import "net/http/pprof"

mux := http.NewServeMux()
mux.HandleFunc("/debug/pprof/", pprof.Index)
mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
```

## 4. CPU profile workflow

A CPU profile is a sampled record of which functions were on-CPU. Collect a 30-second profile:

```bash
go tool pprof http://127.0.0.1:6060/debug/pprof/profile?seconds=30
```

The CLI fetches the profile, writes it to `~/pprof/`, and drops you into an interactive prompt:

```text
(pprof) top10
Showing nodes accounting for 5.33s, 79.31% of 6.72s total
Dropped 84 nodes (cum <= 0.03s)
      flat  flat%   sum%        cum   cum%
     1.21s 18.01% 18.01%      1.45s 21.58%  encoding/json.(*decodeState).object
     0.95s 14.14% 32.14%      1.10s 16.37%  runtime.mapaccess2_faststr
     0.62s  9.23% 41.37%      0.62s  9.23%  runtime.memmove
     ...
```

`flat` is time spent in the function itself; `cum` is time spent in the function plus everything it called. `top -cum` is what you want for "where is the call stack rooted." `top` (no flag) is what you want for "which leaf functions are hot."

Drill into a hot function:

```text
(pprof) list encoding/json.\(\*decodeState\).object
```

This shows source with sample counts per line — invaluable for spotting that a single `strings.ToLower` inside a hot loop is the actual cost.

For a top-down visual:

```bash
(pprof) web         # callgraph SVG opens in your browser
```

Or programmatic profiling for benchmarks and one-shot batch jobs:

```go
import "runtime/pprof"

f, _ := os.Create("cpu.prof")
defer f.Close()
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()
// ... workload to profile ...
```

Then `go tool pprof ./yourbinary cpu.prof`. The binary path is needed so the CLI can resolve symbols; if the profile was collected from an HTTP endpoint of the same binary, the CLI can fetch symbols from `/debug/pprof/symbol` automatically.

## 5. Heap profile workflow

Heap profiles answer "what is using memory right now?":

```bash
go tool pprof http://127.0.0.1:6060/debug/pprof/heap
```

By default, the heap profile shows `inuse_space` — bytes allocated by samples that have **not yet been garbage collected**. Switch views inside the CLI:

```text
(pprof) sample_index = inuse_space    # bytes currently live (default)
(pprof) sample_index = inuse_objects  # object count currently live
(pprof) sample_index = alloc_space    # cumulative bytes ever allocated
(pprof) sample_index = alloc_objects  # cumulative object count
```

The four views answer different questions. A struct that is large but rare may dominate `inuse_space` while being invisible in `alloc_objects`. A tiny struct allocated millions of times per second (every JSON decode, say) dominates `alloc_objects` while being trivial in `inuse_space` — but it's exactly what the GC cares about.

A common debugging session: RSS is climbing in production over the course of a day. Take a heap profile at hour 1 and another at hour 6, then diff:

```bash
go tool pprof -base hour1.heap hour6.heap
```

The `-base` flag subtracts the first profile from the second — the result shows only what was added between them, which is usually exactly the leak.

For allocation churn (GC pressure), use the allocs profile and look at `alloc_space`:

```bash
go tool pprof http://127.0.0.1:6060/debug/pprof/allocs
(pprof) top10
```

The top entries here are usually the levers for [reducing escape-analysis-driven heap allocation](../runtime/02-escape-analysis-and-allocation.md) — preallocating a slice, switching from `fmt.Sprintf` to a `strings.Builder`, or returning a pointer instead of a value.

## 6. Goroutine profile — diagnosing leaks

The goroutine profile is exhaustive (not sampled): one entry per live goroutine, grouped by stack. Two ways to capture:

```bash
# Compact summary, grouped by stack with counts
curl http://127.0.0.1:6060/debug/pprof/goroutine?debug=1

# Full text dump, every goroutine listed individually with full stack
curl http://127.0.0.1:6060/debug/pprof/goroutine?debug=2

# Binary protobuf for go tool pprof analysis
go tool pprof http://127.0.0.1:6060/debug/pprof/goroutine
```

The `debug=1` form is the workhorse for goroutine leak diagnosis. The output looks like:

```text
goroutine profile: total 14182
13950 @ 0x103c8c5 0x10038c5 ...
#       0x...   net/http.(*persistConn).readLoop+0x...
#       0x...   net/http.(*Transport).dialConn+0x...
```

If `total` is climbing over time and one stack accounts for most of them, that's your leak. The fix is almost always missing `context` cancellation or a missing `defer` to close a channel — see the [goroutine lifecycle and leak patterns](../concurrency/01-goroutines-and-scheduler.md) for the catalog.

A useful production pattern is to scrape `/debug/pprof/goroutine?debug=1` periodically and feed the count into a metric. A steadily climbing goroutine count is the canonical leading indicator of a leak.

## 7. Block and mutex profiles

These two profiles cover the time goroutines spend **not running**:

- **Block profile** records stacks where goroutines blocked on synchronization primitives — channels, `sync.Mutex.Lock`, `sync.WaitGroup.Wait`, `time.Sleep`, etc.
- **Mutex profile** records stacks of goroutines that **held** a contended mutex (i.e., another goroutine had to wait for it).

Both are off by default. Enable them:

```go
runtime.SetBlockProfileRate(10000)         // sample one event per 10 µs of blocking
runtime.SetMutexProfileFraction(100)       // sample 1 of every 100 contended unlocks
```

Then collect:

```bash
go tool pprof http://127.0.0.1:6060/debug/pprof/block
go tool pprof http://127.0.0.1:6060/debug/pprof/mutex
```

When does this matter? When CPU is low and latency is high — the classic "the box is bored but my P99 is bad" pattern. CPU profiling will show nothing (because the goroutines are not on CPU). Block and mutex profiling show exactly where they are waiting.

A real example: a service had a `sync.Mutex` around a logging context. Every request acquired it briefly. CPU profile was clean. Mutex profile showed every request stacking up on that one lock. The fix was switching to a `sync.RWMutex` and avoiding the lock entirely on the hot path.

## 8. Flame graphs and the web UI

Brendan Gregg's flame graphs are now the most-used visualization for sampled profiles. `go tool pprof` ships them:

```bash
go tool pprof -http=:8080 http://127.0.0.1:6060/debug/pprof/profile?seconds=30
```

This launches a local web UI that includes:

- **Top** — the same `top` table.
- **Graph** — the call graph SVG.
- **Flame Graph** — the icicle-style flame graph (top-down).
- **Source** — annotated source listings.
- **Peek** — focused view on a single function's callers/callees.

The flame graph reads top-down: the top bar is the entry point, each layer below is a function it called. The **width** of a bar is proportional to time; the **vertical position** is just the call stack. Look for wide bars high up — those are functions that account for a lot of time without much delegation. Look for plateaus — repeated wide bars at the same level suggest a hot loop.

For more on flame graph reading, see Brendan Gregg's reference page in the references below. The conceptual model is identical for V8, JFR, perf, and pprof — the visualization is a portable tool, not a Go-specific one.

## 9. `runtime/trace` — different from pprof

`runtime/trace` is **not** a sampled profile. It records **every** scheduling event, GC pause, syscall enter/exit, and goroutine state transition with nanosecond-precision timestamps. The output is a per-goroutine timeline.

Capture a 5-second trace:

```bash
curl -o trace.out 'http://127.0.0.1:6060/debug/pprof/trace?seconds=5'
go tool trace trace.out
```

The browser opens to a multi-tab UI:

- **View trace** — a Chrome-style timeline showing each P (logical processor) and the goroutines running on it, with GC overlays.
- **Goroutine analysis** — per-goroutine breakdown of scheduling latency, GC time, syscall time, on-CPU time.
- **Network blocking profile**, **Synchronization blocking profile**, **Syscall blocking profile** — pprof-style flame graphs but rooted in "where did goroutines wait."
- **Scheduler latency profile** — where the runtime took time getting a runnable goroutine onto a P.

Use cases:

- "GC pauses are too long" — the GC overlay on the timeline shows pause durations and which Ps were involved.
- "P99 latency spikes randomly" — find the slow request in the goroutine analysis, then jump to its lifeline on the timeline.
- "Why isn't this parallel workload using all my CPUs?" — the timeline shows whether Ps are idle and goroutines are pending, or whether goroutines are not being created.

The cost of tracing is non-trivial — a few percent of CPU during the trace, plus a large output file (a few hundred MB for a busy 30-second capture). It's a "turn it on, capture briefly, turn it off" tool, not an always-on instrument like pprof's HTTP endpoint.

The contrast with pprof: pprof tells you **where** time is spent (aggregates of stacks), trace tells you **when** and **why a goroutine wasn't running** (per-goroutine, time-ordered).

## 10. Production-safe practices

A short checklist for running pprof safely in production:

- **Bind the pprof listener to localhost or a private interface.** Never expose `/debug/pprof/*` on a public listener. The CPU profile alone reveals enough of your call graph to be a reconnaissance gift to an attacker.
- **If you must expose pprof remotely, gate it behind authentication.** A separate, mutually-authenticated debug port. Or, more simply, require shelling onto the host and using `kubectl port-forward`.
- **Keep block/mutex profile rates conservative.** `SetBlockProfileRate(10000)` and `SetMutexProfileFraction(100)` are reasonable defaults. Setting them to 1 captures everything but adds measurable overhead.
- **CPU profiling at 100 Hz is cheap.** It is fine to leave the endpoint enabled and capture occasional 30-second profiles in production. It is **not** fine to capture continuously.
- **`runtime/trace` is expensive.** Use it for short, deliberate captures. Leaving it always on is not a thing.
- **Profile under realistic load.** A profile of an idle service shows you nothing useful. Capture during peak traffic, or under a load test.
- **Diff profiles.** A single profile is hard to read. Two profiles, one before and one after a change (or one at a known-good time and one during a regression), make problems obvious.

## 11. Comparison with V8 inspector and JVM profilers

For a TS/Node and Java/Spring Boot dev, the conceptual mapping helps:

| Need | Go | Node.js / V8 | JVM |
|---|---|---|---|
| CPU sampling | `pprof` cpu profile | `--inspect` + Chrome DevTools or `--prof` | async-profiler, JFR |
| Heap snapshot | `pprof` heap profile | `--inspect` heap snapshot, `v8.writeHeapSnapshot()` | JFR, `jmap -dump` + Eclipse MAT |
| Allocation tracking | `pprof` allocs | `--inspect` allocation profiler | JFR allocation events |
| Live blocking/contention | block + mutex profiles | (limited; mostly via tracing) | JFR thread events, async-profiler `lock` mode |
| Per-goroutine/thread timeline | `runtime/trace` | `--inspect` performance timeline | JFR timeline, `jstack` for snapshots |
| Always-on, low-overhead production profiling | pprof endpoint at 100 Hz | `--inspect-brk` is for dev; production uses 0x or Clinic.js | JFR at default rate (~1% overhead) |
| Endpoint format | `pprof.proto` | V8 cpuprofile / heapsnapshot JSON | JFR binary, HPROF binary |

A few specifics worth knowing:

- **Go pprof corresponds most closely to JFR + Eclipse MAT.** Both are sampled, both are designed to run in production, both produce a self-contained file you can analyze offline. JFR has a richer event taxonomy (it records JIT compilations, class loads, lock events as discrete typed events); pprof is conceptually simpler but covers the cases that matter for application code.
- **`net/http/pprof` is similar in spirit to Spring Boot Actuator's `/actuator/heapdump` and `/actuator/threaddump`** — an HTTP-exposed introspection surface. Both have the same "do not expose this on the public internet" warning. Spring Actuator at least has Spring Security integration baked in; Go's pprof requires you to bring your own auth.
- **Node's `--inspect` is interactive-debugger-first.** Profiling is a sub-feature of the Chrome DevTools UI. Go pprof is profiling-first, with no debugger integration (use Delve for that). The advantage: pprof works against a running production process without altering its execution model. The Node inspector pauses execution.
- **Compared to `--prof`**, Node's older flat-tick profiler, pprof is roughly equivalent in concept but better integrated. `--prof` produces a tick log you have to post-process with `--prof-process`; pprof produces a self-contained file with embedded symbols, ready for the CLI.

## Related

- [Slices, Arrays & Maps](../fundamentals/04-slices-arrays-maps.md) — `pprof allocs` is how you find the allocation pressure points
- [Goroutines & the Scheduler (G-M-P)](../concurrency/01-goroutines-and-scheduler.md) — the model that `runtime/trace` visualizes
- [The Garbage Collector](../runtime/01-garbage-collector.md) — how to read GC events in `runtime/trace`
- [Escape Analysis and Allocation](../runtime/02-escape-analysis-and-allocation.md) — what to do once `pprof` shows you a hot allocation site
- [Performance — Latency and Profiling](../../performance/INDEX.md) — flame graphs and percentiles in general
- [Observability for Go Services](02-observability.md) — the next step after profiling: long-term telemetry
- [TypeScript — Profiling and Debugging](../../typescript/production/profiling-and-debugging.md) — V8 inspector / `--prof` workflow

## References

- `net/http/pprof` package documentation — https://pkg.go.dev/net/http/pprof
- `runtime/pprof` package documentation — https://pkg.go.dev/runtime/pprof
- `runtime/trace` package documentation — https://pkg.go.dev/runtime/trace
- "Profiling Go Programs" (Go blog) — https://go.dev/blog/pprof
- "Diagnostics" official Go documentation — https://go.dev/doc/diagnostics
- pprof project (the format and the original tool) — https://github.com/google/pprof
- Brendan Gregg, "Flame Graphs" — https://www.brendangregg.com/flamegraphs.html
- `go tool pprof` reference — https://github.com/google/pprof/blob/main/doc/README.md
- Veracode, "Exposed pprof: the default Go debugger" (the security incident category) — https://www.veracode.com/blog/research/exposed-pprof-default-debugging-profiler-revealing-sensitive-services
