---
title: "TCP — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, tcp, transport, sockets, networking]
---

# TCP — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `tcp` `transport` `sockets` `networking`

---

## Table of Contents
1. [How to use this doc](#how-to-use-this-doc)
2. [Easy (warm-up traps)](#1-easy-warm-up-traps)
3. [Subtle (review-passers)](#2-subtle-review-passers)
4. [Senior trap (production-only failures)](#3-senior-trap-production-only-failures)
5. [Solutions](#4-solutions)
6. [Related](#related)
7. [References](#references)

## Summary
Active-recall practice for TCP timeouts and connection lifecycle. The 22 traps span TIME_WAIT exhaustion, Nagle vs. delayed ACK interaction, RST-on-close vs. graceful FIN, half-open detection, keepalive misconfiguration, ephemeral-port and accept-backlog limits, retransmission and slow-start surprises, partial reads/writes, path-MTU black holes, and BSD-vs-Linux semantic divergence. Examples use Linux semantics with C/POSIX socket calls and shell sysctls; BSD differences are called out where they matter. Each bug cites RFC 9293 / 7414 / 6298 / 5681 / 1122 / 7323, the Linux `tcp(7)` man page, kernel.org networking docs, or a recognized authority (Cloudflare, Vincent Bernat, Stevens). Suggested cadence: one section per sitting; revisit after every postmortem reading.

## How to use this doc
- Try to spot the bug before opening `<details>`.
- Hints are one line; full root cause + fix in §4 keyed by bug number.
- Difficulty sections are cumulative — Subtle assumes Easy, Senior assumes Subtle.

## 1. Easy (warm-up traps)

### Bug 1 — TIME_WAIT exhaustion on an outbound connection-heavy client
```c
/* Worker that opens one short HTTP/1.0 connection per job. */
for (;;) {
    int s = socket(AF_INET, SOCK_STREAM, 0);
    connect(s, (struct sockaddr*)&srv, sizeof srv);
    write(s, req, req_len);
    read(s, buf, sizeof buf);
    close(s);                       /* active close -> TIME_WAIT */
}
/* After ~28k requests/min: connect() returns EADDRNOTAVAIL. */
```
<details><summary>Hint</summary>
The active closer holds the 4-tuple for 2*MSL; ephemeral ports run out.
</details>

### Bug 2 — `SO_REUSEADDR` thought to mean "share the listening port"
```c
int s = socket(AF_INET, SOCK_STREAM, 0);
int yes = 1;
setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes);
bind(s, ...);                       /* port 8080 */
listen(s, 128);
/* Two worker processes both try this. Second one: EADDRINUSE. */
```
<details><summary>Hint</summary>
`SO_REUSEADDR` is about TIME_WAIT remnants, not multi-process load balancing.
</details>

### Bug 3 — `close()` on a socket with unread data sends RST, not FIN
```c
/* Server reads request, sends huge response, then closes. */
write(s, big_response, n);
close(s);                           /* kernel buffers may still hold rx data */
/* Client: ECONNRESET. Sometimes truncated response. */
```
<details><summary>Hint</summary>
If the receive queue is non-empty at close, Linux aborts with RST (RFC 1122 §4.2.2.13).
</details>

### Bug 4 — `recv()` returning 0 treated as "try again"
```c
ssize_t n = recv(s, buf, sizeof buf, 0);
if (n < 0) { perror("recv"); continue; }
if (n == 0) { continue; }           /* "spurious wakeup, retry" */
```
<details><summary>Hint</summary>
`recv() == 0` is the orderly-shutdown signal, not a transient.
</details>

### Bug 5 — `send()` short-write assumed to be "all or nothing"
```c
ssize_t n = send(s, buf, len, 0);
if (n < 0) die("send");
/* App moves on, assumes len bytes were sent. */
```
<details><summary>Hint</summary>
Stream sockets may accept fewer bytes than requested; you must loop.
</details>

### Bug 6 — `connect()` to wrong address "succeeded"
```c
connect(s, (struct sockaddr*)&wrong, sizeof wrong);  /* returns 0 */
/* Hours later, first send() finally returns EHOSTUNREACH or times out. */
```
<details><summary>Hint</summary>
`connect()` to a routable-but-dead host can complete locally before the SYN ever fails.
</details>

### Bug 7 — Tiny `listen()` backlog under burst
```c
listen(s, 5);                       /* developer thought "5 is plenty" */
/* During a burst: clients see ECONNRESET / connection refused;
   netstat -s shows "X SYNs to LISTEN sockets dropped". */
```
<details><summary>Hint</summary>
The kernel caps the backlog at `net.core.somaxconn`; small backlogs drop SYNs under burst.
</details>

## 2. Subtle (review-passers)

### Bug 8 — Nagle + delayed ACK = 200 ms latency on small request/response
```c
/* RPC client: write header, then write body. */
write(s, hdr, 8);                   /* small */
write(s, body, 24);                 /* small */
read(s, reply, sizeof reply);
/* p99 latency: 200 ms instead of 1 ms. Reproduces only on Linux. */
```
<details><summary>Hint</summary>
Sender holds small segment waiting for ACK; receiver delays ACK waiting for data.
</details>

### Bug 9 — `TCP_NODELAY` set everywhere as a "performance fix"
```c
int yes = 1;
setsockopt(s, IPPROTO_TCP, TCP_NODELAY, &yes, sizeof yes);
/* Now a streaming uploader writes 40 bytes at a time and ships
   ~40-byte segments + 40-byte headers across the WAN. */
```
<details><summary>Hint</summary>
`TCP_NODELAY` is for latency-sensitive RPC, not for bulk streaming.
</details>

### Bug 10 — Half-open connection: peer crashed, no RST, client hangs forever
```c
/* Long-lived control channel. Server's NIC is yanked.
   No FIN, no RST. Client's read() blocks indefinitely. */
read(s, buf, sizeof buf);           /* never returns */
```
<details><summary>Hint</summary>
Without keepalives or app-level heartbeats, TCP cannot detect a dead silent peer.
</details>

### Bug 11 — Keepalive enabled, but defaults are 2 hours
```c
int yes = 1;
setsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &yes, sizeof yes);
/* "We have keepalive, so half-open is handled." */
/* Reality: tcp_keepalive_time defaults to 7200s on Linux. */
```
<details><summary>Hint</summary>
`SO_KEEPALIVE` alone honors system defaults; tune the per-socket knobs.
</details>

### Bug 12 — Ephemeral port range too narrow for a connection-heavy proxy
```sh
sysctl net.ipv4.ip_local_port_range
# 32768  60999    (default: ~28k usable ports per (src,dst,dport))
# Proxy fronting one upstream IP+port runs out at ~28k concurrent.
```
<details><summary>Hint</summary>
The 4-tuple is bounded by the local port range when peer (ip, port) is fixed.
</details>

### Bug 13 — `SO_LINGER{0}` used to "fix TIME_WAIT" — corrupts the protocol
```c
struct linger lg = { .l_onoff = 1, .l_linger = 0 };
setsockopt(s, SOL_SOCKET, SO_LINGER, &lg, sizeof lg);
close(s);                           /* now sends RST instead of FIN */
/* TIME_WAIT vanishes... and so does any unsent data, plus peer gets ECONNRESET. */
```
<details><summary>Hint</summary>
`l_linger=0` is "abort the connection now"; it discards data and breaks half-close semantics.
</details>

### Bug 14 — Receive window = 0 because the app stopped reading
```c
/* Slow consumer in a thread that occasionally blocks on disk. */
for (;;) {
    process(buf);                   /* sometimes 5s */
    recv(s, buf, sizeof buf, 0);
}
/* tcpdump: "win 0" advertised; sender stalls on zero-window probes. */
```
<details><summary>Hint</summary>
TCP flow control is the receive window; if you don't read, the window closes.
</details>

### Bug 15 — `ECONNRESET` on `write()` because peer half-closed and we kept writing
```c
/* Peer sent FIN and closed its read side, then exited.
   Our next write() goes through; the one after that returns ECONNRESET. */
write(s, chunk, n);                 /* OK */
write(s, chunk, n);                 /* ECONNRESET */
```
<details><summary>Hint</summary>
Writing to a peer that has fully closed produces RST on the next ACK round.
</details>

### Bug 16 — Connection pool reuses a stale socket holding a queued FIN
```python
conn = pool.borrow()                # idle for 10 minutes
conn.send(request)                  # returns OK
conn.recv()                         # 0 bytes -> mistaken as "empty response"
```
<details><summary>Hint</summary>
The peer (or a NAT/LB) closed the idle conn; the FIN is sitting in the kernel buffer.
</details>

### Bug 17 — Path-MTU black hole: ICMP filtered, large writes stall
```sh
# Tunnel link advertises MTU 1500 but a hop in the path enforces 1400.
# ICMP "Fragmentation Needed" is dropped by an upstream firewall.
curl -X POST --data-binary @50mb.bin https://api.example.com/upload
# Stalls forever after the first ~1400-byte segment. Small POSTs work.
```
<details><summary>Hint</summary>
PMTUD relies on ICMP; black-holed ICMP breaks DF=1 large segments.
</details>

## 3. Senior trap (production-only failures)

### Bug 18 — Retransmission backoff multiplies a brief outage into a 15-minute outage
```c
/* Default Linux: ~15 retries; RTO doubles per RFC 6298. After a 30s
   blip the connection has already been silent ~3 minutes; the app's
   write() has been blocked on the kernel queue for the whole window. */
write(s, chunk, n);                 /* eventually ETIMEDOUT, ~924s later */
```
<details><summary>Hint</summary>
Bound it explicitly with `TCP_USER_TIMEOUT` (RFC 5482) instead of relying on `tcp_retries2`.
</details>

### Bug 19 — Slow-start restart after idle collapses throughput
```c
/* HTTP/1.1 keep-alive connection, idle for 10 seconds, then a 10 MB POST.
   Throughput: 200 KB/s instead of the 50 MB/s the link sustains. */
```
<details><summary>Hint</summary>
Linux resets cwnd to IW after an idle period (RFC 5681 §4.1) unless told otherwise.
</details>

### Bug 20 — `TCP_USER_TIMEOUT` confused with `SO_RCVTIMEO`/`SO_SNDTIMEO`
```c
struct timeval tv = { .tv_sec = 5 };
setsockopt(s, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof tv);  /* per-call */
/* Team thought this also bounded write stalls and half-open detection. */
```
<details><summary>Hint</summary>
SO_RCVTIMEO/SO_SNDTIMEO bound a single syscall; they don't cap unacked data lifetime.
</details>

### Bug 21 — BSD/Linux divergence: FIN_WAIT_2 never expires on BSD
```c
/* Linux: tcp_fin_timeout (default 60s) reaps orphaned FIN_WAIT_2.
   BSD/macOS: no equivalent; orphaned FIN_WAIT_2 lingers until reset.
   Long-running daemon on BSD slowly accumulates FIN_WAIT_2 sockets. */
```
<details><summary>Hint</summary>
Linux's `tcp_fin_timeout` is documented as "strictly a violation of the TCP spec".
</details>

### Bug 22 — TLS over short-lived TCP without session resumption
```python
# Mobile client opens HTTPS, runs one query, closes, reopens 30s later.
# Each connection: full TLS 1.2 handshake (2 RTTs) + TCP handshake (1 RTT).
# p95 first-byte latency on cellular: 900 ms.
```
<details><summary>Hint</summary>
Without session tickets / TLS 1.3 0-RTT, each new TCP gets a full handshake.
</details>

## 4. Solutions

### Bug 1 — TIME_WAIT exhaustion on outbound clients
**Root cause:** Whoever calls `close()` first enters TIME_WAIT and holds the 4-tuple `(srcip, srcport, dstip, dstport)` for 2*MSL (typically 60 s on Linux). When the client is the active closer and the destination `(ip, port)` is fixed, the only varying coordinate is the local port — capped by `net.ipv4.ip_local_port_range`. A client doing thousands of short connections per minute exhausts that pool.
**Fix:**
```sh
# 1. Reuse one connection (HTTP keep-alive, connection pool).
# 2. Widen the ephemeral range:
sysctl -w net.ipv4.ip_local_port_range="10000 65535"
# 3. Allow reuse of TIME_WAIT slots for *outgoing* connections:
sysctl -w net.ipv4.tcp_tw_reuse=1
# Do NOT enable tcp_tw_recycle — removed in Linux 4.12 and unsafe behind NAT.
```
**Reference:** Vincent Bernat — ["Coping with the TCP TIME-WAIT state on busy Linux servers"](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux); RFC 9293 §3.6.1 (TIME-WAIT).

### Bug 2 — `SO_REUSEADDR` vs `SO_REUSEPORT`
**Root cause:** `SO_REUSEADDR` lets a new socket bind a port that has a TIME_WAIT remnant from a previous instance. It does **not** allow two live listeners on the same port. `SO_REUSEPORT` (Linux 3.9+) does — and balances incoming SYNs across them.
**Fix:**
```c
int yes = 1;
setsockopt(s, SOL_SOCKET, SO_REUSEPORT, &yes, sizeof yes);
bind(s, ...);
listen(s, 1024);
/* All workers must be owned by the same effective UID. */
```
**Reference:** Linux [`socket(7)`](https://man7.org/linux/man-pages/man7/socket.7.html) (SO_REUSEADDR/SO_REUSEPORT); LWN — ["The SO_REUSEPORT socket option"](https://lwn.net/Articles/542629/).

### Bug 3 — RST on close because receive queue not drained
**Root cause:** RFC 1122 §4.2.2.13 specifies that if `close()` is called on a socket with unread data in the receive queue, the implementation MUST send a RST instead of a FIN. Linux follows this. The peer sees `ECONNRESET` and may have unsent response bytes silently dropped from its send queue.
**Fix:**
```c
/* Drain before close, or use shutdown() for half-close: */
shutdown(s, SHUT_WR);               /* send FIN, keep reading */
while ((n = read(s, buf, sizeof buf)) > 0) { /* discard or process */ }
close(s);
```
**Reference:** [RFC 1122 §4.2.2.13](https://datatracker.ietf.org/doc/html/rfc1122#section-4.2.2.13); Linux `tcp(7)` "When the writing process closes the socket, it discards any unread data".

### Bug 4 — `recv() == 0` is EOF
**Root cause:** On a stream socket, a zero-length read means the peer has shut down its sending side. It is the orderly equivalent of EOF on a file. Treating it as a transient causes the loop to busy-spin a closed socket.
**Fix:**
```c
ssize_t n = recv(s, buf, sizeof buf, 0);
if (n == 0) { /* peer closed; clean up and break */ break; }
if (n < 0)  { if (errno == EINTR) continue; perror("recv"); break; }
```
**Reference:** Linux [`recv(2)`](https://man7.org/linux/man-pages/man2/recv.2.html) — RETURN VALUE: "When a stream socket peer has performed an orderly shutdown, the return value will be 0".

### Bug 5 — Short writes
**Root cause:** `send()`/`write()` on a stream socket return the number of bytes accepted into the kernel send buffer, which can be smaller than `len` whenever the buffer fills. POSIX guarantees nothing larger.
**Fix:**
```c
ssize_t send_all(int s, const char *buf, size_t len) {
    size_t off = 0;
    while (off < len) {
        ssize_t n = send(s, buf + off, len - off, MSG_NOSIGNAL);
        if (n < 0) { if (errno == EINTR) continue; return -1; }
        off += n;
    }
    return (ssize_t)off;
}
```
**Reference:** Linux [`send(2)`](https://man7.org/linux/man-pages/man2/send.2.html); Stevens, *UNIX Network Programming* — "writen()" idiom.

### Bug 6 — `connect()` "succeeds" to wrong address
**Root cause:** A blocking `connect()` returns success when the SYN/SYN-ACK/ACK handshake completes. If the destination is reachable but accepts every SYN (a misconfigured proxy, a wrong host that happens to listen) or if the route is silently null-routed, errors only surface on the first `read()`/`write()` (or after retransmissions exhaust). Equally, with non-blocking connect, you must check `SO_ERROR` after the socket becomes writable.
**Fix:**
```c
/* Non-blocking connect with SO_ERROR check: */
connect(s, ...);                          /* EINPROGRESS */
poll(&pfd, 1, 5000);
int err = 0; socklen_t el = sizeof err;
getsockopt(s, SOL_SOCKET, SO_ERROR, &err, &el);
if (err) { errno = err; return -1; }
/* Validate the peer at the application layer (e.g., TLS SNI/cert). */
```
**Reference:** Linux [`connect(2)`](https://man7.org/linux/man-pages/man2/connect.2.html); RFC 9293 §3.5 (Establishing a Connection).

### Bug 7 — `listen()` backlog under burst
**Root cause:** Linux maintains two queues: SYN queue (incomplete handshakes, sized by `tcp_max_syn_backlog`) and accept queue (completed but not yet `accept()`ed, sized by `min(somaxconn, listen_backlog)`). When the accept queue fills, fresh SYN-ACKs are dropped or — depending on `tcp_abort_on_overflow` — RSTed. Burst-driven `connection refused` is the symptom.
**Fix:**
```sh
sysctl -w net.core.somaxconn=4096
# In code:
# listen(s, 4096);
# Confirm queue health:
ss -ltn                              # Recv-Q (current accept-q) vs Send-Q (max)
nstat | grep -E 'ListenOverflow|ListenDrops'
```
**Reference:** Linux kernel docs — [networking/ip-sysctl.rst](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html) (somaxconn, tcp_max_syn_backlog); Cloudflare — ["SYN packet handling in the wild"](https://blog.cloudflare.com/syn-packet-handling-in-the-wild/).

### Bug 8 — Nagle + delayed ACK 200 ms stall
**Root cause:** Nagle's algorithm (RFC 896) holds small segments at the sender until either the buffer reaches MSS or the previous outstanding segment is ACKed. The receiver runs delayed ACKs (RFC 1122 §4.2.3.2) — typically up to 200 ms. Two small back-to-back writes from the client trigger Nagle's wait; the server is delaying the ACK; both stall until the 200 ms timer fires.
**Fix:**
```c
/* Option A: combine the writes into one buffer / use writev(). */
struct iovec iov[2] = { {hdr,8}, {body,24} };
writev(s, iov, 2);

/* Option B (RPC patterns): TCP_NODELAY for latency-critical sockets. */
int yes = 1;
setsockopt(s, IPPROTO_TCP, TCP_NODELAY, &yes, sizeof yes);

/* Option C (Linux): TCP_CORK to batch then flush deliberately. */
```
**Reference:** [RFC 896 (Nagle)](https://datatracker.ietf.org/doc/html/rfc896); [RFC 1122 §4.2.3.2 (Delayed ACK)](https://datatracker.ietf.org/doc/html/rfc1122#section-4.2.3.2); John Nagle's own commentary on the interaction (Hacker News, 2015).

### Bug 9 — `TCP_NODELAY` everywhere
**Root cause:** Disabling Nagle ships a segment per write call. For bulk streams that write small chunks (logs, telemetry, naive serializers), this multiplies overhead — header bytes can exceed payload, and the sender bursts tiny segments that compete with other flows.
**Fix:**
```c
/* Buffer in user space, then write large chunks: */
fwrite(rec, 1, n, stream);          /* stdio buffer */
/* Or use TCP_CORK to batch:                              */
int on = 1;
setsockopt(s, IPPROTO_TCP, TCP_CORK, &on, sizeof on);
write(s, hdr, hdr_len);
write(s, body, body_len);
on = 0;
setsockopt(s, IPPROTO_TCP, TCP_CORK, &on, sizeof on);  /* flush */
```
**Reference:** Linux [`tcp(7)`](https://man7.org/linux/man-pages/man7/tcp.7.html) (TCP_NODELAY, TCP_CORK).

### Bug 10 — Half-open hang
**Root cause:** When the peer's host disappears without a clean shutdown (power loss, NIC pull, kernel panic without RST) and no segments arrive at our side, TCP has no way to know. The local socket sits in ESTABLISHED forever; `read()` blocks; `write()` succeeds into the buffer until retransmissions eventually time out (minutes).
**Fix:**
```c
/* Enable keepalive AND tune it: */
int on = 1, idle = 60, intvl = 10, cnt = 3;
setsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &on,    sizeof on);
setsockopt(s, IPPROTO_TCP, TCP_KEEPIDLE,  &idle, sizeof idle);
setsockopt(s, IPPROTO_TCP, TCP_KEEPINTVL, &intvl,sizeof intvl);
setsockopt(s, IPPROTO_TCP, TCP_KEEPCNT,   &cnt,  sizeof cnt);

/* Better for write-heavy paths: bound unacked data lifetime. */
unsigned int ms = 30000;
setsockopt(s, IPPROTO_TCP, TCP_USER_TIMEOUT, &ms, sizeof ms);

/* Best: application heartbeats (PING/PONG) so you control liveness semantics. */
```
**Reference:** Linux [`tcp(7)`](https://man7.org/linux/man-pages/man7/tcp.7.html); [RFC 1122 §4.2.3.6 (TCP Keep-Alives)](https://datatracker.ietf.org/doc/html/rfc1122#section-4.2.3.6); [RFC 5482 (TCP_USER_TIMEOUT)](https://datatracker.ietf.org/doc/html/rfc5482).

### Bug 11 — Keepalive defaults are 2 hours
**Root cause:** `SO_KEEPALIVE` enables the feature but uses system-wide defaults: `tcp_keepalive_time=7200`, `tcp_keepalive_intvl=75`, `tcp_keepalive_probes=9`. That is ~2h11m to detect a dead peer — useless behind NAT/LB devices that drop idle flows in 60–300 s.
**Fix:** Set `TCP_KEEPIDLE`, `TCP_KEEPINTVL`, `TCP_KEEPCNT` per socket (see Bug 10 fix). Match values to the shortest middlebox idle timeout in the path (e.g., AWS NLB idle ≈ 350 s; pick `idle < 60 s`).
**Reference:** Linux [`tcp(7)`](https://man7.org/linux/man-pages/man7/tcp.7.html); kernel.org [networking/ip-sysctl.rst](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html); Cloudflare — ["When TCP sockets refuse to die"](https://blog.cloudflare.com/when-tcp-sockets-refuse-to-die/).

### Bug 12 — Ephemeral port range
**Root cause:** Outgoing connections to a single `(dstip, dstport)` are uniquely keyed by source port. Default range `32768–60999` gives ~28 232 simultaneous connections to that endpoint per source IP. A connection-heavy proxy bottlenecks here long before kernel limits.
**Fix:**
```sh
# Widen the range:
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
# Allow safe TIME_WAIT reuse for outbound:
sysctl -w net.ipv4.tcp_tw_reuse=1
# Architectural fix: multiple source IPs (SO_BINDTODEVICE / IP_BIND_ADDRESS_NO_PORT)
# or pool/reuse connections.
int one = 1;
setsockopt(s, IPPROTO_IP, IP_BIND_ADDRESS_NO_PORT, &one, sizeof one);
```
**Reference:** kernel.org [networking/ip-sysctl.rst](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html) (`ip_local_port_range`, `tcp_tw_reuse`); Vincent Bernat — TIME-WAIT article cited above.

### Bug 13 — `SO_LINGER{0}` abuse
**Root cause:** With `l_onoff=1, l_linger=0`, `close()` causes an immediate RST instead of FIN, skipping TIME_WAIT — but it also discards any data still in the send queue and signals the peer that the conversation was aborted. Used as a "fix" for TIME_WAIT it produces silent data loss and `ECONNRESET` storms at the peer.
**Fix:**
```c
/* Don't. Solve the real problem — too many active closes — instead. */
/* If you truly want abortive close (e.g., on a security event):     */
struct linger lg = { .l_onoff = 1, .l_linger = 0 };
setsockopt(s, SOL_SOCKET, SO_LINGER, &lg, sizeof lg);
/* and document it as a deliberate RST. */
```
**Reference:** Linux [`socket(7)`](https://man7.org/linux/man-pages/man7/socket.7.html) (SO_LINGER); Stevens, *TCP/IP Illustrated, Vol. 1* — connection termination chapter.

### Bug 14 — Zero receive window from slow consumer
**Root cause:** TCP advertises window = (recv buffer free space). If the application stops reading, the buffer fills, the window goes to zero, and the sender stalls — periodically sending zero-window probes. Throughput collapses to whatever the consumer can drain.
**Fix:**
```c
/* 1. Decouple I/O from work: read into a queue, process in another thread. */
/* 2. Size the buffer for bandwidth-delay product:                          */
int rcvbuf = 4 * 1024 * 1024;
setsockopt(s, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof rcvbuf);
/* Linux autotunes within tcp_rmem; only override when you know better.     */
```
**Reference:** [RFC 9293 §3.8.6 (Flow Control)](https://datatracker.ietf.org/doc/html/rfc9293#section-3.8.6); [RFC 7323 (TCP Window Scale Option)](https://datatracker.ietf.org/doc/html/rfc7323); Linux [`tcp(7)`](https://man7.org/linux/man-pages/man7/tcp.7.html) (`tcp_rmem`).

### Bug 15 — `ECONNRESET` from writing to a closed peer
**Root cause:** When a peer fully closes (FIN sent, then no more data accepted), our first write may still be queued and ACKed normally; the next one elicits a RST because the peer has no socket left. POSIX delivers this as `EPIPE` or `ECONNRESET` plus `SIGPIPE` (default: terminate process).
**Fix:**
```c
/* Always mask SIGPIPE per write or globally: */
ssize_t n = send(s, buf, len, MSG_NOSIGNAL);   /* Linux */
if (n < 0 && (errno == EPIPE || errno == ECONNRESET)) {
    /* Peer is gone; tear down and reconnect. */
}
/* Globally:
signal(SIGPIPE, SIG_IGN);
*/
```
**Reference:** Linux [`send(2)`](https://man7.org/linux/man-pages/man2/send.2.html); RFC 9293 §3.5.2 (Reset Generation).

### Bug 16 — Connection pool returning stale conn with queued FIN
**Root cause:** TCP connections idle in a pool can be closed by the peer or an intermediary (LB, NAT, firewall idle timeout). The FIN sits in the local socket buffer; the next `send()` on that socket may succeed (it queues into the kernel), but the next `recv()` returns 0. Pools that don't validate before lending out hand callers a corpse.
**Fix:**
```python
# Validate idle conns before lending:
def borrow(self):
    while self.idle:
        c = self.idle.pop()
        if c.idle_seconds() > MAX_IDLE: c.close(); continue
        if not c.is_alive():           c.close(); continue   # MSG_PEEK / TCP_INFO
        return c
    return self.connect()

# Cap pool idle time below NAT/LB idle timeout (typ. 60–350s).
```
On Linux, `getsockopt(s, IPPROTO_TCP, TCP_INFO, ...)` exposes `tcpi_state` — anything other than `TCP_ESTABLISHED` means dead.
**Reference:** Linux [`tcp(7)`](https://man7.org/linux/man-pages/man7/tcp.7.html) (TCP_INFO); Heroku — ["Cedar router H13/H18 errors"](https://devcenter.heroku.com/articles/error-codes#h13-connection-closed-without-response) (idle-conn pool failure mode in the wild).

### Bug 17 — PMTU black hole
**Root cause:** PMTU Discovery sets DF=1 and relies on intermediate routers returning ICMP "Fragmentation Needed" so the sender can lower its segment size. Many networks filter ICMP. Result: small segments succeed, large ones are silently dropped, retransmitted at the same size, and dropped again.
**Fix:**
```sh
# Workaround: enable Linux's PMTUD black-hole detection (gradually shrinks MSS):
sysctl -w net.ipv4.tcp_mtu_probing=1
# Or pin a safe MSS via iptables / per-route MSS clamping at the gateway:
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu
# Permanent fix: stop filtering ICMP type 3 code 4 (frag-needed).
```
**Reference:** [RFC 4821 (Packetization Layer PMTU Discovery)](https://datatracker.ietf.org/doc/html/rfc4821); kernel.org [networking/ip-sysctl.rst](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html) (`tcp_mtu_probing`); Cloudflare — ["Path MTU discovery in practice"](https://blog.cloudflare.com/path-mtu-discovery-in-practice/).

### Bug 18 — Retransmission backoff = 15-minute outage
**Root cause:** RFC 6298 mandates exponential RTO backoff. Linux retries up to `tcp_retries2` times (default 15) before giving up — total around 924 s. During the entire window, application `write()`s on a long-lived connection block in the kernel queue. A 30 s upstream blip becomes a 15-minute hang for every in-flight request.
**Fix:**
```c
/* Bound the wait at the connection level: */
unsigned int ms = 30000;            /* 30s */
setsockopt(s, IPPROTO_TCP, TCP_USER_TIMEOUT, &ms, sizeof ms);
/* Now ETIMEDOUT after 30s of unacked data, regardless of retransmissions. */
```
For listeners: cap `tcp_retries2` system-wide if you want a global ceiling. For HTTP/RPC, also set application-layer deadlines (`context.WithDeadline`, etc.).
**Reference:** [RFC 6298 (Computing TCP's Retransmission Timer)](https://datatracker.ietf.org/doc/html/rfc6298); [RFC 5482 (TCP_USER_TIMEOUT)](https://datatracker.ietf.org/doc/html/rfc5482); kernel.org [networking/ip-sysctl.rst](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html) (`tcp_retries2`).

### Bug 19 — Slow-start restart after idle
**Root cause:** RFC 5681 §4.1 specifies that a TCP sender SHOULD restart at the initial window after an idle period exceeding the RTO. Linux implements this via `tcp_slow_start_after_idle` (default 1). On long-lived connections (HTTP keep-alive, gRPC, queue consumers) this collapses cwnd whenever traffic pauses, even on perfect networks.
**Fix:**
```sh
# If your workload is bursty and the path is well-behaved:
sysctl -w net.ipv4.tcp_slow_start_after_idle=0
# Or keep the connection warm with a cheap heartbeat smaller than the RTO.
```
Document the choice; SSR exists to protect shared paths from stale congestion estimates.
**Reference:** [RFC 5681 §4.1 (Restarting Idle Connections)](https://datatracker.ietf.org/doc/html/rfc5681#section-4.1); kernel.org [networking/ip-sysctl.rst](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html) (`tcp_slow_start_after_idle`).

### Bug 20 — `TCP_USER_TIMEOUT` vs `SO_RCVTIMEO`/`SO_SNDTIMEO`
**Root cause:** Three different timers, three different semantics:
- `SO_RCVTIMEO`/`SO_SNDTIMEO`: bound a single blocking `recv`/`send` call. Returns `EAGAIN`. Doesn't fail the connection.
- `TCP_USER_TIMEOUT`: bound how long unacknowledged data may sit in the send queue before TCP forcibly closes the connection. Detects half-open / dead peer for write paths.
- Application deadlines: bound an entire request lifecycle.

Confusing them produces "we set a timeout but the connection still hangs for 15 minutes" (Bug 18) or "writes return success but no one is listening".
**Fix:** Use them together — per-call `SO_*TIMEO` for syscall granularity, `TCP_USER_TIMEOUT` for connection liveness, and application-layer deadlines for end-to-end SLOs. Don't rely on any single one.
**Reference:** [RFC 5482 (TCP_USER_TIMEOUT)](https://datatracker.ietf.org/doc/html/rfc5482); Linux [`socket(7)`](https://man7.org/linux/man-pages/man7/socket.7.html), [`tcp(7)`](https://man7.org/linux/man-pages/man7/tcp.7.html).

### Bug 21 — BSD/Linux divergence on FIN_WAIT_2
**Root cause:** RFC 9293 has no FIN_WAIT_2 timeout — a peer holding open the read side can keep us in FIN_WAIT_2 forever. Linux added `tcp_fin_timeout` (default 60 s) as a pragmatic DoS defence and explicitly documents this as "strictly a violation of the TCP specification". BSD/macOS do not have an equivalent reaper, so long-running daemons there can accumulate orphan FIN_WAIT_2 sockets and exhaust file descriptors / port pool.
**Fix:**
```sh
# Linux: confirm the reaper is enabled (it is by default):
sysctl net.ipv4.tcp_fin_timeout
# BSD/macOS: rely on application-level shutdown discipline.
# Always shutdown(s, SHUT_WR) and then drain rather than relying on close()
# alone to push the peer out of CLOSE_WAIT.
```
**Reference:** Linux [`tcp(7)`](https://man7.org/linux/man-pages/man7/tcp.7.html) (`tcp_fin_timeout`); Cloudflare — ["This is strictly a violation of the TCP specification" (Marek Majkowski)](https://blog.cloudflare.com/this-is-strictly-a-violation-of-the-tcp-specification/); [RFC 9293 §3.6](https://datatracker.ietf.org/doc/html/rfc9293#section-3.6).

### Bug 22 — TLS over short-lived TCP without resumption
**Root cause:** Each fresh TCP connection costs one RTT for the handshake; layered TLS 1.2 adds two more. On cellular RTTs of 200–400 ms, that is ~1 s before any application byte. Without TLS session tickets / IDs, every connection pays the full cost; with TLS 1.3 you can also use 0-RTT for repeat requests.
**Fix:**
```python
# Server: enable session tickets (most TLS libraries do by default).
# Client: keep the TLS session cache across TCP connections.
ctx = ssl.create_default_context()
# Reuse `ctx` and its session cache across requests; or use HTTP/2/3 to
# multiplex many requests over one TCP connection.
```
For mobile-first: prefer HTTP/2 (single long-lived TCP) or HTTP/3 (QUIC, 0/1-RTT handshakes built in).
**Reference:** [RFC 8446 §2.2 (TLS 1.3 0-RTT)](https://datatracker.ietf.org/doc/html/rfc8446#section-2.2); [RFC 5077 (TLS Session Tickets)](https://datatracker.ietf.org/doc/html/rfc5077); Cloudflare — ["Introducing 0-RTT"](https://blog.cloudflare.com/introducing-0-rtt/).

## Related
- [TCP deep dive](tcp-deep-dive.md)
- [Socket programming](socket-programming.md)
- [Connection pooling](../network-programming/connection-pooling.md)
- [Network debugging](../network-programming/network-debugging.md)

## References
- RFCs: [RFC 9293 (TCP)](https://datatracker.ietf.org/doc/html/rfc9293), [RFC 7414 (TCP Roadmap)](https://datatracker.ietf.org/doc/html/rfc7414), [RFC 6298 (RTO)](https://datatracker.ietf.org/doc/html/rfc6298), [RFC 5681 (Congestion Control)](https://datatracker.ietf.org/doc/html/rfc5681), [RFC 7323 (TCP Extensions)](https://datatracker.ietf.org/doc/html/rfc7323), [RFC 1122 (Host Requirements)](https://datatracker.ietf.org/doc/html/rfc1122), [RFC 5482 (TCP_USER_TIMEOUT)](https://datatracker.ietf.org/doc/html/rfc5482), [RFC 4821 (PLPMTUD)](https://datatracker.ietf.org/doc/html/rfc4821), [RFC 896 (Nagle)](https://datatracker.ietf.org/doc/html/rfc896), [RFC 8446 (TLS 1.3)](https://datatracker.ietf.org/doc/html/rfc8446), [RFC 5077 (TLS Session Tickets)](https://datatracker.ietf.org/doc/html/rfc5077)
- Linux man pages: [`tcp(7)`](https://man7.org/linux/man-pages/man7/tcp.7.html), [`socket(7)`](https://man7.org/linux/man-pages/man7/socket.7.html), [`send(2)`](https://man7.org/linux/man-pages/man2/send.2.html), [`recv(2)`](https://man7.org/linux/man-pages/man2/recv.2.html), [`connect(2)`](https://man7.org/linux/man-pages/man2/connect.2.html)
- Linux kernel docs: [networking/ip-sysctl.rst](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
- Authority blogs: Vincent Bernat — ["Coping with the TCP TIME-WAIT state on busy Linux servers"](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux); Cloudflare — ["This is strictly a violation of the TCP specification"](https://blog.cloudflare.com/this-is-strictly-a-violation-of-the-tcp-specification/), ["When TCP sockets refuse to die"](https://blog.cloudflare.com/when-tcp-sockets-refuse-to-die/), ["SYN packet handling in the wild"](https://blog.cloudflare.com/syn-packet-handling-in-the-wild/), ["Path MTU discovery in practice"](https://blog.cloudflare.com/path-mtu-discovery-in-practice/), ["Introducing 0-RTT"](https://blog.cloudflare.com/introducing-0-rtt/); LWN — ["The SO_REUSEPORT socket option"](https://lwn.net/Articles/542629/)
- Books: W. Richard Stevens, *TCP/IP Illustrated, Vol. 1*; Stevens et al., *UNIX Network Programming, Vol. 1*
