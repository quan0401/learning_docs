---
title: "API Design Styles — REST, GraphQL, gRPC"
date: 2026-04-26
updated: 2026-04-26
tags: [system-design, api, rest, graphql, grpc, protobuf]
---

# API Design Styles — REST, GraphQL, gRPC

**Date:** 2026-04-26 | **Updated:** 2026-04-26
**Tags:** `system-design` `api` `rest` `graphql` `grpc` `protobuf`

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [Key Concepts](#key-concepts)
  - [REST — Resource-Oriented HTTP](#rest--resource-oriented-http)
  - [GraphQL — Client-Driven Query Language](#graphql--client-driven-query-language)
  - [gRPC — Protobuf + HTTP/2 Streaming](#grpc--protobuf--http2-streaming)
  - [Data Formats — JSON, Protobuf, Avro, MessagePack](#data-formats--json-protobuf-avro-messagepack)
  - [Versioning Strategies](#versioning-strategies)
- [Trade-offs Matrix — Style × Use Case](#trade-offs-matrix--style--use-case)
- [Code Examples](#code-examples)
  - [REST Endpoint (Spring Boot)](#rest-endpoint-spring-boot)
  - [GraphQL Schema, Query, and Resolver](#graphql-schema-query-and-resolver)
  - [gRPC Proto and Service](#grpc-proto-and-service)
- [Real-World Uses](#real-world-uses)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

Three styles dominate modern service APIs: **REST** (resource-oriented, HTTP verbs over JSON, ubiquitous on the public web), **GraphQL** (a single endpoint where the client picks fields via a query language, ideal for heterogeneous clients), and **gRPC** (Protobuf-encoded RPC over HTTP/2 with bidirectional streams and cross-language code-gen, dominant in internal microservice meshes). The choice is rarely "which is best" — it's "which style matches this audience, this network, and this evolution model." Layer onto that **data formats** (JSON for human eyes, Protobuf for tight RPCs, Avro for streaming with schema registry, MessagePack as binary JSON) and a **versioning strategy** (URL prefix, header, accept-version, or evolutionary), and you have the full design space.

## Overview

When a service needs to expose behavior to another process, the bytes on the wire and the contract above them are what we call its **API style**. The style determines:

- How the contract is described (OpenAPI, GraphQL SDL, `.proto`).
- How clients discover what's available (HATEOAS, introspection, server reflection).
- How the wire format is encoded (text JSON vs binary Protobuf/Avro/MessagePack).
- How the network layer is used (HTTP/1.1 request-response vs HTTP/2 streams).
- How the API evolves over time without breaking clients.

REST emerged from Roy Fielding's 2000 dissertation as a constraint set on top of HTTP — it is not a protocol, it's an architectural style. GraphQL was open-sourced by Facebook in 2015 to solve mobile client over-fetching. gRPC was open-sourced by Google in 2016 as the public-facing successor to internal Stubby. Each carries the fingerprints of the problem it was designed to solve, and that's why the same style does not fit every shape of system.

The trap is treating these as interchangeable. A "REST API with verbs in the URL" is REST in name only. A GraphQL endpoint without DataLoader is a database self-DDoS. A gRPC service exposed directly to browsers without a translator (gRPC-Web, Envoy, Connect) is unreachable. Each style has a happy path; the rest of this doc is about staying on it.

## Key Concepts

### REST — Resource-Oriented HTTP

**Core idea.** Model the domain as **resources** identified by URLs, then use the HTTP verb to express the operation. State transitions are uniform across resource types.

The Fielding constraints (the academic definition):

| Constraint | What it means in practice |
|------------|---------------------------|
| **Client-server** | Separate UI from data; clients can evolve independently. |
| **Stateless** | Every request carries its own auth and context; no server-side session affinity required. |
| **Cacheable** | Responses declare cacheability via `Cache-Control`/`ETag`; intermediaries can cache. |
| **Uniform interface** | Same verbs (`GET`/`POST`/`PUT`/`PATCH`/`DELETE`) across all resources, with consistent semantics. |
| **Layered system** | Proxies, gateways, CDNs can sit transparently between client and origin. |
| **Code-on-demand** _(optional)_ | Server can ship executable code (rarely used). |
| **HATEOAS** _(the one most ignore)_ | Responses contain links to next available actions; clients navigate by following links, not by hardcoding URLs. |

In real production "REST" almost always means **resource-oriented JSON over HTTP** with sensible status codes and predictable URL shapes. Strict HATEOAS is rare outside of HAL-/JSON-API-/Siren-style hypermedia formats.

**Verb semantics that production code actually relies on:**

- `GET` — safe and idempotent. Cache-friendly. No body in request (typically).
- `POST` — neither safe nor idempotent. Used for creates and for "actions that don't fit a verb."
- `PUT` — idempotent full replacement. `PUT` of the same body N times equals one `PUT`.
- `PATCH` — partial update. RFC 7396 (JSON Merge Patch) or RFC 6902 (JSON Patch) define the body shape.
- `DELETE` — idempotent. Re-deleting a deleted resource should still return 2xx or 404, not 500.

**Status codes that matter:**

| Code | Meaning | When to use |
|------|---------|-------------|
| 200 | OK | Successful read or update with body |
| 201 | Created | Resource created; include `Location` header |
| 202 | Accepted | Async work queued; client should poll |
| 204 | No Content | Success, no body (e.g., DELETE) |
| 400 | Bad Request | Client-side validation failure |
| 401 | Unauthorized | Missing/invalid credentials |
| 403 | Forbidden | Authenticated but not allowed |
| 404 | Not Found | Resource does not exist |
| 409 | Conflict | State conflict (duplicate key, version mismatch) |
| 422 | Unprocessable Entity | Validation errors with body details |
| 429 | Too Many Requests | Rate-limit exceeded; include `Retry-After` |
| 5xx | Server-side errors | Never leak stack traces; log internally |

**Strengths.** Universally understood, debuggable with `curl`, cacheable by CDNs/proxies, browser-native, leverages decades of HTTP tooling (load balancers, WAFs, OpenAPI generators, Postman).

**Weaknesses.** No formal schema unless you bolt on OpenAPI. Versioning is conventional, not enforced. Easy to over-fetch or under-fetch (return-the-whole-user vs N+1 round-trips). Bidirectional streaming is awkward (SSE or WebSockets, both outside REST).

See the resource-shaping deep dive in [REST API Design In Depth](./rest-api-design-in-depth.md).

### GraphQL — Client-Driven Query Language

**Core idea.** A single endpoint (`POST /graphql`) accepts a query in a strongly-typed query language; the server returns exactly the fields the client asked for, no more, no less. The schema is the contract.

```graphql
# Schema (SDL)
type User {
  id: ID!
  name: String!
  email: String!
  orders(limit: Int = 10): [Order!]!
}

type Order {
  id: ID!
  total: Money!
  items: [OrderItem!]!
}

type Query {
  user(id: ID!): User
}
```

```graphql
# Client query — picks only what it needs
query MobileHome($id: ID!) {
  user(id: $id) {
    name
    orders(limit: 5) { id total { amount currency } }
  }
}
```

**Why it exists.** Mobile clients on slow networks needed exactly the data for a screen, not the whole user object plus a separate orders fetch plus a separate items fetch. GraphQL collapses N round-trips into one and lets the client decide the shape.

**Operation types:**

- `query` — read (idempotent, cacheable per-field).
- `mutation` — write (analogous to `POST`/`PUT`/`PATCH`/`DELETE`).
- `subscription` — push (typically over WebSocket or SSE).

**The N+1 problem and DataLoader.** A naive resolver that runs a SQL query per parent field will produce one query for the parent and one per child:

```text
query { users { name orders { total } } }
# Naive resolver:
SELECT * FROM users;             -- 1
SELECT * FROM orders WHERE user_id = 1;  -- N
SELECT * FROM orders WHERE user_id = 2;
...
```

**DataLoader** (the canonical pattern, originally Facebook's `dataloader` JS library) batches calls inside a single tick of the event loop and dedupes by key:

```text
loader.load(1) ─┐
loader.load(2) ─┼─► batchFn([1,2,3]) → SELECT * FROM orders WHERE user_id IN (1,2,3)
loader.load(3) ─┘
```

Every list-bearing field on a GraphQL resolver should be backed by a per-request DataLoader (or its language equivalent: `graphql-java`'s `DataLoader`, Hot Chocolate's `DataLoader<TKey, TValue>`, etc.). Shipping a GraphQL service without batching is the most common GraphQL outage.

**Schema federation (Apollo Federation v2).** A federated graph composes one supergraph from many subgraphs owned by different teams:

```text
       ┌─── users-subgraph (User, owns id/name/email)
gateway┤
       ├─── orders-subgraph (Order, extends User with .orders)
       └─── catalog-subgraph (Product, extends Order with item details)
```

Each subgraph declares which entities it owns and which entity fields it extends. The gateway plans a query across subgraphs and fans out. Federation v2 (`@key`, `@external`, `@requires`, `@provides`) is the de facto large-org pattern; alternatives include schema stitching (legacy) and single-graph monolith (works fine for smaller teams).

**Strengths.** No over- or under-fetching. Single round-trip for nested data. Strongly typed schema with introspection. Rich tooling (Apollo Studio, GraphiQL, code generation across languages).

**Weaknesses.** Caching is harder — `POST /graphql` is not CDN-cacheable by default; you need persisted queries or `@defer`/`@stream` plus query-aware caches. File uploads and binary blobs are awkward. Authorization is per-field, which is more powerful but also more places to forget. Schema design mistakes (cyclic types, unbounded lists) are easy to make and hard to undo. Without query-cost analysis you have a self-inflicted DDoS surface (`{ users { friends { friends { friends ... } } } }`).

### gRPC — Protobuf + HTTP/2 Streaming

**Core idea.** Define services and messages in `.proto` files, code-gen typed clients and servers in any language, talk over HTTP/2 with binary Protobuf payloads.

```proto
syntax = "proto3";
package commerce.v1;

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc StreamOrderEvents(StreamOrderEventsRequest) returns (stream OrderEvent);  // server stream
  rpc UploadOrderBatch(stream OrderInput) returns (UploadSummary);              // client stream
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);                    // bidirectional
}
```

**Four call patterns** flow naturally from HTTP/2 streams:

| Pattern | Wire shape | Use case |
|---------|-----------|----------|
| **Unary** | one request → one response | Standard RPC: `GetOrder`, `CreateOrder` |
| **Server streaming** | one request → stream of responses | Server pushes (price ticks, log tail, search-as-you-type) |
| **Client streaming** | stream of requests → one response | Bulk upload, telemetry batch ingest |
| **Bidirectional streaming** | stream ↔ stream | Chat, collaborative editing, voice transcription |

**Protobuf wire format.** Tag-length-value binary; numeric field numbers on the wire (not field names), so renaming a field is free, but reusing or reordering tag numbers is a breaking change. Optional/required/default rules are baked into the encoding. Roughly 3–10× smaller than the equivalent JSON, and 2–10× faster to parse depending on language.

**Server reflection.** Servers can expose their `.proto` definitions at runtime via the reflection service so that tools like `grpcurl`, BloomRPC, Postman, or Buf Studio can introspect the service without prior `.proto` access. Useful in dev, often disabled in prod for attack-surface reasons.

**Errors.** gRPC has its own status code space (16 codes: `OK`, `CANCELLED`, `UNKNOWN`, `INVALID_ARGUMENT`, `DEADLINE_EXCEEDED`, `NOT_FOUND`, `ALREADY_EXISTS`, `PERMISSION_DENIED`, `RESOURCE_EXHAUSTED`, `FAILED_PRECONDITION`, `ABORTED`, `OUT_OF_RANGE`, `UNIMPLEMENTED`, `INTERNAL`, `UNAVAILABLE`, `DATA_LOSS`, `UNAUTHENTICATED`). Don't try to map these to HTTP codes 1:1 — they're a richer space tailored to RPC semantics.

**Deadlines.** Every gRPC call carries a deadline propagated through the call chain. A 5-second deadline at the edge becomes ~4.5s at the next hop, ~4s at the hop after, and so on. Services that ignore the deadline and keep working after the client has given up are wasting the cluster's CPU. Treat deadlines as first-class.

**Strengths.** Compact wire, high throughput, true streaming, language-neutral code-gen, tight schema. Excellent fit for east-west service-to-service traffic where both ends ship together.

**Weaknesses.** Browsers cannot speak gRPC natively — you need **gRPC-Web** with a proxy (Envoy, grpc-web-proxy, Connect). Public-internet exposure is awkward (binary, opaque to most WAFs/CDNs, hard to debug without `grpcurl`). HTTP/2 dependency means HTTP/1.1-only intermediaries can't proxy it. Backwards-compatible schema evolution requires discipline (never change a field number, never change a type, only add fields).

### Data Formats — JSON, Protobuf, Avro, MessagePack

The wire format is partly orthogonal to the API style. REST usually rides JSON but can ride Protobuf or MessagePack; gRPC defaults to Protobuf but supports JSON; Avro is mostly seen on Kafka and other streaming pipes.

| Format | Encoding | Schema | Size vs JSON | Read/parse speed | Best for |
|--------|----------|--------|--------------|------------------|----------|
| **JSON** | Text (UTF-8) | Optional (JSON Schema, OpenAPI) | 1.0× (baseline) | Slow (text parse) | Public APIs, browser, debugging |
| **Protobuf** | Binary, tag-length-value | Required (`.proto`) | 0.2–0.3× | Fast (zero-copy in some langs) | gRPC, internal services |
| **Avro** | Binary, schema-resolved | Required (registry) | 0.3–0.5× | Fast | Streaming (Kafka), data lake, event sourcing |
| **MessagePack** | Binary, JSON-like model | Optional | 0.5–0.7× | Fast | Mobile, embedded, Redis values |

**Avro and the schema registry.** Avro's superpower is **schema evolution** mediated by a central registry (Confluent Schema Registry is the canonical one). Producers register a schema and tag every message with a schema ID; consumers fetch the schema by ID and resolve fields by name (not by tag like Protobuf). This makes adding/removing/renaming fields safe across producers and consumers that ship at different times — exactly the problem you have with long-lived Kafka topics.

**Protobuf vs Avro for events.** A common production pattern:

- **Protobuf** for synchronous RPC where producer and consumer ship roughly together.
- **Avro** for asynchronous event streams where producer and consumer evolve independently and you need strong replay/forward-compat guarantees.

**MessagePack.** Think "JSON, but binary." Same data model (objects, arrays, numbers, strings, booleans, null), but tighter and faster to parse. Often used as a Redis value format or as a mobile-API format where you want JSON's flexibility without the byte tax.

### Versioning Strategies

APIs evolve. The four common approaches:

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **URL path** | `/v1/users`, `/v2/users` | Visible, cache-friendly, easy to A/B route | URL changes = consumer churn; "v" in the URL is arguably non-RESTful |
| **Header / Accept** | `Accept: application/vnd.api.v2+json` | Clean URLs, content-negotiation native | Less visible, cache key needs `Vary: Accept`, harder to test in browser |
| **Query param** | `/users?api-version=2` | Easy to test | Mixes versioning into business params; CDN cache awkward |
| **Evolutionary (no version)** | One endpoint, only additive changes; never remove or repurpose fields | No version sprawl | Requires discipline; hard to deprecate; bad changes cannot be undone |

**The pragmatic stance.** Public APIs almost always end up with URL-path versioning because consumers are external and need a visible signal. Internal APIs (especially gRPC) lean **evolutionary** — Protobuf field-number rules make additive evolution a first-class concept, and ops becomes "never break wire compatibility" rather than "ship v1/v2/v3."

GraphQL takes a third route: **schema evolution with `@deprecated`**. There is conventionally no v1/v2 GraphQL endpoint; you add new fields, mark old ones `@deprecated(reason: "...")`, and remove them after telemetry shows zero usage.

```graphql
type User {
  fullName: String! @deprecated(reason: "Use firstName + lastName")
  firstName: String!
  lastName: String!
}
```

## Trade-offs Matrix — Style × Use Case

When the use case looks like... reach for...

| Use case | REST | GraphQL | gRPC |
|----------|:----:|:-------:|:----:|
| Public API for third-party developers | **Best** | Good | Bad (binary, browser-unfriendly) |
| Internal microservice ↔ microservice | OK | OK | **Best** |
| Mobile client with heterogeneous screens | OK | **Best** | Good (with gRPC-Web) |
| Bandwidth-constrained mobile | Bad (JSON bloat) | Good | **Best** (Protobuf) |
| Server pushes events to client | Bad (poll/SSE) | Good (subscriptions) | **Best** (server streaming) |
| Bidirectional streaming (chat, voice) | Bad | Good (subscriptions, awkward) | **Best** |
| Browser app (no proxy budget) | **Best** | **Best** | Bad (needs gRPC-Web proxy) |
| CDN-cacheable reads | **Best** | Hard (POST) | Bad (binary) |
| Strict cross-language type safety | Weak (OpenAPI) | Strong (SDL) | **Strongest** (`.proto`) |
| Analytics / event-sourced pipeline | OK (JSON) | Bad | Good (Protobuf) — but Avro often wins here |
| Aggregate-from-many-services for a UI | Hard (BFF needed) | **Best** (federation) | Hard |

## Code Examples

### REST Endpoint (Spring Boot)

```java
@RestController
@RequestMapping("/v1/orders")
@RequiredArgsConstructor
public class OrderController {
    private final OrderService orderService;

    @GetMapping("/{id}")
    public ResponseEntity<OrderDto> get(@PathVariable UUID id) {
        return orderService.findById(id)
            .map(o -> ResponseEntity.ok()
                .eTag(o.version().toString())
                .cacheControl(CacheControl.maxAge(60, TimeUnit.SECONDS))
                .body(OrderDto.from(o)))
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<OrderDto> create(
            @Valid @RequestBody CreateOrderRequest req,
            @RequestHeader("Idempotency-Key") String idemKey,
            UriComponentsBuilder uri) {
        Order created = orderService.create(req, idemKey);
        return ResponseEntity
            .created(uri.path("/v1/orders/{id}").buildAndExpand(created.id()).toUri())
            .body(OrderDto.from(created));
    }

    @PatchMapping("/{id}")
    public OrderDto patch(@PathVariable UUID id,
                          @RequestBody JsonMergePatch patch,
                          @RequestHeader("If-Match") String ifMatch) {
        return OrderDto.from(orderService.patch(id, patch, Long.parseLong(ifMatch)));
    }

    @ExceptionHandler(OptimisticLockException.class)
    public ResponseEntity<ErrorBody> conflict(OptimisticLockException e) {
        return ResponseEntity.status(409).body(new ErrorBody("version_conflict", e.getMessage()));
    }
}
```

Note the small touches: `ETag` for caching and concurrency, `Idempotency-Key` header for safe retries on `POST`, `If-Match` for optimistic concurrency on `PATCH`, status codes mapped to error semantics.

### GraphQL Schema, Query, and Resolver

```graphql
# schema.graphql
type Query {
  order(id: ID!): Order
  ordersByUser(userId: ID!, first: Int = 20, after: String): OrderConnection!
}

type Order {
  id: ID!
  user: User!
  items: [OrderItem!]!
  total: Money!
  status: OrderStatus!
}

type OrderConnection {
  edges: [OrderEdge!]!
  pageInfo: PageInfo!
}

enum OrderStatus { PENDING PAID SHIPPED DELIVERED CANCELLED }
```

```javascript
// resolvers.js — DataLoader-backed resolver in Node/Apollo
import DataLoader from 'dataloader';

export function buildLoaders(db) {
  return {
    userById: new DataLoader(async (ids) => {
      const rows = await db.users.where('id', 'in', ids);
      const byId = new Map(rows.map((r) => [r.id, r]));
      return ids.map((id) => byId.get(id) ?? null);
    }),
    itemsByOrderId: new DataLoader(async (orderIds) => {
      const rows = await db.orderItems.where('order_id', 'in', orderIds);
      const grouped = new Map(orderIds.map((id) => [id, []]));
      for (const r of rows) grouped.get(r.order_id).push(r);
      return orderIds.map((id) => grouped.get(id));
    }),
  };
}

export const resolvers = {
  Query: {
    order: (_, { id }, ctx) => ctx.db.orders.findById(id),
  },
  Order: {
    user:  (parent, _, ctx) => ctx.loaders.userById.load(parent.userId),
    items: (parent, _, ctx) => ctx.loaders.itemsByOrderId.load(parent.id),
  },
};
```

The loaders are **per-request** (`buildLoaders` runs in the Apollo `context` callback) so cache and batching are scoped to one query and never leak across users.

### gRPC Proto and Service

```proto
// proto/commerce/v1/orders.proto
syntax = "proto3";
package commerce.v1;

import "google/protobuf/timestamp.proto";

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  rpc StreamOrderEvents(StreamOrderEventsRequest) returns (stream OrderEvent);
}

message GetOrderRequest {
  string id = 1;
}

message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItemInput items = 2;
  string idempotency_key = 3;
}

message Order {
  string id = 1;
  string user_id = 2;
  repeated OrderItem items = 3;
  Money total = 4;
  OrderStatus status = 5;
  google.protobuf.Timestamp created_at = 6;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_PAID = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}
```

```java
// Java server impl (Spring Boot + grpc-spring-boot-starter)
@GrpcService
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {
    private final OrderService orders;

    @Override
    public void getOrder(GetOrderRequest req, StreamObserver<Order> resp) {
        orders.findById(UUID.fromString(req.getId()))
            .ifPresentOrElse(
                o -> { resp.onNext(toProto(o)); resp.onCompleted(); },
                () -> resp.onError(Status.NOT_FOUND
                        .withDescription("order " + req.getId())
                        .asRuntimeException())
            );
    }

    @Override
    public void streamOrderEvents(StreamOrderEventsRequest req,
                                   StreamObserver<OrderEvent> resp) {
        var sub = orders.subscribe(req.getUserId(), evt -> resp.onNext(toProto(evt)));
        Context.current().addListener(ctx -> sub.cancel(), Runnable::run);
    }
}
```

Note `Status.NOT_FOUND` (gRPC status), the `StreamObserver` for server-streamed events, and `Context.current()` to react to client cancellation — all idiomatic gRPC.

## Real-World Uses

- **Stripe API** — REST/JSON with URL-path versioning (`/v1/`) and pinned-version-on-account semantics; one of the most-cited examples of well-disciplined REST. They publish a changelog and give every account an "API version pinned at signup" so old integrations don't break silently.
- **GitHub API** — historically REST (still primary); added a public **GraphQL v4 API** in 2016 because mobile and desktop GitHub apps needed flexible field selection across nested entities (issues → comments → reactions → users).
- **Shopify Storefront and Admin APIs** — GraphQL is the recommended path; it lets storefront themes and headless commerce clients fetch exactly the product/cart/checkout shape they render.
- **Google Cloud APIs** — define everything in `.proto` (gRPC) first, then auto-generate REST/JSON gateways via [google.api.http](https://github.com/googleapis/googleapis/blob/master/google/api/http.proto) annotations. One contract, two transports.
- **Netflix internal mesh** — gRPC for east-west, GraphQL federation (Studio Edge) for the BFF layer that fans out to dozens of subgraphs to render a TV/web UI.
- **Uber, Lyft, Square, Slack** — gRPC dominates internal service-to-service; public APIs are REST or GraphQL.
- **Confluent / Kafka ecosystem** — Avro plus the Schema Registry is the default pattern; producers and consumers evolve schemas independently, with backward/forward compatibility enforced by the registry.
- **Discord** — voice/video signalling rides gRPC streams; their realtime gateway uses WebSockets with a binary message format (Erlang Term Format, ETF) for low-latency fan-out.
- **Apollo Federation in production** — Netflix, Expedia, Wayfair, and others run federated GraphQL supergraphs as their BFF layer.

## Anti-Patterns

- **REST with verbs in the URL.** `POST /getUser`, `POST /deleteOrder`, `POST /sendEmail`. This is RPC-over-HTTP wearing a REST t-shirt. You lose every benefit (cacheability, idempotency, uniform interface) for nothing in return. The fix: `GET /users/{id}`, `DELETE /orders/{id}`, `POST /emails` (treat the email itself as a resource).
- **REST returning 200 OK with `{"error": "..."}` in the body.** Status codes exist; use them. Clients, proxies, and dashboards all key off the code.
- **GraphQL without DataLoader (or batching).** A single `query { users { orders { items } } }` becomes O(users × orders × items) database round-trips. Mandatory: every list-bearing resolver loads through a per-request DataLoader.
- **GraphQL without query cost analysis or depth limits.** A malicious client sends `{ users { friends { friends { friends ... 50 deep } } } }` and exhausts your DB. Use `graphql-cost-analysis`, depth limits, and persisted queries on public endpoints.
- **GraphQL versioning by URL.** `/graphql/v2` defeats the schema-evolution model. Use `@deprecated` and remove fields on telemetry, not by URL bump.
- **gRPC over the public internet without translation.** Browsers cannot speak gRPC natively, most WAFs cannot inspect Protobuf, debugging across the internet is painful. Either expose REST/GraphQL at the edge and gRPC behind it (typical), or front gRPC with **gRPC-Web** + Envoy and accept the operational tax.
- **Reusing or reordering Protobuf field numbers.** `int32 user_id = 3` becoming `string email = 3` is a silent corruption — old clients deserialize garbage. Field numbers are forever; reserve removed ones with `reserved 3;`.
- **No deadlines on gRPC calls.** A wedged downstream becomes an unbounded queue at the caller. Always set deadlines and propagate them.
- **Mixing schema and transport.** Treating REST and JSON as inseparable, or gRPC and Protobuf as inseparable. They're not — REST can carry Protobuf or MessagePack; gRPC can carry JSON. Pick each axis for its own reason.
- **Versioning everything ("v1/v2/v3 sprawl").** Each version is a forever-maintenance liability. Prefer evolutionary additive changes; reserve a major version bump for genuinely breaking redesigns.
- **Hand-writing GraphQL or gRPC clients.** Code-gen exists for both; using it gives you compile-time safety and saves hours of glue code.
- **One giant GraphQL schema with no ownership boundaries.** Every team mutating the same SDL = constant merge conflicts, no clear owner, mystery deprecations. Federation or stitched subgraphs scale this.

## Related

- [REST API Design In Depth](./rest-api-design-in-depth.md) — resource modeling, pagination, filtering, error envelopes, idempotency keys, and concurrency control.
- [Sync vs Async Communication](../communication/sync-vs-async-communication.md) — when an API call should be a request-response and when it should be an event you publish.
- [API Gateways and BFF](./api-gateways-and-bff.md) — the layer that often translates between public REST/GraphQL and internal gRPC, enforces auth, applies rate limiting, and aggregates per-client.
- [Designing an API Gateway (Case Study)](../case-studies/distributed-infra/design-api-gateway.md) — building the routing/auth/translation layer that sits in front of a multi-style service mesh.

## References

- Roy T. Fielding, ["Architectural Styles and the Design of Network-based Software Architectures"](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm) (PhD dissertation, 2000) — the original REST definition; chapter 5 is the canonical source.
- ["GraphQL Specification"](https://spec.graphql.org/) — current draft of the language and execution semantics.
- Lee Byron, ["GraphQL: A data query language"](https://engineering.fb.com/2015/09/14/core-infra/graphql-a-data-query-language/) (Facebook Engineering, 2015) — the announcement and the why.
- ["Apollo Federation 2 Docs"](https://www.apollographql.com/docs/federation/) — `@key`, `@external`, `@requires`, subgraph composition, and routing.
- Facebook, ["dataloader"](https://github.com/graphql/dataloader) — the canonical batching/caching pattern for resolvers.
- ["gRPC Documentation"](https://grpc.io/docs/) — concepts, four call patterns, language guides.
- ["gRPC Status Codes"](https://grpc.io/docs/guides/status-codes/) — the canonical 16-code error space.
- ["Protocol Buffers Documentation"](https://protobuf.dev/) — `.proto` language guide, wire format, and evolution rules.
- ["Apache Avro Specification"](https://avro.apache.org/docs/current/specification/) — schema evolution rules and binary encoding.
- ["Confluent Schema Registry"](https://docs.confluent.io/platform/current/schema-registry/index.html) — registry-mediated Avro/Protobuf/JSON schema evolution for Kafka.
- ["MessagePack Specification"](https://github.com/msgpack/msgpack/blob/master/spec.md) — the binary JSON-compatible format.
- ["Google API Design Guide"](https://cloud.google.com/apis/design) — Google's pragmatic guide to designing API surfaces that work as both gRPC and REST.
- Phil Sturgeon, ["APIs You Won't Hate"](https://apisyouwonthate.com/) — practical REST evolution, versioning trade-offs, hypermedia.
