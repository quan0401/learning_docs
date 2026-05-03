---
title: "Syscalls & the ABI — int 0x80 → syscall, vDSO, strace, eBPF"
date: 2026-05-03
updated: 2026-05-03
tags: [operating-systems, linux, syscall, abi, vdso, strace, ebpf, seccomp]
---

# Syscalls & the ABI — int 0x80 → syscall, vDSO, strace, eBPF

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `operating-systems` `linux` `syscall` `abi` `vdso` `strace` `ebpf` `seccomp`

---

## Table of Contents

- [Summary](#summary)
- [1. What a Syscall Is](#1-what-a-syscall-is)
- [2. The Syscall Mechanism](#2-the-syscall-mechanism)
  - [2.1 int 0x80 (legacy)](#21-int-0x80-legacy)
  - [2.2 sysenter / syscall](#22-sysenter--syscall)
  - [2.3 The x86_64 Linux ABI](#23-the-x86_64-linux-abi)
- [3. The vDSO](#3-the-vdso)
- [4. ABI vs API](#4-abi-vs-api)
- [5. errno and Return Conventions](#5-errno-and-return-conventions)
- [6. The glibc Wrapper Layer](#6-the-glibc-wrapper-layer)
- [7. Syscall Cost](#7-syscall-cost)
- [8. strace — Watching the Boundary](#8-strace--watching-the-boundary)
  - [8.1 Anatomy of strace output](#81-anatomy-of-strace-output)
  - [8.2 strace on a simple HTTP server](#82-strace-on-a-simple-http-server)
- [9. seccomp — Filtering Syscalls](#9-seccomp--filtering-syscalls)
- [10. eBPF — Extending the Kernel from User Space](#10-ebpf--extending-the-kernel-from-user-space)
- [11. Backend Engineer Takeaways](#11-backend-engineer-takeaways)
- [Related](#related)
- [References](#references)

---

## Summary

A **system call** is the only legitimate way for user code to ask the kernel to do something privileged: open a file, send bytes on a socket, fork a process, map memory. On modern Linux/x86_64, a syscall is a single `syscall` instruction that traps to ring 0, dispatches through a per-arch table, and returns. The cost is low (hundreds of nanoseconds) but not free, and on a server doing millions of `read()`/`write()`/`epoll_wait()` calls per second it adds up to real CPU. This document covers how syscalls actually work, the ABI that makes them callable from any language, the vDSO that elides them entirely for cheap operations like `gettimeofday`, the tools to observe them (`strace`, `perf`, `ltrace`), and the modern alternatives — `seccomp` to restrict, `eBPF` to extend.

---

## 1. What a Syscall Is

User code runs in CPU **ring 3** (unprivileged); the kernel runs in **ring 0** (privileged). Ring 3 cannot directly access hardware, switch page tables, or read kernel memory. Anything privileged requires a controlled transition into the kernel — a syscall.

```
   User space (ring 3)            Kernel space (ring 0)
   ─────────────────              ──────────────────────
   write(fd, buf, n)
        │
        │ syscall instruction
        │ rax = SYS_write
        │ rdi = fd, rsi = buf, rdx = n
        │
        │  ─── ring 3 → ring 0 ──►   sys_write()
        │                            ├─ vfs_write()
        │                            ├─ generic_file_write_iter()
        │                            └─ writeback to page cache
        │
        │  ◄── ring 0 → ring 3 ───
        │
        │ rax = number of bytes written, or -errno
        ▼
   resume after syscall
```

The kernel never trusts user inputs: every pointer is validated (`copy_from_user`, `copy_to_user` check that the address is within the user's address space and not faulting), every `fd` is looked up in the file descriptor table, every length checked.

---

## 2. The Syscall Mechanism

### 2.1 int 0x80 (legacy)

Pre-Pentium, x86 Linux used `int 0x80` — a software interrupt. The CPU saved registers, switched to ring 0, and dispatched through the IDT to the syscall handler. Slow (hundreds of cycles) because of the heavyweight interrupt mechanism.

### 2.2 sysenter / syscall

Modern x86 introduced fast path instructions:

- **`sysenter`** (Intel, 32-bit) — fast ring transition, but inflexible.
- **`syscall`** (AMD, adopted everywhere on x86_64) — the standard mechanism on 64-bit Linux. Sets up RIP, RSP, and segment registers from MSRs (`MSR_LSTAR`, etc.) in tens of cycles instead of hundreds.

When you compile a glibc-using program on x86_64 today, every syscall site lowers to a `syscall` instruction. ARM64 uses `svc #0`.

### 2.3 The x86_64 Linux ABI

Each architecture has a defined **system call ABI** — which registers carry which arguments, where the return value goes, which registers are clobbered:

| Register | Role on x86_64 syscall |
|----------|------------------------|
| `rax` | Syscall number on entry; return value on exit |
| `rdi` | arg1 |
| `rsi` | arg2 |
| `rdx` | arg3 |
| `r10` | arg4 (note: not `rcx` — `syscall` uses `rcx` for the saved RIP) |
| `r8` | arg5 |
| `r9` | arg6 |
| `rcx`, `r11` | Clobbered by `syscall` (saves RIP and RFLAGS) |

This is **different** from the x86_64 System V function-call ABI (which uses `rcx` instead of `r10`). Glibc's syscall wrappers do the rearrangement.

Syscall numbers are stable within a release line and architecture (see `<asm/unistd_64.h>` or the table at https://filippo.io/linux-syscall-table/). They differ across architectures — `read` is 0 on x86_64 but 63 on aarch64.

---

## 3. The vDSO

The **vDSO** (virtual dynamic shared object) is a small shared library that the kernel **maps into every process's address space** at exec time. You can see it in `/proc/<pid>/maps`:

```
7ffd5c974000-7ffd5c976000 r-xp 00000000 00:00 0  [vdso]
```

The vDSO implements a handful of syscalls **entirely in user space** by exposing kernel-maintained data:

- `clock_gettime`, `gettimeofday`, `time` — read clock data the kernel updates in a vvar page.
- `getcpu` — read the CPU number from a TLS or vvar slot.

Why? These are hot — a JVM, Go runtime, or Node process may call `clock_gettime` thousands of times per second for timestamps and timing. Doing them via real syscalls would burn meaningful CPU. The vDSO turns them into ordinary function calls that take ~10 ns instead of ~100–300 ns.

You don't call the vDSO directly — glibc resolves `clock_gettime` to the vDSO version via a special ELF symbol (`__vdso_clock_gettime`).

---

## 4. ABI vs API

| Term | Defined by | Stability | Examples |
|------|-----------|-----------|----------|
| **API** (Application Programming Interface) | Source-level — function signatures, header files, language semantics | Compiler-time | `int read(int, void *, size_t)`; the POSIX C API |
| **ABI** (Application Binary Interface) | Binary-level — register/stack layout, calling conventions, struct padding, syscall numbers, error conventions | Runtime; must hold across compiler versions | x86_64 System V calling convention; the Linux syscall ABI |

Linus Torvalds's "**we don't break userspace**" rule applies to the **ABI**: a binary built against a 1995 kernel header should still run on Linux 6.x. The kernel never reuses or renumbers a syscall, never changes a struct field's offset, never returns a different errno for the same input. This is why the glibc/musl/Go-runtime/Java-JNI stack just works across kernel versions.

---

## 5. errno and Return Conventions

Linux syscalls return:

- A non-negative value on success (often the result, like bytes read).
- A negative value `-E<errno>` on failure (e.g., `-2` = `-ENOENT`).

**glibc translates** the negative value into setting `errno` (a thread-local variable) and returning `-1`. Without glibc (assembly or `syscall(2)` direct invocation), you read the negative value yourself.

```c
ssize_t n = read(fd, buf, count);
if (n < 0) {
    perror("read");      // uses errno
    return errno;
}
```

Common errnos for backend devs:

| errno | Meaning |
|-------|---------|
| `EAGAIN` / `EWOULDBLOCK` | Non-blocking I/O would have blocked; retry later |
| `EINTR` | Syscall interrupted by signal; commonly need to retry |
| `EBADF` | Bad file descriptor (use-after-close) |
| `EPIPE` | Wrote to a closed pipe/socket; SIGPIPE delivered |
| `ECONNRESET` | TCP RST received |
| `ENOMEM` | Out of memory |
| `EMFILE` | Per-process fd limit hit |
| `ENFILE` | System-wide fd limit hit |
| `ENOSPC` | Disk full |
| `EFAULT` | Bad pointer (kernel couldn't `copy_from_user`) |

Always check return values; never ignore them. Most "mysterious silent failures" are unchecked syscall errors.

---

## 6. The glibc Wrapper Layer

You don't usually call `syscall()` directly. Instead glibc (or musl) provides C wrappers that:

1. Marshal arguments into the syscall ABI registers.
2. Issue the `syscall` instruction.
3. Translate the return into POSIX error semantics (errno + -1).
4. Sometimes do **more** — e.g., `fork()` runs `pthread_atfork` handlers; `system()` does setup/teardown around `execve()`.
5. Sometimes call **multiple** kernel calls — `getaddrinfo()` runs DNS lookups, NSS lookups, etc., none of which is one syscall.

When you `strace` a program, you see the *kernel* level — `clone3`, `openat`, `read` — not the libc functions. A single library call may produce many syscalls.

For languages that don't link glibc (Go's runtime, mostly-static binaries with musl, kernel-direct languages) the same translation happens in their runtime.

---

## 7. Syscall Cost

A modern x86_64 `syscall` instruction round-trip to a do-nothing kernel function (`getppid`) is on the order of **150–300 ns**. The cost is dominated by:

- Mode switch (saving/restoring registers, swapping kernel stack).
- **Spectre/Meltdown mitigations** — KPTI (kernel page-table isolation) doubles the cost on affected CPUs because the entire page-table base must change on every syscall.
- L1/L2 cache pollution from kernel code.

For perspective:

- A function call: ~1 ns.
- vDSO `clock_gettime`: ~10 ns.
- A syscall that does almost nothing: ~150–300 ns.
- A syscall that does real work (`read` from page cache hit): ~500 ns – 1 µs.
- A syscall that blocks on disk I/O: 0.1 – 10 ms.

Implications:

- A loop calling `read()` for one byte at a time is catastrophic — buffer your reads (`fgetc` and `BufferedReader` exist for this reason).
- **`vmsplice`/`splice`/`sendfile` save copies AND syscalls** for high-throughput data movement.
- **`io_uring` batches** many operations per syscall — see [epoll / kqueue / io_uring](07-epoll-kqueue-iouring.md).

You can measure your service's syscall rate:

```bash
# System-wide syscall rate
$ pidstat -d 1 | tail
$ perf stat -e raw_syscalls:sys_enter -a sleep 5

# Per-process top syscalls
$ strace -c -p <pid> -f          # then Ctrl-C; produces a sorted summary
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 45.32    0.123456          12     10000           epoll_wait
 32.10    0.087654           3     30000           read
 …
```

A backend service spending >30% of its CPU in syscalls usually has a buffering or batching opportunity.

---

## 8. strace — Watching the Boundary

`strace` attaches with `ptrace`, intercepts every syscall, decodes arguments, and prints them. It's the most useful single tool for "what is this process actually doing?"

### 8.1 Anatomy of strace output

```
$ strace -e openat,read,write,connect curl -s http://example.com/ > /dev/null
…
openat(AT_FDCWD, "/etc/resolv.conf", O_RDONLY|O_CLOEXEC) = 3
read(3, "nameserver 1.1.1.1\n", 4096)   = 19
…
connect(5, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("93.184.216.34")}, 16) = 0
write(5, "GET / HTTP/1.1\r\nHost: example.com"..., 75) = 75
read(5, "HTTP/1.1 200 OK\r\nContent-Type: t"..., 16384) = 1256
…
```

Useful flags:

| Flag | Effect |
|------|--------|
| `-p <pid>` | Attach to running process (requires ptrace permissions) |
| `-f` | Follow forks/threads |
| `-e <set>` | Trace only listed syscalls (e.g., `-e network`, `-e file`, `-e openat,read,write`) |
| `-c` | Summary mode — no per-call output, just totals |
| `-s 256` | Print up to 256 bytes of string args (default 32) |
| `-y` | Decode fd numbers to file/socket names |
| `-T` | Show time spent in each syscall |
| `-tt` | Show wall-clock timestamps |
| `-o file` | Output to file |
| `-k` | Print user-space stack trace per syscall (Linux 5.13+) |

`strace` slows the process significantly because of the ptrace round-trips. For low-overhead production tracing, prefer `bpftrace` / `perf trace` (which use eBPF or ftrace and don't ptrace-stop the target).

### 8.2 strace on a simple HTTP server

A minimal Node.js HTTP server:

```javascript
const http = require('http');
http.createServer((req, res) => {
  res.end('hello\n');
}).listen(3000);
```

A single request, traced (abbreviated):

```
$ strace -e network,read,write,close -p <node-pid> -f
[pid 1234] accept4(15, {sa_family=AF_INET, sin_port=htons(54321), sin_addr=inet_addr("127.0.0.1")}, [16], SOCK_CLOEXEC|SOCK_NONBLOCK) = 17
[pid 1234] epoll_ctl(13, EPOLL_CTL_ADD, 17, {events=EPOLLIN|EPOLLRDHUP, ...}) = 0
[pid 1234] read(17, "GET / HTTP/1.1\r\nHost: localhost"..., 65536) = 87
[pid 1234] write(17, "HTTP/1.1 200 OK\r\nContent-Type: t"..., 137) = 137
[pid 1234] epoll_ctl(13, EPOLL_CTL_DEL, 17, NULL) = 0
[pid 1234] close(17) = 0
```

You can read the entire request lifecycle: accept, register with epoll, read the HTTP request, write the response, deregister, close. This is what every HTTP server does, regardless of language. See [Socket Programming](../../networking/transport/socket-programming.md) and [epoll / kqueue / io_uring](07-epoll-kqueue-iouring.md).

---

## 9. seccomp — Filtering Syscalls

`seccomp` (secure computing mode) filters which syscalls a process can make, killing or signaling on disallowed calls. Two modes:

- **`SECCOMP_MODE_STRICT`** — only `read`, `write`, `_exit`, `sigreturn`. Useful for sandboxed compute (Chromium's PNaCl historically).
- **`SECCOMP_MODE_FILTER` (seccomp-bpf)** — install a BPF program that examines syscall number + arguments and returns `ALLOW`, `KILL`, `ERRNO`, `TRACE`, etc.

Container runtimes ship default seccomp profiles:

- Docker default profile blocks ~50 dangerous syscalls (`mount`, `kexec_load`, `reboot`, `clock_settime`, etc.) by default.
- Kubernetes can apply per-pod profiles via `securityContext.seccompProfile`.

You can dump the live filter for a pid:

```
# Requires CAP_SYS_PTRACE
$ ls /proc/<pid>/seccomp           # mode (0/1/2)
```

Writing seccomp filters by hand is painful; libraries like `libseccomp` or runtime support (Go's `seccomp-golang`) make it tractable.

For backend services, the default container runtime profile is usually fine. Custom profiles matter when:

- Building a multi-tenant code execution sandbox (Cloud Run, AWS Lambda, online judges).
- Hardening a known-narrow workload (a static-file server doesn't need `clone3`, `ptrace`, `mount`, etc.).

---

## 10. eBPF — Extending the Kernel from User Space

eBPF is **the** transformative kernel feature of the last decade. It lets you load verified user-supplied programs into the kernel that run on hooks (kprobes, uprobes, tracepoints, network filters, scheduler hooks) without modifying the kernel.

For backend developers, eBPF replaces strace/ptrace for production observability:

- **`bpftrace`** — awk-like one-liners that compile to eBPF. Brendan Gregg's tool.

  ```bash
  # Top 10 syscalls system-wide for 10 seconds
  bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[ksym(args->sysno)] = count(); }'

  # Latency of read() syscalls per pid
  bpftrace -e '
      tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }
      tracepoint:syscalls:sys_exit_read /@start[tid]/ {
          @ms[pid] = hist((nsecs - @start[tid]) / 1000000);
          delete(@start[tid]);
      }'
  ```

- **`bcc`** — Python/C frontend with prepackaged tools (`opensnoop`, `execsnoop`, `tcplife`, `ext4slower`, `biolatency`).
- **`Cilium`** — networking/security/load-balancing using eBPF instead of iptables.
- **`Pixie`, `Falco`, `Tracee`** — observability/security platforms built on eBPF.

eBPF programs are JIT-compiled to native code, run in-kernel with strict verification (no unbounded loops, no out-of-bounds memory access). The overhead is typically a few percent for production-scale tracing, vs the multi-x slowdown of strace.

For the network layer, see [Network Observability](../../networking/advanced/network-observability.md).

---

## 11. Backend Engineer Takeaways

- **Every blocking I/O is a syscall.** Buffer aggressively when doing many small reads/writes.
- **vDSO calls (`clock_gettime`) are free; treat them as ordinary function calls.** Don't optimize timestamps further.
- **Errno is the answer** when something fails. Always check return values; always log errno (or `os.strerror` / Java exception messages).
- **`strace -c -p <pid>`** is the fastest way to diagnose "what is this server doing" — even on production for short windows.
- **For production tracing, use eBPF tools** (`bpftrace`, `bcc`, `perf trace`) instead of strace.
- **Don't roll your own syscalls.** Use the runtime's wrappers; they handle EINTR, errno, and signal interactions correctly.
- **The Linux syscall ABI is forever stable** — your binary built today will run on a 2035 kernel.
- **Default seccomp profiles in container runtimes are good security hygiene**; don't relax them without knowing what you're allowing.
- **`io_uring` collapses many syscalls into one** — see the [epoll / kqueue / io_uring](07-epoll-kqueue-iouring.md) doc for when it pays off.

---

## Related

- [Process & Thread Model](01-process-thread-model.md)
- [File Descriptors & ulimits](06-file-descriptors-and-ulimits.md)
- [epoll / kqueue / io_uring](07-epoll-kqueue-iouring.md)
- [Socket Programming](../../networking/transport/socket-programming.md)
- [Network Observability (eBPF)](../../networking/advanced/network-observability.md)

---

## References

- **`syscalls(2)` man page** — list of Linux syscalls. https://man7.org/linux/man-pages/man2/syscalls.2.html
- **`syscall(2)` man page** — direct invocation. https://man7.org/linux/man-pages/man2/syscall.2.html
- **`vdso(7)` man page** — vDSO mechanics. https://man7.org/linux/man-pages/man7/vdso.7.html
- **`errno(3)` man page** — error number list and semantics. https://man7.org/linux/man-pages/man3/errno.3.html
- **`strace(1)` man page** — https://man7.org/linux/man-pages/man1/strace.1.html
- **System V AMD64 ABI** — https://gitlab.com/x86-psABIs/x86-64-ABI
- **Linux syscall table (cross-architecture)** — Filippo Valsorda's reference: https://filippo.io/linux-syscall-table/
- **Linux kernel: seccomp documentation** — https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html
- **Linux kernel: BPF documentation** — https://www.kernel.org/doc/html/latest/bpf/index.html
- **Brendan Gregg, *BPF Performance Tools*** (Addison-Wesley, 2019). The reference for eBPF observability. https://www.brendangregg.com/bpf-performance-tools-book.html
- **Brendan Gregg, *Systems Performance*, 2nd ed.** — Chapter 5 (Applications): syscall analysis methodology.
- **LWN: "A seccomp overview"** — https://lwn.net/Articles/656307/
- **Docker default seccomp profile** — https://docs.docker.com/engine/security/seccomp/
