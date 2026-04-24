# Worker Threads & Concurrency

**Updated:** 2026-04-24

> Node is "single-threaded" — but that is an oversimplification that collapses the moment you look inside V8 and libuv. When your code needs real parallelism, Node gives you three distinct mechanisms. This doc covers all of them, with Java parallels throughout. For the JVM side of the same ideas, keep [Java Concurrency Basics for TypeScript Developers](../../java/java-fundamentals/concurrency-basics.md), [Java Multithreading Deep Dive](../../java/java-fundamentals/multithreading-deep-dive.md), and [Virtual Threads in Java](../../java/java-fundamentals/virtual-threads.md) nearby.

---

## The Truth About Node's Threads

The statement "Node is single-threaded" means: **your JavaScript executes on a single thread (the main thread)**. But Node the process is not single-threaded at all.

### Threads You Already Have

| Thread(s) | Owner | What It Does |
|-----------|-------|--------------|
| Main thread | V8 | Runs your JS, the event loop, microtasks |
| libuv thread pool (4 by default) | libuv | `fs.*` (sync I/O turned async), `dns.lookup`, `crypto.pbkdf2`, `zlib`, some `crypto` |
| V8 helper threads | V8 | Background compilation (TurboFan), GC (concurrent marking, sweeping) |
| Watcher thread | libuv | `epoll`/`kqueue` I/O polling on some platforms |

### libuv Thread Pool Size

The default pool has **4 threads**. For workloads that hammer `fs` or `dns.lookup`, this bottleneck matters.

```bash
# Increase pool size (must be set before Node starts)
UV_THREADPOOL_SIZE=16 node server.js

# Maximum is 1024
```

```typescript
// You can also set it programmatically, but it must happen
// before any async operation touches the pool
process.env.UV_THREADPOOL_SIZE = "16";
```

**Java parallel**: Java has no direct equivalent because it does not use a thread pool for filesystem I/O — the JVM issues blocking syscalls on the calling thread (or virtual thread with Loom). The libuv pool is Node's workaround for the single-threaded constraint.

---

## `worker_threads` Module

Added in Node 10.5 (stable in 12). Each worker runs a **separate V8 isolate** with its own event loop, heap, and module scope. Workers share the process address space (unlike `cluster` or `child_process`).

### Basic Worker

```typescript
// main.ts
import { Worker, isMainThread, parentPort, workerData } from "worker_threads";

if (isMainThread) {
  const worker = new Worker(new URL("./worker.ts", import.meta.url), {
    workerData: { iterations: 1_000_000_000 },
  });

  worker.on("message", (result: number) => {
    console.log(`Result from worker: ${result}`);
  });

  worker.on("error", (err) => {
    console.error("Worker error:", err);
  });

  worker.on("exit", (code) => {
    if (code !== 0) console.error(`Worker exited with code ${code}`);
  });
} else {
  // Inside the worker
  const { iterations } = workerData as { iterations: number };
  let sum = 0;
  for (let i = 0; i < iterations; i++) {
    sum += Math.sqrt(i);
  }
  parentPort!.postMessage(sum);
}
```

### Communication: `MessageChannel` and `MessagePort`

By default workers communicate through `parentPort` (worker side) and the `Worker` instance (parent side). For more complex topologies, create explicit channels.

```typescript
import { Worker, MessageChannel } from "worker_threads";

const { port1, port2 } = new MessageChannel();

const worker = new Worker("./processor.js", {
  workerData: { port: port2 },
  transferList: [port2], // Transfer ownership — zero-copy
});

port1.on("message", (msg) => {
  console.log("Direct channel message:", msg);
});

port1.postMessage({ task: "compress", data: largeBuffer });
```

**Java parallel**: `MessageChannel` maps to `BlockingQueue<T>` — both are typed conduits between threads. The key difference: `postMessage` uses structured cloning (deep copy) by default, while Java threads share heap references directly.

### Transferable Objects (Zero-Copy)

Structured cloning copies data. For large `ArrayBuffer` instances, this is expensive. Transferring moves ownership instead — the sender loses access.

```typescript
// Parent
const buffer = new ArrayBuffer(1024 * 1024); // 1 MB
console.log(buffer.byteLength); // 1048576

worker.postMessage({ buffer }, [buffer]); // Transfer, not copy
console.log(buffer.byteLength); // 0 — neutered
```

Transferable types: `ArrayBuffer`, `MessagePort`, `ReadableStream`, `WritableStream`, `TransformStream`, `ImageBitmap` (in browser), `OffscreenCanvas`.

### Full Example: Offloading CPU-Heavy JSON Parsing

```typescript
// json-worker.ts
import { parentPort } from "worker_threads";

parentPort!.on("message", (rawJson: string) => {
  try {
    const parsed = JSON.parse(rawJson);
    // Heavy transformation
    const result = parsed.records.map((r: any) => ({
      id: r.id,
      score: computeScore(r),
    }));
    parentPort!.postMessage(result);
  } catch (err) {
    parentPort!.postMessage({ error: (err as Error).message });
  }
});

function computeScore(record: any): number {
  // Simulate CPU-intensive scoring
  let score = 0;
  for (let i = 0; i < 10_000; i++) {
    score += Math.sin(record.value * i) * Math.cos(record.weight * i);
  }
  return score;
}
```

```typescript
// main.ts
import { Worker } from "worker_threads";
import { readFile } from "fs/promises";

async function processLargeJson(filePath: string): Promise<any[]> {
  const raw = await readFile(filePath, "utf-8");

  return new Promise((resolve, reject) => {
    const worker = new Worker(new URL("./json-worker.ts", import.meta.url));
    worker.postMessage(raw);
    worker.on("message", (result) => {
      if (result.error) reject(new Error(result.error));
      else resolve(result);
    });
    worker.on("error", reject);
  });
}
```

---

## `SharedArrayBuffer` and `Atomics`

Workers normally communicate through message passing (structured clone = copy). `SharedArrayBuffer` lets multiple workers **read and write the same memory** — no copies, no messages.

### Enabling SharedArrayBuffer

Since Spectre, `SharedArrayBuffer` requires cross-origin isolation in browsers. In Node, it is available by default.

```typescript
// Shared between main thread and worker
const sharedBuffer = new SharedArrayBuffer(4); // 4 bytes = 1 Int32
const sharedArray = new Int32Array(sharedBuffer);

const worker = new Worker("./counter-worker.js", {
  workerData: { sharedBuffer },
});
```

### The Race Condition Problem

Without atomics, concurrent writes produce wrong results. This is identical to Java's non-volatile field access from multiple threads.

```typescript
// BROKEN: Race condition — both threads increment the same counter
// counter-worker.ts (run two of these)
import { workerData } from "worker_threads";

const arr = new Int32Array(workerData.sharedBuffer);
for (let i = 0; i < 1_000_000; i++) {
  arr[0] = arr[0] + 1; // Read-modify-write is NOT atomic
}
```

```typescript
// main.ts — launches two workers, expects 2,000,000
import { Worker } from "worker_threads";

const sharedBuffer = new SharedArrayBuffer(4);
const arr = new Int32Array(sharedBuffer);

function launchWorker(): Promise<void> {
  return new Promise((resolve, reject) => {
    const w = new Worker(new URL("./counter-worker.ts", import.meta.url), {
      workerData: { sharedBuffer },
    });
    w.on("exit", () => resolve());
    w.on("error", reject);
  });
}

await Promise.all([launchWorker(), launchWorker()]);
console.log(`Expected: 2000000, Got: ${arr[0]}`);
// Typical output: Expected: 2000000, Got: 1387429 (varies each run)
```

### Fixing It With `Atomics`

```typescript
// CORRECT: Atomic increment
// counter-worker-atomic.ts
import { workerData } from "worker_threads";

const arr = new Int32Array(workerData.sharedBuffer);
for (let i = 0; i < 1_000_000; i++) {
  Atomics.add(arr, 0, 1); // Atomic read-modify-write
}
```

Now the result is deterministically `2000000`.

### Key `Atomics` Operations

| Operation | Description | Java Equivalent |
|-----------|-------------|-----------------|
| `Atomics.load(arr, idx)` | Atomic read | `AtomicInteger.get()` |
| `Atomics.store(arr, idx, val)` | Atomic write | `AtomicInteger.set(val)` |
| `Atomics.add(arr, idx, val)` | Atomic add, returns old value | `AtomicInteger.getAndAdd(val)` |
| `Atomics.sub(arr, idx, val)` | Atomic subtract | No direct equivalent (use `addAndGet(-val)`) |
| `Atomics.compareExchange(arr, idx, expected, replacement)` | CAS — returns old value | `AtomicInteger.compareAndSet(expected, update)` |
| `Atomics.wait(arr, idx, val)` | Block until `arr[idx] !== val` | `LockSupport.park()` / `Object.wait()` |
| `Atomics.notify(arr, idx, count)` | Wake waiting threads | `LockSupport.unpark()` / `Object.notify()` |
| `Atomics.exchange(arr, idx, val)` | Atomic swap, returns old value | `AtomicInteger.getAndSet(val)` |

### CAS (Compare-and-Swap) Spin Lock

```typescript
// Simple spin lock using CAS — same concept as Java's AbstractQueuedSynchronizer
const UNLOCKED = 0;
const LOCKED = 1;

function acquire(lock: Int32Array, idx: number): void {
  while (Atomics.compareExchange(lock, idx, UNLOCKED, LOCKED) !== UNLOCKED) {
    // Spin — in practice, use Atomics.wait for blocking
  }
}

function release(lock: Int32Array, idx: number): void {
  Atomics.store(lock, idx, UNLOCKED);
  Atomics.notify(lock, idx, 1); // Wake one waiter
}
```

### `Atomics.wait` / `Atomics.notify` (Producer-Consumer)

```typescript
// Worker: wait for signal from main thread
const signal = new Int32Array(workerData.sharedBuffer);

// Block until signal[0] is no longer 0 (timeout after 5s)
const result = Atomics.wait(signal, 0, 0, 5000);
// result: "ok" | "not-equal" | "timed-out"

if (result === "ok") {
  const value = Atomics.load(signal, 0);
  console.log(`Received signal with value: ${value}`);
}
```

```typescript
// Main thread: signal the worker
// NOTE: Atomics.wait() cannot be called on the main thread — it would block the event loop.
// Use Atomics.waitAsync() (Node 16+) on the main thread instead.
Atomics.store(signal, 0, 42);
Atomics.notify(signal, 0, 1);
```

**Important**: `Atomics.wait()` **blocks the calling thread**. Never call it on the main thread — it will freeze the event loop. Use `Atomics.waitAsync()` (returns a Promise) on the main thread.

---

## Worker Thread Pool Pattern

Spawning a worker per task is expensive (~30-50ms startup, separate V8 heap allocation). For repeated CPU work, pool workers and reuse them.

### DIY Minimal Pool

```typescript
import { Worker } from "worker_threads";
import { EventEmitter } from "events";

interface PoolTask<T> {
  data: unknown;
  resolve: (value: T) => void;
  reject: (reason: Error) => void;
}

class WorkerPool<T = unknown> {
  private readonly workers: Worker[] = [];
  private readonly freeWorkers: Worker[] = [];
  private readonly queue: PoolTask<T>[] = [];

  constructor(
    private readonly workerPath: string | URL,
    private readonly size: number,
  ) {
    for (let i = 0; i < size; i++) {
      const worker = new Worker(workerPath);
      worker.on("message", (result: T) => {
        const task = (worker as any).__currentTask as PoolTask<T>;
        task.resolve(result);
        this.freeWorkers.push(worker);
        this.drain();
      });
      worker.on("error", (err) => {
        const task = (worker as any).__currentTask as PoolTask<T>;
        task.reject(err);
        this.freeWorkers.push(worker);
        this.drain();
      });
      this.workers.push(worker);
      this.freeWorkers.push(worker);
    }
  }

  exec(data: unknown): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push({ data, resolve, reject });
      this.drain();
    });
  }

  private drain(): void {
    while (this.queue.length > 0 && this.freeWorkers.length > 0) {
      const worker = this.freeWorkers.pop()!;
      const task = this.queue.shift()!;
      (worker as any).__currentTask = task;
      worker.postMessage(task.data);
    }
  }

  async destroy(): Promise<void> {
    await Promise.all(this.workers.map((w) => w.terminate()));
  }
}
```

### Use `piscina` Instead

In practice, use a battle-tested pool. `piscina` (Latin for "pool") is maintained by the Node.js team members.

```bash
npm install piscina
```

```typescript
// main.ts
import Piscina from "piscina";

const pool = new Piscina({
  filename: new URL("./math-worker.ts", import.meta.url).href,
  maxThreads: 4,
  minThreads: 2,
  idleTimeout: 30_000, // Shut down idle threads after 30s
});

// Submit tasks — returns a Promise
const results = await Promise.all([
  pool.run({ n: 1_000_000 }),
  pool.run({ n: 2_000_000 }),
  pool.run({ n: 3_000_000 }),
]);

console.log(results);
```

```typescript
// math-worker.ts
export default function ({ n }: { n: number }): number {
  let sum = 0;
  for (let i = 0; i < n; i++) {
    sum += Math.sqrt(i);
  }
  return sum;
}
```

### When to Pool vs Spawn One-Off

| Scenario | Strategy |
|----------|----------|
| Repeated CPU tasks (image resize, hashing) | Pool — amortize startup cost |
| One-time heavy computation at startup | One-off worker — no pool overhead |
| Low-frequency, high-latency tasks | Pool with `minThreads: 0`, `idleTimeout` |
| Thousands of concurrent micro-tasks | Pool with bounded queue to apply backpressure |

**Java parallel**: `piscina` is the direct equivalent of `ExecutorService.newFixedThreadPool(n)`. The `exec()` method returns a `Promise<T>` — analogous to `executor.submit(callable)` returning a `Future<T>`. The idle timeout maps to `ThreadPoolExecutor.setKeepAliveTime()`.

---

## `cluster` Module

`cluster` forks the entire Node process. Each fork is a **separate OS process** with its own V8 heap, event loop, and memory space. No shared memory.

### Basic Cluster

```typescript
import cluster from "cluster";
import http from "http";
import { availableParallelism } from "os";

const numCPUs = availableParallelism();

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on("exit", (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (${signal || code}). Restarting...`);
    cluster.fork(); // Auto-restart
  });
} else {
  http
    .createServer((req, res) => {
      res.writeHead(200);
      res.end(`Handled by worker ${process.pid}\n`);
    })
    .listen(8000);

  console.log(`Worker ${process.pid} started`);
}
```

### Load Balancing Strategies

| Platform | Default Strategy | How It Works |
|----------|-----------------|--------------|
| Linux | Round-robin | Primary accepts connections, distributes to workers in order |
| macOS / Windows | OS-level | All workers compete for `accept()` — can cause thundering herd |

```typescript
// Force round-robin on all platforms
cluster.schedulingPolicy = cluster.SCHED_RR;
```

### PM2 Cluster Mode

In production, use PM2 instead of hand-rolling cluster logic.

```bash
# Start 4 instances in cluster mode
pm2 start server.js -i 4

# Use all available CPUs
pm2 start server.js -i max

# Zero-downtime restart
pm2 reload server.js
```

PM2 handles worker restarts, log aggregation, and monitoring. It also supports `ecosystem.config.js` for declarative configuration.

**Java parallel**: `cluster.fork()` maps to running multiple JVM processes behind a load balancer (e.g., multiple Spring Boot instances behind nginx). Java does not need this pattern as often because a single JVM can use all CPU cores via platform threads or virtual threads.

---

## `child_process` Module

Spawns entirely separate OS processes. Unlike `worker_threads` (same process, different V8 isolate) or `cluster` (same code, forked process), `child_process` runs **any executable**.

### The Four Methods

| Method | Shell | Buffering | IPC | Best For |
|--------|-------|-----------|-----|----------|
| `spawn(cmd, args)` | No | Stream | Optional | Long-running processes, large output |
| `exec(cmd)` | Yes | Buffered | No | Shell commands, small output |
| `execFile(file, args)` | No | Buffered | No | Direct binary execution |
| `fork(module)` | No | Stream | **Yes (built-in)** | Node-to-Node IPC |

### `spawn` — Streaming Output

```typescript
import { spawn } from "child_process";

const ffmpeg = spawn("ffmpeg", ["-i", "input.mp4", "-codec", "copy", "output.mkv"]);

ffmpeg.stdout.on("data", (data) => console.log(`stdout: ${data}`));
ffmpeg.stderr.on("data", (data) => console.error(`stderr: ${data}`));
ffmpeg.on("close", (code) => console.log(`ffmpeg exited with code ${code}`));
```

### `exec` — Buffered Shell Command

```typescript
import { exec } from "child_process";
import { promisify } from "util";

const execAsync = promisify(exec);
const { stdout, stderr } = await execAsync("ls -la /tmp");
console.log(stdout);
```

**Warning**: `exec` runs through a shell. Never interpolate user input — shell injection risk.

### `fork` — Node-to-Node IPC

`fork` is specifically for spawning another Node.js module with a built-in IPC channel.

```typescript
// parent.ts
import { fork } from "child_process";

const child = fork("./compute.js");

child.send({ task: "fibonacci", n: 45 });

child.on("message", (result) => {
  console.log("Result:", result);
});
```

```typescript
// compute.ts
process.on("message", (msg: { task: string; n: number }) => {
  if (msg.task === "fibonacci") {
    process.send!(fibonacci(msg.n));
  }
});

function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
```

### When to Use `child_process` Over `worker_threads`

- Running **non-Node executables** (ffmpeg, Python scripts, system tools)
- Need **full process isolation** (crash in child does not affect parent)
- Running a **different Node version** (via `execPath` option)
- Legacy code that uses `process.exit()` liberally

---

## Decision Matrix

| Dimension | `worker_threads` | `cluster` | `child_process` |
|-----------|-------------------|-----------|------------------|
| **Isolation** | Separate V8 isolate, same process | Separate OS process | Separate OS process |
| **Shared memory** | Yes (`SharedArrayBuffer`) | No | No |
| **Communication** | `postMessage` (structured clone or transfer) | IPC (JSON serialization) | IPC / stdio / pipe |
| **Startup cost** | ~30-50ms | ~100-300ms (full process fork) | ~50-200ms (depends on executable) |
| **Memory overhead** | ~10-30 MB per worker (new V8 heap) | Full process duplicate (~50-100 MB) | Full process |
| **Use case** | CPU-heavy computation in Node | Horizontal scaling of HTTP servers | External commands, full isolation |
| **Can run non-JS** | No | No | Yes |
| **Crash containment** | Worker crash can be caught | Worker crash = process death | Child crash = independent |
| **When to reach for it** | Parsing, hashing, image processing, ML inference | Multi-core HTTP serving | Shell commands, FFmpeg, Python |

### Quick Decision Guide

```
Need parallelism for CPU work in JS?
  └─ Yes → worker_threads (or piscina pool)

Need to scale an HTTP server across cores?
  └─ Yes → cluster (or PM2 cluster mode)

Need to run an external binary or script?
  └─ Yes → child_process.spawn

Need full crash isolation for untrusted code?
  └─ Yes → child_process.fork with sandbox

Need shared memory between threads?
  └─ Yes → worker_threads + SharedArrayBuffer
```

---

## Java Parallels

This section maps Node concurrency primitives to their Java counterparts for developers who think in JVM terms.

| Node.js | Java | Notes |
|---------|------|-------|
| `worker_threads` Worker | `ExecutorService` + `Callable<T>` | Worker returns result via message; Callable returns via `Future.get()` |
| `piscina` pool | `Executors.newFixedThreadPool(n)` | Both pool threads and dispatch tasks from a queue |
| `pool.run(data)` → `Promise<T>` | `executor.submit(callable)` → `Future<T>` | Identical semantics: submit work, await result |
| `SharedArrayBuffer` | Shared heap (default in Java) | Java threads share heap by default; JS workers are isolated by default |
| `Atomics.add` / `Atomics.compareExchange` | `AtomicInteger.getAndAdd()` / `compareAndSet()` | Same hardware CAS instructions underneath |
| `Atomics.wait` / `Atomics.notify` | `LockSupport.park()` / `unpark()` | Both block the calling thread until signaled |
| `MessageChannel` / `MessagePort` | `BlockingQueue<T>` | Typed inter-thread communication channel |
| `cluster.fork()` | Multiple JVM processes | Java rarely needs this — one JVM uses all cores |
| `child_process.spawn` | `ProcessBuilder` / `Runtime.exec()` | Both spawn OS-level processes |
| `child_process.fork` (Node IPC) | No direct equivalent | Java processes communicate via sockets, not built-in IPC |
| `postMessage` (structured clone) | Serialize + deserialize across threads | Java avoids this because threads share heap |
| `transferList` (zero-copy) | Direct `ByteBuffer` hand-off | Conceptually similar: ownership transfer without copying |

### The Fundamental Difference

Java threads share the heap by default. Concurrent access to shared objects is the norm, guarded by `synchronized`, `volatile`, or `java.util.concurrent` utilities.

Node workers are **isolated by default**. Sharing is opt-in via `SharedArrayBuffer`, and even then only for typed arrays of primitives. You cannot share arbitrary JS objects between workers.

This means:
- Java developers fight **accidental sharing** (data races, visibility bugs)
- Node developers fight **serialization overhead** (structured clone cost)

With Java 21's virtual threads, the gap narrows further: Java gets lightweight concurrency without callbacks, while Node still requires `worker_threads` for CPU parallelism even with `async/await` handling I/O concurrency well.

---

## Common Pitfalls

### 1. Using Workers for I/O-Bound Work

Workers add overhead. If the work is I/O-bound, `async/await` on the main thread is faster.

```typescript
// BAD: Worker for HTTP fetch
const worker = new Worker("./fetch-worker.js");
worker.postMessage({ url: "https://api.example.com/data" });

// GOOD: Just use fetch on the main thread
const data = await fetch("https://api.example.com/data");
```

### 2. Forgetting to Handle Worker Errors

An unhandled error in a worker terminates the worker silently. Always attach an `error` listener.

### 3. Blocking the Main Thread With `Atomics.wait`

`Atomics.wait()` blocks synchronously. On the main thread, this freezes the entire event loop. Use `Atomics.waitAsync()` on the main thread.

### 4. Transferring and Then Accessing the Buffer

After transferring an `ArrayBuffer`, it is **neutered** — zero bytes, unusable. The sender must not touch it after transfer.

### 5. Large `workerData` Payloads

`workerData` is structured-cloned at worker creation. Passing large objects here blocks the main thread during cloning. Prefer `SharedArrayBuffer` or stream data via `postMessage` after creation.

---

## Related

- [Event Loop Internals](event-loop-internals.md) — the single-threaded scheduling model worker threads complement rather than replace.
- [Java Concurrency Basics for TypeScript Developers](../../java/java-fundamentals/concurrency-basics.md) — shared memory, executors, `CompletableFuture`, and atomics on the JVM.
- [Java Multithreading Deep Dive](../../java/java-fundamentals/multithreading-deep-dive.md) — locks, synchronizers, and thread-pool internals beyond the basics.
- [Virtual Threads in Java](../../java/java-fundamentals/virtual-threads.md) — Java's answer to large-scale I/O concurrency without callback-heavy code.

## References

- [Node.js `worker_threads` documentation](https://nodejs.org/api/worker_threads.html)
- [Node.js `cluster` documentation](https://nodejs.org/api/cluster.html)
- [Node.js `child_process` documentation](https://nodejs.org/api/child_process.html)
- [MDN SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)
- [MDN Atomics](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics)
- [`piscina` GitHub repository](https://github.com/piscinajs/piscina)
- [V8 blog: Shared Array Buffers](https://v8.dev/blog/shared-array-buffer)
- [libuv thread pool documentation](https://docs.libuv.org/en/v1.x/threadpool.html)
