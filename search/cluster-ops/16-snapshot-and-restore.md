---
title: "Snapshot and Restore — Repositories, SLM, and Cross-Cluster Recovery"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, snapshot, restore, slm, backup, disaster-recovery]
---

# Snapshot and Restore — Repositories, SLM, and Cross-Cluster Recovery

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `snapshot` `restore` `slm` `backup` `disaster-recovery`

---

## Table of Contents

- [Summary](#summary)
- [1. The Repository Abstraction](#1-the-repository-abstraction)
- [2. Snapshot Mechanics — Incremental at the Segment Level](#2-snapshot-mechanics--incremental-at-the-segment-level)
- [3. Snapshot Consistency and Refresh Semantics](#3-snapshot-consistency-and-refresh-semantics)
- [4. Snapshot Lifecycle Management (SLM)](#4-snapshot-lifecycle-management-slm)
- [5. Restore — Full, Partial, Renamed, Cross-Cluster](#5-restore--full-partial-renamed-cross-cluster)
- [6. Searchable Snapshots and the Frozen Tier](#6-searchable-snapshots-and-the-frozen-tier)
- [7. Repository Plugin Differences and Special Repo Modes](#7-repository-plugin-differences-and-special-repo-modes)
- [8. Practical Patterns](#8-practical-patterns)
- [9. Practical Traps](#9-practical-traps)
- [Related](#related)
- [References](#references)

---

## Summary

Snapshot and restore is the only built-in mechanism for backing up Elasticsearch and OpenSearch data. It is **not** a logical export — snapshots reference Lucene segment files directly, which is what makes them efficient (incremental at the segment level, not the document level) and also what makes them a strict superset of any plausible "rebuild from source" recovery plan: you get the index exactly as it was, doc IDs and all, with the same shard layout and the same internal `_id`.

A snapshot is a coordinated copy of one or more shards' on-disk segments to a registered **repository** — an abstraction over a backing store (filesystem, S3, GCS, Azure Blob, HDFS). Each subsequent snapshot copies only the new segments that have appeared since the last snapshot for that index, because Lucene segments are immutable (see [foundations/01-inverted-index-internals.md](../foundations/01-inverted-index-internals.md) §7). This is why a daily snapshot of a 10 TB index typically transfers tens of GB, not 10 TB.

This doc walks the repository registration API, the incremental-snapshot model, SLM (the scheduler + retention layer on top), the restore variants you actually need (rename pattern, settings overrides, cross-cluster), and searchable snapshots (which let frozen-tier indices serve queries directly from the repository). It closes with the traps that bite teams in production: shared-FS locking, snapshot-generation mismatches after split-brain, and version compatibility windows on cross-cluster restore.

---

## 1. The Repository Abstraction

A repository is a named, cluster-wide registration of a backing store. Every snapshot lives inside one repository; every restore reads from one repository.

```bash
# S3 repository
PUT _snapshot/prod-backups
{
  "type": "s3",
  "settings": {
    "bucket": "my-es-backups",
    "region": "us-east-1",
    "base_path": "prod/cluster-a",
    "compress": true,
    "server_side_encryption": true
  }
}

# GCS
PUT _snapshot/prod-backups-gcs
{
  "type": "gcs",
  "settings": {
    "bucket": "my-es-backups",
    "client": "default",
    "base_path": "prod/cluster-a"
  }
}

# Azure Blob
PUT _snapshot/prod-backups-azure
{
  "type": "azure",
  "settings": {
    "container": "es-backups",
    "client": "default",
    "base_path": "prod/cluster-a"
  }
}

# Shared filesystem (NFS mount that every node can see at the same path)
PUT _snapshot/local-backups
{
  "type": "fs",
  "settings": {
    "location": "/mnt/es-snapshots",
    "compress": true
  }
}
```

The `type` selects a repository plugin. `fs` and `url` (read-only HTTP) ship in core; `s3`, `gcs`, `azure`, and `hdfs` are official plugins, preinstalled on Elastic Cloud and on the OpenSearch distribution. Credentials live in the keystore (`bin/elasticsearch-keystore add s3.client.default.access_key`) — never in `elasticsearch.yml` and never inline in the registration body.

`compress: true` compresses the snapshot **metadata** (the JSON manifest files), not segment data — Lucene segments are already compressed via doc-values codecs and postings codecs, so zipping them again is mostly wasted CPU.

Every node must have credentials and network access to the backing store. A snapshot is a parallel write from every data node holding a primary shard of an indexed-being-snapshotted; a node that cannot reach S3 will fail its part of the snapshot and the whole snapshot will be marked `PARTIAL`.

---

## 2. Snapshot Mechanics — Incremental at the Segment Level

This is the single most important fact about ES/OS snapshots:

> A snapshot is a copy of *new Lucene segment files* since the last snapshot for that shard, plus a small JSON manifest. It is **not** a document-level diff and it is **not** a full copy.

Lucene segments are immutable (see foundations doc §7). Once segment `_5x.cfs` has been written to disk, it never changes; deletes are tombstones in `.liv` files, updates are delete+insert. So a snapshot can simply record "this shard contains segments `_3a, _4f, _5x`" and copy any of those segment files that aren't already in the repository. The next snapshot looks at the same shard, sees segments `_5x, _6c, _7d` (because `_3a` and `_4f` were merged into `_6c` in the meantime), and only copies `_6c, _7d` — `_5x` is already in the repository.

The repository is therefore a **deduplicated content-addressable store** of segment files keyed by name+checksum, plus a per-snapshot manifest describing which files belong to which shard of which index at which point in time. Deleting a snapshot does not delete the underlying segment files unless no surviving snapshot references them; this is why `DELETE _snapshot/repo/snap-name` can be cheap or expensive depending on what merges happened between snapshots.

```bash
# Take a snapshot
PUT _snapshot/prod-backups/snap-2026-05-07?wait_for_completion=false
{
  "indices": "logs-*,orders-*",
  "ignore_unavailable": true,
  "include_global_state": false
}

# Status
GET _snapshot/prod-backups/snap-2026-05-07/_status
```

`include_global_state: false` is the default that most teams want — `true` snapshots cluster-wide state (templates, ILM policies, transforms, persistent settings) and on restore will overwrite those on the target cluster, which is rarely desirable for routine backups.

A snapshot of a million tiny indices is slow not because of segment data but because of metadata round-trips per shard. Snapshot performance is dominated by the small-file rate of the backing store, not its bulk throughput.

---

## 3. Snapshot Consistency and Refresh Semantics

A snapshot is "as-of the moment the `_snapshot` call lands" — but only for *segments already on disk*. The in-memory indexing buffer (the documents not yet flushed into a segment) is not part of any snapshot. Every primary shard issues an internal flush at the start of the snapshot, which forces the buffer to a new segment, and the snapshot then references that segment. Inflight bulk requests that have been acknowledged are durable in the translog and will land in a segment on flush; ones not yet acknowledged are not part of the snapshot.

The practical consequence: a snapshot taken during a steady-state bulk-indexing workload will contain everything the cluster had acknowledged at the moment of the snapshot call, plus or minus a small window of in-flight work. It is **not** a transactionally consistent multi-index snapshot — each shard's flush is independent. For a write workload that requires "all indices at the same logical instant," you need application-layer coordination (e.g., pause writes, then snapshot).

Refresh interval matters because the segment-level incremental dedup works best when segments are stable. An index with `index.refresh_interval: 1s` produces many small segments; each snapshot copies and references all of them, and the merge churn means yesterday's segments may be gone (merged) and replaced with new ones, so the dedup ratio degrades. Bulk-indexing pipelines that set `refresh_interval: 30s` or `-1` during heavy ingest see dramatically smaller delta snapshots.

---

## 4. Snapshot Lifecycle Management (SLM)

SLM is the scheduler + retention layer on top of `_snapshot`. It runs inside the cluster, executes on a cron schedule, and applies a retention policy after each run.

```bash
PUT _slm/policy/daily-prod
{
  "schedule": "0 30 1 * * ?",
  "name": "<daily-prod-{now/d}>",
  "repository": "prod-backups",
  "config": {
    "indices": ["*"],
    "ignore_unavailable": true,
    "include_global_state": true
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 7,
    "max_count": 60
  }
}

# Trigger manually (useful for testing)
POST _slm/policy/daily-prod/_execute

# Status
GET _slm/stats
GET _slm/policy/daily-prod
```

The retention block has all three of `expire_after`, `min_count`, `max_count` for a reason: `expire_after` deletes old snapshots, `min_count` protects against accidentally deleting all snapshots if the policy stops running, `max_count` caps growth if `expire_after` is too generous. SLM's retention task runs by default at `01:30` cluster-local time (`slm.retention_schedule`) and processes all policies.

The `<daily-prod-{now/d}>` syntax is a date-math snapshot name — `{now/d}` resolves to the current UTC date at execution time. Use it; otherwise you have to manage uniqueness yourself.

OpenSearch's equivalent is `_sm/policies` (Snapshot Management); the surface area is similar but the API names differ.

---

## 5. Restore — Full, Partial, Renamed, Cross-Cluster

Restore reads from the repository and reconstructs indices on the target cluster. The target index **must not exist** unless you provide a rename pattern or close+restore.

```bash
# Full restore of one index
POST _snapshot/prod-backups/snap-2026-05-07/_restore
{
  "indices": "orders-2026-05",
  "include_global_state": false
}

# Restore with rename pattern (recover into a parallel index for inspection)
POST _snapshot/prod-backups/snap-2026-05-07/_restore
{
  "indices": "orders-*",
  "rename_pattern": "orders-(.+)",
  "rename_replacement": "orders-restored-$1",
  "include_aliases": false
}

# Restore with settings override (e.g., bring back as a single-shard, no-replica index for offline use)
POST _snapshot/prod-backups/snap-2026-05-07/_restore
{
  "indices": "orders-2026-05",
  "rename_pattern": "(.+)",
  "rename_replacement": "$1-offline",
  "index_settings": {
    "index.number_of_replicas": 0,
    "index.refresh_interval": "30s"
  },
  "ignore_index_settings": ["index.lifecycle.name"]
}
```

The most useful pattern for production debugging is **rename-restore into a parallel index**: an engineer suspects yesterday's data was corrupted; you restore yesterday's snapshot as `orders-restored-*` alongside the live `orders-*`, diff the two with a query, and either swap an alias to roll back or copy specific docs. This avoids the catastrophe of a "fix" that destroys good data.

**Cross-cluster restore** works as long as the source and target are version-compatible. The compatibility window is tight: a snapshot taken in version `N` can be restored on the same major (`N`) or the next major (`N+1`) if both honor the index format compatibility window. Specifically, an index has to have been created in a version no older than `N-1` major to be restorable on `N+1` — you cannot resurrect an Elasticsearch 6 index on Elasticsearch 9 even with intermediate hops, because the segment format is too old for the newer Lucene.

---

## 6. Searchable Snapshots and the Frozen Tier

A searchable snapshot is an index whose segment files live in the repository and are streamed on demand instead of being fully copied to local disk. Querying a frozen-tier index reads directly from S3 (or GCS/Azure), with a local cache layer to amortize re-reads.

```bash
# Mount a snapshot as a searchable index in the frozen tier
POST _snapshot/prod-backups/snap-2026-05-07/_mount?wait_for_completion=true
{
  "index": "logs-2026-04",
  "renamed_index": "logs-2026-04-frozen",
  "index_settings": {
    "index.number_of_replicas": 0
  }
}
```

The frozen tier is the last stop in the ILM hot→warm→cold→frozen progression (cross-link to [ILM](./15-index-lifecycle-management.md) when that doc lands). Storage cost drops to the cost of S3 with no per-node disk; query latency goes up because cold reads fault from S3. Searchable snapshots are an Elasticsearch feature; OpenSearch added an equivalent in 2.x.

The searchable-snapshots feature is part of Elasticsearch's commercial subscription tier (Enterprise on the self-managed side); the open-source-licensed parts of the stack do not include it. OpenSearch's searchable-snapshots feature is in the Apache 2.0 distribution.

---

## 7. Repository Plugin Differences and Special Repo Modes

The official plugins (`s3`, `gcs`, `azure`, `hdfs`) are cosmetically similar — same snapshot model, same `compress`/`base_path`/`chunk_size` settings — but each has its own credential and network knobs:

- **S3** — credentials via keystore (`s3.client.default.access_key`/`secret_key`) or instance profile/IRSA on EKS; supports `storage_class` (`STANDARD`/`STANDARD_IA`/`GLACIER` is a trap — restore from Glacier requires a thaw and is not transparent).
- **GCS** — service-account JSON in keystore (`gcs.client.default.credentials_file`).
- **Azure Blob** — account name + key, or SAS token, in keystore.
- **HDFS** — Kerberos integration is supported but operationally painful; most teams avoid HDFS in favor of object storage.

**Read-only repositories** are useful for the cross-cluster restore case: the production cluster registers the repository read-write, the disaster-recovery or analytics cluster registers the same bucket as `readonly: true`. The DR cluster cannot accidentally write a snapshot that would corrupt the prod-cluster's view of the repo.

```bash
PUT _snapshot/prod-backups-readonly
{
  "type": "s3",
  "settings": {
    "bucket": "my-es-backups",
    "base_path": "prod/cluster-a",
    "readonly": true
  }
}
```

**Source-only repositories** (Elasticsearch only) store only `_source` and minimal metadata — no postings, no doc values. They are roughly 50% smaller but can only be restored by reindexing into a real index; you cannot search them directly. Useful for very-long-term archival of data you intend to reanalyze rarely.

---

## 8. Practical Patterns

**Hot index daily snapshot.** SLM policy `daily-prod` running at 01:30 with 30-day retention, snapshotting all indices to S3 in the same region as the cluster. Use `wait_for_completion=false` so the cluster API returns immediately; SLM tracks it.

**Disaster-recovery cross-region repo.** A second SLM policy `weekly-dr` writes to a *different* bucket in a *different* region, on a less-frequent schedule, with longer retention (e.g., 12 weeks). This is the "us-east-1 caught fire" copy. The bucket should have S3 replication or equivalent on top — your snapshot is not a backup of itself.

**Restore drills.** Once a quarter, pick a recent snapshot and restore it as `*-restoredrill-*` on a non-prod cluster. Verify doc counts, run a known query, time the restore. A backup that has never been restored is not a backup.

**Pre-upgrade snapshot.** Before any major-version upgrade, take a manual snapshot and verify it before starting the upgrade. The upgrade docs explicitly recommend this; cross-major restores back to the older version are not supported, so the snapshot is your only rollback if the upgrade goes sideways.

**Bulk-indexing window.** During a backfill of historical data, set the index's `refresh_interval: -1`, `number_of_replicas: 0`. Take a snapshot at the end of the backfill, then re-enable replicas and refresh. This avoids snapshotting the noisy intermediate segment churn and keeps the next incremental small.

---

## 9. Practical Traps

**Shared-filesystem repo locking.** Type `fs` requires the same path mounted on every master and data node, with read-write access from each. NFS implementations vary in lock semantics; an NFS server with broken `flock` can let two snapshots step on each other's manifest writes and corrupt the repo. Object storage repos do not have this class of bug. If you must use `fs`, use `nfsv4` with strict locking and prefer a single dedicated NFS server.

**Snapshot generation mismatch.** Each repository has a generation file (`index.latest`); after a network partition or split-brain, two different masters can both believe they are the writer. The newer master's generation will be ahead, and the lagging one's snapshots will fail with "expected generation X, found Y." The fix is `POST _snapshot/<repo>/_cleanup` and, in pathological cases, re-registering the repo. The Elastic snapshot reference describes the cleanup API in detail.

**Cross-cluster version compatibility.** A snapshot from version `N` can be restored on version `N` or `N+1`, *but only if* the index itself was created in a version `≥ N-1`. An index originally created in 7.x, snapshotted in 8.x, can be restored on 9.x; an index originally created in 6.x, snapshotted in 7.x, cannot be restored on 9.x even via 8.x as a hop. Always check `index.version.created` before relying on a snapshot for a multi-version recovery plan.

**Glacier and lifecycle policies on the bucket.** S3 lifecycle rules that transition objects to Glacier will silently break restores — Elasticsearch's S3 plugin does not initiate a thaw. If you want long-term cold archival, use a source-only repo or a dedicated archive bucket without lifecycle rules.

**Repository concurrency.** Two clusters writing to the *same* `base_path` is unsupported and will corrupt the repo. If you have both a primary and a DR cluster, give each its own `base_path` (or its own bucket). Use `readonly: true` for any cluster that should only restore.

**Snapshot does not capture security state.** User accounts, role mappings, API keys live in the `.security` system index; `include_global_state: true` includes them in the snapshot, but restoring them on a different cluster overwrites that cluster's security state. Most teams keep authentication infrastructure (LDAP, SSO) external to ES so the snapshot doesn't have to carry it.

---

## Related

- [Foundations — Inverted Index Internals](../foundations/01-inverted-index-internals.md) — segment immutability is what makes incremental snapshots possible
- _ILM and Hot/Warm/Cold/Frozen Tiers (planned)_ — the policy that drives indices into searchable-snapshot storage
- _Cluster Sizing and Capacity Planning (planned)_ — how snapshot bandwidth and S3 cost factor into a cluster budget
- [Database — Backup and Restore Strategies](../../database/INDEX.md) — pg_basebackup / WAL archiving as the relational analog
- [Kubernetes — StatefulSets and Storage](../../kubernetes/INDEX.md) — when ES runs on K8s, snapshot-to-S3 is usually the only durable backup; PVC snapshots alone are not enough

## References

- [Elasticsearch Reference — Snapshot and Restore](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html) — overview of the repository model, snapshot semantics, and restore options
- [Elasticsearch Reference — Register a Snapshot Repository](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-register-repository.html) — the type-specific settings for `fs`, `url`, `s3`, `gcs`, `azure`, `hdfs`
- [Elasticsearch Reference — Snapshot Lifecycle Management (SLM)](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-lifecycle-management.html) — schedule, retention, and the `_slm` API
- [Elasticsearch Reference — Restore a Snapshot](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-restore-snapshot.html) — full vs partial restore, rename pattern, settings overrides, version compatibility
- [Elasticsearch Reference — Searchable Snapshots](https://www.elastic.co/guide/en/elasticsearch/reference/current/searchable-snapshots.html) — frozen-tier mechanics and the `_mount` API
- [Elasticsearch Reference — S3 Repository Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/current/repository-s3.html) — credentials, storage classes, and the Glacier caveat
- [Elasticsearch Reference — GCS Repository Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/current/repository-gcs.html)
- [Elasticsearch Reference — Azure Repository Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/current/repository-azure.html)
- [OpenSearch Documentation — Snapshots](https://opensearch.org/docs/latest/tuning-your-cluster/availability-and-recovery/snapshots/index/) — repository registration, snapshot/restore, and the `_sm` Snapshot Management API
- [OpenSearch Documentation — Searchable Snapshots](https://opensearch.org/docs/latest/tuning-your-cluster/availability-and-recovery/snapshots/searchable_snapshot/) — Apache-licensed equivalent of Elastic's frozen-tier feature
- [Elastic Blog — Incremental Snapshots in Elasticsearch](https://www.elastic.co/blog/found-elasticsearch-snapshot-and-restore) — segment-level dedup model, how the manifest references shared files
