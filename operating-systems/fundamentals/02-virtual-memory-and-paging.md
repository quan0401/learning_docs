---
title: "Virtual Memory & Paging — page tables, TLB, faults, swap, mmap"
date: 2026-05-03
updated: 2026-05-03
tags: [operating-systems, linux, virtual-memory, paging, mmap, oom, numa, thp]
---

# Virtual Memory & Paging — page tables, TLB, faults, swap, mmap

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `operating-systems` `linux` `virtual-memory` `paging` `mmap` `oom` `numa` `thp`

---

## Table of Contents

- [Summary](#summary)
- [1. Virtual vs Physical Addresses](#1-virtual-vs-physical-addresses)
- [2. Page Tables](#2-page-tables)
  - [2.1 Multi-Level Tables on x86_64](#21-multi-level-tables-on-x86_64)
  - [2.2 PTE Bits](#22-pte-bits)
- [3. The TLB](#3-the-tlb)
- [4. Page Faults](#4-page-faults)
  - [4.1 Minor Faults](#41-minor-faults)
  - [4.2 Major Faults](#42-major-faults)
  - [4.3 Invalid / Segmentation Faults](#43-invalid--segmentation-faults)
- [5. Swap](#5-swap)
- [6. mmap vs malloc](#6-mmap-vs-malloc)
  - [6.1 What malloc actually does](#61-what-malloc-actually-does)
  - [6.2 What mmap does](#62-what-mmap-does)
  - [6.3 When to use each](#63-when-to-use-each)
- [7. RSS, VSZ, PSS, USS](#7-rss-vsz-pss-uss)
- [8. The OOM Killer](#8-the-oom-killer)
- [9. Transparent Huge Pages](#9-transparent-huge-pages)
- [10. NUMA Basics](#10-numa-basics)
- [11. Reading /proc/meminfo](#11-reading-procmeminfo)
- [12. Backend Engineer Takeaways](#12-backend-engineer-takeaways)
- [Related](#related)
- [References](#references)

---

## Summary

Every memory address your code touches is virtual. The MMU translates it through a chain of page tables to a physical RAM location, caches the recent translations in the TLB, and traps to the kernel on a page fault when the translation is missing. This indirection is what enables process isolation, copy-on-write, demand paging, memory-mapped files, and swap. As a backend engineer you don't write page-table code, but you absolutely will debug a process whose RSS keeps climbing, a Postgres instance starved by NUMA imbalance, or a container OOM-killed at 2 GB despite "having plenty of free memory." This document maps the abstractions to the diagnostic tools.

---

## 1. Virtual vs Physical Addresses

A 64-bit Linux process operates entirely on **virtual addresses**. The kernel + MMU translate these to **physical addresses** (actual DRAM cell locations) on each memory access.

```
   user code               MMU + page tables             physical RAM
   ─────────                ──────────────                ────────────
   load [0x7f00…]   ──→     translate VA→PA      ──→     [DIMM @ 0x12340000]
                            (cached in TLB)
```

Why bother?

- **Isolation** — process A cannot reference process B's memory because they have different page tables.
- **Sparse address spaces** — your 16 GB virtual heap is fine even on an 8 GB machine, as long as the *resident* set fits.
- **Sharing** — the same physical page can be mapped (read-only) into many processes (shared libraries, page cache, CoW).
- **Lazy allocation** — `malloc(1 GB)` doesn't reserve 1 GB of RAM. Pages are allocated on first touch.
- **Swap** — pages can be evicted to disk when RAM is short.

Pages are typically **4 KB** on x86_64 (also 2 MB and 1 GB "huge pages"; ARM also supports 16 KB and 64 KB).

---

## 2. Page Tables

### 2.1 Multi-Level Tables on x86_64

A flat array indexing every virtual page would need 2^36 entries (with 48-bit VAs and 4 KB pages) — impossible. Linux/x86_64 uses a **4-level radix tree** (5-level on newer CPUs with LA57):

```
Virtual address (48-bit canonical):
 47          39 38         30 29         21 20         12 11        0
+--------------+-------------+-------------+-------------+----------+
|    PGD idx   |   PUD idx   |   PMD idx   |   PTE idx   |  offset  |
+--------------+-------------+-------------+-------------+----------+
       |              |              |              |          |
       v              v              v              v          v
    PGD[i]   →    PUD[j]    →    PMD[k]    →    PTE[l]    →  byte in 4K page
   (top of    (next-level    (next-level    (final entry,
    page-table  page-dir     page-dir       points to physical
     root)     ptr table)    middle)         page frame)

CR3 register holds the physical address of the PGD for the current process.
```

A TLB miss triggers a **page walk**: the MMU reads up to 4 cache lines from the page-table tree. With huge pages (2 MB), one PMD entry maps directly to a 2 MB physical region — no PTE level walk needed.

### 2.2 PTE Bits

Each PTE (Page Table Entry) packs the physical page-frame number plus flags:

| Flag | Meaning |
|------|---------|
| P (Present) | Page is in RAM. Cleared = page fault on access (could be swapped out, never allocated, or invalid). |
| RW | Writable from user mode (cleared = read-only; CoW pages have this cleared). |
| US | User-mode accessible (cleared = kernel-only). |
| A (Accessed) | Set by the MMU on any access; used by the page-out scanner. |
| D (Dirty) | Set on write; tells the kernel the page must be written back before eviction. |
| NX | No-execute — instruction fetch from this page faults (defeats certain exploits). |

---

## 3. The TLB

The Translation Lookaside Buffer is the CPU's cache of recent VA→PA translations. Modern x86 cores have separate L1 dTLB and iTLB (~64 entries) and a larger shared L2 TLB (~1500 entries).

A TLB miss costs a page walk — typically tens of nanoseconds. A TLB **hit** is a few cycles. A `mmap` on a 1 GB region with 4 KB pages can blow out the TLB; the same region mapped with 2 MB huge pages stays cached.

Process switches with different `mm` historically flushed the entire TLB. **PCID** (Process Context ID, x86) and **ASID** (ARM) tag TLB entries so that switches don't have to flush — Linux enables PCID by default since 4.14.

You can observe TLB pressure with `perf`:

```
$ perf stat -e dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses ./prog
```

A miss rate above ~1% on a hot workload is worth investigating with huge pages or access-pattern improvements.

---

## 4. Page Faults

A page fault is a CPU exception that traps into the kernel when an access can't be satisfied by the current page tables.

### 4.1 Minor Faults

The page is in RAM but not (yet) mapped into this process's page table.

- **First touch** of a `mmap`'d region — kernel maps a zero page (anonymous) or reads from the file (file-backed).
- **CoW write** — a previously shared read-only page is duplicated for the writer.
- **Page cache hit on a file read** — the data is already in the page cache (see [Page Cache & Buffered I/O](03-page-cache-and-buffered-io.md)).

Minor faults are cheap (~microseconds). Visible in `ps -o min_flt,maj_flt`.

### 4.2 Major Faults

The page is **not** in RAM and the kernel must read it from disk.

- **File-backed mmap** of a not-cached page → blocking disk I/O.
- **Swap-in** — a page that was paged out has to be read back.

Major faults are *catastrophic* on a hot path: milliseconds, not microseconds. A backend service generating sustained major faults is either swapping or has its working set exceeding the page cache.

### 4.3 Invalid / Segmentation Faults

The access is outside any mapped region or violates permissions (writing to a read-only page, executing from NX page). Kernel sends `SIGSEGV` to the process.

```bash
# Track faults across the system
$ vmstat 1
procs ----memory--- ---swap-- ----io--- -system- ----cpu----
 r  b  free  cache    si  so    bi   bo  in cs  us sy id wa
                       ↑       ↑
                  swap-in  block-in (potential major faults)
```

Per-process: `ps -o pid,comm,min_flt,maj_flt` or `pidstat -r 1`.

---

## 5. Swap

When physical RAM is full, the kernel can write less-active anonymous pages to a **swap area** (a partition or file) and reuse their frames. Pages are tracked by an LRU-like list and the **kswapd** kernel thread.

Swap is essential for the OOM killer to be a last resort, but actively swapping is usually a sign of trouble for a backend server. Two settings to know:

| Sysctl | Meaning |
|--------|---------|
| `vm.swappiness` (0–200) | Bias toward swapping anon pages vs evicting page cache. Default 60. Database/JVM tuners often set it to 1–10 to keep heap/working set in RAM. Setting to 0 doesn't disable swap (it just makes swap a true last resort). |
| `vm.overcommit_memory` | 0 = heuristic (default); 1 = always commit; 2 = strict (deny commits over `CommitLimit`, which is `swap + ram * overcommit_ratio/100`). |

Containers under cgroup memory limits may be configured with `memory.swap.max=0` (cgroup v2) — meaning the cgroup can't swap, so memory pressure goes straight to OOM.

---

## 6. mmap vs malloc

### 6.1 What malloc actually does

`malloc()` is a **userspace** allocator (glibc ptmalloc, jemalloc, tcmalloc, mimalloc). It maintains its own free list of chunks and only talks to the kernel when it needs more memory:

- **Small allocations** — served from the heap, grown by `brk()` syscall.
- **Large allocations** (typically ≥ 128 KB in glibc) — served by `mmap(MAP_ANONYMOUS)` directly. On `free()`, the region is `munmap`'d back to the kernel.

When you `free()`, glibc usually doesn't return memory to the OS — it keeps it on the free list. Hence "RSS doesn't shrink after `free()`." Switching to jemalloc or tuning `MALLOC_TRIM_THRESHOLD_` can help.

### 6.2 What mmap does

`mmap()` is a syscall that adds a region to your address space, optionally backed by a file:

```c
void *mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);
```

| Flag | Effect |
|------|--------|
| `MAP_ANONYMOUS` | Not backed by a file; zero-filled pages allocated on first touch. |
| `MAP_PRIVATE` | CoW — writes are private to this process. |
| `MAP_SHARED` | Writes go through to the file and are visible to other processes mapping it. |
| `MAP_POPULATE` | Pre-fault all pages now (avoid lazy allocation). |
| `MAP_HUGETLB` | Use huge pages from the static hugepage pool. |

A file mmap shares pages with the **page cache** — reads through the mapping are page-cache reads, no extra copy. This is why databases (LMDB, RocksDB, classic Postgres `mmap` debate, MongoDB pre-WiredTiger) lean on mmap.

### 6.3 When to use each

| Scenario | Use |
|----------|-----|
| General heap allocation | `malloc` (let the allocator decide) |
| Allocating a large buffer you'll touch sparsely | `mmap(MAP_ANONYMOUS)` directly |
| Reading a large file mostly sequentially | `read()` — better with read-ahead |
| Random access into a large file | `mmap(MAP_PRIVATE, fd, ...)` |
| Shared memory between processes | `mmap(MAP_SHARED)` on a `shm_open` fd or memfd |

JVM specifics: the JVM heap is one big `mmap(MAP_ANONYMOUS)` region. The JIT code cache is RWX-mapped (often via `mmap` + `mprotect`).

---

## 7. RSS, VSZ, PSS, USS

These memory accounting metrics confuse everyone:

| Metric | Meaning | Where to find |
|--------|---------|---------------|
| **VSZ** (Virtual Size) | Total virtual address space mapped — includes regions never touched. Often huge and meaningless on its own. | `ps -o vsz`, `/proc/<pid>/status` VmSize |
| **RSS** (Resident Set Size) | Pages currently in RAM owned by this process. **Counts shared pages once per process**, so summing RSS overcounts. | `ps -o rss`, `/proc/<pid>/status` VmRSS |
| **PSS** (Proportional Set Size) | Like RSS, but shared pages are divided by the number of sharing processes. Summing PSS gives a true total. | `/proc/<pid>/smaps` (Pss field) |
| **USS** (Unique Set Size) | Pages mapped only by this process (would be freed if the process died). | `smem` tool, `/proc/<pid>/smaps` Private_* |

For a Postgres instance with 100 backends sharing a 2 GB shared_buffers segment:

- Each backend's RSS reports ~2 GB (counts the shared mapping).
- Each backend's PSS reports ~20 MB shared + private = small.
- Summing RSS across all 100 backends suggests 200 GB used. Wrong.
- Summing PSS gives the actual cost.

```bash
# Real total memory cost of a process group
$ smem -t -P postgres
```

---

## 8. The OOM Killer

When the kernel cannot reclaim enough memory to satisfy an allocation (after evicting page cache and swapping), it invokes the **OOM killer**: it picks a process to kill based on a score (oom_score) that weighs RSS, runtime, and adjustment factors.

```bash
# Per-process score (higher = more likely to be killed)
$ cat /proc/<pid>/oom_score
$ cat /proc/<pid>/oom_score_adj   # tunable, -1000 disables

# Last OOM kills in dmesg
$ dmesg -T | grep -i 'killed process'
[Mon May  3 14:22:19 2026] Out of memory: Killed process 12345 (java) total-vm:8388608kB, anon-rss:7340032kB, …
```

In containers, the **cgroup** has its own memory limit; exceeding it triggers a **cgroup OOM** that kills processes within the cgroup, not system-wide. With cgroup v2: `memory.max`, `memory.events` (track `oom_kill` count), `memory.pressure` (PSI metrics).

Java in a container that sets a 2 GB cgroup limit but configures `-Xmx4g` will be OOM-killed. JVM ergonomics (`-XX:+UseContainerSupport`, default since JDK 10) read the cgroup limit and size heap accordingly. See [Resource Management in Kubernetes](../../kubernetes/configuration/resource-management.md) and [JVM Garbage Collectors](../../java/jvm-gc/collectors.md).

---

## 9. Transparent Huge Pages

Huge pages (2 MB on x86_64) reduce TLB pressure and page-table size. Linux supports two flavors:

- **HugeTLB** (static) — reserved at boot via `vm.nr_hugepages`; databases like Oracle and Postgres `huge_pages=on` use this.
- **THP** (Transparent Huge Pages) — `khugepaged` opportunistically promotes 4 KB pages to 2 MB.

THP modes (`/sys/kernel/mm/transparent_hugepage/enabled`):

- `always` — every anonymous mapping is a candidate.
- `madvise` — only regions marked `MADV_HUGEPAGE`. Recommended.
- `never` — disabled.

THP has burned operators on multiple production workloads:

- **Latency spikes** — `khugepaged` compaction pauses can stall tens of ms.
- **Memory bloat** — fragmentation can cause RSS to balloon.

Most database vendors (MongoDB, Redis, Couchbase) recommend setting THP to `never` or `madvise`. Consult their docs before flipping the switch.

---

## 10. NUMA Basics

On multi-socket servers, RAM is partitioned across **NUMA nodes** (typically one per socket). A core accessing memory on its local node is faster (lower latency, higher bandwidth) than accessing a remote node. The penalty is real — typically 1.5x–2x latency for remote access.

```bash
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0-15
node 0 size: 32768 MB
node 1 cpus: 16-31
node 1 size: 32768 MB
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

Linux defaults are reasonable: allocate from the local node, fall back if needed. Trouble:

- A workload that pins to one node (`numactl --cpunodebind`) but allocates from another → constant remote access.
- An imbalanced workload (Postgres on a 2-socket server with all backends on socket 0) wastes half the bandwidth.

For databases and JVMs that want predictability:

```bash
# Pin a process to NUMA node 0 (CPUs and memory)
numactl --cpunodebind=0 --membind=0 ./prog

# Interleave memory across all nodes
numactl --interleave=all ./prog
```

JVM flag: `-XX:+UseNUMA` enables NUMA-aware heap allocation in supported collectors (Parallel GC, G1).

---

## 11. Reading /proc/meminfo

```
$ cat /proc/meminfo
MemTotal:       65921080 kB     ← total RAM
MemFree:         2156432 kB     ← truly unused (typically small)
MemAvailable:   58823104 kB     ← what apps can grab without swap pressure ★ the one to look at
Buffers:          894016 kB     ← block-device metadata cache
Cached:         48211200 kB     ← page cache (file contents)
SwapCached:        12288 kB
Active:         32108544 kB     ← recently used
Inactive:       17328064 kB     ← LRU candidates
Active(anon):    8112000 kB
Inactive(anon):  1234560 kB
Active(file):   23996544 kB
Inactive(file): 16093504 kB
Dirty:            324120 kB     ← modified, not yet written to disk (see page-cache doc)
Writeback:           512 kB     ← currently being written to disk
AnonPages:       9230336 kB     ← anonymous (heap, stack, anon mmaps)
Mapped:          1928384 kB     ← memory-mapped files (executables, mmap'd files)
Shmem:            514048 kB     ← tmpfs and SysV/POSIX shared memory
Slab:            2890112 kB     ← kernel slab allocator (dentries, inodes, etc.)
SReclaimable:    2403456 kB     ← kernel cache that can be reclaimed under pressure
SUnreclaim:       486656 kB
KernelStack:       28144 kB
PageTables:       108096 kB     ← memory used by page tables themselves
SwapTotal:       8388604 kB
SwapFree:        8376316 kB     ← swap usage
HugePages_Total:       0
HugePages_Free:        0
Hugepagesize:       2048 kB
DirectMap4k:      131072 kB
DirectMap2M:    33554432 kB
DirectMap1G:    33554432 kB     ← kernel's direct map of physical RAM, by page size
```

**The key insight:** `MemFree` is misleadingly small because Linux uses spare RAM as page cache. **`MemAvailable`** is what you want — the kernel's estimate of memory available without inducing swap. "Free memory" alarms based on `MemFree` are almost always wrong.

---

## 12. Backend Engineer Takeaways

- **`malloc(1 GB)` doesn't allocate 1 GB.** RSS grows on first write to each page. Watch `VmRSS` in `/proc/<pid>/status`, not `VmSize`.
- **`free()` rarely returns memory to the OS.** Use jemalloc or accept that a long-running server's RSS is its peak working set, not its current heap.
- **Major faults on a request path are catastrophic.** They mean disk I/O. Audit working-set vs RAM and turn off swap on latency-critical hosts (or set `swappiness=1`).
- **`MemAvailable` > `MemFree`.** Always.
- **In containers, the cgroup memory limit is the OOM ceiling.** JVMs need `-XX:+UseContainerSupport`; Node.js needs `--max-old-space-size` set or `NODE_OPTIONS` aware of the cgroup.
- **THP defaults bite databases.** Read your DB's THP recommendation before deploying.
- **NUMA-pin or NUMA-interleave** explicitly for predictable performance on multi-socket boxes.
- **mmap a file** when you want zero-copy random access; **read** when you want sequential streaming with read-ahead.

---

## Related

- [Process & Thread Model](01-process-thread-model.md)
- [Page Cache & Buffered I/O](03-page-cache-and-buffered-io.md)
- [CPU Scheduling (CFS)](04-cpu-scheduling-cfs.md)
- [JVM Garbage Collectors](../../java/jvm-gc/collectors.md)
- [Kubernetes Resource Management](../../kubernetes/configuration/resource-management.md)

---

## References

- **`mmap(2)`, `madvise(2)`, `mlock(2)` man pages** — https://man7.org/linux/man-pages/man2/mmap.2.html
- **Linux kernel: Memory Management documentation** — https://www.kernel.org/doc/html/latest/admin-guide/mm/index.html
- **Linux kernel: Transparent Hugepage Support** — https://www.kernel.org/doc/html/latest/admin-guide/mm/transhuge.html
- **Linux kernel: NUMA Memory Policy** — https://www.kernel.org/doc/html/latest/admin-guide/mm/numa_memory_policy.html
- **Mel Gorman, *Understanding the Linux Virtual Memory Manager*** — the deep reference, free PDF. https://www.kernel.org/doc/gorman/
- **Brendan Gregg, *Systems Performance*, 2nd ed.** — Chapter 7 (Memory) for diagnostic methodology, Chapter 8 (File Systems) for the page-cache angle.
- **Ulrich Drepper, "What Every Programmer Should Know About Memory"** — RedHat, 2007. Still the canonical reference for memory hierarchy. https://people.freebsd.org/~lstewart/articles/cpumemory.pdf
- **LWN: "Transparent huge pages in 2.6.38"** — https://lwn.net/Articles/423584/
- **MongoDB THP guidance** — https://www.mongodb.com/docs/manual/tutorial/transparent-huge-pages/
- **Linux kernel: cgroup-v2 memory controller** — https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html
