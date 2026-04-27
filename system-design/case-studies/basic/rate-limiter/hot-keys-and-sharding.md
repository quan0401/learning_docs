---
title: "Rate Limiter Deep Dive — Hot Keys and Sharding"
date: 2026-04-27
updated: 2026-04-27
tags: [system-design, case-study, rate-limiter, deep-dive, hot-spot, sharding]
---

# Rate Limiter Deep Dive — Hot Keys and Sharding

**Date:** 2026-04-27 | **Updated:** 2026-04-27
**Tags:** `system-design` `case-study` `rate-limiter` `deep-dive` `hot-spot` `sharding`

## Table of Contents

- [Summary](#summary)
- [Overview — Why Hot Keys Are Special For Rate Limiters](#overview--why-hot-keys-are-special-for-rate-limiters)
- [Detection — Finding The Single Key That Is Killing You](#detection--finding-the-single-key-that-is-killing-you)
- [Random-Suffix Sharding](#random-suffix-sharding)
- [Hash-Based Suffix — Stable Routing With Cache Locality](#hash-based-suffix--stable-routing-with-cache-locality)
- [Local Pre-Aggregation](#local-pre-aggregation)
- [Per-Category Fast Path](#per-category-fast-path)
- [CountMinSketch Heavy-Hitter Detection](#countminsketch-heavy-hitter-detection)
- [Adaptive Sharding](#adaptive-sharding)
- [Pre-Warming And Cache Stampede](#pre-warming-and-cache-stampede)
- [Redis Cluster Hash Tags](#redis-cluster-hash-tags)
- [Monitoring Hot Keys In Production](#monitoring-hot-keys-in-production)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

A distributed rate limiter is, by construction, a workload that funnels traffic onto a small number of counter keys. That makes it the perfect victim of the hot-key problem: one celebrity key — a misconfigured CI bot, a viral endpoint, an anonymous-IP bucket that catches every botnet hit — can saturate one Redis primary while the rest of the cluster sits idle. This document is a deep dive on the mitigations summarised in section 7.4 of the [parent rate-limiter case study](../design-rate-limiter.md): how to detect a hot key (slow log, `MONITOR`, `redis-cli --hotkeys`, `OBJECT FREQ`), how to shard a single counter into many (random suffix vs hash-based suffix), how to absorb counts in-process before they ever hit Redis (local pre-aggregation, per-category fast paths), how to detect heavy hitters automatically with Count-Min Sketch, how to migrate shard counts safely when the topology must change, and the production monitoring you need to spot the next hot key before it spots you. The thesis: **a rate limiter without a hot-key story is one bad client away from an outage.**

## Overview — Why Hot Keys Are Special For Rate Limiters

Most caching workloads have a Zipfian access pattern — a small number of keys are read very often. A rate limiter has the same shape, but with two additional properties that make hot keys catastrophic instead of merely annoying:

1. **Every operation is a write.** `INCR`, `INCRBY`, Lua-scripted token-bucket updates — there is no way to serve a rate-limit decision from a read replica. Writes go to the primary owning the slot, full stop.
2. **The hot key is often unbounded.** A normal cache key cools off when the underlying object stops being requested. A rate-limit key for `anonymous_ip` keeps getting hit as long as bots exist, which is forever.

When one Redis primary owns the hot slot, you observe four symptoms in roughly this order:

- **Single shard CPU saturated.** One node sits at 100% CPU, the other 15 nodes sit at 8%. Per-node CPU graphs in Datadog or Prometheus show a textbook hockey stick on one line and flat lines everywhere else.
- **Tail latency climbs.** p50 stays calm, p99 and p99.9 explode. Every request that happens to hash to the hot slot waits behind a queue of in-flight Lua scripts.
- **Slow-log entries multiply.** Lua scripts that normally take 50 µs start taking 5 ms because the event loop is saturated.
- **Eventually, timeouts.** Clients give up at their request timeout (often 100 ms or 1 s) and you start seeing `fail_to_local` triggers in your rate-limiter dashboards if you wired them — see [section 7.5 of the parent case study](../design-rate-limiter.md).

The mitigations all do the same thing in different ways: **spread the writes for one logical key across either many physical keys or many local processes**, then reconcile.

## Detection — Finding The Single Key That Is Killing You

You cannot fix what you cannot see. Three production-tested techniques, in order of how disruptive they are to the running cluster.

### `redis-cli --hotkeys`

The cheapest first step. It uses `OBJECT FREQ` (LFU statistics, requires `maxmemory-policy allkeys-lfu` or `volatile-lfu`) to find keys with the highest access frequency.

```bash
$ redis-cli --hotkeys
# Scanning the entire keyspace to find hot keys as well as
# average sizes per key type. You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Hot key 'rl:edge:tb:endpoint=/v1/health:anon' found so far with counter 18234551
[12.34%] Hot key 'rl:edge:tb:endpoint=/v1/login:ip=203.0.113.42' found so far with counter 9824
...
```

Caveats: it requires LFU to be configured (LRU does not track frequency the same way), and it does a `SCAN` of the keyspace. On a multi-million-key cluster this is acceptable but not free; rate-limit at `-i 0.1` if you are paranoid.

### `SLOWLOG GET`

If a hot key is causing slow scripts, you will see them here. Anything taking more than 10 ms in a Lua-only workload is a smell.

```bash
redis-cli SLOWLOG GET 20
# Each entry: id, timestamp, duration_us, command_args
```

Look for the same key appearing in many slow-log entries — that is your hot key.

### `MONITOR` (use with extreme caution)

`MONITOR` streams every command the server processes. It is invaluable for forensics and almost always a bad idea on a production primary because it can use a substantial fraction of CPU just for the streaming. If you must use it, do it for seconds, not minutes, and pipe through `head` or a counter:

```bash
redis-cli MONITOR | head -n 100000 | awk '{print $4}' | sort | uniq -c | sort -rn | head
```

Output is a frequency table of the keys touched. The top entry is your suspect.

Better, in production: enable Redis Streams of the slow log (Redis 7+) or scrape `INFO commandstats` periodically and watch for `cmdstat_eval` calls per script SHA pulling away from the rest.

### Application-side detection

Add a Count-Min Sketch in the rate-limiter pod itself (see [Count-Min Sketch and Top-K](../../../data-structures/count-min-sketch-and-top-k.md)) and emit the top-K descriptors every minute. This catches hot keys before Redis does, because the sketch updates on every request without making a network round trip.

## Random-Suffix Sharding

The classic mitigation. Pick a fan-out `N` and split the logical key into `N` physical keys:

```text
rl:edge:tb:endpoint=/v1/health:anon  (logical)
  ├── rl:edge:tb:endpoint=/v1/health:anon:0
  ├── rl:edge:tb:endpoint=/v1/health:anon:1
  ├── ...
  └── rl:edge:tb:endpoint=/v1/health:anon:15
```

On each request, pick `i = rand() % N`, run the limiter against shard `i` with a per-shard limit of `floor(total_limit / N)`. The effective limit is `N × per_shard_limit`. With CRC16 routing in Redis Cluster, those 16 keys land on (probably) 16 different slots and (with a healthy cluster) several different primaries. The hot write is now distributed.

A token-bucket Lua script for one shard:

```lua
-- KEYS[1] = "rl:edge:tb:<descriptor>:<shard_idx>"
-- ARGV[1] = capacity (per-shard)        e.g. 100
-- ARGV[2] = refill_rate (tokens/sec)    e.g. 100
-- ARGV[3] = now_ms
-- ARGV[4] = cost (default 1)

local capacity    = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now_ms      = tonumber(ARGV[3])
local cost        = tonumber(ARGV[4])

local data = redis.call("HMGET", KEYS[1], "tokens", "ts")
local tokens = tonumber(data[1]) or capacity
local ts     = tonumber(data[2]) or now_ms

local elapsed = math.max(0, now_ms - ts) / 1000.0
tokens = math.min(capacity, tokens + elapsed * refill_rate)

local allowed = 0
if tokens >= cost then
  tokens = tokens - cost
  allowed = 1
end

redis.call("HMSET", KEYS[1], "tokens", tokens, "ts", now_ms)
redis.call("PEXPIRE", KEYS[1], 60000)

return {allowed, math.floor(tokens)}
```

### The over-allow tax

The trade-off is real and worth quantifying. With `N = 16` shards and a `1000 req/min` total limit, each shard gets `floor(1000/16) = 62` tokens per minute, total = 992 (rounding loses 8 req/min). That is fine — but at low rates the math goes sideways:

| Total limit (req/min) | Shards N | Per-shard floor | Effective ceiling | Worst-case over-allow |
|---|---|---|---|---|
| 1000 | 16 | 62 | 992 | 0% (slight under) |
| 100  | 16 | 6  | 96  | 0% (slight under) |
| 16   | 16 | 1  | 16  | 0% baseline, BUT ... |
| 16   | 16 | 1  | 16  | up to 16× burst across shards if requests arrive concurrently in a single second |
| 8    | 16 | 0  | 0  | totally broken — under-allows everything |

The dangerous case is **bursts at the boundary**. If the per-shard ceiling is 1 and the user sprays 16 concurrent requests across a 100 ms window, every shard says "yes" and you get a 16× burst. The fix is one of:

- **Only shard at high rates.** Don't shard a 10 req/min limit; shard a 10 000 req/min limit.
- **Round up rather than down**, accepting a small over-allow at the ceiling but no boundary bursts.
- **Use a token bucket with refill, not a fixed window** — the refill smooths the boundary.
- **Probabilistic shard selection.** Bias selection so that under-utilised shards get more traffic, but this gets complicated quickly.

Rule of thumb: only random-shard a key if `total_limit / N >= 100` and the period is at least one second.

## Hash-Based Suffix — Stable Routing With Cache Locality

Random selection has two problems: every request flips a coin and ends up on a different shard, which means **no cache locality** at any layer (Lua script cache, connection-level pipelining, even L1 cache on the Redis box), and **observability is harder** because you cannot trace a single client's history to one shard.

The alternative is hash-based suffix selection: route by `hash(client_id) % N`. The same client always lands on the same shard.

```go
// pickShardHashed returns the shard index for a given descriptor and client.
// Stable: same (descriptor, client) pair always routes to the same shard.
func pickShardHashed(descriptor, clientID string, shards int) int {
    h := fnv.New64a()
    _, _ = h.Write([]byte(descriptor))
    _, _ = h.Write([]byte{0}) // separator to avoid hash(a||b) == hash(ab')
    _, _ = h.Write([]byte(clientID))
    return int(h.Sum64() % uint64(shards))
}

func keyForShard(descriptor, clientID string, shards int) string {
    idx := pickShardHashed(descriptor, clientID, shards)
    return fmt.Sprintf("rl:edge:tb:%s:%d", descriptor, idx)
}
```

**Pros:**

- **Stable routing** — same client always uses the same shard. Easier to debug and to enforce a per-client local cache.
- **No boundary bursts** — a single client cannot fan out across all shards and burn the whole cumulative budget.
- **Cache locality on the Redis side** — the Lua script for that key stays hot in the script cache for that node.

**Cons:**

- **One pathological client can still saturate one shard.** If client `X` is the hot one and they all hash to shard 7, you've moved the hot key, not eliminated it.
- **Skew if descriptor cardinality is low** — if the only thing varying is the IP, and 10% of traffic comes from one IP, that IP's shard takes 10% of total writes regardless.

In practice, hash-based suffix is the right default for **client-bucket** style rate limits (per user, per API key) and random-suffix is the right default for **shared global** limits (per endpoint, per anonymous bucket). The two coexist and are picked per rule.

## Local Pre-Aggregation

The cheapest Redis call is the one you don't make. If a single edge pod is doing 50 000 INCRs/sec against the same shard key, an in-process aggregator can collapse them into one `INCRBY 50000` call per flush window.

```go
// Not production code; see the parent doc for the full architecture.
type LocalAggregator struct {
    mu        sync.Mutex
    counters  map[string]int64
    flushTick *time.Ticker
    flushFn   func(ctx context.Context, batch map[string]int64) error
}

func NewLocalAggregator(flushEvery time.Duration, flushFn func(context.Context, map[string]int64) error) *LocalAggregator {
    a := &LocalAggregator{
        counters:  make(map[string]int64),
        flushTick: time.NewTicker(flushEvery),
        flushFn:   flushFn,
    }
    go a.loop()
    return a
}

// Add records cost against key, returns the per-pod cumulative count
// (used for the cheap local check before flush).
func (a *LocalAggregator) Add(key string, cost int64) int64 {
    a.mu.Lock()
    defer a.mu.Unlock()
    a.counters[key] += cost
    return a.counters[key]
}

func (a *LocalAggregator) loop() {
    for range a.flushTick.C {
        a.mu.Lock()
        snapshot := a.counters
        a.counters = make(map[string]int64, len(snapshot))
        a.mu.Unlock()

        if len(snapshot) == 0 {
            continue
        }
        ctx, cancel := context.WithTimeout(context.Background(), 250*time.Millisecond)
        _ = a.flushFn(ctx, snapshot)
        cancel()
    }
}
```

`flushFn` issues one `INCRBY` per key (or pipelines them). With 100 ms flush windows and 200 pods, the Redis QPS for a hot key drops from `pods × per-pod-rate` to `(1 / flush_window) × pods` = `10 × 200 = 2000 QPS` regardless of underlying request rate.

### The trade-off — enforcement reaction time

Pre-aggregation is correctness-preserving over long horizons but **not** over windows shorter than the flush interval. With a 100 ms flush, a misbehaving client can burn `100 ms × per-pod-rate × pod-count` extra requests beyond the limit before any pod sees the bumped Redis counter and starts rejecting. For short-burst protection (e.g. anti-credential-stuffing), keep the flush window under 50 ms or skip pre-aggregation entirely.

A safer variant: **flush eagerly when a pod-local threshold is exceeded**. The pod still flushes on the timer for low-rate keys, but a key that crosses, say, 50% of its likely fair share inside a window flushes immediately. This caps over-allow at a single window's worth.

## Per-Category Fast Path

Some descriptors are known to be hot before any traffic arrives. The classic example is `anonymous` — every botnet, every CI run from a misconfigured IaC repo, every script-kiddie scanner — they all share the same descriptor. Rate-limiting them through Redis means paying the network round trip for traffic you intend to throttle to the floor.

The fast path: keep an **in-process LRU cache of token buckets** keyed by hot descriptor. Decide locally; periodically upload aggregate stats to Redis for visibility but never wait on Redis to make the decision.

```go
type LocalBucket struct {
    capacity   float64
    refillRate float64 // tokens per second
    tokens     float64
    lastNs     int64 // unix nanos of last refill
}

func (b *LocalBucket) take(nowNs int64, cost float64) bool {
    elapsed := float64(nowNs-b.lastNs) / 1e9
    if elapsed > 0 {
        b.tokens = math.Min(b.capacity, b.tokens+elapsed*b.refillRate)
        b.lastNs = nowNs
    }
    if b.tokens >= cost {
        b.tokens -= cost
        return true
    }
    return false
}

// LRU-backed map; sized so it fits comfortably in memory.
// Use github.com/hashicorp/golang-lru/v2 in production rather than rolling one.
type FastPath struct {
    mu      sync.Mutex
    cache   *lru.Cache[string, *LocalBucket]
    factory func(descriptor string) *LocalBucket
}

func (f *FastPath) Allow(descriptor string, cost float64) bool {
    f.mu.Lock()
    defer f.mu.Unlock()

    b, ok := f.cache.Get(descriptor)
    if !ok {
        b = f.factory(descriptor)
        f.cache.Add(descriptor, b)
    }
    return b.take(time.Now().UnixNano(), cost)
}
```

This pattern is what Stripe's rate-limiter blog calls "make the cheap decision locally first." Per-pod budgets are imperfect (pods cannot coordinate without a central store) but for descriptors where the goal is "throttle these to the floor regardless," coordination doesn't matter — you just want the request to die quickly.

## CountMinSketch Heavy-Hitter Detection

You don't always know which descriptors will be hot. A new endpoint goes viral. A customer's API key leaks. A scraper rotates user agents but stays on one ASN. You need automatic detection.

A Count-Min Sketch is a probabilistic frequency estimator that uses `O(log(1/δ) × 1/ε)` space to estimate counts with `ε`-bounded error and `1-δ` confidence. Combined with a `Top-K` heap, it gives you a streaming "what are the most-hit descriptors right now?" view in a few KB of memory. See the [data structures deep dive](../../../data-structures/count-min-sketch-and-top-k.md) for the math.

A minimal heavy-hitter detector for the rate limiter:

```go
type HeavyHitterDetector struct {
    cms       *cms.CountMinSketch     // depth=5, width=4096 ≈ 80 KB
    topK      *topk.Heap              // size=64
    threshold int64
    onHot     func(descriptor string) // promote to sharded path
}

func (h *HeavyHitterDetector) Observe(descriptor string) {
    h.cms.Add([]byte(descriptor), 1)
    estimate := h.cms.Estimate([]byte(descriptor))
    h.topK.Update(descriptor, estimate)
    if estimate >= h.threshold && !alreadyPromoted(descriptor) {
        h.onHot(descriptor)
    }
}

// Reset on a rotating window so old hot keys decay out.
func (h *HeavyHitterDetector) Rotate() {
    h.cms.Reset()
    h.topK.Reset()
}
```

Wiring in the limiter pipeline:

1. Every request observes its descriptor.
2. When a descriptor crosses the heavy-hitter threshold, the detector publishes an event ("`anonymous_ip:203.0.113.42` is hot").
3. The limiter's policy resolver re-routes that descriptor to the sharded path (random suffix or hash suffix) on the next request.
4. After a cool-off window (and absence from the top-K), the descriptor is demoted back to the single-key path.

Cloudflare describes a similar approach in their abuse-mitigation stack: streaming heavy-hitter detection feeds a denylist that absorbs DDoS hot keys before they reach the origin. The same data structure, different action.

## Adaptive Sharding

Once you have detection, you can change the shard count `N` dynamically: more shards when a key is hot, fewer when it cools. This is appealing because it bounds the over-allow cost of sharding to actual hot-key incidents.

The hard part is **migrating shard counts without losing in-flight tokens**. Naive `N=8 → N=16` resharding loses the half of requests that, during the transition, write to a key under the new scheme but read under the old (or vice versa). Three patterns work:

### Dual-write transition

For a transition window equal to the longest TTL of any shard:

1. Writers `INCR` the key under both old (`N=8`) and new (`N=16`) schemes.
2. Readers continue using the old scheme.
3. After the transition window, switch readers to the new scheme.
4. Stop writing to old shards.
5. Old shard keys expire naturally.

Cost: 2× write amplification during the window. Acceptable for minutes-scale TTLs.

### Generation-tagged keys

Encode the shard generation in the key:

```text
rl:edge:tb:endpoint=/v1/health:anon:gen=42:0
rl:edge:tb:endpoint=/v1/health:anon:gen=42:1
...
```

When you reshard, increment the generation. Readers that compute the wrong generation get a clean miss (or read zero, which is the correct behaviour for a brand-new bucket). The downside is you've thrown away the accumulated count, which means a rate-limit reset at every reshard.

### Migrate counts via SCAN + MERGE Lua

For schemes where you cannot afford the reset:

```lua
-- KEYS = old shard keys
-- ARGV = new shard key
local total = 0
for i, k in ipairs(KEYS) do
  local v = tonumber(redis.call("GET", k) or "0")
  total = total + v
  redis.call("DEL", k)
end
redis.call("INCRBY", ARGV[1], total)
return total
```

Run this script under a distributed lock during a brief lockout window. Practically, this is rare — most rate limiters can afford the reset.

In production, **manual sharding wins over adaptive sharding most of the time**. The adaptive logic is a non-trivial source of bugs, and the cases that benefit are the same cases where you should just leave a high `N` running permanently. Reach for adaptive only when shard count costs become noticeable (≥1000 hot descriptors with `N=64` ≈ 64 000 keys plus replicas — non-trivial memory).

## Pre-Warming And Cache Stampede

The first request after a hot key cools and its in-process state evicts is expensive. Two consequences:

- **Cold-cache thundering herd.** A burst of requests after a quiet period all miss the local LRU, all go to Redis, all execute the same Lua script. The Lua cache miss is cheap; the network round trip is not.
- **Synchronous TTL expiries.** If many shard keys share the same TTL (because they were all created at the same boot time), they expire together, and the next batch of requests recreates them all at once.

Mitigations:

- **Jittered TTLs.** Instead of `EXPIRE 60`, use `EXPIRE 60 + rand(0..10)`. Shards expire on a smear, not a wall.
- **Background pre-warm.** A goroutine periodically refreshes the shard's local state for known-hot descriptors so request-path code never observes a cold miss.
- **Single-flight on cold misses.** Use a `golang.org/x/sync/singleflight` group keyed by descriptor — only one in-flight Redis round trip per descriptor at a time. The rest wait for the result.
- **Don't expire hot keys.** If a key is in the heavy-hitter set, set a long TTL (or no TTL) and rely on LFU eviction instead.

These are the same techniques you use for any cache-stampede problem; the rate limiter is just one more cache.

## Redis Cluster Hash Tags

Redis Cluster routes by `CRC16(key) mod 16384`, then maps slots to nodes. Hash tags let you force a substring of the key to be the only thing the hash sees:

```text
KEY                                     SLOT
rl:edge:tb:user=alice:0                 CRC16(full string)   → varies
rl:edge:tb:user=alice:1                 CRC16(full string)   → varies
rl:edge:tb:{user=alice}:0               CRC16("user=alice")  → same as below
rl:edge:tb:{user=alice}:1               CRC16("user=alice")  → same as above
```

When and why you'd want hash tags in a rate limiter:

- **Multi-key Lua atomicity.** Lua scripts can only touch keys on the same node. If you need a script that compares the count from two related shards in one call, hash-tag them so they co-locate.
- **Pipeline efficiency.** Pipelined commands work best when consecutive ops target the same node. Co-locating reduces cross-slot ping-pong.
- **Atomic resharding migrations.** If you want one Lua script to merge `gen=41:*` into `gen=42:0`, all those keys must be on the same node.

The pitfall — and this is the fatal one — is **bad hash-tag design that puts every hot key on one node**. If you naively hash-tag everything by descriptor type:

```text
rl:edge:tb:{type=anonymous}:client=A:0
rl:edge:tb:{type=anonymous}:client=B:0
rl:edge:tb:{type=anonymous}:client=C:0
...
```

then every anonymous-bucket request, across all clients, ends up on whichever node owns the slot for `CRC16("type=anonymous")`. Hot key, but worse: hot **node**. The whole point of sharding the key was to spread load, and the hash tag undid that.

Rule: **only hash-tag when you positively need atomic multi-key access**, and tag by something high-cardinality (e.g. `{client=alice}` not `{type=anonymous}`).

For typical sharded rate-limit keys, **no hash tag** is the right default. Each shard hashes independently and lands on different nodes. That is the entire point.

## Monitoring Hot Keys In Production

You need standing dashboards, not ad-hoc investigation. The minimum viable set:

| Metric | Source | Why |
|---|---|---|
| Per-node Redis CPU | Datadog `redis.cpu.user`, Prometheus `redis_cpu_usage` | First indicator of skew. One line above the rest = hot key. |
| Per-node ops/sec | `INFO commandstats`, Datadog `redis.net.commands` | Confirms skew is in command volume, not just CPU. |
| Per-key estimated frequency | `OBJECT FREQ <key>` (LFU mode) | Direct hot-key signal at the key level. |
| Top-N descriptors per minute | App-side Count-Min + Top-K, emitted as a metric | Heat-map material; catches hot keys before Redis does. |
| Lua script latency p99 | `INFO` → `latency_history` per script SHA, or app-side timing | Hot key = slow script. |
| Slow log entry rate | `redis-cli SLOWLOG LEN` scraped periodically | Trend, not absolute. Sudden jumps = trouble. |
| Fail-to-local trigger rate | App-emitted counter | When the hot key exceeds your timeout budget, this fires. |
| Per-shard token-bucket utilisation | App-emitted, dimensioned by shard index | Shows when one shard is consistently hot — sharding strategy needs revisiting. |

Dashboards:

- **Heat map** — descriptor on Y axis, time on X axis, colour = ops/sec. The hot bar jumps out instantly. Grafana `Heatmap` panel handles this; Datadog has `top_list` widgets that work too.
- **Per-node ops bar chart** — bars sorted by ops, compared against a rolling 7-day baseline. One bar 10× the rest = page someone.
- **Top-K descriptors over time** — sparkline per top descriptor, refresh every minute. Lets you spot sudden viral keys.

Alerts:

- **CPU skew alert.** `max(node_cpu) - median(node_cpu) > 30%` for 5 minutes.
- **Lua p99 alert.** Scripted-EVAL p99 > 10 ms for 2 minutes.
- **Top-K reshuffle alert.** A descriptor that wasn't in yesterday's top-32 has jumped to top-3 — page someone or auto-promote to sharded path.

Production-grade ops on a hot rate-limit key looks like: Count-Min flags it, Heavy-Hitter detector promotes it to the sharded path, dashboards show the spread across new shards, on-call confirms within an SLO. If any one of those steps is missing, you're flying blind.

## Anti-Patterns

Mistakes that will bite you, in roughly the order people make them:

- **Sharding a low-rate key.** Rounding losses or boundary bursts will dominate. Only shard when `total_limit / N >= 100` and the period is at least one second.
- **Hash-tagging by descriptor type.** Forces every anonymous bucket onto one node. Defeats the entire purpose of sharding.
- **Adaptive sharding everywhere by default.** Adds operational complexity for cases where static `N=16` would have worked fine. Reach for adaptive only when memory cost forces your hand.
- **Pre-aggregation without backstop.** If your flush is 1 s and your limit is 100 req/s, a malicious client can burn 100 × pods extra requests per window. For abuse-protection rules, keep flush under 50 ms.
- **No fail-to-local for hot-key incidents.** When the hot key takes Redis down, you need a degraded mode that still rate-limits something. See [section 7.5 of the parent case study](../design-rate-limiter.md).
- **Detection without action.** Logging "hot key detected" without auto-promoting to a sharded path means a human has to wake up. Wire the action.
- **Assuming `OBJECT FREQ` is free.** It's a SCAN-style scan over the keyspace. On a 50M-key cluster, run it sparingly.
- **One TTL value for all shards.** Synchronised expiry creates synchronised cold-miss bursts. Always jitter.
- **Treating the rate limiter as stateless.** It is the most stateful service in your edge tier. Hot keys, expiries, shard topology — all of it requires the same operational discipline as a database.
- **Sharding without observability.** You cannot fix hot shards you cannot see. Per-shard QPS, per-node CPU, top-K descriptors are non-negotiable.

## Related

- [Sharding Strategies — Range, Hash, Directory, Geo, Consistent Hashing](../../../scalability/sharding-strategies.md) — the general sharding playbook this document specialises for rate-limit counters.
- [Count-Min Sketch and Top-K](../../../data-structures/count-min-sketch-and-top-k.md) — the data structure powering heavy-hitter detection.
- [Designing a Distributed Rate Limiter](../design-rate-limiter.md) — parent case study; section 7.4 is the seed for this deep dive, sections 7.5–7.6 cover failure modes and multi-tier limits.
- [Backpressure, Bulkheads, Circuit Breakers](../../../scalability/backpressure-bulkhead-circuit-breaker.md) — what to do downstream when the rate limiter itself is being overwhelmed.
- [Read/Write Splitting & Cache Strategies](../../../scalability/read-write-splitting-and-cache-strategies.md) — stampede protection patterns reused by the pre-warming section above.

## References

- [Redis — `redis-cli --hotkeys` documentation](https://redis.io/docs/latest/develop/tools/cli/#identify-hot-keys-using-the-redis-cli) — official guidance on detecting hot keys with LFU stats.
- [Redis — Cluster specification (hash tags)](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/) — exact rules for `{...}` hash-tag extraction and the 16384-slot scheme.
- [Redis — `OBJECT FREQ` command](https://redis.io/docs/latest/commands/object-freq/) — LFU access-frequency counter.
- [Redis — Scaling with Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/) — slot migration, resharding, and the operational shape of a cluster.
- [Cormode & Muthukrishnan, "An improved data stream summary: the count-min sketch and its applications" (2005)](https://sites.cs.ucsb.edu/~suri/cs290/CormodeMuthu.pdf) — the foundational paper for the heavy-hitter detector.
- [Cloudflare blog — "How we built rate limiting capable of scaling to millions of domains"](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/) — counting at scale, sketch-based heavy-hitter detection in production.
- [Stripe — "Scaling your API with rate limiters"](https://stripe.com/blog/rate-limiters) — production rate-limiter patterns, including the local-first decision philosophy.
- [Envoy — Global rate limiting](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_features/global_rate_limiting) — fail-to-local fallback and shadow-mode patterns referenced for hot-key incidents.
- [AWS — DynamoDB partition key design and adaptive capacity](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html) — different system, same hot-partition problem and a comparable mitigation surface.
