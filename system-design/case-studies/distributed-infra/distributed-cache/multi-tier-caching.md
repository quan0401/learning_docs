---
title: "Multi-Tier Caching — L1 In-Process, L2 Distributed, and Coherence Trade-offs"
date: 2026-05-01
updated: 2026-05-01
tags: [system-design, deep-dive, caching, hierarchy, coherence]
---

# Multi-Tier Caching — L1 In-Process, L2 Distributed, and Coherence Trade-offs

**Date:** 2026-05-01 | **Updated:** 2026-05-01
**Tags:** `system-design` `deep-dive` `caching` `hierarchy` `coherence`
**Parent:** [`../design-distributed-cache.md`](../design-distributed-cache.md)

## Table of Contents

- [Summary](#summary)
- [The Latency Stack — Why a Layer Below the Network Matters](#the-latency-stack--why-a-layer-below-the-network-matters)
- [Sizing the Tiers — Hottest 0.1% in L1, Working Set in L2](#sizing-the-tiers--hottest-01-in-l1-working-set-in-l2)
- [L1 Implementations — Caffeine, Ristretto, golang-lru, lru_cache](#l1-implementations--caffeine-ristretto-golang-lru-lru_cache)
- [L1 Size Budget — Heap Pressure and GC Tail Latency](#l1-size-budget--heap-pressure-and-gc-tail-latency)
- [L2 Implementations — Redis, Memcached, Hazelcast, Couchbase](#l2-implementations--redis-memcached-hazelcast-couchbase)
- [Negative Caching at L1 — Caching the Miss](#negative-caching-at-l1--caching-the-miss)
- [The Coherence Problem — N Application Nodes, N Different Views](#the-coherence-problem--n-application-nodes-n-different-views)
- [Coherence Solutions — Short TTL, Pub/Sub, Versioned Reads, Coalescing](#coherence-solutions--short-ttl-pubsub-versioned-reads-coalescing)
- [Redis 6 Client-Side Caching — RESP3 Invalidation Tracking](#redis-6-client-side-caching--resp3-invalidation-tracking)
- [Read Path — L1 → L2 → Origin, Populate on the Way Back](#read-path--l1--l2--origin-populate-on-the-way-back)
- [Write Path — Invalidate Locally, Invalidate L2, Broadcast to Peers](#write-path--invalidate-locally-invalidate-l2-broadcast-to-peers)
- [Memory Accounting and Eviction Policy Per Tier](#memory-accounting-and-eviction-policy-per-tier)
- [Bloom Filter in Front of L2 — Skipping Definite Misses](#bloom-filter-in-front-of-l2--skipping-definite-misses)
- [Stampedes at Each Tier — Same Problem, Different Layer](#stampedes-at-each-tier--same-problem-different-layer)
- [CDN as L0 — The Same Principle, One Layer Higher](#cdn-as-l0--the-same-principle-one-layer-higher)
- [Worked Example — A Hot Product Page Across All Tiers](#worked-example--a-hot-product-page-across-all-tiers)
- [Configuration Sketch — Sizes and TTLs in One Place](#configuration-sketch--sizes-and-ttls-in-one-place)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

The parent case study at [`../design-distributed-cache.md`](../design-distributed-cache.md) summarizes multi-tier caching in roughly a page: "put a small in-process cache in front of Redis to absorb the hottest keys." That sentence hides the entire design space. The naive interpretation — "wrap a `HashMap` around your Redis client" — produces a cache that is *faster than L2 by 4000×* on hits and *wronger than the database* on writes, because every application node carries an independent stale snapshot of the truth.

This document is the deep-dive on how to build **L1 in-process + L2 distributed** correctly: how to size each tier, which library to use in which language, how to keep N application nodes' L1s coherent enough that callers don't see ping-pong staleness, and where the protocol-level support (Redis 6 RESP3 client tracking, Hazelcast Near Cache, Spring Cache multi-level abstraction) earns its keep.

It mirrors the structure of the Instagram CDN deep-dive at [`../../social-media/instagram/cdn-strategy.md`](../../social-media/instagram/cdn-strategy.md) — the same hierarchical-cache pattern, applied to mutable application data instead of immutable image bytes. Where the CDN tier story can lean on content-addressing to make every entry safely immutable, the L1/L2 story has to confront the **mutation problem head-on**: the same key has different values at different times, and N independent in-process caches will see those mutations at different times, in different orders, and possibly never at all.

The single most important insight: **L1 and L2 must share the same key space and be invalidated together. The moment your L1 invalidation policy diverges from your L2 invalidation policy, you have built a distributed bug, not a distributed cache.**

## The Latency Stack — Why a Layer Below the Network Matters

The numbers that drive this whole design come from Jeff Dean's "Latency Numbers Every Programmer Should Know" — see the [annotated gist by Jonas Bonér](https://gist.github.com/jboner/2841832) and Peter Norvig's [original teaching post](https://norvig.com/21-days.html#answers). The orders of magnitude matter more than the exact figures:

| Layer | Typical latency | What it actually is |
|---|---:|---|
| CPU L1 cache | ~1 ns | Per-core SRAM; touched by every load instruction. |
| CPU L2 / L3 cache | ~5–50 ns | Shared across cores; still on the die. |
| Main memory (DRAM) | ~100 ns | A `HashMap.get()` of a hot key. |
| In-process cache hit (Caffeine, Ristretto) | ~100–500 ns | Memory + a hash, atomic counter bump, maybe a CLHM. |
| Same-DC network round-trip (Redis hit) | ~200 µs – 1 ms | TCP, kernel, NIC, switch, NIC, kernel, TCP, parse. |
| Cross-AZ (within region) | ~1–2 ms | Add a few switch hops and a longer fibre run. |
| SSD random read | ~100 µs | Origin DB on a hot row that misses Postgres' shared_buffers. |
| Disk-bound DB query | ~5–20 ms | Index + heap fetch when the working set spills RAM. |
| Cross-region | ~30–150 ms | Speed of light through fibre; not a cache, a planet. |

The decisive jump is **L1 (memory) → L2 (network round-trip): ~1000–4000× slower**. That ratio is not a software problem. It is the ratio of a memory bus latency (nanoseconds, no kernel involved) to a TCP request (microseconds, plus syscalls, plus a switch, plus kernel scheduling on the other side, plus serialization). No amount of "faster Redis" closes that gap. The only way to break the network-RTT floor is **don't go to the network**.

This is why an in-process tier is non-negotiable for any system whose p99 latency budget is tighter than the same-DC round-trip. A 200 µs Redis call, on its own, eats most of a 1 ms p99 budget. With a 99% L1 hit rate on top, the *amortized* read latency drops to:

```text
amortized = p_l1_hit × t_l1 + p_l2_hit × t_l2 + p_origin × t_origin
          = 0.99 × 200 ns + 0.0099 × 200 µs + 0.0001 × 10 ms
          ≈ 200 ns + 2 µs + 1 µs
          ≈ ~3 µs amortized
```

Without L1, the same workload averages 200 µs. **L1 is a 60× amortized speedup**, and — crucially — it converts a steady-state network-bound workload into a memory-bound one, which is the only kind that scales to single-digit-microsecond p99.

It's the same hierarchy story the parent tells about CPU caches. Your CPU has L1/L2/L3 caches because RAM is too slow; your application has L1 (in-process) and L2 (Redis) caches because the database is too slow. The ratios are similar, the trade-offs (capacity vs latency, coherence vs simplicity) are isomorphic, and the lessons from CPU cache design (write-back vs write-through, MESI, false sharing) translate directly upward.

## Sizing the Tiers — Hottest 0.1% in L1, Working Set in L2

Hierarchical caches obey a simple rule: **each lower-cardinality, faster tier holds a strict subset of the next tier**. L1 ⊂ L2 ⊂ origin. Sizing is therefore a question of "what fraction of the working set does each tier hold?"

| Tier | Capacity | Fraction of keys | What lives here |
|---|---|---|---|
| L1 in-process | 10 MB – a few GB per node | **Hottest 0.1–1%** of keys | The Zipf head: top 1k–100k keys. |
| L2 distributed | 100 GB – many TB cluster-wide | **Working set** (typically 5–30% of all keys) | Everything queried in the last hour/day. |
| Origin DB | All data | 100% | Source of truth. |

Why those fractions? Real workloads are heavily Zipfian — see Facebook's [Memcache study](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final197.pdf) and Twitter's [TwemCache analysis](https://engineering.twitter.com/en/insights/insights-from-one-year-of-cache-workload-analysis). The top **1% of keys typically receive 50–90% of traffic**. Caching those in L1 captures most of the read amplification. L2 absorbs the next two orders of magnitude — keys that are warm but not red-hot — and the long tail goes to origin.

A worked sizing for an e-commerce product catalog with 100 M SKUs:

```text
Total keys:         100,000,000 SKUs
L1 per node:        10,000 entries × ~2 KB = 20 MB
L2 cluster total:   30,000,000 entries × ~2 KB = 60 GB
Origin DB:          full 100M product rows + indexes

Coverage:
  L1: top 0.01% of SKUs by traffic, ~70–80% of read volume
  L2: top 30% of SKUs (working set), ~95–99% of read volume
  Origin: long tail, ~1–5% of read volume
```

**Why "small L1, big L2" beats "big L1, no L2."** It's tempting to skip L2 and grow L1 instead — fewer moving parts, no network. The math says no:

- **GC pressure is superlinear in heap size.** A 32 GB JVM heap is not 32× the GC pause of a 1 GB heap; it's worse. L1 sized in the GB range starts to dominate p99 with stop-the-world pauses. See [`../../../../java/garbage-collection/zgc-and-shenandoah.md`](../../../../java/garbage-collection/zgc-and-shenandoah.md) for the details on low-pause collectors.
- **Coherence cost is linear in L1 size.** Every entry in L1 is an entry that can go stale; the invalidation broadcast has to track them all (or the TTL has to be short enough that staleness washes out, which limits L1 hit rate).
- **L2 holds the working set across all instances** — one shared 60 GB cache vs N × 60 GB private caches. The shared cache wins on cost, freshness, and coherence.

The structural answer: **L1 small enough to fit in a young-generation budget; L2 big enough to hold the working set; origin holds the tail**.

## L1 Implementations — Caffeine, Ristretto, golang-lru, lru_cache

L1 is a library, not a service. Choices by language:

### Caffeine (Java) — the gold standard

[Caffeine](https://github.com/ben-manes/caffeine/wiki) by Ben Manes is the modern in-process cache for the JVM. Key properties:

- **W-TinyLFU admission policy** — combines recency (LRU) and frequency (LFU) signals, beats both individually. [Efficiency benchmarks](https://github.com/ben-manes/caffeine/wiki/Efficiency) show 5–15% higher hit rates than plain LRU.
- **Lock-free reads via a ring buffer**; writes batched into the eviction structure asynchronously.
- **Asynchronous loading** via `AsyncCache` so L1 misses don't block the caller.
- **Size, weight, or time-based eviction** — `maximumSize`, `maximumWeight`, `expireAfterWrite`, `expireAfterAccess`.
- **Removal listeners** for hooking invalidation broadcasts.

```java
LoadingCache<String, Product> l1 = Caffeine.newBuilder()
    .maximumSize(10_000)                      // bound the heap
    .expireAfterWrite(Duration.ofSeconds(5))  // bound the staleness
    .recordStats()                             // hit-rate observability
    .removalListener((key, value, cause) -> {
        if (cause == RemovalCause.EXPLICIT) {
            metrics.invalidations.increment();
        }
    })
    .build(key -> l2.get(key)); // fall through to L2 on miss
```

Spring Boot's [Cache abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html) supports Caffeine as the in-process tier and a custom `CacheManager` to compose it with a Redis-backed L2; see the JPA caching companion at [`../../../../java/jpa/jpa-cache-traps.md`](../../../../java/jpa/jpa-cache-traps.md) for the related second-level cache hazards.

### Ristretto (Go) — TinyLFU with a Bloom-filter admission gate

[Ristretto](https://github.com/dgraph-io/ristretto) is Dgraph's Go cache, engineered to handle the **memory-bandwidth contention** that kills naive Go caches under concurrency ([design rationale](https://dgraph.io/blog/post/introducing-ristretto-high-perf-go-cache/)):

- **Cost-based admission and eviction** — capacity is cost-units, not items.
- **TinyLFU admission filter** — Bloom-filter frequency sketch prevents one-hit wonders from polluting the cache.
- **Sharded internal structure** to avoid lock contention.
- **Sampled-LFU eviction** — sample candidates, evict least-frequent.

```go
cache, _ := ristretto.NewCache(&ristretto.Config{
    NumCounters: 1e7,     // 10× expected entries (TinyLFU sketch sizing)
    MaxCost:     1 << 30, // 1 GB
    BufferItems: 64,      // ring-buffer batching
})
cache.Set("product:42", product, 1)
v, ok := cache.Get("product:42")
```

### HashiCorp golang-lru — when you just want LRU

[golang-lru](https://github.com/hashicorp/golang-lru) is the pragmatic default for "I need an LRU map." 2Q and ARC variants included. Lower hit rate than Ristretto on Zipfian workloads, but simpler and predictable under bursty traffic.

```go
import lru "github.com/hashicorp/golang-lru/v2"

l1, _ := lru.New[string, *Product](10_000)
l1.Add("product:42", product)
v, ok := l1.Get("product:42")
```

### Python `functools.lru_cache` and `cachetools`

`functools.lru_cache` is fine for pure functions with bounded keyspace; for service caches with TTL, [`cachetools`](https://cachetools.readthedocs.io/) provides `TTLCache`, `LFUCache`, `LRUCache`. For multi-process Python (gunicorn, uWSGI), each worker has an independent L1 — coherence broadcasts must reach every worker, not just every host.

### Hazelcast Near Cache — when L2 itself runs in your JVM

If L2 is Hazelcast, [Near Cache](https://docs.hazelcast.com/imdg/4.2/performance/near-cache) is a first-class L1 with built-in invalidation tracking. The cluster pushes invalidations to clients; staleness is bounded by the network rather than a TTL guess. Cleanest L1/L2 model you can buy off the shelf, but locks you into the Hazelcast operational model.

### What they all share

Every viable L1 library exposes a size/weight cap, a TTL (`expireAfterWrite`/`expireAfterAccess`), a stats interface (`hitRate`, `loadAverage`, `evictionCount`), and a removal hook for broadcasts. Lacking any of these, it's a `HashMap`, not a cache.

## L1 Size Budget — Heap Pressure and GC Tail Latency

The single most-violated rule of L1 sizing on the JVM: **L1 hit rate is bounded by GC tail latency, not by raw memory cost**.

A 4 GB Caffeine cache on a JVM with G1GC will, eventually, produce a 200–500 ms stop-the-world pause that immediately spikes p99 above any reasonable SLO. The cache is "fast" in the steady state and ruinous at every collection. Two responses, both correct, both with trade-offs:

### 1. Cap L1 small enough to live in young gen

If L1 fits in a Caffeine `maximumSize` of ~50–100k entries (~50–100 MB depending on object size), most entries die young and get collected in cheap young-gen passes. The top of the Zipf distribution is so concentrated that you don't need millions of entries; you need the right ten thousand.

### 2. Pick a low-pause collector

[ZGC](https://openjdk.org/jeps/377) and [Shenandoah](https://wiki.openjdk.org/display/shenandoah) target sub-10ms pauses regardless of heap size by doing collection concurrently with the application. Carries a multi-GB L1 without the pause penalty, at the cost of more CPU and a more demanding ops story. See [`../../../../java/garbage-collection/zgc-and-shenandoah.md`](../../../../java/garbage-collection/zgc-and-shenandoah.md).

### 3. Off-heap entries

For multi-GB L1s on G1GC, store entry **values** off-heap (Caffeine + [Chronicle Map](https://github.com/OpenHFT/Chronicle-Map), or a `DirectByteBuffer` arena). Keys stay on-heap; values live in native memory and aren't GC-scanned. Operationally heavier; reserve for cases where cardinality demands it.

Go's concurrent GC still **doubles working-set RSS** with a multi-GB Ristretto cache; Python's generational GC has its own pathology on large dict caches. The structural rule (cap L1, watch p99) is universal.

**Rule of thumb:** L1 must not exceed 5–10% of process heap; total cache pause budget must not exceed 10% of the p99 latency target. Track `cache.hitRate`, `gc.pause.p99`, and request `p99` together; halve L1 when the curve bends.

## L2 Implementations — Redis, Memcached, Hazelcast, Couchbase

| L2 | Strengths | Trade-offs |
|---|---|---|
| **Redis** ([redis.io](https://redis.io/docs/)) | Rich data types, pub/sub, replication, **Redis 6 RESP3 client-side caching** (huge for L1 coherence). | Single-threaded per node; memory overhead per key. |
| **Memcached** ([memcached.org](https://memcached.org/)) | Multi-threaded; small per-key overhead; battle-tested at Facebook scale. | Pure key-value, no data types, no replication, no pub/sub. |
| **Hazelcast IMDG** ([docs.hazelcast.com](https://docs.hazelcast.com/)) | JVM integration; **Near Cache** is a L1/L2 protocol out of the box; peer-to-peer. | JVM-centric. |
| **Couchbase** ([docs.couchbase.com](https://docs.couchbase.com/)) | Memcached protocol + persistence + N1QL; **observe** for write ack. | Heavier ops footprint. |

The decisive feature for multi-tier is **whether L2 can push invalidations to L1 clients**. Redis 6 (RESP3 tracking) and Hazelcast (Near Cache) get this right out of the box. With Memcached you build the broadcast yourself (typically a Kafka topic of mutation events). From the multi-tier perspective, **Redis 6+ wins** because tracking turns the coherence problem into a server-driven push.

## Negative Caching at L1 — Caching the Miss

A subtle but important question: when L2 returns *not found*, what does L1 store? Three options:

1. **Store nothing.** Every subsequent miss for that key re-pays the L2 round-trip, and on a key that doesn't exist (404 product, banned user, deleted post), every request hammers L2 and then origin.
2. **Store a tombstone with short TTL.** Cache "this key is missing" for a few seconds. Subsequent misses are L1 hits returning "missing" instantly.
3. **Bloom-filter the miss space.** See the dedicated section below.

Option 2 is the workhorse:

```java
public Optional<Product> get(String key) {
    Object cached = l1.getIfPresent(key);
    if (cached == NOT_FOUND_TOMBSTONE) return Optional.empty();
    if (cached != null) return Optional.of((Product) cached);

    Optional<Product> fromL2 = l2.get(key);
    if (fromL2.isPresent()) {
        l1.put(key, fromL2.get());
    } else {
        l1.put(key, NOT_FOUND_TOMBSTONE, Duration.ofSeconds(5)); // negative cache
    }
    return fromL2;
}
```

**Why short TTL on negative entries.** The opposite of a hot hit is a hot miss — a malformed product ID hitting your API a million times a second. A 5-second negative cache absorbs 99.9% of those even if the real product gets created later. Long negative TTL is dangerous: a user who creates a product immediately after a negative-caching read will see "doesn't exist" for the entire TTL, which feels like a bug. 1–10 seconds is the typical sweet spot.

This is the same pattern DNS resolvers use for `NXDOMAIN` ([RFC 2308 — Negative Caching of DNS Queries](https://www.rfc-editor.org/rfc/rfc2308)): cache the negative answer briefly to absorb retries, but not so long that real changes are masked.

## The Coherence Problem — N Application Nodes, N Different Views

Here is the structural problem with L1: **the same key, on N different application nodes, can hold N different values, all stale to different degrees**.

```text
t=0:  All nodes: L1["price:42"] = $9.99   (in sync)
t=1:  DB updated:  price:42 := $11.99
t=1:  L2 invalidated.
t=2:  Node A reads, populates L1["price:42"] = $11.99 from L2.
t=2:  Node B serves request from cached L1["price:42"] = $9.99.   (stale)
t=2:  Node C serves request from cached L1["price:42"] = $9.99.   (stale)
t=10: Node A's L1 entry expires; refetches $11.99.
t=10: Node B and C still serving $9.99 until their L1 TTL expires.
```

Two requests from the same user, hitting different nodes via load balancer, can see *different prices*. From the user's perspective: "the price keeps changing every time I refresh." From the database's perspective: "we updated the value once, why is the cache lying about it?"

This is the **cache coherence problem**, well-known from CPU design. CPUs solve it with the [MESI protocol](https://en.wikipedia.org/wiki/MESI_protocol) — every cache line has a state (Modified/Exclusive/Shared/Invalid) and writes broadcast to peers. Distributed caches solve it with weaker, looser approximations of the same idea.

The hard truth: **you cannot have all of (1) microsecond-fast L1 reads, (2) strong consistency across N nodes, (3) reasonable cost**. Pick two. Most production systems pick (1) and (3) and bound the staleness in (2) with TTL + best-effort invalidation broadcast.

## Coherence Solutions — Short TTL, Pub/Sub, Versioned Reads, Coalescing

Four approaches, ordered by complexity and consistency strength:

### 1. Short TTL — accept staleness, bound it

The simplest answer: L1 entries expire after 1–5 seconds. Any divergence between nodes is automatically erased on the next refresh.

- **Pro:** No coordination, no dependencies, no protocol.
- **Con:** Hit rate is bounded by `(1 - 1/avg_request_rate_per_key)`; very hot keys get high hit rate, cold-but-cached keys get hammered. And the staleness window is exactly the TTL.
- **When to pick it:** Read-mostly data where "up to 5 seconds stale" is acceptable. Product catalogs, feature flags, configuration. **Most workloads.**

### 2. Pub/Sub invalidation broadcast

L2 (or the application layer) publishes a "key X invalidated" message; every application node subscribes and clears its L1 entry on receipt.

- **Pro:** Sub-second propagation; L1 TTL can be longer (30s, 5min, even 1 hour).
- **Con:** Broadcast cost is `O(N × invalidations/sec)`. At a few thousand subscribers, this becomes a meaningful load on the broadcast channel. At tens of thousands it stops scaling.
- **Failure mode:** Lost messages mean stale L1 forever. Pair with a TTL safety net (long, e.g., 5 minutes) to bound the worst case.
- **When to pick it:** Medium-scale services (low-thousands of nodes), high-mutation rate data where short TTL gives unacceptable hit rates.

### 3. Versioned reads

Store `(value, version)` in L1; on read, validate the version against L2 (`GET version:X`) before returning. If the version is current, return L1; otherwise refetch.

- **Pro:** Strong-ish consistency — staleness window is the version-check round-trip, not the TTL.
- **Con:** Defeats most of the L1 latency benefit (you still pay an L2 round-trip for the version check). Worth it only if the value itself is large enough that *value transfer* dominates and the version check is cheap by comparison.
- **When to pick it:** Cached entries that are large (KB-MB) and expensive to fetch, where the version check is fractionally cheap.

### 4. Request coalescing layer

Deduplicate concurrent fetches for the same key. Even without explicit coherence, a coalescer turns N concurrent misses into 1 origin fetch and N callers waiting on the same `Future`.

- **Pro:** Solves stampede at L1 *and* L2.
- **Con:** Doesn't solve coherence on its own — only protects the back end during the brief window of mass-miss after invalidation.
- **When to pick it:** Always. Pair it with one of the above coherence strategies.

The default production stack is **(1) short TTL + (2) best-effort pub/sub broadcast + (4) request coalescing**. Versioned reads (3) are a niche tool; reach for them only when the value transfer cost makes the round-trip-on-validate worth it.

## Redis 6 Client-Side Caching — RESP3 Invalidation Tracking

Redis 6 introduced a first-class protocol for L1 coherence: [client-side caching with invalidation tracking](https://redis.io/docs/manual/client-side-caching/). It uses the [RESP3 protocol](https://github.com/antirez/RESP3/blob/master/spec.md) to push invalidation messages from server to client.

The model:

1. Client connects with `HELLO 3` (RESP3 upgrade) and runs `CLIENT TRACKING on`.
2. Every `GET key` Redis serves to that client is **registered** in an in-memory tracking table on the server side.
3. When *any* client (the same one or another) modifies that key, the server sends an **invalidation push** to every tracking client.
4. The library receives the push and removes the key from L1.

```
┌──────────┐               ┌──────────────┐
│ App Node │  GET k     →  │ Redis Server │
│  L1      │  ←  v          │  + tracking  │
│ {k:v}    │                │  table       │
└──────────┘               └──────────────┘

[ Some other client mutates k ]

┌──────────┐               ┌──────────────┐
│ App Node │  ← invalidate │ Redis Server │
│  L1      │      "k"      │              │
│ {}       │                │              │
└──────────┘               └──────────────┘
```

Two implementation modes:

- **Default tracking** — server tracks every key the client reads. Memory cost on the server scales with `(clients × keys per client)`; works well for moderate cardinality.
- **Broadcasting tracking** — client subscribes to a *prefix*; server pushes invalidations for any key matching that prefix to that client, regardless of whether the client has read it. Lower server-side memory but higher invalidation rate at the client.

The Redis docs include a [protocol-level walkthrough](https://redis.io/docs/manual/client-side-caching/) and a discussion of when to use each mode. The Lettuce Java client and `redis-py` (with `redis-py>=4.5`) support tracking natively.

The coherence story this enables:

```text
t=0:  All nodes: L1["price:42"] = $9.99      (Redis tracks each GET)
t=1:  Some node SETs price:42 := $11.99
t=1:  Redis pushes "invalidate price:42" to every client that read it
t=1:  Every node clears L1["price:42"] within milliseconds
t=2:  Any node serving a request reads from L2 fresh, populates L1
```

Staleness window is bounded by **Redis push latency**, typically 1–10 ms across a healthy data center. That's two orders of magnitude better than a 5-second TTL, with no application-level pub/sub bus to operate.

The cost: Redis is now responsible for a tracking table and outbound push connections. A misbehaving client that doesn't process pushes will accumulate state. The protocol has [deal-breakers documented](https://redis.io/docs/manual/client-side-caching/#what-the-client-needs-to-implement) — read them before adopting.

A skeleton Lettuce-style invalidation handler:

```java
// Lettuce — registering an invalidation listener
StatefulRedisConnection<String, Product> conn = redisClient.connect(productCodec);
conn.sync().clientTracking(TrackingArgs.Builder.enabled());

// The connection emits TrackingMessage events
((RedisChannelHandler<?, ?>) conn).addListener(message -> {
    if (message instanceof TrackingMessage tm) {
        for (String key : tm.getKeys()) {
            l1.invalidate(key);
            metrics.invalidationsReceived.increment();
        }
    }
});
```

The combination — **Redis 6 tracking + Caffeine L1 + short TTL safety net** — is the modern default for Java services that need L1 in front of Redis.

## Read Path — L1 → L2 → Origin, Populate on the Way Back

The canonical multi-tier read function:

```java
public Product get(String key) {
    // L1
    Product v = l1.getIfPresent(key);
    if (v == NOT_FOUND) return null;
    if (v != null) {
        metrics.l1Hit.increment();
        return v;
    }

    // L1 miss — coalesce concurrent fetches by key
    return loader.load(key, () -> {
        // L2
        Optional<Product> fromL2 = l2.get(key);
        if (fromL2.isPresent()) {
            metrics.l2Hit.increment();
            l1.put(key, fromL2.get()); // populate up
            return fromL2.get();
        }

        // Origin
        Optional<Product> fromDb = db.findById(key);
        if (fromDb.isPresent()) {
            metrics.originHit.increment();
            l2.put(key, fromDb.get(), L2_TTL); // populate L2
            l1.put(key, fromDb.get());         // populate L1
            return fromDb.get();
        }

        // Definite miss — negative cache at both tiers
        metrics.miss.increment();
        l2.put(key, TOMBSTONE, NEGATIVE_TTL);
        l1.put(key, NOT_FOUND, Duration.ofSeconds(5));
        return null;
    });
}
```

Three things to notice:

1. **L1 populated from both L2 hits and origin hits.** The origin path warms L1 directly; the L2 path warms L1 indirectly. Either way, the next read from this node hits L1.
2. **The `loader.load(key, ...)` call is a request-coalescer.** Implementations: `LoadingCache.get` in Caffeine (built-in), `singleflight.Group` in Go, custom `ConcurrentHashMap<String, CompletableFuture<V>>` in plain Java. Without coalescing, an invalidated hot key triggers a stampede the moment its L1 entry clears.
3. **Negative caching at both tiers.** L2's negative entry has a longer TTL (minutes) than L1's (seconds), because a stampede on a missing key would otherwise hammer the database every time L1's negative entry expires.

The corresponding Go shape with Ristretto and `singleflight`:

```go
func (s *Service) Get(ctx context.Context, key string) (*Product, error) {
    if v, ok := s.l1.Get(key); ok {
        if v == notFound { return nil, nil }
        return v.(*Product), nil
    }

    res, err, _ := s.coalescer.Do(key, func() (any, error) {
        if v, err := s.l2.Get(ctx, key); err == nil && v != nil {
            s.l1.Set(key, v, 1)
            return v, nil
        }
        v, err := s.db.FindByID(ctx, key)
        if err != nil { return nil, err }
        if v == nil {
            s.l1.SetWithTTL(key, notFound, 1, 5*time.Second)
            s.l2.SetWithTTL(ctx, key, tombstone, 60*time.Second)
            return nil, nil
        }
        s.l2.Set(ctx, key, v, l2TTL)
        s.l1.Set(key, v, 1)
        return v, nil
    })
    if err != nil { return nil, err }
    if res == nil { return nil, nil }
    return res.(*Product), nil
}
```

## Write Path — Invalidate Locally, Invalidate L2, Broadcast to Peers

The write path is where naive multi-tier implementations break. The contract: **after a successful write, no node may serve the old value from L1 for longer than the agreed staleness bound**.

```java
public void update(Product p) {
    db.save(p);                          // 1. write through to origin
    l2.invalidate(p.id());               // 2. drop L2 (don't update — see invalidation vs update below)
    l1.invalidate(p.id());               // 3. drop local L1
    invalidationBus.publish(p.id());     // 4. tell peers to drop their L1
}
```

The order matters:

1. **Origin first.** If the origin write fails, you must not have invalidated L2 — otherwise the next reader populates L2 with the *old* value (re-read from origin) and L2 is wrong. Origin write succeeds → invalidate downstream. Origin write fails → leave caches alone, return error.
2. **Invalidate L2, don't update.** Updating L2 races with concurrent writers; whoever writes last wins, and that may not be the latest value. *Deleting* the entry forces the next reader to repopulate from origin, which is the source of truth. The "double-delete" pattern in [Cache-Aside design](../design-distributed-cache.md#caching-strategies--cache-aside-vs-write-through-vs-write-behind) extends this with a delayed second delete to handle the rare race where a stale read populates the cache between origin write and L2 invalidation.
3. **Invalidate local L1 immediately.** This node, at least, is now consistent.
4. **Broadcast to peers.** Every other application instance must drop its L1 entry. The broadcast is `at-least-once`; duplicate invalidations are harmless idempotent operations.

A skeleton broadcast subscriber, Redis-pub/sub style:

```java
@PostConstruct
void subscribe() {
    redis.pubsub().subscribe("cache.invalidate", (channel, message) -> {
        String key = message;
        l1.invalidate(key);
        metrics.invalidationsReceived.increment();
    });
}

void publishInvalidation(String key) {
    redis.pubsub().publish("cache.invalidate", key);
}
```

For Redis 6 tracking, you don't write the broadcast yourself — Redis sends pushes natively, as shown in the previous section. For Memcached or systems where tracking isn't available, the broadcast bus is your responsibility and a Kafka topic is usually the right transport at scale (durable, ordered per partition, replayable).

**Loss of broadcast messages is survivable** because of the L1 TTL safety net: even if a node misses a broadcast, the entry expires and is refetched within seconds. The broadcast is an *optimization* on top of the TTL, not a replacement for it. A system that *only* uses broadcast (no TTL) and then drops a message permanently has poisoned an L1 forever — pair them.

## Memory Accounting and Eviction Policy Per Tier

L1 and L2 have **independent eviction policies, capacities, and observability**. Designing them together (or, worse, sharing a single policy across both tiers) is a category error.

| Concern | L1 (in-process) | L2 (distributed) |
|---|---|---|
| Capacity | 10 MB – few GB per node | 100 GB – many TB per cluster |
| Eviction policy | W-TinyLFU (Caffeine), TinyLFU (Ristretto), 2Q/ARC | LRU (Redis `allkeys-lru`), LFU (Redis `allkeys-lfu`), or `volatile-*` variants |
| TTL | 1–60 seconds typical | minutes to hours |
| Hit rate target | 80–99% on hottest keys | 95–99% on working set |
| Observability | per-process metrics | per-cluster metrics |

The parent's [`./eviction-policies.md`](./eviction-policies.md) covers the L2 eviction-policy choices in depth. The point here is to make sure the L1 policy is chosen for the **hot-key workload** (W-TinyLFU's frequency-aware admission keeps cold keys from displacing the Zipf head) while L2 is tuned for the **working-set workload** (a more uniform LRU is fine because the cardinality is high enough that frequency tracking buys less).

A common mistake: configure L1 with a large `maximumSize` and assume the eviction policy will sort it out. It will — but every entry that lives in L1 is an entry the invalidation broadcast must reach, an entry the GC must scan, and an entry whose staleness must be bounded. **Evict aggressively at L1; evict patiently at L2.**

## Bloom Filter in Front of L2 — Skipping Definite Misses

For datasets where most lookups are misses (user-by-email when most email addresses don't have accounts; product-by-SKU when scrapers probe random SKUs), the negative-cache TTL is too short to absorb the load and L2's miss path dominates.

A [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter) in front of L2 can short-circuit definite misses with no L2 round-trip:

```java
public Optional<User> findByEmail(String email) {
    if (!emailBloom.mightContain(email)) {
        // Definite miss; no point asking L2 or DB.
        metrics.bloomFilterShortCircuit.increment();
        return Optional.empty();
    }
    // mightContain may have a false-positive rate (e.g., 1%); follow normal path.
    return getCachedOrLoad(email);
}
```

Properties:

- **No false negatives.** If the Bloom filter says "definitely not present," it isn't.
- **Tunable false-positive rate** (typically 0.1–1%). False positives just fall through to the normal cache lookup, which is cheap.
- **Compact.** ~10 bits per item for 1% FPR; a billion items fit in ~1.2 GB.
- **Must be rebuilt or use a counting Bloom filter for deletes.** Plain Bloom filters don't support removal.

For Redis-resident Bloom filters, the [RedisBloom module](https://redis.io/docs/data-types/probabilistic/bloom-filter/) provides `BF.ADD`, `BF.EXISTS`. For in-process filters, [Guava's `BloomFilter`](https://guava.dev/releases/snapshot/api/docs/com/google/common/hash/BloomFilter.html) (Java) and `bloom` packages in Go and Python are standard. The Bloom filter is effectively a fourth tier: L0 → Bloom → L1 → L2 → origin. It only short-circuits *definite* misses, but on workloads that are mostly misses, that's most of the load.

## Stampedes at Each Tier — Same Problem, Different Layer

[Cache stampede](https://en.wikipedia.org/wiki/Cache_stampede) (a.k.a. "thundering herd," "dogpile") happens at every tier:

| Stampede location | Trigger | Fix |
|---|---|---|
| L1 stampede | Hot key expires from L1 on every node simultaneously | Per-node `LoadingCache` or `singleflight` coalescer |
| L2 stampede | Hot key expires from L2; thousands of nodes miss simultaneously | TTL jitter + L2-side coalescing via `SETNX` lock or [Redis script](https://redis.io/docs/manual/scripting/) |
| Origin stampede | L2 entry never warmed; mass miss hits DB | Pre-warm hot keys; circuit-break origin on overload |

The L1 case is solved entirely in-process by libraries that natively coalesce loaders (Caffeine `LoadingCache`, Ristretto's blocking `Get` semantics, Go's `singleflight.Group`). One thread fetches; concurrent waiters block on the same `Future`.

The L2 case is harder because it crosses processes: when L2 evicts a key, every application node that next requests it issues its own L2 GET, gets a miss, and falls through to DB. The classic fix is **probabilistic early expiration** ([Mookie's algorithm](https://www.vldb.org/pvldb/vol8/p886-vattani.pdf), aka XFetch): each fetcher has a small probability of refreshing the entry before its TTL expires, so the refresh load is spread across time rather than concentrated at expiration. Combined with a **distributed lock** (`SET key value NX PX 5000`) that designates one fetcher per cluster, this collapses N concurrent DB reads into 1.

The parent's [`./topology-awareness.md`](./topology-awareness.md) discusses how single-flight at the *shield* level (i.e., a designated proxy in front of L2 that coalesces cluster-wide misses) is the cleanest architecture for Redis-class L2 deployments. It's the same pattern the Instagram CDN deep-dive applies at the origin shield — fundamentally, **collapse misses upstream, before they fan out**.

## CDN as L0 — The Same Principle, One Layer Higher

Hierarchical caching extends both directions. Below L1 there's the CPU cache. Above L2 there's the **CDN edge**, which is L0 from the application's perspective: a cache between the user and the origin region, holding immutable HTML/JSON/static assets.

The trade-offs translate one-for-one:

| L0 (CDN) | L1 (in-process) | L2 (distributed) |
|---|---|---|
| Tens of POPs globally | One per process | One cluster per region |
| ~10–50 ms client→edge | ~100 ns | ~200 µs–1 ms |
| Holds public, often immutable content | Holds private hot snapshot | Holds shared working set |
| Invalidation by purge API | Invalidation by broadcast | Invalidation by `DEL` |
| `Cache-Control: max-age, immutable` | TTL + invalidation push | TTL + `DEL` |

The Instagram CDN deep-dive at [`../../social-media/instagram/cdn-strategy.md`](../../social-media/instagram/cdn-strategy.md) is the L0 story for an image-heavy product: content-addressing makes invalidation trivial, year-long TTLs make hit ratio extreme, and origin shielding collapses misses. **The same coherence/staleness/cost trade-offs reappear at L1/L2** — without content-addressing as an escape hatch, which is why L1/L2 needs the explicit invalidation protocol.

For HTML and JSON application responses, an edge cache is L0; for static images, fonts, and JS bundles, the CDN owns the entire stack and L1/L2 don't enter the picture.

## Worked Example — A Hot Product Page Across All Tiers

A request for `/products/42` on a hot product (1000 RPS sustained, viewed in every region):

```text
Request 1 (cold across the stack)
─────────────────────────────────
Browser → Edge (miss, this is dynamic/cacheable=no for HTML)
  Edge → App Node #17 (us-east-1)
    Node #17:
      L1 miss      [~100 ns wasted]
      L2 GET       [~500 µs]
      L2 miss
      DB SELECT    [~8 ms]
      Populate L2  [~500 µs, async-fire-and-forget]
      Populate L1  [~100 ns]
    ← response   [~10 ms total]

Request 2 (50 ms later, same node)
──────────────────────────────────
Browser → Edge → App Node #17
  Node #17:
    L1 hit         [~150 ns]
  ← response      [~150 ns + serialization + network]

Request 3 (50 ms later, different node #43, never seen this key)
────────────────────────────────────────────────────────────────
Browser → Edge → App Node #43
  Node #43:
    L1 miss        [~100 ns wasted]
    L2 GET         [~500 µs]
    L2 hit (warm from req #1)
    Populate L1    [~100 ns]
  ← response      [~600 µs]

Request 4 (after invalidation broadcast)
────────────────────────────────────────
Some other request triggered a price update:
  Origin write  → L2 DEL price:42  → Redis push "invalidate price:42"
  Every app node clears L1["price:42"] within ~5 ms

Request 5 (immediately after the push, on Node #17)
───────────────────────────────────────────────────
Node #17:
  L1 miss (just cleared)
  L2 GET (also just deleted; miss)
  DB SELECT      [~8 ms]
  Populate L2 + L1
← response      [~10 ms]
                ^ The first request after invalidation pays origin cost.
                  Subsequent requests on every node get L1 hits within 600 µs.
```

In steady state (no invalidations), the **read mix** for a hot key on a 100-node cluster looks like:

| Tier | Per-key share of reads | Per-request latency contribution |
|---|---|---|
| L1 | ~99% (after warm-up) | 100–500 ns |
| L2 | ~1% (cross-node first-touch + L1 evictions) | 200–500 µs |
| Origin | <0.01% (steady-state TTL refreshes) | 5–20 ms |

The amortized read latency is dominated by L1, which is what makes the architecture work. Each node pays a ~600 µs first-touch tax once per L1 lifetime per key, then serves nanoseconds.

## Configuration Sketch — Sizes and TTLs in One Place

The single configuration block that captures the whole multi-tier story for a typical service:

```yaml
cache:
  l1:
    library: caffeine
    maximum_size: 10000        # entries; bound for GC
    expire_after_write: 5s     # staleness bound; safety net for missed broadcasts
    expire_after_access: 30s   # evict cold entries
    record_stats: true

    # Eviction policy: W-TinyLFU (Caffeine default; do not change)
    # Removal listener: forwards EXPLICIT removals to invalidationBus

    negative_cache:
      enabled: true
      ttl: 5s                  # short — never confuse "doesn't exist yet" with permanent miss

  l2:
    backend: redis-cluster
    nodes: redis-0:6379,redis-1:6379,redis-2:6379
    client_tracking: enabled   # RESP3 invalidation push to L1
    expire_after_write: 1h     # holds the working set; TTL primarily for capacity, not freshness
    eviction_policy: allkeys-lru

    bloom_filter:
      enabled: true
      false_positive_rate: 0.01
      expected_inserts: 1_000_000_000

  invalidation_bus:
    transport: redis-pubsub      # OR kafka for higher durability
    channel: cache.invalidate
    delivery: at-least-once
    fallback: l1.expire_after_write   # if a broadcast is lost, TTL washes out within 5s
```

Three values in this block carry most of the design weight:

- `l1.maximum_size: 10000` — small enough to live in young-gen on G1GC, large enough to capture the Zipf head.
- `l1.expire_after_write: 5s` — the **staleness floor** on best-effort coherence. Tighter values trade hit rate for freshness; looser values rely more heavily on broadcast reliability.
- `l2.client_tracking: enabled` — the protocol-level mechanism that turns "5s stale" into "milliseconds stale" for every key actually mutated.

The corresponding [performance budget](../../../performance-observability/performance-budgets.md) for this service derives from these settings: with ~99% L1 hit rate, ~99% of requests should complete in <1 ms; with ~1% falling through to L2, the overall p99 budget should be ~2–5 ms; the rare origin fetch determines the p999 and pulls the histogram tail.

## Anti-Patterns

- **L1 without invalidation broadcast on writes.** Each node's L1 drifts independently; users see stale data for the full TTL after every mutation. Always pair L1 with a broadcast (Redis tracking, pub/sub, Kafka); let TTL be the safety net, not the primary mechanism.
- **L1 with unbounded TTL on mutating data.** "Never expire" + "writes mutate" guarantees permanent staleness.
- **Oversized L1 causing GC pauses.** A 4 GB Caffeine cache on G1GC produces 200–500 ms stop-the-world pauses. Cap L1 to fit in young gen, or move to ZGC/Shenandoah, or off-heap entries.
- **Separate L1 invalidation policy from L2.** If L2 invalidates on DELETE, L1 must too. L1 TTL must never exceed L2 TTL. Diverging policies create rarely-reproducible bugs that only appear under load.
- **Updating L2 instead of deleting on writes.** Concurrent writers race; last writer wins regardless of DB commit order. Always invalidate L2 and let the next reader repopulate.
- **Invalidating L1 before the origin write commits.** If origin write fails post-invalidation, the next reader populates L1 from origin's old value — masking the error. Origin first, invalidate after.
- **Forgetting negative-caching at L1.** A scraper probing random IDs hammers L2 forever. Add a 5-second negative cache.
- **Permanent negative-caching.** Long negative TTLs mask real creates. Seconds, not minutes.
- **Coherence solely via TTL on data that changes faster than the TTL.** A 60s TTL on inventory updated every 5s is wrong 92% of the time. Tighten TTL or add broadcast.
- **Cache key drift between L1 and L2.** L1 keyed by `productId`, L2 keyed by `tenant:productId` leaks tenant data across L1 hits. Centralize key derivation.
- **Not coalescing concurrent loaders.** Hot key expires, 1000 RPS all miss simultaneously, all hit origin. Use `LoadingCache` / `singleflight`.
- **Not jittering L2 TTLs.** Populate 10k entries at startup, all expire simultaneously an hour later — synchronized stampede. Add ±10–20% jitter at write time.
- **Treating L1 as authoritative.** Reads that *must* be fresh (financial transactions, authz checks) skip L1 or use versioned validation.
- **Ignoring hit-rate observability.** Without per-tier `hit_rate`, `eviction_rate`, `load_p99`, you cannot tune. Export the library's stats interface.
- **Mixing private user data and public reference data in one L1.** Eviction policy right for one is wrong for the other. Separate cache instances per data class.

## Related

- Parent case study: [`../design-distributed-cache.md`](../design-distributed-cache.md), specifically [§8 Multi-Tier Caching — L1 In-Process + L2 Distributed](../design-distributed-cache.md#8-multi-tier-caching--l1-in-process--l2-distributed).
- Sibling deep dive — partitioning: [`./partitioning-and-hash-slots.md`](./partitioning-and-hash-slots.md).
- Sibling deep dive — eviction policies: [`./eviction-policies.md`](./eviction-policies.md).
- Sibling deep dive — topology awareness: [`./topology-awareness.md`](./topology-awareness.md).
- Sibling deep dive — write strategies: [`./write-strategies.md`](./write-strategies.md).
- Building block — caching layers theory: [`../../../building-blocks/caching-layers.md`](../../../building-blocks/caching-layers.md).
- Performance observability — performance budgets across tiers: [`../../../performance-observability/performance-budgets.md`](../../../performance-observability/performance-budgets.md).
- Sibling design (CDN as L0 of the cache hierarchy): [`../../social-media/instagram/cdn-strategy.md`](../../social-media/instagram/cdn-strategy.md).

## References

- [Caffeine — Java caching library (wiki)](https://github.com/ben-manes/caffeine/wiki) — design, W-TinyLFU benchmarks, removal listeners, async loading.
- [Caffeine — efficiency comparison](https://github.com/ben-manes/caffeine/wiki/Efficiency) — empirical hit-rate comparison across cache policies.
- [Ristretto — Go cache by Dgraph](https://github.com/dgraph-io/ristretto) — TinyLFU admission, cost-based eviction, sharded structure for high-concurrency Go.
- [Introducing Ristretto — Dgraph blog](https://dgraph.io/blog/post/introducing-ristretto-high-perf-go-cache/) — design rationale.
- [HashiCorp golang-lru](https://github.com/hashicorp/golang-lru) — LRU/2Q/ARC implementations for Go.
- [Redis — Client-side caching (server-assisted)](https://redis.io/docs/manual/client-side-caching/) — RESP3 invalidation tracking, default vs broadcasting modes.
- [RESP3 Specification](https://github.com/antirez/RESP3/blob/master/spec.md) — protocol foundation for out-of-band invalidation pushes.
- [Hazelcast Near Cache](https://docs.hazelcast.com/imdg/4.2/performance/near-cache) — first-class L1 with built-in invalidation for Hazelcast IMDG.
- [Memcached](https://memcached.org/) — multi-threaded distributed memory cache; canonical L2 alternative to Redis.
- [Couchbase documentation](https://docs.couchbase.com/) — Memcached-compatible distributed cache with persistence.
- [Latency Numbers — annotated gist (Jonas Bonér)](https://gist.github.com/jboner/2841832) — Jeff Dean's table that motivates the latency hierarchy.
- [Peter Norvig — latency table](https://norvig.com/21-days.html#answers) — original presentation of the numbers.
- [Spring Framework — Cache Abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html) — `@Cacheable`/`@CacheEvict` and custom `CacheManager` for multi-level Caffeine + Redis stacks.
- [Facebook TAO (USENIX ATC 2013)](https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf) — multi-level caching of social-graph data at production scale.
- [Scaling Memcache at Facebook (NSDI 2013)](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final197.pdf) — Zipf distribution analysis; lease and regional-replica patterns.
- [Twitter — TwemCache workload analysis](https://engineering.twitter.com/en/insights/insights-from-one-year-of-cache-workload-analysis) — production Zipfian distribution evidence.
- [Pinterest Engineering blog](https://medium.com/pinterest-engineering/) — multi-tier caching evolution; single-tier to L1 + L2 + origin shielding.
- [Mookie's algorithm (VLDB 2015)](https://www.vldb.org/pvldb/vol8/p886-vattani.pdf) — XFetch probabilistic early expiration for L2 stampede prevention.
- [RFC 2308 — Negative Caching of DNS Queries](https://www.rfc-editor.org/rfc/rfc2308) — protocol-level reasoning for negative-caching TTL bounds.
- [RedisBloom](https://redis.io/docs/data-types/probabilistic/bloom-filter/) — `BF.ADD`/`BF.EXISTS` for L2-side definite-miss short-circuiting.
- [Guava BloomFilter](https://guava.dev/releases/snapshot/api/docs/com/google/common/hash/BloomFilter.html) — JVM in-process Bloom filter.
- [MESI cache coherence](https://en.wikipedia.org/wiki/MESI_protocol) — CPU-level coherence model that informs distributed-cache design.
- [cachetools (Python)](https://cachetools.readthedocs.io/) — TTL/LRU/LFU caches for Python services.
