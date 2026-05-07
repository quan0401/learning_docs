# Web Scraping & Recon Documentation Index — Learning Path

A progressive path through external attack-surface discovery, bulk data extraction, and browser-side reverse-engineering. Backend-oriented and tool-grounded — covers the recon you do before scraping, the mechanics of scraping at scale, and the reverse-engineering tricks for sites where the data is locked behind dynamic JavaScript or undocumented APIs.

The use case is **balanced**: legitimate data scraping for pipelines, competitive/market intelligence, and authorized bug-bounty/pentest recon. Each doc surfaces ethics and legal context where it matters (residential proxy sourcing, CAPTCHA solver ToS violations, CFAA/GDPR boundaries) rather than picking one camp.

Cross-references to the [Networking learning path](../networking/INDEX.md) (DNS, TLS, HTTP/2 — the protocols you scrape over), the [Security learning path](../security/INDEX.md) (OSINT and recon are the offensive mirror of OWASP/STRIDE; OAuth/JWT topics overlap with API reverse-engineering), the [TypeScript learning path](../typescript/INDEX.md) (Node-based scraping with Crawlee/Playwright, source-map reading), the [Java learning path](../java/INDEX.md), the [Go learning path](../golang/INDEX.md) (Colly, JA3 fingerprinting in `crypto/tls`), the [Database learning path](../database/INDEX.md) (storing scraped corpora, dedup), the [Kubernetes learning path](../kubernetes/INDEX.md) (containerized scraper fleets, egress IP control), the [Low-Level Design learning path](../low-level-design/INDEX.md), the [System Design learning path](../system-design/INDEX.md) (queues, dedup, politeness layers as system-design problems), the [Operating Systems learning path](../operating-systems/INDEX.md) (event loops behind concurrent crawlers), the [Observability learning path](../observability/INDEX.md) (block-rate, ASN-level success metrics), the [Performance Engineering learning path](../performance/INDEX.md) (concurrency vs throughput in crawlers), the [Search learning path](../search/INDEX.md) (where scraped corpora often land), the [Linux learning path](../linux/INDEX.md) (scraper-host operations, egress IP control), and the [Git Internals learning path](../git-internals/INDEX.md) where topics overlap.

**Markers:** **★** = core must-learn (everyday scraping/recon work — what you reach for first). **○** = supporting deep-dive (specialized tooling or vendor-specific landscape). Internalize all ★ before going deep on ○.

---

## Tier 1 — Discovery: Finding What's Out There

Before you scrape, you find. This tier covers external reconnaissance: enumerating an organization's hosts and assets, querying internet-wide scanners, mapping crawl surface from `sitemap.xml`/`robots.txt`/`.well-known`, and fingerprinting the technologies in play. The skills here are equally useful for bug-bounty recon, market intel, and finding scrape-worthy endpoints in the first place.

1. [★ Subdomain & Asset Enumeration — Passive and Active](discovery/01-subdomain-and-asset-enumeration.md) — crt.sh and CT logs, DNS aggregators (SecurityTrails, VirusTotal, AlienVault OTX), subfinder/amass/assetfinder/findomain/puredns/dnsx/massdns, wildcard DNS handling, brute-force resolvers, GitHub/code-search pivots, recursive enumeration _(2026-05-06)_
2. [★ Internet Asset Scanners — Censys, Shodan, and Friends](discovery/02-internet-asset-scanners.md) — Censys, Shodan, BinaryEdge, FOFA, ZoomEye, Hunter.how, Netlas search syntax, certificate pivots, favicon hashing (mmh3 + FavFreak), banner fingerprinting, free-tier vs paid quotas _(2026-05-06)_
3. [★ Sitemap, robots.txt, and Crawl Surface Discovery](discovery/03-sitemap-robots-and-crawl-surface.md) — sitemaps.org protocol, sitemap indexes, robots.txt parsing semantics, llms.txt, security.txt (RFC 9116), `.well-known/`, Wayback Machine CDX API, Common Crawl indexes _(2026-05-06)_
4. [○ HTML and Tech-Stack Fingerprinting — PublicWWW, BuiltWith, SimilarWeb](discovery/04-html-and-techstack-fingerprinting.md) — PublicWWW (HTML/JS source search), BuiltWith, Wappalyzer (DOM + headers + script signatures), SimilarWeb traffic estimation, SimilarSites/Sitelike for related-site discovery, accuracy limits _(2026-05-06)_
5. [○ IP, ASN, and Network Reconnaissance](discovery/05-ip-asn-and-network-recon.md) — IP→ASN→organization, bgp.he.net, RIPE/ARIN/APNIC, reverse DNS, ipinfo.io, MaxMind, IP reputation feeds (Spamhaus, IPQS, AbuseIPDB), CIDR enumeration, IPv6 considerations _(2026-05-06)_

---

## Tier 2 — Extraction: Getting the Data

Once you know where to look, this tier covers the mechanics of pulling data at scale: HTTP clients vs headless browsers, proxy and IP rotation, and the modern bot-detection stack you have to navigate (TLS/HTTP fingerprints, browser fingerprints, CAPTCHAs). Detection-mechanics first — once you understand how Cloudflare and Akamai actually score requests, both legitimate compliance and unavoidable evasion become engineering decisions rather than guesswork.

6. [★ HTTP Scraping Fundamentals](extraction/06-http-scraping-fundamentals.md) — `requests`/`httpx`/`curl`, sessions and cookies, redirect chains, header hygiene, gzip/brotli, HTTP/2 differences, Scrapy architecture, Colly (Go), Crawlee (Node) _(2026-05-06)_
7. [★ Headless Browsers — Playwright, Puppeteer, Selenium](extraction/07-headless-browsers.md) — when JS-rendered DOM is required vs raw HTTP, undetected-chromedriver, stealth plugins, request interception, throttling, cost trade-offs _(2026-05-06)_
8. [★ Proxies and IP Rotation](extraction/08-proxies-and-ip-rotation.md) — datacenter vs residential vs mobile vs ISP proxies, sticky vs rotating sessions, geographic targeting, sourcing-ethics on residential pools, vendor landscape, retry/backoff with pool health checks _(2026-05-06)_
9. [★ Bot Detection Internals](extraction/09-bot-detection-internals.md) — Cloudflare Bot Management, Akamai Bot Manager, DataDome, PerimeterX/HUMAN, kasada; JA3/JA4 TLS fingerprinting, HTTP/2 fingerprint, browser fingerprinting layers (canvas, WebGL, fonts, audio, WebRTC), behavioral signals _(2026-05-06)_
10. [○ CAPTCHA Systems and Solvers](extraction/10-captcha-systems-and-solvers.md) — reCAPTCHA v2/v3, hCaptcha, Cloudflare Turnstile, FunCaptcha (Arkose Labs), Geetest; how scoring works, solver services (2Captcha, CapSolver, Anti-Captcha), ML solvers, ToS implications _(2026-05-06)_

---

## Tier 3 — Reverse-Engineering: When the Data Is Hidden

When data sits behind dynamic JavaScript, undocumented APIs, or minified bundles, the scrape becomes a reverse-engineering problem. This tier covers the browser-side tools (Chrome DevTools breakpoints, source maps, chunked-bundle archaeology) and the network-side tools (mitmproxy, Burp, GraphQL introspection, signed-request analysis) that turn a black-box site into a clean API call. Closes with production hygiene and the legal landscape that keeps this work sustainable.

11. [★ Chrome DevTools for Scraping — Breakpoints and Network Forensics](reverse-engineering/11-chrome-devtools-for-scraping.md) — Network panel filtering, copy-as-curl/HAR, conditional/XHR/Fetch breakpoints, DOM breakpoints (subtree/attribute/removal), event-listener breakpoints, blackboxing, Workspaces, Performance and Memory panels for handler-chain tracing _(2026-05-06)_
12. [★ Source Maps and Chunked Bundles — Practical Use](reverse-engineering/12-source-maps-and-chunked-bundles.md) — Source Map V3 spec essentials, sourceMappingURL header and comment, common leak patterns, webpack chunk model (runtime/vendor/route/dynamic-import), source-map-explorer, unwebpack-sourcemap, fetching minified+map → readable TypeScript/JSX _(2026-05-06)_
13. [★ API Reverse-Engineering](reverse-engineering/13-api-reverse-engineering.md) — replaying flows from DevTools/HAR, mitmproxy and Burp Suite Community for active interception, GraphQL introspection (clairvoyance/graphql-cop), gRPC-Web inspection, signed requests (HMAC, AWS Sigv4), bearer-token reuse, websocket capture _(2026-05-06)_
14. [○ Production Scraping Hygiene and Legal Landscape](reverse-engineering/14-production-scraping-hygiene-and-legal.md) — politeness (robots.txt respect, Crawl-delay, sitemap-driven scheduling), client-side rate limiting and concurrency caps, URL canonicalization, content-hash dedup, monitoring (block-rate by ASN), legal landscape (CFAA, *hiQ Labs v. LinkedIn*, *Van Buren v. United States*, EU CJEU TDM rulings, GDPR/CCPA on scraped PII) _(2026-05-06)_

---

## Quick Reference by Topic

### Discovery

- [Subdomain & Asset Enumeration](discovery/01-subdomain-and-asset-enumeration.md)
- [Internet Asset Scanners](discovery/02-internet-asset-scanners.md)
- [Sitemap, robots, Crawl Surface](discovery/03-sitemap-robots-and-crawl-surface.md)
- [HTML & Tech-Stack Fingerprinting](discovery/04-html-and-techstack-fingerprinting.md)
- [IP, ASN & Network Recon](discovery/05-ip-asn-and-network-recon.md)

### Extraction

- [HTTP Scraping Fundamentals](extraction/06-http-scraping-fundamentals.md)
- [Headless Browsers](extraction/07-headless-browsers.md)
- [Proxies & IP Rotation](extraction/08-proxies-and-ip-rotation.md)
- [Bot Detection Internals](extraction/09-bot-detection-internals.md)
- [CAPTCHA Systems & Solvers](extraction/10-captcha-systems-and-solvers.md)

### Reverse-Engineering

- [Chrome DevTools for Scraping](reverse-engineering/11-chrome-devtools-for-scraping.md)
- [Source Maps & Chunked Bundles](reverse-engineering/12-source-maps-and-chunked-bundles.md)
- [API Reverse-Engineering](reverse-engineering/13-api-reverse-engineering.md)
- [Production Hygiene & Legal](reverse-engineering/14-production-scraping-hygiene-and-legal.md)

---

## Bug Spotting

Active-recall practice docs. Each presents 22+ broken snippets organized by difficulty (Easy / Subtle / Senior trap), with one-line `<details>` hints inline and full root-cause + fix in a Solutions section. Every bug cites a real reference (RFC, CVE, library issue, postmortem). Use these to pressure-test concept knowledge after working through the tiers above.

- [Scraping Pitfalls — Bug Spotting](extraction/scraping-pitfalls-bug-spotting.md) ★ — async crawler races, retry storms, robots.txt misparsing, sitemap-index recursion, IP rotation deadlocks, CAPTCHA token reuse, JA3 mismatches from default TLS, redirect/cookie-stripping, HTTP/2 header casing, source-map URL leaks that expose the scraper itself _(2026-05-06)_

---

## Related Learning Paths

- [Networking — DNS, TLS, HTTP/2](../networking/INDEX.md) — the protocols you scrape over and fingerprint on
- [Security — OWASP, threat modeling, OAuth/JWT](../security/INDEX.md) — recon is the offensive mirror; bearer-token analysis in API RE
- [System Design — queues, caches, dedup](../system-design/INDEX.md) — production scraper architecture as a system-design problem
- [Operating Systems — epoll, fds](../operating-systems/INDEX.md) — what makes concurrent crawlers fast (or slow)
- [Observability — RED/USE, structured logs](../observability/INDEX.md) — how to instrument a scraper that meets a freshness SLO
- [Performance — Little's Law, tail latency](../performance/INDEX.md) — concurrency vs throughput in crawler design
