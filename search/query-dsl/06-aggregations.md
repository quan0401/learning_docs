---
title: "Aggregations — Buckets, Metrics, Pipelines, and the Approximation Tradeoffs"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, aggregations, analytics, hll]
---

# Aggregations — Buckets, Metrics, Pipelines, and the Approximation Tradeoffs

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `aggregations` `analytics` `hll`

---

## Table of Contents

- [Summary](#summary)
- [1. Three Families and How They Compose](#1-three-families-and-how-they-compose)
- [2. Bucket Aggregations](#2-bucket-aggregations)
- [3. Metric Aggregations](#3-metric-aggregations)
- [4. Pipeline Aggregations](#4-pipeline-aggregations)
- [5. doc_values, fielddata, and the Text-Field Trap](#5-doc_values-fielddata-and-the-text-field-trap)
- [6. Approximation #1 — Cardinality (HyperLogLog++)](#6-approximation-1--cardinality-hyperloglog)
- [7. Approximation #2 — Terms Across Shards](#7-approximation-2--terms-across-shards)
- [8. Approximation #3 — Percentiles (t-digest vs HDR)](#8-approximation-3--percentiles-t-digest-vs-hdr)
- [9. Cost Knobs and Circuit Breakers](#9-cost-knobs-and-circuit-breakers)
- [10. Practical Patterns](#10-practical-patterns)
- [Related](#related)
- [References](#references)

---

## Summary

Aggregations are the analytical half of Elasticsearch / OpenSearch. Where a query returns matching documents ranked by relevance, an aggregation walks those documents and summarizes them — group-by-status, average-latency, p99-by-region, count-of-distinct-users. The execution model is column-oriented: aggregations read **doc values** (the `.dvd` files Lucene writes per segment), not the inverted index, so they look more like an OLAP scan than a search.

Three families compose into arbitrarily deep trees. **Bucket aggregations** partition the matched docs into groups (`terms`, `date_histogram`, `range`). **Metric aggregations** compute a number per bucket (`avg`, `percentiles`, `cardinality`). **Pipeline aggregations** run on the *output* of other aggregations to compute derivatives, moving averages, and inter-bucket scripts. Sub-aggregations nest inside buckets, which is how you express "for each status code, give me the p99 latency by hour."

Three of the most-used aggregations are **approximate**, not exact: `cardinality` (HyperLogLog++), `terms` across shards (top-N with shard_size over-fetch), and `percentiles` (t-digest by default, HDR optional). This doc walks the families, the doc-values requirement, and — critically — the three approximation tradeoffs so you know when "close enough" is actually close enough.

---

## 1. Three Families and How They Compose

A `_search` request body can contain an `aggs` block alongside (or instead of) a `query`. The query selects documents; the aggs operate on that selected set.

```json
POST /orders/_search
{
  "size": 0,
  "query": { "term": { "status": "shipped" } },
  "aggs": {
    "by_region": {
      "terms": { "field": "region", "size": 10 },
      "aggs": {
        "avg_total": { "avg": { "field": "total_cents" } },
        "p99_latency_ms": {
          "percentiles": { "field": "ship_latency_ms", "percents": [99] }
        }
      }
    }
  }
}
```

`size: 0` skips returning hit documents — common when the caller only wants the aggregation result. `by_region` is a bucket agg; `avg_total` and `p99_latency_ms` are metric sub-aggregations executed once per bucket. The response mirrors that shape: a `by_region.buckets` array, each with `key`, `doc_count`, and the metric outputs.

The three families:

| Family | Returns | Examples |
|--------|---------|----------|
| Bucket | A set of buckets, each with a `doc_count` and (recursively) sub-aggs | `terms`, `date_histogram`, `range`, `filter`, `composite` |
| Metric | A single number (or small object) computed across the bucket's docs | `avg`, `sum`, `cardinality`, `percentiles`, `top_hits` |
| Pipeline | A value computed from *other* aggregations' results | `derivative`, `moving_avg`, `bucket_sort`, `cumulative_sum` |

Pipeline aggs are siblings of the aggs they reference, identified via a `buckets_path` string.

---

## 2. Bucket Aggregations

### 2.1 `terms`

Group by a field value. The classic group-by:

```json
"by_status": { "terms": { "field": "status", "size": 5 } }
```

Defaults: `size: 10` returns the top 10 buckets ranked by `doc_count` descending. `shard_size` defaults to `size * 1.5 + 10` and controls how many candidate terms each shard contributes before the coordinating node merges. Both are documented in the official terms-aggregation reference. Why this matters is in §7 — across shards, top-N is fundamentally approximate.

```json
"by_status": {
  "terms": {
    "field": "status",
    "size": 10,
    "shard_size": 200,
    "show_term_doc_count_error": true
  }
}
```

`show_term_doc_count_error: true` returns a `doc_count_error_upper_bound` per bucket — the upper bound on how many docs might be missing from that bucket's count.

### 2.2 `date_histogram`

Group by time bucket. Uses calendar-aware intervals (`calendar_interval: "day"`, `"month"`) or fixed intervals (`fixed_interval: "30m"`, `"1h"`). `min_doc_count: 0` plus `extended_bounds` produces empty buckets for days with no data — required if downstream charts assume one entry per day.

```json
"by_day": {
  "date_histogram": {
    "field": "@timestamp",
    "calendar_interval": "day",
    "time_zone": "America/Los_Angeles",
    "min_doc_count": 0,
    "extended_bounds": { "min": "2026-01-01", "max": "2026-01-31" }
  }
}
```

### 2.3 `histogram`, `range`, `filter`, `filters`

`histogram` is `date_histogram` for numeric fields with a fixed `interval`. `range` accepts explicit ranges (e.g., `[{from: 0, to: 100}, {from: 100, to: 1000}]`). `filter` is a single bucket of docs matching a sub-query; `filters` produces named buckets, each from its own filter — useful for SLI/SLO splits or A/B/C breakdowns.

### 2.4 `composite`

Composite is the cursor-based aggregation. Standard `terms` returns the top N; if you have 5 million distinct keys and want every one in pages of 10k, `terms` cannot do that without `size: 5000000` (which the request circuit breaker will reject). `composite` walks the ordered combination of one or more sources and returns an `after_key` you pass back to fetch the next page:

```json
"all_status_region": {
  "composite": {
    "size": 1000,
    "sources": [
      { "status": { "terms": { "field": "status" } } },
      { "region": { "terms": { "field": "region" } } }
    ],
    "after": { "status": "shipped", "region": "us-west" }
  }
}
```

This is the only safe way to enumerate every distinct combination of high-cardinality fields. The tradeoff: composite is *not* a top-N — it walks in lex order of source values, so you cannot ask for "top 10 by doc_count globally" via composite.

---

## 3. Metric Aggregations

Single-number summaries:

| Agg | Output |
|-----|--------|
| `avg`, `sum`, `min`, `max` | One value |
| `stats` | `{count, min, max, avg, sum}` in one pass |
| `extended_stats` | Adds variance, std dev, std-dev bounds |
| `value_count` | Count of values (not docs — multi-valued fields count more) |
| `cardinality` | Approximate distinct-count via HyperLogLog++ (§6) |
| `percentiles` | Approximate percentiles via t-digest or HDR (§8) |
| `percentile_ranks` | Inverse: "what percentile is this value?" |
| `top_hits` | The top K *documents* per bucket — see §10 |

The `stats` family is preferable to running `min` + `max` + `avg` separately when you need all of them: one pass over doc values rather than three.

```json
"latency_stats": { "extended_stats": { "field": "latency_ms", "sigma": 3 } }
```

---

## 4. Pipeline Aggregations

Pipeline aggs run *after* their sibling aggregations finish and reference them through a `buckets_path` string. The result is a new field on each bucket (or a new bucket-level result).

### 4.1 `derivative`, `cumulative_sum`, `moving_fn`

```json
"by_day": {
  "date_histogram": { "field": "@timestamp", "calendar_interval": "day" },
  "aggs": {
    "orders": { "value_count": { "field": "_id" } },
    "orders_delta": { "derivative": { "buckets_path": "orders" } },
    "orders_running_total": { "cumulative_sum": { "buckets_path": "orders" } },
    "smoothed": {
      "moving_fn": {
        "buckets_path": "orders",
        "window": 7,
        "script": "MovingFunctions.unweightedAvg(values)"
      }
    }
  }
}
```

`derivative` gives day-over-day delta; `cumulative_sum` gives the running total; `moving_fn` (the modern replacement for `moving_avg`) takes a Painless script. The `MovingFunctions` namespace ships with `min`, `max`, `sum`, `unweightedAvg`, `linearWeightedAvg`, `ewma`, `holt`, and `holtWinters`.

### 4.2 `bucket_sort` and `bucket_script`

`bucket_sort` re-sorts and paginates the parent's buckets after metric aggs are computed — sort `terms` by a sub-agg's value, not by `doc_count`. `bucket_script` computes a derived value per bucket from other metric outputs (e.g., `params.revenue / params.orders` to get profit-per-order).

---

## 5. doc_values, fielddata, and the Text-Field Trap

Aggregations read **doc values**, Lucene's column-oriented per-document storage written to `.dvd`/`.dvm` files. Doc values exist by default for `keyword`, numeric, `date`, `boolean`, `ip`, and `geo_point` fields. They do *not* exist for `text` fields by default, because `text` goes through the analyzer and the post-tokenization tokens are not what you'd want to aggregate on anyway.

If you try to run `terms` on a `text` field, Elasticsearch fails the request unless `fielddata: true` is set on the mapping. Enabling fielddata loads an in-memory data structure built from the inverted index — it is expensive in heap, slow on first load, and almost always wrong. The right pattern:

```json
"mappings": {
  "properties": {
    "title": {
      "type": "text",
      "fields": {
        "keyword": { "type": "keyword", "ignore_above": 256 }
      }
    }
  }
}
```

Search on `title` (analyzed text), aggregate on `title.keyword` (raw value, doc-values backed). This is the canonical multi-field mapping, and it is the answer to ~90% of "why is my agg slow" questions on existing indexes.

---

## 6. Approximation #1 — Cardinality (HyperLogLog++)

`cardinality` returns approximate distinct-count using the **HyperLogLog++** algorithm from Heule et al. (Google, 2013). The single tunable knob is `precision_threshold`:

```json
"distinct_users": {
  "cardinality": { "field": "user_id", "precision_threshold": 3000 }
}
```

Per the official cardinality-aggregation reference: **default `precision_threshold` is 3000**, **maximum is 40000** (values above 40000 behave the same as 40000). Memory cost is roughly `precision_threshold * 8` bytes per shard per agg — so the 40000 ceiling is `~320 KB per shard`, not a lot.

Below the threshold, counts are highly accurate; above it, accuracy degrades gracefully. The Elastic docs note that "even with a threshold as low as 100, the error remains very low (1-6%)" when counting millions of items.

**When 40000 is enough:** daily-active-user dashboards where ±1% is invisible; distinct-IP counts for capacity planning; anything that drives a chart, not a billing line.

**When it isn't:** billing or audit counts where someone will ask "is this exact?" — for those, do a `composite` walk and count on the client, or maintain a separate exact counter. Also skip HLL for tiny cardinalities where exact is trivially cheap (50-value enum: just `terms` with `size: 50`).

---

## 7. Approximation #2 — Terms Across Shards

Top-N across distributed shards is **provably impossible to compute exactly** without shipping every distinct value from every shard to the coordinating node. Each shard returns its local top `shard_size`; the coordinating node merges and selects the global top `size` from the union.

The failure mode: a term ranks #11 on every shard but, summed across all shards, would actually be #3 globally. Each shard's local top-`shard_size` doesn't include it, so it never reaches the merge. `doc_count_error_upper_bound` bounds how many docs *might* have been missed for each returned bucket because the term wasn't returned by some shard.

Mitigations: increase `shard_size` much higher than `size` (the Elastic docs say outright: "It is much cheaper to increase the `shard_size` than to increase the `size`" — common pattern is `size: 10, shard_size: 1000`); set `show_term_doc_count_error: true` so the response carries the error bound; use `composite` if you need to enumerate every term exhaustively. There is no `shard_size` value short of "every distinct value on every shard" that gives provably exact top-N — this is a structural property of MapReduce-style approximate-top-N, not an Elasticsearch bug.

---

## 8. Approximation #3 — Percentiles (t-digest vs HDR)

`percentiles` defaults to **t-digest** (Ted Dunning's algorithm), a sketch that maintains a small ordered set of weighted centroids and supports merging across shards.

```json
"latency_pcts": {
  "percentiles": {
    "field": "latency_ms",
    "percents": [50, 95, 99, 99.9],
    "tdigest": { "compression": 100 }
  }
}
```

Default `compression` is 100, capping the digest at `20 * compression = 2000` nodes. Higher compression means more nodes, more accuracy, more memory. T-digest is good across the full value range and especially accurate at the tails (q=0.99, q=0.999), which is exactly the percentile region you actually care about for latency SLOs.

The alternative is **HDR Histogram** (also Ted Dunning, distinct codebase). Per the official percentile-aggregation reference, HDR "can be faster than the t-digest implementation with the trade-off of a larger memory footprint" and has fixed worst-case error based on a number-of-significant-digits parameter. Restrictions: HDR only supports positive values, and you must define the value range.

```json
"latency_pcts": {
  "percentiles": {
    "field": "latency_ms",
    "percents": [50, 95, 99],
    "hdr": { "number_of_significant_value_digits": 3 }
  }
}
```

Choose t-digest as the default. Switch to HDR when you have a fixed-range, positive-only value (typically latencies in microseconds or milliseconds with a known cap) and you've measured that t-digest's tail accuracy isn't enough.

---

## 9. Cost Knobs and Circuit Breakers

**`search.max_buckets`** defaults to **65,536** per the search-settings reference and caps total buckets in one response. Dynamic cluster setting (changeable without restart). A `terms` agg with `size: 100000` fails; deeply nested `terms { terms { date_histogram {} } }` whose Cartesian product exceeds the cap fails. The fix is rarely "raise the limit" — it's usually "use `composite` for full enumeration."

**Request circuit breaker.** Each shard tracks per-request in-flight memory (including aggregation state). Exceeding `indices.breaker.request.limit` (defaults around 60% of heap) fails with `CircuitBreakingException` rather than OOM-ing the node. Safety net for "I aggregated on a high-cardinality field and forgot."

**Bucket-source cardinality** is the single biggest cost driver. `terms` on a 10-value enum is cheap; `terms` on `user_id` with 50M distinct values is a node-killer even before `max_buckets` rejects it.

---

## 10. Practical Patterns

### 10.1 `top_hits` — representative docs per bucket

Get the top-scoring (or most-recent) document inside each bucket of a `terms` agg:

```json
"by_user": {
  "terms": { "field": "user_id", "size": 100 },
  "aggs": {
    "latest_order": {
      "top_hits": {
        "size": 1,
        "sort": [{ "@timestamp": "desc" }],
        "_source": { "includes": ["order_id", "total_cents", "@timestamp"] }
      }
    }
  }
}
```

This is the canonical "for each user, show their latest order" pattern. `_source.includes` keeps the response small.

### 10.2 `composite` for paginating analytics

```json
{
  "size": 0,
  "aggs": {
    "report": {
      "composite": {
        "size": 5000,
        "sources": [
          { "day": { "date_histogram": { "field": "@timestamp", "calendar_interval": "day" } } },
          { "region": { "terms": { "field": "region" } } }
        ]
      },
      "aggs": {
        "revenue": { "sum": { "field": "total_cents" } }
      }
    }
  }
}
```

Iterate, copying `aggregations.report.after_key` from each response into the next request's `composite.after`. Stop when the response has no `after_key`. This is how you stream a full grouped report into a downstream warehouse without ever exceeding `search.max_buckets`.

### 10.3 Filter-then-aggregate vs `filter` agg

If you want stats on a subset, prefer a top-level `query` to narrow the docs before the agg runs (cheaper). Use a `filter` *agg* only when you need multiple side-by-side stats from different subsets in one request — e.g., `shipped` and `cancelled` each as their own filter bucket with an `avg_total` sub-agg. One request, parallel computations, same matched-doc scan.

---

## Related

- [Inverted Index Internals](../foundations/01-inverted-index-internals.md) — `.dvd` doc-values files (§5) are part of the same Lucene segment family
- _Mapping and Field Types (planned)_ — the `keyword` vs `text` decision that determines whether a field can be aggregated
- _Query DSL — Bool, Term, Range (planned)_ — the `query` half of `_search`; aggs operate on its output
- [Database — Polyglot Persistence](../../database/INDEX.md) — when "approximate count" is unacceptable and you should be using Postgres
- [Performance — Latency, Throughput, Percentiles](../../performance/INDEX.md) — why t-digest's tail accuracy matters for SLO dashboards
- [Observability — Metrics and RED/USE](../../observability/INDEX.md) — where most percentile-agg traffic actually originates

## References

- [Elasticsearch Reference — Aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html) — top of the official aggregations docs; family taxonomy and execution model
- [Elasticsearch Reference — Terms aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html) — `size`, `shard_size` defaults (`size * 1.5 + 10`), `show_term_doc_count_error`, the "increase shard_size, not size" guidance cited in §7
- [Elasticsearch Reference — Composite aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-composite-aggregation.html) — `after_key` cursor model used in §10.2
- [Elasticsearch Reference — Cardinality aggregation (8.15)](https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-metrics-cardinality-aggregation.html) — confirms `precision_threshold` default 3000, max 40000, and `c * 8` bytes memory cost cited in §6
- [Elasticsearch Reference — Percentiles aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-metrics-percentile-aggregation.html) — t-digest as the default, default compression 100, HDR tradeoffs cited in §8
- [Elasticsearch Reference — Search settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-settings.html) — `search.max_buckets` default 65,536, dynamic cluster setting, cited in §9.1
- [Heule, Nunkesser, Hall — HyperLogLog in Practice (Google, EDBT 2013)](https://research.google/pubs/pub40671/) — the HyperLogLog++ paper underlying the `cardinality` aggregation
- [Dunning — Computing Extremely Accurate Quantiles Using t-Digests](https://github.com/tdunning/t-digest/blob/main/docs/t-digest-paper/histo.pdf) — the t-digest paper underlying the default `percentiles` implementation
- [HdrHistogram — High Dynamic Range Histogram](http://hdrhistogram.org/) — reference site for the HDR histogram alternative to t-digest
- [Elastic Blog — Improving the cardinality estimator](https://www.elastic.co/blog/count-and-sort-trade-offs-and-limitations-of-the-terms-aggregation) — extended treatment of the across-shards top-N approximation problem from §7
