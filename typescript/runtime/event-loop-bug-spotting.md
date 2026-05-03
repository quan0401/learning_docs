---
title: "Node.js Event Loop — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, event-loop, nodejs, async, libuv, typescript]
---

# Node.js Event Loop — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `event-loop` `nodejs` `async` `libuv` `typescript`

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

Active-recall practice for the Node.js event loop. Twenty-two broken snippets covering libuv phase ordering, microtask vs. macrotask scheduling, blocking the loop with sync work, `process.nextTick` starvation, unhandled rejections, AbortSignal propagation, stream backpressure, and worker-thread/IPC pitfalls. Run through Easy in one sitting, Subtle in a second pass, and treat Senior trap as a code-review interview drill. Every bug cites Node.js docs, libuv docs, V8 blog, or a public Node.js GitHub issue.

## How to use this doc
- Try to spot the bug before opening the `<details>` hint.
- Hints are one line; solutions are in §4 keyed by bug number.
- Don't skip Easy unless you've already nailed those traps in code review.

## 1. Easy (warm-up traps)

### Bug 1 — sync crypto on a hot path
```typescript
import { randomBytes } from "node:crypto";
import express from "express";

const app = express();
app.get("/token", (_req, res) => {
  const token = randomBytes(48).toString("hex"); // sync overload
  res.json({ token });
});
```
<details><summary>Hint</summary>
Look up the two overloads of `randomBytes` — one of them blocks the loop for entropy.
</details>

### Bug 2 — `fs.readFileSync` in a request handler
```typescript
import { readFileSync } from "node:fs";
import http from "node:http";

http.createServer((_req, res) => {
  const cfg = readFileSync("./config.json", "utf8");
  res.end(cfg);
}).listen(3000);
```
<details><summary>Hint</summary>
Every byte of disk latency stalls every other connection.
</details>

### Bug 3 — `await` in a `for` loop, no concurrency
```typescript
async function fetchAll(urls: string[]) {
  const out: Response[] = [];
  for (const u of urls) {
    out.push(await fetch(u)); // serial round-trips
  }
  return out;
}
```
<details><summary>Hint</summary>
Each iteration waits for the previous network call before starting.
</details>

### Bug 4 — `Promise.all` when one failure shouldn't kill the rest
```typescript
async function loadDashboard(userId: string) {
  const [profile, orders, recs] = await Promise.all([
    getProfile(userId),
    getOrders(userId),
    getRecommendations(userId), // optional, often 5xx
  ]);
  return { profile, orders, recs };
}
```
<details><summary>Hint</summary>
First rejection short-circuits the whole batch — was that the intent?
</details>

### Bug 5 — fire-and-forget swallows the error
```typescript
function onUserSignup(user: User) {
  sendWelcomeEmail(user); // returns a Promise, never awaited
  return { ok: true };
}
```
<details><summary>Hint</summary>
A rejection here becomes an unhandled-rejection warning at best.
</details>

### Bug 6 — timer with too-large delay
```typescript
function scheduleNextRun(callback: () => void, days: number) {
  setTimeout(callback, days * 24 * 60 * 60 * 1000); // 30 days
}
scheduleNextRun(() => console.log("monthly"), 30);
```
<details><summary>Hint</summary>
Node's timer delay is a 32-bit signed int — anything over ~24.8 days wraps.
</details>

### Bug 7 — emitting before the subscriber attaches
```typescript
import { EventEmitter } from "node:events";

class Loader extends EventEmitter {
  start() {
    queueMicrotask(() => this.emit("ready"));
  }
}
const l = new Loader();
l.start();
l.on("ready", () => console.log("got it"));
```
<details><summary>Hint</summary>
The microtask runs before control returns to the line that subscribes.
</details>

## 2. Subtle (review-passers)

### Bug 8 — `process.nextTick` recursion starves I/O
```typescript
function drain(queue: Task[]) {
  if (queue.length === 0) return;
  const t = queue.shift()!;
  t.run();
  process.nextTick(() => drain(queue)); // never yields to poll phase
}
```
<details><summary>Hint</summary>
The nextTick queue is fully drained before the loop moves on — recursion never lets timers or I/O run.
</details>

### Bug 9 — `queueMicrotask` vs `setImmediate` confusion
```typescript
function chunkedWork(items: Item[]) {
  if (items.length === 0) return;
  process(items.shift()!);
  queueMicrotask(() => chunkedWork(items)); // intended to "yield"
}
```
<details><summary>Hint</summary>
Microtasks run inside the current phase — they don't yield to I/O at all.
</details>

### Bug 10 — async `EventEmitter` listener swallows errors
```typescript
emitter.on("data", async (chunk) => {
  await processChunk(chunk); // throws sometimes
});
```
<details><summary>Hint</summary>
Rejections from an async listener are not propagated to the emitter.
</details>

### Bug 11 — unhandled rejection in a `setTimeout` callback
```typescript
setTimeout(async () => {
  const data = await fetchSomething(); // can reject
  log(data);
}, 5000);
```
<details><summary>Hint</summary>
Timer callbacks are not awaited by anyone — the rejection has no handler.
</details>

### Bug 12 — double-resolve when mixing callbacks and promises
```typescript
function readConfig(path: string): Promise<string> {
  return new Promise((resolve, reject) => {
    fs.readFile(path, "utf8", (err, data) => {
      if (err) reject(err);
      resolve(data); // runs even on error
    });
  });
}
```
<details><summary>Hint</summary>
After `reject`, control falls through and `resolve` runs too.
</details>

### Bug 13 — `AbortSignal` not threaded through the pipeline
```typescript
async function search(q: string, signal: AbortSignal) {
  const a = await fetch(`/v1/a?q=${q}`, { signal });
  const b = await fetch(`/v1/b?q=${q}`); // no signal — keeps running after abort
  return [await a.json(), await b.json()];
}
```
<details><summary>Hint</summary>
The second leg of the pipeline ignores the cancellation token.
</details>

### Bug 14 — listener added inside a hot handler
```typescript
http.createServer((req, res) => {
  process.on("uncaughtException", (e) => log(e, req.url)); // grows per request
  // ...
}).listen(3000);
```
<details><summary>Hint</summary>
Every request adds another listener — eventually you'll see a `MaxListenersExceededWarning`.
</details>

### Bug 15 — ignoring `stream.write` backpressure
```typescript
function pump(src: Readable, dest: Writable) {
  src.on("data", (chunk) => {
    dest.write(chunk); // ignores the boolean return
  });
}
```
<details><summary>Hint</summary>
`write` returns `false` when the internal buffer is full — you're supposed to pause.
</details>

### Bug 16 — Buffer concat in a tight loop
```typescript
let acc = Buffer.alloc(0);
for (const chunk of chunks) {
  acc = Buffer.concat([acc, chunk]); // O(n^2) allocations
}
```
<details><summary>Hint</summary>
Every iteration allocates a new buffer the size of everything seen so far.
</details>

### Bug 17 — catastrophic backtracking regex on user input
```typescript
function isLooseEmail(s: string): boolean {
  return /^([a-zA-Z0-9]+)+@example\.com$/.test(s); // nested quantifiers
}
isLooseEmail("a".repeat(40) + "!"); // hangs
```
<details><summary>Hint</summary>
Nested `(…+)+` against a non-matching tail is the classic ReDoS shape.
</details>

## 3. Senior trap (production-only failures)

### Bug 18 — top-level `await` blocking CommonJS interop
```typescript
// esm-only module: db.mjs
export const pool = await createPool(); // top-level await

// CommonJS consumer
const { pool } = require("./db.mjs"); // throws ERR_REQUIRE_ASYNC_MODULE on older Node
```
<details><summary>Hint</summary>
Top-level await turns the module graph asynchronous; pre-22 CJS `require` cannot drive it.
</details>

### Bug 19 — non-transferable `Buffer` to a Worker
```typescript
import { Worker } from "node:worker_threads";

const big = Buffer.alloc(256 * 1024 * 1024);
worker.postMessage(big); // copies, doubling RSS
```
<details><summary>Hint</summary>
`postMessage` clones unless you pass the underlying `ArrayBuffer` in the transfer list.
</details>

### Bug 20 — async_hooks instrumentation slows the whole process
```typescript
import { createHook } from "node:async_hooks";

createHook({
  init(asyncId, type, triggerAsyncId) {
    fs.writeSync(1, `init ${type}\n`); // sync log on every async op
  },
}).enable();
```
<details><summary>Hint</summary>
Every promise, timer, and socket allocation now performs a sync write.
</details>

### Bug 21 — HTTP keep-alive sockets leak after `uncaughtException`
```typescript
process.on("uncaughtException", (err) => {
  log(err); // swallow, keep serving
  // server keeps accepting; in-flight sockets never close
});
```
<details><summary>Hint</summary>
After an uncaught exception the process state is undefined — keep-alive sockets and their resources are not safely reusable.
</details>

### Bug 22 — cluster IPC race on graceful shutdown
```typescript
process.on("SIGTERM", () => {
  server.close(); // stops accepting, returns immediately
  process.disconnect(); // cuts IPC before in-flight requests finish
});
```
<details><summary>Hint</summary>
`server.close` doesn't wait for active connections, and `disconnect` severs the worker before the master sees you finish.
</details>

## 4. Solutions

### Bug 1 — sync crypto on a hot path
**Root cause:** `randomBytes(size)` (no callback) is the synchronous overload. It blocks the event loop while reading from the kernel CSPRNG. On a busy server this stalls every other connection during entropy reads.
**Fix:**
```typescript
import { randomBytes } from "node:crypto";
import { promisify } from "node:util";
const randomBytesAsync = promisify(randomBytes);

app.get("/token", async (_req, res) => {
  const token = (await randomBytesAsync(48)).toString("hex");
  res.json({ token });
});
```
**Reference:** [Node.js `crypto.randomBytes` docs](https://nodejs.org/api/crypto.html#cryptorandombytessize-callback)

### Bug 2 — `fs.readFileSync` in a request handler
**Root cause:** `readFileSync` blocks the single event-loop thread. Every concurrent request waits on disk latency from a previous request.
**Fix:**
```typescript
import { readFile } from "node:fs/promises";
http.createServer(async (_req, res) => {
  const cfg = await readFile("./config.json", "utf8");
  res.end(cfg);
}).listen(3000);
```
Better: cache the value at boot.
**Reference:** [Don't Block the Event Loop (Node.js)](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)

### Bug 3 — `await` in a `for` loop, no concurrency
**Root cause:** `await` suspends the loop body — each `fetch` waits for the prior to settle. Throughput collapses to 1/RTT.
**Fix:**
```typescript
async function fetchAll(urls: string[]) {
  return Promise.all(urls.map((u) => fetch(u)));
}
```
For bounded concurrency use `p-limit` or `Promise.allSettled` with a semaphore.
**Reference:** [MDN — `for await...of` and parallelism](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of)

### Bug 4 — `Promise.all` when one failure shouldn't kill the rest
**Root cause:** `Promise.all` rejects on the first rejection and discards every other settled value. If `getRecommendations` is non-essential, the dashboard fails for the wrong reason.
**Fix:**
```typescript
const [profile, orders, recs] = await Promise.allSettled([
  getProfile(userId), getOrders(userId), getRecommendations(userId),
]);
return {
  profile: profile.status === "fulfilled" ? profile.value : null,
  orders: orders.status === "fulfilled" ? orders.value : [],
  recs: recs.status === "fulfilled" ? recs.value : [],
};
```
**Reference:** [`Promise.allSettled` (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)

### Bug 5 — fire-and-forget swallows the error
**Root cause:** A returned Promise that nobody awaits or `.catch`-es becomes an unhandled rejection — by default Node logs a warning, and in newer versions terminates the process.
**Fix:**
```typescript
function onUserSignup(user: User) {
  sendWelcomeEmail(user).catch((err) => log.error("welcome-email", err));
  return { ok: true };
}
```
For real fan-out work use a queue, not a dangling promise.
**Reference:** [Node.js `--unhandled-rejections` CLI flag](https://nodejs.org/api/cli.html#--unhandled-rejectionsmode)

### Bug 6 — timer with too-large delay
**Root cause:** Node clamps `setTimeout` delays bigger than 2^31-1 ms (~24.8 days) to 1 ms, so the callback fires almost immediately. Documented behavior, not a "bug".
**Fix:**
```typescript
function scheduleNextRun(cb: () => void, ms: number) {
  const MAX = 2_147_483_647;
  if (ms <= MAX) return setTimeout(cb, ms);
  return setTimeout(() => scheduleNextRun(cb, ms - MAX), MAX);
}
```
For real "run on date X" use a scheduler (cron, Temporal, BullMQ).
**Reference:** [Node.js `setTimeout` docs — delay limits](https://nodejs.org/api/timers.html#settimeoutcallback-delay-args)

### Bug 7 — emitting before the subscriber attaches
**Root cause:** `queueMicrotask` runs at the end of the current synchronous turn — that's *before* the next line in the calling code. The `on("ready")` listener attaches after the event already fired.
**Fix:**
```typescript
const l = new Loader();
l.on("ready", () => console.log("got it")); // subscribe FIRST
l.start();
```
Or change the API to return a Promise.
**Reference:** [Node.js `EventEmitter` docs — synchronous emission](https://nodejs.org/api/events.html#asynchronous-vs-synchronous)

### Bug 8 — `process.nextTick` recursion starves I/O
**Root cause:** The nextTick queue is fully drained between every libuv phase, before any I/O callback runs. Recursive `process.nextTick` chains create an infinite microtask flood — timers and sockets never get a turn.
**Fix:**
```typescript
function drain(queue: Task[]) {
  if (queue.length === 0) return;
  queue.shift()!.run();
  setImmediate(() => drain(queue)); // yields to I/O
}
```
**Reference:** [Node.js — Event Loop, Timers, and `process.nextTick()`](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)

### Bug 9 — `queueMicrotask` vs `setImmediate` confusion
**Root cause:** Microtasks run at the end of the current task before the loop advances. They do not yield to the poll/check phases. Using `queueMicrotask` to "break up" CPU work still blocks I/O.
**Fix:** Use `setImmediate` (runs in the check phase, after I/O callbacks) when you actually need to yield.
```typescript
function chunkedWork(items: Item[]) {
  if (items.length === 0) return;
  process(items.shift()!);
  setImmediate(() => chunkedWork(items));
}
```
**Reference:** [HTML spec — microtask checkpoint](https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint) and [Node.js `setImmediate`](https://nodejs.org/api/timers.html#setimmediatecallback-args)

### Bug 10 — async `EventEmitter` listener swallows errors
**Root cause:** `EventEmitter.emit` is synchronous and ignores listener return values. A rejected Promise from an async listener has no handler.
**Fix:** Wrap or use the captureRejections option:
```typescript
const emitter = new EventEmitter({ captureRejections: true });
emitter[Symbol.for("nodejs.rejection")] = (err) => log.error(err);
emitter.on("data", async (chunk) => { await processChunk(chunk); });
```
Or use `events.on(emitter, "data")` with an async iterator.
**Reference:** [Node.js `EventEmitter` — `captureRejections`](https://nodejs.org/api/events.html#capture-rejections-of-promises)

### Bug 11 — unhandled rejection in a `setTimeout` callback
**Root cause:** Timer callbacks are not awaited. A rejected Promise inside one becomes an `unhandledRejection`, which on modern Node terminates the process.
**Fix:**
```typescript
setTimeout(async () => {
  try {
    log(await fetchSomething());
  } catch (err) {
    log.error("scheduled-fetch", err);
  }
}, 5000);
```
**Reference:** [Node.js `process` event: `unhandledRejection`](https://nodejs.org/api/process.html#event-unhandledrejection)

### Bug 12 — double-resolve when mixing callbacks and promises
**Root cause:** After `reject(err)` the function continues and calls `resolve(data)` (where `data` is `undefined`). Promises ignore the second settle, but the body still ran — harmless here but an instant bug if `data` had side effects. Worse, the `err` branch is masked from anyone reading the code.
**Fix:**
```typescript
return new Promise((resolve, reject) => {
  fs.readFile(path, "utf8", (err, data) => {
    if (err) return reject(err);
    resolve(data);
  });
});
```
Or skip the wrapper: `fs.promises.readFile(path, "utf8")`.
**Reference:** [Node.js `util.promisify`](https://nodejs.org/api/util.html#utilpromisifyoriginal)

### Bug 13 — `AbortSignal` not threaded through the pipeline
**Root cause:** Cancellation only works if every leaf operation honors the signal. Forgetting one leg means aborted requests still consume CPU and downstream quota.
**Fix:**
```typescript
async function search(q: string, signal: AbortSignal) {
  const [a, b] = await Promise.all([
    fetch(`/v1/a?q=${q}`, { signal }),
    fetch(`/v1/b?q=${q}`, { signal }),
  ]);
  return [await a.json(), await b.json()];
}
```
**Reference:** [Node.js `AbortController` / `AbortSignal`](https://nodejs.org/api/globals.html#class-abortcontroller)

### Bug 14 — listener added inside a hot handler
**Root cause:** Each request appends a listener that is never removed. `EventEmitter` warns at 10; eventually you leak handles and memory.
**Fix:** Register listeners once at startup, or use `emitter.once`.
```typescript
process.on("uncaughtException", (e) => log.error(e)); // module scope, once
```
**Reference:** [Node.js `EventEmitter` — `MaxListenersExceededWarning`](https://nodejs.org/api/events.html#emittersetmaxlistenersn)

### Bug 15 — ignoring `stream.write` backpressure
**Root cause:** When the writable's internal buffer is full, `write` returns `false`. Continuing to write piles bytes into memory until you OOM. The whole point of streams is the boolean.
**Fix:** Use `pipeline` (or `pipe`) which handles backpressure for you:
```typescript
import { pipeline } from "node:stream/promises";
await pipeline(src, dest);
```
**Reference:** [Node.js Streams — backpressure / `pipeline`](https://nodejs.org/api/stream.html#stream-backpressuring-in-streams)

### Bug 16 — Buffer concat in a tight loop
**Root cause:** Each `Buffer.concat([acc, chunk])` allocates a brand-new buffer of length `acc.length + chunk.length` and copies both. Over `n` chunks that's O(n²) bytes copied.
**Fix:**
```typescript
const acc = Buffer.concat(chunks); // single pass with known total length
```
Or push chunks into an array and concat once at the end.
**Reference:** [Node.js `Buffer.concat`](https://nodejs.org/api/buffer.html#static-method-bufferconcatlist-totallength)

### Bug 17 — catastrophic backtracking regex on user input
**Root cause:** `^([a-zA-Z0-9]+)+@example\.com$` has nested quantifiers. On input that almost matches but fails at the end, the engine backtracks exponentially. Classic ReDoS — a single request can hang the loop for seconds.
**Fix:** Linear-time matcher, no nested quantifiers:
```typescript
function isLooseEmail(s: string): boolean {
  return /^[a-zA-Z0-9]+@example\.com$/.test(s);
}
```
On Node 20+ consider `RegExp` `v` flag and stay away from nested `+`/`*`. For untrusted input prefer a parser or `re2` (linear-time).
**Reference:** [Don't Block the Event Loop — REDOS section](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)

### Bug 18 — top-level `await` blocking CommonJS interop
**Root cause:** Top-level await makes the ESM module asynchronous. CommonJS `require` is synchronous, so importing an async ESM module from CJS used to throw `ERR_REQUIRE_ASYNC_MODULE` — Node 22 added experimental sync `require(esm)` only for synchronous ESM graphs.
**Fix:** Either drop top-level await and export an init function, or import from an ESM consumer:
```typescript
// db.mjs
export async function getPool() { return cached ??= await createPool(); }
```
**Reference:** [Node.js — `require()` of ESM, `ERR_REQUIRE_ASYNC_MODULE`](https://nodejs.org/api/modules.html#loading-ecmascript-modules-using-require)

### Bug 19 — non-transferable `Buffer` to a Worker
**Root cause:** `worker.postMessage(value)` uses structured clone by default — a 256 MiB Buffer is *copied* into the worker's heap, doubling RSS and pausing both threads during copy. Buffers share an `ArrayBuffer` you can transfer.
**Fix:**
```typescript
worker.postMessage(big, [big.buffer]); // transfer; original is detached
```
For shared mutable memory use `SharedArrayBuffer`.
**Reference:** [Node.js `worker.postMessage(value, transferList)`](https://nodejs.org/api/worker_threads.html#workerpostmessagevalue-transferlist)

### Bug 20 — async_hooks instrumentation slows the whole process
**Root cause:** `createHook({ init })` fires for every async resource creation — promises, timers, sockets, fs ops. Doing any I/O (especially synchronous) inside `init` multiplies work across the entire app. The Node docs explicitly warn about this; the perf impact is large enough that production tracing libraries use AsyncLocalStorage instead.
**Fix:** Prefer `AsyncLocalStorage` for context propagation; avoid `init` callbacks in production. If you must observe, batch and write asynchronously.
**Reference:** [Node.js `async_hooks` — performance considerations](https://nodejs.org/api/async_hooks.html)

### Bug 21 — HTTP keep-alive sockets leak after `uncaughtException`
**Root cause:** The Node docs are explicit: after `uncaughtException` the process is in an undefined state — resource handles, sockets, file descriptors, DB connections may be inconsistent. Continuing to serve on those keep-alive sockets risks corrupted responses, leaked memory, and stuck connections that survive into the next "healthy" period.
**Fix:** Log, drain, and exit. Let the supervisor restart you.
```typescript
process.on("uncaughtException", (err) => {
  log.fatal(err);
  server.close(() => process.exit(1));
  setTimeout(() => process.exit(1), 5_000).unref(); // hard timeout
});
```
**Reference:** [Node.js `process` — `uncaughtException` warning](https://nodejs.org/api/process.html#warning-using-uncaughtexception-correctly)

### Bug 22 — cluster IPC race on graceful shutdown
**Root cause:** `server.close` stops accepting new connections but returns immediately; in-flight requests are still running. `process.disconnect` cuts the IPC channel to the master, which may then think the worker has finished and recycle it before requests complete. Result: 502s and dropped connections during deploys. The Node.js cluster docs and a long history of `nodejs/node` issues document the dance.
**Fix:**
```typescript
process.on("SIGTERM", () => {
  server.close((err) => {
    if (err) log.error(err);
    process.disconnect();
  });
  // optional hard timeout
  setTimeout(() => process.exit(1), 30_000).unref();
});
```
Track in-flight sockets and `socket.destroy()` after the timeout.
**Reference:** [Node.js Cluster — graceful shutdown](https://nodejs.org/api/cluster.html#workerdisconnect) and [nodejs/node issue #2642 (graceful shutdown discussion)](https://github.com/nodejs/node/issues/2642)

## Related
- [Event Loop Internals](./event-loop-internals.md)
- [Worker Threads](./worker-threads.md)
- [V8 Memory and GC](./v8-memory-and-gc.md)
- [Profiling and Debugging](../production/profiling-and-debugging.md)
- [Error Handling Architecture](../patterns/error-handling-architecture.md)

## References
- Node.js docs — Event Loop, Timers, and `process.nextTick()`: https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick
- Node.js docs — Don't Block the Event Loop (or the Worker Pool): https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop
- Node.js API — `timers`: https://nodejs.org/api/timers.html
- Node.js API — `events` (`captureRejections`, max listeners): https://nodejs.org/api/events.html
- Node.js API — `process` (`unhandledRejection`, `uncaughtException`): https://nodejs.org/api/process.html
- Node.js API — `worker_threads`: https://nodejs.org/api/worker_threads.html
- Node.js API — `async_hooks`: https://nodejs.org/api/async_hooks.html
- Node.js API — `stream` (backpressure, `pipeline`): https://nodejs.org/api/stream.html
- Node.js API — `cluster`: https://nodejs.org/api/cluster.html
- Node.js API — Modules and `require(esm)`: https://nodejs.org/api/modules.html
- libuv design overview — event loop phases: https://docs.libuv.org/en/v1.x/design.html
- HTML spec — microtask checkpoint: https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint
- nodejs/node issue #2642 — graceful shutdown semantics: https://github.com/nodejs/node/issues/2642
