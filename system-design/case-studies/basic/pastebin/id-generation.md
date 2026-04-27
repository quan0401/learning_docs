---
title: "Pastebin Deep Dive — ID Generation"
date: 2026-04-27
updated: 2026-04-27
tags: [system-design, case-study, pastebin, deep-dive, id-generation]
---

# Pastebin Deep Dive — ID Generation

**Date:** 2026-04-27 | **Updated:** 2026-04-27
**Tags:** `system-design` `case-study` `pastebin` `deep-dive` `id-generation`
**Companion to:** [`../design-pastebin.md`](../design-pastebin.md)

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [Pastebin-Specific Constraints](#pastebin-specific-constraints)
  - [The Paste is an Opaque Blob, Often Sensitive](#the-paste-is-an-opaque-blob-often-sensitive)
  - [Why Aliases Are Not the Norm](#why-aliases-are-not-the-norm)
  - [Longer Codes Are Tolerable](#longer-codes-are-tolerable)
- [Random Base62 vs UUID](#random-base62-vs-uuid)
  - [8 / 10 / 12-Character Base62 — When Each is Enough](#8--10--12-character-base62--when-each-is-enough)
  - [UUIDv4 as a URL — The Trade-offs](#uuidv4-as-a-url--the-trade-offs)
  - [UUIDv7 (RFC 9562) — Sortable but Long](#uuidv7-rfc-9562--sortable-but-long)
- [Content-Addressed IDs](#content-addressed-ids)
  - [The Dedup Win](#the-dedup-win)
  - [SHA-256 Truncation Math](#sha-256-truncation-math)
  - [The Cross-User Privacy Leak](#the-cross-user-privacy-leak)
- [Salting Per-User and Per-Session](#salting-per-user-and-per-session)
- [Burst-Friendly Allocation](#burst-friendly-allocation)
  - [Pre-Allocated ID Pools](#pre-allocated-id-pools)
  - [Sizing the Pool](#sizing-the-pool)
- [Visibility Tiers and ID Design](#visibility-tiers-and-id-design)
  - [Public, Unlisted, Private, Password-Protected](#public-unlisted-private-password-protected)
  - [The ID Must Not Reveal Visibility](#the-id-must-not-reveal-visibility)
  - [Pastebin.com vs GitHub Gist Conventions](#pastebincom-vs-github-gist-conventions)
- [Crawler Resistance](#crawler-resistance)
  - [robots.txt Defaults](#robotstxt-defaults)
  - [Per-Response X-Robots-Tag](#per-response-x-robots-tag)
  - [ID Sparsity as a Defense](#id-sparsity-as-a-defense)
- [Migration to Longer IDs](#migration-to-longer-ids)
- [Auditability — Embedding Tenant Identity](#auditability--embedding-tenant-identity)
- [Comparison Table — Pastebin.com vs GitHub Gist vs Hastebin](#comparison-table--pastebincom-vs-github-gist-vs-hastebin)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

The parent case study at [`../design-pastebin.md`](../design-pastebin.md) compares three paste-ID strategies (random base62, content hash, monotonic counter) in about thirty lines and lands on random base62 as the default. That conclusion mirrors the URL-shortener decision but the *reasons* are different in subtle, important ways: a paste is an **opaque blob** that often contains source code, secrets pasted by accident, or messages traded outside chat; an attacker enumerating the corpus is harvesting **content**, not just dereferencing redirects. This deep-dive focuses on what is **Pastebin-specific** about ID generation — content-addressed dedup and its privacy trap, visibility-tier design, crawler resistance, audit trails for enterprise tiers, and the burst dynamics of "someone shared a Pastebin link in a 50K-member chat."

For the **shared math** (birthday paradox, fill density, base62 encoding mechanics, Bloom-filter front-doors, range allocation), see the URL-shortener companion at [`../url-shortener/code-generation-strategies.md`](../url-shortener/code-generation-strategies.md). This doc deliberately does not repeat it.

## Overview

ID generation for a paste service answers four questions at once:

1. **Uniqueness** — no two pastes share an ID.
2. **Unguessability** — a third party cannot enumerate or guess paste IDs.
3. **Throughput** — minting an ID must not bottleneck `POST /paste`.
4. **Visibility-respecting** — the ID itself must not leak whether the paste is public, unlisted, private, or password-protected.

A URL shortener cares about (1) and (3) primarily; (2) is nice-to-have for tracking-link privacy. A Pastebin must hit all four — and the constraint that **the ID is the access token for unlisted pastes** means weakness here is a direct content-leak vector. A six-character code is fine for a marketing redirect; it is a privacy violation for a paste containing leaked credentials.

## Pastebin-Specific Constraints

### The Paste is an Opaque Blob, Often Sensitive

A URL shortener's payload is itself a public URL — you redirect to a destination the shortener does not own. A paste's payload is **content the service stores**, which can be:

- Source code with embedded secrets pasted by mistake.
- Server logs shared between engineers debugging an incident.
- Chat transcripts forwarded to a moderator.
- Build output containing internal hostnames and IPs.

Treat the corpus as **adversarially interesting**. The threat model includes search-engine crawlers, third-party indexers (Pastebin scrapers and credential-harvesting bots are a known industry), competitors, and attackers who want to enumerate any organization's "code I quickly threw onto a paste site." The ID is the **only access control** for unlisted pastes — make it count.

### Why Aliases Are Not the Norm

URL shorteners commonly let users pick a custom slug (`bit.ly/my-launch`). Pastebin services generally do not, for two reasons:

- **Memorability is not the use case.** Users send paste URLs over chat or email; nobody is typing `pastebin.com/my-snippet` from memory. Auto-generated codes work fine.
- **Aliases re-introduce enumeration risk.** A user who picks `prod-config-2026` reveals what's inside; an attacker scanning common slugs (`config`, `passwords`, `keys`, `dump`) hits gold occasionally.

GitHub Gist breaks this rule slightly — the URL embeds the GitHub username (`gist.github.com/<user>/<id>`) — but the random hex ID is still the unique identifier; the username is decoration. Pure paste services keep the namespace flat and machine-generated.

### Longer Codes Are Tolerable

URL shorteners obsess over short codes because the whole point is brevity. Pastebin codes can be 8, 10, even 12 characters and nobody complains — users copy/paste the URL; they never type it. **The freedom to use longer codes is the most important lever Pastebin has** for unguessability without exotic schemes. A 10-character base62 code (`62^10 ≈ 8.39 × 10^17`) gives 60 bits of entropy — comfortably beyond brute-force reach even at 1M probes/sec.

## Random Base62 vs UUID

### 8 / 10 / 12-Character Base62 — When Each is Enough

For the entropy and brute-force math, see the [URL-shortener entropy table](../url-shortener/code-generation-strategies.md#entropy-at-6--7--8-characters). For a paste service the practical bands are:

| Length | Code-space | Bits | Brute-force at 1M probes/sec | Recommended For |
|---|---|---|---|---|
| 8 | `62^8 ≈ 2.18 × 10^14` | 47.6 | ~7 years | Public-only services with aggressive 404 rate-limits |
| 10 | `62^10 ≈ 8.39 × 10^17` | 59.5 | ~26,000 years | Default for unlisted-by-default services |
| 12 | `62^12 ≈ 3.23 × 10^21` | 71.5 | ~10^11 years | Privacy-emphasizing, enterprise, secret-by-default |

**Pastebin.com uses 8-character keys.** They lean on paid-tier abuse controls and high-volume rate limiting, not on entropy alone. A new service with no abuse-detection investment should default to 10 characters and revisit later — the marginal cost (two extra URL chars) is invisible to users, and the unguessability gain is six orders of magnitude.

```python
"""Pastebin-grade random base62 ID generator.

Uses ``secrets`` (CSPRNG) — never ``random``, which is predictable.
"""
import secrets
import string

ALPHABET = string.ascii_letters + string.digits  # 62 chars: 0-9A-Za-z


def new_paste_id(length: int = 10) -> str:
    """Generate a cryptographically random base62 paste ID.

    10 chars is the recommended default: ~60 bits of entropy,
    and codes are still short enough to share comfortably.
    """
    return "".join(secrets.choice(ALPHABET) for _ in range(length))


def is_well_formed(code: str, min_len: int = 8, max_len: int = 12) -> bool:
    """Validate user-supplied codes before any DB lookup.

    Reject malformed codes at the edge — a 100-char input must
    not become a B-tree probe.
    """
    if not (min_len <= len(code) <= max_len):
        return False
    return all(c in ALPHABET for c in code)
```

The collision-handling pattern is unchanged from the URL-shortener case study: `INSERT … ON CONFLICT DO NOTHING` and retry on conflict. At 10 chars and 1B pastes, the retry rate is ~10⁻⁹ per insert — invisible.

### UUIDv4 as a URL — The Trade-offs

A UUIDv4 is 122 bits of entropy, rendered as `f47ac10b-58cc-4372-a567-0e02b2c3d479` (36 chars with hyphens, 32 without). Three reasons it's the wrong public ID for a paste service:

- **URL length.** 32 hex chars vs 10 base62 chars — three times longer for no useful gain.
- **No alias semantics.** UUIDs don't admit human-friendly customization (which most pasters don't want anyway, but it forecloses the option entirely).
- **No locality.** UUIDv4 indexes are random across the B-tree, hurting write throughput vs a more sortable scheme. (For a small `pastes` table this barely matters; for a 10B-row table it does.)

UUIDv4 makes sense for **internal record IDs** (the primary key of `paste_metadata` rows) where the ID never appears in a URL. Pair an internal UUIDv7 PK with a separately-generated 10-char base62 public code; you get sortable internal IDs and short public URLs.

### UUIDv7 (RFC 9562) — Sortable but Long

[RFC 9562 (May 2024)](https://www.rfc-editor.org/rfc/rfc9562) defines UUIDv7 as a time-ordered UUID: 48 bits of Unix-millisecond timestamp followed by 74 bits of randomness, with 4 version bits and 2 variant bits in between. Properties:

- **Monotonically sortable** by creation time — B-tree-friendly indexes, no hot-shard problem.
- **No coordinator** — generated independently per node.
- **128 bits is overkill** for a public paste code; even base62-encoded it's ~22 chars.

The right fit: **UUIDv7 for the internal primary key** (sorted, indexed, joined against), **random 10-char base62 for the public-facing paste ID**. The two are stored side-by-side; lookups by public ID hit a unique index that maps to the internal UUIDv7.

## Content-Addressed IDs

The seductive alternative: `paste_id = base62(sha256(content))[:10]`. Same content → same ID; **automatic deduplication**.

### The Dedup Win

Real workloads see substantial duplication:

- The same Stack Overflow answer pasted by hundreds of users debugging the same error.
- Boilerplate `package.json` snippets shared across tutorials.
- Build logs pasted by every developer hitting the same CI failure today.
- Common configs (nginx defaults, `.gitignore` templates).

A paste service that dedups by content hash can cut storage and CDN costs by 20–40% on a public-paste-heavy workload. That's not nothing. **But it is the wrong default**, for the privacy reason below.

### SHA-256 Truncation Math

The hashing arithmetic is the same as for URL shorteners — see [Birthday Paradox Math](../url-shortener/code-generation-strategies.md#birthday-paradox-math). Briefly: SHA-256 has 256 uniformly random output bits; truncating to 60 bits (10 base62 chars) is mathematically clean.

```python
"""SHA-256 truncation to a 10-char base62 paste ID.

Demonstrates the *naive* content-addressed approach. Read the
privacy section before deploying anything like this to production.
"""
import hashlib

ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
BASE = len(ALPHABET)


def encode_base62(n: int, min_length: int = 0) -> str:
    if n == 0:
        return ALPHABET[0].rjust(min_length, ALPHABET[0])
    out = []
    while n > 0:
        n, rem = divmod(n, BASE)
        out.append(ALPHABET[rem])
    s = "".join(reversed(out))
    return s.rjust(min_length, ALPHABET[0])


def content_addressed_id(content: bytes, length: int = 10) -> str:
    """Naive: same content always yields the same ID. See privacy notes."""
    digest = hashlib.sha256(content).digest()
    # Take the first 8 bytes (64 bits) → encode → truncate to 10 base62 chars.
    truncated = int.from_bytes(digest[:8], "big")
    return encode_base62(truncated, min_length=length)[:length]
```

At a 60-bit truncation and 1 billion pastes, collision probability is `≈ 1 - exp(-10^18 / (2 × 1.15 × 10^18))` ≈ 35%. That's unacceptable for a primary key. Truncate to 72 bits (12 base62 chars) for billion-paste scale, or — better — drop content addressing entirely.

### The Cross-User Privacy Leak

The fatal flaw: **two different users who paste the same content end up at the same URL**.

Imagine Alice pastes a Slack DM transcript about a layoff, intending to share it with one journalist via "unlisted" link. Bob, separately, pastes the same transcript a day later. Bob's URL is identical to Alice's. Bob now has access to whatever metadata, comments, or revisions are attached to "this paste." If the service exposes a creator field, Bob now knows Alice's account.

Even worse: **anyone who has the content can compute the URL.** An attacker who suspects "did anyone paste my company's internal Q3 plan?" can hash candidate documents and probe `pastebin.example.com/<hash>`. A 200 OK confirms the leak. Content-addressed IDs without per-user salting **invert the security model** — instead of needing the ID to read the content, you need the content to find the ID, which the attacker already has.

This is why the parent doc lists content addressing as the wrong default. Use content hashing only as an **opt-in deduplication signal** at the storage layer (S3 object key), behind a layer of public IDs that are random.

## Salting Per-User and Per-Session

If you genuinely want intra-user dedup (the same user re-pasting the same content gets the same URL, saving a row) without the cross-user leak, **salt the hash per-user**:

```python
"""Per-user salted content addressing.

Same user + same content → same paste ID (intra-user dedup preserved).
Different users + same content → different paste IDs (privacy preserved).
"""
import hashlib
import hmac
import os


# Server-wide secret — rotate via key-versioning if compromised.
# Stored in a secret manager (AWS Secrets Manager, Vault), NEVER in source.
SERVER_PEPPER = os.environ["PASTE_ID_PEPPER"].encode()


def user_salted_paste_id(
    content: bytes,
    user_id: str,
    length: int = 10,
) -> str:
    """HMAC the content with a per-user key derived from a server pepper.

    The HMAC construction is critical: a plain concatenation
    ``hash(user_id || content)`` is vulnerable to length-extension on
    older hash functions and leaks structure. ``hmac`` is the right tool.
    """
    user_key = hmac.new(SERVER_PEPPER, user_id.encode(), hashlib.sha256).digest()
    digest = hmac.new(user_key, content, hashlib.sha256).digest()
    truncated = int.from_bytes(digest[:8], "big")
    return encode_base62(truncated, min_length=length)[:length]
```

Variants:

- **Per-session salt** — re-paste within one session yields the same ID; across sessions, fresh ID. Tighter privacy; less aggressive dedup.
- **Anonymous-bucket salt** — for not-logged-in pasters, bucket by IP + day. Still leaks via IP-correlated requests, but better than no salting.
- **No dedup for private/password-protected pastes** — never content-address protected content; always random. Saves you from accidentally letting the wrong tier inherit dedup semantics.

A pragmatic split: **public pastes** are content-addressed with no salt (dedup wins, the content is meant to be public anyway); **unlisted, private, and password-protected** pastes use random base62 with no content addressing. The visibility tier picks the strategy.

## Burst-Friendly Allocation

### Pre-Allocated ID Pools

Paste creation traffic is **bursty**, not Poisson. The traffic profile is "someone shared a Pastebin link in a 50K-member Discord, now everyone is creating a paste of their related snippet." Steady-state 100 writes/sec can spike to 5K writes/sec for thirty seconds.

Random base62 generation is stateless — every app server can mint IDs independently — so the bottleneck is **DB collision-check round-trips** during the burst, not allocation. But two related concerns benefit from pre-allocation:

- The internal numeric primary key (UUIDv7 or sequence-backed `bigint`) needs to handle the burst without hammering a central allocator.
- A Bloom-filter cache of existing public IDs (warmed lazily) needs to absorb writes without falling out of sync.

The pattern, lifted from the URL-shortener doc's [range allocation](../url-shortener/code-generation-strategies.md#centralized-vs-range-allocated-counters):

- Each app server reserves a 10K block of internal `bigint` PKs at startup (or every five minutes).
- Public 10-char base62 codes are random per insert; collision retries hit the unique index, not a central allocator.
- The Bloom filter is per-server, replicated via Redis Bloom or rebuilt from DB on a schedule.

### Sizing the Pool

Three numbers drive block sizing:

```text
block_size = peak_writes_per_minute_per_server × refill_interval_minutes × safety_factor
```

For a service with peak burst of 5K writes/sec spread across 10 app servers (500/sec/server, ~30K/min/server), refilling every 5 minutes with safety factor 2:

```text
block_size = 30,000 × 5 × 2 = 300,000
```

That sounds large; it isn't. A 300K block of 64-bit integers is 2.4 MB of address space. Crashing mid-block "leaks" up to 300K integer IDs — irrelevant in a `bigint` namespace of 9.2 × 10^18.

Rule of thumb: **size the block so a server can sustain a 30-minute burst on its current allocation**. Bigger blocks beat tighter blocks because the operational cost of refill misses is much higher than the cost of leaked IDs.

## Visibility Tiers and ID Design

### Public, Unlisted, Private, Password-Protected

Most paste services support some subset of:

| Tier | Discoverability | Access Control | Common Use |
|---|---|---|---|
| **Public** | Listed in search/index pages, indexed by Google | None — anyone with the URL | Sharing reproducible bug reports, demos |
| **Unlisted** | Not listed; not indexed; URL is the secret | URL itself | Default for "ad-hoc share with one person" |
| **Private** | Owner only; requires login | Authenticated session | Personal notes, drafts |
| **Password-protected** | URL public; content gated | URL + password | "I'll DM you the password" workflows |
| **Burn-after-read** | Self-deleting on first GET | URL + atomic delete | Sharing one-time secrets, OTP-like flows |

### The ID Must Not Reveal Visibility

A common mistake: prefixing IDs with the tier (`pub-aZ3kQ9pX`, `priv-bR7tM2qN`). Three reasons not to:

- **Information leak** — an attacker scanning the namespace can target only `priv-` codes and ignore `pub-` ones, doubling their effective probe rate.
- **Tier mutations** — users sometimes change a paste from unlisted to public. If the ID encodes the tier, you must either re-issue (breaking links) or live with the lie.
- **Migration friction** — adding a new tier requires either reusing a prefix (semantics drift) or expanding the alphabet (code-space breaks).

The visibility tier belongs **in the metadata row**, not in the ID. The ID is opaque; the row stores `visibility ENUM('public', 'unlisted', 'private', 'password')`. The application enforces access at lookup time. The ID being unguessable substitutes for access control on unlisted pastes; for private and password-protected ones, it's an additional defense-in-depth layer.

The corollary: **never use sequential or short IDs for any tier**. If unlisted pastes have 6-char codes (assuming security through rate-limiting) and private pastes have 12-char codes, an attacker has just learned to ignore the long ones — the tier is encoded in length even if not in alphabet.

### Pastebin.com vs GitHub Gist Conventions

- **Pastebin.com** uses 8-character alphanumeric keys for all visibility levels (`pastebin.com/0b42rwhf`). Visibility is stored in metadata; the URL doesn't differ. Verified against the [Pastebin API documentation](https://pastebin.com/doc_api).
- **GitHub Gist** uses 32-character hexadecimal IDs (`gist.github.com/<user>/abc123…`). Public and "secret" gists share the same ID format and length. Per [GitHub's gist docs](https://docs.github.com/en/get-started/writing-on-github/editing-and-sharing-content-with-gists/creating-gists), secret gists are not listed publicly but are not access-controlled — anyone with the URL can read them. Same model as Pastebin's "unlisted."
- **Hastebin / Toptal Hastebin** uses 10-character lowercase alphanumeric (`hastebin.com/abcdefghij`). One tier (public-by-default-after-creation) and a flat namespace.

The shared design: **flat namespace, opaque ID, visibility in metadata**. Don't reinvent this.

## Crawler Resistance

### robots.txt Defaults

A paste service must instruct search engines not to index unlisted pastes. The cheap default — block everything below `/`:

```text
# robots.txt — paste.example.com
User-agent: *
Disallow: /raw/
Disallow: /<paste-id>/

# Public-listing pages are crawlable; specific pastes are not.
Allow: /trending
Allow: /archive
Allow: /docs/

Sitemap: https://paste.example.com/sitemap.xml
```

The challenge: search engines may have already crawled URLs they discovered before you tightened the policy. `robots.txt` is advisory — well-behaved crawlers honor it; bad actors do not. **Combine with the per-response header below.**

### Per-Response X-Robots-Tag

For unlisted, private, and password-protected pastes, return a `noindex` directive in the HTTP response itself. Per [Google's block-indexing documentation](https://developers.google.com/search/docs/crawling-indexing/block-indexing), this is more authoritative than `robots.txt` because it's per-resource and applies even to crawlers that already discovered the URL.

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
X-Robots-Tag: noindex, nofollow, noarchive, nosnippet
Cache-Control: private, no-cache
Vary: Cookie

<!doctype html>
<html><head>
  <title>Paste — paste.example.com</title>
  <meta name="robots" content="noindex,nofollow,noarchive,nosnippet">
</head>
<body>...</body></html>
```

For raw paste endpoints (`/raw/<id>`), the same header on the plain-text response works — `X-Robots-Tag` applies to non-HTML resources too.

For **public** pastes you might emit `X-Robots-Tag: index, follow` (or omit the header). Make it explicit per visibility tier; don't let the default leak unlisted content into Google.

### ID Sparsity as a Defense

The third leg: make the namespace too sparse to enumerate. At 10-char base62, the code space is `8.39 × 10^17`. Even 1 billion pastes occupy `1.2 × 10^-9` of the space — fill density of one paste per billion candidate codes. A crawler probing random 10-char strings hits a real paste roughly **once per billion probes**. At 1M probes/sec (aggressive for a public service that should be rate-limiting much harder), one hit per ~17 minutes of continuous probing.

This argues for two layered defenses:

- **Aggressive 404 rate-limiting.** Rate limit GET requests to non-existent codes per IP, per ASN, per session — far more aggressively than 200 responses. Flag and CAPTCHA-challenge IPs that 404 more than 100 times per minute.
- **Tarpit responses.** For repeat offenders, slow the 404 response to 5–10 seconds. Crawlers stall; legitimate users (who don't 404 hundreds of times) never notice.

For the rate-limiter implementation patterns, see [`../design-rate-limiter.md`](../design-rate-limiter.md).

## Migration to Longer IDs

Eventually a paste service either grows past safe density on its current code length or upgrades its threat model and decides 8 chars is no longer sufficient. The migration mirrors the URL-shortener migration pattern in [`../url-shortener/code-generation-strategies.md`](../url-shortener/code-generation-strategies.md#migrating-to-a-longer-code-length): **only new writes change; existing codes keep working forever**.

What's Pastebin-specific:

- **Visibility tier may dictate length per-tier.** Public pastes might keep 8 chars (low security need); unlisted upgrade to 12. The router accepts both; the generator emits the right length per tier.
- **Search and crawler policies should re-check.** A namespace expansion shouldn't accidentally re-expose previously unlisted pastes. Re-test crawler-blocking after the change.
- **Cache key shape may change.** A Redis cache keyed on `paste:<id>` works fine for variable-length IDs; one keyed on `paste:<exactly-8-chars>` does not. Audit cache wrappers before flipping the generator.
- **Validator regex must be widened first, narrowed never.** Deploy `/^[A-Za-z0-9]{8,12}$/` before flipping to 12-char generation. Reverse order means a brief outage where 12-char codes get rejected at the routing layer.

The forbidden direction: **shrinking IDs**. You can't compress a 12-char code into 8 without re-issuing. Don't promise users it's possible.

## Auditability — Embedding Tenant Identity

Enterprise tiers (think GitHub Enterprise Gists, Atlassian's confluence-style code-share, internal corporate Pastebins) often need creator-traceable IDs for audit and DLP (data-loss-prevention) integration. Two approaches, both with trade-offs against unguessability.

**Approach A — Tenant prefix.** The ID becomes `<tenant>-<random>`:

```text
acme-aZ3kQ9pX1m       # acme corp, random body
globex-bR7tM2qNpL     # globex corp
```

Pros:

- Audit logs read naturally — `acme-aZ3kQ9pX1m` is unambiguous.
- Tenant routing at the load balancer is trivial (parse prefix, route to the right backend pool).
- Cross-tenant collisions are impossible by construction.

Cons:

- The tenant is publicly visible. For competitive-intelligence reasons (does Acme use Pastebin? how often?) some tenants reject this.
- Brute-force scope is reduced — an attacker who suspects Acme has secrets to find probes only `acme-*`, ignoring 99% of the namespace.

**Approach B — Opaque ID + audit join table.** Public ID stays random; tenant is recovered by joining `paste_id → tenant_id` in a server-side audit table the user never sees.

Pros:

- Public namespace is uniform and unguessable.
- Tenants are not externally identifiable.

Cons:

- Audit queries require a join. Fast on indexed tables, but more moving parts than parsing a prefix.
- Recovery scenarios (what if the join table is corrupted?) are scarier.

For most enterprise services, **Approach B is correct**. Tenants who want a visible prefix can negotiate it as a customer-visible feature; the default should not leak tenant identity. The unguessability budget is too valuable to spend on routing convenience. For a deeper treatment of authorization layers, see [`../../../security/authorization.md`](../../../security/authorization.md).

A subtlety for both approaches: **the audit log itself must not be content-addressed**. Logging "user X created paste with content hash Y" lets a future attacker who exfiltrates the log replay the privacy-leak attack from [the cross-user section](#the-cross-user-privacy-leak). Log opaque paste IDs only.

## Comparison Table — Pastebin.com vs GitHub Gist vs Hastebin

| Property | Pastebin.com | GitHub Gist | Hastebin |
|---|---|---|---|
| **Code length / format** | 8-char alphanumeric (`0b42rwhf`) | 32-char hex (`abc123def456…`) | 10-char lowercase alphanumeric |
| **Code-space size** | `62^8 ≈ 2.18 × 10^14` | `16^32 ≈ 3.4 × 10^38` | `36^10 ≈ 3.66 × 10^15` |
| **Visibility tiers** | Public, Unlisted, Private (paid) | Public, Secret | Public only |
| **Visibility encoded in ID?** | No | No | N/A (one tier) |
| **Custom aliases** | No | No | No |
| **Owner in URL?** | No | Yes (`/<user>/<id>`) | No |
| **Indexed by Google by default?** | Public yes; Unlisted no | Public yes; Secret no | Yes (no opt-out) |
| **Expiration** | Configurable (10m to never) | None (manual delete) | None |
| **Visible enumeration vectors** | Public-paste archive page | Public gist Discover feed | Recent-pastes feed |

What the choices imply:

- **Pastebin.com's 8-char keys** signal a service that monetizes through ads + subscriptions and accepts moderation cost as a business expense — short codes drive high engagement at the cost of higher abuse exposure. Their well-known credential-leak indexing problem (third-party scrapers like Pastebin scrapers harvest pasted secrets) is partly an artifact of cheap ID enumeration plus high paste volume.
- **GitHub Gist's 32-char hex** is overkill for unguessability but consistent with GitHub's broader use of full SHA-1-derived IDs for everything. They prefer aesthetic and engineering consistency over URL brevity.
- **Hastebin's 10-char lowercase** is what a privacy-naive but density-conscious greenfield service would build today. Lowercase-only halves the alphabet from 62 to 36 — a deliberate choice for case-insensitive verbal sharing, with the entropy hit (~10 bits) absorbed by adding two characters relative to 8-char base62.

If you're designing a new service, **the Hastebin shape with case-sensitive base62 (so 10 chars but 60 bits) is the modern default**. Add visibility tiers in metadata; never encode them in the ID.

## Anti-Patterns

- **Sequential or counter-derived public IDs.** Lets anyone iterate the corpus. The URL-shortener doc has the math; for Pastebin the consequences are worse because the payload is content, not a redirect.
- **Content-addressed IDs without per-user salt.** The cross-user privacy leak is severe and silent — the first incident report is the customer who finds someone else's paste at their URL.
- **Visibility encoded in the ID** (prefix, length, alphabet). Now the namespace is partitioned for the attacker; they can target the high-value tier directly.
- **Short codes (≤6 chars) for unlisted pastes.** Six characters is 36 bits — brute-forceable in days at 10K probes/sec, hours at 1M. Unlisted pastes are protected by the URL alone; that URL needs real entropy.
- **`robots.txt` only, without per-response `X-Robots-Tag`.** Crawlers that already discovered URLs ignore `Disallow`; per-resource headers are how you actually opt out.
- **Custom alias support that allows short or guessable slugs** (`/secrets`, `/dump`, `/passwords`). Either disallow custom aliases entirely, or enforce a minimum length and reject a denylist. Nobody should be able to claim `pastebin.example.com/keys`.
- **Tenant ID embedded in the public paste ID.** Leaks customer identity; reduces brute-force scope. Use a server-side audit join instead.
- **Reusing one Bloom filter across visibility tiers.** Membership-test patterns differ; cross-tier false positives can leak existence ("we returned a maybe-present for an ID that turned out to be in the private pool"). Filter per tier or use per-tier salts.
- **Logging the paste content alongside the public ID.** Audit logs that include content allow later replay of the content-addressed-leak attack against any historical hash truncation. Log IDs only.
- **Testing crawler policy in production by waiting for Googlebot.** Use the `URL Inspection` tool in Google Search Console and `curl -I` for `X-Robots-Tag` verification before relying on the policy.

## Related

- Parent case study — [`../design-pastebin.md`](../design-pastebin.md)
- Shared ID-generation math (birthday paradox, base62 mechanics, range allocation) — [`../url-shortener/code-generation-strategies.md`](../url-shortener/code-generation-strategies.md)
- Bloom and Cuckoo filters (front-door collision check) — [`../../../data-structures/bloom-and-cuckoo-filters.md`](../../../data-structures/bloom-and-cuckoo-filters.md)
- Authorization patterns for tier enforcement — [`../../../security/authorization.md`](../../../security/authorization.md)
- Rate limiting brute-force probes — [`../design-rate-limiter.md`](../design-rate-limiter.md)

## References

- [RFC 9562 — Universally Unique IDentifiers (UUIDs)](https://www.rfc-editor.org/rfc/rfc9562) — IETF standard (May 2024) defining UUIDv7 with 48-bit Unix-millisecond timestamp + 74 random bits; the canonical sortable UUID format for B-tree-friendly internal record IDs.
- [Pastebin.com — API documentation](https://pastebin.com/doc_api) — official feature and API reference, source for the 8-char alphanumeric paste-key format, three visibility levels (Public / Unlisted / Private), and expiration value taxonomy used in the comparison table.
- [GitHub Docs — Creating Gists](https://docs.github.com/en/get-started/writing-on-github/editing-and-sharing-content-with-gists/creating-gists) — official GitHub docs, source for the 32-char hex gist ID and the public-vs-secret visibility model (secret gists are URL-gated, not access-controlled).
- [Google Search Central — Block Search Indexing](https://developers.google.com/search/docs/crawling-indexing/block-indexing) — authoritative reference for `noindex` semantics, the `X-Robots-Tag` HTTP header, and the requirement that pages must not be blocked by `robots.txt` for `noindex` to be honored.
- [Wikipedia — Birthday Problem](https://en.wikipedia.org/wiki/Birthday_problem) — formal treatment of the collision-probability math used throughout the SHA-256-truncation analysis; same source as the URL-shortener companion.
- [RFC 9110 — HTTP Semantics § Vary](https://www.rfc-editor.org/rfc/rfc9110#section-12.5.5) — defines the `Vary` header used in the `X-Robots-Tag` example to keep visibility-gated responses out of shared caches.
- [Python `secrets` module documentation](https://docs.python.org/3/library/secrets.html) — official guidance that `secrets.choice` is the correct CSPRNG primitive for security-relevant tokens; `random.choice` is explicitly insecure.
- [OWASP — Insecure Direct Object References (IDOR) Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html) — background on why guessable identifiers are an authorization anti-pattern, directly applicable to the unlisted-paste-as-secret-URL model.
