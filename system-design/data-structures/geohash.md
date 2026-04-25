---
title: "Geohash"
date: 2026-04-25
updated: 2026-04-25
tags: [system-design, data-structures, geo, indexing]
---

# Geohash — Encoding 2D Coordinates as Strings

**Date:** 2026-04-25 | **Updated:** 2026-04-25
**Tags:** `system-design` `data-structures` `geo` `indexing`

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [Key Concepts](#key-concepts)
  - [Bit Interleaving — Turning 2D Into 1D](#bit-interleaving--turning-2d-into-1d)
  - [The Base32 Alphabet](#the-base32-alphabet)
  - [Precision Table](#precision-table)
  - [Neighbor Calculation](#neighbor-calculation)
  - [The Prefix Property](#the-prefix-property)
- [Trade-offs vs S2 and H3](#trade-offs-vs-s2-and-h3)
- [Code Examples](#code-examples)
  - [Encoding a Coordinate](#encoding-a-coordinate)
  - [Decoding a Geohash](#decoding-a-geohash)
  - [Computing Neighbors](#computing-neighbors)
- [Real-World Uses](#real-world-uses)
  - [Redis GEO Commands](#redis-geo-commands)
  - [Elasticsearch geohash_grid](#elasticsearch-geohash_grid)
  - [Ride-Hailing and Mapping](#ride-hailing-and-mapping)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

A **geohash** is a short alphanumeric string that encodes a `(latitude, longitude)` point on the globe. It was invented by Gustavo Niemeyer in 2008 and published at `geohash.org`. The trick is simple: interleave the bits of latitude and longitude, group the result into 5-bit chunks, and base32-encode each chunk. The resulting string has a useful property — **two points that share a longer prefix are usually closer together** — which lets you turn a 2D nearest-neighbor problem into a 1D string-prefix problem and ride the same B-tree, sorted-set, or LSM index that already powers your database. This doc covers how the encoding works, the precision/length table, how to compute the eight neighbors, where geohash falls down (boundary "edge case" between cells with different prefixes, distortion near the poles), and how it compares to **Google S2** (Hilbert curve on a cube projection) and **Uber H3** (hexagonal grid). It closes with how Redis GEO and Elasticsearch use geohash internally.

## Overview

You have a billion `(lat, lng)` points. You want to answer questions like:

- "What restaurants are within 2 km of this user?"
- "Group every Uber pickup in the last hour into a heat map of 100 m × 100 m cells."
- "Find me the 10 nearest available drivers to this pin."

Doing this with a generic 2D index (R-tree, KD-tree) works, but it requires a specialized index structure. Geohash takes a different bet: **convert the 2D problem into a 1D problem**, then use an off-the-shelf 1D index — a B-tree, a Redis sorted set, an LSM range scan. That works because the geohash string ordering, when read as a 1D sequence, traces a **Z-order (Morton) curve** across the lat/lng plane. Nearby points on the curve are nearby on the plane (most of the time).

```text
   Lat/Lng plane                Geohash strings
   +---+---+---+                "u4pruyd"  -> Berlin-ish
   | A | B | C |                "u4pruyf"  -> a few meters away
   +---+---+---+                "u4prv00"  -> a couple km away
   | D | E | F |                "u4q....."  -> different cell, maybe close maybe not
   +---+---+---+                "u4r....."  -> farther
   | G | H | I |
   +---+---+---+
```

The longer the shared prefix, the smaller the bounding cell that contains both points. A 5-character prefix narrows the location to roughly a 4.9 km × 4.9 km box; an 8-character prefix narrows it to about 38 m × 19 m. That's enough to feed almost any "find nearby" workflow.

## Key Concepts

### Bit Interleaving — Turning 2D Into 1D

Geohash works by **bisecting the world** at every step.

- Latitude lives in `[-90, +90]`. Longitude lives in `[-180, +180]`.
- For each bit position, you decide: is the point in the upper or lower half of the current range?
  - Upper → bit `1`, narrow the range to the upper half.
  - Lower → bit `0`, narrow the range to the lower half.
- You alternate longitude and latitude bits — **longitude first** by convention.

For a target point `(40.7128, -74.0060)` (Manhattan), the first few bits look like:

```text
bit 1: lng in [-180, 180]?  -74.0 < 0 → 0, range becomes [-180,   0]
bit 2: lat in [ -90,  90]?   40.7 > 0 → 1, range becomes [   0,  90]
bit 3: lng in [-180,   0]?  -74.0 > -90 → 1, range becomes [-90,    0]
bit 4: lat in [   0,  90]?   40.7 < 45 → 0, range becomes [   0,  45]
...
```

After `N` rounds you have an `N`-bit binary string. Group it into **5-bit chunks** and base32-encode each chunk. That's the geohash.

### The Base32 Alphabet

Geohash uses a custom base32 alphabet (sometimes called "geohash base32"), which is **not** standard RFC 4648 base32. It deliberately omits `a`, `i`, `l`, and `o` — characters that are easy to confuse with digits or each other:

```text
0 1 2 3 4 5 6 7 8 9 b c d e f g h j k m n p q r s t u v w x y z
```

That gives 32 symbols (5 bits per character). Each character encodes 5 bits = 2.5 lat bits + 2.5 lng bits (the bit-pair shifts by character).

### Precision Table

The precision halves with every additional **bit**, so it improves by a factor of √32 ≈ 5.66 with each character (alternating lat/lng halving):

| Length | ~ Lat error | ~ Lng error (equator) | Cell size at equator | Practical use |
|--------|-------------|------------------------|-----------------------|---------------|
| 1 | ±23 ° | ±23 ° | ~5,000 km × 5,000 km | continent-ish |
| 2 | ±2.8 ° | ±5.6 ° | ~1,250 km × 625 km | country / region |
| 3 | ±0.70 ° | ±0.70 ° | ~156 km × 156 km | metro area |
| 4 | ±0.087 ° | ±0.18 ° | ~39.1 km × 19.5 km | small city |
| 5 | ±0.022 ° | ±0.022 ° | ~4.9 km × 4.9 km | neighborhood |
| 6 | ±0.0027 ° | ±0.0055 ° | ~1.2 km × 0.6 km | block-level |
| 7 | ±0.00068 ° | ±0.00068 ° | ~153 m × 153 m | building cluster |
| 8 | ±0.000085 ° | ±0.00017 ° | ~38 m × 19 m | building / address |
| 9 | ±0.000021 ° | ±0.000021 ° | ~4.8 m × 4.8 m | parking spot |
| 10 | ±0.0000026 ° | ±0.0000054 ° | ~1.2 m × 0.6 m | tighter than GPS noise |

**Important caveat:** longitudinal degrees shrink as you move toward the poles (cos(latitude) factor). The table above assumes the equator. At 60° latitude, longitudinal cells are half as wide. At the poles, all longitudes converge to a point, and "cells" become slivers. This is one of geohash's structural weaknesses.

### Neighbor Calculation

A point is in a single geohash cell, but a query of "all points within radius r" usually crosses cell boundaries. The standard trick is to query the **target cell plus its 8 surrounding neighbors** (north, south, east, west, NE, NW, SE, SW), filter by exact distance, and merge results.

```text
   +-----+-----+-----+
   | NW  |  N  | NE  |
   +-----+-----+-----+
   |  W  | ME  |  E  |       at the chosen precision
   +-----+-----+-----+
   | SW  |  S  | SE  |
   +-----+-----+-----+
```

The neighbors of a geohash string can be computed without re-encoding the lat/lng. The geohash spec defines lookup tables (`borders` and `neighbors`) keyed by direction and "even/odd character index". The algorithm walks one character at a time from the right; if the rightmost character is on the border in the required direction, it recurses into the parent geohash, computes that neighbor, and re-appends. This is `O(geohash length)` in the worst case, which is constant for any practical precision.

### The Prefix Property

The defining feature of geohash:

> If two points share the first `k` characters, they are inside the same `k`-character bounding cell.

The converse is **not** strictly true — two points can be a few meters apart and have completely different geohashes if they straddle a cell boundary. But the prefix property gives you a usable index:

- Want all points in a 4.9 km × 4.9 km cell? Range scan `WHERE geohash LIKE 'u4pru%'`.
- Want all points within ~1 km? Take the 5- or 6-character prefix, then add the eight neighbors at that precision, then filter by exact great-circle distance.
- Want points in a polygon? Cover the polygon with a set of geohash prefixes and union them (this is what Elasticsearch's `geohash_grid` aggregation does internally).

This composes beautifully with **any sorted index**: a B-tree on a `geohash` text column, a Redis sorted set keyed by the integer form of the geohash, an LSM-tree range scan, or a Bloom-filtered LSM block index. **You do not need a specialized geo index** to ship a "find nearby" feature.

## Trade-offs vs S2 and H3

Geohash is not the only space-filling-curve scheme. The two dominant alternatives both fix specific weaknesses of geohash:

| Property | **Geohash** | **Google S2** | **Uber H3** |
|----------|-------------|---------------|-------------|
| Cell shape | Lat/lng rectangles | Spherical quadrilaterals (cube faces) | Hexagons (with 12 pentagons) |
| Curve | Z-order (Morton) | Hilbert curve | Hierarchical hex addressing |
| Cell distortion | Severe near poles, mild elsewhere | Bounded across globe (max ~2:1) | Very small (hexagons are near-uniform) |
| Neighbor distance | Different for diagonal vs side | Roughly equal | All 6 neighbors equidistant |
| Cell ID type | Variable-length string (or 64-bit int) | 64-bit integer | 64-bit integer |
| Hierarchy | 4× per character (well, 2× lat × 2× lng = 4×) | 4× per level | 7× per level |
| Children fit cleanly in parent | Yes | Yes | **No** (hex children don't tile parent) |
| Origin | Niemeyer 2008 | Google ~2014 internal, open-sourced 2017 | Uber 2018 |
| Sweet spot | Easy 1D indexing on existing DBs | Globally accurate, in big systems | Spatial analytics, ride-hailing surge zones |

**Why hexagons?** All six neighbors of a hex cell are at the same distance from its centroid. With geohash squares, the four diagonal neighbors are √2 ≈ 1.41× farther than the side neighbors. For computing things like "average travel time to neighboring zones", that asymmetry matters. H3's downside is that hexagons don't tile cleanly into smaller hexagons — a parent H3 cell of resolution `r` does not exactly equal seven child cells at resolution `r+1`, only approximately. That makes hierarchical aggregation lossy in a way geohash and S2 are not.

**Why S2's Hilbert curve?** A Hilbert curve has the property that **consecutive cells in 1D are always adjacent in 2D** (with no jumps), unlike Z-order which can hop across the plane at certain bit boundaries. S2 also projects onto the **six faces of an enclosing cube** before applying the Hilbert curve, which keeps cell distortion bounded everywhere — including the poles, where geohash falls apart.

**When to pick which:**

- **Geohash** — you want the simplest possible scheme, you are storing points (not regions), your database has decent string range scans, and you are not doing serious analytics near the poles. This covers most product use cases.
- **S2** — you are Google-scale, indexing global regions (not just points), or you need accurate spherical geometry (containment, intersection on the sphere). S2 is what Google Maps and many mapping startups use under the hood.
- **H3** — you do spatial analytics that benefit from uniform-distance neighbors: heat maps, surge pricing zones, fleet utilization, ridership flows. Uber, Foursquare, and many GIS-flavored data products use H3.

For most ride-hailing or "find nearby" features, geohash is the pragmatic default — and the rest of this doc is about using it well.

## Code Examples

### Encoding a Coordinate

```python
# pure-Python geohash encoder. For production, use the `python-geohash`
# or `geohash2` package; this is here to show the algorithm.

BASE32 = "0123456789bcdefghjkmnpqrstuvwxyz"

def encode_geohash(lat: float, lng: float, precision: int = 12) -> str:
    """Encode (lat, lng) to a base32 geohash string.

    Raises ValueError on out-of-range coordinates.
    """
    if not -90.0 <= lat <= 90.0:
        raise ValueError(f"latitude out of range: {lat}")
    if not -180.0 <= lng <= 180.0:
        raise ValueError(f"longitude out of range: {lng}")

    lat_lo, lat_hi = -90.0, 90.0
    lng_lo, lng_hi = -180.0, 180.0

    bits: list[int] = []
    is_lng_turn = True   # geohash interleaves lng-first

    while len(bits) < precision * 5:
        if is_lng_turn:
            mid = (lng_lo + lng_hi) / 2
            if lng >= mid:
                bits.append(1)
                lng_lo = mid
            else:
                bits.append(0)
                lng_hi = mid
        else:
            mid = (lat_lo + lat_hi) / 2
            if lat >= mid:
                bits.append(1)
                lat_lo = mid
            else:
                bits.append(0)
                lat_hi = mid
        is_lng_turn = not is_lng_turn

    # group into 5-bit chunks, map to base32
    out = []
    for i in range(0, len(bits), 5):
        chunk = bits[i:i + 5]
        value = (chunk[0] << 4) | (chunk[1] << 3) | (chunk[2] << 2) | (chunk[3] << 1) | chunk[4]
        out.append(BASE32[value])
    return "".join(out)


# Example
print(encode_geohash(40.7128, -74.0060, precision=8))  # Manhattan → "dr5regw3"
print(encode_geohash(52.5200,  13.4050, precision=8))  # Berlin    → "u33db1pw"
```

### Decoding a Geohash

Decoding gives you back a **bounding box**, not a point — geohash is a lossy encoding. You typically return the box, or its center, depending on the use case.

```python
BASE32_DECODE = {c: i for i, c in enumerate(BASE32)}

def decode_geohash(gh: str) -> tuple[float, float, float, float]:
    """Decode a geohash to (lat_lo, lat_hi, lng_lo, lng_hi)."""
    lat_lo, lat_hi = -90.0, 90.0
    lng_lo, lng_hi = -180.0, 180.0
    is_lng_turn = True

    for ch in gh:
        if ch not in BASE32_DECODE:
            raise ValueError(f"invalid geohash character: {ch}")
        value = BASE32_DECODE[ch]
        for shift in (4, 3, 2, 1, 0):
            bit = (value >> shift) & 1
            if is_lng_turn:
                mid = (lng_lo + lng_hi) / 2
                if bit:
                    lng_lo = mid
                else:
                    lng_hi = mid
            else:
                mid = (lat_lo + lat_hi) / 2
                if bit:
                    lat_lo = mid
                else:
                    lat_hi = mid
            is_lng_turn = not is_lng_turn

    return lat_lo, lat_hi, lng_lo, lng_hi


def decode_geohash_center(gh: str) -> tuple[float, float]:
    lat_lo, lat_hi, lng_lo, lng_hi = decode_geohash(gh)
    return (lat_lo + lat_hi) / 2, (lng_lo + lng_hi) / 2
```

### Computing Neighbors

The standard trick uses precomputed lookup tables that depend on whether the rightmost character is at an "even" or "odd" index (because lng/lat bits alternate per character). Here is the well-known reference algorithm:

```python
# Adapted from the canonical Niemeyer / geohash.org reference implementation.

NEIGHBORS = {
    "n": ("p0r21436x8zb9dcf5h7kjnmqesgutwvy", "bc01fg45238967deuvhjyznpkmstqrwx"),
    "s": ("14365h7k9dcfesgujnmqp0r2twvyx8zb", "238967debc01fg45kmstqrwxuvhjyznp"),
    "e": ("bc01fg45238967deuvhjyznpkmstqrwx", "p0r21436x8zb9dcf5h7kjnmqesgutwvy"),
    "w": ("238967debc01fg45kmstqrwxuvhjyznp", "14365h7k9dcfesgujnmqp0r2twvyx8zb"),
}

BORDERS = {
    "n": ("prxz",     "bcfguvyz"),
    "s": ("028b",     "0145hjnp"),
    "e": ("bcfguvyz", "prxz"),
    "w": ("0145hjnp", "028b"),
}

def neighbor(gh: str, direction: str) -> str:
    """Return the neighbor of geohash `gh` in the given direction
    ('n', 's', 'e', 'w'). Same precision as input."""
    if not gh:
        raise ValueError("cannot compute neighbor of empty geohash")
    last = gh[-1]
    parent = gh[:-1]
    type_idx = (len(gh) - 1) % 2  # 0 = even, 1 = odd

    if last in BORDERS[direction][type_idx] and parent:
        # walk up: the neighbor is in a different parent cell
        parent = neighbor(parent, direction)

    return parent + NEIGHBORS[direction][type_idx][BASE32_DECODE[last]]


def all_neighbors(gh: str) -> dict[str, str]:
    """Return all 8 neighbors plus the cell itself."""
    n  = neighbor(gh, "n")
    s  = neighbor(gh, "s")
    e  = neighbor(gh, "e")
    w  = neighbor(gh, "w")
    return {
        "self": gh,
        "n":  n,  "s":  s,  "e":  e,  "w":  w,
        "ne": neighbor(n, "e"),
        "nw": neighbor(n, "w"),
        "se": neighbor(s, "e"),
        "sw": neighbor(s, "w"),
    }


# Example: 9 geohashes covering ~12 km × 12 km around Berlin
print(all_neighbors("u33db1"))
```

In production you would use a battle-tested package — `python-geohash`, `pygeohash`, `node-geohash`, `geohash-java`, or your database's built-in functions (PostGIS `ST_GeoHash`, MySQL `ST_GeomFromGeoHash`). These avoid subtle off-by-one bugs near the date line and poles.

## Real-World Uses

### Redis GEO Commands

Redis since 3.2 ships **GEO commands** (`GEOADD`, `GEOSEARCH`, `GEOPOS`, `GEODIST`, etc.). Under the hood, Redis stores each member in a **sorted set** scored by a **52-bit integer geohash** (rather than the base32 string form, but the same encoding). Because a sorted set is just a B-tree-equivalent structure indexed by score, range queries on the integer geohash give you "all members in this rectangular cell" in `O(log N + M)` time.

```text
GEOADD drivers -73.985 40.748 driver:42
GEOADD drivers -73.965 40.758 driver:99

GEOSEARCH drivers FROMLONLAT -73.98 40.75 BYRADIUS 1 km ASC
  -> driver:42, driver:99
```

Internally `GEOSEARCH BYRADIUS` does roughly:

1. Compute the geohash cell at a precision wide enough to cover the radius.
2. Take that cell plus its 8 neighbors.
3. Range-scan the sorted set for each cell's `[min_score, max_score]` interval.
4. For each candidate, decode and run a precise great-circle distance check.
5. Return the survivors, sorted.

This is why Redis GEO scales to millions of members per key and serves "find nearby" with sub-millisecond latency. The flip side is that Redis GEO is in-memory only and assumes a single sorted set per "world" — partitioning across Redis Cluster requires per-shard logic.

### Elasticsearch geohash_grid

Elasticsearch (and OpenSearch) ship a `geohash_grid` aggregation that buckets documents by their geohash prefix at a chosen precision:

```json
{
  "aggs": {
    "rides": {
      "geohash_grid": {
        "field": "pickup_point",
        "precision": 6
      }
    }
  }
}
```

This is how heat maps, density tiles, and "popular areas" features are typically built on top of Elasticsearch. Internally it uses the same prefix grouping described above — the index already has per-document geohash terms, and the aggregation just groups by the chosen length. Elasticsearch also supports `geotile_grid` (Web Mercator tiles) and `geohex_grid` (H3 hexagons) for the same shape of query when you need different cell geometry.

### Ride-Hailing and Mapping

- **Driver discovery** — store every available driver's location in a Redis sorted set keyed by geohash. When a rider opens the app, query the surrounding 9 cells and rank by distance. (See [Design Uber](../case-studies/location-based/design-uber.md).)
- **Delivery dispatch** — DoorDash, Deliveroo, etc. partition active couriers by geohash prefix to keep dispatch search local. (See [Design DoorDash](../case-studies/location-based/design-doordash.md).)
- **Listing search** — Airbnb, Zillow, and similar use geohash prefixes to coarsely filter listings before applying expensive ranking, especially when serving from a sharded search index. (See [Design Airbnb](../case-studies/location-based/design-airbnb.md).)
- **Dating apps** — Tinder and similar prune the candidate set to "people in nearby geohash cells" before applying preference filters. (See [Design Tinder](../case-studies/social-media/design-tinder.md).)
- **Tile pyramids and map tiles** — geohash is sometimes used as a deterministic key for cached map tile bundles, although tile coordinates `(z, x, y)` are more common at the rendering layer.

In all of these, geohash is the **coarse filter**. The fine-grained ranking — distance, ETA, preferences, surge — happens after the geohash range scan trims the candidate set from "every member" to "a few hundred plausible members".

## Anti-Patterns

**Assuming uniform cell size across the globe.** A 6-character cell in Singapore is roughly square. The same 6-character cell at 70°N is a tall, thin sliver. If your code computes "drivers within 1 km" by counting cells, you will under-count near the equator and wildly over-count near the poles. **Always filter by actual great-circle distance after the geohash prefix range scan**, never by cell count.

**Naive nearest-neighbor at boundaries.** If you query only the geohash cell containing the user, you will miss every point in the adjacent cell that is, in real distance, closer than points in your own cell. The classic example: a user 1 m from the eastern border of cell `u4pru0` will not see the driver 2 m away in cell `u4pru1`, because `u4pru0` and `u4pru1` may not even share a long prefix. **Always query the cell plus its 8 neighbors**, then filter and rank.

**Using geohash for regions that span the date line or poles.** Cells that straddle the antimeridian (`±180°` longitude) are split across the geohash space. A bounding box `lng_lo = 170°, lng_hi = -170°` (which most humans read as "the small wedge over the Pacific") covers nearly the whole globe in geohash terms. Either pre-split queries at the date line, or use S2 if you do this often.

**Storing geohash as the only index and never the raw lat/lng.** Geohash is lossy. If you ever need a precise location — to draw a marker, to compute an exact distance to a fixed business address, to feed a routing engine — you need the original lat/lng. Store both: the lat/lng for truth and the geohash for indexing.

**Picking the precision once and forever.** If your product spans both city-density use cases (Manhattan, 100 m per driver) and rural cases (Montana, 50 km per driver), no single geohash precision is right. Use a precision that matches the query radius — start narrow and widen if the result count is too small. Some systems store **multiple geohash columns** (length 5, 6, 7) and pick at query time.

**Treating the geohash space as Euclidean.** Distances in geohash-bit space are not real-world distances. Two geohashes can differ in only the last bit and be tens of kilometers apart on the curve's "long edge"; or share a long prefix and still be on opposite sides of a cell boundary that happened to align with a power-of-two latitude. Always come back to **great-circle distance** for any decision that affects a user.

**Reinventing geohash arithmetic.** Computing neighbors, parents, and base32 conversions correctly across the date line is fiddly. Use a vetted library: `python-geohash`, `geohash-java`, `geohash-go`, PostGIS's built-ins, or your DB's native GEO commands. Hand-rolled implementations consistently miss the edge cases.

## Related

- [Quadtrees and R-Trees](./quadtrees-and-r-trees.md) — the alternative tree-based families of 2D indexes; better than geohash for dynamic regions and rectangle queries, worse for prefix-keyed B-tree storage
- [Design Uber](../case-studies/location-based/design-uber.md) — driver-rider matching, where geohash is the canonical pre-filter before fine-grained dispatch
- [Design DoorDash](../case-studies/location-based/design-doordash.md) — courier dispatch and delivery zone partitioning using geohash buckets
- [Design Airbnb](../case-studies/location-based/design-airbnb.md) — listing search, map-bounds queries, and how geohash combines with attribute filters
- [Design Tinder](../case-studies/social-media/design-tinder.md) — distance-bounded candidate sets for matching, the simplest production use of geohash

## References

- Gustavo Niemeyer, ["Geohash" (original announcement, archived)](https://web.archive.org/web/20080305223755/http://blog.labix.org/2008/02/26/geohashorg-and-the-geohash-algorithm) — the 2008 post that introduced the algorithm and `geohash.org`
- [Geohash on Wikipedia](https://en.wikipedia.org/wiki/Geohash) — the most comprehensive reference for the encoding, base32 alphabet, precision tables, and neighbor lookup tables
- [Redis GEO Commands Documentation](https://redis.io/docs/latest/commands/geoadd/) — `GEOADD`, `GEOSEARCH`, `GEORADIUS` and how Redis stores geohashes as sorted-set scores
- [Elasticsearch `geohash_grid` aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-geohashgrid-aggregation.html) — bucketing documents by geohash prefix for heat maps and density tiles
- [Google S2 Geometry Library — Overview](http://s2geometry.io/about/overview) — the Hilbert-curve-on-a-cube alternative used at Google
- [Uber H3 — Hexagonal Hierarchical Geospatial Indexing System](https://h3geo.org/) — the hex grid alternative; the [Uber Engineering blog post](https://www.uber.com/blog/h3/) explains why hexagons
- [PostGIS `ST_GeoHash`](https://postgis.net/docs/ST_GeoHash.html) — the canonical SQL implementation; useful for grounding any custom encoder against a reference
- [Robert Coup, "Visualizing Geohashes"](https://www.movable-type.co.uk/scripts/geohash.html) — an interactive explainer that makes the bit-interleaving and cell layout tangible
