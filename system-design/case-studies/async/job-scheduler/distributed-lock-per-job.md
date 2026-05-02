---
title: "Distributed Lock per Job — Lease, Fencing, and Avoiding Concurrent Execution"
date: 2026-05-01
updated: 2026-05-01
tags: [system-design, deep-dive, scheduler, locking, fencing]
---

# Distributed Lock per Job — Lease, Fencing, and Avoiding Concurrent Execution

**Date:** 2026-05-01 | **Updated:** 2026-05-01
**Tags:** `system-design` `deep-dive` `scheduler` `locking` `fencing`

> **Parent case study:** [Design a Job Scheduler](../design-job-scheduler.md). This deep-dive expands "§7.1 Distributed Lock per Job".

The parent case study sketches three properties the per-job lock must satisfy — mutual exclusion, lease-based expiry, and fencing — and presents a Postgres `SKIP LOCKED` claim plus an optional Redis/etcd fenced lock for handler execution. This doc gets concrete: what the duplicate-execution problem actually looks like in production, why locks alone are insufficient under stop-the-world pauses, what fencing tokens are and how downstream systems check them, and how to combine the lock primitives with idempotent handlers so the system survives every realistic failure mode.

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [The Duplicate-Execution Problem](#the-duplicate-execution-problem)
  - [Why Two Workers End Up With the Same Trigger](#why-two-workers-end-up-with-the-same-trigger)
  - [What "Duplicate Execution" Costs](#what-duplicate-execution-costs)
- [Lock Key Design — Per-Fire, Not Per-Job](#lock-key-design--per-fire-not-per-job)
  - [Why `(job_id, scheduled_run_at)` Beats `(job_id)`](#why-job_id-scheduled_run_at-beats-job_id)
  - [Encoding the Key](#encoding-the-key)
- [Leases and TTLs](#leases-and-ttls)
  - [What a Lease Is](#what-a-lease-is)
  - [Sizing the TTL](#sizing-the-ttl)
  - [Heartbeat Refresh Cadence](#heartbeat-refresh-cadence)
- [Fencing Tokens — Kleppmann's Argument](#fencing-tokens--kleppmanns-argument)
  - [The GC-Pause Failure Mode](#the-gc-pause-failure-mode)
  - [How Fencing Defeats It](#how-fencing-defeats-it)
  - [Where the Token Has to Be Checked](#where-the-token-has-to-be-checked)
  - [Why Redlock Alone Is Insufficient](#why-redlock-alone-is-insufficient)
- [Backend Choices](#backend-choices)
  - [ZooKeeper — Sequential Ephemeral Nodes](#zookeeper--sequential-ephemeral-nodes)
  - [etcd — Lease + Compare-And-Swap](#etcd--lease--compare-and-swap)
  - [Postgres — `SELECT … FOR UPDATE SKIP LOCKED`](#postgres--select--for-update-skip-locked)
  - [Redis — `SET NX EX` with Lua Refresh](#redis--set-nx-ex-with-lua-refresh)
  - [Consul Sessions](#consul-sessions)
- [Worker Self-Fencing on Heartbeat Failure](#worker-self-fencing-on-heartbeat-failure)
- [Idempotency Is Mandatory](#idempotency-is-mandatory)
- [Worked Example — 5 Workers, One Crash Mid-Run](#worked-example--5-workers-one-crash-mid-run)
- [Code — Postgres SKIP LOCKED Claim](#code--postgres-skip-locked-claim)
- [Code — Redis SET NX EX with Lua Heartbeat](#code--redis-set-nx-ex-with-lua-heartbeat)
- [Code — Fencing Token Verifier on Side-Effect Endpoint](#code--fencing-token-verifier-on-side-effect-endpoint)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

A scheduled job that fires once should run **at most once at a time**, even when the scheduler tier is replicated for HA, even when workers crash mid-run, and even when a paused worker wakes up after another worker has taken over. The naive failure mode: scheduler shard A and scheduler shard B both poll the schedule table, both see the trigger is due, both dispatch — two workers run, two charges hit Stripe, one customer is double-billed. The defense is a per-fire lock, keyed by `(job_id, scheduled_run_at)` rather than `job_id` alone, held under a **lease** (TTL) that auto-expires when the holder crashes, and protected by a **fencing token** so downstream side effects reject writes from a holder whose lease silently expired during a stop-the-world pause. Lease TTL must exceed the job's worst-case runtime by 2–3×; the worker refreshes the lease on a heartbeat at roughly one-third the TTL so a single missed heartbeat does not lose the lock. Backend choice is pragmatic: Postgres `SELECT … FOR UPDATE SKIP LOCKED` for dispatch claim (same DB as the schedule state, no extra service), Redis `SET key value NX EX ttl` plus a Lua script for heartbeat refresh when sub-millisecond acquire matters, ZooKeeper or etcd when linearizable semantics and native lease/watch primitives are worth the operational overhead. Whichever backend, the lock itself is **best-effort** — Kleppmann's "How to Do Distributed Locking" proves that under realistic GC pauses and network delays no lease-based lock alone can guarantee mutual exclusion. The only correct defense is fencing: a monotonic token issued at acquire time, stamped onto every side-effect call, and checked at the downstream resource. Idempotent handlers are mandatory; they turn "exactly-once delivery" (impossible) into "exactly-once effect" (achievable). The anti-patterns — lock without TTL, lock keyed by `job_id` only, heartbeat slower than the lease, downstream side effects without token checks, worker that keeps running after heartbeat failure — are individually how production schedulers turn one fired trigger into double-billed customers.

## Overview

```mermaid
flowchart TD
    SCHED[Scheduler tier<br/>fires at 03:00 UTC] --> CLAIM[N workers race<br/>to claim lock]
    CLAIM --> ARB[Lock arbiter<br/>Postgres / Redis / etcd / ZK]
    ARB -->|win + token=42| W1[Worker A holds lease<br/>heartbeat every TTL/3]
    ARB -->|lose| W2[Workers B–E<br/>back off]
    W1 --> SE[Side-effect call<br/>POST /charge { token: 42 }]
    SE --> DOWN[Downstream resource<br/>checks token > last_seen]
    DOWN -->|42 > 41 OK| OK[Accept, persist last_seen=42]
    DOWN -->|stale token| REJ[Reject as superseded]
    W1 -.->|crash or pause| EXP[Lease TTL expires]
    EXP --> RECLAIM[Worker B claims<br/>token=43, takes over]
```

The diagram shows the four moving parts: the scheduler tier that fires the trigger; the workers that race to claim the lease; the arbiter that issues the lease and a monotonic token; and the downstream resource that fences out stale writes. A worker holds the lease only as long as it heartbeats; if it crashes or pauses, the lease expires and another worker claims it with a strictly greater token. The downstream resource is the **only** place where mutual exclusion is actually enforced — the lock is just a hint.

## The Duplicate-Execution Problem

### Why Two Workers End Up With the Same Trigger

Production schedulers are replicated for HA. The scheduler tier itself runs at least two instances (active-passive via leader election, or active-active sharded by tenant — see `./leader-election-for-scheduler.md`). When the leader fires, it writes a row to a `runs` table or pushes a message to a queue, which a worker pool dequeues. Workers themselves are also replicated.

The race surfaces in several places:

- **Scheduler split-brain.** A network partition isolates the leader; the standby promotes itself; both fire the same trigger at `03:00:00`. Two `runs` rows; two workers pick them up.
- **Retry after timeout.** A worker takes 90 s on a job; the dispatcher's visibility timeout was 60 s; the dispatcher re-enqueues the job; another worker picks up the duplicate.
- **At-least-once queues.** SQS, Kafka, NATS, RabbitMQ — none guarantee exactly-once delivery. A consumer ack lost in transit means redelivery.
- **Worker GC pause.** Worker A grabs the job, then a 45 s STW GC starts. Visibility timeout expires; the job reappears; worker B picks it up; worker A wakes up still believing it owns the job.
- **Manual re-trigger.** An operator clicks "rerun" while the original is still in flight.

Each of these is normal — the system has to handle them. The lock's purpose is to **funnel them through a single arbitration point** so only one worker actually runs the side effects, even when the queue or scheduler hands the job to multiple consumers.

### What "Duplicate Execution" Costs

The cost depends on what the handler does. A read-only report regenerator: nothing, second run produces the same output. A handler that writes to a downstream API:

| Handler does | Without lock + idempotency |
|---|---|
| Charge a credit card | Customer billed twice |
| Send a notification | User gets two emails |
| Increment a counter | Counter double-counts |
| Allocate inventory | Two units reserved for one order |
| Insert a row | Duplicate row, FK conflicts later |
| Call a non-idempotent partner API | Partner-side double-effect; manual reconciliation |

The job scheduler is the place where these effects originate, so it is the place where the protection has to live. The downstream API "trusts" its caller; the caller has to be the one that doesn't fire twice.

## Lock Key Design — Per-Fire, Not Per-Job

### Why `(job_id, scheduled_run_at)` Beats `(job_id)`

A naive design locks on `job_id`. That is correct for "no two runs of this job overlap in wall-clock time" but it is wrong for "every scheduled fire of this job runs exactly once." Consider a job scheduled every 5 minutes that occasionally takes 12 minutes to run:

```text
03:00 fire 1 starts; lock("job:42") acquired
03:05 fire 2 due, but lock("job:42") still held → SKIPPED entirely
03:10 fire 3 due, lock still held → SKIPPED
03:12 fire 1 finishes; lock released
03:15 fire 4 starts (skipping fires 2 and 3 silently)
```

A `job_id`-only lock turns every overlap into a silent skip. The user expected fires at 03:00, 03:05, 03:10, 03:15; they got fires at 03:00 and 03:15. The runs table doesn't even record fires 2 and 3 unless you separately materialize them, which you should — see `./missed-fire-policies.md` for how to handle the overlap explicitly (skip, queue, coalesce).

The fix: key the lock on the **specific fire**, not the job. `(job_id, scheduled_run_at)` is unique per trigger:

```text
03:00 lock("job:42|2026-04-30T03:00:00Z") → acquired by worker A
03:05 lock("job:42|2026-04-30T03:05:00Z") → acquired by worker B (independent key)
```

Workers A and B run concurrently because they are distinct fires, even though they are the same job. Whether you actually want them to run concurrently is a separate policy question — but the lock is no longer the thing forcing serialization. The missed-fire policy is.

### Encoding the Key

The key has to be deterministic across nodes. `scheduled_run_at` should be the **planned** fire instant, not the actual claim time, and should be encoded in UTC ISO-8601 with microsecond precision (or a UNIX timestamp with the same precision):

```text
lock:job:42:2026-04-30T03:00:00.000000Z
```

Avoid:

- Local-time encodings (DST ambiguity, see `./time-zone-correctness.md` if it exists, or §7.4 of the parent doc).
- Floating-point timestamps (precision loss, `1745983200.0` may not equal the value rehydrated by another worker).
- The actual claim time `claimed_at` (different per worker, defeats the lock).

If the schedule materializes fires into a `runs` table at fire time (Quartz model), the row's primary key serves as the lock key directly — `lock:run:<run_uuid>` is even better because UUIDs are immune to clock-skew or rounding drift across nodes.

## Leases and TTLs

### What a Lease Is

A lease is a lock with an expiration. The semantics: "the holder owns this lock for at most TTL seconds; after TTL, the arbiter is free to grant it to anyone." Without a lease, a crashed holder pins the lock forever — operators have to manually intervene to release it. With a lease, recovery is automatic.

The lease's job is **liveness, not safety**. It guarantees the system makes progress. Safety (no two holders at once) is a property the lock service tries to provide and mostly does, but cannot guarantee absolutely under realistic system pauses — see fencing below.

### Sizing the TTL

The TTL must satisfy:

```text
TTL > p99_runtime + p99_GC_pause + safety_margin
```

For a typical handler:

- p99 runtime: observed from history, e.g. 60 s.
- p99 GC pause: 1–5 s for a JVM with G1, longer for old generations or pathological allocation.
- Safety margin: 2× the sum, to absorb spikes.

So a 60 s p99 handler with a 3 s GC p99 wants a TTL of **~120 s**. Empirically, schedulers default to anywhere from 30 s (short batch jobs) to 30 minutes (long ETL). The right number is "long enough that you almost never lose the lock to the wrong cause, short enough that recovery from a crash is timely."

The cost of an oversized TTL: when a worker actually crashes, the next worker has to wait the full TTL before it can take over. If TTL is 30 min and a worker crashes 10 s in, the job blocks for 29 min 50 s. Counter-balance with a dedicated **liveness check** (see "Worker Self-Fencing" below) that releases the lock voluntarily when the worker detects its own failure.

The cost of an undersized TTL: the holder's heartbeat occasionally misses (network blip, scheduler delay, GC), the lease expires, a second worker claims — duplicate execution risk that the fencing token then has to absorb.

### Heartbeat Refresh Cadence

The holder extends the lease periodically by re-writing it. Cadence matters:

```text
heartbeat_period = TTL / 3
```

The factor of three is the standard rule. It tolerates **two consecutive missed heartbeats** before the lease expires. With heartbeat at TTL/2, a single missed beat plus normal jitter is enough to lose the lock. With heartbeat at TTL/4 or finer, the load on the lock service rises without much added safety — diminishing returns.

For a 120 s TTL, the worker heartbeats every ~40 s. Each heartbeat is a single Redis `SET … XX EX` (set only if it exists, refresh TTL) or an etcd `LeaseKeepAlive`, microseconds of work. The heartbeat thread runs **separately** from the work thread so a busy worker still beats; this is a non-trivial implementation detail (see worked example below for what happens when it isn't separate).

## Fencing Tokens — Kleppmann's Argument

### The GC-Pause Failure Mode

Martin Kleppmann's [2016 essay](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) describes the canonical failure:

```text
00:00 Worker A: lock acquired, lease 30 s, fencing token=33
00:01 Worker A: starts JVM full GC, pauses
00:31 Worker A: still paused; lease expires; lock released
00:31 Worker B: acquires lock, fencing token=34, starts work
00:35 Worker B: writes result with token=34
00:50 Worker A: GC finishes, wakes up, doesn't yet know lease expired
00:50 Worker A: writes result with token=33 → OVERWRITES Worker B's data
```

The lock service did its job — it issued the lease, expired it on schedule, granted it to B. The TTL was correct. But the lock alone cannot prevent A's write from landing after B's. A is unaware that time has passed in the rest of the system; from its perspective, less than 50 ms of CPU time passed between acquiring the lock and writing.

### How Fencing Defeats It

The fix: every acquire returns a **monotonically increasing token**. The downstream resource (the database, the queue, the side-effect API) records the highest token it has accepted. Writes with a lower token are rejected.

```text
00:00 Worker A: lock + token=33
00:31 Worker B: lock + token=34  (because monotonic, must be > 33)
00:35 Worker B: write(value, token=34); resource records last_seen_token=34
00:50 Worker A: write(value, token=33); resource sees 33 < 34 → REJECT
```

The token is enforced at the **resource**, not at the lock service. The lock service issues it; the resource validates it. This is the essential split: locks coordinate; resources enforce.

Implementations of monotonic tokens:

- **etcd revision number.** Every write to etcd increments the cluster-wide revision; reading the lease's creation revision gives a monotonic ID.
- **ZooKeeper sequential znode.** `CreateMode.EPHEMERAL_SEQUENTIAL` returns a name with a monotonic suffix (`/lock-0000000033`). The suffix is the token.
- **Postgres `nextval('seq')`.** A dedicated sequence per lock namespace. Works as long as the lock acquire happens in the same transaction as the `nextval`.
- **Redis `INCR fence:<key>`.** After a successful `SET NX EX`, atomically `INCR` a parallel counter. Not strictly atomic with the SET, but for cache-stampede-style use cases the slack is acceptable.

### Where the Token Has to Be Checked

Every state-mutating operation downstream of the lock must include the token, and the downstream must verify:

```sql
-- The resource is a Postgres row.
UPDATE jobs
SET status = 'completed', result = $result, last_token = $token
WHERE id = $job_id
  AND ($token > last_token OR last_token IS NULL);
-- 0 rows updated means our token was stale; abort.
```

Or against an external API:

```http
POST /v1/charges
X-Fencing-Token: 42
{ "amount": 1000, "currency": "USD", "idempotency_key": "run-abc-123" }
```

The API checks both the idempotency key (for retry safety) and the fencing token (for split-brain safety). An idempotency key alone allows a stale write to overwrite a fresh one if the keys differ; a token alone allows duplicate effects from genuine retries. Both together give exactly-once effect.

### Why Redlock Alone Is Insufficient

Antirez's [Redlock](https://redis.io/docs/manual/patterns/distributed-locks/) algorithm extends single-instance Redis locks to a quorum of N independent Redis instances, surviving the failure of up to (N−1)/2 of them. Kleppmann's critique: Redlock's **safety** depends on bounded clock drift and bounded process pauses, neither of which holds in real systems. A long GC pause can cause the same overwrite scenario as a single-Redis lock; quorum doesn't help because the issue is on the client side.

Antirez's [response](http://antirez.com/news/101) accepts the critique on theoretical grounds but argues that for many practical workloads the failure modes are bounded enough. For a job scheduler the answer is the same as for cache stampedes: **don't rely on Redlock for correctness**. Use Redis (single instance with replication, or Redlock if you want extra availability) for the lock, and use fencing tokens at the downstream resource for correctness. The lock is opportunistic; the token is the actual guarantee.

If you find yourself thinking "but the side effect is non-idempotent and the resource doesn't support a token check" — fix the resource. Add a token column. Wrap the API in a proxy that enforces it. The alternative is hoping no GC ever takes longer than the lease, which is hope-as-strategy.

## Backend Choices

### ZooKeeper — Sequential Ephemeral Nodes

The classical recipe (per [ZooKeeper docs](https://zookeeper.apache.org/doc/current/recipes.html#sc_recipes_Locks)):

1. Each contender creates an `EPHEMERAL_SEQUENTIAL` znode under `/locks/<job_key>/`.
2. ZooKeeper assigns a monotonic suffix: `/locks/job:42:fire-1/lock-0000000033`.
3. Each contender lists children; the lowest suffix is the holder.
4. Non-holders watch the next-lower znode; when it disappears, they re-check.
5. The ephemeral flag means session expiry (worker crash, network partition long enough to lose the session) auto-deletes the znode — lease semantics for free.

The suffix is the fencing token. The session timeout is the lease TTL. The watch mechanism is the wait-for-grant primitive.

**Strengths:** Linearizable, battle-tested (Hadoop, Kafka), native lease semantics via session, native fencing via sequential nodes.

**Weaknesses:** Operationally heavy (an odd-numbered ensemble, careful tuning of `tickTime` and `syncLimit`), JVM-based, lower throughput than Redis (~10k ops/s on a 3-node ensemble).

Use when you already run ZooKeeper for other coordination (Kafka brokers, HBase region assignment) and the marginal cost of one more recipe is zero.

### etcd — Lease + Compare-And-Swap

[etcd's concurrency package](https://etcd.io/docs/v3.5/dev-guide/api_concurrency_reference_v3/) provides higher-level lock and election primitives:

```go
session, _ := concurrency.NewSession(client, concurrency.WithTTL(120))
mu := concurrency.NewMutex(session, "/locks/job:42:fire-1")
if err := mu.Lock(ctx); err != nil { ... }
defer mu.Unlock(ctx)

// Fencing token = lease ID + revision; both monotonic.
token := mu.Header().Revision
```

The session encapsulates the lease; `KeepAlive` runs in a background goroutine. `Mutex.Lock` uses sequential keys under the hood (same pattern as ZooKeeper) and returns the etcd revision as a fencing token.

**Strengths:** Built for Kubernetes-era infra (it's the K8s control-plane store), native lease, Raft-based linearizability, gRPC and excellent client libraries.

**Weaknesses:** Same operational care as ZooKeeper (odd-sized cluster, disk fsync sensitivity), and lower lock throughput than Redis under high contention.

Use when you're already on Kubernetes and etcd is part of the substrate.

### Postgres — `SELECT … FOR UPDATE SKIP LOCKED`

The pragmatic option, especially when the schedule state already lives in Postgres:

```sql
WITH claimed AS (
  SELECT id FROM runs
  WHERE state = 'PENDING'
    AND scheduled_run_at <= now()
  ORDER BY scheduled_run_at, priority DESC
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
UPDATE runs r
SET state = 'RUNNING',
    locked_by = $worker_id,
    locked_until = now() + interval '120 seconds',
    fence_token = nextval('runs_fence_seq'),
    version = version + 1
FROM claimed
WHERE r.id = claimed.id
RETURNING r.*;
```

`FOR UPDATE SKIP LOCKED` is the magic. It's a Postgres extension (also in MySQL 8+ and Oracle): rows that another transaction has locked are silently skipped instead of blocking. Multiple workers can poll the same query simultaneously; each gets a different unlocked row. See [Postgres docs](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE).

**Strengths:** No extra infra; transactional with the rest of the schedule state; the fence token is a sequence value, monotonic by construction.

**Weaknesses:** The lock is held by an **active database session**. Long-running jobs hold a connection. For many jobs you transition from "row-level lock" to "advisory lock" or to a Redis lock so the connection can be released as soon as the row is claimed.

The standard pattern: use `SKIP LOCKED` for the **dispatch claim** (acquire fast, release the row lock immediately by committing the `UPDATE`), then use Redis or etcd for the **handler execution lease** that lives the duration of the job. The `runs` row's `locked_until` and `fence_token` columns are the durable record; the Redis/etcd lock is the in-flight coordination.

### Redis — `SET NX EX` with Lua Refresh

Redis's `SET key value NX EX ttl` is the atomic acquire (see [Redis docs](https://redis.io/commands/set/)). The historical broken pattern is `SETNX` followed by a separate `EXPIRE`; if the client crashes between the two, the lock is held forever. Always use the combined form.

```text
SET lock:run:abc-123 worker-uuid-xyz NX EX 120
```

Returns `OK` if acquired, `nil` otherwise. The `value` is the owner ID — used at unlock to verify "I'm releasing my own lock, not someone else's." A naive `DEL lock:run:abc-123` is unsafe because it might delete a lock that has since been re-acquired by another worker. The atomic check-and-delete is a Lua script:

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

Heartbeat refresh — re-extend TTL only if I still own the lock — is also a Lua script (see code below).

**Strengths:** Sub-millisecond acquire, trivial to scale (sharded Redis, Redis Cluster), most polyglot ecosystem support of any lock service.

**Weaknesses:** Single-instance Redis can lose locks on failover; Redlock has correctness debates; the fencing token (via `INCR`) is not strictly atomic with the SET. Mitigated by treating the lock as opportunistic and the token as the real correctness primitive.

### Consul Sessions

[Hashicorp Consul](https://developer.hashicorp.com/consul/docs/dynamic-app-config/sessions) provides session-based locking: a session has a TTL, an associated set of held keys, and a lock-delay (cooldown after session invalidation before another holder can acquire). Acquiring a lock uses `PUT /kv/key?acquire=<session-id>`; releasing uses `release=<session-id>`. The session's `ModifyIndex` is the fencing token.

Strengths and weaknesses are similar to etcd: linearizable, lease-native, but operationally heavier than Redis. Use when you already run Consul for service discovery.

## Worker Self-Fencing on Heartbeat Failure

A subtle requirement: the worker must stop work if its own heartbeat fails to refresh the lease. Otherwise the worker keeps running, the lease expires, another worker takes over, and you have two concurrent runs.

The pattern:

```pseudo
on heartbeat_thread tick (every TTL/3):
  ok = lock_service.refresh(key, owner_id, ttl)
  if not ok:
    # We've lost the lock. Either we crashed and didn't notice,
    # or the lock service rebooted and our state is gone.
    log.error("lost lock; aborting work")
    work_thread.signal_abort()
    exit
```

The work thread checks an `aborted` flag at safe points (between transactions, between RPC calls, at the top of every loop iteration) and exits voluntarily if set. For long-running side effects (a multi-hour ETL), check the flag inside the inner loop.

The opposite (worker keeps running after heartbeat failure) is the most insidious form of split-brain because everything looks fine from the worker's perspective — its internal state says "I have the lock; I am working." It's only by checking with the arbiter that the worker discovers it has been kicked out.

For finer-grained protection, **wrap every side-effect call** with a lease-validity check. If the side effect supports a fencing token, the resource will reject the stale write anyway; the self-check is a faster local abort.

## Idempotency Is Mandatory

The lock is best-effort. The fencing token is enforcement. **The handler must still be idempotent.** Why? Three reasons:

1. **Token-aware downstream is rare.** Most external APIs (Stripe, SendGrid, Twilio, internal microservices that pre-date your lock design) don't accept a fencing token. Idempotency keys are far more common — Stripe's `Idempotency-Key` header is the model.

2. **The token only protects state-mutating writes.** Non-resource side effects — sending a webhook, writing to an immutable log — can fire twice even with the token. The recipient has to dedupe on its end.

3. **Retries also call the handler twice.** A genuinely failed call retried 30 s later is not a fencing-token issue; it's an idempotency issue. The same handler must absorb both.

The minimum idempotent handler:

```pseudo
function handle(run):
  idempotency_key = run.id   # unique per fire
  fencing_token = run.fence_token

  # Check we haven't already done this work.
  prior = state_store.get(idempotency_key)
  if prior is not None:
    return prior   # already done

  # Do the work.
  result = side_effect(
    idempotency_key=idempotency_key,
    fencing_token=fencing_token,
    ...
  )

  # Persist with optimistic CAS on the fencing token.
  state_store.upsert(
    key=idempotency_key,
    value=result,
    where_token_lt=fencing_token,
  )
  return result
```

`run.id` is unique per fire (matches the lock key). `fence_token` is monotonic per fire (issued at lock acquire). The state store rejects writes whose token is not strictly greater than the last persisted token for that key. See `./retry-and-idempotency.md` for the full taxonomy of idempotent handler patterns.

## Worked Example — 5 Workers, One Crash Mid-Run

Setup: a daily ETL job scheduled at 03:00 UTC. The scheduler tier fires; the run row is materialized with `id=run-abc-123`, `fence_token=42`. Five workers in the pool poll for runs.

```text
T+0.000s   All 5 workers see run-abc-123 in PENDING; race to claim.
T+0.001s   Worker A wins SELECT FOR UPDATE SKIP LOCKED; UPDATE sets state=RUNNING, locked_by=A, locked_until=T+120s, fence_token=42.
T+0.002s   Workers B–E see no PENDING rows (or get the next row); back off.
T+0.003s   Worker A starts handler; spawns heartbeat thread refreshing every 40s.
T+40s      Heartbeat 1: SET lock:run-abc-123 A NX EX 120 (XX form; OK if I own it). locked_until → T+160s.
T+80s      Heartbeat 2: OK. locked_until → T+200s.
T+95s      Worker A crashes (OOM kill).
T+120s     Heartbeat 3 doesn't fire.
T+200s     Lease expires (locked_until in the past).
T+201s     Workers B–E poll again; one of them (say B) sees run-abc-123 with state=RUNNING but locked_until < now().
           B issues: UPDATE runs SET state=RUNNING, locked_by=B, locked_until=T+320s, fence_token=43, version=version+1
                     WHERE id=run-abc-123 AND version=$expected_version;
T+202s     B starts the handler. Idempotency check: state_store.get(run-abc-123) returns nothing (A crashed before completing).
T+202s     B does the work; sends side effects with X-Fencing-Token: 43.
T+250s     B completes; persists result with token=43.
```

Now the resurrection scenario: A doesn't crash, just pauses (45 s GC).

```text
T+95s     Worker A pauses (GC).
T+120s    Heartbeat 3 doesn't fire.
T+200s    Lease expires. Worker B claims with fence_token=43.
T+202s    Worker B does the work, side effect with token=43, succeeds.
T+250s    Worker B persists result.
T+260s    Worker A wakes up. Has no idea time has passed. Sends side effect with token=42.
T+260s    Side-effect API: "your token 42 is older than my last_seen 43" → reject.
T+260s    Worker A then tries to persist result with token=42 → state_store rejects.
T+260s    Worker A's heartbeat thread tries to refresh; lock service says "you don't hold this lock" → abort signal.
T+260s    Worker A logs the abort, exits.
```

No duplicate effect. The lease expired correctly; the new claim got a higher token; the side-effect API and the state store both rejected the late writer. Without fencing, A's side effect would have landed.

## Code — Postgres SKIP LOCKED Claim

Pseudocode for the dispatch query, with a fencing token from a sequence and the lease tracked in the same row.

```sql
-- Schema
CREATE TABLE runs (
    id UUID PRIMARY KEY,
    job_id UUID NOT NULL,
    scheduled_run_at TIMESTAMPTZ NOT NULL,
    state TEXT NOT NULL,                 -- PENDING | RUNNING | DONE | FAILED
    locked_by TEXT,                      -- worker UUID
    locked_until TIMESTAMPTZ,            -- lease deadline
    fence_token BIGINT,                  -- monotonic per-fire token
    version INT NOT NULL DEFAULT 0,      -- optimistic CAS
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX runs_dispatch_idx
  ON runs (state, scheduled_run_at)
  WHERE state = 'PENDING';

CREATE SEQUENCE runs_fence_seq;
```

```sql
-- Dispatch claim (run on a single dedicated connection per worker).
WITH expired AS (
    -- Step 1: reclaim leases whose holder died.
    UPDATE runs
    SET state = 'PENDING'
    WHERE state = 'RUNNING'
      AND locked_until < now()
    RETURNING id
), claimed AS (
    -- Step 2: pick one PENDING row this worker doesn't already hold.
    SELECT id, version FROM runs
    WHERE state = 'PENDING'
      AND scheduled_run_at <= now()
    ORDER BY scheduled_run_at, id
    FOR UPDATE SKIP LOCKED
    LIMIT 1
)
UPDATE runs r
SET state = 'RUNNING',
    locked_by = $worker_id,
    locked_until = now() + interval '120 seconds',
    fence_token = nextval('runs_fence_seq'),
    version = r.version + 1
FROM claimed
WHERE r.id = claimed.id
  AND r.version = claimed.version
RETURNING r.id, r.fence_token, r.locked_until;
```

Notes:

- `expired` reclaims dead-worker leases as part of the same dispatch query — no separate sweeper needed for small/medium scale. For high-volume schedulers, run a dedicated reaper process so the dispatch query stays cheap.
- The `WHERE r.version = claimed.version` is optimistic CAS. If two workers race to claim the same `expired`-reclaimed row, only one wins.
- `FOR UPDATE SKIP LOCKED` requires the SELECT to be inside a transaction. The whole CTE runs in a single statement-level transaction.
- The fence token is a `nextval` — guaranteed monotonic by Postgres even under concurrent acquisitions.

To **renew the lease** during work:

```sql
UPDATE runs
SET locked_until = now() + interval '120 seconds'
WHERE id = $run_id
  AND locked_by = $worker_id
  AND state = 'RUNNING'
RETURNING locked_until;
```

If 0 rows are returned, the worker has lost the lease — abort the work.

## Code — Redis SET NX EX with Lua Heartbeat

Redis-based lock with atomic acquire, owner-checked release, atomic heartbeat refresh, and a fencing token from `INCR`.

```pseudo
-- Acquire: SET key owner NX PX ttl_ms ; on success, INCR fence:key
function acquire(redis, key, owner_id, ttl_ms):
    ok = redis.set("lock:" + key, owner_id, NX=true, PX=ttl_ms)
    if not ok:
        return None
    token = redis.incr("fence:" + key)
    return { owner_id, token, key }
```

Heartbeat refresh — extend TTL only if I still own the lock. This is critical: a naive `SET … EX ttl` would re-acquire the lock if it had silently expired and been claimed by someone else, blowing away their ownership.

```lua
-- KEYS[1] = lock key; ARGV[1] = expected owner_id; ARGV[2] = new TTL ms
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("pexpire", KEYS[1], ARGV[2])
else
    return 0
end
```

```pseudo
function refresh(redis, lock, ttl_ms):
    return redis.eval(LUA_REFRESH,
                      keys=["lock:" + lock.key],
                      args=[lock.owner_id, ttl_ms])

function release(redis, lock):
    return redis.eval(LUA_RELEASE,
                      keys=["lock:" + lock.key],
                      args=[lock.owner_id])
```

Where `LUA_RELEASE` is the canonical owner-checked DEL:

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

Worker integration with self-fencing:

```pseudo
function run_with_lock(redis, run, ttl_ms=120000, heartbeat_period_ms=40000):
    lock = acquire(redis, run.lock_key, generate_uuid(), ttl_ms)
    if lock is None:
        return  # someone else got it

    aborted = AtomicBool(false)

    # Heartbeat thread.
    spawn:
        while not aborted:
            sleep(heartbeat_period_ms)
            ok = refresh(redis, lock, ttl_ms)
            if ok == 0:
                log.error("lost lock; aborting", run=run.id)
                aborted.set(true)
                return

    # Work thread.
    try:
        for step in run.steps:
            if aborted.get():
                raise AbortError("lease lost mid-run")
            do_step(step, fence_token=lock.token)
    finally:
        release(redis, lock)
        aborted.set(true)
```

The heartbeat and work run on independent threads/coroutines. The work thread checks `aborted` at safe points. On lease loss, it aborts as soon as it can.

## Code — Fencing Token Verifier on Side-Effect Endpoint

The downstream resource enforces the token. This is the actual correctness boundary.

A Postgres-backed state store with token CAS:

```sql
CREATE TABLE side_effect_log (
    idempotency_key TEXT PRIMARY KEY,
    fence_token BIGINT NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```sql
-- The application calls this. Token check is in the WHERE clause.
INSERT INTO side_effect_log (idempotency_key, fence_token, payload)
VALUES ($key, $token, $payload)
ON CONFLICT (idempotency_key) DO UPDATE
SET fence_token = EXCLUDED.fence_token,
    payload = EXCLUDED.payload,
    updated_at = now()
WHERE side_effect_log.fence_token < EXCLUDED.fence_token
RETURNING fence_token, payload;
```

If the row exists and its `fence_token >= $token`, the `WHERE` clause filters out the update and `RETURNING` returns no rows — the application reads "no rows" as "your write was superseded; abort."

A REST-style resource:

```pseudo
# Server: POST /v1/charges
function create_charge(request):
    idempotency_key = request.header("Idempotency-Key")
    token_str = request.header("X-Fencing-Token")
    if idempotency_key is None or token_str is None:
        return 400, "missing idempotency or fencing header"

    token = int(token_str)

    # Atomic CAS: only proceed if no prior write or token is strictly greater.
    prior = db.exec_one("""
        INSERT INTO charges (idempotency_key, fence_token, status, payload)
        VALUES ($1, $2, 'pending', $3)
        ON CONFLICT (idempotency_key) DO UPDATE
        SET fence_token = EXCLUDED.fence_token,
            updated_at = now()
        WHERE charges.fence_token < EXCLUDED.fence_token
          AND charges.status IN ('pending', 'failed')
        RETURNING id, status, fence_token
    """, idempotency_key, token, request.body)

    if prior is None:
        # Either: someone with a higher token already wrote (split-brain loser),
        # or: the charge already succeeded (genuine retry, idempotent return).
        existing = db.fetch_one("""
            SELECT id, status, fence_token, response
            FROM charges WHERE idempotency_key = $1
        """, idempotency_key)
        if existing.fence_token > token:
            return 409, "superseded by newer fencing token"
        return 200, existing.response  # idempotent replay

    # First time we see this token. Actually do the work.
    result = call_payment_processor(request.body)
    db.exec("""
        UPDATE charges
        SET status = $1, response = $2, updated_at = now()
        WHERE idempotency_key = $3 AND fence_token = $4
    """, result.status, result.body, idempotency_key, token)

    return 200, result.body
```

Two checks together: **idempotency key** prevents duplicate effects on retries; **fencing token** prevents stale writes from a paused-then-resumed worker. Both are enforced atomically in the database. The application's role is to pass them through; the database's role is to refuse to violate either invariant.

For external APIs you don't control (Stripe, SendGrid), there's no fencing-token header. The fallback: persist the side-effect attempt in your own database **with the fencing token** before calling the external API. If your own write succeeds, proceed; if it fails because the token check rejected, abort without calling the external API. The external API only ever sees attempts that survived the fence.

## Anti-Patterns

- **Lock without TTL.** A `SETNX` followed by a separate `EXPIRE` (or, worse, no expire at all). A worker crash between the two leaves the lock held forever; manual intervention required. Always use the atomic combined form: `SET key value NX EX ttl` or `SET key value NX PX ttl_ms`.
- **Lock keyed by `job_id` only, not `(job_id, scheduled_run_at)`.** Long-running runs of the same job silently skip subsequent fires because the lock is still held by the previous fire. Use a per-fire key; let `./missed-fire-policies.md` decide whether overlapping fires run concurrently or get coalesced.
- **Lock without fencing tokens on side-effect calls.** The lease can be lost during a stop-the-world pause; the next holder runs concurrently; both write to the downstream resource. Without a fencing token at the resource, the late writer overwrites the fresh state. Kleppmann's argument is the canonical reference.
- **Worker that keeps running after heartbeat failure.** The heartbeat thread should signal the work thread to abort if a refresh fails. Otherwise the worker has no idea its lease has expired; it keeps writing with a now-stale fencing token. Self-fence on heartbeat failure.
- **Heartbeat cadence equal to or slower than the lease TTL.** A single missed beat (network blip, scheduler tick miss, GC pause) loses the lock. Standard rule: heartbeat at TTL/3 to absorb two missed beats.
- **TTL shorter than p99 runtime + p99 GC pause.** The lease expires while the worker is still legitimately working. Multiple workers end up running. Size with 2–3× margin.
- **TTL hours-long for safety.** When a worker actually crashes, the next worker can't take over until the entire TTL elapses. Combine a generous TTL with a separate liveness signal (worker pings a registry; on missed pings, an external reaper releases the lock).
- **Heartbeat thread shares the work thread's CPU/event loop.** A long synchronous handler step blocks the heartbeat. The lease expires; another worker claims; the original worker doesn't notice because it's still in the synchronous step. Heartbeat must run on an independent scheduler.
- **Treating Redlock as a correctness boundary.** Redlock survives N/2 Redis instance failures, but it cannot prevent a paused client from writing late with a stale lock. Use Redlock for availability of the lock service if you need it; use fencing tokens for correctness regardless.
- **No fencing token on the downstream resource.** "We have a lock; we trust it." You don't. Add a token column. Wrap the API. The lock alone never enforces mutual exclusion past the next GC pause.
- **Idempotency key reused across fires.** Using `job_id` as the idempotency key instead of `run_id` (or `(job_id, scheduled_run_at)`) means the first run "wins" and every subsequent fire is treated as a retry of the first. Idempotency key must be unique per fire.
- **Fence token from a non-monotonic source.** A wall-clock timestamp can go backwards (NTP correction, leap second). A random UUID isn't comparable. Use a database sequence, a ZooKeeper sequential node, or an etcd revision. Redis `INCR` is monotonic within a single instance; on failover with replication lag it can regress — accept this for opportunistic locking, not for correctness.
- **`SELECT FOR UPDATE` without `SKIP LOCKED`.** All workers serialize on the first PENDING row they find, blocking each other. Throughput collapses to one dispatch per row-lock-acquire roundtrip. `SKIP LOCKED` is what makes the multi-consumer queue pattern work at all.
- **No reaper for orphaned `RUNNING` rows.** A row stuck at `state=RUNNING` with `locked_until < now()` is invisible to a naive dispatch query that only looks at `PENDING`. Either the dispatch query reclaims expired leases (CTE pattern shown above), or a separate reaper sweeps them.
- **Lock contention masked as latency.** Workers waiting on the lock service appear as tail-latency spikes in unrelated metrics. Instrument lock acquire success rate, lock acquire latency p99, lease-loss count, fence-token-rejected count. These are direct correctness signals; without them the duplicate-execution failure mode is invisible until a customer complaint.

## Related

- [`./retry-and-idempotency.md`](./retry-and-idempotency.md) — companion deep-dive on idempotency keys, retry budgets, and turning at-least-once delivery into exactly-once effect; the lock provides coordination, idempotency provides the actual at-most-once-effect guarantee.
- [`./leader-election-for-scheduler.md`](./leader-election-for-scheduler.md) — the scheduler tier itself must elect a leader (or shard cleanly) so it doesn't fire the same trigger from two nodes; the per-job lock is the second line of defense after scheduler-tier coordination.
- [`./missed-fire-policies.md`](./missed-fire-policies.md) — what to do when a fire arrives while the previous one is still running: skip, queue, coalesce, or run-concurrent. The per-fire lock key choice depends on which policy you adopt.
- [`../design-job-scheduler.md`](../design-job-scheduler.md) — parent case study; this deep-dive expands §7.1 of that document.
- [`../../distributed-infra/design-distributed-locking.md`](../../distributed-infra/design-distributed-locking.md) — the building-block doc on distributed locking primitives across backends; this deep-dive applies those primitives to the specific job-scheduler use case.
- [`../../../communication/idempotency-and-exactly-once.md`](../../../communication/idempotency-and-exactly-once.md) — the broader treatment of why exactly-once delivery is impossible, why exactly-once effect is achievable, and how idempotency keys and fencing tokens combine to get there.

## References

- Martin Kleppmann, ["How to Do Distributed Locking"](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) (2016) — the canonical critique of lease-based distributed locks; introduces the fencing-token pattern and demonstrates the GC-pause failure mode that no lease alone can prevent.
- Salvatore Sanfilippo (antirez), ["Is Redlock Safe?"](http://antirez.com/news/101) — Redis creator's response to Kleppmann; argues Redlock's failure modes are bounded enough for practical use. Read alongside Kleppmann to understand the actual tradeoff.
- Redis, [SET command (NX, XX, EX, PX, KEEPTTL options)](https://redis.io/commands/set/) — the atomic `SET key value NX EX ttl` is the primitive for Redis-based locks; the `value` argument as owner identity is the basis for safe release.
- Redis, [Distributed Locks with Redis (Redlock)](https://redis.io/docs/manual/patterns/distributed-locks/) — antirez's specification of the multi-instance lock algorithm; documents the explicit safety and liveness assumptions, including clock-drift bounds.
- Apache ZooKeeper, [Recipes — Locks](https://zookeeper.apache.org/doc/current/recipes.html#sc_recipes_Locks) — the sequential-ephemeral-node lock recipe, used by Hadoop, HBase, and Kafka. The sequence number is the fencing token by construction.
- etcd, [Concurrency Reference (election + lock)](https://etcd.io/docs/v3.5/dev-guide/api_concurrency_reference_v3/) — etcd's high-level lock and leader-election primitives built on top of leases and watch-prefix; the etcd revision number serves as a monotonic fencing token.
- PostgreSQL, [SELECT — `FOR UPDATE … SKIP LOCKED`](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE) — the row-locking option that makes Postgres a workable multi-consumer queue; designed explicitly for this dispatch-claim use case.
- HashiCorp Consul, [Sessions](https://developer.hashicorp.com/consul/docs/dynamic-app-config/sessions) — Consul's session-based locking with TTL, lock-delay, and the session's `ModifyIndex` as a monotonic fencing equivalent.
- AWS Builder's Library, ["Avoiding Insurmountable Queue Backlogs"](https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/) — adjacent: how Amazon-scale queue consumers handle visibility timeouts and orphan reclaim, the same pattern as the dispatch lock + lease.
- Temporal, ["At-least-once Activity Execution"](https://docs.temporal.io/concepts/what-is-an-activity-execution) — explicit framing of why workflow engines deliver at-least-once and offload exactly-once-effect responsibility to the activity (handler) implementation; the same model the job scheduler adopts.
