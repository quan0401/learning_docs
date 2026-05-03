---
title: "Escape Analysis & Allocation"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, runtime, escape-analysis, allocation, performance, optimization]
---

# Escape Analysis & Allocation

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `runtime` `escape-analysis` `allocation` `performance` `optimization`

---

## Table of Contents

1. [Stack vs heap in Go](#1-stack-vs-heap-in-go)
2. [The basic escape rules](#2-the-basic-escape-rules)
3. [Reading `-gcflags='-m'` output](#3-reading--gcflags-m-output)
4. [Common allocation sources you can fix](#4-common-allocation-sources-you-can-fix)
5. [`sync.Pool` for object reuse](#5-syncpool-for-object-reuse)
6. [Profiling allocations with pprof](#6-profiling-allocations-with-pprof)
7. [When **not** to optimize](#7-when-not-to-optimize)
8. [Tools: `go test -benchmem`, `pprof`, linters](#8-tools-go-test--benchmem-pprof-linters)

## Summary

Go has two places memory can live: the **stack** (one per goroutine, free to allocate, freed when the function returns) and the **heap** (managed by the GC, expensive to allocate, freed only when the collector says so). The **compiler** decides which by running an **escape analysis** pass. You don't manage memory explicitly, but you can read the analysis output, see why values escape, and remove escapes that don't need to happen.

This is where most Go performance work lives. The collector covered in [the previous doc](01-garbage-collector.md) is fast, but the cheapest allocation is the one that never happens. Reading `-gcflags='-m'` output, profiling with `pprof -alloc_space`, and applying a small set of patterns (avoid interface boxing in hot paths, preallocate slices and maps, pool reusable buffers, use `strings.Builder`/`bytes.Buffer` instead of `+`) gets you most of the way to allocation-efficient Go.

A warning before you optimize: this is exactly the area where engineers waste effort. **Profile first.** Allocation-driven micro-optimization without a profile is how you write uglier code that's no faster. We'll cover the pattern of doing it right at the end.

## 1. Stack vs heap in Go

Every goroutine has a **stack** that grows and shrinks dynamically (covered in [the goroutines doc](../concurrency/01-goroutines-and-scheduler.md)). Stack allocation is essentially free — bumping a pointer. Stack memory is reclaimed automatically when the function returns; the GC never sees it.

The **heap** is shared across all goroutines and managed by the garbage collector. Heap allocation goes through the runtime allocator (size-segregated spans, similar to TCMalloc), which is fast for small objects but not free. More importantly, every byte allocated on the heap eventually becomes scan work for the collector and growth on the heap-trigger budget — so allocations don't just cost the allocation, they cost a fraction of every future GC cycle for as long as the object lives.

The compiler's job during escape analysis is to prove, statically, that a value's lifetime is bounded by its containing function frame. If the proof succeeds, the value goes on the stack. If the proof fails — or the analysis is conservative and refuses to try — the value **escapes** to the heap.

```go
func sum() int {
    nums := []int{1, 2, 3}   // backing array on stack — never escapes
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

func leak() *int {
    x := 42
    return &x   // x escapes — must outlive the function frame
}
```

The Go compiler's escape analysis lives in [`cmd/compile/internal/escape`](https://github.com/golang/go/blob/master/src/cmd/compile/README.md) and runs as part of the middle-end optimization passes. It's a **flow-sensitive interprocedural analysis** with deliberate limits — it gives up on hard cases and over-approximates (escapes when uncertain) rather than risking unsafety.

## 2. The basic escape rules

The rules below are heuristics, not a complete spec. The compiler is more sophisticated than this list — and it changes between releases — but these capture 95% of what you'll see in practice.

| Pattern | Result |
|---|---|
| Take address with `&`, use only inside the function | stays on stack |
| Take address with `&`, **return** the pointer | escapes |
| Take address with `&`, store in a global, channel, slice element, or struct field that itself escapes | escapes |
| Pass by value to another function | typically stays |
| Pass by pointer to another function | depends on what the callee does (interprocedural) |
| Store a value into an `interface{}` / `any` variable | usually escapes (interface holds a pointer) |
| Capture a variable in a closure that's returned, stored, or passed to a goroutine | escapes |
| `make([]T, n)` where `n` is a non-constant or large constant | escapes (allocator chooses) |
| `make([]T, n)` where `n` is a small constant and the slice doesn't escape | may stay on stack |
| `new(T)` with the result not stored beyond the frame | may stay on stack despite the name |

A few of these deserve unpacking.

**Interface assignment usually escapes.** When you do `var x any = 42`, the runtime needs to box the integer behind a pointer because an `any` is a (type-pointer, data-pointer) pair. If the data is small enough and the compiler can see the assignment is local-only, it can sometimes avoid the heap allocation, but in practice — anywhere the interface might escape, like passing to `fmt.Println`, calling a method, returning — the boxing allocates.

```go
func format(x int) string {
    return fmt.Sprintf("%d", x)   // x escapes via the variadic ...any
}
```

This is one of the most common avoidable allocations in production Go code. Hot-path logging with `fmt.Sprintf` boxes every argument.

**Closure capture by goroutine always escapes.** If `go f()` captures variables from the enclosing frame, those variables must outlive the frame, so they go to the heap.

```go
func handle(req *Request) {
    go func() {
        process(req)   // req captured — escapes
    }()
}
```

**`new(T)` is not always heap.** Despite its C++ heritage, `new(T)` in Go does not force a heap allocation. If the result doesn't escape, the compiler will stack-allocate it just like `&T{}` would. This was [explicitly designed that way in the FAQ](https://go.dev/doc/faq#new_and_make) — the allocator chooses, not the keyword.

## 3. Reading `-gcflags='-m'` output

The compiler will tell you what it decided. Pass `-gcflags='-m'` to `go build`, `go test`, or `go run`:

```bash
go build -gcflags='-m' ./...
```

The output looks like:

```text
./main.go:12:6: can inline newCounter
./main.go:14:9: &c escapes to heap
./main.go:14:9: moved to heap: c
./main.go:20:13: ... argument does not escape
./main.go:25:6: leaking param: x
```

The vocabulary, decoded:

| Phrase | Meaning |
|---|---|
| `can inline FUNC` | Compiler will inline this function — relevant because inlining changes escape decisions for callers |
| `&X escapes to heap` | Taking the address caused X to be promoted |
| `moved to heap: X` | The variable's storage was placed on the heap |
| `does not escape` | Compiler proved the value/pointer doesn't outlive the frame |
| `leaking param: X` | Parameter X (or something derived from it) escapes — usually because it's stored or returned |
| `X escapes to heap: TYPE` | Some part of a more complex expression caused the escape |

Increase verbosity with `-m=2` to get reasons for each decision:

```bash
go build -gcflags='-m=2' ./...
```

A worked example. Save as `escape.go`:

```go
package main

import "fmt"

type User struct {
    ID   int
    Name string
}

func makeUser(id int, name string) *User {
    return &User{ID: id, Name: name}
}

func printID(u User) {
    fmt.Println(u.ID)
}

func main() {
    u := makeUser(1, "alice")
    printID(*u)
}
```

Run:

```bash
go build -gcflags='-m' escape.go
```

Output (lightly trimmed):

```text
./escape.go:10:6: can inline makeUser
./escape.go:11:9: &User{...} escapes to heap
./escape.go:14:6: can inline printID
./escape.go:15:13: ... argument does not escape
./escape.go:19:16: inlining call to makeUser
./escape.go:19:16: &User{...} escapes to heap
./escape.go:20:11: *u does not escape
```

Reading it: `makeUser` returns `&User{...}`, so the `User` literal escapes — it must outlive the function. After inlining, the same conclusion is drawn at the call site (line 19). Passing `*u` (a copy by value) into `printID` does not escape, but `fmt.Println` is variadic over `any`, and the interface boxing for `u.ID` produces an additional escape that you'd see if you ran `-m=2`.

## 4. Common allocation sources you can fix

A short catalog. Each entry is a real pattern that shows up frequently in production Go code.

**String concatenation in loops.** Strings are immutable in Go. `s += part` allocates a new string each time.

```go
// BAD: O(n²) allocations
s := ""
for _, p := range parts {
    s += p
}

// GOOD: one growing buffer, amortized O(n)
var b strings.Builder
b.Grow(estimatedSize)   // optional, avoids reallocs
for _, p := range parts {
    b.WriteString(p)
}
s := b.String()
```

**`fmt.Sprintf` in hot paths.** Boxing every argument plus parser overhead.

```go
// BAD: hot path allocates 2 boxed any values per call
key := fmt.Sprintf("user:%d:session:%d", userID, sessID)

// GOOD: strconv + concatenation, no boxing
key := "user:" + strconv.Itoa(userID) + ":session:" + strconv.Itoa(sessID)
```

For frequent integer-to-string conversion, `strconv.Itoa`/`strconv.AppendInt` are the right tool; `fmt.Sprintf` is the wrong one.

**Interface boxing of small values.** Putting an `int` into an `any` allocates a pointer-sized header and a heap-allocated copy.

```go
// BAD: every value is boxed when stored
var cache []any
for _, n := range nums {
    cache = append(cache, n)
}

// GOOD: typed slice, no boxing
var cache []int
for _, n := range nums {
    cache = append(cache, n)
}
```

In Go 1.18+, generics let you write the typed version without losing reusability. Avoid `any` in storage that has a known type.

**Slice growth without preallocation.** Each `append` past capacity doubles the backing array, copying the contents — see [the slices doc, §3](../fundamentals/04-slices-arrays-maps.md). When you know the final length, hint it.

```go
// BAD: starts at cap=0, grows by doubling
out := []User{}
for _, row := range rows {
    out = append(out, parseUser(row))
}

// GOOD: one allocation
out := make([]User, 0, len(rows))
for _, row := range rows {
    out = append(out, parseUser(row))
}
```

The same applies to maps — `make(map[K]V, expectedSize)`.

**Returning a tiny subslice from a giant buffer.** The backing array of a slice is kept alive by any subslice. If you parse a 100 MB document and return a few-byte slice, the 100 MB stays in memory. Discussed in detail in [the slices doc, §5](../fundamentals/04-slices-arrays-maps.md). Fix by copying when retaining.

**Struct-vs-pointer choice in hot paths.** A `[]Point` of small structs is often faster than `[]*Point` because the values live contiguously, the slice doesn't need a pointer-write-barrier hit per element during marking, and no per-element allocation is needed.

**`map[string]X` where the key is constructed per lookup.** `m[fmt.Sprintf("%d-%d", a, b)]` allocates a string per call. Either use a struct key (`map[struct{ a, b int }]X`) or use a struct-keyed `sync.Map` if concurrent access is needed.

The Segment engineering team's "Allocation efficiency in high-performance Go services" article walks through several of these patterns with real before/after profiles — the URL has moved with the company's rebrand to Twilio Segment, but the article is still cited widely under the original title.

## 5. `sync.Pool` for object reuse

`sync.Pool` is the standard library's tool for reusing short-lived heap allocations. Conceptually it's a free list with per-P caching and GC-aware eviction.

```go
import "sync"

var bufPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func handle(w http.ResponseWriter, r *http.Request) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf)
    }()

    // ... use buf ...
}
```

Three rules to use it correctly:

1. **Reset before reuse.** Pool entries retain whatever state they had when `Put` was called. Always reset — `buf.Reset()` for `bytes.Buffer`, `b = b[:0]` for slices.
2. **Pool only what's safe to reuse.** Anything caller-visible after `Put` is a bug. Pooling structures with embedded mutexes, pointers to caller-owned data, or finalizer-bearing types is dangerous.
3. **Don't pool everything.** Pool wins when allocation rate is high *and* objects are non-trivial to construct. For tiny structs the allocator is faster than the pool's atomic operations.

`sync.Pool` items can be evicted at any GC. The pool is not a cache for state retention — it's an allocator amortization tool. If you need eviction-resistant caching, use a real cache library, not `sync.Pool`.

This will be revisited in the Tier 2 sync primitives doc.

## 6. Profiling allocations with pprof

The path from "GC CPU is high" to "fix the right allocation" runs through pprof. Two profile types are relevant:

**Allocation profile** (`-alloc_space`, `-alloc_objects`): every allocation since program start, by stack. Useful for finding allocation rate hotspots — code that allocates a lot, even if those allocations don't accumulate.

**Heap profile** (`-inuse_space`, `-inuse_objects`): live objects right now, by allocation stack. Useful for finding memory leaks and retained-but-unused state.

The distinction matters. A function that allocates 10 million strings that are immediately discarded shows up in `-alloc_space` (high allocation rate, GC pressure) but not in `-inuse_space` (the strings die quickly). A function that allocates a 1 GB buffer once and never frees it shows up in `-inuse_space` but barely in `-alloc_space`.

From the [official "Profiling Go Programs"](https://go.dev/blog/pprof) post, the typical workflow:

```bash
# In a benchmark
go test -bench=. -memprofile=mem.prof ./mypackage

# Analyze
go tool pprof -alloc_space mem.prof
(pprof) top
(pprof) list FunctionName
(pprof) web        # opens flame graph in browser

# Or, against a running server with net/http/pprof mounted
go tool pprof http://localhost:6060/debug/pprof/heap
```

The [`runtime/pprof`](https://pkg.go.dev/runtime/pprof) docs catalog the profile types: `goroutine`, `allocs`, `heap`, `threadcreate`, `block`, `mutex`. For allocation analysis, `allocs` (== `-alloc_space` view) is what you want.

The flame graph (`pprof -http=:8080 mem.prof`) is the right starting point. Wide stacks at the top of the flame graph are where allocation is concentrated. From there, `list FUNCTION` shows the source line by line with annotated allocation totals — this is the fastest path from "the GC is busy" to "this specific line is allocating."

```bash
(pprof) list parseUser
Total:     128MB
ROUTINE ======================== example.com/svc/user.parseUser
       0     128MB (flat, cum) 100%
         .          .     14:func parseUser(row Row) *User {
       0       64MB     15:    name := fmt.Sprintf("%s %s", row.First, row.Last)  // ← here
         .          .     16:    return &User{Name: name, ID: row.ID}
         .          .     17:}
```

That's the loop you want.

## 7. When **not** to optimize

The single most expensive failure mode in Go performance work is **optimizing without a profile**. The pattern looks like:

1. Engineer reads a blog post about avoiding allocations.
2. Engineer rewrites a function to use `strings.Builder` and `sync.Pool`.
3. The function is called 100 times an hour and was never on the critical path.
4. The code is uglier, the benchmark unchanged, and a future reader has to understand the optimization to change anything.

Before optimizing allocations, demand evidence:

- A profile showing the function on the hot path (>1% of allocation, ideally >5%).
- A benchmark that reproduces the hot-path call pattern.
- A measurement that the optimization actually moves the metric you care about.

The metric matters. Reducing `B/op` in a benchmark is satisfying but irrelevant if your tail latency is bound by network I/O or database calls. Reducing GC CPU from 8% to 4% on a service whose CPU bottleneck is JSON parsing changes nothing visible to users. Look at the SLO before you look at the profile.

A useful default rule: **don't optimize allocations until GC CPU exceeds 10% sustained, or the heap profile shows a specific function dominating.** Below that threshold, the cost of more complex code outweighs the benefit, and you'll spend your optimization budget where it doesn't matter.

The exceptions are libraries on the hot path of every caller (logging, serialization, HTTP routing) — those have economic leverage that makes per-allocation optimization worthwhile even when each individual call is small. For application code, profile first.

## 8. Tools: `go test -benchmem`, `pprof`, linters

The toolkit, with the entry point for each:

**`go test -benchmem`.** Runs benchmarks and reports allocations per operation alongside time. From the [`testing`](https://pkg.go.dev/testing) package docs, `-benchmem` enables `B/op` (bytes per op) and `allocs/op` columns. The same effect can be enabled for a single benchmark with `b.ReportAllocs()`.

```go
func BenchmarkParseUser(b *testing.B) {
    b.ReportAllocs()
    row := makeRow()
    for i := 0; i < b.N; i++ {
        _ = parseUser(row)
    }
}
```

```text
BenchmarkParseUser-8    5_000_000    240 ns/op    96 B/op    2 allocs/op
```

This is the right granularity for "did my change reduce allocations?" Run before and after with `benchstat`:

```bash
go test -bench=. -benchmem -count=10 ./mypackage > before.txt
# (apply change)
go test -bench=. -benchmem -count=10 ./mypackage > after.txt
benchstat before.txt after.txt
```

`benchstat` (`golang.org/x/perf/cmd/benchstat`) reports the delta with statistical confidence. A 5% improvement in a noisy benchmark is not real until `benchstat` says so.

**`pprof`.** Covered in §6. Use `go tool pprof` for the CLI, `pprof -http=:8080 file.prof` for the web UI with flame graphs. The Google [`pprof`](https://github.com/google/pprof) repository hosts the underlying tool — Go ships its own bundled copy.

**`-gcflags='-m'`.** Covered in §3. The compiler-level view: which decisions did it make and why.

**Linters.** `go vet` doesn't catch allocation issues directly, but `golangci-lint` aggregates several allocation-aware linters:
- `prealloc` — flags `append`-in-loop where capacity could be hinted.
- `ineffassign` — flags assignments whose value is never read.
- `gocritic` — broader style checks including some allocation patterns.

Run them in CI; treat their output as suggestions in production code (most are heuristic) and as guides during optimization passes.

**Beyond the standard tools:** runtime tracing (`runtime/trace`, viewed with `go tool trace`) lets you see allocation timing relative to GC cycles and goroutine activity. It's the right tool when allocations correlate with latency spikes and you want to see the temporal pattern, not just the call-stack distribution.

## Related

- [Go's Garbage Collector](01-garbage-collector.md) — what the heap costs once you've allocated on it
- [Pointers, Methods & Receivers — escape analysis primer](../fundamentals/07-pointers-methods-receivers.md)
- [Slices, Arrays & Maps — preallocation, slice retention](../fundamentals/04-slices-arrays-maps.md)
- [Goroutines & the Scheduler — goroutine stacks](../concurrency/01-goroutines-and-scheduler.md)
- [Operating Systems — Virtual Memory & Paging](../../operating-systems/fundamentals/02-virtual-memory-and-paging.md)
- [Operating Systems — Page Cache & Buffered I/O](../../operating-systems/fundamentals/03-page-cache-and-buffered-io.md)
- [Performance INDEX](../../performance/INDEX.md) — when allocation reduction actually moves user-visible metrics

## References

- Go compiler README, escape analysis pass — https://github.com/golang/go/blob/master/src/cmd/compile/README.md
- "A Guide to the Go Garbage Collector" — https://tip.golang.org/doc/gc-guide
- "Profiling Go Programs" (official Go blog) — https://go.dev/blog/pprof
- `runtime/pprof` package — https://pkg.go.dev/runtime/pprof
- `net/http/pprof` package — https://pkg.go.dev/net/http/pprof
- `pprof` tool — https://github.com/google/pprof
- `testing` package, "Benchmarks" — https://pkg.go.dev/testing#hdr-Benchmarks
- `sync.Pool` — https://pkg.go.dev/sync#Pool
- Go FAQ, "When are function parameters passed by value?" / "What's the difference between new and make?" — https://go.dev/doc/faq#new_and_make
- "Allocation efficiency in high-performance Go services" by Achille Roussel (originally at segment.com/blog, now redirects post-Twilio acquisition) — https://segment.com/blog/allocation-efficiency-in-high-performance-go-services/
- `golang.org/x/perf/cmd/benchstat` — https://pkg.go.dev/golang.org/x/perf/cmd/benchstat
