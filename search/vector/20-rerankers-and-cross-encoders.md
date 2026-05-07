---
title: "Rerankers and Cross-Encoders — Two-Stage Retrieval in Practice"
date: 2026-05-07
updated: 2026-05-07
tags: [search, rerank, cross-encoder, retrieval, vector]
---

# Rerankers and Cross-Encoders — Two-Stage Retrieval in Practice

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `rerank` `cross-encoder` `retrieval` `vector`

---

## Table of Contents

- [Summary](#summary)
- [1. Why Two Stages](#1-why-two-stages)
- [2. Bi-Encoder vs Cross-Encoder — The Architectural Split](#2-bi-encoder-vs-cross-encoder--the-architectural-split)
- [3. Cross-Encoder Cost — Why n=100, Not n=1M](#3-cross-encoder-cost--why-n100-not-n1m)
- [4. The Reranker Landscape — Hosted and Open](#4-the-reranker-landscape--hosted-and-open)
- [5. Vendor-Hosted vs Self-Hosted Trade-offs](#5-vendor-hosted-vs-self-hosted-trade-offs)
- [6. Wiring Rerankers into Search Engines](#6-wiring-rerankers-into-search-engines)
- [7. Score Combination — RRF, Pure Rerank Order, and Hybrid](#7-score-combination--rrf-pure-rerank-order-and-hybrid)
- [8. Practical Patterns by Workload](#8-practical-patterns-by-workload)
- [9. When NOT to Rerank](#9-when-not-to-rerank)
- [10. Practical Traps](#10-practical-traps)
- [Related](#related)
- [References](#references)

---

## Summary

A reranker is the second stage of a two-stage retrieval pipeline: a fast, coarse first stage (BM25, vector kNN, or both fused with RRF) returns 50–200 candidates, and a slower, more accurate model rescoring those candidates produces the final top-10. The accuracy lever is the **cross-encoder** — a transformer that takes the `[query; document]` pair as joint input and outputs a single relevance score. Cross-encoders cannot be precomputed or ANN-indexed because the query must enter the model alongside each candidate; they cost `O(n)` forward passes per query, which makes them viable at `n=100` and unviable at `n=1M`.

The Reimers and Gurevych sentence-transformers documentation frames this cleanly: a **bi-encoder** independently embeds query and document into single vectors that are compared by cosine, enabling ANN indexing of the corpus once; a **cross-encoder** sees both at once and computes a richer comparison, achieving substantially higher MS MARCO ranking quality at substantially higher per-pair cost. Two-stage retrieval combines them — bi-encoder (or BM25) for recall, cross-encoder for precision. The rest of this doc covers the model landscape (Cohere Rerank, Voyage rerank-2, Jina Reranker v2, BGE rerankers, sentence-transformers `cross-encoder/ms-marco-*`), the integration surfaces in Elasticsearch and OpenSearch, score-combination tactics, and the operational traps that bite in production: P99 latency dominated by reranker calls, per-pair API billing, and reranker-model drift over time.

---

## 1. Why Two Stages

The hard constraint in production retrieval is that you cannot afford to run an expensive scorer over every document in the corpus. A 10M-document index with a 100ms cross-encoder would need ~1,000,000 GPU-seconds per query; even with batching and dedicated hardware that is unusable. The structural fix is to split the work:

1. **Stage 1 — Recall.** Cheap, scalable retrieval over the whole corpus. BM25 against an inverted index, or kNN against an HNSW vector index, or both fused via Reciprocal Rank Fusion. Returns 50–200 candidates. Latency: single-digit to low-double-digit milliseconds.
2. **Stage 2 — Precision.** Expensive, accurate scoring over only the candidate set. A cross-encoder, an LLM-as-judge, or a hosted Rerank API. Returns the top-K. Latency: 50–500 ms depending on model and `n`.

The split works because the failure modes are complementary. Stage 1 is good at "is this document plausibly about this query?" but bad at fine-grained ranking — BM25 cannot distinguish "passenger jet" from "fighter jet" when both contain `jet`, and pure vector kNN famously gets fooled by superficially similar but semantically wrong matches. Stage 2 is good at "of these 100 plausible candidates, which actually answer the query?" but cannot scale to the corpus.

The Cohere Rerank documentation, Voyage Rerank guides, and Elastic's Inference API documentation all describe this same shape: send the query plus the candidate texts, receive ordered relevance scores, take the top-K. The mental model to keep is *retrieve wide, rerank narrow*.

---

## 2. Bi-Encoder vs Cross-Encoder — The Architectural Split

This distinction underpins every modern dense retrieval stack. The clearest exposition lives in the sentence-transformers project documentation by Nils Reimers and Iryna Gurevych.

### 2.1 Bi-encoder

```text
                  ┌─ Encoder ─► [query vector]    ─┐
   query ────────►│                                 ├─ cosine ─► score
                  └─ (separate forward pass)       ─┘
                  ┌─ Encoder ─► [doc vector]      ─┘
   document ─────►│
                  └─ (precomputed at index time)
```

Two independent forward passes produce two fixed-size vectors; the score is a cheap dot product or cosine similarity. Because the document side is computed without seeing the query, every doc embedding can be precomputed once at index time and stored in an ANN index (HNSW, IVF, ScaNN). This is what makes vector search at corpus scale possible at all.

Examples: OpenAI `text-embedding-3-large`, Voyage `voyage-3-large`, Cohere `embed-v4`, sentence-transformers `all-MiniLM-L6-v2`, BGE `bge-large-en-v1.5`.

### 2.2 Cross-encoder

```text
   query ────────┐
                  ├─► Transformer ─► [score]
   document ─────┘
   (joint input: "[CLS] query [SEP] document [SEP]")
```

A single forward pass over the concatenated `[query; document]` sequence; the model attends across both halves and emits a relevance score. The model can model query-document interactions at every transformer layer — "Does the verb in the query match the action in this paragraph?" — which a bi-encoder cannot, because the bi-encoder reduced each side to a fixed vector before comparison.

The accuracy gap is large and well-documented. On MS MARCO passage ranking, the original sentence-transformers blog post by Reimers and Gurevych reports cross-encoder rerankers (e.g. `cross-encoder/ms-marco-MiniLM-L-6-v2`) substantially outperforming pure bi-encoder baselines on NDCG@10 — at the cost of being unable to precompute the document side.

Examples: sentence-transformers `cross-encoder/ms-marco-MiniLM-L-6-v2`, `cross-encoder/ms-marco-MiniLM-L-12-v2`, BGE `bge-reranker-v2-m3`, Jina `jina-reranker-v2-base-multilingual`, Cohere `rerank-v3.5`, Voyage `rerank-2`.

### 2.3 The combined pattern

> "*Use a bi-encoder to retrieve candidates from a large corpus, then use a cross-encoder to rerank a small subset.*" — paraphrased from the sentence-transformers cross-encoder documentation, which explicitly prescribes this two-stage design.

---

## 3. Cross-Encoder Cost — Why n=100, Not n=1M

The per-query cost of a cross-encoder is `O(n)` forward passes where `n` is the candidate count. There is no shortcut — every candidate must enter the model alongside the query.

| Candidate count `n` | Approximate latency (small reranker, batched) |
|--------------------:|----------------------------------------------|
| 10  | 10–30 ms |
| 50  | 50–150 ms |
| 100 | 100–300 ms |
| 1,000 | 1–3 s |
| 1,000,000 | 15+ minutes |

Real numbers depend on model size (MiniLM-L-6 vs MiniLM-L-12 vs BGE reranker base/large), sequence length (longer documents = more tokens = more compute), batch size, and hardware (GPU vs CPU vs hosted API). The table is order-of-magnitude only.

The practical envelope is therefore `n` in the **50–200** range for interactive search. Anything larger needs either: a smaller stage-1 candidate set, a cheaper "stage 1.5" filter (e.g. a fast classifier), or accepting the latency for non-interactive workloads (offline batch reranking, evals).

---

## 4. The Reranker Landscape — Hosted and Open

### 4.1 Hosted APIs

- **Cohere Rerank** (`rerank-v3.5`, `rerank-english-v3.0`, `rerank-multilingual-v3.0`). Single endpoint: send a query, an array of documents (strings or objects), and a `top_n`; receive an ordered list with relevance scores in `[0, 1]`. Multilingual model supports 100+ languages. Pricing and rate limits change — check the live Cohere pricing page rather than trusting any number copied into a doc like this one.
- **Voyage Rerank** (`rerank-2`, `rerank-2-lite`). Similar API shape, optimized for use with Voyage's own embedding models. The `lite` variant trades some quality for lower latency and cost.
- **Jina Reranker** (`jina-reranker-v2-base-multilingual`, `jina-reranker-m0`). Hosted via Jina's Reranker API and also published with open weights on Hugging Face.

### 4.2 Open-weights models you can self-host

- **BGE rerankers** from BAAI: `BAAI/bge-reranker-large`, `BAAI/bge-reranker-base`, `BAAI/bge-reranker-v2-m3` (multilingual), `BAAI/bge-reranker-v2-gemma`. Available on Hugging Face, runnable via the `FlagEmbedding` package or sentence-transformers' `CrossEncoder` wrapper.
- **sentence-transformers cross-encoders** trained on MS MARCO: `cross-encoder/ms-marco-MiniLM-L-6-v2`, `cross-encoder/ms-marco-MiniLM-L-12-v2`, `cross-encoder/ms-marco-electra-base`. Listed in the sentence-transformers Pretrained Cross-Encoders documentation with reported MS MARCO MRR@10 figures and per-model latency hints.
- **Jina Reranker v2** weights, also on Hugging Face.

### 4.3 LLM-as-reranker

A separate pattern: instead of a dedicated cross-encoder, prompt an LLM (Claude, GPT-4-class, or a smaller instruction-tuned model) to rank or score the candidates. Uses one of two shapes:

- **Listwise:** "Here are 20 candidates; return their IDs in ranked order."
- **Pointwise / function-calling rerank:** call a `score_relevance(query, document) -> float` tool per candidate, often in parallel.

Slower and more expensive per query than a dedicated reranker model, but flexible — you can include reasoning instructions ("prefer recent documents", "prefer official sources") that a fixed cross-encoder cannot follow.

---

## 5. Vendor-Hosted vs Self-Hosted Trade-offs

| Concern | Hosted API (Cohere, Voyage, Jina) | Self-hosted (BGE, sentence-transformers cross-encoder) |
|---------|-----------------------------------|--------------------------------------------------------|
| Setup | API key, ~10 lines of code | GPU provisioning, model download, serving (TGI, vLLM, BentoML, sentence-transformers server) |
| Per-query cost | Per-pair pricing scales with `n × QPS` | Amortized GPU cost; high QPS amortizes well, low QPS does not |
| P99 latency | Add network RTT to the model latency | Single-digit-ms RTT inside your VPC |
| Data locality | Document text leaves your network | Stays in-VPC — required for some compliance regimes |
| Model drift / updates | Vendor controls — surprise quality shifts on the next API version | You control upgrade cadence |
| Multilingual / domain | Cohere multilingual and Jina v2 multilingual cover 100+ languages out of the box | BGE-v2-m3 covers many; finetuning your own is possible |

A useful default for most TS/Node and Spring Boot teams: start with a hosted reranker to validate the pipeline and the recall/precision lift, then move to self-hosted BGE or sentence-transformers if cost or data-locality forces it. The migration is small because the integration shape — "send query + candidates, receive scores" — is identical across providers.

---

## 6. Wiring Rerankers into Search Engines

### 6.1 Elasticsearch Inference API

Elasticsearch ships an **inference API** that wraps external model providers behind a single `_inference/<task_type>/<inference_id>` endpoint. The `rerank` task type accepts a query and a list of inputs and returns scored results. Supported services include Cohere, Hugging Face, ELSER (Elastic's own sparse model), and others — see the Elastic inference API documentation for the current list.

A retrieval pipeline using a `text_similarity_reranker` retriever combines stage 1 (a `standard` retriever doing BM25 or kNN) with stage 2 (the inference endpoint reranking the top `rank_window_size` results):

```json
{
  "retriever": {
    "text_similarity_reranker": {
      "retriever": {
        "standard": {
          "query": { "match": { "body": "kafka exactly once semantics" } }
        }
      },
      "field": "body",
      "rank_window_size": 100,
      "inference_id": "cohere-rerank-v3-5",
      "inference_text": "kafka exactly once semantics"
    }
  }
}
```

### 6.2 OpenSearch ML Commons Rerank Processor

OpenSearch's ML Commons plugin offers a **rerank search response processor** that calls a configured ML model on the top results of a query. The supported reranker types include cross-encoders (locally deployed) and connector-based remote rerankers (Cohere, SageMaker, Bedrock).

Pattern: register the model once via ML Commons, attach a `rerank` response processor to a search pipeline, then run searches against that pipeline; the rerank step happens server-side and returns reordered hits.

### 6.3 Application-side rerank (no engine support needed)

For engines without first-class rerank integration, do it in the application:

```ts
// Pseudocode — Node/TypeScript
const candidates = await search.knn({ index: "docs", k: 100, vector: q });
const rerankResp = await cohere.rerank({
  model: "rerank-v3.5",
  query: queryText,
  documents: candidates.map(c => c._source.body),
  top_n: 10,
});
const final = rerankResp.results.map(r => candidates[r.index]);
```

This is the lowest-friction option for a Spring Boot or Node service that already owns the search response.

---

## 7. Score Combination — RRF, Pure Rerank Order, and Hybrid

Once the reranker has produced scores, three patterns dominate:

### 7.1 Replace — take rerank order

The reranker score is the source of truth; sort candidates by it, return the top-K. Simplest, most common, and what hosted Rerank APIs implicitly assume. The original stage-1 score is discarded.

### 7.2 Reciprocal Rank Fusion over rerank scores

If you have multiple rerankers (or a reranker plus a business-rules scorer), combine via RRF on rank position:

```text
score(d) = Σ_i  1 / (k + rank_i(d))    where k is typically 60
```

This is the same RRF used to fuse BM25 + vector lists; the difference is the inputs are now rerank scores, not retrieval scores.

### 7.3 Linear blend

`final = α × rerank_score + (1−α) × business_signal` where `business_signal` is freshness, popularity, vendor weighting, or a learned signal. Requires that the reranker returns a calibrated score; Cohere's relevance score in `[0, 1]` is roughly suitable, though not formally calibrated across queries. For uncalibrated cross-encoder logits, normalize per query (min-max or softmax) before blending.

The common mistake is to blend the *stage-1* score with the rerank score, e.g. `α × bm25 + (1−α) × rerank`. BM25 scores are unbounded and corpus-dependent; mixing them with a `[0, 1]` rerank score produces unstable rankings. If you must combine stages, do it via rank position (RRF) rather than raw scores.

---

## 8. Practical Patterns by Workload

| Workload | Stage 1 | Stage 2 | Why |
|----------|---------|---------|-----|
| Keyword-heavy enterprise search (legal, compliance, ticketing) | BM25 | Cross-encoder rerank (BGE, Cohere) | Users type literal terms; BM25 wins on recall, reranker fixes "5 candidates all mention `kafka`, which actually answers the question?" |
| General semantic / RAG retrieval | Hybrid (BM25 ⊕ kNN via RRF) | Cross-encoder rerank | Hybrid catches both literal and semantic matches; reranker disambiguates |
| Long-document QA | BM25 over chunks | Cross-encoder over the same chunks | Reranker scores at the passage level, which is what generation needs |
| Nuanced ranking with business rules ("prefer official docs, prefer recent") | Hybrid | LLM-as-rerank with explicit instructions in the prompt | Cross-encoders cannot follow ranking instructions; LLMs can |
| Multilingual support | Hybrid with multilingual embedder | Cohere multilingual rerank or BGE-v2-m3 | Both stages must be language-aware; mixing English-only with multilingual breaks recall |

---

## 9. When NOT to Rerank

Reranking is a hammer that does not fit every nail.

- **Very high QPS / low margin.** A 200ms reranker on a 10K-QPS search system is a hard cost line. Reserve reranking for the queries that benefit (e.g. the long-tail of ambiguous queries) and skip it for head queries with cached top-K.
- **Navigation queries.** "facebook", "amazon login" — the user wants a known URL. Reranking cannot improve this and only adds latency.
- **Filtered lookups.** `WHERE customer_id = 12345` — filter cardinality is the dominant signal; relevance is irrelevant.
- **Already-precise stage 1.** If your hybrid retrieval already achieves NDCG@10 above the threshold the product needs, reranking is pure cost. Measure first.
- **Strict latency SLOs.** A P99 budget of 50ms cannot accommodate a 100ms reranker round-trip. Either downsize `n`, switch to a smaller in-VPC reranker, or accept that this query path skips rerank.

---

## 10. Practical Traps

- **P99 latency dominated by the reranker.** Stage 1 is fast and consistent; the reranker (especially hosted) introduces a long tail from network, batching delays, and cold model paths. Measure the rerank step separately in your traces — see [observability/INDEX.md](../../observability/INDEX.md).
- **Per-pair billing surprises.** Hosted rerank APIs charge per `(query, document)` pair processed, not per request. A `top_n=10` request over 100 candidates is *100 pairs*, not 10. Audit the pricing model before scaling.
- **Reranker-model drift.** Hosted vendors update model versions; pinning to `rerank-v3.5` rather than `rerank-english-v3` (latest) is a defensive practice. Run an offline eval with held-out judged pairs (NDCG@10, MRR@10) on every model upgrade.
- **Mismatch between stage-1 recall and rerank ceiling.** A reranker cannot add documents that stage 1 missed. If recall@100 from stage 1 is 70%, the reranker's NDCG ceiling is bounded by that 70%. Tune stage 1 first; the reranker amplifies recall, it does not replace it.
- **Document length truncation.** Cross-encoders have a max sequence length (often 512 tokens). Long documents get truncated, sometimes silently. Chunk before reranking, or use a long-context reranker (BGE-v2-m3 supports 8192 tokens).
- **Score caching across queries is not safe.** Unlike bi-encoder embeddings, cross-encoder scores depend on both query and document; caching by document ID alone is wrong. Cache by `hash(query) × document_id` if at all.
- **Batch tradeoffs.** Larger batches improve GPU throughput but increase per-query latency (batch of 100 finishes after the slowest item). Tune to the SLO, not to the throughput curve.

---

## Related

- _Hybrid Search — BM25 ⊕ Vector via RRF (planned)_ — the stage-1 fusion that typically feeds the reranker
- _ANN Indexes — HNSW, IVF, and ScaNN (planned)_ — how the bi-encoder side scales
- [Inverted Index Internals](../foundations/01-inverted-index-internals.md) — the BM25 substrate underneath the first stage
- [Observability — Tracing and SLO Design](../../observability/INDEX.md) — measuring rerank-stage latency in a search pipeline
- [Performance — Latency, Throughput, Percentiles](../../performance/INDEX.md) — framework for the P99 trade-offs in §10

## References

- [Sentence-Transformers — Cross-Encoders Documentation](https://sbert.net/examples/cross_encoder/applications/README.html) — Reimers & Gurevych's authoritative explanation of the bi-encoder vs cross-encoder split and the retrieve-then-rerank pattern
- [Sentence-Transformers — Pretrained Cross-Encoders](https://sbert.net/docs/cross_encoder/pretrained_models.html) — list of MS MARCO cross-encoder models (`cross-encoder/ms-marco-MiniLM-L-6-v2`, `-L-12-v2`, etc.) with reported quality and speed
- [Reimers & Gurevych — Sentence-BERT (EMNLP 2019)](https://arxiv.org/abs/1908.10084) — the original paper introducing the bi-encoder framing later extended with cross-encoder rerankers
- [Cohere Rerank API Documentation](https://docs.cohere.com/docs/rerank-overview) — endpoint shape, model names (`rerank-v3.5`, `rerank-multilingual-v3.0`), and integration patterns
- [Voyage AI — Reranker Documentation](https://docs.voyageai.com/docs/reranker) — `rerank-2` and `rerank-2-lite` model details
- [Jina AI — Reranker v2 model card](https://huggingface.co/jinaai/jina-reranker-v2-base-multilingual) — multilingual reranker open weights
- [BAAI — BGE Reranker on Hugging Face](https://huggingface.co/BAAI/bge-reranker-v2-m3) — open-weights cross-encoder reranker, multilingual, long-context
- [Elastic — Inference API Reference](https://www.elastic.co/docs/api/doc/elasticsearch/group/endpoint-inference) — `rerank` task type and supported services
- [Elastic — Semantic Reranking with the `text_similarity_reranker` Retriever](https://www.elastic.co/guide/en/elasticsearch/reference/current/retriever.html) — the retriever-shaped integration used in §6.1
- [OpenSearch — Reranking Search Results](https://docs.opensearch.org/latest/search-plugins/search-relevance/rerank-cross-encoder/) — the ML Commons rerank processor and connector model
