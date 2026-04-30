---
title: "Uber Deep Dive — H3 Geo-Indexing"
date: 2026-04-29
updated: 2026-04-29
tags: [system-design, case-study, uber, deep-dive, h3, geospatial]
---

# Uber Deep Dive — H3 Geo-Indexing

**Date:** 2026-04-29 | **Updated:** 2026-04-29
**Tags:** `system-design` `case-study` `uber` `deep-dive` `h3` `geospatial`

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [Why Hexagons](#why-hexagons)
  - [Uniform Neighbor Distance](#uniform-neighbor-distance)
  - [Six Neighbors Instead of Eight](#six-neighbors-instead-of-eight)
  - [Smoother Movement and Flow Modeling](#smoother-movement-and-flow-modeling)
  - [The Honest Cost of Hexagons](#the-honest-cost-of-hexagons)
- [16 Resolution Levels](#16-resolution-levels)
- [Hex IDs (64-bit)](#hex-ids-64-bit)
- [k-Ring Queries](#k-ring-queries)
- [Pentagon Distortion](#pentagon-distortion)
- [Hex-Edge Crossing](#hex-edge-crossing)
- [vs S2](#vs-s2)
- [vs Geohash](#vs-geohash)
- [Resolution by Use Case](#resolution-by-use-case)
- [Compactness](#compactness)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

**[H3](https://h3geo.org/)** is Uber's open-source hierarchical hexagonal geospatial index. It tiles the surface of the Earth with hexagons at **16 resolutions** (res 0 ≈ continent-sized, res 15 ≈ ~0.9 m² per cell), assigns each hex a **64-bit integer ID**, and exposes a small, sharp API for two operations that ride-hailing systems do constantly: "what is the cell at this point?" and "what cells are within `k` rings of this cell?". The reason Uber built H3 — and the reason it has stuck — is that **hexagons have uniform neighbor distance**: every hex has six neighbors, all the same distance from the center, unlike a square grid where the four diagonal neighbors are √2× farther than the four side neighbors. For a system that constantly asks "show me the drivers in this area" and "what is the average travel time between this zone and that zone", that symmetry compounds into cleaner analytics, smoother heat maps, and simpler dispatch logic. The trade-off is real but small: hexagons cannot perfectly tile a sphere using only hexagons, so H3 inserts **exactly 12 pentagons** at vertices of the underlying icosahedral projection, and parent-child containment is approximate rather than exact (the 1:7 aperture). This deep-dive walks through why hexagons are the right shape, what each of the 16 resolutions buys you, how the 64-bit ID is laid out, how `k`-ring queries support real-time dispatch, where the pentagons live and why they don't matter for most workloads, how H3 compares to **Google S2** and **geohash**, and which resolution to pick for which use case (dispatch ≈ res 9, surge zones ≈ res 7, city-level analytics ≈ res 6, country-level rollups ≈ res 4).

## Overview

H3 was open-sourced by Uber in 2018 after several years of internal use for surge pricing, dispatch, and analytics. It is a **hierarchical hexagonal hierarchical spatial index** — long phrase, three real ideas:

1. **Hexagonal:** the unit cell is a hexagon, chosen for its neighbor-distance and packing properties.
2. **Hierarchical:** every cell at resolution `r` is associated with seven children at resolution `r+1` (1:7 aperture). Children fit *approximately*, not perfectly.
3. **Spatial index:** the cell ID is a fixed 64-bit integer with predictable layout, suitable for storing in any database column, hashing, sorted-set scoring, or routing-key generation.

The library's two most-used operations are tiny:

```python
import h3

cell = h3.latlng_to_cell(40.7128, -74.0060, 9)        # Manhattan @ res 9
ring = h3.grid_disk(cell, 2)                          # all cells within 2 rings
```

Everything else — parent/child traversal, polygon coverage, edge geometry, distance-on-the-grid, compaction — is built on top of these two primitives. The minimalism is deliberate: dispatch needs answers in microseconds, so H3 ships a C library with thin wrappers in Python, Java, JavaScript, Go, Rust, R, and a handful of others. ([h3geo.org/docs](https://h3geo.org/docs/))

```text
   World, projected to icosahedron, tiled with hexagons + 12 pentagons
   ┌─────────────────────────────────────────────────────────┐
   │       /\        /\        /\        /\        /\        │
   │      /  \      /  \      /  \      /  \      /  \       │
   │  ●─P────●────●────●────●────●────●────●────●────●─P─●   │  res 0: 122 cells
   │   \    /\    /\    /\    /\    /\    /\    /\    /      │  (110 hex + 12 pent)
   │    \  /  \  /  \  /  \  /  \  /  \  /  \  /  \  /       │
   │     \/    \/    \/    \/    \/    \/    \/    \/        │
   └─────────────────────────────────────────────────────────┘
   P = pentagon. There are exactly 12 of them, located on the 12 vertices
   of an icosahedron — almost all over open ocean.
```

For ride-hailing, the practical payoff is:

- **Driver dispatch.** Index every available driver by their current cell at res 9. To find candidates near a pickup, query the pickup's cell plus a small `k`-ring (typically 1–2). The candidate set goes from "every driver" to "drivers in ~10–30 cells".
- **Surge pricing zones.** Compute supply/demand ratios per cell at res 7 (~5 km² per cell). Smooth across neighbors using `grid_disk(cell, 1..2)`.
- **Heat maps and analytics.** Aggregate trip pickups, ETAs, fare distributions, and idle time per cell. Hexagons render as visually smooth heat maps without the diagonal artifacts of a square grid.
- **Sharding.** Use the hex ID prefix (e.g., the cell at a coarser resolution like res 4 or res 5) as the shard key for the in-memory geo-index. Drivers that move across cell boundaries trigger a brief shard hand-off (Uber's so-called "moving driver" problem).

This doc unpacks the design choices that make H3 fit ride-hailing so well — and the small set of gotchas (pentagons, hierarchical lossiness, edge straddling) you should know about before adopting it.

## Why Hexagons

If you sit down to design a global grid, three regular polygons can tile the plane: equilateral triangles, squares, and hexagons. (Pentagons cannot tile a flat plane at all — but on a sphere, you'll see why they have to appear anyway.) Of those three, hexagons are the right pick for a geo-index. Here is why, in detail.

### Uniform Neighbor Distance

Take any cell in a square grid. It has **eight neighbors** — four on the sides (N, S, E, W), four on the corners (NE, NW, SE, SW). The four side neighbors share an edge with the cell. The four corner neighbors share only a single point. The center-to-center distance is different for the two groups:

- Side neighbor: `d`
- Corner neighbor: `d × √2 ≈ 1.414d`

That asymmetry is invisible if you just want to find points "in the same cell", but it shows up the moment you do anything that involves *neighbors as a group*: averaging metric across nearby zones, smoothing a heat map, computing flows, or running an A\* search where step cost depends on adjacency.

Hexagons fix this. Every hex has **six neighbors**, all sharing one full edge with the cell, all at the **same** center-to-center distance. The "ring" around a hex is a clean polygonal annulus, not a stretched octagonal one. When you query "all points within `k` rings of this cell", you get a roughly circular region — `k` rings on a hex grid covers approximately `1 + 3k(k+1)` cells arranged in a hexagonal disk that approximates a Euclidean circle far better than a square's k-ring (which is a square of side `2k+1`).

```text
   Square 1-ring (8 neighbors)         Hex 1-ring (6 neighbors)
   ┌────┬────┬────┐                       _____
   │ NW │ N  │ NE │                      /  N  \____
   ├────┼────┼────┤                  ___/ \___/  NE \
   │ W  │ ME │  E │                 / NW  \   /     │
   ├────┼────┼────┤                 \_____/ME \____/
   │ SW │ S  │ SE │                 /  SW \   /  SE \
   └────┴────┴────┘                 \_____/_S_\____/
   side dist d, corner d√2          all six edge-adjacent
   neighbors at same distance d
```

This is the big idea of H3, and it is the single property that drove Uber to build it instead of using S2 or geohash.

### Six Neighbors Instead of Eight

Six neighbors instead of eight matters in three concrete ways:

1. **Simpler adjacency code.** Many algorithms — dispatch fan-out, smoothing, polygon traversal — branch on "which neighbor are we visiting?". Six is fewer cases than eight, and they're symmetric, so you can index neighbors by `(0..5)` and rotate them with modular arithmetic.
2. **Lower variance in neighbor metrics.** If your "average ETA from this zone to neighbors" averages over six equidistant neighbors, you get a clean radial estimate. With eight non-equidistant neighbors, you have to weight by distance, or accept that NE/NW/SE/SW pull the average up.
3. **Cleaner flow modeling.** "How many trips moved from cell A to cell B?" is a flow on edges of the dual graph. With hexagons, the dual graph has uniform edge weights for all neighbor transitions, which makes flow analysis numerically nicer.

### Smoother Movement and Flow Modeling

Real-world driver and rider movement is approximately Euclidean — drivers don't prefer NE direction over E direction. In a square grid, modeling motion forces you to choose: either treat all 8 neighbors as adjacent (and accept the √2× distance asymmetry) or treat only the 4 side neighbors as adjacent (and accept that diagonal motion takes two steps when one would do). Hexagons sidestep the dilemma — direction is approximately uniform across the six neighbors.

For Uber's surge-pricing model, this means a demand spike can "spread" across hex neighbors with consistent decay, instead of having to special-case diagonal vs orthogonal smoothing. For ETA isochrones, the boundary that "everywhere reachable in 5 minutes" looks closer to a real isochrone (an irregular blob) instead of an obviously gridded square stack.

### The Honest Cost of Hexagons

There is exactly one mathematical cost: **hexagons cannot tile a sphere using only hexagons**. The Euler-characteristic argument: a sphere has Euler characteristic 2, and a tiling by polygons with `f` faces, `e` edges, `v` vertices satisfies `v - e + f = 2`. If every face is a hexagon (6 edges, each shared by 2 faces) and every vertex meets exactly 3 faces (which is the only way hexagons can fit edge-to-edge), then `e = 3f`, `v = 2f`, and `v - e + f = 0`, not 2. The deficit forces some faces to have fewer than 6 sides. The solution is to **insert exactly 12 pentagons**. (You can prove that 12 is the minimum number; it falls out directly from the Euler equation when the rest of the tiling is hexagonal.)

H3 places those 12 pentagons at the vertices of an underlying icosahedron. The icosahedron is chosen because it minimizes distortion when projecting onto the sphere — its 20 triangular faces give the smoothest globally uniform tiling, and its 12 vertices are the only places where pentagons need to live. We discuss the practical implications in [Pentagon Distortion](#pentagon-distortion); for almost all workloads outside the open ocean, the 12 pentagons are simply not in the part of the world your data touches.

The second cost is **non-perfect parent-child containment**. Because of the 1:7 aperture (one parent contains approximately seven children), child cells at resolution `r+1` do not exactly tile the parent at resolution `r` — they overflow slightly at the edges. H3's hierarchical traversal uses a deterministic "centroid child" rule, but if you literally union all seven children, you do not get back the parent's exact polygon. This makes hierarchical aggregation slightly lossy compared to geohash (a clean 4× hierarchy) or S2 (a clean 4× hierarchy via Hilbert curve subdivision). For analytics, the loss is well below noise; for any code that requires exact set containment between resolutions, **H3 is the wrong tool**.

## 16 Resolution Levels

H3 defines **16 resolutions**, numbered 0 through 15. Each finer resolution has approximately 7× as many cells as the previous (the 1:7 aperture mentioned above), so the cell area shrinks by roughly a factor of 7 per level.

| Resolution | Avg cell area | Avg edge length | Cells worldwide  | Practical use |
|------------|--------------|------------------|------------------|----------------|
| 0  | ~4,250,547 km² | ~1,107.71 km | 122              | Continent-scale partitioning |
| 1  | ~607,221 km²   | ~418.68 km   | 842              | Sub-continental regions |
| 2  | ~86,746 km²    | ~158.24 km   | 5,882            | Country / large region |
| 3  | ~12,393 km²    | ~59.81 km    | 41,162           | Sub-country / state |
| 4  | ~1,770 km²     | ~22.61 km    | 288,122          | Metro region / large city footprint |
| 5  | ~252 km²       | ~8.54 km     | 2,016,842        | City / county |
| 6  | ~36 km²        | ~3.23 km     | 14,117,882       | Surge-zone candidate, city quadrant |
| 7  | ~5.16 km²      | ~1.22 km     | 98,825,162       | **Surge zones** (Uber default) |
| 8  | ~0.74 km²      | ~461.35 m    | 691,776,122      | District / large neighborhood |
| 9  | ~0.105 km²     | ~174.38 m    | 4,842,432,842    | **Driver dispatch** (Uber default) |
| 10 | ~0.015 km²     | ~65.91 m     | 33,897,029,882   | Block-level dispatch, micro-mobility |
| 11 | ~2,150 m²      | ~24.91 m     | 237,279,209,162  | Building cluster |
| 12 | ~307 m²        | ~9.42 m      | 1,660,954,464,122| Building / parking lot |
| 13 | ~43.9 m²       | ~3.56 m      | 11,626,681,248,842| Per-spot resolution |
| 14 | ~6.27 m²       | ~1.35 m      | 81,386,768,741,882| Sub-meter precision |
| 15 | ~0.895 m²      | ~0.51 m      | 569,707,381,193,162 | Below GPS noise floor |

(Numbers are from the [H3 documentation](https://h3geo.org/docs/core-library/restable/); they are global averages — actual cell areas vary by latitude due to the icosahedral projection.)

The hierarchy is **lossy at the boundary**. A parent cell at resolution `r` has *approximately* seven children at resolution `r+1`. The "centroid child" — the child whose centroid is closest to the parent's — is well defined; the other six children are the centroid child's neighbors at the finer resolution. This means:

- `h3.cell_to_children(cell, r+1)` returns exactly 7 cells.
- The union of those 7 cells covers slightly more than the parent.
- Conversely, `h3.cell_to_parent(cell, r-1)` returns one cell, but a different child at resolution `r` might also map to the same parent — it's a many-to-one but not a strict tile-cover.

For analytics that need exact set membership across resolutions (rare in practice), this is a real limitation. For dispatch, surge, and heat maps, it is invisible.

## Hex IDs (64-bit)

Every H3 cell has a 64-bit integer identifier. The bits are laid out so that the resolution, base-cell index, and per-resolution digit path can be read off directly without a lookup table:

```text
H3 cell index, 64 bits, MSB → LSB:

 1 bit  | 4 bits  | 3 bits     | 4 bits      | 7 bits          | 3 bits × 15 = 45 bits
 reserved | mode  | edge/vertex| resolution  | base cell index | 15 digits, one per resolution

 mode = 1 for cells (also 2 for unidirectional edges, 4 for vertices)
 base cell index = 0..121, the res-0 cell this cell descends from
 digits = 0..6 each, indicating which child at each resolution
          7 = "unused" (cells coarser than res 15 use 7 for unused trailing slots)
```

Two practical consequences of this layout:

- **Resolution is recoverable from the ID alone.** No external metadata; `h3.get_resolution(cell)` is just a bit extraction.
- **The base cell tells you which of the 122 res-0 cells you are in.** For sharding by region, a stable hash of the base-cell index gives you ~120 shards "for free", or you can map base cells onto fewer shards by region.

H3 IDs are typically displayed in **lowercase hex**:

```text
res 9 cell over Times Square: 8a2a1072b59ffff
res 7 surge zone:             872a1072affffff
res 0 base cell:              8001fffffffffff   (one of 122)
```

The trailing `f`s are the "unused" digits at coarser resolutions — every digit slot is `0..7` (3 bits), and `7` means "this slot is past the cell's resolution and carries no information". This means the binary representation of a coarser cell is a prefix of one of its descendants in a structured way, but **not a literal string prefix** — H3 IDs are not designed for prefix range scans the way geohash strings are. You do parent/child traversal through the API, not through string operations.

For database storage, the typical shapes are:

- **PostgreSQL:** `BIGINT` or `TEXT`. Use `BIGINT` for joins, `TEXT` for human-readable debugging. The [`h3-pg`](https://github.com/zachasme/h3-pg) extension exposes H3 as a native type.
- **MySQL:** `BIGINT UNSIGNED`.
- **Cassandra / DynamoDB:** the hex string as a partition key, optionally combined with a coarser cell as the partition key and the finer cell as the cluster key (sharding by surge zone, sub-sorting by dispatch cell).
- **Redis:** the integer form as a sorted-set score works in principle, but most teams use H3 ID strings as plain set/hash keys and don't try to range-scan by ID.

## k-Ring Queries

The defining query in H3 is `grid_disk(cell, k)` (formerly called `kRing`) — return all cells within `k` "rings" of the input cell. A ring of distance 1 is the six immediate neighbors. A ring of distance 2 is 12 cells. Distance `k` contributes `6k` cells (except in the special-case of pentagons, which contribute one less). Total cells in a `k`-disk:

```text
N(k) = 1 + 6 × (1 + 2 + ... + k) = 1 + 3k(k + 1)

k=0  →    1 cell  (just the center)
k=1  →    7 cells (center + 6 neighbors)
k=2  →   19 cells
k=3  →   37 cells
k=4  →   61 cells
k=5  →   91 cells
k=10 →  331 cells
```

For Uber's dispatch:

```python
import h3

# pickup point
pickup_cell = h3.latlng_to_cell(40.7589, -73.9851, 9)  # Times Square @ res 9

# search drivers in cell + 2 rings (~19 cells, ~520 m radius at res 9)
search_cells = h3.grid_disk(pickup_cell, 2)

candidate_drivers = []
for cell in search_cells:
    candidate_drivers.extend(geo_index.lookup(cell))

# precise filtering: great-circle distance, eligibility, status, ETA
eligible = [d for d in candidate_drivers
            if great_circle_km(pickup, d.location) < 1.0
               and d.status == "available"
               and pickup.product in d.eligible_products]
```

`grid_disk` is `O(k²)` in cells returned but constant-time per cell in pure CPU work — H3 has a closed-form unrolling of the disk so it doesn't BFS. In practice, on a modern CPU, a `grid_disk(cell, 3)` returning 37 cells takes a fraction of a microsecond.

The variants you'll see in dispatch code:

- **`grid_disk(cell, k)`** — the cell and all cells within `k` steps.
- **`grid_ring(cell, k)`** — only the cells *exactly* at distance `k` (a hex annulus). Useful for "expand the search outward" loops where you don't want to re-process inner rings.
- **`grid_distance(a, b)`** — the integer hex-grid distance between two cells. Constant time, exact, no haversine. Useful for fast bucketing before precise distance computation.
- **`grid_path_cells(a, b)`** — the line of cells traversed when walking from `a` to `b` along the grid. Cheap approximation for ETA-along-route at zoom levels too coarse for true routing.

The classic dispatch loop is:

```python
def find_nearest_drivers(pickup, k_max=4):
    cell = h3.latlng_to_cell(pickup.lat, pickup.lng, 9)
    for k in range(0, k_max + 1):
        ring = h3.grid_ring(cell, k) if k > 0 else {cell}
        candidates = []
        for c in ring:
            candidates.extend(geo_index.lookup(c))
        if len(candidates) >= MIN_CANDIDATES:
            return rank_by_eta_and_score(candidates, pickup)
    # fallback: widen search, dispatch alert, etc.
    return []
```

Notice that the k-ring expansion doesn't have to commit to a fixed radius — you grow until you have enough candidates, then stop. With hexagons, each new ring adds a known number of cells in a near-circular boundary, so there's no diagonal-vs-side bias in how the search expands. A square grid would either over-search corners or under-search sides.

### Pentagon-Aware Disks

A pentagon has only five neighbors instead of six, and the ring count formula breaks slightly there. H3's `grid_disk` handles this transparently — the returned set is correct, just one cell smaller than the "ideal" `1 + 3k(k+1)`. There is also `grid_disk_safe` and `grid_disk_unsafe` in the C API: the "safe" variant is correct around pentagons but slightly slower; the "unsafe" variant is faster but may return incorrect results if the disk crosses a pentagon. For server-side dispatch, you almost always want the safe variant. ([h3geo.org/docs/api/traversal](https://h3geo.org/docs/api/traversal/))

## Pentagon Distortion

There are exactly **12 pentagons** in the H3 grid, one at each vertex of the icosahedron H3 projects from. They live at predictable but obscure spots on Earth, almost all of them in open ocean:

- 1 in the Indian Ocean off Madagascar
- 1 west of Norway, in the North Atlantic
- 1 in the Pacific west of South America
- 9 more, all over open ocean

The exact coordinates are documented in the H3 source. The pentagons exist at every resolution — there are 12 pentagons at res 0, 12 (different) pentagons at res 1, and so on — each one is a child or descendant of one of the 12 res-0 pentagons.

The reason it is **exactly 12** comes from the Euler characteristic of a sphere. For any polyhedral tiling of `S²`:

```text
v - e + f = 2

If every face is a hexagon (6 edges) and every vertex meets exactly 3 faces:
  e = 6f / 2 = 3f
  v = 6f / 3 = 2f
  v - e + f = 2f - 3f + f = 0 ≠ 2

The deficit of 2 forces some faces to drop a side. If you replace n hexagons
with pentagons (5 edges each), the equation gives n = 12. There is no way to
tile a sphere with only hexagons; 12 pentagons is the unavoidable minimum.
```

You can see the same number on a soccer ball (the classic truncated icosahedron has 20 hexagons and exactly 12 pentagons), in a fullerene molecule (C₆₀ with 60 carbon atoms arranged in 20 hexagons and 12 pentagons), and in any geodesic dome made from triangles. H3 is just the natural geo-indexing version of the same topological fact.

What "distortion" actually means for pentagons:

- A pentagon has 5 neighbors, not 6. `grid_disk(pentagon, 1)` returns 6 cells (the pentagon plus 5 hexes), not 7.
- The cells immediately around a pentagon have slightly distorted shapes — they are still hexagons but slightly stretched.
- The grid-distance metric is still well-defined; pentagons are just a degree-5 vertex in the dual graph instead of degree-6.

For ride-hailing, **the pentagons are simply not on land you care about**. Uber's dispatch in NYC, SF, London, Mumbai, São Paulo never touches a pentagon. The library handles them correctly, and your code doesn't need special cases unless you are doing global analytics or maritime work. If you are — say, fleet logistics for a shipping company — the pentagons matter, and you should use the `is_pentagon` predicate plus the `_safe` variants of disk functions.

The non-trivial caveat: certain operations have different runtime costs near pentagons. `grid_disk_unsafe` may produce wrong answers; `grid_path_cells` may take a slightly longer path; and any algorithm that assumes a strict 6-degree adjacency invariant may need a check. The H3 docs flag these explicitly. ([h3geo.org/docs/library/restable/#h3-pentagon-cells](https://h3geo.org/docs/library/restable/))

## Hex-Edge Crossing

H3 represents not just cells but also **directed edges between adjacent cells** — `H3DirectedEdge` indexes, with their own 64-bit IDs (mode = 2 in the index layout). This is useful for two ride-hailing-specific scenarios.

**1. Routing along the grid.** When you don't have a real road graph (or you're aggregating to a coarse level for ETA models), a hex-grid path approximates a straight-line route. `h3.origin_to_directed_edges(cell)` returns the 6 outbound edges; `h3.directed_edge_to_cells(edge)` returns the (origin, destination) pair. This is enough to build a graph for shortest-path approximation at coarse resolutions.

**2. Boundary crossings as events.** A driver who pings every 4 seconds will, over the course of a trip, cross many hex edges. Treating "crossing edge `E`" as a discrete event lets you:

- Stream "driver entered cell `C`" events to subscribers.
- Compute flows: how many trips crossed edge `E` in the last hour?
- Detect congestion: if the average speed across edge `E` drops, the cells on either side may be congested.
- Trigger shard hand-off: the geo-index shard responsible for cell `A` hands off to the shard for cell `B` when a driver crosses the edge from `A` to `B`.

The hand-off case is the trickiest piece of correctness in Uber's dispatch architecture: a driver moving across cells must briefly appear in both shards (or neither) for a few hundred milliseconds, and dispatch logic must tolerate either state. The standard pattern is a `driver_id → current_cell` authoritative map (often Redis) that the shards consult on collision; when a hand-off happens, both shards reconcile against the map.

**Edge geometry.** `h3.directed_edge_to_boundary(edge)` returns the great-circle line segment of the shared edge between two cells, suitable for rendering or geometric intersection. This is how you can render a "flow map" of trips by drawing edges weighted by traversal count.

## vs S2

Google's [**S2**](http://s2geometry.io/about/overview) is the most direct competitor to H3 at planet scale. They differ in five ways that matter for system design.

| Property | H3 | S2 |
|----------|----|----|
| Cell shape | Hexagons + 12 pentagons | Spherical quadrilaterals (cube faces) |
| Curve | Hex hierarchical IDs | Hilbert curve over a cube projection |
| Resolutions | 16 (res 0–15) | 31 (level 0–30) |
| Aperture | 1:7 (parent has ~7 children) | 1:4 (parent has exactly 4 children) |
| Children tile parent? | Approximately | **Exactly** |
| Cell ID type | 64-bit integer | 64-bit integer |
| Sweet spot | Spatial analytics, dispatch, hexagonal heat maps | Region indexing, polygon containment, prefix range scans on a Hilbert curve |
| Open source | Apache 2.0, Uber 2018 | Apache 2.0, Google 2017 (used internally since ~2014) |

**Where H3 wins:**

- **Uniform neighbor distance.** S2 cells have 8 neighbors with the same √2 asymmetry as a square grid (S2 cells are quadrilaterals on a cube projection). For analytics that lean on neighbor symmetry — surge smoothing, average-of-neighbors features, hex-bin heat maps — H3 is structurally better.
- **Heat map rendering.** Hexagonal heat maps look smoother to humans than square ones; this is a genuine UX advantage when surge zones or trip density are user-visible.
- **Closer to circular `k`-ring shapes.** A `k`-ring on a hex grid is closer to a true circle than a `k`-ring on a square grid.

**Where S2 wins:**

- **Exact hierarchical containment.** S2's children tile the parent perfectly. If you need "all cells inside this region at this level, exactly", S2 is the right tool — used heavily in Google Maps, Foursquare's geo-index (originally), and most general-purpose mapping stacks.
- **Region indexing.** S2 has very mature region-covering code (`S2RegionCoverer`) that produces a minimal set of cells covering an arbitrary polygon. H3 has `polygon_to_cells`, but S2's coverer is more sophisticated for tight bounds.
- **Prefix range scans.** S2 cell IDs along the Hilbert curve are dense in 1D, so consecutive IDs are usually adjacent in 2D. You can range-scan an S2 ID interval to get a cell strip, which composes very nicely with B-tree indexes. H3 IDs do **not** have this property — they are not designed for range scans.

**Practical guidance:**

- Use **H3** if you index *points* (drivers, restaurants, listings), do analytics over local neighborhoods, run a dispatch system, or render hex-bin heat maps.
- Use **S2** if you index *regions* (delivery areas, school districts, geofences), need exact polygon coverage, or want range-scannable cell IDs in a generic SQL DB.

You can run both in the same system. Uber's stack uses H3 for dispatch and surge but historically used other tools for region indexing.

## vs Geohash

[**Geohash**](../../../data-structures/geohash.md) is the simplest and oldest alternative.

| Property | H3 | Geohash |
|----------|----|---------|
| Cell shape | Hexagons + 12 pentagons | Lat/lng rectangles |
| Curve | Hex hierarchical IDs | Z-order (Morton) curve |
| Cell distortion | Very small (small variation by latitude) | Severe near poles |
| Neighbor distance | All 6 neighbors equidistant | √2× asymmetry on diagonals |
| Cell ID type | 64-bit integer | Variable-length string (or 64-bit integer) |
| Hierarchy | 1:7 (approximate) | 1:4 (exact, alternating lat/lng halving) |
| Prefix property | None | "Same prefix → same parent cell" |
| Origin | Uber 2018 | Niemeyer 2008 |

**Where H3 wins:**

- **Uniform cell area across the globe.** Geohash cells near the poles become slivers (longitude shrinks with latitude). H3 has small variation but no catastrophic distortion at the poles.
- **Better neighbor analytics.** All the hexagon advantages discussed above.
- **Pentagon-aware globally.** Geohash falls apart near the poles and the antimeridian. H3 handles them with predictable special cases (the 12 pentagons).

**Where geohash wins:**

- **Plays with any 1D index.** Geohash strings or 64-bit integer geohashes range-scan cleanly on a B-tree, sorted set, LSM-tree, or Bloom-filtered index. Redis GEO, Elasticsearch `geohash_grid`, PostGIS `ST_GeoHash` — all use this. H3 has no equivalent prefix property.
- **Trivial implementation.** Geohash is 200 lines of bit-twiddling. H3 is a sophisticated C library that you must depend on.
- **No special cases.** No pentagons. No icosahedral projection. Just lat/lng halving.

**Practical guidance:**

- Reach for **geohash** when you want to ship "find nearby" with minimal infra — your existing DB or Redis already supports it, and the 1D prefix property makes it easy.
- Reach for **H3** when neighbor-distance symmetry, smooth heat maps, or hex-bin analytics actually matter for the product.
- Reach for **S2** when you index regions globally at scale.

For Uber-scale dispatch, the math of dispatch (kNN, neighbor smoothing, surge zone aggregation) makes H3's properties pay back the dependency cost. For most apps that just need "find drivers within 1 km", geohash is more than enough.

## Resolution by Use Case

H3's 16 resolutions span 6 orders of magnitude in cell area. Picking the right one is the most important configuration decision when you adopt H3. Pick too coarse and your queries return too many cells; pick too fine and your geo-index has pointless cardinality and cells are smaller than your GPS noise.

| Use case | Resolution | Why |
|----------|-----------|-----|
| Country-level rollups | 4 (~1,770 km²) | A country has tens of cells. Easy to chart. |
| State / metro region analytics | 5 (~252 km²) | A metro area has dozens of cells. |
| City-level analytics, fleet supply heat maps | 6 (~36 km²) | A city has hundreds of cells. |
| **Surge pricing zones** | **7 (~5.16 km²)** | A neighborhood has 1–5 cells; a city quadrant ~10. Big enough for stable supply/demand averages, small enough for meaningful price differentiation. |
| District / large neighborhood | 8 (~0.74 km²) | A block fits in 2–3 cells. |
| **Driver dispatch (kNN search index)** | **9 (~0.105 km²)** | Cell side ≈ 175 m. A `grid_disk(cell, 2)` ≈ 19 cells ≈ ~500 m radius, ideal for kNN over a few hundred drivers. |
| Block-level dispatch, micro-mobility (e-scooters) | 10 (~0.015 km²) | Cell side ≈ 65 m. Dense urban; tight scooter discovery. |
| Building cluster / parking lot | 11–12 (~2,150 m²–~307 m²) | Building-level granularity. |
| Per-spot precision | 13 (~44 m²) | Parking-spot tracking. |
| Sub-meter | 14–15 | Below GPS noise; usually overkill. |

### The Two Defaults Worth Memorizing

- **Res 9 for dispatch.** Cell side ≈ 175 m, area ≈ 0.105 km². At a typical driver density in a busy urban core (a few hundred drivers per km²), a 9-cell dispatch query (`grid_disk(cell, 1)`) returns ~50–100 drivers — manageable for full ranking. The radius covered by `grid_disk(cell, 2)` is ~500 m, about the maximum reasonable pickup distance.
- **Res 7 for surge zones.** Cell area ≈ 5.16 km². A typical mid-size city has 100–300 res-7 cells, which is enough for distinct pricing zones without fragmenting the supply/demand statistics into noise.

### The Multi-Resolution Pattern

For most production systems, you don't pick one resolution — you pick **multiple** and use them together:

- **Res 9** for the in-memory dispatch geo-index (kNN search).
- **Res 7** for the surge pricing service (zone-level pricing).
- **Res 5 or 6** for analytics rollups (city-level dashboards).
- **Res 4** for cross-region routing (which datacenter handles which area).

Convert between them with `h3.cell_to_parent(cell, target_res)`. A driver pings their res-9 cell; the surge service joins it to the res-7 parent for pricing; the analytics pipeline rolls up to res-5 for dashboards. Because the parent function is constant time, this composition is cheap.

### Picking Resolution by Density

A useful heuristic: **pick the resolution where a typical query radius is covered by `k = 2..3` rings.**

- Query radius 500 m → res 9 (cell side ~175 m, `grid_disk(cell, 2)` ≈ 525 m).
- Query radius 2 km → res 8 (cell side ~460 m, `grid_disk(cell, 2)` ≈ ~1.4 km, `grid_disk(cell, 3)` ≈ ~2.1 km).
- Query radius 10 km → res 7 (cell side ~1.2 km, `grid_disk(cell, 4)` ≈ ~6.8 km).

If you can't decide between two resolutions, pick the finer one and let the `k` grow — the cost of an extra ring is only `6k` more cells, and you can always cap the search.

## Compactness

H3 has a notion of **compactness** — representing a connected set of cells with a *mixed-resolution* set, where coarse parents replace contiguous groups of children. The two API entry points are:

- `h3.compact_cells(cells)` — given a set of cells at the same resolution, return a smaller mixed-resolution set covering the same area.
- `h3.uncompact_cells(cells, target_res)` — expand the mixed-resolution set back to a single resolution.

Why this matters:

- **Storage.** A region described as "all of Manhattan" at res 9 is hundreds of thousands of cells. Compacted, it might be a few hundred mixed-resolution cells. Storing the compacted set in DB rows or in-memory uses orders of magnitude less space.
- **Network transit.** Geofence definitions sent from a server to a fleet-management dashboard, or from one service to another, are small after compacting.
- **Set operations.** Computing intersections, unions, and differences of large regions is faster when the inputs are compacted (fewer total cells to compare).

**Compactness ratio caveat.** Because the H3 hierarchy is *not* an exact 1:7 tile, compaction is not lossless in the strict sense — `compact(uncompact(set))` is not always identical to the original `set`, and there are edge cases involving pentagons. For most practical purposes (GIS applications, fleet management, surge zones) the round-trip is good enough. For applications that require exact cell-set fidelity across resolutions, do not rely on compaction.

```python
import h3

# A bounding polygon over a small area, expanded to res 9 cells
polygon = [(40.748, -73.985), (40.748, -73.965),
           (40.760, -73.965), (40.760, -73.985)]
cells_res9 = h3.polygon_to_cells(polygon, 9)
print(len(cells_res9))                # ~5,000 cells

# Compact: replace contiguous res-9 groups with res-8 or res-7 parents
compact = h3.compact_cells(cells_res9)
print(len(compact))                   # ~50 cells, mixed resolutions
```

For dispatch, you typically *don't* compact — you want a single resolution for cache locality and simple set operations. Compact when storing geofence definitions, transmitting coverage areas, or doing analytics over large regions.

## Anti-Patterns

**Treating H3 IDs as prefix-scannable.** Geohash strings have the property that "same prefix → same parent cell". H3 IDs do **not**. Two H3 cells with sequential integer IDs are not necessarily adjacent on the globe; the bit layout encodes resolution, base cell, and digit path, none of which are sortable in a useful way for spatial range scans. **Use the H3 API for parent/child traversal, not string operations or BIGINT comparisons.**

**Picking one resolution and using it for everything.** Dispatch wants ~175 m cells (res 9). Surge wants ~1 km cells (res 7). Heat maps want ~3 km cells (res 6). Country dashboards want ~22 km cells (res 4). Using a single resolution for all four use cases means either dispatch is too coarse (poor candidate sets) or analytics is too fine (noisy). Adopt multi-resolution from day one and use `cell_to_parent` to cheaply roll up.

**Assuming pentagon cells don't exist.** The 12 pentagons are out in open ocean and won't affect a city dispatch system, but they will affect global analytics, fleet logistics for shipping, and any code path that traverses the entire grid (e.g., bulk migration, "uncompact the world"). Use `is_pentagon(cell)` and the `_safe` API variants when running global passes.

**Hierarchical aggregation that requires exact containment.** Children at resolution `r+1` cover slightly more than the parent at resolution `r`. If your analytics joins compute "sum of children = parent", you will get small drift, especially at larger `k`. For tight financial accounting, do the rollup at a single resolution and don't traverse the hierarchy.

**Dispatching from a single global geo-index.** H3 makes regional sharding easy — base cell index, or any coarser parent — but if you collapse the world into one geo-index, you lose the locality benefit and create a single point of failure. Region-isolate by base cell or coarser parent.

**Using H3 for fine-grained routing.** H3 cells at res 11–12 are still meters across, far coarser than a road graph. Use a real road network (OSRM, Valhalla, GraphHopper, or in-house) for routing; use H3 only as a coarse approximation when you don't have a graph or when zoomed out.

**Confusing grid-distance with great-circle distance.** `grid_distance(a, b)` returns the integer hex steps between cells. It is not kilometers; it is cells. A grid distance of 5 at res 9 is roughly `5 × 175 m ≈ 875 m`, but only if the path doesn't cross resolution distortion zones. **Always reduce to lat/lng + haversine for any user-visible distance.**

**Storing only the H3 cell, not the lat/lng.** Like geohash, H3 is lossy — a cell is a hex polygon, not a point. If you ever need to draw a marker, compute exact distance, or feed a routing engine, you need the original lat/lng. Store both: lat/lng for truth, H3 for indexing.

**Using H3 for indexing regions that often span the antimeridian or polar zones.** H3 handles these correctly via the icosahedral projection, but pentagon-aware code paths are slower. If your business is shipping or polar logistics, profile carefully.

**Skipping the safe `grid_disk`.** `grid_disk_unsafe` is faster but produces wrong results around pentagons. The default `grid_disk` is safe; the unsafe variant exists only for hot paths where you have proven the disk doesn't touch a pentagon. **For dispatch, always use the safe variant.**

## Related

- [`../../../data-structures/geohash.md`](../../../data-structures/geohash.md) — the simplest space-filling curve for geo-indexing; H3's main alternative when neighbor symmetry doesn't matter.
- [`../../../data-structures/quadtrees-and-r-trees.md`](../../../data-structures/quadtrees-and-r-trees.md) — alternative tree-based 2D indexes; better for dynamic regions and rectangle queries, worse for prefix-keyed B-tree storage.
- [`../design-uber.md`](../design-uber.md) — the parent case study; H3 is the geo-index core of Uber's dispatch architecture.

## References

- [H3 official site — h3geo.org](https://h3geo.org/) — documentation, resolution tables, API references, and conceptual overviews.
- [H3 documentation — Cell areas at each resolution](https://h3geo.org/docs/core-library/restable/) — exact cell area, edge length, and worldwide cell counts per resolution.
- [H3 GitHub — github.com/uber/h3](https://github.com/uber/h3) — the canonical C library, with Java/Python/JS/Go/Rust bindings linked from the README.
- [Uber Engineering — "H3: Uber's Hexagonal Hierarchical Spatial Index"](https://www.uber.com/blog/h3/) — Isaac Brodsky's launch post explaining why hexagons, the icosahedral projection, and the 1:7 aperture.
- [Uber Engineering — eng.uber.com/h3](https://www.uber.com/blog/h3/) — alternate URL for the H3 launch post.
- [H3 docs — Indexing fundamentals](https://h3geo.org/docs/core-library/h3Indexing) — the 64-bit cell index layout, resolutions, base cells, and digit slots.
- [H3 docs — Traversal API](https://h3geo.org/docs/api/traversal/) — `grid_disk`, `grid_ring`, `grid_distance`, `grid_path_cells`, and the safe/unsafe variants.
- [H3 docs — Hierarchy API](https://h3geo.org/docs/api/hierarchy/) — `cell_to_parent`, `cell_to_children`, `compact_cells`, `uncompact_cells`.
- [H3 docs — Pentagons](https://h3geo.org/docs/library/restable/) — the 12 pentagons, their properties, and the safe/unsafe trade-off.
- [Google S2 Geometry Library — Overview](http://s2geometry.io/about/overview) — the Hilbert-curve-on-a-cube alternative; H3's main competitor at planet scale.
- [Google S2 — about](http://s2geometry.io/) — S2 home, with explanations of the Hilbert curve and the cube projection.
- [`h3-pg` — H3 PostgreSQL extension](https://github.com/zachasme/h3-pg) — native H3 type and operators for Postgres.
- [Foursquare on H3](https://medium.com/foursquare-direct/foursquare-h3-and-the-hexagons-bb2a3a6e6b88) — production write-up from another large user, with notes on hierarchical aggregation.
- [Wikipedia — Geodesic grid](https://en.wikipedia.org/wiki/Geodesic_grid) — the broader family of icosahedral and hex-on-sphere grids that H3 belongs to.
