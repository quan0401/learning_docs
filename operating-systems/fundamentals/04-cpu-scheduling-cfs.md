---
title: "CPU Scheduling (CFS) — runqueues, vruntime, cgroups, throttling"
date: 2026-05-03
updated: 2026-05-03
tags: [operating-systems, linux, scheduler, cfs, cgroup, throttling, numa]
---

# CPU Scheduling (CFS) — runqueues, vruntime, cgroups, throttling

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `operating-systems` `linux` `scheduler` `cfs` `cgroup` `throttling` `numa`

---

## Table of Contents

- [Summary](#summary)
- [1. Scheduler Classes](#1-scheduler-classes)
- [2. CFS — The Completely Fair Scheduler](#2-cfs--the-completely-fair-scheduler)
  - [2.1 vruntime and the red-black tree](#21-vruntime-and-the-red-black-tree)
  - [2.2 Time slices](#22-time-slices)
  - [2.3 Sleeper bonus and wakeup latency](#23-sleeper-bonus-and-wakeup-latency)
- [3. Niceness and Weights](#3-niceness-and-weights)
- [4. Real-Time Classes](#4-real-time-classes)
  - [4.1 SCHED_FIFO and SCHED_RR](#41-sched_fifo-and-sched_rr)
  - [4.2 SCHED_DEADLINE](#42-sched_deadline)
  - [4.3 RT throttling](#43-rt-throttling)
- [5. cgroup CPU Control](#5-cgroup-cpu-control)
  - [5.1 cpu.weight (v2) / cpu.shares (v1)](#51-cpuweight-v2--cpushares-v1)
  - [5.2 cpu.max — bandwidth quotas](#52-cpumax--bandwidth-quotas)
  - [5.3 The container CPU-throttling problem](#53-the-container-cpu-throttling-problem)
- [6. CPU Affinity and Pinning](#6-cpu-affinity-and-pinning)
- [7. isolcpus, nohz_full, and Tickless Cores](#7-isolcpus-nohz_full-and-tickless-cores)
- [8. Inspecting the Scheduler](#8-inspecting-the-scheduler)
  - [8.1 schedstat](#81-schedstat)
  - [8.2 Pressure Stall Information (PSI)](#82-pressure-stall-information-psi)
  - [8.3 perf sched](#83-perf-sched)
- [9. Why Noisy Neighbors Happen in Containers](#9-why-noisy-neighbors-happen-in-containers)
- [10. Backend Engineer Takeaways](#10-backend-engineer-takeaways)
- [Related](#related)
- [References](#references)

---

## Summary

The Linux scheduler decides which `task_struct` runs on each CPU at each moment. For backend services, the relevant scheduler is **CFS** (Completely Fair Scheduler) — a vruntime-based weighted-fair-share scheduler that preempts tasks roughly every few milliseconds and is integrated with cgroups for container-grade resource control. Understanding CFS clears up why your Spring Boot pod gets throttled under a CPU limit despite low average utilization, why pinning a latency-sensitive service to dedicated cores helps, and what `cpu.weight` actually buys you. This document covers CFS internals, the niche real-time classes, cgroup CPU control, and the diagnostic interfaces (`schedstat`, PSI, `perf sched`).

---

## 1. Scheduler Classes

Linux organizes schedulable entities into **classes**, consulted in priority order:

| Class | Policy values | Used by |
|-------|---------------|---------|
| `stop_sched_class` | (internal) | Migration threads, machine-check handlers |
| `dl_sched_class` | `SCHED_DEADLINE` | Soft real-time with deadline + budget + period |
| `rt_sched_class` | `SCHED_FIFO`, `SCHED_RR` | Hard real-time (audio, robotics) — fixed priority 1–99 |
| `fair_sched_class` | `SCHED_NORMAL` (a.k.a. `SCHED_OTHER`), `SCHED_BATCH`, `SCHED_IDLE` | Almost all userspace processes — CFS |
| `idle_sched_class` | (internal) | Per-CPU idle thread |

When the scheduler picks the next task on a CPU, it asks each class in order; CFS only runs if no RT or DEADLINE task is runnable. Almost every backend workload — Java services, Node servers, Postgres, Redis — runs under `SCHED_NORMAL` and is managed by CFS.

---

## 2. CFS — The Completely Fair Scheduler

CFS replaces the traditional time-slice scheduler with a model that, conceptually, gives every runnable task an equal share of CPU time. It does this without explicit time slices by tracking each task's **vruntime**.

### 2.1 vruntime and the red-black tree

Each schedulable entity has a `vruntime` (virtual runtime, in ns) that increases as the task runs:

```
delta_vruntime = delta_actual_runtime * (NICE_0_LOAD / task_weight)
```

A task with `nice=0` (weight 1024) accrues vruntime 1:1 with wall time. A higher-priority (`nice=-5`, larger weight) task accrues vruntime *slower* — so it falls behind in vruntime and the scheduler picks it more often. A `nice=+5` task accrues faster and is picked less.

Each per-CPU runqueue holds a **red-black tree** keyed on vruntime. The leftmost node (smallest vruntime) is always next to run.

```
                   (per-CPU runqueue)
                        ┌─────────────┐
                        │  rbtree     │
                        │  (sorted by │
                        │   vruntime) │
                        │             │
   leftmost ───┐        │   12, 18,   │
   "next"      │        │   25, 30… │ ◄── enqueue at correct position
               ▼        │             │
           ┌──────────┐ └─────────────┘
           │ task A   │ runs on CPU; vruntime increments;
           │ vrt=12   │ on tick (or higher-prio wakeup), re-queued.
           └──────────┘
```

CFS picks the leftmost node, runs it for at most some target time, then re-inserts based on the new vruntime.

### 2.2 Time slices

CFS does not use a fixed time slice. Instead it maintains a **target latency** — a window during which it tries to run every runnable task at least once. Two relevant tunables:

```
$ sysctl kernel.sched_min_granularity_ns      # 750000   (0.75 ms) — min slice per task
$ sysctl kernel.sched_latency_ns              # 6000000  (6   ms) — target window for ≤8 tasks
```

When the runqueue has many tasks, CFS starts shrinking each task's effective slice toward `sched_min_granularity_ns`. With hundreds of runnable tasks per CPU, slices become tiny and context-switch overhead grows.

(Note: `kernel.sched_*_ns` knobs require `CONFIG_SCHED_DEBUG=y` and may not be exposed on all distros.)

### 2.3 Sleeper bonus and wakeup latency

When a task wakes from sleep (e.g., its blocking `read()` returned), CFS adjusts its vruntime so it isn't unfairly behind — but caps the bonus to prevent woken tasks from monopolizing the CPU. This is what makes CFS reasonably responsive for interactive workloads despite its "fairness" framing.

The **EEVDF scheduler** (Earliest Eligible Virtual Deadline First, Linux 6.6+) replaces classic CFS internals with a deadline-flavored variant that improves latency fairness. Userspace API and tooling are unchanged.

---

## 3. Niceness and Weights

`nice` values range from -20 (highest priority) to +19 (lowest). Each nice value maps to a weight; weights differ by ~10% per nice level (the table is in `kernel/sched/core.c`). Concretely:

| nice | weight | relative share with one nice-0 peer |
|------|--------|--------------------------------------|
| -20 | 88761 | ~99% vs ~1% |
| -10 | 9548 | ~90% vs ~10% |
| -5  | 3121 | ~75% vs ~25% |
| 0   | 1024 | 50% / 50% |
| +5  | 335  | ~25% vs ~75% |
| +10 | 110  | ~10% vs ~90% |
| +19 | 15   | ~1.5% vs ~98.5% |

Set with `nice -n 5 ./prog` or `renice -n 10 -p <pid>`. Lowering nice (raising priority) below 0 requires `CAP_SYS_NICE` (root, or capabilities-granted).

Weight matters only **between siblings on the same runqueue**. A nice-0 task on a quiet CPU still gets 100% of that CPU.

---

## 4. Real-Time Classes

### 4.1 SCHED_FIFO and SCHED_RR

Both use static priorities 1–99 (higher = higher priority). They preempt everything in `SCHED_NORMAL`.

- **SCHED_FIFO** — runs until it blocks or yields. No time slicing within priority.
- **SCHED_RR** — round-robin among same-priority tasks with a small slice (`sched_rr_timeslice_ms`).

Used in audio servers, robotics, market-data feed handlers, kernel threads that must respond promptly. **A bug in a SCHED_FIFO task can lock a CPU forever** — there's no preemption above it short of SCHED_DEADLINE.

```c
struct sched_param p = { .sched_priority = 50 };
sched_setscheduler(0, SCHED_FIFO, &p);
```

### 4.2 SCHED_DEADLINE

Each task declares a `(runtime, deadline, period)` triple. The kernel admits the task only if it can prove all current SCHED_DEADLINE tasks together fit. Stronger guarantee than FIFO/RR but rarely used outside specialized workloads.

### 4.3 RT throttling

To prevent a runaway RT task from starving the system, the kernel enforces RT bandwidth: by default, RT tasks may consume at most 950 ms out of each 1000 ms window:

```
$ sysctl kernel.sched_rt_runtime_us    # 950000
$ sysctl kernel.sched_rt_period_us     # 1000000
```

Set `sched_rt_runtime_us = -1` to disable (only on dedicated boxes with controlled RT workloads).

---

## 5. cgroup CPU Control

Linux cgroups (v1 `cpu` controller, v2 unified) layer hierarchical CPU control on top of CFS. Used by every container runtime (Docker, containerd, runc), systemd, and Kubernetes.

### 5.1 cpu.weight (v2) / cpu.shares (v1)

Sets the **proportional share** when CPUs are contended. Only matters when more is being requested than is available.

- **v2**: `cpu.weight` ∈ [1, 10000], default 100. A cgroup with weight 200 gets twice the CPU of a sibling with weight 100 under contention.
- **v1**: `cpu.shares`, default 1024.

In Kubernetes, `resources.requests.cpu` maps to cgroup weight. A pod with `requests: 500m` versus a sibling with `requests: 1000m` will, under contention, get half the CPU.

### 5.2 cpu.max — bandwidth quotas

Sets a **hard cap** on CPU time consumed within a period. v2 syntax:

```
$ cat /sys/fs/cgroup/.../cpu.max
50000 100000          # quota_us period_us  — 50ms per 100ms window = 0.5 CPU
```

When the cgroup exhausts its quota, all its tasks are **throttled** (made non-runnable) until the next period.

In Kubernetes, `resources.limits.cpu` maps to `cpu.max`. A pod with `limits.cpu: 500m` is capped at 50 ms per 100 ms period.

### 5.3 The container CPU-throttling problem

A common production trap:

- Pod has `limits.cpu: 1000m` (1 core).
- Average utilization is 30%.
- But the workload has **bursty parallelism** — at request arrival, 8 threads briefly want to run.
- Within a 100 ms period, those 8 threads can collectively exhaust the 100 ms quota in ~12 ms of wall time, then sit throttled for ~88 ms.
- Latency p99 spikes catastrophically while average CPU looks low.

Diagnostic: `cat /sys/fs/cgroup/<path>/cpu.stat` shows `nr_throttled` and `throttled_usec`.

```
$ cat /sys/fs/cgroup/system.slice/myapp.service/cpu.stat
usage_usec 1234567890
user_usec 789012345
system_usec 456789012
nr_periods 543210
nr_throttled 12345        ← number of periods we hit the cap
throttled_usec 6543210    ← total time spent throttled
```

Mitigations:

- **Increase the limit** (or remove it — `requests` alone gives the cgroup a guaranteed minimum without a hard cap).
- **Decrease parallelism** (smaller thread pool).
- **Increase the period** (`cpu.cfs_period_us` on v1; v2 uses the period field of `cpu.max`) — but this can hurt fairness.

See [Kubernetes Resource Management](../../kubernetes/configuration/resource-management.md) for the orchestration angle.

---

## 6. CPU Affinity and Pinning

`sched_setaffinity()` restricts a task to a subset of CPUs. Userspace tooling:

```bash
# Run a process on CPUs 0-3
taskset -c 0-3 ./prog

# Change a running process
taskset -p 0x0F <pid>     # bitmask: 0x0F = CPUs 0-3

# Inspect current mask
taskset -p <pid>
```

In Kubernetes, the `cpuManager: static` policy on the kubelet pins guaranteed-class containers to dedicated cores.

Affinity helps when:

- **Cache locality** matters — keeping a thread on the same core preserves L1/L2 caches across reschedules.
- **NUMA locality** — keep allocating threads on the same socket as their RAM.
- **Avoiding noisy neighbors** — pinning the latency-critical service away from batch workloads.

It hurts when:

- The constrained set is busier than the free set — the scheduler has fewer options.
- Affinity masks aren't updated after CPU hotplug.

---

## 7. isolcpus, nohz_full, and Tickless Cores

For latency-critical workloads (HFT, telecom, real-time control), Linux can dedicate cores that the scheduler will not place general tasks on:

- **`isolcpus=2-7`** kernel boot parameter — exclude listed CPUs from the general-purpose runqueue. Tasks must explicitly `taskset` onto them.
- **`nohz_full=2-7`** — disable the periodic scheduler tick on those cores when only one task is runnable. Eliminates the ~1 ms timer interrupt overhead.
- **`rcu_nocbs=2-7`** — offload RCU callbacks from those cores.

Combined with SCHED_FIFO, these isolate cores well enough that a pinned thread can run for hours without preemption. Used by DPDK applications, low-latency trading, and dedicated game servers.

---

## 8. Inspecting the Scheduler

### 8.1 schedstat

Per-task scheduler stats (requires `CONFIG_SCHEDSTATS=y`):

```bash
$ cat /proc/<pid>/schedstat
1234567890 9876543210 12345
#   |          |        └─ pcount: number of times this task was scheduled
#   |          └────────── time waiting on runqueue (ns)
#   └───────────────────── time spent running (ns)
```

Per-task latency: `awk '{print $2/$3}' /proc/<pid>/schedstat` — average ns waited per schedule. High values = CPU oversubscribed for this task.

`/proc/<pid>/sched` provides a richer view including vruntime, exec_start, nr_voluntary_switches, etc.

### 8.2 Pressure Stall Information (PSI)

PSI (since Linux 4.20) reports the **fraction of time** runnable tasks were stalled waiting for CPU, memory, or I/O — system-wide and per cgroup.

```
$ cat /proc/pressure/cpu
some avg10=2.34 avg60=1.20 avg300=0.50 total=12345678901
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

- `some` — at least one task was stalled.
- `full` — *all* runnable tasks were stalled (cgroup-only meaningful).
- `avgN` — percentage of the last N seconds.

Key signal: cgroup `cpu.pressure` `full > 0` for a container means every task in that container is stuck waiting for CPU — the cleanest signal of CPU starvation. Wire this into your alerting.

### 8.3 perf sched

`perf sched record / perf sched latency` and `perf sched timehist` give per-task scheduler latency histograms across the whole system.

```bash
$ perf sched record -- sleep 10
$ perf sched latency
 Task            | Runtime ms | Switches | Avg delay ms | Max delay ms |
 -----------------------------------------------------------------------
 java:1234       |   8523.401 |    34521 |       0.045  |       12.34  |
 nginx:5678      |    234.123 |     1234 |       0.012  |        2.45  |
```

`Max delay` of tens of ms for a hot service indicates CPU contention or throttling.

---

## 9. Why Noisy Neighbors Happen in Containers

Putting it all together — a typical Kubernetes node:

- Many pods share the same kernel and hence the same CFS runqueues.
- `requests.cpu` translates to **weight** — only matters under contention.
- `limits.cpu` translates to **bandwidth** — hard cap, can throttle.
- A pod with no `limits` can briefly steal far more than its `requests` if a peer is idle.
- A pod with `limits` smaller than its actual burstiness gets throttled even when the node has spare capacity.
- Short bursts (sub-period) are invisible to per-second utilization graphs.

Best practices for backend services:

- **Set `requests` to your steady-state utilization** so the scheduler weights you appropriately.
- **Set `limits` cautiously, or omit them**, especially for latency-sensitive workloads. (See LWN discussions on whether limits should ever be set on long-lived containerized services.)
- **Monitor cgroup throttling** (`nr_throttled`, `throttled_usec`) as a top-tier metric.
- **Keep thread counts ≤ allocated cores** for CPU-bound work; oversubscribing thread pools causes context-switch storms within your quota.

---

## 10. Backend Engineer Takeaways

- **CFS is fairness-first, not latency-first.** Wakeup latency is bounded but not optimized. For tens-of-microseconds latency, you need pinning + RT class + isolated cores.
- **Niceness only matters under contention** and only between siblings on the same runqueue.
- **Containers run under CFS + cgroup quotas.** "Throttled despite low utilization" is the classic burst-vs-quota mismatch.
- **PSI (`cpu.pressure`) is the cleanest CPU-starvation signal** for containerized services — better than CPU utilization graphs.
- **`schedstat` and `/proc/<pid>/sched`** show per-task wait times — use them when one process feels slow on a busy host.
- **Pinning + isolated cores + SCHED_FIFO** is the recipe for sub-millisecond latency, but your service must be designed to never block.
- **JVM virtual threads, Go goroutines, libuv** all sit on top of CFS via 1:1 carrier threads. CFS scheduling decisions still drive their behavior.

---

## Related

- [Process & Thread Model](01-process-thread-model.md)
- [Virtual Memory & Paging](02-virtual-memory-and-paging.md)
- [Kubernetes Resource Management](../../kubernetes/configuration/resource-management.md)
- [Java Virtual Threads](../../java/java-fundamentals/concurrency/virtual-threads.md)
- [Async I/O Models](../../networking/network-programming/async-io-models.md)

---

## References

- **Linux kernel: Scheduler documentation** — https://www.kernel.org/doc/html/latest/scheduler/index.html
- **Linux kernel: CFS Scheduler design** — https://www.kernel.org/doc/html/latest/scheduler/sched-design-CFS.html
- **Linux kernel: cgroup v2 documentation** — https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html
- **Linux kernel: PSI** — https://www.kernel.org/doc/html/latest/accounting/psi.html
- **`sched(7)` man page** — overview of scheduling policies. https://man7.org/linux/man-pages/man7/sched.7.html
- **`taskset(1)`, `chrt(1)`, `nice(1)`, `renice(1)` man pages**
- **LWN: "An EEVDF CPU scheduler for Linux"** — Jonathan Corbet, 2023. https://lwn.net/Articles/925371/
- **LWN: "CPU bandwidth control for CFS"** — https://lwn.net/Articles/428230/
- **LWN: "Containerized Linux schedulers: time to retire CFS bandwidth?"** — discussion thread on the throttling issue. https://lwn.net/Articles/844976/
- **Brendan Gregg, *Systems Performance*, 2nd ed.** — Chapter 6 (CPUs); covers PSI, perf sched, scheduler latency methodology.
- **Robert Love, *Linux Kernel Development*, 3rd ed.** — Chapter 4 (Process Scheduling).
- **Kubernetes: CPU Manager** — https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/
