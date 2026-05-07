---
title: "PostgreSQL Full-Text Search vs Elasticsearch — When Staying in SQL Is the Right Call"
date: 2026-05-07
updated: 2026-05-07
tags: [search, postgresql, fts, elasticsearch, tsvector, polyglot]
---

# PostgreSQL Full-Text Search vs Elasticsearch — When Staying in SQL Is the Right Call

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `postgresql` `fts` `elasticsearch` `tsvector` `polyglot`

---

## Table of Contents

- [Summary](#summary)
- [1. The Honest Comparison](#1-the-honest-comparison)
- [2. PostgreSQL FTS Primitives](#2-postgresql-fts-primitives)
- [3. Indexing — GIN, GiST, and Recheck Cost](#3-indexing--gin-gist-and-recheck-cost)
- [4. Stored vs Computed `tsvector`](#4-stored-vs-computed-tsvector)
- [5. Ranking — `ts_rank`, `ts_rank_cd`, and the BM25 Gap](#5-ranking--ts_rank-ts_rank_cd-and-the-bm25-gap)
- [6. Dictionaries, Stemming, and Configurations](#6-dictionaries-stemming-and-configurations)
- [7. Highlighting with `ts_headline`](#7-highlighting-with-ts_headline)
- [8. Phrase, Prefix, AND/OR via `tsquery`](#8-phrase-prefix-andor-via-tsquery)
- [9. Fuzzy and Trigram Similarity with `pg_trgm`](#9-fuzzy-and-trigram-similarity-with-pg_trgm)
- [10. When PG FTS Is Enough](#10-when-pg-fts-is-enough)
- [11. When Elasticsearch Earns Its Keep](#11-when-elasticsearch-earns-its-keep)
- [12. Polyglot Patterns — PG as System of Record + ES as Index](#12-polyglot-patterns--pg-as-system-of-record--es-as-index)
- [13. pgvector — Staying in PG Even for Vector](#13-pgvector--staying-in-pg-even-for-vector)
- [14. Practical Traps](#14-practical-traps)
- [Related](#related)
- [References](#references)

---

## Summary

A separate search engine is one of the most expensive infrastructure decisions a backend team can make: another stateful system to operate, another consistency boundary to reason about, another schema to keep in sync. **PostgreSQL's built-in full-text search is good enough for a surprising fraction of real workloads** — small-to-medium corpora (under ~10M documents), modest write rates, no need for true BM25, no faceted-aggregation latency requirements. When you already operate Postgres, staying in SQL means transactional writes to the search index, JOINs against the same row you're searching, and one fewer thing to page on at 03:00.

This doc walks the PG FTS primitives (`tsvector`, `tsquery`, GIN indexes, `to_tsvector`, ranking functions), names the specific gaps that push teams to Elasticsearch (BM25 quality on head queries, sub-100ms aggregation over 100M+ docs, multi-language stemming pipelines, vector recall at scale), and covers the polyglot pattern of running both with logical-decoding-based CDC. The default move for a TS/Node or Java/Spring Boot service is: **prototype in PG FTS, measure, migrate to ES when a specific gap forces the move** — not before.

---

## 1. The Honest Comparison

The Elasticsearch-by-default reflex comes from blog posts about Wikipedia-scale search, not from the dataset you actually have. A few honest observations:

- A typical SaaS backend's "searchable documents" table is **under 1M rows**. PG FTS with a GIN index answers most queries in single-digit milliseconds at that scale.
- Most product-search UIs care more about **filter + sort + paginate** than about relevance fine-tuning. PG handles `WHERE tenant_id = $1 AND status = 'active' AND tsv @@ plainto_tsquery('english', $2) ORDER BY created_at DESC` in one query plan; ES requires you to either denormalize the filter fields into the index or call out to PG anyway.
- The **operational cost** of Elasticsearch is real: JVM heap tuning, shard/replica capacity planning, snapshot lifecycle, version upgrades that occasionally break mappings. None of that disappears at small scale.
- The **consistency cost** is real too: PG → ES is eventually consistent, and "user wrote a comment, then searched for it, but it's not there yet" is a class of bug you don't have when search lives in the same transaction as the write.

The right framing: Elasticsearch is the right answer when *search is the workload*. PG FTS is the right answer when search is *one feature* of an OLTP application.

---

## 2. PostgreSQL FTS Primitives

PostgreSQL's FTS is built around two types and one operator, all defined in the official `Full Text Search` chapter of the docs:

- **`tsvector`** — a preprocessed document: a sorted, deduplicated list of *lexemes* (normalized terms) with their positions in the original text. Built by passing raw text through a text-search configuration (analyzer + dictionaries).
- **`tsquery`** — a parsed query expression made of lexemes combined with the operators `&` (AND), `|` (OR), `!` (NOT), `<->` (followed-by), and `:*` (prefix).
- **`@@`** — the match operator: `tsvector @@ tsquery` returns `true` when the query matches the document.

Construction functions:

| Function | Purpose |
|----------|---------|
| `to_tsvector(config, text)` | Build a `tsvector` from text, applying the named text-search configuration (e.g. `'english'`). |
| `to_tsquery(config, text)` | Parse a strict tsquery expression — caller must use the operator syntax (`'cat & dog'`). |
| `plainto_tsquery(config, text)` | Treat input as plain words; AND them together. Forgiving for user input. |
| `phraseto_tsquery(config, text)` | Like `plainto_tsquery` but uses `<->` so words must be adjacent. |
| `websearch_to_tsquery(config, text)` | Parses Google-style syntax — quoted phrases, `OR`, leading `-` for negation. The right default for most user-facing search boxes. |

A minimal example end-to-end:

```sql
SELECT to_tsvector('english', 'The quick brown foxes jumped over the lazy dog');
-- 'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2

SELECT to_tsvector('english', 'The quick brown foxes jumped over the lazy dog')
       @@ websearch_to_tsquery('english', 'jumping fox');
-- t  (stemming makes 'jumping' match 'jump' and 'fox' match 'fox')
```

Note `the` and `over` are dropped as stopwords and `foxes`/`jumped` are stemmed. That's the `english` configuration doing its job.

---

## 3. Indexing — GIN, GiST, and Recheck Cost

A `tsvector @@ tsquery` predicate against an unindexed column is a sequential scan: PG re-tokenizes every row's text and tests it. Useful for small tables, fatal at scale.

PostgreSQL ships two index types for `tsvector`, both documented in the GIN and GiST chapters:

| Index | Strengths | Weaknesses |
|-------|-----------|------------|
| **GIN** (Generalized Inverted Index) | Fast lookups (one entry per lexeme pointing at matching rows). The right default for static-ish documents. | Slower to update; larger on disk than GiST in some cases. |
| **GiST** | Smaller, faster to update, can be lossy. | Returns false positives — PG must recheck each candidate row, costing extra I/O. Lookups slower than GIN. |

For full-text search on documents that don't change every second, **GIN is the default**. The PG docs are unusually direct about this, recommending GIN for static data and GiST only when the index needs to be small or write-heavy.

```sql
-- GIN index on a stored tsvector column
CREATE INDEX idx_articles_tsv ON articles USING GIN (tsv);
```

The recheck behavior matters for `ts_rank`: even with GIN, PostgreSQL must read the heap row to compute the rank. So a query that retrieves the top 10 by rank scans the GIN, fetches matching heap rows, computes ranks, and sorts. Pagination over deep result sets forces all matching heap rows to be touched — a known scaling cliff.

---

## 4. Stored vs Computed `tsvector`

You almost never want to compute `to_tsvector(...)` inline in your `WHERE` clause for indexed search — the index is on a *column*, not on an expression evaluated per-query (unless you use an expression index, which works but is less flexible).

Two common patterns to materialize the `tsvector`:

### 4.1 Generated column (preferred, PG 12+)

```sql
CREATE TABLE articles (
  id BIGSERIAL PRIMARY KEY,
  title TEXT NOT NULL,
  body  TEXT NOT NULL,
  tsv   TSVECTOR
        GENERATED ALWAYS AS (
          setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
          setweight(to_tsvector('english', coalesce(body,  '')), 'B')
        ) STORED
);

CREATE INDEX idx_articles_tsv ON articles USING GIN (tsv);
```

`setweight` lets `ts_rank` weigh title hits higher than body hits. The `STORED` generated column is recomputed on insert/update automatically — no triggers, no drift.

### 4.2 Trigger-maintained column (pre-12 or when you need imperative logic)

```sql
CREATE TRIGGER articles_tsv_update BEFORE INSERT OR UPDATE
ON articles FOR EACH ROW
EXECUTE FUNCTION tsvector_update_trigger(tsv, 'pg_catalog.english', title, body);
```

The built-in `tsvector_update_trigger` covers the common case. Custom triggers handle multi-table aggregation (e.g. include comment text in an article's tsvector).

---

## 5. Ranking — `ts_rank`, `ts_rank_cd`, and the BM25 Gap

PostgreSQL provides two ranking functions, defined in the FTS controls section of the manual:

- **`ts_rank(tsvector, tsquery)`** — TF-style rank weighted by lexeme weights (the `setweight` letters above) and configurable normalization for document length.
- **`ts_rank_cd(tsvector, tsquery)`** — *cover density* rank: rewards documents where matched lexemes appear close together, requires positions to be indexed.

Neither is BM25. Elasticsearch and OpenSearch use Lucene's BM25 (with default `k1=1.2`, `b=0.75`) as the default similarity since ES 5.0. The relevance gap between PG's `ts_rank` and BM25 is small for short queries against curated corpora and noticeable for long, ambiguous queries against heterogeneous corpora — exactly the head-query case that drives user satisfaction in a search-first product.

If you need BM25 in Postgres, the `pg_search`/`ParadeDB` extension implements Lucene-style BM25 over `tsvector`-style storage. That's a real option but introduces a new extension to operate; it does not ship with stock Postgres.

```sql
SELECT id, title, ts_rank(tsv, query) AS rank
FROM articles, websearch_to_tsquery('english', 'postgres index tuning') AS query
WHERE tsv @@ query
ORDER BY rank DESC
LIMIT 10;
```

---

## 6. Dictionaries, Stemming, and Configurations

A text-search configuration ties together a parser (which splits text into tokens) and a chain of dictionaries (which normalize tokens to lexemes or drop them). PostgreSQL ships with `simple` (lowercase only, no stemming) and one configuration per supported language (`english`, `french`, `german`, ...) backed by snowball stemmers. Ispell-style dictionaries can be loaded for richer morphology.

Practical notes:

- The `simple` configuration is the right choice for identifiers, SKUs, and codes — anywhere you don't want stemming.
- **There is no built-in language detection.** You pick the configuration at index time (`to_tsvector('english', ...)`). For mixed-language corpora this is a real limitation; ES has language-detection plugins and per-field analyzers that handle the same problem more naturally.
- Stopword lists are configurable per dictionary but changing them requires a reindex.

---

## 7. Highlighting with `ts_headline`

`ts_headline(config, document, query, options)` returns the document text with matching terms wrapped in tags, like an Elasticsearch highlighter:

```sql
SELECT ts_headline(
  'english',
  body,
  websearch_to_tsquery('english', 'gin index'),
  'StartSel=<mark>, StopSel=</mark>, MaxFragments=2, MaxWords=20, MinWords=10'
)
FROM articles
WHERE tsv @@ websearch_to_tsquery('english', 'gin index');
```

`ts_headline` reads the *original* text — not the indexed `tsvector` — so it cannot use the GIN index to narrow text scanning. It's an `O(rows_returned × document_length)` operation; cap result counts before highlighting and avoid running it over very large body fields per page.

---

## 8. Phrase, Prefix, AND/OR via `tsquery`

`tsquery` operator support is closer to ES than people expect:

```sql
-- AND
SELECT to_tsquery('english', 'postgres & index');
-- OR
SELECT to_tsquery('english', 'postgres | mysql');
-- NOT
SELECT to_tsquery('english', 'postgres & !cassandra');
-- Phrase (exactly adjacent)
SELECT to_tsquery('english', 'gin <-> index');
-- Phrase with N-token gap allowed
SELECT to_tsquery('english', 'gin <2> tuning');
-- Prefix
SELECT to_tsquery('english', 'postg:*');
```

`websearch_to_tsquery` parses Google-style syntax into the same operator tree:

```sql
SELECT websearch_to_tsquery('english', '"gin index" tuning -mysql');
-- 'gin' <-> 'index' & 'tune' & !'mysql'
```

---

## 9. Fuzzy and Trigram Similarity with `pg_trgm`

`tsvector` is exact-match-after-stemming; it does not handle typos like `postgres` → `postgers`. The `pg_trgm` contrib extension adds trigram similarity that complements FTS:

```sql
CREATE EXTENSION pg_trgm;

CREATE INDEX idx_articles_title_trgm ON articles USING GIN (title gin_trgm_ops);

-- Find titles within 0.3 trigram distance of the query
SELECT title, similarity(title, 'postgers indxes') AS sim
FROM articles
WHERE title % 'postgers indxes'
ORDER BY sim DESC
LIMIT 10;
```

A common pattern: run `tsvector @@ tsquery` first; if the result set is empty, fall back to a `pg_trgm` similarity query. Or `UNION` both with rank weighting. This combination handles a large fraction of "did-you-mean" cases without leaving Postgres.

---

## 10. When PG FTS Is Enough

Reach for PG FTS when most of these are true:

- **Corpus under ~10M documents** with modest growth.
- **Write rate is low to moderate** (say, fewer than a few hundred indexed updates per second on a single primary).
- **You already operate Postgres** — adding ES doubles your stateful surface area.
- **You need transactional writes**: search index updated in the same transaction as the row.
- **You need SQL JOINs against the search results**: filter by tenant, price range, status, then search.
- **BM25 isn't required**: relevance is "good enough" if the right document is in the top 5 most of the time.
- **No need for sub-100ms faceted aggregations** over the full corpus.

A surprising number of product-search and log-search workloads fit this profile.

---

## 11. When Elasticsearch Earns Its Keep

Move to ES (or OpenSearch) when one or more of these is the binding constraint:

- **Hundreds of millions to billions of documents.** PG FTS performance falls off without aggressive partitioning; Lucene's segment architecture absorbs scale far better.
- **Real BM25 relevance tuning** matters — head queries are the product, you A/B test similarity parameters, and the gap from `ts_rank` is visible in user behavior.
- **Faceted aggregations** over large corpora — count-by-category, price histograms, date histograms, returned in the same request as the hits. ES's doc-values columnar format is built for this.
- **Geo queries** — bounding-box, distance-sorted, polygon-contains. PostGIS exists but combining it with FTS in a single query is awkward and slow.
- **Vector / hybrid search at scale** — ES has native dense-vector kNN; pgvector exists too (see §13) but ES wins above ~50–100M vectors.
- **Multi-language pipelines** with per-language analyzers, per-field analyzers, and synonym graphs.
- **Search is the product**, not a feature, and the team has the operational appetite for a JVM cluster.

---

## 12. Polyglot Patterns — PG as System of Record + ES as Index

The common production shape:

- **Postgres** holds the canonical row.
- **Elasticsearch** holds a denormalized search-optimized projection.
- A **change-data-capture pipeline** keeps ES in sync.

Two CDC approaches:

1. **Logical decoding + Debezium**: Postgres writes to a replication slot, Debezium streams changes to Kafka, a consumer projects each change into an ES document. Works without application changes; lag is typically sub-second under healthy load.
2. **Application-level dual-write**: the service writes to PG and then enqueues an indexing job. Simpler to start with; failure modes (PG commits, ES write fails) require an outbox or retry queue to avoid permanent drift.

Either way you accept eventual consistency between PG and ES, and you must answer two questions explicitly:

- **What's the staleness budget?** "Search reflects writes within 5 seconds, p99" is a measurable SLO.
- **What happens if ES is down?** Common patterns: fall back to a degraded PG FTS query, or return a cached set with a banner.

A nice middle ground: keep PG FTS on the canonical table for the read path that must reflect transactional writes (e.g. "search inside this user's own documents") and use ES only for cross-tenant / global search where eventual consistency is fine.

---

## 13. pgvector — Staying in PG Even for Vector

`pgvector` adds a `vector` column type and approximate-nearest-neighbor indexes (IVFFlat, HNSW) to Postgres. Combined with FTS this becomes a credible "stay in PG" story for hybrid search:

```sql
CREATE EXTENSION vector;

ALTER TABLE articles ADD COLUMN embedding vector(1536);

CREATE INDEX idx_articles_embedding
  ON articles
  USING hnsw (embedding vector_cosine_ops);

-- Hybrid: BM25-ish lexical + vector, combined client-side or via SQL
SELECT id, title,
       1 - (embedding <=> $1) AS sim,
       ts_rank(tsv, websearch_to_tsquery('english', $2)) AS lex_rank
FROM articles
WHERE tsv @@ websearch_to_tsquery('english', $2)
   OR embedding <=> $1 < 0.3
ORDER BY (sim + lex_rank) DESC
LIMIT 20;
```

`pgvector` is the right call up to roughly tens of millions of vectors with HNSW; recall and tail-latency degrade beyond that compared to dedicated vector databases or ES's kNN. For a typical SaaS product with semantic search over its own corpus, pgvector means you don't need a third stateful system at all.

---

## 14. Practical Traps

- **GIN index bloat without `fastupdate` tuning.** GIN supports a "pending list" that batches updates; under heavy write load it can grow until queries take seconds because they scan both the main index and the pending list. Tune `gin_pending_list_limit` and run `gin_clean_pending_list()` if you see this.
- **`ts_rank` is not BM25.** Don't promise relevance parity with ES if you migrate later — the ranking math is genuinely different and head queries will rank differently.
- **No language detection.** Mixed-language corpora pick one configuration and live with the suboptimal stemming on the other languages, or store one tsvector per language and OR them.
- **`ts_headline` doesn't use the index.** Highlight only the page you return.
- **Pagination cost.** `ORDER BY rank LIMIT 10 OFFSET 1000` forces PG to compute rank for at least 1010 matching rows. Use keyset pagination based on a stable secondary sort, or cap depth.
- **Sequential scan on `text @@ to_tsquery` without an indexed column.** Always store the `tsvector` (generated column or trigger) and index it.
- **Cross-database transactions don't exist.** Once you split into PG + ES, you've taken on the consistency boundary forever — design for it on day one rather than retrofitting outboxes later.

---

## Related

- [Inverted Index Internals — Postings Lists, Skip Lists, and FSTs](../foundations/01-inverted-index-internals.md) — the storage shape Lucene uses, contrasted with PG's GIN entries
- _BM25 and TF-IDF — Relevance Scoring from First Principles (planned)_ — the scoring math the PG `ts_rank` gap is measured against
- [Database — Indexes, Storage, and Polyglot Persistence](../../database/INDEX.md) — adjacent path covering Postgres internals, GIN/GiST mechanics, and the polyglot persistence patterns referenced in §12
- _Vector Search and pgvector vs Dedicated Vector Stores (planned)_ — deeper treatment of §13
- _CDC, Outbox, and Keeping Search Indexes in Sync (planned)_ — pattern in §12

## References

- [PostgreSQL — Full Text Search (chapter 12)](https://www.postgresql.org/docs/current/textsearch.html) — authoritative source for `tsvector`, `tsquery`, configurations, and ranking
- [PostgreSQL — Text Search Functions and Operators](https://www.postgresql.org/docs/current/functions-textsearch.html) — `to_tsvector`, `to_tsquery`, `plainto_tsquery`, `phraseto_tsquery`, `websearch_to_tsquery`, `ts_rank`, `ts_rank_cd`, `ts_headline`, `setweight`
- [PostgreSQL — GIN Indexes](https://www.postgresql.org/docs/current/gin.html) — when to choose GIN, pending-list tuning (`gin_pending_list_limit`, `gin_clean_pending_list`)
- [PostgreSQL — GiST Indexes](https://www.postgresql.org/docs/current/gist.html) — lossy index behavior and recheck cost
- [PostgreSQL — Text Search Indexes (12.9)](https://www.postgresql.org/docs/current/textsearch-indexes.html) — explicit GIN-vs-GiST guidance for FTS
- [PostgreSQL — Generated Columns](https://www.postgresql.org/docs/current/ddl-generated-columns.html) — `STORED` generated columns used for the `tsvector`
- [PostgreSQL — `pg_trgm`](https://www.postgresql.org/docs/current/pgtrgm.html) — trigram similarity, `%` operator, `gin_trgm_ops`
- [pgvector README](https://github.com/pgvector/pgvector) — vector type, IVFFlat and HNSW indexes, distance operators
- [Elasticsearch — Index Modules: Similarity (BM25)](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-similarity.html) — BM25 default since ES 5.0; `k1` and `b` parameters
- [Debezium — PostgreSQL Connector](https://debezium.io/documentation/reference/stable/connectors/postgresql.html) — logical-decoding-based CDC referenced in §12
- [ParadeDB / pg_search](https://github.com/paradedb/paradedb) — third-party PG extension implementing BM25 inside Postgres, referenced in §5
