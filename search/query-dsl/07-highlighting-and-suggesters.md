---
title: "Highlighting and Suggesters — Snippets, Did-You-Mean, and Autocomplete"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, highlighting, suggesters, autocomplete, completion]
---

# Highlighting and Suggesters — Snippets, Did-You-Mean, and Autocomplete

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `highlighting` `suggesters` `autocomplete` `completion`

---

## Table of Contents

- [Summary](#summary)
- [1. Highlighting — What It Actually Is](#1-highlighting--what-it-actually-is)
- [2. The Three Highlighters — unified, plain, fvh](#2-the-three-highlighters--unified-plain-fvh)
- [3. Highlight Options That Matter](#3-highlight-options-that-matter)
- [4. The Analyzer-Mismatch Bug](#4-the-analyzer-mismatch-bug)
- [5. Suggesters — The Four Flavors](#5-suggesters--the-four-flavors)
- [6. Term Suggester — Levenshtein Per Token](#6-term-suggester--levenshtein-per-token)
- [7. Phrase Suggester — Language-Model Smoothing](#7-phrase-suggester--language-model-smoothing)
- [8. Completion Suggester — FST-Backed Autocomplete](#8-completion-suggester--fst-backed-autocomplete)
- [9. Context Suggesters — Filtering Suggestions](#9-context-suggesters--filtering-suggestions)
- [10. Did-You-Mean vs Autocomplete UX](#10-did-you-mean-vs-autocomplete-ux)
- [11. Practical Traps](#11-practical-traps)
- [Related](#related)
- [References](#references)

---

## Summary

Highlighting and suggesters are the two "second-pass" features of an Elasticsearch query: highlighting takes a hit and returns the snippet that matched (so you can bold the query terms in your result list), and suggesters take a query string and return alternatives (so you can offer "did you mean" or autocomplete). They share nothing in their implementation — highlighting reads stored fields and re-runs the analyzer, while suggesters fall into four distinct subsystems (`term`, `phrase`, `completion`, `context`) — but they share one operational truth: both are easy to wire up wrong, and both are common sources of latency surprises in production.

This doc walks the three highlighter implementations (`unified`, `plain`, `fvh`) and when each is appropriate, the highlight options you actually tune, the single most common highlighting bug (analyzer mismatch between index time and search time), and then the suggester family — including the FST-backed `completion` suggester whose dedicated `completion` field type lives in heap and is the standard production answer for type-ahead. Audience is a TS/Node + Java/Spring engineer building search-backed product surfaces.

---

## 1. Highlighting — What It Actually Is

A highlighter takes a search hit and emits a string with the matching terms wrapped in marker tags (`<em>…</em>` by default). To do that, the engine needs three things: the **original text** of the field, the **terms that matched** from the query, and a **way to locate** those terms inside the original text.

The original text comes from the stored `_source` (or from a separately-stored field). The matching terms come from extracting positive clauses out of the executed query. The location step is where the three highlighter implementations diverge — that is the entire design conversation around highlighting.

Highlighting runs **per hit, after retrieval**. It is not part of the matching loop, so it does not affect which documents are returned; it only affects what the response looks like. But because it can re-analyze the source for every returned hit, it is often the hottest single line item in a search response's latency budget — especially for large `_source` documents and large `size` values.

---

## 2. The Three Highlighters — unified, plain, fvh

Elasticsearch ships three highlighter implementations. OpenSearch inherits the same three (with the same names). They differ in how they locate matched terms inside the source text.

| Highlighter | How it locates matches | Term-vector requirement | Best for |
|-------------|------------------------|-------------------------|----------|
| `unified` (default) | Lucene's `UnifiedHighlighter` — uses postings offsets if available, term vectors if available, otherwise re-analyzes | None required; uses what the field has | Default choice for almost all cases |
| `plain` | Re-runs the query against an in-memory mini-index built from the field text | None | Small fields, simple queries; correctness over speed |
| `fvh` (fast vector highlighter) | Reads precomputed term vectors with positions and offsets directly from disk | **Requires `term_vector: with_positions_offsets`** in the mapping | Large fields where you've paid the term-vector storage cost |

A few consequences worth internalizing:

- `unified` is the default since Elasticsearch 7.0 (when it replaced the older `postings` highlighter). It is what you should use unless you have a measured reason not to.
- `fvh` is fast at runtime because the work happens at index time — but it forces you to index term vectors with positions and offsets, which inflates the field's on-disk footprint substantially (covered in [01-inverted-index-internals.md](../foundations/01-inverted-index-internals.md) §4 — the `.pay` file family is what holds offsets).
- `plain` is the slowest at search time but needs no special mapping. It is the safe fallback for fields you cannot reindex.

Set the highlighter per request via `"type": "unified" | "plain" | "fvh"` inside the `highlight` block, or per field. You can mix highlighters across fields in one request.

---

## 3. Highlight Options That Matter

```json
{
  "query": { "match": { "body": "kafka rebalance" } },
  "highlight": {
    "pre_tags":  ["<mark>"],
    "post_tags": ["</mark>"],
    "encoder":   "html",
    "fields": {
      "body": {
        "fragment_size": 150,
        "number_of_fragments": 3,
        "no_match_size": 200
      }
    }
  }
}
```

- **`pre_tags` / `post_tags`** — the wrapper markers around each match. Default is `<em>` / `</em>`. Use `<mark>` for accessibility (it carries semantic meaning), or a CSS-only class for app-driven theming. They are arrays so you can rotate tags across multiple highlights in one fragment.
- **`fragment_size`** — target character length of each snippet. Default 100. The highlighter does not cut at exactly this number — it splits on sentence/word boundaries near it.
- **`number_of_fragments`** — how many snippets to return per field. Default 5. Set to `0` to highlight the *entire field* instead of returning fragments (useful for short fields like titles).
- **`no_match_size`** — when the highlighter cannot find any match in the field (which happens, e.g., for `should` clauses or `multi_match` against fields the document does not actually contain the query terms in), return the first `N` characters of the field instead. Without this, the field is omitted from the highlight response, which is usually not what UIs want.
- **`encoder`** — `default` (no escaping) or `html` (escape source HTML before inserting tags). Use `html` whenever you render user-supplied content; it is the difference between a highlight feature and a stored XSS.
- **`require_field_match`** — defaults to `true`; only highlight fields that the query actually targets. Set to `false` to highlight matches in *any* field the query terms appear in (useful for search-across-many-fields UIs).
- **`fragmenter`** — `simple` or `span` (only relevant for the `plain` highlighter). `span` keeps phrase matches inside one fragment.
- **`max_analyzed_offset`** — caps how far into the field the highlighter will analyze. The default is the index-level `index.highlight.max_analyzed_offset` (1,000,000 chars in modern Elastic). On very large `_source` documents this is the lever that prevents one slow highlight from blowing out tail latency.

---

## 4. The Analyzer-Mismatch Bug

This is the bug everyone hits at least once.

**Symptom.** Your query matches the document. The hit comes back. But the `highlight` field is empty (or shows the wrong terms highlighted).

**Cause.** The query was analyzed with one analyzer (the field's `search_analyzer`, or the default), and the highlighter is analyzing the *source text* with a different analyzer (the field's `analyzer` at index time, or — for the `plain` highlighter — whatever the field's mapping says). The two analyzers produce different token streams, and the highlighter cannot find an overlap.

The classic shape: the field is mapped with a custom analyzer that strips diacritics and lowercases, but the document's `_source` contains the original `"Café"`. The query for `"cafe"` matches the indexed term `cafe`. The highlighter re-analyzes `_source` (`"Café"`) — possibly with a different analyzer if you've misconfigured `search_analyzer` — and produces a token stream that doesn't contain `cafe` at the same offsets. No highlight.

The variant on this: you indexed with `edge_ngram` or `synonym_graph` filters at index time, the highlighter sees the post-tokenization stream, and the offsets don't line up with what the user reads on screen.

The fix is one of:

1. **Use the same analyzer for index and search.** This is the default if you only set `analyzer`. Problems usually start when someone introduces a custom `search_analyzer`.
2. **Use the `unified` highlighter with offsets-in-postings (`index_options: offsets`).** When offsets are stored in postings, the highlighter does not need to re-analyze — it reads the offsets directly. This sidesteps analyzer mismatch entirely for the cost of a larger postings file.
3. **Use the `fvh` highlighter with `term_vector: with_positions_offsets`.** Same reasoning — the offsets are precomputed, no re-analysis.

The Elastic docs name this bug explicitly in the highlighting reference under "highlighting requires the offsets" — it is well-known but easy to walk into.

---

## 5. Suggesters — The Four Flavors

Suggesters are a separate top-level block on the search request. Four kinds, all with different use cases and very different runtime costs.

| Suggester | Mechanism | Use case |
|-----------|-----------|----------|
| `term` | Per-token Levenshtein edit distance against the field's term dictionary | Per-word "did you mean" |
| `phrase` | n-gram language model on top of `term`; scores whole rewrites | Multi-word "did you mean" with context |
| `completion` | FST-backed prefix lookup against a dedicated `completion` field | Type-ahead / autocomplete |
| `context` | A `completion` suggester filtered by category or geo context | Type-ahead scoped to a tenant, category, etc. |

Critically, `term` and `phrase` operate on the existing inverted index — no extra storage cost beyond what your text fields already carry. `completion` and `context` require a dedicated `completion` field type, which is stored differently and lives in heap (see §11).

---

## 6. Term Suggester — Levenshtein Per Token

The `term` suggester takes each token in the query string, looks up candidate terms in the field's term dictionary that are within a configurable Levenshtein edit distance, and returns them ranked by edit-distance + corpus frequency.

```json
{
  "suggest": {
    "spell-check": {
      "text": "kafak rebalanc",
      "term": {
        "field": "body",
        "suggest_mode": "popular",
        "min_word_length": 4,
        "prefix_length": 1
      }
    }
  }
}
```

`suggest_mode: missing` only suggests for tokens not in the index; `popular` only suggests when the alternative is more common than the input; `always` suggests for every token.

**When it works well.** Single-word typos against a healthy corpus. `"kafak"` → `"kafka"` because `kafka` is in the dictionary, edit distance 1, and far more frequent than the typo.

**When it fails.** Multi-word context. `"the quick brown fox"` with a typo in `quick` will see `quick` suggested only if a single-word lookup ranks it well — the suggester does not know that `quick brown fox` is a phrase. It also can't catch real-word errors (`"there"` vs `"their"` are both real words, both in the dictionary, both spelled correctly token-by-token). For those, you need `phrase`.

The implementation is the FST trick from [01-inverted-index-internals.md](../foundations/01-inverted-index-internals.md) §3 again: Lucene intersects the term FST with a Levenshtein automaton, enumerating only terms within edit distance `k`. This is fast — the hot path is FST traversal, not a scan of the dictionary.

---

## 7. Phrase Suggester — Language-Model Smoothing

The `phrase` suggester wraps `term` with an n-gram language model. It generates candidate rewrites of the *whole input*, scores each one against an n-gram model built from the indexed text, and returns the top-`k` rewrites with their model scores.

```json
{
  "suggest": {
    "did-you-mean": {
      "text": "noble prize laurate",
      "phrase": {
        "field": "body.trigram",
        "size": 3,
        "gram_size": 3,
        "direct_generator": [{ "field": "body.trigram", "suggest_mode": "always" }],
        "highlight": { "pre_tag": "<em>", "post_tag": "</em>" }
      }
    }
  }
}
```

The `field` typically points at a sub-field analyzed with a `shingle` filter (n-grams of words, not characters). The `direct_generator` block tells the suggester how to produce per-token candidates (the same Levenshtein lookup the `term` suggester uses). The phrase suggester then picks the rewrite whose n-grams best match what's in the index.

The output is markedly smoother than `term` — `"noble prize laurate"` resolves to `"nobel prize laureate"` as a whole, not as three independent token suggestions. It is the right primitive for a "Did you mean" line under a search box.

The cost: building the shingled sub-field roughly doubles the index size for the field, and the suggester does more per-request work than `term`.

---

## 8. Completion Suggester — FST-Backed Autocomplete

Type-ahead is a different problem from spell-correction. You want sub-50-ms response times against partial inputs (`"kaf"` → `["kafka", "kafkaesque", "kafkian"]`), often per keystroke. `term` and `phrase` are too slow for this; the `completion` suggester is the answer.

It requires a dedicated `completion` field type:

```json
{
  "mappings": {
    "properties": {
      "suggest": {
        "type": "completion",
        "analyzer": "simple"
      }
    }
  }
}
```

You then index suggestion strings into that field (often with weights and contexts):

```json
{
  "suggest": {
    "input": ["Kafka in Action", "Kafka, Franz"],
    "weight": 34
  }
}
```

And query it:

```json
{
  "suggest": {
    "book-suggest": {
      "prefix": "kaf",
      "completion": { "field": "suggest", "size": 5, "fuzzy": { "fuzziness": 1 } }
    }
  }
}
```

**Implementation.** The `completion` field stores its suggestions in a Finite State Transducer — the same FST data structure described in [01-inverted-index-internals.md](../foundations/01-inverted-index-internals.md) §3 — purpose-built for prefix queries. Lucene's `SuggestField` / `Completion` family uses a dedicated FST per segment for this. Lookup is `O(prefix_length)` plus the cost of enumerating matches, and the FST is held in memory for fast traversal.

**Near-real-time considerations.** Like all Elasticsearch indexing, completion suggestions are not visible to queries until the next refresh (default 1 s). For autocomplete this usually doesn't matter (the new product you just added is rarely typed in within the next second), but it does matter for cases like "I just signed up — autocomplete my own name." If you need write-then-read-immediately behavior, force a refresh on the indexing call (`?refresh=wait_for`) or accept the gap.

**Fuzzy completion.** The `fuzzy` block enables Levenshtein-tolerant matching, so `"kafak"` still completes to `"kafka"`. Use `fuzziness: 1` for short inputs and `AUTO` for variable input.

---

## 9. Context Suggesters — Filtering Suggestions

Often you want autocomplete restricted to a context: only books in stock, only restaurants in the user's city, only documents in the user's tenant. Plain `completion` cannot filter — it is a global FST per field. The **context suggester** solves this by attaching named context values to each suggestion at index time, and requiring matching context values at query time.

Two context types exist:

- **`category`** — arbitrary string contexts (e.g., tenant ID, language, in-stock flag).
- **`geo`** — geohash contexts; suggestions are bucketed into geohash prefixes and matched by query location.

```json
{
  "mappings": {
    "properties": {
      "suggest": {
        "type": "completion",
        "contexts": [
          { "name": "tenant", "type": "category" },
          { "name": "loc",    "type": "geo", "precision": 4 }
        ]
      }
    }
  }
}
```

At query time you must supply the context, otherwise the suggester returns nothing. This is the production pattern for multi-tenant SaaS autocomplete — the alternative (one completion field per tenant, or filtering after the fact) does not scale.

---

## 10. Did-You-Mean vs Autocomplete UX

The two patterns look similar but solve different problems and use different primitives.

| Pattern | When the user sees it | Implementation |
|---------|------------------------|----------------|
| Did-you-mean | After submitting a query, when the result count is low or zero | `phrase` suggester (preferred) or `term` suggester |
| Autocomplete / type-ahead | While the user is typing, before they submit | `completion` suggester |

A common mistake is reaching for the `completion` suggester for did-you-mean. It can return prefix matches but does not know about edit-distance-based corrections of full tokens; it will not turn `"noble prize"` into `"nobel prize"`. Conversely, reaching for `phrase` for autocomplete is too slow per keystroke — it executes against the regular inverted index and does an n-gram scoring pass per request.

A mature search UI usually wires both: `completion` against a dedicated suggestion field for the type-ahead dropdown, and `phrase` on the result page when the query returns weak results.

---

## 11. Practical Traps

- **Completion field lives in heap.** Each completion FST is held in JVM heap per segment. Large catalogs with millions of entries can consume several GB of heap. Monitor `indices.<idx>.completion.size_in_bytes` in the node stats API; budget heap accordingly. This is a different scaling axis from your regular indexed text fields, which are mmaped from disk.
- **Highlighting against large `_source` is slow.** `unified` and `plain` re-analyze the field text per hit. A 1 MB `_source` highlighted across 50 hits is 50 MB of re-analysis on the request path. Cap with `max_analyzed_offset`, store a separate small "highlight body" field, or use `fvh` with precomputed offsets.
- **`require_field_match: false` is misleading.** Setting it to `false` so highlights show up "more" can produce highlights on fields the query did not actually score against, leading to confusing UIs. Prefer fixing the query/mapping over flipping this flag.
- **Term vectors inflate index size.** Enabling `term_vector: with_positions_offsets` for `fvh` can roughly double a text field's on-disk footprint. Decide if the latency win is worth the storage cost.
- **`source_filter` matters.** When highlighting, the engine needs the field's source text. If you've turned off `_source` or excluded the field via `_source.excludes`, highlighting silently produces nothing. Either re-enable source for that field or use `store: true` on the field itself.
- **OpenSearch parity.** OpenSearch (forked from Elasticsearch 7.10) carries the same three highlighters and the same four suggesters with the same names and parameters. Newer Elasticsearch parameters (`max_analyzed_offset` defaults, security-related changes) may diverge over time; check the OpenSearch reference for the exact version.

---

## Related

- [01 — Inverted Index Internals](../foundations/01-inverted-index-internals.md) — the FST and postings structures that completion suggesters and offset-based highlighters build on
- _02 — Analyzers and Tokenization (planned)_ — the source of the analyzer-mismatch bug in §4
- _03 — Bool, Match, Term Queries (planned)_ — the queries whose matches the highlighter renders
- _05 — Aggregations and Faceting (planned)_ — the other "second-pass" feature on a search response
- [Operating Systems — Page Cache and Buffered I/O](../../operating-systems/operating-systems-tier1/03-page-cache-and-buffered-io.md) — why mmaped postings beat heap-resident structures, and the contrast with completion-suggester FSTs
- [Performance — Latency, Throughput, Percentiles](../../performance/INDEX.md) — framework for measuring the tail-latency cost of highlighting on large `_source`

## References

- [Elasticsearch Reference — Highlighting](https://www.elastic.co/guide/en/elasticsearch/reference/current/highlighting.html) — authoritative reference for `unified`, `plain`, `fvh`, all options listed in §3, and the offsets/term-vector requirements
- [Elasticsearch Reference — Suggesters](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html) — top-level suggester documentation; covers `term`, `phrase`, `completion`, `context`
- [Elasticsearch Reference — Completion suggester](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#completion-suggester) — dedicated `completion` field type, near-real-time/refresh behavior, fuzzy options
- [Elasticsearch Reference — Context suggester](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#context-suggester) — `category` and `geo` context filtering for completions
- [OpenSearch Documentation — Highlight](https://opensearch.org/docs/latest/search-plugins/searching-data/highlight/) — OpenSearch parity reference for the three highlighter types
- [OpenSearch Documentation — Did you mean](https://opensearch.org/docs/latest/search-plugins/searching-data/did-you-mean/) — OpenSearch reference covering `term` and `phrase` suggesters
- [OpenSearch Documentation — Autocomplete functionality](https://opensearch.org/docs/latest/search-plugins/searching-data/autocomplete/) — OpenSearch reference for `completion` suggester usage
- [Apache Lucene — `org.apache.lucene.search.suggest.document` package](https://lucene.apache.org/core/9_10_0/suggest/org/apache/lucene/search/suggest/document/package-summary.html) — Lucene's `SuggestField` and FST-based completion implementation that powers Elasticsearch's completion suggester
- [Mike McCandless — Using Finite State Transducers in Lucene](https://blog.mikemccandless.com/2010/12/using-finite-state-transducers-in.html) — Lucene committer's account of the FST data structure underlying both the terms dictionary and the completion suggester
