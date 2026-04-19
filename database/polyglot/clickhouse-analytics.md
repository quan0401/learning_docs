---
title: "ClickHouse for Real-Time Analytics"
date: 2026-04-19
updated: 2026-04-19
tags: [clickhouse, analytics, columnar, olap, real-time, polyglot]
---

# ClickHouse for Real-Time Analytics

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `clickhouse` `analytics` `columnar` `olap` `real-time` `polyglot`

## Table of Contents

- [Summary](#summary)
- [Architecture: Why Columns Beat Rows](#architecture-why-columns-beat-rows)
- [MergeTree Engine Family](#mergetree-engine-family)
  - [MergeTree](#mergetree)
  - [ReplacingMergeTree](#replacingmergetree)
  - [AggregatingMergeTree](#aggregatingmergetree)
  - [SummingMergeTree](#summingmergetree)
- [Primary Key and Ordering Key](#primary-key-and-ordering-key)
- [Materialized Views](#materialized-views)
- [Data Ingestion](#data-ingestion)
- [Query Patterns](#query-patterns)
- [Dictionaries](#dictionaries)
- [Distributed Tables](#distributed-tables)
- [ClickHouse vs PostgreSQL](#clickhouse-vs-postgresql)
- [Integration Patterns](#integration-patterns)
- [Operational Concerns](#operational-concerns)
- [Related](#related)
- [References](#references)

## Summary

ClickHouse is a column-oriented OLAP database designed for sub-second analytical queries over billions of rows. It achieves extreme read performance through columnar storage, vectorized execution, and aggressive compression. This document covers when to use it, how to model data for it, and how to integrate it alongside a transactional database.

## Architecture: Why Columns Beat Rows

```mermaid
flowchart LR
    subgraph "Row-Oriented (PostgreSQL)"
        R1["Row 1: id=1, name='Alice', age=30, city='NYC'"]
        R2["Row 2: id=2, name='Bob', age=25, city='LA'"]
        R3["Row 3: id=3, name='Charlie', age=35, city='NYC'"]
    end

    subgraph "Column-Oriented (ClickHouse)"
        C1["id:    [1, 2, 3]"]
        C2["name:  ['Alice', 'Bob', 'Charlie']"]
        C3["age:   [30, 25, 35]"]
        C4["city:  ['NYC', 'LA', 'NYC']"]
    end
```

**Why this matters for analytics**:

| Factor | Row Store | Column Store |
|---|---|---|
| `SELECT avg(age) FROM users` | Reads all columns for every row | Reads only the `age` column |
| Compression | Mixed types per block, poor ratio | Same-type data compresses 10-30x |
| Vectorized execution | Row-at-a-time processing | SIMD on column batches |
| Point lookups (WHERE id = 1) | Fast (one read per row) | Slow (must reconstruct row) |
| Full table scans for aggregation | Slow | Extremely fast |

**Key insight**: ClickHouse is not a replacement for PostgreSQL. It is a complement for read-heavy analytical queries that would cripple an OLTP database.

## MergeTree Engine Family

### MergeTree

The foundational engine. Data is written in parts (immutable sorted batches) that are periodically merged in the background.

```sql
CREATE TABLE events (
    event_id     UUID DEFAULT generateUUIDv4(),
    event_type   LowCardinality(String),
    user_id      UInt64,
    properties   Map(String, String),
    amount       Decimal64(2),
    created_at   DateTime64(3, 'UTC')
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (event_type, user_id, created_at)
TTL created_at + INTERVAL 1 YEAR DELETE
SETTINGS index_granularity = 8192;
```

### ReplacingMergeTree

Deduplicates rows with the same `ORDER BY` key during background merges. Useful for CDC ingestion where the same row may arrive multiple times.

```sql
CREATE TABLE products (
    product_id   UInt64,
    name         String,
    price        Decimal64(2),
    updated_at   DateTime64(3, 'UTC'),
    _version     UInt64
) ENGINE = ReplacingMergeTree(_version)
ORDER BY product_id;

-- Query with deduplication forced at read time:
SELECT * FROM products FINAL WHERE product_id = 1001;
```

**Caveat**: `FINAL` forces deduplication at query time (slower). Without it, you may see duplicate rows until the next background merge.

### AggregatingMergeTree

Stores pre-aggregated state using `-State` aggregate functions. Used as the target for materialized views.

```sql
CREATE TABLE hourly_revenue (
    hour         DateTime,
    product_id   UInt64,
    revenue      AggregateFunction(sum, Decimal64(2)),
    order_count  AggregateFunction(count, UInt64)
) ENGINE = AggregatingMergeTree()
ORDER BY (hour, product_id);
```

### SummingMergeTree

Automatically sums numeric columns for rows with the same `ORDER BY` key during merges.

```sql
CREATE TABLE daily_counters (
    date         Date,
    page_url     String,
    views        UInt64,
    unique_users UInt64
) ENGINE = SummingMergeTree((views, unique_users))
ORDER BY (date, page_url);

-- Inserting incremental counts
INSERT INTO daily_counters VALUES ('2026-04-19', '/home', 150, 80);
INSERT INTO daily_counters VALUES ('2026-04-19', '/home', 200, 120);
-- After merge: views = 350, unique_users = 200
```

## Primary Key and Ordering Key

In ClickHouse, the `ORDER BY` clause defines how data is physically sorted on disk. The primary key (by default, the `ORDER BY` key) creates a sparse index used for data pruning.

```text
ORDER BY (event_type, user_id, created_at)

Data on disk (sorted):
Part 1: [click, 1001, 2026-04-01] ... [click, 5000, 2026-04-15]
Part 2: [click, 5001, 2026-04-01] ... [purchase, 2000, 2026-04-15]
Part 3: [purchase, 2001, 2026-04-01] ... [view, 9000, 2026-04-15]

Sparse index (every 8192 rows):
Mark 0: event_type='click', user_id=1001, created_at=2026-04-01
Mark 1: event_type='click', user_id=3500, created_at=2026-04-08
Mark 2: event_type='click', user_id=5001, created_at=2026-04-01
...
```

**Query**: `WHERE event_type = 'purchase' AND user_id = 1500`
- The sparse index narrows to the relevant granules (marks).
- Only a fraction of data parts are read.

**Rule**: Place columns used in `WHERE` clauses first in `ORDER BY`, ordered by cardinality (lowest first).

## Materialized Views

Materialized views in ClickHouse act as real-time triggers that transform data on INSERT. They are not periodic refreshes like in PostgreSQL.

```sql
-- Source table (raw events)
CREATE TABLE raw_orders (
    order_id     UInt64,
    product_id   UInt64,
    amount       Decimal64(2),
    created_at   DateTime64(3, 'UTC')
) ENGINE = MergeTree()
ORDER BY (created_at, order_id);

-- Target table (pre-aggregated)
CREATE TABLE hourly_revenue (
    hour         DateTime,
    product_id   UInt64,
    revenue      AggregateFunction(sum, Decimal64(2)),
    order_count  AggregateFunction(count, UInt64)
) ENGINE = AggregatingMergeTree()
ORDER BY (hour, product_id);

-- Materialized view (triggers on INSERT to raw_orders)
CREATE MATERIALIZED VIEW mv_hourly_revenue
TO hourly_revenue AS
SELECT
    toStartOfHour(created_at) AS hour,
    product_id,
    sumState(amount) AS revenue,
    countState(order_id) AS order_count
FROM raw_orders
GROUP BY hour, product_id;

-- Query the pre-aggregated data
SELECT
    hour,
    product_id,
    sumMerge(revenue) AS total_revenue,
    countMerge(order_count) AS total_orders
FROM hourly_revenue
WHERE hour >= '2026-04-01'
GROUP BY hour, product_id
ORDER BY total_revenue DESC;
```

### Cascading Materialized Views

Chain multiple views for multi-level aggregation:

```text
raw_orders -> mv_hourly_revenue -> mv_daily_revenue -> mv_monthly_revenue
```

## Data Ingestion

### Batch Inserts

ClickHouse is optimized for bulk inserts. Never insert row by row.

```sql
-- Insert from a SELECT (internal ETL)
INSERT INTO analytics.events
SELECT * FROM staging.raw_events
WHERE created_at >= '2026-04-19';
```

**Best practice**: Batch at least 1000 rows per INSERT, ideally 10,000-100,000. Each INSERT creates a new data part; too many small parts degrades performance.

### Kafka Engine

```sql
CREATE TABLE events_kafka (
    event_type   String,
    user_id      UInt64,
    properties   String,
    created_at   DateTime64(3, 'UTC')
) ENGINE = Kafka()
SETTINGS
    kafka_broker_list = 'kafka-1:9092,kafka-2:9092',
    kafka_topic_list = 'events',
    kafka_group_name = 'clickhouse-consumer',
    kafka_format = 'JSONEachRow';

-- Materialized view to move Kafka data into MergeTree
CREATE MATERIALIZED VIEW mv_events_kafka
TO events AS
SELECT * FROM events_kafka;
```

### S3 Integration

```sql
-- Read directly from S3
SELECT count(*)
FROM s3(
    'https://bucket.s3.amazonaws.com/data/2026/04/*.parquet',
    'AKIAIOSFODNN7EXAMPLE',
    'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY',
    'Parquet'
);

-- S3 as external table engine
CREATE TABLE s3_archive (
    event_id UUID,
    event_type String,
    created_at DateTime
) ENGINE = S3('https://bucket.s3.amazonaws.com/archive/', 'Parquet');
```

## Query Patterns

### GROUP BY with High Cardinality

```sql
-- Top 100 pages by unique visitors in the last 7 days
SELECT
    page_url,
    uniq(user_id) AS unique_visitors,
    count() AS total_views,
    avg(duration_ms) AS avg_duration
FROM page_views
WHERE created_at >= now() - INTERVAL 7 DAY
GROUP BY page_url
ORDER BY unique_visitors DESC
LIMIT 100;
```

### Approximate Functions

ClickHouse provides fast approximate aggregates for large datasets:

```sql
-- Approximate unique count (HyperLogLog, ~1.6% error)
SELECT uniq(user_id) FROM events;

-- Exact unique count (slower)
SELECT uniqExact(user_id) FROM events;

-- Approximate quantiles (t-digest)
SELECT
    quantile(0.50)(response_time_ms) AS p50,
    quantile(0.95)(response_time_ms) AS p95,
    quantile(0.99)(response_time_ms) AS p99
FROM api_requests
WHERE created_at >= now() - INTERVAL 1 HOUR;
```

### Array Functions

```sql
SELECT
    user_id,
    groupArray(event_type) AS event_sequence,
    arrayDistinct(groupArray(event_type)) AS unique_events,
    length(groupArray(event_type)) AS event_count
FROM events
WHERE created_at >= today()
GROUP BY user_id
HAVING has(groupArray(event_type), 'purchase');
```

## Dictionaries

External lookup tables loaded into memory for JOIN-free enrichment at query time.

```sql
CREATE DICTIONARY product_dict (
    product_id UInt64,
    name       String,
    category   String,
    brand      String
)
PRIMARY KEY product_id
SOURCE(CLICKHOUSE(TABLE 'products' DB 'default'))
LAYOUT(HASHED())
LIFETIME(MIN 300 MAX 600);

-- Use in queries (zero-cost lookup)
SELECT
    event_type,
    dictGet('product_dict', 'category', product_id) AS category,
    count() AS events
FROM product_events
GROUP BY event_type, category
ORDER BY events DESC;
```

## Distributed Tables

```mermaid
flowchart TB
    subgraph "ClickHouse Cluster"
        D["Distributed Table<br/>(virtual, query router)"]

        subgraph "Shard 1"
            S1R1["Replica 1<br/>(local MergeTree)"]
            S1R2["Replica 2<br/>(local MergeTree)"]
        end

        subgraph "Shard 2"
            S2R1["Replica 1"]
            S2R2["Replica 2"]
        end

        subgraph "Shard 3"
            S3R1["Replica 1"]
            S3R2["Replica 2"]
        end
    end

    D --> S1R1
    D --> S2R1
    D --> S3R1
    S1R1 <-->|ZooKeeper| S1R2
    S2R1 <-->|ZooKeeper| S2R2
    S3R1 <-->|ZooKeeper| S3R2
```

```sql
-- Local table on each shard
CREATE TABLE events_local ON CLUSTER 'analytics_cluster' (
    event_id     UUID,
    event_type   LowCardinality(String),
    user_id      UInt64,
    created_at   DateTime64(3, 'UTC')
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
ORDER BY (event_type, user_id, created_at)
PARTITION BY toYYYYMM(created_at);

-- Distributed table for querying across shards
CREATE TABLE events ON CLUSTER 'analytics_cluster' AS events_local
ENGINE = Distributed('analytics_cluster', 'default', 'events_local', rand());
```

## ClickHouse vs PostgreSQL

| Aspect | PostgreSQL | ClickHouse |
|---|---|---|
| Storage model | Row-oriented | Column-oriented |
| Primary use | OLTP (transactions) | OLAP (analytics) |
| Write pattern | Single-row CRUD | Bulk batch inserts |
| Query pattern | Point lookups, joins | Full scans, aggregations |
| UPDATE/DELETE | Native, MVCC | Mutations (async, expensive) |
| Transactions | Full ACID | Limited/experimental single-node transactions (since 22.6) |
| Compression | Moderate (TOAST) | 10-30x on columnar data |
| Latency | ms for point queries | ms for aggregations over billions |

**When to offload to ClickHouse**:
- Analytical queries scanning > 1M rows are slowing down OLTP.
- Dashboards need sub-second aggregations over months of data.
- You need real-time rollups and pre-aggregation.
- Data volume exceeds what PostgreSQL handles comfortably (> 500GB analytical data).

## Integration Patterns

### PostgreSQL to ClickHouse via CDC (Debezium)

```text
PostgreSQL WAL -> Debezium -> Kafka -> ClickHouse Kafka Engine -> MergeTree
```

```mermaid
flowchart LR
    PG["PostgreSQL<br/>(OLTP)"]
    DEB["Debezium<br/>(CDC)"]
    KFK["Kafka"]
    CH["ClickHouse<br/>(OLAP)"]
    DASH["Dashboard<br/>(Grafana)"]

    PG -->|WAL changes| DEB
    DEB -->|CDC events| KFK
    KFK -->|Kafka Engine| CH
    CH -->|Queries| DASH
```

### Dual-Write Pitfalls

**Do not dual-write** from application code to both PostgreSQL and ClickHouse:

```java
// WRONG: inconsistency if either write fails
orderRepository.save(order);           // PostgreSQL
clickHouseClient.insert(orderEvent);   // ClickHouse -- may fail silently
```

Use CDC or a transactional outbox pattern to ensure consistency between stores.

### Direct ETL for Historical Data

```sql
-- One-time backfill from PostgreSQL using the JDBC table function
INSERT INTO analytics.orders
SELECT
    order_id, customer_id, total, status, created_at
FROM jdbc('postgresql://pg-host:5432/mydb', 'orders');
```

## Operational Concerns

### Disk Space

- ClickHouse compresses data heavily (LZ4 by default, ZSTD for better ratio).
- Monitor with `system.parts` and `system.disks`.
- Use tiered storage: SSD for hot data, S3/HDD for cold.

```sql
SELECT
    table,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size,
    sum(rows) AS total_rows,
    formatReadableSize(sum(bytes_on_disk) / sum(rows)) AS bytes_per_row
FROM system.parts
WHERE active AND database = 'default'
GROUP BY table
ORDER BY sum(bytes_on_disk) DESC;
```

### TTL for Data Retention

```sql
-- Row-level TTL
ALTER TABLE events MODIFY TTL created_at + INTERVAL 90 DAY DELETE;

-- Column-level TTL (keep summary, drop details)
ALTER TABLE events MODIFY COLUMN properties Map(String, String)
    TTL created_at + INTERVAL 30 DAY;

-- Move to cold storage instead of deleting
ALTER TABLE events MODIFY TTL
    created_at + INTERVAL 7 DAY TO VOLUME 'warm',
    created_at + INTERVAL 30 DAY TO VOLUME 'cold',
    created_at + INTERVAL 365 DAY DELETE;
```

### Mutations

`ALTER TABLE UPDATE` and `ALTER TABLE DELETE` are asynchronous background operations, not instant like SQL `UPDATE`/`DELETE`:

```sql
-- Asynchronous mutation (rewrites affected parts)
ALTER TABLE events DELETE WHERE user_id = 0;
ALTER TABLE events UPDATE status = 'cancelled' WHERE order_id = 5001;

-- Check mutation progress
SELECT * FROM system.mutations WHERE is_done = 0;
```

**Mutations are expensive**. They rewrite entire data parts. For frequent updates, use `ReplacingMergeTree` with versioning instead.

## Related

- [./decision-framework.md](./decision-framework.md) -- When to choose ClickHouse vs other engines
- [./elasticsearch-deep-dive.md](./elasticsearch-deep-dive.md) -- Elasticsearch for search analytics (different access pattern)
- [./dynamodb-patterns.md](./dynamodb-patterns.md) -- DynamoDB for key-value at scale

## References

- [ClickHouse Documentation](https://clickhouse.com/docs)
- [ClickHouse MergeTree Engine](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree)
- [ClickHouse Materialized Views](https://clickhouse.com/docs/en/guides/developer/cascading-materialized-views)
- [Debezium CDC Documentation](https://debezium.io/documentation/)
- [ClickHouse Performance Tips](https://clickhouse.com/docs/en/operations/tips)
- [Altinity ClickHouse Knowledge Base](https://kb.altinity.com/)
