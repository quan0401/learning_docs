# Search Documentation Index — Learning Path

A progressive path through full-text and vector search engineering for backend developers. Anchored on Lucene-family engines (Elasticsearch, OpenSearch) with cross-references to PostgreSQL `tsvector` and modern vector retrieval. Practical and protocol-grounded — covers the inverted index, scoring, query DSL, indexing strategy, sharding/replication, and the production patterns that keep search fast and correct.

Cross-references to the [Database learning path](../database/INDEX.md) (PostgreSQL full-text, JSONB indexing — when SQL is enough), the [System Design learning path](../system-design/INDEX.md) (search as a system-design building block), the [Performance learning path](../performance/INDEX.md) (latency tails on shard fan-out), the [Observability learning path](../observability/INDEX.md) (cluster health, slow-query logs), the [Java learning path](../java/INDEX.md) (Spring Data Elasticsearch, OpenSearch Java client), the [TypeScript learning path](../typescript/INDEX.md) (Node clients, query builders), the [Web Scraping learning path](../web-scraping/INDEX.md) (corpora that often land in a search index), and the [Security learning path](../security/INDEX.md) (query injection in DSL bodies, document-level security) where topics overlap.

**Markers:** **★** = core must-learn (everyday search engineering — what every backend dev shipping a search bar should know). **○** = supporting deep-dive (specialized tooling, cluster ops, niche scoring techniques). Internalize all ★ before going deep on ○.

---

## Tier 1 — Foundations: Inverted Index and Scoring

The mental model. Why search engines exist as a separate system from your relational database, and the data structure that makes them fast.

1. [★ Inverted Index Internals — Postings Lists, Skip Lists, and FSTs](foundations/01-inverted-index-internals.md) — terms dictionary as FST (Lucene's `.tip`/`.tim`), postings files (`.doc`/`.pos`/`.pay`), delta + Frame-of-Reference + PFOR compression, skip-list overlay for `advance()`, immutable segment model, the full Lucene file family, latency rules of thumb _(2026-05-07)_
2. [★ BM25 and TF-IDF — Relevance Scoring from First Principles](foundations/02-bm25-and-tf-idf.md) — TF-IDF mechanics, length normalization, BM25 saturation (`k1=1.2`) and length pivot (`b=0.75`), Lucene `.nvd`/`.nvm` norms storage, ES 5.0 default switch, per-field similarity overrides, stop-word and `bool.filter` pitfalls _(2026-05-07)_
3. [★ Analyzers — Tokenization, Filters, and Language-Specific Pipelines](foundations/03-analyzers.md) — char filters → tokenizer → token filters, per-field index-vs-search analyzer asymmetry, language analyzers, CJK plugins (`analysis-kuromoji`, `analysis-smartcn`, `analysis-icu`), `_analyze` API, `synonym_graph` since Lucene 6.4 / ES 5.2 _(2026-05-07)_
4. [★ Lucene Segments and Merge Policy](foundations/04-segments-and-merge-policy.md) — buffer → refresh → flush → durability lifecycle, translog knobs, refresh interval tuning, TieredMergePolicy defaults (segmentsPerTier=10, maxMergedSegmentMB=5GB, deletesPctAllowed=20), force-merge guardrails, `.liv` tombstones, `?refresh=wait_for` patterns _(2026-05-07)_

---

## Tier 2 — Query DSL and Aggregations

Match, term, bool, and the difference. Aggregations as the underrated half of the API.

5. [★ Query DSL — match, term, bool, range, prefix, wildcard, regex](query-dsl/05-query-dsl.md) — query vs filter context, term-level + full-text leaves, `bool`/`dis_max`/`function_score` compounds, joining queries, PIT + `search_after` pagination, `max_result_window=10000` default, scroll deprecation, seven daily pitfalls _(2026-05-07)_
6. [★ Aggregations — bucket, metric, pipeline; cardinality estimation tradeoffs](query-dsl/06-aggregations.md) — three families, doc_values vs fielddata trap, HyperLogLog++ cardinality (`precision_threshold` default 3000, max 40000), shard top-N over-fetch, t-digest vs HDR percentiles, `search.max_buckets=65536`, `top_hits` and `composite` patterns _(2026-05-07)_
7. [★ Highlighting and Suggesters — completion, phrase, term suggesters](query-dsl/07-highlighting-and-suggesters.md) — unified/plain/fvh highlighters with term-vector and offset requirements, analyzer-mismatch bug, all four suggester types, FST-backed completion field heap cost, did-you-mean vs autocomplete UX _(2026-05-07)_
8. [○ Search Templates and Stored Queries](query-dsl/08-search-templates.md) — Mustache parameterization, store/run/render/msearch APIs, stored Painless scripts, parameterization-as-injection-defense (cross-link to security/web-attacks/sql-injection-deep-dive), version-suffix + indirection-table pattern for blue/green template swaps _(2026-05-07)_

---

## Tier 3 — Indexing Strategy

Mappings, analyzers, ingest pipelines, and reindexing without downtime.

9. [★ Mapping Design — dynamic vs strict, multi-fields, denormalization](indexing/09-mapping-design.md) — `dynamic` modes, datatype shortlist, multi-fields, denormalize→nested→join preference, array-of-objects flattening gotcha, `_source` trimming, `doc_values` vs `fielddata`, `index_options`/`norms` storage knobs, runtime fields _(2026-05-07)_
10. [★ Ingest Pipelines and Reindex API](indexing/10-ingest-pipelines-and-reindex.md) — processors, default/final pipeline binding, two-level `on_failure`, Simulate API, Reindex (local/remote/scripted), throughput tuning (slices, requests_per_second, batch size), update/delete by query _(2026-05-07)_
11. [★ Index Lifecycle Management — hot/warm/cold/frozen tiers](indexing/11-ilm-and-tiers.md) — data tiers and node roles, ILM phases and actions, rollover and `is_write_index`, searchable snapshots as the frozen-tier mechanism, OpenSearch ISM equivalent, log/metrics retention patterns _(2026-05-07)_
12. [★ Aliases and Zero-Downtime Reindexing](indexing/12-aliases-and-zero-downtime-reindexing.md) — read vs write aliases, `is_write_index`, atomic `_aliases` actions, blue/green reindex pattern, catch-up indexing with `version_type: external`, six production traps (in-flight bulks, replication lag, version conflicts, …) _(2026-05-07)_

---

## Tier 4 — Sharding, Replication, and Cluster Ops

The distributed-systems half. Shard sizing, allocation, hot spots, split-brain history.

13. [★ Sharding and Replication Mechanics](cluster-ops/13-sharding-and-replication.md) — primary vs replica roles, current 10-50 GB / ~200 GB shard guidance, default `_id`-hash routing and custom routing, allocation awareness, hot-spot causes, primary-then-parallel-replicas write path, adaptive replica selection, OpenSearch 2.x segment replication, `index.sort.field` _(2026-05-07)_
14. [★ Cluster State, Master Election, and the Coordinator Path](cluster-ops/14-cluster-state-and-coordination.md) — master/cluster-manager role, cluster state propagation, post-7.0 coordination subsystem, voting configuration and quorum, `cluster.initial_master_nodes` bootstrap data-loss warning, query→reduce→fetch coordinator phase, fan-out cost, three caches _(2026-05-07)_
15. [○ Cross-Cluster Search and Replication](cluster-ops/15-cross-cluster-search-and-replication.md) — sniff vs proxy CCS connection modes, `<alias>:<index>` syntax, `skip_unavailable`, `ccs_minimize_roundtrips`, Elastic CCR (Enterprise tier) vs OpenSearch's Apache-2.0 replication plugin, async-replication RPO semantics _(2026-05-07)_
16. [★ Snapshot and Restore — Repositories, S3, and Recovery](cluster-ops/16-snapshot-and-restore.md) — repository abstraction (fs/S3/GCS/Azure), segment-level incremental mechanics, snapshot consistency and refresh semantics, SLM scheduling, restore variants, searchable snapshots, NFS/Glacier/version-window traps _(2026-05-07)_

---

## Tier 5 — Vector Search and Hybrid Retrieval

ANN search with HNSW, dense vectors alongside lexical, and rerankers.

17. [★ Vector Search Fundamentals — HNSW, IVF, dimensions, distance metrics](vector/17-vector-search-fundamentals.md) — distance metrics (cosine/dot/L2), HNSW multi-layer graph (`M=16`, `ef_construction=100` Lucene defaults), IVF clustering alternative, scalar/product/binary quantization, dimension-vs-memory math, ES `dense_vector` + `int8_hnsw`, OpenSearch k-NN engines (lucene/faiss/nmslib) _(2026-05-07)_
18. [★ Hybrid Search — BM25 + Dense, Reciprocal Rank Fusion](vector/18-hybrid-search.md) — BM25/dense complementarity, RRF (`k=60`) vs convex combination after min-max normalization, ES `retriever.rrf` API, OpenSearch `hybrid` query + normalization-processor, when to hybrid and when NOT to vector, NDCG/MRR evaluation _(2026-05-07)_
19. [★ Embedding Pipelines — Where to Generate, Where to Store](vector/19-embedding-pipelines.md) — index-time/query-time/in-cluster decision, model versioning via alias double-write reindex, dimensionality cost, batching against vendor APIs, OpenAI/Cohere/SBERT/TEI landscape, recall-killing traps (prompt prefix mismatch, silent token truncation, chunking) _(2026-05-07)_
20. [○ Rerankers and Cross-Encoders — Cost vs Quality](vector/20-rerankers-and-cross-encoders.md) — bi-encoder vs cross-encoder, two-stage retrieval pipeline, hosted (Cohere Rerank, Voyage, Jina) vs open (BGE, sentence-transformers MS MARCO) reranker landscape, Elastic Inference + OpenSearch ML Commons integration, ten production traps _(2026-05-07)_

---

## Tier 6 — Production Patterns

What goes wrong at scale and how to keep it from waking you up.

21. [★ Slow-Query Logs, Profile API, and Hot-Threads](production/21-slow-logs-and-profile-api.md) — per-shard slow log thresholds (default `-1` disabled), Profile API field semantics (`next_doc`/`advance`/`score`/`build_scorer`), `_nodes/hot_threads` stack patterns, `_tasks` cancellation, six expensive query patterns, cache stats _(2026-05-07)_
22. [★ Capacity Planning — Heap, FS Cache, Shard Count](production/22-capacity-planning.md) — heap-vs-page-cache two-pool model, 50%/~31 GB heap rule with the compressed-oops cliff, 20-shards-per-GB-heap and 10-50 GB (200 GB time-series) shard targets, vertical-vs-horizontal tradeoffs, hot/warm/cold sizing, ten production traps _(2026-05-07)_
23. [★ Multi-Tenancy — Document-Level Security and Routing](production/23-multi-tenancy.md) — three models (index-per-tenant, single-index-with-filter, single-index-with-routing), filtered aliases, DLS/FLS license callouts (Elastic Platinum vs OpenSearch Apache-2.0), aggregation isolation, hot-tenant carve-out, routing skew traps _(2026-05-07)_
24. [★ PostgreSQL Full-Text vs Elasticsearch — When to Stay in SQL](production/24-postgres-fts-vs-elasticsearch.md) — `tsvector`/`tsquery`/`@@`, GIN-vs-GiST, generated-column indexing, the `ts_rank` BM25 gap, `pg_trgm` fuzzy fallback, PG-as-source-of-record + ES-via-Debezium polyglot, pgvector for hybrid in PG _(2026-05-07)_

---

## Quick Reference by Topic

### Foundations

- [Inverted Index Internals](foundations/01-inverted-index-internals.md) _(2026-05-07)_
- [BM25 and TF-IDF](foundations/02-bm25-and-tf-idf.md) _(2026-05-07)_
- [Analyzers](foundations/03-analyzers.md) _(2026-05-07)_
- [Lucene Segments and Merge Policy](foundations/04-segments-and-merge-policy.md) _(2026-05-07)_

### Query DSL

- [Query DSL](query-dsl/05-query-dsl.md) _(2026-05-07)_
- [Aggregations](query-dsl/06-aggregations.md) _(2026-05-07)_
- [Highlighting and Suggesters](query-dsl/07-highlighting-and-suggesters.md) _(2026-05-07)_
- [Search Templates](query-dsl/08-search-templates.md) _(2026-05-07)_

### Indexing

- [Mapping Design](indexing/09-mapping-design.md) _(2026-05-07)_
- [Ingest Pipelines and Reindex](indexing/10-ingest-pipelines-and-reindex.md) _(2026-05-07)_
- [Index Lifecycle Management](indexing/11-ilm-and-tiers.md) _(2026-05-07)_
- [Aliases and Zero-Downtime Reindexing](indexing/12-aliases-and-zero-downtime-reindexing.md) _(2026-05-07)_

### Cluster Ops

- [Sharding and Replication](cluster-ops/13-sharding-and-replication.md) _(2026-05-07)_
- [Cluster State and Coordination](cluster-ops/14-cluster-state-and-coordination.md) _(2026-05-07)_
- [Cross-Cluster Search and Replication](cluster-ops/15-cross-cluster-search-and-replication.md) _(2026-05-07)_
- [Snapshot and Restore](cluster-ops/16-snapshot-and-restore.md) _(2026-05-07)_

### Vector Search

- [Vector Search Fundamentals](vector/17-vector-search-fundamentals.md) _(2026-05-07)_
- [Hybrid Search](vector/18-hybrid-search.md) _(2026-05-07)_
- [Embedding Pipelines](vector/19-embedding-pipelines.md) _(2026-05-07)_
- [Rerankers and Cross-Encoders](vector/20-rerankers-and-cross-encoders.md) _(2026-05-07)_

### Production

- [Slow-Query Logs and Profile API](production/21-slow-logs-and-profile-api.md) _(2026-05-07)_
- [Capacity Planning](production/22-capacity-planning.md) _(2026-05-07)_
- [Multi-Tenancy](production/23-multi-tenancy.md) _(2026-05-07)_
- [Postgres FTS vs Elasticsearch](production/24-postgres-fts-vs-elasticsearch.md) _(2026-05-07)_

---

## Bug Spotting

Active-recall practice docs will land here once the tiers above are populated. Pattern matches the rest of the repo: 22+ broken snippets organized by difficulty (Easy / Subtle / Senior trap), one-line `<details>` hints inline, full root-cause + fix in a Solutions section. Every bug cites a real reference (CVE, official docs, postmortem, library issue).

---

## Related Learning Paths

- [Database — PostgreSQL `tsvector`, JSONB indexing](../database/INDEX.md) — when SQL is enough and you don't need a separate engine
- [System Design — search as a building block](../system-design/INDEX.md) — fan-out, dedup, freshness SLOs
- [Performance — Little's Law, p99 tails](../performance/INDEX.md) — shard fan-out is a tail-latency multiplier
- [Observability — slow-query logs, RED metrics](../observability/INDEX.md) — how you notice a sick cluster
- [Web Scraping — corpora to index](../web-scraping/INDEX.md) — what you often end up loading into search
