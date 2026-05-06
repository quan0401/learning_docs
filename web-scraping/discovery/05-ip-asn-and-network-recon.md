---
title: "IP, ASN, and Network Reconnaissance"
date: 2026-05-06
updated: 2026-05-06
tags: [ip, asn, bgp, dns, geoip, network-recon, ip-reputation]
---

# IP, ASN, and Network Reconnaissance

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `ip` `asn` `bgp` `dns` `geoip` `network-recon` `ip-reputation`

---

## Table of Contents

1. [Why Network-Level Recon Matters](#1-why-network-level-recon-matters)
2. [IP → Hostname → ASN — The Resolution Chain](#2-ip--hostname--asn--the-resolution-chain)
3. [ASN Lookup and BGP Toolkits](#3-asn-lookup-and-bgp-toolkits)
4. [Reverse DNS and PTR Records](#4-reverse-dns-and-ptr-records)
5. [GeoIP and Its Accuracy Limits](#5-geoip-and-its-accuracy-limits)
6. [IP Reputation Feeds](#6-ip-reputation-feeds)
7. [CIDR Enumeration and ASN Sweeps](#7-cidr-enumeration-and-asn-sweeps)
8. [IPv6 — The Reconnaissance Geometry Changes](#8-ipv6--the-reconnaissance-geometry-changes)
9. [Practical Pipelines](#9-practical-pipelines)

## Summary

Hostname enumeration (doc 01) ends at A and AAAA records — IPs. Network-level recon takes those IPs and pivots to organizational ownership: which **autonomous system (AS)** owns the prefix, what **organization** announces it via BGP, what **other prefixes** that organization announces, and what other hosts live in the same network. The ASN-level pivot is the single best way to find an organization's full IP footprint, because IP allocations are publicly documented in the RIRs (ARIN, RIPE, APNIC, LACNIC, AFRINIC) and BGP announcements are observable globally. From there, GeoIP databases (MaxMind, ipinfo.io) and IP reputation feeds (Spamhaus, IPQS, AbuseIPDB) tell you whether an IP is residential, datacenter, hosting, or has historically generated abuse — which matters both for understanding a target and for choosing your own scraping egress (covered in doc 08). IPv6 inverts the address-space geometry: brute-force enumeration is impossible, so passive observation and DNS-driven discovery dominate.

---

## 1. Why Network-Level Recon Matters

When you've enumerated `app.example.com → 198.51.100.42`, you've found one host on one IP. But:

- The same `/24` may host the staging app, the admin tools, and an internal API — none of which appear in DNS.
- The organization's whole IP footprint may span multiple `/24` and `/22` allocations across two RIRs.
- The same ASN may announce production prefixes alongside corporate-office prefixes (where employee VPN endpoints live).
- For scrape egress (your own IPs, doc 08), the target's ASN-level fingerprinting is what flags datacenter ranges as suspicious.

Network recon answers: **what IPs does this organization own**, **what other organizations are in the same network neighborhood**, and **how is this IP categorized by reputation systems**.

---

## 2. IP → Hostname → ASN — The Resolution Chain

The chain:

```text
DNS A/AAAA: app.example.com  →  198.51.100.42
PTR (reverse DNS):           198.51.100.42  →  app-edge.cdn.example.com
WHOIS (RIR):                 198.51.100.0/22  →  Example Corp, allocated by ARIN, 2018-03-15
BGP origin AS:               198.51.100.0/22  →  AS64500 "EXAMPLE-CORP-AS"
Other prefixes from AS64500: 198.51.96.0/22, 192.0.2.0/24, 2001:db8::/32, ...
```

Each step is independent and pivot-friendly:

- WHOIS at the RIR tells you the legal owner of the prefix (or its delegate).
- BGP tells you the operational announcer (which may differ from the WHOIS owner — common with cloud customers).
- PTR records (when present) often reveal hosting topology.

---

## 3. ASN Lookup and BGP Toolkits

### 3.1 The Web UIs

- **[bgp.he.net](https://bgp.he.net/)** (Hurricane Electric) — the gold standard. Free, no auth, search by IP, AS number, AS name, prefix, or DNS name. Returns prefixes announced, peers, BGP route history, and country.
- **[RIPE Stat](https://stat.ripe.net/)** — RIPE's BGP/RIR/geo dashboard. Free, JSON API for everything visible in the UI.
- **[Hurricane Electric API](https://bgp.he.net/api)** (limited public API; for serious automation, use RIPE Stat)
- **[ipinfo.io](https://ipinfo.io/)** — friendly UI, IP→ASN and IP→org lookups, generous free tier with API
- **[bgpview.io](https://bgpview.io/)** — clean UI, free API
- **[asn.cymru.com](https://team-cymru.com/community-services/ip-asn-mapping/)** — Team Cymru's IP-to-ASN mapping; historical industry standard, DNS-based bulk lookup

### 3.2 Command-Line

```bash
# Team Cymru DNS-based bulk IP→ASN
echo "198.51.100.42" | awk '{ split($1,a,"."); print a[4]"."a[3]"."a[2]"."a[1]".origin.asn.cymru.com TXT" }' \
  | xargs -I{} dig +short TXT {}

# whois with RIR routing
whois -h whois.cymru.com " -v 198.51.100.42"

# RIPE Stat API: which prefixes does this AS announce?
curl -s "https://stat.ripe.net/data/announced-prefixes/data.json?resource=AS64500" \
  | jq '.data.prefixes[].prefix'

# RIPE Stat: who is the holder of this prefix?
curl -s "https://stat.ripe.net/data/whois/data.json?resource=198.51.100.0/22" \
  | jq '.data.records'
```

### 3.3 Looking Glasses and Route Servers

For raw BGP visibility:

- [RouteViews](http://www.routeviews.org/routeviews/) — University of Oregon project; archives BGP from many vantage points
- [RIPE RIS](https://www.ripe.net/analyse/internet-measurements/routing-information-service-ris) — RIPE's equivalent
- Public **looking glasses** (HE.net, NTT, etc.) — interactively run `show ip bgp` from the operator's perspective

These are needed when you want to know *who* is announcing a prefix from *where* — useful for detecting BGP hijacks or spotting operational complexity.

---

## 4. Reverse DNS and PTR Records

PTR (`in-addr.arpa` for IPv4, `ip6.arpa` for IPv6) reverse records map IPs back to hostnames. They are owned by whoever controls the IP allocation, so they often disclose the *real* hostname even when forward DNS hides it behind a CDN.

### 4.1 Bulk PTR Sweeps

```bash
# Bulk PTR for a /24 (256 hosts)
for ip in $(seq 0 255); do
  host 198.51.100.$ip 2>/dev/null \
    | awk '/pointer/ {print $1, $5}'
done

# With dnsx
echo 198.51.100.0/24 | dnsx -ptr -resp
```

### 4.2 What PTR Tells You

- **Hosting provider naming** — `ec2-198-51-100-42.compute-1.amazonaws.com` reveals AWS region
- **Hostname conventions** — `db-prod-01.internal.example.com` (sometimes leaked through PTR)
- **CDN POPs** — `xx-yyy-1.global.cdnprovider.net`
- **Mail server topology** — `mail-out-01.example.com`

### 4.3 PTR Caveat

PTR records require the IP owner to maintain them. Most production cloud workloads have generic provider PTRs; only well-managed self-hosted environments have meaningful PTRs. So PTR is high-signal when present, low-coverage overall.

---

## 5. GeoIP and Its Accuracy Limits

### 5.1 The Databases

- **[MaxMind GeoIP2](https://www.maxmind.com/)** — commercial standard; GeoLite2 free tier. Country/city/ASN/ISP databases.
- **[IP2Location](https://www.ip2location.com/)** — competitor, similar coverage
- **[ipinfo.io](https://ipinfo.io/)** — API-first, free tier
- **[DB-IP](https://db-ip.com/)** — free downloadable databases monthly
- **[GeoIPLookup.net / iplocation.net](https://www.iplocation.net/)** — multi-source comparison
- **[free.ipgeolocation.io](https://ipgeolocation.io/)** — free API tier

### 5.2 Accuracy Reality

GeoIP at the country level is highly accurate (>99%). At the city level, it falls off sharply:

- **Country**: typically >99% accurate
- **Region/state**: 80–90%
- **City**: 50–80% — frequently the ISP's headquarters or a major POP, not the user
- **Coordinates**: a centroid; never trust to street level

The *biggest* error mode: VPN exit nodes, datacenter ranges, and mobile-carrier NAT pools. A mobile carrier's NAT IP may serve users across an entire country.

### 5.3 What GeoIP Is Useful For

- Routing scrape egress through proxies in target regions (doc 08)
- Detecting "is this likely a residential or datacenter IP" — most providers tag this
- Geolocating CDN POPs vs origin servers
- Coarse compliance signals (GDPR-applicable region inference)

It is *not* useful for legal personal-data attribution, individual user identification, or address-level localization.

---

## 6. IP Reputation Feeds

Various services classify IPs by abuse history, residential vs. datacenter, VPN/proxy status, and threat-actor association.

| Feed | Focus |
|------|-------|
| [Spamhaus](https://www.spamhaus.org/) | DROP/EDROP lists, SBL, XBL — known mail-abuse and infrastructure |
| [AbuseIPDB](https://www.abuseipdb.com/) | Crowdsourced abuse reports, free/paid API |
| [IPQualityScore](https://www.ipqualityscore.com/) | Fraud-score classification — bot, VPN, proxy, fraud risk |
| [IPinfo Privacy Detection](https://ipinfo.io/products/privacy-vpn-detection-api) | Residential vs hosting vs VPN classification |
| [Project Honey Pot](https://www.projecthoneypot.org/) | Comment-spam and harvester IPs |
| [Cisco Talos Reputation](https://talosintelligence.com/) | Reputation by IP and domain |
| [GreyNoise](https://www.greynoise.io/) | "Is this IP scanning the entire internet" — separates targeted from background noise |
| [Shodan](https://www.shodan.io/) | (Doc 02) — banner and vuln data per IP |

### 6.1 GreyNoise — Why It Earns Its Own Mention

[GreyNoise](https://www.greynoise.io/) runs distributed honeypots and classifies any IP scanning the internet broadly as "noise." When you see an unfamiliar IP probing your server, GreyNoise tells you whether it's a known mass-scanner (Shodan, Censys, security researcher) or a targeted one. For the recon side, GreyNoise also maps which IPs are scanning, which is useful intel about active threat-actor infrastructure.

### 6.2 Reputation as a Scrape-Side Signal

When choosing scrape egress (doc 08), high-reputation residential proxies sometimes have abuse history from previous tenants — querying IPQualityScore or Spamhaus on your own egress before scraping a sensitive target helps avoid landing on a pre-blocked IP.

---

## 7. CIDR Enumeration and ASN Sweeps

Once you know `Example Corp` announces `198.51.100.0/22, 198.51.96.0/22, 192.0.2.0/24`, enumerate every IP in those ranges:

### 7.1 Enumerating CIDR Blocks

```bash
# Get prefixes for an AS, expand to host IPs
curl -s "https://stat.ripe.net/data/announced-prefixes/data.json?resource=AS64500" \
  | jq -r '.data.prefixes[].prefix' \
  | while read prefix; do
      nmap -sL -n "$prefix" \
        | awk '/Nmap scan report/ {print $5}'
    done > target_ips.txt
```

### 7.2 Reverse-DNS Sweep on Each Range

```bash
cat target_ips.txt | dnsx -silent -ptr -resp > ptr_results.txt
```

### 7.3 Service Discovery

```bash
# Probe HTTP/HTTPS on ranges
cat target_ips.txt | httpx -silent -ports 80,443,8080,8443 -title -tech-detect -json
```

This is exactly the "expand from a known host to the full IP footprint" pattern. For bug-bounty programs that scope `*.example.com` *and* "any host on AS64500," it dramatically expands the surface.

### 7.4 ASN Sweep Caveats

- Many cloud customers don't have their own ASN; they're on AWS/GCP/Azure ASNs alongside everyone else. ASN sweeping AS16509 (AWS) is useless.
- For self-announcing organizations, ASN sweep finds the corporate-IT footprint (office IPs, VPN endpoints) alongside production.
- Consider authorization carefully — even within an authorized scope, sweeping every IP can trip detection systems and cause real noise.

---

## 8. IPv6 — The Reconnaissance Geometry Changes

IPv4 has 4 billion addresses; sweeping a `/24` is 256 hosts. IPv6 has 340 undecillion addresses; sweeping a `/64` (the standard host allocation) is 2^64 hosts — impossible by brute force.

The IPv6 recon model is therefore **passive-only**:

### 8.1 Sources of IPv6 Hosts

- **DNS AAAA records** — enumerate via the same techniques as IPv4 (doc 01)
- **Certificate Transparency** (doc 01) — lists IPv6 SAN entries for hosts that publish them
- **Passive DNS feeds** (doc 01) — capture observed AAAA queries
- **Mailing list archives, public repositories, log files** that leaked addresses
- **EUI-64 patterns** — older IPv6 stateless auto-config derived host bits from MAC; if you know an organization's MAC OUI, you can sometimes guess subnet host bits
- **Common low-bit patterns** — `::1`, `::2`, `::abcd`, vanity addresses (`::face:b00c`)

### 8.2 What's Different

- No equivalent of `nmap -sP 198.51.100.0/24`
- Passive observation dominates
- Many CDN/cloud workloads are IPv6-dual-stacked; you can find the IPv6 from DNS and reach it directly
- Some bot-detection differs by IP family — IPv6 from a residential ISP vs IPv6 from a VPS provider have different reputation signals

---

## 9. Practical Pipelines

### 9.1 From One IP to a Full ASN Footprint

```bash
TARGET_IP=198.51.100.42

# 1. Resolve to ASN
ASN=$(curl -s "https://ipinfo.io/$TARGET_IP/json" | jq -r '.org' | awk '{print $1}')
# e.g., AS64500

# 2. All prefixes announced by that AS
curl -s "https://stat.ripe.net/data/announced-prefixes/data.json?resource=$ASN" \
  | jq -r '.data.prefixes[].prefix' > prefixes.txt

# 3. Expand to host IPs (small ranges only — never sweep AWS/GCP)
cat prefixes.txt | while read p; do
  size=$(echo $p | awk -F/ '{print $2}')
  if [ "$size" -ge 22 ]; then  # arbitrary cutoff
    nmap -sL -n "$p" | awk '/Nmap scan report/ {print $5}'
  fi
done > all_hosts.txt

# 4. Probe interesting ports
cat all_hosts.txt | httpx -silent -ports 80,443,8080,8443,3000,8000 -title -tech-detect -json > probed.jsonl
```

### 9.2 Categorizing Your Own Scrape Egress

```bash
MY_IP=$(curl -s ifconfig.me)
curl -s "https://ipinfo.io/$MY_IP/json"
# { "ip": "...", "org": "AS... Cloud Provider", "country": "...", ... }
# Now you know what the target sees — datacenter? residential? what country?
```

### 9.3 Detecting CDN-Origin Mismatch

The advertised IP for `www.example.com` is the CDN; the *origin* IP (where the application actually lives) is hidden behind the CDN. Several techniques recover origins:

- Search Censys for the SSL cert's SAN list — find any host with the same cert *not* on the CDN range (often staging or a subdomain forgot to put behind CDN)
- Check historical DNS (SecurityTrails) for periods before CDN was deployed
- Look for SSRF/info-leak vulnerabilities that disclose backend IPs (out of scope for passive recon)
- Check email server IPs — outbound mail often comes directly from the origin

---

## Related

- [Subdomain & Asset Enumeration](01-subdomain-and-asset-enumeration.md) — feeds IPs into this layer
- [Internet Asset Scanners](02-internet-asset-scanners.md) — banner data per IP; complements ASN-level enumeration
- [Proxies and IP Rotation](../extraction/08-proxies-and-ip-rotation.md) — flips the picture: how *your* egress IP is categorized by targets
- [Bot Detection Internals](../extraction/09-bot-detection-internals.md) — ASN reputation is a primary anti-bot signal
- [DNS in the Networking learning path](../../networking/INDEX.md) — protocol foundation for everything above

## References

- [Hurricane Electric BGP Toolkit](https://bgp.he.net/) — free interactive ASN/prefix explorer
- [RIPE Stat API](https://stat.ripe.net/) — JSON-API access to RIR + BGP data
- [Team Cymru IP→ASN Mapping Service](https://team-cymru.com/community-services/ip-asn-mapping/)
- [MaxMind GeoIP2 documentation](https://dev.maxmind.com/geoip)
- [Spamhaus Block Lists](https://www.spamhaus.org/zen/)
- [AbuseIPDB API](https://docs.abuseipdb.com/)
- [GreyNoise Visualizer](https://viz.greynoise.io/)
- [RFC 7404 — IPv6 Provider Independent Allocations](https://datatracker.ietf.org/doc/html/rfc7404)
- [RFC 4193 — Unique Local IPv6 Unicast Addresses](https://datatracker.ietf.org/doc/html/rfc4193)
- [RFC 4941 / RFC 8981 — IPv6 Privacy Extensions](https://datatracker.ietf.org/doc/html/rfc8981) — why SLAAC EUI-64 inference often fails today
- [Caida — BGP and Routing Data](https://www.caida.org/data/) — research-grade datasets
