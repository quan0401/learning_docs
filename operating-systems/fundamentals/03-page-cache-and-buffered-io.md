---
title: "Page Cache & Buffered I/O — dirty pages, writeback, fsync, O_DIRECT"
date: 2026-05-03
updated: 2026-05-03
tags: [operating-systems, linux, page-cache, fsync, buffered-io, postgres, sendfile]
---

# Page Cache & Buffered I/O — dirty pages, writeback, fsync, O_DIRECT

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `operating-systems` `linux` `page-cache` `fsync` `buffered-io` `postgres` `sendfile`

---

## Table of Contents

- [Summary](#summary)
- [1. What the Page Cache Is](#1-what-the-page-cache-is)
- [2. The Read Path](#2-the-read-path)
  - [2.1 Read-Ahead](#21-read-ahead)
- [3. The Write Path and Dirty Pages](#3-the-write-path-and-dirty-pages)
  - [3.1 Writeback Tunables](#31-writeback-tunables)
- [4. fsync, fdatasync, sync_file_range](#4-fsync-fdatasync-sync_file_range)
  - [4.1 What fsync actually guarantees](#41-what-fsync-actually-guarantees)
  - [4.2 The fsync crash-safety landmine](#42-the-fsync-crash-safety-landmine)
- [5. O_DIRECT — Bypassing the Page Cache](#5-o_direct--bypassing-the-page-cache)
- [6. sendfile and Zero-Copy](#6-sendfile-and-zero-copy)
- [7. Inspecting the Page Cache](#7-inspecting-the-page-cache)
  - [7.1 vmtouch, pcstat, fincore](#71-vmtouch-pcstat-fincore)
  - [7.2 posix_fadvise](#72-posix_fadvise)
- [8. Why "free memory" Lies](#8-why-free-memory-lies)
- [9. Database Interactions](#9-database-interactions)
  - [9.1 PostgreSQL](#91-postgresql)
  - [9.2 MySQL / InnoDB](#92-mysql--innodb)
  - [9.3 Other engines](#93-other-engines)
- [10. Backend Engineer Takeaways](#10-backend-engineer-takeaways)
- [Related](#related)
- [References](#references)

---

## Summary

Linux keeps a unified **page cache** of file data in RAM. Every `read()` and `write()` against a regular file goes through it: a read fills cache pages from disk; a write marks pages **dirty** and the kernel writes them back asynchronously. This caching is fast, transparent, and almost always what you want — but it changes the durability story (your data isn't on disk after `write()` returns), distorts memory accounting, and forces databases to make a hard choice: trust the page cache, manage their own buffer pool with `O_DIRECT`, or do both. This document explains the structure of the page cache, the fsync semantics every engineer should be paranoid about, and the page-cache-shaped reasons your "free memory" alarm is misleading.

---

## 1. What the Page Cache Is

The page cache is the kernel's cache of file contents, indexed by `(inode, page-offset)`. A 4 KB chunk of a file resides at most once in RAM, regardless of how many processes have it open or mmap'd. The page cache is unified with:

- **Buffered `read()`/`write()`** — copy from/to user buffer.
- **`mmap(MAP_SHARED|PRIVATE)`** — direct mapping of cache pages into the process's address space.
- **`sendfile()`** — kernel-internal page-cache-to-socket copy (see [Section 6](#6-sendfile-and-zero-copy)).

Cache pages are tracked on LRU-like lists (active vs inactive). Under memory pressure, the kernel evicts clean pages first; dirty pages must be written back before eviction (see [Section 3](#3-the-write-path-and-dirty-pages)).

```
       userspace process
             │
             │ read(fd, buf, n)
             ▼
   ┌─────────────────────┐         miss
   │     Page Cache      │ ────────────────────► block layer ──► disk
   │  (inode, offset) →  │ ◄────────────────────  fill page
   │       4K page       │           hit
   └─────────────────────┘
             │
             │ copy_to_user
             ▼
        user buffer
```

The **buffer cache** of legacy Unix is, on Linux, just metadata blocks (inodes, dentries, free-space bitmaps) sitting in the same page-cache infrastructure. Hence `Buffers:` in `/proc/meminfo` is small and `Cached:` is the big number.

---

## 2. The Read Path

When user code calls `read(fd, buf, count)` on a regular file:

1. VFS finds the inode, computes `(inode, page_index)` for the relevant pages.
2. For each page:
   - **Hit** — page is in the cache. Copy to user buffer. Counted as a **minor fault** for `mmap` access.
   - **Miss** — kernel allocates a page, queues a block I/O request, blocks the calling thread (in `D` state) until the read completes, then copies to user buffer.
3. Read-ahead may queue additional pages.

### 2.1 Read-Ahead

The kernel detects sequential access patterns and **reads ahead** — fetching pages the application hasn't asked for yet — into the page cache. Tunables:

```bash
# Per-block-device read-ahead in 512-byte sectors (default usually 256 = 128 KB)
$ blockdev --getra /dev/sda
$ blockdev --setra 4096 /dev/sda    # 2 MB read-ahead

# Per-file hint (call from your code)
posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);  // double read-ahead
posix_fadvise(fd, 0, 0, POSIX_FADV_RANDOM);      // disable read-ahead
```

For sequential workloads (log scans, full-table scans, file copies), aggressive read-ahead massively reduces wall-clock time. For random-access workloads (OLTP databases), it wastes I/O bandwidth — which is why databases often disable it.

---

## 3. The Write Path and Dirty Pages

```
   write(fd, buf, n)                            background flusher
        │                                              │
        ▼                                              ▼
   copy buf → page cache page                ┌─────────────────┐
   mark page DIRTY ──────────────────────────►   writeback     │
   return                                     │  flusher thread │
                                              └─────────────────┘
                                                       │
                                                       ▼
                                                  block layer ──► disk
```

A `write()` that returns success means the bytes are in the **page cache**, not on disk. They are written back asynchronously by per-bdi (backing device info) **flusher threads** triggered by:

- **Time** — pages dirty for longer than `dirty_expire_centisecs` (default 30s).
- **Quantity** — total dirty pages exceed `dirty_background_ratio`/`dirty_background_bytes` → background flush; exceed `dirty_ratio`/`dirty_bytes` → blocking flush from the writing process (latency hiccup).
- **Explicit** — `fsync()`, `fdatasync()`, `sync()`, `sync_file_range()`.
- **Memory pressure** — kernel needs the pages back.

### 3.1 Writeback Tunables

```bash
# Defaults shown for a typical Linux server
$ sysctl vm.dirty_background_ratio   # 10  — start background writeback at 10% of available memory
$ sysctl vm.dirty_ratio              # 20  — block writers at 20%
$ sysctl vm.dirty_expire_centisecs   # 3000  (30s)
$ sysctl vm.dirty_writeback_centisecs # 500  (5s) — flusher wake-up interval
```

For database hosts with large RAM, the percentage-based defaults can produce huge bursts of dirty data that lead to seconds-long write stalls when the threshold is crossed. Switching to byte-based limits (`dirty_background_bytes`, `dirty_bytes`) caps the burst size:

```bash
# Cap dirty pages at 256 MB / 1 GB on a 64 GB box
sysctl -w vm.dirty_background_bytes=$((256*1024*1024))
sysctl -w vm.dirty_bytes=$((1024*1024*1024))
sysctl -w vm.dirty_background_ratio=0
sysctl -w vm.dirty_ratio=0
```

Observe pressure with `cat /proc/meminfo | grep -iE 'Dirty|Writeback'` and `iostat -x 1` (look at `await` and `%util`).

---

## 4. fsync, fdatasync, sync_file_range

### 4.1 What fsync actually guarantees

`fsync(fd)` blocks until:

1. All dirty pages of the file have been submitted to the block layer.
2. The block layer has reported completion (which means written to the **storage device**, including issuing a **cache flush** command — `FLUSH CACHE` for ATA, `SYNCHRONIZE CACHE` for SCSI/NVMe).
3. The file's metadata (size, mtime, allocated extents) is also durable.

`fdatasync(fd)` skips step 3's metadata flush **unless** the metadata change affects readability (e.g., a file that grew). It's faster for append-heavy workloads (write-ahead logs) where mtime changes don't matter.

`sync_file_range(fd, offset, nbytes, flags)` — Linux-specific, lets you write back a byte range without an associated cache flush. **Does not provide durability** by itself; use it to nudge writeback before a final `fsync()`.

### 4.2 The fsync crash-safety landmine

Bugs and subtleties in fsync semantics have repeatedly broken databases:

- **The 2018 "fsyncgate"** — Postgres discovered that on some Linux versions, when an asynchronous writeback fails (e.g., an EIO from the disk), the kernel cleared the dirty bit on the page **and** lost the error. A subsequent `fsync()` returned success. Postgres now panics on any `fsync` error to avoid silent corruption. See *Craig Ringer's writeup*, lwn.net/Articles/752063/.

- **`write() + close()` is not durable.** Closing an fd does not fsync. A power loss after `close()` can leave you with a file containing zeros (filesystems may journal metadata before data).

- **Renaming the parent directory is not durable** until you `fsync` the directory itself. The "rename atomic write" pattern (`write(tmp); fsync(tmp); rename(tmp, real); fsync(parent)`) requires the final `fsync(dir_fd)` to be crash-safe.

- **`O_DIRECT` writes still need fsync** for the device cache flush.

For TypeScript/Node:
```ts
import { open } from 'node:fs/promises';
const fh = await open('data.bin', 'w');
await fh.write(buf);
await fh.sync();        // fsync(2)
// or fh.datasync();    // fdatasync(2)
await fh.close();
```

For Java:
```java
try (FileChannel ch = FileChannel.open(path, WRITE, CREATE)) {
    ch.write(buf);
    ch.force(true);   // fsync (false = fdatasync semantics)
}
```

---

## 5. O_DIRECT — Bypassing the Page Cache

Opening a file with `O_DIRECT` makes the kernel transfer data **directly between the user buffer and the device**, skipping the page cache. Constraints (Linux):

- The user buffer must be **aligned** to the device block size (typically 512 B or 4 KB) — use `posix_memalign`.
- The transfer **size and offset** must also be block-aligned.
- Mixing buffered and direct I/O on the same file leads to undefined behavior (since the cached and on-disk views diverge).

When to use O_DIRECT:

- **Database engines that manage their own buffer pool** — InnoDB (`innodb_flush_method=O_DIRECT`), Oracle, MS SQL on Linux. Caching twice (in InnoDB buffer pool and the kernel page cache) wastes RAM.
- **Bulk copies** that you don't want to evict the page cache (`dd oflag=direct`).

When *not* to use O_DIRECT:

- General application code. The page cache is usually a win.
- PostgreSQL — it deliberately uses buffered I/O and a smaller `shared_buffers`, leaning on the OS cache. (See [Section 9.1](#91-postgresql).)

---

## 6. sendfile and Zero-Copy

The classic file-to-socket path:

```
disk → page cache → user buffer (via read) → kernel socket buffer (via write) → NIC
                ↑                          ↑
            copy 1                     copy 2
```

`sendfile(out_fd, in_fd, offset, count)` collapses these into a kernel-internal transfer:

```
disk → page cache ──────────────────────────────────► socket buffer → NIC
                  (no user-space copy, no context switch per chunk)
```

Modern Linux extends this with `splice()` (move data between fds via a pipe) and `MSG_ZEROCOPY` for outbound socket sends.

Used by:
- **nginx** (`sendfile on;`), Apache, HAProxy serving static files.
- **Kafka** — broker-to-consumer delivery uses `sendfile` to move log segments from page cache to socket without copying through the JVM heap. This is a significant part of why Kafka throughput is high.
- **Java NIO** — `FileChannel.transferTo()` lowers to `sendfile` when the destination is a `SocketChannel` on Linux.

---

## 7. Inspecting the Page Cache

### 7.1 vmtouch, pcstat, fincore

```bash
# How much of /var/log/syslog is in the page cache?
$ vmtouch /var/log/syslog
           Files: 1
     Directories: 0
  Resident Pages: 12345/45678  48.2M/178M  27%
         Elapsed: 0.00481 seconds

# Pre-warm a file into cache (lock pages in)
$ vmtouch -t mydata.bin              # touch all pages → load into cache
$ vmtouch -l mydata.bin              # mlock pages so they can't be evicted
                                     # ← careful: this competes with everything else

# Drop the cache (NEVER on a production server with a hot working set)
$ echo 3 > /proc/sys/vm/drop_caches  # 1 = pagecache, 2 = dentries+inodes, 3 = both
```

`pcstat` (Go binary by Brendan Gregg/davecheney) gives per-file cached-percentage breakdowns. `fincore` is a coreutils-friendly alternative.

### 7.2 posix_fadvise

Tell the kernel your access pattern:

```c
posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);   // increase read-ahead
posix_fadvise(fd, 0, 0, POSIX_FADV_RANDOM);       // disable read-ahead
posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);     // drop pages from cache
posix_fadvise(fd, 0, 0, POSIX_FADV_WILLNEED);     // pre-fault pages now
```

`POSIX_FADV_DONTNEED` after sequentially streaming a giant file is the classic way to avoid evicting the rest of the cache for a one-shot scan (`rsync --drop-cache`, modern backup tools).

---

## 8. Why "free memory" Lies

`MemFree` in `/proc/meminfo` is RAM not used for anything — including page cache. On a healthy long-running server, `MemFree` will be near zero because the kernel uses every unused page as cache. **This is correct behavior.**

The metric that matters is **`MemAvailable`** — the kernel's estimate of how much memory is reclaimable for new allocations without triggering swap. If `MemAvailable` stays comfortably above your peak working-set spike, you're fine.

```
MemTotal:    65 GB
MemFree:    250 MB    ← scary!
Cached:      40 GB    ← but this is reclaimable
MemAvailable: 42 GB   ← actually fine
```

Alarms wired to `MemFree` on a database/file server fire constantly and mean nothing. Wire them to `MemAvailable / MemTotal < 0.10` instead.

---

## 9. Database Interactions

### 9.1 PostgreSQL

Postgres uses **buffered I/O** and a relatively small `shared_buffers` (default 128 MB; recommended 25% of RAM but capped at a few GB). It deliberately relies on the OS page cache as a second-tier buffer pool. Implications:

- **Double buffering**: a hot page sits in `shared_buffers` *and* in the page cache. Wasted? Marginally — but Postgres's CLOCK-style replacement and the kernel's LRU tend to differ, so each catches different things.
- **`pg_prewarm` extension** can pre-load relations into `shared_buffers` and the page cache at startup.
- **Checkpoint storms**: at checkpoint time, Postgres writes out dirty buffers; if the kernel's dirty-page limit is high, this can hit the disk all at once. `vm.dirty_bytes` byte-based caps prevent this.
- **fsync configuration**: `fsync = on` (default; never turn off in production), `wal_sync_method = fdatasync|fsync|open_datasync|open_sync` — choose per-platform per Postgres docs.

See also [PostgreSQL pages on disk](../../database/INDEX.md) for the storage-layer perspective.

### 9.2 MySQL / InnoDB

InnoDB owns its buffer pool (`innodb_buffer_pool_size`, often 50–80% of RAM). To avoid double-buffering with the page cache:

```ini
innodb_flush_method = O_DIRECT
```

This makes data file reads/writes bypass the page cache. The redo log can additionally use `O_DIRECT_NO_FSYNC` if your storage stack handles FUA properly.

InnoDB's redo log is fsync'd (or equivalent) per transaction commit by default (`innodb_flush_log_at_trx_commit=1`). Setting it to `2` writes to the page cache and only fsyncs once per second — faster but loses up to a second of committed transactions on a host crash.

### 9.3 Other engines

- **RocksDB / LevelDB** — use buffered I/O; rely on the page cache. Provide `Options::use_direct_reads` for analytic workloads.
- **MongoDB / WiredTiger** — its own cache + page cache. Tune `wiredTiger.engineConfig.cacheSizeGB`.
- **Redis** — entirely in-process; AOF/RDB persistence uses fsync at a configurable cadence.
- **Kafka** — relies entirely on the page cache for log segment caching, plus `sendfile` for delivery (see [Section 6](#6-sendfile-and-zero-copy)).

---

## 10. Backend Engineer Takeaways

- **`write()` returning success means "in the page cache".** Durability requires `fsync()`/`fdatasync()`. Code that writes critical state must call one of these — and check the return value.
- **`close()` does not fsync.** Many crash-safety bugs collapse to this.
- **fsync errors are non-recoverable.** If `fsync()` returns EIO, the only safe action is to crash and recover from a known-good state. Postgres's "fsyncgate" reaction is the right one.
- **`MemAvailable`, not `MemFree`.** Wire your alerts correctly.
- **Tune `vm.dirty_bytes` for write-heavy hosts.** Don't accept percentage-based defaults on big-RAM servers.
- **`O_DIRECT` is for engines that own their cache.** Don't reach for it in application code.
- **`sendfile`/zero-copy** is a real win for static-file servers and brokered messaging; understand whether your runtime exposes it (Java NIO `transferTo`, nginx `sendfile`, Kafka).
- **`drop_caches` is a debugging tool, never a production knob.** It will tank performance until the cache rebuilds.
- **Database tuning is page-cache tuning.** Know whether your engine wants buffered or direct I/O before sizing buffer pools.

---

## Related

- [Virtual Memory & Paging](02-virtual-memory-and-paging.md)
- [Syscalls & the ABI](05-syscalls-and-the-abi.md)
- [File Descriptors & ulimits](06-file-descriptors-and-ulimits.md)
- [Connection Pooling](../../networking/network-programming/connection-pooling.md)
- [Database INDEX](../../database/INDEX.md)

---

## References

- **`fsync(2)`, `fdatasync(2)`, `sync_file_range(2)`, `posix_fadvise(2)` man pages** — https://man7.org/linux/man-pages/man2/fsync.2.html
- **`open(2)` (`O_DIRECT` semantics)** — https://man7.org/linux/man-pages/man2/open.2.html
- **`sendfile(2)`, `splice(2)`** — https://man7.org/linux/man-pages/man2/sendfile.2.html
- **Linux kernel: filesystem documentation** — https://www.kernel.org/doc/html/latest/filesystems/index.html
- **LWN: "PostgreSQL's fsync() surprise"** — Jonathan Corbet, 2018. https://lwn.net/Articles/752063/
- **LWN: "Improving fsync() error reporting"** — https://lwn.net/Articles/724307/
- **Brendan Gregg, *Systems Performance*, 2nd ed.** — Chapter 8 (File Systems): page-cache analysis, write-back behavior, methodology.
- **Brendan Gregg's pcstat** — https://github.com/tobert/pcstat
- **vmtouch** — https://hoytech.com/vmtouch/
- **PostgreSQL: Reliability and the Write-Ahead Log** — https://www.postgresql.org/docs/current/wal-reliability.html
- **MySQL InnoDB: Configuring innodb_flush_method** — https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_flush_method
- **Kafka: filesystem and OS** — https://kafka.apache.org/documentation/#filesystems
