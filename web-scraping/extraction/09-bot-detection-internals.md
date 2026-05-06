---
title: "Bot Detection Internals — Cloudflare, Akamai, JA3/JA4"
date: 2026-05-06
updated: 2026-05-06
tags: [bot-detection, cloudflare, akamai, datadome, ja3, ja4, fingerprinting, perimeterx]
---

# Bot Detection Internals — Cloudflare, Akamai, JA3/JA4

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `bot-detection` `cloudflare` `akamai` `datadome` `ja3` `ja4` `fingerprinting` `perimeterx`

---

## Table of Contents

1. The detection stack — three layers
2. Cloudflare Bot Management
3. Akamai Bot Manager
4. DataDome
5. PerimeterX / HUMAN Bot Defender
6. Kasada
7. JA3 — TLS ClientHello fingerprinting
8. JA4 — the modern successor
9. JA3S and server-side TLS fingerprints
10. HTTP/2 fingerprinting
11. Browser fingerprinting layers
12. Behavioral signals
13. Tools and libraries
14. Practical implications per scraper type
15. Ethics, law, and the line you do not cross
16. Related
17. References

---

## Summary

Modern bot detection is not a single check. It is a layered defense that inspects your IP and ASN, the bytes of your TLS ClientHello, the shape of your HTTP/2 frames, the JavaScript-derived fingerprint of your browser, and the rhythm of your mouse and keyboard. Cloudflare, Akamai, DataDome, HUMAN (PerimeterX), and Kasada each layer their own probes on top of these primitives — proprietary cookies, sensor payloads, and JS challenges that re-evaluate you continuously. Knowing how each layer constructs its signal is the first step to understanding why a `requests`-based scraper fails on a site a real Chrome can browse, and is also the foundation any defender needs to write correct rules. This doc explains the mechanics; evasion is treated as the natural consequence with explicit ethical and legal call-outs at the end.

---

## 1. The detection stack — three layers

Every commercial bot management product organizes its checks into roughly three layers. Different vendors use different names but the structure is consistent.

| Layer | Where it lives | What it inspects | Cost to fake |
|-------|---------------|------------------|--------------|
| Network / IP | Edge L3-L4 | Source IP, ASN, GeoIP, IP reputation, prior abuse history | Low (rotate proxies) but residential IPs are pricey |
| Transport / TLS | Edge L5 | ClientHello bytes (JA3/JA4), HTTP/2 SETTINGS, header order | Medium — needs `curl-impersonate` / `utls` |
| Application / JS+behavior | Origin or edge worker | Cookies, JS-collected fingerprint payloads, mouse/keyboard, sensor data | High — needs a real browser or full sensor reverse-engineering |

A request must pass all three layers cleanly. A datacenter IP plus a perfect Chrome JA4 still trips IP reputation. A residential IP plus Python `requests` trips TLS. A residential IP plus a headless Chrome with `webdriver` flag set trips JS fingerprinting.

The vendors below each implement these three layers. The rest of the doc walks through each vendor and then deconstructs the primitives.

---

## 2. Cloudflare Bot Management

Cloudflare sits in front of an estimated 20%+ of the public web, so understanding its bot stack pays off broadly.

### 2.1 The bot score

Cloudflare scores every request on `1` to `99`:

- `1`-`29`: almost certainly automated.
- `30`-`99`: more human-like, with `99` reserved for verified humans (e.g., Cloudflare Access).

The score is exposed to the origin via the `cf-bot-score` request header and to Workers via `request.cf.botManagement.score`. Site operators write firewall rules like "challenge if score < 30" rather than binary block/allow.

### 2.2 The `__cf_bm` cookie

When a request first reaches a protected zone, Cloudflare issues a `__cf_bm` cookie. It is an opaque, signed token tied to the visitor's TLS fingerprint, IP, and a short timestamp. The cookie is valid for 30 minutes and is rotated on roughly that cadence. Stripping or forging it is detected immediately because the signature must validate against Cloudflare's edge secret.

### 2.3 JS challenge ("I'm Under Attack" mode)

When suspicion rises Cloudflare returns a `503` with a JS challenge — a small obfuscated script that performs a proof-of-work calculation, collects a fingerprint payload, and posts it back to `/cdn-cgi/challenge-platform/...`. On success the response sets `cf_clearance` (a longer-lived clearance cookie) and the original request is replayed.

### 2.4 Turnstile

Turnstile is Cloudflare's reCAPTCHA replacement. It is a private-attestation widget that runs the same kind of fingerprinting as the JS challenge but rolled into a reusable iframe. Turnstile tokens carry a server-verifiable signature; the site verifies via `/turnstile/v0/siteverify`.

### 2.5 Pages "Bot Fight Mode"

A simpler tier available to free Cloudflare Pages users. It does not score; it just challenges anything with a known-bad UA, a known-bad ASN (datacenter ranges), or no `__cf_bm` cookie. Easier to bypass than full Bot Management but still defeats trivially-naive scrapers.

---

## 3. Akamai Bot Manager

Akamai's bot product (now branded "BMP" — Bot Manager Premier) is the longest-tenured enterprise solution and is heavily used by airlines, banks, and ticketing.

### 3.1 The `_abck` cookie

The single most-recognizable Akamai artifact. Issued on the first response, it is opaque, signed, and contains a state machine. The cookie has well-known internal markers:

- `~0~` — sensor payload not yet collected; the next request must include one.
- `~-1~` — sensor accepted, session validated.
- `~-1~-1~` — invalidated; the visitor was reclassified as a bot.

If you scrape with `requests` and just paste a `_abck` from a real browser, the very next request will flip the cookie to `~-1~-1~` because Akamai sees a JA3/HTTP2 mismatch.

### 3.2 The sensor data payload

Akamai requires the client to `POST` an encrypted sensor payload to a per-site endpoint (often `/`-rooted under the site's path) within a few hundred ms of page load. The payload is constructed by an obfuscated JavaScript bundle ("the sensor") and contains:

- Browser fingerprint (canvas, WebGL, audio, fonts, navigator attributes).
- Mouse-move histograms, scroll deltas, focus events.
- Page-load timing (DNS, TLS, TTFB, FCP, LCP).
- A challenge-response derived from a token in the page HTML.

The sensor JS is rotated frequently — the obfuscation pattern changes monthly. Reverse-engineering it is a full-time job for several scraping vendors.

### 3.3 BMP integration

BMP wires the `_abck` and sensor scoring into customer-defined enforcement: outright block, queue (e.g., for ticketing), serve different prices, or transparently capture for analytics.

---

## 4. DataDome

DataDome sells itself on speed (sub-2ms scoring at the edge) and on producing low false-positive rates. Two visible artifacts:

### 4.1 The `datadome` cookie

A signed token issued on first contact. Its presence and validity is checked on every subsequent request. Like Cloudflare's `__cf_bm`, it is bound to the original TLS fingerprint and IP class.

### 4.2 The DataDome JS challenge

When risk is high, DataDome returns a `403` with a `text/html` body that contains the DataDome captcha widget. The widget runs DataDome's collection script, then either passes the visitor automatically (silent challenge) or escalates to a hCaptcha-style image puzzle. On success the site sets a fresh `datadome` cookie and the user retries the original URL.

DataDome's silent path is harder to spot than Cloudflare's JS challenge because there is no obvious interstitial — the cookie just gets re-issued.

---

## 5. PerimeterX / HUMAN Bot Defender

PerimeterX rebranded to HUMAN Bot Defender after the HUMAN Security acquisition. It is heavy on collection — a single page often loads 200-500KB of obfuscated PX sensor JS.

### 5.1 The `_px*` cookie family

PX uses a chain of cookies, each with a different role:

| Cookie | Purpose |
|--------|---------|
| `_px2` | Short-lived primary token bound to the current request's risk score |
| `_px3` | Long-lived "session" token |
| `_pxhd` | Hardened token used in cross-domain validation |
| `_pxvid` | Visitor ID, persists across sessions |
| `_pxff_*` | Feature flags / A/B markers from the PX backend |

The full chain must be present and consistent. A scraper that scrapes only `_px3` is detected immediately.

### 5.2 The sensor

PX collects everything the others collect — canvas, WebGL, audio, navigator entropy, mouse/keyboard, accelerometer on mobile — and posts it to `https://collector-{customer}.perimeterx.net`. The collection is rotated and checksummed; the script will refuse to run if it detects modification or if `Function.prototype.toString` has been hooked.

---

## 6. Kasada

Kasada's pitch is that classical fingerprinting is losing because it is reverse-engineerable; instead Kasada uses a heavy proof-of-work + cryptographic puzzle approach, executed in a dedicated worker.

### 6.1 KP challenge tokens

Kasada-protected sites return a `429` with the body containing the `kpsdk` JS bootstrap. The bootstrap fetches a binary WebAssembly puzzle, solves it (computationally expensive — designed to cost a bot farm dollars per page), and returns a `KP` challenge token in subsequent requests. Tokens are short-lived and cannot be replayed across sessions.

This is architecturally different: less about *who you are* and more about *did you pay the CPU bill* to be here. It does not eliminate fingerprinting but reduces its weight.

---

## 7. JA3 — TLS ClientHello fingerprinting

JA3 was published by Salesforce in 2017 (Althouse, Atkinson, Atkins). It captures the client side of the TLS handshake before any application data flows.

### 7.1 What JA3 hashes

Every TLS ClientHello carries:

- TLS version offered.
- Ordered list of cipher suites.
- Ordered list of extensions.
- Elliptic curves (supported groups).
- Elliptic curve point formats.

JA3 concatenates these as comma-separated decimal numbers, dash-joined within each list:

```
TLSVersion,Ciphers,Extensions,EllipticCurves,EllipticCurvePointFormats
```

Then takes the MD5. Example:

```
769,47-53-5-10-49161-49162-49171-49172-50-56-19-4,0-10-11-13-15,23-24-25,0
```

MD5 of that string is the JA3 fingerprint. The MD5 is just a 32-char identifier; the underlying string is what matters.

### 7.2 Why scrapers get caught

`python-requests` linked against OpenSSL produces a JA3 different from Chrome's BoringSSL build. The ordering of extensions and ciphers is different. Detection vendors maintain a denylist of known-automation JA3 hashes (Python, Go's `crypto/tls`, Node's TLS, curl). One unfamiliar JA3 from a residential IP is suspicious; a known-bad JA3 from a datacenter IP is auto-blocked.

### 7.3 Tooling

`go-fingerprint` and similar libraries compute JA3 from a captured handshake. Many sites compute it inline at the edge from the TLS record.

---

## 8. JA4 — the modern successor

JA4 was published by FoxIO in 2024 and is now widely adopted (released under a permissive but trademark-protected license). It addresses three weaknesses of JA3:

1. JA3 changes when Chrome reorders extensions (which Chrome started doing for ECH/GREASE).
2. JA3 collapses too aggressively under TLS 1.3.
3. The MD5 is opaque — operators cannot read it.

### 8.1 Structure

JA4 produces a *human-readable* fingerprint with three parts joined by underscores:

```
t13d1516h2_8daaf6152771_b186095e22b6
```

- `t13d1516h2` — protocol prefix (TLS 1.3, with SNI, 15 ciphers, 16 extensions, ALPN h2).
- `8daaf6152771` — truncated SHA-256 of the sorted cipher list.
- `b186095e22b6` — truncated SHA-256 of the sorted extension + signature-algorithm list.

The protocol prefix is human-inspectable, so an operator can see at a glance "TLS 1.3 + h2 + SNI" without decoding a hash. Sorting before hashing makes the fingerprint stable against benign reordering.

### 8.2 The JA4+ family

| Variant | What it fingerprints |
|---------|----------------------|
| JA4 | TLS ClientHello (the core fingerprint) |
| JA4S | TLS ServerHello (server-side fingerprint) |
| JA4H | HTTP request — method, version, headers, cookies present, language |
| JA4X | X.509 certificate (issuer + subject + extensions) |
| JA4T | TCP fingerprint (window size, MSS, options ordering) |
| JA4L | Latency-pair fingerprint |

JA4H is particularly powerful for HTTP/1.1: it captures which headers the client sent, in what order, casing, and whether `Cookie` was present. A `requests`-default `User-Agent`, `Accept-Encoding`, `Accept`, `Connection` block looks nothing like Chrome's.

---

## 9. JA3S and server-side TLS fingerprints

JA3S is the server-side counterpart of JA3 — the ServerHello has its own fingerprint constructed from the chosen TLS version, chosen cipher, and extensions returned. Its applications are different:

- A site computes JA3S of *its own* server to detect MITM-style proxies that re-terminate TLS upstream.
- Some corporate networks run inspection proxies that strip JA3 entropy by re-terminating TLS — those proxies have their own JA3S that can be detected and force-redirected.

JA3S is less commonly used in commercial bot management than JA3/JA4, but it appears in custom rules at financial and government sites.

---

## 10. HTTP/2 fingerprinting

Akamai's Roy Shuster presented "Passive Fingerprinting of HTTP/2 Clients" at BlackHat EU 2017, and the technique is now standard. HTTP/2 has many more degrees of freedom than HTTP/1.1, and clients leak identity through every one.

### 10.1 What is fingerprinted

| Frame / field | Variation |
|---------------|-----------|
| `SETTINGS` frame | Order and values of `HEADER_TABLE_SIZE`, `ENABLE_PUSH`, `MAX_CONCURRENT_STREAMS`, `INITIAL_WINDOW_SIZE`, `MAX_FRAME_SIZE`, `MAX_HEADER_LIST_SIZE` |
| `WINDOW_UPDATE` | Size of the connection-level window increment |
| `PRIORITY` | Stream priority dependencies (unique per client) |
| Pseudo-header order | Chrome sends `:method, :authority, :scheme, :path`; Firefox sends `:method, :path, :authority, :scheme`; Go's net/http sends a different order |
| Header capitalization | HTTP/2 forbids uppercase; some scrapers still send `User-Agent` cased — illegal but visible |

### 10.2 Akamai's notation

Akamai canonicalized this as a four-part string:

```
S[settings]|WU[window_update]|P[priority]|HF[header_flags]
```

For example a real Chrome H2 fingerprint looks like:

```
1:65536;3:1000;4:6291456;6:262144|15663105|0|m,a,s,p
```

Send the same TLS as Chrome but the wrong H2 fingerprint and you light up Akamai's bot signal.

---

## 11. Browser fingerprinting layers

When the request reaches application code, JavaScript can collect signals invisible at network level. These are the workhorses of HUMAN, DataDome, and Akamai's sensor.

### 11.1 Canvas fingerprinting

Mowery and Shacham (USENIX 2012, "Pixel Perfect: Fingerprinting Canvas in HTML5") showed that drawing the same text + emoji + colored shapes on `<canvas>` produces pixel output that varies by GPU, driver, OS font rendering, and antialiasing. The script reads the bytes via `canvas.toDataURL()` or `getImageData()`, hashes them, and the resulting hash is stable per machine and high-entropy across machines.

```javascript
const canvas = document.createElement('canvas');
const ctx = canvas.getContext('2d');
ctx.textBaseline = 'top';
ctx.font = '14px Arial';
ctx.fillStyle = '#f60';
ctx.fillRect(125, 1, 62, 20);
ctx.fillStyle = '#069';
ctx.fillText('BotDetect-test 🛡️', 2, 15);
const fingerprint = canvas.toDataURL();
```

### 11.2 WebGL

```javascript
const gl = canvas.getContext('webgl');
const dbg = gl.getExtension('WEBGL_debug_renderer_info');
const vendor = gl.getParameter(dbg.UNMASKED_VENDOR_WEBGL);
const renderer = gl.getParameter(dbg.UNMASKED_RENDERER_WEBGL);
```

Returns strings like `"Google Inc. (Apple)"` and `"ANGLE (Apple, Apple M2, OpenGL 4.1)"`. Headless Chrome on Linux returns `"SwiftShader"` — instantly identifiable.

### 11.3 AudioContext fingerprinting

`OfflineAudioContext` running an oscillator through a compressor produces FFT output that varies by audio stack. The summed magnitude is hashed; same logic as canvas.

### 11.4 Font enumeration

Render a set of strings in a known fallback font and a target font; compare bounding-box widths via `measureText`. Fonts present on the system render differently from the fallback. Yields a list of installed fonts without any permissions prompt.

### 11.5 Navigator entropy

```javascript
navigator.userAgent
navigator.platform
navigator.languages          // ordered array
navigator.hardwareConcurrency
navigator.deviceMemory
navigator.plugins.length
navigator.mimeTypes.length
navigator.webdriver          // true under raw Selenium/Puppeteer
```

`navigator.webdriver === true` is the single most-checked signal in the world. Headless detection heuristics also look for `window.chrome.runtime` existence, `Notification.permission === 'denied'` while `Permission.query` says `default`, and other inconsistencies between objects that should agree.

### 11.6 WebRTC IP leak

Even with a proxy, WebRTC's `RTCPeerConnection` ICE candidates leak the local LAN IP and the public IP from STUN. A request whose claimed Geo IP is in São Paulo but whose WebRTC public IP is a DigitalOcean datacenter is obvious.

---

## 12. Behavioral signals

When the page is loaded in a real browser, sensors record:

- Mouse path: bezier-like with jitter, varying speed, occasional reverses. Bots often produce ruler-straight lines or perfectly identical bezier curves.
- Click timing: humans pause 200-800ms before first click; bots click sub-50ms.
- Scroll cadence: humans scroll in bursts of 3-7 lines with 1-3 second pauses. Bots scroll smoothly to the bottom in 200ms.
- Time-on-page: anything under 800ms with a form submission is a strong bot signal.
- Keyboard cadence: real typing has variance; bots paste or emit at uniform intervals.

Modern detectors do not just check whether mouse moves exist — they verify the *distribution* matches a human profile (the FingerprintJS team has published several research articles on this).

---

## 13. Tools and libraries

These exist to defeat the network and TLS layers. None of them defeats JS fingerprinting — that requires a real browser.

| Tool | Language | What it does |
|------|----------|--------------|
| `curl-impersonate` (lwthiker) | C | Patched libcurl + nss/boringssl that produces Chrome / Firefox / Edge / Safari TLS + H2 fingerprints byte-for-byte. The reference baseline. |
| `utls` (refraction-networking) | Go | Drop-in replacement for `crypto/tls` that lets you choose `HelloChrome_120`, `HelloFirefox_120`, etc., and renders the matching ClientHello. Used inside many Go-based scrapers. |
| `tls-client` (bogdanfinn) | Go core, Python/Node bindings | Wraps `utls` and adds H2 frame matching. The Python and Node packages let JS-ecosystem scrapers get Chrome-grade TLS without writing Go. |
| `go-fingerprint` | Go | Computes JA3/JA4 from captured handshakes; useful for testing. |
| FingerprintJS open-source | TypeScript | The browser-side fingerprinting library used by FingerprintJS Pro. The OSS version is a great reference for *what gets collected*. |
| `playwright-extra` + `puppeteer-extra-plugin-stealth` | Node | Patches a real headed Chrome to remove `webdriver`, fix `navigator.plugins`, fix WebGL UNMASKED_VENDOR — addresses JS fingerprint *symptoms*. |

A stack that defeats both transport and JS layers usually looks like: residential proxy → `tls-client` for non-rendering API calls → Playwright with stealth for rendering paths.

---

## 14. Practical implications per scraper type

| Scraper | Where it fails |
|---------|----------------|
| `python-requests` from datacenter IP | IP layer (datacenter ASN) + JA3 (Python's TLS) + H2 (or no H2 at all) |
| `python-requests` from residential proxy | JA3 + H2; passes IP |
| `requests` + manual headers from residential | Still fails JA3; JA4H now fails too because header order is wrong |
| `tls-client` + residential | Passes IP, TLS, H2. Fails sensor / cookie if site has Cloudflare BM, Akamai, DataDome, PX |
| Headless Chrome (Puppeteer) | Often fails: `navigator.webdriver`, WebGL `SwiftShader`, missing `chrome` object |
| Playwright + stealth | Closer; still detectable on Akamai/HUMAN through behavioral and timing channels |
| Real Chrome controlled via CDP from residential | Hardest to detect; can still be caught by behavioral models if mouse/keyboard are scripted |

The general lesson: each vendor stacks orthogonal checks deliberately so that no single tool wins everywhere.

---

## 15. Ethics, law, and the line you do not cross

The same techniques that let a security team spot account-takeover bots also let a scraper evade detection. The mechanics are symmetric; the legal exposure is not.

### 15.1 Authorized vs unauthorized

| Activity | Generally OK |
|----------|--------------|
| Scraping your own site | Yes |
| Authorized pentest with written scope | Yes — and recommended; defenders should test their own stack |
| Scraping a public site whose ToS you respect, at a low rate, on data made publicly available, for a research/journalism purpose | Often defensible; HiQ v. LinkedIn (9th Cir.) supports public-data scraping but it is fact-specific |
| Scraping a public site by *circumventing* technical access controls (CAPTCHA solving services, stolen cookies, brute force) | Risky, especially in the US |
| Account-takeover, credential stuffing, sneaker-bot purchasing in violation of ToS | Not OK |
| Reselling scraped content as if you produced it | Often a copyright issue independent of CFAA |

### 15.2 CFAA exposure

The US Computer Fraud and Abuse Act (18 U.S.C. § 1030) criminalizes accessing a computer "without authorization" or "exceeding authorized access". *Van Buren v. United States* (2021) narrowed "exceeds" to gates-up vs gates-down access. *HiQ Labs v. LinkedIn* (9th Cir., on remand 2022) clarified that scraping *publicly accessible* data is generally not unauthorized access, but *circumventing technical access controls* (like an IP block specifically targeting you) is contested ground. Authorities are jurisdiction-specific; in the EU the Database Directive and GDPR add overlapping constraints.

If you are reading this doc to defend a site, none of the above is a problem. If you are reading to scrape, the rule of thumb is: **scraping public data politely and slowly with no credentials and no circumvention of technical countermeasures is the safest end of the spectrum; anything that involves stolen sessions, CAPTCHA-solving services for accounts you do not own, or credential reuse is the unsafe end.** Talk to a lawyer for anything in the middle.

### 15.3 Defender's symmetric obligation

If you operate one of these stacks, false positives are also harmful — you are denying legitimate users (assistive tech, researchers, low-end devices, Tor users). Tune for the actual threat model, log decisions, and provide an appeal path. The ethical defender posture mirrors the ethical scraper posture.

---

## 16. Related

- [HTTP Scraping Fundamentals](06-http-scraping-fundamentals.md)
- [Headless Browsers](07-headless-browsers.md)
- [Proxies and IP Rotation](08-proxies-and-ip-rotation.md)
- [CAPTCHA Systems and Solvers](10-captcha-systems-and-solvers.md)
- [Source Maps and Chunked Bundles](../reverse-engineering/12-source-maps-and-chunked-bundles.md)
- [TLS Handshake and Certificates](../../security/fundamentals/06-tls-handshake-and-certificates.md)

---

## 17. References

- Althouse, Atkinson, Atkins. "JA3 — A method for profiling SSL/TLS Clients." Salesforce Engineering (2017). https://github.com/salesforce/ja3
- FoxIO LLC. "JA4+ Network Fingerprinting." (2024). https://github.com/FoxIO-LLC/ja4
- Shuster, Roy. "Passive Fingerprinting of HTTP/2 Clients." Black Hat Europe 2017. https://www.blackhat.com/eu-17/briefings.html#passive-fingerprinting-of-http/2-clients
- lwthiker. `curl-impersonate` — curl with Chrome/Firefox/Edge/Safari TLS fingerprints. https://github.com/lwthiker/curl-impersonate
- Refraction Networking. `utls` — drop-in TLS fingerprint library for Go. https://github.com/refraction-networking/utls
- bogdanfinn. `tls-client` — TLS-fingerprinting HTTP client with Python and Node bindings. https://github.com/bogdanfinn/tls-client
- FingerprintJS. Open-source browser fingerprinting library. https://github.com/fingerprintjs/fingerprintjs
- Mowery, K., Shacham, H. "Pixel Perfect: Fingerprinting Canvas in HTML5." W2SP 2012. https://hovav.net/ucsd/dist/canvas.pdf
- Cloudflare. "Bot Management" product documentation. https://developers.cloudflare.com/bots/
- Cloudflare. "Turnstile" documentation. https://developers.cloudflare.com/turnstile/
- HUMAN Security (formerly PerimeterX). "Bot Defender" product page. https://www.humansecurity.com/products/bot-defender/
- Akamai. "Bot Manager" product documentation. https://www.akamai.com/products/bot-manager
- DataDome. "Bot Protection" documentation. https://datadome.co/products/bot-protection/
- Kasada. "Modern Bot Defense" documentation. https://www.kasada.io/platform/
- *HiQ Labs, Inc. v. LinkedIn Corp.*, 31 F.4th 1180 (9th Cir. 2022).
- *Van Buren v. United States*, 593 U.S. ___ (2021).
- 18 U.S.C. § 1030 (Computer Fraud and Abuse Act).
