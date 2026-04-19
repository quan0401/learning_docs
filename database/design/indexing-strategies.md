---
title: "Indexing Strategies for PostgreSQL"
date: 2026-04-19
updated: 2026-04-19
tags: [database, indexing, postgresql, performance, schema-design]
---

# Indexing Strategies for PostgreSQL

**Date:** 2026-04-19
**Tags:** `database` `indexing` `postgresql` `performance` `schema-design`

---

## Table of Contents

- [Summary](#summary)
- [Multi-Column Indexes and the Leftmost Prefix Rule](#multi-column-indexes-and-the-leftmost-prefix-rule)
- [Covering Indexes with INCLUDE](#covering-indexes-with-include)
- [Partial Indexes](#partial-indexes)
- [Expression Indexes](#expression-indexes)
- [Index-Only Scans](#index-only-scans)
- [GIN Indexes for JSONB](#gin-indexes-for-jsonb)
- [Full-Text Search Indexes](#full-text-search-indexes)
- [Over-Indexing: The Hidden Cost](#over-indexing-the-hidden-cost)
- [Index Maintenance](#index-maintenance)
- [Decision Flowchart: Which Index Type?](#decision-flowchart-which-index-type)
- [Related](#related)
- [References](#references)

---

## Summary

Indexes are the primary tool for read performance, but they are not free -- every index slows writes, consumes storage, and requires maintenance. This doc covers practical PostgreSQL indexing strategies: multi-column ordering, covering indexes, partial and expression indexes, GIN for JSONB and full-text search, and how to detect and remove unused indexes.

---

## Multi-Column Indexes and the Leftmost Prefix Rule

A B-tree index on `(a, b, c)` is usable for queries that filter on:
- `a`
- `a, b`
- `a, b, c`

It is **not** usable for queries filtering only on `b` or `c` without `a`.

```sql
-- This index supports all three queries below
CREATE INDEX idx_orders_status_date_customer
    ON orders (status, order_date, customer_id);

-- Uses the index (leftmost prefix: status)
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'SHIPPED';

-- Uses the index (prefix: status + order_date)
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'SHIPPED' AND order_date > '2026-01-01';

-- Does NOT use the index (skips status)
EXPLAIN ANALYZE
SELECT * FROM orders WHERE order_date > '2026-01-01';
```

**Column ordering heuristic:**
1. Equality predicates first (highest selectivity)
2. Range predicates second
3. Columns used only in ORDER BY last

```sql
-- For: WHERE status = ? AND order_date BETWEEN ? AND ? ORDER BY customer_id
-- Optimal order: (status, order_date, customer_id)
CREATE INDEX idx_orders_optimal
    ON orders (status, order_date, customer_id);
```

---

## Covering Indexes with INCLUDE

PostgreSQL 11+ lets you add non-key columns to an index with `INCLUDE`. These columns are stored in the index leaf pages but are not part of the B-tree search structure.

```sql
-- Without INCLUDE: index lookup -> heap fetch for name and email
CREATE INDEX idx_users_status ON users (status);

-- With INCLUDE: index-only scan can return name and email directly
CREATE INDEX idx_users_status_covering
    ON users (status)
    INCLUDE (name, email);
```

```sql
-- This query can now be satisfied with an index-only scan
EXPLAIN ANALYZE
SELECT name, email FROM users WHERE status = 'ACTIVE';
```

**When to use INCLUDE:**
- The query always selects a small, known set of columns alongside the filter
- You want index-only scans without bloating the B-tree keys
- The included columns are not used in WHERE or ORDER BY

---

## Partial Indexes

A partial index covers only rows matching a `WHERE` clause. Smaller index = faster lookups + less storage + less write overhead.

```sql
-- Only 2% of orders are unshipped at any time, but that's the hot query
CREATE INDEX idx_orders_unshipped
    ON orders (order_date, customer_id)
    WHERE status != 'SHIPPED' AND status != 'CANCELLED';
```

```sql
-- The planner uses the partial index when the query predicate matches
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE status = 'PENDING'
  AND order_date > CURRENT_DATE - INTERVAL '7 days';
```

**Common use cases:**
- Active/pending rows in a status column (`WHERE deleted_at IS NULL`)
- Unprocessed queue items (`WHERE processed = false`)
- Unique constraints on a subset (`CREATE UNIQUE INDEX ... WHERE active = true`)

```sql
-- Unique email only among non-deleted users
CREATE UNIQUE INDEX idx_users_email_active
    ON users (email)
    WHERE deleted_at IS NULL;
```

---

## Expression Indexes

Index the result of a function or expression, not just a raw column.

```sql
-- Case-insensitive email lookup
CREATE INDEX idx_users_email_lower ON users (LOWER(email));

SELECT * FROM users WHERE LOWER(email) = 'john@example.com';
```

```sql
-- Index on date truncation for time-series grouping
CREATE INDEX idx_events_day ON events (DATE_TRUNC('day', created_at));

SELECT DATE_TRUNC('day', created_at) AS day, COUNT(*)
FROM events
WHERE DATE_TRUNC('day', created_at) = '2026-04-19'
GROUP BY day;
```

```sql
-- Index on a JSONB path
CREATE INDEX idx_products_brand ON products ((attributes->>'brand'));

SELECT * FROM products WHERE attributes->>'brand' = 'Apple';
```

> The expression in the query **must match** the index expression exactly. `WHERE LOWER(email)` works with the index, but `WHERE email ILIKE` does not.

---

## Index-Only Scans

An index-only scan reads data directly from the index without visiting the heap (table). This is the fastest scan type but requires two conditions:

1. **All returned columns** are in the index (key columns + INCLUDE columns).
2. **The visibility map** confirms all tuples on the relevant pages are all-visible (no recent uncommitted changes).

```sql
-- Check if a query uses an index-only scan
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, order_date
FROM orders
WHERE status = 'SHIPPED';
```

Look for `Index Only Scan` in the plan. If you see `Heap Fetches: 12345`, the visibility map was not fully up to date.

```sql
-- Force a visibility map refresh
VACUUM orders;
```

**Why index-only scans fail:**
- Heavy UPDATE/DELETE workload means many pages are not all-visible
- VACUUM has not run recently
- The query SELECTs a column not in the index

---

## GIN Indexes for JSONB

GIN (Generalized Inverted Index) supports the containment and existence operators for JSONB.

```sql
-- Default GIN: supports @>, ?, ?|, ?&
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);

-- jsonb_path_ops GIN: supports only @> but is smaller and faster
CREATE INDEX idx_products_attrs_pathops
    ON products USING GIN (attributes jsonb_path_ops);
```

**Operator support:**

| Operator | Meaning                            | Default GIN | jsonb_path_ops |
|----------|------------------------------------|-------------|----------------|
| `@>`     | Contains                           | Yes         | Yes            |
| `?`      | Key exists                         | Yes         | No             |
| `?|`     | Any key exists                     | Yes         | No             |
| `?&`     | All keys exist                     | Yes         | No             |

```sql
-- Containment: products where attributes contain {"brand": "Apple"}
SELECT * FROM products WHERE attributes @> '{"brand": "Apple"}';

-- Key existence: products that have a "warranty" key
SELECT * FROM products WHERE attributes ? 'warranty';

-- Any key exists: products with "color" or "size" attribute
SELECT * FROM products WHERE attributes ?| ARRAY['color', 'size'];
```

**When to use `jsonb_path_ops`:** when you only need `@>` containment queries. It produces a ~30% smaller index.

---

## Full-Text Search Indexes

PostgreSQL has built-in full-text search via `tsvector` and `tsquery`.

```sql
-- Add a tsvector column (or use a generated column in PG12+)
ALTER TABLE articles ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(body, '')), 'B')
    ) STORED;

-- GIN index for full-text search
CREATE INDEX idx_articles_fts ON articles USING GIN (search_vector);
```

```sql
-- Search with ranking
SELECT
    title,
    ts_rank(search_vector, query) AS rank
FROM
    articles,
    to_tsquery('english', 'postgresql & indexing') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

**GIN vs GiST for FTS:**

| Aspect        | GIN                          | GiST                        |
|---------------|------------------------------|------------------------------|
| Build time    | Slower                       | Faster                       |
| Query speed   | Faster (exact posting lists) | Slower (lossy, recheck)      |
| Update cost   | Higher                       | Lower                        |
| Best for      | Read-heavy, exact search     | Write-heavy, proximity search|

Use GIN for most FTS workloads. Use GiST only when build time matters or you need proximity operators like `<->`.

---

## Over-Indexing: The Hidden Cost

Every index adds:
- **Write amplification:** each INSERT/UPDATE/DELETE must update every relevant index
- **Storage:** indexes can easily double the total table size
- **VACUUM overhead:** dead index tuples must be cleaned
- **Planner overhead:** more indexes = more options for the planner to evaluate

```sql
-- Check index sizes for a table
SELECT
    indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'orders'
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Rule of thumb:** if a table has more indexes than columns, audit for redundancy.

---

## Index Maintenance

### Detecting Unused Indexes

```sql
-- Indexes with zero scans since last stats reset
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'  -- keep primary keys
ORDER BY pg_relation_size(indexrelid) DESC;
```

> Wait at least one full business cycle (1-2 weeks minimum, ideally a month) before dropping an unused index. Some indexes are only hit by monthly batch jobs or reports.

### Rebuilding Bloated Indexes

```sql
-- REINDEX CONCURRENTLY (PG12+) avoids blocking writes
REINDEX INDEX CONCURRENTLY idx_orders_status;

-- Or rebuild all indexes on a table
REINDEX TABLE CONCURRENTLY orders;
```

### Monitoring Index Health

```sql
-- Index bloat estimation (simplified)
SELECT
    nspname AS schema,
    relname AS index_name,
    round(100 * pg_relation_size(indexrelid) /
        NULLIF(pg_relation_size(indrelid), 0)) AS index_to_table_pct
FROM pg_stat_user_indexes
JOIN pg_index USING (indexrelid)
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;
```

---

## Decision Flowchart: Which Index Type?

```mermaid
flowchart TD
    A[What is the query pattern?] --> B{Equality / range on scalar columns?}
    B -->|Yes| C[B-tree index]
    B -->|No| D{JSONB containment or key existence?}

    D -->|Only @>| E[GIN jsonb_path_ops]
    D -->|@> + ? + ?| + ?&| F[GIN default]
    D -->|No| G{Full-text search?}

    G -->|Yes, read-heavy| H[GIN on tsvector]
    G -->|Yes, write-heavy| I[GiST on tsvector]
    G -->|No| J{Array containment or overlap?}

    J -->|Yes| K[GIN on array column]
    J -->|No| L{Geometric / spatial?}

    L -->|Yes| M[GiST or SP-GiST]
    L -->|No| N{Nearest-neighbor on high-dim vector?}

    N -->|Yes| O[HNSW or IVFFlat via pgvector]
    N -->|No| P[Re-evaluate: do you need an index?]

    C --> Q{Only a subset of rows queried?}
    Q -->|Yes| R[Partial index with WHERE clause]
    Q -->|No| S{Need index-only scans?}
    S -->|Yes| T[Add INCLUDE columns]
    S -->|No| U[Standard B-tree]
```

---

## Related

- [normalization-and-tradeoffs.md](normalization-and-tradeoffs.md) -- schema design directly determines which indexes are viable
- [partitioning-and-sharding.md](partitioning-and-sharding.md) -- partitioning interacts with index strategies (each partition gets its own indexes)
- [data-modeling-patterns.md](data-modeling-patterns.md) -- patterns like soft deletes and JSONB attributes need specific index types

---

## References

- [PostgreSQL Documentation: Indexes](https://www.postgresql.org/docs/current/indexes.html)
- [PostgreSQL Documentation: Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [PostgreSQL Documentation: Partial Indexes](https://www.postgresql.org/docs/current/indexes-partial.html)
- [PostgreSQL Documentation: Expression Indexes](https://www.postgresql.org/docs/current/indexes-expressional.html)
- [PostgreSQL Documentation: GIN Indexes](https://www.postgresql.org/docs/current/gin.html)
- [PostgreSQL Documentation: Full Text Search](https://www.postgresql.org/docs/current/textsearch.html)
- [PostgreSQL Documentation: REINDEX](https://www.postgresql.org/docs/current/sql-reindex.html)
- [PostgreSQL Wiki: Index Maintenance](https://wiki.postgresql.org/wiki/Index_Maintenance)
