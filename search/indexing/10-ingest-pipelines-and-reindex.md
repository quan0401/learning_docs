---
title: "Ingest Pipelines and the Reindex API — In-Cluster ETL and Bulk Document Movement"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, ingest, reindex]
---

# Ingest Pipelines and the Reindex API — In-Cluster ETL and Bulk Document Movement

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `ingest` `reindex`

---

## Table of Contents

- [Summary](#summary)
- [1. What an Ingest Pipeline Is, and Where It Runs](#1-what-an-ingest-pipeline-is-and-where-it-runs)
- [2. Default and Final Pipelines](#2-default-and-final-pipelines)
- [3. The Processor Catalog You Will Actually Use](#3-the-processor-catalog-you-will-actually-use)
- [4. Failure Handling — `on_failure` at Two Levels](#4-failure-handling--on_failure-at-two-levels)
- [5. The Simulate API — Test Before You Index](#5-the-simulate-api--test-before-you-index)
- [6. The Reindex API — Local, Filtered, Transformed](#6-the-reindex-api--local-filtered-transformed)
- [7. Reindex from Remote](#7-reindex-from-remote)
- [8. Throughput Tuning — Slices, Throttle, Batch Size](#8-throughput-tuning--slices-throttle-batch-size)
- [9. Update by Query and Delete by Query](#9-update-by-query-and-delete-by-query)
- [10. Common Patterns](#10-common-patterns)
- [11. Practical Traps](#11-practical-traps)
- [Related](#related)
- [References](#references)

---

## Summary

Ingest pipelines and the Reindex API are the two in-cluster mechanisms Elasticsearch and OpenSearch give you for transforming documents at index time and for moving documents between indices after the fact. An **ingest pipeline** is a named, ordered sequence of *processors* (set, rename, grok, script, geoip, …) that runs on an *ingest node* before each document is written to its destination shard — think of it as cluster-internal ETL with no extra hop. The **Reindex API** is a server-side bulk read-then-bulk-write operation that pulls documents from a source index, optionally transforms them, and writes them to a destination index, and it can do the same across clusters when the source side is whitelisted.

For a TS/Node or Java/Spring Boot backend dev, the rough mental model is: ingest pipelines replace a thin transformation step you would otherwise put in a Logstash filter, a Kafka Connect SMT, or your own writer service; the Reindex API replaces hand-rolled `scroll → bulk` Node scripts you would write to migrate data between indices. Both run inside the cluster, both consume cluster CPU/heap/IO, and both have non-obvious failure and throughput knobs. This doc walks the controls, the processor catalog, and the patterns that come up repeatedly in real backfills.

---

## 1. What an Ingest Pipeline Is, and Where It Runs

A pipeline is a JSON object with a `description` and a `processors` array, registered via `PUT _ingest/pipeline/<name>`. Each processor receives the document (as `ctx`), mutates fields, and passes it to the next. The whole pipeline runs *before* Lucene sees the doc — what lands in `_source` and in the inverted index is the post-pipeline doc.

```json
PUT _ingest/pipeline/access-log-pipeline
{
  "description": "Parse and enrich access logs at index time",
  "processors": [
    { "grok": { "field": "message", "patterns": ["%{COMBINEDAPACHELOG}"] } },
    { "date": { "field": "timestamp", "formats": ["dd/MMM/yyyy:HH:mm:ss Z"] } },
    { "geoip": { "field": "clientip", "target_field": "geo" } },
    { "remove": { "field": "message" } }
  ]
}
```

Pipelines run on **ingest nodes** — every Elasticsearch node has the `ingest` role enabled by default; in larger clusters you typically dedicate ingest-only nodes so pipeline CPU doesn't contend with shard routing or query work. The coordinating node forwards each document to an ingest node for pipeline execution and then on to the primary shard. OpenSearch keeps the same model and role name.

Apply a pipeline by naming it on the request — `POST my-index/_doc?pipeline=access-log-pipeline` — or by setting it as the index's default pipeline so writers don't have to know about it.

---

## 2. Default and Final Pipelines

Two index settings let you bind pipelines to an index instead of to each request:

- **`index.default_pipeline`** — runs if the request does not name a pipeline. Use this to attach standard parsing logic to an index without changing client code.
- **`index.final_pipeline`** — runs *after* the user-specified or default pipeline. Use this for cluster-wide invariants (audit fields, retention metadata, the `event.ingested` timestamp) that should always run last regardless of what the writer asked for.

```json
PUT my-index/_settings
{
  "index.default_pipeline": "access-log-pipeline",
  "index.final_pipeline": "audit-stamp-pipeline"
}
```

A request that explicitly sets `?pipeline=other` overrides the default but still runs the final pipeline afterwards. This is the standard place to put an immutable `ingested_at` timestamp or a tenant-tagging step.

---

## 3. The Processor Catalog You Will Actually Use

The full processor list in the Elastic reference is over forty entries; the ones that come up in real pipelines are a small subset. The names below are exact and match the Elastic and OpenSearch ingest references.

| Processor | What it does |
|-----------|--------------|
| `set` | Assign a field to a literal or templated value |
| `remove` | Drop one or more fields |
| `rename` | Move a field to a new name |
| `convert` | Coerce a field to `integer`, `long`, `float`, `double`, `boolean`, `string`, `ip`, or `auto` |
| `date` | Parse a string into a date and write to `@timestamp` (or a target field) |
| `grok` | Regex-with-named-captures parser (Logstash heritage); slow on complex patterns |
| `dissect` | Faster, simpler delimiter-based extraction; preferred over `grok` when the input is structured |
| `script` | Run a Painless script — the escape hatch for anything the declarative processors can't express |
| `pipeline` | Invoke another pipeline by name; lets you compose reusable building blocks |
| `enrich` | Lookup against an *enrich index* built from a source index — joins data at index time |
| `geoip` | Resolve an IP address to country/city/lat-lon via the bundled MaxMind-format database |
| `user_agent` | Parse a User-Agent string into structured fields |
| `fingerprint` | Hash selected fields into a deterministic ID (commonly used for de-dup keys) |

A few notes worth internalizing:

- **`dissect` over `grok` when the input has fixed delimiters** — dissect is much cheaper per document; grok's regex engine dominates pipeline CPU when used carelessly.
- **`enrich` requires an explicit enrich policy** — you build a policy that snapshots a source index, then reference it from the processor. Lookups hit the snapshot, not live data.
- **`script` is Painless** — sandboxed, compiled and cached, with `ctx` access. Use it where declarative processors can't reach, but treat it as the most expensive tool in the box.

---

## 4. Failure Handling — `on_failure` at Two Levels

By default, a processor failure aborts the document and Elasticsearch responds with a bulk-item error. You override that by attaching an `on_failure` block at the **processor level** (handles only that processor's failures) or at the **pipeline level** (handles any unhandled processor failure).

```json
{
  "processors": [
    {
      "rename": {
        "field": "old_name",
        "target_field": "new_name",
        "on_failure": [
          { "set": { "field": "rename_error", "value": "{{ _ingest.on_failure_message }}" } }
        ]
      }
    }
  ],
  "on_failure": [
    { "set": { "field": "_index", "value": "failed-{{ _index }}" } },
    { "set": { "field": "error.message", "value": "{{ _ingest.on_failure_message }}" } }
  ]
}
```

The `_ingest.on_failure_processor_type`, `_ingest.on_failure_processor_tag`, and `_ingest.on_failure_message` metadata fields are populated inside the failure handler. The pipeline-level pattern above is the standard "dead-letter" approach: rewrite `_index` to a per-index failure bucket so bad documents land somewhere triageable instead of being rejected. A second knob, `ignore_failure: true` on a single processor, swallows that processor's error silently — use only for genuinely optional steps (e.g. a `geoip` lookup against a private IP).

---

## 5. The Simulate API — Test Before You Index

The Simulate API runs a pipeline against documents you provide and returns the post-pipeline document, **without** writing anything. It is the equivalent of running unit tests on your pipeline.

```bash
POST _ingest/pipeline/access-log-pipeline/_simulate
{
  "docs": [
    { "_source": { "message": "127.0.0.1 - - [01/Jan/2026:10:00:00 +0000] \"GET / HTTP/1.1\" 200 1234" } }
  ]
}
```

You can also simulate an unsaved pipeline by inlining it under a `pipeline` key alongside `docs`. Add `?verbose=true` to see the document state after each processor — invaluable when debugging a `grok` pattern or a chained `enrich`. Treat the Simulate API as the test harness in your pipeline development loop.

---

## 6. The Reindex API — Local, Filtered, Transformed

The Reindex API copies documents from a source index to a destination index. Internally it does a server-side scroll-and-bulk: there is no client streaming, the work happens entirely on the data nodes.

```bash
POST _reindex
{
  "source": { "index": "events-2025" },
  "dest":   { "index": "events-2025-v2" }
}
```

Three refinements you will reach for constantly:

**Query filter** — copy a subset by adding a `query` to `source`:

```json
{
  "source": { "index": "events-2025", "query": { "term": { "tenant_id": "acme" } } },
  "dest":   { "index": "events-2025-acme" }
}
```

**Pipeline transformation** — run an ingest pipeline on every document on the way in. This is the canonical backfill: write the new mapping, write a pipeline that converts old fields to new, reindex through it.

```json
{
  "source": { "index": "events-2025" },
  "dest":   { "index": "events-2025-v2", "pipeline": "events-rewrite-v2" }
}
```

**Inline `script`** — for one-off field rewrites where a full pipeline is overkill:

```json
{
  "source": { "index": "events-2025" },
  "dest":   { "index": "events-2025-v2" },
  "script": { "source": "ctx._source.severity = ctx._source.severity?.toUpperCase()" }
}
```

The script can also set `ctx.op = 'noop'` to skip a doc or `'delete'` to drop it. Run async with `?wait_for_completion=false` to get a `task_id` and poll the Tasks API instead of holding an HTTP connection open for hours.

---

## 7. Reindex from Remote

The same API can pull from a *different* cluster by adding a `remote` block in `source`. The destination cluster must explicitly whitelist the source host in `elasticsearch.yml`:

```yaml
reindex.remote.whitelist: ["oldcluster.example.com:9200", "10.0.0.5:9200"]
```

Then:

```json
POST _reindex
{
  "source": {
    "remote": {
      "host": "https://oldcluster.example.com:9200",
      "username": "reindex_user",
      "password": "...",
      "socket_timeout": "1m",
      "connect_timeout": "10s"
    },
    "index": "events-2025"
  },
  "dest": { "index": "events-2025" }
}
```

Practical considerations: the destination cluster opens HTTP(S) connections *to* the source (firewalls and ACLs need to permit that direction); documents flow as JSON over scroll batches and can saturate links faster than expected; each slice is a separate scroll on the source, so wide parallelism eats into `search.max_open_scroll_context`; and mappings are *not* copied — you create the destination with its mapping first.

---

## 8. Throughput Tuning — Slices, Throttle, Batch Size

A single-threaded reindex of a billion-document index will not finish in any reasonable time. Three knobs control throughput:

- **`slices`** — split the reindex into N parallel sub-tasks, each processing a disjoint slice of the source. `slices: "auto"` picks one slice per source shard. This is by far the highest-impact knob.
- **`requests_per_second`** — throttle in bulk-requests-per-second. Default `-1` (unthrottled). Reindex inserts artificial pauses between batches to hit the rate.
- **`source.size`** — batch size, default 1000 documents per bulk request. Increase for small docs, decrease for large docs to cap bulk payload size (held in heap on the receiver).

```json
POST _reindex?slices=auto&requests_per_second=500&wait_for_completion=false
{
  "source": { "index": "events-2025", "size": 500 },
  "dest":   { "index": "events-2025-v2" }
}
```

Tune by running async, watching the Tasks API for `created`/`updated`/`batches`, and using `POST _reindex/<task_id>/_rethrottle?requests_per_second=X` mid-flight if the cluster runs hot or idle.

---

## 9. Update by Query and Delete by Query

Two close cousins of the Reindex API operate **in place** on a single index:

```bash
POST events-2025/_update_by_query?conflicts=proceed
{
  "script": { "source": "ctx._source.tier = 'gold'" },
  "query":  { "term": { "customer_id": "acme" } }
}
```

```bash
POST events-2025/_delete_by_query
{
  "query": { "range": { "@timestamp": { "lt": "2024-01-01" } } }
}
```

They share the slicing, throttling, and `wait_for_completion=false` semantics with `_reindex`. Internally they are also scroll-and-bulk, just with the destination being the same index. The version conflict default is `abort`; pass `conflicts=proceed` to keep going past concurrent updates and report the count at the end.

The mental model: every "in-place" operation on an Elasticsearch index is really a delete-and-reinsert at the segment level (segments are immutable — see the inverted index doc in `foundations/`). Update-by-query just hides that.

---

## 10. Common Patterns

- **Extract field on read, then bulk reindex.** Old index has a useful value buried in a JSON blob: write a pipeline that uses `json` + `set` to lift the inner field, reindex through it into a new index with the right mapping. The writer service never changes.
- **Backfill via reindex with a script.** Field semantics changed (e.g. `severity` int → string). Reindex with a Painless script that rewrites the field, throttled with `?slices=auto&requests_per_second=N`.
- **GeoIP enrichment at index time.** Add `geoip` to the index's `default_pipeline`; every doc gets `geo.country_iso_code`, `geo.city_name`, `geo.location` populated. The MaxMind-format database ships with the cluster and refreshes via the GeoIP database service.
- **Per-tenant final pipeline.** A `final_pipeline` that stamps `ingested_at` and a tenant tag means writers cannot forget the audit fields — the cluster adds them last regardless of which user pipeline ran.
- **Dead-letter on parse failure.** A top-level `on_failure` that rewrites `_index` to `<original>-failed` makes bad docs visible instead of rejected; ops alerts on growth in `*-failed` indices.

---

## 11. Practical Traps

- **Ingest pipelines vs Logstash/Filebeat.** Same processor types, different CPU location. Ingest pipelines burn cluster CPU and pressure data nodes during write spikes; Logstash and Beats run outside the cluster. Rule of thumb: cheap (`set`, `rename`, `convert`, `date`) inside the cluster; heavy (multi-pattern `grok`, large enrichment, complex Painless) outside.
- **Painless cost is real.** Scripts are compiled and cached, but per-document execution still costs heap and CPU. A tight script that runs on every write will pin your ingest tier.
- **Large docs + reindex = memory pressure.** Batch size is in *documents*, not bytes. `size: 1000` against 200KB docs puts ~200MB of `_source` in memory per batch per slice. Lower `source.size` for large documents.
- **Reindex copies `_source`.** If `_source` is disabled on the source mapping, you cannot reindex from it — a strong argument against ever disabling `_source`.
- **Mappings do not auto-copy.** Reindex into a missing destination uses dynamic mapping, almost never what a backfill wants. Create the destination with the explicit mapping first.
- **Pipelines and aliases.** `default_pipeline` is an *index* setting, not alias. For rollover aliases, set it in the index template so new backing indices inherit.
- **Scroll contexts are bounded.** Wide `slices` open one scroll per slice on the source; `search.max_open_scroll_context` (default 500) caps the cluster-wide count, and remote reindex against a busy source can hit it.

---

## Related

- [Inverted Index Internals — Postings Lists, Skip Lists, and FSTs](../foundations/01-inverted-index-internals.md) — why "update in place" is delete-and-reinsert at the segment level
- _Mappings, Dynamic Mapping, and Index Templates (planned)_ — the mapping side of reindex backfills
- _Aliases, Rollover, and Index Lifecycle Management (planned)_ — where `default_pipeline` and rollover meet
- [Observability — Structured Logging](../../observability/INDEX.md) — the canonical upstream for log pipelines that target Elasticsearch
- [Operating Systems — Page Cache and Buffered I/O](../../operating-systems/operating-systems-tier1/03-page-cache-and-buffered-io.md) — why the receiving cluster's IO budget caps reindex throughput more often than CPU does

## References

- [Elasticsearch Reference — Ingest pipelines](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html) — pipeline lifecycle, default and final pipelines, ingest node role
- [Elasticsearch Reference — Ingest processor reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/processors.html) — authoritative processor list and parameter docs
- [Elasticsearch Reference — Handling pipeline failures](https://www.elastic.co/guide/en/elasticsearch/reference/current/handling-pipeline-failures.html) — `on_failure`, `ignore_failure`, `_ingest.on_failure_*` metadata
- [Elasticsearch Reference — Simulate pipeline API](https://www.elastic.co/guide/en/elasticsearch/reference/current/simulate-pipeline-api.html)
- [Elasticsearch Reference — Reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html) — query, script, slices, throttling, reindex-from-remote
- [Elasticsearch Reference — Update by query](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update-by-query.html) and [Delete by query](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-delete-by-query.html)
- [Elasticsearch Reference — Enrich processor](https://www.elastic.co/guide/en/elasticsearch/reference/current/enrich-processor.html) and [Set up an enrich processor](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest-enriching-data.html)
- [Elasticsearch Reference — GeoIP processor](https://www.elastic.co/guide/en/elasticsearch/reference/current/geoip-processor.html) — bundled MaxMind-format database and update service
- [Elasticsearch Reference — Painless scripting language](https://www.elastic.co/guide/en/elasticsearch/painless/current/index.html)
- [OpenSearch Documentation — Ingest pipelines](https://opensearch.org/docs/latest/ingest-pipelines/) and [Reindex document API](https://opensearch.org/docs/latest/api-reference/document-apis/reindex/)
