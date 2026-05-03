---
title: "epoll / kqueue / io_uring — readiness vs completion, level/edge triggered"
date: 2026-05-03
updated: 2026-05-03
tags: [operating-systems, linux, bsd, epoll, kqueue, io_uring, async-io, libuv, netty]
---

# epoll / kqueue / io_uring — readiness vs completion, level/edge triggered

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `operating-systems` `linux` `bsd` `epoll` `kqueue` `io_uring` `async-io` `libuv` `netty`

---

## Table of Contents

- [Summary](#summary)
- [1. Two Models: Readiness vs Completion](#1-two-models-readiness-vs-completion)
- [2. The Pre-epoll World — select and poll](#2-the-pre-epoll-world--select-and-poll)
- [3. epoll on Linux](#3-epoll-on-linux)
  - [3.1 The three syscalls](#31-the-three-syscalls)
  - [3.2 Level-triggered vs edge-triggered](#32-level-triggered-vs-edge-triggered)
  - [3.3 EPOLLONESHOT and EPOLLEXCLUSIVE](#33-epolloneshot-and-epollexclusive)
  - [3.4 Thundering herd and SO_REUSEPORT](#34-thundering-herd-and-so_reuseport)
- [4. kqueue on BSD and macOS](#4-kqueue-on-bsd-and-macos)
- [5. io_uring on Linux](#5-io_uring-on-linux)
  - [5.1 Submission and completion queues](#51-submission-and-completion-queues)
  - [5.2 What's new vs epoll](#52-whats-new-vs-epoll)
  - [5.3 Advanced features](#53-advanced-features)
  - [5.4 Caveats and current state](#54-caveats-and-current-state)
- [6. How libuv and Netty Use These](#6-how-libuv-and-netty-use-these)
  - [6.1 libuv (Node.js)](#61-libuv-nodejs)
  - [6.2 Netty (JVM)](#62-netty-jvm)
- [7. Choosing Between Them](#7-choosing-between-them)
- [8. A Minimal epoll Echo Server](#8-a-minimal-epoll-echo-server)
- [9. Performance Notes](#9-performance-notes)
- [10. Backend Engineer Takeaways](#10-backend-engineer-takeaways)
- [Related](#related)
- [References](#references)

---

## Summary

Modern servers handle tens of thousands of concurrent connections by **not** dedicating a thread per connection. Instead, they register fds with a kernel notification mechanism that wakes the application only when something is ready (or, with io_uring, when a requested operation has completed). The original mechanisms — `select`, `poll` — scaled poorly. The replacements are platform-specific: **epoll** on Linux, **kqueue** on BSD/macOS, **IOCP** on Windows, and the newer **io_uring** on Linux that switches the model from readiness to true asynchronous completion. Every modern HTTP server, message broker, and database speaks one of these. This document explains the conceptual split, the practical APIs, and how Node.js (libuv) and Java (Netty) actually use them.

---

## 1. Two Models: Readiness vs Completion

| Model | Semantics | Examples |
|-------|-----------|----------|
| **Readiness** (a.k.a. notification, polling) | Tell me **when** I can read/write without blocking. I'll then issue the read/write syscall myself. | `select`, `poll`, **epoll** (Linux), **kqueue** (BSD/macOS) |
| **Completion** | I want this read/write performed; tell me **when it's done** with the result. | Windows **IOCP**, Linux **io_uring**, Solaris event ports (sort of), POSIX AIO (badly) |

The readiness model assumes **non-blocking** fds: when the kernel says "you can read fd 7 now," your code calls `read(7, ...)` knowing it won't block. Each I/O takes at least two syscalls (the wait + the read).

The completion model lets you **submit** an operation to the kernel and **collect** results later. A single batched syscall can submit and harvest hundreds of operations — a much better fit for high-throughput storage and networking.

Both models scale; the constants differ.

---

## 2. The Pre-epoll World — select and poll

`select()` and `poll()` were the original POSIX I/O multiplexers:

- **`select(maxfd, &readfds, &writefds, &exceptfds, &timeout)`** — fixed-size bitsets (typically 1024 fds), passed and recopied on each call.
- **`poll(fds[], nfds, timeout)`** — array of `pollfd` structs, no fixed cap, but still **O(n)** per call: the kernel scans every fd, the user code does too.

Both are fine up to ~hundreds of fds. At thousands they collapse: the kernel walks the whole array on every call, the user copies the array in and out, and you get cache-misses everywhere. The "C10K" problem of the early 2000s was largely a `select`/`poll` problem.

You'll still see `select`/`poll` in:
- Cross-platform code that needs to run on systems without epoll/kqueue.
- One-off utilities with a handful of fds.

For a server, never use them.

---

## 3. epoll on Linux

`epoll` (Linux 2.6, 2002) replaces `select`/`poll` with a **stateful** kernel object. You register fds once; subsequent waits are **O(ready)** — only ready events are reported.

### 3.1 The three syscalls

```c
int epfd = epoll_create1(EPOLL_CLOEXEC);            // create the epoll instance

struct epoll_event ev;
ev.events = EPOLLIN | EPOLLET;                      // read events, edge-triggered
ev.data.fd = listen_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);     // register

struct epoll_event events[64];
int n = epoll_wait(epfd, events, 64, -1);           // block until ≥1 event
for (int i = 0; i < n; i++) {
    int fd = events[i].data.fd;
    // handle fd
}
```

The epoll instance itself is an fd (`epfd`), so you can register one epoll inside another (recursive multiplexing — useful for thread-per-loop architectures).

`epoll_pwait2` (5.11+) adds nanosecond-precision timeouts and signal masking; the original `epoll_pwait` adds signal masking only.

### 3.2 Level-triggered vs edge-triggered

Two semantics for "is fd 7 readable":

- **Level-triggered (LT, the default)** — `epoll_wait` returns fd 7 every time you call it as long as fd 7 is readable. Forgiving: you can read partially, return to the loop, and you'll be told again. Behaves like `poll`.

- **Edge-triggered (ET, with `EPOLLET`)** — `epoll_wait` returns fd 7 **only when its readability state transitions from "not ready" to "ready"** (data arrives, peer closes, etc.). You **must** drain the fd completely (read until you get `EAGAIN`) on each notification, or you'll miss data until the next state change.

ET trades correctness pressure for performance: fewer wakeups for steady-streaming connections. Most high-performance servers (nginx, HAProxy, Netty's epoll transport) use edge-triggered mode.

```c
// Edge-triggered drain pattern
while (1) {
    ssize_t n = read(fd, buf, sizeof buf);
    if (n > 0) { /* consume */ }
    else if (n < 0 && errno == EAGAIN) break;  // drained
    else if (n == 0) { /* peer closed */ break; }
    else { /* real error */ break; }
}
```

### 3.3 EPOLLONESHOT and EPOLLEXCLUSIVE

- **`EPOLLONESHOT`** — after one event is reported, the fd is **disabled** in the epoll set. The thread that handled the event must re-arm it with `EPOLL_CTL_MOD`. Useful when one fd is processed by a pool of worker threads and you want exactly one worker per event.

- **`EPOLLEXCLUSIVE`** (since 4.5) — when multiple epoll instances watch the same fd, only **one** is woken per event. Solves a thundering-herd version where N workers share an epfd.

### 3.4 Thundering herd and SO_REUSEPORT

Classic thundering herd: a listening socket is registered in N threads' epoll sets. A SYN arrives, the kernel wakes all N; all call `accept()`, only one succeeds, the rest get `EAGAIN`. CPU wasted.

Modern fixes:

- **`EPOLLEXCLUSIVE`** (above) — kernel wakes only one waiter.
- **`SO_REUSEPORT`** (since 3.9) — multiple sockets bind to the same port; the kernel **load-balances** incoming connections across them via a flow hash. Each thread has its own listening socket and its own epoll set, no contention.

`SO_REUSEPORT` is what nginx 1.9.1+ uses, what HAProxy uses, what Envoy uses. It's the standard pattern for multi-thread accept loops on Linux today.

---

## 4. kqueue on BSD and macOS

`kqueue` (FreeBSD 4.1, 2000) is older than epoll and arguably more elegant. It's a single mechanism for all "kernel events" — fd I/O, timers, signals, process exit, file modification, even custom user events.

```c
int kq = kqueue();

struct kevent ev;
EV_SET(&ev, listen_fd, EVFILT_READ, EV_ADD | EV_CLEAR, 0, 0, NULL);
//              ^         ^                ^^^^^^^
//              fd       filter         flags (EV_CLEAR ≈ edge-triggered)
kevent(kq, &ev, 1, NULL, 0, NULL);    // register

struct kevent events[64];
int n = kevent(kq, NULL, 0, events, 64, NULL);   // wait
```

Filters include `EVFILT_READ`, `EVFILT_WRITE`, `EVFILT_VNODE` (file changes), `EVFILT_PROC` (process state), `EVFILT_SIGNAL`, `EVFILT_TIMER`, `EVFILT_USER`. This unification means a server's main loop can also watch for child-process exit and config-file changes through the same mechanism.

`kqueue` is what macOS, FreeBSD, OpenBSD, and NetBSD use. Cross-platform runtimes (libuv, Tokio, Netty's KQueue transport) abstract over kqueue/epoll with similar semantics.

---

## 5. io_uring on Linux

`io_uring` (Linux 5.1, 2019) is Jens Axboe's redesign of asynchronous I/O on Linux. Unlike epoll, it follows the **completion** model — and unlike POSIX AIO (which never worked well outside of `O_DIRECT` files), io_uring works for any fd.

### 5.1 Submission and completion queues

io_uring uses **two ring buffers shared between user space and kernel**:

```
   user space                      kernel
   ──────────                      ──────
                                   ┌──────────────┐
                                   │ kernel polls │
                                   │   the SQ for │
                                   │ new SQEs     │
                                   └──────────────┘
   ┌─────────────────┐                     │
   │ Submission Queue│  ◄──────────────────┘
   │       (SQ)      │
   │ [SQE][SQE][SQE] │  ──── shared mmap'd memory ────
   └─────────────────┘
   user pushes SQEs (Submission Queue Entries)
   describing operations: read, write, recv, send,
   accept, openat, fsync, splice, …

   ┌─────────────────┐
   │ Completion Queue│  ──── shared mmap'd memory ────
   │       (CQ)      │
   │ [CQE][CQE]      │  ◄────── kernel posts CQEs
   └─────────────────┘
   user reads CQEs (Completion Queue Entries) when
   operations finish; each carries a user-supplied
   tag and the result (or -errno).
```

Three syscalls:
- `io_uring_setup(entries, params)` — create the rings.
- `io_uring_enter(fd, to_submit, min_complete, flags, sig)` — kick the kernel; can submit, can wait, or both.
- `io_uring_register(fd, opcode, ...)` — pre-register fds, buffers, files for zero-overhead ops.

In practice you use **liburing**, which provides a friendly C API and avoids dealing with raw rings.

### 5.2 What's new vs epoll

| Aspect | epoll | io_uring |
|--------|-------|----------|
| Model | Readiness | Completion |
| Per-op syscalls | 2 (wait + read) | Often 0 (with SQPOLL kernel-side polling) |
| Batching | Manual (read in a loop after epoll_wait) | Kernel-batched naturally |
| Storage I/O | Doesn't help (still blocks on disk) | Truly async — disk I/O without blocking caller |
| Cancellation | None per-op | `IORING_OP_ASYNC_CANCEL` |
| Fixed-buffer / fixed-fd registration | None | `io_uring_register` for zero-copy hot paths |
| Linked operations | None | `IOSQE_IO_LINK` (e.g., accept → recv → send chain) |

For storage workloads, this matters enormously. epoll on a regular file just runs the read synchronously (the kernel will block your thread on disk I/O); io_uring really does it in the background.

### 5.3 Advanced features

- **SQPOLL** — kernel polls the submission queue from a kernel thread. Userspace can submit *zero syscalls per op* in steady state. Netty 5+ exposes this; ScyllaDB uses it heavily.
- **IORING_OP_ACCEPT, OP_RECV, OP_SEND** — full network ops, not just file I/O.
- **MSG_RING** — pass CQEs between rings (cheap inter-thread signaling).
- **Fixed buffers** (`io_uring_register_buffers`) — avoid `copy_from_user` on every op.
- **Multishot** — one SQE produces many CQEs (e.g., one accept SQE keeps producing accepts).
- **Network zero-copy send** (`IORING_OP_SEND_ZC`).

### 5.4 Caveats and current state

- **Security**: io_uring's surface area exposed several vulnerabilities (CVE-2022-1116, CVE-2022-29582, CVE-2023-0179, etc.). Google disabled it for unprivileged users on ChromeOS and in some sandboxes; some container runtimes restrict it via seccomp.
- **Kernel age**: io_uring grew rapidly. A feature added in 5.13 won't work on 5.4 (RHEL 8). Production code must check feature availability.
- **Library maturity**: liburing is the only reasonable C interface. Java (`io_uring`-aware Netty), Rust (tokio-uring, glommio), and others are still evolving.
- **Programming model**: substantially more complex than epoll. Not worth it unless you're storage-IOPS-bound or chasing single-digit-microsecond latencies.

For LWN's authoritative writeups and benchmark figures, see lwn.net/Articles/776703/ ("Ringing in a new asynchronous I/O API") and lwn.net/Articles/810414/ ("The rapid growth of io_uring").

---

## 6. How libuv and Netty Use These

### 6.1 libuv (Node.js)

libuv is the cross-platform event-loop library underneath Node.js. Its design:

- One **event loop** per Node thread (single, by default).
- A **backend** abstraction over the OS notification mechanism:
  - Linux → `epoll` (level-triggered, with some edge-triggered specifics).
  - macOS / BSD → `kqueue`.
  - Windows → IOCP (a completion-model API).
  - Solaris → event ports.
- A **thread pool** (default 4 threads, `UV_THREADPOOL_SIZE`) for operations that lack good async kernel APIs: file system calls, DNS via `getaddrinfo`, crypto, zlib.

A Node.js HTTP request lifecycle:

```
1. epoll_wait wakes on the listening socket (EPOLLIN).
2. accept4() returns a new connected socket (CLOEXEC + nonblocking).
3. The new fd is registered with epoll.
4. epoll_wait wakes again when bytes arrive; libuv reads, hands buffers to JS as 'data' events.
5. App calls res.end(...); libuv writes (may register EPOLLOUT if the kernel buffer fills).
6. close() when the response is done (or kept alive for the next request).
```

io_uring support in libuv started landing in 1.45+ (file ops only); not yet the default for sockets.

See [Async I/O Models](../../networking/network-programming/async-io-models.md) for the broader pattern.

### 6.2 Netty (JVM)

Netty offers three transports:

- **NIO** — uses `java.nio.channels.Selector`, which on Linux uses epoll under the hood through the JDK. Cross-platform, the safe default.
- **Epoll (native)** — Netty's own JNI binding to epoll. Edge-triggered, supports `SO_REUSEPORT`, `TCP_FASTOPEN`, `TCP_QUICKACK`, and other Linux-only options. Lower latency than NIO. Linux-only.
- **KQueue (native)** — same idea for BSD/macOS.
- **IO_URING** (incubating in netty-incubator-transport-io_uring) — completion-model transport. Production use is still niche.

Netty's event loop pattern: one event loop per OS thread (typically `2 * cores`), each loop owns its own epoll fd, accept work distributed via `SO_REUSEPORT` or by having a single boss loop hand off to worker loops. This is the design behind Spring WebFlux's reactor-netty and many other JVM async frameworks.

For why Netty matters in the JVM ecosystem and how it interacts with virtual threads, see [Reactor Netty DNS Resolver](../../java/webclient-netty-dns-resolver-fix.md) and [Reactor Schedulers and Threading](../../java/reactive/schedulers-and-threading.md).

---

## 7. Choosing Between Them

| Workload | Recommended |
|----------|-------------|
| Cross-platform server in 2026 | **epoll** on Linux, **kqueue** on BSD/macOS — through your runtime's abstraction |
| Linux-only, latency-critical | **epoll edge-triggered + SO_REUSEPORT**, multi-loop |
| Linux-only, storage-IOPS heavy | **io_uring** — the win is large for direct-IO databases |
| Linux-only, tens of millions of ops/sec | **io_uring with SQPOLL** — single-digit microseconds achievable |
| Embedded / single-handful of fds | **poll** is fine; don't over-engineer |
| You don't write the loop yourself | Trust your runtime (libuv, Netty, Tokio) |

Don't reach for io_uring just because it's new. The complexity is real, the kernel-version matrix is annoying, and for typical request/response services the throughput is already saturated by application code or upstream dependencies, not by the I/O syscall layer.

---

## 8. A Minimal epoll Echo Server

```c
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

#define MAX_EVENTS 64

static int set_nonblock(int fd) {
    int f = fcntl(fd, F_GETFL, 0);
    return fcntl(fd, F_SETFL, f | O_NONBLOCK);
}

int main(void) {
    int lfd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0);
    int one = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &one, sizeof one);

    struct sockaddr_in addr = { .sin_family = AF_INET, .sin_port = htons(8080),
                                .sin_addr.s_addr = htonl(INADDR_ANY) };
    bind(lfd, (struct sockaddr *)&addr, sizeof addr);
    listen(lfd, 1024);

    int epfd = epoll_create1(EPOLL_CLOEXEC);
    struct epoll_event ev = { .events = EPOLLIN, .data.fd = lfd };
    epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);

    struct epoll_event events[MAX_EVENTS];
    char buf[4096];

    for (;;) {
        int n = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (int i = 0; i < n; i++) {
            int fd = events[i].data.fd;
            if (fd == lfd) {
                // accept loop — drain the listen queue
                for (;;) {
                    int cfd = accept4(lfd, NULL, NULL, SOCK_NONBLOCK | SOCK_CLOEXEC);
                    if (cfd < 0) break;            // EAGAIN — drained
                    struct epoll_event cev = {
                        .events = EPOLLIN | EPOLLET | EPOLLRDHUP,
                        .data.fd = cfd
                    };
                    epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &cev);
                }
            } else {
                // edge-triggered drain
                for (;;) {
                    ssize_t r = read(fd, buf, sizeof buf);
                    if (r > 0) write(fd, buf, r);  // echo
                    else if (r == 0) { close(fd); break; }   // peer closed
                    else { if (errno != EAGAIN) close(fd); break; }
                }
            }
        }
    }
}
```

This is the skeleton inside every nginx, HAProxy, and Netty event loop, give or take TLS, HTTP parsing, SO_REUSEPORT, and several thousand lines of edge-case handling.

---

## 9. Performance Notes

The honest truth about benchmarks:

- **`select`/`poll` vs `epoll`** — for thousands of fds, epoll is dramatically faster. This was settled 20 years ago.
- **`epoll` LT vs ET** — ET shaves wakeups; the win depends on traffic shape. Steady streaming wins more; bursty request/response wins less.
- **`epoll` vs `io_uring` for network** — for typical request/response servers, the difference is single-digit percent. The win is in storage-heavy workloads.
- **`io_uring` with SQPOLL** — *can* achieve syscall-free steady-state, with throughput numbers documented in Jens Axboe's talks and the LWN io_uring articles. Real numbers depend hugely on the workload.

For concrete published benchmarks see:
- LWN's *"Ringing in a new asynchronous I/O API"* and follow-ups: https://lwn.net/Articles/776703/
- Jens Axboe's slides from Linux Plumbers / Kernel Recipes (search "io_uring Jens Axboe slides" for the year).
- ScyllaDB's blog on io_uring in production (they were early adopters).
- The TechEmpower web framework benchmarks, where Linux-native epoll/io_uring stacks consistently top the throughput charts.

Numbers from any blog post are workload-specific; reproduce on your own hardware before quoting.

---

## 10. Backend Engineer Takeaways

- **Readiness (epoll/kqueue) vs completion (io_uring/IOCP)** is the foundational distinction; pick the model that matches your workload.
- **For Linux servers, epoll is the default**; reach for io_uring when you're storage-IOPS-bound or chasing microsecond latencies and have the kernel version to support it.
- **Edge-triggered epoll requires draining**. If you forget to drain, you lose events until the next state transition.
- **`SO_REUSEPORT` solves the multi-thread accept thundering herd** and is the standard pattern in nginx/HAProxy/Envoy.
- **You almost certainly do not write your own event loop.** Use libuv, Netty, Tokio, asio. They've already debugged the corner cases.
- **`io_uring` is powerful but immature for general use.** Pin a kernel version, audit the security advisories, and benchmark before adopting.
- **Cross-platform abstractions hide the differences** — Java NIO Selector, libuv handle, Tokio runtime — but understanding the underlying mechanism explains the failure modes.
- **Storage I/O is fundamentally different**: epoll doesn't actually make file reads asynchronous; only io_uring (or POSIX AIO with caveats) does.
- **For networking, the bottleneck is rarely the multiplexing API** — TLS, HTTP parsing, and application logic dominate. Profile before optimizing.

---

## Related

- [Process & Thread Model](01-process-thread-model.md)
- [Syscalls & the ABI](05-syscalls-and-the-abi.md)
- [File Descriptors & ulimits](06-file-descriptors-and-ulimits.md)
- [Async I/O Models](../../networking/network-programming/async-io-models.md)
- [Socket Programming](../../networking/transport/socket-programming.md)
- [Connection Pooling](../../networking/network-programming/connection-pooling.md)

---

## References

- **`epoll(7)`, `epoll_create1(2)`, `epoll_ctl(2)`, `epoll_wait(2)` man pages** — https://man7.org/linux/man-pages/man7/epoll.7.html
- **`kqueue(2)` (FreeBSD)** — https://man.freebsd.org/cgi/man.cgi?query=kqueue
- **`io_uring(7)` man page** — https://man7.org/linux/man-pages/man7/io_uring.7.html
- **liburing** — Jens Axboe's library and examples. https://github.com/axboe/liburing
- **Jens Axboe, "Efficient IO with io_uring"** — design paper. https://kernel.dk/io_uring.pdf
- **LWN: "Ringing in a new asynchronous I/O API"** — Jonathan Corbet, Jan 2019. https://lwn.net/Articles/776703/
- **LWN: "The rapid growth of io_uring"** — May 2020. https://lwn.net/Articles/810414/
- **LWN: "An introduction to io_uring"** — series. https://lwn.net/Articles/776703/
- **Dan Kegel, "The C10K problem"** — historical context for why select/poll didn't scale. http://www.kegel.com/c10k.html
- **libuv design overview** — http://docs.libuv.org/en/v1.x/design.html
- **Netty epoll transport** — https://netty.io/wiki/native-transports.html
- **nginx, "Socket Sharding in NGINX Release 1.9.1"** — SO_REUSEPORT. https://www.f5.com/company/blog/nginx/socket-sharding-nginx-release-1-9-1
- **Linus Torvalds on edge vs level triggered epoll** — https://lkml.org/lkml/2019/10/30/684 (and related threads)
- **Brendan Gregg, *Systems Performance*, 2nd ed.** — Chapter 10 (Network) covers the syscall-level I/O multiplexing options.
