---
title: "Inverted Index Internals — Postings Lists, Skip Lists, and FSTs"
date: 2026-05-07
updated: 2026-05-07
tags: [search, lucene, elasticsearch, opensearch, inverted-index, fst, postings]
---

# Inverted Index Internals — Postings Lists, Skip Lists, and FSTs

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `lucene` `elasticsearch` `opensearch` `inverted-index` `fst` `postings`

---

## Table of Contents

- [Summary](#summary)
- [1. The Problem an Inverted Index Solves](#1-the-problem-an-inverted-index-solves)
- [2. The Two Halves — Terms Dictionary and Postings Lists](#2-the-two-halves--terms-dictionary-and-postings-lists)
- [3. The Terms Dictionary as an FST](#3-the-terms-dictionary-as-an-fst)
- [4. Postings — What's Inside `.doc`, `.pos`, `.pay`](#4-postings--whats-inside-doc-pos-pay)
- [5. Compression — Delta + Frame-of-Reference + Patched Outliers](#5-compression--delta--frame-of-reference--patched-outliers)
- [6. Skip Lists — Cheap Random Access into a Compressed Stream](#6-skip-lists--cheap-random-access-into-a-compressed-stream)
- [7. Segments — The Unit of Immutability](#7-segments--the-unit-of-immutability)
- [8. The Full Lucene File Family](#8-the-full-lucene-file-family)
- [9. What This Means for Query Latency](#9-what-this-means-for-query-latency)
- [Related](#related)
- [References](#references)

---

## Summary

An inverted index is two data structures glued together: a **terms dictionary** (sorted set of every term in the corpus, with a pointer per term) and **postings lists** (per-term sorted lists of document IDs, with positions and offsets if you indexed them). Lucene — the engine inside Elasticsearch and OpenSearch — implements the dictionary as a Finite State Transducer (FST) for prefix-shared, RAM-resident lookup, and stores postings as block-compressed delta-encoded integers on disk with a thin skip list overlay so query execution can `advance(targetDocID)` through millions of postings without decompressing the whole list. This doc walks the structure end-to-end, names the actual files Lucene writes (`.tim`, `.tip`, `.doc`, `.pos`, `.pay`, …), and explains why each design choice exists.

The single most useful mental shift for backend engineers coming from a relational background: **the inverted index is the read path's only data structure**. There is no row store consulted "later." A SQL query against `WHERE name = 'foo'` typically traverses a B-tree to a row pointer, then heap-fetches the row; an inverted-index query against `name:foo` traverses an FST to a postings-list pointer, walks that list to a doc ID, and the doc ID is everything. Stored field retrieval (`_source`) is a separate, optional file family (`.fdt`/`.fdx`) accessed only when the query asks for it.

---

## 1. The Problem an Inverted Index Solves

Given a corpus of documents and a query *"find documents containing the term `kafka`"*, the naive approach is a full scan: read every document, check whether it contains `kafka`. That is `O(N)` in corpus size on every query and unusable beyond a small dataset.

An inverted index trades a one-time indexing cost for `O(log T + |postings|)` queries: `log T` to find the term in the dictionary (where `T` is unique-term count, dramatically smaller than corpus token count), then a linear walk over the matching documents. For `name:foo` against a 10-million-document index where `foo` appears in 1,000 docs, the engine reads ~1,000 sorted integers — milliseconds, not minutes.

Two further capabilities fall out for free:

- **Boolean queries** (`a AND b`) — intersect the postings lists of `a` and `b`. Both lists are sorted, so this is a merge-style walk.
- **Phrase queries** (`"a b"`) — postings can store *positions*, so the engine walks the doc IDs that contain both `a` and `b`, then checks each match for adjacent positions.

The cost is on the write path: every indexed token enters the dictionary, every doc ID enters a postings list, and the whole structure is rebuilt as new documents arrive (more on segments in §7).

---

## 2. The Two Halves — Terms Dictionary and Postings Lists

Every inverted index has the same shape:

```text
Terms Dictionary           Postings Lists
─────────────────          ──────────────
"alice"   → ptr────────►   [3, 17, 42, 105, ...]
"bob"     → ptr────────►   [1, 17, 88]
"carol"   → ptr────────►   [42, 99, 250, ...]
...
"zachary" → ptr────────►   [7, 12]
```

The dictionary is sorted (so the engine can do range scans, prefix queries, and binary-style lookups); each entry points to a per-term postings list elsewhere on disk.

In Lucene, the dictionary lives in two files — `.tim` (term info: the actual term bytes, doc frequency, and a pointer into postings) and `.tip` (term index: an in-memory FST that points into `.tim`). The postings live across `.doc` (doc IDs and term frequencies), `.pos` (per-occurrence positions), and `.pay` (payloads and character offsets, when present). Each of those files is described in the official Lucene index file format documentation.

The split between *index* (`.tip`) and *info* (`.tim`) is deliberate: the `.tip` is small enough to keep in RAM and answer "where do I land in `.tim` for term `T`?", and the `.tim` lives on disk and answers "what's the doc-frequency and postings pointer for `T`?". Together they bound RAM usage while keeping lookup to one disk seek per term.

---

## 3. The Terms Dictionary as an FST

Lucene's term index (`.tip`) is a **Finite State Transducer**, not a sorted array or a B-tree. An FST is a finite-state machine where each accepted input string emits an output value — in Lucene's case, the input is a term's bytes and the output is the offset into `.tim` for that term's metadata.

The reason is RAM economy. A 10-million-term Wikipedia index built into an FST is on the order of **69 MB** of RAM in Lucene committer Mike McCandless's reported measurements, versus hundreds of MB for the equivalent `TreeMap<BytesRef, Long>`. The savings come from prefix and suffix sharing: terms `"kafka"`, `"kafkaesque"`, `"kafkian"` share the prefix `"kafk"` once in the FST graph rather than three times.

The structure was added to Lucene's terms dictionary path in the early 4.x series, replacing earlier in-memory term-index implementations. Lookup is a walk down the arcs of the automaton, summing per-arc outputs to compute the final pointer:

```text
       k         a       f       k       a
START ────► s1 ──► s2 ──► s3 ──► s4 ──► s5 (terminal: output = 4)
```

Walking `k → a → f → k → a` accumulates the output along each arc to land at the postings pointer for `kafka`. The traversal is `O(term_length)` and bounded by the number of arcs traversed, not by the corpus size.

The FST also accelerates non-exact matches: prefix queries (`kaf*`) walk the prefix once and enumerate all states reachable from there; fuzzy queries use Levenshtein automata that can be intersected with the term FST to enumerate all terms within edit distance `k`.

---

## 4. Postings — What's Inside `.doc`, `.pos`, `.pay`

Once the dictionary lookup has produced a postings pointer, the engine starts walking the postings list itself.

- **`.doc`** — for each document containing the term, the doc ID and the term frequency (how many times the term appears in that doc, used by BM25). This is the file that drives `WHERE term = X` queries that don't need positions.
- **`.pos`** — per-occurrence positions. Required for phrase queries (`"new york"`), span queries, and any query that cares *where* in the document the term appeared.
- **`.pay`** — payloads (arbitrary per-position bytes, used by some advanced relevance models) and character offsets (used by highlighters to underline the matched substring in a result snippet).

You pay for the storage of `.pos` and `.pay` only if the field's mapping requested positions/offsets — keyword-only fields (`index_options: docs`) skip them and the postings file family is correspondingly smaller. This is one of the most consequential mapping decisions: a `text` field with positions and offsets can be 5-10× larger on disk than the same field stored as `keyword`-only.

---

## 5. Compression — Delta + Frame-of-Reference + Patched Outliers

A naïve postings list of 32-bit doc IDs would dwarf the original document text. Lucene compresses postings in three layered passes.

### 5.1 Delta encoding

Doc IDs are sorted, so the engine stores the *gap* between consecutive IDs rather than the IDs themselves. A list `[3, 17, 42, 105]` becomes `[3, 14, 25, 63]`. The deltas are smaller integers on average, which feeds the next pass.

### 5.2 Block-based Frame of Reference (FOR)

Postings are split into fixed-size blocks (Lucene's default postings format uses blocks on the order of 128 doc IDs in modern versions; the exact size has shifted over Lucene's history and is documented per codec). Each block is bit-packed: the engine finds the largest delta in the block, computes how many bits that requires (say, 9), and stores every delta in the block as a 9-bit field. A block of 128 deltas where the max needs 9 bits costs `128 × 9 = 1,152` bits (144 bytes) — vastly less than `128 × 32 = 4,096` bits (512 bytes) for raw 32-bit IDs.

### 5.3 Patched Frame of Reference (PFOR)

If one outlier in a block requires 28 bits but everything else fits in 6, FOR alone wastes 22 bits per element. PFOR fixes this by storing the typical width (6 bits) for the block and patching the outliers separately as exceptions. Lucene supports several variants — FOR-delta, PFOR-delta, Simple9 — selected by the codec. Elastic's "Frame of Reference and Roaring Bitmaps" article describes the engineering tradeoffs.

The combined effect: postings lists in production Lucene indexes typically compress to a small fraction of their uncompressed size, and the cost of decompression is a tight per-block loop that modern CPUs handle at near-memory-bandwidth rates.

---

## 6. Skip Lists — Cheap Random Access into a Compressed Stream

Compression is great for storage but bad for random access — to read the 50,000th doc ID in a postings list, you would in principle have to decompress every block before it. That breaks intersection: `term_a AND term_b` wants to advance `term_b`'s postings to "the first doc ID ≥ X" without scanning every doc ID below X.

Lucene solves this with a **skip list** layered over the postings blocks. A skip list records, for every Nth block, the first doc ID and the file pointer where that block starts. To `advance(targetDocID)`:

1. Binary-search the skip list for the largest entry with `firstDocID ≤ targetDocID`.
2. Seek directly to that block's file pointer.
3. Decompress only that block and walk forward.

Lucene's modern postings format uses a single-level skip list with one entry per block. The cost of `advance` becomes `O(log B + block_size)` where `B` is the number of blocks, instead of `O(N)` for a linear scan. This is what makes `AND` queries fast: the engine repeatedly advances each term's postings to the other's current doc ID, jumping over millions of unmatched docs without decompressing them.

---

## 7. Segments — The Unit of Immutability

A Lucene index is not a single inverted index — it is a collection of **immutable segments**, each a complete miniature index with its own `.tim`/`.tip`/`.doc`/`.pos`/etc. files. Documents land in an in-memory buffer; when the buffer fills (or on a refresh interval), Lucene flushes a new segment to disk. Once written, a segment is never modified.

Three things follow from segment immutability:

- **Deletes are tombstones.** Deleting a document writes an entry into the segment's `.liv` ("live docs") bitmap that marks the doc ID as deleted; the postings entries themselves stay. Search still walks the entries but filters out deleted IDs at the top of the loop.
- **Updates are delete + insert.** An "update" deletes the old segment's entry via `.liv` and writes a new entry into the current in-memory segment.
- **Segments accumulate, then merge.** Lucene's merge policy periodically reads several small segments, drops their tombstoned docs, and writes one larger segment. This is when deleted docs actually leave disk, and when segment count stays bounded over time.

The merge policy is itself a tunable subsystem — TieredMergePolicy is the default and balances merge cost against query cost (more segments = more parallelism but more file handles and more dictionary lookups per query). Tier 4 of this learning path will cover merge tuning in detail; for now the model to keep is "indexing is append-only writes to immutable files, plus occasional background merges."

---

## 8. The Full Lucene File Family

For reference when reading an Elasticsearch data directory, here is the full set of file extensions Lucene's default codec writes per segment, drawn from the official Lucene 9.x package documentation:

| Extension | Contents |
|-----------|----------|
| `.tim` | Term info (term bytes, doc frequency, postings pointer) |
| `.tip` | Term index — the FST that points into `.tim` |
| `.doc` | Postings: doc IDs and term frequencies |
| `.pos` | Postings: per-occurrence positions |
| `.pay` | Postings: payloads and character offsets |
| `.fdt` | Stored fields data (`_source` and any `store: true` fields) |
| `.fdx` | Stored fields index (offsets into `.fdt`) |
| `.dvd` | Doc values data — column-oriented per-doc values for sorting/aggregations |
| `.dvm` | Doc values metadata |
| `.nvd` | Norms data — per-field length and boost factors used by BM25 |
| `.nvm` | Norms metadata |
| `.vec`, `.vem`, `.veq`, `.vex` | Vector formats for dense-vector / kNN search |
| `.liv` | Live docs bitmap — which doc IDs in the segment have not been deleted |
| `.cfs`, `.cfe` | Compound file (small segments bundle their files into a single `.cfs`) |
| `.si` | Segment info — references to all of the above |

When you see a segment described as "`_5x.tim`", the `_5x` is the segment generation/name; every file in the segment shares that prefix. An Elasticsearch index is a *shard* (which is a Lucene index = one or more segments), times the number of primary shards, times the replica count.

---

## 9. What This Means for Query Latency

A few rules of thumb that fall out of the structure above:

- **Term-frequency-bound queries are fast.** A query that matches 1,000 docs in a 1B-doc index reads ~1,000 entries in `.doc` plus a single FST traversal; this routinely completes in under 10 ms even on cold cache once `.tip` is loaded.
- **Cardinality-bound queries are slow.** A query matching 100M docs reads 100M entries in `.doc`; even with PFOR-delta and skip lists, you cannot escape walking the matching postings. Aggregations multiply this cost by reading `.dvd` for every matched doc.
- **Heap pressure scales with segment count.** Each open segment keeps its `.tip` FST in heap. Indexes with thousands of small segments per shard burn heap on FSTs and are slow to query because each term lookup hits N FSTs in parallel. This is why merge policy and refresh interval are tuning knobs that cost real money in production.
- **Filesystem cache matters more than heap.** `.doc`/`.pos`/`.pay` live on disk and are mmaped; a hot index is one whose postings sit in the OS page cache, not one with a giant JVM heap. The "give Elasticsearch 50% of RAM, leave 50% for the OS" rule of thumb in the docs is exactly this.
- **Scoring is cheap relative to I/O.** BM25 (covered in the next doc in this tier) is a small per-doc arithmetic operation on top of `.nvd` lookups. The hot path is decompressing postings, not computing scores.

These are the levers Tier 6 (Production Patterns) will revisit when capacity-planning a search cluster.

---

## Related

- _BM25 and TF-IDF — Relevance Scoring from First Principles (planned)_ — the scoring layer that consumes the term frequencies stored in `.doc` and the norms in `.nvd`
- _Analyzers — Tokenization, Filters, and Language-Specific Pipelines (planned)_ — the write-path stage that decides what becomes a "term" in the dictionary
- _Lucene Segments and Merge Policy (planned)_ — Tier 1 deep dive on §7 of this doc
- [Database — B-tree and Index Internals](../../database/INDEX.md) — adjacent storage model in the relational world; useful contrast to FST + postings
- [Operating Systems — Page Cache and Buffered I/O](../../operating-systems/operating-systems-tier1/03-page-cache-and-buffered-io.md) — why mmaped postings + OS page cache is the dominant performance lever
- [Performance — Latency, Throughput, Percentiles](../../performance/INDEX.md) — the framework for reasoning about cardinality-bound search queries

## References

- [Apache Lucene 9.9 — Index File Format Overview](https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/codecs/lucene99/package-summary.html) — authoritative source for the file extensions and what each contains
- [Found / Elastic — Elasticsearch from the Bottom Up](https://www.elastic.co/blog/found-elasticsearch-from-the-bottom-up) — segment immutability, merge policy, postings/dictionary split
- [Mike McCandless — Using Finite State Transducers in Lucene](https://blog.mikemccandless.com/2010/12/using-finite-state-transducers-in.html) — Lucene committer's account of the FST terms dictionary, including the 69 MB / 9.8M-term measurement cited in §3
- [Lucene 4.0 — `org.apache.lucene.util.fst` package](https://lucene.apache.org/core/4_0_0/core/org/apache/lucene/util/fst/package-summary.html) — official FST package documentation; references the Mihov/Maurel construction algorithm
- [Elastic — Frame of Reference and Roaring Bitmaps](https://www.elastic.co/blog/frame-of-reference-and-roaring-bitmaps) — describes the postings compression strategy walked through in §5
- [Mike McCandless — Lucene performance with the PForDelta codec](https://blog.mikemccandless.com/2010/08/lucene-performance-with-pfordelta-codec.html) — PFOR-delta in Lucene postings, with measurements
- [Elasticsearch Reference — Similarity (BM25 default)](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-similarity.html) — confirms BM25 default since Elasticsearch 5.0; parameters `k1=1.2`, `b=0.75` referenced for the next doc in this tier
- [Elastic — BM25 vs Lucene Default Similarity](https://www.elastic.co/blog/found-bm-vs-lucene-default-similarity) — context for the TF-IDF → BM25 transition referenced in the summary
