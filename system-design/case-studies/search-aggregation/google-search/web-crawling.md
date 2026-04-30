---
title: "Google Search Deep Dive — Web Crawling at Scale"
date: 2026-04-30
updated: 2026-04-30
tags: [system-design, case-study, google-search, deep-dive, crawler, freshness]
---

# Google Search Deep Dive — Web Crawling at Scale

**Date:** 2026-04-30 | **Updated:** 2026-04-30
**Tags:** `system-design` `case-study` `google-search` `deep-dive` `crawler` `freshness`

## Summary

Googlebot is the most-watched crawler on the planet and the one with the most adversarial environment: every webmaster wants more of it (more crawl = more chances to rank), every CDN wants less of it (more crawl = more bandwidth bill), and every spammer wants to fool it. The parent doc — [Design Google Search](../design-google-search.md) — sketched the crawl tier in a few paragraphs. This companion goes deep on the moves that actually make a planet-scale crawler safe, fast, fresh, and dedup-correct: how the **URL frontier** is prioritized so high-value pages re-fetch in minutes while long-tail pages re-fetch in months, how **politeness** is enforced per-host (not per-IP, not per-page), how **HTTP/2 connection reuse** changes the politeness math, how **canonicalization + content fingerprinting** collapse the long tail of duplicate URLs, why **sitemaps** are a hint and not a contract, why **mobile-first indexing** rewired the whole rendering pipeline in 2018–2024, and the brutal cost reality of running a **rendering tier** that executes JavaScript for hundreds of billions of URLs. None of this is theoretical — it is documented behavior in Google Search Central plus the public crawler-engineering literature (Mercator, BUbiNG, Heritrix, Common Crawl) that Google's design echoes.

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [Googlebot Architecture](#googlebot-architecture)
- [URL Frontier and Prioritization](#url-frontier-and-prioritization)
- [Politeness — robots.txt and Crawl-Delay](#politeness--robotstxt-and-crawl-delay)
- [Per-Host Rate Limiting](#per-host-rate-limiting)
- [HTTP/2 Connection Reuse](#http2-connection-reuse)
- [Deduplication — Canonical URLs and Content Hashing](#deduplication--canonical-urls-and-content-hashing)
- [Freshness vs Depth — Priority Queues by Segment](#freshness-vs-depth--priority-queues-by-segment)
- [User-Agent Identification and Verification](#user-agent-identification-and-verification)
- [Sitemap Ingestion](#sitemap-ingestion)
- [Mobile-First Indexing](#mobile-first-indexing)
- [JS Rendering and the Rendering Tier](#js-rendering-and-the-rendering-tier)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Overview

Googlebot is best understood as **three loosely-coupled pipelines** running over a shared frontier and a shared content store:

1. **Discovery and fetch.** Pull URLs from the frontier, fetch them respecting politeness and robots, store the raw bytes, hand off the parsed body for link extraction.
2. **Render.** For URLs flagged as JS-dependent, send the fetched HTML through a headless Chromium pool, capture the post-render DOM, and feed *that* into indexing.
3. **Schedule re-fetch.** Score each known URL against a model of its change rate and importance, and choose when (if ever) to come back.

These pipelines share a single source of truth: the **URL Database** (Google's internal name varies — historically called the "URLserver" lineage from the original 1998 paper). Every URL Google has ever heard of has a record there with status, last-fetched, last-modified, content fingerprint, computed priority, and the host it belongs to.

The flow that produces 99% of the bandwidth bill is the discovery-and-fetch pipeline. The flow that determines whether the index is *useful* is the scheduler. The flow that determines whether modern JS-heavy sites are *visible at all* is the renderer. Each of these breaks differently at scale, and each is the focus of its own section below.

## Googlebot Architecture

```text
                         ┌────────────────────────────────────────────┐
                         │              URL Database                   │
                         │  url_id → (host, path, status, last_fetch,  │
                         │   last_modified, content_hash, priority,    │
                         │   robots_allowed, render_required, …)       │
                         └─────────────┬───────────────┬───────────────┘
                                       │               │
                  reads / updates      │               │  re-fetch decisions
                                       ▼               ▼
   ┌──────────────────────────┐   ┌─────────────────────────────────┐
   │      URL Frontier        │   │   Re-fetch Scheduler            │
   │ (per-host queues, prio.) │◀──┤  (change-rate × importance)     │
   └──────┬───────────────────┘   └─────────────────────────────────┘
          │ pop URL respecting politeness
          ▼
   ┌────────────────────┐    fetched    ┌──────────────────────────┐
   │  Fetcher Pool      │──────HTML────▶│  Parser / Link Extractor │
   │  (HTTP/2, robots,  │               │  (canonicalize, hash,    │
   │   per-host caps)   │               │   extract <a>, sitemaps) │
   └─────┬──────────────┘               └────────┬─────────────────┘
         │ JS-required flag                      │ new URLs
         ▼                                       ▼
   ┌───────────────────────────┐         (back into frontier
   │  Rendering Tier           │          via URL Database)
   │  (headless Chromium pool, │
   │   post-render DOM capture)│
   └───────────────────────────┘
```

A few things to internalize before the deep dives:

- **The frontier is not a queue.** It is *thousands* of per-host queues with a shared scheduler that picks which host to serve next under a global politeness budget.
- **The fetcher does not decide priority.** It reads from whichever host queue the scheduler says is up next.
- **Rendering is asynchronous.** Pages flagged for render are queued; they do not block the fetch pipeline. This is why JS-only content can show up in Google's index *days* after the raw HTML did, even when the HTML fetched in seconds.
- **There is no single Googlebot instance.** Crawler workers run in dozens of clusters worldwide. The frontier is sharded by host so all fetches for `example.com` land on a single shard, which makes per-host pacing tractable.

## URL Frontier and Prioritization

The frontier is the data structure that answers "which URL should we fetch next?" At Google's scale (hundreds of billions of known URLs, low-trillions counting historical), it cannot be a flat priority queue.

### Mercator-style two-level structure

The pattern Google's frontier follows is well-described in Heydon and Najork's [Mercator paper (1999)](https://www.cs.cornell.edu/courses/cs685/2002fa/mercator.pdf), which is the *direct ancestor* of all modern web crawlers:

- **Front queues** — partitioned by *priority class* (e.g., "must fetch within 10 minutes," "fetch within an hour," "fetch eventually"). A scheduler pulls from front queues in priority order.
- **Back queues** — partitioned by *host*. Each back queue holds URLs for one host. A worker that pops from a back queue is guaranteed to be talking to one host at a time, which lets the politeness governor enforce per-host pacing trivially.

URLs flow front → back. The number of back queues equals the number of crawler worker threads, so each worker has one host to itself at any moment, and host assignment is rotated as queues drain.

### Priority signals

Google's crawl-budget documentation ([Crawl Budget Management](https://developers.google.com/search/docs/crawling/large-site-managing-crawl-budget)) calls out the inputs explicitly:

| Signal | Direction | Why |
|---|---|---|
| **PageRank-class importance** | higher = sooner | Important pages should be fresh more often |
| **Observed change rate** | higher change → sooner | News-y pages need short re-fetch intervals |
| **Internal link prominence** | more inbound site links → sooner | Hub pages matter more than orphans |
| **Freshness budget for the host** | host-level cap | Don't blow the whole budget on one site |
| **Crawl health / 5xx rate** | bad health → less crawl | Back off on struggling origins |
| **Sitemap priority hint** | weak nudge | Hint, not contract |
| **Discovery recency** | brand-new URL gets a one-shot priority bump | Surface new content fast |

The product of these is a per-URL `next_fetch_due_ts` plus a tiebreak score. The scheduler picks the smallest `next_fetch_due_ts` that the politeness budget allows.

### Why importance dominates change rate

A naive scheduler that fetches "whatever changed most recently" gets eaten alive by infinite-calendar pages, query-string trap URLs, and farm sites that shake their content every minute to game freshness. Google explicitly pairs change-rate with importance, so a site nobody links to does not get the same crawl rate as the BBC just because it edits its homepage every 30 seconds.

### Capacity envelope at the planet scale

The scheduler operates against a fixed total budget — fetcher fleet capacity is finite. Order-of-magnitude:

- **~200B+ known URLs** in the URL Database (estimated; Google has not published a recent number — the 2016 figure was 130 trillion *unique* URLs ever discovered, of which only a fraction are kept active).
- **Tens of billions of URLs** in the *active* re-fetch rotation.
- **Hundreds of billions of fetches per day** across all Googlebot variants combined.

Even at that throughput, the long-tail web cannot be re-crawled daily. The scheduler must be ruthless about which URLs *actually deserve* a re-fetch slot today vs next week vs never again.

## Politeness — robots.txt and Crawl-Delay

`robots.txt` is the contract between crawlers and webmasters. Google's compliance is documented in [How Google Interprets the robots.txt Specification](https://developers.google.com/search/docs/crawling-indexing/robots/robots_txt) and standardized in [RFC 9309](https://www.rfc-editor.org/rfc/rfc9309.html).

### Fetch and cache rules

- robots.txt is fetched at `https://<host>/robots.txt` (HTTP-specific; per RFC 9309 the protocol scheme matters — `http://` and `https://` are distinct robots scopes).
- It is cached **per-host for up to 24 hours**, but can be re-fetched sooner if the host has high crawl traffic.
- A 4xx (other than 429) is treated as "no robots restrictions." A 5xx is treated as "all crawling disallowed temporarily" — a conservative bias that protects the origin during outages. Google's docs are explicit: "If we can't fetch your robots.txt, we'll treat the site as fully disallowed for a short time."
- A redirect chain is followed up to 5 hops; after that, treated as 404.

### Directives Google actually honors

| Directive | Honored | Notes |
|---|---|---|
| `User-agent` | yes | Specific match wins over wildcard `*` |
| `Disallow` | yes | Path prefix; supports `*` and `$` per Google's extension |
| `Allow` | yes | Most-specific path match wins on conflicts |
| `Sitemap` | yes | Triggers sitemap ingestion |
| `Crawl-delay` | **no** for Googlebot | Use Search Console crawl-rate setting instead |
| `Host` | no | Yandex extension; Google ignores |
| `Noindex` | no | Removed from robots.txt support in 2019; use `<meta name="robots">` or `X-Robots-Tag` |

The `Crawl-delay` non-support is a practical, not philosophical, choice: Google's per-host rate limiter is adaptive and feedback-driven (see next section), and a static `Crawl-delay: 30` from a webmaster who hasn't thought about it for five years would systematically cap crawl rate well below what the site can actually handle.

### Robots.txt is not authentication

`Disallow` keeps a well-behaved crawler off a path. It does **not** keep the URL out of search results — Google can still list a URL it never fetched if other pages link to it with anchor text, displaying just the URL and the link's snippet. To remove a URL from results, use `noindex` (which requires the page to be *fetchable*) or password-protect it.

## Per-Host Rate Limiting

Per-host rate limiting is where the politeness rubber meets the throughput road.

### What "per-host" means at Google's scale

The unit of rate-limiting is the **registered domain** (`example.com`), not the IP, not the subdomain, and not the URL prefix. Why:

- IP-level limiting punishes shared hosting. A cheap VPS might serve 1,000 unrelated sites; rate-limiting them as one would starve them all.
- Subdomain-level limiting (`shop.example.com` vs `blog.example.com`) is too granular. Google does in fact track host-level signals separately for important subdomains, but the politeness budget aggregates up.

### Adaptive rate, not fixed

The actual per-host fetch rate is computed from a feedback loop:

1. **Start conservative.** A new host gets a low crawl rate (e.g., a small number of fetches per second).
2. **Probe.** As fetches succeed quickly with low latency, ramp up.
3. **Back off on signals.**
   - HTTP 429 → cut rate sharply, honor `Retry-After`.
   - HTTP 5xx → cut rate; if persistent, pause the host.
   - Latency spike (origin getting slow) → cut rate proportionally.
   - Search Console manual cap → respect.
4. **Re-probe periodically.** A host that earned a 5xx-induced timeout will get tested again later.

This is essentially **AIMD** (additive-increase, multiplicative-decrease, the same algorithm that runs TCP congestion control), tuned for hours-to-days time scales rather than milliseconds.

### Crawl budget vs crawl capacity

Google distinguishes:

- **Crawl capacity limit** — what the host can serve without degrading. Determined by site health.
- **Crawl demand** — how badly Google wants to crawl this site. Determined by site importance, freshness needs, and discovered-but-unfetched URL count.

The actual crawl rate is `min(capacity, demand)`. Webmasters whose sites are crawled "too little" usually have a *demand* problem (low importance, low change rate); webmasters whose sites are crawled "too much" usually have a *capacity* problem the crawler is correctly probing for.

## HTTP/2 Connection Reuse

Googlebot has supported HTTP/2 since November 2020 ([announcement on Search Central](https://developers.google.com/search/blog/2020/09/googlebot-will-soon-speak-http2)). The shift fundamentally changes the politeness math.

### Why HTTP/2 matters for crawling

Under HTTP/1.1, each fetch is one request per TCP connection (or with keep-alive, sequentially per connection). To fetch N pages on a host concurrently, the crawler opens N connections — which the politeness governor caps low (typically 1–2 concurrent connections per host) to avoid hammering origins.

HTTP/2 multiplexes many requests on **one** connection. Googlebot can pull dozens of resources from a host simultaneously without opening additional connections. Origin server sees one TCP connection but a stream of requests; bandwidth is the only real bottleneck.

### What changes for webmasters

- **Fewer TLS handshakes** — Googlebot reuses one HTTPS connection for many requests, cutting CPU and round-trips at the origin.
- **Higher effective fetch rate per host** — without opening new connections.
- **Server push is mostly ignored.** Google does not rely on HTTP/2 server push for indexing (Chrome itself disabled it). Don't push.
- **Header compression (HPACK)** — small wins for crawl bandwidth on repeated headers.

### The dark side

A single HTTP/2 connection can keep an origin's worker pool saturated more efficiently than a few HTTP/1.1 connections did. Origins running per-connection limits (e.g., max 100 concurrent requests per connection in some servers) can be DoS-shaped by an aggressive HTTP/2 client. Googlebot self-throttles concurrency per connection; site owners running their own crawlers must do the same.

### What about HTTP/3 and QUIC

Googlebot does not (as of this writing) use HTTP/3 for indexing fetches. HTTP/3 over QUIC offers per-stream loss isolation (one dropped packet doesn't head-of-line-block sibling streams the way it does in HTTP/2 over TCP), which is mainly a win for high-RTT, lossy networks. Crawl traffic is mostly fat-pipe origin-to-Google, so HTTP/2 is sufficient. The connection-reuse posture is what matters; the wire protocol underneath is implementation detail.

## Deduplication — Canonical URLs and Content Hashing

Most URLs on the web don't point to unique content. Pagination variants, session-ID query strings, mobile vs desktop subdomains, syndicated content, mirrors, and outright scrapers mean that for every 10 URLs Googlebot fetches, maybe 2–3 yield genuinely new content. The dedup pipeline pays for itself many times over.

### Layer 1 — URL canonicalization

Per [Google's URL canonicalization docs](https://developers.google.com/search/docs/crawling-indexing/canonicalization), canonicalization happens **at the URL level**, before fetch where possible:

- Strip well-known tracking parameters (`utm_*`, `fbclid`, `gclid`, etc.).
- Normalize trailing slashes (host root and directory paths).
- Lowercase the host (URLs are case-sensitive on the path; hostnames are not).
- Resolve uppercase percent-encoding to lowercase.
- Normalize default ports (`:80`, `:443`).
- Remove fragment (`#`) — fragments are client-side; the server sees them stripped.
- Follow `<link rel="canonical">` from the fetched HTML.
- Honor cross-domain canonicals only with strong corroborating signals (link patterns, content match) — Google explicitly distrusts spammy cross-domain canonicals.

The canonical URL becomes the storage key. Every variant URL points to it.

### Layer 2 — Content fingerprinting

Even after canonicalization, near-duplicate content remains: scraped articles, syndicated press releases, boilerplate-heavy templated pages. Two techniques run side by side:

- **Exact-match content hash.** SHA-256 (or similar) of the normalized body. If the hash is already known, the URL is recorded as a duplicate of the existing canonical.
- **Near-duplicate fingerprint — SimHash.** Manku, Jain, and Sarma's [Detecting Near-Duplicates for Web Crawling](https://research.google.com/pubs/archive/33026.pdf) (WWW 2007) is the foundational paper. SimHash compresses a document to a 64-bit fingerprint such that documents differing in a small number of words have fingerprints differing in a small number of bits. A bit-permutation index enables O(1) near-neighbor lookup at scale.

When two URLs collapse to the same SimHash bucket, ranking signals (PageRank, anchor text, link count) merge under the canonical, and only one document carries forward into indexing.

### Layer 3 — Bloom filters at the frontier

Before enqueueing a freshly-discovered URL, Googlebot tests it against a Bloom filter of "URLs we've already seen." False positives are tolerable — at worst, a URL doesn't get re-enqueued this minute and is picked up later. False negatives cannot happen by Bloom design. The memory savings are massive: a Bloom filter for 100B URLs at 1% false positive rate costs ~120 GB; storing the full URL set in a hash table would cost terabytes.

## Freshness vs Depth — Priority Queues by Segment

The crawl problem cleaves naturally into two regimes that need very different scheduling.

### News-class freshness (minutes)

Breaking news must surface *fast*. The frontier carries a **fresh-tier priority queue** for hosts and URL patterns flagged as news-like:

- News publishers identified via Google News partnership and editorial signals.
- URL patterns matching observed news structures (e.g., `/2026/04/29/...`, `/article/...`).
- Hosts with high observed change rate and high importance.

This tier feeds into the **fresh index** described in the parent doc — small, fast-updating, sub-minute crawl cadence. It runs against a tiny share of the web (millions of URLs, not billions).

### Deep-tail discovery (weeks to months)

A long-tail blog last edited in 2014 does not need to be re-fetched daily. The deep-tail scheduler uses an **exponentially-weighted change-rate model**: if the last 5 fetches yielded the same content hash, push `next_fetch_due_ts` further out. If a fetch finally produces new content, snap the interval back down.

Pages can sit idle for months and that is *correct*. The crawl budget should not be spent confirming that a 2014 blog post still says what it said in 2014.

### Why segmenting matters

If you put news and blog-archive into the same priority queue, the news pages starve everything else (high priority + frequent change drives them to the front constantly), or you cap news priority and miss breaking stories. Segmenting acknowledges that the two regimes are fundamentally different distributions and gives each its own budget.

### Implementation

In practice this looks like multiple priority queues feeding a single fetcher pool, with the scheduler weighting how often it drains each:

```text
fresh tier queue       ─┐
news-vertical queue    ─┤
deep-tail queue        ─┼──▶ scheduler ──▶ per-host back queues ──▶ fetchers
sitemap-prompt queue   ─┤
recrawl-due queue      ─┘
```

The weights are tuned so that, e.g., 30% of fetcher capacity is reserved for fresh-tier, 60% for deep-tail recrawl, 10% for newly-discovered URLs.

## User-Agent Identification and Verification

Googlebot's user-agent strings are listed authoritatively in [Overview of Google crawlers](https://developers.google.com/search/docs/crawling-indexing/overview-google-crawlers). Major variants:

- **Googlebot Smartphone** (mobile-first, the primary crawler since 2018).
- **Googlebot Desktop** (small-share complement).
- **Googlebot Image / Video / News** (vertical crawlers).
- **AdsBot, Mediapartners-Google** (ad-related, separate budgets).
- **Google-InspectionTool** (manual URL inspection from Search Console).

### Why you can't trust the User-Agent header alone

Anyone can claim to be Googlebot. Spammers and scrapers do, hoping to be allow-listed past WAFs. Google publishes explicit verification guidance in [Verifying Googlebot](https://developers.google.com/search/docs/crawling-indexing/verifying-googlebot):

1. Reverse-DNS the source IP. The PTR record must be in `googlebot.com`, `google.com`, or `googleusercontent.com`.
2. Forward-DNS the resulting hostname. The result must include the original IP.
3. Optionally, cross-check against the published [Googlebot IP ranges JSON](https://developers.google.com/search/apis/ipranges/googlebot.json).

Sites that do firewalling-by-user-agent without this check expose themselves to spam and miss legitimate Googlebot fetches when scrapers exhaust the User-Agent reputation.

### What a real Googlebot request looks like

```text
GET /article/123 HTTP/2
:authority: example.com
user-agent: Mozilla/5.0 (Linux; Android 6.0.1; Nexus 5X Build/MMB29P) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/W.X.Y.Z Mobile Safari/537.36 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)
accept-encoding: gzip, deflate, br
from: googlebot(at)googlebot.com
```

The `+http://www.google.com/bot.html` URL is contact info for site owners; the `From:` header is part of the etiquette baked in since the early-2000s NCSA crawler era.

## Sitemap Ingestion

The sitemap protocol ([sitemaps.org](https://www.sitemaps.org/protocol.html)) is a hint mechanism for discovery. It is **not** a crawl directive.

### What sitemaps are good for

- **Discovery of orphan URLs.** Pages with no inbound links would never be found via link-following alone; a sitemap surfaces them.
- **Hint about importance.** The `<priority>` field nudges the scheduler.
- **Hint about change frequency.** The `<changefreq>` field nudges the recrawl scheduler.
- **Hint about last modification.** The `<lastmod>` field tells Googlebot whether it needs to re-fetch.
- **Cross-protocol/host signals.** A sitemap can declare URLs across subdomains (with verified ownership in Search Console).

### What sitemaps are *not* good for

- **Forcing crawl.** A sitemap entry that points at a low-quality page won't get crawled fast just because it's listed.
- **Bypassing robots.txt.** Sitemap ≠ allow-list.
- **Deduplication.** Listing the same content under multiple URLs in a sitemap doesn't make Google pick a different canonical.

### Discovery channels

Google finds sitemaps through several paths:

1. **`Sitemap:` directive in `robots.txt`.** Preferred — automatic.
2. **Submission via Search Console.**
3. **Standard locations.** Googlebot probes `/sitemap.xml` and `/sitemap_index.xml` even when not declared.
4. **Sitemap index files.** Sitemaps can chain — a sitemap-index lists multiple sitemaps; this is required at the 50,000-URL / 50 MB per-file limit.

### Format limits and gotchas

Per the [protocol spec](https://www.sitemaps.org/protocol.html) and [Google's sitemap docs](https://developers.google.com/search/docs/crawling-indexing/sitemaps/overview):

- Max 50,000 URLs per sitemap file.
- Max 50 MB uncompressed per sitemap file.
- gzip is supported and encouraged.
- Up to 500 sitemap files per sitemap-index.
- `<lastmod>` must be a valid W3C Datetime; Google ignores it if it's clearly bogus (e.g., always "now").

### News sitemaps are a stronger signal

The [News sitemap extension](https://developers.google.com/search/docs/crawling-indexing/sitemaps/news-sitemap) is treated as a higher-priority hint — articles published in the last 48 hours, with publication metadata. This is one of the few cases where a sitemap genuinely accelerates crawl.

## Mobile-First Indexing

In July 2019 Google announced [mobile-first indexing by default for new sites](https://developers.google.com/search/blog/2019/05/mobile-first-indexing-by-default-for); migration of the rest of the web finished in October 2023. The crawl-tier consequence is that **the Smartphone Googlebot is the primary crawler** for almost every site, with the desktop crawler running in a much smaller complement role.

### What changed

- The User-Agent on most fetches advertises the smartphone Chromium build.
- The viewport on rendered pages defaults to mobile.
- Indexing uses the *mobile* version of a page's content as the source of truth. If the mobile version omits content the desktop version has, the omitted content does not exist for ranking purposes.
- Structured data, alt text, and metadata must be consistent between mobile and desktop variants.

### Why this is operationally significant

Pre-2018, sites with separate `m.example.com` mobile templates often shipped a stripped-down mobile version. Mobile-first indexing made this *catastrophic* for ranking: thinner content = lower ranking. The shift forced the industry toward **responsive design** (one HTML, CSS-driven layout), which is now the dominant pattern.

For Googlebot's crawl pipeline, the consequence is that the smartphone crawler is the high-traffic identity. The desktop crawler still runs (some sites serve genuinely different desktop content; Google still wants to know), but as a much smaller share of the budget.

## JS Rendering and the Rendering Tier

Modern web apps ship a thin HTML shell and assemble the actual page with JavaScript. Googlebot has to render JS or it will see empty `<div id="root">` for the entire SPA web. This was historically Google's weakest crawler capability (around 2014–2017 it was common for SPAs to be effectively invisible to Google), and the [Web Rendering Service (WRS)](https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics) is the answer.

### Two-pass indexing

The historical model was explicitly two-pass:

1. **First pass:** Fetch raw HTML. Index immediately based on what's in the static HTML. Extract links, enqueue.
2. **Second pass:** Send the URL to WRS. Render with headless Chromium. Capture the post-render DOM. Re-index using rendered content. Re-extract links.

The second pass could lag the first by **days to weeks** at peak. Sites whose primary content was JS-rendered would appear in Google with empty snippets initially, then fill in once WRS caught up.

Google has since [stated](https://developers.google.com/search/blog/2019/05/the-new-evergreen-googlebot) that the gap has shrunk dramatically — most pages render within minutes — but the architectural truth remains: **rendering is asynchronous and rate-limited separately from fetching.** When the rendering tier is saturated, the gap stretches.

### Why rendering is so expensive

A headless Chromium instance is *orders of magnitude* more expensive than a plain HTTP fetch:

| Cost dimension | Plain fetch | Rendered fetch | Multiplier |
|---|---|---|---|
| CPU per page | ~1 ms | 100–500 ms (page execution) | 100–500× |
| Memory | KB | 50–200 MB per instance | 50,000× |
| Network | One HTTP request | Many sub-resources (JS, CSS, fonts, images, XHR) | 10–50× |
| Time to "ready" | < 1 s | 3–10 s (waiting for `networkidle`, post-load XHR) | 5–10× |

At Google's URL volume (hundreds of billions known), rendering everything is impossible. The renderer is gated.

### Gating heuristics

Pages that get sent to WRS:

- Pages where the rendered DOM differs meaningfully from the raw HTML (detected by content fingerprint comparison on a sample).
- Pages on hosts known to be JS-heavy (single-page-app frameworks detected from prior renders).
- Pages flagged via signals like `<meta name="fragment">` (legacy AJAX crawling) or where structured data only appears post-render.
- Pages with priority high enough to justify the cost.

Pages that *don't* get rendered:

- Pure-HTML pages with no JS-driven content.
- Low-importance pages where the raw HTML is sufficient.
- Pages above the host's rendering budget for the period.

### Rendering tier scale

Google does not publish exact numbers. Public hints (Search Off the Record podcasts, Search Central Live talks):

- The renderer is an "evergreen" Chromium — Googlebot's Chromium version tracks stable Chrome (since 2019; before that it was stuck on Chrome 41 for years and was a major source of SEO pain).
- Resources are shared across pages where possible — JS bundles fetched once are reused.
- 5-second client-side budget for "main content" rendering is a reasonable mental model for SEO planning; pages that take longer may be indexed before render finishes.

### What this means for site owners

- **Ship critical content in server-rendered HTML.** SSR or SSG (Next.js, Nuxt, Astro, Remix) is dramatically more SEO-friendly than CSR-only.
- **Don't gate content behind user interaction.** Click-to-load tabs may not get rendered.
- **Avoid blocking JS in robots.txt.** A common SEO disaster: `Disallow: /static/js/` blocks the renderer from fetching the very files needed to assemble the page.
- **Test with the [URL Inspection tool](https://support.google.com/webmasters/answer/9012289) in Search Console.** It shows what WRS actually sees.

### Resource sharing across renders

Even at orders-of-magnitude cost-per-page, rendering becomes tractable through aggressive caching. The renderer:

- Caches static JS bundles, CSS, fonts, and images by URL + content hash, reused across pages on the same site for hours.
- Skips third-party scripts that are non-essential (analytics, ads) when their absence wouldn't change the rendered DOM.
- Uses a shared Chromium process pool with per-page sandboxing, not a fresh process per page.

A site that ships a 2 MB JS bundle pays for the parse-and-execute cost on the *first* render; subsequent pages on the same site reuse the warm bundle.

### When rendering goes wrong

The most common rendering-tier failure modes site owners hit:

1. **Long-running JS that never resolves to "ready."** Googlebot's renderer has a budget; pages that wait on external XHRs that never complete get captured at the budget boundary, missing late-loaded content.
2. **Errors thrown during render.** Uncaught exceptions can short-circuit the post-load chain. The renderer captures errors and moves on, but the page is indexed in its broken state.
3. **Differential responses by User-Agent in JS.** A site that gates content based on `navigator.userAgent` may serve degraded content to the smartphone Googlebot, then complain that it isn't being indexed correctly. The fix is universal markup, not UA gating.
4. **Lazy-loading without intersection-observer fallback.** Content that only loads on scroll won't render — the renderer doesn't scroll. Use `loading="lazy"` on images (renderer-aware) and ensure non-image lazy content has eager-load fallbacks.

## Anti-Patterns

- **Blocking essential resources in robots.txt.** Disallowing `/css/`, `/js/`, or asset paths that the rendering tier needs leaves Googlebot with a half-rendered DOM. Allow them.
- **Cloaking (serving different content to Googlebot than to users).** Detected via desktop/smartphone crawl with non-Googlebot User-Agent samples; punished hard. The exception is **Dynamic Rendering** (server-side render for bots, CSR for users), which Google has explicitly deprecated in favor of universal SSR.
- **Soft 404s.** Returning HTTP 200 with a "Not Found" page wastes crawl budget on pages that should have been 404s.
- **Infinite calendar pages, infinite pagination, infinite faceted-search URLs.** Crawler traps eat budget; use `nofollow`, robots.txt disallow, or canonical pointers.
- **Same content on tens of URL variants without canonical tags.** Crawl budget bleeds across duplicates.
- **Treating a sitemap as a guarantee of indexing.** Sitemaps are hints; quality, links, and content win.
- **Returning 5xx on legitimate requests.** Triggers Google's politeness back-off; persistent 5xx pauses crawl entirely.
- **`Crawl-delay: 30` "to be safe."** Googlebot ignores `Crawl-delay`. Use Search Console crawl-rate setting instead, and only when you actually have a capacity problem.
- **Geo-blocking Googlebot's IPs.** Googlebot crawls from US ranges primarily; geo-fencing the US locks Google out of your site.
- **Putting `noindex` in robots.txt.** Removed from support in 2019. Use the meta tag.
- **Cache-buster query strings on every static asset link.** Inflates the URL space; the rendering tier sees a different URL for every fetch.
- **Aggressive bot-protection that fails Googlebot's TLS handshake.** Cloudflare, Akamai, etc., have allow-listing for verified Googlebot; check WAF logs for false-positive blocks.
- **Ignoring `Last-Modified` / `ETag` headers.** Letting Googlebot do conditional GETs (304 responses) saves both sides bandwidth.

## Related

- [`../design-google-search.md`](../design-google-search.md) — parent doc; sketches the crawl tier, then indexing, ranking, and serving.
- [`../design-web-crawler.md`](../design-web-crawler.md) — general-purpose crawler design (Mercator/Heritrix/Common Crawl style); much overlap on frontier, dedup, and politeness.
- [`../design-news-aggregator.md`](../design-news-aggregator.md) — focused vertical crawler with much tighter freshness requirements and curated seeds.
- [`../../../building-blocks/rate-limiters.md`](../../../building-blocks/rate-limiters.md) — token-bucket and AIMD patterns underpinning per-host pacing.
- [`../../../building-blocks/object-and-blob-storage.md`](../../../building-blocks/object-and-blob-storage.md) — where the fetched bytes (WARC archives) actually live.
- [`../../../scalability/sharding-strategies.md`](../../../scalability/sharding-strategies.md) — frontier sharding by registered domain.

## References

- Google Search Central. ["Overview of Google crawlers and fetchers (user agents)."](https://developers.google.com/search/docs/crawling-indexing/overview-google-crawlers) Authoritative list of Googlebot variants and user-agent strings.
- Google Search Central. ["Crawling and indexing topics."](https://developers.google.com/search/docs/crawling-indexing) Top-level docs covering fetch, render, index pipeline.
- Google Search Central. ["How Google Interprets the robots.txt Specification."](https://developers.google.com/search/docs/crawling-indexing/robots/robots_txt) Practical interpretation of RFC 9309 edge cases.
- Google Search Central. ["Large site owner's guide to managing your crawl budget."](https://developers.google.com/search/docs/crawling/large-site-managing-crawl-budget) Crawl capacity vs crawl demand model.
- Google Search Central. ["Verifying Googlebot and other Google crawlers."](https://developers.google.com/search/docs/crawling-indexing/verifying-googlebot) Reverse-DNS and IP-range verification.
- Google Search Central. ["Sitemaps overview."](https://developers.google.com/search/docs/crawling-indexing/sitemaps/overview) Sitemap discovery, format, limits.
- Google Search Central. ["News sitemaps."](https://developers.google.com/search/docs/crawling-indexing/sitemaps/news-sitemap) Higher-priority sitemap variant.
- Google Search Central. ["URL canonicalization."](https://developers.google.com/search/docs/crawling-indexing/canonicalization) `rel=canonical`, dedup, signal consolidation.
- Google Search Central. ["Understand the JavaScript SEO basics."](https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics) Web Rendering Service expectations.
- Google Search Central blog. ["Mobile-first indexing by default for new sites."](https://developers.google.com/search/blog/2019/05/mobile-first-indexing-by-default-for) The 2019 default switch.
- Google Search Central blog. ["Googlebot will soon speak HTTP/2."](https://developers.google.com/search/blog/2020/09/googlebot-will-soon-speak-http2) HTTP/2 rollout announcement.
- Google Search Central blog. ["The new evergreen Googlebot."](https://developers.google.com/search/blog/2019/05/the-new-evergreen-googlebot) Renderer Chromium version unstuck.
- IETF. ["RFC 9309: Robots Exclusion Protocol."](https://www.rfc-editor.org/rfc/rfc9309.html) Formal robots.txt specification (Sept 2022).
- sitemaps.org. ["Sitemap Protocol."](https://www.sitemaps.org/protocol.html) The protocol spec that Google, Bing, and others honor.
- Heydon, A. and Najork, M. ["Mercator: A Scalable, Extensible Web Crawler."](https://www.cs.cornell.edu/courses/cs685/2002fa/mercator.pdf) *World Wide Web*, vol. 2, no. 4, 1999. Front-queue/back-queue frontier, the canonical reference.
- Manku, G. S., Jain, A., and Sarma, A. D. ["Detecting Near-Duplicates for Web Crawling."](https://research.google.com/pubs/archive/33026.pdf) WWW 2007. SimHash plus the bit-permutation lookup trick at Google scale.
- Boldi, P., Marino, A., Santini, M., and Vigna, S. ["BUbiNG: Massive Crawling for the Masses."](https://arxiv.org/pdf/1601.06919) 2016. Modern HTTP/2-aware decentralized crawler design.
- Common Crawl. ["Get Started."](https://commoncrawl.org/get-started) Petabyte-scale public crawl dataset; useful real-world reference.
- Common Crawl. ["Navigating the WARC file format."](https://commoncrawl.org/blog/navigating-the-warc-file-format) WARC layout used by the open-source crawler ecosystem and consumed by many indexing pipelines.
- Internet Archive. [Heritrix3 user manual](https://heritrix.readthedocs.io/en/latest/). Production open-source crawler with explicit per-host queuing, scope rules, and politeness governors.
