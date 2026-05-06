---
title: "Subdomain & Asset Enumeration — Passive and Active"
date: 2026-05-06
updated: 2026-05-06
tags: [recon, osint, dns, subdomains, certificate-transparency, bug-bounty]
---

# Subdomain & Asset Enumeration — Passive and Active

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `recon` `osint` `dns` `subdomains` `certificate-transparency` `bug-bounty`

---

## Table of Contents

1. [Why Enumerate Subdomains](#1-why-enumerate-subdomains)
2. [Passive Enumeration — Sources That Already Know](#2-passive-enumeration--sources-that-already-know)
3. [Active Enumeration — DNS Brute Force and Resolution](#3-active-enumeration--dns-brute-force-and-resolution)
4. [Wildcard DNS — The Footgun That Inflates Every Result](#4-wildcard-dns--the-footgun-that-inflates-every-result)
5. [The ProjectDiscovery and OWASP Amass Toolchain](#5-the-projectdiscovery-and-owasp-amass-toolchain)
6. [GitHub, Code Search, and Source-Leak Pivots](#6-github-code-search-and-source-leak-pivots)
7. [Pipelining: From a Domain to a Triage-Ready Asset List](#7-pipelining-from-a-domain-to-a-triage-ready-asset-list)
8. [Ethics and Authorization](#8-ethics-and-authorization)

## Summary

Subdomain enumeration is the first step of external recon and the prerequisite for almost all downstream work — vulnerability scanning, scrape-target discovery, attack-surface mapping. There are two complementary techniques: **passive** (query third-party datasets that already know subdomains exist — Certificate Transparency logs, DNS aggregators, search engines) and **active** (brute-force candidate names against DNS resolvers). Passive is fast, quiet, and finds anything that ever issued a public TLS certificate; active fills the gaps, especially for internal-only or never-publicly-served names. The modern toolchain — `subfinder`, `amass`, `assetfinder`, `puredns`, `dnsx`, `massdns`, `httpx` — pipelines into a Unix-style flow where each tool does one thing and outputs newline-delimited domains. The non-obvious traps are wildcard DNS records (which can make brute-force return millions of "valid" but identical names) and rate-limiting on both passive sources and your own resolver pool.

---

## 1. Why Enumerate Subdomains

The attacker (or auditor) wants the union of all hostnames belonging to a target organization, including:

- The flagship app (`www.example.com`)
- Marketing landing pages (`careers.example.com`, `blog.example.com`)
- Internal tools accidentally published (`admin.example.com`, `staging.example.com`, `kibana.example.com`)
- Customer-facing portals (`partners.example.com`, `support.example.com`)
- Forgotten infra from acquisitions and pet projects (`oldproduct.example.com`, `legacy.example.com`)

In bug-bounty terms, the **scope** is usually `*.example.com` and the bounty depends on what you find — staging environments running unauthenticated debug endpoints, internal admin panels exposed via misconfigured ingress, or S3 buckets pointed at by abandoned CNAMEs ([subdomain takeover](https://github.com/EdOverflow/can-i-take-over-xyz)).

In legitimate scraping terms, subdomain enumeration finds API endpoints (`api.example.com`, `gql.example.com`), lightweight mobile endpoints (`m.example.com`, `mobile.example.com`), and partner/data subdomains often missing from the marketing site's navigation.

In market intel, it reveals the size and shape of an organization's tech estate.

---

## 2. Passive Enumeration — Sources That Already Know

Passive enumeration queries datasets that already know subdomains exist. You never send a packet to the target's DNS or HTTP infrastructure.

### 2.1 Certificate Transparency Logs

Every publicly-trusted TLS certificate issued since 2015 is logged in append-only **Certificate Transparency** logs (specified in [RFC 6962](https://datatracker.ietf.org/doc/html/rfc6962), updated to [RFC 9162](https://datatracker.ietf.org/doc/html/rfc9162)). Browsers refuse certificates not present in CT logs. This makes CT logs the single best passive source for subdomains: any name that ever got a Let's Encrypt or commercial cert is in there forever, even if the host is now retired.

The two everyday CT search interfaces:

- **`crt.sh`** ([crt.sh](https://crt.sh)) — operated by Sectigo, free, allows wildcard queries. The HTML output supports `?q=%25.example.com&output=json` for JSON. No documented rate limit but be courteous.
- **Censys CT search** ([search.censys.io](https://search.censys.io)) — paid for serious use; has structured filtering by CN, SAN, validity dates.

```bash
# crt.sh JSON, dedup by name, strip wildcards
curl -s 'https://crt.sh/?q=%25.example.com&output=json' \
  | jq -r '.[].name_value' \
  | sed 's/\*\.//g' \
  | tr '[:upper:]' '[:lower:]' \
  | sort -u
```

**Caveat:** CT only sees certificates that were submitted to public logs. Internal CAs, self-signed certs, and pre-2015 certs are invisible.

### 2.2 DNS Aggregators

Several services collect passive DNS — historical records observed by recursive resolvers worldwide. They show what *was* live, not just what is.

- [SecurityTrails](https://securitytrails.com/) — passive DNS + WHOIS history, free tier with API key
- [VirusTotal](https://www.virustotal.com/) — its "Relations" tab shows subdomains observed from URL submissions
- [AlienVault OTX](https://otx.alienvault.com/) — passive DNS + threat intel
- [URLScan.io](https://urlscan.io/) — public scan results expose every subdomain a submitter visited
- [DNSdumpster](https://dnsdumpster.com/) — free, web-only, decent baseline
- [Hurricane Electric BGP](https://bgp.he.net/) — `dns.bufferover.run` style passive DNS (covered in doc 05)

These are the bread and butter of passive recon. `subfinder` (below) wraps ~50 such sources behind one CLI.

### 2.3 Search Engine and Archive Sources

- **Google/Bing dorks** — `site:example.com -www` reveals indexed subdomains
- **Wayback Machine** ([web.archive.org](https://web.archive.org)) via the [CDX API](https://github.com/internetarchive/wayback/tree/master/wayback-cdx-server) — see doc 03 for the protocol
- **Common Crawl** ([commoncrawl.org](https://commoncrawl.org)) — petabyte-scale web index; URL indexes are queryable per-month

### 2.4 The crt.sh Pivot

A recurring trick: when an org reuses a non-default Subject Alternative Name (SAN), search CT for the SAN to find sibling certificates. `gau` (`getallurls`) and `waybackurls` complement this by returning historical URLs from Wayback + CommonCrawl + URLscan in one go.

---

## 3. Active Enumeration — DNS Brute Force and Resolution

Active enumeration tests candidate hostnames by sending DNS queries. The candidates come from wordlists; the queries hit recursive resolvers (typically a curated list of public/healthy resolvers).

### 3.1 Wordlists

The community has built large, well-known wordlists tuned for subdomain discovery:

| Wordlist | Maintainer | Size | Use case |
|----------|------------|------|----------|
| [SecLists `dns/` directory](https://github.com/danielmiessler/SecLists/tree/master/Discovery/DNS) | Daniel Miessler | thousands → millions | The community standard; multiple sizes |
| [`commonspeak2-wordlists`](https://github.com/assetnote/commonspeak2-wordlists) | Assetnote | varies | Built from BigQuery scans of GitHub, Reddit, etc. — discovers names humans actually use |
| [`jhaddix/all.txt`](https://gist.github.com/jhaddix/86a06c5dc309d08580a018c66354a056) | Jason Haddix | ~2M | Aggregated from years of bug-bounty work |
| [`n0kovo_subdomains`](https://github.com/n0kovo/n0kovo_subdomains) | n0kovo | tiered | Tiered by frequency — small/medium/large |

Choosing wordlist size is a budget call: a 100k-line list against ~20 resolvers takes minutes; a 5M-line list takes hours and saturates resolvers (and may get you rate-limited by upstreams).

### 3.2 Resolvers

Public resolvers (`1.1.1.1`, `8.8.8.8`, `9.9.9.9`) rate-limit aggressive bursts. Serious enumeration uses curated resolver lists (`trickest/resolvers`, `proabiral/Fresh-Resolvers`) of healthy public servers. `dnsvalidator` (Bishop Fox) generates one fresh by testing for resolver lying (servers that return `NXDOMAIN` consistently and do not poison results).

### 3.3 The Tools

- **[`massdns`](https://github.com/blechschmidt/massdns)** — the underlying high-throughput DNS stub resolver; tens of thousands of queries/sec
- **[`puredns`](https://github.com/d3mondev/puredns)** — wraps `massdns` with wildcard handling and trusted-resolver verification
- **[`shuffledns`](https://github.com/projectdiscovery/shuffledns)** — ProjectDiscovery's wrapper; also handles wildcards
- **[`dnsx`](https://github.com/projectdiscovery/dnsx)** — fast resolver and DNS toolkit (A, AAAA, CNAME, MX, NS, TXT)
- **[`amass enum -active`](https://github.com/owasp-amass/amass)** — full-featured, slower, integrates passive+active+permutation in one run

```bash
# Brute-force enumerate with puredns
puredns bruteforce wordlists/all.txt example.com \
  -r resolvers.txt \
  --rate-limit 10000 \
  -w resolved.txt
```

---

## 4. Wildcard DNS — The Footgun That Inflates Every Result

If `*.example.com` resolves to a single IP (a CDN catch-all, a "did you mean?" page), then a brute-force query for `nonexistent-random-name.example.com` returns a valid A record — and your wordlist generates millions of false positives.

The fix is **wildcard fingerprinting**: query a randomly-generated subdomain (`x9f7a2k1.example.com`) and capture the response. Then filter out any candidate whose response matches the wildcard fingerprint. `puredns` and `shuffledns` do this automatically; `amass` does it; ad-hoc `massdns` runs do not.

A subtler trap: some wildcards return *different* IPs round-robin (e.g., a fleet of CDN edges). Single-IP wildcard detection misses these. Robust wildcard handling uses multiple probe queries and treats the response set as a fingerprint.

---

## 5. The ProjectDiscovery and OWASP Amass Toolchain

The two most-used toolchains:

### 5.1 ProjectDiscovery

[ProjectDiscovery](https://github.com/projectdiscovery) is a suite of focused Go tools that pipe together via stdin/stdout:

| Tool | Job |
|------|-----|
| [`subfinder`](https://github.com/projectdiscovery/subfinder) | Passive subdomain enumeration over ~50 sources |
| [`dnsx`](https://github.com/projectdiscovery/dnsx) | Bulk DNS resolution and record-type enumeration |
| [`shuffledns`](https://github.com/projectdiscovery/shuffledns) | Active brute force with wildcard handling |
| [`httpx`](https://github.com/projectdiscovery/httpx) | HTTP probing — title, status, tech detection |
| [`naabu`](https://github.com/projectdiscovery/naabu) | Fast port scanner |
| [`nuclei`](https://github.com/projectdiscovery/nuclei) | Template-based vulnerability scanner |
| [`katana`](https://github.com/projectdiscovery/katana) | JS-aware crawler |

```bash
# Classic ProjectDiscovery pipeline
subfinder -d example.com -silent \
  | dnsx -silent -a -resp-only \
  | httpx -silent -title -status-code -tech-detect
```

### 5.2 OWASP Amass

[Amass](https://github.com/owasp-amass/amass) is the older, more comprehensive alternative — Go-based, sponsored by OWASP. It runs passive + active + name-permutation + ASN-based discovery in one invocation, persists results in a graph DB, and supports differential runs. Slower than `subfinder` but finds more, especially with `-active` and `-brute`.

```bash
amass enum -d example.com -active -brute -w wordlists/best-dns-wordlist.txt
```

### 5.3 Other Notable Tools

- [`assetfinder`](https://github.com/tomnomnom/assetfinder) (Tomnomnom) — minimal passive-only enumerator
- [`findomain`](https://github.com/Findomain/findomain) — Rust, fast passive
- [`chaos` CLI](https://github.com/projectdiscovery/chaos-client) — queries ProjectDiscovery's curated dataset of 100M+ subdomains
- [`subbrute`](https://github.com/TheRook/subbrute) — older Python tool, still serviceable

---

## 6. GitHub, Code Search, and Source-Leak Pivots

Internal subdomains often leak through:

- **GitHub code search** for the apex domain — repos referencing `internal.example.com` in config, CI, or docs
- **Public S3/GCS buckets** named after subdomains
- **JS bundles** containing API base URLs (covered in docs 11–13)
- **CSP headers** listing allowed origins
- **Slack/Discord/Telegram leaks** of staging URLs
- **Postman public workspaces** ([go.postman.co/explore](https://go.postman.co/explore)) often expose internal API base URLs

```bash
# GitHub code search via API (requires auth)
gh api -X GET search/code -f q='example.com extension:env' \
  --jq '.items[].html_url'
```

The [`github-subdomains`](https://github.com/gwen001/github-subdomains) and [`github-search`](https://github.com/gwen001/github-search) tools automate the GitHub angle.

---

## 7. Pipelining: From a Domain to a Triage-Ready Asset List

A typical recon pipeline:

```bash
TARGET=example.com

# 1. Passive enumeration
subfinder -d "$TARGET" -all -silent > subs_passive.txt

# 2. Active brute force with wildcard handling
puredns bruteforce wordlists/best-dns-wordlist.txt "$TARGET" \
  -r resolvers.txt --quiet > subs_active.txt

# 3. Merge, dedup, resolve, drop wildcards
cat subs_passive.txt subs_active.txt | sort -u \
  | dnsx -silent -a -resp -wd "$TARGET" \
  | tee resolved.txt

# 4. HTTP probe, capture title and status
cat resolved.txt | awk '{print $1}' \
  | httpx -silent -title -status-code -tech-detect -json > probed.jsonl

# 5. Triage — alive hosts with interesting status codes
jq -r 'select(.status_code != 404) | [.url, .status_code, .title] | @tsv' probed.jsonl
```

The output is an actionable list: live hostnames, their HTTP status, page title, and detected technology stack. From here, downstream work splits by intent — vuln scanning (`nuclei`), scrape-endpoint discovery (`katana`), or visual triage (`gowitness` / `aquatone` for screenshots).

---

## 8. Ethics and Authorization

Subdomain enumeration itself is **passive observation of publicly-published infrastructure** — querying CT logs and public DNS is not illegal in any jurisdiction. Active brute force generates DNS traffic against the target's authoritative nameservers (and their upstream resolvers), which is also generally legal but can be a load problem.

What changes with **scope and intent**:

- **Bug-bounty programs** publish authorized scopes (`*.example.com`); enumerate freely within scope, stop at scope edges. Read the program rules — some explicitly disallow brute force.
- **Pentesting under contract** has a Statement of Work that authorizes specific activities. Subdomain enumeration is universally allowed; exfiltration or active exploitation has its own rules.
- **Unauthorized active probing of vulnerabilities** discovered through enumeration crosses into [Computer Fraud and Abuse Act](https://www.justice.gov/criminal/criminal-ccips/computer-fraud-and-abuse-act) territory in the US (cf. *Van Buren v. United States*, 593 U.S. ___ (2021), which narrowed CFAA but did not legalize unauthorized access).
- **GDPR Article 6** doesn't prohibit collecting non-personal infrastructure data, but if your enumeration accidentally captures personal data (e.g., from a leaked staging site), retention rules attach.

The professional norm: enumerate broadly, but only probe deeper when you're authorized.

---

## Related

- [Internet Asset Scanners — Censys, Shodan, and Friends](02-internet-asset-scanners.md) — pivots from CT and banner data
- [Sitemap, robots.txt, and Crawl Surface](03-sitemap-robots-and-crawl-surface.md) — once you have hosts, enumerate paths
- [HTML & Tech-Stack Fingerprinting](04-html-and-techstack-fingerprinting.md) — the next step after `httpx -tech-detect`
- [IP, ASN, and Network Recon](05-ip-asn-and-network-recon.md) — pivots from resolved IPs to whole CIDR ranges
- [TLS Handshake and Certificates](../../security/fundamentals/06-tls-handshake-and-certificates.md) — what CT logs actually contain
- [DNS Deep Dive](../../networking/application-layer/dns-deep-dive.md) — the protocol behind every active resolution

## References

- [RFC 6962 — Certificate Transparency](https://datatracker.ietf.org/doc/html/rfc6962) — the spec that makes CT-based recon possible
- [RFC 9162 — Certificate Transparency Version 2.0](https://datatracker.ietf.org/doc/html/rfc9162) — current version
- [crt.sh](https://crt.sh) — Sectigo's free CT search
- [OWASP Amass](https://github.com/owasp-amass/amass) — the comprehensive enumerator
- [ProjectDiscovery](https://github.com/projectdiscovery) — the modern CLI toolchain
- [SecLists](https://github.com/danielmiessler/SecLists) — community wordlists
- [Assetnote `commonspeak2-wordlists`](https://github.com/assetnote/commonspeak2-wordlists) — BigQuery-derived lists
- [Computer Fraud and Abuse Act, 18 U.S.C. § 1030](https://www.law.cornell.edu/uscode/text/18/1030)
- [*Van Buren v. United States*, 593 U.S. ___ (2021)](https://www.supremecourt.gov/opinions/20pdf/19-783_k53l.pdf)
- [Jason Haddix — *The Bug Hunter's Methodology*](https://github.com/jhaddix/tbhm) — the canonical bug-bounty recon reference
