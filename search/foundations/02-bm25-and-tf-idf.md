---
title: "BM25 and TF-IDF — Relevance Scoring from First Principles"
date: 2026-05-07
updated: 2026-05-07
tags: [search, lucene, bm25, scoring, relevance, tf-idf]
---

# BM25 and TF-IDF — Relevance Scoring from First Principles

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `lucene` `bm25` `scoring` `relevance` `tf-idf`

---

## Table of Contents

- [Summary](#summary)
- [1. Why Scoring Exists at All](#1-why-scoring-exists-at-all)
- [2. TF-IDF — The Two Signals and Their Multiplication](#2-tf-idf--the-two-signals-and-their-multiplication)
- [3. Length Normalization — Why a Hit in a Short Doc Wins](#3-length-normalization--why-a-hit-in-a-short-doc-wins)
- [4. BM25 — Saturation via `k1`, Length Pivot via `b`](#4-bm25--saturation-via-k1-length-pivot-via-b)
- [5. Defaults — `k1=1.2`, `b=0.75`, and Where They Come From](#5-defaults--k112-b075-and-where-they-come-from)
- [6. What BM25 Fixes Over Classic TF-IDF](#6-what-bm25-fixes-over-classic-tf-idf)
- [7. When BM25 Became Lucene/Elasticsearch's Default](#7-when-bm25-became-luceneelasticsearchs-default)
- [8. Per-Field Similarity — Override Locally, Not Globally](#8-per-field-similarity--override-locally-not-globally)
- [9. Where the Norms Live — `.nvd` and `.nvm`](#9-where-the-norms-live--nvd-and-nvm)
- [10. Practical Pitfalls](#10-practical-pitfalls)
- [Related](#related)
- [References](#references)

---

## Summary

Scoring is what turns "find documents that match" into "find documents that *answer*" — without it, an inverted-index intersection just hands back an unordered set, and a 1-million-result set is useless to a human. BM25 (Okapi BM25, "Best Matching 25") is the function Lucene, Elasticsearch, and OpenSearch use by default to rank that set, and it is built from three first-principles signals: how often the query term appears in the document (term frequency), how rare the term is across the corpus (inverse document frequency), and how that document's length compares to the average (length normalization). BM25 takes those signals and adds two refinements over classic TF-IDF — a saturation curve on term frequency controlled by `k1`, and a tunable length-pivot controlled by `b`.

The single most useful mental shift here for a backend engineer: **BM25 is arithmetic, not magic**. The numbers it consumes (term frequency from `.doc`, document frequency from `.tim`, field-length norm from `.nvd`) all come straight out of the inverted-index files described in the previous doc in this tier. The score is computed at query time per matched document and is cheap relative to the postings I/O. Tuning relevance is rarely about changing the formula; it is about controlling what gets *into* those files (analyzer choices) and which fields participate in scoring vs. filtering.

---

## 1. Why Scoring Exists at All

Boolean retrieval — "return every document where `term` is in the postings list" — answers a yes/no question. For a query like `kafka` against a 10M-doc corpus where 50,000 documents contain the word, boolean retrieval returns 50,000 hits with no order. The user wanted the *best* 10.

Scoring assigns a real-valued number to each matched document so the engine can sort and truncate. The question scoring tries to answer is: "given the query terms, how relevant is this document compared to the others that also match?" Two documents can both match `kafka cluster sizing` and still be wildly different — one is a Confluent reference page where those three terms anchor the title and recur throughout, another mentions all three in passing inside a 40,000-word book chapter on distributed systems. A score lets the engine put the first one above the second.

Three principles drive what a useful score has to capture:

1. A document that contains the query term **more often** is, all else equal, more about that term.
2. A query term that is **rarer in the corpus** carries more discriminating power than a common one — `kafka` is more informative than `the`.
3. A document where the term takes up a **larger share of its content** is more focused than one where the same raw count is buried in a much longer text.

TF-IDF and BM25 are two formalizations of these three principles. They differ in how they combine them, not in what they're trying to capture.

---

## 2. TF-IDF — The Two Signals and Their Multiplication

TF-IDF, introduced in information retrieval in the 1970s and codified by Karen Spärck Jones's 1972 paper *A Statistical Interpretation of Term Specificity and Its Application in Retrieval*, multiplies two per-term values:

- **Term Frequency (TF)** — how many times the term appears in this document. The classical form is the raw count; some variants take the square root or a log to dampen runaway counts.
- **Inverse Document Frequency (IDF)** — how rare the term is across the corpus. The standard form is `log(N / df)` where `N` is the total document count and `df` is the number of documents containing the term. A term in 1 of 10M docs has `IDF ≈ log(10^7) ≈ 16`; a term in 5M of 10M has `IDF ≈ log(2) ≈ 0.7`.

The multiplication `TF × IDF` is the load-bearing idea. It says: a term contributes to the score in proportion to *both* how prominent it is in this document *and* how informative it is across the corpus. A document that mentions `the` 200 times scores essentially zero on `the` because IDF is near zero. A document that mentions `kafka` once scores meaningfully because IDF is large, even though TF is small.

For multi-term queries, TF-IDF sums the per-term contributions across the query terms — `score(d, q) = Σ_{t ∈ q} tf(t, d) × idf(t)`. Documents that match more query terms tend to score higher because more positive contributions accumulate.

This is the model Lucene shipped with for most of its history. It works, but it has two known weaknesses that BM25 was designed to fix (see §6).

---

## 3. Length Normalization — Why a Hit in a Short Doc Wins

Without normalization, longer documents win every time. A 100,000-word document mentions almost any term more often than a 100-word document, just because there's more text. But intuitively, the 100-word document where the term appears 5 times is *more about* the term than the 100,000-word document where it appears 50 times — the term occupies 5% of one document and 0.05% of the other.

Length normalization corrects for this by dividing TF (or some transformation of it) by a function of document length. The simplest form is `tf / doc_length`. Lucene stores a per-document, per-field length value precomputed at index time so this division is cheap at query time.

There's a tension here, though: pure length normalization over-rewards short documents, including very short ones that are essentially noise. A title field of `"kafka"` would score perfectly on a `kafka` query — TF=1, length=1, ratio=1.0 — even though a longer, more substantial title might be the better answer. BM25's `b` parameter (next section) makes the strength of this normalization tunable, and Elastic's "Practical BM25 Part 2" walks through exactly this tradeoff.

---

## 4. BM25 — Saturation via `k1`, Length Pivot via `b`

BM25 keeps the IDF half of TF-IDF essentially unchanged but rewrites the TF half. The intuition you should carry around without memorizing the formula:

### 4.1 Saturation — `k1`

In TF-IDF, term frequency contributes linearly: a document with TF=20 scores roughly 10× a document with TF=2. That's not how human relevance judgments work. Going from 0 to 1 occurrence of a term is a huge signal change ("this document is about this thing"). Going from 19 to 20 is essentially noise.

BM25 replaces the linear TF with a **saturating curve**: each additional occurrence adds less to the score than the previous one. The curve approaches an asymptote — past some point, more occurrences barely matter at all. The `k1` parameter controls *where* on the saturation curve the contribution flattens. Lower `k1` saturates faster (one or two occurrences nearly max out the term's contribution); higher `k1` keeps TF more linear (more occurrences keep mattering). Elastic's "Practical BM25 Part 2" shows this graphically and is worth reading once.

### 4.2 Length Pivot — `b`

BM25's length normalization is also tunable. The model uses the ratio `doc_length / avg_doc_length` (the *pivot* is the average), and `b` interpolates how strongly that ratio affects the score:

- `b = 0` — length is ignored entirely. A 1,000-word doc and a 10-word doc with the same TF score the same.
- `b = 1` — length is fully normalized. A doc twice as long as average needs roughly twice the TF to score the same.
- `b = 0.75` (default) — somewhere in between, biased toward "length matters but doesn't dominate."

Pivoting around the corpus *average* (rather than against absolute length) means BM25 doesn't punish documents for being a normal length — it punishes them for being *unusually* long relative to peers in the same field.

Together, `k1` shapes "how much does each occurrence count" and `b` shapes "how much does length matter." Both knobs are per-field — see §8.

---

## 5. Defaults — `k1=1.2`, `b=0.75`, and Where They Come From

Elasticsearch's BM25 similarity ships with `k1 = 1.2` and `b = 0.75`, documented in the [Similarity settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-similarity.html) reference. These are the same defaults Lucene's `BM25Similarity` uses. The values are the conventional defaults used in the Okapi BM25 literature for general-purpose English text retrieval — they are reasonable for almost any text corpus and are rarely worth changing without a measurement to back the change up.

When tuning is justified, the Elastic blog on practical BM25 sketches the directions: increase `k1` if your domain rewards repeated mentions (e.g. legal documents where repetition signals emphasis); decrease `b` if your fields are highly variable in length and length-based punishment is over-correcting.

A common mistake is to adjust these globally because relevance "feels off." Almost always the real fix is upstream — analyzer choice, field mapping, query construction — not a different `k1`.

---

## 6. What BM25 Fixes Over Classic TF-IDF

Two specific, well-documented problems with classic TF-IDF that BM25 addresses:

| Problem | TF-IDF Behavior | BM25 Behavior |
|---------|-----------------|---------------|
| **Repetition runaway** | A doc with 100 occurrences of a term scores 100× a doc with 1 occurrence (linear in TF). | Saturation curve — going from 1 to 100 occurrences gives a much smaller score multiplier; past `k1`-controlled threshold, it nearly plateaus. |
| **Length cliff** | Length normalization is either absent (raw TF) or applied as a fixed function (`tf / length`) that can over-reward very short docs. | Tunable, average-pivoted normalization — short docs get a boost but not an unbounded one, long docs are penalized only relative to the corpus average. |

A third, more subtle improvement: BM25's IDF formula uses `log((N - df + 0.5) / (df + 0.5) + 1)` (the "+1" inside the log was added to Lucene's BM25 implementation to keep the IDF strictly positive, avoiding negative scores for very common terms). Classic TF-IDF's `log(N/df)` can go negative when `df > N/e`, which produces non-monotone scoring — BM25's variant doesn't.

Elastic's "BM25 vs Lucene's Classic Similarity" blog walks through measured ranking differences between the two on real corpora and is the canonical short read on why the switch was made.

---

## 7. When BM25 Became Lucene/Elasticsearch's Default

BM25 was added to Lucene as `BM25Similarity` years before it became the default. The change to make BM25 the default similarity was made in **Elasticsearch 5.0** — specifically merged in the 5.0.0-alpha4 release as PR #18948 ("Change default similarity to BM25"), addressing issue #18944. The corresponding Lucene change made `BM25Similarity` the default returned by `IndexSearcher.getDefaultSimilarity()` in Lucene 6.0.

Indexes created against Elasticsearch 5.0+ (and OpenSearch 1.0+, which forked from Elasticsearch 7.10) score with BM25 unless you explicitly override. Indexes created on older Elasticsearch and then upgraded carry their original similarity setting on existing fields — an upgrade does not retroactively rescore.

For a TypeScript/Node engineer who has been using `@elastic/elasticsearch` against any cluster running 5.x or later, every `match` query you've ever run has used BM25.

---

## 8. Per-Field Similarity — Override Locally, Not Globally

Elasticsearch lets you set similarity at three levels: the `index.similarity.default` setting (changes the default for the whole index), per-field in the mapping, and via custom-named similarity definitions. The reference docs at [Similarity settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-similarity.html) show all three.

The right default is **never override globally**. Almost every relevance-tuning case is field-scoped: a `title` field behaves differently from a `body` field, which behaves differently from a `tags.keyword` field. Setting a global `k1=2.0` because titles need it will quietly degrade body-field scoring, and you won't notice until users complain.

A typical override looks like this in the mapping:

```json
{
  "settings": {
    "similarity": {
      "title_bm25": {
        "type": "BM25",
        "k1": 1.0,
        "b": 0.3
      }
    }
  },
  "mappings": {
    "properties": {
      "title":   { "type": "text", "similarity": "title_bm25" },
      "body":    { "type": "text" },
      "tags":    { "type": "keyword" }
    }
  }
}
```

Here `title` uses a custom similarity (faster TF saturation, lighter length penalty — sensible for a short, deliberately-written field), while `body` keeps the BM25 default. This is the pattern to reach for. Custom similarity per field is also how you switch a single field to a non-BM25 model (e.g. `boolean` similarity for fields you want to participate in matching but not in scoring).

---

## 9. Where the Norms Live — `.nvd` and `.nvm`

The "field-length norm" — the precomputed length value BM25 needs to compute the length-pivot term — is stored at index time in two files per segment:

- **`.nvd`** — norms data: the per-doc, per-field length values, encoded compactly (Lucene compresses the norm to a single byte per doc per field by default, which means you lose precision on huge fields but save enormous amounts of space).
- **`.nvm`** — norms metadata: per-field info about which `.nvd` ranges belong to which field.

These are the files mentioned in the [inverted-index internals doc, §8](./01-inverted-index-internals.md). Norms are loaded on demand when the similarity needs them, which is essentially "every scoring query." If you set `"norms": false` on a field, the engine doesn't store norms for it and BM25 falls back to a length-agnostic mode for that field — useful for keyword-style fields where length normalization is actively wrong.

Two practical implications:

- **Disabling norms saves disk and heap** for fields that don't need scoring. A `tags.keyword` field used only in `term` filters should have `norms` disabled. You cannot re-enable norms on an existing index without reindexing.
- **The 1-byte encoding loses precision**. For typical text fields this is invisible; for fields with tens of thousands of tokens it can cause docs of length 30,000 and 35,000 to encode to the same norm value. Almost never a problem in practice but worth knowing when debugging "why do these two docs score identically."

---

## 10. Practical Pitfalls

### 10.1 Stop words and BM25 are awkward together

BM25's IDF naturally down-weights very common terms — `the` has near-zero IDF, so its contribution to the score is near zero whether or not it's in the document. This means **removing stop words via the analyzer is largely redundant for relevance** under BM25 and was much more important under TF-IDF.

But removing stop words still affects positional queries and phrase matching — `"to be or not to be"` is unrecoverable if `to`, `be`, `or`, `not` are all dropped. The Lucene/Elastic guidance has shifted: keep stop words in the index by default, and let BM25's IDF handle the down-weighting. Use a stop-word filter only if you have a measured indexing-cost reason to do so.

### 10.2 Very short fields make `b` misbehave

The length-pivot calculation uses `doc_length / avg_doc_length`. For a field where most documents are 1-3 tokens (like `title.short` or `sku`), the average is tiny and the ratio is volatile. A 4-token title becomes "much longer than average" and gets penalized harder than expected. The fix is usually to *lower* `b` toward 0.3-0.5 for that field — or disable norms entirely if length really doesn't matter.

### 10.3 Score-aware vs score-free filtering — `bool.filter` vs `bool.must`

This is the single most common Elasticsearch performance mistake from backend engineers used to SQL. In a `bool` query:

- Clauses under `must` and `should` **contribute to the score**. Each matched doc gets BM25 computed against each must/should clause.
- Clauses under `filter` and `must_not` **do not contribute to the score** — they're pure boolean inclusion/exclusion, no scoring math runs, and the result can be cached in the query cache.

A query like *"posts with `kafka` in body, written by user 42, after 2025-01-01"* should put `body:kafka` under `must` and `user_id:42` plus `published_at >= 2025-01-01` under `filter`. Putting them all under `must` triggers BM25 scoring against the user-id and date-range clauses (which is meaningless for a keyword/date) and disables the filter cache. The Spring Boot equivalent (Spring Data Elasticsearch's `Criteria` / `NativeQuery` builders) has the same distinction — `withFilter` vs `withQuery`.

When you don't need a relevance score at all (e.g. a "list all my orders" endpoint), use `constant_score` over a pure `filter` and skip BM25 entirely. The latency difference on large result sets is real.

---

## Related

- [Inverted Index Internals — Postings Lists, Skip Lists, and FSTs](./01-inverted-index-internals.md) — where the term frequencies, document frequencies, and field-length norms BM25 consumes actually live on disk
- _Analyzers — Tokenization, Filters, and Language-Specific Pipelines (planned)_ — what becomes a "term" before BM25 ever sees it; analyzer choice is the dominant lever for relevance
- _Lucene Segments and Merge Policy (planned)_ — Tier 1 deep dive on segment structure
- [Database — INDEX](../../database/INDEX.md) — relational scoring is a different problem, but ranking via `ORDER BY` and `LIMIT` is the closest SQL analogue
- [Performance — INDEX](../../performance/INDEX.md) — latency framework for reasoning about scoring cost vs filter cost

## References

- [Elasticsearch Reference — Similarity settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-similarity.html) — current authoritative source for the BM25 default, parameter defaults `k1=1.2` / `b=0.75`, and per-field override syntax
- [Elastic — Practical BM25 Part 2: The BM25 Algorithm and its Variables](https://www.elastic.co/blog/practical-bm25-part-2-the-bm25-algorithm-and-its-variables) — graphical walk-through of `k1` saturation and `b` length pivot
- [Elastic — Practical BM25 Part 1: How Shards Affect Relevance Scoring](https://www.elastic.co/blog/practical-bm25-part-1-how-shards-affect-relevance-scoring-in-elasticsearch) — companion piece on per-shard IDF computation
- [Elastic — BM25 vs Lucene's Classic Similarity](https://www.elastic.co/blog/found-bm-vs-lucene-default-similarity) — measured ranking comparison and rationale for the default change
- [Elasticsearch 5.0.0-alpha4 release notes](https://www.elastic.co/guide/en/elasticsearch/reference/5.0/release-notes-5.0.0-alpha4.html) — confirms BM25 became the default in 5.0.0-alpha4 (PR #18948, issue #18944)
- [Apache Lucene `BM25Similarity` Javadoc](https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/search/similarities/BM25Similarity.html) — formula and parameter definitions as implemented in Lucene
- [Apache Lucene `Similarity` Javadoc](https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/search/similarities/Similarity.html) — abstract class describing how custom similarities plug in
- Karen Spärck Jones (1972), *A Statistical Interpretation of Term Specificity and Its Application in Retrieval*, Journal of Documentation 28(1): 11–21 — origin of IDF
- Robertson, S. E. and Walker, S. (1994), *Some Simple Effective Approximations to the 2-Poisson Model for Probabilistic Weighted Retrieval*, SIGIR '94 — origin of the Okapi BM25 family
- Robertson, S. E. and Zaragoza, H. (2009), *The Probabilistic Relevance Framework: BM25 and Beyond*, Foundations and Trends in Information Retrieval 3(4): 333–389 — the long-form treatment of where the BM25 formula comes from and why
