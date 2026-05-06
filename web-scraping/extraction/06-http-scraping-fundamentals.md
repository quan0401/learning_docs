---
title: "HTTP Scraping Fundamentals"
date: 2026-05-06
updated: 2026-05-06
tags: [http, scraping, requests, httpx, scrapy, colly, crawlee, sessions, cookies]
---

# HTTP Scraping Fundamentals

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `http` `scraping` `requests` `httpx` `scrapy` `colly` `crawlee` `sessions` `cookies`

---

## Table of Contents

1. [The Two-Mode Decision: HTTP vs Headless Browser](#1-the-two-mode-decision-http-vs-headless-browser)
2. [HTTP Clients — `requests`, `httpx`, `curl`](#2-http-clients--requests-httpx-curl)
3. [Sessions and Cookies — Stateful Scraping](#3-sessions-and-cookies--stateful-scraping)
4. [Redirect Chains and the Cookie/Authorization Stripping Trap](#4-redirect-chains-and-the-cookieauthorization-stripping-trap)
5. [Header Hygiene — Sounding Like a Browser](#5-header-hygiene--sounding-like-a-browser)
6. [Compression — gzip, brotli, deflate](#6-compression--gzip-brotli-deflate)
7. [HTTP/2 and HTTP/3 Differences That Affect Scraping](#7-http2-and-http3-differences-that-affect-scraping)
8. [Crawler Frameworks — Scrapy, Colly, Crawlee](#8-crawler-frameworks--scrapy-colly-crawlee)
9. [Concurrency, Rate Limiting, and Backoff](#9-concurrency-rate-limiting-and-backoff)

## Summary

Scraping is HTTP. Before reaching for a headless browser, ask whether the data you want is in the initial HTML or in JSON behind XHR/Fetch — if the latter, raw HTTP is 10–100× faster, cheaper, and more reliable. The Python ecosystem standardized on `requests` for years; `httpx` is the modern successor (HTTP/2, async, identical API for sync and async). Go has `net/http` directly and `Colly` for crawler-shaped work. Node has `fetch` (built in since Node 18) and `Crawlee` for crawler infrastructure. The non-obvious operational hazards: **session and cookie persistence** (you must reuse a session object or you lose login state every request), **redirect handling** (most clients strip `Authorization` cross-origin but not `Cookie` — sometimes the inverse is needed), **HTTP/2 header casing** (lowercased on the wire — surprises servers that case-check), and **client-default Accept-Encoding** (clients that advertise gzip/brotli but don't decode them silently corrupt responses).

---

## 1. The Two-Mode Decision: HTTP vs Headless Browser

The first question for any scrape: does the data render in HTML the server returns directly, or does it appear after JavaScript runs?

```bash
# Cheap test — does curl give you the data?
curl -sL https://example.com/products/123 | grep -i 'price'
```

If yes, raw HTTP is the answer. If no, two sub-questions:

- **Is the data in an XHR/Fetch call you can see in DevTools Network tab?** If yes, replay that request directly with HTTP. (Doc 13 covers API reverse-engineering.)
- **Does the data require client-side state, browser APIs, or JS-driven layout that no API exposes?** Only then reach for headless. (Doc 07.)

**Rule of thumb cost ratio:** Headless browser ~30–100× the CPU and memory of an HTTP request, and ~3–10× the latency. For 100k pages/day, this is the difference between $5/day and $500/day.

### 1.1 Server-Side Rendering Reveals Itself

Modern Next.js, Nuxt, Remix, SvelteKit, and similar frameworks server-render initial pages. Look for:

- `<script id="__NEXT_DATA__" type="application/json">{...}</script>` (Next.js)
- `window.__NUXT__ = {...}` (Nuxt)
- `__remixContext = {...}` (Remix)

The JSON inside these script tags often contains all the page data, sourced directly. Parse the JSON, skip the headless browser entirely.

---

## 2. HTTP Clients — `requests`, `httpx`, `curl`

### 2.1 Python — `requests` and `httpx`

`requests` (Kenneth Reitz, 2011) was the de-facto standard. It's still maintained but is HTTP/1.1-only and synchronous-only.

`httpx` ([encode/httpx](https://github.com/encode/httpx)) is the modern successor: same ergonomic API, plus HTTP/2, async, and a connection pool sized for crawlers.

```python
# requests — sync, HTTP/1.1
import requests
session = requests.Session()
r = session.get("https://example.com", timeout=15)

# httpx — sync HTTP/2
import httpx
with httpx.Client(http2=True, timeout=15) as client:
    r = client.get("https://example.com")

# httpx — async with concurrency
import httpx, asyncio
async def fetch(url):
    async with httpx.AsyncClient(http2=True) as client:
        r = await client.get(url)
        return r.text
asyncio.run(fetch("https://example.com"))
```

For high-concurrency Python scraping, `httpx.AsyncClient` paired with `asyncio.gather` is the standard. For ~100 concurrent requests, install [`hpack`](https://pypi.org/project/hpack/) and confirm HTTP/2 is negotiating; for 1000+, look at queueing (doc 14).

### 2.2 Go — `net/http` and Friends

Go's standard library is good enough for most scraping. The default client follows redirects, handles compression (with `Transport.DisableCompression: false`), and uses HTTP/2 transparently when the server supports it.

```go
client := &http.Client{
    Timeout: 15 * time.Second,
    Transport: &http.Transport{
        MaxIdleConnsPerHost: 100,
        IdleConnTimeout:     90 * time.Second,
    },
}
resp, err := client.Get("https://example.com")
```

For TLS fingerprint control (doc 09), the standard `crypto/tls` package is the substrate. [`utls`](https://github.com/refraction-networking/utls) lets you mimic any browser's ClientHello.

### 2.3 Node — `fetch` and `undici`

Node 18+ includes `fetch` natively, backed by [`undici`](https://github.com/nodejs/undici). For browsers-style ergonomics:

```js
const r = await fetch("https://example.com", {
  headers: { "User-Agent": "..." },
  redirect: "follow",
});
const text = await r.text();
```

For long-running crawlers, `undici.Agent` with explicit pool sizing is preferred:

```js
import { Agent, fetch as undiciFetch } from "undici";
const agent = new Agent({ connections: 100, pipelining: 1 });
const r = await undiciFetch("https://example.com", { dispatcher: agent });
```

### 2.4 `curl`, `wget`, and Friends

For ad-hoc scraping and cron jobs, `curl` is unbeatable. Useful flags:

```bash
curl -sL \                        # silent, follow redirects
     -A "Mozilla/5.0 ..." \       # User-Agent
     --compressed \               # accept gzip/brotli, decode automatically
     -H "Accept-Language: en" \   # extra headers
     -b cookies.txt -c cookies.txt \  # use+save cookie jar
     --max-time 30 \
     "https://example.com"
```

[`curl-impersonate`](https://github.com/lwthiker/curl-impersonate) is a fork that replicates exact browser TLS/HTTP/2 fingerprints; relevant for doc 09.

---

## 3. Sessions and Cookies — Stateful Scraping

Most non-trivial sites maintain server-side state via cookies. Without a cookie jar, every request is a fresh visitor.

### 3.1 The Session Object Pattern

Always use a session object (or its language equivalent), not bare functions:

```python
# requests — Session reuses connections AND cookies
session = requests.Session()
session.headers.update({"User-Agent": "..."})
session.get("https://example.com/login")  # CSRF cookie set
session.post("https://example.com/login", data=creds)  # session cookie set
session.get("https://example.com/account")  # uses both cookies
```

```js
// Node fetch with a manual cookie jar (fetch doesn't persist by default)
import { CookieJar } from "tough-cookie";
import { fetch as ffetch } from "fetch-cookie";  // wraps fetch with a jar
const jar = new CookieJar();
const fetch = ffetch(globalThis.fetch, jar);
await fetch("https://example.com/login", { method: "POST", body: form });
await fetch("https://example.com/account");  // jar carries cookies forward
```

### 3.2 Cookie Attributes That Affect Scraping

Cookies have attributes the browser respects:

| Attribute | Effect |
|-----------|--------|
| `Domain=` | Cookie sent to this domain and subdomains |
| `Path=` | Cookie sent only to URLs starting with this path |
| `Secure` | Sent only over HTTPS |
| `HttpOnly` | Not visible to client-side JS (browser-only flag; HTTP clients see it) |
| `SameSite=Strict\|Lax\|None` | Browser cross-site behavior — your HTTP client doesn't enforce this |
| `Max-Age=` / `Expires=` | TTL |
| `__Secure-` / `__Host-` prefix | Browser-enforced security; HTTP clients see them like any cookie |

For HTTP scraping you don't enforce `SameSite` (no cross-site context exists at the HTTP-client level), but you must respect `Domain` and `Path` to avoid leaking credentials.

### 3.3 Cookie Jar Persistence

For multi-run scraping, persist the cookie jar to disk:

```python
import http.cookiejar, requests
jar = http.cookiejar.MozillaCookieJar("cookies.txt")
session = requests.Session()
session.cookies = jar
try:
    jar.load(ignore_discard=True)
except FileNotFoundError:
    pass
# ... scrape ...
jar.save(ignore_discard=True)
```

---

## 4. Redirect Chains and the Cookie/Authorization Stripping Trap

By default, HTTP clients follow redirects (`301`, `302`, `303`, `307`, `308`). Two non-obvious behaviors:

### 4.1 Authorization Header Stripping

`requests`, `httpx`, and `curl` strip the `Authorization` header on cross-host redirects to prevent credential leakage. If your scraper depends on `Authorization: Bearer ...` and the site redirects through an SSO flow, your token disappears mid-redirect. The [`httpx` docs](https://www.python-httpx.org/compatibility/#redirect-headers) describe this. Workaround: handle redirects manually and re-attach the header on the same-origin chain only.

### 4.2 Cookie Behavior on Redirects

Cookies are sent or stripped based on `Domain` and `Path`, not on the redirect chain itself. So if `a.example.com` sets a cookie and redirects to `b.example.com`, the cookie does *not* travel unless the cookie's `Domain` is `.example.com`. This is the source of subtle login-redirect failures.

### 4.3 Method Changes

- `301`, `302`: many old clients (and historical browsers) change `POST` → `GET` on follow. Modern clients differ.
- `303`: explicitly mandates method change to `GET`.
- `307`, `308`: explicitly preserve the method and body.

For idempotent scrapes this rarely matters; for login flows that POST and follow into a GET landing page, watch the redirect codes.

### 4.4 Detecting Loops

Always cap redirect depth. Default in `requests` is 30; in `httpx` it's 20. Some pathological login flows (broken ones) loop forever; the cap saves you.

---

## 5. Header Hygiene — Sounding Like a Browser

Default HTTP-client headers identify the client immediately:

| Default User-Agent | Identifies |
|---------------------|------------|
| `python-requests/2.31.0` | requests |
| `python-httpx/0.27.0` | httpx |
| `curl/8.4.0` | curl |
| `Go-http-client/1.1` | Go |
| `node-fetch/3.x` | Node fetch |

### 5.1 Minimum Browser-Like Header Set

```text
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Cache-Control: no-cache
Pragma: no-cache
Sec-Ch-Ua: "Chromium";v="124", "Not-A.Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Upgrade-Insecure-Requests: 1
```

The `Sec-Ch-Ua-*` and `Sec-Fetch-*` headers are sent by Chromium browsers automatically; their absence is a strong scraper signal (doc 09).

### 5.2 Don't Lie Half-Truthfully

Common mistake: claim Chrome 124 in `User-Agent` but send `Accept-Encoding: gzip` only (real Chrome sends `gzip, deflate, br`). Or claim Chrome but send `Accept: */*` (Chrome sends a richer Accept). Mismatches between User-Agent and other headers are exactly the signal anti-bot uses.

The fix: pick a real recent Chrome User-Agent and copy the full header set from a real browser session.

---

## 6. Compression — gzip, brotli, deflate

Most HTTP responses are compressed. Three encodings:

- **`gzip`** — universal; every client supports it
- **`deflate`** — supported but less common
- **`br` (brotli)** — modern, ~20% smaller than gzip; widely supported but Python `requests` did *not* support brotli until [`urllib3` 2.0 / requests 2.31](https://github.com/urllib3/urllib3/blob/main/CHANGES.rst). Older code may receive brotli responses and fail to decode them.

`httpx` supports brotli when [`brotli`](https://pypi.org/project/Brotli/) or [`brotlicffi`](https://pypi.org/project/brotlicffi/) is installed. If you advertise `Accept-Encoding: br` and don't have the decoder, you get garbage.

```bash
# Test what your client receives
python -c 'import httpx; r = httpx.get("https://example.com", headers={"Accept-Encoding": "br"}); print(r.headers["content-encoding"])'
```

### 6.1 The `--compressed` flag

`curl --compressed` adds `Accept-Encoding: ...` and decodes the response transparently. Without it, you must add the header manually and decode yourself.

---

## 7. HTTP/2 and HTTP/3 Differences That Affect Scraping

### 7.1 Header Case Lowercasing

HTTP/1.1 headers are case-insensitive but conventionally `Title-Case`. **HTTP/2 mandates lowercase** ([RFC 7540 §8.1.2](https://datatracker.ietf.org/doc/html/rfc7540#section-8.1.2), preserved in [RFC 9113](https://datatracker.ietf.org/doc/html/rfc9113)). So when you send `User-Agent: ...` over HTTP/2, the wire format is `user-agent: ...`. Most servers ignore case, but some buggy servers and some bot-detectors flag deviations.

### 7.2 Multiplexing

HTTP/2 multiplexes many requests over one connection. A naive scraper opening 100 connections to one host now uses one — if the server has a per-connection rate limit, you blow through it 100× faster. Watch for this.

### 7.3 Server Push (Mostly Dead)

HTTP/2 server push has been [removed by Chrome](https://groups.google.com/a/chromium.org/g/blink-dev/c/K3rYLvmQUBY). Don't expect it; don't rely on it.

### 7.4 HTTP/3 / QUIC

HTTP/3 runs over QUIC (UDP). For scraping it largely behaves like HTTP/2 from the application layer. Library support is uneven — Go has experimental, Python via [`aioquic`](https://github.com/aiortc/aioquic) and [`httpx[http3]`](https://www.python-httpx.org/), Node has it via newer `undici`. Most scrapers can stay on HTTP/2 indefinitely.

### 7.5 HTTP/2 Settings Frame as a Fingerprint

The `SETTINGS` frame a client sends after the connection preface includes specific values (window sizes, stream limits, header table size). The combination is a recognizable fingerprint per client library — Akamai's well-known [HTTP/2 fingerprint](https://www.blackhat.com/docs/eu-17/materials/eu-17-Shuster-Passive-Fingerprinting-Of-HTTP2-Clients-wp.pdf) hashes this and identifies real browsers vs `requests`/`httpx`/Go. Covered in doc 09.

---

## 8. Crawler Frameworks — Scrapy, Colly, Crawlee

Once a scrape is more than a script, frameworks pay for themselves.

### 8.1 Scrapy (Python)

[Scrapy](https://scrapy.org/) is the heavyweight: a full crawler architecture with engine, scheduler, downloader, pipelines, middlewares, and item models. It's old, mature, and absolutely battle-tested.

Architecture:

```text
spider.start_requests() → scheduler → downloader → spider.parse() → pipelines
                              ↑                          ↓
                              └── new requests ──────────┘
```

Strengths: durability (resumable crawls), retry middleware, deduplication, asyncio bridge (Scrapy 2.0+), large ecosystem of plugins.

Weakness: opinionated, not the fastest at small-scale per-request work, and the integration with modern async Python is bolted on.

### 8.2 Colly (Go)

[Colly](https://github.com/gocolly/colly) is the dominant Go crawler:

```go
c := colly.NewCollector(
    colly.AllowedDomains("example.com"),
    colly.Async(true),
)
c.Limit(&colly.LimitRule{Parallelism: 8, Delay: 250 * time.Millisecond})
c.OnHTML("a[href]", func(e *colly.HTMLElement) {
    e.Request.Visit(e.Attr("href"))
})
c.OnHTML("h1", func(e *colly.HTMLElement) {
    fmt.Println(e.Text)
})
c.Visit("https://example.com")
c.Wait()
```

Idiomatic Go — small binary, fast, easy to deploy.

### 8.3 Crawlee (Node)

[Crawlee](https://crawlee.dev/) (Apify, 2022) is the modern Node crawler. It abstracts over both raw HTTP (`CheerioCrawler`) and headless (`PlaywrightCrawler`) with the same API, plus session pools, proxy rotation, and persistent queues.

```ts
import { CheerioCrawler } from "crawlee";

const crawler = new CheerioCrawler({
  maxConcurrency: 10,
  async requestHandler({ $, request, enqueueLinks }) {
    const title = $("h1").text();
    console.log(`${request.url}: ${title}`);
    await enqueueLinks();
  },
});
await crawler.run(["https://example.com"]);
```

Strengths: TypeScript-native, automatic session management, drop-in switch from Cheerio (HTTP) to Playwright (headless) without changing handler logic.

### 8.4 When to Pick Which

| Need | Pick |
|------|------|
| Python team, complex crawl with pipelines | Scrapy |
| Go team, performance-critical, simple crawl | Colly |
| Node/TS team, may need headless later | Crawlee |
| Simple one-off, any language | Just use `requests`/`fetch`/`net/http` |

---

## 9. Concurrency, Rate Limiting, and Backoff

### 9.1 Per-Host vs Global Concurrency

The right control surface is **per-host concurrency**, not global. Crawling 1000 hosts with 10 concurrent requests *each* is very different from crawling 1 host with 10000 concurrent requests. Most frameworks expose this:

- Scrapy: `CONCURRENT_REQUESTS_PER_DOMAIN`
- Colly: `colly.LimitRule{DomainGlob: "*example.com*", Parallelism: 4}`
- Crawlee: `maxConcurrency` (global) + `sessionPoolOptions` for per-host

### 9.2 Rate Limiting

Implement client-side rate limiting: target X requests per second per host. The simplest correct algorithm is **token bucket**:

```python
class TokenBucket:
    def __init__(self, rate, capacity):
        self.rate = rate          # tokens per second
        self.capacity = capacity
        self.tokens = capacity
        self.last = time.monotonic()

    async def acquire(self, n=1):
        while True:
            now = time.monotonic()
            self.tokens = min(self.capacity, self.tokens + (now - self.last) * self.rate)
            self.last = now
            if self.tokens >= n:
                self.tokens -= n
                return
            await asyncio.sleep((n - self.tokens) / self.rate)
```

### 9.3 Retry With Jitter

When a request fails (5xx, 429, transient connect errors), retry with **exponential backoff plus jitter**. Without jitter, simultaneous failures retry in lockstep and create a thundering herd.

The canonical algorithm — full jitter from [AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/):

```python
def backoff(attempt, base=1.0, cap=30.0):
    return random.uniform(0, min(cap, base * 2 ** attempt))
```

For `429 Too Many Requests`, prefer the server-supplied `Retry-After` header if present.

### 9.4 Connection Pool Sizing

A connection pool that's too small bottlenecks; too large saturates the server and gets you blocked. Default sizes:

- `requests`: 10 per host (HTTPAdapter `pool_maxsize=10`)
- `httpx`: defaults to 100 keepalive + 100 max
- `undici`: configurable, default 10
- Go: `MaxIdleConnsPerHost` defaults to 2 (low; raise it)

For a polite crawler, 10–20 per host is plenty. For aggressive crawls, 100+ — but verify the target tolerates it.

---

## Related

- [Subdomain & Asset Enumeration](../discovery/01-subdomain-and-asset-enumeration.md) — produces hosts to scrape
- [Sitemap, robots.txt, and Crawl Surface](../discovery/03-sitemap-robots-and-crawl-surface.md) — produces URLs to scrape
- [Headless Browsers](07-headless-browsers.md) — when HTTP is not enough
- [Bot Detection Internals](09-bot-detection-internals.md) — what makes raw HTTP detectable
- [Production Scraping Hygiene and Legal Landscape](../reverse-engineering/14-production-scraping-hygiene-and-legal.md) — politeness and legal layer
- [HTTP/2 and HTTP/3 in the Networking learning path](../../networking/INDEX.md)

## References

- [RFC 9110 — HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110)
- [RFC 9112 — HTTP/1.1](https://datatracker.ietf.org/doc/html/rfc9112)
- [RFC 9113 — HTTP/2](https://datatracker.ietf.org/doc/html/rfc9113)
- [RFC 9114 — HTTP/3](https://datatracker.ietf.org/doc/html/rfc9114)
- [RFC 6265 — HTTP Cookie](https://datatracker.ietf.org/doc/html/rfc6265)
- [`httpx` documentation](https://www.python-httpx.org/)
- [Scrapy Architecture overview](https://docs.scrapy.org/en/latest/topics/architecture.html)
- [Colly](https://github.com/gocolly/colly)
- [Crawlee](https://crawlee.dev/)
- [`undici`](https://undici.nodejs.org/) — Node's HTTP client
- [AWS Architecture Blog — Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Akamai HTTP/2 Fingerprinting whitepaper (Black Hat EU 2017)](https://www.blackhat.com/docs/eu-17/materials/eu-17-Shuster-Passive-Fingerprinting-Of-HTTP2-Clients-wp.pdf)
