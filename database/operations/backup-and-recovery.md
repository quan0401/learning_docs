---
title: "PostgreSQL Backup and Recovery"
date: 2026-04-19
updated: 2026-04-19
tags: [postgresql, backup, recovery, pitr, pgbackrest, operations]
---

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `postgresql` `backup` `recovery` `pitr` `pgbackrest` `operations`

# PostgreSQL Backup and Recovery

## Table of Contents

- [Summary](#summary)
- [Backup Strategy Decision Tree](#backup-strategy-decision-tree)
- [Logical Backups](#logical-backups)
  - [pg_dump](#pg_dump)
  - [pg_dumpall](#pg_dumpall)
- [Physical Backups](#physical-backups)
  - [pg_basebackup](#pg_basebackup)
  - [Filesystem Snapshots](#filesystem-snapshots)
- [WAL Archiving](#wal-archiving)
  - [archive_command](#archive_command)
  - [archive_library (PG15+)](#archive_library-pg15)
- [Point-in-Time Recovery (PITR)](#point-in-time-recovery-pitr)
  - [Recovery Targets](#recovery-targets)
  - [Timeline Management](#timeline-management)
- [pgBackRest](#pgbackrest)
  - [Configuration](#configuration)
  - [Backup Types](#backup-types)
  - [S3/GCS Backends](#s3gcs-backends)
  - [Retention Policies](#retention-policies)
- [Backup Verification](#backup-verification)
- [RPO and RTO Planning](#rpo-and-rto-planning)
- [Practical Restore Walkthrough](#practical-restore-walkthrough)
- [Related](#related)
- [References](#references)

## Summary

PostgreSQL offers logical backups (pg_dump) for portability and physical backups (pg_basebackup, filesystem snapshots) for speed. Combined with continuous WAL archiving, physical backups enable point-in-time recovery to any moment. pgBackRest is the production standard for managing this pipeline with incremental backups, parallelism, and cloud storage.

## Backup Strategy Decision Tree

```mermaid
flowchart TD
    A[What are your RPO/RTO requirements?] --> B{RPO < 1 minute?}
    B -->|Yes| C[Physical backup + WAL archiving + streaming replication]
    B -->|No| D{Database < 50GB?}
    D -->|Yes| E[pg_dump custom format + optional WAL archiving]
    D -->|No| F[pgBackRest incremental + WAL archiving]
    C --> G{Need cloud storage?}
    F --> G
    G -->|Yes| H[pgBackRest with S3/GCS backend]
    G -->|No| I[pgBackRest with local/NFS storage]
    E --> J[Store dumps on separate volume or object storage]
```

## Logical Backups

### pg_dump

Exports a single database as SQL or a portable archive format.

**Custom format** (recommended for production):

```bash
# Parallel dump with 4 workers, directory format
# Note: --jobs only works with --format=directory, not --format=custom
pg_dump -h localhost -U app_user -d mydb \
  --format=directory --jobs=4 --compress=zstd:3 \
  --file=/backups/mydb_$(date +%Y%m%d_%H%M%S)/
```

**Directory format** (enables parallel restore):

```bash
pg_dump -h localhost -U app_user -d mydb \
  --format=directory --jobs=4 \
  --file=/backups/mydb_$(date +%Y%m%d_%H%M%S)/
```

**Restore from custom format:**

```bash
pg_restore -h localhost -U app_user -d mydb_restored \
  --jobs=4 --clean --if-exists \
  /backups/mydb_20260419_030000.dump
```

**Restore a single table:**

```bash
pg_restore -h localhost -U app_user -d mydb \
  --table=orders --data-only \
  /backups/mydb_20260419_030000.dump
```

**Limitations:**
- Takes a snapshot at dump time; no WAL replay means RPO equals backup frequency
- Slow for databases over 100GB even with parallelism
- Uses `REPEATABLE READ` snapshot for consistent point-in-time dump (without locking tables)

### pg_dumpall

Dumps cluster-wide objects that `pg_dump` cannot: roles, tablespaces, and permissions.

```bash
# Dump globals only (roles + tablespaces)
pg_dumpall -h localhost -U postgres --globals-only \
  --file=/backups/globals_$(date +%Y%m%d).sql
```

Always pair `pg_dumpall --globals-only` with per-database `pg_dump` backups. A full `pg_dumpall` produces plain SQL that cannot be restored in parallel.

## Physical Backups

### pg_basebackup

Creates a binary copy of the entire PostgreSQL data directory.

```bash
pg_basebackup -h primary-host -U replicator \
  -D /backups/base_$(date +%Y%m%d_%H%M%S) \
  --checkpoint=fast \
  --wal-method=stream \
  --compress=server-gzip:5 \
  --progress --verbose
```

Key flags:
- `--checkpoint=fast`: Triggers an immediate checkpoint instead of waiting for the next scheduled one
- `--wal-method=stream`: Streams WAL in parallel so the backup is self-contained
- `--compress=server-gzip:5`: Server-side compression (PG 15+ supports `zstd`, `lz4`)

**Size and speed:** Roughly equal to the data directory size. A 500GB database takes ~30 minutes on modern SSDs with gigabit networking.

### Filesystem Snapshots

If your storage supports atomic snapshots (LVM, ZFS, EBS, persistent disks):

```bash
# 1. Optional: force a checkpoint to reduce recovery time
psql -c "CHECKPOINT;"

# 2. Take the snapshot (example: LVM)
lvcreate --snapshot --name pg_snap --size 50G /dev/vg0/pg_data

# 3. Mount and copy (or keep as-is for instant restore)
mount /dev/vg0/pg_snap /mnt/pg_snap
rsync -a /mnt/pg_snap/ /backups/snapshot_$(date +%Y%m%d)/

# 4. Clean up
umount /mnt/pg_snap
lvremove /dev/vg0/pg_snap
```

**Important:** The snapshot must include both the data directory and any separate WAL or tablespace directories atomically. For EBS, use multi-volume snapshots.

## WAL Archiving

WAL archiving copies completed WAL segments to a safe location, enabling PITR between base backups.

### archive_command

The classic approach — PostgreSQL calls a shell command for each completed WAL segment:

```ini
# postgresql.conf
archive_mode = on
archive_command = 'test ! -f /wal_archive/%f && cp %p /wal_archive/%f'
archive_timeout = 300  # Force archive every 5 min even if WAL segment not full
```

A more robust command using compression:

```ini
archive_command = 'gzip < %p > /wal_archive/%f.gz'
```

For S3:

```ini
archive_command = 'aws s3 cp %p s3://my-wal-bucket/%f --only-show-errors'
```

### archive_library (PG15+)

PostgreSQL 15 introduced `archive_library` as a faster, in-process alternative to shelling out:

```ini
archive_library = 'basic_archive'
basic_archive.archive_directory = '/wal_archive'
```

pgBackRest and other tools provide their own archive modules that are more robust than `basic_archive`.

## Point-in-Time Recovery (PITR)

PITR lets you restore a database to any point between a base backup and the latest archived WAL.

```mermaid
graph LR
    BB[Base Backup<br/>Sunday 02:00] --> W1[WAL Segments]
    W1 --> W2[WAL Segments]
    W2 --> W3[WAL Segments]
    W3 --> TP[Target Point<br/>Wednesday 03:15]
    TP -.-> W4[WAL after target<br/>Not replayed]

    style BB fill:#69b,stroke:#333,color:#fff
    style TP fill:#f96,stroke:#333,color:#fff
```

**Steps:**

1. Restore the base backup to a new data directory
2. Configure recovery settings in `postgresql.conf`:

```ini
# postgresql.conf (PG 12+)
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2026-04-16 03:15:00+00'
recovery_target_action = 'promote'
```

3. Create the recovery signal file:

```bash
touch /var/lib/postgresql/16/main/recovery.signal
```

4. Start PostgreSQL — it replays WAL up to the target, then promotes to read-write.

### Recovery Targets

```sql
-- By timestamp (most common)
recovery_target_time = '2026-04-16 03:15:00+00'

-- By LSN (precise, from pg_current_wal_lsn() before the incident)
recovery_target_lsn = '0/1A2B3C4D'

-- By transaction ID
recovery_target_xid = '12345678'

-- By named restore point
SELECT pg_create_restore_point('before_migration');
-- Then: recovery_target_name = 'before_migration'
```

### Timeline Management

Each PITR creates a new timeline. PostgreSQL tracks timelines in `.history` files within the WAL archive. This lets you branch recovery history without losing the ability to recover to points on the original timeline.

```ini
# Follow the latest timeline (default, usually what you want)
recovery_target_timeline = 'latest'

# Or specify a specific timeline
recovery_target_timeline = '3'
```

## pgBackRest

pgBackRest is the standard tool for production PostgreSQL backups. It handles base backups, WAL archiving, verification, and restore in a single package.

### Configuration

```ini
# /etc/pgbackrest/pgbackrest.conf

[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=4
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=your-encryption-key
process-max=4
compress-type=zst
compress-level=3
start-fast=y
log-level-console=info
log-level-file=detail

[mydb]
pg1-path=/var/lib/postgresql/16/main
pg1-port=5432
pg1-user=postgres
```

Wire pgBackRest into PostgreSQL's WAL archiving:

```ini
# postgresql.conf
archive_mode = on
archive_command = 'pgbackrest --stanza=mydb archive-push %p'
```

### Backup Types

```bash
# Full backup (complete copy)
pgbackrest --stanza=mydb backup --type=full

# Differential (changes since last full)
pgbackrest --stanza=mydb backup --type=diff

# Incremental (changes since last any backup — default)
pgbackrest --stanza=mydb backup --type=incr
```

Typical schedule:
- Full: Weekly (Sunday 02:00)
- Differential: Daily (02:00)
- WAL archiving: Continuous

```bash
# List backups
pgbackrest --stanza=mydb info
```

### S3/GCS Backends

```ini
# S3 backend
[global]
repo1-type=s3
repo1-s3-bucket=my-pg-backups
repo1-s3-region=us-east-1
repo1-s3-endpoint=s3.amazonaws.com
repo1-s3-key=AKIAIOSFODNN7EXAMPLE
repo1-s3-key-secret=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
repo1-path=/pgbackrest

# GCS backend
[global]
repo1-type=gcs
repo1-gcs-bucket=my-pg-backups
repo1-gcs-key=/etc/pgbackrest/gcs-key.json
repo1-path=/pgbackrest
```

### Retention Policies

```ini
# Keep 2 full backups and everything in between
repo1-retention-full=2

# Keep 7 differential backups
repo1-retention-diff=7

# Archive retention: keep WAL for all retained backups
repo1-retention-archive-type=full
repo1-retention-archive=2
```

## Backup Verification

**pg_verifybackup** (PG 13+) checks that a base backup taken with `pg_basebackup` is complete and uncorrupted:

```bash
pg_verifybackup /backups/base_20260419_020000
```

**pgBackRest verification:**

```bash
# Verify backup integrity
pgbackrest --stanza=mydb verify

# Test restore to a temporary location (the real proof)
pgbackrest --stanza=mydb restore \
  --target=/tmp/pg_restore_test \
  --type=immediate \
  --target-action=promote

pg_ctl start -D /tmp/pg_restore_test -o "-p 5433"
psql -p 5433 -c "SELECT count(*) FROM orders;"
pg_ctl stop -D /tmp/pg_restore_test
rm -rf /tmp/pg_restore_test
```

**Automate restore testing.** A backup that has never been restore-tested is not a backup.

## RPO and RTO Planning

| Strategy | RPO | RTO | Complexity |
|----------|-----|-----|------------|
| pg_dump nightly | Up to 24 hours | Hours (depends on DB size) | Low |
| pg_basebackup + WAL archiving | Seconds (last archived WAL) | 30-60 min | Medium |
| pgBackRest incremental + WAL | Seconds | 15-30 min (parallel restore) | Medium |
| Streaming replication + auto failover | ~0 (sync) or seconds (async) | Seconds to minutes | High |
| pgBackRest + replication combo | ~0 | Seconds (failover) or minutes (PITR) | High |

**Choose based on business requirements:**

- If RTO must be < 1 minute: streaming replication with Patroni
- If RPO must be ~0: synchronous replication
- If you need to recover from logical corruption (bad UPDATE, dropped table): PITR via pgBackRest
- Replication alone does not protect against logical corruption — the bad write replicates instantly

## Practical Restore Walkthrough

**Scenario:** Production database was corrupted at 3AM by a bad migration that truncated the `users` table. You discover it at 7AM.

```bash
# 1. Identify the exact time of corruption from application logs or pg_stat_activity history
#    Let's say the bad migration ran at 03:12:00 UTC

# 2. Stop the corrupted instance (or work on a separate server)
pg_ctl stop -D /var/lib/postgresql/16/main

# 3. Move the corrupted data directory aside
mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main_corrupted

# 4. Restore from pgBackRest to just before the corruption
pgbackrest --stanza=mydb restore \
  --type=time \
  --target="2026-04-16 03:11:00+00" \
  --target-action=promote \
  --process-max=4

# 5. Start PostgreSQL
pg_ctl start -D /var/lib/postgresql/16/main

# 6. Verify the data
psql -c "SELECT count(*) FROM users;"
psql -c "SELECT max(created_at) FROM users;"

# 7. If correct, update application connection strings
# 8. Rebuild standbys from the new primary

# 9. Post-mortem: extract the bad migration from the corrupted data dir's logs
# 10. Add a restore point before future migrations:
psql -c "SELECT pg_create_restore_point('before_migration_v42');"
```

**Key lesson:** Always create a named restore point before running migrations:

```sql
SELECT pg_create_restore_point('pre_deploy_v2_5_0');
```

## Related

- [Replication](./replication.md) — streaming replication as a complement to backups
- [Monitoring](./monitoring.md) — monitoring WAL archiving lag and backup freshness
- [Migrations at Scale](./migrations-at-scale.md) — safe migration practices that reduce the need for emergency PITR

## References

- [PostgreSQL Docs: Backup and Restore](https://www.postgresql.org/docs/current/backup.html)
- [PostgreSQL Docs: Continuous Archiving and PITR](https://www.postgresql.org/docs/current/continuous-archiving.html)
- [PostgreSQL Docs: pg_basebackup](https://www.postgresql.org/docs/current/app-pgbasebackup.html)
- [PostgreSQL Docs: pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html)
- [PostgreSQL Docs: pg_verifybackup](https://www.postgresql.org/docs/current/app-pgverifybackup.html)
- [pgBackRest Documentation](https://pgbackrest.org/)
