---
title: "PostgreSQL Monitoring"
date: 2026-04-19
updated: 2026-04-19
tags: [postgresql, monitoring, pg-stat, performance, alerting, operations]
---

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `postgresql` `monitoring` `pg-stat` `performance` `alerting` `operations`

# PostgreSQL Monitoring

## Table of Contents

- [Summary](#summary)
- [Monitoring Architecture](#monitoring-architecture)
- [pg_stat_statements](#pg_stat_statements)
  - [Setup](#setup)
  - [Top Queries Analysis](#top-queries-analysis)
  - [Reset Strategies](#reset-strategies)
- [pg_stat_activity](#pg_stat_activity)
  - [Active and Waiting Queries](#active-and-waiting-queries)
  - [Idle in Transaction](#idle-in-transaction)
- [Table Statistics](#table-statistics)
  - [pg_stat_user_tables](#pg_stat_user_tables)
  - [pg_stat_user_indexes](#pg_stat_user_indexes)
- [Bloat Detection](#bloat-detection)
- [Lock Monitoring](#lock-monitoring)
  - [Lock Types](#lock-types)
  - [Deadlock Detection](#deadlock-detection)
- [Connection Monitoring](#connection-monitoring)
- [Checkpoint and WAL Monitoring](#checkpoint-and-wal-monitoring)
- [Alerting Thresholds](#alerting-thresholds)
- [Prometheus and Grafana Integration](#prometheus-and-grafana-integration)
- [Log Analysis](#log-analysis)
- [Related](#related)
- [References](#references)

## Summary

PostgreSQL exposes a rich set of statistics views (`pg_stat_*`) that give real-time visibility into queries, tables, indexes, locks, replication, and I/O. Combined with pg_stat_statements for query-level metrics, Prometheus/Grafana for dashboarding, and pgBadger for log analysis, you can build comprehensive production observability without commercial tools.

## Monitoring Architecture

```mermaid
graph TD
    PG[PostgreSQL] -->|stats views| PE[postgres_exporter]
    PG -->|logs| PB[pgBadger]
    PG -->|pg_stat_statements| PE
    PE -->|/metrics| PR[Prometheus]
    PR --> GR[Grafana Dashboards]
    PR --> AM[Alertmanager]
    AM --> PD[PagerDuty / Slack]
    PB --> HTML[HTML Reports]

    style PG fill:#336791,stroke:#333,color:#fff
    style GR fill:#f46800,stroke:#333,color:#fff
```

## pg_stat_statements

The single most important extension for PostgreSQL performance work. It tracks execution statistics for all SQL statements.

### Setup

```sql
-- postgresql.conf
-- shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.max = 10000
-- pg_stat_statements.track = top

-- After restart, create the extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

Key settings:

```ini
pg_stat_statements.max = 10000          # Max distinct statements tracked
pg_stat_statements.track = top          # top-level statements only (vs 'all' for nested)
pg_stat_statements.track_utility = on   # Track DDL, COPY, etc.
pg_stat_statements.track_planning = on  # Track planning time (PG 13+)
```

### Top Queries Analysis

**Top queries by total execution time:**

```sql
SELECT
    queryid,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

**Queries with worst mean latency:**

```sql
SELECT
    queryid,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows / GREATEST(calls, 1) AS rows_per_call,
    query
FROM pg_stat_statements
WHERE calls > 100
ORDER BY mean_exec_time DESC
LIMIT 20;
```

**Queries causing the most I/O:**

```sql
SELECT
    queryid,
    calls,
    shared_blks_read + shared_blks_written AS total_blks,
    shared_blks_hit,
    round(100.0 * shared_blks_hit / GREATEST(shared_blks_hit + shared_blks_read, 1), 2) AS hit_pct,
    query
FROM pg_stat_statements
ORDER BY (shared_blks_read + shared_blks_written) DESC
LIMIT 20;
```

### Reset Strategies

```sql
-- Full reset (do this periodically, e.g., weekly, to prevent stale data)
SELECT pg_stat_statements_reset();

-- Reset a specific user's stats (PG 13+)
SELECT pg_stat_statements_reset(userid := 'app_user'::regrole);

-- Reset a specific query
SELECT pg_stat_statements_reset(0, 0, queryid);
```

**Best practice:** Snapshot `pg_stat_statements` into a history table before resetting, so you retain trend data.

## pg_stat_activity

Real-time view of every server process and what it is doing right now.

### Active and Waiting Queries

```sql
-- Currently running queries ordered by duration
SELECT
    pid,
    now() - xact_start AS xact_duration,
    now() - query_start AS query_duration,
    state,
    wait_event_type,
    wait_event,
    left(query, 120) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND pid != pg_backend_pid()
ORDER BY query_start ASC;
```

**Find blocked queries and what is blocking them:**

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    now() - blocked.query_start AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid AND NOT bl.granted
JOIN pg_locks gl ON gl.locktype = bl.locktype
    AND gl.database IS NOT DISTINCT FROM bl.database
    AND gl.relation IS NOT DISTINCT FROM bl.relation
    AND gl.page IS NOT DISTINCT FROM bl.page
    AND gl.tuple IS NOT DISTINCT FROM bl.tuple
    AND gl.pid != bl.pid
    AND gl.granted
JOIN pg_stat_activity blocking ON blocking.pid = gl.pid;
```

### Idle in Transaction

Connections stuck in `idle in transaction` hold locks and prevent vacuum. Find and terminate them:

```sql
-- Find idle-in-transaction connections
SELECT
    pid,
    now() - state_change AS idle_duration,
    usename,
    application_name,
    left(query, 100) AS last_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - state_change > interval '5 minutes';
```

**Auto-kill with server settings:**

```ini
idle_in_transaction_session_timeout = 300000  -- 5 minutes in ms
```

## Table Statistics

### pg_stat_user_tables

```sql
SELECT
    schemaname,
    relname,
    seq_scan,
    idx_scan,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / GREATEST(n_live_tup + n_dead_tup, 1), 2) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

**Red flags:**
- `seq_scan` >> `idx_scan` on large tables: missing index
- `dead_pct` > 20%: vacuum is not keeping up
- `last_autovacuum` is NULL or very old: check autovacuum settings

### pg_stat_user_indexes

```sql
-- Unused indexes (candidates for removal)
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (
    SELECT conindid FROM pg_constraint WHERE contype IN ('p', 'u')
  )
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Cache hit ratio (should be > 99% for OLTP):**

```sql
SELECT
    sum(idx_blks_hit) AS hit,
    sum(idx_blks_read) AS read,
    round(100.0 * sum(idx_blks_hit) / GREATEST(sum(idx_blks_hit) + sum(idx_blks_read), 1), 2) AS hit_pct
FROM pg_statio_user_indexes;
```

## Bloat Detection

Table and index bloat wastes disk space and degrades performance.

**Estimated bloat using statistical heuristics:**

```sql
SELECT
    current_database(),
    t.schemaname,
    t.relname AS tablename,
    pg_size_pretty(pg_relation_size(t.schemaname || '.' || t.relname)) AS table_size,
    round(
        (1 - (t.n_live_tup::float / GREATEST(
            (pg_relation_size(t.schemaname || '.' || t.relname) / 8192)::float
            * current_setting('block_size')::int / (
                CASE WHEN s.avg_width > 0 THEN s.avg_width + 24 ELSE 100 END
            ), 1
        ))) * 100, 2
    ) AS estimated_bloat_pct
FROM pg_stat_user_tables t
JOIN (
    SELECT schemaname, tablename, sum(avg_width) AS avg_width
    FROM pg_stats
    GROUP BY schemaname, tablename
) s ON s.schemaname = t.schemaname AND s.tablename = t.relname
WHERE t.n_live_tup > 1000
ORDER BY pg_relation_size(t.schemaname || '.' || t.relname) DESC
LIMIT 20;
```

For precise measurements, use the `pgstattuple` extension:

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT * FROM pgstattuple('orders');
-- Returns: table_len, tuple_count, dead_tuple_count, free_space, etc.

SELECT * FROM pgstatindex('orders_pkey');
-- Returns: leaf_pages, empty_pages, deleted_pages, avg_leaf_density, etc.
```

**Remediation:** `VACUUM FULL` rewrites the table (acquires ACCESS EXCLUSIVE lock). Prefer `pg_repack` for online de-bloating.

## Lock Monitoring

### Lock Types

| Lock Mode | Conflicts With | Common Cause |
|-----------|---------------|--------------|
| `AccessShareLock` | AccessExclusive | SELECT |
| `RowShareLock` | Exclusive, AccessExclusive | SELECT FOR UPDATE |
| `RowExclusiveLock` | Share, ShareRowExclusive, Exclusive, AccessExclusive | INSERT, UPDATE, DELETE |
| `ShareLock` | RowExclusive, ShareRowExclusive, Exclusive, AccessExclusive | CREATE INDEX (non-concurrent) |
| `AccessExclusiveLock` | Everything | ALTER TABLE, DROP TABLE, VACUUM FULL |

**View current locks:**

```sql
SELECT
    l.pid,
    l.locktype,
    l.mode,
    l.granted,
    l.relation::regclass AS table_name,
    a.state,
    a.wait_event_type,
    left(a.query, 80) AS query
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE l.relation IS NOT NULL
ORDER BY l.granted, a.query_start;
```

**Set lock_timeout to prevent queries from waiting forever:**

```sql
SET lock_timeout = '5s';
ALTER TABLE orders ADD COLUMN status text;
-- If the lock isn't acquired within 5s, the statement fails instead of blocking
```

### Deadlock Detection

PostgreSQL detects deadlocks automatically and aborts one of the transactions. Configure detection frequency:

```ini
deadlock_timeout = 1s  -- default; how long to wait before checking for deadlocks
```

**Find deadlocks in logs:**

```bash
grep -i deadlock /var/log/postgresql/postgresql-16-main.log
```

The log shows the exact queries and lock dependencies involved.

## Connection Monitoring

```sql
SELECT
    count(*) AS total_connections,
    count(*) FILTER (WHERE state = 'active') AS active,
    count(*) FILTER (WHERE state = 'idle') AS idle,
    count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_txn,
    count(*) FILTER (WHERE wait_event_type = 'Lock') AS waiting_on_lock,
    current_setting('max_connections')::int AS max_connections,
    round(100.0 * count(*) / current_setting('max_connections')::int, 1) AS pct_used
FROM pg_stat_activity
WHERE backend_type = 'client backend';
```

Alert when connection usage exceeds 80% of `max_connections`.

## Checkpoint and WAL Monitoring

```sql
-- pg_stat_bgwriter (PG < 17) / pg_stat_checkpointer (PG 17+)
SELECT
    checkpoints_timed,
    checkpoints_req,
    buffers_checkpoint,
    buffers_clean,
    buffers_backend,
    round(100.0 * buffers_backend / GREATEST(buffers_checkpoint + buffers_clean + buffers_backend, 1), 2) AS backend_write_pct
FROM pg_stat_bgwriter;
```

**Red flags:**
- `checkpoints_req` >> `checkpoints_timed`: Checkpoints are being triggered by WAL volume, not the schedule. Increase `max_wal_size`.
- `buffers_backend` is high: The background writer and checkpointer cannot keep up. Backends are writing dirty pages themselves, hurting latency.

**WAL generation rate (PG 14+):**

```sql
SELECT
    wal_records,
    wal_bytes,
    pg_size_pretty(wal_bytes) AS wal_generated,
    stats_reset
FROM pg_stat_wal;
```

## Alerting Thresholds

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| Connection usage | > 70% | > 85% | Scale pool or kill idle |
| Replication lag (bytes) | > 100MB | > 1GB | Check network/standby I/O |
| Replication lag (time) | > 10s | > 60s | Investigate standby |
| Cache hit ratio | < 99% | < 95% | Add RAM or review queries |
| Dead tuple ratio | > 10% | > 20% | Check autovacuum settings |
| Transaction ID age | > 150M | > 200M | Run manual VACUUM FREEZE |
| Long-running queries | > 5 min | > 30 min | Investigate or terminate |
| Idle in transaction | > 5 min | > 30 min | Kill and review app code |
| Locks waiting | > 10 | > 50 | Investigate blocking chain |
| Disk space free | < 20% | < 10% | Expand or clean |

**Dashboard vs. page:**

- Dashboard only: cache hit ratio, sequential scan counts, index usage, tuple insert rates
- Page the on-call: connection exhaustion, replication lag critical, disk < 10%, transaction ID wraparound approaching

## Prometheus and Grafana Integration

Install `postgres_exporter`:

```bash
# Download and run
./postgres_exporter --web.listen-address=:9187 \
  --data-source-name="postgresql://monitoring:password@localhost:5432/postgres?sslmode=disable"
```

Prometheus scrape config:

```yaml
scrape_configs:
  - job_name: postgresql
    static_configs:
      - targets: ['pg-host:9187']
    scrape_interval: 15s
```

Custom queries in `queries.yaml` for `postgres_exporter`:

```yaml
pg_stat_statements_top:
  query: |
    SELECT queryid, calls, total_exec_time, mean_exec_time, rows
    FROM pg_stat_statements
    ORDER BY total_exec_time DESC
    LIMIT 20
  metrics:
    - queryid:
        usage: "LABEL"
    - calls:
        usage: "GAUGE"
    - total_exec_time:
        usage: "GAUGE"
    - mean_exec_time:
        usage: "GAUGE"
    - rows:
        usage: "GAUGE"
```

Community Grafana dashboards:
- Dashboard ID 9628 (PostgreSQL Database)
- Dashboard ID 12273 (PostgreSQL Overview)

## Log Analysis

**Essential logging settings:**

```ini
# postgresql.conf
log_min_duration_statement = 500    # Log queries slower than 500ms
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0                  # Log all temp file usage
log_autovacuum_min_duration = 250   # Log slow autovacuum
log_line_prefix = '%t [%p-%l] %q%u@%d '
```

**auto_explain for slow query plans:**

```ini
shared_preload_libraries = 'pg_stat_statements,auto_explain'
auto_explain.log_min_duration = 1000   # Log plans for queries > 1s
auto_explain.log_analyze = on          # Include actual execution times
auto_explain.log_buffers = on          # Include buffer usage
auto_explain.log_timing = on
auto_explain.log_nested_statements = on
```

**pgBadger for log analysis:**

```bash
pgbadger /var/log/postgresql/postgresql-16-main.log \
  --outfile /var/www/pgbadger/report.html \
  --jobs 4

# Incremental mode for large log files
pgbadger --incremental --outdir /var/www/pgbadger/ \
  /var/log/postgresql/postgresql-16-main.log
```

pgBadger generates reports showing: slowest queries, most frequent queries, error rates, lock analysis, checkpoint distribution, and connection patterns.

## Related

- [Connection Management](./connection-management.md) — monitoring connection pools and saturation
- [Replication](./replication.md) — replication-specific monitoring with pg_stat_replication
- [Backup and Recovery](./backup-and-recovery.md) — monitoring backup freshness and WAL archiving

## References

- [PostgreSQL Docs: Monitoring Database Activity](https://www.postgresql.org/docs/current/monitoring.html)
- [PostgreSQL Docs: The Cumulative Statistics System (PG 15+) / Statistics Collector (PG < 15)](https://www.postgresql.org/docs/current/monitoring-stats.html)
- [PostgreSQL Docs: pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)
- [PostgreSQL Docs: auto_explain](https://www.postgresql.org/docs/current/auto-explain.html)
- [postgres_exporter (GitHub)](https://github.com/prometheus-community/postgres_exporter)
- [pgBadger](https://pgbadger.darold.net/)
