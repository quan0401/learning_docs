---
title: "Virtual Memory & Paging — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, linux, virtual-memory, paging, operating-systems, oom, thp, numa, mmap, cgroups]
---

# Virtual Memory & Paging — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `linux` `virtual-memory` `paging` `operating-systems`

---

## Table of Contents

1. [Summary](#summary)
2. [How to use this doc](#how-to-use-this-doc)
3. [Easy (warm-up traps)](#1-easy-warm-up-traps)
4. [Subtle (review-passers)](#2-subtle-review-passers)
5. [Senior trap (production-only failures)](#3-senior-trap-production-only-failures)
6. [Solutions](#4-solutions)
7. [Related](#related)
8. [References](#references)

---

## Summary

Twenty-three broken snippets covering the ways Linux virtual memory and paging surprise you in production: OOM-killer scoring, transparent huge pages, NUMA placement, swap behaviour, `mmap`/`madvise` semantics, page-cache pollution, dirty-page writeback storms, memory cgroups (v1 vs v2), `vm.overcommit_memory`, hugetlb reservation, KSM, PSI, and `io_uring` SQPOLL accounting. Practice cadence: spot one bug per coffee, peek at the hint only after a real attempt, read the solution last. The bugs are biased toward systems Linux engineers actually trip over — not toy textbook traps.

---

## How to use this doc

- Try to spot the bug before opening `<details>`.
- The hint inside `<details>` is one line; full root cause and fix live in §4 Solutions, keyed by bug number.
- Difficulty is cumulative: don't skip Easy unless you've already nailed those traps in code review.
- Most snippets are shell/`/proc`/sysctl/cgroup/C; a few use Python or systemd unit files where that's the natural surface.

---

## 1. Easy (warm-up traps)

### Bug 1 — "Plenty of free memory" panic
```sh
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           32Gi        21Gi        180Mi       2.0Gi        10Gi        9.5Gi
Swap:         2.0Gi       2.0Gi          0B

# Pager wakes up: "free is 180 MiB, system is OOM!"
$ kill -9 "$(pidof postgres)"
```
<details><summary>Hint</summary>
The wrong column drove the decision.
</details>

### Bug 2 — Critical service OOM-killed first
```sh
# /etc/systemd/system/payments.service
[Service]
ExecStart=/usr/local/bin/payments
MemoryHigh=4G
# (no OOMScoreAdjust set — payments uses ~3.5 GiB)
```
<details><summary>Hint</summary>
Big RSS plus default score equals first to die.
</details>

### Bug 3 — `mmap` of a huge file blocks startup
```c
int fd = open("/var/data/blob-40g.bin", O_RDONLY);
void *p = mmap(NULL, sz, PROT_READ, MAP_PRIVATE | MAP_POPULATE, fd, 0);
// startup probe times out at 30s
```
<details><summary>Hint</summary>
One flag turns lazy paging into eager paging.
</details>

### Bug 4 — RSS doesn't drop after `free()`
```c
void *p = malloc(512 * 1024 * 1024);
memset(p, 0, 512 * 1024 * 1024);
free(p);
// /proc/self/status still shows VmRSS ~ 512 MB. Memory leak?
```
<details><summary>Hint</summary>
The allocator and the kernel are not the same thing.
</details>

### Bug 5 — `mlock` on a 2 GiB region returns `ENOMEM`
```c
if (mlock(buf, 2L << 30) == -1) perror("mlock"); // ENOMEM
// `ulimit -l` shows 64 (KiB)
```
<details><summary>Hint</summary>
A non-root rlimit is silently in the way.
</details>

### Bug 6 — Container OOM-killed at 2 GiB while host has 60 GiB free
```sh
# kubectl describe pod app-7c…
#   Last State: Terminated  Reason: OOMKilled  Exit Code: 137
$ free -h   # on the node
              total   used   free
Mem:           64Gi   4Gi    60Gi
```
<details><summary>Hint</summary>
The kernel is enforcing a different ceiling than `free` reports.
</details>

### Bug 7 — Swap thrash under load
```sh
# /etc/sysctl.d/10-perf.conf
vm.swappiness = 100
vm.vfs_cache_pressure = 1000

# under load, p99 latency jumps 50x; iostat shows the swap device pegged
```
<details><summary>Hint</summary>
The default exists for a reason.
</details>

## 2. Subtle (review-passers)

### Bug 8 — `MADV_DONTNEED` "freed" memory keeps showing up
```c
void *p = mmap(NULL, sz, PROT_READ|PROT_WRITE,
               MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
// ... fill p ...
madvise(p, sz, MADV_FREE); // "I'm done"
memset(p, 0xab, sz);       // reuse later
// expectation: pages never actually reclaimed; reality: zeros after pressure
```
<details><summary>Hint</summary>
`MADV_FREE` and `MADV_DONTNEED` differ on what a write means.
</details>

### Bug 9 — THP `always` causing tail-latency spikes
```sh
$ cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
$ cat /sys/kernel/mm/transparent_hugepage/defrag
[always] defer defer+madvise madvise never

# p99.9 latency jumps from 4 ms to 220 ms during peak
```
<details><summary>Hint</summary>
`defrag=always` makes allocation a synchronous compactor run.
</details>

### Bug 10 — Sequential read evicts hot working set
```sh
# nightly job
tar --create --file=/dev/null /var/lib/data    # 600 GiB scan
# next morning: API p99 doubled, page-cache hit rate collapsed
```
<details><summary>Hint</summary>
Page cache treats the scan and the hot data as equals.
</details>

### Bug 11 — NUMA cross-node access
```sh
$ numactl --hardware | head
available: 2 nodes (0-1)
node 0 size: 256 GB
node 1 size: 256 GB

$ taskset -c 0-15 ./worker  # pinned to node 0 cores
$ numastat -p $(pidof worker)
                 Node 0   Node 1
Private          12 GB    180 GB    # ← surprise
```
<details><summary>Hint</summary>
CPU pinning didn't pin allocations.
</details>

### Bug 12 — `vm.dirty_ratio` tuned for SSD on a slow EBS volume
```sh
$ sysctl vm.dirty_ratio vm.dirty_background_ratio
vm.dirty_ratio = 40
vm.dirty_background_ratio = 20
# 256 GiB host, EBS gp2 baseline. Periodic 30s freezes during fsync.
```
<details><summary>Hint</summary>
40% of 256 GiB has to drain through one slow device.
</details>

### Bug 13 — `vm.overcommit_memory=2` rejects normal Postgres start
```sh
$ sysctl vm.overcommit_memory vm.overcommit_ratio
vm.overcommit_memory = 2
vm.overcommit_ratio  = 50
# 64 GiB RAM, no swap; Postgres fails: "could not map anonymous shared memory"
```
<details><summary>Hint</summary>
Strict mode caps `CommitLimit`; check the math.
</details>

### Bug 14 — Shared-memory collision in two containers
```yaml
# compose.yml
services:
  app-a:
    image: app
    ipc: host
  app-b:
    image: app
    ipc: host
# both call shm_open("/cache", O_CREAT|O_RDWR, 0600); they "see each other's" data
```
<details><summary>Hint</summary>
`ipc: host` shares the namespace by design.
</details>

### Bug 15 — Memory cgroup v1 vs v2 `kmem` accounting
```sh
# Migration from cgroup v1 to v2; same MemoryMax=4G.
# After migration, container OOMs on workloads that fit fine before.
$ cat /proc/cgroups | grep memory   # v1
$ cat /sys/fs/cgroup/cgroup.controllers  # v2
```
<details><summary>Hint</summary>
v2 folds slab into the limit by default; v1 didn't, unless you enabled `kmem`.
</details>

### Bug 16 — Hugetlb reservation fails after uptime grows
```sh
# fresh boot
$ echo 1024 > /proc/sys/vm/nr_hugepages
$ cat /proc/meminfo | grep HugePages_
HugePages_Total: 1024
HugePages_Free:  1024

# 30 days later, same command:
$ echo 1024 > /proc/sys/vm/nr_hugepages
$ cat /proc/meminfo | grep HugePages_
HugePages_Total:  213
```
<details><summary>Hint</summary>
2 MiB contiguous physical extents are not free for the asking.
</details>

### Bug 17 — `available` vs `free` in an alert rule
```yaml
# prometheus rule
- alert: HostLowMemory
  expr: node_memory_MemFree_bytes / node_memory_MemTotal_bytes < 0.05
  for: 5m
# fires constantly on healthy hosts running Postgres
```
<details><summary>Hint</summary>
`free` ignores reclaimable cache; the kernel exports a better number.
</details>

## 3. Senior trap (production-only failures)

### Bug 18 — `MAP_PRIVATE` writable mapping faults the world after `fork`
```c
// parent
void *p = mmap(NULL, 16L<<30, PROT_READ|PROT_WRITE,
               MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
fill_with_data(p, 16L<<30);
pid_t pid = fork();
if (pid == 0) {
    for (size_t i = 0; i < 16L<<30; i += 4096) ((char*)p)[i] = 1; // CoW storm
}
```
<details><summary>Hint</summary>
Every first write in the child copies a page.
</details>

### Bug 19 — Memory leak invisible because `malloc_trim` was never called
```c
// long-lived daemon, glibc malloc
// /proc/<pid>/status shows VmRSS plateaued at 12 GiB for hours,
// but heap profilers say live set is 800 MiB.
// Operator concludes: "leak". Engineer can't reproduce in tests.
```
<details><summary>Hint</summary>
glibc's arena holds onto pages until you nudge it.
</details>

### Bug 20 — Anonymous memory "unswappable" because `noswap` boot param set
```sh
$ cat /proc/cmdline
BOOT_IMAGE=/vmlinuz-6.8 root=UUID=… ro quiet noswap
$ swapon --show
# (empty)
# OOM kills under burst load even though /proc/meminfo Inactive(anon) ~ 18 GiB
```
<details><summary>Hint</summary>
`noswap` (added in 6.3) is doing exactly what it says.
</details>

### Bug 21 — KSM merging slows VM startup
```sh
$ echo 1 > /sys/kernel/mm/ksm/run
$ cat /sys/kernel/mm/ksm/pages_to_scan
1000

# launching 200 cloned VMs: boot p99 goes from 12s to 90s
$ ps -eo pid,comm,pcpu | grep ksmd
  78 ksmd            38.0
```
<details><summary>Hint</summary>
The dedup daemon is competing with your boot path for CPU.
</details>

### Bug 22 — PSI memory pressure ignored, autoscaler thrashes
```yaml
# HPA
metrics:
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 80
# Pods with 70% RSS but constant memory PSI > 30% are starved; autoscaler does nothing.
```
<details><summary>Hint</summary>
Utilization is a level; pressure is a derivative.
</details>

### Bug 23 — `io_uring` SQPOLL pinned memory not accounted to cgroup
```c
struct io_uring_params p = { .flags = IORING_SETUP_SQPOLL };
io_uring_queue_init_params(4096, &ring, &p);
io_uring_register_buffers(&ring, iov, n_bufs);
// Container MemoryMax=2G; RSS sits at 1.6G but host ksoftirqd shows
// extra 600 MB pinned. Node-level OOM, not cgroup OOM.
```
<details><summary>Hint</summary>
Some pinned pages don't go where you think they do.
</details>

---

## 4. Solutions

### Bug 1 — "Plenty of free memory" panic
**Root cause:** `free` is a low-water column, not a pressure signal. With Postgres running, ~10 GiB is page cache (`buff/cache`), most of which is reclaimable. The `available` column (9.5 GiB) is the kernel's estimate of how much can be made available without swapping. Killing Postgres because `free=180Mi` was a false alarm.
**Fix:** alert on `MemAvailable` (or PSI-memory), never on `MemFree`. From `/proc/meminfo`:
```sh
awk '/MemAvailable/ {print $2}' /proc/meminfo
```
**Reference:** kernel.org `Documentation/filesystems/proc.rst` — `meminfo` section; `MemAvailable` was added explicitly so monitoring tools would stop misreading `MemFree`.

### Bug 2 — Critical service OOM-killed first
**Root cause:** Default `oom_score_adj=0`. The OOM killer ranks by `oom_score`, which scales with RSS. The largest non-protected process dies first. A 3.5 GiB payments service with no protection beats every shell, sshd, and metric exporter on the box.
**Fix:** lower the score for critical services; raise it for batch:
```ini
[Service]
OOMScoreAdjust=-500
```
**Reference:** `man 5 oom_score` (`/proc/[pid]/oom_score_adj` ranges -1000..1000); kernel.org `Documentation/filesystems/proc.rst`.

### Bug 3 — `mmap` of a huge file blocks startup
**Root cause:** `MAP_POPULATE` tells the kernel to prefault every page synchronously. For a 40 GiB cold file that's 40 GiB of disk reads before `mmap` returns.
**Fix:** drop `MAP_POPULATE`; rely on demand paging, or explicitly prefetch the hot range:
```c
void *p = mmap(NULL, sz, PROT_READ, MAP_PRIVATE, fd, 0);
madvise(p, hot_sz, MADV_WILLNEED); // bounded, optional
```
**Reference:** `man 2 mmap` — "MAP_POPULATE … For a file mapping, this causes read-ahead on the file."

### Bug 4 — RSS doesn't drop after `free()`
**Root cause:** glibc returns the chunk to its arena, not to the kernel. Pages stay mapped until the arena decides to release them (typically only at process exit, or on `malloc_trim`). This is correct behaviour, not a leak.
**Fix:** if you actually need the pages back:
```c
malloc_trim(0);
```
For huge transient buffers, allocate via `mmap` directly so `free` -> `munmap` actually unmaps.
**Reference:** `man 3 malloc_trim`; glibc manual "The GNU Allocator".

### Bug 5 — `mlock` on a 2 GiB region returns `ENOMEM`
**Root cause:** `RLIMIT_MEMLOCK` caps the bytes a non-root process may lock. Default on most distros is 64 KiB. `mlock` returns `ENOMEM` when the request exceeds it.
**Fix:** raise the limit for the service unit, not globally:
```ini
[Service]
LimitMEMLOCK=infinity
```
**Reference:** `man 2 mlock` — "ENOMEM (Linux 2.6.9 and later) the caller had a nonzero RLIMIT_MEMLOCK soft resource limit, but tried to lock more memory than the limit permitted."

### Bug 6 — Container OOM-killed at 2 GiB while host has 60 GiB free
**Root cause:** `MemoryMax` (cgroup v2) or `memory.limit_in_bytes` (v1) caps the cgroup, not the host. The kernel OOM-kills the pod when its cgroup hits the ceiling, regardless of host headroom.
**Fix:** either raise the limit, or fix the workload. Inspect the actual cgroup:
```sh
cat /sys/fs/cgroup/<path>/memory.current
cat /sys/fs/cgroup/<path>/memory.events   # oom, oom_kill counters
```
**Reference:** kernel.org `Documentation/admin-guide/cgroup-v2.rst` — Memory controller, `memory.max`, `memory.events`.

### Bug 7 — Swap thrash under load
**Root cause:** `vm.swappiness=100` tells the kernel to prefer evicting anonymous pages over reclaiming page cache. Combined with `vfs_cache_pressure=1000` (hyper-aggressive dentry/inode reclaim), the system spends real CPU/IO churning instead of running the workload. Default `vm.swappiness` is 60 for a reason.
**Fix:** keep defaults unless you have a measured reason. For latency-sensitive servers, 10 is a defensible lower bound; 0 disables anonymous swap entirely (risky on cgroup-bound systems).
**Reference:** kernel.org `Documentation/admin-guide/sysctl/vm.rst` — `swappiness`, `vfs_cache_pressure`.

### Bug 8 — `MADV_DONTNEED` "freed" memory keeps showing up
**Root cause:** confusion between two semantics:
- `MADV_DONTNEED` (anonymous): pages are *immediately* discarded, RSS drops, next read returns zero.
- `MADV_FREE`: pages *may* be reclaimed under pressure, but a write before reclaim cancels the freeing — your data is preserved.

The snippet wrote to `p` after `MADV_FREE`, so the kernel kept the original pages. After actual pressure, the kernel had already discarded; the next read got zeros.
**Fix:** pick the right call for the intent. `MADV_FREE` is for "I might come back, save the I/O if you can"; `MADV_DONTNEED` is for "I'm done, drop them now."
**Reference:** `man 2 madvise` — MADV_FREE vs MADV_DONTNEED; kernel.org `Documentation/admin-guide/mm/concepts.rst`.

### Bug 9 — THP `always` causing tail-latency spikes
**Root cause:** with `defrag=always`, allocating a 2 MiB THP triggers synchronous direct compaction inside the faulting thread. On a fragmented host this can stall hundreds of ms. khugepaged's background path doesn't help when the *fault path* is synchronous.
**Fix:** use `madvise` mode globally and let opt-in callers ask for THP:
```sh
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
echo defer+madvise > /sys/kernel/mm/transparent_hugepage/defrag
```
**Reference:** kernel.org `Documentation/admin-guide/mm/transhuge.rst` — "defrag" modes; LWN, "Transparent huge pages in 2.6.38" and follow-ups on tail-latency regressions.

### Bug 10 — Sequential read evicts hot working set
**Root cause:** the page cache uses an LRU-ish policy. A 600 GiB linear scan walks every page once, evicting hot Postgres pages on the way through. Classic cache pollution.
**Fix:** advise the kernel that the scan is one-shot:
```sh
nocache tar --create --file=/dev/null /var/lib/data
# or in code:
posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);
```
Better still, avoid using buffered I/O at all for one-shot scans (`O_DIRECT`).
**Reference:** `man 2 posix_fadvise`; kernel.org `Documentation/admin-guide/mm/concepts.rst` ("Reclaim").

### Bug 11 — NUMA cross-node access
**Root cause:** Linux's default NUMA policy is "first-touch": pages allocate on the node of the CPU that first writes them. If the worker was started by a launcher on node 1, the early heap pages may be on node 1. Pinning CPUs after the fact doesn't migrate the pages.
**Fix:** bind both CPU and memory at launch:
```sh
numactl --cpunodebind=0 --membind=0 ./worker
```
Or, for long-running services, use `MPOL_BIND` via `set_mempolicy(2)`.
**Reference:** `man 7 numa`; kernel.org `Documentation/admin-guide/mm/numa_memory_policy.rst`.

### Bug 12 — `vm.dirty_ratio` tuned for SSD on a slow EBS volume
**Root cause:** `dirty_ratio=40` means 40% of available memory (~100 GiB on a 256 GiB box) can be dirty before *writers* are throttled. When the kernel finally drains, it does so through a device with limited IOPS, blocking everyone behind `fsync`.
**Fix:** size the dirty window to what the device can flush in seconds, not gigabytes:
```sh
sysctl -w vm.dirty_bytes=$((256 * 1024 * 1024))
sysctl -w vm.dirty_background_bytes=$((64 * 1024 * 1024))
```
Use the `*_bytes` variants on big-RAM hosts; the `*_ratio` percentages were sized for 1990s machines.
**Reference:** kernel.org `Documentation/admin-guide/sysctl/vm.rst` — `dirty_ratio`, `dirty_bytes`; LWN, "Flushing out pdflush" and successor articles on writeback.

### Bug 13 — `vm.overcommit_memory=2` rejects normal Postgres start
**Root cause:** mode 2 (strict accounting) caps total committed virtual memory at `CommitLimit = swap + RAM * overcommit_ratio/100`. With 64 GiB RAM, no swap, ratio 50, the limit is 32 GiB — Postgres' shared-memory request bumps into it long before real RAM is exhausted.
**Fix:** raise the ratio (or add swap), or stay on mode 0 (heuristic) for hosts that don't need the strictness:
```sh
sysctl -w vm.overcommit_memory=0
# or
sysctl -w vm.overcommit_ratio=90
```
**Reference:** kernel.org `Documentation/admin-guide/sysctl/vm.rst` — `overcommit_memory`, `overcommit_ratio`; `Documentation/mm/overcommit-accounting.rst`.

### Bug 14 — Shared-memory collision in two containers
**Root cause:** `ipc: host` shares the host's IPC namespace, including POSIX shm names under `/dev/shm`. Two containers calling `shm_open("/cache", ...)` get the *same* segment.
**Fix:** drop `ipc: host` (default is per-container `private`), or namespace the names per-instance:
```c
char name[64];
snprintf(name, sizeof name, "/cache.%d", getpid());
shm_open(name, O_CREAT|O_RDWR, 0600);
```
**Reference:** Docker docs — IPC mode; `man 7 ipc_namespaces`.

### Bug 15 — Memory cgroup v1 vs v2 `kmem` accounting
**Root cause:** under cgroup v1, slab/kernel memory was accounted only when `memory.kmem.limit_in_bytes` was explicitly set. Under cgroup v2, kmem is always counted into `memory.current` against `memory.max`. Workloads with heavy dentry/inode usage suddenly see "more" memory used after migration.
**Fix:** raise the limit, or trim the kmem footprint (smaller mount cache, fewer open files, less aggressive `vfs_cache_pressure`).
**Reference:** kernel.org `Documentation/admin-guide/cgroup-v2.rst` — Memory controller, "Memory ownership"; v1 docs `Documentation/admin-guide/cgroup-v1/memory.rst`.

### Bug 16 — Hugetlb reservation fails after uptime grows
**Root cause:** `nr_hugepages` requires *physically contiguous* 2 MiB extents. After uptime, physical memory is fragmented; the kernel can only assemble 213 of the requested 1024.
**Fix:** reserve at boot via the kernel command line, before fragmentation:
```text
hugepages=1024
```
Or use `hugepagesz=1G` for 1 GiB pages with the same boot-time guarantee.
**Reference:** kernel.org `Documentation/admin-guide/mm/hugetlbpage.rst`.

### Bug 17 — `available` vs `free` in an alert rule
**Root cause:** `MemFree` excludes reclaimable page cache. A healthy DB host running Postgres will pin most of RAM as cache; `MemFree` will hover near zero. `MemAvailable` is the kernel's own estimate of "memory you could obtain without swapping."
**Fix:**
```yaml
expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.10
```
**Reference:** kernel.org `Documentation/filesystems/proc.rst` — `MemAvailable` (introduced in commit 34e431b0ae39).

### Bug 18 — `MAP_PRIVATE` writable mapping faults the world after `fork`
**Root cause:** `fork` shares pages copy-on-write. The child sees the parent's 16 GiB intact but cheaply; the first *write* to each page triggers a fault and a page copy. A linear scan that writes every page faults 4 million pages — minutes of wallclock and a dirty-page storm.
**Fix:** if you want the child to share read-only, leave it. If both processes need to mutate without copying, use `MAP_SHARED`. If you only need to scan, use a worker thread instead of `fork`.
**Reference:** `man 2 fork` ("Memory mappings"); LWN, "Optimizing the copy-on-write fork pattern".

### Bug 19 — Memory leak invisible because `malloc_trim` was never called
**Root cause:** glibc's `malloc` keeps freed chunks in arenas, sized to the high-water mark of the workload. Long-lived daemons with bursty allocations look like leaks under `top`/`ps` even when the live set is small. Heap profilers (jemalloc, heaptrack) see the truth.
**Fix:** call `malloc_trim(0)` after large transient bursts, or switch to jemalloc/tcmalloc which return memory more aggressively. Better: profile before declaring a leak.
**Reference:** `man 3 malloc_trim`; jemalloc design notes; postmortems from Datadog and Cloudflare describing arena-bloat misdiagnoses (e.g., Cloudflare's "Affinity-based allocator" series on blog.cloudflare.com).

### Bug 20 — Anonymous memory "unswappable" because `noswap` boot param set
**Root cause:** `noswap` (kernel cmdline, Linux 6.3+) disables swap support entirely — even active swap files are ignored. Anonymous pages have nowhere to evict to under pressure, so the kernel jumps straight to OOM.
**Fix:** remove `noswap` from `/etc/default/grub` and `update-grub`/`grub-mkconfig`; reboot. Then size swap appropriately (zswap or a swapfile sized for tail bursts is usually enough).
**Reference:** kernel.org `Documentation/admin-guide/kernel-parameters.txt` — `noswap`; commit log on lore.kernel.org introducing the parameter.

### Bug 21 — KSM merging slows VM startup
**Root cause:** KSM (`ksmd`) scans anonymous pages looking for identical content to merge. During mass VM clone start, page contents change rapidly, but ksmd is still scanning aggressively. CPU contention on boot path.
**Fix:** lower `pages_to_scan` (e.g., 100), raise `sleep_millisecs`, or disable KSM during the burst:
```sh
echo 0 > /sys/kernel/mm/ksm/run
# … boot the fleet …
echo 1 > /sys/kernel/mm/ksm/run
```
KSM is per-mapping opt-in via `madvise(MADV_MERGEABLE)` unless you've enabled `merge_across_nodes`.
**Reference:** kernel.org `Documentation/admin-guide/mm/ksm.rst`.

### Bug 22 — PSI memory pressure ignored, autoscaler thrashes
**Root cause:** `memory.utilization` is a level — the workload is "using a lot." Pressure Stall Information (PSI) measures the *time* tasks were stalled waiting for memory. A workload at 70% utilization with sustained 30% memory PSI is being throttled every second; the HPA can't see that.
**Fix:** scrape `/proc/pressure/memory` (or per-cgroup `memory.pressure`) and scale on the `some avg10` value:
```sh
cat /proc/pressure/memory
# some avg10=22.5 avg60=18.0 avg300=12.0 total=...
```
Use it as a custom metric for the HPA.
**Reference:** kernel.org `Documentation/accounting/psi.rst`; LWN, "Pressure stall information for ages" (Facebook's introduction).

### Bug 23 — `io_uring` SQPOLL pinned memory not accounted to cgroup
**Root cause:** `io_uring_register_buffers` pins user pages for DMA. With SQPOLL, a kernel thread services the ring; depending on kernel version and registration path, the pinned bytes have not always been fully accounted to the issuing cgroup, leading to host-level OOM while the cgroup looks under-budget. The exact accounting has changed across releases — verify on your kernel.
**Fix:** size `MemoryMax` with headroom for registered buffers; account for them yourself in capacity planning. Track host-level `vmallocinfo` and `/proc/meminfo` `Mlocked`/`Unevictable` to see the real footprint. Track upstream io_uring fixes in lore.kernel.org `io-uring@vger.kernel.org`.
**Reference:** `man 2 io_uring_register`; `man 2 io_uring_setup` (IORING_SETUP_SQPOLL); kernel.org `Documentation/admin-guide/cgroup-v2.rst` (Memory controller, "Non-reclaimable" memory).

---

## Related

- [Virtual Memory & Paging](02-virtual-memory-and-paging.md) — concept doc this practice anchors against.
- [Page Cache & Buffered I/O](03-page-cache-and-buffered-io.md) — page cache, dirty writeback, fsync paths.
- [File Descriptors & Ulimits](06-file-descriptors-and-ulimits.md) — `RLIMIT_MEMLOCK` and friends.
- [Operating Systems INDEX](../INDEX.md)

## References

- kernel.org `Documentation/admin-guide/mm/transhuge.rst` — Transparent Huge Pages.
- kernel.org `Documentation/admin-guide/mm/concepts.rst` — reclaim, OOM, dirty writeback.
- kernel.org `Documentation/admin-guide/mm/numa_memory_policy.rst` — NUMA memory policy.
- kernel.org `Documentation/admin-guide/mm/hugetlbpage.rst` — explicit hugepages.
- kernel.org `Documentation/admin-guide/mm/ksm.rst` — Kernel Samepage Merging.
- kernel.org `Documentation/admin-guide/cgroup-v2.rst` — Memory controller, `memory.max`, `memory.events`, PSI.
- kernel.org `Documentation/admin-guide/sysctl/vm.rst` — `swappiness`, `dirty_ratio`, `overcommit_memory`, `overcommit_ratio`.
- kernel.org `Documentation/accounting/psi.rst` — Pressure Stall Information.
- kernel.org `Documentation/filesystems/proc.rst` — `/proc/meminfo`, `MemAvailable`.
- `man 2 mmap`, `man 2 madvise`, `man 2 mlock`, `man 5 oom_score`, `man 7 numa`, `man 2 io_uring_register`, `man 3 malloc_trim`.
- LWN articles on THP defrag modes, writeback throttling, PSI, and io_uring buffer registration: lwn.net.
- Brendan Gregg — "Linux Memory" pages on brendangregg.com (RSS vs PSS, page cache observability).
- Linux kernel mailing list archive: lore.kernel.org (search `noswap`, `io_uring SQPOLL accounting`).
