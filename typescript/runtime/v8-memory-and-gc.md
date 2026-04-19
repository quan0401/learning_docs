# V8 Memory & Garbage Collection

Your Node process is using 1.5GB of RAM. Is that a memory leak, or is V8 just being V8? To answer that question you need to understand how V8 structures its heap, when and how it collects garbage, and what "normal" memory behavior actually looks like. This doc gives you the same depth on V8 GC that the Java path gives on G1/ZGC/Shenandoah.

Cross-references: [V8 Engine Pipeline](./v8-engine-pipeline.md) for JIT compilation, [Event Loop Internals](./event-loop-internals.md) for when GC pauses actually bite. Java parallel: [JVM Garbage Collection](../../java/jvm/garbage-collection.md).

---

## V8 Heap Structure

V8 divides its managed heap into several distinct spaces, each with a different allocation and collection strategy.

```
┌─────────────────────────────────────────────────────────────────┐
│                        V8 Managed Heap                          │
│                                                                 │
│  ┌───────────────────────────────────┐  ┌────────────────────┐  │
│  │       New Space (Young Gen)       │  │     Old Space      │  │
│  │         ~4–16 MB total            │  │   (Old Generation) │  │
│  │                                   │  │                    │  │
│  │  ┌──────────┐  ┌──────────┐      │  │  Long-lived objs   │  │
│  │  │ Semi-    │  │ Semi-    │      │  │  Promoted from      │  │
│  │  │ space A  │  │ space B  │      │  │  New Space          │  │
│  │  │ ("from") │  │ ("to")   │      │  │                    │  │
│  │  └──────────┘  └──────────┘      │  │  Collected by      │  │
│  │                                   │  │  Mark-Sweep-Compact│  │
│  │  Collected by Scavenger           │  └────────────────────┘  │
│  └───────────────────────────────────┘                          │
│                                                                 │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Large Object     │  │  Code Space  │  │   Map Space      │  │
│  │ Space            │  │              │  │                  │  │
│  │ Objects >~500KB  │  │ JIT-compiled │  │ Hidden classes   │  │
│  │ Never moved      │  │ machine code │  │ (Maps/Shapes)    │  │
│  │ GC'd in place    │  │ from TurboFan│  │ Object structure │  │
│  └──────────────────┘  └──────────────┘  └──────────────────┘  │
│                                                                 │
│  Total heap controlled by:                                      │
│    --max-old-space-size (Old + Large Object + Code + Map)       │
│    --max-semi-space-size (each semi-space in New Space)          │
└─────────────────────────────────────────────────────────────────┘
```

### Space-by-Space Breakdown

| Space | Default Size | What Lives Here | Collection |
|-------|-------------|-----------------|------------|
| **New Space** | ~4–16 MB total (2 semi-spaces) | Newly allocated objects | Scavenger (minor GC) |
| **Old Space** | Grows up to `--max-old-space-size` | Objects surviving 2+ minor GCs | Mark-Sweep-Compact (major GC) |
| **Large Object Space** | Part of old space limit | Objects larger than ~500 KB | Mark-Sweep (never moved) |
| **Code Space** | Part of old space limit | TurboFan-compiled machine code | Collected with major GC |
| **Map Space** | Part of old space limit | Hidden classes (object shapes) | Collected with major GC |

### Key CLI Flags

```bash
# Set maximum old space size (in MB)
node --max-old-space-size=4096 app.js

# Set semi-space size (in MB, each half of new space)
node --max-semi-space-size=16 app.js

# View current heap limits
node -e "console.log(v8.getHeapStatistics())"
```

---

## Scavenger (Minor GC)

The Scavenger handles New Space using **Cheney's semi-space copying algorithm**. This is the GC you hit most often — it fires every time the active semi-space fills up, which can be hundreds of times per second under heavy allocation.

### How It Works

```
ALLOCATION PHASE:
┌──────────────────────────────────┐
│  Semi-space A ("from")           │   Semi-space B ("to")
│  [obj1][obj2][obj3][obj4][obj5]  │   [  empty  ]
│  ↑ allocation pointer            │
└──────────────────────────────────┘

SCAVENGE (triggered when "from" is full):
1. Scan roots (stack, global handles)
2. For each live object in "from":
   - If survived once before → PROMOTE to Old Space
   - Otherwise → COPY to "to" semi-space
3. Dead objects in "from" are simply abandoned (no sweep needed)
4. Swap labels: "to" becomes "from", old "from" becomes "to"

AFTER SCAVENGE:
┌──────────────────────────────────┐
│  Semi-space B (now "from")       │   Semi-space A (now "to")
│  [obj2][obj5]                    │   [  empty  ]
│  ↑ new allocation pointer        │
└──────────────────────────────────┘
obj1, obj3 → promoted to Old Space (survived before)
obj4       → dead, reclaimed implicitly
```

### Why the Scavenger Is Fast

1. **Small space**: Only scans 4–16 MB, not the entire heap.
2. **Only touches live objects**: Dead objects cost zero work — they are simply overwritten.
3. **No fragmentation**: Copying compacts automatically. The "to" space is always contiguous.
4. **Generational hypothesis**: Most objects die young. Typically 70–90% of New Space is garbage at collection time.

### Write Barriers: Handling Old-to-New References

When an old-space object starts pointing to a new-space object, the Scavenger needs to know about it — otherwise it would miss a root. V8 uses a **write barrier** that records these cross-generational pointers in a **remembered set** (also called the "store buffer").

```typescript
// This assignment triggers a write barrier inside V8:
oldSpaceObject.child = newlyAllocatedObject;
// V8 records: "oldSpaceObject references something in New Space"
// Scavenger will scan these recorded slots as additional roots
```

This is the same concept as G1's remembered sets in the JVM, just simpler because V8 only has two generations.

### Promotion Policy

An object is promoted to Old Space after surviving **two Scavenge cycles** (not one — V8 uses a single-bit "survived once" flag). Objects larger than a threshold may be allocated directly in Old Space or Large Object Space, bypassing New Space entirely.

---

## Mark-Sweep-Compact (Major GC)

When Old Space pressure is high, V8 triggers a major GC cycle. This involves the entire old generation and uses a three-phase approach.

### Phase 1: Marking (Tri-Color Algorithm)

V8 uses a tri-color abstraction identical in concept to what G1 and ZGC use in the JVM:

| Color | Meaning |
|-------|---------|
| **White** | Not yet discovered — presumed garbage |
| **Gray** | Discovered, but children not yet scanned |
| **Black** | Fully scanned — definitely alive |

```
MARKING TRAVERSAL:

1. Start: all objects are WHITE
2. Color roots GRAY, push onto marking worklist
3. Pop a GRAY object:
   a. Scan its fields
   b. Color any WHITE children GRAY
   c. Color this object BLACK
4. Repeat until no GRAY objects remain
5. All remaining WHITE objects are garbage
```

### Phase 2: Sweeping (Free List)

After marking, V8 walks through memory pages and builds **free lists** from the gaps left by dead (white) objects. Sweeping is done **lazily** — V8 only sweeps a page when it actually needs to allocate from it. This spreads the cost over time rather than doing it all at once.

```
BEFORE SWEEP:                    AFTER SWEEP:
┌────────────────────────┐       ┌────────────────────────┐
│ [LIVE][dead][LIVE]     │       │ [LIVE][free][LIVE]     │
│ [dead][dead][LIVE]     │  →    │ [  free    ][LIVE]     │
│ [LIVE][dead][dead]     │       │ [LIVE][  free    ]     │
└────────────────────────┘       └────────────────────────┘
                                  Free chunks → free list
```

### Phase 3: Compaction (Defragmentation)

Over time, sweeping creates fragmentation. Compaction is the fix: V8 evacuates live objects from fragmented pages into fresh pages, then frees the old pages entirely. Compaction is **selective** — V8 only compacts the most fragmented pages, not the entire heap. This is similar to G1's mixed collections targeting high-garbage regions first.

### Reducing Pause Times

A naive stop-the-world mark-sweep-compact on a 2GB heap would pause for hundreds of milliseconds. V8 uses three techniques to keep pauses short:

#### Incremental Marking

Instead of marking the entire heap in one go, V8 interleaves small chunks of marking work with JavaScript execution. Each increment processes a few milliseconds of marking, then yields back to the event loop.

```
JavaScript │████│░░│████│░░│████│░░│████│░░│████│  SWEEP  │████│
           │    │GC│    │GC│    │GC│    │GC│    │  pause  │    │
                ↑    ↑    ↑    ↑
           Incremental marking steps (~1-5ms each)
```

#### Concurrent Marking

Starting with V8 6.x (Node 10+), marking also runs on **helper threads** while JavaScript continues on the main thread. The main thread only needs to pause briefly at the end to finalize marking (the "atomic pause"). This is directly analogous to G1's concurrent marking phase in the JVM.

```
Main thread:     │███████████████████████████│ pause │████████│
Helper threads:  │  marking   marking   marking     │  idle  │
```

#### Lazy Sweeping

Sweeping happens on-demand when allocation needs free memory, not all at once after marking. Pages are swept incrementally as they are needed.

---

## `--max-old-space-size`

This is the single most important V8 memory flag. It controls the maximum size of Old Space (plus Large Object Space, Code Space, and Map Space). New Space is separate and controlled by `--max-semi-space-size`.

### Default Values by Node Version

| Node Version | Default `--max-old-space-size` | Notes |
|-------------|-------------------------------|-------|
| Node 10–11 | ~1.5 GB | Fixed default |
| Node 12+ | Based on available system memory | ~1.5 GB on small machines, up to ~4 GB on larger ones |
| Node 14+ | Same heuristic as 12 | Documentation clarified the dynamic behavior |
| Node 20+ | Same heuristic, but heap limit may be up to ~4 GB | V8 continues to refine the heuristic |

To check the actual limit on your system:

```bash
node -e "const v8 = require('v8'); \
  const stats = v8.getHeapStatistics(); \
  console.log('Heap size limit:', (stats.heap_size_limit / 1024 / 1024).toFixed(0), 'MB')"
```

### Setting It for Containers

In a containerized environment, the default heuristic may see the host machine's memory, not the container's limit. You must set it explicitly.

**Rule of thumb: set `--max-old-space-size` to ~75% of the container's memory limit.** The remaining 25% covers:
- Native memory (C++ objects, libuv handles, Buffer pools outside the V8 heap)
- Thread stacks (main thread + worker threads + libuv thread pool)
- OS overhead and file system caches

```bash
# Container with 2GB memory limit
node --max-old-space-size=1536 app.js  # 1.5 GB ≈ 75% of 2 GB

# Container with 4GB memory limit
node --max-old-space-size=3072 app.js  # 3 GB ≈ 75% of 4 GB
```

```dockerfile
# In a Dockerfile
ENV NODE_OPTIONS="--max-old-space-size=1536"
CMD ["node", "dist/main.js"]
```

### What Happens When You Hit the Limit

V8 does **not** degrade gracefully. When old space reaches the limit and GC cannot free enough memory:

1. V8 triggers increasingly aggressive GC cycles (more stop-the-world pauses).
2. If GC still cannot free enough, V8 throws a **FATAL ERROR**:

```
FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory

 1: 0x10003c4918 node::Abort() [/usr/local/bin/node]
 2: 0x10003c4a9c node::OOMErrorHandler(char const*, v8::OOMDetails const&)
 ...
```

3. The process **crashes**. There is no exception to catch, no event to listen for. The process is terminated by V8 itself.

This is different from the JVM, where you get an `OutOfMemoryError` that is technically catchable (though rarely recoverable). In Node, OOM = immediate process death.

---

## Memory Leak Patterns in Node

These are the five most common ways Node processes leak memory in production. Every one of these produces the same symptom: RSS grows monotonically over hours or days until you hit OOM or get restarted.

### 1. Closures Holding References (Event Listeners Never Removed)

The most common leak. A closure captures a variable from its enclosing scope, and the closure itself is registered as a callback that never gets cleaned up.

```typescript
import { EventEmitter } from "events";

const emitter = new EventEmitter();

function handleRequest(data: Buffer): void {
  // BAD: 'data' is captured by the closure and held as long as the listener exists
  const largePayload = Buffer.from(data);

  emitter.on("process", () => {
    // This closure captures 'largePayload'
    // It will never be garbage collected as long as the listener is registered
    console.log(largePayload.length);
  });
}

// Every call to handleRequest adds ANOTHER listener
// After 10,000 requests: 10,000 closures each holding a Buffer
```

**Fix**: Remove listeners when done, or use `once` for one-shot handlers.

```typescript
function handleRequestFixed(data: Buffer): void {
  const largePayload = Buffer.from(data);

  // Option A: Use 'once' for single-fire listeners
  emitter.once("process", () => {
    console.log(largePayload.length);
  });

  // Option B: Explicitly remove when done
  const handler = (): void => {
    console.log(largePayload.length);
    emitter.off("process", handler);
  };
  emitter.on("process", handler);

  // Option C: Use AbortController (Node 16+)
  const ac = new AbortController();
  emitter.on("process", () => {
    console.log(largePayload.length);
  }, { signal: ac.signal });
  // Later: ac.abort() removes the listener
}
```

### 2. Global Caches Growing Unbounded

```typescript
// BAD: This map grows forever
const userCache = new Map<string, UserProfile>();

async function getUser(id: string): Promise<UserProfile> {
  if (userCache.has(id)) {
    return userCache.get(id)!;
  }
  const user = await fetchUserFromDb(id);
  userCache.set(id, user); // Never evicted
  return user;
}
```

**Fix**: Use an LRU cache with a size limit, or `WeakRef` for entries that can be recollected.

```typescript
import { LRUCache } from "lru-cache";

// Fixed: bounded cache with TTL and max entries
const userCache = new LRUCache<string, UserProfile>({
  max: 500,          // At most 500 entries
  ttl: 1000 * 60 * 5 // 5-minute TTL
});

async function getUser(id: string): Promise<UserProfile> {
  const cached = userCache.get(id);
  if (cached !== undefined) {
    return cached;
  }
  const user = await fetchUserFromDb(id);
  userCache.set(id, user);
  return user;
}
```

### 3. Forgotten Timers

```typescript
// BAD: interval never cleared, closure holds reference to 'heavyData'
function startPolling(): void {
  const heavyData = loadLargeDataset();

  setInterval(() => {
    // This closure captures 'heavyData'
    // Even if nothing references 'startPolling' anymore,
    // the interval callback prevents GC of heavyData
    process(heavyData);
  }, 5000);
}

// Called on each connection — each connection leaks an interval + data
```

**Fix**: Always store and clear timer references.

```typescript
function startPolling(): { stop: () => void } {
  const heavyData = loadLargeDataset();

  const intervalId = setInterval(() => {
    process(heavyData);
  }, 5000);

  return {
    stop(): void {
      clearInterval(intervalId);
      // heavyData is now eligible for GC
    }
  };
}

// Usage:
const poller = startPolling();
// When done:
poller.stop();
```

### 4. Detached DOM Trees (Less Relevant for Backend)

This is primarily a browser concern, but it can appear in server-side rendering (SSR) scenarios where you build DOM-like structures with jsdom or similar libraries and lose the reference to the root while children still reference parents.

```typescript
// SSR scenario with jsdom
import { JSDOM } from "jsdom";

function renderPage(): string {
  const dom = new JSDOM("<!DOCTYPE html><html><body></body></html>");
  const document = dom.window.document;

  const detachedDiv = document.createElement("div");
  // If you store 'detachedDiv' somewhere but lose reference to 'dom',
  // the entire DOM tree stays alive through the detached node's parent chain

  return dom.serialize();
  // Make sure dom.window.close() is called to free resources
}
```

### 5. Large Buffers Accumulated in Streams

```typescript
import { Writable } from "stream";

// BAD: accumulating all chunks in memory
const chunks: Buffer[] = [];

const writable = new Writable({
  write(chunk: Buffer, _encoding, callback): void {
    chunks.push(chunk); // Array grows unbounded
    callback();
  }
});

// If reading a 10 GB file, 'chunks' will hold 10 GB in memory
```

**Fix**: Process chunks incrementally, use backpressure, or use `pipeline` with proper flow control.

```typescript
import { pipeline, Transform } from "stream";
import { createReadStream, createWriteStream } from "fs";

// Fixed: streaming transform with backpressure, no accumulation
const transform = new Transform({
  transform(chunk: Buffer, _encoding, callback): void {
    // Process each chunk immediately, don't accumulate
    const processed = processChunk(chunk);
    callback(null, processed);
  }
});

pipeline(
  createReadStream("input.dat"),
  transform,
  createWriteStream("output.dat"),
  (err) => {
    if (err) console.error("Pipeline failed:", err);
  }
);
```

---

## `WeakRef` and `FinalizationRegistry`

Added in ES2021 (Node 14.6+), these give you weak references and destructor-like callbacks — tools for building caches that cooperate with the garbage collector instead of fighting it.

### `WeakRef`

A `WeakRef` holds a reference to an object without preventing GC from collecting it.

```typescript
const strongRef = { data: "important" };
const weak = new WeakRef(strongRef);

// Access the value — may return undefined if GC'd
const value = weak.deref();
if (value !== undefined) {
  console.log(value.data); // "important"
}
```

### `FinalizationRegistry`

Registers a callback that fires when a tracked object is garbage collected. The callback receives a "held value" — typically a cleanup token like a file handle or cache key.

```typescript
const registry = new FinalizationRegistry<string>((cacheKey: string) => {
  console.log(`Object for key "${cacheKey}" was garbage collected`);
  // Perform cleanup: remove stale cache entry, close handle, etc.
});

function trackObject(key: string, obj: object): void {
  registry.register(obj, key);
  // When 'obj' is GC'd, callback fires with 'key'
}
```

### Non-Determinism Warning

Both `WeakRef.deref()` and `FinalizationRegistry` callbacks are **non-deterministic**. The spec makes no guarantees about:
- When a weakly-held object will be collected
- When (or if) a finalization callback will run
- The order of finalization callbacks

Do not rely on these for correctness. Use them for **optimization** (caching) or **resource cleanup** (as a safety net, not the primary mechanism).

### Practical Example: WeakRef-Based Cache

```typescript
class WeakCache<K, V extends object> {
  private readonly cache = new Map<K, WeakRef<V>>();
  private readonly registry: FinalizationRegistry<K>;

  constructor() {
    // When a cached value is GC'd, remove its stale entry from the map
    this.registry = new FinalizationRegistry<K>((key: K) => {
      const ref = this.cache.get(key);
      // Only delete if the ref is actually dead (avoid race with re-set)
      if (ref !== undefined && ref.deref() === undefined) {
        this.cache.delete(key);
      }
    });
  }

  set(key: K, value: V): void {
    const existingRef = this.cache.get(key);
    if (existingRef !== undefined) {
      // Unregister old value if it existed
      const existing = existingRef.deref();
      if (existing !== undefined) {
        this.registry.unregister(existing);
      }
    }
    this.cache.set(key, new WeakRef(value));
    this.registry.register(value, key, value); // value is also the unregister token
  }

  get(key: K): V | undefined {
    const ref = this.cache.get(key);
    if (ref === undefined) return undefined;

    const value = ref.deref();
    if (value === undefined) {
      // Stale entry — clean it up eagerly
      this.cache.delete(key);
      return undefined;
    }
    return value;
  }

  get size(): number {
    return this.cache.size; // May include stale entries not yet cleaned
  }
}
```

**When to use WeakRef vs. LRU**:
- **WeakRef cache**: When cached values are large objects also referenced elsewhere. The cache is secondary — if nothing else needs the object, let it die.
- **LRU cache**: When the cache is the primary owner of the data. You want deterministic eviction control with a bounded size.

---

## Diagnosing Memory Issues

### Step 1: `process.memoryUsage()`

The cheapest signal. Call it periodically to track trends.

```typescript
function logMemory(): void {
  const usage = process.memoryUsage();
  console.log({
    rss: `${(usage.rss / 1024 / 1024).toFixed(1)} MB`,          // Total process memory
    heapTotal: `${(usage.heapTotal / 1024 / 1024).toFixed(1)} MB`, // V8 heap allocated
    heapUsed: `${(usage.heapUsed / 1024 / 1024).toFixed(1)} MB`,  // V8 heap actually used
    external: `${(usage.external / 1024 / 1024).toFixed(1)} MB`,  // C++ objects (Buffers)
    arrayBuffers: `${(usage.arrayBuffers / 1024 / 1024).toFixed(1)} MB`
  });
}

// Track every 30 seconds
setInterval(logMemory, 30_000);
```

**What to look for**:
- `heapUsed` growing monotonically = V8 heap leak (JS objects)
- `rss` growing but `heapUsed` stable = native memory leak (Buffers, C++ addons)
- `external` growing = Buffer or TypedArray leak

### Step 2: Heap Snapshots

Heap snapshots capture every object on the V8 heap. Take two snapshots at different times, compare them in Chrome DevTools to find what is accumulating.

```bash
# Start your app with the inspector
node --inspect dist/main.js

# Or enable inspector programmatically for on-demand snapshots
```

```typescript
import v8 from "v8";
import fs from "fs";

function takeHeapSnapshot(): string {
  const filename = `heap-${Date.now()}.heapsnapshot`;
  const snapshotStream = v8.writeHeapSnapshot(filename);
  console.log(`Heap snapshot written to: ${snapshotStream}`);
  return snapshotStream;
}

// Expose via HTTP endpoint for production debugging
// GET /debug/heap-snapshot → triggers snapshot and returns filename
```

### Step 3: Compare Snapshots in Chrome DevTools

1. Open `chrome://inspect` in Chrome
2. Click "Open dedicated DevTools for Node"
3. Go to the **Memory** tab
4. Load both `.heapsnapshot` files
5. Select **Comparison** view between the two snapshots
6. Sort by **# Delta** or **Size Delta** to find what grew

**What to look for in comparison view**:
- Large positive deltas in `(string)` — string accumulation
- Growing `(closure)` count — event listener leak
- Growing `(array)` — unbounded arrays or maps
- Objects with a specific constructor name that keep increasing

### Step 4: `--expose-gc` for Manual GC

Useful for testing: force a full GC before taking measurements to ensure you are measuring retained objects, not just objects awaiting collection.

```bash
node --expose-gc dist/main.js
```

```typescript
declare const global: { gc?: () => void };

function forceGC(): void {
  if (global.gc) {
    global.gc();
  } else {
    console.warn("GC not exposed. Start with --expose-gc");
  }
}

// Usage: force GC before taking a heap snapshot for clean data
forceGC();
const snapshot = takeHeapSnapshot();
```

### Leak Hunting Walkthrough

```
1. Reproduce the suspected leak
   - Run load test or repeat the suspicious operation 1,000 times

2. Baseline snapshot
   - Force GC with --expose-gc
   - Take heap snapshot #1 (the "before")

3. Run more load
   - Repeat the operation another 1,000 times

4. Comparison snapshot
   - Force GC again
   - Take heap snapshot #2 (the "after")

5. Analyze in DevTools
   - Comparison view: snapshot #2 vs #1
   - Sort by Size Delta descending
   - The top entries are your leak candidates

6. Retainers view
   - Click a leaked object type
   - Expand the "Retainers" panel to see WHY it is alive
   - Follow the chain: leaked object ← closure ← event listener ← emitter
   - That chain tells you exactly what to fix
```

---

## JVM Parallel: V8 vs. JVM GC Comparison

If you know JVM garbage collection, this table maps every V8 concept to its JVM equivalent.

| Concept | V8 (Node.js) | JVM (Java) |
|---------|--------------|------------|
| **Young generation** | New Space (2 semi-spaces, ~4–16 MB) | Young Gen: Eden + Survivor 0 + Survivor 1 |
| **Old generation** | Old Space (up to `--max-old-space-size`) | Old Gen / Tenured (up to `-Xmx`) |
| **Large object handling** | Large Object Space (>~500 KB, never moved) | Humongous regions in G1 (> 50% of region size) |
| **Minor GC algorithm** | Scavenger (Cheney's semi-space copy) | ParNew / G1 young collection (copy to survivor/old) |
| **Major GC algorithm** | Mark-Sweep-Compact | G1 mixed collection / Full GC / CMS |
| **Tri-color marking** | Same concept | Same concept (G1, ZGC, Shenandoah all use it) |
| **Incremental marking** | Yes — interleaved with JS execution | G1 concurrent marking (similar purpose) |
| **Concurrent marking** | Helper threads mark while JS runs | G1/ZGC/Shenandoah concurrent marking threads |
| **Compaction strategy** | Selective (most fragmented pages only) | G1: mixed collections target high-garbage regions |
| **Write barrier** | Store buffer for old→new references | G1: remembered sets + card table |
| **Promotion threshold** | 2 Scavenge survivals | Configurable (`-XX:MaxTenuringThreshold`, default 15) |
| **Heap limit flag** | `--max-old-space-size` (MB) | `-Xmx` (max heap) |
| **Young gen size flag** | `--max-semi-space-size` (MB, per semi-space) | `-Xmn` or `-XX:NewSize` / `-XX:MaxNewSize` |
| **OOM behavior** | FATAL ERROR → process crash (not catchable) | `OutOfMemoryError` (technically catchable) |
| **GC logging** | `--trace-gc` | `-Xlog:gc*` (JDK 9+) or `-XX:+PrintGCDetails` |
| **Number of GC algorithms** | 1 (no choice — Orinoco is the only collector) | Many: Serial, Parallel, G1, ZGC, Shenandoah |

### Key Differences to Internalize

1. **No GC choice in V8.** The JVM lets you pick G1, ZGC, or Shenandoah. V8 has one collector (Orinoco) with no alternatives. You tune heap sizes, not collector algorithms.

2. **Smaller young generation.** V8's New Space is 4–16 MB. JVM's Young Gen is typically 100s of MB. V8 runs minor GC far more frequently but each pause is sub-millisecond.

3. **OOM is fatal in Node.** In Java, `OutOfMemoryError` is a throwable that can (in theory) be caught. In Node, hitting the heap limit kills the process immediately. This makes right-sizing `--max-old-space-size` critical in production.

4. **Native memory is a bigger deal in Node.** `Buffer` objects live outside the V8 heap. A process can appear healthy on `heapUsed` while leaking native memory through Buffers, streams, or C++ addons. Always monitor RSS alongside heap metrics.

5. **Pause times.** V8's concurrent and incremental approach keeps most GC pauses under 10ms. The JVM's ZGC and Shenandoah target sub-millisecond pauses on much larger heaps. For typical Node workloads (100s of MB to low GB), V8's approach is sufficient. For multi-GB heaps, the JVM has a significant advantage.

---

## GC Logging and Monitoring

### Trace GC Output

```bash
# Print a line for each GC event
node --trace-gc dist/main.js

# Sample output:
# [44049:0x7f8b0a004000] 83 ms: Scavenge 2.1 (3.0) -> 1.8 (4.0) MB, 0.5 / 0.0 ms
# [44049:0x7f8b0a004000] 157 ms: Mark-Compact 4.2 (6.0) -> 3.1 (6.0) MB, 3.2 / 0.0 ms
#
# Format: [pid:isolate] time: Type before (committed) -> after (committed) MB, pause / idle ms
```

### Programmatic GC Events via `perf_hooks`

```typescript
import { PerformanceObserver } from "perf_hooks";

const obs = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // entry.kind: 1 = Scavenge, 2 = Minor Mark-Compact,
    //             4 = Mark-Sweep-Compact, 8 = Incremental marking
    console.log(`GC: ${entry.name} | ${entry.duration.toFixed(2)}ms`);
  }
});

obs.observe({ entryTypes: ["gc"] });
```

---

## References

- [V8 Blog: Trash Talk — the Orinoco Garbage Collector](https://v8.dev/blog/trash-talk) — Comprehensive overview of V8's GC architecture
- [V8 Blog: Concurrent Marking](https://v8.dev/blog/concurrent-marking) — Deep dive into concurrent marking implementation
- [V8 Blog: Orinoco — Young Generation GC](https://v8.dev/blog/orinoco-parallel-scavenger) — Parallel scavenger details
- [Node.js Docs: `process.memoryUsage()`](https://nodejs.org/api/process.html#processmemoryusage) — Official API reference
- [Node.js Docs: V8 Module](https://nodejs.org/api/v8.html) — Heap statistics and snapshots
- [Node.js Docs: `--max-old-space-size`](https://nodejs.org/api/cli.html#--max-old-space-sizesize-in-mib) — CLI flag reference
- [Chrome DevTools: Memory Panel](https://developer.chrome.com/docs/devtools/memory-problems/) — Heap snapshot analysis guide
- [MDN: WeakRef](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef) — WeakRef specification and examples
- [MDN: FinalizationRegistry](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/FinalizationRegistry) — FinalizationRegistry specification
