---
title: "Rate Limiter Deep Dive — Race Conditions and Atomicity"
date: 2026-04-27
updated: 2026-04-27
tags: [system-design, case-study, rate-limiter, deep-dive, concurrency, atomicity]
---

# Rate Limiter Deep Dive — Race Conditions and Atomicity

**Date:** 2026-04-27 | **Updated:** 2026-04-27
**Tags:** `system-design` `case-study` `rate-limiter` `deep-dive` `concurrency` `atomicity`

## Table of Contents

- [Summary](#summary)
- [Overview — Why Rate Limiters Are a Concurrency Problem](#overview--why-rate-limiters-are-a-concurrency-problem)
- [The INCR + EXPIRE Race](#the-incr--expire-race)
  - [The Naive Implementation](#the-naive-implementation)
  - [Failure Mode 1 — Leaked Keys](#failure-mode-1--leaked-keys)
  - [Failure Mode 2 — Lost Expirations and Window Drift](#failure-mode-2--lost-expirations-and-window-drift)
  - [Why MULTI/EXEC Does Not Save You](#why-multiexec-does-not-save-you)
- [Lua Atomicity — The Default Fix on Redis](#lua-atomicity--the-default-fix-on-redis)
  - [What "Atomic" Means in Redis](#what-atomic-means-in-redis)
  - [EVAL vs EVALSHA Performance](#eval-vs-evalsha-performance)
  - [Lua Pitfalls — Determinism and Slow Scripts](#lua-pitfalls--determinism-and-slow-scripts)
- [Redis Functions (7.0+)](#redis-functions-70)
- [Composable Atomic Primitives](#composable-atomic-primitives)
  - [SET ... NX EX — First-Write-With-TTL](#set--nx-ex--first-write-with-ttl)
  - [INCRBYFLOAT — Token-Bucket Fractions](#incrbyfloat--token-bucket-fractions)
  - [Sorted Sets and ZADD NX](#sorted-sets-and-zadd-nx)
- [WATCH/MULTI/EXEC — Optimistic Compare-and-Swap](#watchmultiexec--optimistic-compare-and-swap)
- [Distributed Locks — SET NX, Redlock, and Fencing Tokens](#distributed-locks--set-nx-redlock-and-fencing-tokens)
  - [The Single-Node SET NX Lock](#the-single-node-set-nx-lock)
  - [Redlock and the Kleppmann/Antirez Debate](#redlock-and-the-kleppmannantirez-debate)
  - [Fencing Tokens](#fencing-tokens)
  - [When to Reach for a Lock in a Rate Limiter](#when-to-reach-for-a-lock-in-a-rate-limiter)
- [Idempotent Counter Operations](#idempotent-counter-operations)
- [DynamoDB — Conditional Update and Atomic Counters](#dynamodb--conditional-update-and-atomic-counters)
- [SQL-Based Rate Limiters — SELECT FOR UPDATE and Advisory Locks](#sql-based-rate-limiters--select-for-update-and-advisory-locks)
- [Crash Recovery Semantics](#crash-recovery-semantics)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

Section 7.3 of the parent rate-limiter case study introduced the classic `INCR` + `EXPIRE` race and waved at a Lua fix. This deep dive expands that page-and-a-half into the full atomicity vocabulary you actually need to ship a correct rate limiter: why pipelining and `MULTI/EXEC` do not buy atomicity across between-command interleavings, why Redis Lua scripts and Redis Functions do, which native commands compose safely (`SET NX EX`, `INCRBYFLOAT`, `ZADD NX`), when optimistic CAS via `WATCH/MULTI/EXEC` is preferable, why distributed locks (and Redlock in particular) are usually the wrong tool for rate limiting, and how the same problem looks on DynamoDB and PostgreSQL when Redis is not on the menu. Throughout, the unifying claim is the same: **a counter increment paired with a TTL is not one operation unless the storage system makes it one** — and the cost of getting that wrong is leaked keys, drifted windows, double-charged users, or silently bypassed limits.

## Overview — Why Rate Limiters Are a Concurrency Problem

A rate limiter, stripped to its essentials, runs three operations per request:

1. **Read** the current state for a key (count, token balance, sorted-set window, leaky-bucket level).
2. **Decide** whether the request is allowed, given the limit and the current state.
3. **Update** the state to reflect the request that just happened — and, on first observation of a key, attach a TTL so it does not live forever.

Every one of those three steps can race against another request hitting the same key on a different pod, in a different region, or even in a different goroutine on the same pod. The hot key is the contention point by definition, so a rate limiter is a concurrency problem masquerading as a counting problem. Get the atomicity wrong and you have a security control that silently fails open under load — exactly when you needed it.

The rest of this doc walks the failure modes and the primitives that fix them, in roughly the order you should reach for them.

## The INCR + EXPIRE Race

### The Naive Implementation

The example from section 7.3 of the parent doc is the canonical wrong implementation:

```go
// WRONG — race condition between INCR and EXPIRE
func Allow(ctx context.Context, redis *redis.Client, key string, limit int64, window time.Duration) (bool, error) {
    count, err := redis.Incr(ctx, key).Result()
    if err != nil {
        return false, err
    }
    if count == 1 {
        // Window: only set TTL on the first increment.
        if err := redis.Expire(ctx, key, window).Err(); err != nil {
            return false, err
        }
    }
    if count > limit {
        return false, nil
    }
    return true, nil
}
```

It looks innocuous. `INCR` is atomic, `EXPIRE` is atomic, and we only call `EXPIRE` on the very first increment. Two things go catastrophically wrong at scale.

### Failure Mode 1 — Leaked Keys

If the process crashes, panics, or has its connection killed _between_ the successful `INCR` and the `EXPIRE` call, the key is durably created with no TTL. Redis is now holding that key forever (until manual eviction or `maxmemory` policy). At a million unique keys per minute (think per-IP limiters under DDoS), that is a memory leak that brings the cluster down within hours.

A subtler version: the goroutine is preempted, the request context is cancelled, and the `EXPIRE` call returns a context-cancelled error that the caller swallows because "the rate-limit check passed, that's all we cared about." The key still leaks. The bug is silent until ops notices the working set creeping upward day over day.

### Failure Mode 2 — Lost Expirations and Window Drift

A more interesting race: imagine the key is about to expire. Client A does `INCR`, gets `count = 50` (well under the limit of 100). Between A's `INCR` and any subsequent operation, the key's TTL fires and the key is deleted. Client B does `INCR`, gets `count = 1`, and now sets a fresh `EXPIRE` for a brand-new window — _but it is reasoning about a window that started 59.9 seconds ago_. A and B think they are in the same window but the storage layer disagrees. Multiply by enough clients and you get phantom "windows" that drift by tens of seconds, and the effective limit is somewhere between `limit` and `2 × limit` depending on luck.

A still more interesting race: two clients _both_ do `INCR` on a freshly expired key in close succession. Both get `count == 1`. Both call `EXPIRE`. The second `EXPIRE` resets the TTL. Now the window has been silently extended by the gap between the two calls. Over thousands of requests, the window length you advertise to users is not the window length you actually enforce.

### Why MULTI/EXEC Does Not Save You

The instinctive Redis-developer fix is: "Wrap it in a transaction":

```go
// STILL WRONG — MULTI/EXEC is not atomic across between-command thinking time
pipe := redis.TxPipeline()
incr := pipe.Incr(ctx, key)
pipe.Expire(ctx, key, window)
_, _ = pipe.Exec(ctx)
count := incr.Val()
if count > limit {
    return false, nil
}
return true, nil
```

This is better — Redis will queue the commands and execute them as a single block on the server, so you no longer have a network round-trip between `INCR` and `EXPIRE`. But it still has bugs:

1. **You lost the conditional** — you now `EXPIRE` on every call, which resets the TTL on every increment. The window continuously slides forward and effectively never expires under sustained traffic. You are now running a leaky bucket with infinite capacity.
2. **MULTI/EXEC is not the kind of atomicity you wanted.** It is "all or nothing as a batch" — it does not give you the ability to **branch** on the result of `INCR` inside the transaction. The conditional `if count == 1` lives in your Go code, not in Redis, and that is precisely where the race lives.

To express "INCR, and if this was the first one, also EXPIRE" as a single server-side operation, you need the server to run your decision logic. That is what Lua scripting (or Redis Functions, see below) is for.

## Lua Atomicity — The Default Fix on Redis

The standard fix is to ship the logic as a Lua script that Redis runs server-side as a single atomic unit:

```lua
-- correct: INCR + conditional EXPIRE atomically
-- KEYS[1] = the limiter key
-- ARGV[1] = window seconds
-- ARGV[2] = limit
local count = redis.call("INCR", KEYS[1])
if count == 1 then
  redis.call("EXPIRE", KEYS[1], ARGV[1])
end
if count > tonumber(ARGV[2]) then
  return 0
end
return 1
```

Invoked from the application:

```go
var allowScript = redis.NewScript(`
  local count = redis.call("INCR", KEYS[1])
  if count == 1 then
    redis.call("EXPIRE", KEYS[1], ARGV[1])
  end
  if count > tonumber(ARGV[2]) then
    return 0
  end
  return 1
`)

func Allow(ctx context.Context, rdb *redis.Client, key string, limit int64, window time.Duration) (bool, error) {
    res, err := allowScript.Run(ctx, rdb, []string{key}, int64(window/time.Second), limit).Int()
    if err != nil {
        return false, err
    }
    return res == 1, nil
}
```

### What "Atomic" Means in Redis

Redis's atomicity guarantee for Lua is documented on the [`EVAL` reference page][redis-eval]: Redis executes commands single-threaded, and a Lua script monopolises that single thread for the duration of its execution. **No other command from any client can run between two `redis.call` invocations inside your script.** That is the property `MULTI/EXEC` cannot give you for branch-based logic.

This is also why the script must be _fast_. While your Lua script is running, every other Redis client is blocked. A 50 ms script on a Redis primary that handles 50,000 ops/sec stalls 2,500 operations. Lua scripts in a rate limiter should be a handful of `redis.call` invocations and absolutely no loops over large data sets.

### EVAL vs EVALSHA Performance

`EVAL` ships the entire script body on every call. For a 200-byte rate-limit script at 200,000 RPS, that is 40 MB/s of script bytes hitting the wire — pure waste. `EVALSHA` ships only the SHA-1 digest of a previously-loaded script. The flow is:

1. On startup (or on first call), the client calls `SCRIPT LOAD` once and caches the returned SHA-1.
2. Every subsequent call uses `EVALSHA <sha1> ...`, sending only 40 hex bytes plus arguments.
3. If Redis returns `NOSCRIPT` (the script cache was flushed by a `SCRIPT FLUSH` or a failover to a replica that never saw the load), the client falls back to `EVAL` (which re-caches it) and retries.

Most Redis client libraries do this transparently — the `Script` helper in `go-redis`, `redis-py`'s `register_script`, and `Lettuce`'s `digest()` API all implement the EVALSHA-with-EVAL-fallback dance for you. Use the helper, do not call `EVAL` directly in hot paths.

### Lua Pitfalls — Determinism and Slow Scripts

A few sharp edges:

- **Determinism.** Lua scripts must be deterministic given the same inputs. Do not call `os.time()` or `math.random()` inside a script — pass time and randomness in via `ARGV` so replication and AOF replay produce the same result.
- **Long-running scripts.** The `lua-time-limit` config (default 5 seconds) governs when Redis will start logging that a script is taking too long. Once a script has done a write, it cannot be killed without `SCRIPT KILL --SHUTDOWN NOSAVE`. Keep scripts short and write nothing until you are sure you will commit.
- **Replication mode.** Modern Redis (5+) replicates effects rather than the script itself by default (`replicate_commands` is on). This avoids replicas drifting from a non-deterministic script, but it also means you should not rely on the script body being identical on the replica — only its observable effects.

## Redis Functions (7.0+)

Redis 7.0 introduced [Redis Functions][redis-functions] as the modern alternative to ad-hoc Lua scripts. Functions are the same Lua execution model with three quality-of-life improvements:

1. **Persisted across restarts.** `EVAL` and `SCRIPT LOAD` populate an in-memory cache that is wiped on every restart (or failover). Functions are stored in the RDB and AOF, so they survive restarts and are replicated to replicas at registration time, not at first call. No more `NOSCRIPT` retries.
2. **Versioned namespacing.** Functions live in named libraries, so `FUNCTION LOAD REPLACE` cleanly upgrades them by name rather than by SHA. You can manage them like code instead of like opaque blobs.
3. **Better tooling.** `FUNCTION LIST`, `FUNCTION DUMP`, and `FUNCTION RESTORE` give you proper deploy/rollback semantics.

A rate-limit function looks almost identical to the Lua script:

```lua
#!lua name=ratelimit_lib

redis.register_function('fixed_window_allow', function(keys, args)
  local count = redis.call('INCR', keys[1])
  if count == 1 then
    redis.call('EXPIRE', keys[1], args[1])
  end
  if count > tonumber(args[2]) then
    return 0
  end
  return 1
end)
```

Loaded once with `FUNCTION LOAD`, called with `FCALL fixed_window_allow 1 <key> <window> <limit>`. All the atomicity guarantees of `EVAL` apply. For new development on Redis 7+, prefer Functions over `EVALSHA`. For older Redis or for one-shot scripts where deploy tooling is overkill, Lua + EVALSHA is still the right answer.

## Composable Atomic Primitives

Sometimes you do not need scripting at all — Redis ships several primitives that compose two operations into one atomic unit. Use these wherever they fit; they are simpler to reason about than Lua and impose no scripting surface area.

### SET ... NX EX — First-Write-With-TTL

Plain `SETNX` (deprecated in spirit) sets a key only if it does not exist, but does not let you attach a TTL atomically — leaving you back in the `INCR + EXPIRE` race. The modern form uses `SET` with options:

```redis
SET ratelimit:user:1234 1 NX EX 60
```

This is one atomic operation: "if the key does not exist, set it to `1` with a 60-second TTL; otherwise do nothing." Used for **first-write-with-TTL** patterns where you want to initialise the key and its window in one shot, then increment unconditionally afterwards. A common pattern is:

```redis
SET ratelimit:user:1234 0 NX EX 60   -- idempotent initialisation
INCR ratelimit:user:1234              -- always safe, TTL already set
```

The `INCR` no longer needs to set a TTL because the `SET NX EX` already did. The two-command form is technically two round trips but each is independently atomic and the second cannot leak a no-TTL key.

### INCRBYFLOAT — Token-Bucket Fractions

A token-bucket rate limiter that refills at, say, 1.5 tokens per second cannot use `INCR` (integer-only). `INCRBYFLOAT` lets you do atomic increments with floating-point deltas:

```redis
INCRBYFLOAT bucket:user:1234 0.025
```

`INCRBYFLOAT` is atomic but float-precision: it stores up to 17 significant digits and is subject to the usual IEEE-754 surprises. For most rate limiters this is fine — we are tracking thousands or millions of tokens, not currency cents. If you need exact arithmetic, scale by 1000 or 10000 and use plain `INCRBY` on integers.

Note: `INCRBYFLOAT` is one atomic step, but a typical token-bucket _refill_ also needs to read elapsed time, compute the increment, and **cap at the bucket size**. Capping requires comparing against a max, which is logic — so token-bucket refill in Redis is one of the cases where you fall back to Lua. See section 7.2 of the parent doc for the full token-bucket script.

### Sorted Sets and ZADD NX

A sliding-window rate limiter using sorted sets stores per-request entries scored by timestamp. `ZADD ... NX` adds only if the member is not present, in one atomic step:

```redis
ZADD ratelimit:user:1234:window NX <timestamp> <request-id>
ZREMRANGEBYSCORE ratelimit:user:1234:window 0 <timestamp - window-ms>
ZCARD ratelimit:user:1234:window
EXPIRE ratelimit:user:1234:window <window-seconds>
```

Each command is atomic but the four-command sequence is not — you would still wrap it in a Lua script (or a `MULTI/EXEC` if you do not need branching) to make the whole sliding-window check atomic. The `ZADD NX` flag is what saves you from double-counting a request that was somehow submitted with a duplicate request-id.

## WATCH/MULTI/EXEC — Optimistic Compare-and-Swap

Redis has an honest-to-goodness CAS primitive: `WATCH`. The protocol is:

1. `WATCH key1 key2 ...` — tell Redis you intend to read these and conditionally write.
2. Read the current values with `GET`, `MGET`, `HGET`, etc.
3. Compute the desired update in your client.
4. `MULTI; <writes>; EXEC` — submit the writes as a transaction.
5. If any watched key changed between step 1 and step 4, `EXEC` returns `nil` and you retry from step 1.

```python
import redis

r = redis.Redis()

def consume_token(key: str, capacity: int, refill_per_sec: float, max_retries: int = 5) -> bool:
    """Token-bucket consume via WATCH/MULTI/EXEC. Returns True if a token was consumed."""
    for _ in range(max_retries):
        with r.pipeline() as pipe:
            try:
                pipe.watch(key)
                state = pipe.hgetall(key)
                now = time.time()
                tokens = float(state.get(b"tokens", capacity))
                last = float(state.get(b"last", now))
                tokens = min(capacity, tokens + (now - last) * refill_per_sec)
                if tokens < 1.0:
                    pipe.unwatch()
                    return False
                tokens -= 1.0

                pipe.multi()
                pipe.hset(key, mapping={"tokens": tokens, "last": now})
                pipe.expire(key, 3600)
                pipe.execute()
                return True
            except redis.WatchError:
                continue  # someone else changed the key, retry
    return False  # gave up after retries
```

This works, and it has one nice property over Lua: the **compute step is in your application language**, so complex logic that would be painful in Lua (e.g., decisions involving JSON parsing, regexes, or library calls) stays in Python or Go.

The cost is brutal under contention: every retry is **two round trips** (the watched read and the EXEC), and every contended-key request can fail multiple times in a row. On a hot key with N concurrent clients, expected retries scale as `O(N)` — which is exactly the case where rate limiters spend their time. On the same workload Lua takes one round trip with no retries. **For rate limiters specifically, prefer Lua.** `WATCH/MULTI/EXEC` is the right tool for low-contention compare-and-swap on cold data, not for the hot key contention rate limiters live on.

## Distributed Locks — SET NX, Redlock, and Fencing Tokens

Locks come up in rate-limiter discussions because someone always asks "should we just take a lock per user before deciding?" The answer is almost always **no** — you do not need a lock if you have an atomic primitive that does the same job. But it is worth understanding why.

### The Single-Node SET NX Lock

The simplest distributed lock on Redis is `SET key value NX EX ttl`:

```redis
SET lock:resource <unique-token> NX EX 30
```

If the response is `OK`, you hold the lock. To release safely you must verify you still own it — otherwise you might delete a lock that another client acquired after your TTL expired:

```lua
-- correct unlock: only delete if we still own it
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
```

This is a real lock and works fine inside a single Redis primary's view. It has two known weaknesses:

1. **No protection against TTL expiration mid-critical-section.** If your process pauses (GC, page-fault, disk-stall) longer than the TTL, the lock is auto-released and someone else can acquire it while you are still mid-work.
2. **No protection against Redis primary failover.** If the primary fails after acknowledging your `SET NX` but before replicating it, the new primary will not know the lock exists, and another client can acquire it.

### Redlock and the Kleppmann/Antirez Debate

[Redlock][redlock] is Antirez's proposed multi-master lock algorithm: acquire the same lock on a majority of N independent Redis nodes, with a timeout. The pseudocode:

```text
function acquire(key, value, ttl):
    start = now()
    success = 0
    for node in [r1, r2, r3, r4, r5]:
        if node.SET(key, value, NX, PX=ttl) == OK:
            success += 1
    elapsed = now() - start
    valid_time = ttl - elapsed - clock_drift
    if success >= 3 and valid_time > 0:
        return value, valid_time
    else:
        for node in [r1..r5]:
            node.unlock(key, value)
        return None
```

[Martin Kleppmann's critique][kleppmann-locking] is essentially: **timing-based locks are broken in the presence of GC pauses, network partitions, and clock skew, regardless of how many Redis nodes you replicate to**. A process can acquire the lock, get suspended for longer than the TTL, then resume believing it still holds the lock — and during the gap another client legitimately took ownership. No amount of multi-master replication fixes this; it is a fundamental property of timeout-based mutual exclusion. He argues that for **correctness** (e.g., "only one writer to this row") you need a proper consensus system (Zookeeper, etcd) plus a fencing token.

[Antirez's response][antirez-redlock] pushes back on the specifics of the system model and argues that, with reasonable timing assumptions, Redlock provides safety useful enough for many real workloads.

The honest summary: Kleppmann is right about correctness; Antirez is right that Redlock is good enough for some use cases. But neither side disagrees that **timing-based locks alone do not give you correctness** — both agree you need fencing for that.

### Fencing Tokens

A fencing token is a monotonically increasing number issued at lock-acquire time. The protected resource (the database, the queue) refuses any operation tagged with a token less than the highest one it has seen. So even if your process pauses past the TTL and two clients believe they hold the lock simultaneously, only the one with the higher token can actually mutate state. This requires the protected resource to participate in the protocol — which is why fencing works for "writes to a database with conditional updates" but does not work for "writes to a dumb network service that does not check tokens."

### When to Reach for a Lock in a Rate Limiter

Almost never. The whole point of an atomic primitive (Lua, INCR, conditional update) is that you do not need a lock — the storage system serialises updates for you. You should only reach for a lock when:

- The decision logic is too complex to fit in a Lua script (rare for rate limiters).
- You need to coordinate with an external system that does not have its own atomic update (e.g., calling a billing API to "consume credits" before allowing the request).
- You are doing administrative work like "rebuild the rate-limit configuration" and want exclusivity for a few seconds.

For the per-request hot path, locks are a step backward from the atomic approaches above. The contended-lock failure mode (one slow client blocks everyone else) is exactly what a rate limiter cannot afford.

## Idempotent Counter Operations

"Exactly once" is a phrase that hides a lot of detail. In a rate-limiter context, it means: **a single logical client request should be counted exactly once against the limit, even when the network retries the underlying RPC.**

The naive idea is to deduplicate by request-ID:

```text
on request(req_id, user):
  if SET dedup:{req_id} 1 NX EX 600 == OK:
    INCR limit:{user}
    proceed
  else:
    return cached_result_for(req_id)
```

This _almost_ works but has several real problems:

1. **What is the request-ID?** If the client picks it (a UUID per request), a malicious or buggy client can bypass the limiter by sending the same UUID for every request — every second-and-onward call hits the dedup short-circuit and is not counted. You need server-derived or signed request-IDs to prevent this.
2. **What happens between the dedup-set and the INCR?** If the process crashes here, the dedup entry exists but the counter never moved. The client retries, hits the dedup entry, and is told "already counted" — but the limit budget never decreased. Now the user has effectively a free request. Fixing this requires writing the dedup entry _and_ the counter increment as one Lua script.
3. **TTL of the dedup entry.** Too short and clients can re-submit the same ID after expiry and bypass the limit. Too long and the dedup table grows without bound. The right TTL is "longer than your worst-case retry window."
4. **Stamping with a monotonic ID.** A common production pattern is for the API gateway to stamp every inbound request with a server-generated monotonic ID (e.g., a Snowflake ID) and use _that_ for dedup. The client cannot replay it because it is server-generated; it is monotonic so old IDs can be efficiently aged out; and it carries timestamp metadata so you can shape the dedup TTL precisely.

The full pattern in Lua:

```lua
-- KEYS[1] = limit key (e.g. ratelimit:user:1234)
-- KEYS[2] = dedup key (e.g. dedup:user:1234:reqid:xyz)
-- ARGV[1] = window seconds, ARGV[2] = limit, ARGV[3] = dedup TTL
if redis.call("EXISTS", KEYS[2]) == 1 then
  return 2  -- duplicate, no-op (caller should treat as previous result)
end
local count = redis.call("INCR", KEYS[1])
if count == 1 then
  redis.call("EXPIRE", KEYS[1], ARGV[1])
end
redis.call("SET", KEYS[2], "1", "EX", ARGV[3])
if count > tonumber(ARGV[2]) then
  return 0
end
return 1
```

This is idempotent — replay-safe — _and_ atomic, in one round trip.

## DynamoDB — Conditional Update and Atomic Counters

If your stack does not run Redis, the DynamoDB equivalent is `UpdateItem` with `ADD` and a condition expression. DynamoDB's [`UpdateItem` reference][dynamodb-update] documents `ADD` as an atomic counter primitive: the increment is applied server-side without a read-modify-write loop.

```python
import boto3
from botocore.exceptions import ClientError

ddb = boto3.client("dynamodb")

def allow(user_id: str, limit: int, window_seconds: int) -> bool:
    now = int(time.time())
    window_id = now // window_seconds
    pk = f"rl#{user_id}#{window_id}"
    expires_at = (window_id + 1) * window_seconds + 60  # for TTL cleanup
    try:
        resp = ddb.update_item(
            TableName="rate_limits",
            Key={"pk": {"S": pk}},
            UpdateExpression="ADD #c :one SET #ttl = if_not_exists(#ttl, :exp)",
            ConditionExpression="attribute_not_exists(#c) OR #c < :limit",
            ExpressionAttributeNames={"#c": "count", "#ttl": "expires_at"},
            ExpressionAttributeValues={
                ":one": {"N": "1"},
                ":exp": {"N": str(expires_at)},
                ":limit": {"N": str(limit)},
            },
            ReturnValues="UPDATED_NEW",
        )
        return True
    except ClientError as e:
        if e.response["Error"]["Code"] == "ConditionalCheckFailedException":
            return False  # over the limit
        raise
```

A few notes:

- **`ADD` is the atomic counter operation.** Two concurrent `ADD :one` calls on the same item correctly produce `+2`, not the `+1` you would get from a non-atomic read-modify-write. This is the core primitive behind DynamoDB-based rate limiters.
- **`ConditionExpression`** lets you make the increment conditional — the equivalent of doing the limit check in the same atomic step. If the condition fails, no write happens.
- **TTL is set with `if_not_exists`** so we attach it on first write, never re-extending — analogous to the Lua `if count == 1 then EXPIRE` pattern.
- **DynamoDB TTL is best-effort within 48 hours**, not exact. For rate-limit cleanup that is fine; for security-critical TTLs that is not. If you need exactness, key your buckets by `window_id` (as above) so old keys are simply ignored, and let TTL clean them up lazily.

DynamoDB is typically more expensive per op than Redis but trades that for managed multi-AZ durability and no operational footprint, which can win for low-rate per-user limits where the QPS does not justify a Redis cluster.

## SQL-Based Rate Limiters — SELECT FOR UPDATE and Advisory Locks

Sometimes the right answer is: do not introduce a new system. If you already have PostgreSQL and the rate is low enough — think "5 signups per email per day" or "10 password resets per hour" — a SQL-based rate limiter is fine, debuggable, and one fewer thing to operate.

The two main techniques:

**Row-level locking with `SELECT ... FOR UPDATE`:**

```sql
BEGIN;

INSERT INTO rate_limits (key, count, window_start)
VALUES ('signup:email@example.com', 0, date_trunc('hour', now()))
ON CONFLICT (key) DO NOTHING;

SELECT count, window_start
FROM rate_limits
WHERE key = 'signup:email@example.com'
FOR UPDATE;

-- application logic: if window_start < now() - interval '1 hour', reset count to 0
-- if count >= limit, ROLLBACK and return rate-limited

UPDATE rate_limits
SET count = count + 1,
    window_start = CASE WHEN window_start < now() - interval '1 hour'
                        THEN date_trunc('hour', now())
                        ELSE window_start END
WHERE key = 'signup:email@example.com';

COMMIT;
```

`SELECT FOR UPDATE` takes a row-level lock that is held until the transaction commits or rolls back. Concurrent transactions trying to update the same key block until the first finishes. This gives you correct serialization, but it is **not free**: every check holds an open transaction, and contention serializes throughput on the hot key. At 100 RPS for a single key this is fine; at 10,000 RPS it falls over.

**PostgreSQL advisory locks** are a lighter-weight alternative when you do not need to update a row, just serialize a code path:

```sql
SELECT pg_advisory_xact_lock(hashtext('signup:email@example.com'));
-- now serialized across the cluster for this key, for the duration of this transaction
```

Advisory locks are application-defined locks identified by an integer (or two). They are not tied to a row, so they do not block readers of the underlying table. They are particularly useful when the rate-limiter "row" lives in cache and you only want to serialize the cache-and-decide step in SQL.

These techniques scale roughly to **the throughput of one Postgres primary on the hot key** — typically a few thousand RPS — and that is fine for things like signup-per-email or per-IP password-reset throttles where the rate is naturally low. For Internet-edge rate limiting (millions of RPS) you want Redis, DynamoDB, or a purpose-built system. The PostgreSQL [advisory locks reference][postgres-advisory] is the canonical doc for the API.

## Crash Recovery Semantics

What happens when the rate-limiter pod dies mid-decrement? It depends on _where_ in the flow the death occurs:

1. **Death before the storage round-trip.** The counter never moved; the request never reached the limiter. The client (gateway, retry layer) retries on a different pod and the counter increments exactly once. Correct.
2. **Death after the storage round-trip but before the response is written.** The counter moved; the client did not see a response. The client retries on a different pod, which increments the counter a second time. The user is over-counted by one — they had a "free" failed attempt earlier. **For a rate limiter this is fine** — it errs in the safe direction (limits more, not less). It is also why the dedup-by-request-ID pattern from earlier matters when you want strict idempotence.
3. **Death after writing the response but before any post-response cleanup.** The counter moved, the client got the answer, and there is nothing to clean up because rate-limit decisions are stateless beyond the counter itself. No correctness issue.

The unifying principle: **the counter must be durable; the response can be fire-and-forget.** As long as the storage layer (Redis, DynamoDB, Postgres) treats the increment as committed before the limiter pod returns, a crash anywhere after that is safe. The reverse is _not_ true: a pod that holds the increment in memory and forwards the request to the backend before durably writing it is incorrect — a crash here loses the increment and the user gets a free request.

This has practical consequences for your Redis configuration:

- **AOF with `everysec` is the usual choice.** It loses at most one second of writes on a hard crash, which is acceptable for a rate limiter (under-counts a sub-second worth of requests, no correctness disaster).
- **AOF with `always` (fsync per write)** is correct but kills throughput. Reserve for billing-grade limits where a second of lost writes is unacceptable.
- **RDB-only** is risky — you can lose minutes of writes. Combine with replication-with-failover so a primary crash falls over to a replica that has the data.
- **Replicate, do not wait.** As section 7.2 of the parent doc notes, do not block requests on replication acknowledgement. Async replication is the right default; you can tolerate the rare lost increment on failover.

For DynamoDB the analogous knobs are: it is multi-AZ-durable by default, no configuration needed; the only consideration is whether to use Global Tables (cross-region) for active-active limits, which trades stronger durability for the same approximate-global-counter caveats discussed in section 7.2.

For SQL-based limiters, the durability story is whatever your normal Postgres setup gives you (synchronous_commit, replication slots, base backup cadence). Rate-limit data is normally not worth more rigor than the rest of your transactional data — if your Postgres setup is good enough for orders, it is good enough for rate limits.

## Anti-Patterns

- **Using `MULTI/EXEC` thinking it is atomic across decision logic.** It batches commands but cannot conditionally branch. The classic `INCR + (if first) EXPIRE` is exactly the case where `MULTI/EXEC` is wrong and Lua is right.
- **Wrapping `EXPIRE` unconditionally in a pipeline.** "Always extend the TTL" turns a fixed-window counter into an infinite-window one. The TTL must be set _once_, on first write.
- **Using `EVAL` instead of `EVALSHA` in hot paths.** You are wasting kilobytes per request shipping the script body. Use the `Script` helper in your Redis client; it does the EVALSHA-with-EVAL-fallback for you.
- **Long-running Lua scripts.** Redis is single-threaded. A 100 ms script blocks every other client for 100 ms. Keep scripts short, no loops over large data sets.
- **Distributed locks for the per-request hot path.** If you have an atomic primitive (Lua, INCR, conditional update), you do not need a lock. Locks under contention serialise everything.
- **Redlock as the answer to "we need a really good lock".** If you actually need correctness-level mutual exclusion, you need a real consensus system (etcd, Zookeeper, Consul) plus fencing tokens, not more Redis nodes.
- **Client-supplied request-IDs as the dedup key.** A malicious client picks the same ID forever and gets infinite free requests. Stamp IDs server-side, ideally monotonic.
- **Synchronous fsync (`appendfsync always`) on a high-RPS Redis rate limiter.** It is correct but slow. `everysec` is the right default for rate-limit workloads.
- **`SELECT FOR UPDATE` on a hot rate-limit row at high RPS.** It serialises every request through one Postgres process. Fine at 100 RPS; falls over at 10,000.
- **Reasoning about the limiter as if it were one component.** It is at least three: the in-memory pre-aggregator, the storage layer, and the cleanup/TTL layer. Each has its own atomicity story.

## Related

- [Distributed Transactions — 2PC, 3PC, Sagas, and the Outbox Pattern](../../../data-consistency/distributed-transactions.md) — the broader story of cross-system atomicity, which the rate-limiter problem is a constrained single-store subset of
- [Consensus, Raft, and Paxos](../../../data-consistency/consensus-raft-and-paxos.md) — the consensus background you need before reasoning about Redlock alternatives like etcd or Zookeeper-based locks
- [Design a Rate Limiter (parent case study)](../design-rate-limiter.md) — the full case study; this deep dive expands section 7.3

## References

- [Redis — `EVAL` command reference](https://redis.io/commands/eval/) — official documentation of `EVAL`/`EVALSHA`, atomicity guarantees, and the script cache lifecycle
- [Redis — Programmability and Functions](https://redis.io/docs/latest/develop/interact/programmability/functions-intro/) — Redis Functions intro and the recommended path for new Redis 7+ deployments
- [Redis — `SET` command reference](https://redis.io/commands/set/) — the canonical reference for `NX`, `XX`, `EX`, `PX`, and the atomic-with-TTL semantics
- [Martin Kleppmann — _How to do distributed locking_](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — the critique of Redlock and the case for fencing tokens
- [Antirez (Salvatore Sanfilippo) — _Is Redlock safe?_](http://antirez.com/news/101) — Antirez's response to the Kleppmann critique
- [AWS — DynamoDB `UpdateItem` API reference](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_UpdateItem.html) — `ADD` for atomic counters and `ConditionExpression` for atomic conditional updates
- [PostgreSQL — Advisory Locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS) — `pg_advisory_lock`, `pg_advisory_xact_lock`, and the integer-keyed application lock model
- [PostgreSQL — Explicit Locking and `SELECT FOR UPDATE`](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE) — row-level locking semantics for SQL-based serialisation

[redis-eval]: https://redis.io/commands/eval/
[redis-functions]: https://redis.io/docs/latest/develop/interact/programmability/functions-intro/
[redlock]: https://redis.io/docs/latest/develop/use/patterns/distributed-locks/
[kleppmann-locking]: https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
[antirez-redlock]: http://antirez.com/news/101
[dynamodb-update]: https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_UpdateItem.html
[postgres-advisory]: https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS
