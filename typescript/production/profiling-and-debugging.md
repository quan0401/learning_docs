# Node.js Profiling & Debugging

Your API's p99 latency just spiked from 50ms to 1.2s. Users are timing out. The dashboard shows CPU is fine, memory looks okayish, and there are no obvious errors in the logs. Where do you even start? If you have a Java background, you already know the drill: attach JFR, take a thread dump, fire up VisualVM. Node has equivalent tools, but they look completely different. This doc maps the Java profiling mental model onto the Node/V8 ecosystem and walks through every major technique.

Cross-references: [V8 Memory & Garbage Collection](../runtime/v8-memory-and-gc.md) for heap structure and GC internals, [Event Loop Internals](../runtime/event-loop-internals.md) for understanding where lag comes from, [Worker Threads & Concurrency](../runtime/worker-threads.md) for offloading CPU work.

---

## CPU Profiling with `--prof`

The lowest-level CPU profiling tool built into V8. No npm packages, no external tools. It generates a tick log showing where V8 spent time at the function level. This is the Node equivalent of `async-profiler` or JFR CPU profiling.

### Full Workflow

```bash
# Step 1 — Generate the tick log
node --prof dist/server.js

# Step 2 — Generate load in another terminal
npx autocannon -c 50 -d 10 http://localhost:3000/api/reports

# Step 3 — Stop the server (Ctrl+C), process the log
node --prof-process isolate-0x*.log > processed-profile.txt
```

**Step 4 — Read the output:**

```text
 [JavaScript]:
   ticks  total  nonlib   name
   3842   42.1%   55.2%  LazyCompile: *processRow /app/dist/server.js:142:20
   1205   13.2%   17.3%  LazyCompile: *validateSchema /app/dist/server.js:89:25
    892    9.8%   12.8%  LazyCompile: *JSON.parse

 [C++]:
   ticks  total  nonlib   name
    512    5.6%    7.4%  v8::internal::Runtime_StringCharCodeAt

 [GC]:
    234    2.6%

 [Summary]:
   6951   76.2%  100.0%  JavaScript
   1432   15.7%          C++
    234    2.6%           GC
    512    5.6%           Shared libraries
```

### Reading the Output

| Section | What It Tells You |
|---------|-------------------|
| **JavaScript** | Time in your JS code and libraries. `*` = TurboFan-optimized, `~` = Ignition interpreter |
| **C++** | V8 internals and native bindings. High C++ in string ops = inefficient concatenation |
| **GC** | Garbage collection time. >5% indicates memory pressure |
| **Summary** | Healthy: 70-90% JS, <10% C++, <5% GC |

---

## Chrome DevTools Profiling

More visual than `--prof`, and you can interact with it live. The closest equivalent to connecting VisualVM to a running JVM.

```bash
node --inspect dist/server.js
# Output: Debugger listening on ws://127.0.0.1:9229/...
```

Open `chrome://inspect` in Chrome. Your Node process appears under "Remote Target." Click **inspect**.

**CPU Profiler:** Open **Performance** tab, click **Record**, trigger the slow operation, click **Stop**. The flame chart reads top-to-bottom (call stack) and left-to-right (time). Wide bars = hot functions. Yellow = JavaScript, Purple = system/native.

### Connecting to a Remote Node Process (Containers)

**kubectl port-forward (Kubernetes):**

```bash
# Dockerfile starts node with --inspect=0.0.0.0:9229
kubectl port-forward pod/api-server-abc123 9229:9229
# chrome://inspect now sees the process
```

**SSH tunnel (VMs or docker-compose):**

```bash
# Remote host starts with --inspect=127.0.0.1:9229
ssh -L 9229:127.0.0.1:9229 user@remote-host
```

**Security warning:** The V8 inspector gives full code execution access. In production, always bind to `127.0.0.1` and tunnel. Never expose port 9229 to the network.

---

## Flame Graphs

The x-axis is **not** time — it is the set of stack frames sorted by sample count. Width = proportion of samples where that function was on the stack. Wider = more time spent.

```
┌─────────────────────────────────────────────────────────────┐
│                         (root)                               │
├────────────────────────────────┬────────────────────────────┤
│       handleRequest            │       processQueue          │
├───────────────┬────────────────┤────────────────────────────┤
│  parseBody    │ validateInput  │      transformRecord        │
├───────────────┤────────┬───────┤──────────┬─────────────────┤
│  JSON.parse   │ regex  │ zod   │ encrypt  │ serializeOutput │
└───────────────┘────────┘───────┘──────────┘─────────────────┘
```

- **Bottom** = entry points (root, handleRequest). **Top** = leaf functions doing actual work.
- Wide bar at bottom branching into narrow children = dispatcher overhead.
- Wide bar at top = single hot function.

### Tools

```bash
# 0x — zero-overhead flamegraph generator
npx 0x dist/server.js
# Generate load, Ctrl+C → opens interactive flame graph in browser

# clinic flame — integrated with clinic suite
npx clinic flame -- node dist/server.js
```

### Real Example: Regex-Heavy Endpoint

```typescript
// SLOW: user-provided pattern in a hot loop
function filterResults(items: Item[], pattern: string): Item[] {
  const regex = new RegExp(pattern, 'i');
  return items.filter(item => regex.test(item.name) || regex.test(item.description));
}
```

Flame graph shows `RegExp.prototype.test` consuming 68% of samples.

```typescript
// FIXED: simple string matching — no regex needed
function filterResults(items: Item[], query: string): Item[] {
  const lowerQuery = query.toLowerCase();
  return items.filter(item =>
    item.name.toLowerCase().includes(lowerQuery) ||
    item.description.toLowerCase().includes(lowerQuery)
  );
}
```

After fix: `String.prototype.includes` at 12%. Bottleneck shifted elsewhere.

---

## Heap Snapshots

Captures every object on the V8 heap — type, size, and retainer. Node equivalent of `jmap -dump` + Eclipse MAT.

### Taking Snapshots

```typescript
import { writeHeapSnapshot } from 'node:v8';

// Trigger via a protected debug endpoint
app.post('/debug/heap-snapshot', authMiddleware, adminOnly, (req, res) => {
  const filename = writeHeapSnapshot();
  res.json({ filename });
});
```

Or via Chrome DevTools: **Memory** tab → **Heap snapshot** → **Take snapshot**.

### The Three-Snapshot Technique

```
Take snapshot 1  →  Perform action (1000 requests)  →  Take snapshot 2
                 →  Same action again               →  Take snapshot 3
```

In Snapshot 3, switch to **Comparison** view against Snapshot 1. Sort by **# Delta** or **Size Delta**. Objects growing across all three snapshots are your leak candidates.

### Finding a Closure-Based Memory Leak

```typescript
// LEAKY: closure captures the entire `context` object
const handlers: Array<() => void> = [];

function registerHandler(context: LargeContext) {
  handlers.push(() => {
    console.log(`Processing ${context.id}`); // only uses .id, captures ALL of context
  });
}
```

Heap snapshot comparison shows `(closure) +5000, Size Delta +120 MB`. Expanding **Retainers**: `handlers (Array) → (closure) → context (LargeContext) → payload (Buffer 23KB)`.

```typescript
// FIXED: extract only what the closure needs
function registerHandler(context: LargeContext) {
  const id = context.id;
  handlers.push(() => {
    console.log(`Processing ${id}`);
  });
}
```

---

## Event Loop Lag Detection

When a synchronous operation blocks the event loop, every pending callback, every incoming request waits. Event loop lag = time between when a callback is scheduled and when it fires.

### Measuring with `perf_hooks`

```typescript
import { monitorEventLoopDelay } from 'node:perf_hooks';

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();

setInterval(() => {
  const p50 = histogram.percentile(50) / 1e6;  // nanoseconds → milliseconds
  const p99 = histogram.percentile(99) / 1e6;
  const max = histogram.max / 1e6;

  console.log(`Event loop delay — p50: ${p50.toFixed(1)}ms, p99: ${p99.toFixed(1)}ms, max: ${max.toFixed(1)}ms`);

  if (p99 > 100) {
    console.warn(`EVENT LOOP LAG: p99=${p99.toFixed(1)}ms exceeds 100ms threshold`);
  }

  histogram.reset();
}, 10_000);
```

### Threshold Guidelines

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| **p50** | < 5ms | 5-20ms | > 20ms |
| **p99** | < 50ms | 50-100ms | > 100ms |
| **max** | < 100ms | 100-500ms | > 500ms |

A p99 above 100ms means 1% of event loop turns are delayed by over 100ms — that translates directly to API tail latency.

---

## Memory Leak Hunting

Full walkthrough from symptom to fix.

### Step 1 — Confirm the Symptoms

- **RSS grows continuously** and never plateaus, even during idle.
- **GC pauses increase** as the heap grows.
- **OOM kill** — Kubernetes sends SIGKILL when the container exceeds its memory limit.

```typescript
setInterval(() => {
  const mem = process.memoryUsage();
  console.log({
    rss: `${(mem.rss / 1024 / 1024).toFixed(1)} MB`,
    heapUsed: `${(mem.heapUsed / 1024 / 1024).toFixed(1)} MB`,
    external: `${(mem.external / 1024 / 1024).toFixed(1)} MB`,
  });
}, 30_000);
```

| Field | Leak Location |
|-------|---------------|
| `heapUsed` growing | JS objects (closures, maps, arrays) |
| `external` growing | Native memory (Buffers, C++ addons) |
| `rss` growing, `heapUsed` flat | Outside V8 (native addon, unmanaged Buffers) |

### Step 2 — Three-Snapshot Technique

Use heap snapshots as described above. Focus on the largest `# Delta` and retainer paths.

### Step 3 — Common Leak: EventEmitter Listeners

```typescript
// LEAKY: every request adds a listener, never removes it
const bus = new EventEmitter();

app.get('/api/stream', (req, res) => {
  const handler = (data: unknown) => { res.write(JSON.stringify(data)); };
  bus.on('update', handler);
  // Client disconnects → handler stays attached forever
});
```

In the heap snapshot, `EventEmitter._events.update` array has thousands of function references.

```typescript
// FIXED: clean up on disconnect
app.get('/api/stream', (req, res) => {
  const handler = (data: unknown) => { res.write(JSON.stringify(data)); };
  bus.on('update', handler);

  const cleanup = () => { bus.removeListener('update', handler); };
  req.on('close', cleanup);
  req.on('error', cleanup);
});
```

### Step 4 — Common Leak: Unbounded Caches

```typescript
// LEAKY: cache grows forever
const cache = new Map<string, ProcessedResult>();

// FIXED: use LRU with size limit
import { LRUCache } from 'lru-cache';
const cache = new LRUCache<string, ProcessedResult>({ max: 10_000, ttl: 1000 * 60 * 5 });
```

### Step 5 — Verify

Monitor `heapUsed` over time. Under sustained load it should plateau. Two snapshots 30 minutes apart should show near-zero delta.

---

## Production Profiling

### `--inspect` in Production

```bash
# NEVER: exposes debugger to the network
node --inspect=0.0.0.0:9229 dist/server.js

# ALWAYS: bind to localhost, tunnel in
node --inspect=127.0.0.1:9229 dist/server.js
```

**Enable inspector on a running process (no restart):**

```bash
kill -USR1 <pid>  # Process logs "Debugger listening on ws://127.0.0.1:9229/..."

# Kubernetes
kubectl exec api-server-abc123 -- kill -USR1 1
kubectl port-forward pod/api-server-abc123 9229:9229
```

### Sampling vs Instrumentation

| Approach | Overhead | Use Case |
|----------|----------|----------|
| **Sampling** (stack capture every ~1ms) | 1-5% | Production-safe |
| **Instrumentation** (wrap every function) | 10-50% | Local debugging only |

### Continuous Profiling

Runs sampling in the background at all times. You do not need to reproduce the problem — the data is already there. Analogous to always-on JFR.

```typescript
// Pyroscope (open source)
import Pyroscope from '@pyroscope/nodejs';
Pyroscope.init({
  serverAddress: 'http://pyroscope.internal:4040',
  appName: 'api-server',
  tags: { region: process.env.REGION ?? 'unknown' },
});
Pyroscope.start();

// Datadog Continuous Profiler
import tracer from 'dd-trace';
tracer.init({ profiling: true, service: 'api-server', env: 'production' });
```

---

## The `clinic` Suite

Three diagnostic tools from the Node.js diagnostics team. Each answers a different question.

### `clinic doctor` — Event Loop + GC Analysis

```bash
npx clinic doctor -- node dist/server.js
# Generate load in another terminal, then Ctrl+C. Opens HTML report.
```

Shows four charts (CPU, Memory, Event Loop Delay, Active Handles) with a diagnosis: "I/O issue," "event loop nearly blocked," "potential memory issue," or "healthy."

### `clinic flame` — CPU Flame Graph

```bash
npx clinic flame -- node dist/server.js
# Generate load, stop. Opens interactive flame graph.
```

Same concept as `0x` but integrated into the clinic workflow. Interactive search, zoom, and sample counts.

### `clinic bubbleprof` — Async Flow Visualization

Unique to Node. Visualizes async operation flow — where time is spent waiting and how tasks fan out.

```bash
npx clinic bubbleprof -- node dist/server.js
```

Bubbles = groups of async ops. Bubble size = total time. Lines = async flow. Useful for finding N+1 queries, unnecessary sequential awaits, and pool exhaustion. No direct Java equivalent.

### All Three on the Same Server

```bash
# Step 1: Triage
npx clinic doctor -- node dist/server.js
# → "Event loop nearly blocked" — CPU problem

# Step 2: Find the hot function
npx clinic flame -- node dist/server.js
# → Wide bar on `transformRecord`

# Step 3: Check async patterns
npx clinic bubbleprof -- node dist/server.js
# → Sequential DB queries in transform pipeline (N+1)
```

Now you know: `transformRecord` is CPU-heavy (optimize or offload to worker) and issues sequential DB queries (batch them).

---

## Java Parallels

| Java Tool | Node Equivalent | Key Difference |
|-----------|-----------------|----------------|
| **JFR** | `--prof`, Pyroscope, Datadog Profiler | JFR captures threading events; Node focuses on the single event loop |
| **VisualVM / JMC** | Chrome DevTools (`--inspect`) | Same workflow, runs in Chrome instead of standalone app |
| **`jmap` + Eclipse MAT** | `v8.writeHeapSnapshot()` + DevTools Memory tab | `.heapsnapshot` (JSON) vs `.hprof` (binary). Same concepts: retained size, dominator tree, retainer path |
| **GC logs (`-Xlog:gc*`)** | `--trace-gc`, `clinic doctor` | `clinic doctor` automates what you do manually with GC logs |
| **Thread dumps / `jstack`** | Event loop delay histogram, `--inspect` pause | One thread in Node — "what is on the stack" + "how long is the loop delayed" |
| **`async-profiler`** | `0x`, `clinic flame` | Both produce flame graphs. `async-profiler` shows lock contention — irrelevant in Node |

### Mental Model Translation

**JVM:** Multiple threads share a heap. Find which thread is slow, which lock is contended, which objects leak.

**Node:** One event loop processes all I/O. Find what blocks the loop, what holds references too long, whether async ops are properly parallelized.

The diagnostic questions are the same: Is it CPU? (flame graph) Is it memory? (heap snapshot) Is it waiting? (Java: thread dump `BLOCKED`/`WAITING`. Node: event loop delay + `clinic bubbleprof`) Is it GC? (Java: GC logs. Node: `--trace-gc` or `clinic doctor`)

---

## Quick-Reference Cheat Sheet

```
Problem                          First tool to reach for
──────────────────────────────   ─────────────────────────────
High CPU, slow responses         clinic flame → 0x → --prof
Memory growing over time         heap snapshots (3-snapshot technique)
Intermittent latency spikes      event loop delay histogram + clinic doctor
"Where is time spent?" (broad)   clinic doctor (triage) → flame or bubbleprof
Continuous prod profiling        Pyroscope or Datadog Continuous Profiler
Unknown async bottleneck         clinic bubbleprof
GC pauses suspected              node --trace-gc, clinic doctor
```

---

## References

- [Node.js Diagnostics Guide](https://nodejs.org/en/guides/diagnostics) — Official profiling and debugging documentation
- [V8 CPU Profiler](https://v8.dev/docs/profile) — `--prof` and `--prof-process` documentation
- [Chrome DevTools Memory Panel](https://developer.chrome.com/docs/devtools/memory-problems) — Heap snapshot and allocation timeline
- [Chrome DevTools Performance Panel](https://developer.chrome.com/docs/devtools/performance) — CPU profiling and flame charts
- [clinic.js](https://clinicjs.org/) — `clinic doctor`, `clinic flame`, `clinic bubbleprof`
- [0x](https://github.com/davidmarkclements/0x) — Zero-overhead flame graphs for Node.js
- [Pyroscope Node.js](https://pyroscope.io/docs/nodejs/) — Continuous profiling setup
- [Node.js `perf_hooks` API](https://nodejs.org/api/perf_hooks.html) — `monitorEventLoopDelay`
- [Node.js `v8` Module](https://nodejs.org/api/v8.html) — `writeHeapSnapshot` and heap statistics
