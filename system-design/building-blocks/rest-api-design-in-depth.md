---
title: "REST API Design in Depth"
date: 2026-04-26
updated: 2026-04-26
tags: [system-design, api, rest, http]
---

# REST API Design in Depth — Resources, Pagination, Versioning, Errors

**Date:** 2026-04-26 | **Updated:** 2026-04-26
**Tags:** `system-design` `api` `rest` `http`

## Table of Contents

- [Summary](#summary)
- [Why This Matters](#why-this-matters)
- [Overview — What "RESTful" Actually Means In Production](#overview--what-restful-actually-means-in-production)
- [Key Concepts](#key-concepts)
  - [Resource Naming — Nouns, Plurals, and the Nesting Trap](#resource-naming--nouns-plurals-and-the-nesting-trap)
  - [HTTP Method Semantics — Safety, Idempotency, and What PATCH Actually Means](#http-method-semantics--safety-idempotency-and-what-patch-actually-means)
  - [Pagination — Offset, Cursor, and Keyset](#pagination--offset-cursor-and-keyset)
  - [Versioning — URL, Header, and Evolutionary](#versioning--url-header-and-evolutionary)
  - [Errors — RFC 7807 Problem Details](#errors--rfc-7807-problem-details)
  - [Idempotency at the API Layer — `Idempotency-Key` Header](#idempotency-at-the-api-layer--idempotency-key-header)
  - [HTTP Caching — `ETag`, `Cache-Control`, `Last-Modified`](#http-caching--etag-cache-control-last-modified)
  - [Rate-Limit Headers](#rate-limit-headers)
  - [HATEOAS — The Reality Check](#hateoas--the-reality-check)
  - [OpenAPI / JSON Schema — Contracts as Code](#openapi--json-schema--contracts-as-code)
- [Trade-offs](#trade-offs)
  - [Offset vs Cursor vs Keyset Pagination](#offset-vs-cursor-vs-keyset-pagination)
  - [URL vs Header Versioning](#url-vs-header-versioning)
- [Code Examples](#code-examples)
  - [Cursor-Based Pagination Endpoint](#cursor-based-pagination-endpoint)
  - [RFC 7807 Problem Details Error Response](#rfc-7807-problem-details-error-response)
  - [`Idempotency-Key` Server-Side Handling](#idempotency-key-server-side-handling)
- [Real-World Uses](#real-world-uses)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

REST is not a religion; it is a set of HTTP conventions that, when applied consistently, make APIs cheap to learn and cheap to evolve. The high-leverage decisions are: **name resources as plural nouns**, **respect HTTP method semantics** (GET safe + idempotent, POST creates, PUT idempotent replace, PATCH partial, DELETE idempotent), **paginate with cursors not offsets** once you cross a few thousand rows, **version evolutionarily** with a deprecation timeline, **return errors as RFC 7807 Problem Details**, **support `Idempotency-Key`** on any non-idempotent endpoint that touches money, **use `ETag` and `Cache-Control` for caching**, and **treat OpenAPI as the source of truth** for the contract. HATEOAS is in the textbook but is essentially absent from production APIs — Stripe, GitHub, Twilio, and Slack are not hypermedia-driven, and you do not have to be either.

## Why This Matters

You can spot a junior API design in fifteen seconds: `POST /getUser`, integer IDs that leak creation order, `?page=147` queries that take 3 seconds, error bodies that are bare strings, no versioning policy, and no idempotency on the payment endpoint. Each of these turns into a customer ticket, a P2 incident, or a multi-quarter migration when the API has 200 integrations against it.

The goal of this doc is not to recite "REST 101"; you have read that. It is to give you the precise vocabulary and the precise defaults that hold up at scale, so that in a design review you can:

- Reject `POST /getUser` and explain why a verb in the URL costs you cache, idempotency, and method semantics.
- Push back on offset pagination when the table is past 100k rows.
- Choose between URL versioning and header versioning with reasons, not preferences.
- Defend a `Idempotency-Key` design that survives client retries _and_ proxy retries _and_ partial failures.
- Defend the choice not to do HATEOAS without sounding like you skipped a chapter.

## Overview — What "RESTful" Actually Means In Production

Roy Fielding's dissertation defines REST as a set of architectural constraints: client/server, stateless, cacheable, layered, uniform interface, and (optionally) code-on-demand. In practice, the only constraints that get enforced in production APIs are:

1. **Stateless requests** — every request carries enough context (auth, idempotency key, conditional headers) to be processed independently.
2. **Cacheable GET** — `GET` responses are cacheable by default unless explicitly forbidden.
3. **Uniform interface (mostly)** — consistent resource URLs, consistent method semantics, consistent error shape.

What gets dropped almost universally:

- **HATEOAS** (clients discovering URLs from response links) — replaced by OpenAPI specs and SDK code generation.
- **Strict statelessness** — server-side sessions, idempotency-key tables, and rate-limit counters all require server state. The "stateless" guarantee in production means "no per-connection state", not "no state anywhere."
- **Strict resource-orientation** — every real-world API has at least one RPC-shaped endpoint (`POST /charges/:id/refund`, `POST /messages/:id/resend`). Pretending otherwise is theatre.

The pragmatic definition for this doc: **a REST API is an HTTP API that uses HTTP methods, status codes, headers, and resource-oriented URLs as its uniform interface, evolves with explicit versioning, and expresses errors and pagination consistently.**

## Key Concepts

### Resource Naming — Nouns, Plurals, and the Nesting Trap

The naming rules that almost every production API follows:

| Rule | Example | Anti-example |
|------|---------|--------------|
| Resources are **plural nouns** | `/users`, `/orders`, `/invoices` | `/user`, `/getUser`, `/UserList` |
| Identifiers in path, not query | `/users/123` | `/users?id=123` |
| Verbs as **methods**, not URL segments | `DELETE /users/123` | `POST /deleteUser?id=123` |
| Sub-collections only when truly nested | `/users/123/orders` | `/orders?user_id=123` (often better) |
| Filters / search / sort in query string | `/orders?status=paid&sort=-created_at` | `/orders/paid` |
| `kebab-case` (not snake_case or camelCase) for multi-word path segments | `/payment-methods` | `/paymentMethods` |
| RPC-shaped actions only when CRUD doesn't fit, then namespace clearly | `POST /charges/123/refund` | `POST /refundCharge?id=123` |

**The nesting trap.** Resources can usually be addressed two ways:

- Nested: `GET /users/123/orders` — orders belonging to user 123.
- Flat with filter: `GET /orders?user_id=123` — same result.

Heuristic: nest **only when the child cannot meaningfully exist outside the parent's scope** (`/repos/:owner/:repo/issues` in GitHub) or when **the API key is scoped to the parent** (`/customers/cus_X/sources` in Stripe). Otherwise, prefer the flat form — it composes better with filtering, pagination, and access control, and it never forces you to support both `/users/:id/orders` and `/teams/:id/orders` for the same underlying resource.

**Identifiers.** Avoid leaking integer auto-increment IDs in public APIs. They reveal volume (`order_id=8421` tells a competitor your monthly volume) and ordering (`order 8420` was placed before `8421`, which is sometimes sensitive). Use opaque prefixed IDs like Stripe (`ch_3OabcDEFghi456`, `cus_OnzPqrSTuvw789`) — readable, namespaced, sortable internally, and unguessable externally.

### HTTP Method Semantics — Safety, Idempotency, and What PATCH Actually Means

RFC 9110 (HTTP Semantics) is the authoritative source. Two properties define the contract:

- **Safe**: the request does not change server state. Caches, retries, and prefetchers may issue safe requests freely.
- **Idempotent**: issuing the request N times has the same observable effect as issuing it once. Retries are safe.

| Method | Safe | Idempotent | Cacheable | Typical use |
|--------|------|------------|-----------|-------------|
| `GET`     | yes | yes | yes | Read a resource or list. Never has a body that changes meaning. |
| `HEAD`    | yes | yes | yes | Headers only — `ETag`, `Last-Modified`, existence checks. |
| `OPTIONS` | yes | yes | no  | CORS preflight, capability discovery. |
| `POST`    | no  | no  | rare | Create a resource at a server-chosen URL; or RPC-style action. |
| `PUT`     | no  | yes | no  | Replace a resource at a known URL with the request body. |
| `PATCH`   | no  | not required | no | Partial update — apply a delta. |
| `DELETE`  | no  | yes | no  | Remove a resource. Second `DELETE` returns 404 or 204. |

Common confusions worth nailing down:

- **`PUT` is replace, `PATCH` is partial.** `PUT /users/123 {name: "Quan"}` should leave the resource as `{name: "Quan"}` — every field not in the body is unset. If you mean "update only the name," use `PATCH /users/123 {name: "Quan"}`. Many APIs cheat and treat `PUT` as `PATCH`; this is fine internally but confusing in public contracts.
- **`PATCH` semantics are not standardized by HTTP.** RFC 5789 says "apply a delta" but does not say what the delta format is. The two production-grade choices are **JSON Merge Patch** (RFC 7396, simple, no nested-null support) and **JSON Patch** (RFC 6902, expressive, op-based). Pick one and document it.
- **`POST` is not "the create verb."** It is "the non-idempotent action verb." Creating a user is `POST /users`. Sending a message is `POST /messages`. Refunding a charge is `POST /charges/:id/refund`. Anything that should not be retried by a dumb proxy goes through `POST` (or use `Idempotency-Key` to make it safely retryable — see below).
- **`DELETE` is idempotent.** `DELETE /users/123` twice returns success the first time, 404 the second time, but the *effect* is the same: the resource is gone. Do not return an error on the second call other than 404 — that breaks the idempotency contract.
- **Status codes carry meaning.** `200 OK`, `201 Created` (with `Location` header), `202 Accepted` (async), `204 No Content` (DELETE), `400 Bad Request` (client validation), `401 Unauthorized` (no/invalid auth), `403 Forbidden` (auth ok, not allowed), `404 Not Found`, `409 Conflict` (uniqueness, version mismatch), `410 Gone` (deprecated/removed), `422 Unprocessable Entity` (semantic validation), `429 Too Many Requests`, `500 Internal Server Error`, `502/503/504` (upstream/availability/timeout). Do not invent codes. Do not return `200` with `{success: false}` — use the status code.

### Pagination — Offset, Cursor, and Keyset

Three production patterns. The right answer depends on the dataset size, the consistency requirements, and whether you need a total count.

**1. Offset + limit (a.k.a. "page-based").**

```
GET /orders?page=3&page_size=50
GET /orders?offset=100&limit=50
```

Translates directly to `LIMIT 50 OFFSET 100`. Easy to implement, easy for humans to reason about, supports jumping to arbitrary pages.

Two fatal flaws at scale:

- **Deep pages are slow.** `OFFSET 1000000 LIMIT 50` requires the database to scan and discard one million rows. There is no index that fixes this in the general case.
- **Inserts shift pages.** If a row is inserted while the user is paginating, page boundaries shift — they will see the same row twice on consecutive pages, or skip a row entirely.

Use offset only when (a) the result set is bounded (admin UIs, internal dashboards) and (b) page-jumping is a real requirement.

**2. Cursor-based.**

```
GET /orders?limit=50
→ {data: [...], next_cursor: "eyJpZCI6Im9yZF8xMjMifQ"}

GET /orders?limit=50&cursor=eyJpZCI6Im9yZF8xMjMifQ
```

The cursor is an **opaque, server-issued token** that encodes "where to resume." Internally it is usually a base64-encoded JSON like `{"id": "ord_123", "created_at": "2026-04-25T10:00:00Z"}`. The server uses it to construct a `WHERE` clause on indexed columns:

```sql
WHERE (created_at, id) < ('2026-04-25T10:00:00Z', 'ord_123')
ORDER BY created_at DESC, id DESC
LIMIT 50
```

Properties:

- **Stable across inserts** — a new row inserted ahead of the cursor doesn't shift anything; the next page resumes exactly where it left off.
- **Constant-time per page** — no scan-and-discard; the index seeks directly to the cursor's position.
- **No total count** — you do not get `total: 12847` for free; running a `COUNT(*)` defeats the purpose.
- **No page jumping** — you can only go forward (and optionally backward with a `prev_cursor`); arbitrary jumps are not possible.

This is the right default for any user-facing list of more than a few thousand rows.

**3. Keyset pagination (a.k.a. "seek pagination").**

Identical machinery to cursor-based, but the "cursor" is exposed as the actual sort-key value (or the last item's ID) rather than an opaque token:

```
GET /orders?limit=50&before_id=ord_123
GET /orders?limit=50&after_created_at=2026-04-25T10:00:00Z
```

Same SQL, different ergonomics. Cursor-based hides the implementation; keyset exposes it. Cursor is preferred for public APIs (you can change the underlying sort key without breaking clients); keyset is fine for internal APIs or when you specifically want clients to be able to construct cursors themselves.

### Versioning — URL, Header, and Evolutionary

Three approaches, plus the meta-strategy of trying not to break things in the first place.

**URL versioning (`/v1/users`).** What Stripe, GitHub REST, Twilio, and most public APIs do.

- Pros: trivially visible, trivially routable (different versions can be served by different services), trivially documented, trivially curl-able.
- Cons: a "version" is a coarse-grained promise — any change forces a v2, even tiny ones; or you get version creep where v1 means seventeen different things at seventeen different points in time.

**Header versioning (`Accept: application/vnd.example.v2+json`).** What GitHub used briefly, what some hypermedia APIs do.

- Pros: keeps URLs stable, allows fine-grained content negotiation, works with conditional GET cleanly.
- Cons: invisible in browser address bar, harder to curl, harder to log, easy for clients to forget.

**Evolutionary (Stripe's approach).** Each API key is pinned to an "API version date" (e.g. `2024-09-30`). New behavior ships behind a date; old clients keep their pinned version forever. Breaking changes are deferred to the next version date and announced with a deprecation timeline.

- Pros: clients never break unless they explicitly opt in. Server can deprecate things gracefully. No big-bang migrations.
- Cons: server has to maintain N versions of behavior simultaneously — non-trivial engineering investment. Requires a clean way to express "this transformation runs only for clients pinned at version X or older."

**The non-negotiable rules**, regardless of strategy:

- **Additive changes are not breaking.** Adding a new field to a response, adding an optional request field, adding a new endpoint — these never bump the version.
- **Removing a field is breaking.** Renaming a field is breaking. Tightening validation is breaking. Changing the meaning of a field is breaking.
- **Breaking changes get a deprecation timeline.** Minimum 6 months for serious public APIs, 12+ months for ones with enterprise integrations. Announce in changelog, return `Deprecation` and `Sunset` headers (RFC 8594), email integrators, monitor usage, then remove.
- **Never break v1 silently.** The single fastest way to lose API customers' trust is to "fix" something in production that breaks their integration without notice.

### Errors — RFC 7807 Problem Details

RFC 7807 (recently obsoleted and replaced by RFC 9457, same shape) defines a standard JSON error envelope so clients don't have to learn a new error format per API. The shape:

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Your request parameters didn't validate.",
  "status": 400,
  "detail": "The 'email' field must be a valid RFC 5322 address.",
  "instance": "/orders/ord_8421",
  "errors": [
    {"field": "email", "code": "invalid_format"}
  ]
}
```

Required fields: `type` (URI identifying the error class — typically also a documentation URL), `title` (short human description, same for every instance of this error class), `status` (HTTP status code, also in the response). Optional: `detail` (request-specific human description), `instance` (URI identifying this specific occurrence). You may add custom extension fields like `errors` for per-field validation details, `request_id` for support, `retry_after` for rate limits.

The `Content-Type` for problem responses is `application/problem+json`.

Why this matters: when every API in your platform returns the same shape, your SDKs can centralize error handling, your support team can ask for `request_id`, and your clients don't have to write per-endpoint error parsers. This is the single highest-leverage thing you can do to make an API less painful to integrate against.

### Idempotency at the API Layer — `Idempotency-Key` Header

Network retries are a fact of life: TCP RSTs, gateway timeouts, mobile clients dropping connections mid-request. For `GET`/`PUT`/`DELETE` retries are safe because the methods are idempotent. For `POST`, retries can double-create. For payment endpoints, "double-create" means "double-charged customer."

The pattern (popularized by Stripe and now standard): the client generates a UUID per logical operation and sends it in the `Idempotency-Key` header. The server stores `(key, request_hash, response)` for a bounded window (Stripe: 24 hours).

Server-side state machine on `POST /charges` with `Idempotency-Key: 8e1a-...`:

1. **Key not seen** — process request, store `(key, request_hash, response, status)`, return response.
2. **Key seen, request_hash matches, request still in flight** — return `409 Conflict` with `{type: ".../request-in-progress"}`. Client should retry with backoff; do not let two retries race each other.
3. **Key seen, request_hash matches, response stored** — replay stored response with original status code. Client gets exactly the same answer it would have gotten if the original response hadn't been lost.
4. **Key seen, request_hash differs** — return `422 Unprocessable Entity` with `{type: ".../idempotency-key-reused"}`. Same key with different parameters is a client bug.

Critical implementation notes:

- The key is **scoped per-endpoint or per-account**, not global. Two different endpoints can use the same key.
- The stored response includes the **status code and body**, not just the body — replaying must be byte-identical.
- The window must be long enough to cover client retry timelines (often 24h) but short enough to bound storage. Use TTL-based storage (Redis with TTL, DynamoDB TTL, Postgres with a cron-vacuumed `idempotency_keys` table).
- The `request_hash` should hash **the canonical request body** (sorted keys, normalized whitespace) — not the raw bytes, which can vary between retries due to e.g. Content-Encoding.
- Idempotency at the API layer is **not the same as exactly-once semantics across the system**. The downstream side effects (charging the card, sending the email) need their own idempotency story — see [../communication/idempotency-and-exactly-once.md](../communication/idempotency-and-exactly-once.md).

### HTTP Caching — `ETag`, `Cache-Control`, `Last-Modified`

Caching cuts read latency, server CPU, and bandwidth all at once. Three headers do the heavy lifting:

- **`Cache-Control`** — controls who can cache and for how long. `Cache-Control: public, max-age=300` says "any cache may store this for 5 minutes." `Cache-Control: private, no-cache` says "user-agent only, must revalidate every time." `Cache-Control: no-store` says "do not cache anywhere" — use for sensitive data only; it disables CDNs.
- **`ETag`** — a strong opaque identifier of the resource version (often a hash of the response body). Server returns `ETag: "abc123"` on the response. Client sends `If-None-Match: "abc123"` on the next request. If the resource hasn't changed, server returns `304 Not Modified` with no body.
- **`Last-Modified`** — fallback when ETag is too expensive to compute; based on a timestamp. Client sends `If-Modified-Since`. Less precise (1-second granularity, vulnerable to clock skew) but cheap.

ETag also enables **optimistic concurrency control** for writes: client sends `If-Match: "abc123"` on `PUT`/`PATCH`. If the resource has changed, server returns `412 Precondition Failed` and the client must reload, reconcile, and retry. This is the cheapest mechanism for "two users editing the same record" without server-side locking.

Defaults that hold up:

- Static immutable resources (versioned asset URLs): `Cache-Control: public, max-age=31536000, immutable`.
- Tenant-private GETs that change occasionally: `Cache-Control: private, max-age=60` plus `ETag`.
- Personal feed endpoints: `Cache-Control: private, no-cache` plus `ETag` (must revalidate every time, but 304 saves bandwidth).
- Anything with auth: never `public`. Use `private` so CDNs don't cross-serve it.

### Rate-Limit Headers

There is no single official spec, but the de facto convention (used by GitHub, Stripe, Twitter, and many others) is:

```
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4992
X-RateLimit-Reset: 1714123200
```

The IETF draft `RateLimit-*` (without the `X-`) is gaining traction. Either is fine; pick one and stick with it. On `429 Too Many Requests` always include `Retry-After: 30` (seconds, or HTTP-date). Document the buckets clearly: per-API-key, per-endpoint, per-IP, or all of the above.

See [./rate-limiters.md](./rate-limiters.md) for the algorithms (token bucket, sliding window) under the hood.

### HATEOAS — The Reality Check

HATEOAS (Hypermedia As The Engine Of Application State) is the constraint that responses include links to related actions, and clients navigate by following links rather than constructing URLs:

```json
{
  "id": "ord_123",
  "status": "pending",
  "_links": {
    "self": {"href": "/orders/ord_123"},
    "cancel": {"href": "/orders/ord_123/cancel", "method": "POST"},
    "ship": {"href": "/orders/ord_123/ship", "method": "POST"}
  }
}
```

Roy Fielding's dissertation says this is **the** defining REST constraint. Real APIs almost universally don't do it. Why:

- Public APIs are consumed by **typed SDKs**, not generic hypermedia clients. Stripe ships an official Stripe SDK in 9 languages; nobody is parsing `_links` to find the cancel URL.
- The OpenAPI spec already documents every endpoint; embedding URLs in responses is duplicative.
- Adding `_links` triples response size for tiny benefit.
- The "client navigates by links" promise was that servers could change URLs freely. In practice, URLs in public APIs are part of the contract and don't change.

Where HATEOAS-ish patterns do show up: **state transitions in workflow APIs** (returning the list of currently-allowed actions on an order based on its state — `["cancel", "ship"]` if pending, `["refund"]` if paid). That's worth doing, and you can do it without committing to full hypermedia.

The honest take: don't aim for HATEOAS. Aim for clear OpenAPI specs, clear state-machine documentation, and consistent error responses.

### OpenAPI / JSON Schema — Contracts as Code

OpenAPI (formerly Swagger) is a JSON/YAML schema that describes every endpoint, every parameter, every response, every error. Once you have it, you get for free:

- **Generated SDKs** in TypeScript, Java, Python, Go, etc. via `openapi-generator`.
- **Mock servers** for client teams to develop against before the API is built (Prism, Stoplight).
- **Contract tests** that verify the running server matches the spec (Dredd, Schemathesis).
- **Interactive docs** via Swagger UI, Redoc, Stoplight Elements.
- **Request/response validation** middleware (express-openapi-validator, Spring's springdoc).

The "spec-first" workflow: write the OpenAPI YAML, generate types and validation, implement handlers against the generated types. The "code-first" workflow (annotations on controllers, e.g. springdoc-openapi) generates the YAML from the code. Both work; spec-first scales better across multiple services and across teams that include non-server engineers (mobile, frontend, partner integrations).

JSON Schema (the underlying validation language used by OpenAPI) can also be used standalone for `application/json` request body validation independently of OpenAPI — useful in services that aren't fully spec'd yet.

## Trade-offs

### Offset vs Cursor vs Keyset Pagination

| Dimension | Offset/Limit | Cursor | Keyset |
|-----------|--------------|--------|--------|
| **Performance on deep pages** | O(N) — scans and discards | O(log N) — index seek | O(log N) — index seek |
| **Stable across inserts** | No — pages shift | Yes | Yes |
| **Total count available** | Yes (extra `COUNT(*)`) | No, by design | No, by design |
| **Page jumping** | Yes (`?page=147`) | No (forward/backward only) | No |
| **Client implementation effort** | Trivial | Pass opaque token | Pass actual key value |
| **Server implementation effort** | Trivial | Encode/decode token, ensure stable sort key | Same as cursor minus the encoding |
| **Ergonomics for humans** | Best — page numbers map to UI | Worst — opaque tokens | Middle |
| **API evolution** | Can change underlying impl freely | Token is opaque, server can change format | Key shape is exposed; harder to evolve |

**Default recommendation:**

- Internal admin/back-office UIs with bounded data and page-jumping needs: **offset**.
- Public APIs, infinite-scroll UIs, anything with >10k rows: **cursor** (opaque token).
- Internal services where clients understand the data model and want to construct cursors: **keyset**.

### URL vs Header Versioning

| Dimension | URL (`/v1/users`) | Header (`Accept: vnd.x.v2+json`) | Evolutionary (Stripe-style) |
|-----------|-------------------|-----------------------------------|------------------------------|
| **Visible / debuggable** | Best — visible in logs, browsers, curl | Worst — invisible without inspecting headers | Middle — visible via `Stripe-Version` header or account setting |
| **Routable per-version** | Trivial (different services per `/v1`/`/v2`) | Possible, more complex | Single service, code branches by version |
| **Granularity** | Coarse — one version per major change | Fine — content-negotiation per resource | Finest — per-date, per-behavior |
| **Implementation cost** | Lowest | Middle | Highest — must maintain N concurrent behaviors |
| **Client migration friction** | High — new code paths for new version | Low — just change the header | None — clients don't migrate, they opt-in to new versions |
| **Supports gradual rollout** | Hard — all-or-nothing per client | Yes via `Accept` negotiation | Yes natively |
| **Used by** | Stripe public docs (`/v1`), GitHub REST (`/v3`), Twilio (`/2010-04-01`) | Some hypermedia APIs, GitHub Media Types | Stripe (the pinned version system) |

**Default recommendation:** URL versioning for public APIs unless you have a clear reason for one of the others. Pair it with strict additive-only evolution within each major version, and use `Deprecation` / `Sunset` headers when you do need to retire something.

## Code Examples

### Cursor-Based Pagination Endpoint

A minimal but correct cursor implementation. The cursor encodes the last-seen sort key, sort direction is fixed, the underlying index is on `(created_at, id)`:

```typescript
// GET /orders?limit=50&cursor=...
import { z } from "zod";

const CursorSchema = z.object({
  created_at: z.string().datetime(),
  id: z.string(),
});

function encodeCursor(c: z.infer<typeof CursorSchema>): string {
  return Buffer.from(JSON.stringify(c)).toString("base64url");
}

function decodeCursor(s: string): z.infer<typeof CursorSchema> {
  const json = Buffer.from(s, "base64url").toString();
  return CursorSchema.parse(JSON.parse(json));
}

export async function listOrders(req: Request, res: Response) {
  const limit = Math.min(Number(req.query.limit) || 50, 100);
  const cursor = req.query.cursor
    ? decodeCursor(String(req.query.cursor))
    : null;

  // Fetch limit + 1 to know if there's a next page without a separate count.
  const rows = await db.query<Order>(
    `
    SELECT id, created_at, status, total_cents
    FROM orders
    WHERE tenant_id = $1
      AND ($2::timestamptz IS NULL
           OR (created_at, id) < ($2, $3))
    ORDER BY created_at DESC, id DESC
    LIMIT $4
    `,
    [req.auth.tenantId, cursor?.created_at ?? null, cursor?.id ?? null, limit + 1],
  );

  const hasMore = rows.length > limit;
  const data = hasMore ? rows.slice(0, limit) : rows;
  const nextCursor =
    hasMore && data.length > 0
      ? encodeCursor({
          created_at: data[data.length - 1].created_at,
          id: data[data.length - 1].id,
        })
      : null;

  res.json({
    data,
    next_cursor: nextCursor,
    has_more: hasMore,
  });
}
```

Notes:

- `LIMIT limit + 1` lets the server detect "is there a next page" without a separate `COUNT(*)`.
- Tuple comparison `(created_at, id) < ($2, $3)` correctly handles ties on `created_at` by breaking them on `id`.
- The cursor is opaque (`base64url` of JSON). The server can change the encoding later without breaking clients — they just round-trip whatever opaque string they got.
- No `total` field — by design. Clients that need a count run a separate aggregate endpoint.

### RFC 7807 Problem Details Error Response

Server-side helper plus example responses:

```typescript
type Problem = {
  type: string;
  title: string;
  status: number;
  detail?: string;
  instance?: string;
  // Extensions
  errors?: Array<{ field: string; code: string; message?: string }>;
  request_id?: string;
};

const PROBLEM_BASE = "https://api.example.com/problems";

export function problem(
  res: Response,
  status: number,
  type: string,
  title: string,
  extras: Partial<Problem> = {},
) {
  const body: Problem = {
    type: `${PROBLEM_BASE}/${type}`,
    title,
    status,
    instance: res.req.originalUrl,
    request_id: res.locals.requestId,
    ...extras,
  };
  res.status(status).type("application/problem+json").json(body);
}

// Validation error (400 with per-field details)
problem(res, 400, "validation-error", "Your request parameters didn't validate.", {
  detail: "Some fields failed validation. See `errors` for details.",
  errors: [
    { field: "email", code: "invalid_format", message: "Must be a valid email address." },
    { field: "amount_cents", code: "out_of_range", message: "Must be > 0." },
  ],
});

// Idempotency-key reuse (422)
problem(res, 422, "idempotency-key-reused", "Idempotency key was reused with a different request.", {
  detail:
    "The Idempotency-Key '8e1a-...' was previously used for a request with a different body. " +
    "Generate a new key for the new request.",
});

// Rate limit (429)
problem(res, 429, "rate-limit-exceeded", "Too many requests.", {
  detail: "You have exceeded the per-minute rate limit for /orders. Retry after 30s.",
});
res.setHeader("Retry-After", "30");
```

Sample response on the wire:

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Your request parameters didn't validate.",
  "status": 400,
  "detail": "Some fields failed validation. See `errors` for details.",
  "instance": "/orders",
  "request_id": "req_01HW3K8B6Z4QF2",
  "errors": [
    {"field": "email", "code": "invalid_format", "message": "Must be a valid email address."},
    {"field": "amount_cents", "code": "out_of_range", "message": "Must be > 0."}
  ]
}
```

### `Idempotency-Key` Server-Side Handling

Pseudocode for a payment endpoint with retries handled correctly. The state machine matters more than the language:

```typescript
// POST /charges with Idempotency-Key header
export async function createCharge(req: Request, res: Response) {
  const key = req.header("Idempotency-Key");
  if (!key) {
    return problem(res, 400, "missing-idempotency-key",
      "POST /charges requires an Idempotency-Key header.");
  }

  const reqHash = canonicalHash(req.body); // sorted-keys SHA256
  const scope = `${req.auth.tenantId}:charges`;

  // Atomic: insert if not exists, else fetch existing.
  const existing = await idempotencyStore.acquireOrGet({
    scope,
    key,
    request_hash: reqHash,
    ttl_seconds: 24 * 60 * 60,
  });

  if (existing.kind === "in_progress") {
    // Another worker is processing the exact same request right now.
    return problem(res, 409, "request-in-progress",
      "A previous request with this Idempotency-Key is still being processed. Retry shortly.");
  }

  if (existing.kind === "completed") {
    if (existing.request_hash !== reqHash) {
      // Same key, different body — client bug.
      return problem(res, 422, "idempotency-key-reused",
        "Idempotency key was reused with a different request body.");
    }
    // Replay stored response byte-identically.
    res.status(existing.response_status);
    for (const [k, v] of Object.entries(existing.response_headers)) {
      res.setHeader(k, v);
    }
    return res.json(existing.response_body);
  }

  // existing.kind === "acquired" — we own the key, do the work.
  try {
    const charge = await chargeService.create(req.body, { idempotencyKey: key });
    const responseBody = serializeCharge(charge);

    await idempotencyStore.complete({
      scope,
      key,
      response_status: 201,
      response_headers: { Location: `/charges/${charge.id}` },
      response_body: responseBody,
    });

    res.status(201).setHeader("Location", `/charges/${charge.id}`).json(responseBody);
  } catch (err) {
    // On non-retryable errors (e.g. card_declined), persist the error response too —
    // the client should get the same decline on retry, not a fresh attempt.
    if (isDeterministic(err)) {
      const errBody = toProblem(err);
      await idempotencyStore.complete({
        scope, key,
        response_status: errBody.status,
        response_headers: { "content-type": "application/problem+json" },
        response_body: errBody,
      });
      res.status(errBody.status).type("application/problem+json").json(errBody);
    } else {
      // Transient — release the key so the client can retry and get a fresh attempt.
      await idempotencyStore.release({ scope, key });
      throw err;
    }
  }
}
```

Subtleties this captures:

- **The deterministic-vs-transient distinction** — a card decline must be cached so the client doesn't get a different answer on retry; a database timeout must release the key so the retry has a chance.
- **In-flight protection** — two simultaneous retries don't both call the payment processor.
- **Body-hash verification** — same key with different body is an explicit error, not a silent replay.
- **Bounded TTL** — keys expire (Stripe: 24h), bounding storage growth.

## Real-World Uses

**Stripe API** — The reference implementation of "REST done well at scale." Uses URL versioning (`/v1/`), evolutionary versioning via account-pinned `Stripe-Version` dates, opaque prefixed IDs (`ch_`, `cu_`, `pi_`), cursor pagination, RFC 7807-shaped errors (slightly pre-RFC, but the same idea), and pioneered the `Idempotency-Key` pattern for payments. Read [stripe.com/docs/api](https://stripe.com/docs/api) — it's the closest thing to a textbook for production REST design.

**GitHub REST API v3 → GraphQL v4** — GitHub's REST API used `/v3/` URL versioning, custom media-type headers for previews (`Accept: application/vnd.github.machine-man-preview+json`), and an explicit deprecation calendar. Their migration to GraphQL v4 is a useful case study in "when REST stops being the right answer" — primarily when clients need fine-grained, joined data and the N+1 round-trip cost dominates. They didn't drop REST; they offer both, scope-bounded.

**Twilio API** — Uses date-based URL versioning (`/2010-04-01/`) — an unusual but defensible choice. Heavy use of `Idempotency-Key` (called `Idempotency-Token` in their docs) for messaging endpoints, where double-sending an SMS has direct customer-money cost. Pagination is page-based with `next_page_uri` returned as a fully-formed URL, which is mildly hypermedia-ish without going full HATEOAS.

**GitHub, Stripe, and Slack rate-limit headers** — All three publish `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` (or the IETF `RateLimit-*` variant). Slack additionally returns `Retry-After` on 429s and documents the per-method buckets explicitly.

**OpenAPI in industry** — Used by AWS (via Smithy, which compiles to OpenAPI), Microsoft Azure, Google Cloud, Stripe (internally for SDK generation), and basically every fintech and SaaS that ships a public API. The spec is the contract; the SDKs are generated; the docs are generated; the mocks are generated.

## Anti-Patterns

- **Verbs in URLs.** `POST /getUser`, `POST /createOrder`, `GET /searchInvoices`. You lose method semantics, you lose cacheability, you lose clarity, and you signal to every reviewer that the team didn't read the chapter. Use `GET /users/:id`, `POST /orders`, `GET /invoices?q=...`.

- **Breaking changes without a version bump.** Removing a field, renaming a field, tightening validation, changing pagination shape — all of these break clients and must be deferred to a version that opts in. The fastest way to lose enterprise customers is to ship "fixes" that break their integrations on a Tuesday.

- **Integer auto-increment IDs in public APIs.** `customer_id=8421` leaks volume and ordering. Use opaque prefixed IDs (`cus_3OabcDEF`) — readable, unguessable, scoped.

- **No idempotency on payment / messaging / order endpoints.** A client retry on a flaky network double-charges the customer, double-sends the SMS, double-creates the order. `Idempotency-Key` is not optional on any endpoint with money or external side effects.

- **Returning 200 with `{success: false}`.** The HTTP status code is the structured signal; the body is the unstructured one. Clients, proxies, monitoring, and SDKs all expect status codes to be truthful. Use `4xx` for client errors, `5xx` for server errors. Always.

- **Offset pagination on tables that grow unbounded.** Works fine until the user with 1.2M rows hits page 5000 and your database falls over. Switch to cursor before it bites you, not after.

- **Hand-written error responses in 17 different shapes.** Centralize on RFC 7807. Every endpoint returns the same envelope. Clients write one parser instead of seventeen.

- **Authentication via query string (`?api_key=...`).** Logged everywhere, leaks into Referer headers, easy to screenshot. Use `Authorization: Bearer ...` headers.

- **Mixing concerns in one endpoint.** `POST /users` that takes `{action: "create" | "update" | "delete", ...}` is RPC pretending to be REST. Use `POST /users`, `PATCH /users/:id`, `DELETE /users/:id`.

- **No deprecation policy.** "We'll just remove it when we feel like it" is not a policy. Publish a timeline (minimum 6 months for public APIs), return `Deprecation` and `Sunset` headers, monitor usage, contact integrators directly. Build trust by being predictable.

- **HATEOAS theatre.** Adding `_links` to every response because the textbook said to, while no client uses them, while still requiring out-of-band OpenAPI docs to actually integrate. Either commit to hypermedia all the way (state-driven clients, generic browsers) or skip it; the half-measure is pure cost.

- **Overloading `PATCH` semantics.** "We accept both JSON Merge Patch and JSON Patch and try to detect which" — pick one, document it, stick to it. Detection is brittle and the failure modes are silent data corruption.

## Related

- [./api-design-styles.md](./api-design-styles.md) — REST vs gRPC vs GraphQL vs WebSockets at the architectural level; this doc zooms into the REST half.
- [../communication/idempotency-and-exactly-once.md](../communication/idempotency-and-exactly-once.md) — the deeper story on idempotency: API-layer keys are only the front door; downstream effects (charging, emailing, queueing) need their own semantics.
- [../case-studies/distributed-infra/design-api-gateway.md](../case-studies/distributed-infra/design-api-gateway.md) — where rate limiting, auth, request shaping, and version routing actually live in front of a REST API.
- [./rate-limiters.md](./rate-limiters.md) — token bucket, sliding window, and the algorithms behind the headers in this doc.
- [./api-gateways-and-bff.md](./api-gateways-and-bff.md) — the layer that stitches multiple REST services into one client-facing API surface.
- [../foundations/cap-and-consistency-models.md](../foundations/cap-and-consistency-models.md) — the consistency model your `GET` returns under matters more than the URL shape.

## References

- IETF RFC 9457, ["Problem Details for HTTP APIs"](https://www.rfc-editor.org/rfc/rfc9457) (2023) — the current standard for error envelopes, obsoletes RFC 7807 with the same shape.
- IETF RFC 7807, ["Problem Details for HTTP APIs"](https://www.rfc-editor.org/rfc/rfc7807) (2016) — the original, still cited by most APIs in the wild.
- IETF RFC 9110, ["HTTP Semantics"](https://www.rfc-editor.org/rfc/rfc9110) — the authoritative source for method semantics, status codes, conditional requests, caching headers. Replaces RFC 7230–7235.
- IETF RFC 5789, ["PATCH Method for HTTP"](https://www.rfc-editor.org/rfc/rfc5789); RFC 6902, ["JSON Patch"](https://www.rfc-editor.org/rfc/rfc6902); RFC 7396, ["JSON Merge Patch"](https://www.rfc-editor.org/rfc/rfc7396) — the three docs to read together when designing partial-update endpoints.
- IETF RFC 8594, ["The Sunset HTTP Header Field"](https://www.rfc-editor.org/rfc/rfc8594), and the [`Deprecation` HTTP header draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-deprecation-header/) — for communicating deprecation timelines on the wire.
- [Stripe API Reference](https://stripe.com/docs/api) and [Stripe API Versioning](https://stripe.com/docs/upgrades) — the gold-standard production REST API and the clearest writeup of evolutionary versioning.
- [GitHub REST API v3 docs](https://docs.github.com/en/rest) and [GitHub GraphQL v4 docs](https://docs.github.com/en/graphql) — useful side-by-side for understanding when REST vs GraphQL is the right tool.
- [OpenAPI Specification 3.1](https://spec.openapis.org/oas/v3.1.0) and [JSON Schema 2020-12](https://json-schema.org/specification.html) — the contract languages you should be writing your APIs in.
- Roy Fielding, ["Architectural Styles and the Design of Network-based Software Architectures"](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm) (2000), Chapter 5 — the original REST dissertation. Read it once for vocabulary; do not take HATEOAS as gospel for production.
- Mark Nottingham, ["Web API Versioning"](https://www.mnot.net/blog/2012/12/04/api-evolution) and ["Evolving HTTP APIs"](https://www.mnot.net/blog/2012/12/04/api-evolution) — the canonical pragmatic case for evolutionary versioning over big-bang versioning.
- ["API Stylebook"](http://apistylebook.com/) — collected guidelines from Microsoft, Google, PayPal, Cisco, Heroku, and others. Useful when an internal team wants to argue about a convention; point them at four other companies that already chose.
