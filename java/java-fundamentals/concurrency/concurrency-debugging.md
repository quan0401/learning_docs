---
title: "Concurrency Debugging Playbook — Thread Dumps, JFR, async-profiler, jcstress"
date: 2026-04-24
updated: 2026-04-24
tags: [java, concurrency, debugging, thread-dump, jfr, async-profiler, jcstress]
---

# Concurrency Debugging Playbook — Thread Dumps, JFR, async-profiler, jcstress

**Date:** 2026-04-24 | **Updated:** 2026-04-24
**Tags:** `java` `concurrency` `debugging` `thread-dump` `jfr` `async-profiler` `jcstress`

## Table of Contents

- [Summary](#summary)
- [Getting Thread Dumps](#getting-thread-dumps)
- [Reading a Thread Dump](#reading-a-thread-dump)
- [Deadlock Detection](#deadlock-detection)
- [Livelock and Starvation](#livelock-and-starvation)
- [JFR for Concurrency](#jfr-for-concurrency)
- [async-profiler Lock Contention](#async-profiler-lock-contention)
- [jcstress for Lock-Free Correctness](#jcstress-for-lock-free-correctness)
- [Micrometer Thread Metrics](#micrometer-thread-metrics)
- [Common Patterns in the Wild](#common-patterns-in-the-wild)
- [Related](#related)
- [References](#references)

---

## Summary

95% of production concurrency incidents are solved with four tools: thread dumps (`jstack`, `jcmd`), JDK Flight Recorder, [async-profiler](https://github.com/async-profiler/async-profiler) with `-e lock`, and [jcstress](https://github.com/openjdk/jcstress) for correctness. Thread dumps tell you *what every thread is doing right now*. JFR tells you *what they did over time*. async-profiler gives you *flame graphs of lock contention*. jcstress stress-tests *lock-free code for JMM correctness bugs*. Combined with thread-states awareness (RUNNABLE / WAITING / BLOCKED / TIMED_WAITING), this doc is the field playbook for the deadlock / livelock / starvation / thread-leak incidents that actually happen in production.

---

## Getting Thread Dumps

Five ways to get a thread dump:

```bash
# 1. jstack (simplest, oldest)
jstack <pid>

# 2. jcmd (modern, better format)
jcmd <pid> Thread.print

# 3. SIGQUIT (kill signal, dumps to stdout)
kill -3 <pid>

# 4. Programmatically (inside the JVM)
ThreadMXBean mx = ManagementFactory.getThreadMXBean();
ThreadInfo[] infos = mx.dumpAllThreads(true, true);

# 5. From Java Mission Control / VisualVM (GUI, click "Threads" → "Thread Dump")
```

Rules:
- `jcmd` is preferred — better format, works on all JVMs.
- Collect **three dumps 10 seconds apart** — a single dump is a snapshot; trends matter for "is this thread stuck?".
- Dumps are cheap (< 10 ms usually). Do not hesitate in production incidents.

---

## Reading a Thread Dump

Anatomy of a single thread entry:

```text
"http-nio-8080-exec-3" #42 daemon prio=5 os_prio=0 tid=0x00007f... nid=0x1a23
   java.lang.Thread.State: BLOCKED (on object monitor)
   at com.example.OrderService.placeOrder(OrderService.java:73)
   - waiting to lock <0x00000000f8a> (a com.example.Inventory)
   at com.example.CheckoutController.checkout(CheckoutController.java:21)
   at jdk.internal.reflect.GeneratedMethodAccessor25.invoke(...)
   ...
```

Key fields:

- **`"http-nio-8080-exec-3"`** — thread name. Name your threads! `Thread-42` tells you nothing.
- **State**:
  - `RUNNABLE` — on CPU or waiting for the scheduler.
  - `BLOCKED` — waiting for a `synchronized` / monitor lock.
  - `WAITING` — parked on `LockSupport.park`, `Object.wait` with no timeout, `Thread.join()`.
  - `TIMED_WAITING` — `sleep`, `wait(timeout)`, `LockSupport.parkNanos`, `Future.get(timeout)`.
  - `TERMINATED` — done.
- **`waiting to lock <0x...>`** — monitor address this thread wants.
- **`locked <0x...>`** — monitor address this thread holds.

Skim for patterns:
- Many threads BLOCKED on the same monitor → contention point. Next step: who holds it?
- Thread in WAITING on `LockSupport.parkNanos` → likely a `Future.get()` or `BlockingQueue.poll`.
- Thread stuck in `Object.wait` with an owning `synchronized` lock — classic deadlock precursor.

---

## Deadlock Detection

Modern JVMs auto-detect deadlocks. The dump includes:

```text
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f... (object 0x00000000f8a, a com.example.A),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007f... (object 0x00000000fc3, a com.example.B),
  which is held by "Thread-1"
```

Programmatic check:

```java
ThreadMXBean mx = ManagementFactory.getThreadMXBean();
long[] deadlocked = mx.findDeadlockedThreads();
if (deadlocked != null) {
    ThreadInfo[] infos = mx.getThreadInfo(deadlocked, true, true);
    for (ThreadInfo info : infos) log.error("DEADLOCK:\n{}", info);
}
```

Run this on a schedule in production. Page the oncall when it fires.

Caveat: `findDeadlockedThreads()` detects only monitor + Lock cycles, not logical/semantic deadlocks (two services waiting for each other). For those, [distributed tracing](../../observability/distributed-tracing.md) finds the cycle.

---

## Livelock and Starvation

**Livelock**: threads running, not blocked, but no progress. CPU high, throughput zero. Classic cause: two threads both backing off when they detect contention, both re-trying simultaneously.

Symptoms:
- High CPU (all RUNNABLE).
- No BLOCKED threads.
- Application-level metrics (throughput, latency) go to hell.

Detection: hard. Thread dump looks normal. Use JFR's "CPU sample" view — all samples in the same retry loop. Add retry-backoff with jitter (exponential + randomized) to break the symmetry.

**Starvation**: one thread never gets the lock because another always wins. Fair vs unfair locks:

```java
ReentrantLock fair = new ReentrantLock(true);   // FIFO ordering
ReentrantLock unfair = new ReentrantLock();     // default; faster but can starve
```

Fair locks ~5× slower under contention. Only use them when starvation is an actual problem.

Priority inversion is the classic RTOS concept — rarely relevant on the JVM. Don't set `Thread.setPriority()` (the OS usually ignores it anyway).

---

## JFR for Concurrency

[JDK Flight Recorder](https://docs.oracle.com/en/java/javase/21/jfapi/) captures events with < 1% overhead. Relevant events:

- `jdk.JavaMonitorEnter` — thread waiting to acquire a monitor.
- `jdk.JavaMonitorWait` — thread blocked in `Object.wait`.
- `jdk.ThreadPark` — thread parked in `LockSupport.park`.
- `jdk.SafepointBegin/End` — GC and safepoint pauses (affects concurrency perf).
- `jdk.ThreadCPULoad` — per-thread CPU.
- `jdk.JavaThreadStatistics` — thread count over time.

Enable always-on recording:

```bash
-XX:StartFlightRecording=filename=/var/log/recording.jfr,maxage=24h,maxsize=1g,settings=profile
```

On-demand dump:

```bash
jcmd <pid> JFR.start name=incident duration=60s filename=/tmp/incident.jfr
jcmd <pid> JFR.dump name=incident filename=/tmp/incident.jfr
```

Open with [JDK Mission Control](https://adoptium.net/jmc/). The "Lock Instances" view aggregates hot monitors.

---

## async-profiler Lock Contention

[async-profiler's](https://github.com/async-profiler/async-profiler) `-e lock` mode profiles contention:

```bash
java -jar async-profiler.jar -e lock -d 60 -f /tmp/locks.html <pid>
```

Output: flame graph where each leaf is a hot synchronized / Lock acquisition. The wider the frame, the more wait time spent.

Typical findings:
- `synchronized` on a JDK class (`Hashtable.put`, `Vector.add`) — legacy code; migrate to concurrent structures.
- Collection lock on a `synchronizedList` wrapper — migrate to `CopyOnWriteArrayList` or proper concurrent structure.
- `Lock.lock()` on a shared resource — consider sharding or optimistic read (see [concurrent-collections.md § LongAdder](concurrent-collections.md#longadder-vs-atomiclong)).

Pair with CPU profiling (`-e cpu`) to see whether contention costs real CPU or just wall-clock time.

---

## jcstress for Lock-Free Correctness

[jcstress](https://github.com/openjdk/jcstress) (Aleksey Shipilëv) stress-tests lock-free code for JMM correctness bugs. Example:

```java
@JCStressTest
@Outcome(id = "1, 1", expect = ACCEPTABLE, desc = "Normal")
@Outcome(id = "0, 1", expect = ACCEPTABLE_INTERESTING, desc = "Reordering")
@State
public class MyTest {
    int x, y;

    @Actor
    public void actor1() { x = 1; y = 1; }

    @Actor
    public void actor2(II_Result r) { r.r1 = y; r.r2 = x; }
}
```

jcstress runs millions of actor pairs in parallel, checks every observed outcome against the `@Outcome` annotations, and flags unexpected results (JMM violations, missing `volatile`, broken CAS).

Use this for:
- Writing custom lock-free data structures.
- Verifying that `volatile`/acquire-release barriers do what you think.
- Reproducing subtle concurrency bugs in unit tests.

Don't use this for business-logic code — it's overkill and slow.

---

## Micrometer Thread Metrics

Spring Boot + Micrometer auto-exposes:

| Metric | Purpose |
|--------|---------|
| `jvm.threads.live` | Currently live threads. |
| `jvm.threads.peak` | Max threads since JVM start. |
| `jvm.threads.daemon` | Daemon thread count. |
| `jvm.threads.started` | Total threads ever created (monotonic). |
| `jvm.threads.states{state=...}` | Breakdown by state (RUNNABLE, WAITING, BLOCKED). |

Dashboard rules:
- **`jvm.threads.live` trending up** → thread leak. Find who's creating threads without reclaiming.
- **`jvm.threads.states{state="BLOCKED"}` > 10%** → contention. Investigate.
- **`jvm.threads.started` rate > 10/sec** sustained → missing thread pool somewhere. Look for `new Thread(...)` or `SimpleAsyncTaskExecutor` usage.

Also useful: executor-specific metrics (`executor.pool.size`, `executor.queued`, `executor.completed`) via `ExecutorServiceMetrics.monitor(registry, executor, "name")`.

See [distributed-tracing.md](../../observability/distributed-tracing.md) and [reactive-observability.md](../../reactive-observability.md).

---

## Common Patterns in the Wild

| Symptom | Likely cause | Next step |
|---------|--------------|-----------|
| Threads piling up over time | Pool missing — `SimpleAsyncTaskExecutor` creates unbounded threads | Add `ThreadPoolTaskExecutor` bean |
| HTTP handlers in BLOCKED on DB pool | HikariCP exhausted; more threads than connections | Resize pool ([mvc-high-throughput.md](../../web-layer/mvc-high-throughput.md)) |
| Caller-Runs rejection triggering | Pool saturated; callers doing work they shouldn't | Increase pool or rate-limit upstream |
| `CompletableFuture` callback never fires | `commonPool` starved by blocking work | Move to dedicated executor |
| Lock contention on a single `synchronized` | Hot path needs lock splitting | See [concurrent-collections.md](concurrent-collections.md) |
| InterruptedException silent | Swallowed somewhere | grep `catch.*InterruptedException.*{}` (see [interruption-and-cancellation.md](interruption-and-cancellation.md)) |
| VT pinning under load (pre-JDK-24) | `synchronized` block inside I/O path | Replace with `ReentrantLock` or upgrade JDK ([virtual-threads.md](virtual-threads.md)) |
| Memory grows with VT count | Per-VT `ThreadLocal` | Migrate to `ScopedValue` ([threadlocal-and-context.md](threadlocal-and-context.md)) |
| Parallel stream slower than sequential | Source doesn't split well, or commonPool busy | Measure with JMH ([forkjoinpool-and-parallel-streams.md](forkjoinpool-and-parallel-streams.md)) |

---

## Related

- [Concurrency Basics](concurrency-basics.md) — thread states, monitors, fundamentals.
- [Multithreading Deep Dive](multithreading-deep-dive.md) — `ReentrantLock`, `ThreadPoolExecutor`, detecting deadlocks via `ThreadMXBean`.
- [Interruption and Cancellation](interruption-and-cancellation.md) — the most common "stuck thread" root cause.
- [CompletableFuture Deep Dive](completablefuture-deep-dive.md) — `commonPool` starvation.
- [ForkJoinPool and Parallel Streams](forkjoinpool-and-parallel-streams.md) — work-stealing pitfalls.
- [Concurrent Collections](concurrent-collections.md) — lock-striped alternatives to `synchronized`.
- [Virtual Threads](virtual-threads.md) — pinning and VT-specific debug.
- [ThreadLocal and Context Propagation](threadlocal-and-context.md) — TL leaks in pools.
- [Distributed Tracing](../../observability/distributed-tracing.md) — cross-service "deadlock" = deadline exhaustion.
- [GC Pause Diagnosis](../../jvm-gc/pause-diagnosis.md) — safepoint pauses masquerade as concurrency issues.
- [Scaling MVC Before Virtual Threads](../../web-layer/mvc-high-throughput.md) — HikariCP + thread pool sizing rules.

---

## References

- [`ThreadMXBean` Javadoc (JDK 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.management/java/lang/management/ThreadMXBean.html)
- [JFR Flight Recorder Guide](https://docs.oracle.com/en/java/javase/21/jfapi/flight-recorder-api-guide.html)
- [JDK Mission Control](https://adoptium.net/jmc/)
- [async-profiler](https://github.com/async-profiler/async-profiler)
- [jcstress — Java Concurrency Stress tests](https://github.com/openjdk/jcstress)
- [Aleksey Shipilëv — *JVM Anatomy Quarks*](https://shipilev.net/jvm/anatomy-quarks/) — safepoints, thread states, monitors.
- [Brendan Gregg — *Systems Performance*](https://www.brendangregg.com/sysperfbook.html) — off-CPU flame graphs.
- [Brian Goetz et al. — *Java Concurrency in Practice*](https://jcip.net/) — Chapter 10 ("Avoiding Liveness Hazards").
