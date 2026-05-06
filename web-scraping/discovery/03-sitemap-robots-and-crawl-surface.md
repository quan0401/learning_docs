---
title: "Sitemap, robots.txt, and Crawl Surface Discovery"
date: 2026-05-06
updated: 2026-05-06
tags: [sitemap, robots-txt, well-known, wayback, common-crawl, crawl-surface]
---

# Sitemap, robots.txt, and Crawl Surface Discovery

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `sitemap` `robots-txt` `well-known` `wayback` `common-crawl` `crawl-surface`

---

## Table of Contents

1. [What "Crawl Surface" Means](#1-what-crawl-surface-means)
2. [robots.txt — The Original Crawl Hint](#2-robotstxt--the-original-crawl-hint)
3. [sitemap.xml and the sitemaps.org Protocol](#3-sitemapxml-and-the-sitemapsorg-protocol)
4. [Sitemap Indexes and Recursive Discovery](#4-sitemap-indexes-and-recursive-discovery)
5. [`.well-known/` — RFC 8615 and the Resources It Indexes](#5-well-known--rfc-8615-and-the-resources-it-indexes)
6. [security.txt, llms.txt, ads.txt, humans.txt](#6-securitytxt-llmstxt-adstxt-humanstxt)
7. [Wayback Machine and the CDX API](#7-wayback-machine-and-the-cdx-api)
8. [Common Crawl — Petabyte Web Index](#8-common-crawl--petabyte-web-index)
9. [Putting It Together — A Crawl-Surface Pipeline](#9-putting-it-together--a-crawl-surface-pipeline)

## Summary

Once you have hostnames (from doc 01), you need to enumerate **paths** — the URL surface area of each host. The site's own crawl-control files (`robots.txt`, `sitemap.xml`) often hand you the answer: sitemap.xml typically lists thousands of canonical URLs designed for search engines, and robots.txt names files and directories the operator considered worth excluding from crawl (which is the inverse: a list of paths they care about). RFC 8615's `.well-known/` URI tree adds a standardized location for security.txt, ads.txt, openid-configuration, and dozens of other discovery files. Where current crawl misses, the **Wayback Machine CDX API** and **Common Crawl** indexes provide historical URLs the site once exposed and may still serve. The combination — robots/sitemap + .well-known + Wayback + Common Crawl — typically produces an order of magnitude more known URLs than any active crawl, before you've sent a single request to the target's application servers.

---

## 1. What "Crawl Surface" Means

A web property's **crawl surface** is the set of URLs reachable from the site without authentication or interaction. For SEO, the operator wants this set fully indexed by search engines; for an attacker, it's the prerequisite for finding sensitive endpoints; for a scraper, it's the seed list for the crawl.

There are two complementary discovery modes:

1. **Operator-published** — the site explicitly tells you what's there: sitemap, robots, .well-known
2. **Operator-leaked** — the site let URLs leak into public archives, search engines, or third-party crawls: Wayback, Common Crawl, GitHub, Postman

Both are **passive** — you read public datasets, the target's servers see no traffic.

---

## 2. robots.txt — The Original Crawl Hint

The Robots Exclusion Protocol started as a 1994 informal note and was finally [standardized as RFC 9309 in 2022](https://datatracker.ietf.org/doc/html/rfc9309). The file lives at `/robots.txt`, root of every host, and contains directives like:

```text
User-agent: *
Disallow: /admin/
Disallow: /private/
Disallow: /tmp/

User-agent: Googlebot
Allow: /admin/public-help/

Sitemap: https://example.com/sitemap.xml
Sitemap: https://example.com/sitemap-news.xml
```

### 2.1 Parsing Semantics

RFC 9309 specifies:

- **Group matching**: a robot picks the most-specific `User-agent` group (longest match), or `*` if no specific match exists.
- **Rule matching within a group**: the longest match between `Allow` and `Disallow` wins. `Allow` overrides equal-length `Disallow`.
- **Path matching**: prefix match by default; `$` anchors end-of-URL; `*` matches any character sequence.
- **Multi-group declarations**: `User-agent: Googlebot\nUser-agent: Bingbot` applies the following rules to *both*.

The non-obvious part: **`Disallow: /` does not block crawlers that ignore robots.txt** (it's an honor system), but it *is* a signal that the operator considers the entire site sensitive.

### 2.2 What to Read From It

For recon: the `Disallow` paths are an interesting list precisely because the operator named them. `/admin/`, `/internal/`, `/api/v2/`, `/staging/` — these are paths the operator knows exist and didn't want indexed.

```bash
# Pull robots.txt and extract Disallow paths
curl -s https://example.com/robots.txt \
  | awk '/^[Dd]isallow:/ {print $2}' \
  | sort -u
```

For scraping: respect `Disallow` for any user-agent you claim to be. If your scraper sets `User-Agent: MyScraper/1.0`, you must honor `User-agent: MyScraper` rules (or the `*` fallback). Production hygiene around robots.txt is covered in doc 14.

### 2.3 Common Mistakes

- **Disallow precedence misunderstanding**: `Allow: /docs/` followed by `Disallow: /docs/secret/` correctly hides only `/docs/secret/`. But `Disallow: /docs/` followed by `Allow: /docs/public/` means most parsers correctly allow `/docs/public/` (longest match). Some old parsers don't.
- **Order of groups doesn't matter** in RFC 9309; only specificity does. Pre-RFC parsers sometimes used first-match.
- **`Crawl-delay`** is widely deployed but not part of RFC 9309; Google ignores it, Bing/Yandex/Yahoo honor it.

---

## 3. sitemap.xml and the sitemaps.org Protocol

The [sitemaps.org protocol](https://www.sitemaps.org/protocol.html) (originated by Google in 2005, then adopted by Bing and Yahoo) is a simple XML schema for advertising URLs to search engines.

### 3.1 The Format

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/page1</loc>
    <lastmod>2026-04-30</lastmod>
    <changefreq>weekly</changefreq>
    <priority>0.8</priority>
  </url>
  <url>
    <loc>https://example.com/page2</loc>
    <lastmod>2026-05-01</lastmod>
  </url>
</urlset>
```

### 3.2 Discovery

Three places sitemaps are advertised, in priority order:

1. The `Sitemap:` directive in `robots.txt` (most reliable)
2. The conventional path `/sitemap.xml`, `/sitemap_index.xml`, `/sitemap-index.xml`, `/wp-sitemap.xml` (WordPress), `/sitemap.xml.gz`, `/sitemap1.xml`, `/sitemaps/`
3. Submission to a search-engine console (not externally visible)

Common locations to probe:

```text
/sitemap.xml
/sitemap_index.xml
/sitemap-index.xml
/sitemap.xml.gz
/sitemap_main.xml
/sitemap1.xml
/sitemap/sitemap.xml
/wp-sitemap.xml
```

### 3.3 Limits

The protocol limits each sitemap to **50,000 URLs and 50 MB uncompressed**. Larger sites split into multiple sitemaps and use a **sitemap index** to point at them.

### 3.4 Variants

- **News sitemap** ([Google News spec](https://developers.google.com/search/docs/crawling-indexing/sitemaps/news-sitemap)) — adds publication metadata
- **Image sitemap** — adds `<image:image>` tags
- **Video sitemap** — adds `<video:video>` metadata
- **`hreflang` sitemap** — adds `<xhtml:link rel="alternate" hreflang="..."/>` per URL for multi-language sites

---

## 4. Sitemap Indexes and Recursive Discovery

A **sitemap index** lists other sitemap URLs:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>https://example.com/sitemap-products.xml</loc>
    <lastmod>2026-05-01</lastmod>
  </sitemap>
  <sitemap>
    <loc>https://example.com/sitemap-blog.xml.gz</loc>
    <lastmod>2026-04-15</lastmod>
  </sitemap>
</sitemapindex>
```

Recurse through `<sitemap><loc>` entries to fetch all child sitemaps; collect every `<urlset><url><loc>` from them.

**Pitfall**: sitemap loops. A misconfigured site can have `sitemap-a.xml` reference `sitemap-b.xml` reference `sitemap-a.xml`. A naive recursive crawler stack-overflows or runs forever. Track visited URLs in a set; cap recursion depth.

```python
# Minimal recursive sitemap fetcher
import requests
import xml.etree.ElementTree as ET
import gzip
import io

NS = {"s": "http://www.sitemaps.org/schemas/sitemap/0.9"}

def fetch(url, seen, max_depth=5, depth=0):
    if url in seen or depth >= max_depth:
        return []
    seen.add(url)
    body = requests.get(url, timeout=15).content
    if url.endswith(".gz"):
        body = gzip.decompress(body)
    root = ET.fromstring(body)
    urls = []
    if root.tag.endswith("sitemapindex"):
        for sm in root.findall("s:sitemap/s:loc", NS):
            urls.extend(fetch(sm.text, seen, max_depth, depth + 1))
    else:
        for u in root.findall("s:url/s:loc", NS):
            urls.append(u.text)
    return urls
```

---

## 5. `.well-known/` — RFC 8615 and the Resources It Indexes

[RFC 8615 (2019)](https://datatracker.ietf.org/doc/html/rfc8615) standardizes the `/.well-known/` URI prefix as the location for site-level metadata files. The [IANA Well-Known URIs registry](https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml) lists all registered names.

The high-value entries for recon and integration:

| Path | Purpose | Reference |
|------|---------|-----------|
| `/.well-known/security.txt` | Security disclosure contact | RFC 9116 |
| `/.well-known/openid-configuration` | OIDC discovery doc | OpenID Connect Core |
| `/.well-known/oauth-authorization-server` | OAuth 2.0 metadata | RFC 8414 |
| `/.well-known/jwks.json` | (Common, non-IANA) JSON Web Key Set | RFC 7517 |
| `/.well-known/host-meta` | XRD-format metadata | RFC 6415 |
| `/.well-known/webfinger` | User/service discovery | RFC 7033 |
| `/.well-known/acme-challenge/` | ACME (Let's Encrypt) HTTP-01 validation | RFC 8555 |
| `/.well-known/apple-app-site-association` | iOS universal links | Apple |
| `/.well-known/assetlinks.json` | Android App Links | Google |
| `/.well-known/change-password` | Password-change URL hint | W3C |
| `/.well-known/dnt-policy.txt` | Do-Not-Track policy | EFF |
| `/.well-known/matrix/server` | Matrix server discovery | Matrix.org |
| `/.well-known/nodeinfo` | ActivityPub server info | NodeInfo |

### 5.1 OIDC Discovery — A Goldmine

`/.well-known/openid-configuration` returns a JSON document describing every endpoint and capability of the OAuth/OIDC provider:

```json
{
  "issuer": "https://login.example.com",
  "authorization_endpoint": "https://login.example.com/oauth/authorize",
  "token_endpoint": "https://login.example.com/oauth/token",
  "jwks_uri": "https://login.example.com/.well-known/jwks.json",
  "userinfo_endpoint": "https://login.example.com/userinfo",
  "scopes_supported": ["openid", "email", "profile", "internal:admin"],
  "claims_supported": ["sub", "email", "email_verified", "internal_user_id"]
}
```

The `scopes_supported` and `claims_supported` arrays often disclose the application's auth model — useful for both security review and API reverse-engineering.

### 5.2 ACME Challenge Trail

`/.well-known/acme-challenge/<token>` files are short-lived (used during cert issuance) but `Wayback Machine` historically captured them — sometimes revealing the nginx/apache rewrite layer in front of Let's Encrypt.

---

## 6. security.txt, llms.txt, ads.txt, humans.txt

### 6.1 security.txt — RFC 9116

[RFC 9116 (2022)](https://datatracker.ietf.org/doc/html/rfc9116) standardizes `/.well-known/security.txt` as the location for security-disclosure contact info:

```text
Contact: mailto:security@example.com
Contact: https://example.com/security
Encryption: https://example.com/pgp-key.txt
Acknowledgments: https://example.com/security/hall-of-fame
Preferred-Languages: en, es
Canonical: https://example.com/.well-known/security.txt
Policy: https://example.com/security/policy
Hiring: https://example.com/jobs
Expires: 2026-12-31T23:59:59.000Z
```

If you find a vulnerability, this is the file that tells you who to email and under what disclosure terms.

### 6.2 llms.txt — Proposed (2024)

[llms.txt](https://llmstxt.org/) is a 2024 proposal (not RFC, not finalized) for advertising LLM-friendly content. The format mirrors `robots.txt` semantically but the file is markdown, intended to be readable by LLMs and AI crawlers:

```markdown
# Example Corp

> Example Corp builds payment infrastructure for international transfers.

## Docs
- [API reference](https://docs.example.com/api): full API surface
- [Quickstart](https://docs.example.com/quickstart): 5-minute integration
```

Adoption is uneven (a few hundred sites at time of writing), but it's worth fetching as a recon source — it's a curated map of what the operator considers their important content.

### 6.3 ads.txt and app-ads.txt

[ads.txt](https://iabtechlab.com/ads-txt/) ([IAB Tech Lab spec](https://iabtechlab.com/wp-content/uploads/2017/09/IABOpenRTBAdsTxtPublicSpecVersion-1-0-2.pdf)) declares which entities are authorized to sell the site's ad inventory. Useful for ad-tech competitive intel; reveals exchange and SSP relationships.

### 6.4 humans.txt

[humanstxt.org](http://humanstxt.org/) — informal convention for crediting site authors; sometimes leaks team names useful for social-engineering recon.

---

## 7. Wayback Machine and the CDX API

The [Internet Archive's Wayback Machine](https://web.archive.org) has captured the public web since 1996. Beyond the user-facing browser at `web.archive.org/web/*`, it exposes a [CDX Server API](https://github.com/internetarchive/wayback/tree/master/wayback-cdx-server) that returns every captured URL matching a query.

### 7.1 CDX Query Syntax

```bash
# All URLs ever captured under example.com (deduplicated)
curl -s "https://web.archive.org/cdx/search/cdx?url=example.com/*&output=json&collapse=urlkey" \
  | jq -r '.[1:] | .[] | .[2]'
```

CDX fields (in order): `urlkey, timestamp, original, mimetype, statuscode, digest, length`.

### 7.2 Common Pivots

- **Recover deleted endpoints** — internal admin URLs that existed for a year and were taken down still live in CDX.
- **Find old API versions** — `example.com/api/v1/*` was deprecated for `v2`, but `v1` may still respond.
- **Trace deployment history** — repeated captures of `index.html` show how a homepage evolved.
- **Find leaked secrets** — `.env`, `config.json`, `wp-config.php` accidentally committed to public webroot and immediately removed are still in CDX.

### 7.3 Tools

- [`waybackurls`](https://github.com/tomnomnom/waybackurls) (Tomnomnom) — CLI; one domain → all CDX URLs
- [`gau`](https://github.com/lc/gau) (`getallurls`) — combines Wayback, CommonCrawl, URLscan, Alien Vault OTX
- [`gauplus`](https://github.com/bp0lr/gauplus) — performance-tuned `gau`

---

## 8. Common Crawl — Petabyte Web Index

[Common Crawl](https://commoncrawl.org/) publishes monthly snapshots of a sizable fraction of the public web. Each snapshot includes:

- **WARC files** — raw HTTP request/response pairs
- **WAT files** — extracted HTTP metadata
- **WET files** — extracted plain text
- **URL index** — sortable, queryable

### 8.1 The URL Index API

The URL index supports prefix queries:

```bash
# Latest crawl ID
curl -s 'https://index.commoncrawl.org/collinfo.json' | jq -r '.[0].id'

# Query: every URL captured under example.com in latest crawl
CRAWL=CC-MAIN-2026-...
curl -s "https://index.commoncrawl.org/$CRAWL-index?url=example.com&output=json"
```

Each result includes the WARC offset, so you can fetch just the relevant bytes from S3 (`s3://commoncrawl/`) rather than downloading TB of WARC files.

### 8.2 When to Reach for It

- Wayback's coverage is biased toward popular sites; CommonCrawl is broader
- WARC includes full request/response, including headers; Wayback strips most headers from the public UI
- Common Crawl is the dataset behind much LLM training, so it's also a useful proxy for "what did big crawls see"

### 8.3 Tools

- [`cdx-toolkit`](https://github.com/cocrawler/cdx_toolkit) — Python library for both Wayback and Common Crawl indexes
- [`gau`](https://github.com/lc/gau) — already mentioned, includes Common Crawl

---

## 9. Putting It Together — A Crawl-Surface Pipeline

```bash
TARGET=example.com

# 1. robots.txt + Disallow paths
ROBOTS=$(curl -s https://"$TARGET"/robots.txt)
echo "$ROBOTS" | awk '/^[Dd]isallow:/ {print $2}' > robots_disallow.txt
echo "$ROBOTS" | awk '/^[Ss]itemap:/ {print $2}' > sitemap_urls.txt

# 2. Conventional sitemap probes
for path in sitemap.xml sitemap_index.xml sitemap-index.xml wp-sitemap.xml; do
  curl -sI -o /dev/null -w "%{http_code} %{url_effective}\n" \
    "https://$TARGET/$path"
done

# 3. .well-known probes
for w in security.txt openid-configuration oauth-authorization-server jwks.json humans.txt; do
  curl -sI -o /dev/null -w "%{http_code} https://$TARGET/.well-known/$w\n" \
    "https://$TARGET/.well-known/$w"
done

# 4. Wayback CDX dump
curl -s "https://web.archive.org/cdx/search/cdx?url=$TARGET/*&output=json&collapse=urlkey&fl=original" \
  | jq -r '.[1:] | .[] | .[0]' > wayback_urls.txt

# 5. Common Crawl URL index
gau "$TARGET" > gau_urls.txt

# 6. Merge, dedup
cat wayback_urls.txt gau_urls.txt | sort -u > all_urls.txt
wc -l all_urls.txt
```

After this, you have a corpus of historical and current URLs to triage — alive vs dead, interesting paths vs noise — without ever sending a request to the application's main servers (other than the four meta-files in steps 1–3).

---

## Related

- [Subdomain & Asset Enumeration](01-subdomain-and-asset-enumeration.md) — what to do *before* this; finds the hosts whose crawl surface you then map
- [Internet Asset Scanners](02-internet-asset-scanners.md) — pivots on certificates and banners; complements path enumeration
- [HTTP Scraping Fundamentals](../extraction/06-http-scraping-fundamentals.md) — actually fetching the URLs you discovered
- [API Reverse-Engineering](../reverse-engineering/13-api-reverse-engineering.md) — when sitemap reveals API endpoints worth analyzing
- [Production Scraping Hygiene and Legal Landscape](../reverse-engineering/14-production-scraping-hygiene-and-legal.md) — robots.txt compliance for legitimate crawlers

## References

- [RFC 9309 — Robots Exclusion Protocol](https://datatracker.ietf.org/doc/html/rfc9309)
- [sitemaps.org Protocol](https://www.sitemaps.org/protocol.html)
- [RFC 8615 — Well-Known URIs](https://datatracker.ietf.org/doc/html/rfc8615)
- [RFC 9116 — A File Format to Aid in Security Vulnerability Disclosure (security.txt)](https://datatracker.ietf.org/doc/html/rfc9116)
- [IANA Well-Known URIs Registry](https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml)
- [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html)
- [RFC 8414 — OAuth 2.0 Authorization Server Metadata](https://datatracker.ietf.org/doc/html/rfc8414)
- [Wayback CDX Server API](https://github.com/internetarchive/wayback/tree/master/wayback-cdx-server)
- [Common Crawl Documentation](https://commoncrawl.org/the-data/get-started/)
- [llms.txt proposal](https://llmstxt.org/)
- [IAB Tech Lab — ads.txt v1.0.2](https://iabtechlab.com/wp-content/uploads/2017/09/IABOpenRTBAdsTxtPublicSpecVersion-1-0-2.pdf)
