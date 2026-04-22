# Event Loop Internals

> Not the "setTimeout goes to the callback queue" blog-post version — the actual libuv implementation that determines execution order in Node.js.

Node.js delegates I/O to libuv, a C library that provides an event loop with **six distinct phases**. Each phase has a FIFO queue of callbacks. When the loop enters a phase, it drains that queue (up to a system-dependent limit), then moves to the next phase. Separately from those six phases, Node also drains the `process.nextTick` queue and then the microtask queue at callback boundaries.

---

## The Six libuv Phases

```
   ┌───────────────────────────┐
┌─►│         timers             │  setTimeout, setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │    pending callbacks       │  Deferred I/O callbacks (TCP ECONNREFUSED, etc.)
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │      idle, prepare         │  Internal use only (libuv bookkeeping)
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │          poll              │  Retrieve new I/O events; execute I/O callbacks
│  │                            │  (almost everything except close, timers, check)
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │          check             │  setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │     close callbacks        │  socket.on('close'), etc.
│  └─────────────┬─────────────┘
│                │
│                │
│       after callbacks: drain `nextTick`,
│       then drain microtasks
│                │
└────────────────┘
```

### Timers Phase

Executes callbacks scheduled by `setTimeout()` and `setInterval()`. The loop checks a min-heap of timers sorted by expiry time. It fires every callback whose threshold has elapsed. **Timer thresholds are a minimum, not a guarantee** — I/O or other callbacks can delay execution past the requested time.

```typescript
// Timer fires after *at least* 100ms, not exactly 100ms
setTimeout(() => {
  console.log("Timer fired");
}, 100);
```

Internally, libuv calls `uv__run_timers()` which walks the min-heap and fires expired entries.

### Pending Callbacks Phase

Some system-level callbacks are deferred from the previous loop iteration. Examples:

- TCP connection errors (`ECONNREFUSED`)
- Write errors on streams
- DNS lookup errors from `c-ares`

These did not run in the poll phase of the previous tick because libuv deferred them. They run here via `uv__run_pending()`.

### Idle / Prepare Phase

Used exclusively by libuv internals. You cannot schedule work here from JavaScript. The `idle` handle fires on every loop iteration; the `prepare` handle fires just before polling for I/O. Node uses these for internal bookkeeping like V8 garbage collection scheduling.

### Poll Phase

This is where the loop spends most of its time. The poll phase does two things:

1. **Calculates how long it should block for I/O.** It looks at pending timers to decide the maximum block time. If there are no timers, it may block indefinitely (until I/O arrives).
2. **Processes I/O events in the poll queue.** File reads, network responses, incoming connections — their callbacks fire here.

The poll phase has a hard cap on iterations (system-dependent, typically 1024 on Linux via `epoll`, 64 on macOS via `kqueue`) to prevent starvation of other phases.

If the poll queue is empty:
- If `setImmediate` callbacks are pending, the loop moves to the **check** phase.
- If no `setImmediate` callbacks exist, the loop waits for I/O callbacks to be added, then executes them immediately.

### Check Phase

Executes `setImmediate()` callbacks. This phase exists specifically so you can schedule work that runs after the poll phase completes, before timers are checked again.

```typescript
setImmediate(() => {
  console.log("Check phase callback");
});
```

### Close Callbacks Phase

Fires callbacks from handles closed via `handle.close()`. For example:

```typescript
const server = net.createServer();
server.on("close", () => {
  console.log("Server closed"); // Fires in close callbacks phase
});
server.close();
```

If a socket or handle is destroyed abruptly, its close event fires here, allowing cleanup to happen in a well-defined phase.

---

## Microtask Queue

Microtasks include `Promise.then()`, `Promise.catch()`, `Promise.finally()`, and `queueMicrotask()`. Since Node.js 11, microtasks drain **between every callback** within a phase, not just between phases. This aligns Node with browser behavior.

### Execution Model

```
Phase callback 1 → drain nextTick → drain microtasks →
Phase callback 2 → drain nextTick → drain microtasks →
... until phase queue is empty →
Move to next phase
```

### Starvation via Recursive Microtasks

Microtasks that enqueue more microtasks will keep the microtask queue non-empty. The loop never moves on because it must drain microtasks before proceeding.

```typescript
// WARNING: This starves the event loop. Nothing else ever runs.
function recurse(): void {
  queueMicrotask(() => {
    console.log("Microtask");
    recurse(); // Enqueues another microtask before the queue can drain
  });
}
recurse();

// setTimeout below will NEVER fire
setTimeout(() => {
  console.log("This never prints");
}, 0);
```

This is identical in effect to an infinite synchronous loop. The event loop is blocked in the microtask drain step and never returns to any phase.

---

## `process.nextTick` vs `queueMicrotask`

Both run between phases, but they use **separate queues** with a strict priority order:

```
After each callback:
  1. Drain the entire nextTick queue (including newly added entries)
  2. Drain the entire microtask queue (including newly added entries)
  3. Return to the current phase's next callback (or move to next phase)
```

### Priority Demonstration

```typescript
console.log("1: script start");

process.nextTick(() => {
  console.log("3: nextTick 1");
});

Promise.resolve().then(() => {
  console.log("4: microtask 1");
});

process.nextTick(() => {
  console.log("5: nextTick 2");
});

Promise.resolve().then(() => {
  console.log("6: microtask 2");
});

console.log("2: script end");

// Output:
// 1: script start
// 2: script end
// 3: nextTick 1
// 5: nextTick 2
// 4: microtask 1
// 6: microtask 2
```

All `nextTick` callbacks fire before any microtask callbacks, even though they were interleaved in the source.

### Nested nextTick Starvation

```typescript
// WARNING: Starves the event loop just like recursive microtasks
function tickBomb(): void {
  process.nextTick(() => {
    console.log("tick");
    tickBomb(); // The nextTick queue never empties
  });
}
tickBomb();

// I/O, timers, setImmediate — none of these will ever run
```

`process.nextTick` is more dangerous than `queueMicrotask` for starvation because it runs first. A recursive `nextTick` blocks both the microtask queue and all event loop phases.

### When to Use Each

| Use case | Choose |
|----------|--------|
| Ensure callback runs before any I/O | `process.nextTick` |
| Interop with promise-based APIs | `queueMicrotask` |
| Allow I/O between iterations | `setImmediate` |
| General async deferral | `queueMicrotask` (preferred over `nextTick`) |

The Node.js docs recommend `queueMicrotask` for most cases. Reserve `nextTick` for low-level library code that must run before I/O (e.g., emitting events synchronously after construction but before the user attaches listeners).

---

## `setTimeout` vs `setImmediate`

### Non-deterministic in the Main Module

```typescript
// Order is NOT guaranteed in the top-level script
setTimeout(() => {
  console.log("timeout");
}, 0);

setImmediate(() => {
  console.log("immediate");
});

// Sometimes: timeout → immediate
// Sometimes: immediate → timeout
```

Why? `setTimeout(fn, 0)` is clamped to 1ms internally. When the loop starts, whether the 1ms has already elapsed depends on system clock resolution and how long it took to set up the loop. If the timer has expired, `timers` phase fires it first. If not, the loop passes timers with nothing to do, hits poll (empty), sees a pending `setImmediate`, and moves to check.

### Deterministic Inside an I/O Callback

```typescript
import { readFile } from "fs";

readFile(__filename, () => {
  // Inside an I/O callback, we are in the poll phase.
  // When poll finishes, the loop moves to check (setImmediate)
  // BEFORE looping back to timers.

  setTimeout(() => {
    console.log("2: timeout");
  }, 0);

  setImmediate(() => {
    console.log("1: immediate");
  });
});

// Output is ALWAYS:
// 1: immediate
// 2: timeout
```

After the poll phase callback runs, the loop proceeds to the **check** phase (where `setImmediate` fires), then close callbacks, then loops back to timers. The `setTimeout` callback cannot fire until the next timers phase.

---

## Starvation Scenarios

### CPU-Bound Synchronous Work

```typescript
// Blocks the entire loop for ~5 seconds
function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

const server = http.createServer((req, res) => {
  // Every request blocks all other connections
  const result = fibonacci(45);
  res.end(`${result}`);
});
```

No I/O, timers, or callbacks can fire while `fibonacci` runs on the main thread.

### Detecting Event Loop Lag

Node.js provides `monitorEventLoopDelay()` (available since v11.10) to measure how long the loop is blocked:

```typescript
import { monitorEventLoopDelay } from "perf_hooks";

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();

setInterval(() => {
  // Values in nanoseconds
  const min = histogram.min / 1e6;
  const max = histogram.max / 1e6;
  const mean = histogram.mean / 1e6;
  const p99 = histogram.percentile(99) / 1e6;

  console.log(`Event loop delay — min: ${min.toFixed(1)}ms, ` +
    `max: ${max.toFixed(1)}ms, mean: ${mean.toFixed(1)}ms, ` +
    `p99: ${p99.toFixed(1)}ms`);

  histogram.reset();
}, 5000);
```

A healthy loop shows p99 under 20ms. Spikes above 100ms indicate blocking work.

### Fixing Starvation

**Chunking with `setImmediate`** — yield control back to the loop between chunks:

```typescript
function processLargeArray(
  items: number[],
  index: number,
  callback: (results: number[]) => void,
  results: number[] = []
): void {
  const CHUNK_SIZE = 1000;
  const end = Math.min(index + CHUNK_SIZE, items.length);

  for (let i = index; i < end; i++) {
    results.push(items[i] * 2); // Some CPU work per item
  }

  if (end < items.length) {
    // Yield to the event loop — I/O and timers can run between chunks
    setImmediate(() => processLargeArray(items, end, callback, results));
  } else {
    callback(results);
  }
}
```

**Worker Threads** — offload heavy computation entirely:

```typescript
import { Worker, isMainThread, parentPort, workerData } from "worker_threads";

if (isMainThread) {
  const server = http.createServer((req, res) => {
    const worker = new Worker(__filename, { workerData: { n: 45 } });
    worker.on("message", (result) => res.end(`${result}`));
    worker.on("error", (err) => res.writeHead(500).end(err.message));
  });
  server.listen(3000);
} else {
  function fibonacci(n: number): number {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
  }
  parentPort?.postMessage(fibonacci(workerData.n));
}
```

Each worker has its own V8 isolate and event loop. The main thread's loop stays responsive.

**Summary of starvation sources and fixes:**

| Source | Mechanism | Fix |
|--------|-----------|-----|
| Synchronous CPU work | Blocks the call stack | Worker threads or chunking with `setImmediate` |
| Recursive `process.nextTick` | nextTick queue never drains | Refactor to `setImmediate` |
| Recursive microtasks | Microtask queue never drains | Refactor to `setImmediate` |
| Long synchronous I/O | `fs.readFileSync` in hot path | Use async `fs.readFile` |
| JSON.parse on huge payloads | Blocks during parsing | Stream with `JSONStream` or worker threads |

---

## Event Loop and async/await

### How await Desugars

`async/await` is syntactic sugar over promises. Every `await` inserts a microtask boundary:

```typescript
async function example(): Promise<void> {
  console.log("1: before await");
  await someAsyncOp();
  console.log("3: after await"); // This is a microtask callback
}

// Desugars roughly to:
function example(): Promise<void> {
  console.log("1: before await");
  return someAsyncOp().then(() => {
    console.log("3: after await");
  });
}
```

Everything after `await` is scheduled as a microtask. The function yields control at the `await` point.

### Sequential vs Concurrent: for-await vs Promise.all

```typescript
// SEQUENTIAL — each operation waits for the previous one
async function sequential(urls: string[]): Promise<string[]> {
  const results: string[] = [];
  for (const url of urls) {
    const response = await fetch(url); // Blocks iteration
    const text = await response.text();
    results.push(text);
  }
  return results;
}
// Total time ≈ sum of all request durations

// CONCURRENT — all operations start immediately
async function concurrent(urls: string[]): Promise<string[]> {
  const promises = urls.map(async (url) => {
    const response = await fetch(url);
    return response.text();
  });
  return Promise.all(promises);
}
// Total time ≈ duration of the slowest request
```

### Execution Order with Interleaved Awaits

```typescript
async function a(): Promise<void> {
  console.log("1: a start");
  await Promise.resolve();
  console.log("4: a after first await");
  await Promise.resolve();
  console.log("6: a after second await");
}

async function b(): Promise<void> {
  console.log("2: b start");
  await Promise.resolve();
  console.log("5: b after first await");
  await Promise.resolve();
  console.log("7: b after second await");
}

console.log("0: script start");
a();
b();
console.log("3: script end");

// Output:
// 0: script start
// 1: a start
// 2: b start
// 3: script end
// 4: a after first await
// 5: b after first await
// 6: a after second await
// 7: b after second await
```

Both `a()` and `b()` start synchronously. Each `await` yields to the microtask queue. The continuations interleave because they both land in the microtask queue in the order they were awaited.

### for-await-of with Async Iterables

```typescript
async function* generateNumbers(): AsyncGenerator<number> {
  for (let i = 0; i < 3; i++) {
    await new Promise((resolve) => setTimeout(resolve, 100));
    yield i;
  }
}

// Each iteration awaits the next value — inherently sequential
for await (const num of generateNumbers()) {
  console.log(num); // 0, then 100ms, 1, then 100ms, 2
}
```

`for-await-of` calls `.next()` on the iterator only after the previous value resolves. There is no way to "parallelize" an async iterator — it is sequential by design.

---

## Java Parallel: Thread-per-Request vs Event Loop

### Mental Model Mapping

| Java Concept | Node.js Equivalent | Key Difference |
|---|---|---|
| `ExecutorService` thread pool | libuv event loop + thread pool | Java: multiple OS threads executing tasks. Node: single JS thread + libuv pool for I/O |
| Platform thread | Main thread | Java can have hundreds. Node has exactly one for JS |
| Virtual thread (Loom) | async callback / Promise | Both avoid blocking an OS thread during I/O waits |
| `CompletableFuture` | `Promise` | Nearly identical API semantics. Java: `thenApply`/`thenCompose`. Node: `then`/`then` returning Promise |
| `CompletableFuture.allOf()` | `Promise.all()` | Same fan-out-then-join pattern |
| `synchronized` / `ReentrantLock` | Not needed | Single-threaded JS — no data races on shared state by construction |
| `BlockingQueue` | Streams / async iterables | Java blocks a thread. Node yields to the loop |
| `Thread.sleep(1000)` | `await setTimeout(1000)` | Java: blocks the thread. Node: yields, loop serves other work |

### Where Each Model Wins

**Node event loop wins when:**
- Workload is I/O-bound (HTTP APIs, database proxies, real-time servers)
- Thousands of concurrent connections with small payloads
- You want low memory per connection (~2KB for a callback vs ~1MB for a Java platform thread stack)
- Streaming and backpressure are first-class concerns

**Java thread-per-request wins when:**
- Workload mixes CPU and I/O (each request does significant computation)
- You need true parallelism across CPU cores without IPC overhead
- Debugging sequential blocking code is simpler than tracing callback chains
- Ecosystem expects synchronous APIs (JDBC, legacy libraries)

**Java virtual threads (Project Loom) bridge the gap:**
- Look like blocking code (`Thread.sleep`, blocking I/O) but yield the carrier thread under the hood
- Achieve Node-like scalability (millions of virtual threads) with Java's sequential programming model
- No colored function problem — `async/await` is implicit
- Limitations: `synchronized` blocks can pin the carrier thread; not all native libraries are Loom-aware

### Code Comparison

**Java (Virtual Threads):**
```java
// Looks blocking, but virtual thread yields during I/O
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var future1 = executor.submit(() -> httpClient.send(request1, bodyHandler));
    var future2 = executor.submit(() -> httpClient.send(request2, bodyHandler));

    var response1 = future1.get(); // Yields virtual thread, not OS thread
    var response2 = future2.get();
}
```

**Node.js (async/await):**
```typescript
// Explicit async — every await yields to the event loop
const [response1, response2] = await Promise.all([
  fetch(url1),
  fetch(url2),
]);
```

Both achieve concurrent I/O without blocking OS threads. The difference is syntactic: Java hides the yielding, Node makes it explicit with `await`.

### The Colored Function Problem

In Node/TypeScript, functions are "colored" — sync functions cannot call async functions without becoming async themselves. This forces `async` to propagate up the call stack:

```typescript
// Once any function in the chain is async, everything above must be async
async function getUser(id: string): Promise<User> { /* ... */ }
async function enrichUser(id: string): Promise<EnrichedUser> {
  const user = await getUser(id); // Must await
  return enrich(user);
}
async function handleRequest(req: Request): Promise<Response> {
  const user = await enrichUser(req.params.id); // Must await
  return new Response(JSON.stringify(user));
}
```

Java virtual threads avoid this entirely. A virtual thread can call `Thread.sleep()` or blocking I/O and the runtime handles the yielding transparently. No function signature changes needed.

---

## Putting It All Together: Complete Execution Order

```typescript
import { readFile } from "fs";

console.log("1: script start");

setTimeout(() => {
  console.log("8: setTimeout");
  Promise.resolve().then(() => console.log("9: promise inside setTimeout"));
}, 0);

setImmediate(() => {
  console.log("10: setImmediate");
});

readFile(__filename, () => {
  console.log("6: readFile callback (poll phase)");

  setImmediate(() => {
    console.log("7: setImmediate inside I/O");
  });

  process.nextTick(() => {
    console.log("11: nextTick inside I/O");  // Fires before setImmediate
  });
});

process.nextTick(() => {
  console.log("3: nextTick 1");
});

Promise.resolve().then(() => {
  console.log("4: promise 1");
});

process.nextTick(() => {
  console.log("5: nextTick 2");
});

console.log("2: script end");

// Guaranteed order for lines 1-5:
// 1: script start
// 2: script end
// 3: nextTick 1
// 5: nextTick 2
// 4: promise 1
//
// Lines 6-11 depend on I/O timing. Typical order:
// 8: setTimeout
// 9: promise inside setTimeout
// 6: readFile callback (poll phase)
// 11: nextTick inside I/O
// 7: setImmediate inside I/O
// 10: setImmediate
```

Note: Lines 6-11 are non-deterministic relative to each other because `readFile` completion time is unpredictable. The **relative** ordering within each group (nextTick before microtasks, setImmediate after I/O callback) is deterministic.

---

## References

- [Node.js Event Loop Documentation](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick) — Official guide to timers, `nextTick`, and phases
- [libuv Design Overview](https://docs.libuv.org/en/v1.x/design.html) — The C library architecture behind Node's event loop
- [libuv Source: `uv_run`](https://github.com/libuv/libuv/blob/v1.x/src/unix/core.c) — The actual loop implementation in C
- [Node.js `perf_hooks.monitorEventLoopDelay`](https://nodejs.org/api/perf_hooks.html#perf_hooks_monitorloopdelay_options) — API for measuring event loop lag
- [Node.js Worker Threads](https://nodejs.org/api/worker_threads.html) — Official worker threads documentation
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444) — Project Loom specification for Java virtual threads
