---
title: "HTML and Tech-Stack Fingerprinting — PublicWWW, BuiltWith, Wappalyzer, SimilarWeb"
date: 2026-05-06
updated: 2026-05-06
tags: [fingerprinting, wappalyzer, builtwith, publicwww, similarweb, tech-detection]
---

# HTML and Tech-Stack Fingerprinting — PublicWWW, BuiltWith, Wappalyzer, SimilarWeb

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `fingerprinting` `wappalyzer` `builtwith` `publicwww` `similarweb` `tech-detection`

---

## Table of Contents

1. [What Tech-Stack Detection Actually Looks At](#1-what-tech-stack-detection-actually-looks-at)
2. [Wappalyzer — The Open-Source Detector](#2-wappalyzer--the-open-source-detector)
3. [BuiltWith — The Commercial Census](#3-builtwith--the-commercial-census)
4. [PublicWWW — Search by HTML/JS Source](#4-publicwww--search-by-htmljs-source)
5. [SimilarWeb — Traffic and Engagement Estimation](#5-similarweb--traffic-and-engagement-estimation)
6. [SimilarSites and Other Related-Site Discovery](#6-similarsites-and-other-related-site-discovery)
7. [Accuracy Limits and Confidence Calibration](#7-accuracy-limits-and-confidence-calibration)
8. [Putting It Together — Recon Use Cases](#8-putting-it-together--recon-use-cases)

## Summary

Once a host is alive, fingerprinting tells you what software runs on it: the framework, CMS, analytics suite, ad tech, CDN, payment processor, A/B test tool, fonts, language, and version. Five tools cover most of the field. **Wappalyzer** is the open-source community detector — a regex-driven engine you can run locally on any HTML/headers and self-host. **BuiltWith** is the commercial census, with deeper history and bulk-export pricing. **PublicWWW** is the inverse query: search 600M+ HTML/JS source bodies for a string and get the hosts using it — perfect for finding "every site embedding tracker X" or "every site running plugin Y v1.2.3." **SimilarWeb** estimates traffic and engagement rather than tech, but pairs naturally with the others for competitive intel. **SimilarSites/Sitelike/SimilarSiteSearch** are related-site recommenders. The non-obvious failure mode of all of them: false positives from build-time inclusions of libraries the site never actually uses, plus version detection that often lags the real version by months.

---

## 1. What Tech-Stack Detection Actually Looks At

A modern detector inspects multiple HTTP layers and aggregates signals:

| Signal | Examples |
|--------|----------|
| HTTP response headers | `Server: Apache/2.4`, `X-Powered-By: PHP/8.1`, `Set-Cookie: PHPSESSID=`, `X-Drupal-Cache:` |
| Meta tags | `<meta name="generator" content="WordPress 6.5">`, `<meta name="application-name" content="Shopify">` |
| HTML body fragments | `<script src="/wp-content/themes/...">`, `<div id="__NEXT_DATA__">`, `<noscript>You need to enable...` |
| Script URLs | `googletagmanager.com/gtag/js`, `cdn.shopify.com/s/files/`, `/wp-includes/js/jquery.js` |
| URL path patterns | `/wp-admin/`, `/_next/static/`, `/static/js/main.[hash].js` (CRA) |
| Cookie names | `_shopify_y`, `__cfduid`, `JSESSIONID`, `XSRF-TOKEN` |
| Favicon hash | (See doc 02 — favicon → product mapping) |
| HTTP/2 settings | (See doc 09 — server fingerprint) |
| TLS handshake | (See doc 02/09 — JARM/JA3S) |
| robots.txt patterns | `Disallow: /wp-admin/`, `User-agent: Mediapartners-Google` |

A detector aggregates dozens of these into a single classification with a confidence score. Modern Wappalyzer plus its community rules has signatures for ~3000 technologies.

---

## 2. Wappalyzer — The Open-Source Detector

[Wappalyzer](https://www.wappalyzer.com/) started as an open-source browser extension; the rules engine is open ([`enthec/webappanalyzer`](https://github.com/enthec/webappanalyzer) is one community-maintained fork after the original [AliasIO/wappalyzer](https://github.com/AliasIO/wappalyzer) repo became less active). The detection logic is data-driven — JSON files defining regex patterns per technology.

### 2.1 Rule Schema

A typical rule:

```json
{
  "Shopify": {
    "cats": [6],
    "cookies": {
      "_shopify_y": "",
      "_shopify_s": ""
    },
    "headers": {
      "X-ShopId": "",
      "X-Shopify-Stage": ""
    },
    "html": [
      "<script[^>]+src=\"(?:[^\"]+)?cdn\\.shopify\\.com",
      "<link[^>]+href=\"(?:[^\"]+)?cdn\\.shopify\\.com"
    ],
    "implies": "Liquid",
    "icon": "Shopify.svg",
    "website": "https://www.shopify.com"
  }
}
```

The `implies` field cascades: detecting Shopify implies Liquid; detecting Next.js implies React.

### 2.2 How to Run It

- **Browser extension** — Chrome, Firefox; on-demand inspection of the current tab
- **Self-hosted CLI** — `npm install -g wappalyzer-cli` runs the rules engine over arbitrary URLs
- **`httpx -tech-detect`** — ProjectDiscovery's `httpx` includes Wappalyzer rules; bulk-process URL lists
- **`webanalyze`** — [Go reimplementation](https://github.com/rverton/webanalyze) of the rules engine

```bash
# Bulk fingerprint a hostlist
cat hosts.txt | httpx -silent -tech-detect -json \
  | jq -r '[.url, (.tech // [] | join(","))] | @tsv'
```

### 2.3 Limits

Wappalyzer accuracy depends on rule freshness. The community fork lags behind the SaaS product Wappalyzer became after [acquisition by Wappalyzer Group / Foundation in 2023](https://www.wappalyzer.com/news/). For commercial-grade coverage, the SaaS API has more rules; for self-hosted, accept that newer technologies may be missed.

---

## 3. BuiltWith — The Commercial Census

[BuiltWith](https://builtwith.com/) is the closest thing to a commercial census of the web's tech stack. They run their own continuous crawl, attribute technologies per host, and expose:

- **Site lookup** — what's on this domain, with monthly history
- **Technology lookup** — every site using technology X (e.g., every site using HubSpot)
- **Lead lists** — bulk export with filters (industry, traffic tier, technology presence)
- **Trends** — adoption growth/decline over time

The free UI shows current stack and basic trends. Bulk access is paid (per-list pricing — historically in the thousands of USD for sizable lists).

### 3.1 What It's Best At

- **Competitive intelligence** — "show me every Shopify Plus merchant in apparel with 100k+ sessions/month"
- **Sales prospecting** — "give me every US-based site running Salesforce Marketing Cloud and not running Marketo"
- **Vendor switching analysis** — when sites left X for Y

### 3.2 BuiltWith vs Wappalyzer

| Dimension | Wappalyzer | BuiltWith |
|-----------|------------|-----------|
| Coverage breadth | Open-source, ~3000 techs | Commercial, broader |
| Historical | Limited | Months/years per site |
| Bulk export | DIY + API tier | Paid product |
| Free-tier per-site | Browser extension free | Free site lookup, no export |
| Self-hostable | Yes (rules engine) | No |

### 3.3 Other Census-Style Services

- [Datanyze](https://www.datanyze.com/) — sales-intelligence flavored, similar approach
- [HG Insights](https://hginsights.com/) — enterprise sales intelligence; weaves install-base data with company firmographics
- [SimilarTech](https://www.similartech.com/) — owned by SimilarWeb; integrates traffic-tier filters with tech detection

---

## 4. PublicWWW — Search by HTML/JS Source

[PublicWWW](https://publicwww.com/) is structurally different: it indexes the HTML/JS source bodies of ~700 million sites and lets you full-text search across them. You query a snippet of HTML or JavaScript, you get back the list of hosts whose source contains it.

### 4.1 What This Unlocks

- **Find every site embedding a tracker** — `"src=\"https://cdn.tracker.example/v1.js\""`
- **Find every site using a vulnerable library version** — `"jquery-1.7.2.min.js"`
- **Find unauthorized re-uses of your code** — search for a unique class name or string from your bundle
- **Identify white-label/SaaS deployments** — search for a SaaS template's signature string
- **Find sites referencing a competitor or partner** — search for their tracking pixel or script
- **Recover indirect attribution** — the API key embedded in a JS bundle exposes every site using that tenant

### 4.2 Query Syntax

PublicWWW uses simple substring search with quoted phrases. Operators include `country:US`, `langtitle:en` (HTML language), and basic boolean logic.

```text
"googletagmanager.com/gtm.js?id=GTM-XYZ123"
"intercom-snippet" "intercomSettings"
"X-Powered-By: Express" "powered by node"
```

### 4.3 Pricing

PublicWWW has a generous free tier (limited result count per query, ~100 results) and paid tiers for full result sets and API access. The free tier is enough for spot queries; the paid tier is reasonable for periodic intel pulls.

### 4.4 Alternatives

- [Nerdydata](https://www.nerdydata.com/) — similar concept, different index
- [SearchCode](https://searchcode.com/) — narrower (mostly source code), but relevant for finding leaked secrets
- [GrepApp](https://grep.app/) — grep over public GitHub repos, useful adjacent

---

## 5. SimilarWeb — Traffic and Engagement Estimation

[SimilarWeb](https://www.similarweb.com/) estimates per-site traffic, geographic distribution, traffic sources, engagement metrics, and audience overlap. It's not a tech detector; it's the closest thing to an "Alexa replacement" for traffic ranking after Alexa shut down in 2022.

### 5.1 Methodology (in brief)

SimilarWeb does not have direct measurement of every site (no tag installed). Instead they aggregate:

- A panel of users who installed SimilarWeb's data-collecting browser extensions
- ISP-level clickstream data partnerships
- Direct measurement from sites that installed their analytics
- Public data (search-volume signals, social signals)

The model extrapolates from the panel to the global population. Accuracy is reasonable for sites with millions of monthly visits and decreases sharply below that threshold.

### 5.2 What It Tells You

- **Estimated total visits per month**
- **Country-level traffic breakdown**
- **Traffic source breakdown** — direct, search, social, referrals, display, mail
- **Top referring sites**
- **Top destinations from this site** (where the audience goes next)
- **Audience overlap with competitors**
- **Top organic and paid keywords** (limited free tier)

### 5.3 Free vs Paid

The free product gives a single-site overview with summarized metrics. Paid tiers ("SimilarWeb Pro") unlock per-day data, longer history, comparison charts, and API access.

### 5.4 Why You Care for Recon

For market intel: traffic estimation tells you which competitors actually matter. For scrape prioritization: high-traffic sites need the most-defended scraping setup. For asset discovery: top-referring-sites lists frequently surface partner integrations and white-label deployments.

---

## 6. SimilarSites and Other Related-Site Discovery

For related-site discovery (find sites *similar to* a known one), several tools exist:

| Tool | Approach |
|------|----------|
| [SimilarSites](https://www.similarsites.com/) (consumer) | Algorithmic similarity over content + audience overlap |
| [Sitelike.org](https://www.sitelike.org/) | Free index, simple similarity |
| [SimilarWeb related sites tab](https://www.similarweb.com/) | Audience-overlap based; usually highest-quality |
| [SpyFu related domains](https://www.spyfu.com/) | SEO-keyword-overlap based |
| [Alexa-Successors and proprietary scoring](https://www.semrush.com/) | SEMrush, Ahrefs offer competitor-graph discovery |

The signal these tools express varies — content similarity, audience overlap, keyword overlap, link graph proximity. For competitive intelligence, audience overlap (SimilarWeb) is usually most actionable. For SEO and content competition, keyword overlap (SEMrush, Ahrefs) is more useful.

---

## 7. Accuracy Limits and Confidence Calibration

Every detector has systematic error modes:

### 7.1 False Positives From Build Inclusions

Modern bundlers ship code that *might* be used at runtime. A site can include all of Lodash even if it only calls one function. A detector matching `lodash.min.js` claims Lodash; but the *actual* runtime behavior is one function. For security purposes (vuln scoping), this is a false positive: you can't claim the site is exploitable through Lodash CVE X without confirming reachability.

### 7.2 Stale Version Detection

Detectors infer versions from filenames (`jquery-3.6.0.min.js`), bundle comments, or deprecated meta-generator tags. Many sites strip filenames during bundling, which collapses the version signal entirely. The detector then either reports "jQuery (version unknown)" or guesses.

### 7.3 CDN-Hosted Libraries Hide Origin

A site loading `https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/react.production.min.js` reveals React 18.2.0. If they self-host as `/static/js/main.[hash].js`, you see only "React" with no version, or sometimes nothing.

### 7.4 Anti-Fingerprinting

Defensive sites strip headers (`Server`, `X-Powered-By`), rename cookies (`SESSION` instead of `JSESSIONID`), hash all asset paths, and use generic CSS classes. Detectors return mostly-empty results.

### 7.5 White-Label and Multi-Tenant Confusion

A SaaS platform's many customer sites all look like the SaaS platform. The detector reports Shopify; the *actual entity* is the merchant. Cross-reference with the domain owner and the SaaS template fingerprint to disambiguate.

---

## 8. Putting It Together — Recon Use Cases

### 8.1 Bug-Bounty Recon

After enumerating subdomains (doc 01), fingerprint each:

```bash
cat hosts.txt | httpx -silent -tech-detect -json \
  | jq -r 'select(.tech | contains(["WordPress"])) | .url' \
  > wordpress_hosts.txt
# Now run wpscan against each
```

### 8.2 Competitive Intel

For each direct competitor:

1. BuiltWith for current stack and 12-month change history
2. SimilarWeb for traffic and audience overlap
3. PublicWWW for "every site embedding their analytics tracker" — finds their partners
4. SimilarSites or SimilarWeb related-sites tab — finds adjacent competitors you missed

### 8.3 Pre-Scrape Sizing

Before building a scraper:

- BuiltWith → runs Cloudflare? Akamai? Tells you the anti-bot tier (doc 09)
- SimilarWeb → traffic tier; do they have the engineering investment to defend?
- Wappalyzer → CMS or custom? CMS often means predictable URLs and structure
- robots.txt + sitemap (doc 03) → are they signaling crawl-friendliness?

### 8.4 Find Sibling Properties

If `example.com` runs Google Analytics ID `G-ABCDEF`, search PublicWWW for that tracking ID — every other site sharing the GA property is owned by the same operator. The same trick works for `Adsense client IDs`, `GTM container IDs`, and proprietary SaaS tenant identifiers in script URLs.

```text
publicwww query: "G-ABCDEF"
```

This is one of the most reliable recon techniques for finding undisclosed sibling properties of an organization.

---

## Related

- [Subdomain & Asset Enumeration](01-subdomain-and-asset-enumeration.md) — must come first; fingerprinting needs hosts
- [Internet Asset Scanners](02-internet-asset-scanners.md) — favicon hashing and banner search overlap; complements rules-based detection
- [Sitemap, robots.txt, and Crawl Surface](03-sitemap-robots-and-crawl-surface.md) — robots/sitemap signals feed into Wappalyzer rules
- [Bot Detection Internals](../extraction/09-bot-detection-internals.md) — fingerprinting goes the other way: how the *site* fingerprints *you*
- [HTTP Scraping Fundamentals](../extraction/06-http-scraping-fundamentals.md) — once stack is known, picks scraping approach

## References

- [Wappalyzer technologies dataset (community fork)](https://github.com/enthec/webappanalyzer)
- [`webanalyze` Go implementation](https://github.com/rverton/webanalyze)
- [BuiltWith methodology](https://builtwith.com/api)
- [PublicWWW search documentation](https://publicwww.com/)
- [SimilarWeb methodology overview](https://www.similarweb.com/corp/research/methodology/)
- [HTTPArchive — public crawl of the top web's tech](https://httparchive.org/) — adjacent dataset, free, BigQuery-queryable
- [Web Almanac — annual analysis built on HTTPArchive](https://almanac.httparchive.org/)
