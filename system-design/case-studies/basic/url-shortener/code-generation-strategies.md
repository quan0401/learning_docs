---
title: "URL Shortener Deep Dive — Code Generation Strategies"
date: 2026-04-27
updated: 2026-04-27
tags: [system-design, case-study, url-shortener, deep-dive, id-generation]
---

# URL Shortener Deep Dive — Code Generation Strategies

**Date:** 2026-04-27 | **Updated:** 2026-04-27
**Tags:** `system-design` `case-study` `url-shortener` `deep-dive` `id-generation`
**Companion to:** [`../design-url-shortener.md`](../design-url-shortener.md)

## Table of Contents

- [Summary](#summary)
- [Overview — Four Families, One Decision](#overview--four-families-one-decision)
- [Strategy 1 — Counter + Base62](#strategy-1--counter--base62)
  - [The Alphabet — Why Base62, Not Base64 or Base58](#the-alphabet--why-base62-not-base64-or-base58)
  - [Encoding and Decoding](#encoding-and-decoding)
  - [Sequence Allocation Patterns](#sequence-allocation-patterns)
  - [Centralized vs Range-Allocated Counters](#centralized-vs-range-allocated-counters)
  - [Hot-Shard Problems on Monotonic IDs](#hot-shard-problems-on-monotonic-ids)
- [Strategy 2 — Snowflake-Style Allocators](#strategy-2--snowflake-style-allocators)
  - [The 64-Bit Bit Layout](#the-64-bit-bit-layout)
  - [Failure Modes — Clock Skew and Machine ID Reuse](#failure-modes--clock-skew-and-machine-id-reuse)
  - [Variants — Sonyflake, Instagram, UUIDv7](#variants--sonyflake-instagram-uuidv7)
- [Strategy 3 — Hash + Truncate Pitfalls](#strategy-3--hash--truncate-pitfalls)
  - [Birthday Paradox Math](#birthday-paradox-math)
  - [Why MD5 Truncation Fails at Scale](#why-md5-truncation-fails-at-scale)
  - [Salt and Per-Tenant Prefixing](#salt-and-per-tenant-prefixing)
  - [Determinism — Idempotency vs Information Leak](#determinism--idempotency-vs-information-leak)
- [Strategy 4 — Random + Collision Check](#strategy-4--random--collision-check)
  - [Fill-Density Math](#fill-density-math)
  - [Bloom Filter Front-Door](#bloom-filter-front-door)
  - [Single-Flight Retry Patterns](#single-flight-retry-patterns)
  - [Code-Space Exhaustion Planning](#code-space-exhaustion-planning)
- [Allocator Architecture](#allocator-architecture)
  - [Pull vs Push Allocation](#pull-vs-push-allocation)
  - [Coordinated Counters — ZooKeeper, etcd, Redis](#coordinated-counters--zookeeper-etcd-redis)
  - [Batched Allocation and the Crash-Mid-Block Trade-off](#batched-allocation-and-the-crash-mid-block-trade-off)
- [Code Length and Unguessability](#code-length-and-unguessability)
  - [Entropy at 6 / 7 / 8 Characters](#entropy-at-6--7--8-characters)
  - [Migrating to a Longer Code Length](#migrating-to-a-longer-code-length)
- [Special-Namespace Reservations](#special-namespace-reservations)
- [Choosing the Right Strategy — Decision Matrix](#choosing-the-right-strategy--decision-matrix)
- [Real-World Implementations](#real-world-implementations)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

The parent case study at [`../design-url-shortener.md`](../design-url-shortener.md) sketches three viable code-generation strategies in about thirty lines. That summary is enough to pick a direction in an interview but not enough to actually build one in production. This deep-dive expands every line into the math, code, and operational trade-offs the parent doc could only gesture at: **base62 encoding mechanics**, **Snowflake bit-packing**, **birthday-paradox arithmetic for hash truncation**, **Bloom-filter front-doors for collision checks**, **pull-vs-push allocator architectures**, and **what to actually do when you have to grow the short-code length without breaking five years of QR codes already in the wild**.

The single most important lesson: **code generation looks like a one-line decision but every choice trades unguessability against density, coordination cost, hot-shard behavior, and the abuse-economy attack surface**. Pick the strategy whose failure modes you can live with — not the one with the prettiest math.

## Overview — Four Families, One Decision

Every URL-shortener code generator falls into one of four families:

| Family | One-Line Description | Coordination | Guessability | Collision Handling |
|---|---|---|---|---|
| **Counter + Base62** | Increment a global integer; encode in base62 | Central allocator (sharded) | High (sequential) | None needed |
| **Snowflake-style** | Pack timestamp + machine ID + sequence into 64 bits, base62 it | None at runtime, machine IDs assigned at boot | Medium (timestamp prefix observable) | None needed |
| **Hash + truncate** | `base62(hash(long_url + salt))[:n]` | None | Low if salt unknown | Required (birthday-paradox collisions) |
| **Random + check** | Generate random base62 string; insert with conflict check | None at generation; DB on insert | Low (uniform random) | Required (low at sparse fill) |

The other three deep-dive sections in the parent doc (cache strategy, custom aliases, click analytics) each have their own concerns. Here we are exclusively concerned with **how the seven-or-eight-character string in `https://sho.rt/aZ4kP9` gets minted**.

## Strategy 1 — Counter + Base62

The classic approach: maintain a globally monotonic 64-bit integer counter and convert each value to a base62 string. With the alphabet `[0-9A-Za-z]` (62 symbols), you get:

```text
62^6 ≈ 56.8 billion       (6 chars)
62^7 ≈ 3.52 trillion      (7 chars)
62^8 ≈ 218.3 trillion     (8 chars)
```

For our case-study scale (1B URLs over 5 years), 6 characters is dangerously close — the search-space is only 56× the URL count. 7 chars is the production sweet spot.

### The Alphabet — Why Base62, Not Base64 or Base58

You have several alphabets to choose from:

| Alphabet | Symbols | URL-Safe? | Notes |
|---|---|---|---|
| **Base64** (RFC 4648) | `A-Za-z0-9+/` | No (`+`, `/` need encoding) | Too dense for URLs |
| **URL-safe Base64** | `A-Za-z0-9-_` | Yes | 64 symbols; codes can collide visually with hyphens in path |
| **Base62** | `0-9A-Za-z` | Yes | 62 symbols; canonical for shorteners |
| **Base58** (Bitcoin) | Base62 minus `0`, `O`, `I`, `l` | Yes | Strips visually-ambiguous glyphs; used by Flickr's "short photo IDs" and Bitcoin addresses |
| **Crockford Base32** | `0-9A-HJKMNP-TV-Z` | Yes | Excludes `I`, `L`, `O`, `U`; case-insensitive; humans can read codes aloud |

**Why exclude `0/O/I/l`?** Print a code on a flyer or speak it on a podcast and watch your support tickets. `bDcF7Tx` is fine; `lO0I1` is a humanity test. If your codes are ever displayed in non-monospaced fonts, read aloud, or transcribed by hand, base58 is worth the ~3% density penalty (`58^7 ≈ 2.21T` vs `62^7 ≈ 3.52T`).

For purely-machine consumption (QR codes, in-browser only), base62 wins on density and is universally supported. **Default to base62; switch to base58 the moment a human has to type a code.**

### Encoding and Decoding

```python
"""Pure-Python base62 encode/decode — no dependencies."""

ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
BASE = len(ALPHABET)               # 62
INDEX = {c: i for i, c in enumerate(ALPHABET)}


def encode_base62(n: int, min_length: int = 0) -> str:
    """Encode a non-negative integer as a base62 string."""
    if n < 0:
        raise ValueError("base62 encode requires non-negative integer")
    if n == 0:
        return ALPHABET[0].rjust(min_length, ALPHABET[0])
    out: list[str] = []
    while n > 0:
        n, rem = divmod(n, BASE)
        out.append(ALPHABET[rem])
    s = "".join(reversed(out))
    if len(s) < min_length:
        s = ALPHABET[0] * (min_length - len(s)) + s
    return s


def decode_base62(s: str) -> int:
    """Decode a base62 string back to its integer value."""
    n = 0
    for ch in s:
        if ch not in INDEX:
            raise ValueError(f"invalid base62 character: {ch!r}")
        n = n * BASE + INDEX[ch]
    return n


# Round-trip example:
# encode_base62(125_000_000_007)            == 'dRgaPN9'   (7 chars)
# decode_base62('dRgaPN9')                  == 125000000007
# encode_base62(0, min_length=7)            == '0000000'
```

For Go, the same algorithm with `uint64` arithmetic is ~5× faster than the Python version on the same hardware; use `math/big` only if you need IDs above 64 bits.

**Implementation gotchas:**

- **Pad to a fixed length** for displayed codes (`encode_base62(n, min_length=7)`) so users see consistent URLs from `0000001` onward. Without padding, the first 62 codes are 1 char, the next ~3,800 are 2 chars, etc.
- **Validate input on decode** — never assume user-supplied codes are well-formed. A `ValueError` here is a `400 Bad Request`, not a 500.
- **Be deliberate about leading-zero semantics.** `encode(0)` is `"0"`, but is `"00"` the same as `"0"`? In a shortener, **no** — they are different strings, both valid codes that map to different long URLs. Strip nothing.

### Sequence Allocation Patterns

Three production-grade ways to obtain "the next monotonic integer":

**Postgres `SEQUENCE`:**

```sql
CREATE SEQUENCE url_codes
    START 1_000_000_000        -- avoid 1-3 char codes for the first billion
    INCREMENT 1
    CACHE 1000;                -- pre-allocate 1000 IDs per backend connection
```

```sql
SELECT nextval('url_codes');   -- atomic, transactional
```

`CACHE 1000` is the trick: each backend session pulls 1,000 IDs into its private cache and serves them locally without round-tripping. The trade-off is that **gaps in the sequence are unavoidable** — if a backend disconnects mid-cache, those 1,000 IDs are silently lost. For a URL shortener this is fine (we don't care about gaps; we care about uniqueness). For invoice numbers, it's a compliance problem.

**Redis `INCR`:**

```python
# Redis INCR is atomic and O(1).
next_id = redis.incr("url:counter")
short_code = encode_base62(next_id, min_length=7)
```

Pros: trivially fast, no transactional overhead, atomic by design (single-threaded server). Cons: durability is async by default — `appendfsync everysec` can lose up to 1 second of increments on crash. Configure `appendfsync always` (high write cost) or accept gaps. Redis docs explicitly endorse `INCR` as a counter pattern.

**MySQL `AUTO_INCREMENT`:**

```sql
CREATE TABLE codes (
    id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Pre-allocate ranges with auto_increment_increment for multi-master
SET @@auto_increment_offset = 1;
SET @@auto_increment_increment = 4;   -- 4 masters: 1,5,9,13... | 2,6,10,14...
```

The multi-master `auto_increment_increment` trick was used to early-Twitter-scale write throughput on MySQL before Snowflake. It's awkward (changing master count requires re-tuning) but it works.

### Centralized vs Range-Allocated Counters

A single Postgres `SEQUENCE` or Redis key is a **bottleneck at high write volume** — every shortener service hop must round-trip to the allocator. The standard fix: allocate **ranges**, not individual IDs.

```python
"""Range-allocator client: each app server pre-allocates a 10K block."""

import threading
from dataclasses import dataclass


@dataclass
class IDBlock:
    next_id: int
    end_id: int     # exclusive


class RangeAllocator:
    BLOCK_SIZE = 10_000

    def __init__(self, central_client):
        self._central = central_client      # e.g., a Postgres or Redis client
        self._lock = threading.Lock()
        self._block: IDBlock | None = None

    def _refill(self) -> IDBlock:
        """Atomically reserve the next BLOCK_SIZE IDs from the central allocator."""
        # Postgres example (atomic increment-by-N):
        #   SELECT setval('url_codes', nextval('url_codes') + 9999) - 9999
        # Redis example:
        #   INCRBY url:counter 10000  -> end_id; start = end_id - 10000 + 1
        end_id = self._central.incrby("url:counter", self.BLOCK_SIZE)
        return IDBlock(next_id=end_id - self.BLOCK_SIZE + 1, end_id=end_id + 1)

    def next_id(self) -> int:
        with self._lock:
            if self._block is None or self._block.next_id >= self._block.end_id:
                self._block = self._refill()
            value = self._block.next_id
            self._block.next_id += 1
            return value
```

Properties:

- **Round-trips drop by ~10,000×.** A backend serving 100K writes/sec hits the central allocator 10/sec.
- **Gaps are accepted.** If the backend crashes with 7,500 IDs unused in its block, those IDs are forever skipped. **For URL shorteners this is fine** — base62-encoded skipped IDs just produce codes that never get assigned. We have ~3.5 trillion 7-char codes; losing a few thousand per crash is invisible.
- **Sequence is no longer globally monotonic across backends** — backend A might allocate IDs 10000–19999 while backend B has 20000–29999, and writes interleave. If you depend on "the most recent code is the highest integer", this breaks.
- **Block size is a tuning knob.** Bigger blocks = fewer round-trips but bigger gaps on crash. 1K–100K is the standard band.

### Hot-Shard Problems on Monotonic IDs

If `code` is your shard key (typical for shorteners), monotonic integers cluster all new writes onto **one shard at a time**. Going from `code = 1234567` to `code = 1234568` to `code = 1234569`, all encoded as `bDcF7Tx`, `bDcF7Ty`, `bDcF7Tz` — they hash to nearby positions, all writes hammer the shard owning that range.

Three mitigations:

1. **Hash before sharding.** Don't shard on `code` directly; shard on `hash(code)`. Sequential codes spray across shards. (You lose range-scan capability, which most shorteners don't need.)
2. **Reverse the bytes before encoding.** `bit-reverse(counter)` before base62 turns sequential integers into pseudo-random ones. Old MySQL trick; less common today.
3. **Use a different ID scheme entirely.** Snowflake-style IDs have the timestamp at the high-order bits, but the low-order sequence number is what dominates short-term writes — and a sufficiently random `machine_id` mid-bits fan out the writes naturally.

For deeper sharding context, see [`../../../scalability/sharding-strategies.md`](../../../scalability/sharding-strategies.md).

## Strategy 2 — Snowflake-Style Allocators

[Twitter's Snowflake (2010)](https://github.com/twitter-archive/snowflake) was built to replace MySQL `AUTO_INCREMENT` as the canonical ID generator after Twitter migrated tweets to Cassandra. It's now retired but the design is still the dominant template for distributed ID generation.

### The 64-Bit Bit Layout

Twitter's original layout, packed into a signed 64-bit integer (the high bit is reserved as 0 to keep IDs positive):

```text
| 0 | 41 bits — timestamp (ms since custom epoch) | 10 bits — machine_id | 12 bits — sequence |
  ^   ^                                              ^                     ^
  |   69 years before timestamp wraps                1024 machines         4096 IDs/ms/machine
  |
  reserved sign bit
```

That gives you 4,096 unique IDs per millisecond per machine, across 1,024 machines, for 69 years from your custom epoch — comfortably enough for any consumer service.

```python
"""Snowflake-style 64-bit ID generator — single-machine, thread-safe."""
import threading
import time


class SnowflakeGenerator:
    # Bit allocations (Twitter's original)
    TIMESTAMP_BITS = 41
    MACHINE_BITS = 10
    SEQUENCE_BITS = 12

    MAX_MACHINE_ID = (1 << MACHINE_BITS) - 1     # 1023
    MAX_SEQUENCE = (1 << SEQUENCE_BITS) - 1      # 4095

    # Custom epoch — pick something recent so timestamps fit in 41 bits longer.
    EPOCH_MS = 1_704_067_200_000   # 2024-01-01T00:00:00Z

    def __init__(self, machine_id: int):
        if not 0 <= machine_id <= self.MAX_MACHINE_ID:
            raise ValueError(f"machine_id must be in [0, {self.MAX_MACHINE_ID}]")
        self.machine_id = machine_id
        self._lock = threading.Lock()
        self._last_ts = -1
        self._sequence = 0

    @staticmethod
    def _now_ms() -> int:
        return int(time.time() * 1000)

    def next_id(self) -> int:
        with self._lock:
            ts = self._now_ms() - self.EPOCH_MS
            if ts < self._last_ts:
                # Clock moved backwards. Refuse rather than mint duplicates.
                raise RuntimeError(
                    f"clock skew detected: {self._last_ts - ts}ms backward"
                )
            if ts == self._last_ts:
                self._sequence = (self._sequence + 1) & self.MAX_SEQUENCE
                if self._sequence == 0:
                    # Sequence exhausted in this ms — spin until next ms.
                    while ts <= self._last_ts:
                        ts = self._now_ms() - self.EPOCH_MS
            else:
                self._sequence = 0
            self._last_ts = ts
            return (
                (ts << (self.MACHINE_BITS + self.SEQUENCE_BITS))
                | (self.machine_id << self.SEQUENCE_BITS)
                | self._sequence
            )


# Usage:
# gen = SnowflakeGenerator(machine_id=42)
# id_int = gen.next_id()                       # 64-bit integer
# code = encode_base62(id_int, min_length=11)  # ~11 base62 chars for full 64-bit
```

For a URL shortener you typically take only the **lower bits** of a Snowflake ID and base62-encode them, since you don't need 64 bits of code-space. A 42-bit ID base62-encoded is 7 chars; that's our target. **But truncating a Snowflake destroys its uniqueness guarantee** — the dropped high bits are precisely the timestamp ones that prevent same-machine collisions across milliseconds. Either use the full ID or use a different generation strategy. Don't mix.

### Failure Modes — Clock Skew and Machine ID Reuse

Snowflake's two production hazards are both about coordination state being out of sync with reality:

**Clock skew (clock moving backwards):**

- NTP correction can step the clock backwards on boot or after a long drift.
- VM live-migration occasionally re-syncs the guest clock.
- Manual operator intervention.

If the clock moves back even 1 ms, two distinct calls to `next_id()` could produce the same `(timestamp, machine_id, sequence)` tuple. **Always refuse rather than mint duplicates** — log loudly and let the caller retry. Some implementations (Sonyflake, modern variants) use a monotonic counter instead of wall-clock time to sidestep this entirely, at the cost of human-readable timestamps.

**Machine ID reuse:**

- A machine crashes and reboots; its ID is still in flight in some other allocator's awareness.
- A new VM gets the same machine_id from a recycled pool.
- The orchestrator assigns the same ID to two pods due to a coordinator bug.

Two machines minting IDs with the same `machine_id` will collide on `(timestamp, machine_id, sequence)` precisely when their clocks line up. Mitigations:

- **Persistent assignment via ZooKeeper/etcd.** The machine claims its ID on boot and renews a lease; concurrent claimers fail.
- **Inject the hostname or IP suffix into `machine_id`.** If your fleet has stable, unique IPs, use the low bits of the IP — no coordinator needed.
- **Health-check + sequence-of-zero rule.** When restarting, sleep until 1ms past the last persisted timestamp before issuing IDs.

### Variants — Sonyflake, Instagram, UUIDv7

| Variant | Layout | Trade-off |
|---|---|---|
| [**Sonyflake**](https://github.com/sony/sonyflake) | 39 bits time (10ms units) / 8 bits seq / 16 bits machine_id | 174-year epoch and 65K machines, but only ~25K IDs/sec/node (vs Snowflake's 4M/sec) |
| [**Instagram sharded IDs**](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c) | 41 bits timestamp / 13 bits logical shard / 10 bits auto-inc | Shard ID is the routing key — given an ID, you know which Postgres shard owns it without a lookup |
| [**UUIDv7 (RFC 9562)**](https://www.rfc-editor.org/rfc/rfc9562) | 48 bits Unix ms / 4 ver / 12 rand_a / 2 var / 62 rand_b | Standardized, sortable, B-tree-friendly; **128 bits is too long for 7-char URL codes** (would need 22 chars) |

For a URL shortener, **UUIDv7 is the wrong primitive at the public-code layer** — it's the right primitive for internal record IDs that pair with a separately-generated short code. Snowflake-style 64-bit IDs base62-encoded to 11 chars are workable but still longer than the 7-char ideal. **Snowflake variants shine when your workload genuinely needs cross-region uniqueness without a counter; for a single-region shortener, random + check is simpler.**

## Strategy 3 — Hash + Truncate Pitfalls

The seductive idea: `code = base62(md5(long_url))[:7]`. Same URL → same code; no counter; no coordination. It looks free. It is not.

### Birthday Paradox Math

The birthday paradox says: in a uniformly distributed space of size `D`, collisions become likely after roughly `√D` insertions, not `D`. The standard approximation:

```text
P(collision after n insertions) ≈ 1 - exp(-n² / 2D)
```

For 7-char base62 (`D = 62^7 ≈ 3.521 × 10^12`):

| Insertions (n) | Collision Probability | Notes |
|---|---|---|
| 1,000 | ≈ 0.000014% | Negligible |
| 100,000 | ≈ 0.14% | Tolerable |
| 1,000,000 | ≈ 14% | Noticeable |
| 1,876,000 | ≈ 39% | √D ≈ 1.876M; classic "birthday point" |
| 5,000,000 | ≈ 97% | Near-certain collision |
| 10,000,000 | ≈ 99.99% | Inevitable |

Worked example: at our case-study scale of **1 billion URLs**:

```text
n  = 10^9
D  = 3.521 × 10^12
n² = 10^18
n² / 2D = 10^18 / (7.042 × 10^12) ≈ 142,000
P(collision) ≈ 1 - exp(-142,000) ≈ 1.0   (mathematical certainty)
```

At 1 billion URLs in a 3.5-trillion-code space, **collisions are not a possibility — they are a guarantee, in vast numbers**. Hash-and-truncate without a collision-detection step will silently corrupt mappings. The only question is how many users see the wrong destination before someone notices.

For a deeper treatment of this math, see the [Wikipedia article on the birthday problem](https://en.wikipedia.org/wiki/Birthday_problem).

### Why MD5 Truncation Fails at Scale

MD5 itself is uniformly distributed across its 128-bit output, so truncation to 7 base62 chars (≈42 bits) is mathematically clean — collisions follow the birthday paradox exactly. The failure mode is purely a **search-space size** problem: 7 base62 chars is just too small relative to billion-scale ingestion.

Three real-world consequences:

1. **First-write-wins corrupts the second URL.** A naive `INSERT IGNORE` or `ON CONFLICT DO NOTHING` on the second arrival means the user thinks they got a short URL but it points to someone else's destination. Silent data loss.
2. **Determinism is no longer a feature** once you add collision detection — because now the "deterministic" code can shift to chars `[1:8]` or `[2:9]` of the hash on collision, defeating the idempotency that motivated the design.
3. **Pre-image attacks.** MD5 is broken cryptographically. An attacker who knows the salt structure can generate collisions on demand, hijacking targeted short URLs.

### Salt and Per-Tenant Prefixing

If you're going to use hash-truncate, **salt aggressively**:

```text
code_input = secret_salt + "|" + tenant_id + "|" + long_url + "|" + nanoseconds
code       = base62(sha256(code_input))[:7]
```

The salt prevents enumeration. The tenant prefix prevents cross-tenant code collisions and information leakage (tenant A can't determine that tenant B has shortened the same URL). The nanosecond suffix re-introduces non-determinism — which **defeats the original "deterministic shortening" pitch** but is the only way to make collision rates acceptable.

At which point you've reinvented Strategy 4 (random + check) with extra steps. **That's the conclusion**: hash-truncate is rarely the right answer for public URL codes.

### Determinism — Idempotency vs Information Leak

The one place where deterministic shortening genuinely helps:

- **Idempotency** — a buggy client retrying `POST /shorten` doesn't create duplicate codes for the same URL. This is real and useful.
- **Cache-friendliness** — repeated shortenings of trending links don't bloat the table.

The cost:

- **Information leak** — anyone can hash a URL with your salt (if they exfiltrate it) and discover whether it has been shortened. For shorteners used in marketing campaigns, internal docs, or invitation links, this is a privacy violation.
- **Adversarial discovery** — an attacker who suspects a target has shortened `https://internal.example.com/secret-doc` can hash it and probe `https://sho.rt/<expected_code>`.

The right answer for most production shorteners: **handle idempotency at the application layer** with a separate "request_id → code" cache (TTL ~5 minutes), and use random codes for the actual mapping. You get the idempotency benefit without the privacy leak.

## Strategy 4 — Random + Collision Check

Generate `n` random base62 characters; insert with `ON CONFLICT DO NOTHING`; on conflict, retry with a fresh random code.

```python
import secrets
from typing import Callable

def random_base62(length: int) -> str:
    return "".join(secrets.choice(ALPHABET) for _ in range(length))


def shorten_with_retry(
    long_url: str,
    insert_if_absent: Callable[[str, str], bool],   # returns True on success
    code_length: int = 7,
    max_retries: int = 5,
) -> str:
    for attempt in range(max_retries):
        code = random_base62(code_length)
        if insert_if_absent(code, long_url):
            return code
        # else: code was taken, try again
    raise RuntimeError(
        f"failed to mint unique code after {max_retries} retries — "
        f"code-space may be too dense"
    )
```

`secrets.choice` (Python) and `crypto/rand` (Go) are cryptographically secure RNGs — use them, not `random.choice` / `math/rand`, so the codes are unguessable.

### Fill-Density Math

The retry rate depends on the fraction of code-space already used, the **fill density** `ρ = n/D`. For 7-char base62:

| URLs in DB | Fill Density (ρ) | P(retry per attempt) | Expected Attempts |
|---|---|---|---|
| 10,000 | 0.000003% | 3 × 10^-8 | 1.0000 |
| 1,000,000 | 0.00003% | 3 × 10^-7 | 1.0000 |
| 100,000,000 | 0.003% | 3 × 10^-5 | 1.00003 |
| 1,000,000,000 | 0.028% | 2.8 × 10^-4 | 1.0003 |
| 100,000,000,000 | 2.84% | 0.028 | 1.029 |
| 1,000,000,000,000 | 28.4% | 0.284 | 1.40 |
| 2,500,000,000,000 | 71% | 0.71 | 3.45 |

At 1B URLs in 7-char space, retries are **astronomically rare** — you'll see one retry per ~3,500 inserts. The system is effectively collision-free.

The retry rate climbs sharply only past **~50% fill**. At 71% fill (2.5T URLs), every fourth insert retries — at which point you should already be shipping a migration to 8-char codes. Plan the migration when you cross 30–40% fill, not when you start failing.

### Bloom Filter Front-Door

For very high write rates, you can short-circuit the DB conflict check with a **Bloom filter** of existing codes. The filter is in-memory, sized for your code population, and tells you in O(1):

- "Definitely not present" → safe to insert without DB check.
- "Maybe present" → fall back to DB (rare false-positive).

```python
"""Bloom-filter-fronted random code generator.

The filter holds all live codes; on a 'definitely absent' answer we can
insert without a DB read. On 'maybe present' we fall through to the
existing INSERT ... ON CONFLICT path. False positives only cost a
DB round-trip; false negatives are impossible.
"""
import hashlib
import math
from bitarray import bitarray   # pip install bitarray


class BloomFilter:
    def __init__(self, expected_items: int, false_positive_rate: float = 0.001):
        # Optimal sizing — see https://en.wikipedia.org/wiki/Bloom_filter
        self.size = int(
            -expected_items * math.log(false_positive_rate) / (math.log(2) ** 2)
        )
        self.hash_count = max(1, int(self.size / expected_items * math.log(2)))
        self.bits = bitarray(self.size)
        self.bits.setall(False)

    def _hashes(self, item: str) -> list[int]:
        # Double-hashing trick: 2 cryptographic hashes, derive k indices.
        h1 = int.from_bytes(hashlib.sha256(item.encode()).digest()[:8], "big")
        h2 = int.from_bytes(hashlib.blake2b(item.encode()).digest()[:8], "big")
        return [(h1 + i * h2) % self.size for i in range(self.hash_count)]

    def add(self, item: str) -> None:
        for idx in self._hashes(item):
            self.bits[idx] = True

    def __contains__(self, item: str) -> bool:
        return all(self.bits[idx] for idx in self._hashes(item))


class CodeGenerator:
    def __init__(self, db, expected_codes: int = 2_000_000_000):
        self.db = db
        self.bloom = BloomFilter(expected_codes, false_positive_rate=0.001)
        # Warm the filter from existing DB rows on startup.

    def generate(self, long_url: str, code_length: int = 7) -> str:
        for _ in range(5):
            code = random_base62(code_length)
            if code in self.bloom:
                # Maybe taken — fall through to DB to be sure.
                if not self.db.try_insert(code, long_url):
                    continue
                self.bloom.add(code)
                return code
            # Bloom says absent → no DB read needed for the existence check.
            # Insert is still an upsert (defense-in-depth against false-negatives,
            # which are impossible here, plus race conditions across replicas).
            if self.db.try_insert(code, long_url):
                self.bloom.add(code)
                return code
        raise RuntimeError("code-space too dense; grow code length")
```

For 2 billion expected codes at 0.1% false-positive rate, the Bloom filter is ~3.6 GB — fits comfortably on a modern app server. **For multi-server deployments, use Redis Bloom (`BF.ADD`/`BF.EXISTS`) or a CRDT-replicated Counting Bloom Filter** so all servers share state without the false-negative risk of un-replicated local filters.

For the broader filter family (Cuckoo, Counting, XOR), see [`../../../data-structures/bloom-and-cuckoo-filters.md`](../../../data-structures/bloom-and-cuckoo-filters.md).

### Single-Flight Retry Patterns

When the same long URL is shortened concurrently (twice in the same millisecond), naive retry can mint two codes for one URL. **Single-flight** (Go's `golang.org/x/sync/singleflight` is the canonical implementation) coalesces concurrent calls per-key into a single execution: the first caller runs the function, the rest block on its result.

For URL shorteners specifically, single-flight is most useful at the **idempotency-key layer** — coalesce on a client-supplied `Idempotency-Key` header on `POST /shorten`, not on the URL itself, so the same client retrying the same request always gets the same code, but two different clients shortening the same URL each get their own code.

### Code-Space Exhaustion Planning

Even with random + check, you eventually run out of codes. At 7 chars and 50% fill (1.76 trillion URLs), retry rates start hurting. Triggers for migration:

- Fill density > 30% → start engineering work on 8-char migration.
- Average retries per insert > 1.05 → migration is overdue.
- Code-space age > 5 years → revisit projections regardless.

Migration is non-disruptive if you do it right (see [Migrating to a Longer Code Length](#migrating-to-a-longer-code-length)).

## Allocator Architecture

Whatever generation strategy you pick, the **allocator service** (the process that mints fresh codes) has its own architecture decisions.

### Pull vs Push Allocation

| Pattern | How It Works | Pros | Cons |
|---|---|---|---|
| **Pull** | App server requests a block when its buffer runs low | Simple; back-pressure is automatic | Latency spike on refill if synchronous |
| **Push** | Allocator pre-distributes blocks to known app servers on a schedule | No latency spikes | Coordinator must track all consumers; orphan blocks on rapid scale-down |

Pull dominates in practice because the coordinator is stateless about consumers and recovery is straightforward: a crashed server just abandons its block and the next start-up pulls a fresh one.

### Coordinated Counters — ZooKeeper, etcd, Redis

All three give you atomic increments; they differ on durability, latency, and operational fit:

| System | Latency (p99) | Durability | Operational Cost |
|---|---|---|---|
| **Redis** | < 1 ms | Async (configurable) | Single process; HA needs Sentinel/Cluster |
| **etcd** | 5–20 ms | Strong (Raft) | Complex; right answer for K8s shops already running it |
| **ZooKeeper** | 5–20 ms | Strong (ZAB) | Heaviest; reliable but operationally dated |
| **Postgres** | 1–5 ms | Strong (WAL fsync) | Reuse existing OLTP investment |

For a URL shortener where gaps are tolerable and write rate is moderate (≤10K IDs/sec central, ≤100K with range allocation), **Postgres `SEQUENCE` is hard to beat** — you already have a Postgres cluster, the durability story is good, and `nextval` is fast. Reach for ZooKeeper/etcd only when you need leader-election semantics on top of the counter (e.g., Snowflake-style machine-ID assignment).

### Batched Allocation and the Crash-Mid-Block Trade-off

Range allocation (10K-block per app server) is the universal performance trick. The trade-off is ID-space gaps:

- App server A pulls block `[100_000, 110_000)`.
- App server A serves IDs 100,000 through 105,432 — uses 5,433 IDs.
- App server A crashes. The remaining 4,567 IDs in its block (105,433 through 109,999) **are never minted**.
- App server B pulls the next block `[110_000, 120_000)`.

**For URL shorteners, gaps are fine.** The codes corresponding to skipped integers (e.g., the base62 representation of `105_433`) simply never appear in any URL. The 4,567 lost codes are 0.0000001% of a 3.5-trillion code-space.

**Where it matters:** invoice numbering, ticket IDs, anywhere humans see "but you just had 105432, why is the next one 110000?". For those use cases, use a strict sequence with no caching — and pay the latency cost. For URL codes, embrace the gaps.

A subtle hazard: **don't make block size too large**. A 1M-block on a backend that serves 100 requests before crashing wastes 999,900 IDs. Size the block to match expected lifetime work: `block_size ≈ avg_writes_per_minute × 5`. You refill every five minutes; you lose at most five minutes of IDs on crash.

## Code Length and Unguessability

### Entropy at 6 / 7 / 8 Characters

| Length | Code-Space | Bits of Entropy | Brute-Force Time at 10K probes/sec |
|---|---|---|---|
| 5 | 916 million | 29.8 | 25 hours |
| 6 | 56.8 billion | 35.7 | 65 days |
| 7 | 3.52 trillion | 41.7 | 11 years |
| 8 | 218 trillion | 47.6 | 692 years |
| 10 | 839 quadrillion | 59.5 | 2.6 million years |

For unguessability against a determined attacker rate-limiting at 10K probes/sec, **7 chars is the practical minimum**; 8 is comfortable; 10 is overkill but cheap. Always pair with rate-limiting on `GET /:code` (especially aggressive 404 rate limits) — without rate limits, attackers crawl the whole space.

For a deeper rate-limiting treatment, see the [rate limiter case study](../design-rate-limiter.md).

### Migrating to a Longer Code Length

The good news: **growing code length is non-breaking by construction**. Old codes stay 7 chars and still resolve; new codes are 8 chars. The lookup table (`code → long_url`) doesn't care about length.

Migration steps:

1. **Update the generator** to emit 8-char codes for all new writes. Existing 7-char codes are untouched.
2. **Update the validator regex** on `/:code` to accept 4–10 characters (covers custom aliases too).
3. **Update the data model** if `code` was a fixed-length type — `VARCHAR(16)` already covers it.
4. **Communicate to clients** that codes may now be 8 chars. (Most won't notice.)
5. **Watch the retry-rate metric** drop back to baseline.

The only edge case: **monitoring/alerting based on code length**. Update those dashboards.

You cannot, however, **shrink** code length without re-issuing every code — which breaks every existing link. Don't do this.

## Special-Namespace Reservations

If your shortener domain is `sho.rt/<code>`, you must protect system paths from accidentally being minted as codes. **Always reserve a denylist** before the generator goes live.

| Reserved Pattern | Why |
|---|---|
| `/api`, `/api/*` | API endpoints |
| `/admin`, `/dashboard` | Admin UI |
| `/health`, `/healthz`, `/ready`, `/metrics` | Operational endpoints |
| `/login`, `/logout`, `/signup`, `/auth` | Auth flows |
| `/.well-known/*` | RFC 8615 — ACME, security.txt, OAuth metadata |
| `/robots.txt`, `/sitemap.xml`, `/favicon.ico` | Web standards |
| `/static`, `/assets`, `/public` | Asset serving |
| `/_next`, `/_nuxt`, framework-specific | Next.js, Nuxt internal paths |

Three reservation strategies:

1. **Routing-layer denylist** — Nginx/ingress matches `/api/*` etc. before reaching the redirect handler. Codes that happen to collide are simply never reachable. Simple and safe.
2. **Code-space partition** — auto-generated codes are exactly 7 chars from `[0-9A-Za-z]`; system paths use lowercase keywords; custom aliases are 4–30 chars and may include `-`. The three name-spaces don't overlap.
3. **Reservation table** — a `reserved_codes` table with explicit entries; the generator skips any candidate that exists there. Most flexible; slightly slower path.

In practice, layer 1 + layer 2: routing handles the runtime safety; partitioning prevents the generator from ever minting a collision.

**Banned-word lists** are a separate concern: prevent slurs, brand impersonation (`/google`, `/paypal-login`), and offensive auto-generated codes. The base62 alphabet itself produces occasional accidental words (`/asshole7` is a 7-char base62 string). Maintain a list of banned substrings and reject (or re-generate) on match. This is a moderation cost; budget for it on day one — see [Anti-Patterns](#anti-patterns) on abuse vectors.

## Choosing the Right Strategy — Decision Matrix

| Situation | Recommended Strategy | Why |
|---|---|---|
| Public shortener, billions of URLs, unguessability matters | **Random + Bloom-fronted check** | Simple, unguessable, sub-1% retry at <30% fill |
| Internal service, sequential codes acceptable | **Counter + base62 with range allocation** | Smallest codes, no coordinator on hot path |
| Multi-region active-active writes | **Snowflake-style or UUIDv7** | No central counter; machine_id provides regional dispersion |
| Need idempotent shortening (same URL → same code) | **Hash + truncate with collision detection + salt** | Determinism is the point; accept the salt-management cost |
| Tiny scale (< 10K URLs) | **Anything; use random + check for simplicity** | Overhead doesn't matter |
| Codes will be read aloud or printed | **Random + check, base58 alphabet** | Drop visually-ambiguous glyphs |
| Custom aliases dominate (~50%+ of writes) | **Counter for the rest, namespace-partitioned** | Aliases bypass generation; counter for the auto-codes |

A single-line summary: **use random + collision check unless you have a specific reason not to**. It's the only strategy that gets unguessability for free, scales gracefully past 1B URLs, and tolerates a coordinator failure without losing IDs.

## Real-World Implementations

| Service | Strategy (per public statements) | Notes |
|---|---|---|
| **Bit.ly** | Counter-based with base62 (historically); modern stack uses Snowflake-style IDs | Originally MySQL `AUTO_INCREMENT` + base62; rebuilt for multi-region |
| **TinyURL** | Random + collision check | 6–8 char codes, alphanumeric |
| **Twitter t.co** | Snowflake-derived | Same allocator family as tweet IDs |
| **YouTube video IDs** | Random base64URL, 11 chars | `8.4 × 10^19` code-space; effectively un-collidable |
| **Google goo.gl** (retired 2018) | Random + check | Standard pattern |
| **Discord** | Snowflake (modified) | 64-bit IDs across snowflake-style allocator |
| **Instagram media IDs** | Sharded Snowflake variant | Encodes shard ID for routing |
| **Stripe object IDs** | Random base62 with object-type prefix (`ch_`, `cus_`) | Prefix conveys type at a glance; random body |

Both [Snowflake's bit layout](https://github.com/twitter-archive/snowflake/tree/snowflake-2010) and Sonyflake's deviation from it are public. Instagram's writeup describes the full sharded-ID design.

## Anti-Patterns

- **`MD5(long_url)[:7]` without collision detection.** Birthday-paradox math is unforgiving — at 1B URLs in 3.5T space, collisions are mathematically certain and the first one silently corrupts a mapping.
- **Raw `AUTO_INCREMENT` PK as the public code.** Sequential codes let an attacker crawl your entire link graph by enumerating `1, 2, 3, ...`.
- **Truncating a Snowflake ID and assuming uniqueness survives.** The dropped high bits are precisely the timestamp ones that prevent collisions. Use the full ID or a different generator.
- **Postgres `SEQUENCE` without `CACHE` at high write rate.** Every `nextval` is a row lock; throughput collapses past a few thousand IDs/sec.
- **Block size of 1 in range allocation** (recreates centralized allocation) **or block size of 1M on a flapping server** (each restart burns ~500K IDs).
- **Mixing random and counter-based codes in one space.** Now you can't tell which is which; debugging gets weird. Pick one strategy or partition by length/alphabet.
- **Forgetting to reserve `/api`, `/admin`, `/health`.** A new code happens to be `admin1`; admin route now resolves a redirect.
- **Reusing Snowflake machine IDs after a crash.** Coordinate via ZooKeeper/etcd or derive from a unique stable property (IP, pod name).
- **Using `random.choice` / `math/rand` for code generation.** Predictable RNGs let an attacker guess the next code from observed ones. Use `secrets` / `crypto/rand`.
- **Not monitoring fill density.** Alert at 30% fill; migrate to longer codes at 40% — before retry rates climb and write tail latency explodes.

## Related

- Parent case study — [`../design-url-shortener.md`](../design-url-shortener.md)
- Sharding the mappings table — [`../../../scalability/sharding-strategies.md`](../../../scalability/sharding-strategies.md)
- Consistent hashing for shard placement — [`../../../data-structures/consistent-and-rendezvous-hashing.md`](../../../data-structures/consistent-and-rendezvous-hashing.md)
- Bloom and Cuckoo filters for collision short-circuit — [`../../../data-structures/bloom-and-cuckoo-filters.md`](../../../data-structures/bloom-and-cuckoo-filters.md)
- Capacity-math methodology — [`../../foundations/back-of-envelope-estimation.md`](../../foundations/back-of-envelope-estimation.md)
- Rate limiting brute-force code probes — [`../design-rate-limiter.md`](../design-rate-limiter.md)

## References

- [Twitter Snowflake (archived 2010 release)](https://github.com/twitter-archive/snowflake/tree/snowflake-2010) — original 41-bit timestamp / 10-bit machine / 12-bit sequence layout, the canonical distributed ID generator.
- [RFC 9562 — Universally Unique IDentifiers (UUIDs)](https://www.rfc-editor.org/rfc/rfc9562) — IETF standard defining UUIDv7 (48-bit Unix timestamp + 74 random bits) as the modern, sortable, time-ordered UUID format.
- [Sony Sonyflake](https://github.com/sony/sonyflake) — Snowflake variant with 39-bit time / 8-bit sequence / 16-bit machine; trades throughput for longer epoch and more machines.
- [Sharding & IDs at Instagram (Instagram Engineering)](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c) — encodes logical shard ID into the ID itself for application-layer routing.
- [PostgreSQL — `CREATE SEQUENCE`](https://www.postgresql.org/docs/current/sql-createsequence.html) — official documentation including `CACHE` semantics, gap behavior, and concurrency guarantees.
- [Redis — `INCR` documentation](https://redis.io/docs/latest/commands/incr/) — atomic counter primitive, the foundation of Redis-based ID allocators; includes counter-pattern guidance.
- [Wikipedia — Birthday Problem](https://en.wikipedia.org/wiki/Birthday_problem) — formal treatment of the collision-probability math used in [Strategy 3](#strategy-3--hash--truncate-pitfalls).
- [Lamping & Veach — A Fast, Minimal Memory, Consistent Hash Algorithm (2014)](https://arxiv.org/abs/1406.2294) — Jump consistent hash, relevant when sharding the mapping table by code-derived bucket number.
