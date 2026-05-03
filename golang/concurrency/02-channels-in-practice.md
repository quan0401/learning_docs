---
title: "Channels in Practice"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, concurrency, channels, select, patterns, goroutine-leaks]
---

# Channels in Practice

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `concurrency` `channels` `select` `patterns` `goroutine-leaks`

---

## Table of Contents

1. [Channels: typed conduits between goroutines](#1-channels-typed-conduits-between-goroutines)
2. [Unbuffered vs buffered: synchronization vs queueing](#2-unbuffered-vs-buffered-synchronization-vs-queueing)
3. [Send, receive, close — and the four channel axioms](#3-send-receive-close--and-the-four-channel-axioms)
4. [`select`: non-deterministic dispatch](#4-select-non-deterministic-dispatch)
5. [Patterns: pipeline, fan-in, fan-out, worker pool, done channel](#5-patterns-pipeline-fan-in-fan-out-worker-pool-done-channel)
6. [Goroutine leaks](#6-goroutine-leaks)
7. [When not to use channels](#7-when-not-to-use-channels)

## Summary

Channels are typed conduits over which goroutines exchange values. They are the canonical Go concurrency primitive, and the runtime guarantees that operations on them are safe across goroutines without further locking. The mental model has two halves: an **unbuffered** channel synchronizes a sender and receiver — the send completes only when the receive completes — while a **buffered** channel decouples them by holding up to N values. The `select` statement multiplexes channel operations and is how Go expresses "wait on any of these events."

Most channel bugs in production Go services are not about channels themselves; they are about goroutine leaks caused by a goroutine blocked forever on a channel that no one will send to or receive from. The four channel axioms (§3) plus careful ownership of close are the antidote.

## 1. Channels: typed conduits between goroutines

A channel is created with `make`. It is a typed pointer-like value (a few words on the stack, with the underlying buffer on the heap) — passing a channel to a goroutine shares the same channel.

```go
ch := make(chan int)        // unbuffered
buf := make(chan int, 16)   // buffered, capacity 16

ch <- 42        // send
v := <-ch       // receive
v, ok := <-ch   // receive with ok=false on closed-and-drained
close(ch)       // close
```

Direction can be encoded in the type, useful for documenting intent at function boundaries:

```go
func produce(out chan<- int) { /* send-only */ }
func consume(in  <-chan int) { /* receive-only */ }
func bidirectional(c chan int) { /* read and write */ }
```

A bidirectional channel is implicitly assignable to either directional type — the compiler narrows it. The reverse is not allowed.

The zero value of a channel is `nil`. Operations on a nil channel block forever (see §3, axiom 1 and 2) — this is a feature, used to disable a `case` in a `select`.

## 2. Unbuffered vs buffered: synchronization vs queueing

This is the most important distinction.

**Unbuffered channels (`make(chan T)`)** are a **synchronization point**. A send blocks until a receiver is ready; a receive blocks until a sender is ready. The two operations complete simultaneously. There is no buffer — the value moves from sender to receiver directly.

```go
ch := make(chan string)
go func() {
    ch <- "hello"  // blocks until main goroutine receives
}()
fmt.Println(<-ch)
```

The use case: you actually want the sender and receiver to **rendezvous**. Either side that gets there first waits for the other. This is how you express "I'm done" or "the other side has acknowledged."

**Buffered channels (`make(chan T, N)`)** decouple sender and receiver via a queue:
- Sends succeed immediately as long as the buffer is not full.
- Receives succeed immediately as long as the buffer is not empty.
- Once the buffer is full, sends block; once empty, receives block.

```go
work := make(chan Job, 100)
work <- job1   // returns immediately (buffer has room)
work <- job2   // returns immediately
// ...
job := <-work  // returns immediately (buffer has data)
```

The use case: smoothing burstiness, allowing producers and consumers to run at slightly different rates without blocking each step.

The choice is a design decision, not a "buffered is better." Common errors:
- Picking a buffer size of 1 to "avoid blocking" — usually a code smell, often hiding a goroutine leak.
- Picking a large buffer to "avoid backpressure" — usually wrong; you've just moved the unbounded problem from one place to another. If your producer outpaces your consumer indefinitely, the buffer fills and you're back to blocking, with extra memory consumed in the meantime.

A good heuristic: use unbuffered by default, and reach for a buffered channel when you have a specific reason — e.g., a known burst size, a benchmarked performance need, or a specific decoupling pattern (worker pool, fan-out).

## 3. Send, receive, close — and the four channel axioms

The four axioms, as articulated by Dave Cheney and codified in the Go specification:

| Operation | On nil channel | On open channel | On closed channel |
|---|---|---|---|
| Send `ch <- v` | **blocks forever** | blocks until receiver | **panics** |
| Receive `<-ch` | **blocks forever** | blocks until sender (or buffered value) | returns zero value immediately |
| Close `close(ch)` | **panics** | succeeds | **panics** (already closed) |

Three concrete consequences:

**Receiving from a closed channel returns the zero value of the element type, with `ok == false`.**

```go
ch := make(chan int)
close(ch)
v, ok := <-ch
fmt.Println(v, ok)  // 0 false
```

This is how `range ch` knows when to stop:

```go
for v := range ch {
    use(v)
}
// loop exits cleanly when ch is closed and drained
```

**The sender closes the channel, never the receiver.** Receivers cannot tell whether more sends will happen. Senders know. Closing from the receive side races against the sender and produces a panic.

For multi-sender topologies, no single sender can safely close (because another might still send). The conventional fix is a separate "done" or "stop" channel, plus a `sync.WaitGroup` that the sender that closes the data channel waits on:

```go
var wg sync.WaitGroup
results := make(chan Result)

for _, w := range workers {
    wg.Add(1)
    go func(w Worker) {
        defer wg.Done()
        results <- w.Do()
    }(w)
}

go func() {
    wg.Wait()      // wait until every sender has finished
    close(results) // safe: no more sends possible
}()

for r := range results {
    use(r)
}
```

**You do not have to close every channel.** A channel is garbage-collected when no goroutine references it. Close is required only when receivers need the "no more values" signal — most commonly, a `range` loop or a `select` that wants to detect end-of-stream. If your receivers know how many values to expect (e.g., they receive exactly N times), close is unnecessary.

## 4. `select`: non-deterministic dispatch

`select` is `switch` for channel operations:

```go
select {
case v := <-in:
    handle(v)
case out <- next:
    advance()
case <-time.After(5 * time.Second):
    return errTimeout
default:
    // none of the above are ready right now
}
```

Behavior:
- If multiple cases are ready, the runtime picks one **at random**. This is intentional — you cannot rely on case order for priority. It also prevents a perpetually-ready case from starving the others.
- If no case is ready and there is no `default`, `select` blocks until exactly one becomes ready.
- If `default` is present and no other case is ready, `default` runs immediately.
- A nil channel in a case is a valid operation that **never becomes ready**, effectively disabling that case. Useful for state machines:

```go
var pending chan Event  // nil — disabled
for {
    select {
    case e := <-incoming:
        if needsForward(e) {
            pending = forwardCh  // enable the forward case
        }
    case pending <- nextForward():
        pending = nil  // disable until we have something to forward again
    case <-ctx.Done():
        return
    }
}
```

The timeout pattern is `time.After(d)` returning a channel that fires once after duration `d`. For repeated timeouts in a hot loop, prefer `time.NewTimer` and reuse it — `time.After` allocates a new timer on every call.

For periodic ticks, use `time.NewTicker(d)` (a channel that fires every `d`); remember to call `ticker.Stop()` to release runtime resources, typically in a `defer`.

## 5. Patterns: pipeline, fan-in, fan-out, worker pool, done channel

Five patterns recur across nearly every concurrent Go program.

**Pipeline** — a chain of stages, each goroutine reading from one channel and writing to the next:

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums { out <- n }
    }()
    return out
}

func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in { out <- n * n }
    }()
    return out
}

// Compose
for v := range sq(gen(1, 2, 3, 4)) {
    fmt.Println(v)  // 1 4 9 16
}
```

Each stage owns its output channel and closes it when done. The next stage uses `range` to consume until close.

**Fan-out** — multiple goroutines reading from the same input channel, processing items in parallel:

```go
func fanOut(in <-chan Job, workers int) []<-chan Result {
    outs := make([]<-chan Result, workers)
    for i := 0; i < workers; i++ {
        out := make(chan Result)
        outs[i] = out
        go func() {
            defer close(out)
            for j := range in {
                out <- process(j)
            }
        }()
    }
    return outs
}
```

**Fan-in** — multiple input channels merged into one:

```go
func fanIn(ins ...<-chan Result) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup
    for _, in := range ins {
        wg.Add(1)
        go func(c <-chan Result) {
            defer wg.Done()
            for v := range c { out <- v }
        }(in)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

The pipeline patterns above (pipeline, fan-out, fan-in) are walked through in detail in the Go blog post ["Go Concurrency Patterns: Pipelines and cancellation"](https://go.dev/blog/pipelines) by Sameer Ajmani.

**Worker pool** — fixed N workers consume from a single job channel, write to a single result channel. The bounded variant of fan-out:

```go
jobs    := make(chan Job)
results := make(chan Result)

for i := 0; i < numWorkers; i++ {
    go func() {
        for j := range jobs {
            results <- process(j)
        }
    }()
}
```

Use a worker pool when you need to bound concurrency — for example, capping outbound HTTP requests at 16 in flight to respect a third-party rate limit.

**Done channel** — broadcast cancellation by closing a channel:

```go
done := make(chan struct{})
go worker(done)
// ...
close(done)  // every receiver of `done` immediately unblocks
```

The done-channel pattern was the standard idiom before Go 1.7 added `context.Context`. Today, `context` is the convention for cross-API cancellation; the done-channel pattern still appears in single-package internal coordination. The two are compatible — `ctx.Done()` returns a `<-chan struct{}` that you `select` on exactly like a hand-rolled done channel.

## 6. Goroutine leaks

A goroutine that is blocked forever on a channel operation is a **leak**. The garbage collector cannot reclaim it because runtime references remain. In a long-running service, leaks accumulate and eventually exhaust memory or thread budgets.

Three canonical leak shapes:

**Sender blocks forever because no one is receiving.**

```go
func badQuery(ctx context.Context) (Result, error) {
    ch := make(chan Result)
    go func() {
        ch <- doWork()  // blocks forever if caller times out
    }()
    select {
    case r := <-ch:
        return r, nil
    case <-ctx.Done():
        return Result{}, ctx.Err()  // returns; the goroutine is now leaked
    }
}
```

Fix: buffer the channel so the send never blocks (size 1 is sufficient when there is exactly one send), or have the sender select on `ctx.Done()` itself.

```go
ch := make(chan Result, 1)  // size 1: send never blocks
```

**Receiver blocks forever because no one will close.**

```go
results := make(chan int)
go func() {
    for r := range results {  // waits for close
        process(r)
    }
}()
// ... if the producer panics or returns without close(results),
//     the receiver never exits.
```

Fix: own the close. Whoever produces the values closes the channel when they finish (or fail), typically in a `defer close(ch)`.

**`select` with no exit case.**

```go
for {
    select {
    case v := <-in:
        process(v)
    // missing: <-ctx.Done() or <-done — no way to leave
    }
}
```

Fix: every loop that reads from a channel must also have a way out — context, done, or the channel itself getting closed.

`pprof` goroutine profiles (`go tool pprof http://localhost:6060/debug/pprof/goroutine` once you've imported `net/http/pprof`) show goroutine counts grouped by stack. A growing count of goroutines parked on the same channel operation is the signature of a leak. Tools like [`go.uber.org/goleak`](https://github.com/uber-go/goleak) assert in tests that no goroutines outlive a test case.

## 7. When not to use channels

The proverb "Do not communicate by sharing memory; instead, share memory by communicating" comes from Rob Pike and is restated in Effective Go. It is correct as a default, but it is **not** a rule that channels are always right. The Go FAQ entry ["When should I use channels and when should I use mutexes?"](https://go.dev/doc/faq#sync_mutex_or_channel) is explicit:

> Channels and goroutines are great for communicating between goroutines; they are not so great for protecting individual variables. The standard library provides `sync.Mutex` and `sync.RWMutex` for that use case.

Use a mutex when:
- You're protecting access to a piece of state (a counter, a map, a connection pool's free list).
- The critical section is short.
- There's no obvious "value flowing between goroutines" — just shared data.

Use a channel when:
- You're transferring **ownership** of a value from one goroutine to another.
- You're coordinating events across goroutines (work distribution, completion, cancellation).
- You're modeling a pipeline.

A common mistake is wrapping a counter in a channel + goroutine purely to avoid an atomic add. The atomic-or-mutex version is faster, simpler, and easier to read. The channel version has a use only when you want to serialize a sequence of operations beyond just the increment — e.g., "increment then publish."

The `sync` package and the Go memory model are covered in the next doc.

## Related

- [Goroutines & the Scheduler (G-M-P)](01-goroutines-and-scheduler.md) — what runs the senders and receivers
- [Slices, Arrays & Maps](../fundamentals/04-slices-arrays-maps.md) — `range` on a channel mirrors `range` on a slice / map
- [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) — `<-chan T` and `chan<- T` as direction-typed parameters
- [Errors as Values](../fundamentals/06-errors-as-values.md) — using a channel of `Result{Value, Error}` to send results out of a worker

## References

- The Go Programming Language Specification, "Channel types" — https://go.dev/ref/spec#Channel_types
- The Go Programming Language Specification, "Send statements" — https://go.dev/ref/spec#Send_statements
- The Go Programming Language Specification, "Receive operator" — https://go.dev/ref/spec#Receive_operator
- The Go Programming Language Specification, "Close" — https://go.dev/ref/spec#Close
- The Go Programming Language Specification, "Select statements" — https://go.dev/ref/spec#Select_statements
- Effective Go, "Channels" — https://go.dev/doc/effective_go#channels
- Effective Go, "Share by communicating" — https://go.dev/doc/effective_go#sharing
- Sameer Ajmani, "Go Concurrency Patterns: Pipelines and cancellation" — https://go.dev/blog/pipelines (the canonical reference for the patterns in §5)
- Go FAQ, "When should I use channels and when should I use mutexes?" — https://go.dev/doc/faq#sync_mutex_or_channel
- Dave Cheney, "Channel Axioms" — https://dave.cheney.net/2014/03/19/channel-axioms (the four-axiom formulation in §3; Cheney is a long-time Go contributor)
- `time.After`, `time.NewTimer`, `time.NewTicker` — https://pkg.go.dev/time
- `net/http/pprof` — https://pkg.go.dev/net/http/pprof (goroutine profiling for leak detection)
- `go.uber.org/goleak` — https://github.com/uber-go/goleak (test-time leak detection)
