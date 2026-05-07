---
title: "Analyzers — Tokenization, Filters, and Language Pipelines"
date: 2026-05-07
updated: 2026-05-07
tags: [search, analyzers, tokenization, lucene, elasticsearch, opensearch]
---

# Analyzers — Tokenization, Filters, and Language Pipelines

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `analyzers` `tokenization` `lucene` `elasticsearch` `opensearch`

---

## Table of Contents

- [Summary](#summary)
- [1. The Analyzer Pipeline](#1-the-analyzer-pipeline)
- [2. Character Filters](#2-character-filters)
- [3. Tokenizers](#3-tokenizers)
- [4. Token Filters](#4-token-filters)
- [5. Per-Field Analyzers and the Asymmetric Index/Search Pattern](#5-per-field-analyzers-and-the-asymmetric-indexsearch-pattern)
- [6. Built-in Language Analyzers](#6-built-in-language-analyzers)
- [7. CJK — Why the Defaults Are Wrong, and What to Use](#7-cjk--why-the-defaults-are-wrong-and-what-to-use)
- [8. Defining a Custom Analyzer](#8-defining-a-custom-analyzer)
- [9. Testing Analyzers with `_analyze`](#9-testing-analyzers-with-_analyze)
- [10. Practical Traps](#10-practical-traps)
- [Related](#related)
- [References](#references)

---

## Summary

The previous doc in this tier described the inverted index as a terms dictionary plus postings lists. This doc describes the stage that decides *what becomes a term in the first place*: the **analyzer**, a deterministic pipeline that turns a raw input string into the sequence of normalized tokens written into the dictionary. Analyzers run on every indexed value and on every full-text query string, and a mismatch between the two — different stemmer, missing lowercasing, an asymmetric synonym filter on the wrong side — is the most common reason a query "obviously" matches a document and doesn't.

An analyzer is always three stages in this fixed order: zero or more **character filters** that rewrite the raw byte stream, exactly one **tokenizer** that chops the stream into tokens, and zero or more **token filters** that normalize, drop, expand, or replace tokens. Elasticsearch and OpenSearch both wrap Lucene's `Analyzer` API, so the parts and names below are Lucene's; the JSON mapping syntax is Elastic's. For a TypeScript/Node engineer used to thinking of `String.prototype.toLowerCase().split(/\s+/)` as "tokenization," the right mental upgrade is: replace that one line with a 3–6-stage configurable pipeline whose output you must commit to disk and re-run on every query for the rest of the index's life.

---

## 1. The Analyzer Pipeline

Every Elasticsearch/OpenSearch analyzer has the shape:

```text
   raw string
       │
       ▼
┌───────────────────┐
│  char_filter[ ]   │   0..N stages, run in order, byte-stream → byte-stream
└───────────────────┘
       │
       ▼
┌───────────────────┐
│   tokenizer       │   exactly 1 stage, byte-stream → token-stream
└───────────────────┘
       │
       ▼
┌───────────────────┐
│  filter[ ]        │   0..N stages, run in order, token-stream → token-stream
└───────────────────┘
       │
       ▼
   tokens written to the inverted index
```

The default analyzer if you set nothing is `standard`: no char filters, the `standard` tokenizer (Unicode segmentation), and the `lowercase` token filter. The English string `"The quick brown fox."` becomes `["the", "quick", "brown", "fox"]`. The same string under `english` (a built-in language analyzer) becomes `["quick", "brown", "fox"]` — the article `the` is dropped by the English stop filter, and a stemmer would fold `running` to `run` if it were present.

The pipeline runs on **both** sides of search: when a document is indexed, every analyzed-text field's value flows through the field's mapped analyzer and the resulting tokens go into the dictionary. When a `match` query arrives, the query string flows through the field's *search* analyzer and the resulting tokens are looked up in the dictionary. Token equality is the matching primitive — there is no fuzzy magic around it (other than what filters in the pipeline produce).

---

## 2. Character Filters

Char filters mutate the raw text *before* tokenization. Three are built in.

- **`html_strip`** — strips HTML tags, decodes entities. `"<p>Caf&eacute;</p>"` → `"\nCafé\n"`. Use on fields that store HTML-bearing content (CMS bodies, scraped pages) so that `<` and `>` don't end up as token boundaries and so that entities decode to the characters a stemmer can actually fold.
- **`mapping`** — applies a literal substitution table. Useful for things like `"&" → " and "`, `"©" → ""`, or normalizing emoticons before a tokenizer chops them. Substitutions are byte-level and run before any Unicode-aware step.
- **`pattern_replace`** — applies a Java regex replacement. Common use is collapsing repeated whitespace, or stripping URL fragments before tokenization. Keep the regex cheap; this runs on every indexed character.

Char filters cannot see token boundaries — they only operate on the raw stream — which is why they're for "I want this character to be different before any tokenizer ever sees it."

---

## 3. Tokenizers

The tokenizer is the only required stage and the most consequential single choice in the pipeline. It decides what a "term" even is.

| Tokenizer | What it yields for `"the-quick.brown_fox/2"` | When to use |
|-----------|---------------------------------------------|-------------|
| `standard` | `["the", "quick.brown_fox", "2"]` (UAX #29 word boundaries; behaviour around `.` and `_` depends on Unicode rules) | Default for most prose, multilingual text |
| `whitespace` | `["the-quick.brown_fox/2"]` | When you need to control the rest of the pipeline yourself and only split on spaces |
| `keyword` | `["the-quick.brown_fox/2"]` (one token containing the whole input) | When the entire field value is the term — IDs, tags, enum values |
| `pattern` | depends on regex; default `\W+` produces `["the", "quick", "brown_fox", "2"]` | Custom delimiter rules where `standard` is wrong |
| `ngram` (min=2, max=3) | `["th", "the", "he-", ...]` | Substring search, autocomplete on arbitrary infixes (cost: index size explodes) |
| `edge_ngram` (min=2, max=10) | `["th", "the", "the-", "the-q", ...]` | Prefix-only autocomplete (cheaper than `ngram`) |
| `path_hierarchy` for `"/usr/local/bin"` | `["/usr", "/usr/local", "/usr/local/bin"]` | File paths, taxonomy paths — query `/usr/local` matches everything below it |

The `standard` tokenizer implements the Unicode Standard Annex #29 word-segmentation algorithm, which is why it does the right thing across most European scripts and partly the right thing across CJK (more on that in §7).

For autocomplete, `edge_ngram` at index time + a non-tokenizing search analyzer (`keyword` tokenizer + lowercase) is the canonical recipe: index `"kafka"` as `["k", "ka", "kaf", "kafk", "kafka"]`, then a query for `"kaf"` (analyzed as the single token `"kaf"`) hits the postings for `"kaf"` directly. This is the asymmetric pattern from §5.

---

## 4. Token Filters

Once the tokenizer has produced a token stream, token filters reshape it. The set is large; the indispensable ones are:

- **`lowercase`** — normalizes case. Almost always present; without it `"Kafka"` and `"kafka"` are two different terms.
- **`asciifolding`** — folds Unicode characters to ASCII equivalents (`café` → `cafe`, `naïve` → `naive`). Cheap and dramatically improves recall for accent-insensitive search across European languages. Pair with `preserve_original: true` if you also want exact-accent matches to score higher.
- **`stop`** — removes a configured stopword list (`the`, `a`, `is`, …). Saves index space and eliminates noise terms; the cost is that phrase queries containing stopwords need care because the gap is preserved as a position increment.
- **`stemmer`** — folds inflected forms to a stem. The Lucene `english` stemmer (Porter2) maps `running → run`, `ran → ran`, `cats → cat`. Choice of stemmer is per-language; aggressive stemmers (`english_porter`) over-conflate, light stemmers (`light_english`) under-conflate. There is no neutral choice; pick by measuring against your corpus.
- **`synonym`** / **`synonym_graph`** — expands tokens into synonyms. The graph variant (added with Lucene 6.4 in Elasticsearch 5.2) handles **multi-word synonyms** correctly by emitting a token graph rather than a flat sequence; the older `synonym` filter mishandles `"new york" → "ny"`-style expansions in queries. Use `synonym_graph` at search time only.
- **`shingle`** — produces n-gram-of-tokens. `["the", "quick", "brown"]` with `min=2, max=2` becomes `["the quick", "quick brown"]`. Useful for phrase-prefix scoring and "did you mean" features.
- **`trim`** — strips leading/trailing whitespace from each token. Mostly relevant after a `pattern` tokenizer that left whitespace intact.

Filters run in the declared order, and order matters. `lowercase` before `stop` lets a lowercase stopword list match capitalized stopwords in input; `lowercase` after `stop` requires the stopword list itself to enumerate every case variant. The conventional order in custom analyzers is: `lowercase → asciifolding → stop → stemmer → synonym_graph` (search-side).

---

## 5. Per-Field Analyzers and the Asymmetric Index/Search Pattern

Every `text` field in a mapping has up to two analyzers:

- **`analyzer`** — used at *both* index time and search time unless an explicit `search_analyzer` is set.
- **`search_analyzer`** — used at search time *instead of* `analyzer`, when set. Index-time analysis is unaffected.

The asymmetric pattern most commonly appears in three places:

1. **Autocomplete with edge n-grams.** Index analyzer applies `edge_ngram` so each prefix becomes a term in the dictionary. Search analyzer does not — it just lowercases the query string. If you analyzed the query the same way, `"kaf"` would be tokenized into `["k", "ka", "kaf"]` and you'd get a noisy multi-token match instead of one direct lookup.

2. **Synonyms at search time only.** Index a base form (`television`) and expand at search time (`tv → television, telly`). This avoids re-indexing every time the synonym list changes and keeps the index smaller. `synonym_graph` is designed specifically as a search-only filter for this pattern.

3. **Aggressive stemming asymmetry.** Some teams index with a light stemmer to preserve discriminative tokens, then apply a heavier stemmer at search time; this is rare and usually a sign that the team should split the field into two sub-fields with different analyzers (`text` + `text.stemmed`) instead.

The contract you must hold: **for every term the index analyzer emits, the search analyzer must emit the same surface form** when the query contains the corresponding input. Break this and queries silently miss documents.

---

## 6. Built-in Language Analyzers

Elasticsearch ships per-language analyzers — `english`, `french`, `german`, `spanish`, `italian`, `dutch`, `portuguese`, `russian`, `arabic`, `turkish`, and many others. Each is a pre-composed pipeline of `standard` tokenizer + lowercase + language-specific stopword list + language-specific stemmer + (sometimes) language-specific normalization (e.g. `german_normalization` for ß/ss).

For example the `english` analyzer is roughly:

```text
standard tokenizer
  → english_possessive_stemmer  (drops trailing 's)
  → lowercase
  → english_stop                 (the, a, is, ...)
  → porter_stem                  (running → run)
```

The stemmer choice is the variable that matters most. Lucene packages several English stemmers — Porter (the original), Porter2 (a.k.a. `english`, the modern default), Lovins, KStem, and `light_english`. Porter2 is the safe default; KStem is less aggressive and a better choice for product/brand text where over-stemming destroys recall (e.g. `Singer` → `sing`).

Do not chain multiple language analyzers expecting them to compose. They each include a tokenizer; the second tokenizer would never run because Lucene only allows one. If you need multilingual handling on a single field, use one analyzer with a tokenizer that does the right thing across languages (`standard` or `icu_tokenizer`) and skip language-specific stemming, *or* index the same field multiple times under different analyzers and route queries by detected language.

---

## 7. CJK — Why the Defaults Are Wrong, and What to Use

The `standard` tokenizer's UAX #29 implementation handles Chinese and Japanese conservatively: each CJK character becomes its own token. This is technically searchable but recall suffers because most meaningful terms are multi-character compounds. Three plugin-supplied tokenizers are the practical answer:

- **`icu_tokenizer`** (from the `analysis-icu` plugin) — uses ICU's dictionary-based segmentation, which segments Chinese, Japanese, Korean, Thai, and Lao into word-shaped tokens rather than per-character. Often "good enough" for mixed-language corpora.
- **`kuromoji_tokenizer`** (from the `analysis-kuromoji` plugin) — the Lucene Kuromoji morphological analyzer for Japanese, backed by the MeCab-IPADIC dictionary. Handles inflection, part-of-speech tagging, and produces base forms suitable for indexing. The standard choice for Japanese-heavy text.
- **`smartcn_tokenizer`** (from the `analysis-smartcn` plugin) — a Hidden-Markov-Model-based Chinese segmenter from the Lucene `smartcn` module. Pair with the bundled `smartcn_stop` filter.

All three are official Elastic plugins installed via `bin/elasticsearch-plugin install <name>`. There is no built-in Chinese or Japanese analyzer in core Elasticsearch — if a CJK corpus matters, the plugin install is mandatory and should be part of the cluster bootstrap, not an afterthought.

For Korean, the equivalent is the `nori` plugin (Lucene's Korean morphological analyzer). For Arabic, the built-in `arabic` analyzer is usually adequate.

---

## 8. Defining a Custom Analyzer

A custom analyzer is declared in the index `settings.analysis` block and referenced by name from a field mapping. Example: a search-time analyzer for a product-name field that strips HTML, lowercases, folds accents, and applies a search-only multi-word synonym list.

```json
PUT /products
{
  "settings": {
    "analysis": {
      "char_filter": {
        "html_clean": { "type": "html_strip" }
      },
      "filter": {
        "product_synonyms": {
          "type": "synonym_graph",
          "synonyms": [
            "tv, television, telly",
            "laptop, notebook computer"
          ]
        }
      },
      "analyzer": {
        "product_index": {
          "type": "custom",
          "char_filter": ["html_clean"],
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding"]
        },
        "product_search": {
          "type": "custom",
          "char_filter": ["html_clean"],
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding", "product_synonyms"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "product_index",
        "search_analyzer": "product_search",
        "fields": {
          "raw": { "type": "keyword" }
        }
      }
    }
  }
}
```

Two things worth noting in this example. First, the `name.raw` sub-field with `type: keyword` gives you exact-match and aggregation paths into the same value without re-indexing — a near-universal pattern for `text` fields. Second, the synonym filter only appears in `product_search`, not `product_index`; this is the §5 asymmetric pattern, and the synonym filter type is `synonym_graph` not `synonym` because the entries include multi-word substitutions.

---

## 9. Testing Analyzers with `_analyze`

The `_analyze` API runs an analyzer on an arbitrary string and returns the token stream — the single most useful debugging tool for "why didn't this query match?":

```bash
# Built-in analyzer, no index needed
POST /_analyze
{
  "analyzer": "english",
  "text": "The Running Foxes"
}
# → ["run", "fox"]   (the dropped, "running"/"foxes" stemmed and folded)

# Per-index analyzer with tokenizer + filter chain spelled out
POST /products/_analyze
{
  "tokenizer": "standard",
  "filter": ["lowercase", "asciifolding"],
  "text": "Café Au Lait"
}
# → ["cafe", "au", "lait"]

# Named custom analyzer from a real index
POST /products/_analyze
{
  "analyzer": "product_search",
  "text": "TV under $500"
}
# → ["tv", "television", "telly", "under", "500"]   (synonyms expanded)
```

When you have a "ghost miss" — a query that should match and doesn't — run `_analyze` against the indexed field's `analyzer` with the document's value, then against the field's `search_analyzer` with the query string. The two token streams must overlap on at least one term. If they don't, the bug is in the analyzer config, not in the query DSL.

---

## 10. Practical Traps

The traps below are the ones that actually break production indexes; they are worth a re-read before shipping any analyzer change.

- **`synonym` vs `synonym_graph` for multi-word synonyms.** The legacy `synonym` filter emits multi-word synonyms as a flat token sequence with overlapping positions, which silently breaks phrase queries and `match` query position-aware logic. Use `synonym_graph`, and only at search time. The graph variant has been the right answer since Elasticsearch 5.2 (Lucene 6.4).
- **Synonyms in the index analyzer.** Putting any synonym filter on the index side means re-indexing every time the synonym list changes. Keep synonyms in the search analyzer unless there is an explicit reason (typically: scoring quality on a frozen vocabulary).
- **Stemmer over-aggressiveness.** Porter on English maps `Singer → sing`, `university → univers`, `news → new`. For brand-name and proper-noun-heavy fields, prefer `light_english` or KStem, or store the field twice (once analyzed, once `keyword`) and boost the keyword side.
- **Forgetting the `keyword` sub-field.** A pure `text` field cannot be sorted, aggregated, or term-queried for exact match. The `fields: { raw: { type: keyword } }` pattern is so common it should be a reflex. Without it you end up re-indexing data when a product manager asks for a faceted filter.
- **Char filters that change byte length and break highlighters.** `html_strip` and `mapping` change source offsets. Most highlighters cope, but some custom highlighting code that does its own offset math will misalign. Test highlighting after introducing char filters.
- **Per-character CJK tokens.** `standard` on Chinese gives technically correct but recall-poor results. Install `analysis-icu`, `analysis-kuromoji`, or `analysis-smartcn` *before* indexing — switching tokenizers later requires a full reindex.
- **Asymmetric edge n-grams without a `search_analyzer`.** If you put `edge_ngram` in `analyzer` and forget `search_analyzer`, the query string also gets edge-n-grammed, which both balloons the term set the query has to OR together and produces ranking artifacts (long queries match more aggressively). Always pair index-time `edge_ngram` with a non-grammed search analyzer.

---

## Related

- [Inverted Index Internals — Postings Lists, Skip Lists, and FSTs](./01-inverted-index-internals.md) — what the analyzer's output actually becomes on disk
- _BM25 and TF-IDF — Relevance Scoring from First Principles (planned)_ — token frequency from analyzed text feeds the scoring layer
- _Mapping and Field Types — text vs keyword vs numeric (planned)_ — the `text` + `keyword` sub-field pattern referenced in §8 and §10
- _Search-Time Query DSL — match, term, phrase (planned)_ — `match` runs the search analyzer; `term` does not, which is its own trap class
- [Database — Full-Text Search in PostgreSQL](../../database/INDEX.md) — analyzer-equivalent in Postgres is the `tsvector`/`tsquery` configuration; same problem, different vocabulary

## References

- [Elasticsearch Reference — Anatomy of an analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analyzer-anatomy.html) — the official three-stage pipeline description used in §1
- [Elasticsearch Reference — Tokenizer reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-tokenizers.html) — full list and semantics of built-in tokenizers
- [Elasticsearch Reference — Token filter reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-tokenfilters.html) — full list and semantics of built-in token filters
- [Elasticsearch Reference — Character filter reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-charfilters.html) — `html_strip`, `mapping`, `pattern_replace`
- [Elasticsearch Reference — Language analyzers](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html) — composition of `english`, `french`, etc.
- [Elasticsearch Reference — Synonym graph token filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-synonym-graph-tokenfilter.html) — graph filter for multi-word synonyms; states "designed to be used as part of a search analyzer only"
- [Elastic Blog — Multi-Token Synonyms and Graph Queries in Elasticsearch](https://www.elastic.co/blog/multitoken-synonyms-and-graph-queries-in-elasticsearch) — background on why the graph variant exists
- [Elasticsearch Plugins — Japanese (kuromoji) Analysis Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-kuromoji.html) — install + tokenizer + per-POS filters
- [Elasticsearch Plugins — Smart Chinese (smartcn) Analysis Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-smartcn.html) — `smartcn` analyzer / tokenizer / stop filter
- [Elasticsearch Plugins — ICU Analysis Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-icu.html) — `icu_tokenizer`, `icu_normalizer`, `icu_folding`
- [Elasticsearch Reference — Test an analyzer (`_analyze` API)](https://www.elastic.co/guide/en/elasticsearch/reference/current/test-analyzer.html) — endpoint used in §9
- [Apache Lucene 9.x — `org.apache.lucene.analysis` package](https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/analysis/package-summary.html) — upstream Analyzer API that Elasticsearch and OpenSearch wrap
- [OpenSearch Documentation — Analyzers](https://opensearch.org/docs/latest/analyzers/) — equivalent reference for the OpenSearch fork; semantics and names match Elasticsearch for the components covered here
- [Unicode Standard Annex #29 — Unicode Text Segmentation](https://unicode.org/reports/tr29/) — the word-segmentation algorithm implemented by the `standard` tokenizer
