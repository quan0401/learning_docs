---
title: "PostgreSQL MVCC & Visibility — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, postgres, mvcc, transactions, isolation, database]
---

# PostgreSQL MVCC & Visibility — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `postgres` `mvcc` `transactions` `isolation` `database`

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
Active-recall practice for PostgreSQL MVCC and tuple visibility. The 24 traps span snapshot semantics (READ COMMITTED vs REPEATABLE READ vs SERIALIZABLE), `xmin`/`xmax` lifecycle, vacuum and bloat, the visibility map and index-only scans, transaction-ID wraparound, replication-slot WAL retention, and 2PC orphans. Each bug cites the PostgreSQL manual, a real CVE, a public postmortem, or a recognized PG authority. Suggested cadence: one section per sitting; revisit after every Postgres release-notes read-through.

## How to use this doc
- Try to spot the bug before opening `<details>`.
- Hints are one line; solutions in §4 keyed by bug number.
- Don't skip Easy unless you've already nailed these.

## 1. Easy (warm-up traps)

### Bug 1 — Lost update under READ COMMITTED
```sql
-- Default isolation. Two sessions, same row.
-- session 1
BEGIN;
SELECT balance FROM accounts WHERE id = 7;   -- reads 100
-- ... app computes new = 100 - 30 = 70 ...
UPDATE accounts SET balance = 70 WHERE id = 7;
COMMIT;

-- session 2 (interleaved between SELECT and UPDATE in session 1)
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 7;  -- 100 -> 50
COMMIT;
```
<details><summary>Hint</summary>
The read in session 1 is not part of the same snapshot as its write.
</details>

### Bug 2 — DELETE then expecting disk reclaim
```sql
DELETE FROM big_audit WHERE created_at < now() - interval '90 days';
-- ops dashboard: relation size unchanged. "Did the DELETE not run?"
```
<details><summary>Hint</summary>
DELETE marks tuples dead; it does not free space.
</details>

### Bug 3 — Sequence gap after rollback misread as data loss
```sql
CREATE TABLE orders (id BIGSERIAL PRIMARY KEY, total numeric);

BEGIN;
INSERT INTO orders (total) VALUES (10) RETURNING id;  -- 1
ROLLBACK;

INSERT INTO orders (total) VALUES (20) RETURNING id;  -- 2 (gap!)
-- "Where did order #1 go? We must have lost data."
```
<details><summary>Hint</summary>
Sequences are non-transactional by design.
</details>

### Bug 4 — `count(*)` slow on a "small" table
```sql
-- table has 50k live rows
SELECT count(*) FROM events;
-- 14 seconds. Why?
\dt+ events     -- size: 18 GB
```
<details><summary>Hint</summary>
Every visible row check still has to walk the heap; bloat amplifies it.
</details>

### Bug 5 — `idle in transaction` connection bloating everything
```sql
-- App opens BEGIN, runs a SELECT, then the user goes to lunch.
-- pg_stat_activity shows state='idle in transaction' for 47 minutes.
-- Meanwhile autovacuum is running but reclaiming nothing.
```
<details><summary>Hint</summary>
A held snapshot pins the global xmin horizon.
</details>

### Bug 6 — `CREATE INDEX` blocks writes for 40 minutes
```sql
-- Production migration:
CREATE INDEX idx_orders_user_id ON orders (user_id);
-- All UPDATEs and INSERTs queue behind it. Outage.
```
<details><summary>Hint</summary>
There's a non-blocking variant for online builds.
</details>

### Bug 7 — `VACUUM FULL` "to reclaim space" during business hours
```sql
-- Disk is 90% full. On-call decides to run:
VACUUM FULL big_table;
-- Every query against big_table hangs until it finishes.
```
<details><summary>Hint</summary>
Read the lock level VACUUM FULL takes.
</details>

## 2. Subtle (review-passers)

### Bug 8 — Snapshot per statement, not per transaction (READ COMMITTED)
```sql
BEGIN;  -- READ COMMITTED (default)
SELECT count(*) FROM payments WHERE status = 'pending';   -- 100
-- ... a concurrent COMMIT inserts 5 more 'pending' rows ...
SELECT id FROM payments WHERE status = 'pending';         -- returns 105 ids
-- App logic assumed both queries see the same set.
COMMIT;
```
<details><summary>Hint</summary>
Each statement under READ COMMITTED takes a fresh snapshot.
</details>

### Bug 9 — Write skew under REPEATABLE READ
```sql
-- Constraint: at least one doctor must remain on call.
-- Two doctors, both currently on_call=true. Two sessions run in parallel:

-- session 1                           -- session 2
BEGIN ISOLATION LEVEL REPEATABLE READ; BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM oncall            SELECT count(*) FROM oncall
  WHERE on_call;       -- 2              WHERE on_call;       -- 2
UPDATE oncall SET on_call=false        UPDATE oncall SET on_call=false
  WHERE doctor='Alice';                  WHERE doctor='Bob';
COMMIT;                                COMMIT;
-- Now zero doctors on call. Both transactions committed cleanly.
```
<details><summary>Hint</summary>
REPEATABLE READ in PG does not detect this anomaly.
</details>

### Bug 10 — HOT update missed because indexed column changed
```sql
-- users(id PK, last_seen timestamptz, email text)
-- Indexes: idx_users_email(email), idx_users_last_seen(last_seen)
-- App updates last_seen on every page view (millions/day).
-- Index bloat on idx_users_last_seen grows unbounded.
UPDATE users SET last_seen = now() WHERE id = $1;
```
<details><summary>Hint</summary>
HOT updates require the new tuple to fit on the same page AND no indexed column changed.
</details>

### Bug 11 — Index-only scan returns rows after bulk INSERT, but scan is slow
```sql
COPY events FROM '/tmp/big.csv';   -- inserts 50M rows
-- Immediately:
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM events WHERE day = '2026-05-03';
-- "Index Only Scan ... Heap Fetches: 49,983,201"
-- Why is it hitting the heap on every row?
```
<details><summary>Hint</summary>
The visibility map is updated by VACUUM, not by INSERT.
</details>

### Bug 12 — Subquery sees different snapshot than outer query
```sql
-- READ COMMITTED. Long-running report:
UPDATE summary
   SET total = (SELECT sum(amount) FROM payments WHERE day = current_date)
 WHERE day = current_date;
-- Concurrent INSERTs into payments during the UPDATE.
-- The outer UPDATE's matched row(s) were chosen from one snapshot;
-- the subquery sums from another.
```
<details><summary>Hint</summary>
Each SELECT/UPDATE statement opens its own snapshot at READ COMMITTED.
</details>

### Bug 13 — `INSERT ... ON CONFLICT` on a partial unique index
```sql
CREATE UNIQUE INDEX uq_active_email
  ON users (email) WHERE deleted_at IS NULL;

INSERT INTO users (email, deleted_at)
VALUES ('a@x.com', NULL)
ON CONFLICT (email) DO UPDATE SET deleted_at = NULL;
-- ERROR:  there is no unique or exclusion constraint matching the ON CONFLICT specification
```
<details><summary>Hint</summary>
The conflict target must match the partial index predicate exactly.
</details>

### Bug 14 — `SELECT FOR UPDATE` inside SERIALIZABLE
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT id FROM jobs WHERE status = 'queued' LIMIT 1 FOR UPDATE;
-- Worker processes job, commits.
-- Under heavy contention: many workers see "could not serialize access
-- due to read/write dependencies among transactions" and retry storms.
```
<details><summary>Hint</summary>
SSI predicate locks interact badly with row-locking work-queue patterns.
</details>

### Bug 15 — TOAST bloat after wide UPDATEs
```sql
-- articles(id PK, body text)  -- bodies often > 2KB and stored TOASTed
UPDATE articles SET updated_at = now() WHERE id = $1;
-- Even though body didn't change, full-row UPDATEs create new toast pointers
-- when the trigger rewrites body too.
-- pg_total_relation_size(articles) grows; pg_relation_size(articles) does not.
```
<details><summary>Hint</summary>
Look at the toast relation size, and audit triggers that touch unchanged columns.
</details>

### Bug 16 — Foreign-key deadlock from update order
```sql
-- parent: orders(id), child: order_items(order_id REFERENCES orders)
-- session 1                       -- session 2
BEGIN;                              BEGIN;
UPDATE order_items                  UPDATE orders
   SET qty = 2 WHERE id = 10;          SET status='paid' WHERE id = 1;
UPDATE orders                       UPDATE order_items
   SET status='paid' WHERE id = 1;     SET qty = 3 WHERE id = 10;
-- ERROR: deadlock detected
```
<details><summary>Hint</summary>
FK actions take row-share locks on the parent during child UPDATE.
</details>

### Bug 17 — `pg_dump` on a hot table failing with "snapshot too old"
```sql
-- pg_dump is running for 6 hours against a 2 TB OLTP database.
-- old_snapshot_threshold = '30min'.
pg_dump -t huge_events ...
-- ERROR:  snapshot too old
```
<details><summary>Hint</summary>
This GUC explicitly trades long-snapshot safety for bloat control.
</details>

### Bug 18 — Lock queue starvation: ALTER TABLE behind a long SELECT
```sql
-- session A: 30-minute analytic SELECT on big_table  (AccessShare)
-- session B: ALTER TABLE big_table ADD COLUMN x int; (AccessExclusive — waits)
-- session C: SELECT * FROM big_table LIMIT 1;        (AccessShare — also waits!)
-- New short SELECTs queue behind B even though A still holds AccessShare.
```
<details><summary>Hint</summary>
Postgres locks are FIFO; an exclusive waiter blocks compatible newcomers.
</details>

## 3. Senior trap (production-only failures)

### Bug 19 — Transaction-ID wraparound emergency stop
```sql
-- Symptom from logs:
-- WARNING: database "prod" must be vacuumed within 10485760 transactions
-- HINT: To avoid a database shutdown, execute a database-wide VACUUM ...
-- Eventually: ERROR: database is not accepting commands to avoid wraparound
-- data loss in database "prod"
```
<details><summary>Hint</summary>
A long-running prepared transaction or replication slot held the xmin horizon for weeks.
</details>

### Bug 20 — Logical replication slot retains TBs of WAL after consumer goes offline
```sql
-- Subscriber crashes Friday night. Pager fires Sunday: pg_wal at 92%.
SELECT slot_name, active, restart_lsn, pg_size_pretty(
  pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;
-- slot_name | active | retained
-- analytics | f      | 1873 GB
```
<details><summary>Hint</summary>
Inactive logical slots still pin WAL until either advanced or dropped.
</details>

### Bug 21 — Orphaned PREPARE TRANSACTION holding xmin forever
```sql
-- App framework was reconfigured; nobody runs 2PC anymore.
SELECT gid, prepared, owner, database FROM pg_prepared_xacts;
--  gid          | prepared              | owner | database
-- _saga_77af2c  | 2026-02-14 03:12:09Z  | app   | prod
-- vacuum cannot freeze rows newer than this prepared xact.
-- bloat ticks up by ~2GB/day.
```
<details><summary>Hint</summary>
Prepared transactions survive reconnects, restarts, and crashes by design.
</details>

### Bug 22 — SSI false positive cascade under read-only workload
```sql
-- Reporting service runs many SERIALIZABLE READ ONLY DEFERRABLE-eligible queries,
-- but app forgot DEFERRABLE:
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY;
-- ... long analytic SELECT ...
-- ERROR: could not serialize access due to read/write dependencies among transactions
-- Retries spike; OLTP latency suffers because of predicate-lock memory pressure.
```
<details><summary>Hint</summary>
DEFERRABLE for read-only avoids predicate locks by waiting for a safe snapshot.
</details>

### Bug 23 — Replica far behind because WAL receiver paused, but `streaming` says "yes"
```sql
-- Primary:
SELECT application_name, state, pg_wal_lsn_diff(
  pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
-- replica1 | streaming | 412 GB   <-- 6h of lag, but state still "streaming"
-- Failover would lose 6 hours.
```
<details><summary>Hint</summary>
`state='streaming'` means bytes are flowing; it says nothing about apply progress.
</details>

## 4. Solutions

### Bug 1 — Lost update under READ COMMITTED
**Root cause:** Under READ COMMITTED each statement gets its own snapshot, and a bare `SELECT` followed by application-side compute and `UPDATE` is not atomic. Two sessions can each read 100, each compute their own delta, and the second writer overwrites the first. The DB sees no conflict.
**Fix:**
```sql
-- Option A: do the arithmetic in the UPDATE (atomic on the row)
UPDATE accounts SET balance = balance - 30 WHERE id = 7;

-- Option B: lock the row first
BEGIN;
SELECT balance FROM accounts WHERE id = 7 FOR UPDATE;
UPDATE accounts SET balance = $1 WHERE id = 7;
COMMIT;

-- Option C: REPEATABLE READ + retry on serialization_failure
```
**Reference:** [PostgreSQL 13.2.1 Read Committed Isolation Level](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED)

### Bug 2 — DELETE then expecting disk reclaim
**Root cause:** DELETE only marks tuples as dead (`xmax` set). The space is not reusable until VACUUM runs and the visibility map confirms no transaction can still see those tuples. Even after VACUUM, only the *page* free space is reclaimable; the file does not shrink.
**Fix:**
```sql
DELETE FROM big_audit WHERE created_at < now() - interval '90 days';
VACUUM (VERBOSE, ANALYZE) big_audit;          -- reclaims for reuse
-- to actually shrink the file:
VACUUM FULL big_audit;                        -- exclusive lock; or pg_repack
```
**Reference:** [PostgreSQL 13.1 MVCC Introduction](https://www.postgresql.org/docs/current/mvcc-intro.html) and [25.1 Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)

### Bug 3 — Sequence gap after rollback misread as data loss
**Root cause:** `nextval()` is intentionally non-transactional so concurrent writers don't serialize on the sequence. A rolled-back INSERT consumes a value that is never reused. This is documented behavior, not a bug.
**Fix:**
```sql
-- Don't rely on serial/identity for "no gaps". If you truly need gapless,
-- use a counter table protected by SELECT ... FOR UPDATE:
CREATE TABLE invoice_counter (next_no bigint NOT NULL);
-- Within the same transaction:
UPDATE invoice_counter SET next_no = next_no + 1 RETURNING next_no - 1;
```
**Reference:** [PostgreSQL: CREATE SEQUENCE — Notes](https://www.postgresql.org/docs/current/sql-createsequence.html)

### Bug 4 — `count(*)` slow on a "small" table
**Root cause:** Postgres has no global row count; each row's visibility must be checked from the heap (or visibility map for index-only scans). With heavy bloat, the heap is huge even when live tuples are few. `count(*)` walks every page.
**Fix:**
```sql
-- estimate (instant):
SELECT reltuples::bigint FROM pg_class WHERE relname = 'events';

-- shrink the bloat:
VACUUM (FULL, VERBOSE) events;   -- or pg_repack for online

-- ensure index-only scan eligibility for count queries:
VACUUM ANALYZE events;
```
**Reference:** [PostgreSQL Wiki — Slow Counting](https://wiki.postgresql.org/wiki/Slow_Counting)

### Bug 5 — `idle in transaction` connection bloating everything
**Root cause:** Any open transaction holds a snapshot, and the global `xmin` horizon is the oldest xmin across all backends. Autovacuum can only remove dead tuples that are invisible to *every* live snapshot, so a single idle-in-transaction backend prevents reclamation cluster-wide.
**Fix:**
```sql
-- 1. Bound the wait at the server:
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';
SELECT pg_reload_conf();

-- 2. Find offenders:
SELECT pid, now() - xact_start AS age, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY age DESC;
```
**Reference:** [PostgreSQL: idle_in_transaction_session_timeout](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-IDLE-IN-TRANSACTION-SESSION-TIMEOUT)

### Bug 6 — `CREATE INDEX` blocks writes for 40 minutes
**Root cause:** Plain `CREATE INDEX` takes a `SHARE` lock on the table, which blocks all writes. On a large hot table this is an outage.
**Fix:**
```sql
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders (user_id);
-- Then validate:
SELECT indexrelid::regclass, indisvalid
FROM pg_index WHERE indrelid = 'orders'::regclass;
-- Re-run if indisvalid = false (drop and recreate concurrently).
```
**Reference:** [PostgreSQL: Building Indexes Concurrently](https://www.postgresql.org/docs/current/sql-createindex.html#SQL-CREATEINDEX-CONCURRENTLY)

### Bug 7 — `VACUUM FULL` "to reclaim space" during business hours
**Root cause:** `VACUUM FULL` rewrites the entire table and takes `ACCESS EXCLUSIVE`. No reads or writes proceed while it runs. It is correct for shrinking files, but never during traffic.
**Fix:**
```sql
-- Online alternative (extension):
-- pg_repack -t big_table -d prod
-- Or schedule VACUUM FULL during a maintenance window.
-- Long term: tune autovacuum so FULL is never needed:
ALTER TABLE big_table SET (autovacuum_vacuum_scale_factor = 0.05);
```
**Reference:** [PostgreSQL: VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html) — see "FULL" notes on locking.

### Bug 8 — Snapshot per statement, not per transaction (READ COMMITTED)
**Root cause:** Under READ COMMITTED every statement starts with a fresh snapshot reflecting all transactions committed up to that moment. Two sequential SELECTs in the same transaction can therefore see different sets of rows.
**Fix:**
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM payments WHERE status = 'pending';
SELECT id      FROM payments WHERE status = 'pending';
-- both see the same snapshot taken at first non-trivial statement
COMMIT;
```
**Reference:** [PostgreSQL 13.2.1 Read Committed Isolation Level](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED)

### Bug 9 — Write skew under REPEATABLE READ
**Root cause:** REPEATABLE READ in Postgres is snapshot isolation. It blocks dirty reads, non-repeatable reads, and phantom reads on the *rows you read*, but does not detect write skew where two transactions read overlapping data and each writes a disjoint piece based on the other's stale read. SERIALIZABLE (SSI) does.
**Fix:**
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ... same logic ...
COMMIT;
-- One transaction will get SQLSTATE 40001 and must retry.
```
**Reference:** [PostgreSQL 13.2.3 Serializable Isolation Level](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-SERIALIZABLE)

### Bug 10 — HOT update missed because indexed column changed
**Root cause:** Heap-Only Tuple updates skip index updates and recycle space within the page, but only when (a) the new tuple fits on the same page AND (b) no indexed column changed. Indexing `last_seen` and updating it on every page view voids HOT, producing one new index entry per update and unbounded index bloat.
**Fix:**
```sql
-- Option A: drop the index if not actually used:
DROP INDEX idx_users_last_seen;

-- Option B: keep last_seen in a separate, narrow table without that index,
-- or write to it less often (sample / batch).

-- Option C: ensure fillfactor leaves room for HOT chains:
ALTER TABLE users SET (fillfactor = 80);
```
**Reference:** ["The Internals of PostgreSQL" — Heap-Only Tuple (HOT)](https://www.interdb.jp/pg/pgsql07.html) (Hironobu Suzuki) and [src/backend/access/heap/README.HOT in postgres source](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/backend/access/heap/README.HOT)

### Bug 11 — Index-only scan slow because visibility map stale
**Root cause:** Index-only scans avoid heap fetches only for pages marked all-visible in the visibility map. The VM is updated by VACUUM, not by INSERT/COPY. After a bulk load every page is "not all-visible", forcing heap fetches per row even though the index has the answer.
**Fix:**
```sql
COPY events FROM '/tmp/big.csv';
VACUUM (VERBOSE, ANALYZE) events;   -- builds the visibility map
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM events WHERE day = '2026-05-03';
-- Heap Fetches: 0
```
**Reference:** [PostgreSQL 11.9 Index-Only Scans and Covering Indexes](https://www.postgresql.org/docs/current/indexes-index-only-scans.html) and [25.1.4 Updating the Visibility Map](https://www.postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-VISIBILITY-MAP)

### Bug 12 — Subquery sees different snapshot than outer query
**Root cause:** Under READ COMMITTED each top-level statement opens its own snapshot. Within a single UPDATE, the targeted rows are chosen with one snapshot, and a sub-SELECT inside that UPDATE sees the same snapshot — but a *separate* SELECT before the UPDATE sees a different one. The classic surprise is when the application reads, computes, and then writes, expecting "transaction-wide" consistency it doesn't have.
**Fix:**
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
WITH s AS (
  SELECT sum(amount) AS total FROM payments WHERE day = current_date
)
UPDATE summary SET total = (SELECT total FROM s) WHERE day = current_date;
COMMIT;
```
**Reference:** [PostgreSQL 13.2.1 Read Committed](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED)

### Bug 13 — `INSERT ... ON CONFLICT` on a partial unique index
**Root cause:** ON CONFLICT requires the inference specification to match a specific unique index, including its predicate. Without the same `WHERE` clause, Postgres cannot prove the inserted row is covered by the partial index.
**Fix:**
```sql
INSERT INTO users (email, deleted_at)
VALUES ('a@x.com', NULL)
ON CONFLICT (email) WHERE deleted_at IS NULL
DO UPDATE SET deleted_at = NULL;
```
**Reference:** [PostgreSQL: INSERT — ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT) (see "index_predicate")

### Bug 14 — `SELECT FOR UPDATE` inside SERIALIZABLE
**Root cause:** SERIALIZABLE uses Serializable Snapshot Isolation (SSI) with predicate locks. A work-queue pattern — many workers each scanning for "first queued row" and locking it — generates many overlapping read sets, producing read/write dependency cycles and serialization failures even when the workers do not logically conflict.
**Fix:**
```sql
-- For work queues, use READ COMMITTED + SKIP LOCKED:
BEGIN;
SELECT id FROM jobs
 WHERE status = 'queued'
 ORDER BY id
 LIMIT 1
 FOR UPDATE SKIP LOCKED;
-- ... process ...
COMMIT;
```
**Reference:** [PostgreSQL 13.2.3 Serializable Isolation Level](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-SERIALIZABLE) and [SELECT — The Locking Clause](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE)

### Bug 15 — TOAST bloat after wide UPDATEs
**Root cause:** When any column of a row is updated, the entire heap tuple is rewritten. If the row has TOASTed columns and a BEFORE UPDATE trigger touches them (even setting `NEW.body = OLD.body`), the TOAST chunks are rewritten too, producing dead toast tuples. `pg_relation_size` only shows the heap; the toast relation grows independently.
**Fix:**
```sql
-- Audit triggers that re-assign large columns:
-- BAD:   NEW.body := OLD.body;
-- GOOD:  -- (don't touch NEW.body unless you actually need to change it)

-- Inspect toast bloat:
SELECT pg_size_pretty(pg_relation_size(reltoastrelid))
FROM pg_class WHERE relname = 'articles';

VACUUM (VERBOSE, ANALYZE) articles;
```
**Reference:** [PostgreSQL 73.2 TOAST](https://www.postgresql.org/docs/current/storage-toast.html)

### Bug 16 — Foreign-key deadlock from update order
**Root cause:** Updating a child row takes a row-share lock on the referenced parent row to prevent it from being deleted mid-transaction. Two transactions that update parent and child in opposite orders form a classic A-B / B-A cycle.
**Fix:**
```sql
-- Adopt one canonical lock order across all code paths:
-- 1. Always touch parent first, child second.
BEGIN;
UPDATE orders      SET status='paid' WHERE id = 1;
UPDATE order_items SET qty    = 2     WHERE id = 10;
COMMIT;
```
**Reference:** [PostgreSQL 13.3.4 Deadlocks](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-DEADLOCKS) and [release notes — improvements to FK locking (PG 9.3)](https://www.postgresql.org/docs/current/release-9-3.html)

### Bug 17 — `pg_dump` snapshot too old
**Root cause:** `old_snapshot_threshold` lets Postgres prune dead tuples that are older than the threshold even if a long-lived snapshot still references them; queries against those pages then error with "snapshot too old". A many-hours `pg_dump` exceeds typical settings.
**Fix:**
```sql
-- Either disable the GUC for hosts that run long backups:
ALTER SYSTEM SET old_snapshot_threshold = -1;   -- disabled
SELECT pg_reload_conf();

-- Or take backups via physical means (pg_basebackup / disk snapshots) which
-- don't rely on long MVCC snapshots.
```
**Reference:** [PostgreSQL: old_snapshot_threshold](https://www.postgresql.org/docs/current/runtime-config-resource.html#GUC-OLD-SNAPSHOT-THRESHOLD)

### Bug 18 — Lock queue starvation behind ALTER TABLE
**Root cause:** Postgres lock acquisition is fair/FIFO. When `ALTER TABLE` requests `AccessExclusive` and waits behind a long `AccessShare` holder, every later requester (even compatible ones) queues behind the exclusive waiter. Short SELECTs back up cluster-wide.
**Fix:**
```sql
-- Always set a low lock_timeout for DDL:
SET lock_timeout = '2s';
SET statement_timeout = '30s';
ALTER TABLE big_table ADD COLUMN x int;        -- fails fast if blocked

-- Then retry, or use a migration tool that does this loop for you
-- (e.g., pg-osc, pg_repack).
```
**Reference:** [PostgreSQL 13.3 Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html) and [lock_timeout](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-LOCK-TIMEOUT)

### Bug 19 — Transaction-ID wraparound emergency stop
**Root cause:** Postgres uses 32-bit XIDs and must "freeze" old tuples before the counter wraps (~2 billion). Any backend with an old `xmin` — long-running transaction, idle-in-transaction, prepared transaction, or replication slot — holds back the freeze horizon. Once the cluster nears wraparound it stops accepting writes to protect data.
**Fix:**
```sql
-- 1. Find the holder:
SELECT pid, backend_xmin, state, now() - xact_start AS age
FROM pg_stat_activity ORDER BY backend_xmin NULLS LAST LIMIT 5;

SELECT slot_name, xmin FROM pg_replication_slots ORDER BY xmin;
SELECT gid, prepared FROM pg_prepared_xacts;

-- 2. Kill / drop / commit the holder, then VACUUM (FREEZE, VERBOSE) database-wide.
-- 3. Add monitoring on age(datfrozenxid).
```
**Reference:** [PostgreSQL 25.1.5 Preventing Transaction ID Wraparound Failures](https://www.postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND) and Sentry's public postmortem [Transaction ID Wraparound in Postgres](https://blog.sentry.io/transaction-id-wraparound-in-postgres/)

### Bug 20 — Logical replication slot retains TBs of WAL
**Root cause:** Replication slots guarantee the primary keeps WAL until the consumer confirms it. A logical slot whose subscriber is offline keeps `restart_lsn` pinned indefinitely, filling `pg_wal`. There is no automatic timeout by default.
**Fix:**
```sql
-- Cap the damage in PG 13+:
ALTER SYSTEM SET max_slot_wal_keep_size = '50GB';
SELECT pg_reload_conf();

-- Triage:
SELECT pg_drop_replication_slot('analytics');
-- Recreate after subscriber is healthy. Add monitoring on
-- pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) per slot.
```
**Reference:** [PostgreSQL: max_slot_wal_keep_size](https://www.postgresql.org/docs/current/runtime-config-replication.html#GUC-MAX-SLOT-WAL-KEEP-SIZE) and the [GitLab 2017 outage postmortem](https://about.gitlab.com/blog/2017/02/10/postmortem-of-database-outage-of-january-31/) (WAL-segment retention/replication failure mode)

### Bug 21 — Orphaned `PREPARE TRANSACTION`
**Root cause:** `PREPARE TRANSACTION` writes a prepared xact to disk; it survives crashes, restarts, and connection loss until COMMIT/ROLLBACK PREPARED. Its xmin pins the freeze horizon. Many shops disable 2PC (`max_prepared_transactions = 0`) but a forgotten one before the change is enough.
**Fix:**
```sql
SELECT gid, prepared, owner, database FROM pg_prepared_xacts;
ROLLBACK PREPARED '_saga_77af2c';
-- Then disable until you actually need 2PC:
ALTER SYSTEM SET max_prepared_transactions = 0;
```
**Reference:** [PostgreSQL: PREPARE TRANSACTION](https://www.postgresql.org/docs/current/sql-prepare-transaction.html) and [MVCC Caveats — long-running prepared xacts](https://www.postgresql.org/docs/current/mvcc-caveats.html)

### Bug 22 — SSI false-positive cascade under read-only workload
**Root cause:** SERIALIZABLE READ ONLY DEFERRABLE waits until it can take a *safe snapshot* — one that cannot participate in a serialization anomaly — and then runs without taking predicate locks. Plain SERIALIZABLE READ ONLY still uses SIREAD locks and can be canceled by writers' anomalies.
**Fix:**
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
-- analytic SELECTs ...
COMMIT;
```
**Reference:** [PostgreSQL 13.2.3 Serializable Isolation Level — DEFERRABLE](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-SERIALIZABLE)

### Bug 23 — Replica `streaming` but apply lag huge
**Root cause:** `pg_stat_replication.state` reports the WAL-shipping state, not the apply state. `replay_lsn` lagging `flush_lsn` means bytes arrived but the standby has not applied them — usually a recovery conflict, a cold cache, or single-process replay falling behind a multi-writer primary.
**Fix:**
```sql
-- Monitor APPLY lag, not just streaming state:
SELECT application_name,
       pg_wal_lsn_diff(pg_current_wal_lsn(), write_lsn)  AS write_lag,
       pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn)  AS flush_lag,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag,
       reply_time
FROM pg_stat_replication;

-- Investigate replication conflicts on the standby:
-- log_recovery_conflict_waits = on
-- hot_standby_feedback = on (if bloat on primary is acceptable)
```
**Reference:** [PostgreSQL 28.2.5 The pg_stat_replication View](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-REPLICATION-VIEW) and [27.5 Hot Standby](https://www.postgresql.org/docs/current/hot-standby.html)

## Related
- [Storage and MVCC](storage-and-mvcc.md)
- [Indexing internals](indexing-internals.md)
- [EXPLAIN ANALYZE guide](../query-optimization/explain-analyze-guide.md)
- [Replication](../operations/replication.md)

## References
- Official docs: PostgreSQL manual — Concurrency Control (postgresql.org/docs/current/mvcc.html), Routine Vacuuming (postgresql.org/docs/current/routine-vacuuming.html), Transaction Isolation (postgresql.org/docs/current/transaction-iso.html), Index-Only Scans (postgresql.org/docs/current/indexes-index-only-scans.html), CREATE INDEX CONCURRENTLY (postgresql.org/docs/current/sql-createindex.html), VACUUM (postgresql.org/docs/current/sql-vacuum.html), PREPARE TRANSACTION (postgresql.org/docs/current/sql-prepare-transaction.html), TOAST (postgresql.org/docs/current/storage-toast.html)
- PostgreSQL Wiki: [Slow Counting](https://wiki.postgresql.org/wiki/Slow_Counting)
- Source: [src/backend/access/heap/README.HOT](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/backend/access/heap/README.HOT)
- Authority blog: ["The Internals of PostgreSQL" by Hironobu Suzuki](https://www.interdb.jp/pg/) — chapters 5–7 on concurrency, vacuum, HOT
- Postmortems: [GitLab 2017 database outage](https://about.gitlab.com/blog/2017/02/10/postmortem-of-database-outage-of-january-31/), [Sentry — Transaction ID Wraparound in Postgres](https://blog.sentry.io/transaction-id-wraparound-in-postgres/)
