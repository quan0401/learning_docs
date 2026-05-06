---
title: "Internet Asset Scanners — Censys, Shodan, and Friends"
date: 2026-05-06
updated: 2026-05-06
tags: [recon, osint, censys, shodan, fofa, banner-grabbing, certificate-transparency]
---

# Internet Asset Scanners — Censys, Shodan, and Friends

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `recon` `osint` `censys` `shodan` `fofa` `banner-grabbing` `certificate-transparency`

---

## Table of Contents

1. [What Internet Scanners Actually Are](#1-what-internet-scanners-actually-are)
2. [Shodan — The Search Engine for Banners](#2-shodan--the-search-engine-for-banners)
3. [Censys — Structured, Certificate-First](#3-censys--structured-certificate-first)
4. [FOFA, ZoomEye, Hunter.how, Netlas — The Alternatives](#4-fofa-zoomeye-hunterhow-netlas--the-alternatives)
5. [Certificate Pivots](#5-certificate-pivots)
6. [Favicon Hashing — The Same Login Page in 10,000 Places](#6-favicon-hashing--the-same-login-page-in-10000-places)
7. [Banner-Based Fingerprinting](#7-banner-based-fingerprinting)
8. [Quotas, Cost, and Operational Hygiene](#8-quotas-cost-and-operational-hygiene)
9. [Ethics and Legality](#9-ethics-and-legality)

## Summary

Internet asset scanners are services that continuously scan the entire IPv4 (and increasingly IPv6) address space, capture protocol banners and TLS certificates from every responsive host, and expose the dataset behind a search interface. They flip the recon model: instead of asking "what subdomains does example.com have?" you ask "show me every host on the internet running a JIRA instance with a default certificate," or "show me every host with this organization's TLS certificate," or "show me every host serving this favicon hash." The four pivots that matter — **certificate**, **favicon**, **banner string**, **HTTP body fragment** — turn one fingerprint into the full set of hosts using that fingerprint, regardless of whether they appear in DNS or have a marketing-grade hostname. Shodan and Censys are the dominant Western services; FOFA, ZoomEye, Hunter.how, and Netlas serve the same role with different coverage (especially in APAC) and different pricing. Each has its own search syntax, but the search-by-pivot patterns are universal.

---

## 1. What Internet Scanners Actually Are

A typical scanner like Censys or Shodan operates a continuously-running fleet of scanning probes that:

1. Send packets to every IP address on a configured set of ports (often 100+ TCP ports plus a curated UDP set).
2. For each responsive host, capture the full protocol handshake, TLS certificate, HTTP response headers, body, and any service banners (SSH version, FTP banner, Redis info, etc.).
3. Index everything into a searchable database, typically refreshed every few days to weeks per port.
4. Expose a structured query language so you can ask "what hosts present this certificate's serial number" or "what hosts have an HTTP body containing this string."

The dataset is a snapshot of the public internet's posture. Shodan was founded by John Matherly in 2009 and pioneered the field; Censys spun out of the [University of Michigan zmap](https://github.com/zmap/zmap) project in 2015 and brought structured querying and academic rigor.

---

## 2. Shodan — The Search Engine for Banners

[Shodan](https://www.shodan.io/) is the oldest and most widely-known. Its primary product is keyword search across captured banners and metadata.

### 2.1 Search Syntax

Shodan uses simple keyword + filter syntax. Common filters:

| Filter | Example | Meaning |
|--------|---------|---------|
| `port:` | `port:5432` | Specific port |
| `org:` | `org:"Example Corp"` | Owning organization (from WHOIS) |
| `ssl:` | `ssl:"example.com"` | TLS subject contains string |
| `ssl.cert.serial:` | `ssl.cert.serial:abc123` | Exact certificate serial |
| `hostname:` | `hostname:.example.com` | Reverse DNS suffix match |
| `country:` | `country:US` | Geolocation |
| `http.title:` | `http.title:"Welcome to nginx"` | HTTP title |
| `http.html:` | `http.html:"some-string"` | Substring of body |
| `http.favicon.hash:` | `http.favicon.hash:-310156269` | favicon mmh3 (see §6) |
| `product:` | `product:"Apache httpd"` | Detected product |
| `version:` | `version:"2.4.49"` | Detected version |
| `vuln:` | `vuln:CVE-2021-44228` | Hosts believed vulnerable |

```text
# Find every Spring Boot Actuator exposed publicly
http.title:"Whitelabel Error Page" port:8080

# Find every host with a TLS certificate for example.com
ssl.cert.subject.cn:example.com
```

### 2.2 CLI and API

```bash
# Free CLI (after `pip install shodan` and `shodan init <KEY>`)
shodan search 'org:"Example Corp" port:443' --fields ip_str,port,hostnames

# Raw API
curl -s "https://api.shodan.io/shodan/host/search?key=$KEY&query=org:%22Example%22"
```

### 2.3 Live vs Historical

Shodan's data is a recurring crawl, not real-time. A host may appear in results for days or weeks after going offline. The `last_seen` timestamp on each result indicates freshness.

---

## 3. Censys — Structured, Certificate-First

[Censys](https://search.censys.io) emphasizes structured data and certificate analysis. It runs continuous scans plus full Certificate Transparency log ingestion (every public certificate is in Censys regardless of whether the host is currently up).

### 3.1 Search Syntax

Censys uses a JSON-path-like query language. Documented at [search.censys.io/search/language](https://search.censys.io/search/language).

```text
# Hosts with TLS certificate matching example.com
services.tls.certificates.leaf_data.names: example.com

# All Cobalt Strike servers (fingerprint by JARM hash)
services.jarm.fingerprint: 07d14d16d21d21d07c42d43d000000d4f0f7e8c4ff7e3c4e1f9b5b1d4f6c5b5b

# Specific HTTP body content
services.http.response.body: "X-Powered-By: ColdFusion"
```

The killer Censys feature is its three top-level indexes:

- **Hosts** — every responsive IP and the services on it
- **Certificates** — every public TLS certificate (CT-fed)
- **Virtual Hosts** — HTTP responses keyed by SNI/Host header (one IP can serve many)

### 3.2 Certificate Index

The certificate index is independent of host status. Search for any name field — Common Name, SAN, issuer DN, subject DN — and find every cert ever issued matching it. Then cross-reference back to hosts presenting that cert.

```text
# All certs naming example.com or any subdomain
parsed.names: example.com

# All certs issued by a specific issuer with a specific subject pattern
parsed.issuer.organization: "Let's Encrypt" and parsed.subject.organization: "Example Corp"
```

### 3.3 CLI

```bash
# Censys CLI: pip install censys
censys search 'services.tls.certificates.leaf_data.names: example.com' \
  --index-type hosts --pages 5
```

---

## 4. FOFA, ZoomEye, Hunter.how, Netlas — The Alternatives

Several non-Western services compete with Shodan/Censys, sometimes with better coverage of regional networks:

### 4.1 FOFA

[FOFA](https://en.fofa.info/) — Chinese, broad coverage, popular for APAC asset discovery. Uses an SQL-flavored DSL:

```text
# Hosts with example.com cert
cert="example.com"

# All hosts running a specific application
title="Welcome to JIRA" && body="atlassian"
```

### 4.2 ZoomEye

[ZoomEye](https://www.zoomeye.org/) — also Chinese, Knownsec project. Strong device-search focus (cameras, routers, ICS).

### 4.3 Hunter.how

[Hunter.how](https://hunter.how/) — Chinese, similar feature set, sometimes finds hosts the others miss.

### 4.4 Netlas

[Netlas.io](https://netlas.io/) — newer entrant (Russian-based), good free tier, query DSL similar to Shodan. Especially strong at HTTP responses and DNS records.

### 4.5 BinaryEdge

[BinaryEdge](https://www.binaryedge.io/) — defunct/acquired, but its data feeds historical pivots in some commercial threat-intel platforms.

### 4.6 Coverage Differences

The same query returns different results across services because:

- Each runs its own scanner fleet from different ASNs
- Targets aggressively block some scanner ASNs but not others
- Different default port lists
- Different scan cadences (Shodan ~weekly, Censys ~daily for HTTPS, FOFA varies)

For thorough coverage, query 2–3 services and union results.

---

## 5. Certificate Pivots

A TLS certificate is the single best pivot for cross-organizational asset discovery, because:

- **Many hostnames per cert** — one cert's SAN list reveals the whole product surface
- **One issuer, many certs** — internal CAs leave consistent issuer DNs across all internal hosts
- **Serial numbers are unique per CA** — but reused across reissues; check fingerprints
- **Subject DN organizational fields** — `O=` and `OU=` are often hand-typed and consistent

Pivot pattern:

1. Find one cert with a known target connection (e.g., from a `crt.sh` query or a single host's TLS handshake).
2. Note its **subject DN** (not just CN), **issuer DN**, and **SAN list**.
3. Search Censys/Shodan for other hosts presenting any cert with the same subject DN, especially the same `O=` field.
4. Cross-reference SAN entries against your subdomain list — find subdomains whose certs name *other* hostnames you didn't enumerate yet.

```text
# Censys: every host whose cert subject organization matches
services.tls.certificates.leaf_data.subject.organization: "Example Corp"

# Shodan equivalent
ssl.cert.subject.O:"Example Corp"
```

This frequently surfaces forgotten infrastructure: `internal-tools.example.com` resolves to a private IP but the cert lives in CT logs forever.

---

## 6. Favicon Hashing — The Same Login Page in 10,000 Places

A favicon is the small icon a site serves at `/favicon.ico`. Many web applications (CMS admin pages, default product installs, internal tools) ship with a stock favicon. Two hosts running the same product return the **identical favicon bytes**.

The convention is to hash the favicon with [MurmurHash3 (mmh3)](https://github.com/aappleby/smhasher) (32-bit signed) and use the hash as a fingerprint. Shodan stores `http.favicon.hash` for every scanned HTTP response.

### 6.1 Computing the Hash

```python
# pip install mmh3 requests
import mmh3
import requests
import codecs

response = requests.get("https://target.example.com/favicon.ico")
favicon_b64 = codecs.encode(response.content, "base64")
fav_hash = mmh3.hash(favicon_b64)
print(fav_hash)  # e.g., -310156269
```

### 6.2 Tools

- [`FavFreak`](https://github.com/devanshbatham/FavFreak) (Devansh Batham) — CLI that hashes favicons and queries Shodan
- [`favicon-hash.kmsec.uk`](https://favicon-hash.kmsec.uk/) — web UI; paste a URL, get the hash
- [`favihunt`](https://github.com/sansatart/favihunt) — bulk hashing pipelined through Shodan

### 6.3 Why This Works So Well

Default product installs all share favicons. So once you know "the Cisco IOS XE web UI default favicon hashes to 1265477436," one Shodan query returns every Cisco IOS XE web UI on the internet. Combined with a known-vulnerable banner string, this trivially finds at-risk hosts at scale (which is also why scanners run nonstop on disclosed-vuln favicons during the days after a CVE drops — the [Cisco IOS XE 2023 mass exploitation](https://www.cisa.gov/news-events/cybersecurity-advisories/aa23-289a) is a textbook example).

---

## 7. Banner-Based Fingerprinting

Beyond favicons, the scanners capture full HTTP/TLS/SSH/FTP banners and let you search them. Common pivots:

- **HTTP `Server:` header** — `Server: Apache/2.4.49 (Unix)` finds vulnerable installs
- **HTTP body fragments** — `<title>Default Page</title>` finds default installs
- **TLS JARM fingerprint** ([Salesforce's open-source TLS fingerprint](https://github.com/salesforce/jarm)) — fingerprints the TLS server based on its handshake responses; unique to server library + version
- **JA3S** — server-side TLS fingerprint (covered in doc 09)
- **SSH banners** — `SSH-2.0-OpenSSH_8.0p1` reveals patch level
- **SMB/RDP banners** — Windows version disclosure

Combined queries — `ssl:"Example Corp" AND http.html:"X-Internal-Build"` — narrow tens of thousands of hosts to a handful of internal-tooling instances.

---

## 8. Quotas, Cost, and Operational Hygiene

### 8.1 Free Tiers

| Service | Free tier reality |
|---------|-------------------|
| Shodan | Free account: very limited (1 search, 100 results). $59 lifetime "Membership" upgrades to ~10k results/month |
| Censys | Free 250 queries/month with academic-grade limits per query |
| FOFA | Free w/ very small monthly query budget |
| ZoomEye | Free w/ small query allowance |
| Hunter.how | Free 100 queries/month |
| Netlas | Most generous free tier — 50 requests/day |

### 8.2 Operational Hygiene

- **Don't burn quotas on browsing** — pull bulk data via API and grep locally.
- **Cache results** — scanner snapshots only update every few days; don't repeat the same query.
- **Combine** — don't trust one scanner's coverage; query 2–3 and union.
- **Authenticate properly** — most APIs require an API key; rotate it like any secret.

### 8.3 Bulk Data

For research-grade work, [Internet-Wide Scan Data Repository (Censys + zmap)](https://scans.io/) and [Rapid7's Open Data](https://opendata.rapid7.com/) (recently scaled back) historically published full scan corpora. CommonCrawl publishes the HTML side ([commoncrawl.org](https://commoncrawl.org/the-data/get-started/)).

---

## 9. Ethics and Legality

The scanners scan; you only query their indexes. Querying public banner data is universally legal — you're reading a database, not touching the target.

**Where it gets dicey:**

- Searching for *vulnerable hosts* with intent to exploit unauthorized targets crosses legal lines (CFAA, Computer Misuse Act, etc.). The scan-and-pivot tooling is dual-use; the legality lives in your follow-up.
- Some services' ToS prohibit automated bulk export beyond paid tiers — read them before scripting.
- Aggregating scanner data into a profile of a specific natural person (rare in this domain, but possible if the host runs a personal site) attaches GDPR.
- Sanctions screening — FOFA, ZoomEye, Hunter.how are based in jurisdictions where US/EU data-export and sanctions concerns may apply for regulated industries. Check before integrating into commercial products.

The professional norm: scanners are a recon library, not a license to exploit. Use them within authorized scope.

---

## Related

- [Subdomain & Asset Enumeration](01-subdomain-and-asset-enumeration.md) — the DNS-first complement; the two are paired
- [Sitemap, robots.txt, and Crawl Surface](03-sitemap-robots-and-crawl-surface.md) — what to do once you have hosts
- [HTML & Tech-Stack Fingerprinting](04-html-and-techstack-fingerprinting.md) — extends banner fingerprinting with active probing
- [IP, ASN, and Network Recon](05-ip-asn-and-network-recon.md) — pivots on ownership rather than fingerprint
- [Bot Detection Internals](../extraction/09-bot-detection-internals.md) — JARM/JA3 are also bot-side fingerprints
- [TLS Handshake and Certificates](../../security/fundamentals/06-tls-handshake-and-certificates.md) — why certificate pivots work

## References

- [Shodan API documentation](https://developer.shodan.io/api)
- [Censys Search Language reference](https://search.censys.io/search/language)
- [FOFA query reference](https://en.fofa.info/api)
- [Netlas search documentation](https://docs.netlas.io/)
- [ZMap project (the scanner Censys grew out of)](https://github.com/zmap/zmap)
- [JARM — Active TLS server fingerprinting (Salesforce)](https://github.com/salesforce/jarm)
- [MurmurHash3 (mmh3)](https://github.com/aappleby/smhasher) — favicon hashing primitive
- [FavFreak](https://github.com/devanshbatham/FavFreak) — favicon-hashing Shodan helper
- [CISA Advisory AA23-289A — Cisco IOS XE Mass Exploitation (2023)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa23-289a) — favicon-pivot scale demonstrated
- [Internet-Wide Scan Data Repository — scans.io](https://scans.io/)
