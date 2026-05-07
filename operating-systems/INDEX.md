# Operating Systems Documentation Index — Learning Path

A progressive path through the parts of the operating system that backend engineers actually touch: the process and thread model, virtual memory, the page cache, the CPU scheduler, system calls, file descriptors, and the modern I/O notification mechanisms (epoll, kqueue, io_uring). Linux-first, with macOS/BSD notes where they diverge. Practical and backend-oriented — the goal is to debug a slow service, a runaway memory footprint, or a saturated event loop with confidence rather than reciting kernel trivia.

Cross-references to the [Networking learning path](../networking/INDEX.md), the [Java learning path](../java/INDEX.md) (especially JVM GC and virtual threads), the [TypeScript learning path](../typescript/INDEX.md) (libuv, V8 internals), the [Go learning path](../golang/INDEX.md) (G-M-P scheduler as M:N over OS threads, syscall interaction), the [Database learning path](../database/INDEX.md) (page cache and fsync interactions), the [Kubernetes learning path](../kubernetes/INDEX.md) (cgroups, resource limits), the [Search learning path](../search/INDEX.md), the [Linux learning path](../linux/INDEX.md), the [Git Internals learning path](../git-internals/INDEX.md), and the [Web Scraping learning path](../web-scraping/INDEX.md) (event loops behind concurrent crawlers, fd limits when running thousands of headless browser contexts) where topics overlap.

**Markers:** **★** = core must-learn (everyday backend OS work, common in interviews and production debugging). **○** = supporting deep-dive (more ops/kernel-engineer territory or specialized advanced topics). Internalize all ★ before going deep on ○.

---

## Tier 1 — Foundations: How the Kernel Manages Your Process

The core abstractions every backend developer leans on without realizing it. Process model, memory, scheduling, syscalls, file descriptors, and the I/O readiness mechanisms that power every modern server.

1. [★ Process & Thread Model — fork, exec, address spaces, and M:N threading](fundamentals/01-process-thread-model.md) — process vs thread, fork/exec/copy-on-write, address space layout, kernel vs user threads, M:N (libuv, virtual threads), context switch cost, /proc/<pid> _(2026-05-03)_
2. [★ Virtual Memory & Paging — page tables, TLB, faults, swap, mmap](fundamentals/02-virtual-memory-and-paging.md) — virtual vs physical, page tables, TLB, minor/major faults, swapping, mmap vs malloc, RSS/VSZ, OOM killer, THP, NUMA _(2026-05-03)_
3. [★ Page Cache & Buffered I/O — dirty pages, writeback, fsync, O_DIRECT](fundamentals/03-page-cache-and-buffered-io.md) — page cache structure, dirty pages, writeback, read-ahead, fsync/fdatasync, O_DIRECT, sendfile, why "free memory" lies, Postgres/MySQL interactions _(2026-05-03)_
4. [★ CPU Scheduling (CFS) — runqueues, vruntime, cgroups, throttling](fundamentals/04-cpu-scheduling-cfs.md) — CFS internals, vruntime, niceness, cgroup CPU shares/quotas, SCHED_FIFO/RR, CPU affinity, isolcpus, schedstat, container noisy neighbors _(2026-05-03)_
5. [★ Syscalls & the ABI — int 0x80 → syscall, vDSO, strace, eBPF](fundamentals/05-syscalls-and-the-abi.md) — syscall mechanism, vDSO, strace, ABI vs API, errno, syscall cost, glibc wrappers, seccomp, eBPF as alternative _(2026-05-03)_
6. [★ File Descriptors & ulimits — int → struct file, open files, EMFILE](fundamentals/06-file-descriptors-and-ulimits.md) — what an fd actually is, open files table, soft vs hard limits, ulimit -n, /proc/<pid>/limits, lsof, fd leaks, EMFILE/ENFILE _(2026-05-03)_
7. [★ epoll / kqueue / io_uring — readiness vs completion, level/edge triggered](fundamentals/07-epoll-kqueue-iouring.md) — readiness vs completion, level/edge-triggered epoll, kqueue (BSD/macOS), io_uring SQ/CQ, libuv and Netty, when to choose each _(2026-05-03)_

---

## Tier 2 — Concurrency Primitives (planned)

Synchronization at the OS and kernel level: mutexes (futex), spinlocks vs sleeplocks, condition variables, RCU, atomics and the C11/C++11/Java memory model from a kernel perspective, and the cost of cache-coherent shared state on modern multi-socket hardware.

## Tier 3 — File Systems (planned)

Inodes, dentries, the VFS layer, journaling (ext4/XFS), copy-on-write file systems (Btrfs/ZFS), write barriers, fsync semantics across filesystems, durability guarantees, direct I/O patterns for databases.

## Tier 4 — Linux Kernel Internals (planned)

Boot process, module loading, namespaces and cgroups v2 internals, the network stack from socket to NIC, tracing infrastructure (ftrace, perf, eBPF/BPF CO-RE), and how the kernel exposes diagnostic data through /proc, /sys, and tracefs.

## Tier 5 — Containers & Namespaces Deep (planned)

How container runtimes (runc, containerd, crun) actually work: the seven Linux namespaces, cgroups v2 controllers, capabilities, seccomp-bpf profiles, user namespace remapping, pivot_root and rootless containers, and the OCI runtime specification as it maps to syscalls.

---

## Quick Reference by Topic

### Process & Threads

- [Process & Thread Model](fundamentals/01-process-thread-model.md)

### Memory

- [Virtual Memory & Paging](fundamentals/02-virtual-memory-and-paging.md)
- [Page Cache & Buffered I/O](fundamentals/03-page-cache-and-buffered-io.md)

### CPU & Scheduling

- [CPU Scheduling (CFS)](fundamentals/04-cpu-scheduling-cfs.md)

### Kernel Interface

- [Syscalls & the ABI](fundamentals/05-syscalls-and-the-abi.md)
- [File Descriptors & ulimits](fundamentals/06-file-descriptors-and-ulimits.md)

### I/O Notification

- [epoll / kqueue / io_uring](fundamentals/07-epoll-kqueue-iouring.md)

---

## Bug Spotting

Active-recall practice docs. Each presents 22+ broken snippets organized by difficulty (Easy / Subtle / Senior trap), with one-line `<details>` hints inline and full root-cause + fix in a Solutions section. Every bug cites a real reference (RFC, CVE, official-doc gotcha, postmortem, library issue). Use these to pressure-test concept knowledge after working through the tiers above.

- [Virtual Memory & Paging — Bug Spotting](fundamentals/vm-paging-bug-spotting.md) ★ — _(2026-05-03)_
