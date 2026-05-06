---
title: "Production Scraping Hygiene and Legal Landscape"
date: 2026-05-06
updated: 2026-05-06
tags: [politeness, rate-limiting, dedup, legal, cfaa, gdpr, robots-txt, scraping-ops]
---

# Production Scraping Hygiene and Legal Landscape

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `politeness` `rate-limiting` `dedup` `legal` `cfaa` `gdpr` `robots-txt` `scraping-ops`

---

> **Disclaimer:** This document is a personal engineering study aid. It summarizes case law and statutes for context only. It is **not legal advice**. Any real-world scraping project that touches personal data, commercial use, or any non-trivial target should be reviewed by a qualified attorney in the relevant jurisdiction.

## Table of Contents

1. From a Script to a Production System
2. Politeness — the Engineering Side
3. Client-Side Rate Limiting and Concurrency Caps
4. URL Canonicalization for Dedup
5. Content-Hash Dedup
6. The Crawl Frontier — Queue, Dedup-Set, Scheduler
7. Monitoring Metrics that Matter
8. Storage for Scraped Corpora
9. Cost Control — When Scraping Is a Real Budget Line
10. Legal Landscape — United States
11. Legal Landscape — European Union
12. Legal Landscape — Beyond US/EU
13. Personal Data on Scraped Content — GDPR / CCPA
14. Terms of Service and ToS-as-Contract Analysis
15. Practical Legal Hygiene Checklist
16. When to Escalate to Actual Lawyers
17. Bug-Bounty Contexts
18. Related
19. References

---

## Summary

Scraping at hobby scale is a script: a Node process, a `for` loop, a JSON file. Scraping in production is a *system*: a frontier, a scheduler, a worker pool, a proxy budget, a metrics pipeline, a legal posture, and a paper trail. The transition is mostly about taking things you used to ignore and making them first-class concerns.

This doc covers two halves of that transition. The **engineering half** is the mechanical hygiene that prevents you from being blocked, banned, or sued for sloppiness: politeness (`robots.txt`, per-host concurrency, identifying yourself), rate limiting (token bucket, backoff, circuit breakers), deduplication (URL canonicalization plus content-hash dedup), the crawl-frontier data structures, monitoring, and cost. The **legal half** is the framework you need to think clearly about whether what you're doing is allowed: the CFAA in the US (and the *hiQ* and *Van Buren* cases that narrowed it), state trespass-to-chattels doctrine, GDPR Article 6 lawful-basis analysis in the EU, the sui-generis database right, the Article 4 TDM exception in the DSM Directive, ToS contract questions, and the personal-data obligations that apply even when the scrape itself is fine.

The audience here is a TS/Node backend dev. Code shapes are sketched in TypeScript. Citations are real and verified — every case name, F. Supp. number, and CFR/Regulation referenced exists.

---

## 1. From a Script to a Production System

A one-off scrape that runs on your laptop has a tiny blast radius. A production scraper has all of these failure modes that the script never had to think about:

| Concern | Script-mode default | Production reality |
|---------|---------------------|--------------------|
| Politeness | "Whatever, I'll Ctrl-C if I get rate-limited" | Per-host budget, `robots.txt`, identifying UA |
| Dedup | Set in memory, lost on restart | Persistent, distributed, bloom-filter, content-hash |
| Cost | Free (your laptop, your IP) | Proxy GB, CAPTCHA solver calls, headless browser hours, storage |
| Monitoring | `console.log` | RED metrics per host, p99 latency, queue depth, block rate per ASN |
| Failure recovery | Re-run the script | Frontier persistence, idempotent fetch, dead-letter queue |
| Legal posture | None — assume nothing happens | Scope memo, ToS review, retention policy, deletion procedure |
| Compliance | None | GDPR Art 6 / CCPA / Art 17 erasure handling |

A useful mental shift: **scraping in production is a small distributed system that happens to fetch other people's HTML**. The scraper is the cheap part; the queue, the storage, and the legal/operational hygiene around it are most of the engineering effort.

The rest of this doc walks through each of those concerns.

---

## 2. Politeness — the Engineering Side

Politeness is not just an ethical posture — it is a precondition for staying unblocked. Rude crawlers get IP-banned, ASN-banned, sued, or have their cloud account flagged.

### 2.1 `robots.txt` — what to honor

The relevant standard is **RFC 9309** ("Robots Exclusion Protocol", 2022), which standardized what was a de-facto convention since 1994. Read its `Disallow` and `Allow` semantics carefully — RFC 9309 specifies longest-match-wins on the path patterns.

What's in RFC 9309 (so honor strictly if you claim to respect it):

- `User-agent` group selection
- `Allow` and `Disallow` rules with longest-match precedence
- The 500KB parsing limit
- Caching rules (sec 2.4) — fetch `robots.txt` no more than every 24 hours

What's *not* in RFC 9309 but is widely respected by polite crawlers:

- **`Crawl-delay: N`** — never standardized, but Bing, Yandex, and Yahoo respect it. Google does not. If a host sets `Crawl-delay`, the operator is asking; honoring it when reasonable is the polite default.
- **`Sitemap:` lines** — pointers to a sitemap. Always parse these and prefer them over spidering.
- **Wildcard (`*`) patterns and `$` end-anchor** — common extensions, supported by major crawlers.

If you don't intend to honor `robots.txt`, your User-Agent should not pretend to be a normal crawler. Lying about your UA *and* ignoring robots is the combination that draws lawsuits (this came up in *Craigslist v. 3Taps*).

### 2.2 Per-host concurrency caps

Single digits is sane for an unfamiliar host. Scrapy's defaults are illustrative: `CONCURRENT_REQUESTS_PER_DOMAIN = 8`, `CONCURRENT_REQUESTS_PER_IP = 0` (per-IP cap off by default). Common shapes for a cautious crawler:

- Unknown public site: 1–4 concurrent
- Known robust site (large e-commerce, news org): 4–8 concurrent
- Your own infrastructure / explicit permission: as fast as the target accepts

Concurrency is multiplicative with proxy pool: 4 concurrent × 100 IPs = 400 requests in flight, which is no longer "polite" from the target's perspective even if each IP is doing little. Cap concurrency *per origin host*, independent of pool size.

### 2.3 Sitemap-driven scheduling

Always prefer walking a sitemap to spidering links. Reasons:

- Sitemap URLs are the URLs the operator *wants* indexed, not random query-string variants.
- Sitemaps include `<lastmod>` so you can re-crawl only changed pages.
- Spidering tends to discover infinite calendars, faceted-search loops, and other crawler traps. Sitemaps don't.

A typical order of preference:

1. `sitemap.xml` (or `sitemap_index.xml`) at the host root.
2. `Sitemap:` lines in `robots.txt`.
3. Common conventions: `/sitemap.xml`, `/sitemap_index.xml`, `/sitemap-pages.xml`, `/sitemap-products.xml`.
4. Only if those fail, BFS spider from a known seed page with a hard depth cap.

### 2.4 User-Agent identification

Include a way for the operator to contact you:

```
MyScraper/1.0 (+https://example.com/scraper-info; contact@example.com)
```

That URL should resolve to a page that explains:

- Who runs the scraper
- What it's used for
- How to request rate-limiting changes or exclusion
- An email that a human reads

Operators who can contact you will email; operators who can't will block you. The cheapest way to stay unblocked is to be reachable.

Note: this hygiene is for *legitimate* crawlers. Anti-bot evasion (covered in earlier docs in this path) is a separate context where you're rotating UAs to look like normal browsers. The two postures don't mix in one scraper — pick one.

---

## 3. Client-Side Rate Limiting and Concurrency Caps

The goal is: bounded outbound load, graceful response to back-pressure signals from the server, and a way to stop hammering a host that's clearly broken.

### 3.1 Token bucket vs leaky bucket

Two algorithms cover almost all of this:

- **Token bucket**: fill rate `r` tokens/sec, capacity `b`. A request consumes one token. If empty, wait. *Allows bursts up to `b`*, then steady-state at `r`.
- **Leaky bucket**: requests enter a queue that drains at fixed rate `r`. *Smooths output to exactly `r`*, no bursts.

For scraping, **token bucket is usually the right pick**:

- Real targets tolerate small bursts (browsers do this).
- A burst lets you parallelize a small fan-out (e.g. 8 product detail pages from a category page) without artificially serializing.
- The cap on the bucket size acts as your max burst.

Leaky bucket is preferable when the target has a *strict* rate limit (e.g. an explicit "10 req/s" published in their docs) and you want to be perfectly compliant.

### 3.2 Per-host budget enforcement

This is the most common bug in homegrown scrapers: the rate limit is enforced globally instead of per origin host, which means a slow host starves a fast one. Keep one bucket per `(scheme, host)` tuple:

```ts
type HostKey = `${string}://${string}`;
const buckets = new Map<HostKey, TokenBucket>();

function bucketFor(url: URL): TokenBucket {
  const key = `${url.protocol}//${url.host}` as HostKey;
  let b = buckets.get(key);
  if (!b) {
    b = new TokenBucket({ rate: 2, capacity: 8 }); // per-host defaults
    buckets.set(key, b);
  }
  return b;
}
```

Per-host buckets are independent of your worker pool size. A 50-worker fleet hitting 10 hosts at 2 rps each emits 20 rps total, not 50.

### 3.3 Adaptive throttling on 429/503

When the server tells you to slow down — `429 Too Many Requests`, `503 Service Unavailable`, or honors `Retry-After` headers — back off:

- Read `Retry-After` (RFC 9110 §10.2.3) — it's either seconds or an HTTP-date. If present, use it.
- Otherwise apply exponential backoff with full jitter: `delay = random(0, base * 2^attempt)`, capped at e.g. 60s. Full jitter (vs. equal jitter) avoids thundering-herd retries from a swarm of workers.
- Reduce the per-host bucket rate while you're seeing 429s. AIMD (additive-increase / multiplicative-decrease) works: halve the rate on 429, slowly increase by a small additive step on continued 200s.

### 3.4 Circuit breakers per host

If a host is consistently 5xx, stop hitting it. The standard breaker has three states:

- **Closed**: requests pass through. On consecutive failures (e.g. 5 in a row), trip → Open.
- **Open**: requests fail fast for a cool-down (e.g. 60s). After cool-down → Half-open.
- **Half-open**: allow one probe. Success → Closed. Failure → Open with longer cool-down.

A per-host circuit breaker complements the per-host token bucket: the bucket controls *rate*, the breaker controls *whether* you're sending at all.

```ts
class HostBreaker {
  private state: 'closed' | 'open' | 'half-open' = 'closed';
  private failures = 0;
  private openedAt = 0;
  private cooldownMs = 60_000;

  async run<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.openedAt < this.cooldownMs) {
        throw new Error('circuit-open');
      }
      this.state = 'half-open';
    }
    try {
      const out = await fn();
      this.failures = 0;
      this.state = 'closed';
      return out;
    } catch (err) {
      this.failures += 1;
      if (this.failures >= 5 || this.state === 'half-open') {
        this.state = 'open';
        this.openedAt = Date.now();
      }
      throw err;
    }
  }
}
```

---

## 4. URL Canonicalization for Dedup

The same logical page often has many URL spellings. Without canonicalization, you re-fetch and re-store the same content under different keys.

### 4.1 What to canonicalize

A canonical URL is a function of:

- **Scheme** — preserved as-is, lowercased. `HTTP://` → `http://`.
- **Host** — lowercased. `Example.COM` → `example.com`. (Hosts are case-insensitive per RFC 3986 §3.2.2.)
- **Port** — strip if it's the default for the scheme (`80` for http, `443` for https).
- **Path** — preserve case (paths *are* case-sensitive on most servers). Decode percent-escaped unreserved characters; re-encode only what RFC 3986 says must be encoded. Resolve `.` and `..` segments.
- **Query** — sort parameters by name (then by value for repeated keys). Drop tracking params (see 4.2). Keep semantically meaningful params.
- **Fragment** — drop entirely for crawl dedup. The server doesn't see fragments. (Exception: SPAs with hash routing — those are not really HTTP-distinct URLs anyway and need a different dedup strategy.)
- **Trailing slash** — pick a rule and apply it consistently. Common rule: keep trailing slash on directory-like paths, strip on file-like paths. A safer rule for dedup: probe both, take whichever 200s without redirecting.

### 4.2 Strip tracking parameters

A short denylist removes noise from URL dedup:

```
utm_source, utm_medium, utm_campaign, utm_term, utm_content
fbclid, gclid, gbraid, wbraid, msclkid, dclid, yclid
ref, ref_, _ga, mc_cid, mc_eid
hsCtaTracking, hsa_*
mkt_tok
```

These are session/attribution tags. Stripping them before hashing collapses thousands of distinct URLs back to one resource.

### 4.3 Resolve relative URLs against the base

Use the WHATWG URL parser; don't roll your own:

```ts
const absolute = new URL(linkAttribute, baseUrl).toString();
```

The base is the document's URL after redirects, not the seed URL. If the response had a `<base href>` tag, that wins per HTML spec.

### 4.4 Existing libraries

Don't write canonicalization from scratch. Battle-tested options:

- **w3lib** (Python, used by Scrapy) — `w3lib.url.canonicalize_url`. Sorts query params, strips fragment, re-encodes consistently. Source code is short and is a useful reference for the "correct" rules.
- **`url-canonicalize`** / **`normalize-url`** (Node) — similar coverage in JS.
- **WHATWG URL** (`new URL(...)`) — handles parsing and percent-encoding correctly. Layer the dedup-specific rules (sort query, strip tracking, etc.) on top.

A canonicalizer in the shape of a scraper:

```ts
const TRACKING_PARAMS = new Set([
  'utm_source', 'utm_medium', 'utm_campaign', 'utm_term', 'utm_content',
  'fbclid', 'gclid', 'msclkid', 'yclid', 'gbraid', 'wbraid',
  'ref', 'ref_', 'mc_cid', 'mc_eid', 'mkt_tok',
]);

const DEFAULT_PORTS: Record<string, string> = {
  'http:': '80',
  'https:': '443',
};

export function canonicalize(rawUrl: string, base?: string): string {
  const u = new URL(rawUrl, base);
  u.hash = '';
  u.hostname = u.hostname.toLowerCase();
  if (DEFAULT_PORTS[u.protocol] === u.port) u.port = '';

  const params = Array.from(u.searchParams.entries())
    .filter(([k]) => !TRACKING_PARAMS.has(k.toLowerCase()))
    .sort(([a], [b]) => (a < b ? -1 : a > b ? 1 : 0));

  u.search = '';
  for (const [k, v] of params) u.searchParams.append(k, v);
  return u.toString();
}
```

Note that this still leaves a few decisions to the operator (trailing-slash policy, capitalization in path). Pick a rule, write it down, apply it everywhere — *consistency is more important than which rule you pick*.

---

## 5. Content-Hash Dedup

URL dedup tells you "have I fetched this URL before". Content-hash dedup tells you something different: "**did this URL's content actually change since last fetch**".

### 5.1 The mechanic

Hash the normalized response body. SHA-256 is fine — it's overkill cryptographically, but the speed difference vs. xxhash or BLAKE3 doesn't matter at scraper scale, and SHA-256 is built into every standard library.

```ts
import { createHash } from 'node:crypto';

function contentHash(body: Buffer): string {
  return createHash('sha256').update(body).digest('hex');
}
```

"Normalized" means:

- For HTML: parse, strip volatile parts (CSRF tokens, ad slots, timestamps in the footer, session-injected scripts), then serialize back. Or extract only the content region (the article body, the product card grid) before hashing.
- For JSON: re-serialize with sorted keys. Two semantically-identical JSON objects can have different byte-level representations.

Without normalization, you'll see "content changed" every fetch because the page injects a new request-id or `<meta name="csrf-token">` value each time.

### 5.2 What it catches

- **Content cycling**: same URL, content changes (a price ticks, a comment count increments). Hash differs → re-extract.
- **Identity changes**: same URL, content was previously identical and now differs → schedule it for higher-priority re-crawl.
- **Cross-URL duplicates**: different URLs (a `/?ref=share` variant your canonicalizer missed, a syndicated article on multiple hosts) hashing to the same content. You can choose to keep one and skip the others.

### 5.3 What it doesn't catch

- Soft 404s (a page that returns 200 but says "not found" in the body). Detect with content heuristics, not hash.
- Personalized content (logged-in or geo-targeted). Hashes will differ across crawler instances even though the canonical content is the same. Strip personalization or pin the crawler to a stable region.

### 5.4 Storage shape

A simple table:

| url_canonical | last_fetched_at | content_hash | etag | last_modified | extract_status |
|---------------|-----------------|--------------|------|---------------|----------------|

Combine with `If-None-Match` (ETag) and `If-Modified-Since` headers per RFC 9110 — the server may save you the bandwidth entirely with a `304 Not Modified`.

---

## 6. The Crawl Frontier — Queue, Dedup-Set, Scheduler

The frontier is the in-flight bookkeeping for "what to crawl next". It has three pieces:

1. **Queue** of URLs awaiting fetch.
2. **Dedup set** of URLs already enqueued or fetched (so we don't enqueue the same URL twice).
3. **Scheduler** that picks the next URL respecting per-host budgets, priority, and freshness.

### 6.1 In-memory vs distributed

| Scale | Frontier shape |
|-------|----------------|
| ≤ 100k URLs | In-memory `Set` + `PriorityQueue` |
| 100k – 10M | Single Redis instance with `SET` (dedup) and a sorted-set per priority bucket |
| 10M+ | Redis cluster, or Kafka topic for the queue + Redis bloom for dedup, or a purpose-built crawl frontier (Frontera, StormCrawler) |

The first cliff (in-memory → Redis) is hit when you can't fit the dedup set in RAM or you want the crawl to survive a restart. The second cliff (Redis → bloom + distributed queue) is hit when even the dedup set is too big for one machine.

### 6.2 Bloom filter for dedup at scale

A bloom filter is a probabilistic set: insert and lookup, with one-sided error (false positives possible, false negatives impossible). At scraper scale, false positives mean "I think I've seen this URL, so I skip it" — a tolerable error mode if it's rare.

Sizing: a bloom filter at 1% false-positive rate uses ~9.6 bits per element. Storing 1B URLs costs ~1.2 GB of memory — feasible on one machine. Compare to a `Set<string>` of 1B URLs (each averaging 60 bytes), which would be ~60 GB. Two-orders-of-magnitude savings.

If you can't tolerate false positives — for instance, if "skipped" means "lost forever" — pair the bloom filter with a slower, exact backing store (e.g. RocksDB or a sharded SQL table) and only consult the exact store on bloom-positive.

### 6.3 Priority queue

A flat FIFO works but is suboptimal. Useful priorities:

- **Freshness** — pages where the previous fetch was longest ago.
- **Importance** — sitemap priority, internal link rank, content type (e.g. product pages over tag pages).
- **Failed retries** — pages that errored, scheduled for retry after a backoff.

In Redis, this is naturally a sorted set: `ZADD frontier:high <score> <url>`, `ZPOPMIN frontier:high`. Multiple queues for multiple priority classes; the scheduler picks from high before low.

### 6.4 Scrapy-style request fingerprint

Scrapy's request fingerprint is the canonical example of "URL + small state turned into a stable hash for dedup". Sketched in TS:

```ts
function requestFingerprint(req: {
  method: string;
  url: string;       // already canonicalized
  body?: Buffer;
  headers?: Record<string, string>;
  includeHeaders?: string[]; // explicit allowlist
}): string {
  const h = createHash('sha1');
  h.update(req.method.toUpperCase());
  h.update(req.url);
  if (req.body && req.body.length) h.update(req.body);
  if (req.includeHeaders) {
    for (const name of req.includeHeaders.sort()) {
      const v = req.headers?.[name.toLowerCase()];
      if (v != null) {
        h.update(name.toLowerCase());
        h.update(v);
      }
    }
  }
  return h.digest('hex');
}
```

Key points (matching Scrapy's design):

- Method, URL, and body are always part of the fingerprint.
- Headers are *not* part of the fingerprint by default — most headers are noise (`User-Agent`, `Cookie`, request-id) and would defeat dedup. Only an explicit allowlist of headers is included, when those headers actually change the response (e.g. `Authorization` for a private endpoint).
- The fingerprint is content-addressable — same logical request → same fingerprint, regardless of when or where it's enqueued.

Reference: Scrapy's `scrapy.utils.request.fingerprint` (see References).

---

## 7. Monitoring Metrics that Matter

A scraper at production scale is a system that can quietly degrade in ways that take days to notice. Instrument it like any other backend service: RED (Rate, Errors, Duration) per host, plus scrape-specific signals.

### 7.1 Per-host metrics

- **Success rate** — `2xx_count / total_count` per host. Sudden drop = the host changed its anti-bot config or your IPs got flagged.
- **Block rate per ASN** — split your fleet by the ASN of the egress IP (datacenter vs. residential vs. mobile). When a residential pool's ASN starts seeing 403s while datacenter is fine, that ASN is burned.
- **Latency p50 / p95 / p99** — tail latency reveals throttling. Median moves slowly; p99 spikes the moment the target starts feeding you challenge pages instead of content.
- **Bytes downloaded vs. bytes extracted** — if you're downloading 5 MB and extracting 5 KB, you're wasting transfer. Common cause: a page that's mostly images and ad scripts; switch to fetching only the article-body endpoint if there is one.

### 7.2 Fleet-level metrics

- **Queue depth** per priority class. A growing queue means workers can't keep up; a shrinking queue with no new work means the seeds are exhausted.
- **In-flight requests** vs. concurrency cap.
- **Cost per useful record** — proxy GB cost + CAPTCHA solver cost + compute + storage, divided by number of records that passed quality gates. This is the only metric that tells you whether the scrape is actually worth running.

Cross-link: see the [observability learning path](../../observability/INDEX.md) for the underlying RED/USE method, SLO design, and OpenTelemetry instrumentation.

### 7.3 Alerts that pay off

- Block rate per host > 5% for 10 minutes.
- Queue depth growing for 30 minutes without new seed input.
- Per-host p99 latency 3× baseline for 15 minutes.
- Proxy GB burn-rate > daily budget × 1.5.
- Daily extracted-record count < 50% of trailing 7-day median.

---

## 8. Storage for Scraped Corpora

Two stores, one for raw and one for extracted:

### 8.1 Raw responses → object store

- S3, GCS, R2, or equivalent.
- **Key by content hash**: `s3://corpus-raw/<sha256>`. Two identical pages collapse to one object automatically.
- A small index table maps `(url_canonical, fetched_at) → content_hash`.
- WARC format (RFC 9728-equivalent format used by web archives) is the standard for archival snapshots — useful if you want compatibility with archival tooling.

### 8.2 Extracted records → typed database

- **PostgreSQL** for transactional record-shaped data (one row per product, listing, article).
- **ClickHouse** for high-volume, append-mostly analytics workloads (e.g. price-history tables with 10M+ rows/day).
- **Object store with Parquet** for cold-but-queryable data via Athena/BigQuery.

Cross-link: see the [database learning path](../../database/INDEX.md) for the choice between these and the schema-design tradeoffs.

### 8.3 Idempotency and re-extraction

The pipeline should support **re-extracting from raw**. If your extractor has a bug and you've already discarded the raw HTML, you have to re-scrape. Keeping raw responses by content hash means:

- Bugfix the extractor.
- Re-run the extractor over the raw store (no network calls).
- Replace the typed-DB rows.

This separation pays for itself the first time you find a parsing bug.

---

## 9. Cost Control — When Scraping Is a Real Budget Line

At scale, scraping is not free. The dominant costs:

| Cost line | Rough order of magnitude |
|-----------|--------------------------|
| Residential proxy GB | $5–$15 per GB (varies by provider, contract size) |
| Datacenter proxy GB | $0.5–$2 per GB |
| Mobile proxy GB | $20–$50 per GB |
| CAPTCHA solver | $1–$3 per 1k solves (image), $2–$3 per 1k solves (reCAPTCHA v2) |
| Headless browser hours | Equivalent to general-purpose compute, but a Chromium instance is heavy — easily $0.05–$0.20 per browser-hour at small scale |
| Storage at TB scale | $0.02 per GB-month for hot object storage, less for cold tiers |

These numbers move; the relative ranking is what matters. Residential proxies are the dominant cost in most production scrapers.

### 9.1 Cheap-first strategy

The right default:

1. Start with **datacenter proxies** + a polite UA. Many sites accept this fine.
2. Escalate to **residential** only on persistent 403s or CAPTCHA challenges from datacenter.
3. Escalate to **mobile** only when residential is also burning (rare; mostly relevant for mobile-app endpoints).
4. Only invoke a **CAPTCHA solver** as a last-resort recovery, not as a primary mechanism. If you're solving CAPTCHAs on every request, redesign the approach (different endpoint, session reuse, slowing down).

Track the escalation: a per-host metric for "what proxy tier is this host currently using" lets you spot regressions where a host that used to work on datacenter has silently been bumped to residential.

### 9.2 Compute

- Headless browsers are the single biggest compute line. Use them only when the page is genuinely JS-rendered. For server-rendered HTML, plain HTTP is 10–100× cheaper.
- Reuse browser contexts. Spinning up a fresh Chromium per request costs ~500ms–1s. Pool them.

### 9.3 Storage

- Apply lifecycle rules: move cold raw responses to S3 Glacier / GCS Archive after 30–90 days.
- Compress raw HTML aggressively (zstd or gzip) before storing — text compresses 5–10×.

---

## 10. Legal Landscape — United States

> Reminder: not legal advice. This section sketches the framework so you know which questions to bring to a real lawyer.

The US has no scraping-specific federal statute. The relevant law comes from a mix of:

- A federal hacking statute (CFAA)
- State trespass-to-chattels common law
- Contract law (ToS as contract)
- IP law (copyright, trade secret)
- Sector-specific privacy law (CCPA, HIPAA, GLBA — only applicable in narrow contexts)

### 10.1 The Computer Fraud and Abuse Act — 18 U.S.C. § 1030

The CFAA criminalizes accessing a computer "without authorization" or in a way that "exceeds authorized access". For two decades it was the federal cudgel against scrapers. Two recent decisions narrowed its reach significantly.

### 10.2 *hiQ Labs, Inc. v. LinkedIn Corp.*

- 9th Circuit, 938 F.3d 985 (2019); on remand after *Van Buren*, 31 F.4th 1180 (9th Cir. 2022).
- LinkedIn sent hiQ a cease-and-desist for scraping public profiles. hiQ sought a preliminary injunction.
- The 9th Circuit held: scraping data from a public website (data accessible without authentication) is **generally not "without authorization" under the CFAA**.
- The 2022 remand reaffirmed that holding after the Supreme Court's *Van Buren* decision.
- The case ultimately settled in late 2022; hiQ was enjoined for trade-secret violations on a separate claim, but the CFAA-doesn't-apply-to-public-data principle held.

Practical takeaway: scraping a public, unauthenticated page is generally not a federal crime in the 9th Circuit. It is *not* a license — there are still trespass-to-chattels, copyright, and contract claims to worry about.

### 10.3 *Van Buren v. United States* — 593 U.S. ___ (2021)

- Supreme Court decision that narrowed the meaning of "exceeds authorized access" in the CFAA.
- A police officer accessed a government database he had legitimate access to, but for a purpose forbidden by department policy. The Court held this is *not* a CFAA violation.
- The reasoning is the "gates-up-or-down" interpretation: the CFAA criminalizes accessing parts of a system you're not authorized to enter, not accessing parts you are authorized to enter for a forbidden *reason*.
- For scraping: if a target's robots.txt says "no scrapers", that's a *purpose* restriction, not an access restriction. Under *Van Buren*, violating it is not a CFAA crime by itself.

### 10.4 State trespass-to-chattels claims

Where the CFAA narrowed, state law trespass-to-chattels remained.

- ***eBay, Inc. v. Bidder's Edge, Inc.***, 100 F. Supp. 2d 1058 (N.D. Cal. 2000) — foundational. Bidder's Edge ran an auction-aggregation crawler against eBay despite cease-and-desist letters. Court issued a preliminary injunction on a trespass-to-chattels theory, reasoning that the unwanted automated requests imposed real load and dispossession on eBay's servers.
- ***Craigslist Inc. v. 3Taps Inc.***, 942 F. Supp. 2d 962 (N.D. Cal. 2013) — Craigslist had explicitly revoked 3Taps' access via cease-and-desist and IP block. The court held that *continuing* to scrape after explicit revocation can support both CFAA and trespass-to-chattels claims. The case predates *hiQ* and *Van Buren*, but the trespass-to-chattels theory survives those decisions.

Practical takeaway: even after *hiQ*, an explicit cease-and-desist letter from a target is a serious escalation. Continuing to scrape after one — especially while evading IP blocks — is the fact pattern that produces the loss for the scraper.

### 10.5 DMCA — 17 U.S.C. § 1201

The DMCA's anti-circumvention provision criminalizes circumventing "technological measures" that control access to copyrighted works. Scraping behind a login or behind a captcha challenge can implicate § 1201.

In practice this is rarely the lead theory in a scraping case (operators reach for CFAA and trespass first), but it is available, and it raises the stakes when copyrighted content is involved.

### 10.6 Trade secret claims — Defend Trade Secrets Act (18 U.S.C. § 1836)

If the scraped data qualifies as a trade secret (e.g. proprietary pricing data, customer lists, algorithms), the federal DTSA and state UTSA equivalents are available. *hiQ* was eventually enjoined on a trade-secret theory after losing on CFAA.

### 10.7 Copyright on the data itself

- US copyright protects original *expression*, not facts. Per *Feist Publications, Inc. v. Rural Telephone Service Co.*, 499 U.S. 340 (1991), a database of facts is generally not copyrightable in the US.
- The compilation arrangement may be — if the selection or organization shows originality.
- US has **no sui generis database right** (unlike the EU — see §11).

This means: scraping a public phone-directory-style dataset of facts is generally not copyright infringement in the US, but reproducing the selection/arrangement of a curated compilation may be.

---

## 11. Legal Landscape — European Union

The EU is materially stricter than the US on two fronts: personal data and database rights.

### 11.1 GDPR — Regulation (EU) 2016/679

GDPR applies to processing of *personal data* of EU/EEA residents, regardless of where the processor is. "Personal data" is broad: any information relating to an identified or identifiable natural person. Names, emails, profile photos, comment text — all in scope.

Article 6 lays out the lawful bases for processing. For scraping, the relevant ones are:

- **Consent (Art 6(1)(a))** — almost never available for scraping (you can't realistically obtain consent from people on a public site).
- **Legitimate interest (Art 6(1)(f))** — the most plausible basis. Requires a *balancing test*: your legitimate interest, weighed against the rights and freedoms of the data subjects. Document this analysis. The EDPB has guidelines.
- **Public interest / official authority** — narrow; mostly for academic or journalistic work.

Even with a lawful basis, the data subject's rights still apply: information notice (Art 13/14), right of access (Art 15), right to erasure (Art 17), right to object (Art 21).

### 11.2 Database Directive — Directive 96/9/EC

The EU has a **sui generis database right** that does not exist in the US. Under Article 7, the maker of a database that required substantial investment in obtaining, verifying, or presenting its contents has a 15-year (renewable on substantial update) right to prevent extraction or re-utilization of substantial parts of the contents.

Practical implication: scraping a substantial portion of a curated EU database (job listings on a major board, a price-comparison catalog, etc.) can infringe the sui generis right *even when no individual fact is copyrightable*. This is a pure-EU concept that catches scrapers off guard.

### 11.3 DSM Directive — Directive (EU) 2019/790

The Digital Single Market Copyright Directive created two TDM (text and data mining) exceptions:

- **Article 3** — TDM for *scientific research* by research organizations and cultural-heritage institutions. Mandatory exception, no opt-out possible.
- **Article 4** — General TDM exception for *any* purpose, including commercial. Member states must provide it. **Crucially**, rightsholders can opt out by reserving the right "in an appropriate manner, such as machine-readable means in the case of content made publicly available online" (Art 4(3)). In practice this is what `robots.txt`, `noai`/`noindex` meta tags, and emerging standards like `ai.txt` are being read against.

Practical implication for scrapers in the EU:

- If you're scraping for TDM and the site has not opted out (no clear machine-readable reservation), Article 4 may apply.
- If the site *has* opted out (robots disallow, "noai" tags), Article 4 doesn't help and you're back to needing a different lawful basis.
- Article 3 is more permissive but only for qualifying research orgs.

### 11.4 National implementations

The directives are implemented by each member state in national law, and there's variation in scope, enforcement intensity, and which courts hear what kind of claim. Cross-border scrapers usually plan for the strictest member state likely to assert jurisdiction.

---

## 12. Legal Landscape — Beyond US/EU

### 12.1 United Kingdom

- Post-Brexit, the UK retained GDPR-equivalent law (UK GDPR + Data Protection Act 2018).
- Database right is preserved.
- Computer Misuse Act 1990 is the rough analogue of the CFAA. Section 1 ("unauthorised access") has been read fairly broadly historically.
- The Competition and Markets Authority has issued guidance on data scraping in competitive contexts, especially around price scraping. The thrust: scraping for competitive benchmarking is generally fine if it doesn't burden the target unreasonably; scraping that facilitates collusion or price-fixing is not.

### 12.2 Japan, Korea, Singapore

Generally permissive for public data, with caveats:

- Japan has a copyright exception (Article 30-4 of the Copyright Act, 2018 amendment) for non-enjoyment-purpose use of works, broadly interpreted to include TDM. This is one of the more scraper-friendly jurisdictions in the world for ML training corpora.
- South Korea has an unfair-competition framework (Unfair Competition Prevention Act) that has been used against scrapers in commercial-substitute scenarios.
- Singapore has a TDM exception in the Copyright Act 2021 that permits computational data analysis on lawfully accessed works.

### 12.3 China

More restrictive:

- **Cybersecurity Law (2017)** — broad obligations on operators of "critical information infrastructure", with extraterritorial reach.
- **PIPL (Personal Information Protection Law, 2021)** — China's GDPR-equivalent. Stricter consent requirements; cross-border transfer is heavily regulated.
- **Anti-Unfair Competition Law** — has been used against scrapers in cases involving competitive substitution.

Scraping Chinese targets, or scraping data on Chinese persons from outside China, is a separate legal posture and warrants its own analysis.

### 12.4 Australia, Canada

Common-law jurisdictions with frameworks similar to the US:

- No sui generis database right (similar to the US).
- Privacy law (Privacy Act in Australia, PIPEDA in Canada) imposes obligations on personal data processing, broadly aligned with GDPR principles but less stringent.
- Trespass-to-chattels and contract theories are the typical civil claims.

---

## 13. Personal Data on Scraped Content — GDPR / CCPA

A scrape can be technically permitted (no CFAA violation, no ToS contract claim, no copyright issue) and still create personal-data obligations. Once you store the data, you are a *processor* (or *controller*) of any personal data in it.

### 13.1 GDPR obligations attach on processing

If your scrape captures personal data of EU/EEA residents, GDPR obligations apply regardless of whether the scrape itself was legal:

- **Article 13/14** — provide notice to data subjects. Scraping is the Art 14 "data not collected from the data subject" path; the notice must be given "within a reasonable period after obtaining the personal data, but at the latest within one month."
- **Article 17** — right to erasure. If a subject requests deletion, you must comply (within the exceptions of Art 17(3)).
- **Article 32** — security of processing. Encryption, access control, breach response.
- **Article 33** — breach notification within 72 hours of awareness.
- **Article 5(1)(c)** — data minimization. Store only what you need for the stated purpose.
- **Article 5(1)(e)** — storage limitation. Delete when no longer needed.

### 13.2 CCPA / CPRA — California Civil Code §1798.100 et seq.

California's framework is structurally similar to GDPR but with a different threshold (revenue-based applicability) and different rights (right to know, right to delete, right to opt-out of sale/sharing, right to limit use of sensitive personal information).

CCPA applies if you do business in California and meet the size thresholds; it doesn't automatically apply to every scraper, but for any commercial scrape of US-facing content, it's likely in scope.

### 13.3 Operational implications

Once you accept that scraped data may contain personal data:

- You need a **deletion endpoint or process** that can find and remove a specific person's records on request.
- You need a **retention policy** that auto-expires data you no longer need.
- You need **access controls** on the corpus.
- You need a **breach playbook** that fits a 72-hour notification window.

These are not optional bolt-ons; they are the same data-protection plumbing any backend service handling personal data needs.

---

## 14. Terms of Service and ToS-as-Contract Analysis

ToS (Terms of Service) prohibitions on scraping are nearly universal on commercial sites. Whether ToS violation alone is actionable is jurisdiction-specific and contract-specific.

### 14.1 Click-wrap vs. browse-wrap

- **Click-wrap** — user actively clicks "I agree" before using the service. Generally enforceable in US courts.
- **Browse-wrap** — terms are linked from a footer, with no affirmative agreement. Enforceability depends on whether a reasonable user would have notice. Often *not* enforceable, especially when the link is tiny or below the fold.

For scraping, this matters: a scraper hitting public pages without ever clicking a "Sign up / I agree" flow is at most browse-wrap-bound, which is the weaker of the two.

### 14.2 Cases

- ***Facebook, Inc. v. Power Ventures, Inc.***, 844 F.3d 1058 (9th Cir. 2016) — Power Ventures aggregated social-network data. After Facebook sent a cease-and-desist, continued access could trigger CFAA. Importantly: the court held that a cease-and-desist *revoking authorization* turned subsequent access into "without authorization" under the CFAA. Predates *hiQ* and the public-data nuance, but the cease-and-desist mechanic is still influential.
- ***Ticketmaster L.L.C. v. Prestige Entertainment***, 306 F. Supp. 3d 1164 (C.D. Cal. 2018) — bots scraping ticket inventory at scale. Court allowed CFAA, breach-of-contract, and copyright claims to proceed past motion to dismiss. Notable for treating ToS violations as both a contract claim and (combined with anti-bot circumvention) a CFAA claim.

### 14.3 Practical posture

- A site's ToS prohibiting scraping is *evidence* against you in any later dispute, even if the ToS itself isn't directly enforceable.
- A cease-and-desist explicitly revoking access, followed by continued scraping, is the fact pattern most likely to lose. Don't ignore C&D letters.

---

## 15. Practical Legal Hygiene Checklist

For any scraping project beyond a personal experiment:

- [ ] **Scope memo**: a 1–2 page document recording what's being scraped, what the target is, what data classes are involved (personal data? trade secret? copyrighted expression?), what the scrape is being used for, and who the users of the resulting data are.
- [ ] **ToS review**: read the target's ToS and record what it says about scraping. Note specifically: prohibitions, data-use restrictions, rate-limit clauses, license grants.
- [ ] **Robots.txt posture**: are you respecting it? If not, document why, and reconsider.
- [ ] **Personal data**:
  - Are you collecting personal data?
  - Are you minimizing collection (Art 5(1)(c) GDPR)?
  - Do you have a deletion procedure for Art 17 / CCPA requests?
  - Do you have a retention policy and a way to auto-expire?
- [ ] **Rate limiting**: can you defend the burden you're placing on the target? Document the per-host budget. *eBay v. Bidder's Edge* turned partly on the question of measurable load on the target's servers.
- [ ] **Documentation**: keep a scrape-log (what was fetched, when, from which IPs) and a reasoned legal-basis memo. If a dispute later arises, the contemporaneous record matters.
- [ ] **Cease-and-desist response plan**: name a person who handles legal correspondence and a process for stopping the scrape immediately if a C&D arrives.
- [ ] **Cross-border posture**: if scraping EU subjects, an Art 6 lawful-basis analysis. If scraping UK subjects, the UK GDPR equivalent. If scraping Chinese subjects, a PIPL analysis.

This is operational hygiene, not bulletproof defense. It's what a reasonable person does to be defensible if challenged.

---

## 16. When to Escalate to Actual Lawyers

Some scraping contexts are not ones to navigate from a learning doc. Bring a real attorney into the loop when any of the following apply:

- **Competitive intelligence on a competitor's customer data**, especially their pricing, customer lists, or contracts.
- **Anything in a regulated industry** — financial services (SEC, FINRA), healthcare (HIPAA), education (FERPA), kids' content (COPPA, GDPR-K).
- **Anything where you intend to publish or commercialize** the scraped corpus — sell it, train a model and release the model, embed it in a product. The economic-substitute analysis under copyright fair use, and the sui generis database right in the EU, both kick in here.
- **Scraping behind any kind of authentication** — login walls, paywalls, paid APIs. The DMCA § 1201 exposure and the *Power Ventures* "authorization revoked" mechanic both lurk here.
- **A cease-and-desist letter has arrived**.
- **You receive a GDPR data-subject access request, an Art 17 erasure request, or a CCPA "do not sell" request** and you don't have a process for it.
- **Cross-border situations** where you're scraping a target in jurisdiction A with infrastructure in jurisdiction B serving users in jurisdiction C. Conflicts-of-law questions show up fast.

The cost of an hour of lawyer time at the start of a project is dramatically lower than the cost of litigation later.

---

## 17. Bug-Bounty Contexts

A specific narrow context where scraping intersects with security work: bug bounty programs. Most published bounty rules contain clauses that are de-facto limits on scraping during research:

- **"No harm" clause** — don't degrade service for other users. Implies rate limits during testing; treat the target like production traffic, not a load test.
- **"Reasonable use"** — typically interpreted as "test enough to confirm the bug, then stop." Long-running automated scans across endpoints is usually outside the spirit of the clause and sometimes outside the letter.
- **Personal data minimization** — if you stumble across PII while testing, the standard expectation is: do not exfiltrate, do not share, report immediately, and the program operator will direct destruction.
- **Reporting timeline** — typical published expectations are 24–72 hours for critical issues. Read the specific program's policy; it's a contract.
- **Scope** — bounty programs publish in-scope and out-of-scope assets. Scraping or testing out-of-scope assets is not covered, and "I was only researching" is usually not a defense.

The general principle: a bounty program is an invitation to test specifically the things it lists, in specifically the way the rules describe. It is not a general license to scrape, automate, or stress-test the target.

---

## 18. Related

- [Sitemap, robots.txt, and Crawl Surface](../discovery/03-sitemap-robots-and-crawl-surface.md) — the discovery side of the politeness coin
- [HTTP Scraping Fundamentals](../extraction/06-http-scraping-fundamentals.md) — the request-shape and parsing layer this doc builds on
- [Proxies and IP Rotation](../extraction/08-proxies-and-ip-rotation.md) — proxy-tier costs and ASN block patterns referenced here
- [Bot Detection Internals](../extraction/09-bot-detection-internals.md) — what the target side is doing, which is why politeness matters
- [Database learning path](../../database/INDEX.md) — storing scraped corpora (PostgreSQL, ClickHouse, schema design)
- [Observability learning path](../../observability/INDEX.md) — RED/USE metrics, OpenTelemetry, SLO design for scraper fleets
- [System Design learning path](../../system-design/INDEX.md) — distributed queues, dedup architecture, frontier sharding

---

## 19. References

### Statutes and Regulations

- **18 U.S.C. § 1030** — Computer Fraud and Abuse Act. https://www.law.cornell.edu/uscode/text/18/1030
- **18 U.S.C. § 1836** — Defend Trade Secrets Act. https://www.law.cornell.edu/uscode/text/18/1836
- **17 U.S.C. § 1201** — DMCA anti-circumvention. https://www.law.cornell.edu/uscode/text/17/1201
- **California Civil Code § 1798.100 et seq.** — California Consumer Privacy Act (CCPA), as amended by CPRA. https://leginfo.legislature.ca.gov/faces/codes_displayText.xhtml?division=3.&part=4.&lawCode=CIV&title=1.81.5
- **Regulation (EU) 2016/679** — General Data Protection Regulation (GDPR). https://eur-lex.europa.eu/eli/reg/2016/679/oj
- **Directive 96/9/EC** — Database Directive (sui generis database right). https://eur-lex.europa.eu/eli/dir/1996/9/oj
- **Directive (EU) 2019/790** — Digital Single Market Copyright Directive (Articles 3 and 4 — TDM exceptions). https://eur-lex.europa.eu/eli/dir/2019/790/oj
- **UK Computer Misuse Act 1990**. https://www.legislation.gov.uk/ukpga/1990/18/contents
- **Japan Copyright Act, Article 30-4**. https://www.japaneselawtranslation.go.jp/en/laws/view/4207
- **PRC Personal Information Protection Law (PIPL)**. https://www.npc.gov.cn/npc/c30834/202108/a8c4e3672c74491a80b53a172bb753fe.shtml

### Cases (US)

- ***hiQ Labs, Inc. v. LinkedIn Corp.***, 938 F.3d 985 (9th Cir. 2019); 31 F.4th 1180 (9th Cir. 2022). The 2022 opinion on remand following *Van Buren*.
- ***Van Buren v. United States***, 593 U.S. ___, 141 S. Ct. 1648 (2021). https://www.supremecourt.gov/opinions/20pdf/19-783_k53l.pdf
- ***eBay, Inc. v. Bidder's Edge, Inc.***, 100 F. Supp. 2d 1058 (N.D. Cal. 2000). Foundational trespass-to-chattels case for crawler load.
- ***Craigslist Inc. v. 3Taps Inc.***, 942 F. Supp. 2d 962 (N.D. Cal. 2013). Continued scraping after explicit revocation.
- ***Facebook, Inc. v. Power Ventures, Inc.***, 844 F.3d 1058 (9th Cir. 2016). Cease-and-desist as authorization revocation.
- ***Ticketmaster L.L.C. v. Prestige Entertainment, Inc.***, 306 F. Supp. 3d 1164 (C.D. Cal. 2018). ToS, CFAA, and copyright in combination.
- ***Feist Publications, Inc. v. Rural Telephone Service Co.***, 499 U.S. 340 (1991). Facts are not copyrightable in the US.

### Standards

- **RFC 9309** — Robots Exclusion Protocol. https://www.rfc-editor.org/rfc/rfc9309.html
- **RFC 9110** — HTTP Semantics (including `Retry-After`, conditional requests). https://www.rfc-editor.org/rfc/rfc9110.html
- **RFC 3986** — URI Generic Syntax (canonicalization rules). https://www.rfc-editor.org/rfc/rfc3986.html
- **WHATWG URL Standard**. https://url.spec.whatwg.org/

### Implementation references

- **Scrapy request fingerprinting** — `scrapy.utils.request.fingerprint`. https://docs.scrapy.org/en/latest/topics/request-response.html#request-fingerprints and source at https://github.com/scrapy/scrapy/blob/master/scrapy/utils/request.py
- **w3lib URL canonicalization** — `w3lib.url.canonicalize_url`. https://w3lib.readthedocs.io/en/latest/w3lib.html#w3lib.url.canonicalize_url
- **EDPB Guidelines on Article 6(1)(f) (legitimate interest)**. https://edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-012024-processing-personal-data-based-article_en
- **Bloom Filter — Burton H. Bloom, "Space/Time Trade-offs in Hash Coding with Allowable Errors"**, Communications of the ACM, 13(7), 1970.
- **AWS Architecture Blog — "Exponential Backoff and Jitter"**. https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
