---
title: "API Reverse-Engineering"
date: 2026-05-06
updated: 2026-05-06
tags: [api, mitmproxy, burp, graphql, grpc, websocket, hmac, jwt, signed-requests]
---

# API Reverse-Engineering

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `api` `mitmproxy` `burp` `graphql` `grpc` `websocket` `hmac` `jwt` `signed-requests`

---

## Table of Contents

1. The Reverse-Engineering Loop
2. Capturing Flows: DevTools, mitmproxy, Burp
3. mitmproxy Fundamentals
4. Burp Suite Fundamentals
5. mitmproxy vs Burp: When to Use Which
6. Replaying Requests
7. Signed-Request Patterns
8. Bearer Tokens and JWT
9. CSRF Tokens
10. GraphQL
11. gRPC and gRPC-Web
12. WebSockets
13. Server-Sent Events (SSE)
14. WebRTC (Brief)
15. Worked Examples
16. Detection-Resistance and the API "DRM" Problem
17. Ethics and Legality

## Summary

The browser is a legitimate API client. Anything it can fetch, you can fetch — provided you reproduce the request faithfully enough to satisfy the server's authentication, signing, and shape contracts. API reverse-engineering is the practice of watching what the browser does, isolating the minimum recipe the server actually checks, and replaying it from your scraping language. This doc covers the full toolchain: capture (DevTools, mitmproxy, Burp), replay, and the protocol-specific gotchas for HMAC-signed requests, JWT auth, CSRF, GraphQL, gRPC/gRPC-Web, WebSockets, and SSE.

The goal is not to *understand the entire backend*. The goal is to find the tightest set of headers, cookies, and body fields you must reproduce, and ignore the rest. Most of the work is observation and hypothesis-elimination, not cryptography.

---

## 1. The Reverse-Engineering Loop

The loop has four phases. Cycle them tightly:

| Phase | Question | Tool |
|-------|----------|------|
| **Observe** | What request does the browser actually send? | DevTools, mitmproxy, Burp |
| **Hypothesize** | Which fields are *required* vs decorative? | Reading + intuition |
| **Replay** | Does the request still work with my edits? | curl, Repeater, Python |
| **Automate** | Can I generate this from code? | Your scraping language |

The discipline is that you never skip "Hypothesize → Replay" before automating. A typical signed request has 30+ headers, 5+ cookies, and a body. Maybe four of those actually matter. The cheap loop is "delete a header → does it still work?" — repeat until the request stops working, then put the last thing back. This converges in a few minutes and saves hours of debugging stale request templates.

A good mental model: think of every captured request as a hypothesis space. Each header, cookie, and body field is a binary lever. You're searching for the minimal subset that the server still accepts. Most levers are no-ops.

---

## 2. Capturing Flows: DevTools, mitmproxy, Burp

Three approaches, three trade-offs:

| Tool | Mode | Scope | Best For |
|------|------|-------|----------|
| **Browser DevTools / HAR export** | Passive | Single tab | Quick lookups, sharing flows, single-request triage |
| **mitmproxy** | Active interception | System-wide | Automation, scripting, CLI workflows, recording mobile apps |
| **Burp Suite** | Active interception | Browser-routed | Hand-crafted security testing, fuzzing, manual replay |

DevTools is the cheapest and where you start. It cannot intercept and edit *before* the request is sent (you can only inspect after-the-fact, replay via "Copy as cURL", or block via Network conditions). mitmproxy and Burp sit between you and the server, see TLS-decrypted traffic, and let you mutate requests/responses live.

For mobile apps, native desktop apps, or anything outside the browser, you need mitmproxy or Burp — DevTools won't see those flows.

### HAR export

The HAR (HTTP Archive) format is a W3C-tracked JSON schema for serializing recorded network sessions. Right-click the Network panel → "Save all as HAR with content". HAR is portable: replay it in Burp, mitmproxy (`mitmproxy --read-flows`), Postman, or your own tooling. It contains request/response headers, bodies (when small enough), timing, and cookies.

Caveats:
- HAR can leak credentials. Treat exported files like secrets.
- Bodies above a threshold are sometimes elided depending on browser settings.
- Binary bodies are base64-encoded.

---

## 3. mitmproxy Fundamentals

`mitmproxy` is a CLI/TUI HTTP/HTTPS proxy with a Python addon system. Maintained at `mitmproxy.org`. Three flavors share the same engine:

- `mitmproxy` — interactive TUI
- `mitmweb` — browser-based UI
- `mitmdump` — non-interactive, scripting-friendly

### Install + certificate trust

```bash
brew install mitmproxy            # macOS
pipx install mitmproxy            # cross-platform
mitmproxy                         # first run generates ~/.mitmproxy/mitmproxy-ca-cert.pem
```

To intercept HTTPS, the client must trust the mitmproxy CA. From any browser pointed at the proxy, visit `http://mitm.it` while mitmproxy runs — the page serves the right cert format per platform. For system-wide capture, install the cert into the OS keychain (macOS) or the Android user/system store. Without trust, every TLS request shows as a connection error.

Do not install the mitmproxy CA on shared or production machines. The CA can mint a cert for any hostname. Trust is per-machine and per-user.

### Run modes

```bash
# Explicit proxy: tell the client to use 127.0.0.1:8080
mitmproxy

# Transparent: kernel iptables/pf redirects all traffic into mitmproxy
mitmproxy --mode transparent

# Upstream: chain through another proxy
mitmproxy --mode upstream:http://corp-proxy:3128

# Reverse proxy: pretend to be a server
mitmproxy --mode reverse:https://api.example.com
```

For everyday scraping work, explicit-proxy mode is the simplest: launch a browser via `chromium --proxy-server=127.0.0.1:8080` (a fresh user profile) and you see only that browser's traffic.

### Filtering flows

mitmproxy has a small filter language. Useful operators:

| Filter | Meaning |
|--------|---------|
| `~u <regex>` | Match URL |
| `~s` | Has response (filter out hanging requests) |
| `~b <regex>` | Match body content |
| `~h <regex>` | Match header (req or resp) |
| `~q` / `~rs` | Request-only / response-only |
| `~c 200` | Status code |
| `~m POST` | Method |

Filter `~u graphql ~m POST ~s` shows only completed POSTs to URLs containing "graphql". Filters compose with `&`, `|`, `!`.

### Exporting

From the TUI, press `:` then type `export.clip curl @focus` to copy the focused flow as a curl command. `:save.file @marked /tmp/flows` writes marked flows to disk; `mitmproxy --rfile /tmp/flows` replays them.

### Addons (Python scripting)

This is the killer feature. An addon is a Python module exposing hook functions:

```python
# logger.py
from mitmproxy import http

def request(flow: http.HTTPFlow) -> None:
    if "api.example.com" in flow.request.pretty_host:
        print(f"-> {flow.request.method} {flow.request.path}")
        if flow.request.headers.get("X-Signature"):
            print(f"   sig={flow.request.headers['X-Signature'][:16]}...")

def response(flow: http.HTTPFlow) -> None:
    if flow.response and flow.response.status_code >= 400:
        print(f"<- {flow.response.status_code} {flow.request.path}")

def error(flow: http.HTTPFlow) -> None:
    print(f"!! {flow.error}")
```

Run with `mitmdump -s logger.py`. Hooks run on every request/response, so you can:
- Log the X-Signature header alongside the body that produced it (great for HMAC reverse-engineering).
- Auto-rewrite a stale bearer token into every outgoing request.
- Save GraphQL operations grouped by `operationName`.
- Inject rate-limit-evading delays.

The addon system is documented at `docs.mitmproxy.org/stable/addons-overview/`.

---

## 4. Burp Suite Fundamentals

Burp is the industry-standard security-testing proxy from PortSwigger. The Community edition is free; Pro adds Intruder throttling, the active scanner, and Collaborator.

Core features used in API reverse-engineering:

| Feature | Use |
|---------|-----|
| **Proxy → Intercept** | Pause requests, edit them in-flight |
| **HTTP History** | Searchable log of everything proxied |
| **Repeater** | Hand-edited replay of a single request — the workhorse |
| **Intruder** | Parameterized fuzzing (positions + payload sets) |
| **Logger++** (extension) | Better filtering and column views than the built-in history |
| **Match-and-Replace** | Auto-edit requests/responses (e.g., always strip `Sec-CH-UA`) |
| **Collaborator** | Out-of-band server for SSRF/blind-injection testing |

Workflow for reverse-engineering an endpoint:
1. Browse the site through Burp.
2. Find the interesting request in HTTP History.
3. Right-click → Send to Repeater.
4. Strip headers/cookies one by one. Hit Send. Watch the response.
5. Once you've found the minimal recipe, copy as curl/Python and move on.

Repeater's value is the tight feedback loop — edit, send, diff response — without leaving the keyboard.

Burp documentation lives at `portswigger.net/burp/documentation`.

---

## 5. mitmproxy vs Burp: When to Use Which

Both decrypt TLS, both let you replay, both have plugin systems. Pick by ergonomics:

| Use case | mitmproxy | Burp |
|----------|-----------|------|
| Mobile app traffic | Strong (transparent mode) | Strong |
| CI/automation pipelines | Strong (mitmdump + Python) | Weak (GUI-first) |
| One-off "decode this signature" | Strong (Python addon) | Workable |
| Manual fuzzing | Workable (`mitmproxy` filters) | Strong (Intruder) |
| Hand-crafted security tests | Workable | Strong (Repeater + extensions) |
| Replaying a recorded session | Strong (`mitmdump --rfile`) | Workable (HAR import) |
| Sharing a flow with a colleague | HAR/flowfile | `.burp` project file |

Most scraping engineers settle on mitmproxy because it slots into Python pipelines. Most application-security engineers settle on Burp because Repeater + Intruder + extensions form a tight manual workflow. They are not mutually exclusive — chain Burp upstream of mitmproxy if you want both.

---

## 6. Replaying Requests

Once you've captured a request, replay is straightforward. The pattern:

1. **Copy as cURL** (Network panel right-click, or mitmproxy export).
2. **Run it**. Confirm it works *exactly as captured*.
3. **Strip non-essentials**. Delete `Sec-CH-UA-*`, `Sec-Fetch-*`, `Accept-Language`, `User-Agent`. Re-run after each delete. Stop when the response breaks.
4. **Identify dynamic fields**. What changes between two captures of the same logical action? Cookies (session, CSRF), headers (X-CSRF-Token, X-Signature, Authorization), body fields (timestamps, nonces).
5. **Translate to your scraping language**. With the minimal recipe, write the equivalent in your HTTP client of choice.

Tools to skip the typing:
- `curlconverter.com` (open source) — paste curl, get Python `requests`, Node `fetch`, Go `net/http`, etc.
- `httpie --offline` for prettier curl-equivalent debugging.

A useful TS/Node pattern: define the request as a const-typed object so headers are typed, and build replay variants from spreads:

```ts
const baseReq = {
  url: 'https://api.example.com/v2/search',
  method: 'POST' as const,
  headers: {
    'content-type': 'application/json',
    'authorization': `Bearer ${token}`,
  },
  body: { q: 'cats', page: 1 },
} satisfies HttpRequest;

const nextPage = { ...baseReq, body: { ...baseReq.body, page: 2 } };
```

This catches typos at compile time and makes the "minimal recipe" explicit in code.

---

## 7. Signed-Request Patterns

Signed requests prove the request body and metadata were authored by someone holding a secret. The signature usually lives in a header (`X-Signature`, `X-Hash`, `Authorization`). The server recomputes the signature server-side and rejects mismatches. Common variants:

### 7.1 Generic HMAC

```text
X-Signature: hmac-sha256=<hex>
X-Timestamp: 1714939200
```

The signed string is typically `timestamp + "\n" + method + "\n" + path + "\n" + body_sha256` or some near-relative. The server rejects timestamps too far from "now" (commonly ±5 minutes), so you can't capture once and replay forever.

To reverse:
1. Capture two requests to the same endpoint with different bodies. Note their signatures and timestamps.
2. Look at the frontend bundle for `crypto.subtle.sign`, `hmac`, or string concatenation around the signature header (see doc 11 — Sources panel + XHR breakpoint).
3. Hypothesize the signing string, compute it, compare to captured signature. Iterate on the format until it matches.
4. Find the secret. Often it is bootstrapped at login (a session-scoped HMAC key) or hardcoded in the JS bundle.

### 7.2 AWS Signature Version 4

A specific HMAC scheme codified by AWS, recognizable by the header:

```text
Authorization: AWS4-HMAC-SHA256 Credential=AKIA.../20260506/us-east-1/s3/aws4_request, SignedHeaders=host;x-amz-date, Signature=<hex>
```

The signing process is documented at `docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-signing.html`. SigV4 chains four HMAC operations: derive a date key, region key, service key, signing key, then HMAC the canonical request. Most language SDKs ship a SigV4 implementation; do not write it from scratch.

When you see `AWS4-HMAC-SHA256` on a non-AWS endpoint, it usually means the service uses AWS API Gateway with IAM auth or has copied the scheme. Either way, an `aws4` package in your language will sign correctly given the access key, secret key, and region.

### 7.3 Stripe-style signed webhooks

Stripe's `Stripe-Signature` header looks like:

```text
Stripe-Signature: t=1714939200,v1=<hex>,v1=<hex2>
```

Two `v1=` entries appear during key rotation — the server signs with the current key and the previous key, the client should accept either. The signed string is `t + "." + raw_body`. The verification doc lives at `docs.stripe.com/webhooks/signatures`.

This pattern is widely copied. If a header has `t=`, `v1=`, comma-separated, assume Stripe-style.

### 7.4 Custom HMAC schemes

When the scheme is bespoke, instrument the JS:
- Sources panel → Search across sources for `X-Signature` (the literal header name).
- Set an XHR breakpoint on the URL.
- Step out until you reach the function that builds the headers object.
- Look at the inputs feeding the HMAC call.
- Once the algorithm is clear, port it.

A frequent shape in modern bundles:

```js
// from a webpack chunk, prettified
async function signRequest(method, path, body, secret) {
  const ts = Math.floor(Date.now() / 1000);
  const payload = [ts, method, path, sha256(body)].join("\n");
  const sig = await hmacSha256(secret, payload);
  return { "X-Timestamp": String(ts), "X-Signature": toHex(sig) };
}
```

Port to TS:

```ts
import { createHmac, createHash } from 'node:crypto';

function signRequest(method: string, path: string, body: string, secret: string) {
  const ts = Math.floor(Date.now() / 1000);
  const bodyHash = createHash('sha256').update(body).digest('hex');
  const payload = [ts, method, path, bodyHash].join('\n');
  const sig = createHmac('sha256', secret).update(payload).digest('hex');
  return { 'x-timestamp': String(ts), 'x-signature': sig };
}
```

The hardest part is finding the secret. Sometimes it's in `localStorage` (read it once, copy it). Sometimes it's derived from a login response. Sometimes it's embedded in the bundle and rotated weekly. See doc 12 on bundle analysis.

---

## 8. Bearer Tokens and JWT

Many APIs use bog-standard `Authorization: Bearer <token>` and skip request signing. Capturing the token from the login flow gives you full access until the token expires.

### 8.1 The login-and-grab pattern

1. Run a real login through the browser (or a headless one — see doc 07).
2. Capture the response from `/login` or `/oauth/token`. It contains an `access_token` and often a `refresh_token`.
3. Drop the bearer into your scraping client.
4. When the token expires (401 with `WWW-Authenticate: Bearer error="invalid_token"`), call the refresh endpoint with `grant_type=refresh_token` to mint a new one.

### 8.2 JWT structure

JWT (RFC 7519) is `header.payload.signature`, all base64url-encoded:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NSIsImV4cCI6MTcxNDk0MjgwMH0.
n4bQgYhMfWWaL-qgxVrQFaO_TxsrC4Is0V1sFbDwCgg
```

Decode the payload — no signature verification needed for read-only claim inspection:

```ts
function decodeJwtPayload<T = Record<string, unknown>>(jwt: string): T {
  const [, payload] = jwt.split('.');
  const json = Buffer.from(payload, 'base64url').toString('utf8');
  return JSON.parse(json) as T;
}
```

Common claims to read:
- `exp` — expiration unix seconds. Refresh before this.
- `iat` — issued at.
- `sub` — subject (user id).
- `scope` / `scp` — granted permissions.
- `aud` — intended audience (often the API hostname).

Never *forge* JWTs in scraping — the server verifies the signature. Decoding is for read-only inspection of when to refresh.

For deeper JWT pitfalls (alg confusion, kid traversal, etc.) see the linked security doc.

### 8.3 Token rotation across workers

When fanning out N workers:

```ts
// Single shared token, refreshed by one coordinator
class TokenManager {
  private token: string;
  private expiresAt: number;
  private refreshing: Promise<string> | null = null;

  async getToken(): Promise<string> {
    if (Date.now() < this.expiresAt - 30_000) return this.token;
    if (this.refreshing) return this.refreshing;
    this.refreshing = this.refresh();
    const fresh = await this.refreshing;
    this.refreshing = null;
    return fresh;
  }

  private async refresh(): Promise<string> {
    // call /oauth/token with refresh_token grant
    // update this.token, this.expiresAt
    return this.token;
  }
}
```

The single-flight pattern (`refreshing: Promise | null`) prevents N workers all trying to refresh at the same time and burning the refresh token quota.

---

## 9. CSRF Tokens

CSRF tokens defend stateful POST endpoints against cross-site forgery. They appear in three places:

| Location | Pattern | How to extract |
|----------|---------|----------------|
| HTML `<meta>` tag | `<meta name="csrf-token" content="...">` | Parse with cheerio/regex |
| Cookie + header pair | `XSRF-TOKEN` cookie copied to `X-XSRF-TOKEN` header | Read cookie, send in header |
| Response body | JSON `{ csrfToken: "..." }` | Parse JSON |

The double-submit cookie pattern (cookie + header) is the most common in modern frameworks (Laravel, Django, Spring Security). The server checks that the header value equals the cookie value — defending against cross-origin forgery because the attacker site can't read the cookie.

Scraping recipe:
1. GET the page or a `/csrf` endpoint to receive the cookie.
2. Copy the cookie value into `X-XSRF-TOKEN` (or whatever the framework expects).
3. POST your form with the cookie attached and the header set.

If your HTTP client auto-handles cookies (`requests.Session()`, `axios` with `withCredentials`), step 1 is automatic. You still must extract and forward the value as a header.

---

## 10. GraphQL

GraphQL has a recognizable signature: a single endpoint (commonly `/graphql` or `/api/graphql`), POST with body shape `{ query, variables, operationName }`. Responses use `{ data, errors }`.

### 10.1 Introspection enabled

If `__schema` introspection is on, you can dump the entire schema:

```bash
npm install -g graphql-cli
graphql-cli get-schema -e production -o schema.graphql

# or with a curl one-liner
curl -X POST https://api.example.com/graphql \
  -H 'content-type: application/json' \
  -d '{"query":"query IntrospectionQuery { __schema { types { name fields { name } } } }"}'
```

With the schema, you can write any query you want. Tools like `graphql-cli`, `gql` (Python), `graphql-request` (TS) consume the schema and give you typed clients.

### 10.2 Introspection disabled

Vendors sometimes disable introspection in production. Two paths:

**`clairvoyance`** (`github.com/nikitastupin/clairvoyance`) brute-forces field names against the type-error responses GraphQL sends. The server typically returns errors like `"Did you mean 'firstName'?"`, leaking valid field names. Clairvoyance feeds a wordlist and reconstructs the schema.

**Bundle reading** (doc 12). The frontend has every query string it sends, often as exported `gql` template literals or precompiled into `documentNode` objects. Search the unminified bundle for `query MyOperation` or `mutation MyOperation` — the source of truth lives there.

### 10.3 Security/cost auditing

`graphql-cop` (`github.com/dolevf/graphql-cop`) runs a battery of checks: introspection enabled, debug mode, alias overload, batching, field-suggestion leaks. Useful both as a scraper (to know what's allowed) and as a defender.

### 10.4 Cost amplification gotchas

GraphQL allows aliasing — running the same field many times in one query under different names:

```graphql
query Amplify {
  a: search(q: "cats") { results { id } }
  b: search(q: "cats") { results { id } }
  c: search(q: "cats") { results { id } }
}
```

This sidesteps naive rate limits that count requests rather than resolver invocations. Mature servers use depth limits, complexity scoring, or persisted queries. As a scraper you may stumble into 400-level errors that say "query complexity exceeded" — it usually means you need to flatten or batch your asks differently.

---

## 11. gRPC and gRPC-Web

### 11.1 gRPC over HTTP/2

gRPC is HTTP/2 + protobuf. Bodies are binary, framed with a 5-byte header per message. You cannot reverse-engineer it by reading bodies — you need the `.proto` schema.

If the server has reflection enabled:

```bash
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest
grpcurl -plaintext localhost:50051 list
grpcurl -plaintext localhost:50051 describe my.package.Service
grpcurl -plaintext -d '{"id": 42}' localhost:50051 my.package.Service/GetThing
```

Reflection is off in most production gRPC servers. Without it you need the `.proto` file from elsewhere — the project's GitHub repo, a leaked SDK, the frontend bundle (gRPC-Web compiled clients embed the message shape).

### 11.2 gRPC-Web

gRPC-Web is gRPC adapted to run in browsers, where HTTP/2 trailers and binary framing are not directly accessible. Spec at `github.com/grpc/grpc-web`. Recognizable in DevTools by:

```text
Content-Type: application/grpc-web+proto
Content-Type: application/grpc-web-text       (base64-encoded variant)
```

Since gRPC-Web rides over HTTP/1.1 or HTTP/2, mitmproxy and Burp see the bytes. The framing is:
- 1 byte flags
- 4 bytes big-endian message length
- N bytes protobuf message

Decode with the `.proto`. The frontend bundle ships the compiled `.pb.js` clients; the message types are recoverable.

A practical scraping path: import the same generated `*_pb.js` modules the frontend uses, in your Node scraper, and call them with raw HTTP. You don't need to regenerate from `.proto` if you can lift the modules.

---

## 12. WebSockets

WebSockets carry application-defined messages over a single long-lived connection. Three viewpoints:

### 12.1 DevTools

Network panel → filter "WS" → click the connection → "Messages" tab. You see frames in chronological order, color-coded by direction. Right-click a frame → copy contents.

### 12.2 mitmproxy

mitmproxy intercepts WebSocket frames natively when the upgrade goes through it. The `websocket_message` hook fires on every frame:

```python
from mitmproxy import ctx

def websocket_message(flow):
    msg = flow.websocket.messages[-1]
    direction = "->" if msg.from_client else "<-"
    ctx.log.info(f"{direction} {msg.content[:80]!r}")
```

Burp also intercepts WebSockets and lets you replay individual frames from the WebSockets History tab.

### 12.3 Scraping over WebSocket directly

Once you know the protocol, connect with a native client:

```ts
// Node, ws library
import WebSocket from 'ws';

const ws = new WebSocket('wss://realtime.example.com/socket', {
  headers: { authorization: `Bearer ${token}` },
});

ws.on('open', () => {
  ws.send(JSON.stringify({
    type: 'subscribe',
    channel: 'prices',
    symbols: ['BTC-USD', 'ETH-USD'],
  }));
});

ws.on('message', (data) => {
  const msg = JSON.parse(data.toString());
  // persist or process
});

// Many servers expect periodic pings
setInterval(() => ws.ping(), 30_000);
```

### 12.4 Common WS protocol shapes

| Pattern | Recognition |
|---------|-------------|
| **Phoenix Channels** (Elixir) | Messages `[join_ref, ref, topic, event, payload]`; topics like `prices:btc` |
| **Centrifugo** | `connect` → `subscribe` → `pub`; `id`/`method`/`params` JSON envelope |
| **Socket.IO** | Engine.IO framing prefix `42[...]`; non-trivial framing — use a Socket.IO client |
| **GraphQL subscriptions** | `connection_init` → `start` → `data` → `complete` (graphql-ws or legacy subscriptions-transport-ws) |
| **Custom JSON** | Any `{type, ...}` envelope |

When the server expects `connection_init` or some opening handshake, the scraper must replay it before any data flows. Capture once, replay verbatim.

---

## 13. Server-Sent Events (SSE)

SSE is HTTP/1.1 with `Content-Type: text/event-stream`, defined in the HTML Living Standard. Frames are plain text:

```text
event: price
data: {"symbol":"BTC-USD","px":68000}

data: {"symbol":"BTC-USD","px":68001}

```

Each event ends with a blank line. Compared to WebSockets: simpler, unidirectional (server → client), text-only, automatic reconnection in browsers via `EventSource`. SSE shows up in DevTools Network as a long-running request with the `EventStream` tab populated.

Replay with curl:

```bash
curl -N \
  -H 'accept: text/event-stream' \
  -H 'authorization: Bearer ...' \
  https://api.example.com/v1/stream
```

The `-N` (no buffer) flag is essential or curl will hold output until the connection closes. From Node:

```ts
const res = await fetch('https://api.example.com/v1/stream', {
  headers: { 'authorization': `Bearer ${token}` },
});
const reader = res.body!.getReader();
const decoder = new TextDecoder();
let buffer = '';
while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  buffer += decoder.decode(value, { stream: true });
  let idx;
  while ((idx = buffer.indexOf('\n\n')) !== -1) {
    const event = buffer.slice(0, idx);
    buffer = buffer.slice(idx + 2);
    // parse "event:" / "data:" / "id:" lines
  }
}
```

SSE is often missed because it looks like a hung request. If a Network entry has been "pending" for minutes and the response preview shows incremental text, suspect SSE.

---

## 14. WebRTC (Brief)

Mostly out of scope for HTTP scraping — WebRTC is peer-to-peer UDP for media and arbitrary data channels, negotiated over signaling (often WebSocket). For typical "scrape this dashboard" work, you will not need it. It becomes relevant for live-content scrapers (audio/video streams, multiplayer state). When you encounter it, the entry point is the signaling channel, not the media channel — capture the SDP exchange and ICE candidates.

---

## 15. Worked Examples

### 15.1 HMAC-signed search API

```ts
import { createHmac, createHash } from 'node:crypto';

const SECRET = process.env.API_SECRET!;
const BASE = 'https://api.example.com';

function sign(method: string, path: string, body: string) {
  const ts = Math.floor(Date.now() / 1000);
  const bodyHash = createHash('sha256').update(body).digest('hex');
  const stringToSign = [ts, method, path, bodyHash].join('\n');
  const sig = createHmac('sha256', SECRET).update(stringToSign).digest('hex');
  return { 'x-timestamp': String(ts), 'x-signature': sig };
}

async function search(q: string, page: number) {
  const path = '/v2/search';
  const body = JSON.stringify({ q, page });
  const headers = {
    'content-type': 'application/json',
    ...sign('POST', path, body),
  };
  const res = await fetch(BASE + path, { method: 'POST', headers, body });
  if (!res.ok) throw new Error(`${res.status}: ${await res.text()}`);
  return res.json();
}
```

### 15.2 Bearer token shared across N workers

```ts
import pLimit from 'p-limit';

const tokens = new TokenManager(/* config */);
const limit = pLimit(8);
const ids = await loadIds();

const results = await Promise.all(
  ids.map((id) =>
    limit(async () => {
      const token = await tokens.getToken();
      const res = await fetch(`${BASE}/items/${id}`, {
        headers: { authorization: `Bearer ${token}` },
      });
      return res.json();
    }),
  ),
);
```

`pLimit(8)` caps concurrency. `tokens.getToken()` single-flights refreshes (see section 8.3).

### 15.3 Paginated GraphQL connection (cursor-based)

```ts
const QUERY = `
  query ListItems($cursor: String) {
    items(first: 100, after: $cursor) {
      pageInfo { hasNextPage endCursor }
      edges { node { id name updatedAt } }
    }
  }
`;

async function* walk(token: string) {
  let cursor: string | null = null;
  while (true) {
    const res = await fetch(GRAPHQL_URL, {
      method: 'POST',
      headers: {
        'content-type': 'application/json',
        authorization: `Bearer ${token}`,
      },
      body: JSON.stringify({ query: QUERY, variables: { cursor } }),
    });
    const { data, errors } = await res.json();
    if (errors) throw new Error(JSON.stringify(errors));
    for (const edge of data.items.edges) yield edge.node;
    if (!data.items.pageInfo.hasNextPage) return;
    cursor = data.items.pageInfo.endCursor;
  }
}

for await (const node of walk(token)) {
  // persist
}
```

The async generator hides pagination from the consumer.

### 15.4 WebSocket subscription saved to disk

```ts
import WebSocket from 'ws';
import { createWriteStream } from 'node:fs';

const out = createWriteStream('messages.jsonl', { flags: 'a' });
const ws = new WebSocket('wss://realtime.example.com/socket', {
  headers: { authorization: `Bearer ${token}` },
});

ws.on('open', () => {
  ws.send(JSON.stringify({ type: 'subscribe', channel: 'updates' }));
});

ws.on('message', (data) => {
  out.write(data.toString() + '\n');
});

ws.on('close', () => out.end());
```

JSONL (one JSON object per line) is the right shape for append-only message logs — easy to tail, easy to re-process.

---

## 16. Detection-Resistance and the API "DRM" Problem

Some signed-request schemes are *intentionally* hostile to replay. Patterns:

- **Rotating signing keys** delivered through a heavily obfuscated bootstrap that runs on every page load. The key only lives in JS memory.
- **JS-based proof-of-work** — the client runs SHA-256 N times to produce a header value, with N high enough to slow scrapers but not real users.
- **Browser-environment-bound signatures** — the signing function reads `window.screen`, `navigator.plugins`, `WebGLRenderingContext` parameters, etc. Recomputing them outside a real browser produces a "valid-looking but wrong" signature.
- **WebAssembly-only signers** — the algorithm lives in WASM. Disassembling is tedious; running the WASM in your scraper is often easier.

When the signing scheme fights you harder than the data is worth, fall back to one of:
1. **Headless for auth, HTTP for bulk.** Use Playwright (doc 07) to perform the login or signed handshake, snapshot the cookies/tokens, then drive the bulk in HTTP. The headless cost amortizes if you reuse the session for thousands of requests.
2. **Headless for everything.** Slower, more expensive, but cuts the reverse-engineering time to zero.
3. **Run the JS in your scraper.** Lift the signing function with its dependencies and execute it inside your Node process or a small JS VM. Works when the function is self-contained.

This is the "API DRM" problem: vendors are deliberately raising the cost of replay. The arms race favors the vendor in the long run for high-value endpoints. Pick your battles by the ratio of one-time engineering cost to ongoing data value.

---

## 17. Ethics and Legality

Replaying APIs has the same legal contour as scraping web pages, with two extra wrinkles:

- **Authentication.** If you used credentials you weren't given (someone else's API key, a leaked OAuth token, a stolen session cookie), you are now in CFAA territory in the US and equivalent statutes elsewhere. Stick to credentials you legitimately hold.
- **Rate limits.** Bypassing an explicit rate limit by rotating IPs/accounts is ToS-breaking and, in the worst case, has been argued under CFAA. The post-*Van Buren* (2021) interpretation narrows CFAA to actual access bypass rather than ToS violations, but civil ToS suits remain possible.

Useful precedents to read, not as legal advice:
- **hiQ Labs v. LinkedIn** — the long-running case on scraping public LinkedIn profiles. The Ninth Circuit held that scraping public data probably does not violate the CFAA. LinkedIn won later rounds on contract grounds.
- **Van Buren v. United States** (US Supreme Court, 2021) — narrowed the CFAA's "exceeds authorized access" clause. Improper-purpose access to data you were already authorized to view is not CFAA-unauthorized access.

Practical heuristics:
- Public data, no auth, polite request rate: low risk.
- Authenticated endpoints with your own credentials, polite request rate: low-to-moderate risk depending on ToS.
- Bypassing rate limits via rotation: gray area; varies by jurisdiction and ToS specifics.
- Using credentials not your own: don't. Black letter unauthorized access.
- Data subject to additional regulation (PII under GDPR, health data under HIPAA, payment data under PCI): the scraping question is moot — you have separate legal obligations on the data itself.

When working with a bug-bounty target, scope is the contract; stay inside it. When building a product on top of someone else's API, prefer official partnerships when feasible — they are cheaper than litigation.

---

## Related

- [Chrome DevTools for Scraping](11-chrome-devtools-for-scraping.md)
- [Source Maps and Chunked Bundles](12-source-maps-and-chunked-bundles.md)
- [Headless Browsers](../extraction/07-headless-browsers.md)
- [HTTP Scraping Fundamentals](../extraction/06-http-scraping-fundamentals.md)
- [Bot Detection Internals](../extraction/09-bot-detection-internals.md)
- [JWT Design and Pitfalls](../../security/fundamentals/05-jwt-design-and-pitfalls.md)

## References

- mitmproxy documentation — `docs.mitmproxy.org/stable/`
- mitmproxy addons overview — `docs.mitmproxy.org/stable/addons-overview/`
- Burp Suite documentation — `portswigger.net/burp/documentation`
- HAR 1.2 specification — W3C Web Performance Working Group archive (`w3c.github.io/web-performance/specs/HAR/Overview.html`)
- AWS Signature Version 4 — `docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-signing.html`
- Stripe webhook signatures — `docs.stripe.com/webhooks/signatures`
- RFC 7519 — JSON Web Token (JWT) — `datatracker.ietf.org/doc/html/rfc7519`
- GraphQL specification and learn — `graphql.org/learn/` and `spec.graphql.org`
- `clairvoyance` (GraphQL field-name brute-forcer) — `github.com/nikitastupin/clairvoyance`
- `graphql-cop` (GraphQL security audit tool) — `github.com/dolevf/graphql-cop`
- `grpcurl` — `github.com/fullstorydev/grpcurl`
- gRPC-Web specification — `github.com/grpc/grpc-web/blob/master/doc/PROTOCOL-WEB.md`
- Server-Sent Events (HTML Living Standard, "Server-sent events" section) — `html.spec.whatwg.org/multipage/server-sent-events.html`
- *hiQ Labs, Inc. v. LinkedIn Corp.* — 9th Cir. opinions and subsequent district court rulings (2019, 2022)
- *Van Buren v. United States*, 593 U.S. ___ (2021) — `supremecourt.gov/opinions/20pdf/19-783_k53l.pdf`
