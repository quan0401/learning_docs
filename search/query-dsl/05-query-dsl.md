---
title: "Query DSL — The Queries You Actually Write Daily"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, query-dsl, bool-query, search-after, pagination]
---

# Query DSL — The Queries You Actually Write Daily

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `query-dsl` `bool-query` `search-after` `pagination`

---

## Table of Contents

- [Summary](#summary)
- [1. Query Context vs Filter Context](#1-query-context-vs-filter-context)
- [2. Term-Level Queries](#2-term-level-queries)
- [3. Full-Text Queries](#3-full-text-queries)
- [4. Compound Queries — `bool` and Friends](#4-compound-queries--bool-and-friends)
- [5. Joining Queries — `nested`, `has_child`, `has_parent`](#5-joining-queries--nested-has_child-has_parent)
- [6. Specialized — `more_like_this` and `percolate`](#6-specialized--more_like_this-and-percolate)
- [7. Pagination — `from`/`size`, `search_after`, and Why Scroll Is Out](#7-pagination--fromsize-search_after-and-why-scroll-is-out)
- [8. Source Filtering and `fields`](#8-source-filtering-and-fields)
- [9. Common Pitfalls](#9-common-pitfalls)
- [Related](#related)
- [References](#references)

---

## Summary

The Elasticsearch / OpenSearch Query DSL is a JSON tree of nested clauses, each clause either a leaf query (`term`, `match`, `range`) or a compound query (`bool`, `dis_max`, `function_score`) that combines other clauses. Two ideas dominate the daily-driver subset: **query context vs filter context** (the first scores and is uncached, the second is binary and cached), and **term-level vs full-text queries** (the first is exact bytes against a `keyword` field, the second runs the input through the same analyzer the field was indexed with). Almost every production query is a `bool` with `filter` clauses doing the cheap structural matching and `must` / `should` clauses doing the expensive scored matching on `text` fields.

This doc is organized around what you actually type into Kibana Dev Tools or your repository layer: the dozen leaf queries you reach for weekly, the compound shapes that combine them, the joining queries that come up when the data model isn't quite flat, and the pagination contract that has shifted twice since 5.x — `from`/`size` is still capped at 10,000, scroll is no longer recommended for new code, and `search_after` with a Point-in-Time is the default deep-pagination story for both Elasticsearch ≥ 7.10 and OpenSearch.

---

## 1. Query Context vs Filter Context

Every clause in a Query DSL request runs in one of two contexts. The Elasticsearch reference puts the distinction crisply: query context answers *"how well does this document match?"* and computes a `_score`; filter context answers *"does this document match?"* with yes/no and skips scoring entirely.

| Clause | Context | Affects `_score` | Cached |
|--------|---------|------------------|--------|
| `must` (in `bool`) | query | yes | no |
| `should` (in `bool`) | query | yes | no |
| `filter` (in `bool`) | filter | no | yes |
| `must_not` (in `bool`) | filter | no | yes |
| top-level `query` | query | yes | no |
| `constant_score`'s `filter` | filter (with constant boost) | no (constant) | yes |

The Elasticsearch docs state directly: *"Elasticsearch automatically caches frequently used filters, speeding up subsequent search performance."* The cache is a per-segment node-local LRU keyed by the serialized filter. Move every "is this row visible to this tenant?" / "is this within the last 30 days?" / "is `status = active`?" check into `filter` and you get both correctness (no relevance pollution from infrastructure predicates) and lower CPU.

```json
GET /orders/_search
{
  "query": {
    "bool": {
      "must":   [ { "match": { "description": "wireless headphones" } } ],
      "filter": [
        { "term":  { "tenant_id": "acme" } },
        { "term":  { "status":     "shipped" } },
        { "range": { "created_at": { "gte": "now-30d" } } }
      ]
    }
  }
}
```

Score is computed only over `description`. The three filter clauses prune the candidate set first and are reused on the next request.

---

## 2. Term-Level Queries

Term-level queries do not analyze their input. The bytes you send are the bytes the engine looks up in the inverted index. They are the right tool for `keyword`, numeric, date, IP, and boolean fields — and the wrong tool for `text` fields (see §9).

| Query | What it does | Notes |
|-------|--------------|-------|
| `term` | exact-bytes match on one value | use against `keyword`, never `text` |
| `terms` | exact match against any of N values | bounded by `index.max_terms_count` (default 65536) |
| `range` | `gt` / `gte` / `lt` / `lte` on numeric, date, IP | dates support `now`, `now-7d/d`, etc. |
| `exists` | field has any non-null indexed value | inverse via `must_not` of `exists` |
| `prefix` | term starts with a literal prefix | expensive on long prefixes; consider `index_prefixes` mapping |
| `wildcard` | `*` and `?` glob match | leading wildcard is `O(unique_terms)`; avoid |
| `regexp` | full Lucene regex | bounded by `index.max_regex_length` (default 1000) |
| `fuzzy` | edit-distance match using Levenshtein automaton | default `fuzziness: AUTO` |

```json
{ "term":  { "status": "shipped" } }
{ "terms": { "user_id": ["u1", "u2", "u3"] } }
{ "range": { "price": { "gte": 100, "lt": 500 } } }
{ "prefix": { "sku": { "value": "ACME-" } } }
{ "fuzzy": { "title.keyword": { "value": "kafka", "fuzziness": "AUTO" } } }
```

The Elasticsearch term-query docs warn directly: *"By default, Elasticsearch changes the values of `text` fields as part of analysis."* A `term` query for `"Quick Brown Foxes!"` against an analyzed text field will not match the indexed tokens `[quick, brown, fox]`. This is the single most common confusion for engineers coming from SQL — see §9.

---

## 3. Full-Text Queries

Full-text queries run the query string through the same analyzer that indexed the field. They are the right tool for `text` fields.

### 3.1 `match`

The workhorse. Tokenize, look up each token in the postings, score with BM25.

```json
{ "match": { "description": { "query": "wireless noise cancelling headphones", "operator": "and" } } }
```

`operator` defaults to `or` (any token matches). Switching to `and` requires every token; `minimum_should_match: "75%"` is a softer middle ground.

### 3.2 `match_phrase` and `match_phrase_prefix`

`match_phrase` requires the tokens to appear in order, adjacent (controllable via `slop`). It needs positions in the postings (`.pos` file from the inverted-index doc) and is therefore unavailable on fields indexed with `index_options: docs`.

`match_phrase_prefix` is `match_phrase` plus a prefix match on the last term — what powers naive search-as-you-type.

### 3.3 `multi_match`

One query string, several fields. The Elasticsearch docs list six `type` values; the default is `best_fields`.

| Type | Behavior | When to reach for it |
|------|----------|----------------------|
| `best_fields` (default) | wraps per-field `match` in a `dis_max`; takes the best field's score | terms most likely to live together in one field (title OR body) |
| `most_fields` | sums per-field scores | same text analyzed multiple ways (`title`, `title.english`, `title.shingle`) |
| `cross_fields` | term-centric; treats N fields as one big field | structured names split across `first_name`, `last_name` |
| `phrase` / `phrase_prefix` | per-field `match_phrase` / `match_phrase_prefix` | phrase queries against multiple text fields |
| `bool_prefix` | per-field `match_bool_prefix` | search-as-you-type with last-term prefix matching |

```json
{
  "multi_match": {
    "query": "kafka streams",
    "type":  "best_fields",
    "fields": ["title^3", "body", "tags^2"],
    "tie_breaker": 0.3
  }
}
```

The `^N` suffix is per-field boost. `tie_breaker` blends in non-winning fields' scores under `best_fields` / `cross_fields`.

### 3.4 `query_string` vs `simple_query_string`

`query_string` parses Lucene's full query syntax (`AND`, `OR`, `NOT`, fielded `title:foo`, ranges, regex, etc.). Powerful, brittle, and **dangerous to expose to end users** — a single unbalanced bracket throws an error and the parser is a known source of expensive queries.

`simple_query_string` is the user-facing variant: same idea, lenient grammar (no exceptions on bad syntax), and a smaller operator set (`+`, `|`, `-`, `"phrase"`, `*`, `(...)`). Use this for raw user input; reserve `query_string` for trusted internal callers and Kibana.

---

## 4. Compound Queries — `bool` and Friends

### 4.1 `bool`

The single most-used query in production:

```json
{
  "bool": {
    "must":     [ { "match": { "title": "kafka" } } ],
    "should":   [ { "match": { "tags":  "streaming" } } ],
    "must_not": [ { "term":  { "deleted": true } } ],
    "filter":   [ { "range": { "created_at": { "gte": "now-7d" } } } ],
    "minimum_should_match": 1
  }
}
```

`minimum_should_match` defaults are subtle. The Elasticsearch reference: *"If the `bool` query includes at least one `should` clause and no `must` or `filter` clauses, the default value is `1`. Otherwise, the default value is `0`."* In the example above, `must` is present, so `should` is purely a score boost — drop `minimum_should_match` and untagged docs still match.

### 4.2 `dis_max`

"Disjunction max" — runs N subqueries, returns the document if any match, scores it as the **max** of the per-subquery scores (plus `tie_breaker × sum_of_others`). This is what `multi_match: best_fields` desugars to.

### 4.3 `function_score`

Multiplies or replaces the relevance score with a function — `field_value_factor` (e.g., boost by `popularity`), `gauss` / `linear` / `exp` decay (boost by recency or geo distance), `script_score` (arbitrary Painless), or `random_score` (deterministic shuffling).

```json
{
  "function_score": {
    "query": { "match": { "title": "kafka" } },
    "functions": [
      { "gauss": { "published_at": { "origin": "now", "scale": "30d", "decay": 0.5 } } },
      { "field_value_factor": { "field": "popularity", "modifier": "log1p", "missing": 1 } }
    ],
    "score_mode": "multiply",
    "boost_mode": "multiply"
  }
}
```

### 4.4 `boosting` and `constant_score`

`boosting` demotes documents matching a `negative` query without excluding them — useful for "show old results, but rank them lower." `constant_score` wraps a filter and assigns a fixed `_score` to every match — equivalent in effect to a `bool.filter` clause but explicit when you need a non-zero constant score for downstream sorting.

---

## 5. Joining Queries — `nested`, `has_child`, `has_parent`

JSON-flattening is the default Elasticsearch behavior, and it has a well-known footgun:

```json
{ "items": [ { "name": "shirt", "color": "red" }, { "name": "hat", "color": "blue" } ] }
```

By default this indexes as `items.name: [shirt, hat]` and `items.color: [red, blue]` — a query for `items.name = shirt AND items.color = blue` matches this doc, even though no individual item is a blue shirt.

### 5.1 `nested`

Mapping the field as `type: nested` indexes each object as a hidden separate Lucene document linked to its parent, and the `nested` query runs sub-clauses with cross-field correlation preserved within each inner doc:

```json
{
  "nested": {
    "path": "items",
    "query": {
      "bool": {
        "must": [
          { "term": { "items.name":  "shirt" } },
          { "term": { "items.color": "blue" } }
        ]
      }
    }
  }
}
```

Cost: every parent update rewrites all its nested children, and aggregations need explicit `nested` aggs. Default cap: `index.mapping.nested_objects.limit = 10000` per parent doc.

### 5.2 `has_child` / `has_parent`

Parent-join is for parent-child relationships across separate top-level documents in the same index — chosen when the child set is large enough that nesting it inside the parent is impractical (think: blog posts with millions of comments). Cost is significant: parent and all children must live on the same shard, and queries do an extra join per shard.

Rule of thumb for backend devs: **denormalize first, nest second, parent-join last.** A duplicated field across rows costs cheap disk; a parent-join costs query latency on every read forever.

---

## 6. Specialized — `more_like_this` and `percolate`

`more_like_this` accepts a document (or text) and returns documents with similar term distributions, computed by extracting the top-K terms by TF-IDF and OR-ing them. Useful for "similar articles" / "related products" without an embedding pipeline.

`percolate` inverts the model: index queries instead of documents, then send a document and ask which queries match it. Used to power saved searches, alerting ("notify me when an event matches this filter"), and content-classification pipelines.

Both are niche but worth knowing exist before reaching for an external system.

---

## 7. Pagination — `from`/`size`, `search_after`, and Why Scroll Is Out

### 7.1 The 10,000-hit ceiling

The Elasticsearch docs state plainly: *"By default, you cannot use `from` and `size` to page through more than 10,000 hits."* The cap is `index.max_result_window`, default `10000`. Raising it is a footgun — every shard must compute and return `from + size` hits before the coordinator merges, so cost is `O(shards × (from + size))` on the heap.

### 7.2 `search_after` with a Point-in-Time

Recommended by Elastic for any deep pagination: open a Point-in-Time (PIT), then walk pages by passing the last hit's sort values back as `search_after`.

```json
POST /orders/_pit?keep_alive=1m
// → { "id": "PIT_ID" }

POST /_search
{
  "size": 100,
  "pit":  { "id": "PIT_ID", "keep_alive": "1m" },
  "sort": [ { "created_at": "asc" }, { "_shard_doc": "asc" } ]
}
// next page:
{
  "size": 100,
  "pit":  { "id": "PIT_ID", "keep_alive": "1m" },
  "sort": [ { "created_at": "asc" }, { "_shard_doc": "asc" } ],
  "search_after": [ "2026-05-01T12:00:00Z", 4294967296 ]
}
```

The `_shard_doc` tiebreaker is required for stable ordering when sort values collide. Always close the PIT when done; it pins segments and prevents merges from freeing space.

### 7.3 Scroll — why not for new code

The Elasticsearch reference: *"We no longer recommend using the scroll API for deep pagination."* Scroll snapshots the entire result set up front, holds open shard contexts, and was designed for full-index exports — `search_after` + PIT does the same thing more cheaply and without the per-scroll resource pinning. For new code, reach for PIT + `search_after`. Scroll remains supported for legacy and one-off bulk exports.

---

## 8. Source Filtering and `fields`

Three knobs control what the engine sends back per hit:

- **`_source`** — `true` (default), `false` (omit entirely), or `{ "includes": [...], "excludes": [...] }`. Returns the original indexed JSON exactly as you sent it.
- **`fields`** — runs each requested field through its mapper and returns the indexed/derived value. Honors aliases, multi-fields, and runtime fields. Recommended over `_source` filtering when you need formatted dates or doc-values.
- **`stored_fields`** — only fields with `store: true` in the mapping; rare in modern Elasticsearch since `_source` covers the same ground.

```json
{
  "_source": { "excludes": ["large_blob"] },
  "fields":  ["title", "created_at", "computed_score"]
}
```

For listing endpoints with thousands of hits per page, excluding bulky fields from `_source` is one of the highest-leverage optimizations available — it cuts both network bytes and the JSON serialization cost on the coordinator.

---

## 9. Common Pitfalls

**1. `term` against a `text` field returns nothing.** The classic. The text was analyzed at index time (`"Hello World"` → `[hello, world]`), and `term` does not analyze its input, so `term: "Hello World"` looks for a single token `"Hello World"` that does not exist. Either map a `.keyword` multi-field and query `field.keyword`, or use `match`. The Elasticsearch term-query docs include this warning explicitly.

**2. Analyzer mismatch between index and search.** A field indexed with the `english` analyzer stores stemmed tokens (`running` → `run`), but a `match` query without an explicit `analyzer` uses the field's `search_analyzer` (defaulting to the index analyzer — usually fine). Where this bites: a custom analyzer at index time, default analyzer at search time, and silent zero results. Always test with `_analyze` against both the index-time and search-time analyzers.

**3. Filter context for "I want it scored."** Putting the main user-input clause under `filter` instead of `must` returns the right documents in arbitrary order. Filter context exists to skip scoring; if you want results sorted by relevance, the relevance-bearing clause must be in query context.

**4. Score-context cost on infrastructure predicates.** The mirror image: putting `tenant_id`, `deleted = false`, and date ranges under `must` blows the filter cache and adds BM25 work for every candidate. Move infrastructure into `filter`.

**5. Leading wildcards and unbounded regex.** `wildcard: "*foo*"` and `regexp: ".*foo.*"` enumerate every term in the dictionary. On a 100M-term field this is seconds, not milliseconds. Use the `wildcard` field type (Elasticsearch ≥ 7.9) or n-gram tokenization at index time if you genuinely need substring search.

**6. Raising `max_result_window` to 100k.** Treat as a smell. Every deep page costs `O(shards × from)` heap. Use `search_after` + PIT.

**7. Forgetting the `_shard_doc` tiebreaker in `search_after`.** Without it, two hits with the same `created_at` value can swap order between pages, causing duplicate or missing rows in the consumer.

---

## Related

- [Inverted Index Internals — Postings Lists, Skip Lists, and FSTs](../foundations/01-inverted-index-internals.md) — what the term-level queries actually walk
- _BM25 and TF-IDF — Relevance Scoring from First Principles (planned)_ — what `must` / `should` clauses ultimately compute
- _Analyzers — Tokenization, Filters, and Language-Specific Pipelines (planned)_ — the index-time / search-time pipeline that pitfall #2 is about
- _Mappings — `text` vs `keyword`, Multi-Fields, and Dynamic Templates (planned)_ — the field-type decisions that decide which queries even apply
- _Aggregations — Bucket, Metric, and Pipeline (planned)_ — the read path that runs *after* a query has produced a candidate set
- [Database — Indexing Strategies](../../database/INDEX.md) — useful contrast: B-tree exact lookup vs inverted-index term lookup
- [Performance — Latency, Throughput, Percentiles](../../performance/INDEX.md) — framework for reasoning about deep-pagination cost

## References

- [Elasticsearch Reference — Query and filter context](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html) — authoritative source on the two contexts and the filter cache
- [Elasticsearch Reference — Boolean query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html) — `must` / `should` / `must_not` / `filter` semantics and the `minimum_should_match` default rule quoted in §4.1
- [Elasticsearch Reference — Term query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html) — includes the explicit warning about `term` against `text` fields cited in §2 and §9
- [Elasticsearch Reference — Multi-match query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html) — the six `type` values and `best_fields` default cited in §3.3
- [Elasticsearch Reference — Function score query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html) — `gauss` / `field_value_factor` / `script_score` and the `score_mode` / `boost_mode` semantics
- [Elasticsearch Reference — Nested query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html) and [Joining queries](https://www.elastic.co/guide/en/elasticsearch/reference/current/joining-queries.html) — `nested` / `has_child` / `has_parent` semantics and shard co-location requirements
- [Elasticsearch Reference — Paginate search results](https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html) — source for the `max_result_window: 10000` default, the scroll deprecation guidance ("We no longer recommend using the scroll API for deep pagination"), and the PIT + `search_after` recommendation
- [Elasticsearch Reference — Point in time API](https://www.elastic.co/guide/en/elasticsearch/reference/current/point-in-time-api.html) — opening, refreshing, and closing a PIT
- [Elasticsearch Reference — Retrieve selected fields](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-fields.html) — `_source`, `fields`, and `stored_fields` differences referenced in §8
- [OpenSearch Documentation — Query DSL](https://opensearch.org/docs/latest/query-dsl/) — OpenSearch's mirror of the same DSL; clause names and semantics match Elasticsearch 7.10 (the fork point)
- [OpenSearch Documentation — Paginate results](https://opensearch.org/docs/latest/search-plugins/searching-data/paginate/) — confirms the same `from + size ≤ 10000` default and PIT + `search_after` recommendation in the OpenSearch fork
