---
title: "Vector Search Fundamentals — Embeddings, ANN, and HNSW"
date: 2026-05-07
updated: 2026-05-07
tags: [search, vector, hnsw, ann, embeddings, elasticsearch, opensearch]
---

# Vector Search Fundamentals — Embeddings, ANN, and HNSW

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `vector` `hnsw` `ann` `embeddings` `elasticsearch` `opensearch`

---

## Table of Contents

- [Summary](#summary)
- [1. Why Vector Search Exists](#1-why-vector-search-exists)
- [2. Distance Metrics — Cosine, Dot Product, L2](#2-distance-metrics--cosine-dot-product-l2)
- [3. Why Approximate Instead of Exact](#3-why-approximate-instead-of-exact)
- [4. HNSW — How the Graph Works](#4-hnsw--how-the-graph-works)
- [5. IVF — The Clustering Alternative](#5-ivf--the-clustering-alternative)
- [6. Quantization — Buying Memory Back](#6-quantization--buying-memory-back)
- [7. Dimensions and the Memory Bill](#7-dimensions-and-the-memory-bill)
- [8. Elasticsearch `dense_vector` and kNN Search](#8-elasticsearch-dense_vector-and-knn-search)
- [9. OpenSearch k-NN Plugin](#9-opensearch-k-nn-plugin)
- [10. Practical Patterns and Traps](#10-practical-patterns-and-traps)
- [Related](#related)
- [References](#references)

---

## Summary

Lexical search (BM25, the inverted-index world covered in Tier 1 of this path) matches documents by the literal terms they contain. It is fast, explainable, and blind to meaning: "car" and "automobile" are unrelated tokens unless you also indexed synonyms. **Vector search** sidesteps that limitation by encoding each document into a fixed-dimensional vector with an embedding model, then ranking documents by *geometric* proximity to the query vector. Two texts that mean the same thing land near each other in that space whether or not they share any tokens.

Two engineering problems make vector search non-trivial. First, the brute-force ranker is `O(n × d)` per query — for a billion 768-dimensional vectors, that is hundreds of billions of multiply-adds per query, which is unusable. Second, embeddings are dense float arrays, so a billion 768-d float32 vectors is roughly 3 TB of pure vector data before any index overhead. The solutions are **approximate nearest neighbor (ANN)** algorithms — most prominently HNSW (a multi-layer proximity graph) and IVF (an inverted file over k-means clusters) — paired with **quantization** to compress the vectors themselves. This doc walks the data model end to end and lands on how Elasticsearch's `dense_vector` field and OpenSearch's k-NN plugin expose these primitives.

---

## 1. Why Vector Search Exists

BM25 ranks `bank account fraud` by overlap with the literal tokens `bank`, `account`, `fraud`. A document titled *"Detecting suspicious wire transfers in retail banking"* shares only `banking` (after stemming) and probably ranks below a document that uses the word `fraud` three times in a marketing pitch. The lexical ranker has no way to know that "wire transfer fraud" is the same concept as "account fraud."

Embedding models — sentence-transformers, OpenAI `text-embedding-3`, Cohere Embed, the `all-MiniLM-L6-v2` family, etc. — map a chunk of text to a point in a fixed-dimensional vector space (commonly 384, 768, 1024, 1536, or 3072 dimensions, depending on the model). Training pushes texts with similar meaning to similar coordinates and pushes unrelated texts apart. At query time, you embed the query with the *same model* and ask "which document vectors are closest to this query vector?" The answer is a ranked list by semantic similarity, free of the synonym blindness that plagues BM25.

The catch is that everything you knew from the inverted-index world — exact term matches, prefix queries, phrase queries — does not apply here. There is no term, no posting list. There is only a giant cloud of points and a need to find the nearest ones quickly. Most production systems run vector search **alongside** BM25 (a "hybrid" pattern) rather than replacing it: BM25 catches exact-string queries like SKUs and proper nouns; vectors catch paraphrase and intent.

---

## 2. Distance Metrics — Cosine, Dot Product, L2

Three similarity functions dominate. Choosing among them is determined by how the embedding model was trained, not by personal preference.

**Cosine similarity** measures the angle between two vectors, ignoring magnitude:

```text
cos(a, b) = (a · b) / (||a|| × ||b||)
```

Output is in `[-1, 1]`; `1` means identical direction. Cosine is the right choice for most NLP embeddings because the model was trained with a contrastive loss that normalizes magnitudes anyway. If your vectors are pre-normalized to unit length, cosine and dot product produce *identical* rankings — at which point dot product is the cheaper computation.

**Dot product** is `a · b` with no normalization. It is the natural metric when the model produces unnormalized vectors whose magnitude carries information (for example, some retrieval models trained on Maximum Inner Product Search). Use it when the model card says "use inner product."

**L2 (Euclidean) distance** is `sqrt(sum((a_i - b_i)²))`. It treats the vectors as points and ranks by straight-line distance. L2 is common in image embeddings and in older information-retrieval research; it is less common in modern NLP because magnitude differences across documents (which L2 is sensitive to) usually do not carry semantic information.

A useful identity: for unit-normalized vectors, `||a - b||² = 2 - 2(a · b)`, so L2 ranking and cosine ranking agree. The choice only matters when vectors are not normalized.

---

## 3. Why Approximate Instead of Exact

Exact k-nearest-neighbor against `n` vectors of dimension `d` is `O(n × d)` per query: every query must touch every indexed vector. For 1 million 768-d vectors, that is ~768 million multiply-adds per query — feasible at small scale but linear in `n`. At 1 billion vectors, exact kNN is 768 *billion* operations per query, which is well past any latency budget.

The "curse of dimensionality" makes the problem worse in subtle ways. In high-dimensional spaces, distances between random points concentrate around a single value: most points end up roughly equidistant from any given query, which destroys the pruning power of classical tree-based structures (k-d trees, ball trees) that work well in 2-D or 3-D. By the time `d` exceeds 20 or so, k-d trees degenerate to scanning most of the data anyway.

ANN algorithms accept a small recall loss (you may miss a few of the true top-k results) in exchange for sub-linear query time. A well-tuned HNSW index can return the top-10 with recall@10 above 0.95 while touching well under 1% of the indexed vectors — the difference between a 5 ms query and a 5-second query.

---

## 4. HNSW — How the Graph Works

**Hierarchical Navigable Small World** (Malkov & Yashunin, 2016) is the dominant graph-based ANN structure. It is what Lucene, Elasticsearch, OpenSearch's Lucene engine, Qdrant, Weaviate, and pgvector's `hnsw` index type all use under the hood.

The structure is a multi-layer proximity graph. Every vector is a node; edges connect each node to a small set of its nearest neighbors. The layers form a hierarchy: layer 0 contains *every* node; layer 1 contains a probabilistic subset; layer 2 a smaller subset; and so on, with the top layer holding only a handful of nodes. Each node is randomly assigned a max layer when inserted, with the probability of reaching higher layers decaying exponentially.

Search starts at the top layer at a fixed entry point and **greedily walks toward the query**: at each node, examine its neighbors, move to whichever neighbor is closest to the query, repeat until no neighbor is closer. When the walk converges at a layer, drop down one layer and resume from the same node. At layer 0, instead of greedy descent, the algorithm runs a beam search of width `ef` to refine the candidate list to the final top-k.

Three knobs govern the cost / accuracy trade-off:

- **`M`** — the max number of edges (neighbors) each node keeps per layer. Higher `M` means a denser graph, better recall, more memory. The HNSW paper recommends `M` between 5 and 48; the practical range for production is roughly 16–64. Lucene's default is `M = 16`.
- **`ef_construction`** — the candidate list size used while *inserting* a node. Higher values mean each new node finds better neighbors at build time, improving recall at the cost of slower indexing. Lucene's default is `ef_construction = 100`.
- **`ef_search`** (called `num_candidates` in Elasticsearch's kNN search API, `ef` in many libraries) — the candidate list size at *query* time. Higher means more graph nodes are visited, recall goes up, latency goes up. This is the runtime knob you tune to hit a recall@k target.

Because the graph is built incrementally, new vectors are inserted by running the same search procedure and stitching the new node into its `M` nearest neighbors. There is no global rebuild needed — but the index is also append-only with respect to the underlying graph, and deletions are handled with tombstones similar to inverted-index segments.

---

## 5. IVF — The Clustering Alternative

**Inverted File Index (IVF)** is the other widespread ANN family, popularized by FAISS. It works by clustering: run k-means on a sample of the corpus to learn `nlist` centroids, then assign every vector to its nearest centroid. The "inverted" terminology mirrors the lexical inverted index — each centroid keeps a list of vectors assigned to it.

At query time, the algorithm computes the query's distance to each centroid, picks the `nprobe` nearest centroids, and exhaustively scans only the vectors assigned to those clusters. Tuning `nprobe` trades latency against recall, the same way `ef_search` does in HNSW. IVF builds faster than HNSW and uses less RAM (the index is a list of cluster IDs and centroids, not a graph), but its query path is fundamentally a brute-force scan within the chosen clusters — recall is more sensitive to cluster boundaries.

IVF shines when paired with quantization (next section): the FAISS combo `IVF{nlist},PQ{m}` is the standard way to fit billion-scale vector indexes in RAM. OpenSearch's k-NN plugin exposes IVF through its FAISS engine; Elasticsearch's primary path is HNSW only.

---

## 6. Quantization — Buying Memory Back

A 768-d vector at float32 is `768 × 4 = 3,072 bytes`. Multiply by a billion documents and the raw vectors alone are ~3 TB, before HNSW edge data or IVF metadata. Quantization compresses each vector with a controlled accuracy loss.

**Scalar quantization (SQ)** maps each float32 dimension to a smaller integer — int8 is the common target. Each dimension is rescaled to the `[-128, 127]` range using per-dimension or global min/max, then stored as one byte. Memory drops 4×; recall typically drops by 1–3 percentage points after re-ranking the top candidates with the original vectors. Lucene's `int8_hnsw` index option implements this; Elasticsearch supports `int8_hnsw` (1 byte/dim) and `int4_hnsw` (4 bits/dim) on `dense_vector`.

**Product quantization (PQ)** is more aggressive. Split each `d`-dimensional vector into `m` sub-vectors of length `d/m`, run k-means independently on each sub-vector space to learn a small codebook (commonly 256 centroids per sub-vector, fitting in 1 byte each), then encode each sub-vector as the index of its nearest codebook entry. A 768-d vector with `m = 96` and 256 codebook entries per sub-vector compresses to 96 bytes — a 32× reduction. PQ is the engine behind FAISS's billion-scale indexes.

**Binary quantization** is the most extreme: each dimension is reduced to a single bit (sign of the float). Distance computations become Hamming distance on bit vectors, which CPUs execute extremely fast (`POPCNT`). The accuracy hit is large enough that binary quantization is usually used as a coarse first stage with a re-rank by the original vectors over the top candidates.

The general rule across all three: quantize for the bulk of the index, keep the original vectors available for an optional re-ranking pass on the top `k × oversampling` candidates.

---

## 7. Dimensions and the Memory Bill

A back-of-envelope sizing model:

| Vectors | Dim | Encoding | Raw vector bytes |
|---------|-----|----------|------------------|
| 1 M | 768 | float32 | 3.07 GB |
| 10 M | 768 | float32 | 30.7 GB |
| 100 M | 768 | float32 | 307 GB |
| 1 B | 768 | float32 | 3.07 TB |
| 1 B | 768 | int8 (SQ) | 768 GB |
| 1 B | 768 | PQ (m=96) | 96 GB |

HNSW adds graph overhead on top of the raw vectors — roughly `M × 4` bytes per node per layer for neighbor IDs (Lucene stores neighbor lists as int32). For `M = 16` and a single dominant layer, that is another ~64 bytes per vector. At 1 B vectors with int8 quantization plus HNSW edges, the working set is ~830 GB — large, but fits across a small cluster of memory-optimized nodes. At float32 with no quantization, the same workload is ~3.1 TB and effectively requires sharding plus disk-backed structures.

The single most important decision in vector-search capacity planning is whether your hot vector data needs to live in memory. Elasticsearch's `dense_vector` field is loaded into the JVM heap for the HNSW graph traversal portion, with the actual vector values on disk via mmap; the OS page cache is what makes that survivable. For multi-billion-scale workloads, quantization is not optional.

---

## 8. Elasticsearch `dense_vector` and kNN Search

Elasticsearch defines vector fields with the `dense_vector` mapping. A minimal example:

```json
PUT /docs
{
  "mappings": {
    "properties": {
      "embedding": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "int8_hnsw",
          "m": 16,
          "ef_construction": 100
        }
      }
    }
  }
}
```

`similarity` is one of `cosine`, `dot_product`, `l2_norm`, or `max_inner_product` — verify against the current Elasticsearch reference for your version, since options have evolved. `index_options.type` selects the on-disk representation: `hnsw` for raw float32 + HNSW, `int8_hnsw` for scalar-quantized HNSW, `int4_hnsw` for 4-bit quantization, `flat` for brute-force only, plus quantized flat variants.

Search uses the `knn` block in the `_search` body:

```json
POST /docs/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.12, -0.07, ...],
    "k": 10,
    "num_candidates": 100
  },
  "_source": ["title", "url"]
}
```

`k` is the number of results returned. `num_candidates` is the equivalent of `ef_search` — the size of the candidate pool the HNSW search keeps as it walks the graph; larger means higher recall and higher CPU. Elastic's documentation recommends `num_candidates >= k`, typically several times larger.

The `knn` block can sit alongside a `query` block to combine vector and lexical scoring; Elasticsearch supports several hybrid patterns (`rrf` reciprocal rank fusion is one). The hybrid story is its own doc.

---

## 9. OpenSearch k-NN Plugin

OpenSearch exposes vector search through the **k-NN plugin**, with a slightly different mapping shape and three pluggable engines: `lucene` (HNSW only, mirrors Elasticsearch's behavior), `faiss` (HNSW *and* IVF, with quantization), and the older `nmslib` (HNSW). The engine choice determines which algorithms and quantization options are available.

```json
PUT /docs
{
  "settings": { "index.knn": true },
  "mappings": {
    "properties": {
      "embedding": {
        "type": "knn_vector",
        "dimension": 768,
        "method": {
          "name": "hnsw",
          "space_type": "cosinesimil",
          "engine": "lucene",
          "parameters": { "m": 16, "ef_construction": 100 }
        }
      }
    }
  }
}
```

Querying uses a `knn` query type:

```json
POST /docs/_search
{
  "query": {
    "knn": {
      "embedding": {
        "vector": [0.12, -0.07, ...],
        "k": 10
      }
    }
  }
}
```

The `space_type` maps to the distance metric (`cosinesimil`, `l2`, `innerproduct`, etc.). Quantization options vary by engine — the `faiss` engine supports PQ and SQ; the `lucene` engine supports byte-quantized HNSW (`encoder` parameter). Verify the exact parameter names against the current OpenSearch docs for your version, as the plugin's surface has shifted.

---

## 10. Practical Patterns and Traps

A few things experienced teams learn the hard way:

**Pick `M` and `ef_search` from a recall target, not a guess.** The right workflow is to build a small index with realistic vectors, generate a labeled set of query → ground-truth-top-k pairs (computed by exact kNN on the small index), then sweep `ef_search` until recall@k crosses your target. Only then measure latency. Teams that start by picking `M = 64` "for safety" pay 4× the memory cost and never measure whether they needed it.

**Re-embedding when you change models is the migration cost.** Embeddings from `text-embedding-ada-002` are not comparable to embeddings from `text-embedding-3-small`. If you swap the model, every document in the index must be re-embedded and re-indexed. For a 100M-document corpus at 1,000 docs/sec embedding throughput, that is a 28-hour batch job before reindex even starts.

**Dimension mismatches fail at query time.** If your index expects 768-d vectors and your query embedder ships 1024-d vectors (because someone updated the model), Elasticsearch will reject the query. Pin the embedding model version somewhere your indexing pipeline and your query pipeline both read.

**Vector field changes require reindex.** You cannot change `dims`, `similarity`, or the underlying `index_options.type` on an existing `dense_vector` field. Switching from `hnsw` to `int8_hnsw` is a reindex into a new field or a new index, not an in-place migration. Plan for this in your mapping design from day one.

**Quantization helps only if vectors dominate your memory.** If the index is small and the cluster has plenty of RAM, the integration complexity and accuracy hit of `int8_hnsw` may not be worth it. Quantize when sizing pushes you toward more nodes than the BM25 portion of the workload would otherwise justify.

**Hybrid search usually wins.** BM25 is unbeatable for SKU lookups, exact-quote search, and any query where the user is searching for a literal string. Vector search wins on paraphrase, intent, and natural-language questions. Production retrieval systems combine both — retrieve `k` candidates from each, fuse by reciprocal rank or a learned reranker, and serve the merged list. Vector-only retrieval is a common over-correction by teams new to embeddings.

---

## Related

- [Inverted Index Internals — Postings Lists, Skip Lists, and FSTs](../foundations/01-inverted-index-internals.md) — the lexical world that vector search complements
- _BM25 and TF-IDF — Relevance Scoring from First Principles (planned)_ — the lexical scoring layer that hybrid retrieval pairs with vector ranking
- _Hybrid Retrieval and Reciprocal Rank Fusion (planned)_ — combining BM25 and vector results
- [Database — Polyglot Persistence](../../database/INDEX.md) — pgvector as an alternative when your data already lives in Postgres
- [Performance — Latency, Throughput, Percentiles](../../performance/INDEX.md) — the framework for sizing recall vs latency trade-offs
- [System Design — Capacity Planning](../../system-design/INDEX.md) — context for the memory math in §7

## References

- [Yu. A. Malkov and D. A. Yashunin — *Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs*, 2016 (arXiv:1603.09320)](https://arxiv.org/abs/1603.09320) — the original HNSW paper; introduces the multi-layer graph, greedy descent, and the `M` / `ef_construction` / `ef` parameters
- [Elasticsearch Reference — `dense_vector` field type](https://www.elastic.co/guide/en/elasticsearch/reference/current/dense-vector.html) — authoritative source for `dims`, `similarity`, and `index_options` (including `int8_hnsw` and `int4_hnsw`)
- [Elasticsearch Reference — k-nearest neighbor (kNN) search](https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html) — the `knn` search block, `num_candidates` semantics, and hybrid query patterns
- [OpenSearch Documentation — k-NN plugin](https://opensearch.org/docs/latest/search-plugins/knn/index/) — `knn_vector` mapping, engine choice (`lucene`, `faiss`, `nmslib`), and `space_type` options
- [OpenSearch Documentation — k-NN index settings and methods](https://opensearch.org/docs/latest/search-plugins/knn/knn-index/) — HNSW vs IVF parameters, quantization encoders
- [FAISS Wiki — Indexing 1M vectors](https://github.com/facebookresearch/faiss/wiki/Indexing-1M-vectors) and [Faster search](https://github.com/facebookresearch/faiss/wiki/Faster-search) — IVF, IVF-PQ, and quantization recipes from the library that defined the field
- [FAISS Wiki — Guidelines to choose an index](https://github.com/facebookresearch/faiss/wiki/Guidelines-to-choose-an-index) — practical decision tree for HNSW vs IVF vs IVFPQ at different corpus sizes
- [Lucene 9.x — `KnnVectorsFormat` and `Lucene99HnswVectorsFormat`](https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/codecs/lucene99/Lucene99HnswVectorsFormat.html) — the underlying HNSW implementation used by Elasticsearch and OpenSearch's Lucene engine; documents the default `M = 16` and `beamWidth = 100` (ef_construction)
