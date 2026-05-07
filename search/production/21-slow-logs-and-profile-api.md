---
title: "Slow Logs, Profile API, and Hot Threads — Production Search Diagnostics"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, slow-log, profile, hot-threads, diagnostics]
---

# Slow Logs, Profile API, and Hot Threads — Production Search Diagnostics

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `slow-log` `profile` `hot-threads` `diagnostics`

---

## Table of Contents

- [Summary](#summary)
- [1. The Three-Tool Diagnostic Triangle](#1-the-three-tool-diagnostic-triangle)
- [2. Slow Logs — Search Phase and Fetch Phase](#2-slow-logs--search-phase-and-fetch-phase)
- [3. Indexing Slow Log — The Write-Path Equivalent](#3-indexing-slow-log--the-write-path-equivalent)
- [4. The Profile API](#4-the-profile-api)
- [5. Profile API Limitations and When Not to Use It](#5-profile-api-limitations-and-when-not-to-use-it)
- [6. Hot Threads — Stack-Sample the Busy Threads](#6-hot-threads--stack-sample-the-busy-threads)
- [7. Tasks API and Cancellation](#7-tasks-api-and-cancellation)
- [8. Expensive Query Patterns to Recognize](#8-expensive-query-patterns-to-recognize)
- [9. Cache Stats from `_nodes/stats/indices`](#9-cache-stats-from-_nodesstatsindices)
- [10. A Practical Playbook](#10-a-practical-playbook)
- [Related](#related)
- [References](#references)

---

## Summary

When a search cluster gets slow, three diagnostic tools answer three different questions. **Slow logs** answer "which queries crossed my latency budget?" — they record per-shard query/fetch timings into a separate log file based on per-index threshold settings. **The Profile API** answers "*why* is this one query slow?" — it returns a per-shard, per-collector, per-Lucene-clause breakdown of where time is spent (`next_doc`, `advance`, `score`, `build_scorer`). **Hot threads** answers "what is the cluster *currently* doing?" — it samples stack traces from the busiest threads on each node, which is usually how you diagnose a live incident before any of the other tools have had a chance to record anything useful.

The hard-won production rule: **hot threads is your first call when the cluster is slow right now**, slow logs are how you find recurring offenders over time, and Profile API is a development-only tool you never enable on production traffic because the instrumentation overhead is significant. This doc walks the configuration knobs for each tool, names the actual settings (`index.search.slowlog.threshold.query.warn` and friends), and lists the expensive query patterns — deep `from + size` pagination, leading-wildcard regexes, high-cardinality `terms` aggregations, per-doc scripts — that show up over and over in hot threads dumps.

---

## 1. The Three-Tool Diagnostic Triangle

| Question | Tool | Cost |
|----------|------|------|
| Which queries are slow over time? | Slow log (search/fetch/index) | Cheap — only logs above threshold |
| Why is *this* query slow? | `_search?profile=true` | Significant per-query overhead |
| What is the cluster doing right now? | `_nodes/hot_threads` | Cheap — passive stack sampling |
| What long-running tasks exist? | `_tasks` | Cheap — a metadata read |
| What is the cache effectiveness? | `_nodes/stats/indices` | Cheap — a metadata read |

The mistake junior operators make is reaching for the Profile API during an incident. Profile is for *post-mortem reproduction in a dev cluster*. During an incident the question is "what is happening *now*", and only hot threads + tasks answer that without making things worse.

---

## 2. Slow Logs — Search Phase and Fetch Phase

A search request is two phases: a **query phase** (every shard runs the query, returns top-N doc IDs and scores) and a **fetch phase** (the coordinating node asks the shards holding the global top-N to return their `_source` and stored fields). Elasticsearch and OpenSearch maintain separate slow-log thresholds for each, because slow query phases mean expensive scoring while slow fetch phases usually mean large `_source` documents or excessive highlighting.

Both are **per-index** dynamic settings — they are not cluster-level. Each threshold defaults to `-1` (disabled), and you set thresholds per log level. A request that exceeds the highest threshold whose level is enabled gets logged at that level.

```bash
# Enable search slow log on a single index
PUT /orders/_settings
{
  "index.search.slowlog.threshold.query.warn":  "5s",
  "index.search.slowlog.threshold.query.info":  "2s",
  "index.search.slowlog.threshold.query.debug": "500ms",
  "index.search.slowlog.threshold.query.trace": "100ms",

  "index.search.slowlog.threshold.fetch.warn":  "1s",
  "index.search.slowlog.threshold.fetch.info":  "500ms",
  "index.search.slowlog.threshold.fetch.debug": "200ms",
  "index.search.slowlog.threshold.fetch.trace": "50ms"
}
```

Two important model points:

- **Thresholds measure per-shard time, not coordinator-observed total time.** A query that takes 3 s end-to-end because it fans out to 30 shards each taking 100 ms will not trigger a `query.warn` set at `1s` — every shard was under the threshold. This is a frequent source of "why isn't my slow log catching anything?"
- **Slow logs land in a separate log file** (`<cluster>_index_search_slowlog.json`) configured by `log4j2.properties`, not in the main Elasticsearch log. If you ship logs to Loki/Elastic, make sure you ship that file too.

You can also set `index.search.slowlog.include.user: true` (Elasticsearch 8.x+) to include the authenticated user in each slow-log entry, useful when one tenant's queries are degrading shared infrastructure.

OpenSearch supports the same threshold settings under the same names, with one operational divergence: OpenSearch 2.12+ introduced cluster-level slow-log default settings (`cluster.search.request.slowlog.threshold.warn` etc.) that apply to the *coordinator-observed* request time across all indices, complementing the per-shard per-index settings inherited from Elasticsearch.

---

## 3. Indexing Slow Log — The Write-Path Equivalent

The same shape exists for the indexing path, with thresholds named under `index.indexing.slowlog`:

```bash
PUT /orders/_settings
{
  "index.indexing.slowlog.threshold.index.warn":  "10s",
  "index.indexing.slowlog.threshold.index.info":  "5s",
  "index.indexing.slowlog.threshold.index.debug": "2s",
  "index.indexing.slowlog.threshold.index.trace": "500ms",

  "index.indexing.slowlog.source": "1000"
}
```

`source` controls how many characters of the document body get logged with each entry — `0` to disable, `true` for the whole document, an integer to truncate. Logging the whole `_source` of every slow indexing call is a great way to leak PII into your log pipeline, so default this to a small truncation or `0` and only crank it up while debugging.

Slow indexing entries usually fingerprint as one of: an analyzer doing too much work (a custom tokenizer, deep ngram, or heavy synonym graph), a refresh storm (refresh interval too short for the write rate), or merge pressure backpressuring the indexing thread.

---

## 4. The Profile API

`_search?profile=true` (or `"profile": true` in the request body) returns a per-shard breakdown of query execution time. The response payload includes, for each shard:

- The **rewritten query** — Lucene rewrites your DSL into its internal query graph; the profile shows the post-rewrite tree, which is often very different from what you sent.
- A **breakdown of timing per Lucene method** for each leaf of the query tree.
- A **collectors** section showing how the top-N collector composed work across the segments.
- An **aggregations** section if you sent aggregations, with per-aggregation timing.

The breakdown fields you will see most often:

| Field | What it measures |
|-------|------------------|
| `build_scorer` | Time to construct the per-segment scorer (one-time setup cost per leaf) |
| `next_doc` | Time advancing to the next matching doc in the postings stream |
| `advance` | Time skipping to a specific target doc ID (used in conjunctions / `AND`) |
| `score` | Time computing the BM25 (or similarity) score for matched docs |
| `match` | Time evaluating two-phase iterators (e.g., phrase queries that confirm proximity after an initial postings match) |
| `create_weight` | One-time per-segment weight construction |
| `shallow_advance` | Used by `BlockMaxConjunctionScorer` for skipping over non-competitive blocks |

A typical example of an expensive query: a phrase query against a high-frequency leading term will show large `next_doc` time on the leading term's postings combined with large `match` time on the proximity verifier. A leading-wildcard query (`*foo`) will show enormous `build_scorer` time because Lucene has to enumerate every term that matches the pattern from the FST before scoring can start.

```bash
GET /orders/_search?profile=true
{
  "query": {
    "bool": {
      "must": [
        { "match": { "description": "wireless headphones" } }
      ],
      "filter": [
        { "term": { "status": "shipped" } }
      ]
    }
  }
}
```

The response is large (on the order of tens of KB even for a simple query). Pipe it through a tool that summarizes per-shard time — Kibana's "Search Profiler" UI is the easiest way to read it; cold reading the raw JSON is possible but tedious.

---

## 5. Profile API Limitations and When Not to Use It

The Elastic docs are explicit: the Profile API has **significant overhead** and "should not be used in production settings" as a default-on tool. The instrumentation wraps every Lucene method call with timing logic, which both inflates the timings (the profiled query is slower than the un-profiled one) and adds GC pressure.

Other limitations the docs call out:

- **Suggesters, `_count`, and `_msearch` are not profiled** — sending `profile: true` against those returns no profile data.
- **The reported total time excludes** network transport between coordinator and data nodes, the queue-wait time on the search thread pool, and the `_source`/highlight/fetch phase work. A query whose Profile API total reads "120 ms" can comfortably take 800 ms wall-clock if it queued behind 50 other requests on a saturated thread pool.
- **Aggregation profiling** has its own section; some aggregation internals (like global ordinals build time on first use) appear as one-time costs that distort comparisons between cold and warm runs.
- **Profile output is per-shard.** A request that hit 30 shards returns 30 sections; you have to aggregate them yourself to get a full picture.

The right home for Profile API: a dev or staging cluster, with production data shape, running a single suspect query in isolation to understand its cost structure. Never wired into your production query path or your APM auto-instrumentation.

---

## 6. Hot Threads — Stack-Sample the Busy Threads

`_nodes/hot_threads` is the single most useful endpoint for "the cluster is slow right now." It samples stack traces from threads on each node, then returns the threads burning the most CPU/blocked time during the sample window.

```bash
GET /_nodes/hot_threads?threads=10&interval=500ms&type=cpu
```

Parameters worth knowing:

- `type` — `cpu` (default), `wait`, `block`, `mem`, or `gpu`. Use `wait` to find threads stuck on I/O, `block` to find lock contention.
- `interval` — how long to sample. Default 500 ms; increase to 5 s if you want better coverage.
- `threads` — how many top threads per node to return. Default 3; bump to 10 for noisy clusters.
- `snapshots` — how many stack samples to take per thread within the interval. Default 10.

Patterns to recognize when reading the output:

- **`[search]` thread saturation.** Multiple search threads on multiple nodes all showing `LeafReader.terms` / `next_doc` / `score` in their stacks means you are postings-bound — your queries are reading too many docs. Look for missing filters, wildcard queries, or high-cardinality aggs.
- **`[merge]` thread dominating CPU.** A node where `org.apache.lucene.index.SegmentMerger.merge` shows up consistently is doing background merges. If this co-occurs with slow indexing, your refresh interval is too aggressive or your bulk size is too small (lots of tiny segments = lots of merges).
- **`[write]` threads stuck in `Refresh`.** Indicates refresh storms; consider raising `index.refresh_interval`.
- **GC threads at the top.** If `GC task thread` is your hottest thread, you are heap-bound. The actual work threads are starving for CPU because GC keeps preempting them.
- **`epollWait` / `await` everywhere.** Cluster is idle from a CPU standpoint. The slowness is somewhere else (network, downstream API, client-side).

A senior operator's reflex: SSH-equivalent into the cluster (`curl _nodes/hot_threads`) within the first minute of a slow incident, capture two samples 30 s apart, then look for what changed.

---

## 7. Tasks API and Cancellation

The `_tasks` endpoint lists every in-flight task on every node — searches, reindexes, deletes-by-query, snapshot operations, etc. Each task has an ID, an action name (`indices:data/read/search`), a parent task ID, a running-time-in-nanos, and (sometimes) a cancellable flag.

```bash
GET /_tasks?detailed=true&actions=*search*

POST /_tasks/<task-id>/_cancel
```

Most useful in two scenarios:

- **A user kicked off a runaway query.** A `_search` with a million-bucket `terms` aggregation can run for minutes. Find it in `_tasks`, cancel it, and the search threads it occupies free up immediately. Note that not all task types are cancellable — search and reindex are; some internal cluster tasks are not.
- **A reindex or update-by-query is grinding.** These tasks expose a `status` block with `total`, `updated`, `created`, `deleted`, and `version_conflicts`, which is how you estimate ETA and whether you should cancel and re-shape the operation.

Search cancellation is also cooperative: Elasticsearch checks for cancellation at safe points during query execution, so a cancelled query stops within milliseconds-to-seconds, not instantly.

---

## 8. Expensive Query Patterns to Recognize

The same handful of patterns generate the bulk of search-tier incidents. Spot them in slow logs and Profile output.

### 8.1 Deep pagination

`from + size` greater than `index.max_result_window` (default `10000`) is rejected outright. Below that threshold but still deep (e.g., `from: 9000, size: 100`) is legal but expensive: every shard sorts `from + size` results, ships them to the coordinator, which merges and discards the first `from`. With 30 shards and `from: 9000`, the cluster sorts 273,000 hits to return 100. Use `search_after` with a tiebreaker sort, or `point_in_time` for stable cursors.

### 8.2 Leading wildcards

`*foo` (or `name:*foo`) cannot use the FST efficiently — the engine has to enumerate the full terms dictionary looking for matches. Profile shows huge `build_scorer` time. Either use `wildcard` field type (Elasticsearch 7.9+, optimized for this), or rewrite the data with reverse-indexed text and search the reverse, or use ngram tokenization at index time.

### 8.3 Regex with backtracking

`regexp` queries with patterns like `.*foo.*` or nested quantifiers can backtrack catastrophically. Elasticsearch's `index.allow_expensive_queries` setting (default `true`) lets you disable these cluster-wide; a stricter setup turns it off and forces query authors to use `wildcard` field types and `keyword` matching instead.

### 8.4 High-cardinality `terms` aggregation

`terms` aggregations build a hash table of bucket values. A `terms` agg on a `user_id` field across 100M unique users is a 100M-entry hash table per shard, bounded only by the `size` parameter (default 10) for what gets *returned* — but the engine still enumerates many more candidates internally if `shard_size` (default `size * 1.5 + 10`) is small relative to true cardinality. Use composite aggregations for paginated drilldowns; use `cardinality` (HyperLogLog) for counts only.

### 8.5 Scripts on every doc

A `script_score` or `painless`-based runtime field evaluated per matched doc multiplies the per-doc cost by the script's runtime. A 10 µs script on a 1M-matching-doc query adds 10 s of CPU. Move computed values to index-time fields when possible; profile aggressively when you cannot.

### 8.6 Unfiltered queries

A `match_all` over a billion-doc index is fast (it just walks the live-docs bitmap) — but a `match` against a high-frequency term with no filter to narrow the candidate set is slow. The fix is almost always a `bool` with `filter` clauses that prune the candidate set before scoring.

---

## 9. Cache Stats from `_nodes/stats/indices`

Three caches drive search performance, and `GET /_nodes/stats/indices` exposes hit rates for all of them.

```bash
GET /_nodes/stats/indices/query_cache,request_cache,fielddata
```

| Cache | What it caches | Look for |
|-------|----------------|----------|
| **Query cache** (node-level) | Per-segment bitset results of cacheable filter clauses | Low hit rate = filters too varied; large `cache_count` with low `hit_count` means churn |
| **Request cache** (shard-level) | Full search responses for `size: 0` requests (typically aggregations) on unchanged shards | Hit rate near 0 = enable it explicitly with `index.requests.cache.enable: true` |
| **Fielddata cache** | Heap-resident `text` field data for sort/aggs (legacy — use `keyword` + doc values instead) | Any non-trivial use is a smell; convert to `keyword` |

A high `evictions` counter on any cache means it is too small for your working set; a high `memory_size_in_bytes` close to the configured limit (`indices.queries.cache.size`, default 10% of heap) indicates pressure.

The query cache is *only used for queries inside `filter` clauses* — putting a filter inside `must` defeats it. This is one of the highest-leverage refactors: move every non-scoring predicate into `filter`, and watch query cache hit rate climb.

---

## 10. A Practical Playbook

A condensed runbook combining the above:

1. **Cluster slow right now?** First call: `GET /_nodes/hot_threads?threads=10&interval=2s`. Capture two samples 30 s apart. Look for search-thread saturation, merge dominance, or GC at the top.
2. **In parallel:** `GET /_tasks?detailed=true&actions=*search*` — find any task running >5 s. Cancel obvious offenders.
3. **Slow over time?** Make sure search slow logs are on with thresholds tied to your SLO (e.g., if your p95 SLO is 200 ms, set `query.warn` to 500 ms and `query.info` to 200 ms). Aggregate slow log entries by query fingerprint (the JSON formatter helps).
4. **Suspect query identified?** Reproduce in dev, run with `?profile=true`, read it in Kibana Search Profiler. Look for high `build_scorer` (wildcard / regex), high `next_doc` (cardinality), high `score` (expensive scoring).
5. **Caches:** Verify filters live in `filter` clauses, not `must`. Watch query/request cache hit rates on `_nodes/stats/indices`.
6. **Never enable Profile API on production traffic.** It is a debugging tool, not telemetry.

The discipline of "hot threads first, slow logs for trends, profile in dev only" handles 80% of search incidents without anyone needing to be a Lucene committer.

---

## Related

- [Inverted Index Internals — Postings, Skip Lists, FSTs](../foundations/01-inverted-index-internals.md) — explains why the Profile API fields `next_doc` and `advance` exist as separate measurements
- _Refresh, Flush, and Merge — Why Indexing is Slow (planned)_ — the write-path equivalent of this doc, covers the merge thread pattern from §6
- _Search Thread Pool Sizing and Backpressure (planned)_ — context for why hot threads showing search saturation often means thread pool, not Lucene, is the bottleneck
- [Observability — RED and USE Method](../../observability/INDEX.md) — the framework this playbook sits inside
- [Performance — Tail Latency and Percentiles](../../performance/INDEX.md) — why your slow log thresholds should be tied to a percentile SLO, not an average
- [Operating Systems — Page Cache and Buffered I/O](../../operating-systems/operating-systems-tier1/03-page-cache-and-buffered-io.md) — hot threads showing `pread`/IO wait usually means the page cache is cold

## References

- [Elasticsearch Reference — Search slow log](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-slowlog.html) — authoritative source for the `index.search.slowlog.threshold.query.{warn,info,debug,trace}` and fetch settings, including the `-1` disabled default and the per-shard measurement semantics
- [Elasticsearch Reference — Index slow log](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-slowlog.html#index-slow-log) — `index.indexing.slowlog.threshold.index.*` settings and the `source` truncation parameter
- [Elasticsearch Reference — Profile API](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-profile.html) — the official documentation including the explicit warning that "the Profile API has a non-negligible overhead" and the list of unsupported request types
- [Elasticsearch Reference — Nodes hot threads API](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-nodes-hot-threads.html) — `threads`, `interval`, `snapshots`, `type` parameters covered in §6
- [Elasticsearch Reference — Task management API](https://www.elastic.co/guide/en/elasticsearch/reference/current/tasks.html) — `_tasks` listing, the `_cancel` endpoint, and the cooperative cancellation model
- [Elasticsearch Reference — Tune for search speed](https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-search-speed.html) — covers expensive query patterns, the role of filters and the query cache, and the `index.max_result_window` default
- [Elasticsearch Reference — Paginate search results](https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html) — `from + size` limits and the `search_after` / `point_in_time` alternatives referenced in §8.1
- [Elasticsearch Reference — Node stats API](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-nodes-stats.html) — schema for the query cache, request cache, and fielddata stats sections used in §9
- [Elasticsearch Reference — `index.allow_expensive_queries`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html#query-dsl-allow-expensive-queries) — cluster-wide kill switch for backtracking regex, leading wildcards, and other expensive patterns
- [OpenSearch Documentation — Search slow log](https://opensearch.org/docs/latest/install-and-configure/configuring-opensearch/logs/) — OpenSearch parity on the slow log settings inherited from Elasticsearch and the cluster-level request slowlog additions
- [OpenSearch Documentation — Profile API](https://opensearch.org/docs/latest/api-reference/profile/) — OpenSearch's equivalent of the Profile API with the same `next_doc` / `advance` / `score` / `build_scorer` breakdown semantics
