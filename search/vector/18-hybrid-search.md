---
title: "Hybrid Search — Combining BM25 and Dense Vectors with RRF"
date: 2026-05-07
updated: 2026-05-07
tags: [search, hybrid, rrf, vector, bm25, elasticsearch, opensearch]
---

# Hybrid Search — Combining BM25 and Dense Vectors with RRF

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `hybrid` `rrf` `vector` `bm25` `elasticsearch` `opensearch`

---

## Table of Contents

- [Summary](#summary)
- [1. Why Hybrid — The Two Failure Modes](#1-why-hybrid--the-two-failure-modes)
- [2. The Score Combination Problem](#2-the-score-combination-problem)
- [3. Reciprocal Rank Fusion (RRF)](#3-reciprocal-rank-fusion-rrf)
- [4. Convex Combination — The Normalization Path](#4-convex-combination--the-normalization-path)
- [5. Query-Time Combination vs Reranking Pipeline](#5-query-time-combination-vs-reranking-pipeline)
- [6. Elasticsearch Hybrid Search](#6-elasticsearch-hybrid-search)
- [7. OpenSearch Hybrid Query](#7-opensearch-hybrid-query)
- [8. Practical Patterns — When to Hybrid](#8-practical-patterns--when-to-hybrid)
- [9. When NOT to Use Vector Search](#9-when-not-to-use-vector-search)
- [10. Evaluation — Closing the Loop](#10-evaluation--closing-the-loop)
- [Related](#related)
- [References](#references)

---

## Summary

BM25 and dense-vector retrieval fail in opposite places. BM25 nails exact-match terms — proper nouns, identifiers, error codes, rare technical vocabulary — because its tf-idf core weights *literal* token overlap. Dense vectors nail semantic equivalence — *"car"* matches *"automobile"*, *"cancel my plan"* matches *"end subscription"* — because embeddings collapse synonyms and paraphrases into nearby points in vector space. Each model misses the other's strengths.

**Hybrid search** runs both retrievers in parallel and fuses their results. The hard part is combining scores from two scales that aren't comparable: BM25 is unbounded, cosine similarity is bounded `[-1, 1]`; naïve addition is meaningless. Two fusion strategies dominate: **Reciprocal Rank Fusion (RRF)** — sum `1/(k + rank)` across rankers, normalization-free, the default in modern Elasticsearch and OpenSearch — and **convex combination** — `α·lex + (1-α)·vec` after min-max normalization, which gives more control at the cost of tuning. RRF is the right starting point; convex combination is the lever you reach for when click-through data shows it's worth tuning. This doc covers the math, the API surfaces in Elasticsearch and OpenSearch, and the operational pattern: BM25 alone for structured/filterable fields and navigation, hybrid with RRF for any free-text search bar that's the user's primary entry point.

---

## 1. Why Hybrid — The Two Failure Modes

### 1.1 Where BM25 wins

BM25 scores documents by literal term overlap, weighted by inverse document frequency. It dominates when the user's query contains:

- **Proper nouns and brand names** — `"Stripe"`, `"Llama 3.1"`. The exact token is the signal; embeddings often blur it into nearby brands.
- **Identifiers** — order numbers, SKUs, error codes (`ERR_CONNECTION_REFUSED`), commit hashes. Embeddings trained on natural language treat these as noise.
- **Rare or out-of-vocabulary terms** — domain jargon the embedding model never saw at training time. BM25's IDF weighting *rewards* rarity; embeddings have no representation for unseen tokens.
- **Code, regex, exact phrases** — anything where character-level fidelity matters more than meaning.

### 1.2 Where dense vectors win

Dense vectors score by semantic proximity in an embedding space. They dominate when:

- **Vocabulary mismatch** — user types `"how do I cancel"`, the doc says *"end your subscription"*. BM25 returns nothing useful; the embedding model maps both phrases to nearby vectors.
- **Paraphrase and intent** — *"car insurance for young drivers"* matches a doc titled *"automobile coverage for new policyholders under 25"*.
- **Multilingual or cross-lingual** — multilingual embedding models match queries in one language to docs in another. BM25 cannot.
- **Long natural-language queries** — conversational queries with stop-word noise and grammatical structure.

### 1.3 The shape of the gap

A canonical failure: a support query for *"refund my Stripe subscription"*. BM25 ranks docs with *"Stripe"* and *"refund"* near the top but misses *"how to issue a chargeback for a Stripe customer"*. A dense retriever ranks the chargeback doc highly because *refund ≈ chargeback*, but may bury the literal *"refund"* doc beneath semantically-adjacent fluff. Hybrid surfaces both, in roughly the right order, without the engineer manually picking which model to trust per query.

---

## 2. The Score Combination Problem

The first instinct — "just add the scores" — fails because the two scoring functions live on different scales.

| Scorer | Range | Distribution |
|--------|-------|--------------|
| BM25 | `[0, +∞)` — unbounded; depends on corpus, term IDF, doc length | Long-tailed; top hits in a 1M-doc corpus might be 12.4, 11.8, 11.1, … |
| Cosine similarity | `[-1, 1]`; in practice `[0, 1]` for normalized embeddings of natural text | Compressed; top hits often 0.87, 0.86, 0.85 (small absolute differences) |
| Dot product (unnormalized) | `(-∞, +∞)` — depends on vector magnitudes | Varies wildly; do not mix scales without normalization |

Adding BM25's `12.4` to cosine's `0.87` makes BM25 swamp every vector signal. Multiplying makes the inverse problem. Even reasonable-looking constants like `0.5 * bm25 + 0.5 * vec` are silently weighted by whatever each side's typical magnitude happens to be, which depends on corpus size, query length, and embedding model.

Two principled fixes exist: **rank-based fusion** (ignore scores, use ordering — RRF) and **score normalization** (rescale both scores to a comparable range, then linearly combine — convex combination). RRF is the default modern systems converge on because it has one tuning parameter (`k`), no normalization step, and is robust across corpora.

---

## 3. Reciprocal Rank Fusion (RRF)

RRF was introduced by Cormack, Clarke, and Buettcher in 2009 as a lightweight rank-aggregation method that outperformed Condorcet Fusion and individual learning-to-rank methods on the TREC corpus. The formula is the entire algorithm:

```text
RRF(d) = Σ over rankers r:  1 / (k + rank_r(d))
```

For each document `d`, sum across rankers `r` of the reciprocal of `(k + d's rank in r's result list)`. Documents not present in a particular ranker's top-N contribute zero from that ranker.

### 3.1 Worked example

Two rankers, top-5 results each. `k = 60` (the field-standard default).

```text
BM25 top-5:    [doc_A, doc_B, doc_C, doc_D, doc_E]
Vector top-5:  [doc_C, doc_F, doc_A, doc_G, doc_B]

doc_A: 1/(60+1) + 1/(60+3)  = 0.01639 + 0.01587 = 0.03226
doc_B: 1/(60+2) + 1/(60+5)  = 0.01613 + 0.01538 = 0.03151
doc_C: 1/(60+3) + 1/(60+1)  = 0.01587 + 0.01639 = 0.03226
doc_D: 1/(60+4)             = 0.01563
doc_E: 1/(60+5)             = 0.01538
doc_F: 1/(60+2)             = 0.01613
doc_G: 1/(60+4)             = 0.01563

Fused order: A, C (tied), B, F, D, G, E
```

Notice what happened: `doc_C` appeared #3 in BM25 but #1 in vector, and rose to tied-first overall. `doc_F`, vector-only, made the top-4 because it ranked highly in one of the two systems even with no BM25 signal at all.

### 3.2 Why `k = 60`

The constant `k` flattens the rank curve. With `k = 60`, ranks 1 and 2 contribute `1/61` and `1/62` — almost the same weight. With `k = 0`, rank 1 contributes `1.0` and rank 2 contributes `0.5` — the top hit dominates. Cormack et al.'s original paper found `k = 60` empirically robust; it's the default in Elasticsearch's RRF retriever and the recommended starting point in OpenSearch's literature. Lower `k` (`k=10`) sharpens preference for top-ranked docs in each list; higher `k` (`k=100`) flattens further and lets mid-ranked docs from multiple rankers win.

### 3.3 Properties

- **No score normalization required.** RRF only needs the rank order from each retriever.
- **Robust to scale differences.** Mix BM25, cosine, ColBERT MaxSim, and a learned reranker without rescaling any of them.
- **Insensitive to outliers.** A single retriever returning an anomalously high score cannot dominate.
- **Easy to add rankers.** Three retrievers? Sum `1/(k + rank)` across all three.

---

## 4. Convex Combination — The Normalization Path

When you want explicit control over the lex/vec balance — "60% vector / 40% BM25 wins on this corpus" — RRF's rank-only model can't express that. Convex combination can:

```text
score_final(d) = α · score_lex_normalized(d) + (1 − α) · score_vec_normalized(d)
where α ∈ [0, 1]
```

Both component scores must first be normalized. The standard choice is **min-max normalization** over the result set: `(score − score_min) / (score_max − score_min)`, which rescales each retriever's scores to `[0, 1]` per query. A single `α` then controls the blend.

Tradeoffs vs RRF: more tunable but less robust (`α` must be revisited as index, embedding model, or query mix changes); sensitive to outliers (one anomalously-high BM25 score compresses everything else near zero — RRF is immune); per-query normalization is local (the top doc always normalizes to 1.0 even when it's a weak match). Start with RRF; switch to convex combination only when you have evaluation infrastructure to tune `α` and click data to validate the change.

---

## 5. Query-Time Combination vs Reranking Pipeline

There are two architectural shapes for combining retrievers:

**Query-time fusion (what RRF does).** Run BM25 and vector retrieval in parallel, take top-K from each, fuse with RRF. One round-trip. Result quality is bounded by the union of the two top-K lists.

**Retrieve-and-rerank.** Run a cheap first-pass retriever (BM25, vector, or hybrid) to fetch top-100 or top-1000, then apply an expensive reranker (cross-encoder, learning-to-rank model, LLM) to rescore that smaller set. Two round-trips, but the reranker can use signals the retriever can't — full attention between query and doc, business rules, click-through priors, freshness boosts.

The next doc in this tier covers cross-encoder rerankers in detail. These compose: production systems often do both — hybrid retrieval (BM25 + vector, fused with RRF) → top-100 → cross-encoder rerank → top-10 returned to user.

---

## 6. Elasticsearch Hybrid Search

Elasticsearch's modern hybrid API uses the `retriever` block (the replacement for the older `rank: { rrf }` shape that briefly existed in 8.x). RRF is generally available and listed in the free/Basic subscription tier — it was historically gated to paid tiers but is no longer.

```json
POST /products/_search
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": {
              "match": {
                "description": "wireless noise cancelling headphones"
              }
            }
          }
        },
        {
          "knn": {
            "field": "description_embedding",
            "query_vector_builder": {
              "text_embedding": {
                "model_id": "my_embedding_model",
                "model_text": "wireless noise cancelling headphones"
              }
            },
            "k": 50,
            "num_candidates": 200
          }
        }
      ],
      "rank_window_size": 100,
      "rank_constant": 60
    }
  },
  "size": 10
}
```

Key fields: `retrievers` — array to fuse, each can be `standard` (any query DSL), `knn`, `text_similarity_reranker`, `linear`, or `rule`. `rank_window_size` — top hits from each retriever that participate in fusion (default `100`). `rank_constant` — the `k` in the RRF formula (default `60`, must be `≥ 1`).

Verify the current subscription tier matrix when you ship — RRF is GA and currently listed across Free, Gold, Platinum, and Enterprise tiers, but Elastic has shifted RRF's tier in the past.

---

## 7. OpenSearch Hybrid Query

OpenSearch takes a different shape: a `hybrid` compound query plus a search pipeline that holds the normalization-and-combination processor. Score fusion is decoupled from the query — you configure how to combine once in a pipeline, then point queries at it.

### 7.1 Define the search pipeline

```json
PUT /_search/pipeline/hybrid-norm-pipeline
{
  "description": "Min-max normalize then arithmetic-mean combine",
  "phase_results_processors": [
    {
      "normalization-processor": {
        "normalization": {
          "technique": "min_max"
        },
        "combination": {
          "technique": "arithmetic_mean",
          "parameters": {
            "weights": [0.4, 0.6]
          }
        }
      }
    }
  ]
}
```

`technique` for normalization can be `min_max` or `l2`; for combination, `arithmetic_mean`, `geometric_mean`, `harmonic_mean`, or `rrf` in recent OpenSearch versions.

### 7.2 Execute the hybrid query against the pipeline

```json
GET /products/_search?search_pipeline=hybrid-norm-pipeline
{
  "query": {
    "hybrid": {
      "queries": [
        { "match": { "description": "wireless noise cancelling headphones" } },
        {
          "neural": {
            "description_embedding": {
              "query_text": "wireless noise cancelling headphones",
              "model_id": "my_embedding_model",
              "k": 50
            }
          }
        }
      ]
    }
  },
  "size": 10
}
```

The `weights` array in the combination processor maps positionally to the `queries` array, giving the convex-combination knob (here `α = 0.4` for BM25, `1−α = 0.6` for vector). Switch combination technique to `rrf` to get rank fusion instead.

OpenSearch's split — query expresses *what to retrieve*, pipeline expresses *how to fuse* — pairs well with operating multiple search experiences off one index: same `hybrid` query, different pipelines per surface (one tuned for product search, one for support docs).

---

## 8. Practical Patterns — When to Hybrid

Hybrid is not free: every query runs two retrievers and a fusion step. Use it where the recall lift earns its CPU.

- **Free-text search bar that's the user's primary entry point.** Queries are heterogeneous — sometimes literal IDs, sometimes natural language — and you cannot route per-query.
- **Long-form Q&A or "ask the docs" surfaces.** Conversational queries lean on semantics; named entities and version numbers lean on lexical. Hybrid covers both.
- **E-commerce product search where browsing and exact-match coexist.** Users type `"iPhone 15 Pro 256GB"` (lexical) and *"phone with good camera under $1000"* (semantic) into the same box.

The production pattern: (1) BM25 on the descriptive `text` field (title + description, analyzer-tokenized); (2) dense vector on the same content's embedding; (3) RRF fusion at query time; (4) optional cross-encoder rerank on the fused top-100 (covered in the next doc). Filtering structured fields (`category = "electronics" AND price < 1000`) happens in the lexical retriever as a bool filter; the vector retriever applies the same filter as a pre- or post-filter (engine-dependent — see this tier's earlier ANN/HNSW doc).

---

## 9. When NOT to Use Vector Search

Vector retrieval has overhead and failure modes. Skip it when:

- **Low-cardinality categorical fields and structured filters.** `status = "shipped"`, price ranges, date ranges, attribute facets — these are SQL-shaped predicates with no semantic dimension. BM25/term filter is exact, fast, and cheap.
- **Navigation queries.** Users typing a product SKU, order number, or known-item title want the literal match at #1. Vector retrieval can rank a semantically-adjacent doc above the literal hit — actively worse than BM25.
- **Tiny corpora.** Under a few thousand docs, BM25 already returns near-perfect results; vector index overhead and embedding-API cost dwarf any quality gain.
- **Domains the embedding model wasn't trained on.** Specialized vocabularies (legal citations, biomedical terms, internal jargon) often score worse on a general-purpose embedding model than on BM25 with a good analyzer. Either fine-tune a domain embedding or skip vectors.

A common architectural mistake is replacing BM25 with vector search instead of complementing it. Vector-only systems are weaker on exact-match queries by design; the right move is hybrid, not substitution.

---

## 10. Evaluation — Closing the Loop

You cannot tune RRF's `k`, convex combination's `α`, or the BM25-vs-vector retriever mix without evaluation infrastructure.

**Offline metrics** — run nightly against a test set of `(query, relevant_doc_ids)` pairs (historical click data is the cheapest source), comparing BM25-only, vector-only, and hybrid configurations:

- **NDCG@10** — Normalized Discounted Cumulative Gain at top 10. Rewards relevant docs ranking high; partial-credit aware. Requires graded judgments.
- **MRR (Mean Reciprocal Rank)** — `1 / rank_of_first_relevant`, averaged across queries. Best when there's one right answer per query (Q&A, known-item search).
- **Recall@k** — fraction of all relevant docs in the top-k. Critical for RAG: if the right doc isn't in your top-50, the LLM never sees it.

**Online metrics** — A/B test in production: CTR at top-N, mean reciprocal rank of clicks, zero-click abandonment rate, reformulation rate (did the user re-search immediately?). Click data closes the loop both ways: it scores existing configs and feeds future relevance judgments for the offline test set.

Treat hybrid-search tuning like recommender tuning: a recurring evaluation job, not a one-time configuration. Embedding-model upgrades, corpus drift, and query-mix shifts all change the optimal `k`, `α`, and weights. Without an eval harness, every change is theoretically-motivated guesswork.

---

## Related

- _BM25 and TF-IDF — Relevance Scoring from First Principles (this tier)_ — the lexical half of the hybrid pair
- _Dense Vector Embeddings and ANN Indexes (HNSW, IVF) (this tier)_ — the vector half
- _Cross-Encoder Reranking and Two-Stage Retrieval (next doc)_ — what comes after hybrid retrieval
- [Search — Inverted Index Internals](../foundations/01-inverted-index-internals.md) — the BM25 storage substrate
- [System Design — Search and Ranking Architectures](../../system-design/INDEX.md) — where hybrid retrieval fits in larger search systems
- [Performance — Latency and Percentiles](../../performance/INDEX.md) — measuring the cost of running two retrievers per query

## References

- [Cormack, Clarke, Buettcher (2009) — *Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods*, SIGIR '09](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf) — the original RRF paper; introduces the `1/(k + rank)` formula and reports `k = 60` as empirically robust on TREC
- [Elastic — Reciprocal Rank Fusion API reference](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion) — documents the `retriever.rrf` block, `rank_constant` default of `60`, `rank_window_size` default of `100`
- [Elastic — Subscriptions comparison](https://www.elastic.co/subscriptions) — current tier matrix; verify RRF and hybrid-retriever availability per tier before shipping (RRF was historically Platinum+ but is currently listed across Free, Gold, Platinum, Enterprise)
- [OpenSearch — Hybrid query reference](https://docs.opensearch.org/latest/query-dsl/compound/hybrid/) — `hybrid` compound query syntax
- [OpenSearch — Normalization processor](https://docs.opensearch.org/latest/search-plugins/search-pipelines/normalization-processor/) — search-pipeline `normalization-processor` with `min_max`/`l2` normalization and `arithmetic_mean`/`geometric_mean`/`harmonic_mean`/`rrf` combination
- [Elastic blog — *Hybrid retrieval with RRF*](https://www.elastic.co/search-labs/blog/hybrid-retrieval-elasticsearch-rrf) — implementation walkthrough and benchmarks
- [Robertson & Zaragoza (2009) — *The Probabilistic Relevance Framework: BM25 and Beyond*](https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf) — BM25 reference cited for the score-range claim in §2
