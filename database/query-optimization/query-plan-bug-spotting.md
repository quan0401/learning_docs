---
title: "PostgreSQL Query Plans & Indexing — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, postgres, query-optimization, indexing, database]
---

# PostgreSQL Query Plans & Indexing — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `postgres` `query-optimization` `indexing` `database`

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
Active-recall practice for PostgreSQL query optimization and indexing. The 23 traps span EXPLAIN ANALYZE reading, B-tree/GIN/GiST/BRIN choice, partial and expression indexes, leftmost-prefix and column-order rules, planner statistics and `ANALYZE` timing, the `work_mem` cliff for sorts and hash aggregates, parallel-query disqualifiers, function pushdown and volatility, generic-vs-custom plan caching, and connection-pooler statement-cache pitfalls. Each bug cites the PostgreSQL manual or a recognized authority (Markus Winand, depesz, pganalyze, Tomas Vondra, Robert Haas). Suggested cadence: one section per sitting, paired with re-running `EXPLAIN (ANALYZE, BUFFERS)` against your own slowest query log.

## How to use this doc
- Try to spot the bug before opening `<details>`.
- Hints are one line; solutions in §4 keyed by bug number.
- For plan-shape bugs, read the embedded `EXPLAIN` output before peeking — many traps are visible only there.

## 1. Easy (warm-up traps)

### Bug 1 — `LIKE '%foo'` on a B-tree index, expecting a fast lookup
```sql
CREATE INDEX idx_users_email ON users (email);

EXPLAIN ANALYZE
SELECT * FROM users WHERE email LIKE '%@example.com';
--                              QUERY PLAN
-- Seq Scan on users  (cost=0.00..184213 rows=12 width=...)
--   Filter: (email ~~ '%@example.com'::text)
-- Planning Time: 0.2 ms
-- Execution Time: 1837 ms
```
<details><summary>Hint</summary>
B-tree index walks need an anchored prefix — leading wildcard kills it.
</details>

### Bug 2 — Function on indexed column blocks index use
```sql
CREATE INDEX idx_users_email ON users (email);

EXPLAIN ANALYZE
SELECT id FROM users WHERE lower(email) = 'alice@example.com';
-- Seq Scan on users  (Filter: lower(email) = 'alice@example.com')
```
<details><summary>Hint</summary>
The index is on `email`, not on `lower(email)`.
</details>

### Bug 3 — Implicit type cast disables the index
```sql
-- account_no is text; query passes a number.
CREATE INDEX idx_accounts_acct_no ON accounts (account_no);

EXPLAIN ANALYZE
SELECT * FROM accounts WHERE account_no = 4711;
-- Seq Scan on accounts  (Filter: (account_no)::bigint = 4711)
```
<details><summary>Hint</summary>
Postgres casts the column, not the literal — read the `Filter:` line.
</details>

### Bug 4 — Missing `ANALYZE` after a bulk load
```sql
CREATE TABLE events (id bigserial PRIMARY KEY, day date, kind text);
COPY events FROM '/tmp/events.csv';   -- 50M rows
-- No ANALYZE.
EXPLAIN ANALYZE
SELECT count(*) FROM events WHERE day = '2026-05-03';
-- Plan estimates rows=1 because pg_class.reltuples is still 0.
-- Planner picks a nested loop in a downstream join. Disaster.
```
<details><summary>Hint</summary>
Statistics are not refreshed by `COPY` / `INSERT`.
</details>

### Bug 5 — Boolean column index that the planner refuses to use
```sql
-- 99% of rows have is_active = true.
CREATE INDEX idx_users_active ON users (is_active);

EXPLAIN ANALYZE SELECT * FROM users WHERE is_active = true;
-- Seq Scan on users
```
<details><summary>Hint</summary>
For a value present in nearly every row, an index scan is slower than a seq scan.
</details>

### Bug 6 — Composite index, wrong leading column
```sql
CREATE INDEX idx_orders_status_user ON orders (status, user_id);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42;
-- Seq Scan on orders  (Filter: user_id = 42)
```
<details><summary>Hint</summary>
Leftmost-prefix rule — you can't skip the first key.
</details>

### Bug 7 — `COUNT(*)` walks the heap because no covering index + visibility map cold
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM events WHERE day = '2026-05-03';
-- Index Scan using idx_events_day on events
--   Buffers: shared hit=18 read=204812
-- Heap Fetches: 49,983,201
```
<details><summary>Hint</summary>
Index-only scan needs both a covering index and `all-visible` pages.
</details>

## 2. Subtle (review-passers)

### Bug 8 — Composite index with range on the wrong column position
```sql
-- We frequently filter: status = 'open' AND created_at > now() - '7 days'
CREATE INDEX idx_orders_created_status ON orders (created_at, status);

EXPLAIN ANALYZE
SELECT * FROM orders
 WHERE status = 'open'
   AND created_at > now() - interval '7 days';
-- Bitmap Index Scan on idx_orders_created_status
--   Index Cond: (created_at > now() - interval '7 days')
--   Filter: status = 'open'    -- <-- read past 6 days of every status
```
<details><summary>Hint</summary>
A range scan stops the index walk; equality columns must come first.
</details>

### Bug 9 — Partial index whose predicate doesn't match the query
```sql
CREATE INDEX idx_orders_open ON orders (user_id)
  WHERE status = 'open';

-- Application code:
EXPLAIN ANALYZE
SELECT id FROM orders
 WHERE user_id = 42 AND status IN ('open','pending');
-- Seq Scan on orders
```
<details><summary>Hint</summary>
The planner must prove the partial-index `WHERE` covers the query predicate.
</details>

### Bug 10 — Expression index missing while query uses an expression
```sql
CREATE INDEX idx_users_email ON users (email);

EXPLAIN ANALYZE
SELECT id FROM users WHERE lower(email) = $1;
-- Seq Scan ...
-- Despite "we have an index on email!"
```
<details><summary>Hint</summary>
Index the *expression* you actually query.
</details>

### Bug 11 — `LIMIT 1` flips a hash join to a slow nested loop
```sql
EXPLAIN ANALYZE
SELECT o.* FROM orders o JOIN users u ON u.id = o.user_id
 WHERE u.country = 'NZ'
 LIMIT 1;
-- Limit
--   ->  Nested Loop
--         ->  Seq Scan on users  (Filter: country='NZ', rows=2)
--         ->  Index Scan on orders_user_id_idx  (rows=1 of 600)
-- ... 8 seconds: NZ users have no orders, the loop scans 600 NZ users.
```
<details><summary>Hint</summary>
`LIMIT n` makes the planner assume rows are easy to find — when they aren't, the loop runs forever.
</details>

### Bug 12 — `ORDER BY ... LIMIT 1` sorts the entire table
```sql
CREATE INDEX idx_orders_user ON orders (user_id);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42
 ORDER BY created_at DESC LIMIT 1;
-- Sort  (Sort Key: created_at DESC)
--   Sort Method: external merge  Disk: 412MB
--   ->  Index Scan on idx_orders_user
```
<details><summary>Hint</summary>
There is no index that lets you pull "newest first" without a sort.
</details>

### Bug 13 — `EXISTS` correlated subquery the planner can't flatten
```sql
EXPLAIN ANALYZE
SELECT u.id FROM users u
 WHERE EXISTS (
   SELECT 1 FROM orders o
    WHERE o.user_id = u.id
      AND o.total > (SELECT avg(total) FROM orders WHERE user_id = u.id)
 );
-- SubPlan re-runs avg() per outer row.
```
<details><summary>Hint</summary>
A correlated subquery referencing the outer row inside an aggregate kills decorrelation.
</details>

### Bug 14 — `IN (subquery)` vs `= ANY(array)` chosen by the wrong shape
```sql
-- Variant A
SELECT * FROM orders
 WHERE user_id IN (SELECT id FROM users WHERE country = 'NZ');
-- Hash Semi Join  -- fast

-- Variant B (app builds the list, then sends array)
SELECT * FROM orders WHERE user_id = ANY($1::int[]);
-- For tiny arrays: nested loop with index lookups -- fast
-- For 50k-element arrays: linear scan of the array on each tuple.
```
<details><summary>Hint</summary>
Array literals carry no statistics; the planner guesses a default selectivity.
</details>

### Bug 15 — Index-only scan wanted, but `UNIQUE` multicolumn index forces a heap fetch
```sql
CREATE UNIQUE INDEX uq_users_tenant_email ON users (tenant_id, email);

EXPLAIN (ANALYZE, BUFFERS)
SELECT email FROM users WHERE tenant_id = 7;
-- Index Scan ... Heap Fetches: 12,310
```
<details><summary>Hint</summary>
"Covering" requires every selected column to be in the index — and visibility-map pages.
</details>

### Bug 16 — BRIN index on randomly inserted UUIDs
```sql
CREATE TABLE events (id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
                    created_at timestamptz);
CREATE INDEX brin_events_created ON events USING brin (created_at);
-- App inserts events out-of-order via many parallel workers.

EXPLAIN ANALYZE
SELECT count(*) FROM events
 WHERE created_at >= now() - interval '1 hour';
-- Bitmap Heap Scan ... Recheck Cond ... Heap Blocks: lossy=98%
-- BRIN had to admit nearly every page.
```
<details><summary>Hint</summary>
BRIN summarizes block ranges; randomness defeats the summary.
</details>

### Bug 17 — JSONB `@>` slow because the GIN uses the default `jsonb_ops`
```sql
CREATE INDEX idx_event_payload ON events USING gin (payload);

EXPLAIN ANALYZE
SELECT * FROM events WHERE payload @> '{"user":{"id":42}}';
-- Bitmap Index Scan ... slow, large index, many false positives
```
<details><summary>Hint</summary>
There are two operator classes for `jsonb` GIN, and they are not equivalent.
</details>

### Bug 18 — Hash aggregate spilling because `work_mem` too low
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, count(*) FROM events GROUP BY user_id;
-- HashAggregate (... Disk Usage: 612 MB)
-- Buffers: temp read=78342 written=78342
```
<details><summary>Hint</summary>
Pre-PG13 hash agg never spilled (it just used too much RAM); since PG13 it spills — and that spill is what you're staring at.
</details>

### Bug 19 — Parallel scan disabled by a `VOLATILE` function
```sql
CREATE OR REPLACE FUNCTION redact(t text) RETURNS text
LANGUAGE sql AS 'SELECT regexp_replace($1, ''\d'', ''*'', ''g'')';
-- (no VOLATILITY clause => VOLATILE by default)

EXPLAIN ANALYZE
SELECT redact(body) FROM articles WHERE published;
-- Seq Scan ... no Gather node, single worker, 4× slower than expected.
```
<details><summary>Hint</summary>
Parallel-restricted functions opt the whole query out of parallelism.
</details>

### Bug 20 — Generic plan locks bad for skewed parameters
```sql
PREPARE stmt(int) AS
  SELECT * FROM orders WHERE tenant_id = $1;

EXPLAIN ANALYZE EXECUTE stmt(7);   -- tenant 7: 99% of rows
-- After 5 executions, planner switches to a generic plan
-- assuming average selectivity. Tenant 7 now uses an index scan
-- that touches 99% of pages. p99 explodes.
```
<details><summary>Hint</summary>
Custom-vs-generic-plan switch is heuristic and can be wrong for skewed data.
</details>

## 3. Senior trap (production-only failures)

### Bug 21 — PgBouncer transaction pooling silently invalidates prepared statements
```text
# Symptom in app logs (Postgres ≤ 13 era):
# ERROR: prepared statement "S_42" does not exist
#
# After moving from session to transaction pooling, every Nth request
# fails. The driver expects each connection to remember its prepares;
# PgBouncer hands you a different backend each transaction.
```
<details><summary>Hint</summary>
Transaction pooling breaks the one-connection assumption that `PREPARE` relies on.
</details>

### Bug 22 — Stats target too low for a long-tail column → catastrophic plan flip
```sql
-- column "sku" has a Zipfian distribution: a handful of SKUs dominate.
-- default statistics_target = 100 keeps only top 100 in MCV list.
SELECT * FROM order_items WHERE sku = 'rare-but-real';
-- planner estimates 500 rows (default selectivity), picks hash join.
-- actual rows: 4. Hash build of `users` was unnecessary; query 80× slower.
```
<details><summary>Hint</summary>
A single GUC controls how many "most common values" the planner remembers.
</details>

### Bug 23 — Multi-column statistics absent → planner multiplies independent estimates
```sql
-- ZIP code and city are highly correlated.
SELECT * FROM addresses WHERE zip = '1010' AND city = 'Wellington';
-- planner estimate: rows(zip) * rows(city) / total = 0.0001
-- actual: 0.01 (every Wellington address has zip 1010-ish)
-- Hash join chosen on a 1-row estimate; nested loop would have been 100×.
```
<details><summary>Hint</summary>
Postgres assumes column independence unless told otherwise.
</details>

### Bug 24 — `pg_stat_statements` says slow, hand-typed `EXPLAIN ANALYZE` says fast
```text
# pg_stat_statements:  mean_exec_time = 1240 ms for
#   SELECT * FROM orders WHERE tenant_id = $1;
# DBA reproduces:
#   EXPLAIN ANALYZE SELECT * FROM orders WHERE tenant_id = 7;
#   Execution Time: 14 ms.
#
# "It's fast on my machine."
```
<details><summary>Hint</summary>
The literal-substituted EXPLAIN takes a *custom* plan; production uses the *generic* one.
</details>

### Bug 25 — Foreign-key column unindexed → cascading delete locks for hours
```sql
-- parent: customers(id)
-- child:  invoices(customer_id REFERENCES customers ON DELETE CASCADE)
-- No index on invoices.customer_id.

DELETE FROM customers WHERE id = 999;
-- Postgres seq-scans `invoices` to find children.
-- 200M-row table, every parent delete = 30 seconds + AccessShare on invoices.
```
<details><summary>Hint</summary>
Postgres does *not* auto-index the referencing side of a foreign key.
</details>

## 4. Solutions

### Bug 1 — `LIKE '%foo'` on a B-tree index
**Root cause:** B-tree indexes are walked by binary search on a sortable prefix. `LIKE 'foo%'` constrains a prefix and works; `LIKE '%foo'` does not. With the C locale or `text_pattern_ops` opclass, `LIKE 'foo%'` works even for non-C databases. Suffix and substring search require a different access path entirely.
**Fix:**
```sql
-- Anchored prefix on a non-C locale: use *_pattern_ops
CREATE INDEX idx_users_email_pat ON users (email text_pattern_ops);
-- Suffix / substring: trigram index
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_email_trgm ON users USING gin (email gin_trgm_ops);
SELECT * FROM users WHERE email LIKE '%@example.com';
```
**Reference:** [PostgreSQL — Operator Classes and Operator Families (`text_pattern_ops`)](https://www.postgresql.org/docs/current/indexes-opclass.html) and [Markus Winand, "Use The Index, Luke" — LIKE Performance](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning)

### Bug 2 — Function on indexed column
**Root cause:** A B-tree on `email` indexes the raw value. `lower(email)` is a different value the index does not store. Postgres won't transform `lower(email) = 'x'` into `email = 'X' OR email = 'x' OR ...`.
**Fix:**
```sql
CREATE INDEX idx_users_lower_email ON users (lower(email));
-- or store normalized form and index that
ALTER TABLE users ADD COLUMN email_lc text GENERATED ALWAYS AS (lower(email)) STORED;
CREATE INDEX ON users (email_lc);
```
**Reference:** [PostgreSQL — Indexes on Expressions](https://www.postgresql.org/docs/current/indexes-expressional.html)

### Bug 3 — Implicit type cast disables the index
**Root cause:** When the column type and literal type don't match, Postgres applies an implicit cast. If the cast is on the column side (text → bigint here), the index is unusable because every row would need to be cast to compare. Some casts are pushed onto the literal (e.g., `int4` ↔ `int8`) and the index is preserved; others are not.
**Fix:**
```sql
-- Send the right type from the application:
SELECT * FROM accounts WHERE account_no = '4711';
-- Or fix the column type to what your IDs actually are.
```
**Reference:** [PostgreSQL — Operator Classes and Type Mismatches](https://www.postgresql.org/docs/current/indexes-opclass.html); [Markus Winand — "Functions in the WHERE Clause"](https://use-the-index-luke.com/sql/where-clause/functions)

### Bug 4 — Missing `ANALYZE` after bulk load
**Root cause:** `COPY` and `INSERT` update row counts in `pg_class` only via VACUUM; column histograms come from `ANALYZE`. Until then, the planner uses stale or zero-row estimates and may pick nested loops where hash joins are needed.
**Fix:**
```sql
COPY events FROM '/tmp/events.csv';
ANALYZE events;
-- For partitioned tables, ANALYZE the partitions too — autovacuum may not get to them quickly.
```
**Reference:** [PostgreSQL — Updating Planner Statistics](https://www.postgresql.org/docs/current/planner-stats.html)

### Bug 5 — Low-selectivity boolean index
**Root cause:** When a value matches most of the table, fetching pages via the index *and* visiting each heap page costs more than scanning the heap once. The planner correctly chooses seq scan; building the index was wasted work.
**Fix:**
```sql
-- Index only the rare side as a partial:
CREATE INDEX idx_users_inactive ON users (id) WHERE NOT is_active;
-- Then queries WHERE is_active = false hit the partial index.
```
**Reference:** [PostgreSQL — Partial Indexes](https://www.postgresql.org/docs/current/indexes-partial.html)

### Bug 6 — Composite index, wrong leading column
**Root cause:** A B-tree on `(status, user_id)` is sorted first by `status`, then by `user_id`. A query on `user_id` alone has no anchor and must scan every status group. Multicolumn indexes are useful only when the query filters by a leftmost prefix.
**Fix:**
```sql
CREATE INDEX idx_orders_user_status ON orders (user_id, status);
-- Or a separate index if both filter shapes are common.
```
**Reference:** [PostgreSQL — Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)

### Bug 7 — `COUNT(*)` heap fetches because visibility map is cold
**Root cause:** Index-only scans only avoid the heap when the page is marked all-visible in the visibility map. The VM is updated by VACUUM, not by INSERT or COPY, so a freshly-loaded table forces heap fetches even with a covering index.
**Fix:**
```sql
VACUUM (VERBOSE, ANALYZE) events;
-- Verify:
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM events WHERE day = '2026-05-03';
-- Heap Fetches: 0
```
**Reference:** [PostgreSQL — Index-Only Scans and Covering Indexes](https://www.postgresql.org/docs/current/indexes-index-only-scans.html)

### Bug 8 — Range column placed before equality column
**Root cause:** A B-tree walk uses index keys to narrow the range as a tree descent. Once a range predicate is applied, the index can no longer use later columns as access predicates — they degrade to filters. Equality keys should precede range keys.
**Fix:**
```sql
DROP INDEX idx_orders_created_status;
CREATE INDEX idx_orders_status_created ON orders (status, created_at);
-- Now equality on status pins the subtree; range on created_at walks within it.
```
**Reference:** [Markus Winand — "Multi-Column Indexes"](https://use-the-index-luke.com/sql/where-clause/the-equals-operator/concatenated-keys) and [PostgreSQL — Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)

### Bug 9 — Partial index whose predicate doesn't match the query
**Root cause:** Postgres can use a partial index only when it can prove the partial-index `WHERE` clause is implied by the query's `WHERE`. `status IN ('open','pending')` is not implied by `status = 'open'`. Even semantically obvious cases need exact predicate matching or a stricter query.
**Fix:**
```sql
-- Either match the predicate exactly:
SELECT id FROM orders WHERE user_id = 42 AND status = 'open';
-- Or widen the partial index to cover both states:
CREATE INDEX idx_orders_active ON orders (user_id)
  WHERE status IN ('open','pending');
```
**Reference:** [PostgreSQL — Partial Indexes ("Implication")](https://www.postgresql.org/docs/current/indexes-partial.html)

### Bug 10 — Expression mismatched with index
**Root cause:** Same root as Bug 2: the planner matches index expressions structurally. `email` indexed ≠ `lower(email)` queried.
**Fix:**
```sql
CREATE INDEX idx_users_lower_email ON users (lower(email));
-- Verify with EXPLAIN that the planner actually uses it.
```
**Reference:** [PostgreSQL — Indexes on Expressions](https://www.postgresql.org/docs/current/indexes-expressional.html)

### Bug 11 — `LIMIT 1` flips to a nested loop
**Root cause:** `LIMIT n` makes the planner believe it can stop after n matches; the cheapest start-up plan is often a nested loop driving an indexed inner. When the outer side has many rows that produce *no* inner match, the loop iterates the entire outer relation. This is a classic "fast in EXPLAIN, slow in production for empty results" regression.
**Fix:**
```sql
-- Force a hash plan when you know it's right (last resort):
SET LOCAL enable_nestloop = off;
-- Better: re-shape the query so the empty case is short-circuited
SELECT o.* FROM orders o
 WHERE o.user_id IN (SELECT id FROM users WHERE country = 'NZ')
 LIMIT 1;
-- Or use OFFSET 0 trick to fence the subplan, or pg_hint_plan as a temporary lever.
```
**Reference:** [Hubert "depesz" Lubaczewski — "LIMIT and the planner"](https://www.depesz.com/2010/09/09/why-is-my-index-not-being-used/) and [PostgreSQL — Controlling the Planner with Explicit JOIN Clauses](https://www.postgresql.org/docs/current/explicit-joins.html)

### Bug 12 — `ORDER BY` without an index match → external sort
**Root cause:** A B-tree on `user_id` does not order rows by `created_at`. Without a matching index, Postgres must read all matching rows and sort them. With many rows, the sort spills via `Sort Method: external merge`.
**Fix:**
```sql
CREATE INDEX idx_orders_user_created_desc
  ON orders (user_id, created_at DESC);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42
 ORDER BY created_at DESC LIMIT 1;
-- Index Scan Backward / Forward (no Sort node)
```
Note that `created_at DESC` direction matters for a *single* descending walk; B-trees can be walked in either direction so a plain ASC index also satisfies an ORDER BY DESC, but combined ASC/DESC ordering across multiple columns requires explicit per-column direction.
**Reference:** [PostgreSQL — Indexes and `ORDER BY`](https://www.postgresql.org/docs/current/indexes-ordering.html)

### Bug 13 — Correlated subquery the planner can't flatten
**Root cause:** Postgres can flatten many subqueries (semi-join, anti-join), but a correlated reference inside an aggregate (`avg(...) WHERE user_id = u.id`) prevents decorrelation. The plan executes the inner aggregate per outer row.
**Fix:**
```sql
-- Compute the per-user average once via a window or grouped CTE:
WITH per_user AS (
  SELECT user_id, avg(total) AS avg_total FROM orders GROUP BY user_id
)
SELECT u.id
  FROM users u
  JOIN per_user p ON p.user_id = u.id
  JOIN orders o   ON o.user_id = u.id AND o.total > p.avg_total
GROUP BY u.id;
```
**Reference:** [PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html); [Lukas Fittl — "Correlated subqueries and Postgres" (pganalyze)](https://pganalyze.com/blog)

### Bug 14 — `IN (subquery)` vs `= ANY(array)` selectivity
**Root cause:** A subquery has rows the planner can sample. An array literal is opaque; the planner uses a default selectivity (often 0.5%) and cannot reason about its contents. With huge arrays this defaults to bad plans; with tiny arrays it sometimes underestimates.
**Fix:**
```sql
-- Prefer a subquery / VALUES list when the set is "data":
SELECT * FROM orders
 WHERE user_id IN (SELECT id FROM users WHERE country = 'NZ');
-- Or feed arrays via a temp table when they're large:
CREATE TEMP TABLE wanted_users(id int) ON COMMIT DROP;
COPY wanted_users FROM STDIN;
ANALYZE wanted_users;
SELECT * FROM orders o JOIN wanted_users w USING (id);
```
**Reference:** [PostgreSQL — Row and Array Comparisons](https://www.postgresql.org/docs/current/functions-comparisons.html); [PostgreSQL — Planner Statistics Details](https://www.postgresql.org/docs/current/planner-stats-details.html)

### Bug 15 — Multicolumn `UNIQUE` defeats covering index goal
**Root cause:** The unique index has `(tenant_id, email)`, which does cover `email` — but the planner needs *all selected columns* in the index *and* the relevant pages all-visible. After updates, the visibility map is partially cold and the heap fetch is unavoidable. Sometimes the issue is just that the index isn't laid out as the planner needs (e.g., column order vs. selected columns).
**Fix:**
```sql
-- Use INCLUDE to add covered (non-key) columns explicitly when needed:
CREATE INDEX idx_users_tenant_inc_email
  ON users (tenant_id) INCLUDE (email);
-- Then VACUUM to refresh the visibility map.
VACUUM ANALYZE users;
```
**Reference:** [PostgreSQL — Index-Only Scans and Covering Indexes (`INCLUDE`)](https://www.postgresql.org/docs/current/indexes-index-only-scans.html)

### Bug 16 — BRIN on randomly-inserted data
**Root cause:** BRIN stores per-block-range summaries (min/max). Effectiveness depends on physical correlation between the column value and storage order. Random insertion order destroys correlation; nearly every range overlaps the query and BRIN returns lossy block lists that cover most of the heap.
**Fix:**
```sql
-- If the data is genuinely random, drop BRIN; use B-tree:
DROP INDEX brin_events_created;
CREATE INDEX idx_events_created_at ON events (created_at);

-- If you control insert order, partition by time so BRIN is correlated within partition:
CREATE TABLE events (...) PARTITION BY RANGE (created_at);
-- And/or CLUSTER on the relevant column periodically.
```
**Reference:** [PostgreSQL — BRIN Indexes (Introduction)](https://www.postgresql.org/docs/current/brin.html)

### Bug 17 — JSONB GIN with default operator class
**Root cause:** GIN on `jsonb` has two operator classes. The default `jsonb_ops` indexes every key and value separately and supports many operators (`?`, `?&`, `?|`, `@>`); `jsonb_path_ops` indexes only the hashed path-and-value, supports only `@>`, but is much smaller and faster for containment.
**Fix:**
```sql
DROP INDEX idx_event_payload;
CREATE INDEX idx_event_payload ON events
  USING gin (payload jsonb_path_ops);
-- @> queries get small index, fewer false positives, smaller writes.
-- If you also need ? / ?| / ?& operators, keep jsonb_ops.
```
**Reference:** [PostgreSQL — GIN: Built-in Operator Classes](https://www.postgresql.org/docs/current/gin.html)

### Bug 18 — Hash aggregate spilling
**Root cause:** Since PG 13, `HashAggregate` spills to disk when its memory exceeds `work_mem` (previously it just exceeded RAM and risked OOM). Spill is correct but slow; the fix is right-sizing `work_mem` for the workload, or rewriting the aggregation.
**Fix:**
```sql
-- Per-session lift for big rollups (work_mem is per-sort/hash node, per-process):
SET LOCAL work_mem = '256MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, count(*) FROM events GROUP BY user_id;
-- Or pre-aggregate with a rollup table / continuous aggregate.
```
**Reference:** [PostgreSQL — `work_mem`](https://www.postgresql.org/docs/current/runtime-config-resource.html#GUC-WORK-MEM); [Tomas Vondra — "Hash Aggregate spill in PostgreSQL 13"](https://vondra.me/posts/hash-aggregate-spill-in-postgresql-13/)

### Bug 19 — Volatile function disables parallelism
**Root cause:** Parallel query is enabled per node; functions default to `VOLATILE` unless declared otherwise. A volatile function in the target list or `WHERE` can mark the whole plan parallel-restricted or parallel-unsafe.
**Fix:**
```sql
CREATE OR REPLACE FUNCTION redact(t text) RETURNS text
LANGUAGE sql IMMUTABLE PARALLEL SAFE AS
  'SELECT regexp_replace($1, ''\d'', ''*'', ''g'')';
-- Re-EXPLAIN; you should now see Gather / Parallel Seq Scan.
```
**Reference:** [PostgreSQL — Parallel Safety](https://www.postgresql.org/docs/current/parallel-safety.html); [PostgreSQL — `CREATE FUNCTION` (volatility, PARALLEL)](https://www.postgresql.org/docs/current/sql-createfunction.html)

### Bug 20 — Generic plan locked in for skewed parameters
**Root cause:** Postgres builds custom plans for the first few executions of a prepared statement, then compares the average custom-plan cost to the generic-plan cost; if generic is competitive, it switches. For data with skewed selectivity per parameter, the generic plan is wrong for some parameters even though it's right "on average."
**Fix:**
```sql
-- Force custom plans:
SET plan_cache_mode = force_custom_plan;
EXECUTE stmt(7);
-- Or force generic, or leave at 'auto' and verify per-tenant in
-- pg_stat_statements + auto_explain.
```
**Reference:** [PostgreSQL — `plan_cache_mode`](https://www.postgresql.org/docs/current/runtime-config-query.html#GUC-PLAN-CACHE-MODE) and [PostgreSQL — PL/pgSQL Plan Caching](https://www.postgresql.org/docs/current/plpgsql-implementation.html#PLPGSQL-PLAN-CACHING)

### Bug 21 — PgBouncer transaction pooling vs prepared statements
**Root cause:** Server-side prepared statements (`PREPARE` / extended protocol with named statements) live on a specific backend. In PgBouncer transaction pooling, the client sees a logical connection but transactions can be served by any underlying server connection; a `PREPARE` on backend A is invisible when the next transaction lands on backend B. PgBouncer 1.21+ adds protocol-level prepared statement support; earlier versions force you to disable named prepares in the driver.
**Fix:**
```text
# Driver options:
#   - PostgreSQL JDBC: prepareThreshold=0  (use unnamed/Parse each time)
#   - psycopg3:        prepare_threshold=None
#   - node-postgres:   don't pass `name` on Query
#
# Or upgrade to PgBouncer ≥ 1.21 with `max_prepared_statements > 0`.
```
**Reference:** [PgBouncer — Prepared Statements](https://www.pgbouncer.org/config.html#max_prepared_statements) and [PgBouncer 1.21 release notes](https://github.com/pgbouncer/pgbouncer/blob/master/NEWS.md)

### Bug 22 — `default_statistics_target` too low for skewed columns
**Root cause:** `pg_statistic` keeps only the top-N most common values per column (default 100). A column with thousands of distinct values and a long tail loses most of its distribution; the planner falls back to the assumed selectivity for "not in MCV", which is often wildly off.
**Fix:**
```sql
ALTER TABLE order_items ALTER COLUMN sku SET STATISTICS 1000;
ANALYZE order_items;
-- Verify:
SELECT most_common_vals, most_common_freqs
FROM pg_stats WHERE tablename='order_items' AND attname='sku';
```
**Reference:** [PostgreSQL — Planner Statistics Details](https://www.postgresql.org/docs/current/planner-stats-details.html); [PostgreSQL — `default_statistics_target`](https://www.postgresql.org/docs/current/runtime-config-query.html#GUC-DEFAULT-STATISTICS-TARGET)

### Bug 23 — Column independence assumption
**Root cause:** Without extended statistics, the planner multiplies single-column selectivities, assuming independence. For correlated columns (zip ↔ city, country ↔ language), the multiplied estimate is far too small. A 1-row estimate often selects a nested loop where a hash join would be right by an order of magnitude.
**Fix:**
```sql
CREATE STATISTICS stx_addresses_zip_city (dependencies, ndistinct, mcv)
  ON zip, city FROM addresses;
ANALYZE addresses;
-- Re-run EXPLAIN; planner now uses functional dependencies.
```
**Reference:** [PostgreSQL — Extended Statistics (`CREATE STATISTICS`)](https://www.postgresql.org/docs/current/sql-createstatistics.html); [Tomas Vondra — "Multivariate statistics in PostgreSQL"](https://vondra.me/posts/multi-column-statistics-in-postgresql-10/)

### Bug 24 — `pg_stat_statements` slow, hand-typed `EXPLAIN ANALYZE` fast
**Root cause:** `pg_stat_statements` aggregates server-side prepared/protocol-bound queries that go through generic plans after the custom-plan threshold. Pasting the SQL with literal values into psql gives you a custom plan with the literal-aware selectivity. The "fast" reproduction is therefore not the same plan production runs.
**Fix:**
```sql
-- Capture the actual plan from production:
LOAD 'auto_explain';
SET auto_explain.log_min_duration = '500ms';
SET auto_explain.log_analyze = on;
SET auto_explain.log_nested_statements = on;
-- Or simulate generic plan locally:
SET plan_cache_mode = force_generic_plan;
PREPARE p(int) AS SELECT * FROM orders WHERE tenant_id = $1;
EXPLAIN (ANALYZE, BUFFERS) EXECUTE p(7);
```
**Reference:** [PostgreSQL — `auto_explain`](https://www.postgresql.org/docs/current/auto-explain.html); [PostgreSQL — `plan_cache_mode`](https://www.postgresql.org/docs/current/runtime-config-query.html#GUC-PLAN-CACHE-MODE)

### Bug 25 — Unindexed FK column → cascading delete walks the child
**Root cause:** A primary-key index exists on the parent automatically, but the *referencing* column on the child is not indexed unless you create the index. Cascading delete (or any FK-driven check) seq-scans the child for matching rows; on a large child table this dominates DELETE latency and inflates lock duration.
**Fix:**
```sql
CREATE INDEX CONCURRENTLY idx_invoices_customer_id
  ON invoices (customer_id);
-- Audit all FKs:
SELECT conname, conrelid::regclass AS child, a.attname AS fk_col
FROM pg_constraint c
JOIN pg_attribute a ON a.attrelid = c.conrelid AND a.attnum = ANY(c.conkey)
WHERE c.contype = 'f'
  AND NOT EXISTS (
    SELECT 1 FROM pg_index i
    WHERE i.indrelid = c.conrelid
      AND (i.indkey::int[])[0] = a.attnum
  );
```
**Reference:** [PostgreSQL — `CREATE TABLE`: Foreign Key (Note on indexing)](https://www.postgresql.org/docs/current/sql-createtable.html); [Markus Winand — "Indexes on Foreign Keys"](https://use-the-index-luke.com/sql/foreign-key)

## Related
- [EXPLAIN ANALYZE guide](explain-analyze-guide.md)
- [Optimization patterns](optimization-patterns.md)
- [Indexing strategies](../design/indexing-strategies.md)
- [Indexing internals](../engine-internals/indexing-internals.md)
- [MVCC bug spotting](../engine-internals/mvcc-bug-spotting.md)

## References
- Official docs: PostgreSQL manual — Using EXPLAIN (postgresql.org/docs/current/using-explain.html), Updating Planner Statistics (postgresql.org/docs/current/planner-stats.html), Planner Stats Details (postgresql.org/docs/current/planner-stats-details.html), Index Types (postgresql.org/docs/current/indexes-types.html), Multicolumn Indexes (postgresql.org/docs/current/indexes-multicolumn.html), Partial Indexes (postgresql.org/docs/current/indexes-partial.html), Indexes on Expressions (postgresql.org/docs/current/indexes-expressional.html), Index-Only Scans (postgresql.org/docs/current/indexes-index-only-scans.html), GIN (postgresql.org/docs/current/gin.html), BRIN (postgresql.org/docs/current/brin.html), `plan_cache_mode` (postgresql.org/docs/current/runtime-config-query.html), `auto_explain` (postgresql.org/docs/current/auto-explain.html), Extended Statistics (postgresql.org/docs/current/sql-createstatistics.html), Parallel Safety (postgresql.org/docs/current/parallel-safety.html)
- Authority blogs: [Markus Winand — "Use The Index, Luke"](https://use-the-index-luke.com/), [Hubert "depesz" Lubaczewski](https://www.depesz.com/), [Lukas Fittl / pganalyze](https://pganalyze.com/blog), [Tomas Vondra](https://vondra.me/), [Robert Haas — rhaas.blogspot.com](https://rhaas.blogspot.com/)
- Tooling: [PgBouncer prepared-statement support — pgbouncer.org/config.html](https://www.pgbouncer.org/config.html)
