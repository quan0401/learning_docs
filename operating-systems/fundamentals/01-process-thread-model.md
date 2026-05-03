---
title: "Process & Thread Model — fork, exec, address spaces, and M:N threading"
date: 2026-05-03
updated: 2026-05-03
tags: [operating-systems, linux, process, thread, fork, virtual-threads, libuv]
---

# Process & Thread Model — fork, exec, address spaces, and M:N threading

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `operating-systems` `linux` `process` `thread` `fork` `virtual-threads` `libuv`

---

## Table of Contents

- [Summary](#summary)
- [1. Process vs Thread](#1-process-vs-thread)
  - [1.1 The Process Abstraction](#11-the-process-abstraction)
  - [1.2 Threads as Lightweight Processes](#12-threads-as-lightweight-processes)
  - [1.3 task_struct: One Struct to Rule Them All](#13-task_struct-one-struct-to-rule-them-all)
- [2. fork, exec, and Copy-on-Write](#2-fork-exec-and-copy-on-write)
  - [2.1 The fork()/exec() Split](#21-the-forkexec-split)
  - [2.2 clone() — the Real Underlying Syscall](#22-clone--the-real-underlying-syscall)
  - [2.3 Copy-on-Write](#23-copy-on-write)
  - [2.4 vfork and posix_spawn](#24-vfork-and-posix_spawn)
- [3. Address Space Layout](#3-address-space-layout)
  - [3.1 The Virtual Memory Map](#31-the-virtual-memory-map)
  - [3.2 Reading /proc/\<pid\>/maps](#32-reading-procpidmaps)
  - [3.3 ASLR](#33-aslr)
- [4. Kernel Threads vs User Threads](#4-kernel-threads-vs-user-threads)
  - [4.1 1:1 (Native) Threading](#41-11-native-threading)
  - [4.2 N:1 (Pure User-Level) Threading](#42-n1-pure-user-level-threading)
  - [4.3 M:N (Hybrid) Threading](#43-mn-hybrid-threading)
- [5. Green Threads in Practice](#5-green-threads-in-practice)
  - [5.1 Node.js / libuv Worker Pool](#51-nodejs--libuv-worker-pool)
  - [5.2 Java Virtual Threads (Project Loom)](#52-java-virtual-threads-project-loom)
  - [5.3 Goroutines](#53-goroutines)
- [6. Context Switching](#6-context-switching)
  - [6.1 What Happens on a Switch](#61-what-happens-on-a-switch)
  - [6.2 Cost in Real Numbers](#62-cost-in-real-numbers)
  - [6.3 Voluntary vs Involuntary](#63-voluntary-vs-involuntary)
- [7. PID, TID, and the Process Tree](#7-pid-tid-and-the-process-tree)
  - [7.1 Identifiers](#71-identifiers)
  - [7.2 Zombies and Orphans](#72-zombies-and-orphans)
  - [7.3 PID 1 in Containers](#73-pid-1-in-containers)
- [8. Inspecting Processes via /proc](#8-inspecting-processes-via-proc)
- [9. Backend Engineer Takeaways](#9-backend-engineer-takeaways)
- [Related](#related)
- [References](#references)

---

## Summary

Every server you write runs as a process — a kernel-managed container of resources (memory map, file descriptors, signal handlers, credentials) plus one or more threads of execution. Understanding the line between "the process" and "the thread", how Linux really creates them (`clone()`, not `fork()`), how copy-on-write makes `fork()/exec()` cheap, and how user-space runtimes layer M:N threading on top of 1:1 kernel threads is foundational to debugging anything from a forking shell pipeline to a Node.js worker pool exhaustion to Java virtual thread pinning. This document grounds the abstraction in actual Linux structures and shows what you can observe from `/proc`.

---

## 1. Process vs Thread

### 1.1 The Process Abstraction

A process is the kernel's bundle of:

- A **virtual address space** — its own page tables (see [Virtual Memory & Paging](02-virtual-memory-and-paging.md)).
- A **file descriptor table** (see [File Descriptors & ulimits](06-file-descriptors-and-ulimits.md)).
- **Signal handlers**, **process credentials** (uid/gid), **working directory**, **session/process group**.
- One or more **threads of execution** sharing all of the above.

When people say "process" loosely, they often mean "the address space and its main thread."

### 1.2 Threads as Lightweight Processes

A thread is a schedulable execution context: a stack, a register set (including instruction pointer), and thread-local storage. On Linux, threads share their parent's address space, fd table, and signal handlers but have their own kernel stack and TID.

The Linux kernel does not have a fundamentally different data structure for processes and threads. Both are represented by `struct task_struct`. What we call a "process" is a `task_struct` whose **thread group leader** equals itself; threads of that process are additional `task_struct`s sharing the same `tgid` (thread group id, exposed to userspace as the PID).

### 1.3 task_struct: One Struct to Rule Them All

```
+------------------------+
|  task_struct (per      |
|  schedulable entity)   |
+------------------------+
| pid     (TID in user)  |
| tgid    (PID in user)  |
| state   (R, S, D, Z…)  |
| mm      → mm_struct    | ── shared between threads of same process
| files   → files_struct | ── shared between threads
| signal  → signal_struct| ── shared
| sighand → sighand_struct| ── shared
| fs      → fs_struct    | ── shared (cwd, umask)
| stack                  | ── per-thread (kernel stack)
| sched_entity           | ── per-thread (CFS bookkeeping)
+------------------------+
```

The mm pointer is the deciding factor: two `task_struct`s with the **same** `mm` are threads of one process; with **different** `mm`s, they are separate processes.

| Field | Meaning |
|-------|---------|
| `R` | Running or runnable |
| `S` | Interruptible sleep (most blocking syscalls) |
| `D` | Uninterruptible sleep (typically waiting on disk I/O — `kill -9` will not stop it) |
| `T` | Stopped (SIGSTOP) or traced |
| `Z` | Zombie (terminated, not yet reaped) |

You can see these states in `ps -eo pid,state,comm` or `top`.

---

## 2. fork, exec, and Copy-on-Write

### 2.1 The fork()/exec() Split

Unix invented an unusual model: process creation and program loading are two separate syscalls.

```c
pid_t pid = fork();          // duplicates the calling process
if (pid == 0) {
    // child: replace this address space with a new program
    execve("/usr/bin/ls", argv, envp);
    // execve never returns on success
} else if (pid > 0) {
    // parent: pid is the child's PID
    int status;
    waitpid(pid, &status, 0); // reap the child
}
```

Why split? Because between `fork()` and `execve()` the child can adjust the inherited environment — close file descriptors, redirect stdin/stdout, change uid, set up the controlling terminal — using ordinary syscalls instead of a complex `posix_spawn`-style flag set. This is how shells implement pipelines and redirection.

### 2.2 clone() — the Real Underlying Syscall

On Linux, `fork()`, `vfork()`, and `pthread_create()` are all wrappers around `clone()`:

```c
int clone(int (*fn)(void *), void *stack, int flags, void *arg, ...);
```

The `flags` argument controls **what** is shared with the parent:

| Flag | Effect |
|------|--------|
| `CLONE_VM` | Share address space (makes it a thread, not a process) |
| `CLONE_FILES` | Share file descriptor table |
| `CLONE_FS` | Share filesystem info (cwd, umask, root) |
| `CLONE_SIGHAND` | Share signal handlers |
| `CLONE_THREAD` | Same thread group (for `getpid()` semantics) |
| `CLONE_NEWNS` | New mount namespace (containers) |
| `CLONE_NEWPID` | New PID namespace (containers) |
| `CLONE_NEWNET` | New network namespace (containers) |

`fork()` ≈ `clone(SIGCHLD, ...)` — copy everything (using CoW), separate address space.
`pthread_create()` ≈ `clone(CLONE_VM | CLONE_FILES | CLONE_FS | CLONE_SIGHAND | CLONE_THREAD | ...)`.

Container runtimes use `clone()` with `CLONE_NEW*` flags to construct namespaces.

### 2.3 Copy-on-Write

A naive `fork()` would copy every page of the parent's memory — wasteful, since most children immediately call `execve()` and discard it all. CoW solves this:

1. After `fork()`, parent and child point at the **same physical pages**, all marked **read-only** in both page tables.
2. When either side writes to a page, the CPU raises a page fault.
3. The kernel's fault handler allocates a fresh physical page, copies the contents, marks it writable in the writer's page table, and resumes execution.

Result: `fork()` cost is dominated by duplicating the page tables themselves, not the data. A 16 GB process can fork in milliseconds. Pages that are never written are never duplicated.

The relevant counter in `/proc/<pid>/status` is `VmPeak` and the per-VMA "Shared_Clean" / "Private_Dirty" breakdown in `/proc/<pid>/smaps`.

### 2.4 vfork and posix_spawn

`vfork()` shares the parent's address space and **suspends the parent** until the child calls `execve()` or `_exit()`. It avoids even the page-table copy. Modern kernels with CoW make `fork()` fast enough that `vfork()` is mostly legacy, but it survives in `posix_spawn()` implementations (glibc uses a `vfork`-like clone internally for performance).

For backend code, prefer `posix_spawn()` (or your runtime's process API) over manual `fork()/exec()`:
- Node.js `child_process.spawn()` uses `posix_spawn` where available.
- Java `ProcessBuilder` uses `posix_spawn` on Linux since JDK 11+ (controlled by `jdk.lang.Process.launchMechanism`).

---

## 3. Address Space Layout

### 3.1 The Virtual Memory Map

A 64-bit Linux process sees a 128 TB user address space (with 48-bit virtual addresses; 57-bit on newer hardware). The classic layout, low-to-high:

```
0x0000000000000000  +---------------------+
                    |  (unmapped, NULL)   |
                    +---------------------+
0x0000000000400000  | text  (.text, RX)   |  ← program code
                    +---------------------+
                    | rodata (.rodata, R) |
                    +---------------------+
                    | data (.data, RW)    |  ← initialized globals
                    | bss  (.bss,  RW)    |  ← zero-initialized globals
                    +---------------------+
                    | heap (grows up)     |  ← malloc/brk/mmap
                    | …                   |
                    +---------------------+
                    | mmap region         |  ← shared libraries, file mappings
                    | …                   |
                    +---------------------+
                    | stack (grows down)  |  ← per-thread stack
0x00007fff…         +---------------------+
                    | vDSO, vsyscall      |
0x00007fffffffffff  +---------------------+
                    | (kernel, inaccessible from user mode)
```

Each thread has its own stack region (default 8 MB on Linux, controlled by `ulimit -s`); all share the heap, text, data, and mmap regions.

### 3.2 Reading /proc/\<pid\>/maps

```
$ cat /proc/$$/maps
55b3a4c00000-55b3a4c2a000 r--p 00000000 fd:00 12345    /usr/bin/bash
55b3a4c2a000-55b3a4cf2000 r-xp 0002a000 fd:00 12345    /usr/bin/bash
55b3a4cf2000-55b3a4d2c000 r--p 000f2000 fd:00 12345    /usr/bin/bash
55b3a4d2d000-55b3a4d31000 r--p 0012c000 fd:00 12345    /usr/bin/bash
55b3a4d31000-55b3a4d3a000 rw-p 00130000 fd:00 12345    /usr/bin/bash
55b3a5e8c000-55b3a606e000 rw-p 00000000 00:00 0        [heap]
7f4c8a000000-7f4c8a022000 r--p 00000000 fd:00 67890    /usr/lib/x86_64-linux-gnu/libc.so.6
7f4c8a022000-7f4c8a1b2000 r-xp 00022000 fd:00 67890    /usr/lib/x86_64-linux-gnu/libc.so.6
…
7ffd5c8b6000-7ffd5c8d7000 rw-p 00000000 00:00 0        [stack]
7ffd5c970000-7ffd5c974000 r--p 00000000 00:00 0        [vvar]
7ffd5c974000-7ffd5c976000 r-xp 00000000 00:00 0        [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0  [vsyscall]
```

Columns: address range, permissions (rwxp/s), file offset, device, inode, path. `[heap]` and `[stack]` are anonymous mappings managed by the kernel; `[vdso]` is the virtual dynamic shared object (see [Syscalls & the ABI](05-syscalls-and-the-abi.md)).

### 3.3 ASLR

Address Space Layout Randomization randomizes the base address of stack, heap, and mmap regions on each exec, defeating attacks that rely on fixed addresses. Controlled by `/proc/sys/kernel/randomize_va_space`:

- `0` — off
- `1` — randomize stack and mmap
- `2` — also randomize heap (default)

Disabling ASLR is sometimes necessary for reproducible debugging (`setarch x86_64 -R ./prog`) but must never be done in production.

---

## 4. Kernel Threads vs User Threads

### 4.1 1:1 (Native) Threading

Each user thread maps to exactly one kernel thread (`task_struct`). The kernel scheduler sees and schedules each thread directly.

- **Used by**: pthreads on Linux (NPTL since glibc 2.3), Java platform threads, .NET threads, every modern mainstream runtime.
- **Pros**: True parallelism on multi-core, proper preemption, blocking syscalls only block one thread.
- **Cons**: Each thread costs ~1 MB stack + kernel `task_struct` overhead; context switches go through the kernel; thousands of threads = scheduling overhead and TLB pressure.

### 4.2 N:1 (Pure User-Level) Threading

Many user threads multiplexed onto a single kernel thread by a user-space scheduler.

- **Pros**: Cheap creation, fast context switch (no kernel involvement, just a `swapcontext` or coroutine yield).
- **Cons**: One blocking syscall blocks **all** threads; cannot use multiple cores.
- **Examples**: Early Java green threads (pre-1.3), classical Ruby/Python coroutines, async/await event loops at the language layer.

### 4.3 M:N (Hybrid) Threading

M user threads multiplexed onto N kernel threads, where N is typically `min(num_cpus, configured_max)`.

- **Pros**: Cheap user threads + multi-core parallelism + blocking-friendly if the scheduler can park user threads on syscalls.
- **Cons**: Implementation complexity; integration with blocking C libraries is hard.
- **Modern examples**: Go goroutines, Java virtual threads (Project Loom), Rust `tokio` (with `spawn_blocking`), Erlang/BEAM processes.

---

## 5. Green Threads in Practice

### 5.1 Node.js / libuv Worker Pool

Node.js runs JavaScript on a single main thread (the event loop). For operations that don't have async kernel APIs (file I/O on Linux pre-io_uring, DNS via `getaddrinfo`, crypto, zlib), libuv maintains a **worker pool** of OS threads (default 4, controlled by `UV_THREADPOOL_SIZE`, max 1024).

```javascript
// crypto.pbkdf2 dispatches to the libuv worker pool
const crypto = require('crypto');
crypto.pbkdf2('secret', 'salt', 100000, 64, 'sha512', (err, key) => {
  // callback runs back on the main thread when the worker finishes
});
```

The worker pool is **not** a full M:N runtime — JS user code never runs on workers (except via `worker_threads`, which is a separate, isolated V8 instance). Symptoms of pool exhaustion:

- All 4 workers busy with long DNS lookups → file reads stall behind them.
- Set `UV_THREADPOOL_SIZE=64` for I/O-heavy workloads.

See also [Async I/O Models](../../networking/network-programming/async-io-models.md).

### 5.2 Java Virtual Threads (Project Loom)

JDK 21 made virtual threads (JEP 444) GA. A virtual thread is a user-mode continuation scheduled onto a small pool of **carrier threads** (default = `Runtime.availableProcessors()`).

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i ->
        executor.submit(() -> {
            // each virtual thread is ~few KB, blocking syscalls park the VT
            // and free its carrier
            HttpResponse<String> r = client.send(req, BodyHandlers.ofString());
            log.info("got {}", r.statusCode());
        })
    );
}
```

When a virtual thread executes a blocking JDK API (HTTP, JDBC, NIO, `Thread.sleep`), the JVM **unmounts** the continuation from its carrier and parks it. The carrier picks up another runnable virtual thread. When the I/O completes, the VT becomes runnable and is mounted on any free carrier.

**Pinning** — VT cannot unmount — happens when:
- Inside a `synchronized` block (pre-JDK 24; JDK 24+ unpins synchronized too via JEP 491).
- Inside native (JNI) code holding monitors.

See [Java Virtual Threads](../../java/java-fundamentals/concurrency/virtual-threads.md) and [Spring Boot Virtual Threads](../../java/spring-virtual-threads.md) for the JVM perspective.

### 5.3 Goroutines

Go's runtime implements M:N where:
- **G** = goroutine (user thread, ~2 KB initial stack, growable)
- **M** = OS thread (kernel `task_struct`)
- **P** = logical processor (`GOMAXPROCS`, the parallelism cap)

A goroutine blocked on a syscall causes its M to be detached and a new M is spun up to keep P utilized. This is why Go gracefully scales to 100k goroutines while Java with platform threads collapses around 10k.

---

## 6. Context Switching

### 6.1 What Happens on a Switch

When the scheduler decides thread A should yield to thread B on the same CPU:

1. Save A's register state (general-purpose, FP/SIMD, segment registers) to A's kernel stack.
2. Switch the kernel stack pointer.
3. If B has a different `mm` (different process), reload `CR3` (the page table base register) — this **flushes the TLB** unless PCID/ASID is in use.
4. Restore B's registers.
5. Return to user mode at B's saved instruction pointer.

The scheduler also updates per-CPU runqueue state, accounting, and possibly cgroup bookkeeping (see [CPU Scheduling (CFS)](04-cpu-scheduling-cfs.md)).

### 6.2 Cost in Real Numbers

Direct CPU cost of a context switch is on the order of **1–5 microseconds** on modern hardware (Brendan Gregg's *Systems Performance* gives concrete numbers for various platforms). The **indirect** cost — TLB flushes, cold L1/L2 caches after the switch — is often larger and depends entirely on the working set and access pattern.

You can measure context switches per second:

```
$ vmstat 1 5
procs ---memory-- -swap- -io-- -system-- ----cpu----
 r  b   free  buff cache   si   so    bi    bo   in   cs us sy id wa
 2  0  …………                                          12345  6789 …
                                                              ↑
                                                  context switches/s
```

Or per process: `pidstat -w 1`. Sustained > 100k context switches/s on a backend process usually means lock contention or oversubscribed thread pools.

### 6.3 Voluntary vs Involuntary

- **Voluntary** — thread blocked on I/O, mutex, condition variable. Reported in `/proc/<pid>/status` as `voluntary_ctxt_switches`.
- **Involuntary** — preempted because its CFS time slice expired or a higher-priority task arrived. `nonvoluntary_ctxt_switches`.

A workload with many involuntary switches per second on every thread suggests **CPU oversubscription** — either too many threads, or a noisy neighbor (common in containers; see [CPU Scheduling (CFS)](04-cpu-scheduling-cfs.md)).

---

## 7. PID, TID, and the Process Tree

### 7.1 Identifiers

| ID | Meaning | Userspace API |
|----|---------|---------------|
| PID | Process ID — same as the TID of the thread group leader | `getpid()` |
| TID | Thread ID — unique per `task_struct` | `gettid()` |
| TGID | Thread group ID — equal to PID for the leader | `getpid()` returns TGID |
| PGID | Process group ID — used for terminal job control and `kill -PGID` | `getpgrp()` |
| SID | Session ID — terminal sessions | `getsid()` |
| PPID | Parent PID | `getppid()` |

In `ps -eLf`, the `LWP` column shows TIDs; `PID` shows TGIDs.

### 7.2 Zombies and Orphans

When a child exits, its `task_struct` lingers in **Z (zombie)** state until the parent calls `wait()`/`waitpid()` to retrieve the exit status. A zombie holds almost no resources (just the `task_struct`), but it occupies a PID slot.

- **Orphaned children** (parent died first) are reparented to PID 1 (init), which reaps them.
- **Persistent zombies** mean the parent forgot to wait — a bug. Find them with `ps -eo pid,ppid,state,comm | awk '$3=="Z"'`.

### 7.3 PID 1 in Containers

In a container with its own PID namespace, the entrypoint process is PID 1. It inherits the special init responsibilities:

- Reaping orphaned children.
- Handling default signals (the kernel does not deliver SIGTERM/SIGINT to PID 1 by default — they need explicit handlers).

Apps written assuming a normal init won't reap children → zombie buildup. Solutions:

- Use a real init: `tini`, `dumb-init`, or `--init` flag in Docker/containerd.
- Or run the app under a process supervisor.

This matters for any container that spawns subprocesses (build tools, cron-like jobs, embedded shell escapes).

---

## 8. Inspecting Processes via /proc

`/proc/<pid>/` is the canonical kernel-exposed window into a process:

| Path | Contents |
|------|----------|
| `status` | Human-readable summary: state, PPID, threads count, memory (VmPeak/VmRSS/VmSize), context switches |
| `stat` | Same data, single-line, machine-parseable (used by `ps`, `top`) |
| `maps` | Virtual memory regions (see [3.2](#32-reading-procpidmaps)) |
| `smaps` | Per-region detail: RSS, PSS, swap, shared/private clean/dirty |
| `cmdline` | Argv joined by NUL bytes |
| `environ` | Environment vars at exec time, NUL-separated |
| `cwd` | Symlink to current working directory |
| `exe` | Symlink to the executable |
| `fd/` | Per-fd symlinks → file/socket/pipe targets |
| `task/` | One subdirectory per thread (TID) |
| `limits` | All `getrlimit()` values (see [File Descriptors & ulimits](06-file-descriptors-and-ulimits.md)) |
| `cgroup` | cgroup v1/v2 paths this process belongs to |
| `stack` | Kernel-side call stack (requires CONFIG_STACKTRACE) |
| `wchan` | Kernel function the thread is sleeping in |

Useful one-liners:

```bash
# Threads of a process
ls /proc/$(pgrep -f myapp)/task

# What is the main thread waiting on?
cat /proc/$(pgrep -f myapp)/wchan

# Real memory used (RSS) in KB
awk '/VmRSS/{print $2}' /proc/$(pgrep -f myapp)/status

# All open files including sockets
ls -l /proc/$(pgrep -f myapp)/fd
```

---

## 9. Backend Engineer Takeaways

- **Threads share everything except their stack and registers.** A leaked global, a thread-unsafe library call, a missing memory barrier — these are bugs because of sharing.
- **`fork()` is cheap because of CoW**, but `fork() + write-heavy workload` is expensive. Watch this in pre-fork servers (uWSGI, Gunicorn).
- **1:1 native threads are fine until you need ~10k+ concurrent blocking operations.** Then you want virtual threads, goroutines, or async/await.
- **Pinning kills virtual thread benefits.** Audit `synchronized` blocks and JNI usage on hot paths.
- **PID 1 in a container needs reaping logic** or a tiny init like `tini`.
- **Context switch storms (>100k/s)** are a symptom — usually lock contention or oversubscription, not the root cause.
- **`/proc/<pid>/` is your friend** — every diagnosis starts there before reaching for tracers.

---

## Related

- [Virtual Memory & Paging](02-virtual-memory-and-paging.md)
- [CPU Scheduling (CFS)](04-cpu-scheduling-cfs.md)
- [Syscalls & the ABI](05-syscalls-and-the-abi.md)
- [File Descriptors & ulimits](06-file-descriptors-and-ulimits.md)
- [Java Virtual Threads](../../java/java-fundamentals/concurrency/virtual-threads.md)
- [Async I/O Models](../../networking/network-programming/async-io-models.md)
- [Socket Programming](../../networking/transport/socket-programming.md)

---

## References

- **`fork(2)`, `clone(2)`, `execve(2)`, `wait(2)` man pages** — https://man7.org/linux/man-pages/man2/fork.2.html
- **`proc(5)` man page** — comprehensive `/proc` reference. https://man7.org/linux/man-pages/man5/proc.5.html
- **Robert Love, *Linux Kernel Development*, 3rd ed.** — Chapter 3 (Process Management) and Chapter 4 (Scheduling). Authoritative kernel-level treatment.
- **Brendan Gregg, *Systems Performance*, 2nd ed.** — Chapter 6 (CPUs) covers context-switch cost, scheduler tracing, and the methodology used throughout. https://www.brendangregg.com/systems-performance-2nd-edition-book.html
- **JEP 444: Virtual Threads** — https://openjdk.org/jeps/444
- **JEP 491: Synchronize Virtual Threads without Pinning** — https://openjdk.org/jeps/491
- **libuv design overview** — http://docs.libuv.org/en/v1.x/design.html
- **Go runtime scheduler design** — Dmitry Vyukov's notes; https://golang.org/src/runtime/proc.go (read the comments).
- **LWN: "The seven stages of Linux containers"** — namespace/clone background. https://lwn.net/Articles/531114/
