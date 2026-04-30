---
title: "Uber Deep Dive — Matching and Dispatch"
date: 2026-04-29
updated: 2026-04-29
tags: [system-design, case-study, uber, deep-dive, matching, dispatch]
---

# Uber Deep Dive — Matching and Dispatch

**Date:** 2026-04-29 | **Updated:** 2026-04-29
**Tags:** `system-design` `case-study` `uber` `deep-dive` `matching` `dispatch`

## Table of Contents

- [Summary](#summary)
- [Overview — Matching Is a Constrained Optimization, Not a Lookup](#overview--matching-is-a-constrained-optimization-not-a-lookup)
- [Greedy Nearest-Driver vs Batched Assignment](#greedy-nearest-driver-vs-batched-assignment)
  - [Greedy Nearest-Driver](#greedy-nearest-driver)
  - [Batched Assignment with the Hungarian Algorithm](#batched-assignment-with-the-hungarian-algorithm)
  - [Network Flow Generalization](#network-flow-generalization)
  - [The 30-Second Batching Window](#the-30-second-batching-window)
- [DISCO Geosearch — The Dispatch Microservice](#disco-geosearch--the-dispatch-microservice)
- [Hex-Based Candidate Retrieval — k-Ring Around Pickup](#hex-based-candidate-retrieval--k-ring-around-pickup)
- [Driver Acceptance / Rejection Loop](#driver-acceptance--rejection-loop)
- [Pool / Share vs UberX vs Single-Rider Matching](#pool--share-vs-uberx-vs-single-rider-matching)
- [Latency Budget — The 5-Second Game](#latency-budget--the-5-second-game)
- [Fairness Constraints — Driver Utilization and Rider Wait Time](#fairness-constraints--driver-utilization-and-rider-wait-time)
- [Multi-Objective Optimization](#multi-objective-optimization)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

Matching is the part of Uber's stack where every other layer's hard work either pays off or is wasted. The geo-index keeps drivers retrievable; the location pipeline keeps positions fresh; surge keeps supply where demand is. None of it matters if the dispatcher picks the wrong driver. "Wrong" is itself a multi-dimensional concept — wrong for the rider's ETA, wrong for the driver's earnings fairness, wrong for the marketplace's long-run liquidity, wrong for fraud and acceptance models. Matching is therefore not a lookup. It is a constrained optimization that runs continuously under a tight latency budget against an adversarial environment of stale pings, driver decline behaviour, surge boundaries, and competing concurrent requests.

The architecture has two regimes that coexist in a real production system: a **greedy nearest-driver path** for low-density markets and as a fallback, and a **batched assignment path** that holds new requests inside a small time window (typically a few seconds, with 30 seconds appearing as a high-water mark in some markets and product modes), then solves a **minimum-cost bipartite matching** problem — Hungarian-flavored at small batch sizes, network-flow-based at scale. The batched approach buys assignment quality at the cost of a few seconds of perceived wait. Greedy minimizes wait but produces myopic assignments that hurt the marketplace.

Around this core sit a handful of mechanisms that any HLD-level discussion must address: **hex-based retrieval** via H3 k-ring search to bound the candidate pool, **DISCO** (Uber's dispatcher service, since evolved through several generations) as the geosearch primitive, an **acceptance / rejection loop** that re-dispatches when drivers decline, distinct **product modes** (UberX, UberPool/Share, UberBlack, Reserve, accessibility), a **5-second matching latency target**, and **fairness constraints** that prevent the optimizer from systematically starving any side of the marketplace. This deep dive unpacks each piece.

This is a companion to [`../design-uber.md`](../design-uber.md), expanded around the matching and dispatch stage itself.

## Overview — Matching Is a Constrained Optimization, Not a Lookup

A naive design treats matching as: "Find the closest available driver and offer them the trip." The implementation is one geo-query and a push notification. This works for the first thousand rides per day in a sparse market and falls over almost immediately as density and concurrency grow.

Three structural reasons it fails:

1. **Locally optimal is globally suboptimal.** When two ride requests land within seconds of each other, greedy assignment of the first to the closest driver may leave the second request with a much worse option. Across thousands of concurrent requests, the cumulative "ETA debt" produced by greedy assignment is large. The right framing is: minimize the total cost across all open requests and available drivers, treating the matching as a snapshot bipartite optimization rather than a sequence of independent decisions.

2. **Cost has many components.** ETA to pickup is the biggest single term, but a real cost function blends driver acceptance probability, recent driver earnings (fairness), surge-zone consistency, product eligibility, vehicle capacity, fatigue policy, anti-fraud signals, and rider wait time so far. Reducing all of this to "shortest distance" loses the marketplace.

3. **The environment is non-stationary inside the request lifecycle.** Drivers decline; new requests arrive; positions move; surge updates. A single-shot assignment is wrong by the time it lands. Production systems run a continuous loop with re-dispatch on decline and rolling re-optimization across the open request pool.

The mental model is closer to **air traffic control** than to nearest-neighbor: a controller continuously assigns aircraft to runways under a multi-objective cost function with hard safety constraints and soft fairness constraints, knowing that any assignment is provisional until the pilot accepts and the slot is realized.

## Greedy Nearest-Driver vs Batched Assignment

The two regimes are not philosophical alternatives — they are operating modes selected per market, per product, and per traffic state.

### Greedy Nearest-Driver

The simplest possible policy:

```python
def greedy_dispatch(request):
    candidates = geo_index.k_nearest(
        request.pickup, k=10, product=request.product, max_eta_sec=600
    )
    candidates = filter_eligible(candidates, request)
    candidates.sort(key=lambda d: estimated_eta(d, request.pickup))
    for driver in candidates:
        if try_offer(driver, request):
            return driver
    return None  # no acceptance
```

Properties:

- **Latency**: very low. One k-nearest query plus a sequential offer loop. Suits sparse markets where the candidate set is small and acceptance is the dominant variable.
- **Determinism**: easy to reason about, easy to debug, easy to audit when a regulator asks "why did you pick that driver."
- **Failure mode**: cumulative myopia. Globally suboptimal in dense markets where simultaneous requests share the same candidate pool. The fastest way to see this in practice is at airport pickups: greedy hands the closest driver to the first request that lands, ignoring that two more requests are coming in within seconds and would have been better served by that same driver.

Greedy remains useful as:

- The default policy in low-density markets and off-peak hours.
- A fallback when the batched optimizer is unhealthy or has missed its deadline.
- The serving algorithm for a small fraction of traffic in a holdout cell, used to A/B-test improvements to the optimizer.

### Batched Assignment with the Hungarian Algorithm

In dense markets, the dispatcher accumulates open ride requests inside a short time window (typically a few seconds) and the available drivers eligible for those requests, then solves a **minimum-cost bipartite assignment**:

- One side of the bipartite graph: requests.
- Other side: drivers.
- Edge weight: cost of matching that pair, where cost combines ETA, acceptance probability, fairness, surge, and product eligibility (infeasible matches get an infinite or very large weight).

The classical algorithm for this problem is the **Hungarian algorithm** (Kuhn 1955, Munkres 1957), a polynomial-time exact solver for the linear assignment problem with O(n³) complexity. For a balanced problem with n requests and n drivers, it returns the minimum-cost perfect matching.

Why this beats greedy:

- It considers all open requests simultaneously, so swapping a driver from one request to another to reduce total cost is natural.
- It avoids the tail of "request K gets stuck with the worst driver" because all assignments are decided together.
- It generalizes cleanly to imbalanced problems (more requests than drivers, or vice versa) by adding dummy nodes with appropriate costs.

Why it has practical limits:

- **O(n³) time** is fine for batches of a few hundred but uncomfortable at thousands. Real markets at peak generate batches in the hundreds-to-low-thousands range; sharding by region keeps each batch manageable.
- **Pure cost minimization ignores constraints** like driver fatigue caps, fairness floors, capacity for Pool, or geographic constraints (don't dispatch a driver across a river if a bridge isn't crossable). These show up as hard penalties or as separate filtering passes before the optimizer sees the candidate set.
- **Cost values are estimates.** ETA is from a routing engine; acceptance probability is from a model; surge sensitivity is from a behavioral model. Errors in these inputs cascade into the matching.

Reference: the canonical academic discussion is the [Kuhn-Munkres / Hungarian algorithm](https://en.wikipedia.org/wiki/Hungarian_algorithm); [Brilliant.org — Hungarian Maximum Matching](https://brilliant.org/wiki/hungarian-matching/) is a readable introduction.

### Network Flow Generalization

When the matching problem grows beyond bipartite (e.g., a driver can be considered for the next ride after dropping off the current one, or a Pool driver mid-route can be considered for additional pickups), the right framing is **min-cost max-flow** on a directed graph:

- Source → request nodes (capacity 1).
- Request nodes → driver nodes (capacity 1, cost = match cost).
- Driver nodes → sink (capacity = how many rides the driver can take in the planning horizon).

Algorithms: successive shortest paths, cycle-canceling, or scaling variants. Modern OR libraries (Google [OR-Tools](https://developers.google.com/optimization), commercial solvers like Gurobi) handle these problems efficiently for the scales encountered in regional dispatch.

For Pool/Share specifically, the problem is no longer purely bipartite — riders combine into shared trips, and the optimizer must reason about route insertion costs (where in the existing route does a new pickup fit?). This is closer to **online vehicle routing with time windows** and is attacked with insertion heuristics + local search rather than exact assignment.

Reference: a classic textbook treatment is Ahuja, Magnanti, Orlin, *Network Flows: Theory, Algorithms, and Applications* (Prentice Hall, 1993); see also [Stanford CS261 lecture notes on min-cost flow](https://web.stanford.edu/class/archive/cs/cs261/cs261.1162/).

### The 30-Second Batching Window

The batching window is the key tunable. Too short, and the optimizer reduces to greedy with extra steps; too long, and rider perception of "the app is thinking" degrades.

- **Sub-second windows (e.g., 1–3 seconds).** Common for airports, dense urban centers, and peak hours where request volume is high enough that even a one-second window collects a useful batch.
- **5–10-second windows.** A typical default. Long enough to gather multiple requests; short enough that the rider is still seeing "Finding driver…" within a single screen view.
- **Up to 30-second windows.** Reported in some Uber engineering writeups as a high-water mark for batched optimization in particularly complex markets and for shared products. The trade-off is explicit: the rider waits a few extra seconds in exchange for a measurably better assignment.

Operational reality: the window is **adaptive**. Light traffic → fall back to greedy or near-greedy with a tiny window. Heavy traffic → lengthen the window up to a market-specific cap. The dispatcher monitors batch quality (e.g., what fraction of assignments would have changed if the window were shorter) and tunes per region.

A pseudocode sketch of the batched loop:

```python
def batched_dispatch_loop(region, window_ms=5000):
    while True:
        deadline = now_ms() + window_ms
        open_requests = []
        while now_ms() < deadline:
            req = request_queue.poll(deadline - now_ms())
            if req:
                open_requests.append(req)

        if not open_requests:
            continue

        candidate_pool = collect_candidates_for_batch(open_requests)
        cost_matrix = build_cost_matrix(open_requests, candidate_pool)
        # apply Hungarian or min-cost flow
        assignment = solve_assignment(cost_matrix)
        for req, driver in assignment:
            send_offer(driver, req)
```

Two practical refinements layer on top:

- **Rolling, not fixed, windows.** Some implementations use a sliding-window optimizer that re-solves every N milliseconds against the current open set, so a request that arrives partway through an interval doesn't wait the full window before being considered.
- **Priority lanes.** Premium products, accessibility requests, or already-delayed requests jump the queue with a tighter window or direct-greedy fallback. The marketplace has SLAs by product tier.

A subtler refinement is **two-pass batching**: a first pass with a tight window catches the easy assignments (high-confidence pairs where ETA is small and acceptance probability is high) and dispatches them immediately; a second longer pass re-solves the remaining harder requests against the residual driver pool. The motivating insight is that the easy 80% of assignments do not benefit from waiting in a batch — only the contested 20% do. Splitting the optimization respects this and reduces median rider wait without sacrificing assignment quality on the hard tail.

A complementary technique is **lookahead via demand forecasts.** When the dispatcher has a model of the next 30–60 seconds of incoming demand (informed by historical patterns plus real-time signals), it can deliberately leave a near-optimal driver unassigned in the current batch if a higher-value request is expected to arrive imminently. This is where batched matching shades into **stochastic dynamic programming**, and is the area where Uber and competitors invest most heavily on the optimization side.

## DISCO Geosearch — The Dispatch Microservice

Uber's historical dispatch service is referred to in their engineering writing as **DISCO** (Dispatch Service / Optimization). The system has gone through multiple generations; the high-level role has remained:

- **Geosearch primitive.** Given a pickup point, return a candidate set of drivers indexed by their current cell, filtered by product eligibility and other simple predicates.
- **Cost evaluation.** For each candidate, compute the cost components (ETA, acceptance probability, surge consistency, fairness terms) by calling out to ETA, pricing, and ML prediction services.
- **Optimizer driver.** Run the assignment algorithm — greedy or batched — and produce one or more offers per request.
- **Offer state machine.** Track each outstanding offer (pending, accepted, declined, expired), and re-dispatch on decline or expiry until the request is satisfied or canceled.

DISCO is sharded geographically. Every region has its own dispatcher, with cross-region traffic handled via lead-region election for trips that straddle borders (airport runs, intercity hops). This is the same isolation pattern that limits blast radius: a dispatcher outage in one region does not affect another.

The classical Uber engineering description ("[Engineering Real-Time Pricing — Surge](https://www.uber.com/blog/engineering-surge-pricing/)" series, plus older posts on the marketplace platform) covers the request flow end-to-end: a ride request lands, the geosearch primitive returns candidates, the optimizer assigns, the offer goes out, the driver accepts, the trip materializes. More recent Uber Engineering posts describe ML and policy layers stacked on top of this base, but the geosearch + optimize + offer skeleton has remained.

A subtle architectural property: DISCO is a **stateful service**. It holds the open request set, the outstanding offer set, and a view of driver availability that is updated by the location pipeline. This is in tension with stateless-microservice purism. The state is sharded by region and by H3 cell so that a single shard's state is bounded; checkpoints to a durable store give recovery. Stateless is not a virtue when the workload itself is fundamentally stateful.

A few operational properties that fall out of this stateful design:

- **Sharding key is geographic, not user-based.** A request coming from cell C is routed to the dispatcher shard that owns C. This co-locates the request with the relevant driver state and avoids cross-shard fan-out for the common case.
- **Cross-shard requests are explicit.** When a candidate set spans multiple shards (the request is near a shard boundary), the dispatcher fans out to neighbor shards and merges results. This is uncommon if shards are sized correctly relative to typical search radius, but the path must exist.
- **Failover is geographic.** A shard's state is checkpointed continuously to a regional store; on shard restart, state is reloaded from the checkpoint plus any pings/requests that arrived since. The reload must complete inside the latency budget for the affected cells, which caps shard size.
- **State drift detection.** The dispatcher's view of driver availability lags reality slightly (location pipeline propagation delay, network jitter). Periodic reconciliation against the authoritative driver session store catches drift before it produces dispatch errors (offering a trip to a driver who has already gone offline).
- **Hot shard mitigation.** Times Square at NYE concentrates an absurd amount of state into one shard. Mitigation is dynamic re-sharding: split the hot shard into smaller cells temporarily, with the geo-index aware of the split. The split is reversed when load returns to normal.

The dispatcher's external API surface is small but load-bearing:

```text
RPC dispatch.MatchRide(RideRequest)         → DriverAssignment | NoDriverAvailable
RPC dispatch.GetOpenOffers(driver_id)       → [Offer]
RPC dispatch.RecordOfferDecision(offer_id, accept|decline)
RPC dispatch.RegisterDriverState(driver_id, state, h3_cell)
RPC dispatch.GetDispatchStats(region)       → utilization, p50/p99, queue depth
```

Each of these has its own SLA. `MatchRide` is the user-facing latency-critical path. `RecordOfferDecision` is in the loop of every offer cycle and must be fast. `GetDispatchStats` is asynchronous monitoring, off the hot path. Mixing these on the same threadpool without isolation produces cross-impact incidents.

## Hex-Based Candidate Retrieval — k-Ring Around Pickup

The geosearch primitive's job is to answer: "Within a target radius of this pickup, which drivers are eligible for this request?"

H3 (hexagonal hierarchical spatial index) gives this a clean shape:

```python
def find_candidates(pickup, request, max_rings=4):
    cell = h3.latlng_to_cell(pickup.lat, pickup.lng, res=9)
    candidates = []
    for ring in range(max_rings + 1):
        # grid_disk returns cell + all neighbors at ring distance r
        neighbor_cells = h3.grid_disk(cell, ring)
        for c in neighbor_cells:
            shard_drivers = geo_index.drivers_in_cell(c)
            for d in shard_drivers:
                if eligible(d, request):
                    candidates.append(d)
        if len(candidates) >= TARGET_K:
            break
    return candidates
```

Why this shape:

- **Hexagons have uniform neighbor distance.** Every hex has 6 neighbors at one ring, all at roughly equal distance. Square grids have 4 edge-neighbors and 4 diagonal-neighbors at different distances, which distorts ring-based search.
- **`grid_disk(cell, k)` is the canonical k-ring API** in H3 — it returns the cell itself plus all hexagons within k steps. Calling it with increasing k gives concentric shells.
- **Cell-sharded geo-index turns each shell expansion into a localized lookup.** No range scan; just a hash-keyed read per cell.

Resolution choice (per [H3 documentation](https://h3geo.org/docs/core-library/restable)):

| H3 res | Avg edge length | Avg cell area | Typical use |
|--------|-----------------|---------------|-------------|
| 7      | ~1.2 km         | ~5.16 km²     | Surge zones, coarse routing |
| 8      | ~460 m          | ~0.737 km²    | Initial dispatch search shell |
| 9      | ~174 m          | ~0.105 km²    | Fine-grained pickup matching |
| 10     | ~66 m           | ~0.015 km²    | Last-mile precision (rare) |

Practical search starts at res 9 for the immediate cell, expands rings until a target candidate count is met or a max-radius cap is hit. Caps prevent the search from sweeping the whole city when supply is genuinely scarce — better to surface "no driver found" than to dispatch a driver 20 minutes away.

Uber's own [H3 introduction blog](https://www.uber.com/blog/h3/) documents the design rationale: hexagons were chosen specifically for the uniform neighbor distance and the cleaner spatial-aggregation behaviour compared to squares or triangles. The library is open source at [github.com/uber/h3](https://github.com/uber/h3) and has stable bindings in many languages.

A quirk worth knowing: H3's hierarchy uses a 1:7 aperture, and because hexagons cannot tile recursively without a small projection adjustment, **hexagonal cells do not nest perfectly across resolutions**. A child hex at res 10 may overlap two parents at res 9. For dispatch this is rarely a problem (you operate at one resolution at a time) but matters when aggregating analytics across resolutions; treat parent-child links as approximate. See the [H3 indexing internals](https://h3geo.org/docs/core-library/overview) for the underlying icosahedral projection.

A few additional retrieval-time concerns:

- **Eligibility is a multi-dimensional filter, not a single predicate.** Product (UberX, Black, WAV), vehicle capacity, driver session state (online vs en-route vs on-trip), driver acceptance recent-history, regulatory constraints (some cities require a TLC license for certain pickup zones), surge consistency. The geo-index entry stores enough denormalized state to answer most filter questions in O(1) per candidate; complex policy filters happen after the cheap geometric filter.
- **Radius caps are non-negotiable.** A pickup with no nearby supply should fail fast with "no driver available" rather than dispatch a 30-minute pickup. The radius cap depends on product (UberBlack tolerates farther dispatch than UberX) and on market (rural markets accept longer pickups than dense urban ones). The cap also feeds back into surge: if many requests in a cell repeatedly hit the radius cap with no supply, surge multiplier should rise.
- **Negative caching for empty cells.** A k-ring expansion that finds no eligible drivers in cells 0..2 is unlikely to find them in 0..2 again on the very next request. Short-TTL negative caching (a few hundred milliseconds) on per-cell emptiness reduces redundant work during supply droughts.
- **Concurrent-request contention.** Two simultaneous requests in the same cell competing for the same handful of drivers must coordinate. The simplest mechanism is a per-driver "pending offer" lock: while an offer is outstanding to a driver, that driver is not visible in geosearch results for other requests. The lock has a short TTL bounded by the offer expiry.

For the broader geo-indexing context — including geohash and S2 comparisons — see the parent doc's section [`../design-uber.md#1-geo-indexing-with-h3`](../design-uber.md#1-geo-indexing-with-h3) and the planned companion [`./h3-geo-indexing.md`](./h3-geo-indexing.md).

## Driver Acceptance / Rejection Loop

Once a candidate driver is selected, dispatch sends a **time-bounded offer** over the driver's persistent connection. The driver app shows an offer card with pickup info, ETA, expected fare range, and a countdown (typically 10–15 seconds). The driver accepts, declines, or lets it expire.

The state machine inside DISCO for a single offer:

```text
PENDING ─accept─► ACCEPTED ─► trip flow
   │
   ├─decline──► DECLINED ─► re-dispatch
   │
   └─expire───► EXPIRED  ─► re-dispatch
```

Re-dispatch is the path that runs when an offer fails. It can:

- **Pick the next-best candidate** from the existing batch's solution. Cheap, fast, but may produce a strictly worse assignment.
- **Re-solve the assignment problem** with the failed driver removed and the request still open. Better quality, more expensive.
- **Roll the request into the next batching window**. Acceptable if the wait remains within the latency budget.

Acceptance probability is itself a learned signal. Drivers vary widely in how often they accept, and the variation is correlated with trip features: short pickups, off-peak hours, surge zones, rider rating, prior experience with that pickup type. The cost function for matching folds in `p_accept(driver, request)` so that the optimizer prefers a slightly farther driver who is likely to accept over a closer driver who routinely declines.

This creates a reinforcement-learning-shaped dynamic that has to be managed:

- **Penalizing drivers who decline** can collapse acceptance rates if drivers feel coerced; many regulators and labor frameworks restrict this. Uber's policies have evolved here.
- **Rewarding drivers who accept** with priority on subsequent matches creates a different bias and must be balanced against fairness.
- **Hiding trip details** (e.g., destination) before acceptance can raise acceptance rates but produces friction with regulatory transparency requirements.

The system designer's lesson: the acceptance loop is not just a retry mechanism. It is a feedback signal that the matching cost function consumes, and a policy surface where market design decisions live.

A few additional production realities:

- **Stale offers.** If the network is flaky and the driver's accept arrives after the offer has been re-dispatched, the system needs to detect the race and choose deterministically. Idempotency keys and an authoritative "first-acceptor-wins" check at the dispatcher are the typical mechanism.
- **Cancellation under contention.** A driver who accepts then immediately cancels (within seconds) effectively occupies the offer slot. Repeat behaviour is detected and feeds into the acceptance model.
- **Offline mid-offer.** A driver going offline between offer send and accept has to be detected within the offer expiry, not after. The session keepalive is part of the offer state machine, not a separate concern.

## Pool / Share vs UberX vs Single-Rider Matching

Different products produce different matching problems. The dispatcher cannot apply a single algorithm uniformly.

**UberX / single-rider products.** The standard bipartite assignment problem described above. Each request matches one driver; cost is dominated by ETA and acceptance probability. The Hungarian algorithm or its network-flow generalization is appropriate.

**UberBlack / Premium products.** Same shape as UberX, but with a smaller, vetted driver pool, tighter product eligibility, and stricter quality constraints. The candidate set is smaller; the cost function weights driver rating and vehicle type more heavily.

**UberPool / Share.** The hardest variant. Riders combine into shared trips; the optimizer must reason about:

- **Detour budgets.** Each rider has a max additional time/distance the system will impose (typically a small number of minutes). Violating this is a hard constraint.
- **Capacity.** Vehicle capacity limits how many concurrent riders can share. Often two riders + driver.
- **Route insertion.** When considering adding a new pickup to an in-progress shared trip, the optimizer evaluates "where in the planned route does the new pickup go, and what is the marginal cost?"
- **Time windows.** Riders have soft pickup ETAs that turn into hard SLAs after a threshold.

Algorithmic shape:

1. Maintain a small queue of pending Pool requests.
2. For each new request, generate candidate vehicles: empty Pool drivers (treated like UberX), or already-on-Pool drivers whose detour cost to add this rider is within the budget.
3. Score candidates by **system cost**: total detour added across all riders, vehicle utilization, expected revenue, fairness across drivers.
4. Insert greedily or with local search (2-opt swaps within and across vehicles), bounded by a tight latency budget.
5. Fall back to a single-rider product if no Pool match is found within the window.

This is closer to **online dynamic vehicle routing** (DVRP) than to bipartite matching. The literature (Solomon's insertion heuristics, large neighborhood search) is the right starting point.

Reference: a useful starting point is [Ma, Zheng, Wolfson — T-share: A large-scale dynamic taxi ridesharing service](https://www.cs.uic.edu/~boxu/mp2p/gpx_paper/Tshare_icde2013.pdf) (ICDE 2013). The Lyft engineering blog post on [Matchmaking in Lyft Line](https://eng.lyft.com/matchmaking-in-lyft-line-9c2635fe62c4) covers the same problem from a competitor's perspective; Uber Engineering's [Forecasting at Uber](https://www.uber.com/blog/forecasting-introduction/) touches on the demand-side modeling.

**Reserve / scheduled rides.** A different shape entirely: the request is known in advance, so dispatch can solve **future** assignment problems and pre-position drivers. This becomes a planning problem, not a real-time one, and is typically handled by a different optimizer pipeline.

**Accessibility products (UberWAV, etc.).** The candidate pool is restricted to specifically equipped vehicles. The matching algorithm is the same as UberX, but the candidate retrieval step has a tighter eligibility filter and may need to expand to a wider radius because supply is sparser.

Cross-product interactions are easy to underestimate. A driver may be eligible for multiple products (UberX and Comfort, for example) and the dispatcher must decide which product's queue to draw from. A naive policy ("highest expected revenue first") starves the lower-tier product during peaks and degrades the marketplace's lower-price experience. Production systems use product-mix policies: per-region targets for the fraction of drivers serving each product, soft cost adjustments that reduce the bias toward high-revenue products, and explicit minimum-availability guarantees for accessibility products that override revenue optimization entirely.

A worked example of the Pool insertion calculation:

```text
Existing route:  pickup_A → dropoff_A → pickup_B → dropoff_B
                 (rider A on board, rider B waiting for pickup)

New request:     pickup_C, dropoff_C   (max detour: 6 minutes)

Insertion options (subset):
  1. ... → pickup_C → dropoff_C → pickup_B → dropoff_B → dropoff_A
  2. ... → pickup_B → pickup_C → dropoff_B → dropoff_A → dropoff_C
  3. ... → pickup_B → dropoff_B → pickup_C → dropoff_C → dropoff_A
  ...

For each option:
  - Compute new total route time.
  - For each on-board rider, compute their detour vs original ETA.
  - Reject options where any rider's detour > budget or capacity exceeded.
  - Score remaining options by total system cost.

Pick the lowest-cost feasible option. If none feasible, fall back to UberX.
```

The combinatorial space is bounded (a vehicle holds two riders, so insertion has ~10 candidate orderings) but evaluating each requires a routing call. Caching ETAs at the segment level and using cheap heuristics to prune obviously-bad orderings keeps the per-request cost in the budget.

## Latency Budget — The 5-Second Game

The end-to-end matching budget — request landing to "driver assigned" — is roughly **5 seconds at p99** for standard products in dense markets, with tighter budgets for Premium and looser ones for shared products. Decomposition:

| Stage | Budget | Notes |
|-------|--------|-------|
| Request ingest + create trip row | 100–200 ms | Idempotency dedupe, OLTP write, outbox event |
| Geosearch (k-ring expansion) | 50–150 ms | In-memory geo-index, sharded by H3 cell |
| Candidate filtering + feature lookup | 100–300 ms | Eligibility, recent acceptance, fairness state |
| ETA computation per candidate | 100–500 ms | Routing engine + ML overlay; batched per candidate |
| Cost matrix + optimizer | 50–500 ms | Greedy: ms; Hungarian on 100s: 10s of ms; flow at scale: 100s of ms |
| Offer dispatch (push to driver app) | 50–150 ms | WS push, driver-side render |
| Driver decision window | 10–15 s (separate clock) | Counted against rider wait, not match latency |
| Re-dispatch on decline | another budget cycle | Repeats the above with reduced candidate pool |

Two important subtleties:

1. **The 5-second match latency is the *system's* time to produce an offer**, not the rider's perceived wait. The rider's perceived wait includes the driver's decision window and any re-dispatch cycles. A typical rider-side "Finding driver" wait runs 10–30 seconds in healthy conditions.
2. **Batched optimization explicitly trades a few seconds of latency for assignment quality**, and that trade is acceptable because the rider sees a static "Finding driver…" UI during the window. Beyond a few seconds, the experience degrades visibly; at 30+ seconds it feels broken.

Engineering moves that buy match latency:

- **Pre-compute everything that isn't request-specific.** Driver positions are kept hot in memory; product eligibility is denormalized into the geo-index entry; surge zones are pre-aggregated; routing graphs are built with contraction hierarchies for fast shortest paths.
- **Bound every fan-out.** ETA computation per candidate is the most variable cost; cap the per-candidate timeout and proceed with whatever returned.
- **Co-locate dispatcher with geo-index shards.** A cross-AZ hop on every k-ring expansion is fatal; same-AZ or same-rack placement keeps tail latency tight.
- **Cache ETAs at the cell level.** Within a batching window, the driver→pickup ETA changes slowly; reuse across candidates whose origin shares the same source cell.
- **Async re-dispatch.** When a driver declines, kick off the next-best offer asynchronously rather than blocking the rider's WS update; the UI can show "Still finding driver" without revealing the loop.

Beyond p99, the **tail beyond p99.9** is what causes the worst rider experiences ("the one time I waited two minutes for a driver"). Tail mitigations: hedged requests (offer to two candidates simultaneously, take the first acceptance), aggressive timeouts on slow ETA computations, fallback to a static distance-based ETA when the routing engine misses its budget.

Hedging deserves its own discussion. The classical Tail at Scale (Dean & Barroso, CACM 2013) framing applies directly: at very high request volumes, the slowest dependency dominates p99, and even a small probability of slowness in any one downstream call produces a large p99 effect. Two hedging patterns are common in dispatch:

- **Hedged ETA computation.** Send the same routing request to two replicas; take the first response. Doubles compute cost but flattens the tail when one replica is slow.
- **Hedged offers in low-acceptance contexts.** When historical acceptance is below a threshold, send the offer to the top-1 and top-2 candidates simultaneously, accept the first response. Wastes one offer slot per request but cuts re-dispatch latency in half. Used sparingly because over-hedging produces driver confusion.

The per-stage budgets in the table are also subject to **per-region calibration.** Dense urban markets push tighter budgets (more requests, smaller margin for slow tail responses); rural markets accept looser budgets (sparse supply means re-dispatch is more common and the re-dispatch cycle dominates anyway). A single global SLO is wrong; per-market SLOs with per-market dashboards are right.

A useful frame: every component of the budget table is a **separate negotiation** between dispatch and an upstream service. Geo-index team owns the geosearch budget; ETA team owns the routing budget; ML platform owns the prediction budget. When any of these slips, dispatch p99 slips. Operating dispatch at scale is partly a continuous coordination problem with these upstream teams: their SLOs become dispatch's SLO components, and changes to their internals (model upgrades, infrastructure migrations) are dispatch incidents waiting to happen if not managed jointly.

## Fairness Constraints — Driver Utilization and Rider Wait Time

Pure cost minimization, even with multi-objective weights, can produce systematic unfairness:

- **Driver-side**: a few high-acceptance-rate drivers near hot zones get all the trips while others sit idle.
- **Rider-side**: rider X in a low-density area waits twice as long as rider Y in a hot zone, even though both pay the same product price.
- **Geographic**: rides originating from underserved neighborhoods get worse ETAs because the optimizer always pulls supply toward higher-revenue corridors.

Fairness is enforced as **explicit constraints** layered onto the optimization, not as a hope that the cost function gets it right.

Driver-side fairness mechanisms commonly seen in this class of system:

- **Recent earnings floor.** A driver who has earned much less than the regional median in the last hour gets a small priority boost in the cost function. The boost is bounded (don't sacrifice too much rider experience) and decays as earnings catch up.
- **Trip type rotation.** A driver who has had three short trips in a row is biased toward the next long trip (and vice versa) to even out per-driver revenue.
- **Idle time fairness.** A driver who has been idle for 20 minutes outranks a driver idle for 5 minutes when both are equally close, breaking ties in favor of utilization fairness.
- **Fatigue caps.** A driver who has been online for 10 hours has reduced (or capped) dispatch eligibility, both for safety and to spread work across the supply base.

Rider-side fairness mechanisms:

- **Wait-time aging.** A request that has been open for longer accumulates a priority bonus in the optimizer. This prevents long-pending requests from being repeatedly leapfrogged by fresher requests in the same batch.
- **Geographic coverage targets.** Underserved zones get supply nudges via surge tuning and via dispatch policy: a driver in a hot zone may be preferentially offered a trip whose pickup is in an adjacent cooler zone, repositioning supply organically.
- **Equal product service quality.** Across regions, the median wait time for the same product tier is monitored as a fairness metric; persistent disparities trigger market-design interventions (more aggressive surge, driver incentive campaigns, education for new drivers).

A useful design principle: **fairness is a property of the policy, not of any single match.** Evaluating fairness requires aggregating outcomes across many matches over time. The optimizer enforces it through soft cost terms and hard caps; the marketplace team monitors it via dashboards and adjusts knobs.

This is also where regulatory pressure shows up. Several jurisdictions (NYC TLC, EU labor frameworks) impose explicit fairness floors: minimum earnings per active hour, transparency on cancellation policies, anti-discrimination requirements. These translate into hard constraints inside the optimizer, not optional features.

## Multi-Objective Optimization

The cost function is rarely a single number. It is a weighted combination of objectives, with weights that themselves are policy levers:

```python
cost(driver, request) = (
    w_eta            * eta_seconds(driver, request)
  + w_acceptance     * (1.0 - p_accept(driver, request))
  + w_fairness       * fairness_term(driver)
  + w_surge          * surge_consistency_term(driver, request)
  + w_capacity       * capacity_term(driver, request)             # for Pool
  + w_wait           * (1.0 / (1.0 + rider_wait_so_far))
  + w_fraud          * fraud_risk(driver, request)
  + LARGE_PENALTY    * (1 if not eligible(driver, request) else 0)
)
```

Each `w_*` is tuned via online experiments and per-region calibration. A few worth flagging:

- **`w_eta`**: dominant in most markets. A second of pickup ETA is worth a calibrated dollar amount in rider satisfaction studies; that calibration sets the unit of every other term.
- **`w_acceptance`**: matters more in markets with high decline rates. Pushing this too hard biases toward "easy" trips and starves harder rides.
- **`w_fairness`**: small in absolute terms, but its effect compounds across thousands of matches per driver. Treating it as small-but-persistent rather than as a one-off override is the right framing.
- **`w_surge`**: prevents the "ping-pong across surge boundary" pathology where a driver is dispatched into a non-surge zone, immediately becomes unavailable for surge trips, and the marketplace loses elastic supply.
- **`w_fraud`**: small additive penalty on flagged driver-rider pairs (e.g., known collusion patterns). Hard blocks live in the eligibility filter, not the cost function.

The objective is fundamentally **Pareto**: many configurations are non-dominated, and choosing among them is a product decision rather than a math one. The dispatcher's job is to expose the trade-off; the marketplace team's job is to decide where on the curve to sit.

A few practical tactics for multi-objective management:

- **Online A/B tests with holdout cells.** Run alternative weight configurations on small fractions of traffic; measure both efficiency metrics (wait time, cost per trip) and fairness metrics (variance in driver earnings, geographic disparity).
- **Constraint-then-optimize.** Express fairness and regulatory requirements as hard constraints, then optimize ETA/acceptance subject to those constraints. This is cleaner than packing everything into a soft cost function.
- **Calibration drift monitoring.** Cost weights expressed as "seconds of ETA equivalent" only work if the calibration holds. As behavior shifts (post-pandemic patterns, new driver cohorts, new product launches), recalibrate periodically against ground-truth satisfaction surveys.
- **Per-region policy.** A single global cost function is rarely right. Each region tunes weights to its own marketplace dynamics, regulatory environment, and competitive position.

The harder framing is that matching is a **sequential decision problem**, not a one-shot optimization. Each match consumes a driver and rebalances supply; the right metric is cumulative welfare across a planning horizon, not the cost of any single match. This is where reinforcement-learning-style approaches enter the picture — value functions over (driver position, time, market state) tuples that estimate the expected future contribution of each driver. Rolling these into the dispatch cost as a "future-value penalty" prevents the optimizer from greedily consuming drivers in ways that hurt subsequent demand. Production deployments are cautious here because RL policies are hard to validate and can produce surprising behaviour at the long tail; most teams treat the value-function term as a small additive nudge layered on top of the deterministic cost function rather than as the primary objective.

A separate complication is **mismeasured success.** The most-optimized metric is usually the easiest to measure — pickup ETA, completion rate, p99 latency — and not necessarily the one most aligned with marketplace health. Hard-to-measure outcomes (driver retention over months, rider habit formation, neighborhood coverage equity) require longer measurement windows and indirect signals. Dispatch teams that optimize aggressively on short-window metrics often discover, quarters later, that they've hurt the long-window ones. The discipline is to maintain a balanced scorecard and to gate aggressive cost-function changes on long-window holdout evaluation.

For background reading on the broader marketplace optimization framing, see Uber Engineering's [Marketplace and Real-Time Pricing](https://www.uber.com/blog/marketplace-real-time-pricing/) and the academic literature on **two-sided marketplace matching** (Eduardo M. Azevedo and E. Glen Weyl on matching with continuous types; Itai Ashlagi and collaborators on dynamic matching markets).

## Anti-Patterns

- **Pure greedy in dense markets.** Locally optimal, globally bad. Always at least the option of batched optimization on the table for high-density regions.
- **Treating dispatch as a stateless lookup.** The optimizer needs the open request set, the open offer set, and recent driver state. A "stateless microservice" that re-derives all of this per request will collapse under load.
- **Single global optimizer instance.** One optimizer for the whole world is a single point of failure and a single point of contention. Region-shard the dispatcher and the geo-index together.
- **Ignoring acceptance probability.** The closest driver who declines 80% of the time is worse than a slightly farther driver who accepts 95% of the time. The cost function must include `p_accept`, not just distance/ETA.
- **Optimizing only on ETA.** Real cost has multiple components; pure ETA optimization produces driver-side unfairness and surge boundary pathologies.
- **Hidden coupling between dispatcher and ETA service.** If the ETA service times out, dispatch should fall back to a coarse estimate (haversine + average speed) rather than block. Hard-coupling dispatch latency to ETA latency turns one degraded service into two.
- **No re-dispatch budget.** Every offer can fail; the system must have a finite, well-defined re-dispatch cycle with eventual cancellation. Infinite re-dispatch loops are a real production failure mode.
- **Treating Pool like UberX with extra steps.** Pool is fundamentally a different problem (online vehicle routing with time windows). Cramming it into a bipartite optimizer produces poor matches and bad rider experience.
- **Caching driver positions for too long.** Driver positions are stale at the speed of physics. A 30-second cache produces dispatches based on where the driver was, not where they are.
- **Overfitting to historical acceptance.** A new driver with no acceptance history is treated as low-probability and never gets offers, and therefore never builds history. Bootstrap policies (uniform priors, exploration slots) are mandatory for new driver onboarding.
- **Forgetting region-specific constraints.** Bridges, tolls, one-way streets, ferry-only crossings, time-of-day restrictions. The routing engine must know these; the dispatcher must trust the engine. Hand-coding constraints into the cost function produces fragility.
- **No graceful degradation under optimizer outage.** If the batched solver is unhealthy, fall back to greedy. If geosearch is unhealthy, surface a clear error rather than dispatching wildly. Degradation paths must be explicit.
- **Synchronous logging on the dispatch hot path.** A blocking write to a slow logging system on every match is a latency landmine. Log async via a bounded queue, and accept that some logs will be dropped under load.
- **No anti-fraud signal in matching.** Driver-rider collusion patterns (long fake trips, GPS spoofing, repeated cancellations) can be detected and folded into the cost function as a penalty; ignoring them invites systematic abuse.

## Related

- [`./h3-geo-indexing.md`](./h3-geo-indexing.md) _(planned)_ — the H3 indexing layer that this dispatcher sits on top of; resolution choice, parent-child traversal, sharding strategies.
- [`./driver-location-ingestion.md`](./driver-location-ingestion.md) _(planned)_ — the high-write location pipeline that feeds the live geo-index; how 1M pings/sec become a queryable in-memory index.
- [`../design-uber.md`](../design-uber.md) — parent case study; matching and dispatch is one section of the overall ride-hailing design.
- [`../design-doordash.md`](../design-doordash.md) — three-sided variant where matching is courier-customer-restaurant; reuse of the geo-dispatch ideas with batched-pickups twists.
- [`../../scalability/sharding-strategies.md`](../../scalability/sharding-strategies.md) — geographic sharding patterns including H3-cell sharding with hand-off.

## References

- Uber Engineering — [H3: Uber's Hexagonal Hierarchical Spatial Index](https://www.uber.com/blog/h3/)
- H3 Project — [Documentation](https://h3geo.org/docs/) and [GitHub: uber/h3](https://github.com/uber/h3)
- Uber Engineering — [Engineering Real-Time Pricing (Surge)](https://www.uber.com/blog/engineering-surge-pricing/)
- Uber Engineering — [Marketplace and Real-Time Pricing](https://www.uber.com/blog/marketplace-real-time-pricing/)
- Uber Engineering — [Forecasting at Uber: An Introduction](https://www.uber.com/blog/forecasting-introduction/)
- Uber Engineering — [DeepETA: How Uber Predicts Arrival Times](https://www.uber.com/blog/deepeta-how-uber-predicts-arrival-times/)
- Kuhn, H. W. *The Hungarian Method for the Assignment Problem.* Naval Research Logistics Quarterly, 1955.
- Munkres, J. *Algorithms for the Assignment and Transportation Problems.* SIAM Journal, 1957.
- Wikipedia — [Hungarian algorithm](https://en.wikipedia.org/wiki/Hungarian_algorithm)
- Brilliant.org — [Hungarian Maximum Matching Algorithm](https://brilliant.org/wiki/hungarian-matching/)
- Ahuja, R. K., Magnanti, T. L., Orlin, J. B. *Network Flows: Theory, Algorithms, and Applications.* Prentice Hall, 1993.
- Stanford CS261 — [Lecture notes on min-cost flow](https://web.stanford.edu/class/archive/cs/cs261/cs261.1162/)
- Google OR-Tools — [Linear Assignment and Min-Cost Flow](https://developers.google.com/optimization)
- Ma, S., Zheng, Y., Wolfson, O. *T-share: A Large-Scale Dynamic Taxi Ridesharing Service.* ICDE 2013.
- Lyft Engineering — [Matchmaking in Lyft Line](https://eng.lyft.com/matchmaking-in-lyft-line-9c2635fe62c4)
- Lyft Engineering — [Geosharding at Lyft](https://eng.lyft.com/geosharding-at-lyft-7b4eb5f78fa9)
