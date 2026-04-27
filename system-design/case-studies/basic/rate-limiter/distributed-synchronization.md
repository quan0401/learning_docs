---
title: "Rate Limiter Deep Dive — Distributed Synchronization"
date: 2026-04-27
updated: 2026-04-27
tags: [system-design, case-study, rate-limiter, deep-dive, gossip, crdt, synchronization]
---

# Rate Limiter Deep Dive — Distributed Synchronization

**Date:** 2026-04-27 | **Updated:** 2026-04-27
**Tags:** `system-design` `case-study` `rate-limiter` `deep-dive` `gossip` `crdt` `synchronization`

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [Centralized Baseline](#centralized-baseline)
- [Gossip-Based Aggregation](#gossip-based-aggregation)
- [Periodic Flush with INCRBY](#periodic-flush-with-incrby)
- [CRDT Counters](#crdt-counters)
- [Consensus-Based (Raft / Paxos)](#consensus-based-raft--paxos)
- [Filters for Dedupe (Bloom / Cuckoo)](#filters-for-dedupe-bloom--cuckoo)
- [Vector Clocks for Ordering](#vector-clocks-for-ordering)
- [Synchronization Windows](#synchronization-windows)
- [Drift Bound Formulas](#drift-bound-formulas)
- [Hybrid Patterns — Local Fast Path + Global Slow Truth](#hybrid-patterns--local-fast-path--global-slow-truth)
- [Clock Sync Concerns](#clock-sync-concerns)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

Once you decide to enforce a rate limit across more than one pod, you've stopped asking "how do I count requests?" and started asking "how do N independent processes converge on a shared count fast enough that the limit means something?" That is a distributed systems question, and rate limiting hits it from a peculiar angle: the answer doesn't have to be perfectly correct, but it has to be *fast* — sub-millisecond on the hot path is normal — and the consequences of being wrong are bounded by `N × per-pod-rate × sync-interval`, not "data corruption."

This doc walks the synchronization mechanics — gossip, periodic flush, CRDTs, consensus, dedupe filters, vector clocks — that sit underneath a multi-pod rate limiter. For the *topology* question (centralized vs distributed, when each makes sense, sharding strategy), see [`centralized-vs-distributed.md`](centralized-vs-distributed.md). The two docs are companions: that one tells you *which architecture* to pick; this one tells you *how the pods talk to each other once you've picked it*.

## Overview

A distributed rate limiter has, at minimum, three moving parts:

1. **Local state** — what each pod believes about the current bucket / counter
2. **Synchronization mechanism** — how local states converge to a shared view
3. **Truth source** — what wins on disagreement (centralized counter? merged CRDT? majority?)

Every approach in this doc is just a different combination of those three. The trade-off is always the same triangle:

```text
        strong
       /      \
      /        \
   speed -----  scale
```

You cannot have all three. Strong consistency + low latency forces you onto a single shard (no scale). Low latency + scale forces eventual consistency (no strong). Strong + scale forces consensus latency (no speed).

Rate limiting almost always picks **speed + scale + bounded eventual consistency**, because:

- The cost of a few extra requests slipping through is usually lower than the cost of a sub-millisecond hot-path latency budget being blown.
- Rate limits are *guidance*, not invariants. Going from "100 req/s allowed" to "104 req/s allowed during a 100ms gossip window" is rarely a business problem.
- Strict billing-grade limits get a separate, slow, eventually-consistent reconciliation pass — see [Hybrid Patterns](#hybrid-patterns--local-fast-path--global-slow-truth).

## Centralized Baseline

The simplest synchronization mechanism is *no synchronization at all between pods* — every pod talks to one shared counter store (Redis, DynamoDB, etc.) and that store *is* the truth.

```text
   pod-1 ──┐
   pod-2 ──┼──► [Redis cluster: counter for key foo]
   pod-N ──┘
```

Properties:

- **Strong consistency** within the shard (Redis is single-threaded per shard, so `INCR + EXPIRE` via Lua is linearizable).
- **One network hop on every request** — adds 0.3–2 ms p99 to every API call.
- **Hot-key risk** — if one tenant generates 90% of traffic, their counter pins one shard.
- **Blast radius** — Redis hiccup ripples into every pod simultaneously.

The whole point of the rest of this doc is variations that trade some of that strong consistency for less hot-path latency or less blast radius. For when centralized is the right call (and how to shard it), see [`centralized-vs-distributed.md`](centralized-vs-distributed.md).

Quick recap of the centralized hot path:

```text
pod receives request
  → hash(tenant) → pick Redis shard
  → EVALSHA <token-bucket-lua> <key> <now> <rate> <burst>
  → 0/1 response
  → forward or 429
```

Latency budget on this path is dominated by network RTT, not Redis CPU. Co-locating Redis in the same AZ and pre-warming connections gets it under 1 ms p99 in practice.

## Gossip-Based Aggregation

In gossip mode, **every pod keeps its own counter** and periodically tells some peers about it. Over time, all pods converge on the global view — but during the gap between gossip rounds, they're working off stale information.

### Mechanics

Two flavors:

1. **Pub/sub fanout** — every pod publishes `(key, delta_since_last_tick)` every K ms to a topic; every pod subscribes and applies the deltas.
2. **Peer-to-peer (SWIM-style)** — each pod picks `f` random peers per tick and exchanges state. SWIM uses log(N) rounds to reach the whole cluster ([Das, Gupta, Motivala 2002][swim-paper]).

For rate limiting, peer-to-peer is what most production systems use, because it scales to thousands of pods without a pub/sub bottleneck. Hashicorp's [memberlist][memberlist] (which Consul, Nomad, and Serf use for membership) is the canonical Go implementation; it does the SWIM ping/ack/ping-req dance plus piggybacked state diffs.

### Convergence math

For a SWIM-style gossip with fanout `f` and round interval `T`:

- Information reaches all `N` pods in `O(log_f(N))` rounds with high probability.
- For `N = 1000` pods, fanout `f = 3`, rounds = ~7. At `T = 100ms` per round, full convergence ≈ 700 ms.
- For `N = 100` pods, fanout `f = 3`, rounds = ~5. Convergence ≈ 500 ms.

In practice you don't need *full* convergence — you need *enough* convergence that drift stays within budget. After `r` rounds, roughly `1 - (1 - f/N)^r` fraction of pods have seen the update.

### Over-shoot bound

If the global limit is `L` requests/sec and each pod accepts up to `L/N` locally with gossip interval `I`, the worst case during one interval is:

```text
overshoot ≤ N × per-pod-rate × I
         = N × (L/N) × I
         = L × I
```

So with a 100 ms gossip interval and a global limit of 1000 req/s, you can over-shoot by up to ~100 requests across the cluster in one interval. For anti-abuse limits on signup, captcha-trigger, or login that's almost always fine. For "exactly 100 charges/second" billing-grade limits, it isn't.

### Gossip aggregator pseudocode (Go-flavored, using memberlist)

```go
package ratelimit

import (
    "encoding/json"
    "sync"
    "time"

    "github.com/hashicorp/memberlist"
)

type CounterDelta struct {
    Key   string
    Delta int64
    Pod   string
    TS    int64
}

type GossipLimiter struct {
    mu       sync.Mutex
    local    map[string]int64 // un-flushed local count
    global   map[string]int64 // best-known global count
    list     *memberlist.Memberlist
    interval time.Duration
}

func (g *GossipLimiter) Allow(key string, limit int64) bool {
    g.mu.Lock()
    defer g.mu.Unlock()
    // Fast path: local knowledge only. Global is updated out of band.
    if g.global[key]+g.local[key] >= limit {
        return false
    }
    g.local[key]++
    return true
}

// Run as a background goroutine.
func (g *GossipLimiter) gossipLoop() {
    ticker := time.NewTicker(g.interval)
    for range ticker.C {
        g.mu.Lock()
        deltas := make([]CounterDelta, 0, len(g.local))
        for k, v := range g.local {
            if v == 0 {
                continue
            }
            deltas = append(deltas, CounterDelta{
                Key: k, Delta: v, Pod: g.list.LocalNode().Name, TS: time.Now().UnixMilli(),
            })
            g.global[k] += v // fold local into global; reset local
        }
        g.local = map[string]int64{}
        g.mu.Unlock()

        payload, _ := json.Marshal(deltas)
        // Memberlist piggybacks user data onto SWIM ping/ack.
        for _, n := range g.list.Members() {
            if n.Name == g.list.LocalNode().Name {
                continue
            }
            _ = g.list.SendBestEffort(n, payload)
        }
    }
}

// Delegate hook: called when a peer's gossip arrives.
func (g *GossipLimiter) NotifyMsg(buf []byte) {
    var deltas []CounterDelta
    if err := json.Unmarshal(buf, &deltas); err != nil {
        return
    }
    g.mu.Lock()
    defer g.mu.Unlock()
    for _, d := range deltas {
        // Idempotent merge — dedupe on (Pod, TS) if needed.
        g.global[d.Key] += d.Delta
    }
}
```

The hot path (`Allow`) never blocks on the network. The gossip loop runs at whatever interval you tune — typically 50–200 ms.

## Periodic Flush with INCRBY

This is the workhorse pattern in production. Every pod runs a *local* token bucket. Every K ms, the pod drains "consumed since last flush" to the central store as a single `INCRBY`.

```text
    request                          flush tick (every 1s)
       │                                    │
       ▼                                    ▼
   [local bucket]  ── consumed_delta ──► [Redis INCRBY key delta]
       │                                    │
   admit/reject                       reconcile drift
```

### Why it works

- **Hot path stays local** — sub-millisecond admission decisions.
- **One Redis op per pod per K ms** instead of one per request — orders of magnitude fewer round trips.
- **Redis is still the truth** for billing reconciliation; local buckets just speed up enforcement.

### What gets lost on pod crash

If a pod crashes between flushes, the unflushed local count vanishes. With a 1-second flush interval and ~1000 req/s per pod, that's up to ~1000 requests of accounting drift per crash. Two ways to handle it:

1. **Accept it.** For most rate limits, losing a second of count on a rare pod crash is invisible.
2. **Persist locally.** Write a redo log to disk (or a sidecar Redis) on each admission. Replay on restart. Almost no one does this — the cost outweighs the benefit.

### Reconciliation patterns

The flush is fire-and-forget by default. To make it more robust:

- **Idempotent flush keys** — `INCRBY key:<pod>:<window-start> delta` so re-flushing the same window doesn't double-count. Aggregator sums across pod-keys.
- **Retry with backoff** if Redis is down — but cap the local accumulator so a long Redis outage doesn't let one pod admit infinity.
- **Dead-letter pod metrics** — when a pod misses N flushes in a row, alert; that pod is either crashed or partitioned.

### Periodic flush worker pseudocode

```go
type FlushLimiter struct {
    mu        sync.Mutex
    consumed  map[string]int64
    rdb       redis.Cmdable
    interval  time.Duration
    podID     string
}

func (f *FlushLimiter) Allow(key string, limit int64) bool {
    f.mu.Lock()
    defer f.mu.Unlock()
    // Local bucket logic uses a cached read of the global counter.
    // (Cached value is refreshed by the flush worker after each INCRBY.)
    if f.cachedGlobal(key) >= limit {
        return false
    }
    f.consumed[key]++
    return true
}

func (f *FlushLimiter) flushLoop(ctx context.Context) {
    t := time.NewTicker(f.interval)
    defer t.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-t.C:
            f.mu.Lock()
            batch := f.consumed
            f.consumed = map[string]int64{}
            f.mu.Unlock()

            pipe := f.rdb.Pipeline()
            for k, v := range batch {
                // Idempotent per-pod, per-window key.
                window := time.Now().Truncate(f.interval).UnixMilli()
                pipe.IncrBy(ctx, fmt.Sprintf("%s:{%s}:%d", k, f.podID, window), v)
                pipe.Expire(ctx, fmt.Sprintf("%s:{%s}:%d", k, f.podID, window), 5*time.Minute)
            }
            if _, err := pipe.Exec(ctx); err != nil {
                // On failure: re-merge into f.consumed for next flush, with a cap.
                f.requeueWithCap(batch, 10*1024)
            }
        }
    }
}
```

This is conceptually how Envoy's two-tier "local + global" rate limiter works — local in-process bucket protects hot-path latency, periodic flush keeps the global view fresh.

## CRDT Counters

When you can't tolerate "the unflushed count gets lost on crash" and you can't afford the latency of a single coordinator, you reach for a **conflict-free replicated data type (CRDT)** — a data structure where any merge of any divergent replicas always yields the same final state, regardless of order.

For rate limiting, the relevant CRDTs are **counters**. Two flavors ([Shapiro, Preguiça, Baquero, Zawirski 2011][crdt-survey]):

### G-Counter (grow-only)

Each replica `i` keeps its own slot `P[i]`; only `i` writes `P[i]`. The global value is `sum(P)`. Merge is element-wise max.

```text
Replica A: P = [3, 0, 0]   → value = 3
Replica B: P = [0, 5, 0]   → value = 5
Replica C: P = [0, 0, 2]   → value = 2

After gossip:
Replica A: merge(A, B, C) = [max(3,0,0), max(0,5,0), max(0,0,2)] = [3,5,2]
                          → value = 10
```

Convergence proof sketch: the merge function `(max, max, max)` over a vector of monotonic counters is associative, commutative, and idempotent. Any replicas that have seen the same set of updates (in any order, with any duplicates) will compute the same vector. Therefore: *strong eventual consistency*.

### PN-Counter (positive/negative)

A G-Counter only grows. For a token bucket — where you want to credit tokens back over time — you need increments *and* decrements. A PN-counter is just two G-counters: `P` for increments, `N` for decrements. Value = `sum(P) - sum(N)`.

```go
type GCounter struct {
    Slots map[string]int64 // pod ID → count
}

func (a GCounter) Merge(b GCounter) GCounter {
    out := GCounter{Slots: map[string]int64{}}
    for k, v := range a.Slots {
        out.Slots[k] = v
    }
    for k, v := range b.Slots {
        if v > out.Slots[k] {
            out.Slots[k] = v
        }
    }
    return out
}

func (g GCounter) Value() int64 {
    var sum int64
    for _, v := range g.Slots {
        sum += v
    }
    return sum
}

type PNCounter struct {
    P GCounter
    N GCounter
}

func (a PNCounter) Merge(b PNCounter) PNCounter {
    return PNCounter{P: a.P.Merge(b.P), N: a.N.Merge(b.N)}
}

func (p PNCounter) Value() int64 { return p.P.Value() - p.N.Value() }
```

### Use case: multi-region rate limiting

Where CRDT counters shine is **multi-region** rate limits where you cannot afford a cross-region round trip on the hot path. Each region runs a PN-counter slot for itself. Every K seconds, regions exchange counter state via async replication (or a CRDT store like Redis-CRDT, AntidoteDB, Riak, or a custom DynamoDB-backed merger). The merged value is eventually consistent globally; per-region enforcement uses the latest known global state.

The over-shoot bound is the same shape as gossip: `N_regions × per-region-rate × cross-region-replication-interval`. With 5 regions, 200 req/s/region, and 30 s replication, you can drift up to 30,000 requests globally — fine for "free tier of 1M req/month"; not fine for "exactly 100 charges per second."

CRDTs aren't free: the state grows with the number of writers, and you need garbage collection (e.g., delta CRDTs, causal stability) to keep it bounded. For rate limiting where slots = pods, this is usually manageable.

## Consensus-Based (Raft / Paxos)

If you absolutely need *strongly consistent* counters across pods — every admission decision agreed on by a quorum — you can run the counter inside a Raft or Paxos group. In practice this means:

- **etcd-backed counter** — `etcdctl txn` with compare-and-swap on a counter key.
- **Custom Raft service** — e.g., a `ratelimitd` cluster running [hashicorp/raft][hashicorp-raft] or [etcd-io/raft][etcd-raft].
- **ZooKeeper** — distributed counters via `setData` with version checks.

### When this is justified

Almost never for rate limiting. The latency cost is brutal:

- Raft commit = 1 round trip to leader + 1 round trip to majority of followers + fsync.
- Within an AZ: ~2–5 ms p50, 10+ ms p99.
- Across regions: 100+ ms.

That's slower than the centralized Redis path, with much higher operational complexity. The cases where it pays off:

- **Billing-grade enforcement** where over-shoot translates directly to revenue loss and you need an audit trail with linearizability.
- **Compliance limits** (e.g., regulated trading throttles) where the *order* of admissions matters and you need the consensus log as evidence.
- **Coordination beyond counting** — if the rate limiter also coordinates leases, locks, or leadership, you may already have a Raft cluster handy and can piggyback.

For everything else, gossip + periodic flush is faster, cheaper, and the over-shoot is bounded enough to be a non-issue.

## Filters for Dedupe (Bloom / Cuckoo)

A subtler synchronization problem: **"have we already seen this request ID across the cluster?"** This shows up when:

- The rate limit is "100 *unique* requests per minute" (idempotency-key dedupe).
- You want exactly-once-ish semantics for retried requests across pods.
- You want to cheaply suppress replay storms.

Storing every seen request ID in Redis is wasteful and often impractical (millions of keys). Probabilistic membership filters fix this:

- **Bloom filter** — fixed-size bit array, multiple hash functions. False positive possible (says "seen" when it wasn't), false negatives impossible. No deletion.
- **Cuckoo filter** — supports deletions, lower false positive rate at the same memory budget, slightly worse insert performance.

For rate limiter dedupe across pods, the typical pattern:

1. Each pod keeps a local Bloom/Cuckoo filter for the current window.
2. Periodically (every flush tick), pods exchange filters and OR them together.
3. Admission check: if the request ID is in the local-or-merged filter, treat as duplicate.

```text
pod-1 filter:  10110010
pod-2 filter:  01100010
pod-3 filter:  00010101

merged (OR):   11110111
```

OR-merge of Bloom filters is exact (no extra false positives beyond the union of inputs). Cuckoo filters are harder to merge — usually you exchange the underlying ID set or use a Counting Cuckoo variant.

False positives mean a *real, never-seen* request gets falsely flagged as a duplicate. Tune `m` (bits) and `k` (hash count) so the false positive rate is acceptable for the size of the window — typically 0.1–1% is fine for soft limits.

For deeper coverage, see [`../../../data-structures/bloom-and-cuckoo-filters.md`](../../../data-structures/bloom-and-cuckoo-filters.md).

## Vector Clocks for Ordering

When you need to know not just *how many* events happened but in what *causal order*, you need vector clocks ([Lamport 1978][lamport-time], [Fidge 1988][fidge-vc]).

Use case: a user submits 100 actions across 3 pods. The rate limiter needs to know:

- Did action `A` happen-before action `B`?
- Are `A` and `B` concurrent (no causal relationship)?
- Should we count them in the same window, or were they actually causally sequential?

A **vector clock** is a per-process logical clock: each process keeps a vector `V[i]` of "what I've seen from process `i`." On send, increment your own slot and ship the vector. On receive, take element-wise max with your own and increment your slot.

```go
type VectorClock struct {
    Clocks map[string]uint64 // pod ID → logical time
}

func (v VectorClock) Tick(pod string) VectorClock {
    out := v.copy()
    out.Clocks[pod]++
    return out
}

func (a VectorClock) Merge(b VectorClock) VectorClock {
    out := a.copy()
    for k, vb := range b.Clocks {
        if vb > out.Clocks[k] {
            out.Clocks[k] = vb
        }
    }
    return out
}

// HappensBefore: a < b iff every entry a[i] <= b[i] AND at least one a[i] < b[i].
func (a VectorClock) HappensBefore(b VectorClock) bool {
    strictlyLess := false
    for k, va := range a.Clocks {
        vb := b.Clocks[k]
        if va > vb {
            return false
        }
        if va < vb {
            strictlyLess = true
        }
    }
    // Account for keys present only in b.
    for k := range b.Clocks {
        if _, ok := a.Clocks[k]; !ok && b.Clocks[k] > 0 {
            strictlyLess = true
        }
    }
    return strictlyLess
}
```

In rate-limiter contexts, vector clocks usually show up not in the hot path (too expensive) but in the **reconciliation / audit pass** — proving "this user did exceed the limit in causal order, here is the evidence." The hot path uses physical timestamps with skew tolerance; the audit uses vector clocks to settle disputes.

For deeper coverage, see [`../../data-consistency/time-and-ordering.md`](../../../data-consistency/time-and-ordering.md).

## Synchronization Windows

The right sync interval depends on how strict the limit is. Three regimes:

| Window | Interval | Drift | Use case |
|--------|----------|-------|----------|
| Short (gossip) | 10–100 ms | sub-second | strict abuse limits, login throttle, web scrape defense |
| Medium (flush) | 1–5 s | seconds | per-tenant API limits, free-tier enforcement |
| Long (reconcile) | 30 s – 5 min | minutes | billing meters, audit, fairness analytics |

Picking the window:

- **Tighter limit, shorter window.** A 10 req/s limit cannot tolerate a 5-second flush; a 10,000 req/s limit can.
- **More pods, shorter window.** Drift scales linearly with pod count.
- **Higher cost-per-overshoot, shorter window** — but only down to where the network cost of more frequent sync outweighs the saved drift.

A useful sanity check: pick the window so that the worst-case drift is ≤ 5–10% of the limit. If the math doesn't work, you need a different topology (centralized) or a different SLA (allow 10% drift explicitly in the contract).

## Drift Bound Formulas

Given:

- `N` = number of pods
- `R` = per-pod admission rate (req/s)
- `I` = sync interval (s)
- `L` = global limit (req/s)

Worst-case drift during one interval:

```text
drift = N × R × I
```

For this to satisfy a global limit `L`:

```text
N × R × I ≤ L × I  (i.e. per-pod rate at most L/N)
                   AND
total accepted in interval ≤ L × I + tolerance
```

If each pod admits up to `L/N` per second locally, the steady-state global rate equals `L`. The transient drift is bounded by `L × I` — i.e., during one sync window, you may admit up to `L × I` extra before the next sync corrects it.

### What happens when the bound is exceeded

If you size the interval too generously (e.g., 30 s flush on a 10-req/s limit with 100 pods), drift compounds:

- Pods don't see each other's recent admissions.
- Each pod believes the budget is wider than it is.
- Worst case approaches `N × R × I = 100 × 10 × 30 = 30,000` extra admissions in the window — 300× the limit.

The mitigations are:

1. **Tighten the interval** until drift is acceptable.
2. **Lower per-pod cap** (admit at `L/N × safety_factor`, e.g., 0.7).
3. **Move to centralized** for that specific limit.

The third option is often correct: not every limit deserves the same topology. Strict billing limits go centralized; lax abuse limits use gossip.

## Hybrid Patterns — Local Fast Path + Global Slow Truth

The pattern most production systems converge on:

```text
┌────────────────── hot path (every request) ──────────────────┐
│                                                              │
│  pod-local token bucket  ──►  admit / reject (sub-ms)        │
│                                                              │
└──────────────────────────────────────────────────────────────┘

┌────────── periodic sync (every K ms / s) ───────────────────┐
│                                                             │
│  flush local consumed → Redis INCRBY                        │
│  read updated global → refresh local bucket fill rate       │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌────────── reconciliation (every minute / hour) ─────────────┐
│                                                             │
│  Redis counters → data warehouse / billing meter            │
│  detect drift, trigger alerts on anomalies                  │
│  feed back into rate-rule auto-tuning                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Three timescales, three correctness requirements:

| Timescale | Latency budget | Correctness | Mechanism |
|-----------|----------------|-------------|-----------|
| Hot path | sub-ms | best-effort, bounded drift | local bucket |
| Sync | ms–s | eventual consistency | flush / gossip |
| Reconcile | s–min | exact (audit-grade) | offline ETL |

Stripe's rate limiter [public writeup][stripe-rl] describes essentially this architecture: per-process token buckets for fast admission, Redis for cluster-wide aggregation, and a separate metering pipeline for billing.

The discipline is to be honest about which decisions belong on which timescale. **Admission** is a hot-path decision and tolerates drift. **Billing** is a reconcile-path decision and demands exactness. Conflating the two — running billing through the hot path — burns latency for no gain; running admission through the reconcile path — using the warehouse to decide whether to admit a request — is just broken.

## Clock Sync Concerns

Most rate-limiting algorithms slice time into windows ("100 req per *second*"). That requires every pod to agree, approximately, on what "now" is. They don't.

### Sources of skew

- **NTP** ([Network Time Protocol][ntp]) — typical accuracy 1–10 ms within a datacenter, 10–100 ms over the public internet.
- **PTP** ([Precision Time Protocol][ptp]) — sub-microsecond inside a datacenter with hardware support.
- **Container drift** — VMs and containers can pause arbitrarily under hypervisor scheduling, jumping the wall clock.
- **Leap seconds** — historically caused fun bugs; most systems now use leap smearing.

For sliding-window or fixed-window rate limiters with 1-second windows, NTP is fine. For 100-ms or finer windows you need PTP or you'll see boundary glitches: pod A counts a request in window `t=12s`, pod B's clock is 50 ms behind and counts the same logical second's request in window `t=11s`.

### How real systems handle it

- **Cloudflare** — uses sliding-window counting with ~10-second buckets; clock skew of a few ms is invisible inside that window. Their rate-limiting blog ([Counting Things][cf-counting]) discusses windowing explicitly.
- **Stripe** — rate-limit windows are 1 second or longer; uses a centralized Redis whose own clock is the truth. Pods don't need to agree among themselves; they need to agree with Redis, which is a single host's clock.
- **Envoy** — global rate-limit service uses the rate-limit *server's* clock for window boundaries, not the proxy's clock. Same trick as Stripe: pin truth to one node.

The general principle: **don't rely on distributed clocks for window boundaries.** Either:

1. Pin the window clock to a single source (Redis, the rate-limit service).
2. Use windows large enough that NTP-grade skew is negligible.
3. Use logical time (gossip rounds) instead of wall time.

### NTP-aware timestamp helper

```go
// MonotonicNow returns a (wall, mono) pair where wall is the best-known
// real time and mono is a monotonic stamp resistant to NTP jumps.
//
// On Go, time.Now() already includes a monotonic component used for
// time.Sub; we expose both explicitly for cross-process logging.
func MonotonicNow() (wall time.Time, mono time.Duration) {
    now := time.Now()
    return now, time.Since(processStart)
}

// SkewBudget is the worst-case clock disagreement we tolerate when
// comparing timestamps from different pods. Sized to NTP accuracy
// in our datacenter, with a safety margin.
const SkewBudget = 50 * time.Millisecond

// SafeWindowStart rounds `t` down to the start of a window of size `w`,
// padded by SkewBudget so events near the boundary don't get
// misattributed when pod clocks disagree.
func SafeWindowStart(t time.Time, w time.Duration) time.Time {
    padded := t.Add(-SkewBudget)
    return padded.Truncate(w)
}
```

The pattern: compute window boundaries with a safety pad equal to your maximum tolerated skew, and either reject or double-count requests inside the pad zone (you can choose which side errs on the safe direction for your business).

## Anti-Patterns

- **Gossip with unbounded local accumulation.** A pod that loses gossip connectivity should *not* keep admitting forever. Cap local admission, then fail open or fail closed by policy.
- **Using consensus for every limit.** Raft latency on the hot path will tank p99 for marginal correctness gain. Reserve consensus for billing-grade limits or skip it entirely.
- **CRDT counters without garbage collection.** State grows with every writer; without delta-CRDTs or causal-stability GC, replicas balloon. Bound the slot set or use TTLs.
- **Using local pod time for cross-pod window boundaries.** NTP skew alone will misattribute requests near boundaries. Pin time to a central source or pad with a skew budget.
- **Mixing two synchronization mechanisms inside one limiter.** A limiter that's "gossip for some keys, INCRBY for others, CRDT for the rest" is a debugging nightmare. Pick one per rate-limit class.
- **Logging every gossip message at INFO.** Generates more traffic than the rate-limit hot path itself. Sample, or downgrade to DEBUG.
- **Treating periodic-flush loss on crash as acceptable for billing.** It isn't. Use the reconciliation pass against an exact source (Kafka, S3 access logs) for billing, not the rate-limiter counters.
- **Vector clocks on the hot path.** Their O(N) overhead is wasted; use them in audit/reconciliation only.
- **Filter-based dedupe with no false-positive budget.** A 1% false positive rate is fine for soft limits but not for "deny duplicate payment" — pick a smaller `epsilon` or a different mechanism (idempotency key in a strongly-consistent store).
- **Cross-region synchronous synchronization.** Cross-region RTT is 100+ ms; you cannot do this on every request. Use region-pinned counters with async replication, accept regional drift, and only consult cross-region for hard tenant-global caps.

## Related

- **Topology choice (which architecture):** [`centralized-vs-distributed.md`](centralized-vs-distributed.md) — when to pick centralized Redis vs gossip vs CRDT topologies.
- **Parent case study:** [`../design-rate-limiter.md`](../design-rate-limiter.md) — full rate-limiter design including section 7.7 this doc expands.
- **Failure detection:** [`../../../reliability/failure-detection.md`](../../../reliability/failure-detection.md) — gossip, SWIM, phi-accrual; the same machinery underlying gossip-based rate limit aggregation.
- **Time and ordering:** [`../../../data-consistency/time-and-ordering.md`](../../../data-consistency/time-and-ordering.md) — physical time, logical clocks, vector clocks in depth.
- **Probabilistic dedupe:** [`../../../data-structures/bloom-and-cuckoo-filters.md`](../../../data-structures/bloom-and-cuckoo-filters.md) — Bloom and Cuckoo filter mechanics, false-positive math.
- **Consensus details:** [`../../../data-consistency/consensus-raft-and-paxos.md`](../../../data-consistency/consensus-raft-and-paxos.md) — when consensus is and isn't worth it.

## References

- [Das, Gupta, Motivala — *SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol* (2002)][swim-paper]
- [Hashicorp memberlist — Go SWIM implementation][memberlist]
- [Shapiro, Preguiça, Baquero, Zawirski — *A Comprehensive Study of Convergent and Commutative Replicated Data Types* (2011)][crdt-survey]
- [Lamport — *Time, Clocks, and the Ordering of Events in a Distributed System* (CACM 1978)][lamport-time]
- [Fidge — *Timestamps in Message-Passing Systems That Preserve the Partial Ordering* (1988)][fidge-vc]
- [Stripe — *Scaling your API with rate limiters*][stripe-rl]
- [Cloudflare — *Counting things, a lot of different things*][cf-counting]
- [NTP Project — Network Time Protocol overview][ntp]
- [IEEE 1588 — Precision Time Protocol][ptp]
- [hashicorp/raft — Go Raft implementation][hashicorp-raft]

[swim-paper]: https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf
[memberlist]: https://github.com/hashicorp/memberlist
[crdt-survey]: https://hal.inria.fr/inria-00555588/document
[lamport-time]: https://lamport.azurewebsites.net/pubs/time-clocks.pdf
[fidge-vc]: https://fileadmin.cs.lth.se/cs/Personal/Amr_Ergawy/dist-algos-papers/4.pdf
[stripe-rl]: https://stripe.com/blog/rate-limiters
[cf-counting]: https://blog.cloudflare.com/counting-things-a-lot-of-different-things/
[ntp]: https://www.ntp.org/
[ptp]: https://standards.ieee.org/ieee/1588/4355/
[hashicorp-raft]: https://github.com/hashicorp/raft
