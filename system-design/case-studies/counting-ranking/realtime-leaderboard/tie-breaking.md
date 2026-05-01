---
title: "Tie-Breaking — Composite Score Encoding for Stable Leaderboard Ordering"
date: 2026-05-01
updated: 2026-05-01
tags: [system-design, deep-dive, leaderboard, ordering, encoding]
---

# Tie-Breaking — Composite Score Encoding for Stable Leaderboard Ordering

**Date:** 2026-05-01 | **Updated:** 2026-05-01
**Tags:** `system-design` `deep-dive` `leaderboard` `ordering` `encoding`

## Table of Contents

- [Summary](#summary)
- [Why Tie-Breaking Is Load-Bearing](#why-tie-breaking-is-load-bearing)
- [The Default Redis Tiebreak — Lexicographic Member Name](#the-default-redis-tiebreak--lexicographic-member-name)
- [The Flicker Problem — When Ranks Won't Stay Still](#the-flicker-problem--when-ranks-wont-stay-still)
- [Composite-Score Encoding — One Double, Many Fields](#composite-score-encoding--one-double-many-fields)
  - [The IEEE 754 Capacity Budget](#the-ieee-754-capacity-budget)
  - [The Standard Pattern — `score * 1e10 + (MAX_TS - achieved_at)`](#the-standard-pattern--score--1e10--max_ts---achieved_at)
  - [Bit Budget Worked Example](#bit-budget-worked-example)
  - [Encoding and Decoding](#encoding-and-decoding)
- [Tie-Break Field Choices](#tie-break-field-choices)
- [Encoding Pitfalls — Float Precision and Integer-Only Encoding](#encoding-pitfalls--float-precision-and-integer-only-encoding)
- [Stable Rank — What "Stable" Actually Means](#stable-rank--what-stable-actually-means)
- [ZRANGEBYLEX — A Parallel Approach for Fixed-Score Sets](#zrangebylex--a-parallel-approach-for-fixed-score-sets)
- [Alternative Architectures](#alternative-architectures)
  - [Parallel ZSETs per Tier](#parallel-zsets-per-tier)
  - [Composite Member Name with Rank Suffix](#composite-member-name-with-rank-suffix)
  - [Lua-Side Resolution](#lua-side-resolution)
- [Multi-Criteria Leaderboards — Score then K/D Ratio](#multi-criteria-leaderboards--score-then-kd-ratio)
- [Game-Design Implications — Hidden vs Visible Tiebreakers](#game-design-implications--hidden-vs-visible-tiebreakers)
- [Frontend Sorting — Never Sort by Score Alone](#frontend-sorting--never-sort-by-score-alone)
- [Worked End-to-End Example](#worked-end-to-end-example)
- [Operational Concerns](#operational-concerns)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

Two players with the same score are not equal in the eyes of the player who is ranked #2 instead of #1. The default Redis ZSET tiebreak — lexicographic comparison of member names — is meaningless to a player and produces "alphabetic justice" outcomes that are indefensible from a product perspective. The standard fix is **composite-score encoding**: pack the primary score and a deterministic secondary criterion (most often a timestamp; sometimes level, time spent, region, or an integer rank seed) into a single IEEE 754 double, so the ZSET's native ordering produces the right answer without any client-side sorting. The encoding is constrained by the double's 53-bit mantissa: capacity is `2^53 ≈ 9 × 10^15`, and you have to carve out fields with care or the low-order bits silently disappear. The dominant pattern is `score * 1e10 + (MAX_TIMESTAMP - achieved_at)` — earlier achievement wins ties — and the dominant failure mode is using `score * 1e11` with a billion-point base score and watching the tiebreaker get truncated. Alternatives include parallel ZSETs per tier, composite member names paired with `ZRANGEBYLEX`, and Lua-side resolution. Stable rank — your rank changes only when *your* score moves, never when someone else does — is the property players want and the property a well-designed encoding gives them. Anti-patterns: ignoring ties (visible flicker), random-seed tiebreaks (non-stable), member-name tiebreaks (lexicographic surprises), and frontend resort that disagrees with the server (every cache layer becomes a bug).

## Why Tie-Breaking Is Load-Bearing

A leaderboard's job is to answer one question consistently: *who is ranked above whom*. If the ranking can change without the underlying competitive event changing — without anyone scoring or losing points — players notice immediately. They notice it as flicker, as cheating accusations against the platform, and as support tickets ("I went from #3 to #5 and I haven't played in two hours, what happened?").

Ties happen far more often than the naive intuition suggests:

- **Whole-number scores in a game with a small dynamic range** (e.g., kill count in a 30-minute match, score capped at a daily quest's max value) produce ties at every popular score level.
- **Coarse score increments** (e.g., +100 per win, +250 per perfect run) make ties the norm at the boundary, not the exception.
- **Tournament cutoffs** at the prize tier are the highest-stakes ties — the difference between "made the top 100" and "didn't" is whether your rank is 100 or 101, and ties in that neighborhood determine prize money.
- **Top-of-leaderboard ties** between #1 and #2 are the most visible, because the front page shows them.

The leaderboard architecture lives in [`../design-realtime-leaderboard.md`](../design-realtime-leaderboard.md). The Redis sorted set internals — skip list ordered by score then member, hash table on member — live in [`./redis-sorted-set-internals.md`](./redis-sorted-set-internals.md) (planned). This document is about the contract surface between "two players, same score" and "the visible rank we show them."

The discipline that scales: **the score field is not a number. It is an encoding.** What looks like a `double` is actually a packed tuple. Once you accept that, every other decision in this doc is bookkeeping.

## The Default Redis Tiebreak — Lexicographic Member Name

A Redis ZSET orders by `(score, member)`:

- **Primary key:** the score (a double).
- **Secondary key:** the member string, compared byte-by-byte lexicographically.

Documented at [https://redis.io/commands/zadd/](https://redis.io/commands/zadd/) under the data type semantics. The skip list stores `(score, member)` pairs; ties on score are resolved by `memcmp` on the member.

This means, with two players named `alice` and `bob` both at score 1000:

```text
ZRANGE leaderboard 0 -1 WITHSCORES
1) "alice"
2) "1000"
3) "bob"
4) "1000"
```

`alice` is ranked above `bob` purely because `'a' < 'b'`. If your member is a UUID or numeric user ID, the tiebreak is whatever lexicographic ordering of the ID string falls out — almost certainly not what you want. Player `00000001` always wins ties against player `99999999`, regardless of when, how, or in what context they scored. This is "alphabetic justice" and it is a product failure.

## The Flicker Problem — When Ranks Won't Stay Still

A subtler symptom of weak tiebreaks: **rank flicker**. Two players are at exactly the same score; the ZSET's internal ordering of them is fixed (lexicographic on member), but as nearby players' scores fluctuate the rank numbers reported by `ZREVRANK` swap repeatedly when only one of them updates a session-cached position.

The pathological scenario:

1. Alice and Bob both at 1000 points. Alice ranks #5, Bob #6 (member-name tiebreak).
2. A third player, Carol, sitting at 1000 points just behind them at #7, scores +1 and moves to 1001.
3. Now the front-page UI for Alice's friend list refreshes: her cached rank was #5, but the slow path computes #5 still. No change. Good.
4. But Bob's friend list, cached three minutes ago at #6, now shows him as #6 still. Also no change. Good.

So far so fine. The real flicker shows up when the scores are extremely close, the leaderboard is consulted from multiple cache layers, and the underlying tiebreak is non-deterministic — for example, when a frontend "resorts" by score alone and JavaScript's `Array.sort` is not guaranteed stable across all browsers (it is in modern V8 / SpiderMonkey, but was not historically in Chrome). Every refresh re-sorts; every re-sort can swap two equal-score entries; each swap causes a visible rank change for both. The user sees their rank go 5 → 6 → 5 → 6 every few seconds, even though nothing has happened in the game.

The root cause is always the same: **two equal scores, no deterministic tiebreaker carried with the data**. Either the server's tiebreak is meaningless (lexicographic), or the client is re-deciding the tiebreak with whatever local state it has. The fix is to make the tiebreak part of the score and never re-decide it.

## Composite-Score Encoding — One Double, Many Fields

### The IEEE 754 Capacity Budget

Redis ZSET scores are double-precision floats — IEEE 754 binary64. The format has:

- **1 sign bit**
- **11 exponent bits**
- **52 mantissa bits** (with an implicit leading 1, giving 53 bits of significand)

This gives integer precision up to `2^53 = 9_007_199_254_740_992` — about `9 × 10^15`, or 16 decimal digits of safe integer representation. Beyond that, integers cannot all be represented exactly: `2^53 + 1` rounds to `2^53`, and the gap between representable values doubles every power of two. See the IEEE 754 reference at [https://en.wikipedia.org/wiki/Double-precision_floating-point_format](https://en.wikipedia.org/wiki/Double-precision_floating-point_format).

The 16-digit budget is the entire bit budget you have for composite encoding. Every digit you give to the primary score is a digit you cannot give to the tiebreaker, and vice versa.

### The Standard Pattern — `score * 1e10 + (MAX_TS - achieved_at)`

The dominant tie-breaking pattern in production leaderboards:

```text
composite = base_score * 10^k  +  (MAX_TIMESTAMP - achieved_at)
```

where:

- `base_score` is the player's actual score, an integer.
- `k` is a shift large enough to "make room" for the tiebreaker without colliding with the base score.
- `achieved_at` is a Unix timestamp (in seconds) at which the player reached this score.
- `MAX_TIMESTAMP` is a constant chosen larger than any plausible `achieved_at`, so `(MAX_TS - achieved_at)` is always positive — and crucially, **earlier `achieved_at` produces a larger tiebreaker**, so earlier achievement wins ties.

The intuition: the high-order portion is the score (the dominant ordering), the low-order portion is a "freshness" reward where earlier-achieved-the-score gets a bump. Two players at base score 1000: whichever crossed 1000 first has a slightly higher composite score, so they outrank the later achiever. This matches the player-intuitive rule: "you can't lose your rank just because someone else tied you."

### Bit Budget Worked Example

Suppose the game has:

- Maximum base score: `10^9` (one billion points; a generous cap for almost any game).
- Tiebreaker resolution: seconds since 2020-01-01 UTC, comfortably 31 bits, encodable in 10 decimal digits.
- `MAX_TIMESTAMP = 10^10` (good through year ~2286 in seconds-since-1970).

Then `composite = score * 10^10 + (10^10 - achieved_at)`.

- Maximum composite value: `10^9 * 10^10 + 10^10 = 10^19 + 10^10 ≈ 10^19`.
- Safe integer budget: `9 × 10^15`.

**This overflows.** `10^19` is far past `2^53`. The low-order digits (the tiebreaker portion) will silently lose precision. Two players who scored at slightly different times will be reported as tied because the timestamp difference falls below the representable resolution at that magnitude.

The fix is to shrink one or both fields:

- **Cap base score at `10^6`** (a million; still ample for most games), shift by `10^9`, tiebreaker is 9 digits → composite max `10^15`, fits.
- **Cap base score at `10^7`**, shift by `10^8`, tiebreaker is 8 digits (covers `10^8` seconds ≈ 3 years) → composite max `10^15`, fits.
- **Use a coarser timestamp** — e.g., minutes instead of seconds: timestamp range `10^7` for ~190 years, shift by `10^7`, base score up to `10^8` → composite max `10^15`, fits.

The discipline: **before you ship, verify with extreme cases**. Compute the max composite, run it through a `double` round-trip, and compare to the original integer. If they don't match exactly, you have a precision bug waiting to surface in the prize tier.

### Encoding and Decoding

```python
# Composite score encoding for Redis ZSET tiebreaking.
# Budget: score in [0, 10^7], timestamp in [0, 10^8 seconds]
# Composite max: 10^7 * 10^8 + 10^8 = 10^15, well within 2^53 = 9e15.

SCORE_SHIFT = 10**8         # bits reserved for the tiebreaker portion
TIEBREAK_MAX = 10**8        # MAX_TIMESTAMP for the inversion trick
EPOCH_OFFSET = 1577836800   # 2020-01-01 UTC; subtract to keep timestamps small


def encode_composite(score: int, achieved_at: int) -> int:
    """Pack (score, achieved_at) into a single integer-valued double.

    Earlier achieved_at wins ties (higher composite for same base score).
    Returned int fits exactly in IEEE 754 binary64.
    """
    if not (0 <= score < 10**7):
        raise ValueError(f"score out of budget: {score}")
    relative_ts = achieved_at - EPOCH_OFFSET
    if not (0 <= relative_ts < TIEBREAK_MAX):
        raise ValueError(f"timestamp out of budget: {achieved_at}")
    tiebreak = TIEBREAK_MAX - relative_ts
    composite = score * SCORE_SHIFT + tiebreak

    # Verify no precision loss before we hand it to Redis.
    assert int(float(composite)) == composite, "precision loss"
    return composite


def decode_composite(composite: int) -> tuple[int, int]:
    """Recover (score, achieved_at) from a composite encoded by encode_composite."""
    score = composite // SCORE_SHIFT
    tiebreak = composite % SCORE_SHIFT
    relative_ts = TIEBREAK_MAX - tiebreak
    achieved_at = relative_ts + EPOCH_OFFSET
    return score, achieved_at
```

`ZADD` then takes the composite directly:

```python
import redis

r = redis.Redis()

def submit_score(player_id: str, score: int, achieved_at: int) -> None:
    composite = encode_composite(score, achieved_at)
    # ZADD with GT: only update if the new composite is strictly greater
    # than the player's existing composite. This preserves the "earlier
    # achievement of a given score wins" invariant when a later resubmission
    # of the same base score arrives.
    r.zadd("leaderboard:season42", {player_id: composite}, gt=True)


def get_top_n(n: int) -> list[tuple[str, int, int]]:
    raw = r.zrevrange("leaderboard:season42", 0, n - 1, withscores=True)
    out = []
    for member, composite in raw:
        score, ts = decode_composite(int(composite))
        out.append((member.decode(), score, ts))
    return out
```

The `ZADD ... GT` flag (documented at [https://redis.io/commands/zadd/](https://redis.io/commands/zadd/)) is essential here: without it, a second submission of the same base score with a later timestamp would *replace* the original and the player would lose their tie-breaker advantage. With `GT`, the score only moves up. For "lower is better" leaderboards (e.g., golf, time trials), use `LT` instead and flip the encoding so smaller scores are smaller composites.

Decoding on the server side is trivial; decoding on the client is also trivial but is **not** the same operation as comparing for ranking. The client should never re-rank; it should display the rank the server provides.

## Tie-Break Field Choices

The encoding pattern is general; the field choice is a product decision. Common tiebreakers:

| Tiebreaker | When it makes sense | Encoding shape |
|---|---|---|
| **Earliest timestamp** | "First to the score wins" — most leaderboards, races, speedruns. | `MAX_TS - achieved_at` (invert so earlier is larger). |
| **Latest timestamp** | "Most recently active" — stale-decaying boards, daily resets. | `achieved_at` directly (no inversion). |
| **Player level** | Higher-level players outrank lower-level at same score (XP-tied). | `level` directly, in low bits. |
| **Time spent** | Less play time wins (efficiency-rewarding) — speedrun leaderboards, perfect-clear puzzles. | `MAX_DURATION - duration_seconds`. |
| **Region preference** | Show same-region players first to a same-region viewer (decided per-query, not encoded). | Not encoded; applied at read time. |
| **Integer rank seed** | Deterministic but otherwise-arbitrary tiebreak (e.g., randomly-assigned per-season seed). | `seed` directly; never re-rolls within a season. |
| **K/D ratio, accuracy, win rate** | Multi-criteria leaderboards (see below). | Quantized to integer ppm, packed in low bits. |

The non-negotiable property: the tiebreaker must be **deterministic and stable** over time for a given player-event. A random-each-query seed produces the same flicker as no tiebreaker at all. If you genuinely want tied-score players to be displayed in a randomized order to avoid favoring "earlier" players, the right move is to randomize the *display only*, never the underlying rank. The rank stays stable; only the rendered seat-row order shuffles.

## Encoding Pitfalls — Float Precision and Integer-Only Encoding

Three classes of precision pitfall to watch for:

1. **Composite overflow past `2^53`.** The most common bug. Multiplying score by an over-large shift causes silent truncation in the low bits. Always compute the worst-case composite at the bit-budget design phase, run it through `int(float(x))`, and assert exact round-trip.

2. **Non-integer scores.** If your base score has fractional points (e.g., 1234.5), the encoding has to scale it to integer first: `composite = round(score * 100) * SHIFT + tiebreak`. Once scaled, the same overflow analysis applies, plus the additional risk that floats accumulated by upstream summing (e.g., `score += 0.1` repeated) drift away from the true value. **Keep all upstream score arithmetic in integers.** Floating point is for the *encoded composite*, not for the running totals that feed it.

3. **Subtraction order on the tiebreaker.** `MAX_TS - achieved_at` is correct (earlier is larger); `achieved_at - MIN_TS` is wrong for "earliest wins" because larger means later. The compiler will not catch the inversion; only the test that asserts "Alice scored 1000 at t=10, Bob at t=20, Alice ranks first" will. Write that test.

The core defense is to write the encode/decode pair as a tested utility module, with tests that:

- Round-trip: `decode(encode(s, t)) == (s, t)` for all values across the budget.
- Order: `s1 == s2 and t1 < t2  →  encode(s1, t1) > encode(s2, t2)`.
- Boundary: `encode(MAX_SCORE, MIN_TS)` and `encode(0, MAX_TS)` both round-trip exactly.
- No fractional creep: `encode(s, t)` stored to Redis (a double), read back, decoded — equal to `(s, t)` exactly.

## Stable Rank — What "Stable" Actually Means

A leaderboard's rank is **stable** if a player's rank changes if and only if their *own* composite changes. Other players' submissions can move their relative position only by overtaking or being overtaken — never by silently swapping with a tied score.

Composite-score encoding gives this property by construction:

- Two players at the same base score have **different** composites (because their `achieved_at` differs by at least one second, or because the tiebreak is some other unique field).
- The ZSET's `(score, member)` ordering is total — there is exactly one valid order.
- Querying `ZREVRANK` for a given member is a deterministic function of the current ZSET state.
- Adding a new player at a different composite does not change the relative order of existing players.

Without composite encoding, "stability" is a property of the member-name tiebreak, which is meaningless to the player but at least deterministic to the server. The flicker scenarios all involve a *different* tiebreak being re-decided on the client. Once the encoding is in place and the client trusts the server's rank verbatim, flicker disappears.

This is the same property a sorted database query has when the `ORDER BY` clause is total — i.e., when the trailing key is unique. Anyone who has debugged a "non-deterministic pagination" bug in SQL has seen the relational version of this problem; composite encoding is the ZSET version.

## ZRANGEBYLEX — A Parallel Approach for Fixed-Score Sets

Redis offers `ZRANGEBYLEX` ([https://redis.io/commands/zrangebylex/](https://redis.io/commands/zrangebylex/)), which orders by member name *only when all scores are equal*. This is occasionally useful as a parallel structure for "tied-score sub-board" displays.

The pattern: maintain a separate ZSET per discrete score tier, with all members at score 0 and the tiebreak encoded in the member name itself. Then `ZRANGEBYLEX` walks the lexicographic order, which — if the member name is a deterministic encoding — gives the right tiebreak.

```python
def submit_to_tier(score_tier: int, achieved_at: int, player_id: str) -> None:
    # Member name = "<inverted_ts_zero_padded>:<player_id>"
    # ZRANGEBYLEX walks in lex order, so smaller inverted_ts (= earlier
    # achieved_at) sorts first.
    inv_ts = TIEBREAK_MAX - achieved_at
    member = f"{inv_ts:010d}:{player_id}"
    r.zadd(f"tier:{score_tier}", {member: 0})


def list_tier(score_tier: int, limit: int) -> list[str]:
    raw = r.zrangebylex(f"tier:{score_tier}", b"-", b"+", start=0, num=limit)
    return [m.decode().split(":", 1)[1] for m in raw]
```

This is a niche technique. It's useful when:

- The game has discrete score *tiers* (gold/silver/bronze, or tournament cutoffs) and you want a separate sub-board per tier.
- You want to display "all players who reached this score, in order of who got there first" as a separate view.
- You don't want to deal with composite-encoding bit budgets at all — the tiebreak is the entire member name, and there's no double-precision concern.

It is **not** the general-purpose fix. The composite-score encoding is. `ZRANGEBYLEX` is a complement, not a replacement.

## Alternative Architectures

### Parallel ZSETs per Tier

For very high-precision games where the bit budget genuinely doesn't fit, maintain two ZSETs:

- `lb:score` — primary score only.
- `lb:tiebreak` — tiebreak field only.

Reads do `ZREVRANGEBYSCORE` on the first to find the band of tied players, then `ZRANGE`/`HMGET` the tiebreak field to order within that band. This is `O(log N + tied_count)` per query but trivially correct — there is no double-precision concern at all because the two fields never share a number.

Trade-off: two-key updates. Without Lua or `MULTI`, the two ZSETs can drift mid-update (player visible at one score in `lb:score` but with stale tiebreak). For most games this is tolerable; for prize-tier-sensitive boards, wrap the update in a Lua script:

```lua
-- KEYS[1] = lb:score, KEYS[2] = lb:tiebreak
-- ARGV[1] = player_id, ARGV[2] = new_score, ARGV[3] = new_tiebreak

local existing = redis.call("ZSCORE", KEYS[1], ARGV[1])
if existing and tonumber(existing) >= tonumber(ARGV[2]) then
    return 0  -- not an improvement
end
redis.call("ZADD", KEYS[1],    ARGV[2], ARGV[1])
redis.call("ZADD", KEYS[2], ARGV[3], ARGV[1])
return 1
```

### Composite Member Name with Rank Suffix

For boards where the score is naturally bounded and discrete (e.g., trophy count, capped at 5000), encode `score:tiebreak:player_id` in the member and put all members at score 0. Then `ZRANGEBYLEX` does the entire ordering job. This is a slight generalization of the tier pattern above and works well for slow-changing boards with simple, integer-valued primary scores.

### Lua-Side Resolution

If the encoding is too complex to fit in a double — e.g., the game has three tiebreak fields (score, then K/D, then time-spent) and the budget for all four is genuinely 20+ digits — store the fields in a `HASH` keyed by player and walk the ZSET via Lua, resolving ties in-script. This is `O(tied_count)` per Lua call and not friendly to `ZREVRANGEBYSCORE` paginated queries; the right move is usually to compress the secondary fields (quantize K/D to integer ppm, etc.) until the budget fits, rather than push complexity into Lua.

## Multi-Criteria Leaderboards — Score then K/D Ratio

Imagine a competitive shooter leaderboard ordered first by total kills (the primary score) and second by K/D ratio for tiebreaking — and within the same K/D, by earliest achievement.

Three options:

1. **Quantize K/D and pack into the composite.** K/D is a real number, but ppm (parts per million) is an integer in `[0, 10^7]` covering the meaningful range up to a 10:1 ratio. Encoding: `composite = kills * 10^14 + kd_ppm * 10^7 + (10^7 - day_index)`. Verify the bit budget: max kills, say `10^4`, gives `10^4 * 10^14 = 10^18` — overflow. Cap kills at `10^2` (per-day kill count, capped at 100), then `10^2 * 10^14 = 10^16`, still over `9 × 10^15`. We need to shrink fields further: `kills * 10^13 + kd_ppm * 10^6 + (10^6 - hour_index)`, max `10^2 * 10^13 = 10^15`, fits. The arithmetic is the same; only the field sizes change.

2. **Tuple comparison via two-ZSET architecture.** Read the K/D from a hash on each `ZSCORE` lookup; compare tuples on the client. Simpler to implement, more network round-trips. Acceptable for small top-N queries.

3. **Lua-side comparator.** Same script-based approach as the tiebreak ZSET above, generalized to N fields. Most flexible, hardest to scale.

In practice, option 1 is the default for two-field tiebreaks; option 2 for three or more fields where bit budget fights back; option 3 for one-off audit queries that don't need to be fast.

## Game-Design Implications — Hidden vs Visible Tiebreakers

A subtle product decision: should the player **see** the tiebreaker?

- **Hidden tiebreaker:** the leaderboard shows just the score and rank. Two players at score 1000 see different ranks (#5 and #6) without any visible explanation. Players sometimes feel cheated ("we have the same score, why am I behind?").
- **Visible tiebreaker:** the leaderboard shows score + tiebreaker (e.g., "1000 pts · achieved 3 days ago"). Players understand the ordering.

The visible-tiebreaker UX is generally better; it makes the rank feel earned rather than arbitrary. The hidden-tiebreaker UX is sometimes preferred when the tiebreaker would expose information the game doesn't want shared (e.g., absolute timestamps could correlate with player play schedules in a privacy-sensitive way).

Steamworks and Apple Game Center both expose timestamp tiebreakers in their leaderboard APIs (see [https://partner.steamgames.com/doc/features/leaderboards](https://partner.steamgames.com/doc/features/leaderboards) and [https://developer.apple.com/documentation/gamekit/gkleaderboard](https://developer.apple.com/documentation/gamekit/gkleaderboard)). The platform default is visible-tiebreaker; the game can choose to hide it in the rendering layer.

The architectural rule: **encode visibly, render selectively**. The composite is a server-side concern; what the client renders is a UX concern. Don't conflate the two by trying to decide visibility at encoding time.

## Frontend Sorting — Never Sort by Score Alone

A class of bug that recurs in every team that ships a leaderboard: the frontend has a player list, sorts it client-side by `score` descending, and hands the result to the renderer. If the encoding is composite, this sort is **wrong** — it ignores the tiebreaker, and the displayed order disagrees with the server's authoritative `ZREVRANK`.

Symptoms:

- The cached page shows Alice at #5 and Bob at #6.
- The user filters/refreshes and the client re-sorts; Alice and Bob are now displayed as #6 and #5 (the JS sort, even when stable, may produce a different stable order than Redis's `(composite, member)` ordering, because the client only has `score`, not `composite`).
- The user's friend page shows Alice at #5 (server-rendered).
- The user accuses the platform of cheating.

The fix is one of:

- **Carry the rank with the row.** The server sends `{ player_id, score, rank, achieved_at }`; the client renders by `rank` and never re-sorts. The score is for display only.
- **Carry the composite with the row.** The client can re-sort if needed, but only on the composite. Never on the score alone. Be aware that JavaScript can't represent `2^53` precisely as a number for very large composites — pass the composite as a string and compare as a `BigInt` if so.
- **Re-fetch on filter change** instead of client-sorting. The server is the source of truth; round-trip rather than reorder.

The general principle: **the server's ordering is authoritative; the client never invents one**. This applies equally to leaderboards, search result lists, and any other server-paginated ordered data.

## Worked End-to-End Example

A complete flow for a tournament leaderboard with composite encoding:

```python
import time
import redis

r = redis.Redis(decode_responses=False)

# Bit budget: score in [0, 10^7], timestamp in [0, 10^8 seconds since 2020].
SCORE_SHIFT = 10**8
TIEBREAK_MAX = 10**8
EPOCH_OFFSET = 1577836800  # 2020-01-01 UTC


def encode(score: int, achieved_at: int) -> int:
    rel = achieved_at - EPOCH_OFFSET
    return score * SCORE_SHIFT + (TIEBREAK_MAX - rel)


def decode(composite: int) -> tuple[int, int]:
    score = composite // SCORE_SHIFT
    tiebreak = composite % SCORE_SHIFT
    return score, (TIEBREAK_MAX - tiebreak) + EPOCH_OFFSET


# 1. Submission path: client posts a score; server records it idempotently.
def submit(tournament_id: str, player_id: str, score: int) -> int:
    """Returns the player's new rank (1-indexed)."""
    now = int(time.time())
    composite = encode(score, now)
    key = f"tourney:{tournament_id}:lb"

    # GT ensures we never lower a player's composite. If a later resubmit
    # ties the same base score, the earlier (existing) achievement wins
    # because the existing composite is larger (smaller tiebreak subtraction).
    r.zadd(key, {player_id: composite}, gt=True)

    # ZREVRANK returns 0-indexed; +1 for human display.
    raw_rank = r.zrevrank(key, player_id)
    return raw_rank + 1 if raw_rank is not None else -1


# 2. Read path: top-N for the front page.
def top_n(tournament_id: str, n: int = 100) -> list[dict]:
    key = f"tourney:{tournament_id}:lb"
    raw = r.zrevrange(key, 0, n - 1, withscores=True)
    out = []
    for i, (member, composite) in enumerate(raw, start=1):
        score, ts = decode(int(composite))
        out.append({
            "rank": i,
            "player_id": member.decode(),
            "score": score,
            "achieved_at": ts,
        })
    return out


# 3. Read path: a single player's neighborhood.
def around_player(tournament_id: str, player_id: str, radius: int = 5) -> list[dict]:
    key = f"tourney:{tournament_id}:lb"
    rank = r.zrevrank(key, player_id)
    if rank is None:
        return []
    start = max(0, rank - radius)
    end = rank + radius
    raw = r.zrevrange(key, start, end, withscores=True)
    out = []
    for i, (member, composite) in enumerate(raw):
        score, ts = decode(int(composite))
        out.append({
            "rank": start + i + 1,
            "player_id": member.decode(),
            "score": score,
            "achieved_at": ts,
        })
    return out
```

A scenario that exercises tie-breaking:

```text
t=100: Alice submits score=1000 → composite = 10^11 + (10^8 - (100 - epoch_offset))
t=200: Bob   submits score=1000 → composite = 10^11 + (10^8 - (200 - epoch_offset))

Alice's composite is *larger* (smaller subtrahend), so Alice ranks above Bob.

t=300: Carol submits score=1001 → composite = 1.001 * 10^11 + ...

Carol's composite is much larger, so Carol ranks above both. Alice and Bob's
relative order is unchanged.

t=400: Alice resubmits score=1000 → encode produces a smaller composite
       (later achieved_at), but ZADD GT rejects the update. Alice keeps her
       earlier composite and stays ahead of Bob.
```

The flicker scenario does not occur: Alice and Bob always have distinct composites; their relative order is fixed at the moment they each first achieved the score; no later event can swap them unless one of them genuinely improves their score.

## Operational Concerns

- **Schema migrations.** If you ship encoding v1 with `SCORE_SHIFT = 10^8` and discover you need `10^10`, you cannot just change the constant — every existing composite in Redis is wrong. Plan a migration: write a tool that scans the ZSET, decodes with v1, re-encodes with v2, and rewrites in a single Lua transaction per player. Run it during a maintenance window or with double-write during a transition.
- **Clock skew across submitters.** If multiple application servers timestamp submissions, their clocks must agree to within the tiebreak resolution (e.g., 1 second). NTP is sufficient for second-resolution tiebreaks; for sub-second, use a centralized sequencer (the same pattern as in matching engines — see the parallel discussion in [`./tournament-mode.md`](./tournament-mode.md), planned).
- **Out-of-order submissions.** A score submission delayed by network/queue may arrive *after* a later submission. With `ZADD GT`, this is harmless — the later (already-applied) composite is larger, so the late-arriving one is rejected. Without `GT`, you'd overwrite. Always use `GT` (or `LT` for inverse boards).
- **Daily resets and decay.** A "daily best score" board needs a daily reset. The composite encoding is per-board; resetting means deleting the ZSET and starting fresh. Decaying boards (where scores decrease over time) are harder — see [`./top-k-queries.md`](./top-k-queries.md) (planned) for the time-window aggregation pattern.
- **Audit and replay.** Score submissions should be journaled to Kafka or equivalent before they hit Redis, so the leaderboard is rebuildable. The journal stores `(player_id, raw_score, achieved_at)` — *not* the composite. The composite is a Redis-side concern; the source of truth is the raw fields.

## Anti-Patterns

1. **Ignoring ties.** "We'll just use the score; the chance of ties is low." It isn't — discrete scoring rules produce ties at every popular boundary. Players notice flicker immediately. Encode the tiebreak from day one.
2. **Random-seed tiebreaker.** Generating a per-submission random nonce as the tiebreak seems clever but is non-stable: two players who tied today may swap tomorrow if you re-roll. Players see this as the platform "deciding" to reorder them. The seed must be deterministic and persist with the score.
3. **Member-name tiebreaker.** Letting the default lexicographic ordering on player ID stand. Player `00000001` always wins; player `99999999` always loses. Indefensible at any scale.
4. **Client-side tiebreak inconsistent with server.** The client sorts by score alone, the server sorts by `(score, tiebreak)`, and the displayed order disagrees with `ZREVRANK`. Symptoms: friend lists, search results, and front-page boards all show different orders for the same players. Fix: server sends rank; client never re-sorts.
5. **Encoding overflows `2^53` silently.** Multiplying a billion-point base score by `10^11` produces `10^20`, well past the double's safe integer range. The low bits — exactly the tiebreaker portion — get truncated, and players at "different" composites are silently merged. Audit the bit budget with the worst-case case before shipping.
6. **Fractional scores accumulated as floats.** Running `score += 0.1` 10,000 times does not give 1000.0 — it gives 999.9999999999... due to binary representation. Keep score arithmetic in integer cents (or tenths) and only convert at encoding time.
7. **Hardcoding `MAX_TIMESTAMP` too small.** Choosing `10^9` for second-resolution timestamps means the encoding wraps around in 2033. Choose generously (`10^10` for centuries) and verify the budget still fits.
8. **Different tiebreak fields per code path.** The submission path uses `achieved_at`, the read path uses `last_active_at`, and the leaderboard subtly disagrees with itself. Pick one tiebreak field and use it everywhere.
9. **Tiebreaking by `ZSCORE` alone in a multi-board setup.** Two boards (e.g., daily and all-time) with different encodings: a player's daily composite is not comparable to their all-time composite. Always decode before comparing across boards.
10. **Skipping the encode/decode round-trip test.** "It looks right" is not enough. Write the test that submits scores at every order of magnitude and asserts decoded values match. The bug is silent until the prize tier.
11. **Tiebreaker tied to mutable state.** Using "current player level" as the tiebreaker means two players at the same score swap rank when one of them levels up — even though their score didn't change. Tiebreakers should be event-time properties, frozen at submission.
12. **Visible tiebreaker without explanation.** Showing "1000 pts · 3 days ago" without UX context makes players think the game is buggy ("why is my older score worse?"). Either render the tiebreaker with a tooltip ("earlier score wins ties") or hide it.

## Related

- [Designing a Real-Time Leaderboard](../design-realtime-leaderboard.md) — parent case study; this doc expands the "Tie-Breaking — Composite Score Encoding" section with full implementation detail.
- [Redis Sorted Set Internals](./redis-sorted-set-internals.md) — the skip-list + hash-table data structure that backs every operation discussed here, and the `(score, member)` ordering rule the encoding piggybacks on _(planned)_.
- [Top-K Queries](./top-k-queries.md) — the read-path patterns (`ZREVRANGE`, `ZREVRANGEBYSCORE`, neighborhood queries) that consume the composite-encoded scores _(planned)_.
- [Tournament Mode](./tournament-mode.md) — time-boxed leaderboards with prize-tier cutoffs, where tie-breaking determines real prize allocation _(planned)_.
- [Core Trade-offs](../../../foundations/core-trade-offs.md) — the consistency/latency/correctness trade-offs that constrain ZSET-based leaderboards.
- [Databases as a Component](../../../building-blocks/databases-as-a-component.md) — broader context for choosing Redis ZSETs vs SQL `ORDER BY` for ranked-data problems.

## References

- [IEEE 754 Double-Precision Floating-Point Format — Wikipedia](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) — the canonical summary of binary64 layout, the 53-bit significand, and the exact integer range `[-2^53, 2^53]` that the composite encoding has to fit inside.
- [Redis ZADD Command Reference](https://redis.io/commands/zadd/) — official documentation for `ZADD`, including the `GT`/`LT` flags that gate composite updates and the score-type semantics (IEEE 754 double).
- [Redis ZRANGEBYLEX Command Reference](https://redis.io/commands/zrangebylex/) — the lexicographic-ordered range query used in the parallel "all members at score 0" pattern for fixed-score sub-boards.
- [Redis ZREVRANK Command Reference](https://redis.io/commands/zrevrank/) — the read-path query that returns a member's 0-indexed reverse rank, the value the server hands the client to render.
- [Redis Sorted Set Tutorial](https://redis.io/docs/data-types/sorted-sets/) — the data-type-level documentation for ZSETs, including the `(score, member)` ordering invariant the encoding leverages.
- [Steamworks API — Leaderboards](https://partner.steamgames.com/doc/features/leaderboards) — Steam's leaderboard API, including its handling of tie-breaking via secondary score fields and the platform-level conventions that game studios inherit.
- [Apple GameKit — GKLeaderboard](https://developer.apple.com/documentation/gamekit/gkleaderboard) — the Game Center leaderboard reference, including timestamp tiebreaking semantics for scores submitted with the same value.
- [antirez (Salvatore Sanfilippo) — Redis blog and weblog](http://antirez.com/) — Redis author's writings on sorted set internals and composite-encoding patterns; search for "sorted set" and "score encoding" for primary-source design notes.
- [Erlang — `term_to_binary` Sortable Encoding](https://www.erlang.org/doc/man/erlang.html#term_to_binary-1) — Erlang's canonical pattern for sortable timestamp prefixes in keys, an analogous technique used in distributed databases for the same "tiebreak via prefix" reason.
- [Designing Data-Intensive Applications — Martin Kleppmann (O'Reilly, 2017)](https://dataintensive.net/) — Chapter 9 ("Consistency and Consensus") covers the ordering and stability properties that composite encoding gives a leaderboard; the broader design context for why "stable rank" matters.
