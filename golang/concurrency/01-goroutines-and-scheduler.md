---
title: "Goroutines & the Scheduler (G-M-P)"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, concurrency, scheduler, goroutines, runtime, gmp]
---

# Goroutines & the Scheduler (G-M-P)

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `concurrency` `scheduler` `goroutines` `runtime` `gmp`

---

## Table of Contents

1. [What a goroutine is, vs an OS thread and a Java virtual thread](#1-what-a-goroutine-is-vs-an-os-thread-and-a-java-virtual-thread)
2. [The G-M-P model](#2-the-g-m-p-model)
3. [`GOMAXPROCS` and the parallelism dial](#3-gomaxprocs-and-the-parallelism-dial)
4. [Run queues and work stealing](#4-run-queues-and-work-stealing)
5. [Network I/O: the netpoller](#5-network-io-the-netpoller)
6. [Blocking syscalls: M handoff](#6-blocking-syscalls-m-handoff)
7. [Preemption: from cooperative to asynchronous](#7-preemption-from-cooperative-to-asynchronous)
8. [`sysmon`: the runtime's watchdog](#8-sysmon-the-runtimes-watchdog)
9. [`runtime.LockOSThread` and CGo](#9-runtimelockosthread-and-cgo)

## Summary

Goroutines are user-space tasks scheduled onto OS threads by the Go runtime. The scheduler is an M:N design — many goroutines (G) multiplexed onto a smaller pool of OS threads (M), with parallelism gated by a logical-processor count (P) defaulting to `runtime.NumCPU()`. Goroutines start with a small stack that grows on demand, switching between them costs roughly nanoseconds rather than microseconds, and the runtime turns blocking network I/O into non-blocking polling under the hood so a single OS thread can serve thousands of concurrent connections.

This doc explains the moving parts a backend engineer needs to reason about: G-M-P, run queues, the netpoller, syscall handoff, and the asynchronous preemption that arrived in Go 1.14. It does not cover channels (next doc) or the memory model (later doc).

## 1. What a goroutine is, vs an OS thread and a Java virtual thread

A goroutine is created with the `go` keyword:

```go
go doWork()
go func() { /* ... */ }()
```

The runtime allocates a goroutine struct (`g` in `runtime/runtime2.go`) and a small initial stack of a few kilobytes that grows and shrinks dynamically. The pre-Go 1.4 runtime used segmented stacks with chunked extension; since Go 1.4 the runtime uses **contiguous stacks** that copy and resize on overflow ("morestack" / "copystack"), avoiding the "hot split" cost of the older design.

Comparing user-space concurrency primitives:

| | OS thread (Linux NPTL) | Goroutine | Java virtual thread (Loom, JDK 21+) |
|---|---|---|---|
| Default stack | 8 MB virtual (lazy) | a few KB, grows on demand | a few KB on heap, growable |
| Creation cost | ~10–100 µs (kernel call) | ~hundreds of ns | comparable to goroutines |
| Context switch | ~1–10 µs (kernel involvement) | ~tens to hundreds of ns | similar to goroutines |
| Scheduler | kernel CFS | Go runtime (M:N) | JVM ForkJoinPool carrier threads (M:N) |
| Blocking I/O | blocks the thread | runtime hooks redirect to netpoller | runtime hooks unmount from carrier |
| Per-process limit (typical) | thousands | millions | millions |

The headline number — "you can have a million goroutines" — comes from the per-goroutine memory footprint plus stack-growth-on-demand, not from the scheduler. The scheduler then ensures only a `GOMAXPROCS`-sized subset is running on actual CPUs at any moment.

## 2. The G-M-P model

Three runtime structures, all visible in `runtime/runtime2.go`:

| Letter | Stands for | What it represents |
|---|---|---|
| **G** | goroutine | One unit of work — stack, instruction pointer, scheduling state |
| **M** | machine | An OS thread, created by the runtime via `pthread_create` (Linux) |
| **P** | processor | A logical resource that an M needs to hold in order to execute Go code; count fixed to `GOMAXPROCS` |

The invariant: a G runs on an M only when that M holds a P. P holds the **local run queue** of runnable Gs. There is also a **global run queue** for overflow.

```text
                    ┌─────────────┐    ┌─────────────┐
                    │      M      │    │      M      │
                    │ (OS thread) │    │ (OS thread) │
                    └──────┬──────┘    └──────┬──────┘
                           │                   │
                    ┌──────▼──────┐    ┌──────▼──────┐
                    │      P      │    │      P      │
                    │ run queue:  │    │ run queue:  │
                    │ G G G G     │    │ G G G       │
                    └─────────────┘    └─────────────┘

                    Global run queue: G G G ...
                    Network poller:   waiting Gs ...
```

Three states for an M:
- **Executing Go code** — holds a P and is running a G.
- **In a syscall** — released its P (see §6), but the M itself is alive.
- **Idle / parked** — sleeping on a futex, waiting for work.

Three states for a G:
- **Runnable** — sitting on a run queue, waiting for an M with a P.
- **Running** — actively executing on an M.
- **Waiting** — blocked on a channel send/receive, mutex, network I/O, timer, etc. Off all run queues until the wake event.

## 3. `GOMAXPROCS` and the parallelism dial

`GOMAXPROCS` sets the number of P structures, and therefore the maximum number of OS threads that can execute Go code simultaneously. Since Go 1.5, the default is `runtime.NumCPU()` — the number of logical CPUs visible to the process at startup. Before 1.5 it defaulted to 1, which is one of the most common gotchas in old Go code lifted forward.

You can change it at runtime:

```go
import "runtime"

old := runtime.GOMAXPROCS(4)   // returns the previous value
fmt.Println("was", old)
```

…or at process start with the `GOMAXPROCS` environment variable.

Common reasons to override the default:
- **Containers with CPU limits.** Go reads `runtime.NumCPU()` from the kernel, which sees the host's CPU count, not the container's CPU quota. A pod with `cpu: "2"` running on a 64-core node will start with `GOMAXPROCS=64`, oversubscribing every scheduler decision. The community fix is [`uber-go/automaxprocs`](https://github.com/uber-go/automaxprocs), which reads cgroup quotas at startup and sets `GOMAXPROCS` accordingly.
- **Latency-sensitive workloads.** Lowering `GOMAXPROCS` can reduce GC pause variance and L1/L2 cache thrashing on highly parallel boxes.

`GOMAXPROCS` does not cap the **total** number of OS threads. The runtime spawns extra threads as needed (e.g., for blocked syscalls, see §6); only the threads actively executing Go code are bounded by `GOMAXPROCS`.

## 4. Run queues and work stealing

Each P has a **local run queue** — a fixed-size (256 entries) circular buffer of runnable Gs. New goroutines created from running code go on the local queue of the current P. This is fast: no atomics needed for local-only operations.

The runtime also has a **global run queue**, protected by a mutex, for:
- Overflow when a local queue is full.
- Goroutines awoken from waits with no obvious P affinity.

When a P's local queue empties, its M does **work stealing**:

1. Try to grab a G from the global run queue.
2. Try to poll the network poller for ready Gs (see §5).
3. Steal half of another P's local queue.
4. If all P queues are empty, give up the P and park the M.

The "steal half" rule is the canonical work-stealing strategy from Cilk and many descendants — it amortizes the cost of stealing across multiple subsequent local schedules. The [original scheduler design doc](https://golang.org/s/go11sched) by Dmitry Vyukov walks through this.

To prevent starvation of the global queue, the scheduler periodically (every 61st schedule on a P) checks the global queue first instead of the local queue. The constant 61 is in `runtime/proc.go` and is small-and-coprime so it doesn't accidentally synchronize with batch sizes.

## 5. Network I/O: the netpoller

The single biggest reason Go services scale to many thousands of concurrent connections on a small thread pool: **blocking network I/O on a goroutine does not block the M**.

When you write `conn.Read(buf)`, internally:
1. The goroutine calls into the runtime's network read.
2. The runtime issues a non-blocking read against the underlying file descriptor.
3. If the syscall returns `EAGAIN` (no data available), the runtime parks the goroutine, registers the fd with the OS-specific I/O notification mechanism, and returns the M to the scheduler.
4. The M picks up another runnable G.
5. When the fd becomes readable, the netpoller produces the goroutine back into a run queue.

The notification mechanism is platform-specific:
- Linux: `epoll`
- BSD / macOS: `kqueue`
- Windows: I/O Completion Ports

This is the same mechanism libuv uses for Node.js — see the [Operating Systems doc on epoll/kqueue/io_uring](../../operating-systems/fundamentals/07-epoll-kqueue-iouring.md) for the kernel side. The difference is that in Node you **explicitly** write callback or `await` chains; in Go the model is hidden behind a synchronous API. From the goroutine's perspective, `Read` blocks; from the OS's perspective, the M is free to do other work.

Tools like [`netpolldescriptor`](https://pkg.go.dev/runtime#NumGoroutine) and `GODEBUG=schedtrace=1000` show this in action.

## 6. Blocking syscalls: M handoff

The netpoller covers network I/O. **Other** blocking syscalls — file I/O on most filesystems, DNS resolution via cgo, calls into C code — actually block the OS thread. The runtime handles this with **M-P handoff**:

1. The G enters a syscall via the runtime's `entersyscall`.
2. The runtime detaches the P from the M and lists the M as "in syscall."
3. If there are runnable Gs and no idle M with a P, the runtime spawns or wakes a fresh M to attach to the now-free P, so other Gs can keep running.
4. When the syscall returns, the runtime tries to reattach the original M to a P. If none is available (because step 3 already used it), the G is queued and the M is parked.

The consequence: a Go program with many blocking file reads can end up with significantly more OS threads than `GOMAXPROCS`. This is **expected**, not a leak. `runtime.NumGoroutine()` shows goroutine count; `runtime/debug.SetMaxThreads` (default 10,000) caps the OS thread count and crashes the process if exceeded — a guardrail against runaway thread growth from misuse.

If your program creates many goroutines that all do blocking syscalls, consider:
- Whether the underlying I/O could be done via the netpoller-friendly stdlib calls.
- Whether you need a worker pool to bound concurrency at the I/O layer.

## 7. Preemption: from cooperative to asynchronous

Until Go 1.14, the scheduler was **cooperative**. A goroutine yielded to the scheduler only at:
- Channel operations.
- Function call prologues (which check the stack-grow flag).
- Explicit `runtime.Gosched()`.
- Blocking syscalls.

The pathological case: a goroutine running a tight loop with no function calls — e.g., `for {}` — could starve the entire scheduler and hang GC indefinitely, because GC requires every goroutine to reach a safe point so the runtime can stop the world.

Go 1.14 (February 2020) introduced **asynchronous preemption**: the runtime sends `SIGURG` to a thread running a goroutine that has been on-CPU too long; the signal handler checks if the goroutine is at a preemptable point and, if not, redirects execution to a function that yields. This is the [non-cooperative goroutine preemption proposal](https://github.com/golang/proposal/blob/master/design/24543-non-cooperative-preemption.md) by Austin Clements.

Practical effect: the `for {}` scheduler-hang bug is gone. GC stop-the-world phases are bounded. You will likely never have to think about this in application code, but if you find a profile dominated by `runtime.asyncPreempt`, the underlying goroutine is doing tight work that would have hung the scheduler in pre-1.14 Go.

## 8. `sysmon`: the runtime's watchdog

A dedicated OS thread, `sysmon`, runs without a P. It wakes periodically (the interval grows from microseconds to milliseconds based on activity) and:
- Polls the netpoller in case it has been idle for too long.
- Retakes Ps from Ms blocked in syscalls (so other goroutines can use them).
- Detects goroutines that have been on-CPU for too long and triggers asynchronous preemption.
- Forces GC if no GC has happened in the last two minutes.

`sysmon` is why Go programs do not need a dedicated GC thread or I/O thread at the application level — it acts as both the scheduler's heartbeat and the runtime's last-resort enforcer.

## 9. `runtime.LockOSThread` and CGo

A goroutine can pin itself to its current OS thread:

```go
runtime.LockOSThread()
defer runtime.UnlockOSThread()
```

After `LockOSThread`, that M will run only this goroutine until `UnlockOSThread`. Other goroutines that need a thread will get a different M. Required when:
- The goroutine calls a C library that uses thread-local state (most OpenGL contexts, some GPU/graphics APIs).
- The goroutine sets the OS thread's state via syscall (`setns`, `unshare`, signal masks).
- The goroutine is `main` and is required by an external framework to stay on the original thread.

The `main` goroutine is a special case: the runtime starts on thread 0 and the program's `main` function runs there. Many GUI frameworks and a few system libraries assume thread 0 is special; `LockOSThread` from `init()` plus running the framework loop on `main` is the common pattern.

CGo calls implicitly cross the Go/C boundary on the same M for the duration of the call. The runtime detaches the P (as in §6) so other Gs keep running. Heavy CGo workloads can therefore inflate OS thread count similarly to blocking syscalls.

## Related

- [Slices, Arrays & Maps](../fundamentals/04-slices-arrays-maps.md) — concurrent map writes are a fatal runtime error
- [Pointers, Methods & Receivers](../fundamentals/07-pointers-methods-receivers.md) — copying a `sync.Mutex` is a data race the goroutine model exposes immediately
- [Operating Systems — Process & Thread Model](../../operating-systems/fundamentals/01-process-thread-model.md) — kernel side of M:N threading
- [Operating Systems — epoll / kqueue / io_uring](../../operating-systems/fundamentals/07-epoll-kqueue-iouring.md) — the OS notification mechanism the netpoller sits on top of
- [TypeScript — Event Loop Internals](../../typescript/runtime/event-loop-internals.md) — single-threaded async model, contrast with M:N
- [TypeScript — Worker Threads & Concurrency](../../typescript/runtime/worker-threads.md) — TS/Node's answer to multi-core parallelism

## References

- The Go Programming Language Specification, "Go statements" — https://go.dev/ref/spec#Go_statements
- Dmitry Vyukov, "Scalable Go Scheduler Design Doc" (2012) — https://golang.org/s/go11sched (the canonical design document for the current scheduler)
- Austin Clements, "Proposal: Non-cooperative goroutine preemption" (2018) — https://github.com/golang/proposal/blob/master/design/24543-non-cooperative-preemption.md
- Go 1.14 release notes (asynchronous preemption shipped) — https://go.dev/doc/go1.14#runtime
- Go 1.5 release notes (`GOMAXPROCS` default became `NumCPU()`) — https://go.dev/doc/go1.5#runtime
- Go 1.4 release notes (contiguous stacks replaced segmented stacks) — https://go.dev/doc/go1.4#runtime
- `runtime` package documentation — https://pkg.go.dev/runtime
- `GODEBUG` runtime instrumentation flags (`schedtrace`, `scheddetail`) — https://pkg.go.dev/runtime#hdr-Environment_Variables
- `uber-go/automaxprocs` — https://github.com/uber-go/automaxprocs (cgroup-aware GOMAXPROCS for containers)
- The Go source tree, `src/runtime/proc.go` — the scheduler implementation; the file's top-of-file comment is itself a useful overview. https://github.com/golang/go/blob/master/src/runtime/proc.go
- JEP 444: Virtual Threads (Java 21) — https://openjdk.org/jeps/444 (for the Java comparison in §1)
