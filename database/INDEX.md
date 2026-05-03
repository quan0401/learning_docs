# Database Documentation Index — Learning Path

A progressive path from PostgreSQL engine internals through schema design, operations, query optimization, and polyglot persistence. Language-agnostic — complements the [Java/Spring learning path](../java/INDEX.md), the [Networking learning path](../networking/INDEX.md), the [Kubernetes learning path](../kubernetes/INDEX.md), the [Low-Level Design learning path](../low-level-design/INDEX.md) (for repository pattern, ORM mapping, and domain-model design), the [System Design learning path](../system-design/INDEX.md), the [Operating Systems learning path](../operating-systems/INDEX.md) (page cache, fsync, virtual memory — what Postgres lives on top of), the [Observability learning path](../observability/INDEX.md) for slow-query and metric instrumentation, the [Performance Engineering learning path](../performance/INDEX.md) for query/connection-pool tuning, and the [Security learning path](../security/INDEX.md) for authn/authz, secrets, and TLS to the database.

If you already work with JPA/R2DBC and Spring Data, **start at Tier 1 (Engine Internals)** to understand what your ORM is actually doing under the hood.

**Markers:** **★** = core must-learn (everyday backend work, common in interviews and production tuning). **○** = supporting deep-dive (specialized engine, advanced ops, or polyglot specifics). Internalize all ★ before going deep on ○.

---

## Tier 1 — PostgreSQL Engine Internals

Understand the machine before you tune it. How PostgreSQL processes queries, stores data, and maintains consistency.

1. [★ PostgreSQL Architecture — Process Model, Shared Memory, WAL, and VACUUM](engine-internals/architecture.md) — postmaster, backends, shared buffers, WAL durability, checkpoints, autovacuum _(2026-04-19)_
2. [★ Storage and MVCC — Heap Tuples, Visibility, and Bloat](engine-internals/storage-and-mvcc.md) — page layout, tuple headers, xmin/xmax, snapshot isolation, dead tuples, HOT, TOAST _(2026-04-19)_
3. [★ Indexing Internals — B-tree, GIN, GiST, BRIN, and Hash](engine-internals/indexing-internals.md) — internal structure of each index type, when and why to use each _(2026-04-19)_
4. [★ Query Planner and Executor — Cost Model, Statistics, and Join Algorithms](engine-internals/query-planner-executor.md) — parse → plan → execute, cost estimation, pg_stats, scan and join types _(2026-04-19)_

---

## Tier 2 — Query Optimization

Bridge your application-layer knowledge with database-level tuning. Read execution plans and systematically fix slow queries.

5. [★ EXPLAIN ANALYZE Guide — Reading and Interpreting Execution Plans](query-optimization/explain-analyze-guide.md) — node types, cost vs actual time, buffers, annotated real examples _(2026-04-19)_
6. [★ Query Optimization Patterns — Systematic Approaches to Faster Queries](query-optimization/optimization-patterns.md) — N+1 at DB level, JOIN tuning, window functions, keyset pagination, batch ops _(2026-04-19)_

---

## Tier 3 — Database Design

Schema modeling from normalization through advanced patterns. PostgreSQL-focused but broadly applicable.

7. [★ Normalization and Tradeoffs — 1NF Through BCNF and When to Denormalize](design/normalization-and-tradeoffs.md) — functional dependencies, strategic denormalization, materialized views, JSONB _(2026-04-19)_
8. [★ Indexing Strategies — Beyond "Add an Index on the WHERE Column"](design/indexing-strategies.md) — multi-column ordering, covering indexes, partial, expression, GIN for JSONB, FTS _(2026-04-19)_
9. [○ Partitioning and Sharding — Scaling PostgreSQL Tables](design/partitioning-and-sharding.md) — declarative partitioning, partition pruning, Citus, shard key selection, FDW _(2026-04-19)_
10. [★ Data Modeling Patterns — Common Schema Patterns for Real Applications](design/data-modeling-patterns.md) — soft deletes, temporal tables, audit trails, EAV, polymorphic, trees, multi-tenancy _(2026-04-19)_

---

## Tier 4 — Operations & DBA

Production PostgreSQL: keep it running, keep it safe, keep it fast.

11. [★ Replication — Streaming, Logical, and High Availability](operations/replication.md) — primary/standby, sync vs async, logical replication, Patroni, failover _(2026-04-19)_
12. [★ Backup and Recovery — pg_dump, WAL Archiving, and PITR](operations/backup-and-recovery.md) — logical/physical backups, pgBackRest, point-in-time recovery, restore walkthrough _(2026-04-19)_
13. [○ Monitoring — pg_stat, Bloat Detection, and Alerting](operations/monitoring.md) — pg_stat_statements, pg_stat_activity, lock monitoring, Prometheus/Grafana _(2026-04-19)_
14. [★ Connection Management — PgBouncer, Pooling, and Connection Storms](operations/connection-management.md) — process-per-connection model, pooling modes, HikariCP interaction, cloud poolers _(2026-04-19)_
15. [★ Migrations at Scale — Zero-Downtime DDL and Schema Changes](operations/migrations-at-scale.md) — lock-safe DDL, CREATE INDEX CONCURRENTLY, expand-contract, batched backfills _(2026-04-19)_

---

## Tier 5 — Polyglot Persistence

When PostgreSQL isn't enough. Deep dives into complementary database engines and when to reach for them.

16. [★ Polyglot Persistence Decision Framework](polyglot/decision-framework.md) — relational vs document vs key-value vs search vs time-series vs graph, decision tree _(2026-04-19)_
17. [★ Redis Beyond Caching — Data Structures, Streams, and Persistence](polyglot/redis-beyond-caching.md) — sorted sets, streams vs Kafka, Lua scripting, cluster, Spring Data Redis _(2026-04-19)_
18. [○ Elasticsearch Deep Dive — Search, Analytics, and Relevance](polyglot/elasticsearch-deep-dive.md) — inverted index, analyzers, Query DSL, BM25, aggregations, ILM tiers _(2026-04-19)_
19. [○ MongoDB — When and How to Use Document Databases](polyglot/mongodb-when-and-how.md) — document modeling, aggregation pipeline, sharding, change streams, vs JSONB _(2026-04-19)_
20. [○ ClickHouse for Real-Time Analytics](polyglot/clickhouse-analytics.md) — columnar storage, MergeTree, materialized views, CDC from PostgreSQL _(2026-04-19)_
21. [○ DynamoDB Patterns — Single-Table Design and Serverless Persistence](polyglot/dynamodb-patterns.md) — PK/SK overloading, GSI, streams, capacity modes, cost model _(2026-04-19)_

---

## Quick Reference by Topic

### Engine Internals

- [PostgreSQL Architecture](engine-internals/architecture.md)
- [Storage and MVCC](engine-internals/storage-and-mvcc.md)
- [Indexing Internals](engine-internals/indexing-internals.md)
- [Query Planner and Executor](engine-internals/query-planner-executor.md)

### Query Optimization

- [EXPLAIN ANALYZE Guide](query-optimization/explain-analyze-guide.md)
- [Optimization Patterns](query-optimization/optimization-patterns.md)

### Schema Design

- [Normalization and Tradeoffs](design/normalization-and-tradeoffs.md)
- [Indexing Strategies](design/indexing-strategies.md)
- [Partitioning and Sharding](design/partitioning-and-sharding.md)
- [Data Modeling Patterns](design/data-modeling-patterns.md)

### Operations

- [Replication](operations/replication.md)
- [Backup and Recovery](operations/backup-and-recovery.md)
- [Monitoring](operations/monitoring.md)
- [Connection Management](operations/connection-management.md)
- [Migrations at Scale](operations/migrations-at-scale.md)

### Polyglot Persistence

- [Decision Framework](polyglot/decision-framework.md)
- [Redis Beyond Caching](polyglot/redis-beyond-caching.md)
- [Elasticsearch Deep Dive](polyglot/elasticsearch-deep-dive.md)
- [MongoDB — When and How](polyglot/mongodb-when-and-how.md)
- [ClickHouse for Analytics](polyglot/clickhouse-analytics.md)
- [DynamoDB Patterns](polyglot/dynamodb-patterns.md)
