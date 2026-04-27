---
title: "Pastebin Deep Dive — Expiry Handling"
date: 2026-04-27
updated: 2026-04-27
tags: [system-design, case-study, pastebin, deep-dive, ttl, lifecycle]
---

# Pastebin Deep Dive — Expiry Handling (TTL, Lifecycle Policies, and Sweepers)

**Date:** 2026-04-27 | **Updated:** 2026-04-27
**Tags:** `system-design` `case-study` `pastebin` `deep-dive` `ttl` `lifecycle`

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [Expiry Semantics](#expiry-semantics)
- [TTL Representation](#ttl-representation)
- [Reactive Expiry (Lazy)](#reactive-expiry-lazy)
- [Active Sweeper Job](#active-sweeper-job)
- [Indexed Expiry Queue](#indexed-expiry-queue)
- [Time-Partitioned Tables](#time-partitioned-tables)
- [S3 Lifecycle for Content](#s3-lifecycle-for-content)
- [Redis TTL for Hot Cache](#redis-ttl-for-hot-cache)
- [Sweepers at Scale](#sweepers-at-scale)
- [Burn-After-Reading](#burn-after-reading)
- [Per-Tier Expiry Policies](#per-tier-expiry-policies)
- [Compliance Deletes (GDPR)](#compliance-deletes-gdpr)
- [Auditability of Expiry](#auditability-of-expiry)
- [Soft Delete vs Hard Delete](#soft-delete-vs-hard-delete)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

Pastebin's expiry layer looks deceptively simple: each paste has a TTL, and when it elapses the content disappears. In practice it is a four-system distributed contract spanning the metadata DB, the blob store, the cache, and the CDN — each with its own clock, its own deletion semantics, and its own failure modes. Get it wrong and a "self-destructs in 10 minutes" paste lives on for 23 more hours, or a GDPR delete request waits a day for the next sweeper run, or a "burn-after-reading" link gets read three times because CloudFront cached the body.

This deep dive expands the parent case study's [Expiry Handling section](../design-pastebin.md#expiry-handling--ttl-lifecycle-policies-and-sweepers) into the design space behind a correct, scalable expiry pipeline. The mental model: **the metadata `expires_at` is the source of truth, every other layer is eventual-consistency cleanup**, and the cleanup layers exist to reclaim storage cost — not to enforce visibility.

## Overview

Three classes of work happen at expiry time:

1. **Visibility cutoff.** After the expiry instant, reads must not return content. This is enforced on the **read path** by comparing `expires_at` to `now()`. Cheap, always correct, fast.
2. **Storage reclamation.** The bytes (DB rows, S3 objects, cache entries) must eventually be removed so storage cost stays bounded. This is enforced by **sweepers** and **lifecycle policies**, which run asynchronously and can lag.
3. **Compliance erasure.** When a user invokes a right-to-erasure request (GDPR Art. 17, CCPA), deletion must propagate within a defined window — typically 30 days — and must be auditable. This is a stricter version of (2), with hard SLAs.

The single most common bug in Pastebin-class systems is conflating these three. A "fast" lifecycle policy is not a substitute for a metadata read check; a metadata check alone is not a substitute for storage reclamation; neither covers compliance deletes. Each layer has a job; each layer has its own SLA.

```mermaid
flowchart LR
  R[Read request] --> M{expires_at < now()?}
  M -- yes --> G[410 Gone]
  M -- no --> C[Cache lookup]
  C --> S[S3 fetch]

  T[Sweeper / cron] -.scans.-> DB[(pastes table)]
  T -.deletes.-> S3[(S3 bucket)]
  L[S3 lifecycle policy] -.daily scan.-> S3
  X[Cache TTL] -.auto-evict.-> Redis[(Redis)]
```

## Expiry Semantics

Before designing storage, decide what "expired" actually means to the user:

| Semantic | Behavior at `t = expires_at` | Use case |
|---|---|---|
| **Strict instant** | Read at `t + 1 ms` returns `410 Gone` | Self-destructing pastes, secrets, BAR (burn-after-reading) |
| **Eventually consistent (minutes)** | Read may succeed up to ~5 min after `t`; then `410` | Default Pastebin paste; user expectation is "around a day" |
| **Grace period** | Read succeeds until `t + grace`; then `410`; storage purged at `t + grace + sweeper_lag` | Soft cleanup, anti-flapping for clock skew |
| **Best-effort cleanup only** | Read always succeeds while bytes exist; lifecycle reclaims later | NOT recommended — surprises users, fails compliance |

The **strict instant** semantic is the only one that survives a privacy-sensitive review. It costs nothing extra: a single comparison on the read path.

```sql
-- Read-path gate. Run on every GET /pastes/:id
SELECT id, content_ref, content_type
FROM   pastes
WHERE  id = $1
  AND  deleted_at IS NULL
  AND  (expires_at IS NULL OR expires_at > NOW());
-- Zero rows -> 410 Gone (or 404 if you don't want to disclose existence)
```

A subtle question: should an expired paste return `410 Gone` (resource _existed_, now permanently gone) or `404 Not Found` (concealing that it ever existed)? RFC 9110 says `410` is correct when the server _knows_ the resource is gone. For privacy-sensitive pastes — especially burn-after-reading — `404` may be preferable to avoid confirming the paste ever existed to a non-original viewer.

## TTL Representation

Two ways to encode TTL on a row:

```sql
-- Option A: absolute timestamp (RECOMMENDED)
expires_at TIMESTAMPTZ NULL

-- Option B: relative duration + creation time
created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
ttl_seconds INTEGER NULL
-- Effective expiry = created_at + (ttl_seconds || ' seconds')::interval
```

**Always pick A.** Reasons:

- **Indexable.** `WHERE expires_at < NOW()` uses a B-tree index directly. With B you'd index a generated/computed column or filter at runtime.
- **Single source of truth.** Sweepers, lifecycle tags, S3 metadata, and read checks all compare against one column.
- **Re-encodable.** "Extend by 1 day" is `UPDATE pastes SET expires_at = expires_at + INTERVAL '1 day'`. With B you'd recompute or store a separate offset.
- **Time-zone safe.** `TIMESTAMPTZ` stores UTC internally; comparisons are unambiguous regardless of client timezone or DST transitions.

**Never use `TIMESTAMP WITHOUT TIME ZONE`** for expiry. Daylight-saving "spring-forward" creates a non-existent local hour and "fall-back" creates an ambiguous one. A naive timestamp can move by an hour relative to UTC; a sweeper looking 1 hour ahead may delete future-but-not-yet-expired rows or skip already-expired ones.

For **client-supplied TTLs** (`ttl_seconds=600`), compute `expires_at` server-side at write time. Never trust the client to send a timestamp — clock skew between client and server can produce already-expired or far-future values.

```ts
// Server-side TTL → expires_at conversion
function expiryFromTtl(ttlSeconds: number | null): Date | null {
  if (ttlSeconds === null) return null;          // never expires
  if (ttlSeconds < 60)        ttlSeconds = 60;   // floor: 1 min
  if (ttlSeconds > 365*86400) ttlSeconds = 365*86400; // ceiling: 1 year
  return new Date(Date.now() + ttlSeconds * 1000);
}
```

## Reactive Expiry (Lazy)

The simplest design: **never run a sweeper.** Every read checks `expires_at < now()`; expired rows return `410`. Storage grows monotonically.

**Pros:**

- Zero background machinery.
- No sweeper bugs, no cron, no DLQ.
- Always correct from the user's perspective.

**Cons:**

- Storage grows unboundedly. At 1 M pastes/day × 10 KB metadata + 10 KB content × 365 days = ~7 TB of dead rows after a year.
- Indexes balloon. B-trees, partial indexes, and the heap all carry every paste ever created.
- Backups, replication, and `VACUUM` time scale with dead data, not live data.
- Compliance officers will not sign off — "deleted" data is still readable to a DBA.

Reactive-only is acceptable for **prototypes and low-volume** systems (≤100 K rows). At any real scale, combine it with active reclamation. **The read check stays** — it is the visibility gate. The sweeper handles bytes.

## Active Sweeper Job

A scheduled worker that periodically deletes expired rows. The basic shape from the parent doc:

```sql
WITH expired AS (
  SELECT id, content_ref
  FROM   pastes
  WHERE  expires_at < NOW()
    AND  deleted_at IS NULL
  LIMIT  10000
  FOR UPDATE SKIP LOCKED
)
UPDATE pastes
SET    deleted_at = NOW()
FROM   expired
WHERE  pastes.id = expired.id
RETURNING pastes.id, pastes.content_ref;
```

Design choices:

- **Cron-based vs continuous.** Cron (every 5 min, 1 h, 1 day) is simpler but bursty. Continuous (a long-running worker that loops, sleeps briefly between batches) gives smoother DB load and lower expiry-to-deletion latency. For Pastebin, 5-minute cron is usually fine; for burn-after-reading or compliance, prefer continuous.
- **Soft delete first, hard delete later.** Set `deleted_at` immediately so reads stop returning content; physically `DELETE` after a short tombstone window (e.g., 24 h). See [Soft Delete vs Hard Delete](#soft-delete-vs-hard-delete).
- **Batch size.** Too small → high overhead per row. Too large → long transactions, lock contention, replication lag. 1 K – 10 K rows per iteration is a typical sweet spot.
- **`FOR UPDATE SKIP LOCKED`.** Allows multiple sweeper workers to run in parallel without stepping on each other; each grabs a different chunk.

### Throughput math

Suppose 1 M new pastes/day, average TTL 7 days, steady state. Daily expirations = 1 M. To stay caught up running every 5 minutes (288 runs/day): each run must delete `1_000_000 / 288 ≈ 3500` rows. At a comfortable Postgres delete rate of ~1 K rows/second on a single connection, a 3500-row batch takes ~3.5 s. Plenty of headroom. At 100 M/day this becomes 350 K/run — now batching, partitioning, and parallel workers matter.

### DELETE vs TRUNCATE-by-partition

Row-by-row `DELETE` writes a WAL record per row, fires triggers, updates indexes, and produces dead tuples that `VACUUM` must later reclaim. For high-volume sweepers, **`DROP PARTITION` or `TRUNCATE PARTITION`** on a time-partitioned table is 10–100× cheaper — see [Time-Partitioned Tables](#time-partitioned-tables).

## Indexed Expiry Queue

Without an index, the sweeper does a full table scan to find expired rows. With 100 M live pastes that is unacceptable. The fix: a **partial B-tree index** restricted to rows that can possibly need work.

```sql
-- Only rows with a non-null expiry are indexed; never-expires rows are skipped.
CREATE INDEX CONCURRENTLY idx_pastes_expires_at
  ON pastes (expires_at)
  WHERE expires_at IS NOT NULL
    AND deleted_at IS NULL;
```

Why this matters:

- **Size.** If 30% of pastes are "never expires" (paid tier) and 90% of the rest have already been deleted, the partial index covers ~7% of the table. A full index on the same column is 10–14× larger.
- **Maintenance cost.** Every `INSERT`/`UPDATE` on `pastes` updates fewer index pages. Vacuum is faster.
- **Sweeper plan.** `WHERE expires_at < NOW() AND deleted_at IS NULL` becomes an index range scan with a tight upper bound; the planner reads the leftmost leaves of the B-tree and stops as soon as it crosses `NOW()`.

A common mistake is to index `expires_at` without the partial predicate, then wonder why the index is the size of the heap. Postgres's [partial indexes documentation](https://www.postgresql.org/docs/current/indexes-partial.html) is the canonical reference.

If you also frequently query "soon-to-expire" pastes (e.g., "warn the user 1 h before their paste expires"), the same index serves both queries. Re-indexing concurrently (`REINDEX INDEX CONCURRENTLY`) keeps it healthy.

## Time-Partitioned Tables

Once daily volume crosses ~10 M rows, single-table sweeping breaks down. Time-partitioning by `created_at` (or by `expires_at` if TTL distribution is narrow) turns expiry from a row-by-row delete into a metadata operation.

```sql
-- Parent table; rows live only in child partitions
CREATE TABLE pastes (
  id          BIGSERIAL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at  TIMESTAMPTZ,
  content_ref TEXT,
  ...
) PARTITION BY RANGE (created_at);

-- Monthly partitions, created in advance
CREATE TABLE pastes_2026_04 PARTITION OF pastes
  FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE pastes_2026_05 PARTITION OF pastes
  FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

When every paste in `pastes_2026_04` has expired (because the maximum supported TTL is, say, 30 days and we are now in June), drop the whole partition:

```sql
-- Detach so the drop is non-blocking, then drop
ALTER TABLE pastes DETACH PARTITION pastes_2026_04 CONCURRENTLY;
DROP TABLE pastes_2026_04;
```

This is **O(metadata)**, not O(rows). 50 M rows vanish in seconds. No `VACUUM` needed. No index bloat. WAL impact is minimal (just the catalog change). Replicas pick it up trivially.

Caveats:

- Partition boundaries must align with the **maximum** allowed TTL. If a single paste in a month has a 1-year TTL, you cannot drop that partition for a year.
- Cross-partition queries (e.g., "all pastes by user X") require partition-wise scans; foreign keys to the partitioned table need careful handling.
- Mixed long-lived and short-lived rows fight each other. A common solution: **split into two tables** — `pastes_ephemeral` (partitioned, short TTL) and `pastes_permanent` (unpartitioned, never-expires).

The Postgres [table partitioning docs](https://www.postgresql.org/docs/current/ddl-partitioning.html) cover the operational details (constraint exclusion, partition pruning, attaching/detaching).

## S3 Lifecycle for Content

For metadata, the DB sweeper is the primary tool. For S3 content blobs, **let S3 do it for you.** S3 lifecycle rules run server-side, cost nothing extra, and reclaim storage without any application work.

```json
{
  "Rules": [
    {
      "ID": "expire-ephemeral-pastes",
      "Status": "Enabled",
      "Filter": { "Prefix": "ephemeral/" },
      "Expiration": { "Days": 31 },
      "NoncurrentVersionExpiration": { "NoncurrentDays": 1 }
    },
    {
      "ID": "transition-cold-pastes",
      "Status": "Enabled",
      "Filter": { "Prefix": "permanent/" },
      "Transitions": [
        { "Days": 30,  "StorageClass": "STANDARD_IA" },
        { "Days": 180, "StorageClass": "GLACIER_IR" }
      ]
    }
  ]
}
```

Lifecycle has known limits:

- **Granularity is days, not minutes.** A 10-minute TTL paste cannot rely on S3 lifecycle for cleanup. The application sweeper must issue `DeleteObject` for short-TTL pastes.
- **Once-per-day evaluation.** S3 evaluates rules approximately once per day. Even a 1-day rule may leave objects up to 24 h past expiry.
- **Per-object TTL is not native.** S3 does not honor a per-object `expires_at` tag automatically. You can _emulate_ it: tag the object with `expires_on=2026-05-01` at PUT time, then run a daily lifecycle filter `tag:expires_on<today`. AWS supports this via `ObjectTags` filters.

### The race-condition trap

Two systems delete the same object based on independent clocks. If S3 lifecycle wins, the DB row still says "live" but `GetObject` returns `404`. The user sees a confusing error: the paste exists but its body is missing.

**Fix:** make the metadata DB the only deletion trigger. Either:

1. Sweeper sets `deleted_at`, then issues `DeleteObject`. No lifecycle policy at all (or only as a long-tail safety net at, say, 90 days post-expiry).
2. Or: read path treats `expires_at < now()` as `410 Gone` _before_ touching S3. Lifecycle can delete whenever; the user never reaches S3 for an expired paste.

Path 2 is simpler. Path 1 is needed when S3 holds large legitimate objects for non-expiring pastes alongside short-TTL ones.

The [S3 lifecycle documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html) covers the precise evaluation timing and tag-based filtering.

## Redis TTL for Hot Cache

The cache layer (Redis or Memcached) sits in front of the DB to absorb read-heavy traffic on popular pastes. Cache entries **must** have a TTL ≤ the DB TTL, otherwise expired pastes leak through.

```text
SETEX paste:abc123 600 '{"content":"...","content_type":"text/plain"}'
```

Or with `EXPIRE` post-hoc:

```text
SET    paste:abc123 '{"content":"..."}'
EXPIRE paste:abc123 600
```

Rules:

1. **Cache TTL = min(remaining_paste_ttl, max_cache_ttl).** If the paste has 9 days left and max cache TTL is 1 hour, cache for 1 hour. If the paste has 90 seconds left, cache for 90 seconds (or just don't cache).
2. **Don't cache near-expiry pastes.** A paste with 5 seconds of life left is not worth caching; the cost of a stale-read window outweighs the saved DB read.
3. **Active invalidation on delete.** When the sweeper soft-deletes, it must also `DEL paste:<id>`. Otherwise the cache holds an expired paste for up to its TTL.
4. **Burn-after-read pastes should not be cached at all** — or only with a 1-second TTL — to avoid serving the body twice. See [Burn-After-Reading](#burn-after-reading).

The Redis [`EXPIRE` command documentation](https://redis.io/commands/expire/) describes the semantics, including how `EXPIRE` interacts with `PERSIST`, key replacement, and the active-expiration sampling algorithm.

## Sweepers at Scale

Once expiry volume crosses ~10 K rows/minute, a naive sweeper saturates the DB. Production-grade sweepers add:

```go
// Chunked, throttled sweeper loop (Go pseudocode)
const (
    batchSize        = 1000
    targetRowsPerSec = 5000
    sleepBetween     = 200 * time.Millisecond
)

func sweepLoop(ctx context.Context, db *sql.DB, s3 S3Client, q Queue) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }

        start := time.Now()
        rows, err := db.QueryContext(ctx, `
            WITH expired AS (
              SELECT id, content_ref
              FROM   pastes
              WHERE  expires_at < NOW()
                AND  deleted_at IS NULL
              ORDER  BY expires_at
              LIMIT  $1
              FOR UPDATE SKIP LOCKED
            )
            UPDATE pastes
            SET    deleted_at = NOW()
            FROM   expired
            WHERE  pastes.id = expired.id
            RETURNING pastes.id, pastes.content_ref;
        `, batchSize)
        if err != nil {
            metrics.SweepErrors.Inc()
            time.Sleep(5 * time.Second)
            continue
        }

        var refs []string
        for rows.Next() {
            var id int64
            var ref string
            _ = rows.Scan(&id, &ref)
            refs = append(refs, ref)
        }
        rows.Close()

        if len(refs) == 0 {
            time.Sleep(30 * time.Second) // idle backoff
            continue
        }

        // S3 deletes go through a queue: best-effort, retried, decoupled
        for _, r := range refs {
            _ = q.Enqueue(ctx, "s3.delete", r)
        }

        metrics.SweptRows.Add(float64(len(refs)))
        metrics.SweepBatchDuration.Observe(time.Since(start).Seconds())

        // Throttle: aim for targetRowsPerSec
        time.Sleep(sleepBetween)
    }
}
```

Operational guardrails:

- **Watchdog metrics.** Export `pastebin_sweeper_lag_seconds` (= `now() - oldest unsoft-deleted expired row`). Alert if > 15 min. Export `pastebin_sweeper_rows_swept_total` and `pastebin_sweeper_errors_total`.
- **Replica lag awareness.** Bulk deletes generate WAL. If you have synchronous replicas, large batches can stall commits. Monitor `pg_stat_replication.write_lag`.
- **Off-peak scheduling.** If your traffic has a clear diurnal pattern, increase batch size and worker concurrency during low-traffic hours, decrease during peak.
- **DLQ for S3 deletes.** S3 `DeleteObject` can fail transiently. Failed deletes go to a retry queue with exponential backoff; after N failures, alert a human.

## Burn-After-Reading

A paste that disappears the moment it is read. Useful for one-time secrets, password handoffs, paste-and-forget. Two design knobs:

1. **Atomic decrement-and-delete on read.** Use a single SQL statement that returns the content _and_ marks the row deleted in one transaction. No race window where two clients can both succeed.

```sql
UPDATE pastes
SET    deleted_at = NOW(),
       view_count = view_count + 1
WHERE  id = $1
  AND  deleted_at IS NULL
  AND  burn_after_read = TRUE
  AND  view_count = 0
RETURNING content_ref, content_type;
-- Zero rows -> already burned -> 410 Gone
```

The `view_count = 0` guard is the atomic gate. Postgres's row lock ensures only one transaction wins.

For Redis-backed counters, the same idea via Lua (atomic on the Redis side):

```lua
-- KEYS[1] = "paste:abc123"
-- ARGV[1] = max_views (1 for BAR)
local n = redis.call("HINCRBY", KEYS[1], "views", 1)
if n > tonumber(ARGV[1]) then
  redis.call("DEL", KEYS[1])
  return nil
end
return redis.call("HGET", KEYS[1], "content")
```

2. **CDN cache-leakage problem.** Even with atomic DB delete, a CDN edge node may have cached the response. The "first" read populates the edge cache; subsequent reads from other clients in the same region get the cached body without ever hitting your origin. The DB row is deleted, but the bytes live on at the edge for hours.

**Mitigations:**

- `Cache-Control: no-store, private` on burn-after-read responses. Most CDNs (CloudFront, Fastly, Cloudflare) honor this.
- Use a **distinct URL per request** if practical (signed URL with short expiry).
- Serve burn-after-read pastes from a path that the CDN is configured _not_ to cache (`/pastes/secret/*`).
- For the highest-assurance case (true one-time secret), bypass the CDN entirely and require the request to hit origin.

## Per-Tier Expiry Policies

Different users get different TTL budgets. A clean way to model this is a **policy engine** that maps `(tier, requested_ttl)` to an effective `expires_at`:

| Tier | Default TTL | Max TTL | Min TTL | Never-expires? |
|---|---|---|---|---|
| Anonymous | 24 h | 7 days | 5 min | No |
| Registered (free) | 30 days | 1 year | 5 min | No |
| Paid | 30 days | unlimited | 5 min | Yes |
| Enterprise | 1 year | unlimited | 5 min | Yes (with audit) |

```ts
function resolveExpiry(tier: Tier, requestedTtl: number | null): Date | null {
  const policy = TIER_POLICY[tier];

  if (requestedTtl === null) {
    return policy.allowsNever ? null : new Date(Date.now() + policy.defaultTtl * 1000);
  }
  const clamped = Math.max(policy.minTtl, Math.min(requestedTtl, policy.maxTtl));
  return new Date(Date.now() + clamped * 1000);
}
```

Why a single function:

- **Auditable.** Log every (input, tier, output) tuple. Compliance asks "why does this paste live forever?" — the log answers.
- **Testable.** Pure function over (tier, ttl) — exhaustive table-driven tests.
- **Changeable.** Tier upgrades may extend existing pastes; tier downgrades shorten them. Policy is one place to encode this.

Tie the policy to **subscription state changes**: when a user downgrades from paid to free, run a job that clamps `expires_at` on their existing pastes to the free-tier maximum. Keep a 30-day grace before actual expiry to allow re-upgrade.

## Compliance Deletes (GDPR)

[GDPR Article 17](https://gdpr-info.eu/art-17-gdpr/) ("right to erasure") gives users the right to demand deletion of their personal data, and the controller must comply "without undue delay" — typically interpreted as ≤ 30 days. The same is true for CCPA in California and similar laws elsewhere.

The sweeper-and-lifecycle pipeline is **not enough**. A user who deletes their account at 14:00 should not have to wait until tomorrow's lifecycle run for their pastes to vanish. You need a **synchronous delete path** that fires on demand and reports completion.

```text
function gdprErase(userId):
  1. Lookup all pastes owned by userId in DB
     For each row: set deleted_at = NOW(), redact title/owner, schedule hard delete
  2. Issue S3 DeleteObject for each content_ref (sync or async)
  3. DEL all cache keys for these paste IDs
  4. Issue CDN purge for the URL paths (CloudFront CreateInvalidation, etc.)
  5. Append an audit row: { user_id, request_id, deleted_count, completed_at }
  6. Send confirmation email to user
  7. Return success only when steps 1-5 complete; partial failures retry async
```

Notes:

- **Idempotent.** GDPR requests can be retried. Each step must tolerate "already deleted."
- **Hard delete required.** Soft delete is insufficient — GDPR considers the data still present. Schedule an immediate physical `DELETE` (or move to a separately-encrypted tombstone table — see below).
- **CDN purge.** A paste body cached at the edge survives DB deletion. CloudFront, Fastly, and Cloudflare all support invalidation APIs. Issue them.
- **Backups.** Backups are the hard part. Most regulators accept "the data will be removed from backups during the normal rotation cycle" as long as you can prove restored data is re-redacted before processing.
- **Auditable.** Each request produces an immutable audit trail with timestamps.

A 30-day SLA is the legal floor; aim for hours, not days, in your application's normal-case path.

## Auditability of Expiry

Every deletion — whether by sweeper, lifecycle policy, manual admin action, GDPR request, or burn-after-read — should leave a trail. A simple expiry-events table:

```sql
CREATE TABLE paste_lifecycle_events (
  event_id    UUID         PRIMARY KEY,
  paste_id    BIGINT       NOT NULL,
  event_type  TEXT         NOT NULL CHECK (event_type IN (
                  'created','expired_sweeper','expired_lifecycle',
                  'manual_delete','gdpr_erase','burn_read','restored')),
  actor       TEXT,         -- user_id, 'sweeper', 'lifecycle', 'admin:<id>'
  occurred_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  details     JSONB
);
CREATE INDEX ON paste_lifecycle_events (paste_id, occurred_at);
CREATE INDEX ON paste_lifecycle_events (event_type, occurred_at);
```

What this enables:

- **Compliance reports.** "Show all GDPR erasures completed in Q1, with timestamps and elapsed time per request."
- **Forensics.** A user complains their paste vanished early — query events, find which actor deleted it.
- **Sweeper SLO tracking.** Diff `expired_sweeper.occurred_at - pastes.expires_at` for the median expiry-to-delete latency.
- **Anomaly detection.** A spike in `manual_delete` by a single admin is a signal worth investigating.

Keep these events for at least the legal minimum (varies; 1–7 years is typical) regardless of paste lifetime.

## Soft Delete vs Hard Delete

The two-stage delete:

```text
t = expires_at         : visible to users (read-path gate enforces 410)
t = sweeper_run        : deleted_at = NOW()  (soft delete; row still in heap)
t = sweeper + grace    : DELETE FROM pastes  (hard delete; row removed)
```

The grace window (often 24 h – 7 days) gives you:

- **Restore-from-tombstone.** "I accidentally deleted my paste" → admin restore from the soft-deleted row within the window.
- **Moderation review.** A flagged-for-abuse paste sits in tombstone for review before permanent removal.
- **Forensics.** Bytes are still recoverable for incident investigation.

The trade-off: storage cost during the grace window. Tune grace to your needs:

- **Burn-after-read:** zero grace; hard delete immediately.
- **Standard expired pastes:** 24 h grace, then hard delete.
- **GDPR erasure:** zero grace (compliance trumps recoverability).
- **Moderation tombstones:** 30 days, then hard delete _or_ archive to cold storage if legally required (e.g., DMCA preservation for litigation).

A separate **tombstone table** can isolate soft-deleted rows from the live working set, keeping the live B-tree small:

```sql
-- Move deleted rows to a separate table, then truncate quickly
WITH moved AS (
  DELETE FROM pastes
  WHERE  deleted_at < NOW() - INTERVAL '24 hours'
  RETURNING *
)
INSERT INTO pastes_tombstone SELECT * FROM moved;
```

## Anti-Patterns

- **Polling `WHERE expires_at < NOW()` without an index.** Full scan every minute. DB CPU pegged. Fix: partial B-tree index.
- **Trusting client-supplied `expires_at` timestamps.** Client clocks lie. Compute server-side from a TTL.
- **Cache TTL longer than DB TTL.** Expired pastes leak through the cache for minutes. Fix: `cache_ttl = min(remaining, max_cache)`.
- **Lifecycle-policy-only deletion for short TTLs.** S3 lifecycle runs ~daily; "10-minute paste" survives 23 h 50 min. Fix: app sweeper for short TTLs, lifecycle as cost-saving long-tail.
- **No CDN invalidation on burn-after-read or GDPR erase.** Bytes live on at the edge. Fix: explicit invalidation step.
- **Sweeper `DELETE` without `LIMIT`.** Long transaction, replication lag, lock contention. Fix: chunked deletes with `LIMIT` + `SKIP LOCKED`.
- **Conflating soft-delete and visibility.** Forgetting to add `AND deleted_at IS NULL` on read queries. Fix: always include the predicate, or use a view.
- **Missing audit trail.** "We deleted that paste." "When? Why? Who?" — no answer. Fix: lifecycle events table.
- **One sweeper for all tiers.** Burn-after-read needs sub-second; standard pastes can wait minutes. Fix: separate workers or priority queues.
- **Hard-deleting before backups capture the deletion intent.** Restore-from-backup re-creates a paste the user thought was gone. Fix: redact during backup restore, or use logical (not physical) backups that respect deletes.

## Related

- [Pastebin design (parent)](../design-pastebin.md)
- [Databases as a component](../../../building-blocks/databases-as-a-component.md)
- [Object and blob storage](../../../building-blocks/object-and-blob-storage.md)
- [Caching as a component](../../../building-blocks/caches-as-a-component.md)
- [Change Data Capture](../../../data-consistency/change-data-capture.md)

## References

- PostgreSQL — [Partial Indexes](https://www.postgresql.org/docs/current/indexes-partial.html)
- PostgreSQL — [Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- AWS — [S3 Object Lifecycle Management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- AWS — [S3 Lifecycle configuration elements](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html)
- Redis — [`EXPIRE` command](https://redis.io/commands/expire/)
- Redis — [Key expiration internals](https://redis.io/docs/latest/develop/use/keyspace/#how-redis-expires-keys)
- GDPR — [Article 17 — Right to erasure ("right to be forgotten")](https://gdpr-info.eu/art-17-gdpr/)
- IETF — [RFC 9110 §15.5.11 — 410 Gone](https://www.rfc-editor.org/rfc/rfc9110.html#name-410-gone)
