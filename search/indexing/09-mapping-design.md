---
title: "Mapping Design — Schema Decisions That Determine Everything Downstream"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, mapping, schema]
---

# Mapping Design — Schema Decisions That Determine Everything Downstream

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `mapping` `schema`

---

## Table of Contents

- [Summary](#summary)
- [1. Dynamic, Explicit, Strict — Three Schema Postures](#1-dynamic-explicit-strict--three-schema-postures)
- [2. Field Datatypes That Actually Matter](#2-field-datatypes-that-actually-matter)
- [3. Multi-Fields — `text` + `keyword.raw` and Friends](#3-multi-fields--text--keywordraw-and-friends)
- [4. Modeling Relationships — Denormalize vs Nested vs Join](#4-modeling-relationships--denormalize-vs-nested-vs-join)
- [5. nested vs object — When Document Boundaries Matter](#5-nested-vs-object--when-document-boundaries-matter)
- [6. `_source` — The Original JSON, and How to Trim It](#6-_source--the-original-json-and-how-to-trim-it)
- [7. doc_values vs fielddata — Columnar Access for Sort/Aggs](#7-doc_values-vs-fielddata--columnar-access-for-sortaggs)
- [8. `index_options` — The Storage Knob Behind Postings Files](#8-index_options--the-storage-knob-behind-postings-files)
- [9. `norms` — When BM25 Doesn't Need to Score](#9-norms--when-bm25-doesnt-need-to-score)
- [10. Coercion, ignore_malformed, null_value](#10-coercion-ignore_malformed-null_value)
- [11. Practical Patterns](#11-practical-patterns)
- [12. Migration — Mappings Are Mostly Immutable](#12-migration--mappings-are-mostly-immutable)
- [Related](#related)
- [References](#references)

---

## Summary

A mapping is the schema that tells Elasticsearch (or OpenSearch) what to do with each field at index time and at query time. Unlike a SQL schema, a mapping is not just "column types" — it dictates which Lucene files get written (postings, doc values, norms, stored fields), how text gets analyzed, what queries are even *expressible* against the field, and how much disk and heap a shard burns. Most production search problems trace back to a mapping decision made in the first ten minutes of the project.

This doc walks the high-leverage decisions: how lenient to be about new fields (`dynamic`), which datatype to pick, how to model the same content for both search and aggregations (multi-fields), how to model relationships (denormalize / nested / join — in that preference order), and which storage knobs (`_source`, `doc_values`, `index_options`, `norms`) can be tuned without losing correctness. The single most important fact to keep in mind: **mappings are mostly immutable.** Adding new fields is free; changing existing ones requires reindex. Pick well the first time, and read/write through aliases so you can swap underneath running clients.

---

## 1. Dynamic, Explicit, Strict — Three Schema Postures

The `dynamic` mapping setting controls what happens when an indexed document contains a field that isn't in the mapping yet. The four valid values, per the Elastic mapping reference:

| Value | Behavior |
|-------|----------|
| `true` (default) | New fields are added to the mapping. Type is inferred from the first value. |
| `runtime` | New fields are added as runtime fields — not indexed, evaluated from `_source` at query time. |
| `false` | New fields are ignored — not indexed and not searchable, but kept in `_source`. |
| `strict` | Document is rejected with an exception if any field is unmapped. |

The default (`true`) is convenient for prototypes and miserable for production: a typo (`user.eamil`) becomes a permanent mapping entry, the first document's quirky type wins (a string `"42"` locks the field to `text`+`keyword`), and the field count drifts toward `index.mapping.total_fields.limit` without anyone noticing.

The production-leaning choices:

- **`strict`** for first-class business indices — every field must be declared; schema review becomes part of code review.
- **`runtime`** for log-shaped indices — flexibility on read, no index build cost, CPU paid per query.
- **`false` at the root + `strict` on nested objects** — middle ground when upstream occasionally adds a top-level field but inner shape is fixed.

```json
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "id":        { "type": "keyword" },
      "createdAt": { "type": "date" },
      "title":     { "type": "text" }
    }
  }
}
```

OpenSearch matches Elasticsearch on this setting (same four values, same defaults).

---

## 2. Field Datatypes That Actually Matter

There are dozens of types in the reference. Most production indices use a small subset.

| Type | Use for | Lucene impact |
|------|---------|---------------|
| `text` | Full-text fields — title, body, description | Analyzed, builds postings (`.doc`/`.pos`/`.pay`), no doc values |
| `keyword` | IDs, enums, tags, anything matched exactly or aggregated | Not analyzed; postings + doc values |
| `long` / `integer` / `short` / `byte` | Counts, IDs that must be numeric, ranges | BKD tree (points), doc values |
| `double` / `float` / `half_float` / `scaled_float` | Real-valued metrics, scores, prices | BKD tree, doc values; `scaled_float` stored as a `long` with a fixed `scaling_factor` |
| `date` | Timestamps; takes ISO strings or epoch millis | BKD tree, doc values |
| `boolean` | Flags | Doc values + small inverted index |
| `ip` | IPv4 / IPv6 addresses | BKD tree (range / CIDR queries become trivial) |
| `geo_point` | `{lat, lon}` for radius / bbox queries | BKD tree |
| `geo_shape` | Polygons, lines, multi-geometries | More expensive; supports complex spatial relations |
| `dense_vector` | Embeddings for kNN / semantic search | HNSW (or `int8_hnsw`, `int4_hnsw`, `bbq_hnsw`, `flat` variants); `dims` ≤ 4096 |

A few decisions that bite people in production:

- **Pick the smallest integer type that fits.** A `byte` doc-values column is 8× smaller than a `long` — real disk and page-cache savings on a billion-doc index.
- **`scaled_float` for currency.** Floats are wrong for money; `scaled_float` stores a `long × scaling_factor` and compresses far better than `double`.
- **`keyword`, not `text`, for anything aggregated, sorted, or filtered exactly.** A document ID stored as `text` gets analyzed and broken on punctuation — exact-match queries will mysteriously fail.
- **`ip`, not `keyword`, for addresses** — range queries and CIDR matching come free.
- **`dense_vector` parameters are immutable.** `dims`, `similarity`, `index_options.type` — switching after the fact is a reindex.

---

## 3. Multi-Fields — `text` + `keyword.raw` and Friends

The same content frequently needs two indexing strategies: analyzed for full-text search, and exact for sorting / faceting / aggregations. The mapping answer is **multi-fields** — index the same source value into multiple sub-fields with different analyzers / types.

```json
{
  "properties": {
    "title": {
      "type": "text",
      "fields": {
        "raw":    { "type": "keyword", "ignore_above": 256 },
        "english":{ "type": "text", "analyzer": "english" }
      }
    }
  }
}
```

Now `title` (default analyzer) drives full-text search, `title.raw` drives `terms` aggregations and exact filters, and `title.english` drives stemmed English search. The source value is read once; Lucene writes three independent fields. Querying is just `"title.raw"` instead of `"title"`.

This pattern is also the official answer to "I want to aggregate on a `text` field." The Elastic `text` reference is explicit: do not enable `fielddata` (it's disabled by default for a reason — it loads tokens into JVM heap and is the cause of more cluster-wide OOMs than any other single mapping mistake). Use a multi-field `keyword` sub-field instead.

`ignore_above` truncates `keyword` indexing at a length threshold (256 is a common default) so a stray 50 KB string doesn't blow up the term dictionary.

---

## 4. Modeling Relationships — Denormalize vs Nested vs Join

Three ways to model a one-to-many or many-to-many relationship. They are listed in **strict preference order** — denormalize first, nested when document boundaries matter, join only as a last resort.

**Denormalize** (default): flatten the related data into the document. An order with line items becomes one document, line items as an array. Joins disappear; query speed is excellent; updates to a referenced entity require updating every document that embedded it. The Elastic `join` reference says it directly: *"In Elasticsearch the key to good performance is to de-normalize your data into documents."*

**Nested**: use when each element in an array of objects has fields that must be queried *together* on the same element (§5). Each nested element becomes a separate hidden Lucene document, inflating segment doc count, and requires a `nested` query — more expensive than a flat `bool`.

**Join (parent/child)**: declares parent/child relations within one index. Use only when you genuinely cannot denormalize. Constraints from the Elastic reference: one `join` field per index; parent and child must live on the same shard (route children by parent ID); a document is either parent or child, not both; *"each join field, `has_child` or `has_parent` query adds a significant tax to your query performance"*; global ordinals get rebuilt on changes, costing heap and latency.

If your instinct is "this looks like a foreign key in Postgres," that's the signal you probably want denormalization, not `join`.

---

## 5. nested vs object — When Document Boundaries Matter

The default `object` type **flattens** arrays of objects. Given:

```json
{
  "users": [
    { "first": "Alice", "last": "White" },
    { "first": "Bob",   "last": "Smith" }
  ]
}
```

the index sees `users.first = ["Alice", "Bob"]` and `users.last = ["White", "Smith"]`. The pairing is gone. A query for `first:Alice AND last:Smith` will match this document — even though no single user object has those values together. This is the canonical "array of objects gotcha" and it produces wrong results, not slow results.

`nested` fixes it by storing each array element as a separate hidden Lucene document, joined back to the parent via segment-level relationships. Queries against nested fields use the `nested` query, which enforces "same element" semantics.

Rule of thumb: **if the array contains scalar values, `object` is fine; if the array contains objects whose fields must match together, use `nested`.** The cost is real (more hidden docs, slower queries) but the correctness gain is non-negotiable when the semantics demand it.

---

## 6. `_source` — The Original JSON, and How to Trim It

`_source` is the original JSON document body, stored verbatim in Lucene's stored-fields files (`.fdt` / `.fdx` from §8 of the inverted-index doc). It is *not* searchable; it's the payload the engine returns on a hit.

The Elastic source-field reference is blunt: *"Do not disable the `_source` field, unless absolutely necessary."* Disabling `_source` breaks `update`, `update_by_query`, `reindex`, highlighting, and the standard reindex-to-fix-mapping workflow — the very escape hatch you'll need when (not if) you have to migrate.

A safer middle path is **source filtering at index time** via `_source.includes` / `excludes`:

```json
{
  "mappings": {
    "_source": {
      "excludes": ["raw_html", "embedding"]
    },
    "properties": { "...": "..." }
  }
}
```

This keeps `_source` enabled for everything else (so reindex still works for the kept fields) but drops the heavyweight blobs from stored fields. Be aware: anything excluded from `_source` cannot be retrieved or reindexed. Treat it as a one-way pruning.

For routine "I just don't want to ship this field over the wire," prefer query-time `_source` filtering (in the search request body) — it doesn't touch storage and keeps reindex available.

---

## 7. doc_values vs fielddata — Columnar Access for Sort/Aggs

The inverted index is great for "find docs where term X appears" and terrible for "give me the value of field Y for each of these 10,000 docs." Sorting, aggregations, and script field access need *columnar* access — given a doc ID, return the field's value in O(1).

**Doc values** are Lucene's columnar store, written into `.dvd`/`.dvm` files per segment at index time. They are enabled by default for every type that supports them. Per the Elastic doc-values reference, the unsupported types are `text` and `annotated_text` (and `wildcard` cannot be disabled). Disable them only on fields you are certain you will never sort on, aggregate on, or read in a script — typically a high-cardinality `keyword` field used only for filtering by exact value:

```json
{
  "user_id": { "type": "keyword", "doc_values": false }
}
```

This saves disk; the cost is that any future `terms` aggregation or sort on `user_id` will fail.

**Fielddata** is the legacy, in-memory equivalent for `text` fields. It is disabled by default and should stay that way. Per the Elastic `text` reference, fielddata loads analyzed tokens into the JVM heap on first access; on a large `text` field this can hit gigabytes per node and cause cluster-wide GC death spirals. The supported pattern is the multi-field from §3: `text` for search, `keyword` sub-field with doc values for aggregations.

---

## 8. `index_options` — The Storage Knob Behind Postings Files

`index_options` controls how much information goes into the postings files (`.doc`, `.pos`, `.pay` from §4 of the inverted-index doc). Four values, per the Elastic reference:

| Value | Stored | Files written |
|-------|--------|---------------|
| `docs` | Doc IDs only | `.doc` |
| `freqs` | + term frequencies | `.doc` |
| `positions` | + per-occurrence positions | `.doc` + `.pos` |
| `offsets` | + character offsets | `.doc` + `.pos` + `.pay` |

The default for `text` is `positions` (you get phrase-query support out of the box). The default for `keyword` is `docs` (no analysis, no positions, smallest postings). Bumping a `text` field to `offsets` is the standard way to enable the unified highlighter without re-fetching `_source` at query time — at the cost of larger postings on disk.

Going the other direction: if you have a `text` field that you only ever match as terms (no phrase queries, no proximity), dropping it to `freqs` or even `docs` shrinks postings noticeably. Most teams don't do this because they don't know whether downstream queries will use phrases; the safe default is to leave `index_options` alone unless you have a measured reason.

---

## 9. `norms` — When BM25 Doesn't Need to Score

Norms are per-field, per-document length normalization factors. BM25 uses them to penalize matches in longer documents (a query term in a 5-word title is worth more than the same term in a 5,000-word body). Per the Elastic norms reference, norms cost roughly *one byte per document per field* and live in `.nvd`/`.nvm`.

If a field is used **only for filtering or aggregations** — never scored — disabling norms is free disk and free heap:

```json
{
  "status": { "type": "keyword", "norms": false }
}
```

`keyword` fields default to `norms: false` already. The relevant case is `text` fields you never want to influence the score — say, a synthetic `tags` text field used only as a filter.

Caveat from the reference: *once disabled, norms cannot be re-enabled.* Re-enabling requires reindex. Pick the direction with care.

---

## 10. Coercion, ignore_malformed, null_value

Three small parameters that prevent a category of write-time failures.

- **`coerce`** (default `true`) — cleans up "dirty" values to fit the type. `"5"` becomes `5` for an integer field; `5.0` truncates to `5`. Sometimes welcome, sometimes hides a producer bug. Disable when you want strict typing.
- **`ignore_malformed`** (default `false`) — when a value cannot be parsed for the field's type, the field is skipped for that document instead of the document being rejected. Useful at log-ingestion boundaries; dangerous on first-class business fields where a silent drop is worse than a rejection.
- **`null_value`** — substitutes a sentinel value for `null`, making the field searchable as that sentinel. Works on most non-`text` types. Use for "missing" semantics that downstream queries need to filter on.

```json
{
  "tag": { "type": "keyword", "null_value": "__NA__" },
  "score": { "type": "integer", "ignore_malformed": true, "coerce": false }
}
```

---

## 11. Practical Patterns

### 11.1 Search-as-you-type

The `search_as_you_type` field type creates a small family of sub-fields under one mapping declaration. Per the Elastic reference, declaring `{ "type": "search_as_you_type" }` produces:

- the root field (default analyzer)
- `<field>._2gram` — shingle filter, size 2
- `<field>._3gram` — shingle filter, size 3
- `<field>._index_prefix` — edge n-gram on top of the largest shingle

`max_shingle_size` is configurable (range 2–4, default 3). A `multi_match` with type `bool_prefix` over `["field", "field._2gram", "field._3gram"]` gives instant prefix + completion behavior without rolling your own n-gram analyzer.

### 11.2 Currency as `scaled_float`

Pick a scaling factor that captures every decimal place you care about (`100` for cents, `10000` for fractional cents). Internally Elasticsearch stores a `long`, which compresses far better than a `double` and avoids `0.1 + 0.2 ≠ 0.3` arithmetic surprises in scripts.

```json
{
  "price_eur": { "type": "scaled_float", "scaling_factor": 100 }
}
```

### 11.3 Runtime fields for ad-hoc analytics

When a stakeholder asks "can we slice by `domain` extracted from `email`?" and you don't want to reindex, define a runtime field with a Painless script that emits the substring after `@`. The field is computed from `_source` (or doc values) at query time. Per the Elastic runtime-fields reference: flexibility for free, query CPU as the cost. Index it for real once it proves load-bearing.

---

## 12. Migration — Mappings Are Mostly Immutable

The Elastic explicit-mapping reference states it plainly: you can add new fields freely; you cannot change the type or core parameters of an existing field. Renames don't exist (use field aliases for read-side compatibility).

The standard migration is the **reindex pattern**:

1. Define the new mapping in a new index (e.g., `orders-v2`).
2. Run `_reindex` from `orders-v1` to `orders-v2`. Streaming, resumable, throttleable.
3. Atomically flip an alias (`orders` → `orders-v2`).
4. Delete `orders-v1` once you're confident.

Two operational habits that make this painless:

- **Always read and write through an alias, never the index name directly.** The index becomes a swappable artifact behind the alias.
- **Use index templates** so new indices in a rolling pattern (e.g., `logs-2026.05.07`) inherit the right mapping without anyone editing JSON by hand.

Allowed in place (per the reference): adding new fields, adding sub-fields under multi-fields, increasing `ignore_above` on a `keyword`, and tweaking search-time parameters that don't touch storage. Anything that changes how bytes are written to disk is reindex.

---

## Related

- [Inverted Index Internals — Postings Lists, Skip Lists, and FSTs](../foundations/01-inverted-index-internals.md) — `index_options` (§8) and `norms` (§9) of this doc are the mapping-side controls for the postings and `.nvd` files described there
- _Analyzers — Tokenization, Filters, and Language-Specific Pipelines (planned)_ — the `analyzer` parameter referenced in §3 multi-field examples
- _BM25 and TF-IDF — Relevance Scoring from First Principles (planned)_ — the scoring layer that consumes norms (§9) and term frequencies (§8)
- _Index Templates, Aliases, and Reindex Patterns (planned)_ — Tier 4 deep dive on the migration workflow in §12
- _Dense Vectors and Approximate kNN (planned)_ — the `dense_vector` type from §2 in detail
- [Database — Schema Design and DDL Migrations](../../database/INDEX.md) — the relational counterpart; useful contrast to "mappings are mostly immutable"

## References

- [Elastic — Mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html) — top-level reference and entry point for every parameter cited here
- [Elastic — Dynamic mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic.html) — authoritative source for the four values in §1
- [Elastic — Field data types](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html) — full datatype catalog for §2
- [Elastic — Numeric field types](https://www.elastic.co/guide/en/elasticsearch/reference/current/number.html) — `scaling_factor` semantics in §2 and §11.2
- [Elastic — `dense_vector`](https://www.elastic.co/guide/en/elasticsearch/reference/current/dense-vector.html) — `dims ≤ 4096`, HNSW and quantized index variants in §2
- [Elastic — `nested` field type](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html) — array-of-objects flattening and the Alice/Smith example in §5
- [Elastic — `join` field type](https://www.elastic.co/guide/en/elasticsearch/reference/current/parent-join.html) — constraints and the "denormalize first" recommendation in §4
- [Elastic — `_source` field](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-source-field.html) — `includes`/`excludes` and the "do not disable" warning in §6
- [Elastic — `doc_values`](https://www.elastic.co/guide/en/elasticsearch/reference/current/doc-values.html) — defaults and unsupported types in §7
- [Elastic — `text` field and `fielddata`](https://www.elastic.co/guide/en/elasticsearch/reference/current/text.html) — fielddata heap risk and multi-field alternative in §3 and §7
- [Elastic — `index_options`](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-options.html) — the four values and defaults in §8
- [Elastic — `norms`](https://www.elastic.co/guide/en/elasticsearch/reference/current/norms.html) — ~1 byte/doc/field cost and the disable-then-reindex caveat in §9
- [Elastic — `coerce`](https://www.elastic.co/guide/en/elasticsearch/reference/current/coerce.html) — default behavior in §10
- [Elastic — Runtime fields](https://www.elastic.co/guide/en/elasticsearch/reference/current/runtime.html) — query-time evaluation in §11.3
- [Elastic — `search_as_you_type`](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-as-you-type.html) — sub-field structure in §11.1
- [Elastic — Explicit mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/explicit-mapping.html) — what is and isn't mutable, basis for §12
- [OpenSearch — Mappings and field types](https://opensearch.org/docs/latest/field-types/) — OpenSearch counterpart, tracks the Elastic reference for §1–§12
- [Apache Lucene 9.x — Index file formats](https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/codecs/lucene99/package-summary.html) — file extensions written by each mapping knob
