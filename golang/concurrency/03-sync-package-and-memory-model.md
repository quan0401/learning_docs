---
title: "The sync Package & the Go Memory Model"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, concurrency, sync, mutex, memory-model, race-detector]
---

# The `sync` Package & the Go Memory Model

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `concurrency` `sync` `mutex` `memory-model` `race-detector`

---

## Table of Contents

1. [The Go memory model in five minutes](#1-the-go-memory-model-in-five-minutes)
2. [`sync.Mutex` and `sync.RWMutex`](#2-syncmutex-and-syncrwmutex)
3. [`sync.WaitGroup`](#3-syncwaitgroup)
4. [`sync.Once`](#4-synconce)
5. [`sync.Pool`](#5-syncpool)
6. [`sync/atomic` and the typed atomics](#6-syncatomic-and-the-typed-atomics)
7. [The race detector](#7-the-race-detector)
8. [Comparison: Java's JMM and TS's single-threaded model](#8-comparison-javas-jmm-and-tss-single-threaded-model)

## Summary

The Go memory model is small and opinionated: a program is correct only if every read of a variable that may be written by another goroutine is ordered against that write by some synchronization operation. Channels, mutexes, `sync.Once`, and `sync/atomic` operations are the synchronizing primitives that establish the **synchronized-before** edges that build a happens-before order. Without one of those edges, two goroutines touching the same variable have no defined ordering — that is a data race, and the race detector will (with high probability, given enough exercise) catch it.

This doc covers the `sync` package APIs you reach for daily — `Mutex`, `RWMutex`, `WaitGroup`, `Once`, `Pool`, and the `sync/atomic` typed wrappers added in Go 1.19 — through the lens of what each one guarantees in the memory model, what it costs, and how the failure modes show up in production.

## 1. The Go memory model in five minutes

The official spec at [go.dev/ref/mem](https://go.dev/ref/mem) is short and worth reading top to bottom. Three claims do most of the work:

**(1) Happens-before is the only ordering you can rely on.** Reads and writes within a single goroutine are sequenced in program order. Across goroutines, ordering exists only where a synchronization operation creates an explicit edge. The relation Go calls **synchronized before** is defined by the `sync` and `sync/atomic` packages and by channel operations.

**(2) Race-free programs see sequential consistency.** The spec calls this DRF-SC: a data-race-free program behaves as some sequentially consistent interleaving of its goroutines. You do not have to reason about reordering, store buffers, or cache lines — as long as every shared access is synchronized.

**(3) Without synchronization, anything is allowed.** The spec gives this example almost verbatim:

```go
var a, b int

func f() {
    a = 1
    b = 2
}

func g() {
    print(b)
    print(a)
}

func main() {
    go f()
    g()
}
```

The spec notes "it can happen that `g` prints `2` and then `0`." The compiler or hardware can reorder `a = 1` and `b = 2` because nothing in `f` requires the order. There is no synchronization edge between `f` and `g`, so `g` may observe a partial, out-of-order view.

The spec is also blunt about what to do with this knowledge:

> If you must read the rest of this document to understand the behavior of your program, you are being too clever. Don't be clever.

**Synchronizing primitives** that establish `synchronized-before` edges:

| Primitive | Edge |
|---|---|
| Channel send / receive | A send is synchronized before the corresponding receive completes. |
| `sync.Mutex` / `sync.RWMutex` | The *n*-th `Unlock` is synchronized before the *m*-th `Lock` returns, for *n* < *m*. |
| `sync.Once` | The single execution of `f()` inside `once.Do(f)` is synchronized before the return of any `once.Do(f)` call. |
| `sync/atomic` operation | If atomic op A's effect is observed by atomic op B, then A is synchronized before B. All atomic ops behave as if executed in some sequentially consistent order. |
| `sync.WaitGroup` | A `Done` (or call to `Add(-1)`) before the counter hits zero is synchronized before the corresponding `Wait` returns. |

Goroutine *exit* is **not** a synchronizing event — the spec is explicit: "the exit of a goroutine is not guaranteed to be synchronized before any event in the program." If you need to observe the result of a goroutine, use a channel, a wait group, or some other primitive — never just "wait a bit and read the variable."

This is the single biggest mental shift from TypeScript. Node's runtime is single-threaded; you cannot have two pieces of JS code observing different orderings of writes to the same variable, because there is only ever one piece of JS code running. In Go, every shared variable is a memory-model question.

It is closer to Java in spirit (see §8) but with a notably smaller set of synchronizing primitives — Java's JMM has to cover `volatile`, `synchronized`, `final`, `Lock`, `AtomicInteger`, thread starts, and thread joins. Go has channels, mutexes, `Once`, atomics, and wait groups, and a corresponding philosophy of "use one of these; don't get clever."

## 2. `sync.Mutex` and `sync.RWMutex`

A `sync.Mutex` is a mutual-exclusion lock. The zero value is an unlocked mutex, ready to use — you do not call `NewMutex`. The two methods you reach for daily are `Lock` and `Unlock`; Go 1.18 added `TryLock`, which most idiomatic code does not need.

```go
type Counter struct {
    mu sync.Mutex
    n  int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.n
}
```

Three idioms are doing real work here:

**Pointer receiver.** `Inc` and `Value` take `*Counter`, not `Counter`. Methods that take a `Counter` by value would copy the embedded `Mutex`, and a copied `Mutex` is a different lock that protects nothing. (See §2 of [Pointers, Methods & Receivers](../fundamentals/07-pointers-methods-receivers.md) for the receiver rules.)

**`defer mu.Unlock()` directly after `mu.Lock()`.** This is the canonical pattern. Even early-return paths and panics will run the deferred unlock, which is what you want. The cost of `defer` was effectively erased in Go 1.14's open-coded defers for the common case — do not micro-optimize this away unless a profile demands it.

**Mutex embedded in the struct it protects.** Standard layout. The mutex sits next to (and in the cache line of) the data it guards.

### `RWMutex`: only reach for it when reads vastly dominate

`sync.RWMutex` allows any number of concurrent `RLock` holders or one exclusive `Lock` holder. It seems strictly better than `Mutex` for read-heavy workloads, but in practice it is *slower than `Mutex`* under low contention because the read-path bookkeeping is more expensive. Reach for `RWMutex` only when:

- The protected critical section is non-trivial work (not just a field load).
- Reads heavily outnumber writes — e.g., a config map read on every request, updated rarely.
- You have benchmarked that `Mutex` is the bottleneck.

`RWMutex` is also not recursive: a goroutine that already holds an `RLock` cannot call `RLock` again under contention without risking deadlock with a waiting writer, and you cannot upgrade `RLock` to `Lock`. The package doc spells this out.

### Common mistakes

**Copying a mutex.** Passing a struct that contains a `Mutex` by value silently breaks locking, because each copy has its own lock. The standard library `vet` tool runs the [copylocks analyzer](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/copylock) by default — it flags lock types passed by value, returned by value, ranged over, or assigned. Always pass things containing locks by pointer.

```go
// vet: passes lock by value: ... contains sync.Mutex
func badPrint(c Counter) { fmt.Println(c.n) }

// fine
func goodPrint(c *Counter) { fmt.Println(c.Value()) }
```

**Forgetting `defer Unlock` and panicking inside the critical section.** If the critical section panics and you unlocked manually after the work, the unlock is skipped and every subsequent `Lock` deadlocks. `defer` is not just convenient; it is panic-safe.

**Lock granularity.** Holding a single mutex around a giant critical section turns a concurrent program into a sequential one. Splitting a struct into fine-grained locks (per shard, per row, per connection) buys back parallelism — at the cost of more lock-ordering discipline. The shard pattern (e.g., `[]struct{mu sync.Mutex; m map[K]V}` keyed by `hash(k) % N`) was the standard answer to "how do I make a high-throughput concurrent map" before `sync.Map`, and is still the right answer for most workloads (`sync.Map` is tuned for narrow access patterns; see its package doc).

**Goroutine-blocking work inside the critical section.** Never hold a `Mutex` while making a network call, an RPC, or a `time.Sleep`. The lock is global to every other goroutine that wants the protected state. Build the result outside the lock, then take the lock only to publish it.

## 3. `sync.WaitGroup`

A `WaitGroup` waits for a set of goroutines to finish. Three methods: `Add(delta)`, `Done()` (which is `Add(-1)`), and `Wait()`. The zero value is ready to use.

```go
var wg sync.WaitGroup
for _, url := range urls {
    wg.Add(1)
    go func(u string) {
        defer wg.Done()
        fetch(u)
    }(u)
}
wg.Wait()
```

Two rules cover most bugs:

**Call `Add` before `go`.** It is tempting to write `Add(1)` *inside* the goroutine, but then `Wait` may run before any goroutine has incremented the counter. `Wait` returns immediately, the program continues, and the goroutines run concurrently with cleanup that assumes they are done. This is a textbook race, and `vet` will flag it via the [`waitgroup` pass](https://pkg.go.dev/cmd/vet) introduced in Go 1.22.

```go
// WRONG
go func() {
    wg.Add(1)        // racy: Wait may have already returned
    defer wg.Done()
    work()
}()

// RIGHT
wg.Add(1)
go func() {
    defer wg.Done()
    work()
}()
```

**Loop-variable capture is fixed in Go 1.22+** — before that, the canonical bug was `for _, u := range urls { go func() { fetch(u) }() }` where every goroutine saw the same final value of `u`. Since [Go 1.22](https://go.dev/blog/loopvar-preview) the `for` clause introduces a fresh variable per iteration, but the older patterns still appear in legacy code; the safe form `func(u string) { ... }(u)` works on any version.

**`WaitGroup.Go` (Go 1.25+).** The `pkg.go.dev/sync` listing shows `Go(f)`: it both increments the counter and runs `f` in a new goroutine, calling `Done` when `f` returns. This compresses the `Add(1)` + `go func()` + `defer Done()` triple into a single call:

```go
var wg sync.WaitGroup
for _, u := range urls {
    wg.Go(func() { fetch(u) })   // Go 1.25+
}
wg.Wait()
```

Use it when available; the older three-line pattern is not going away.

`WaitGroup` does not aggregate errors — pair it with an error channel, or use [`golang.org/x/sync/errgroup`](https://pkg.go.dev/golang.org/x/sync/errgroup) which adds context cancellation on first error. `errgroup` is the right choice for almost any "fan out N goroutines, fail fast on the first error" pattern.

## 4. `sync.Once`

`Once.Do(f)` runs `f` exactly once across all goroutines. Subsequent calls return immediately, *and* every caller has a happens-before guarantee that `f`'s writes are visible.

```go
var (
    initOnce sync.Once
    cfg      *Config
)

func Cfg() *Config {
    initOnce.Do(func() {
        cfg = loadConfig()
    })
    return cfg
}
```

The Go memory model guarantees: "the completion of a single call of `f()` from `once.Do(f)` is synchronized before the return of any call of `once.Do(f)`." Translation: callers that race into `Cfg` for the first time will all observe the fully-constructed `cfg`, with all of its fields visible.

Two notes:

- `Once` does not retry on panic. If `f` panics, `Once` records that `f` ran; subsequent calls see `cfg == nil` and do not retry. If retry on failure matters, write your own initialization with a `sync.Mutex` and an explicit "initialized" flag.
- Go 1.21 added `sync.OnceFunc`, `sync.OnceValue`, and `sync.OnceValues`, which return memoizing closures. They are convenient when you want lazy initialization without a package-level `sync.Once`. The package doc covers them.

## 5. `sync.Pool`

`sync.Pool` is a free-list of reusable values, scoped to reduce allocator and GC pressure. `Get` returns an existing value or calls `New`; `Put` returns a value to the pool.

```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func renderJSON(w io.Writer, v any) error {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufPool.Put(buf)

    if err := json.NewEncoder(buf).Encode(v); err != nil {
        return err
    }
    _, err := w.Write(buf.Bytes())
    return err
}
```

What `Pool` is **not**:

- **Not a connection pool.** It is fine to drop pooled objects on the floor at any time; the GC may reclaim entries between calls. Connections, file handles, and other resources with a close obligation must not live in `sync.Pool`. Use `database/sql`'s pool, an `http.Transport`, or a dedicated pool library.
- **Not a leak-proof store.** Anything not retrieved before the next GC cycle may vanish. The 1.13 release introduced a *victim cache*: items that survive one GC cycle are stashed and may survive a second cycle — this smooths spikes in workloads with bursty allocation patterns. The package doc covers this.
- **Not always a win.** Pooling tiny objects often costs more than allocating them, because pool bookkeeping is per-P sharding plus atomic operations. Profile before pooling. The classic high-leverage cases are large `[]byte` or `bytes.Buffer` objects in hot encode/decode paths — `encoding/json` uses pools internally.

The discipline: always `Reset` (or otherwise zero out) on `Get`, never assume `Put` keeps your value, and never put something into a pool you'd be unhappy to lose.

## 6. `sync/atomic` and the typed atomics

`sync/atomic` provides lock-free read/write/CAS on integer and pointer-sized values. The package guarantees sequentially consistent semantics: from the doc, "if the effect of an atomic operation A is observed by atomic operation B, then A 'synchronizes before' B. Additionally, all the atomic operations executed in a program behave as though executed in some sequentially consistent order." That is the same guarantee Java gives `volatile` variables and C++ gives `memory_order_seq_cst` atomics.

### Typed atomics (Go 1.19)

Before Go 1.19 you used `atomic.AddInt64(&x, 1)` and friends, with `&x` being a raw `*int64`. It worked, but two footguns dogged it: misaligned `int64` access on 32-bit platforms, and the ability to accidentally read or write `x` non-atomically in some other code path.

Go 1.19 added typed wrappers — `atomic.Bool`, `atomic.Int32`, `atomic.Int64`, `atomic.Uint32`, `atomic.Uint64`, `atomic.Uintptr`, and the generic `atomic.Pointer[T]`. The release notes note two ergonomic wins: "these types hide the underlying values so that all accesses are forced to use the atomic APIs," and `Int64`/`Uint64` are guaranteed 64-bit-aligned even on 32-bit platforms. `Pointer[T]` removes the `unsafe.Pointer` dance for atomic pointer swaps.

```go
type Server struct {
    requests atomic.Int64
    config   atomic.Pointer[Config]
}

func (s *Server) handle() {
    s.requests.Add(1)
    cfg := s.config.Load()  // *Config, type-safe
    use(cfg)
}

func (s *Server) reload(c *Config) {
    s.config.Store(c)       // publication: a subsequent Load on any goroutine sees c
}
```

Use the typed atomics for any new code. Mixing atomic and non-atomic access on the same variable is a race; the wrappers prevent that statically.

### When atomics, when a mutex

Atomics are appropriate when the operation is a single read, a single write, an `Add`, or a `CompareAndSwap` on a fixed-size value. The moment you need to read multiple fields consistently, or to do a read-modify-write on a non-trivial object, you need a mutex. `atomic.Pointer[T]` plus copy-on-write (build a new `*Config`, then `Store` it) is a powerful pattern for read-heavy state with infrequent updates — it is what `atomic.Value`'s typical use case looked like before generics.

The Go FAQ entry on channels-vs-mutexes (linked in the previous doc) closes with: "use whichever is most expressive and/or most simple." Atomics are the smallest, most expressive primitive when they fit; reach for them, but do not turn every counter into an `atomic.Int64` if a mutex makes the code clearer.

## 7. The race detector

`go test -race`, `go run -race`, `go build -race`. The race detector instruments memory accesses and channel/sync events; if two goroutines access the same memory location with at least one write and no synchronization edge between them, the race detector prints both stack traces and the goroutine creation sites.

From [the official article](https://go.dev/doc/articles/race_detector):

- **Memory cost: ~5–10x. CPU cost: 2–20x.** Run race-enabled binaries in CI and in load tests, not in production. Even so, many large Go shops run a race-instrumented build of their main service in a small canary slice in staging — the cost is real but tolerable for a non-production tier.
- **It only finds races that actually execute.** If a race needs a specific input, scheduling, or rare condition, you have to exercise it. This is why race-aware integration and end-to-end tests pay back.
- **Default exit code is 66** when a race is reported. Fail your CI on it. Configure with the `GORACE` env var (`exitcode`, `halt_on_error`, etc.).

Interpreting a report: the first stack is the offending read/write; the second stack is the prior conflicting access; the goroutine-creation lines tell you who started each goroutine. Look at the synchronization between them — usually nothing — and decide whether the right fix is a mutex around the field, a channel for ownership transfer, or an atomic for the single counter.

What the race detector **does not** find: deadlocks (nothing accesses the variable at all), goroutine leaks (no race; the goroutine simply never exits), or logic bugs in your synchronization (a mutex held but for the wrong field). For deadlocks the runtime detects "all goroutines asleep" and panics; for leaks, see §6 of [Channels in Practice](02-channels-in-practice.md).

A note on terminology: a Go data race is a runtime *fatal error*, not a regular panic — `recover()` does not catch it. The runtime prints the report and aborts. This is intentional: the program is now in undefined territory by the spec, and continuing risks corruption.

## 8. Comparison: Java's JMM and TS's single-threaded model

**Java's JMM.** Conceptually similar — happens-before plus synchronizing actions — but the surface is larger. `volatile` reads and writes are synchronizing; `synchronized` blocks pair with monitor enter/exit; `Thread.start` synchronizes-before the first action of the thread; `Thread.join` synchronizes-after the last action of the joined thread; `final` fields have special "construction visibility" rules. `java.util.concurrent.atomic.AtomicLong` is the closest cousin to `atomic.Int64`. The Go memory model is deliberately the smaller set: channels, mutexes, atomics, `Once`, `WaitGroup`. The Go 1.19 memory-model revision (mentioned in the [Go 1.19 release notes](https://go.dev/doc/go1.19)) explicitly aligned Go's atomics with the C/C++/Java/Rust/Swift sequentially-consistent baseline, which makes cross-language porting simpler.

If you are coming from Spring Boot: a Spring service relies heavily on the JMM through `synchronized`, `volatile`, `ConcurrentHashMap`, and `@Transactional` semantics. The Go equivalents are `sync.Mutex`, `atomic.Bool`/`atomic.Pointer[T]`, `sync.Map` (or sharded mutex maps), and the database driver's `*sql.Tx`. The mental load is comparable.

**TypeScript / Node.** Single-threaded JS means there is no JMM analog *within a process* — the event loop guarantees that no two pieces of JS code interleave. Workers and SharedArrayBuffer change that picture: when you cross into them, you suddenly need atomics (`Atomics.load`, `Atomics.store`, `Atomics.compareExchange`) and the spec around them looks much more like Go's memory model. Most Node services never hit that surface.

The practical implication is that a TS engineer learning Go should treat *every* shared variable as a problem, the way they would treat shared variables across worker threads — not the way they treat a module-level cache in Node. The good news is that Go's APIs are small and the race detector is excellent, so the transition is mostly mechanical: add a mutex (or pick a channel) at every shared access, run with `-race`, fix the reports, and you converge.

The next doc covers `context.Context`, the cross-cutting cancellation and deadline API that ties this all together at request boundaries.

## Related

- [Channels in Practice](02-channels-in-practice.md) — channels are the *other* synchronizing primitive in the memory model
- [Goroutines & the Scheduler (G-M-P)](01-goroutines-and-scheduler.md) — what's running across the synchronization edges
- [Slices, Arrays & Maps](../fundamentals/04-slices-arrays-maps.md) — concurrent map writes are a fatal runtime error; this is why
- [Pointers, Methods & Receivers](../fundamentals/07-pointers-methods-receivers.md) — why you must use pointer receivers around `Mutex`
- [Errors as Values](../fundamentals/06-errors-as-values.md) — error handling around critical sections
- [TypeScript event loop internals](../../typescript/runtime/event-loop-internals.md) — single-threaded contrast
- [Process & thread model](../../operating-systems/fundamentals/01-process-thread-model.md) — what the OS thinks Go is doing under the hood

## References

- The Go Memory Model — https://go.dev/ref/mem
- `sync` package — https://pkg.go.dev/sync
- `sync/atomic` package — https://pkg.go.dev/sync/atomic
- "Data Race Detector" — https://go.dev/doc/articles/race_detector
- Go 1.7 release notes (context promoted to stdlib) — https://go.dev/doc/go1.7
- Go 1.19 release notes (typed atomics, memory-model revision) — https://go.dev/doc/go1.19
- Go 1.22 release notes (loopvar fix) — https://go.dev/doc/go1.22
- `golang.org/x/tools/go/analysis/passes/copylock` — https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/copylock
- `golang.org/x/sync/errgroup` — https://pkg.go.dev/golang.org/x/sync/errgroup
- "Loop variables in Go 1.22" — https://go.dev/blog/loopvar-preview
