---
title: "Diagnostics and Tracing — strace, perf, bpftrace, ftrace, Memory"
date: 2026-05-07
updated: 2026-05-07
tags: [linux, diagnostics, ebpf, perf, tracing]
---

# Diagnostics and Tracing — strace, perf, bpftrace, ftrace, Memory

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `linux` `diagnostics` `ebpf` `perf` `tracing`

---

## Table of Contents

- [Summary](#summary)
- [1. strace and ltrace — System Call and Library Call Tracing](#1-strace-and-ltrace--system-call-and-library-call-tracing)
- [2. perf — CPU Profiling and Hardware Counters](#2-perf--cpu-profiling-and-hardware-counters)
- [3. bpftrace and bcc — eBPF for Production Tracing](#3-bpftrace-and-bcc--ebpf-for-production-tracing)
- [4. ftrace and the /sys/kernel/debug/tracing Tree](#4-ftrace-and-the-syskerneldebugtracing-tree)
- [5. Memory Diagnostics — free, smem, /proc/meminfo, OOM killer logs](#5-memory-diagnostics--free-smem-procmeminfo-oom-killer-logs)
- [6. Operator Walkthrough](#6-operator-walkthrough)
- [Related](#related)
- [References](#references)

## Summary

When the application logs run out, you reach for the kernel. This doc covers the five tracing surfaces a backend operator on Linux actually uses — strace for syscalls, perf for CPU profiling, bpftrace/bcc for safe production eBPF tracing, ftrace for in-kernel function tracing, and the memory-diagnostic stack (`free`, `/proc/meminfo`, OOM killer). Each section explains the underlying mechanism, the overhead, and a small set of commands you can paste at 3 a.m. without crashing prod.

---

## 1. strace and ltrace — System Call and Library Call Tracing

### 1.1 What strace actually does

`strace` is a **userspace ptrace client**. It uses the `ptrace(2)` system call to attach to a target and intercept every syscall the target makes, printing the syscall name, arguments, and return value.

From the `ptrace(2)` man page: "one process (the 'tracer') may observe and control the execution of another process (the 'tracee'), and examine and change the tracee's memory and registers." When `strace -p <pid>` runs, it issues `PTRACE_ATTACH` (or modern `PTRACE_SEIZE`), which "sends `SIGSTOP` to this thread" — the tracee is paused, then resumed under tracer control.

The cost is real. Every traced syscall produces **two** ptrace stops (entry and exit), and each stop is a kernel→user context switch into the tracer. The strace man page is explicit:

> A traced process runs more slowly than a non-traced one. The performance impact can be mitigated by using the `--seccomp-bpf` option.

`--seccomp-bpf` lets the kernel filter which syscalls trigger a stop, so strace only pays the ptrace cost for the syscalls you asked about.

**Operator rule:** never `strace -p` a hot-path production process for more than a few seconds without `--seccomp-bpf` and a `-e trace=` filter. The slowdown is multiplicative on syscall-heavy workloads.

### 1.2 Core flags

| Flag | What it does |
|------|--------------|
| `-p <pid>` | Attach to a running process. Detaches cleanly on Ctrl-C. |
| `-f` | Follow children created via `fork(2)`, `vfork(2)`, `clone(2)`. |
| `-e trace=<set>` | Filter by syscall set. E.g. `-e trace=openat,read,write` or named sets like `network`, `file`, `process`. |
| `-c` | Suppress per-call output; print a summary on exit (counts, time, errors per syscall). |
| `-o file` | Write trace to a file instead of stderr. |
| `-s <n>` | Show first `n` bytes of strings (default is short; raise to e.g. `4096`). |
| `-y` | Print paths next to file descriptor numbers. Hugely improves readability. |
| `--seccomp-bpf` | Kernel-side filter to reduce overhead. Pair with `-e trace=`. |

### 1.3 Reading the output

A typical line:

```text
openat(AT_FDCWD, "/etc/hosts", O_RDONLY|O_CLOEXEC) = 5
read(5, "127.0.0.1 localhost\n::1 lo"..., 4096) = 64
```

- Left of `=` is the syscall and decoded arguments.
- Right of `=` is the return value. For most syscalls a non-negative number is success (often a file descriptor, a byte count, or 0).
- Errors look like `-1 ENOENT (No such file or directory)`. The `errno` symbol is the diagnostic — write it down before you go googling.

Common syscalls and what they tell you:

| Syscall | What you learn |
|---------|----------------|
| `openat`, `open` | What files the process tries to read. Permission bugs and missing config files surface here. |
| `read`, `write` | I/O sizes and which fd is doing them (use `-y`). Tiny reads in a loop = bad batching. |
| `mmap`, `munmap` | Memory mappings — large allocations, shared libraries loading, mmap-based file I/O. |
| `clone` | Thread or process creation. Argument flags tell you whether it's a thread (`CLONE_THREAD`) or a fork. |
| `futex` | Userspace mutex contention. Lots of `FUTEX_WAIT` = threads blocked on locks. |
| `epoll_wait` | Event-loop blocked waiting for I/O. Common in Node.js, nginx, async runtimes. |
| `connect`, `accept`, `sendto`, `recvfrom` | Network activity. `-e trace=network` is the right filter. |

### 1.4 The summary view

`strace -c` is often more useful than the full trace for "where is the time going" questions:

```bash
strace -c -p $(pidof myapp) -- sleep 10
```

Output is a table: `% time`, calls, errors, per-syscall. Wide spread across many syscalls = busy syscall pattern; concentrated in `futex` or `epoll_wait` = the process is mostly sleeping; concentrated in `read`/`write` = I/O-bound.

### 1.5 ltrace caveats

`ltrace` traces **dynamic library calls** (e.g. `malloc`, `strcpy`, `gethostbyname`) instead of syscalls. The man page says it intercepts "dynamic library calls which are called by the executed process and the signals which are received by that process."

Caveats from the same man page:

- "ltrace's usage is very similar to strace(1)" — same ptrace mechanism, similar flags.
- "shares most of the bugs stated in strace(1)" — including overhead.
- Symbol resolution is fragile: `-l` filters by library, but "there's no actual guarantee that the call won't be directed elsewhere due to e.g. `LD_PRELOAD`."

Reach for ltrace when you specifically want to see which **library** function is hit, but expect noisier and less reliable output than strace. For most backend debugging, strace is the better default.

---

## 2. perf — CPU Profiling and Hardware Counters

### 2.1 What perf is

`perf` is the kernel's official profiling and tracing toolkit, exposed through the `perf_event_open(2)` syscall. It is a single binary with many subcommands. The `perf(1)` man page describes it as "a framework for all things performance analysis" covering both hardware and software events.

The subcommands a backend operator uses most:

| Subcommand | Use |
|------------|-----|
| `perf record` | Sample-based profiling. Writes `perf.data`. |
| `perf report` | Interactive viewer for `perf.data`. |
| `perf stat` | Run a command and print hardware counter totals. |
| `perf top` | Live, refreshing view of where the system is spending CPU. |
| `perf list` | Enumerate all available events on this kernel and CPU. |
| `perf annotate` | Map samples back to source/asm. |
| `perf trace` | Strace-like syscall tracing on top of `perf_event_open`. |
| `perf sched` | Scheduler latency analysis. |
| `perf lock` | Lock contention. |

### 2.2 What perf samples

By default `perf record` samples on the **CPU cycles** event — the CPU is interrupted at a configurable frequency, and at each interrupt the kernel records the current instruction pointer and (with `-g`) the call stack.

This is **on-CPU profiling**: it tells you what the CPU is *doing* when it's running. It does not show time spent **off-CPU** (waiting on I/O, locks, sleeping). For off-CPU time you need a different approach — typically eBPF tools that sample the scheduler (see §3).

Key flags for `perf record`:

| Flag | Meaning |
|------|---------|
| `-F <hz>` | Sample frequency. `-F 99` is the de facto default (99 Hz avoids aliasing with periodic 100 Hz timers). |
| `-a` | All CPUs (system-wide). |
| `-p <pid>` | A single process. |
| `-g` | Capture call stacks. |
| `--call-graph fp` / `dwarf` / `lbr` | Stack-unwind method (see §2.4). |
| `-- <cmd>` | Run a command and profile it. |

A canonical "profile the whole system for 10 seconds" command from Brendan Gregg's perf page:

```bash
perf record -F 99 -ag -- sleep 10
perf report
```

### 2.3 `perf stat` and hardware counters

`perf stat` measures, rather than samples. It reads CPU performance monitoring counters (PMCs) for the duration of a command and prints totals.

```bash
perf stat -d -- ./my-binary
```

Useful counters for backend work:

| Counter | What it tells you |
|---------|-------------------|
| `cycles`, `instructions` | Instructions-per-cycle (IPC). Low IPC (<1) suggests stalls. |
| `cache-references`, `cache-misses` | LLC miss rate. High miss rate hints at memory-bandwidth issues. |
| `branch-misses` | Mispredicted branches. |
| `task-clock`, `context-switches`, `cpu-migrations` | Scheduler behavior. |

`perf list` shows everything available on this kernel/CPU; the set varies by hardware vendor.

### 2.4 Stack unwinding: frame pointers, DWARF, LBR

`perf record -g` captures stacks, but how it walks them is a per-binary detail.

| Method | How | Trade-off |
|--------|-----|-----------|
| Frame pointers (`fp`) | Walk the `%rbp` chain. Requires binaries built with `-fno-omit-frame-pointer`. | Cheapest, deepest. Many distro libraries omit frame pointers, so stacks break at libc. |
| DWARF (`dwarf`) | Sample stack memory; unwind offline using DWARF debug info. | Works on optimized binaries. Heavier (samples copy ~8KB of stack). |
| LBR (`lbr`) | Use Intel's Last Branch Record hardware buffer. | Cheapest at runtime. Limited depth — Brendan Gregg's perf page notes LBR is "limited to shallow stack depths (8, 16, or 32 frames)" and "may not be suitable for deep stacks or flame graph generation." |

For Java, Node.js, Go, Python: the language runtime usually needs a **perf map** (`/tmp/perf-<pid>.map`) to translate JIT'd addresses to symbol names. Each runtime ships its own perf-map generator (e.g. `--perf-basic-prof` for Node.js).

### 2.5 Flame graphs

`perf report` is interactive but text-oriented. For the "where does the CPU go" question, a flame graph is faster to read.

The canonical workflow from Brendan Gregg's FlameGraph repo:

```bash
perf record -F 99 -ag -- sleep 30
perf script > out.perf
./stackcollapse-perf.pl out.perf > out.folded
./flamegraph.pl out.folded > out.svg
```

Open `out.svg` in a browser. X-axis is sample count (not time), Y-axis is stack depth. Wide bars = more samples = more CPU.

### 2.6 `perf top`

Live, top-style view of hottest functions on the system:

```bash
sudo perf top -F 99
```

Fast diagnostic when something is melting a CPU right now and you don't know what.

---

## 3. bpftrace and bcc — eBPF for Production Tracing

### 3.1 What eBPF is

eBPF (extended Berkeley Packet Filter) is a small in-kernel virtual machine. You write a program in restricted C or a higher-level DSL, the kernel **verifier** statically checks it (no unbounded loops, no out-of-bounds memory access, terminates), the JIT compiles it to native instructions, and it runs attached to a kernel hook.

Why operators care: the verifier means a buggy probe **cannot crash the kernel**. The bcc README puts it directly: eBPF programs "cannot crash, hang or interfere with the kernel negatively." Compared to `strace` (every syscall context-switches into a userspace tracer) or kprobe scripts (no verifier), eBPF is the only safe tracing surface for production.

### 3.2 bpftrace one-liners

`bpftrace` is "a general purpose tracing tool and language for Linux. It leverages eBPF to provide powerful, efficient tracing capabilities with minimal overhead." (bpftrace README)

It supports four major probe families per the README:

- **Kernel dynamic tracing (kprobes)** — attach to any non-inlined kernel function.
- **Hardware and software perf events** — same events `perf` uses.
- **User-level dynamic tracing (USDT, uprobes)** — attach to userspace function entry/exit.
- **Tracepoints** — stable kernel instrumentation points (preferred over kprobes when available).

Counts of events into a map:

```bash
# Count openat() per process for 10 seconds
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'
```

The `@` prefix declares a map. `count()` is an aggregation. On Ctrl-C bpftrace prints all maps.

Histograms with `hist()`:

```bash
# Distribution of read() sizes
bpftrace -e 'tracepoint:syscalls:sys_exit_read /args->ret > 0/ { @bytes = hist(args->ret); }'
```

uprobes (userspace function entry):

```bash
# Trace every call to malloc() in a specific binary
bpftrace -e 'uprobe:/usr/bin/myapp:malloc { @[ustack] = count(); }'
```

### 3.3 bcc canned tools

bcc ships dozens of pre-written tools. The README highlights canonical ones:

| Tool | What it shows |
|------|---------------|
| `opensnoop` | Every `open()`/`openat()` syscall, with PID, FD, errno, path. |
| `execsnoop` | Every `exec()`. The fastest way to see what a system is launching. |
| `tcplife` | TCP session lifetimes — duration, bytes, addresses, ports. |
| `biolatency` | Block-device I/O latency as a histogram. |
| `runqlat` | Scheduler run-queue latency. Tells you whether CPU contention is hurting you. |

These are usually packaged as `bcc-tools` (RHEL family) or `bpfcc-tools` (Debian/Ubuntu).

### 3.4 Why eBPF beats kprobes-via-ftrace for production

- **Verified.** A buggy program is rejected at load, not at runtime.
- **Lower overhead.** JIT'd; per-CPU maps avoid most contention.
- **Composable.** Maps, aggregations, and stack capture in-kernel; userspace just reads totals. No per-event userspace wakeup.

### 3.5 Kernel and distro requirements

bcc's README states "much of what BCC uses requires Linux 4.1 and above." For modern bpftrace features and CO-RE (Compile Once, Run Everywhere), kernels with **BTF** (BPF Type Format) — typically 5.x with `CONFIG_DEBUG_INFO_BTF=y` — are strongly preferred. The bpftrace upstream README does not catalog kernel requirements explicitly; check your distro packaging notes.

---

## 4. ftrace and the /sys/kernel/debug/tracing Tree

### 4.1 What ftrace is

ftrace is the kernel's own built-in tracer. The kernel docs describe it as "an internal tracer designed to help out developers and designers of systems to find what is going on inside the kernel" — and importantly, it is "a framework of several assorted tracing utilities" rather than a single tool.

Unlike eBPF, ftrace does not require any userspace toolchain: it's always there if `CONFIG_FTRACE` is enabled.

### 4.2 The tracefs tree

Per the kernel docs, ftrace is controlled through files under `tracefs`:

```bash
mount -t tracefs nodev /sys/kernel/tracing
```

On modern kernels the directory is `/sys/kernel/tracing`. Older kernels (and most working scripts on the internet) use `/sys/kernel/debug/tracing`, which "remains available for backward compatibility."

Important files inside:

| File | Role |
|------|------|
| `current_tracer` | Which tracer is active. |
| `available_tracers` | What you can write into `current_tracer`. |
| `tracing_on` | `1` = collect, `0` = pause. |
| `set_event` | Enable specific tracepoint events. |
| `trace` | Static buffer dump. |
| `trace_pipe` | Streaming consumer. |
| `set_ftrace_filter` | Restrict function tracer to specific functions. |

### 4.3 Available tracers

Per the docs, `current_tracer` accepts values including:

- `function` — "Function call tracer to trace all kernel functions"
- `function_graph` — "a graph of function calls similar to C code source" (entry + exit)
- `irqsoff`, `preemptoff`, `preemptirqsoff` — interrupt and preemption disabled periods
- `wakeup`, `wakeup_rt`, `wakeup_dl` — task-scheduling latency
- `hwlat` — "used to detect if the hardware produces any latency"
- `nop` — "the 'trace nothing' tracer"

### 4.4 Event-based tracing

Tracepoints are stable, named instrumentation points compiled into the kernel. To enable one, write into `set_event`:

```bash
cd /sys/kernel/tracing
echo 1 > tracing_on
echo 'sched:sched_switch' > set_event
cat trace_pipe
```

The kernel docs describe `set_event`: "By echoing in the event into this file, will enable that event."

### 4.5 trace_pipe vs trace

`trace` is a static snapshot. `trace_pipe` is "meant to be streamed with live tracing" — the docs note that "reading from this file causes sequential reads to display more current data" and consumed data won't reappear. Use `trace_pipe` when you want a tail, `trace` when you want a one-shot dump.

### 4.6 trace-cmd

`trace-cmd` is the friendlier frontend. Per the man page it "interacts with the Ftrace tracer that is built inside the Linux kernel" and "interfaces with the Ftrace specific files found in the debugfs file system."

```bash
trace-cmd record -e sched:sched_switch sleep 5
trace-cmd report
```

Subcommands per the man page:

- `record` — "record a live trace and write a trace.dat file to the local disk or to the network"
- `report` — "reads a trace.dat file and converts the binary data to a ASCII text readable format"
- `start` — "start the tracing without recording to a trace.dat file"
- `stop` — "stop tracing (only disables recording, overhead of tracer is still in effect)"

### 4.7 ftrace vs bpftrace — when to use each

| Question | ftrace | bpftrace |
|----------|--------|----------|
| Kernel only, no userspace toolchain? | Built-in | Needs BPF userspace tools |
| Need aggregations / histograms in-kernel? | Limited | Native (`hist()`, `count()`) |
| Need to trace userspace (uprobes, USDT)? | No | Yes |
| Older kernel without BTF? | Works | May be limited |
| Need extreme low-overhead function tracing? | Function tracer is very cheap | Comparable, varies |

The honest answer: on a modern kernel with bpftrace installed, reach for bpftrace first. ftrace is your fallback when you can't install anything new on the host.

---

## 5. Memory Diagnostics — free, smem, /proc/meminfo, OOM killer logs

### 5.1 `free -h`

Per the `free(1)` man page, `free` "displays the total amount of free and used physical and swap memory in the system, as well as the buffers and caches used by the kernel."

The columns and what they actually mean:

| Column | Definition (from free(1)) |
|--------|---------------------------|
| `total` | Physical + swap minus reserved — `MemTotal`/`SwapTotal` in `/proc/meminfo`. |
| `used` | "calculated as **total** − **available**" — *not* simple "in use by processes". |
| `free` | Truly unused — `MemFree`/`SwapFree`. |
| `shared` | `Shmem` — mostly tmpfs. |
| `buff/cache` | `Buffers` + page cache + reclaimable slabs. |
| `available` | "Estimation of how much memory is available for starting new applications, without swapping." |

The trap: `free` is small but `available` is large. That's normal. The kernel deliberately keeps memory in the page cache and reclaims it under pressure. **Read `available`, ignore `free`.**

### 5.2 `/proc/meminfo` deep-dive

Per `proc_meminfo(5)`:

| Field | Meaning |
|-------|---------|
| `MemAvailable` | "An estimate of how much memory is available for starting new applications, without swapping." (Linux 3.14+) |
| `Buffers` | "Relatively temporary storage for raw disk blocks that shouldn't get tremendously large (20 MB or so)." |
| `Cached` | "In-memory cache for files read from the disk (the page cache). Doesn't include SwapCached." |
| `Active(file)`, `Inactive(file)` | Page-cache LRU lists, split into recently-used and reclaim candidates. (Documented as present since 2.6.28.) |
| `SReclaimable` | "Part of Slab, that might be reclaimed, such as caches." (Linux 2.6.19+) |
| `Slab` | "In-kernel data structures cache" (see `slabinfo(5)`). |

Reading patterns:

- **Lots of `Cached` is healthy.** It's the page cache doing its job.
- **Big `Slab` with little `SReclaimable`** can mean a kernel-side memory leak (e.g. dentries pinned by an unbounded fs walk). Drill in with `slabtop`.
- **`MemAvailable` is the truth.** It already accounts for reclaimable cache and non-reclaimable slab. Use it for capacity decisions.

### 5.3 RSS lies; PSS doesn't

`ps` and `top` show RSS (Resident Set Size) — every page mapped by the process, including pages shared with other processes. If 100 forked workers each map the same 200 MB binary, each shows RSS = 200+ MB and the total RSS sum is wildly higher than physical memory used.

`smem` per `smem(8)` "reports physical memory usage, taking shared memory pages into account." It splits shared pages proportionally:

- **USS** — "Unshared memory is reported as the USS (Unique Set Size)." Memory only this process has. Frees on exit.
- **PSS** — "The unshared memory (USS) plus a process's proportion of shared memory is reported as the PSS (Proportional Set Size)." Sum of PSS across processes ≈ actual physical memory in use.

```bash
smem -k -t -P myapp
```

`-k` shows abbreviated unit suffixes; `-t` adds a totals line.

### 5.4 The OOM killer

When the kernel cannot reclaim memory and an allocation must succeed, it picks a process to kill. Per `proc_pid_oom_score(5)`, `/proc/<pid>/oom_score` is "the current score that the kernel gives to this process for the purpose of selecting a process for the OOM-killer." Higher = more likely to die.

You can bias the choice with `/proc/<pid>/oom_score_adj`, per `proc_pid_oom_score_adj(5)`:

- Range **−1000 to +1000**.
- The constants are `OOM_SCORE_ADJ_MIN` (−1000) and `OOM_SCORE_ADJ_MAX` (+1000).
- "The lowest possible value, −1000, is equivalent to disabling OOM-killing entirely for that task, since it will always report a badness score of 0."

When it fires, look in the kernel ring buffer:

```bash
journalctl -k --since "15 min ago" | grep -i 'killed process\|out of memory'
dmesg -T | grep -i 'killed process\|out of memory'
```

The log line names the killed process, its RSS, and the cgroup it belonged to.

### 5.5 cgroup memory and OOM-in-cgroup

In containerized workloads, the OOM killer is more often a **cgroup OOM killer** — the cgroup hit its `memory.max` limit. Per the cgroup-v2 docs:

| File | Behavior |
|------|----------|
| `memory.current` | "The total amount of memory currently being used by the cgroup and its descendants." |
| `memory.high` | Throttle limit. "If a cgroup's usage goes over the high boundary, the processes of the cgroup are throttled and put under heavy reclaim pressure." Notably, exceeding `memory.high` "never invokes the OOM killer." |
| `memory.max` | Hard limit. "If a cgroup's memory usage reaches this limit and can't be reduced, the OOM killer is invoked in the cgroup." |
| `memory.oom.group` | "Determines whether the cgroup should be treated as an indivisible workload by the OOM killer." When set, "all tasks belonging to the cgroup or to its descendants are killed together or not at all." |

Kubernetes pod OOM kills, container-runtime OOM kills, and systemd-slice memory limits are all cgroup-v2 OOMs in disguise. Reading `memory.current` vs `memory.max` for the cgroup tells you how close you are.

---

## 6. Operator Walkthrough

The following commands are designed to run on a throwaway VM or container — anything you control. Do not run them against a production process you cannot afford to slow down.

### 6.1 Setup

```bash
# A small workload to trace.
cat > /tmp/work.sh <<'EOF'
#!/usr/bin/env bash
while true; do
  cat /etc/hostname > /dev/null
  cat /etc/os-release > /dev/null
  sleep 0.1
done
EOF
chmod +x /tmp/work.sh
/tmp/work.sh &
WORK_PID=$!
echo "PID=$WORK_PID"
```

### 6.2 strace: who opens what

```bash
# Watch openat() for 5 seconds.
sudo strace -f -e trace=openat -p "$WORK_PID" -- 2>&1 | head -40

# Summary view.
sudo strace -c -p "$WORK_PID" -- sleep 5
```

You should see `openat(...)` calls for `/etc/hostname` and `/etc/os-release`, and the summary should show `openat`, `read`, `close` dominating.

### 6.3 perf: profile a tight loop

```bash
# Build a small CPU-bound C program.
cat > /tmp/spin.c <<'EOF'
#include <stdio.h>
static long busy(long n) { long s = 0; for (long i = 0; i < n; i++) s += i*i; return s; }
int main(void) { long s = 0; for (int i = 0; i < 50; i++) s += busy(50000000L); printf("%ld\n", s); }
EOF
cc -O2 -fno-omit-frame-pointer -g -o /tmp/spin /tmp/spin.c

# Profile and report.
sudo perf record -F 99 -g --call-graph fp -- /tmp/spin
sudo perf report --stdio | head -60
```

You should see `busy` and `main` near the top of the report. Try without `-fno-omit-frame-pointer` to see how stacks degrade.

For a flame graph (requires the FlameGraph repo cloned somewhere):

```bash
sudo perf script > /tmp/out.perf
~/FlameGraph/stackcollapse-perf.pl /tmp/out.perf > /tmp/out.folded
~/FlameGraph/flamegraph.pl /tmp/out.folded > /tmp/spin.svg
```

`perf stat` view of the same workload:

```bash
sudo perf stat -d -- /tmp/spin
```

Inspect IPC (`instructions / cycles`) and cache miss rate.

### 6.4 bpftrace: count openat by process

```bash
# For 10 seconds, count openat() calls per command.
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }
interval:s:10 { exit(); }
'
```

Output is a sorted table at the end. Your `work.sh` (visible as `cat`) should appear with hundreds of opens.

A latency histogram on block I/O — handy when disks feel slow:

```bash
sudo /usr/sbin/biolatency-bpfcc 5 1   # bcc tool name varies by distro
```

### 6.5 /proc/meminfo and cache drop

```bash
grep -E '^(MemTotal|MemFree|MemAvailable|Buffers|Cached|SReclaimable|Slab):' /proc/meminfo

# Generate cache pressure.
dd if=/dev/zero of=/tmp/big bs=1M count=1024
grep -E '^(MemFree|Cached|MemAvailable):' /proc/meminfo

# Drop caches (test box only, never prod).
sudo sync
echo 3 | sudo tee /proc/sys/vm/drop_caches
grep -E '^(MemFree|Cached|MemAvailable):' /proc/meminfo

rm /tmp/big
```

You should see `Cached` rise after the `dd`, then collapse after the drop, while `MemAvailable` stays roughly stable across both — the kernel was already willing to return that memory to applications without the explicit drop.

### 6.6 Force an OOM kill, read the log

On a test VM with cgroup-v2:

```bash
# Create a cgroup with a tiny memory limit.
sudo mkdir /sys/fs/cgroup/oomtest
echo "+memory" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
echo $((50*1024*1024)) | sudo tee /sys/fs/cgroup/oomtest/memory.max  # 50 MB

# Run a memory hog inside it.
sudo bash -c 'echo $$ > /sys/fs/cgroup/oomtest/cgroup.procs; \
  python3 -c "x=bytearray(200*1024*1024); print(\"alive\")"'

# Inspect what just happened.
journalctl -k --since "1 min ago" | grep -i 'memory\|killed process'
sudo cat /sys/fs/cgroup/oomtest/memory.events
```

The journal entry shows the process killed, its RSS, and the originating cgroup. `memory.events` shows counters like `oom`, `oom_kill`, `max`. Cleanup:

```bash
sudo rmdir /sys/fs/cgroup/oomtest
kill "$WORK_PID"
```

### 6.7 Decision tree

When a backend feels wrong, the order to reach for tools is roughly:

1. **Logs first.** Usually they tell you.
2. **`top`/`htop` + `free -h` + `journalctl -k`** — establish whether it's CPU, memory, I/O, or "the kernel killed something."
3. **CPU-bound?** `perf top` for the live view, then `perf record -F 99 -ag -- sleep 30` + flame graph.
4. **Stuck or slow but CPU is idle?** It's blocked. `strace -p` (briefly) to see what it's waiting on, or `bpftrace` off-CPU sampling.
5. **Mysterious file/permission issue?** `strace -f -e trace=openat,stat,connect`.
6. **Memory pressure / cgroup OOM?** `/proc/meminfo`, `smem -tk`, `journalctl -k`, the cgroup's `memory.events`.
7. **Kernel-side mystery?** ftrace event tracing or bpftrace kprobes — but only after the cheaper tools above have come up empty.

---

## Related

- [../../operating-systems/fundamentals/01-process-thread-model.md](../../operating-systems/fundamentals/01-process-thread-model.md) — the process/thread model that strace and perf are observing.
- [../../operating-systems/fundamentals/03-page-cache-and-buffered-io.md](../../operating-systems/fundamentals/03-page-cache-and-buffered-io.md) — why `Cached` is large and why that's healthy.
- [../../operating-systems/fundamentals/05-syscalls-and-the-abi.md](../../operating-systems/fundamentals/05-syscalls-and-the-abi.md) — what `strace` is actually intercepting.
- [../../operating-systems/fundamentals/07-epoll-kqueue-iouring.md](../../operating-systems/fundamentals/07-epoll-kqueue-iouring.md) — context for `epoll_wait` lines in strace output.
- [../../performance/fundamentals/05-flame-graphs-cpu-and-off-cpu.md](../../performance/fundamentals/05-flame-graphs-cpu-and-off-cpu.md) — flame-graph theory and on-CPU vs off-CPU profiling.
- [../../performance/fundamentals/03-tail-latency-and-p99.md](../../performance/fundamentals/03-tail-latency-and-p99.md) — the latency questions that often drive you to perf and bpftrace.
- [../../observability/fundamentals/04-red-and-use-methods.md](../../observability/fundamentals/04-red-and-use-methods.md) — USE method points you at the saturation/utilization questions diagnostic tools answer.

## References

- strace(1) man page — <https://man7.org/linux/man-pages/man1/strace.1.html>
- ltrace(1) man page — <https://man7.org/linux/man-pages/man1/ltrace.1.html>
- ptrace(2) man page — <https://man7.org/linux/man-pages/man2/ptrace.2.html>
- perf(1) man page — <https://man7.org/linux/man-pages/man1/perf.1.html>
- Brendan Gregg, *Linux perf Examples* — <https://www.brendangregg.com/perf.html> (used as supporting context for flame-graph workflow and stack-unwinding methods).
- FlameGraph repo, Brendan Gregg — <https://github.com/brendangregg/FlameGraph>
- bpftrace project README — <https://github.com/bpftrace/bpftrace>
- BCC (BPF Compiler Collection) README — <https://github.com/iovisor/bcc>
- Kernel documentation, `Documentation/trace/ftrace.rst` — <https://docs.kernel.org/trace/ftrace.html>
- trace-cmd(1) man page — <https://man7.org/linux/man-pages/man1/trace-cmd.1.html>
- free(1) man page — <https://man7.org/linux/man-pages/man1/free.1.html>
- smem(8) man page — <https://man7.org/linux/man-pages/man8/smem.8.html>
- proc_meminfo(5) man page — <https://man7.org/linux/man-pages/man5/proc_meminfo.5.html>
- proc_pid_oom_score(5) man page — <https://man7.org/linux/man-pages/man5/proc_pid_oom_score.5.html>
- proc_pid_oom_score_adj(5) man page — <https://man7.org/linux/man-pages/man5/proc_pid_oom_score_adj.5.html>
- Kernel documentation, cgroup-v2 (Memory controller) — <https://docs.kernel.org/admin-guide/cgroup-v2.html>
