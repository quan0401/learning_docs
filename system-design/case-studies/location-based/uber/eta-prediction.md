---
title: "Uber Deep Dive — ETA Prediction"
date: 2026-04-29
updated: 2026-04-29
tags: [system-design, case-study, uber, deep-dive, eta, ml]
---

# Uber Deep Dive — ETA Prediction

**Date:** 2026-04-29 | **Updated:** 2026-04-29
**Tags:** `system-design` `case-study` `uber` `deep-dive` `eta` `ml`

## Table of Contents

- [Summary](#summary)
- [Overview — Why ETA Is Two Problems](#overview--why-eta-is-two-problems)
- [Pickup ETA vs Trip ETA](#pickup-eta-vs-trip-eta)
- [The Base Routing Engine](#the-base-routing-engine)
  - [Road Graph Representation](#road-graph-representation)
  - [Dijkstra and A\*](#dijkstra-and-a)
  - [Contraction Hierarchies and CRP](#contraction-hierarchies-and-crp)
  - [Live Traffic Edge Weights](#live-traffic-edge-weights)
- [The ML Correction Layer](#the-ml-correction-layer)
- [Feature Engineering](#feature-engineering)
- [Training the Correction Model](#training-the-correction-model)
- [Serving Latency Budget](#serving-latency-budget)
- [Confidence Intervals](#confidence-intervals)
- [User-Facing Display and Rounding](#user-facing-display-and-rounding)
- [Accuracy Metrics](#accuracy-metrics)
- [Per-Region Model Variants](#per-region-model-variants)
- [DeepETA — Uber's Transformer Architecture](#deepeta--ubers-transformer-architecture)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

ETA prediction is the most-used number Uber displays. It appears on the rider home screen before any ride is requested ("cars 3 min away"), inside the request flow ("your driver will arrive in 4 min"), throughout the trip ("11 min to destination"), and on every product surface that depends on supply visibility — surge, dispatch scoring, batched matching, Pool/Share routing. Get it wrong by 30 seconds at the bottom of the funnel and rider trust erodes; get it wrong systematically across a city and dispatch, pricing, and driver supply all drift with it. ETA accuracy is therefore not a feature — it is a **load-bearing primitive** that every other system in the marketplace consumes.

The architecture Uber settled on is a **two-layer hybrid**: a classical road-network routing engine produces a base ETA by running shortest-path on a directed weighted graph with live traffic edge weights, and an **ML correction layer** sits on top, taking the routing engine's output as one feature among many and predicting the residual error. The correction model has access to features the routing engine cannot encode well — driver-specific behavior, pickup-point quirks (airport curbs vs corner pickups), weather, day-of-week / hour-of-day cyclicality, recent observed traversal times on the chosen route, and the rider/driver pair's matching context. The whole stack must produce an answer in under ~10 ms p99 because it is called inside the dispatch hot path, on every rider screen refresh, and from every map redraw.

This deep dive traces that architecture from the road graph up through DeepETA — the transformer-based correction model Uber published in 2022 — covering the latency budget, the calibration story, the user-facing rounding behavior, and the regional model variants that make the same architecture serve hundreds of cities with very different traffic regimes. It is a companion to the parent case study [`../design-uber.md`](../design-uber.md), expanded around the ETA subsection.

## Overview — Why ETA Is Two Problems

A naive observer might think "ETA = distance / speed." That misses the structural split that drives the whole architecture:

1. **A physical-world routing problem.** Given a road graph, a start, an end, and current traffic, what is the fastest path and how long does it take? This is a graph algorithms problem with decades of research (Dijkstra 1959, A\* 1968, Contraction Hierarchies 2008, Customizable Route Planning 2013). It is *deterministic given the graph and edge weights*, and it can be made very fast with the right preprocessing.

2. **A real-world prediction problem.** The routing engine's answer is wrong in predictable ways. Drivers don't drive at the speed limit. Pickup points are not where the map pin says. Some intersections are slower at school dismissal time. Some drivers know shortcuts the routing engine doesn't. Weather slows everything by a multiplier that varies by city. The "true" travel time is a *random variable* whose mean differs from the routing engine's output by a residual that depends on dozens of features.

The classical approach combines them. The routing engine gives you the **point estimate of the geometric travel time**. The ML layer gives you **the correction and the uncertainty**. Each layer is necessary; neither is sufficient.

A useful framing: think of the routing engine as the *physics simulator* and the ML correction layer as the *empirical adjustment* that captures everything the simulator can't model. The simulator gives you a coherent baseline that respects road topology, turn restrictions, and traffic. The ML layer is what turns "geometrically reasonable" into "actually accurate."

Uber's published progression through this design space is, roughly:

- **Era 1 (early 2010s).** Pure routing engine with average-speed edge weights derived from historical pings. Decent for sparse cities, embarrassing in dense ones.
- **Era 2 (mid 2010s).** Routing engine + gradient-boosted tree (XGBoost-class) correction. Big accuracy jump. Trained per-city or per-region.
- **Era 3 (2022, DeepETA).** Routing engine + transformer-based correction model with self-attention over heterogeneous features, trained globally with regional embeddings. Lower MAPE, faster serving, easier to refresh than the tree ensemble it replaced.

The same two-layer skeleton survived all three eras. What changed was the correction model.

## Pickup ETA vs Trip ETA

Two distinct ETAs ride on the same infrastructure, with very different shapes.

| Property | Pickup ETA | Trip ETA |
|---|---|---|
| **Origin** | Driver's current GPS position | Pickup location |
| **Destination** | Pickup location | Rider's drop-off |
| **Distance** | Usually short (city blocks) | Variable (can be long) |
| **Dominant source of error** | Driver finding the rider, parking, last-meters | Traffic variance, route choice |
| **Sensitivity** | Very high — rider is watching the car icon move | Moderate — riders forgive ±2 min on a 25-min trip |
| **Refresh rate** | High during pre-pickup phase (every map update) | Moderate during trip |
| **Use cases** | Marketplace UI, dispatch scoring, surge | Fare estimate, navigation, trip status |

The two ETAs share the same routing engine and correction model architecture, but the **features dominate differently** and the **error structure is different**:

- Pickup ETAs are biased pessimistic-too-optimistic. The routing engine assumes the driver is on the road segment they're pinging from; in reality the driver may be behind a building, at a red light, in a parking lot, or facing the wrong way. The ML correction lands hard on "last 200 meters" features — pickup geometry, presence of one-way streets, history of similar pickups at this point.
- Trip ETAs accumulate error over distance. Small per-segment biases compound across long routes. The correction model has to handle this without overfitting to specific routes; the dominant features are aggregate (traffic regime, time bucket, weather) rather than route-specific.

A subtle but important detail: the **pickup ETA shown to the rider before they even tap "Request"** is computed across many candidate drivers (the kNN result of the geo-index query — see [`./matching-and-dispatch.md`](./matching-and-dispatch.md)). The displayed number is typically the minimum (or low percentile) across candidates, not the ETA of a specific assigned driver. After dispatch, the displayed ETA switches to that specific driver's ETA. This handoff has to be smooth — if the displayed ETA jumps from "2 min" (best candidate) to "5 min" (actually-assigned driver), the rider notices.

## The Base Routing Engine

### Road Graph Representation

The road network is modeled as a **directed weighted graph**:

- **Nodes**: intersections and road endpoints.
- **Edges**: road segments. One physical road typically becomes two directed edges (one per direction) unless it is one-way.
- **Edge attributes**: length (meters), road class (highway, arterial, residential), speed limit, turn restrictions at the destination node, time-of-day variance signature, current traffic multiplier.
- **Edge weight = expected travel time**, which is the load-bearing quantity for shortest-path.

For a city like New York, this graph is on the order of ~10⁵ nodes and ~10⁶ edges. For a country, ~10⁷–10⁸ edges. Uber sources this graph from OpenStreetMap and licensed providers, then maintains it as a continuously-updated artifact: roads are added, turn restrictions change, construction is reflected, and seasonal patterns are folded in.

### Dijkstra and A\*

The classical shortest-path algorithms:

- **Dijkstra (1959)**. From the source, expand the closest unvisited node, relax outgoing edges, repeat. O((V + E) log V) with a binary heap. Correct for any graph with non-negative edge weights. **Bidirectional Dijkstra** (search from both source and target, meet in the middle) is roughly 2× faster on average.
- **A\* (Hart, Nilsson, Raphael 1968)**. Dijkstra with a heuristic — at each step, prefer expanding nodes that look closer to the target. With an *admissible* heuristic (one that never overestimates the remaining distance), A\* is provably correct and typically much faster than Dijkstra. For road networks, the great-circle distance divided by the maximum road speed is a valid heuristic and gives meaningful pruning.

For ride-hailing scale, neither plain Dijkstra nor plain A\* is fast enough. A single shortest-path query on a country-scale graph would be tens of milliseconds at minimum, and the routing engine is called *millions of times per second* across the fleet. Preprocessing is mandatory.

### Contraction Hierarchies and CRP

**Contraction Hierarchies (CH)** — Geisberger, Sanders, Schultes, Delling 2008 — and **Customizable Route Planning (CRP)** — Delling, Goldberg, Pajor, Werneck 2013 — are the two preprocessing approaches that dominate production routing engines (OSRM uses CH; Google Maps' published architecture uses CRP-style overlay graphs).

The intuition behind Contraction Hierarchies:

1. Order nodes by an *importance* heuristic (highway intersections rank higher than residential cul-de-sacs).
2. Process nodes from least-important to most-important. For each node, add **shortcut edges** that summarize the shortest paths through that node, then "contract" the node away.
3. The result is an augmented graph plus a hierarchy. Queries run a bidirectional search that only goes *upward* in the hierarchy, drastically reducing the search space.

Query times on country-scale graphs drop from tens of milliseconds to single-digit microseconds. The trade-off is preprocessing cost: building the hierarchy is minutes-to-hours, and any change to edge weights (live traffic!) requires a re-build.

**Customizable Route Planning** addresses this. The graph is partitioned into nested cells; the metric (edge weights) is decoupled from the topology. When traffic changes, only the metric needs re-customization, which is much faster than rebuilding the full hierarchy. This is the architecture you want when traffic data is updated continuously.

Uber's internal routing engine takes the same shape: partition the graph, precompute overlays, plug in live traffic as a fast metric update, query in microseconds. The exact implementation details are not fully public, but the algorithmic family is well-documented in academic and industry literature.

### Live Traffic Edge Weights

Edge weights aren't static — they reflect current traffic. Sources:

- **Driver pings.** Every active driver pings position every ~4–5 seconds. As drivers traverse edges, the system measures observed travel time per edge and aggregates into a live traffic estimate. This is Uber's competitive moat for ETA: a giant fleet means dense, fresh observations on the very edges customers are about to drive.
- **Historical baselines.** For each edge, a time-of-day / day-of-week speed profile derived from months of pings. Used as a fallback when live data is sparse and as a smoothing prior even when it isn't.
- **Map-matching.** Raw GPS is noisy; pings get snapped to road segments via Hidden Markov Model / Viterbi map-matching before they're used to update edge weights. Without map-matching, drivers appear to drive through buildings and edge weights get poisoned.
- **External data.** Weather, road closures, scheduled events (concerts, sporting events). These are blended in as multiplicative or additive corrections.

The edge-weight pipeline is itself a streaming system: pings flow through Kafka, a stream processor (Flink-class) updates per-edge sliding-window estimates, and the routing service reads from a hot index. Latency from ping → updated edge weight is single-digit minutes. See [`../../../batch-and-stream/modern-streaming-engines.md`](../../../batch-and-stream/modern-streaming-engines.md) for the underlying machinery.

## The ML Correction Layer

The routing engine produces a base ETA. The correction layer takes that as one feature, plus dozens of others, and predicts the residual.

Why a residual model rather than a model that predicts ETA from scratch? Several reasons converge:

- **Strong inductive bias.** The routing engine encodes road topology and traffic — the ML model would have to relearn all of it. Residual learning is dramatically more sample-efficient.
- **Graceful degradation.** If the ML layer fails or is bypassed, the routing engine's output is still a usable ETA. There's no scenario where the system falls back to a worse-than-routing prediction.
- **Interpretability and debuggability.** A 30-second ETA error decomposed into "routing thought 240s, model added +30s correction" is much easier to debug than a single opaque scalar.
- **Stability across regions.** The routing engine's output has the same physical meaning everywhere; only the residual distribution varies by city. This makes regional adaptation easier (see [Per-Region Model Variants](#per-region-model-variants)).

The model's input is a wide feature vector; its output is a corrected ETA (usually expressed as the routing baseline plus a learned delta) and, in modern variants, a quantile-style uncertainty estimate.

```text
                          ┌──────────────────────┐
   pickup, dropoff ─────► │  Routing Engine      │ ─── base_eta ──┐
   live traffic state     │  (Dijkstra/A* + CH)  │                │
                          └──────────────────────┘                │
                                                                  ▼
   driver features ─────────────────────────────────►  ┌─────────────────┐
   rider features  ─────────────────────────────────►  │   ML Correction │ ─── final_eta
   route features  ─────────────────────────────────►  │   (DeepETA-class)│
   contextual feats ────────────────────────────────►  └─────────────────┘
   (time, weather, region)
```

The architectural property to internalize: the correction layer is a *universal post-processor* on top of routing. Same pattern as a calibration layer on a classifier — the underlying model produces a coherent score, the calibration layer aligns it to observed reality.

## Feature Engineering

The features fed into the correction layer are where most of the practical accuracy comes from. They cluster into a few groups.

**Routing engine output (the baseline)**

- Predicted travel time (the headline number).
- Predicted distance.
- Number of turns, number of left turns, number of unprotected lefts.
- Highway vs surface street fractions of the route.
- Number of traffic-signal-controlled intersections.

**Temporal features**

- Hour of day (one-hot or cyclical sin/cos encoding).
- Day of week.
- Is-holiday / is-day-after-holiday.
- Minutes since the last major traffic event in the region.
- Position within the local "rush hour curve" (a learned per-region scalar).

Time features are the highest-impact corrections after the routing baseline itself. A 7am vs 7pm vs 11pm trip on the same route differs by 30%+ in observed travel time even with the same routing baseline.

**Weather features**

- Current precipitation (rain, snow).
- Recent precipitation history (the road is wet from rain that stopped 20 min ago).
- Temperature (snow vs rain regime).
- Visibility / fog signals.
- Wind (matters in some markets for two-wheeler products).

Weather effects are *non-linear* and *regional*. A half-inch of snow in Atlanta is gridlock; the same in Buffalo is a Tuesday. The model has to learn weather-by-region interactions.

**Driver features**

- Recent observed driver-specific speed multiplier (some drivers are 5–10% faster or slower than the median).
- Driver's experience in this region (new driver vs veteran).
- Vehicle class (sedan vs SUV vs scooter — affects effective speed on some routes).
- Whether the driver has done this exact pickup point before.

**Pickup / drop-off features**

- Pickup point category (airport curb, hotel, residential, corner, transit hub, stadium).
- Historical pickup-difficulty score for this geocoded point.
- Distance from the routing graph to the actual pickup pin (if the pin is in a parking garage, the routing endpoint is at the entrance and the last meters are unmodeled).
- Drop-off type (curbside vs gated entry vs valet).

These pickup-point features are some of the most powerful in the model for *pickup ETA* and have small but real impact on trip ETA via drop-off.

**Recent traffic / route features**

- Observed travel time on this exact route in the last K minutes (if any).
- Observed travel time on this route during this time-of-day bucket over the last K days.
- Live incident reports on the route.
- Recent variance of traversal times — high variance increases the predicted uncertainty.

**Contextual / business features**

- Product type (UberX, Pool, Black, Reserve, Eats delivery — different speed regimes).
- Surge multiplier (correlated with demand spike, often correlated with traffic).
- Region embedding (a learned per-region vector — see [Per-Region Model Variants](#per-region-model-variants)).

The production feature pipeline is a streaming + batch hybrid: real-time features (current traffic, recent observations) come from the ping stream via a feature store, while slow-moving features (driver baseline speeds, pickup-point histories) come from a daily batch job written into the same store. The model's feature lookup at request time is a single batched read against the online feature store, budgeted at 1–2 ms.

## Training the Correction Model

The training set is constructed from completed trips:

- **Label**: the actual observed travel time, measured from when the driver started the leg (pickup leg or trip leg) to when they completed it. Server-stamped timestamps, not client-stamped.
- **Features**: all the features above, snapshotted *as they were at the moment the leg started*. This is critical — using "future" features (e.g., the actual weather at trip end) would leak information and make the model useless in production.
- **Granularity**: one example per leg. A trip generates two examples: the pickup leg and the trip leg. They go through the same model but with different feature distributions.

Volume: with millions of trips per day and two legs per trip, the daily training set is in the tens of millions. Multi-week windows are typical for batch training, with a sliding cut-off so the most recent days dominate.

**Loss function**

A naive choice would be MSE on residuals (predict the delta, square the error). Two issues:

1. **Asymmetric cost.** A 60-second underestimate is worse than a 60-second overestimate (rider waits, driver-pickup gets late, downstream dispatch decisions are wrong). MSE treats them symmetrically.
2. **Heavy tails.** A few catastrophic ETA errors (driver got stuck, road closed) shouldn't dominate the gradient. MSE is sensitive to outliers.

In practice the correction model is trained with a **quantile-style or asymmetric Huber loss** that biases slightly toward overestimation and is robust to outliers. DeepETA uses an asymmetric Huber-like loss specifically because of this asymmetry property — the paper makes the point that being slightly pessimistic is the user-friendly equilibrium.

**Training cadence**

- **Batch retraining.** Full retrain on a multi-week window, run daily or every few days. New model goes through offline evaluation (compared against the production model on a held-out time slice) before promotion.
- **Online refresh of fast-moving sub-components.** Things like per-driver speed multipliers and per-edge live traffic don't go through the heavy retrain — they're updated continuously by streaming jobs and read at inference time.
- **A/B promotion.** New model version is shipped to a small fraction of traffic, accuracy and downstream metrics (acceptance rate, trip completion rate, rider rating) are monitored, and only on positive deltas does it get full rollout.

**Validation against time-shift**

Standard ML cross-validation splits randomly. For ETA, that leaks information — the training set contains examples from the same hour as the test set, and time-of-day patterns leak across the split. The correct discipline is **time-based holdout**: train on weeks 1–4, validate on week 5, test on week 6. This is the only honest way to measure whether the model generalizes forward in time.

## Serving Latency Budget

The correction model is on the dispatch hot path and on every map redraw. The latency budget is brutal: **p99 under ~10 ms** for the inference call (not counting feature lookup).

Decomposition of the end-to-end ETA call (illustrative, order-of-magnitude):

| Stage | Budget |
|---|---|
| Feature lookup (online feature store) | 1–3 ms |
| Routing engine query | 0.5–2 ms (microseconds with CH for short routes; ms range for cross-city) |
| ML inference (correction model) | 2–8 ms |
| Postprocessing (rounding, calibration, packaging) | <1 ms |
| **Total p99** | **~10–15 ms** |

Engineering moves that buy this latency:

- **Pre-built routing graph with CH/CRP.** No live shortest-path computation from scratch.
- **Co-located inference.** Correction model runs on the same machine (or rack) as the routing engine and feature store. Cross-AZ network hops are fatal.
- **Quantization.** Production correction models are typically int8-quantized to fit in CPU L2/L3 caches and avoid GPU round-trip overhead for small batches. CPU inference at <5 ms is achievable for transformers in the 10s of MB range.
- **Batching across requests.** When many ETA requests arrive within a few milliseconds, they are batched into a single GPU/CPU forward pass. Batch sizes of 32–128 fit in the latency budget while amortizing fixed overhead.
- **Feature caching.** Slow-moving features (pickup-point histories, region embeddings) are cached in process memory; only fresh features (current traffic, current driver position) are looked up per request.
- **Tail-cutoff.** If the model fails to respond within budget, fall back to the routing baseline. Better a 5%-worse ETA than a request that times out and brings down dispatch.

The DeepETA paper (2022) explicitly cites *halving inference latency vs the prior gradient-boosted ensemble* as one of the motivations for moving to a transformer architecture — the previous XGBoost-class model had a long tail under feature-rich requests that the transformer doesn't.

**Why CPU and not GPU**

A common reaction is "transformers want GPUs." For DeepETA-style serving the answer is more nuanced:

- The model is small (10s of MB after quantization), so it fits in CPU L2/L3 cache.
- Per-request batches are small (single-digit to low-double-digit), where GPU throughput advantages are dominated by host-device round-trip overhead.
- CPU inference avoids the operational cost of GPU fleets, kernel-driver issues, and memory-pinning gymnastics.
- Mixing CPU-served correction inference with CPU-served routing in the same process keeps the data path local; sending the routing output across PCIe to a GPU would add ~hundreds of microseconds of fixed overhead.

GPUs still dominate the *training* path, where batch sizes are large and model updates are throughput-bound. The CPU-vs-GPU choice for serving is a per-system economic decision, not a model-architecture decision.

## Confidence Intervals

A point ETA is not enough. Several downstream consumers want uncertainty:

- **User-facing display** ("3 min" vs "2–4 min" — see next section).
- **Dispatch scoring.** When choosing among candidate drivers, prefer the one with both low expected ETA *and* low ETA variance. A high-variance candidate might be "5 min ± 3 min"; a low-variance candidate might be "5.5 min ± 0.5 min" — the latter is often the better dispatch choice.
- **Surge / pricing.** Confidence intervals on ETA cascade into confidence intervals on fare estimates, which cascade into the upfront-fare-vs-actual-fare difference policies.
- **Pool/Share matching.** Detour budget calculations are sensitive to ETA uncertainty; over-tight intervals cause Pool matches that fail their detour SLA.

The correction model emits uncertainty in one of a few ways:

- **Quantile regression.** Train multiple model heads (or a single model with multiple output heads), each predicting a different quantile (e.g., p10, p50, p90). The interval p10–p90 is the 80% confidence interval. This is robust and doesn't assume a parametric distribution.
- **Heteroscedastic regression.** Predict both mean and variance simultaneously, training with a Gaussian negative-log-likelihood loss. Cheaper than quantile regression but assumes Gaussianity.
- **Conformal prediction.** Wrap any point predictor with a calibration step that produces distribution-free intervals with provable coverage guarantees. Increasingly common in production ML systems.

In the DeepETA architecture, the model produces a calibrated point estimate, and uncertainty is derived from a combination of model output and historical residual variance for similar route signatures. The interval is then post-processed for display.

## User-Facing Display and Rounding

This is where ETA stops being a pure ML problem and becomes a UX problem. The displayed number is *not* the model's raw output.

**Why rounding matters**

- A model output of 184.7 seconds is honest but feels fake. "3 min" is what a human would say.
- Sub-minute precision suggests false certainty. Showing "3 min" implies a window roughly 2:30–3:30; that matches the actual confidence interval far better than "3 min 4 sec" would.
- Stable display values reduce flicker. If the underlying number jiggles between 183 and 198 seconds, displaying "3 min" the whole time is correct UX; displaying "3:03 → 3:18 → 3:01 → 3:18" is anxiety-inducing.

**Rounding policies**

A typical policy:

- 0–60s: "less than 1 min" (or "arriving").
- 1–9 min: round to the nearest integer minute.
- 10–60 min: round to the nearest integer minute, possibly with hysteresis to avoid flicker.
- 60+ min: round to the nearest 5 minutes.

Hysteresis is the trick that prevents flicker: don't switch from "5 min" to "4 min" until the underlying value has been below 4:15 for at least one update cycle. This costs honesty (the displayed number lags reality slightly) but buys perceived stability.

**Range vs point display**

Some surfaces show a range ("2–4 min"), others a point ("3 min"). Trade-offs:

- **Point display** is cleaner, sets a firmer expectation, and is what most riders prefer for *short* ETAs.
- **Range display** is more honest under high uncertainty (e.g., long airport pickups, severe weather), and reduces the felt impact of being off by a minute.

The choice is contextual. A pre-request "cars 3 min away" is point. A trip ETA at the start of a 30-minute ride is sometimes a range. A pickup ETA where the driver is one block away is point.

**Asymmetric rounding**

Some products round *up* (always pessimistic) — under-promise, over-deliver. Others round symmetrically. The asymmetric Huber loss in training already biases the underlying point estimate slightly pessimistic, and rounding can either preserve or amplify that. Per-region tuning matters: in some markets, riders strongly prefer "I'd rather you say 5 min and be there in 4"; in others, "say what you mean."

## Accuracy Metrics

Several metrics matter, and they capture different aspects.

**MAPE (Mean Absolute Percentage Error)**

```text
MAPE = mean(|predicted - actual| / actual)
```

The headline metric. Industry-quoted Uber numbers historically ran ~15% MAPE for trip ETAs and lower for pickup ETAs. DeepETA paper reports meaningful improvements over the prior production model (concrete numbers are in the linked blog post).

MAPE has known weaknesses:

- It's undefined / unbounded for very short trips (denominator → 0).
- It's asymmetric — a 50% overestimate (actual=10s, predicted=15s, MAPE=50%) is treated the same as a 33% underestimate (actual=15s, predicted=10s, MAPE=33%).

For these reasons, MAPE is usually paired with other metrics.

**MAE (Mean Absolute Error)**

In seconds. Easier to interpret for very short or very long durations than MAPE.

**Within-X-percent**

The fraction of trips where `|predicted - actual| / actual < X`. Common thresholds: 10%, 20%. Easier for product/exec audiences than MAPE because it's a percentage of trips, not a percentage of time.

**Asymmetric splits**

Underestimation vs overestimation tracked separately. If the model is on average right but always off by ±5 minutes in either direction, that's a bad model even though MAPE looks fine in aggregate. The asymmetric loss in training already biases this toward the safer direction; the metric needs to confirm.

**Calibration**

For confidence intervals: of the trips where the model said "interval covers 80% of the probability," does the actual travel time fall in that interval 80% of the time? If the model says "80%" but only 60% of trips actually fall in the interval, the model is *over-confident* and downstream consumers (dispatch scoring, Pool detour budgets) will silently misbehave.

**Downstream business metrics**

Ultimately the model is judged on what it does to the marketplace, not its raw error:

- Driver acceptance rate (drivers reject offers with ETAs that look weird).
- Match latency (better ETA → better dispatch decisions → faster matches).
- Trip completion rate.
- Rider rating distribution.
- Pool/Share detour SLA compliance.

A new ETA model that improves MAPE by 1% but degrades acceptance rate by 0.1% is a regression. The full evaluation chain is offline metrics → online A/B test → business-metric impact → promotion decision.

**Slice-aware monitoring**

Aggregate metrics hide local regressions. A model that improves global MAPE by 1% may have made airport pickups 5% worse and corner pickups 2% better. Production monitoring tracks accuracy across many slices simultaneously: pickup-point category, time bucket, weather regime, region, product type, distance bucket, driver tenure bucket. A regression in any meaningful slice — even if the aggregate is unchanged — is a finding worth investigating before promotion.

This slice discipline is one of the unsung reasons production ML systems improve year over year. The headline metric is the easy part. The hard part is catching the regressions that hide inside it.

## Per-Region Model Variants

A single global ETA model would be wrong everywhere. Cities differ on every dimension that matters:

- **Traffic regime.** Manhattan rush hour vs Phoenix arterial vs Mumbai narrow streets — fundamentally different speed distributions.
- **Driver behavior.** Cultural and infrastructural norms produce different speed/route choices.
- **Pickup geometry.** Tokyo train-station pickups vs São Paulo airport vs Lagos roadside hailing — different last-meters dynamics.
- **Weather sensitivity.** Same rainfall has different effects on different road systems.
- **Product mix.** Some cities are dominated by Pool/Share, others by Black, others by motorcycle (e.g., UberMoto in India).

Three architectural strategies, increasingly sophisticated:

1. **One model per region.** Train and serve separately. Simple. Works fine for big regions; suffers in small ones where the training set is too thin.
2. **Shared backbone, regional heads.** Train one big model with shared lower layers and regional output heads. Sample-efficient (small regions benefit from the shared representation) and serving-efficient (one forward pass through the backbone, then route to the region-specific head).
3. **Single model with regional embeddings.** Train one global model where region is just another feature, fed in as a learned embedding. Lets the model learn cross-region patterns and gracefully handle new regions (start with a similar-region embedding, fine-tune as data accumulates). DeepETA takes this approach.

The DeepETA paper specifies a global model with region embeddings rather than per-region models. The justification is operational as much as statistical: maintaining hundreds of separate model artifacts, training pipelines, and monitoring dashboards is expensive and error-prone. One model with a regional embedding feature collapses the operational footprint while preserving regional specialization.

There are still per-region overrides on top of the model: rounding policies, display range vs point, calibration windows, holiday calendars, weather feature normalization. These are configuration, not separate models.

A subtler point about regional embeddings: they should be **learned end-to-end with the ETA loss**, not pre-clustered by hand. A naive engineer might bucket cities into "dense Asian", "American suburban", "European medieval-core", etc., and embed those clusters. This works but throws away signal. Letting the model learn its own regional embedding from data lets cities with similar traffic regimes (regardless of geography) end up nearby in embedding space — Bangkok and Lagos may turn out closer than Bangkok and Tokyo despite the geographic intuition. The learned representation captures *traffic similarity*, which is what the ETA model actually cares about.

Onboarding a new region is a real operational story: until enough trips have flowed through to train the regional embedding, the model uses a similarity-based initialization (start the new city with the embedding of the most similar already-onboarded city) and updates it as data accumulates. This is faster and more robust than training a fresh per-region model from a thin training set.

A related operational concern: **label drift**. The same city looks different over time. New highways open, traffic patterns shift after pandemics, fuel prices change driver behavior, ride-sharing competitors come and go. The training pipeline must use a sliding window that emphasizes recent data without abandoning older history entirely (older data is still useful for stable patterns like rush-hour cyclicality). The window length is itself a hyperparameter and is typically tuned per-region.

## DeepETA — Uber's Transformer Architecture

The 2022 DeepETA paper and accompanying engineering blog post describe the current-generation correction model. Key design choices:

**Why a transformer**

- The features are heterogeneous (continuous like routing baseline ETA, categorical like pickup-point type, sequence-like like recent ping history) and have many cross-interactions. Transformer self-attention is a natural way to learn cross-feature interactions without manual feature crosses.
- The previous gradient-boosted ensemble had a feature-interaction ceiling — adding the 50th cross feature gave less and less marginal gain. A transformer attends across all features in the same pass.
- Inference latency, surprisingly to many, came out *better* than the GBM baseline once int8-quantized and properly batched. The fixed overhead of the GBM tree traversals (especially with hundreds of trees and high feature counts) was higher than expected.

**Self-attention over features**

Features are tokenized — each feature becomes a learned vector. Continuous features go through a discretization + embedding step (binning then lookup), categorical features go through an embedding table, and the routing baseline ETA becomes its own special token. Self-attention runs across these feature tokens, letting the model learn interactions like "high temperature × urban pickup × weekend × airport drop-off".

**Linear-complexity attention variants**

Standard self-attention is O(N²) in sequence length. The DeepETA paper uses a linear-complexity variant (the paper specifically discusses linear self-attention variants suitable for this regime) to keep inference latency manageable. The number of feature tokens is small (dozens, not thousands), so even quadratic attention would be OK in absolute terms — the linear variant is mostly an engineering safety margin.

**Training signal**

- Asymmetric Huber-like loss on residuals (as described above).
- Multi-task auxiliary heads in some variants (predict pickup-leg ETA and trip-leg ETA simultaneously to share representation).
- Continuous features run through the discretization step partly for embedding ergonomics, partly because the discretization itself is a form of robustness (small perturbations in continuous inputs don't cause output jitter).

**Serving infrastructure**

- Int8 quantization for CPU inference. The paper reports meeting the latency SLO on CPU; GPU is reserved for batch-training and offline scoring.
- Co-location with the routing engine and feature store.
- Online-online split: the model itself is updated on a daily-to-weekly cadence; per-driver and per-edge dynamic features are streamed continuously.

The blog post is the canonical public reference: <https://www.uber.com/blog/deepeta-how-uber-predicts-arrival-times/>. The ideas in it have since been picked up and adapted by other ride-hailing and logistics companies (Lyft, DoorDash, Grab — see references for related public posts where available).

**What DeepETA replaced and why**

The pre-DeepETA production model was a gradient-boosted decision tree ensemble (XGBoost-class). That family of models had served Uber well for years — they're sample-efficient, robust, easy to debug, and ship cleanly. Three pressures pushed the team off it:

1. **Diminishing returns on feature engineering.** Each new cross-feature gave less marginal accuracy. Manual feature crosses had hit a wall.
2. **Tail latency at high feature counts.** Deep trees with many splits have non-trivial CPU traversal cost; with hundreds of features and hundreds of trees, the p99 was creeping above the latency budget.
3. **Operational fragmentation.** Different cities had different feature sets and different trained ensembles. Maintenance was expensive.

A transformer with feature-token attention solved all three: cross-feature interactions are learned automatically (no manual crossing), inference is a single dense matmul that benefits from quantization and modern CPU vector instructions (so latency is more predictable), and one model with regional embeddings replaces N regional ensembles.

This is an instructive case study for the broader question of "when do you replace tree ensembles with neural networks for tabular-ish problems." The honest answer is *not always* — the GBM was probably still ahead in pure single-trip MAPE for the first year of DeepETA's existence, with the transformer winning on operational and tail-latency dimensions before fully matching on accuracy. The migration was justified on the *whole-system* outcome, not the headline accuracy number alone.

**What stayed the same**

The two-layer architecture (routing engine + correction model) survived the migration unchanged. The features mostly survived, with some reshaping for the transformer's tokenization scheme. The training data pipeline barely changed. The serving infrastructure (feature store, online lookups, fallback to routing baseline on failure) was preserved. This is the typical pattern for production ML: the model swaps but the surrounding system endures, which is how teams ship these migrations without 18 months of downtime.

## Anti-Patterns

- **Single-stage ETA.** Predicting ETA end-to-end without a routing engine baseline forces the model to relearn road topology from data. Sample inefficient and brittle.
- **Routing engine without ML correction.** Geometrically reasonable but persistently wrong by 10–20% in the real world. Riders notice. Dispatch scoring degrades.
- **Static edge weights.** Speed limits or month-old historical averages, no live traffic. Routing engine looks fast but is unaware of every actual traffic regime.
- **Skipping map matching.** Raw GPS pings used directly to update edge weights. The driver "drives through buildings" and edge weights get poisoned by impossible traversals.
- **Random cross-validation on time-series data.** Leaks future information into training, gives optimistic offline metrics that collapse in production.
- **One global model, no regional adaptation.** A model tuned for North American highways will be embarrassing in Lagos or Mumbai.
- **Per-region models with no shared backbone.** Operational nightmare and statistically wasteful in low-volume regions.
- **No latency budget enforcement.** ETA inference creeps from 5 ms to 50 ms over a year of feature additions; dispatch starts timing out; nobody notices because individual feature additions look harmless.
- **Showing raw model outputs to the user.** "3 min 17 sec" implies false precision and induces flicker. Round, hysteresis, and pick point-vs-range deliberately.
- **Symmetric loss in training.** A 60-second underestimate is genuinely worse than a 60-second overestimate; the loss should reflect that.
- **No confidence intervals.** Downstream consumers (dispatch, surge, Pool) silently treat point estimates as certain. Match quality and Pool detour SLAs degrade.
- **Treating MAPE as the only metric.** A model can have great MAPE and terrible business metrics if it's systematically underestimating or its calibration is broken.
- **No fallback to routing baseline.** If the ML correction service is unhealthy, the system should silently degrade to the routing engine output, not return errors. ETA is too critical to fail open.
- **Re-deriving features per request rather than using a feature store.** Recomputing "driver's recent average speed" on every request is wasted compute and creates cross-request inconsistency. Compute it once in a stream job, store it, look it up.
- **Ignoring pickup-point geometry.** Trip ETA is the headline number, but pickup ETA is what riders watch second-by-second. Pickup-point features are the most underrated feature group.
- **Forgetting to rotate / refresh the routing graph.** OSM data drifts, road closures happen, construction zones come and go. A routing graph that hasn't been refreshed in months silently degrades ETA accuracy.

## Related

- [`../design-uber.md`](../design-uber.md) — parent case study; the ETA section there is the high-level summary that this document deep-dives.
- [`./h3-geo-indexing.md`](./h3-geo-indexing.md) — H3 cells are the spatial primitive used for many of the ETA features (regional embeddings, pickup-point clustering, edge-weight aggregation).
- [`./matching-and-dispatch.md`](./matching-and-dispatch.md) — dispatch scoring consumes ETA + ETA confidence as its dominant input. Better ETA → better dispatch.
- [`../../social-media/tiktok/for-you-page.md`](../../social-media/tiktok/for-you-page.md) — companion ML-driven prediction deep dive with the same two-stage shape (cheap candidate generation → heavy scoring).
- [`../../../batch-and-stream/modern-streaming-engines.md`](../../../batch-and-stream/modern-streaming-engines.md) — the streaming machinery underneath live edge weights and online feature updates.

## References

- Uber Engineering — *DeepETA: How Uber Predicts Arrival Times Using Deep Learning* (2022). <https://www.uber.com/blog/deepeta-how-uber-predicts-arrival-times/>
- Hu, X., et al. (Uber). *DeepETA: How Uber Predicts Arrival Times.* Engineering blog companion to the production rollout. <https://www.uber.com/blog/deepeta-how-uber-predicts-arrival-times/>
- Dijkstra, E. W. *A note on two problems in connexion with graphs.* Numerische Mathematik, 1959.
- Hart, P. E., Nilsson, N. J., Raphael, B. *A Formal Basis for the Heuristic Determination of Minimum Cost Paths.* IEEE Transactions on Systems Science and Cybernetics, 1968.
- Geisberger, R., Sanders, P., Schultes, D., Delling, D. *Contraction Hierarchies: Faster and Simpler Hierarchical Routing in Road Networks.* WEA, 2008.
- Delling, D., Goldberg, A. V., Pajor, T., Werneck, R. F. *Customizable Route Planning in Road Networks.* Transportation Science, 2015 (earlier conference version 2013).
- Google Maps Engineering — *Google Maps 101: How AI helps predict traffic and determine routes.* <https://blog.google/products/maps/google-maps-101-how-ai-helps-predict-traffic-and-determine-routes/>
- DeepMind / Google Maps — *Traffic prediction with advanced Graph Neural Networks.* <https://deepmind.google/discover/blog/traffic-prediction-with-advanced-graph-neural-networks/>
- OSRM Project — *Open Source Routing Machine.* <https://project-osrm.org/>
- OpenStreetMap — *OSM data and road graph source.* <https://www.openstreetmap.org/>
- Uber Engineering — *Engineering Real-Time Pricing (Surge).* <https://www.uber.com/blog/engineering-surge-pricing/> (consumes ETA as a primary input).
- Uber Engineering — *H3: Uber's Hexagonal Hierarchical Spatial Index.* <https://www.uber.com/blog/h3/>
- Microsoft Research — *Customizable Route Planning project page.* <https://www.microsoft.com/en-us/research/project/customizable-route-planning/>
