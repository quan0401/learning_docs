---
title: "Proxies and IP Rotation"
date: 2026-05-06
updated: 2026-05-06
tags: [proxies, ip-rotation, residential-proxies, datacenter, mobile-proxies, bot-detection, ethics]
---

# Proxies and IP Rotation

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `proxies` `ip-rotation` `residential-proxies` `datacenter` `mobile-proxies` `bot-detection` `ethics`

---

## Table of Contents

1. [Why Proxies Are Required for Production Scraping](#1-why-proxies-are-required-for-production-scraping)
2. [The Four Proxy Types — and Why Reputation Matters More Than Speed](#2-the-four-proxy-types--and-why-reputation-matters-more-than-speed)
3. [Sticky vs Rotating Sessions](#3-sticky-vs-rotating-sessions)
4. [Geographic Targeting](#4-geographic-targeting)
5. [Sourcing Ethics — The Residential Proxy Problem](#5-sourcing-ethics--the-residential-proxy-problem)
6. [The Vendor Landscape](#6-the-vendor-landscape)
7. [Pool Health, Retry, and Backoff](#7-pool-health-retry-and-backoff)
8. [Self-Hosted Egress on Cloud Infrastructure](#8-self-hosted-egress-on-cloud-infrastructure)
9. [Operational Architecture for Proxy-Aware Crawlers](#9-operational-architecture-for-proxy-aware-crawlers)

## Summary

Past a few thousand requests/day, a single egress IP gets you blocked. Proxies are how production scrapers route requests through diverse IPs and distribute load below per-IP rate-limit thresholds. The four proxy types — **datacenter**, **residential**, **ISP** (a hybrid), and **mobile** — differ primarily in *IP reputation*: how scrapers' targets categorize the source. Datacenter is cheap, fast, and visibly datacenter (hosting providers' ranges show up in MaxMind, IPQS, and Cloudflare's datasets). Residential and mobile come from real consumer devices and are very hard to flag as scraper traffic — but the supply chain raises serious sourcing-ethics questions. **Sticky** sessions keep the same IP for a session window (essential for stateful logins); **rotating** sessions switch IPs per-request (essential for blanket coverage). The vendor landscape (Bright Data, Oxylabs, IPRoyal, Smartproxy, NetNut, etc.) competes on pool size, geographic coverage, and the legal/ethical defensibility of their sourcing. Pool health monitoring — track success rate per IP, ASN, and country — is the operational discipline that keeps the whole system working.

---

## 1. Why Proxies Are Required for Production Scraping

Anti-bot systems use IP as the primary rate-limit dimension. The thresholds vary, but the order of magnitude is consistent:

| Site profile | Typical per-IP threshold |
|--------------|-------------------------|
| Permissive site, no anti-bot | 1000+ requests/hour |
| Default Cloudflare WAF | ~100–300 requests/hour |
| Cloudflare Bot Management or Akamai BMP | ~10–60 requests/hour from datacenter ASNs |
| Sites that aggressively block scrapers | Single-digit requests before challenge |

Past those thresholds you start seeing 403, 429, CAPTCHA challenges, cookie-stripping middlewares, or fingerprint-based shadow-bans. The fix is to *distribute* requests across many IPs.

**Beyond rate limiting**, proxies enable:

- **Geographic targeting** — see prices/inventory as a user in another country
- **Session diversity** — many concurrent independent "users"
- **Egress IP control** — separate scraper traffic from your application IPs
- **Compliance** — some data is licensable only with regional access

---

## 2. The Four Proxy Types — and Why Reputation Matters More Than Speed

### 2.1 Datacenter Proxies

IPs from cloud providers, hosting providers, and dedicated proxy networks. Cheap (often $0.50–$2.00 per IP/month, or sub-cent per GB), fast (no last-mile latency), abundant.

The fundamental problem: **datacenter IPs are visibly datacenter**. Public databases (MaxMind's GeoIP `traits.is_anonymous_proxy` / `traits.user_type`, IPQualityScore, IP2Proxy, Spamhaus) classify them as such, and bot-detectors aggressively flag them. Major sites block whole datacenter ASNs — AWS (`AS16509`), DigitalOcean (`AS14061`), GCP (`AS396982`), Hetzner (`AS24940`), OVH (`AS16276`).

For *scraping sites without sophisticated anti-bot*, datacenter is fine. For Cloudflare-protected, Akamai-protected, or financial/e-commerce sites, datacenter proxies are usually a non-starter at scale.

### 2.2 Residential Proxies

IPs from consumer ISPs — Comcast, Verizon, BT, Deutsche Telekom, etc. The proxy traffic egresses through a real consumer's home internet connection (their devices, their router, their ISP). To the target site, the request looks indistinguishable from a real user.

Reputation: typically clean (residential ranges are categorized as such, but not as proxies). Cost: $5–$15 per GB historically; the price has come down somewhat.

The supply: this is where it gets ethically fraught (§5).

### 2.3 ISP Proxies

A middle ground: IPs registered to ISPs but hosted in datacenters, often through partnerships between proxy providers and ISPs. Reputation appears residential to most classifiers, but they're stable and fast like datacenter.

Cost is between datacenter and residential. Quality is highly variable per provider.

### 2.4 Mobile Proxies

IPs from cellular carriers — actual phones (or SIM banks) on AT&T, T-Mobile, Verizon Wireless, EE, Vodafone. Mobile carriers use carrier-grade NAT (CGNAT), so each mobile IP serves thousands of real users. Blocking a mobile IP blocks legitimate users wholesale, which makes anti-bot extremely conservative about flagging mobile IPs.

The result: mobile IPs are effectively un-blockable. They are also expensive ($50–$300 per GB) and slower (mobile network latency, NAT overhead).

For high-value, well-defended targets, mobile proxies are often the only thing that works.

### 2.5 Reputation Visibility

You can check what your egress looks like:

```bash
MY_IP=$(curl -s ifconfig.me)
curl -s "https://ipinfo.io/$MY_IP/privacy?token=$IPINFO_TOKEN"
# { "vpn": false, "proxy": false, "tor": false, "relay": false, "hosting": true, "service": "" }
```

[IPQualityScore](https://www.ipqualityscore.com/), [ipinfo.io Privacy Detection](https://ipinfo.io/products/privacy-vpn-detection-api), [IP2Proxy](https://www.ip2location.com/database/ip2proxy), and [MaxMind GeoIP2 Anonymous IP](https://www.maxmind.com/en/geoip2-anonymous-ip-database) are the major commercial reputation feeds; expect targets to use one of them.

---

## 3. Sticky vs Rotating Sessions

The two access patterns:

### 3.1 Rotating Sessions

Each request egresses through a different IP from the pool. The provider exposes a single endpoint; the gateway picks an IP per connection.

```text
$ for i in {1..3}; do curl --proxy http://user:pass@gw.proxy.example:8000 ifconfig.me; done
198.51.100.42
203.0.113.7
198.51.100.99
```

Use cases: stateless scraping (search results, public listings), high-throughput crawling.

### 3.2 Sticky Sessions

The provider associates a session ID (often via username syntax) with a specific IP for a configurable window — 10 minutes, 30 minutes, "as long as the IP stays available," etc.

```text
# Bright Data syntax: session-XXXX in username binds to one IP
$ curl --proxy http://user-session-mysession1:pass@gw.proxy.example:22225 ifconfig.me
198.51.100.42
$ curl --proxy http://user-session-mysession1:pass@gw.proxy.example:22225 ifconfig.me
198.51.100.42  # same IP for "session1"
```

Use cases: anything stateful — login flows, multi-step form submissions, shopping carts, captcha-token continuations.

### 3.3 Sticky Session Lifetime

Three failure modes that end a sticky session:

1. **TTL expiry** — provider rotates after the configured window
2. **Underlying IP goes offline** — the residential device disconnected
3. **Target blocks that IP** — your session goes dead even though the proxy is still serving

Robust scrapers monitor for blocking signals (HTTP 403, CAPTCHA challenges) and rotate proactively before the target hardens its ban.

### 3.4 Per-Request vs Per-Session Mix

Realistic crawlers mix the two: rotating for the bulk of unauthenticated requests, sticky for any flow that requires login or multi-step state.

---

## 4. Geographic Targeting

Most providers expose country-level (and sometimes city-level) targeting via username syntax or endpoint:

```text
# Bright Data: country code in username
http://user-country-us:pass@gw.proxy.example:22225

# Smartproxy: subdomain endpoint
http://user:pass@us.smartproxy.example:10000

# Many providers also support: city, state/region, ASN
http://user-country-us-state-ca-city-sanfrancisco:pass@...
```

### 4.1 Why Geography Matters for Scraping

- **Pricing differs by region** — flights, hotels, e-commerce promotions
- **Inventory differs by region** — product availability, model variants
- **Content differs by region** — language defaults, GDPR-restricted content, geo-blocked media
- **Anti-bot tolerances differ by region** — some sites are stricter on traffic from outside their primary market

### 4.2 City-Level Targeting Quality

Provider city targeting accuracy is uneven. A "Los Angeles" filter often returns IPs MaxMind-tagged as anywhere in California. For state-level granularity, expect ~80% accuracy; for city, ~50–60%.

---

## 5. Sourcing Ethics — The Residential Proxy Problem

Residential proxy networks need real consumer devices. The supply comes from a few sources:

### 5.1 Compensated Opt-In SDKs

The legitimate model: SDK companies (Honeygain, IPRoyal Pawns, EarnApp, Pawns.app) embed in apps that pay users (cents per GB). The user knowingly trades bandwidth for compensation. Providers building on this model have a defensible supply chain.

### 5.2 SDK Bundling Without Clear Consent

The dicey middle ground: SDKs bundled into free apps where the disclosure is buried in the EULA. The user technically consented, but most users have no idea their device is being used as a proxy node. Several providers have been [investigated and sued](https://krebsonsecurity.com/2023/10/proxy-scammers-targeted-by-blockchain-fraud/) over this.

### 5.3 Botnet-Sourced Proxies (Illegitimate)

The illegitimate end: residential proxies sourced from malware-infected devices. Operators and customers have been [criminally prosecuted](https://www.justice.gov/opa/pr/911-s5-botnet-dismantled-and-its-administrator-arrested-coordinated-international-operation):

- **911 S5** (May 2024 takedown) — DOJ described it as "likely the world's largest residential proxy service"; YunHe Wang arrested for operating the botnet
- **RSOCKS** (June 2022 takedown) — DOJ disrupted; sourcing was IoT/router malware
- **Faceless** (2022 ESET research) — long-running residential proxy service tied to malware
- Multiple other defunct providers were quietly malware-sourced

### 5.4 What This Means Operationally

If you use residential proxies professionally:

- **Pick providers with documented supply chains** — Bright Data, Oxylabs, NetNut, IPRoyal, Smartproxy publish opt-in SDK disclosures
- **Avoid suspiciously cheap providers** — sub-$1 per GB residential strongly correlates with botnet sourcing
- **Read provider ToS** — what categories of sites they prohibit (most prohibit health, finance, fraud, account-takeover; some prohibit competitive scraping in specific verticals)
- **Document your due diligence** — for legal departments and downstream compliance

For market intel and bug-bounty work the question of supply-chain provenance becomes a real reputational issue. For LLM training data collection, ML training pipelines, or any product whose data you'll publicly publish, this is a board-level question.

### 5.5 The Mobile Proxy Variant

Mobile proxies often come from SIM farms operated by the provider. This is operationally legitimate but watch for the "device-bundled" mobile networks where the supply chain is uncertain.

---

## 6. The Vendor Landscape

A non-exhaustive cut of the major commercial providers:

| Provider | Strengths | Notes |
|----------|-----------|-------|
| [Bright Data](https://brightdata.com/) | Largest pool, strongest geographic coverage, Web Unlocker product | Originally Luminati; Hola Networks parent has historical sourcing controversies |
| [Oxylabs](https://oxylabs.io/) | Large pool, enterprise focus | Transparent SDK opt-in for residential |
| [IPRoyal](https://iproyal.com/) | Mid-tier pricing, good developer UX | Operates Pawns.app SDK explicitly |
| [Smartproxy](https://smartproxy.com/) | Developer-friendly, decent free trial | Self-serve focus |
| [NetNut](https://netnut.io/) | Direct-ISP partnerships, low latency | More expensive; less variety |
| [Soax](https://soax.com/) | Mobile + residential | Mid-tier |
| [GeoSurf](https://geosurf.com/) | Targeted geo coverage | Niche |
| [ScraperAPI](https://www.scraperapi.com/) | "Render-and-return" abstraction | Wraps proxies + browser fleet |
| [ZenRows](https://www.zenrows.com/) | Same; "AI" anti-bot bypass branding | |
| [Apify Proxy](https://apify.com/proxy) | Bundled with Apify platform | |

### 6.1 Specialty Players

- **Webshare**, **Storm Proxies**, **Proxy-Cheap** — budget-tier, mostly datacenter
- **AirProxy**, **Mobile-Proxies-Online** — mobile specialists
- **Proxyrack** — older, mixed-quality

### 6.2 Pricing Sketch (2025 range)

- Datacenter: $0.10–$1.00 per GB, or $1–$10 per IP/month
- ISP: $1–$3 per GB
- Residential: $4–$15 per GB
- Mobile: $30–$300 per GB

For a 10 TB/month scraping operation, residential proxies alone cost $40k–$150k/month. This drives architecture: route what you can through datacenter (cheap), escalate to residential only when datacenter fails.

---

## 7. Pool Health, Retry, and Backoff

### 7.1 Per-IP Success Tracking

Track per-IP outcomes:

```python
# Pseudocode
class IPHealth:
    requests: int
    successes: int       # 2xx
    blocks: int          # 403, 429, captcha
    errors: int          # 5xx, connect failure
    last_success: float
    blocked_until: float

def select_ip(pool):
    candidates = [ip for ip in pool if time.time() > ip.blocked_until]
    candidates.sort(key=lambda ip: ip.success_rate(), reverse=True)
    return candidates[0]
```

### 7.2 ASN-Level Health

Scale up: track success rate per ASN. If `AS396982` (GCP) drops from 90% success to 30%, the target started ASN-blocking. Stop sending to that ASN globally.

### 7.3 Retry With Different IP

Treat per-IP failures (especially 403/429) as a signal to *different* IP, not the same IP retried with backoff. Same-IP retries on a 429 just confirm the block.

### 7.4 Pre-Validation

For sticky-session use, pre-validate the IP before the first protected request:

```python
def validate(ip):
    r = requests.get("https://target.example.com/", proxies=ip.proxy_config(), timeout=5)
    return r.status_code == 200 and "captcha" not in r.text.lower()
```

---

## 8. Self-Hosted Egress on Cloud Infrastructure

For some workloads, building your own egress is competitive:

### 8.1 IPv4 Allocations on Major Clouds

- AWS: Elastic IPs are free up to a point; secondary IPs and additional ranges have small fees. AWS published [`/24` IP ranges](https://docs.aws.amazon.com/vpc/latest/userguide/aws-ip-ranges.html) covering all VPC traffic.
- GCP: Static external IPs cost ~$3/month each
- Azure: Public IP costs ~$3/month each
- OVH/Hetzner/Vultr: cheap, but heavily flagged as datacenter

### 8.2 Patterns

- **NAT Gateway with multiple Elastic IPs** — round-robin across a small pool you own
- **Many small VPSes** — one IP each, low cost per IP, more management
- **IPv6 prefix delegation** — for IPv6-aware targets, AWS/GCP give you `/56` or `/64` blocks; randomize the host bits per request
- **Cloud functions** (Lambda, Cloud Run) — random egress from large provider pools; sometimes useful for low-volume, geographically-distributed scraping

### 8.3 Limits

Self-hosted egress is the wrong choice when:

- The target blocks the entire ASN (datacenter detection)
- You need geographic diversity beyond the cloud's regions
- You need sticky residential-looking sessions

For target-permissive scraping with moderate volume, self-hosted is cost-effective. For anti-bot-heavy targets, residential is the only path.

---

## 9. Operational Architecture for Proxy-Aware Crawlers

A production scraper design:

```text
[Crawler] ─→ [Routing/policy layer] ─→ [Proxy pool manager] ─→ [Provider gateways]
                                              │
                                              ↓
                                        [IP health DB]
                                              ↑
                                              │
                                       [Block detection]
                                              ↑
                                              │
                                     [Response analyzer]
```

The **routing/policy layer** decides per-request:

- Datacenter or residential? (default datacenter; escalate on block)
- Geo: which country/state?
- Session: rotating or sticky?
- Concurrency: per-host limits

The **proxy pool manager** abstracts the provider — your code says "give me a US residential sticky session for 10 minutes," the manager reaches the appropriate provider with the right username syntax.

The **response analyzer** classifies outcomes (success / soft-block / hard-block / captcha) and feeds the IP health DB, which then influences future routing.

This separation of concerns makes it possible to swap providers, add a new geo, or change the rotation strategy without touching crawler code.

### 9.1 Open-Source Building Blocks

- **`scrapoxy`** ([scrapoxy.io](https://scrapoxy.io/)) — open-source proxy manager with pluggable backends (multiple cloud providers + commercial proxies)
- **`proxychains`** — LD_PRELOAD-style for ad-hoc traffic
- **HAProxy / Envoy** — for fronting a self-managed pool with load balancing
- Provider SDKs — Bright Data and Oxylabs publish official Python/Node clients

---

## Related

- [HTTP Scraping Fundamentals](06-http-scraping-fundamentals.md) — applies the same to your proxied connections
- [Headless Browsers](07-headless-browsers.md) — proxies + headless is the standard combination for anti-bot-heavy targets
- [Bot Detection Internals](09-bot-detection-internals.md) — the system on the other end your IP reputation feeds into
- [IP, ASN, and Network Recon](../discovery/05-ip-asn-and-network-recon.md) — the inverse: how IP reputation databases categorize *targets*
- [Production Scraping Hygiene and Legal](../reverse-engineering/14-production-scraping-hygiene-and-legal.md) — politeness layer over the rotation layer

## References

- [DOJ — 911 S5 Botnet Dismantled (May 2024)](https://www.justice.gov/opa/pr/911-s5-botnet-dismantled-and-its-administrator-arrested-coordinated-international-operation)
- [DOJ — RSOCKS Proxy Service Disrupted (June 2022)](https://www.justice.gov/opa/pr/justice-department-announces-court-authorized-disruption-rsocks-botnet)
- [Brian Krebs — Proxy Scammers (2023)](https://krebsonsecurity.com/2023/10/proxy-scammers-targeted-by-blockchain-fraud/)
- [MaxMind GeoIP2 Anonymous IP Database](https://www.maxmind.com/en/geoip2-anonymous-ip-database)
- [IPQualityScore Proxy/VPN Detection](https://www.ipqualityscore.com/documentation/proxy-detection-api/overview)
- [Bright Data — proxy-network documentation](https://brightdata.com/proxy-types)
- [Oxylabs — residential proxy documentation](https://oxylabs.io/products/residential-proxy-pool)
- [`scrapoxy`](https://github.com/fabienvauchelles/scrapoxy)
- [Apify — anti-blocking and proxy guide](https://docs.apify.com/academy/anti-scraping)
