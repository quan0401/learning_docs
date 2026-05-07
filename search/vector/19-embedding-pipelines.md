---
title: "Embedding Pipelines — Where Vectors Are Generated, Stored, and Kept In Sync"
date: 2026-05-07
updated: 2026-05-07
tags: [search, vector, embeddings, pipelines, elasticsearch, opensearch]
---

# Embedding Pipelines — Where Vectors Are Generated, Stored, and Kept In Sync

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `vector` `embeddings` `pipelines` `elasticsearch` `opensearch`

---

## Table of Contents

- [Summary](#summary)
- [1. The Three-Way Decision: Where Does Embedding Run?](#1-the-three-way-decision-where-does-embedding-run)
- [2. App-Side Embedding at Index Time](#2-app-side-embedding-at-index-time)
- [3. In-Cluster Embedding via Ingest Pipeline](#3-in-cluster-embedding-via-ingest-pipeline)
- [4. Embedding at Query Time — The Symmetry Constraint](#4-embedding-at-query-time--the-symmetry-constraint)
- [5. Model Versioning and Reindex Migrations](#5-model-versioning-and-reindex-migrations)
- [6. Dimensionality — 384 vs 768 vs 1024+](#6-dimensionality--384-vs-768-vs-1024)
- [7. Batching, Throughput, and Bulk Indexing](#7-batching-throughput-and-bulk-indexing)
- [8. Vendor APIs and Self-Hosted Options](#8-vendor-apis-and-self-hosted-options)
- [9. Streaming vs Batch Reembedding](#9-streaming-vs-batch-reembedding)
- [10. Storage Layout — Same Document or Sidecar Index?](#10-storage-layout--same-document-or-sidecar-index)
- [11. Practical Traps](#11-practical-traps)
- [Related](#related)
- [References](#references)

---

## Summary

A vector search system has three independent decisions to make about where embedding work happens: at index time on the application, at index time inside the cluster (via an ingest processor), and at query time on the user's input string. The choices interact — most importantly, the model used to embed the corpus and the model used to embed the query must be the same model with the same prompt format, or the cosine similarities are meaningless. This doc walks the pipeline end-to-end: what each stage is for, what controls you give up by pushing the work into the cluster, what changes when you swap models, why dimensionality matters more than it looks, how to batch through a vendor API without setting your bill on fire, and the small handful of subtle bugs (instruction-prefix mismatch, silent token-limit truncation, chunking choices) that quietly destroy recall in production.

The single most useful framing: **embedding is a data transformation, not a search feature**. Treat it like an ETL stage that produces a vector column on every document, and the operational questions answer themselves — versioning, batching, retries, and drift are all the same problems you already solve for any derived column in a data warehouse.

---

## 1. The Three-Way Decision: Where Does Embedding Run?

Every vector search deployment makes three independent choices:

| Stage | What runs | What you gain | What you give up |
|-------|-----------|---------------|------------------|
| Index-time, app-side | App calls embedding API, writes vector + doc to ES | Full control over batching, retries, model versioning, observability | One more service in your write path |
| Index-time, in-cluster | ES inference processor or OpenSearch ML connector calls the model on every ingest | One pipeline to manage; no app-side code for embedding | Model lifecycle is now coupled to the cluster; debugging crosses a process boundary |
| Query-time | Same model called on the query string | Mandatory — there is no "without it" option for a learned dense vector query | Latency budget includes one round trip to the embedding model |

The query-time leg is non-negotiable for a learned dense vector model. The index-time choice is "app-side or in-cluster, pick one." Hybrid setups (some content embedded offline in a batch pipeline, fresh content embedded by the app at write time) are normal — what is not normal is using two *different models* for the two paths.

---

## 2. App-Side Embedding at Index Time

The shape of the app-side path:

```text
Source doc ──► App ──► Embedding API ──► App ──► Bulk index to ES
                       (OpenAI / Cohere /         (vector field +
                        TEI / vLLM)                _source)
```

A minimal Node example using the OpenAI Embeddings API and the Elasticsearch JS client:

```ts
import OpenAI from "openai";
import { Client } from "@elastic/elasticsearch";

const openai = new OpenAI();
const es = new Client({ node: process.env.ES_URL });

async function indexDocs(docs: Array<{ id: string; text: string }>) {
  // Batch up to the API's input limit. Embeddings APIs are heavily-batched-by-design.
  const resp = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: docs.map((d) => d.text),
  });

  const operations = docs.flatMap((doc, i) => [
    { index: { _index: "articles", _id: doc.id } },
    { text: doc.text, embedding: resp.data[i].embedding },
  ]);

  await es.bulk({ refresh: false, operations });
}
```

What you own when you do it this way:

- **Batching policy.** You choose the batch size against the embedding API's input cap and your tail-latency budget.
- **Retry and dead-letter handling.** A 429 on embedding doesn't have to fail the index write; queue and retry.
- **Model versioning.** The model name is a string in your code, version-controlled with everything else.
- **Observability.** Embedding latency, token counts, and error rates land in the same metrics pipeline as the rest of your service.

Trade-off: the app must know the model. If three services write to the same index, three services need to agree on the model name. That coordination cost is the main reason teams reach for the in-cluster option.

---

## 3. In-Cluster Embedding via Ingest Pipeline

Elasticsearch's [inference processor](https://www.elastic.co/guide/en/elasticsearch/reference/current/inference-processor.html) and OpenSearch's ML Commons connectors push the embedding call inside the cluster. A document arrives at the ingest pipeline; the processor calls the configured model (a deployed local model, an OpenAI connector, a Cohere connector, etc.); the resulting vector is added to the document before it lands in the index.

The Elasticsearch processor configuration looks roughly like:

```json
{
  "description": "Embed article body via the configured inference endpoint",
  "processors": [
    {
      "inference": {
        "model_id": "my-embedding-endpoint",
        "input_output": [
          { "input_field": "text", "output_field": "embedding" }
        ]
      }
    }
  ]
}
```

What you gain:

- **One source of truth** for which model produces vectors for the index.
- **No app-side code path** for embedding — the indexer just sends `{ text: "..." }`.
- **Tighter coupling with semantic_text** in modern ES, which can manage chunking and inference together.

What you give up:

- **Operational coupling.** Model upgrades, capacity planning, and embedding errors are now cluster concerns. A slow embedding model can back up the ingest queue and stall indexing.
- **Cross-process debugging.** "Why did this document get a zero vector?" now spans the cluster's logs and the model server's logs.
- **Bulk-throughput ceilings.** A processor that calls a remote vendor API serially per doc is much slower than an app-side bulk batch.

Rule of thumb: pick in-cluster when there is one writer and the model is hosted close to the cluster (or deployed inside it). Pick app-side when multiple services write, when you need fine-grained retry control, or when the embedding workload is already part of an existing offline pipeline.

---

## 4. Embedding at Query Time — The Symmetry Constraint

Whatever produced the index-time vectors must also produce the query-time vector. The constraint is exact: same model, same version, same prompt format, same normalization. A query embedded by `text-embedding-3-small` will not retrieve documents embedded by `text-embedding-ada-002`, even though both are 1536-dim and both are "OpenAI embeddings." Cosine similarity between vectors from different models is not meaningful — the spaces are unrelated.

In Elasticsearch, the kNN query can either accept a precomputed `query_vector` (you embed in the app) or a `query_vector_builder` that calls the same inference endpoint used at index time:

```json
{
  "knn": {
    "field": "embedding",
    "k": 10,
    "num_candidates": 100,
    "query_vector_builder": {
      "text_embedding": {
        "model_id": "my-embedding-endpoint",
        "model_text": "how do refresh tokens work"
      }
    }
  }
}
```

The query-time embedding adds one round trip to the model on every search, so latency budgets need to account for it. Cohere, OpenAI, and others publish their embedding latencies, and self-hosted TEI / vLLM are usually faster end-to-end because there is no public-internet hop.

Some embedding families require **asymmetric prompts**: a prefix on documents and a different prefix on queries. The Cohere `embed-v3` family explicitly takes an `input_type` parameter (`search_document` for the corpus, `search_query` for queries), and the embeddings differ accordingly per Cohere's docs. Several BGE and instruction-tuned sentence-transformers models do the same trick with a literal text prefix like `"Represent this query for retrieving relevant passages: "`. Forgetting the asymmetry on the query path silently hurts recall.

---

## 5. Model Versioning and Reindex Migrations

Vectors are immutable in the dimension a model produces them in. Changing models means re-embedding the entire corpus — a full reindex, not an in-place migration.

The standard pattern uses an alias and a double-write window:

1. Create a new index `articles_v2` with the new vector field (possibly different dimensions).
2. Backfill `articles_v2` from a snapshot of `articles_v1`, embedding every document with the new model.
3. Configure the writer to **double-write**: every new doc lands in both indexes, embedded by both models. Skip if you can tolerate the gap.
4. Switch the read alias `articles` from `v1` to `v2`.
5. Stop double-writing; drop `v1` once you are sure rollback is unnecessary.

Two pitfalls in this dance:

- **Don't forget the query path.** The reader needs to swap the model name in the same atomic step as the alias swap, or queries hit `v2` with the old query embeddings.
- **Plan capacity for the double-write window.** You are paying for two embedding calls per document and two index writes for as long as the cutover lasts.

For very large corpora, the backfill is the expensive part — it can take days to weeks against a vendor API, and the embedding bill for that backfill is often the largest single line item in a vector-search project.

---

## 6. Dimensionality — 384 vs 768 vs 1024+

Dimensions are not free. Every vector costs `dim × 4` bytes (float32), and HNSW graph storage roughly doubles that. For 100M documents:

| Dim | Per-vector bytes | Raw storage | Notes |
|-----|------------------|-------------|-------|
| 384 | 1,536 | ~143 GB | `all-MiniLM-L6-v2` baseline |
| 768 | 3,072 | ~286 GB | Default for many BERT-family models |
| 1024 | 4,096 | ~382 GB | `text-embedding-3-large` family |
| 1536 | 6,144 | ~572 GB | OpenAI `text-embedding-3-small` default |
| 3072 | 12,288 | ~1.1 TB | OpenAI `text-embedding-3-large` default |

Newer models support [Matryoshka representation learning](https://platform.openai.com/docs/guides/embeddings) — OpenAI's `text-embedding-3-small` and `-large` accept a `dimensions` parameter that truncates the vector while preserving most of its quality. This is the right knob to reach for when storage costs sting: drop `text-embedding-3-large` from 3072 to 1024 dims and you cut vector storage by roughly 3× with a small recall penalty per OpenAI's published benchmarks.

Higher dims also slow down the HNSW search itself — distance computations are linear in dim, and graph traversal touches many neighbors per query.

---

## 7. Batching, Throughput, and Bulk Indexing

Embedding APIs are designed to be batched. The OpenAI Embeddings API accepts an array of inputs in a single call; Cohere's `/embed` endpoint takes a `texts: [...]` array; Voyage and Jina behave similarly. The per-token cost is the same, but the per-request overhead disappears and you stay well under request-rate limits.

Three rules for bulk-indexing pipelines:

1. **Embed in batches that match the API's input cap, not your ES bulk size.** OpenAI's documented cap is 2,048 input items per request and a token cap that varies by model — batch up to the smaller of those two limits.
2. **Decouple embedding throughput from ES bulk throughput.** Use a queue between the two stages so a slow ES bulk doesn't back up embedding, and an embedding 429 doesn't stall the indexer.
3. **Track tokens, not just requests.** Embedding APIs bill on tokens; a runaway tokenizer (a doc with no whitespace, a base64 blob accidentally indexed) can blow your bill before it shows up in request counts.

For a 10M-doc backfill, expect the embedding stage to be the bottleneck, not ES. A vendor API at, say, 1M tokens per minute and 500 tokens per doc gives ~2,000 docs/min — about 3.5 days for 10M docs. Self-hosted TEI on a single GPU can move 10–100× faster but introduces capacity planning of its own.

---

## 8. Vendor APIs and Self-Hosted Options

The realistic shortlist as of 2026:

**Hosted APIs:**

- **OpenAI** — `text-embedding-3-small` (1536 dim default, configurable down) and `text-embedding-3-large` (3072 dim default, configurable down). Documented at the [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings).
- **Cohere** — `embed-english-v3.0` and `embed-multilingual-v3.0` families, 1024 dim, with the asymmetric `input_type` parameter described above. See [Cohere Embed docs](https://docs.cohere.com/docs/embeddings).
- **Voyage AI** — `voyage-3` and domain-specific variants (code, finance, law).
- **Jina AI** — `jina-embeddings-v3` family, multilingual, supports task-specific prompts.

**Open-weights / self-hostable:**

- **sentence-transformers** — the canonical Python library; the [sbert.net documentation](https://www.sbert.net/) catalogs hundreds of models. `all-MiniLM-L6-v2` (384 dim) is the lightweight workhorse; `all-mpnet-base-v2` (768 dim) is the higher-quality default.
- **BGE family** (BAAI) — `bge-small-en`, `bge-base-en`, `bge-large-en` and the multilingual `bge-m3`. Use the documented query prefix.
- **Hugging Face Text Embeddings Inference (TEI)** — a production server for embedding models, supporting most of the open-weights families above. See [TEI docs](https://huggingface.co/docs/text-embeddings-inference).
- **vLLM** — primarily an LLM serving engine but increasingly used for embedding workloads at high throughput.

For a TS/Node service that needs an offline embedding job, the common shape is: spin up TEI in a container behind a small HTTP client, batch through it from a worker process, write to ES via the bulk API. For Spring Boot, the equivalent uses Spring AI's [`EmbeddingModel`](https://docs.spring.io/spring-ai/reference/api/embeddings.html) abstraction with the relevant client (OpenAI, Cohere, or a local Ollama/TEI endpoint).

---

## 9. Streaming vs Batch Reembedding

Two scheduling shapes:

- **Streaming (real-time).** Every new or updated document is embedded immediately and written to the index. Right for content where freshness matters — news, chat messages, support tickets.
- **Batch (nightly or weekly).** A scheduled job re-embeds either the full corpus (after a model upgrade) or just the changed slice. Right for stable corpora where a few hours of staleness is fine.

Most production systems do both: the writer embeds new documents in real time, and a separate batch job handles model migrations and bulk reembedding. The two paths share the same code for "embed this text" but call it from different orchestrators.

---

## 10. Storage Layout — Same Document or Sidecar Index?

Two options:

- **Same document.** The vector lives as a `dense_vector` field on the existing doc, alongside the indexed text and metadata.
- **Sidecar index.** A separate index keyed by doc ID holds only `{id, embedding}`; queries join the two.

In nearly every case, **same document wins** — locality matters for hybrid search. A typical ranking strategy is BM25 + kNN combined via [Reciprocal Rank Fusion](https://www.elastic.co/guide/en/elasticsearch/reference/current/rrf.html) or a `rank` clause; both legs hit the same shard, the same segment files, the same OS page cache. Splitting the vector into a sidecar doubles the I/O and adds an application-side join.

The only good reason to use a sidecar is when the vectors are owned by a different team or service and you cannot accept the coupling. Even then, an alias or transform that materializes the vectors into the main index is usually cleaner.

---

## 11. Practical Traps

The recall-killers that look fine in code review:

- **Prompt-prefix mismatch.** Indexing with `"Represent this passage: "` and querying without it (or vice versa). The vectors land in nearby-but-wrong neighborhoods of the space, and recall drops without any error showing up.
- **Silent token-limit truncation.** OpenAI `text-embedding-3-small` accepts up to 8,191 tokens per input and silently truncates beyond that on the API side. A 50-page PDF dumped in as one input is embedded as its first ~30 KB only — the rest of the doc is invisible to search. Always tokenize and chunk before sending.
- **Chunking strategy chosen by accident.** Sentence chunks miss cross-sentence context; paragraph chunks bloat token cost and average out topical signal; semantic chunks (split on topic shifts) are the best default for prose but require a chunker. Pick deliberately and write the chunker as its own module so you can change it.
- **Mixing models across the index.** A backfill that used model A and a real-time writer that uses model B produce a corpus that no single query model can search well. Enforce model name as a per-index invariant.
- **Forgetting normalization.** Some models output normalized vectors; some don't. Cosine similarity is invariant to magnitude, but dot-product scoring is not. Match the similarity metric in the index mapping (`cosine`, `dot_product`, `l2_norm`) to what the model produces.
- **Embedding stale text.** If the source doc updates but the embedding doesn't, the index still answers from the old vector. Tie the embedding job to the same change-data-capture stream that drives the doc write.

These all fail the same way: zero errors, lower recall, a slow drift in answer quality that takes weeks to attribute. Catch them with a held-out evaluation set that runs against every index after a model or pipeline change — even 50 hand-labeled query/doc pairs are enough to detect a regression that scaled metrics will miss.

---

## Related

- _Vector Search Foundations — Dense Embeddings, ANN, and HNSW (planned)_ — what the vectors produced here are used for
- _Hybrid Search — BM25 + Vectors with RRF (planned)_ — how the vectors generated by this pipeline combine with the lexical scoring covered in earlier tier docs
- [Inverted Index Internals](../foundations/01-inverted-index-internals.md) — the lexical side of the same hybrid query
- [Database — Polyglot Persistence](../../database/INDEX.md) — vectors as one more derived column problem
- [Performance — Latency, Throughput, Percentiles](../../performance/INDEX.md) — framework for budgeting the query-time embedding round trip

## References

- [OpenAI — Embeddings Guide](https://platform.openai.com/docs/guides/embeddings) — `text-embedding-3-small` (1536 default dim) and `text-embedding-3-large` (3072 default dim), the `dimensions` parameter for Matryoshka truncation, and the per-request input/token caps cited in §6 and §7
- [Cohere — Embed API](https://docs.cohere.com/docs/embeddings) — `embed-english-v3.0` / `embed-multilingual-v3.0` families and the `input_type` parameter (`search_document` vs `search_query`) used to produce asymmetric query/document embeddings
- [Sentence Transformers (SBERT) Documentation](https://www.sbert.net/) — `all-MiniLM-L6-v2` (384 dim) and `all-mpnet-base-v2` (768 dim) reference models and the prompt-prefix conventions for instruction-tuned models
- [Hugging Face — Text Embeddings Inference (TEI)](https://huggingface.co/docs/text-embeddings-inference) — the official self-hosted serving option referenced in §8
- [Elastic — Inference Processor](https://www.elastic.co/guide/en/elasticsearch/reference/current/inference-processor.html) — ingest-pipeline-side embedding configuration described in §3
- [Elastic — kNN Search and `query_vector_builder`](https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html) — the query-time embedding configuration used in §4
- [Elastic — Reciprocal Rank Fusion (RRF)](https://www.elastic.co/guide/en/elasticsearch/reference/current/rrf.html) — combining BM25 and vector legs for hybrid search referenced in §10
- [OpenSearch — ML Commons](https://opensearch.org/docs/latest/ml-commons-plugin/index/) — connector framework that plays the same role as Elastic's inference processor
- [Spring AI — Embeddings](https://docs.spring.io/spring-ai/reference/api/embeddings.html) — the `EmbeddingModel` abstraction referenced for the Spring Boot side
