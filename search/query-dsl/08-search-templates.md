---
title: "Search Templates and Stored Queries — Mustache, Parameterization, and Versioning"
date: 2026-05-07
updated: 2026-05-07
tags: [search, elasticsearch, opensearch, mustache, search-templates, security]
---

# Search Templates and Stored Queries — Mustache, Parameterization, and Versioning

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `search` `elasticsearch` `opensearch` `mustache` `search-templates` `security`

---

## Table of Contents

- [Summary](#summary)
- [1. Why Templates Exist](#1-why-templates-exist)
- [2. Mustache, Briefly](#2-mustache-briefly)
- [3. Storing and Calling a Template](#3-storing-and-calling-a-template)
- [4. Mustache Inside JSON — The Awkward Part](#4-mustache-inside-json--the-awkward-part)
- [5. Multi-Search Templates](#5-multi-search-templates)
- [6. Stored Scripts vs Search Templates](#6-stored-scripts-vs-search-templates)
- [7. Security — Templates as an Injection Boundary](#7-security--templates-as-an-injection-boundary)
- [8. Versioning and Blue/Green Template Swap](#8-versioning-and-bluegreen-template-swap)
- [9. OpenSearch Equivalents](#9-opensearch-equivalents)
- [10. Practical Pitfalls](#10-practical-pitfalls)
- [Related](#related)
- [References](#references)

---

## Summary

A **search template** is a parameterized query body stored server-side in Elasticsearch (or OpenSearch) and rendered through the **Mustache** template engine at request time. The client sends a template ID plus a `params` map; the cluster substitutes the parameters into the stored shape and runs the resulting query. This is the search-cluster equivalent of a prepared statement: query *shapes* are version-controlled and live in the cluster, query *values* travel from clients, and there is no string concatenation on the client to splice user input into a JSON DSL blob.

The benefits are operational (one place to fix a query bug across every client language), architectural (the search shape is deployable infrastructure, not a string buried in five service repos), and security-relevant (parameters cannot escape their position in the rendered tree). The pitfalls are mostly Mustache-shaped: the default escaper is HTML-aware and will mangle JSON, and iteration sections need discipline to keep output valid. This doc walks the API surface, the templating quirks, the security argument, and a versioning pattern that lets you ship a new template shape without a flag day.

---

## 1. Why Templates Exist

A typical search-backed feature starts with a backend service composing a JSON query body in code:

```ts
const body = {
  query: {
    bool: {
      must: [{ match: { title: req.query.q } }],
      filter: [{ term: { status: "published" } }]
    }
  }
};
await es.search({ index: "articles", body });
```

That works until a second service (Java, Python, Go) needs the same query and copy-pastes the shape; until a relevance engineer wants to tune `should` clauses or add a `function_score` wrapper without coordinating a deploy of every client; or until somebody concatenates `req.query.q` directly into a JSON string and opens an injection-shaped bug.

A search template moves the *shape* of the query into the cluster and leaves the *values* in the client. The call becomes "run template `articles_search_v3` with `{q, status, page}`" and the relevance team owns the template's contents. This is the same reasoning that produced prepared statements in JDBC and parameterized queries in PostgreSQL: separate code (the query shape) from data (the user-supplied values), and let the engine handle the substitution safely.

---

## 2. Mustache, Briefly

Elasticsearch's template engine is **Mustache** — a logic-less templating language whose canonical spec is hosted at `mustache.github.io/mustache.5.html`. "Logic-less" means there are no expressions, no arithmetic, no string functions; the entire feature surface is variable substitution and a handful of section tags.

The four constructs that matter for search templates:

- **Variable**: `{{name}}` substitutes the value of `name` from the params map.
- **Section**: `{{#name}} ... {{/name}}` renders the inner block when `name` is truthy. If `name` is a list, the block renders once per element with the element pushed onto the context stack.
- **Inverted section**: `{{^name}} ... {{/name}}` renders when `name` is missing or falsy — the way you express "default this value."
- **Triple-mustache / unescaped**: `{{{name}}}` substitutes without HTML-escaping. This is the form you almost always want inside JSON because Mustache's default `{{name}}` HTML-escapes `&`, `<`, `>`, `"`, `'`, `/` per the spec, which corrupts JSON values that contain quotes or angle brackets.

That last point is the single most common Mustache footgun in Elasticsearch templates. The spec says variable tags "should be HTML escaped," and Elasticsearch follows the spec faithfully, so a query parameter containing `"` becomes `&quot;` in your JSON body and the cluster returns a parse error. Use `{{{value}}}` for any string that flows into a JSON string, or pre-encode values via the `params.toJson` helper described in §4.

---

## 3. Storing and Calling a Template

Templates have three endpoints worth memorising. All examples below are against Elasticsearch 8.x; the OpenSearch equivalents are mechanically identical (see §9).

**Store** a template under an ID:

```http
POST _scripts/articles_search_v3
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "bool": {
          "must": [
            { "match": { "title": "{{query_string}}" } }
          ],
          "filter": [
            { "term": { "status": "{{status}}" } }
          ]
        }
      },
      "from": "{{from}}",
      "size": "{{size}}"
    }
  }
}
```

**Run** the template against an index:

```http
POST articles/_search/template
{
  "id": "articles_search_v3",
  "params": {
    "query_string": "kafka rebalance",
    "status": "published",
    "from": 0,
    "size": 20
  }
}
```

**Render** without executing — invaluable for debugging. The cluster substitutes parameters and returns the JSON it *would* have run, so you can verify the shape before anything hits Lucene:

```http
POST _render/template
{
  "id": "articles_search_v3",
  "params": { "query_string": "kafka", "status": "published", "from": 0, "size": 20 }
}
```

You can also pass `"source": { ... }` instead of `"id"` for an **inline** template — the same Mustache rendering against an ad-hoc body. Inline templates are useful for one-off scripts and tests; stored templates are the right choice for production code paths because they version-control the query shape inside the cluster's `.scripts` index.

The `_render/template` endpoint is the closest thing to a unit test your template will get. A CI step that renders each template with representative params and JSON-parses the result catches Mustache typos before they ship.

---

## 4. Mustache Inside JSON — The Awkward Part

Mustache was designed for HTML. JSON is fussier; a few patterns recur often enough to memorise.

**Numbers**: writing `"size": "{{size}}"` renders `"size": "20"` (a string) which Elasticsearch coerces. A cleaner pattern is `"size": {{size}}` with no quotes, but the un-rendered template body is then invalid JSON, which trips up linters. The pragmatic answer is the `{{#toJson}}` helper.

**Arrays / objects**: Elasticsearch ships a Mustache extension that serializes a parameter as JSON via `{{#toJson}}param{{/toJson}}`. Use this for list or object parameters — for instance, an array of term filters:

```json
{
  "query": {
    "terms": { "tags": {{#toJson}}tags{{/toJson}} }
  }
}
```

Called with `"params": { "tags": ["kafka", "lucene"] }`, this renders `"tags": ["kafka","lucene"]`. Without `toJson`, you would iterate manually and worry about trailing commas.

**String joining**: `{{#join}}param{{/join}}` joins an array with commas, useful when you want a comma-separated list rather than a JSON array.

**Conditionals as defaults**: combine a section with an inverted section to express "use the param if present, else fall back":

```json
{
  "size": {{#size}}{{size}}{{/size}}{{^size}}10{{/size}}
}
```

If `size` is omitted, the inverted block fires and the rendered body has `"size": 10`. Mustache has no `else`; this two-block pattern is the idiomatic equivalent.

**Lists of clauses**: Mustache sections iterate. The standard hack for "comma between every element except the last" is `{{^last}},{{/last}}` with the caller marking the final element with `"last": true` — Mustache has no built-in `@last`. In practice `toJson` is almost always cleaner than hand-iterating.

---

## 5. Multi-Search Templates

The `_msearch/template` endpoint runs many templated searches in one request and returns an array of responses. The body is the same NDJSON shape as `_msearch`: alternating header lines (index, template ID) and body lines (params):

```text
POST _msearch/template
{ "index": "articles" }
{ "id": "articles_search_v3", "params": { "query_string": "kafka", "status": "published", "from": 0, "size": 20 } }
{ "index": "articles" }
{ "id": "articles_search_v3", "params": { "query_string": "lucene", "status": "published", "from": 0, "size": 20 } }
```

This is the right tool when a single page needs several parallel searches (results + suggestions + facet counts). The cluster runs them concurrently. Note the trailing newline at the end of the body — it is part of the NDJSON contract and easy to omit when hand-rolling requests.

---

## 6. Stored Scripts vs Search Templates

Both live under `_scripts/<id>` in the cluster, distinguished by `lang`:

- `lang: mustache` → search template, used for query-body substitution as described above.
- `lang: painless` → stored script, used for in-query computation: scripted fields, scripted queries (`script_score`, `script` query), update-by-query operations, and ingest pipelines.

Reach for Painless when a query needs to compute something the DSL cannot — a custom relevance score combining BM25 with a recency decay, or a runtime field derived from `_source` at query time. The cluster compiles Painless once and caches the bytecode, so storing the script under an ID is faster than sending the source every call.

The two are complementary: a search template can reference a stored Painless script by ID inside its rendered body. A common pattern is a Mustache template whose `script_score.script` block is `{ "id": "recency_decay_v2", "params": { "now": "{{now}}" } }` — Mustache handles the shape, Painless the scoring math. Both share the cluster's scripting subsystem (compilation cache, circuit breakers, `script.max_compilations_rate` tuning).

---

## 7. Security — Templates as an Injection Boundary

Search templates are the cluster-side analog of parameterized SQL queries. A naive client-side query builder splices user input into a JSON string:

```ts
const body = `{ "query": { "match": { "title": "${userInput}" } } }`;
await es.search({ index: "articles", body });
```

If `userInput` contains `"} } }, "size": 10000, "_source": ["password"], "x": "`, the resulting string is a parser-valid JSON document that asks the cluster for ten thousand documents including a `password` field the legitimate query never exposed. This is structurally the same vulnerability as SQL injection — user data treated as code because it crosses the code/data boundary unescaped. See [SQL Injection Deep Dive](../../security/web-attacks/sql-injection-deep-dive.md) for the full argument and OWASP's reference.

Templates close the boundary. The template body is *the* query shape; parameters substitute into named slots and cannot extend the JSON tree. Even if `userInput` contains JSON-shaped syntax, Mustache renders it *as a string value* into the `match.title` slot — it cannot escape that slot, add new top-level keys, or change `size`. The escaping caveat from §2 applies: use `{{{query_string}}}` for raw substitution or `{{#toJson}}query_string{{/toJson}}` which serializes the value as a properly quoted JSON string.

What templates do *not* defend against:

- **Application-layer overreach**: a template exposing `_source: ["*"]` returns whatever fields the caller asks for. Pair with field-level security or post-filter in the application.
- **Resource exhaustion**: an unbounded `size`/`from` parameter is a deep-pagination DoS. Validate numeric params at the boundary and prefer `search_after` for deep pagination.
- **Expensive query patterns**: a wildcard or regex pattern in a parameter still runs against the inverted index. Strip wildcards from untrusted input or use the `match` family rather than `wildcard`/`regexp` for free text.

Templates give you the equivalent of a prepared statement; you still need the rest of the input-validation playbook.

---

## 8. Versioning and Blue/Green Template Swap

Stored templates are mutable — `POST _scripts/articles_search` overwrites whatever was there, with no built-in rollback. Two patterns make this safer.

**Suffix the template ID with a version**: `articles_search_v3`, `articles_search_v4`. Clients call a specific version, so changing the template means deploying a new ID, not mutating the old one. Old versions stay until you garbage-collect them, giving you a clean rollback (revert the client's template-ID constant).

**Blue/green swap via an indirection table**: there is no alias system for stored scripts, but a thin lookup table in application config achieves the same effect:

```ts
const TEMPLATE_REGISTRY = {
  articles_search: "articles_search_v3", // bump to v4 when ready
} as const;
```

The same pattern supports percentage rollouts — route a fraction of traffic to `_v4` while the rest stays on `_v3` and compare relevance metrics. Pair this with a CI job that renders each registered version against a fixture set of params and snapshot-compares the output to catch accidental shape changes.

---

## 9. OpenSearch Equivalents

OpenSearch forked from Elasticsearch 7.10.2 in 2021 and the search-template surface is unchanged. The same endpoints (`POST _scripts/<id>`, `_search/template`, `_render/template`, `_msearch/template`), the same Mustache engine, the same `toJson` and `join` helpers. An Elastic 7.x template is portable to OpenSearch with no changes.

Divergences worth knowing:

- **Painless** is preserved and works the same; no licensing wrinkles around stored scripts.
- **Field-level security** lives in the OpenSearch Security plugin (formerly Open Distro); configuration UX differs but the §7 caveat applies in both engines.
- **Cluster setting names** that gate scripting (`script.max_compilations_rate`, `script.allowed_types`, `script.allowed_contexts`) are the same.

For a Spring Boot service that targets both engines, the high-level Java client's `searchTemplate` method works against both — the wire format is identical.

---

## 10. Practical Pitfalls

A grab-bag of things that bite in practice.

- **HTML escaping corrupts JSON.** Per §2, `{{value}}` runs the spec's HTML escaper and turns `"` into `&quot;`. Use `{{{value}}}` for any string flowing into JSON, or `{{#toJson}}value{{/toJson}}`.
- **Numeric coercion via quoted templates.** `"size": "{{size}}"` renders as `"size": "20"`. Elasticsearch coerces; Painless and some clients do not. Prefer `"size": {{size}}` or `"size": {{#toJson}}size{{/toJson}}`.
- **Missing parameters are empty strings.** A template that references `{{status}}` with no `status` in params renders an empty string, producing `"term": { "status": "" }` and a search that matches nothing. Pair optional params with `{{^status}}default{{/status}}` or validate at the client.
- **Trailing-comma tax in iteration.** Hand-iterating with `{{#items}}...{{/items}}` produces trailing commas unless you mark the last element. `toJson` sidesteps this.
- **`_render/template` is your unit test.** Render every template in CI against representative params and JSON-parse the result. Catches every Mustache typo in seconds.
- **Compilation rate limits.** Frequent overwrites during deploys can trip `script.max_compilations_rate`. Roll out under new IDs (per §8) rather than overwriting in place.
- **No transactional swap.** A template overwrite is not atomic with respect to in-flight queries; the version-suffix pattern in §8 sidesteps this.

---

## Related

- [Security — SQL Injection Deep Dive](../../security/web-attacks/sql-injection-deep-dive.md) — the broader argument for parameterization-as-injection-defense; search templates are the search-cluster expression of the same pattern
- _Inverted Index Internals (../foundations/01-inverted-index-internals.md)_ — what the rendered query body actually traverses on the read path
- _Query DSL — Bool, Match, and Filter Context (planned)_ — the DSL shapes that templates wrap
- _Painless Scripting and Script Score (planned)_ — when to reach for stored scripts inside a template
- [Database — Prepared Statements and Query Plan Caching](../../database/INDEX.md) — the relational analog to stored templates
- [Observability — Structured Logs and Tracing](../../observability/INDEX.md) — how to instrument templated search calls so the template ID and params land in your traces

## References

- [Elasticsearch Reference — Search template API](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-template.html) — authoritative reference for `POST _scripts/<id>`, `_search/template`, `_render/template`, and `_msearch/template`
- [Elasticsearch Reference — How to use scripts](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-scripting-using.html) — stored vs inline scripts, compilation cache, and `script.max_compilations_rate`
- [Elasticsearch Reference — Painless scripting language](https://www.elastic.co/guide/en/elasticsearch/painless/current/index.html) — the scripting language used for non-template stored scripts referenced in §6
- [Mustache(5) — Logic-less templates](https://mustache.github.io/mustache.5.html) — the canonical Mustache spec; documents the default HTML-escaping behavior of `{{name}}` and the unescaped `{{{name}}}` form referenced in §2
- [OpenSearch Documentation — Search templates](https://opensearch.org/docs/latest/search-plugins/search-template/) — OpenSearch's mirror of the Elasticsearch search-template API referenced in §9
- [OpenSearch Documentation — Painless scripting language](https://opensearch.org/docs/latest/api-reference/script-apis/index/) — stored-script API surface in OpenSearch
- [OWASP — SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html) — the parameterization argument that §7 generalizes from SQL to search-template DSL
