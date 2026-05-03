---
title: "File Descriptors & ulimits — int → struct file, open files, EMFILE"
case-sensitive: true
date: 2026-05-03
updated: 2026-05-03
tags: [operating-systems, linux, file-descriptors, ulimit, emfile, lsof, fd-leak]
---

# File Descriptors & ulimits — int → struct file, open files, EMFILE

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `operating-systems` `linux` `file-descriptors` `ulimit` `emfile` `lsof` `fd-leak`

---

## Table of Contents

- [Summary](#summary)
- [1. What an fd Actually Is](#1-what-an-fd-actually-is)
  - [1.1 The three-table model](#11-the-three-table-model)
  - [1.2 fd 0/1/2](#12-fd-012)
- [2. fds Are Per-Process; struct file Is Shared](#2-fds-are-per-process-struct-file-is-shared)
- [3. The Lifecycle of an fd](#3-the-lifecycle-of-an-fd)
- [4. Limits — soft, hard, and system-wide](#4-limits--soft-hard-and-system-wide)
  - [4.1 Per-process limits](#41-per-process-limits)
  - [4.2 System-wide limits](#42-system-wide-limits)
  - [4.3 Setting limits](#43-setting-limits)
- [5. EMFILE vs ENFILE](#5-emfile-vs-enfile)
- [6. Inspecting Open fds](#6-inspecting-open-fds)
  - [6.1 /proc/\<pid\>/fd and /proc/\<pid\>/limits](#61-procpidfd-and-procpidlimits)
  - [6.2 lsof](#62-lsof)
  - [6.3 ss (socket statistics)](#63-ss-socket-statistics)
- [7. fd Leaks — How to Find Them](#7-fd-leaks--how-to-find-them)
- [8. Common Limits to Raise for High-Connection Servers](#8-common-limits-to-raise-for-high-connection-servers)
- [9. Language-Level Notes](#9-language-level-notes)
  - [9.1 Node.js](#91-nodejs)
  - [9.2 Java](#92-java)
- [10. Backend Engineer Takeaways](#10-backend-engineer-takeaways)
- [Related](#related)
- [References](#references)

---

## Summary

A file descriptor is just a non-negative integer — but behind it sits a kernel-managed object that can be a regular file, a directory, a socket, a pipe, an `eventfd`, a `signalfd`, a memory map, or any other "thing the kernel can do I/O on." Every TCP connection, every open log file, every `epoll` instance is an fd. When your server crashes with `EMFILE: too many open files`, you've hit the per-process fd cap, which is almost always too low on a stock Linux. This document explains what's on the other side of the integer, the soft/hard/system-wide limit hierarchy, the diagnostic tools (`/proc`, `lsof`, `ss`), and the limits you actually need to raise for a server handling tens of thousands of concurrent connections.

---

## 1. What an fd Actually Is

A file descriptor is an **index** into the calling process's **file descriptor table**. The kernel maintains three layers:

```
   Process A's                  System-wide                Per-inode
   FD table                  open file table             (or socket / pipe / etc.)
   ───────────                ────────────────             ────────────────────────
   fd  → struct file *
   ───   ─────────────────
    0  → ──┐                  ┌─ struct file ─┐         ┌─ struct inode ─┐
    1  → ──┼─────────────────►│  f_op         │ ───────►│ i_mode, i_size │
    2  → ──┘                  │  f_pos        │         │ i_op           │
    3  → ───────────────────► │  f_count = 3  │         │ i_mapping (page│
    4  →                      │  f_flags      │         │   cache)       │
    …                         └───────────────┘         └────────────────┘
                              (open file description)
```

### 1.1 The three-table model

1. **fd table** (per-process) — array of pointers indexed by the integer fd. Stored in `task_struct->files->fdt`.
2. **Open file description** (system-wide) — `struct file`. Holds the file position, mode, flags, and a pointer to the file operations vtable. **Reference-counted.**
3. **Inode / socket / pipe-buffer / etc.** — the actual underlying object. Many `struct file`s can point to one inode (multiple opens of the same path), and the inode points to its page-cache mapping.

### 1.2 fd 0/1/2

By POSIX convention:
- **fd 0 = stdin**
- **fd 1 = stdout**
- **fd 2 = stderr**

A freshly `exec`'d process inherits these from its parent. Daemons that detach from the terminal usually `dup2()` `/dev/null` onto 0/1/2 so that any stray prints go to the bit bucket.

---

## 2. fds Are Per-Process; struct file Is Shared

Two situations create multiple fds pointing to the same `struct file`:

- **`dup`/`dup2`/`dup3`** — explicit duplication.
- **`fork()`** — the child inherits a copy of the fd table. Each fd in parent and child points to the **same** `struct file`.

A `struct file` has a position pointer. So:

```c
int fd = open("data", O_RDONLY);
pid_t pid = fork();
read(fd, buf, 100);   // both processes share f_pos
                      // whichever reads first advances the position for both
```

This is how shells implement pipelines: `cat foo | grep bar` opens the pipe, forks twice, and the children share the same `struct file`s for the read/write ends of the pipe.

When the **last** fd referencing a `struct file` is closed, its refcount drops to zero, the kernel calls the file_operations' `release` method, and resources (page cache pages with no other reference, socket buffers, pipe buffers) are freed.

This is also why `close(fd)` does not necessarily flush data to disk — it only decrements the refcount. To guarantee durability, call `fsync(fd)` *before* `close(fd)` (see [Page Cache & Buffered I/O](03-page-cache-and-buffered-io.md)).

---

## 3. The Lifecycle of an fd

```
   open()/socket()/pipe()/epoll_create()/eventfd()/...
                       │
                       ▼
                allocate fd (lowest free index by default)
                       │
                       ▼
                use:  read/write/recv/send/ioctl/fcntl/...
                       │
                       ▼
                close(fd)  → decrement struct file refcount
                       │
                       ▼
                if refcount hits 0:
                       │
                       ▼
                struct file released; underlying resource freed
```

Critical invariant: **every fd you create must eventually be closed**, exactly once. Closing the same fd twice is a bug (the second close may close a different fd if a new one was allocated in between — a security-relevant pattern).

The **`O_CLOEXEC`** flag (or `SOCK_CLOEXEC` for sockets, `EFD_CLOEXEC` for eventfd) marks an fd as "close on exec" — when the process calls `execve`, the kernel automatically closes it. This is critical for child-process safety: without it, your child inherits all your open log files, listening sockets, and database connections.

Modern code should always use the `*_CLOEXEC` variants (`openat(... O_CLOEXEC)`, `pipe2(..., O_CLOEXEC)`, `socket(..., SOCK_CLOEXEC, ...)`). Most language runtimes set this by default; check yours before relying on it.

---

## 4. Limits — soft, hard, and system-wide

### 4.1 Per-process limits

`getrlimit(RLIMIT_NOFILE)` controls how many fds a process may have open. Two values:

- **soft limit** — current cap; can be raised by the process up to the hard limit without privilege.
- **hard limit** — ceiling on the soft limit; only `CAP_SYS_RESOURCE` (root) can raise it.

```bash
$ ulimit -Sn   # soft
1024
$ ulimit -Hn   # hard
524288
```

The default soft limit on many Linux distros has historically been 1024 — way too low for any server. Modern distros are moving the default toward 1024 with a hard limit in the hundreds of thousands.

### 4.2 System-wide limits

```
/proc/sys/fs/file-max        ← max struct files in the system
/proc/sys/fs/file-nr         ← currently allocated / max
/proc/sys/fs/nr_open         ← upper bound on RLIMIT_NOFILE per process
```

```bash
$ cat /proc/sys/fs/file-nr
12845    0    9223372036854775807
#  ↑       ↑              ↑
# allocated free      max (here, infinite practical)
```

Modern kernels essentially uncap `file-max` (or set it to billions). The system-wide cap is not your bottleneck; per-process is.

### 4.3 Setting limits

**Via shell** (current shell + descendants):
```bash
ulimit -n 1048576           # set soft = hard = 1048576 (if hard allows)
```

**Via `/etc/security/limits.conf`** (PAM-applied at login):
```
*  soft  nofile  1048576
*  hard  nofile  1048576
```

**Via systemd unit file** (the only one that matters for managed services):
```ini
[Service]
LimitNOFILE=1048576
```

**Via Docker / Kubernetes**:
```yaml
# Docker
docker run --ulimit nofile=1048576:1048576 ...

# Kubernetes podspec - via initContainer / sysctl, depending on version
spec:
  securityContext:
    sysctls:
      - name: fs.file-max
        value: "1048576"
```

After setting, **verify**:
```bash
$ cat /proc/<pid>/limits | grep "Max open files"
Max open files            1048576              1048576              files
```

---

## 5. EMFILE vs ENFILE

Both mean "you can't open another file/socket," but they're different errors:

| Errno | Cause | Fix |
|-------|-------|-----|
| `EMFILE` | **Per-process** fd limit reached (RLIMIT_NOFILE) | Raise the per-process limit |
| `ENFILE` | **System-wide** open files limit reached | Raise `fs.file-max` (rarely necessary) |

Real-world: 99% of "too many open files" failures are EMFILE caused by an undersized RLIMIT_NOFILE plus an fd leak.

---

## 6. Inspecting Open fds

### 6.1 /proc/\<pid\>/fd and /proc/\<pid\>/limits

```bash
# All open fds (each is a symlink to its target)
$ ls -l /proc/12345/fd
lrwx------ 1 user user 64 May  3 14:00 0 -> /dev/null
lrwx------ 1 user user 64 May  3 14:00 1 -> /var/log/myapp.log
lrwx------ 1 user user 64 May  3 14:00 2 -> /var/log/myapp.log
lrwx------ 1 user user 64 May  3 14:00 3 -> 'socket:[12345678]'
lrwx------ 1 user user 64 May  3 14:00 4 -> '/tmp/cache.db (deleted)'
lrwx------ 1 user user 64 May  3 14:00 5 -> 'anon_inode:[eventpoll]'
lrwx------ 1 user user 64 May  3 14:00 6 -> 'anon_inode:[eventfd]'

# Count open fds quickly
$ ls /proc/12345/fd | wc -l
3457

# All limits, soft + hard
$ cat /proc/12345/limits
Limit                     Soft Limit           Hard Limit           Units
…
Max open files            1048576              1048576              files
…
```

`(deleted)` means the file was unlinked but the process still has it open — disk space won't be reclaimed until close. Common root cause of "df shows 99% used but du shows 50%."

### 6.2 lsof

`lsof` (LiSt Open Files) is the swiss army knife. Listing:

```bash
# All fds for one process
lsof -p 12345

# Who has /var/log/messages open?
lsof /var/log/messages

# All TCP connections from a host
lsof -i tcp@10.0.0.5

# Listening sockets
lsof -i -sTCP:LISTEN

# Files held by deleted inodes (disk-leak finder)
lsof | grep '(deleted)'
```

`lsof` can be slow on big systems because it scans every `/proc/*/fd`. For sockets specifically, `ss` is faster.

### 6.3 ss (socket statistics)

```bash
# Summary
$ ss -s
Total: 12345
TCP:   8765 (estab 5432, closed 1098, orphaned 12, timewait 1234)

Transport Total     IP        IPv6
RAW       1         0         1
UDP       7         5         2
TCP       7654      6543      1111
INET      7662      6548      1114

# Established connections to port 5432 (Postgres)
$ ss -tnp 'sport = :5432'

# All TCP connections in TIME_WAIT
$ ss -tn state time-wait | wc -l
```

`ss` reads from `/proc/net/tcp` (and netlink) — much cheaper than `lsof` for socket inspection.

---

## 7. fd Leaks — How to Find Them

Symptoms:
- Open-fd count grows monotonically over hours/days.
- Eventually `EMFILE` and the service stops accepting connections.
- `accept()` failing with `EMFILE` but not crashing — backend logs full of "too many open files."

Diagnosis:

1. **Confirm growth**: `watch -n5 'ls /proc/<pid>/fd | wc -l'` — does it climb without bound?
2. **Categorize**: `ls -l /proc/<pid>/fd | awk '{print $11}' | sort | uniq -c | sort -rn` — what kind of fd dominates?
3. **Reproduce minimally**: trigger one request type at a time, watch the count.
4. **Trace allocation**: `bpftrace -e 'tracepoint:syscalls:sys_exit_openat /pid == TARGET/ { @[ustack] = count() }'` — see the user-space stack of every successful open. Top stack frames are your leak source.

Common leak patterns:

- **Forgotten `close()`** in error paths: `if (err) return -1;` with no cleanup.
- **Long-lived HTTP clients without connection pooling** — every request opens a fresh socket and never closes (or close-on-GC depends on a finalizer that never runs).
- **Database connections held by aborted transactions**.
- **`tail -f` style file watchers** where each rotation creates a new fd and the old one is never released.

Language-level guards:

- **Java**: `try-with-resources` for any `AutoCloseable` (Streams, Connections, Sockets). Never use raw `try`/`finally` in modern code.
- **Node**: ensure `req`, `res`, and any `fs.createReadStream` go through their `'close'`/`'finish'` paths; use `pipeline()` instead of manual `pipe()`.
- **Go**: `defer file.Close()` on every successful `os.Open` / `net.Dial`. Lint with `errcheck`.

---

## 8. Common Limits to Raise for High-Connection Servers

For a server expected to handle thousands of concurrent connections:

| Limit | Default | Recommended | Purpose |
|-------|---------|-------------|---------|
| `RLIMIT_NOFILE` (soft and hard) | 1024 | **1048576** | Per-process fd cap — every connection is at least one fd |
| `fs.file-max` | very high | leave default | System-wide; rarely a bottleneck |
| `fs.nr_open` | 1048576 | 2097152 if pushing past 1M fds | Upper bound on RLIMIT_NOFILE |
| `net.core.somaxconn` | 4096 (modern) | 65535 | listen() backlog cap |
| `net.ipv4.tcp_max_syn_backlog` | 1024 | 8192+ | SYN queue size (half-open) |
| `net.ipv4.ip_local_port_range` | 32768-60999 | 1024-65535 | Ephemeral port range for outbound connections |
| `net.ipv4.tcp_tw_reuse` | 0/2 | 1 | Reuse TIME_WAIT sockets for outbound |
| `net.netfilter.nf_conntrack_max` | varies | bump if using NAT/conntrack | Connection-tracking table size |

For a 10,000-concurrent-connection HTTP server, you minimally need:
- `RLIMIT_NOFILE >= 20000` (each connection is ~2 fds — server socket + maybe a backend pool socket).
- `somaxconn` of a few thousand to absorb listen-queue spikes.

For a million-connection chat / push server, you need the full table above plus tuning of TCP buffers (`net.ipv4.tcp_rmem`/`tcp_wmem`).

See [TCP Deep Dive](../../networking/transport/tcp-deep-dive.md) Section 10.5 for the TCP-side sysctls in context.

---

## 9. Language-Level Notes

### 9.1 Node.js

- The libuv pool size (`UV_THREADPOOL_SIZE`) doesn't affect fd count directly, but a pool exhaustion can stack pending opens.
- Built-in `net.Server` connections each consume one fd plus an `epoll` registration.
- HTTP keep-alive in `undici`/`http.Agent` reuses sockets — **enable it** to avoid fd churn (see [Connection Pooling](../../networking/network-programming/connection-pooling.md)).
- Verify via Node:
  ```js
  process.getActiveResourcesInfo();          // names of currently-pending handles
  // or for raw fd count
  require('fs').readdirSync(`/proc/${process.pid}/fd`).length;
  ```

### 9.2 Java

- Sockets, FileInputStream/OutputStream, FileChannel all hold fds.
- **Always use try-with-resources.** A `Connection` from a JDBC pool that escapes a try-with-resources scope eventually starves the pool.
- Netty (`epoll` transport) holds one fd per connection plus eventfd/epoll fds per event loop.
- HTTP clients: always configure connection pooling and idle eviction (`HttpClient.newBuilder().connectTimeout(...)`, Apache HttpClient `setMaxTotal`, OkHttp `ConnectionPool`).
- Inspect: `jcmd <pid> VM.native_memory summary` shows JVM internal accounting; `lsof -p <pid>` for the kernel view.

For the Spring Boot side of this discussion: see also `server.tomcat.max-connections`, which is *not* a fd limit but caps how many simultaneous connections the embedded server will accept regardless of OS limits.

---

## 10. Backend Engineer Takeaways

- **Every connection costs at least one fd.** Plan accordingly.
- **The default `ulimit -n 1024` is wrong for any production server.** Set it to 1M as your baseline.
- **`/proc/<pid>/fd` and `/proc/<pid>/limits` answer 90% of fd questions.** Learn them before reaching for `lsof`.
- **`(deleted)` files in `lsof` output explain mysterious disk-space discrepancies.** Restart the holder or unmask the deletion.
- **Always use `*_CLOEXEC` variants** when opening fds in code that may exec.
- **Use `try-with-resources` (Java) or RAII / `defer close()` (Go) / `finally`-equivalent patterns (Node)**. Manual close is a fd-leak generator.
- **Connection pooling and HTTP keep-alive eliminate fd churn** for outbound traffic. Always enable them.
- **`ss` is faster than `lsof` for sockets.** Use it for routine TCP inspection.
- **Container fd limits are inherited from the runtime config**, not from `limits.conf` — set them via Docker/Kubernetes.
- **An fd leak is found by (1) confirming monotonic growth, (2) categorizing by target type, (3) tracing allocations with eBPF.**

---

## Related

- [Process & Thread Model](01-process-thread-model.md)
- [Syscalls & the ABI](05-syscalls-and-the-abi.md)
- [epoll / kqueue / io_uring](07-epoll-kqueue-iouring.md)
- [TCP Deep Dive](../../networking/transport/tcp-deep-dive.md)
- [Socket Programming](../../networking/transport/socket-programming.md)
- [Connection Pooling](../../networking/network-programming/connection-pooling.md)

---

## References

- **`open(2)`, `close(2)`, `dup(2)`, `fcntl(2)` man pages** — https://man7.org/linux/man-pages/man2/open.2.html
- **`getrlimit(2)`, `setrlimit(2)`, `prlimit(2)`** — https://man7.org/linux/man-pages/man2/getrlimit.2.html
- **`proc(5)` man page** — `/proc/<pid>/fd` and `/proc/<pid>/limits` semantics. https://man7.org/linux/man-pages/man5/proc.5.html
- **`lsof(8)` man page** — https://man7.org/linux/man-pages/man8/lsof.8.html
- **`ss(8)` man page** — https://man7.org/linux/man-pages/man8/ss.8.html
- **systemd: `LimitNOFILE` and resource limits** — `systemd.exec(5)`. https://www.freedesktop.org/software/systemd/man/systemd.exec.html
- **Robert Love, *Linux System Programming*, 2nd ed.** — Chapter 2 (File I/O) and Chapter 4 (Advanced File I/O) cover the fd model thoroughly.
- **Brendan Gregg, *Systems Performance*, 2nd ed.** — fd-leak diagnosis methodology in Chapter 8.
- **LWN: "Looking at the file table"** — https://lwn.net/Articles/494158/
