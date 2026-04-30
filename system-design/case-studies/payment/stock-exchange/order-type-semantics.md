---
title: "Order Type Semantics — Market, Limit, Stop, IOC, FOK, and Iceberg"
date: 2026-04-30
updated: 2026-04-30
tags: [system-design, deep-dive, fintech, semantics, order-types]
---

# Order Type Semantics — Market, Limit, Stop, IOC, FOK, and Iceberg

**Date:** 2026-04-30 | **Updated:** 2026-04-30
**Tags:** `system-design` `deep-dive` `fintech` `semantics` `order-types`

## Table of Contents

- [Summary](#summary)
- [Why Order Type Semantics Are Load-Bearing](#why-order-type-semantics-are-load-bearing)
- [The Matching Primitive — One Verb, Many Wrappers](#the-matching-primitive--one-verb-many-wrappers)
- [Market Order — Fast, Dangerous, Often Disabled](#market-order--fast-dangerous-often-disabled)
  - [Naive Semantics](#naive-semantics)
  - [Slippage and Why Pure Market Orders Are Rare](#slippage-and-why-pure-market-orders-are-rare)
  - [Market-with-Protection](#market-with-protection)
- [Limit Order — The Resting Workhorse](#limit-order--the-resting-workhorse)
  - [Crossing vs Resting](#crossing-vs-resting)
  - [Marketable Limit and Locked Markets](#marketable-limit-and-locked-markets)
  - [Limit Matching Pseudocode](#limit-matching-pseudocode)
- [Stop and Stop-Limit — Triggered Orders](#stop-and-stop-limit--triggered-orders)
  - [Trigger Semantics — Last-Trade vs NBBO](#trigger-semantics--last-trade-vs-nbbo)
  - [Stop Trigger Event Handler](#stop-trigger-event-handler)
  - [Stop-Limit Pitfalls](#stop-limit-pitfalls)
- [Time-in-Force (TIF)](#time-in-force-tif)
  - [DAY, GTC, GTD](#day-gtc-gtd)
  - [IOC — Immediate-or-Cancel](#ioc--immediate-or-cancel)
  - [FOK — Fill-or-Kill](#fok--fill-or-kill)
  - [ATO and ATC — Auction-Only TIFs](#ato-and-atc--auction-only-tifs)
- [FOK Two-Phase Logic](#fok-two-phase-logic)
- [Iceberg / Reserve Orders](#iceberg--reserve-orders)
  - [Display Slice vs Hidden Reserve](#display-slice-vs-hidden-reserve)
  - [Iceberg Refresh Logic](#iceberg-refresh-logic)
- [Hidden / Dark Orders](#hidden--dark-orders)
- [Pegged Orders](#pegged-orders)
- [Discretionary Orders](#discretionary-orders)
- [MOO and MOC — Auction Orders](#moo-and-moc--auction-orders)
- [Auction Mechanics — Opening, Closing, Periodic](#auction-mechanics--opening-closing-periodic)
  - [The Uncrossing Algorithm](#the-uncrossing-algorithm)
- [Self-Match Prevention (SMP)](#self-match-prevention-smp)
- [Cancel/Replace Semantics — When You Lose Time Priority](#cancelreplace-semantics--when-you-lose-time-priority)
- [Order ID Lineage — client\_order\_id, exchange\_order\_id, execution\_id](#order-id-lineage--client_order_id-exchange_order_id-execution_id)
- [Routing and Sharded Matching Engines](#routing-and-sharded-matching-engines)
- [ISO — Intermarket Sweep Orders Under Reg NMS](#iso--intermarket-sweep-orders-under-reg-nms)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

Order type semantics are where intent meets the matching engine. A market order is not "buy now at any price" — at most modern exchanges it is a limit order with a protection band, because pure market orders cause clearing-price disasters in thin books. A limit order is the universal primitive: it crosses immediately if marketable and rests as passive liquidity otherwise. Stop and stop-limit orders are not on the book at all until a trigger event (last-trade or NBBO touch) converts them. Time-in-force (DAY, GTC, IOC, FOK, GTD, ATO, ATC) is an orthogonal axis that controls how long the order lives and what counts as a fill. IOC fills what it can immediately and cancels the rest; FOK is atomic — fill the whole quantity or reject — and is implemented as a probe-then-commit two-phase pass. Iceberg orders show only a display slice in the L2 feed and refresh from a hidden reserve after each fill; hidden and dark orders are not on the public book at all. Pegged, discretionary, MOO, MOC, and ISO orders each layer additional semantics on top of the same matching primitive. Self-match prevention, cancel/replace priority handling, and ID lineage (`client_order_id` / `exchange_order_id` / `execution_id`) are the cross-cutting concerns. Get any of these wrong and you ship either a regulatory violation, a corruption bug, or both. The discipline that scales is to keep the matching engine's vocabulary tiny — one matching primitive plus a small set of post-processing rules — and let the order type drive the wrapper, never the engine.

## Why Order Type Semantics Are Load-Bearing

A matching engine has exactly one job: walk opposite-side levels and fill while a price condition holds. Every "order type" you read about in an exchange rulebook is either:

1. A **pre-condition** — should this order even reach the engine right now? (Stop, MOO/MOC, ISO routing.)
2. A **price/quantity wrapper** — what do "best price" and "fillable quantity" mean for this order? (Market with protection, peg, discretionary.)
3. A **post-processing rule** — what happens to the unfilled remainder? (Limit rests; IOC cancels; FOK reverses; Iceberg refreshes.)
4. A **visibility wrapper** — what does the L2 feed reveal about this order? (Iceberg display slice; hidden; dark.)

If you can keep that decomposition straight, the rest of this document is the mapping table. If you cannot, you will end up with a 50-state if-ladder inside the hot path of the matching engine, and that is the road to the 3 a.m. determinism bug that erases time priority for ten thousand orders.

The reference architecture lives in [`./matching-engine-determinism.md`](./matching-engine-determinism.md). The price-time-priority data structure that backs it lives in [`./order-book-data-structure.md`](./order-book-data-structure.md). The fan-out path that makes the resulting fills visible lives in [`./market-data-fan-out.md`](./market-data-fan-out.md). This document is about the contract surface that sits between the gateway and the engine.

## The Matching Primitive — One Verb, Many Wrappers

The single primitive every modern matching engine exposes is approximately:

```text
match(side, price_bound, quantity, opposite_book) -> [fills...], remaining_qty
```

- `side` is BUY or SELL.
- `price_bound` is the worst price the order is willing to take. For a limit it is the limit price; for a market-with-protection it is `last_trade ± protection_band`; for a pegged order it is computed at trigger time from the NBBO.
- `quantity` is the requested size; the engine returns however many fills it could produce while the opposite-side top price stays inside the bound.
- The function is **pure** with respect to the input book and the order — same inputs always produce the same outputs. This determinism is what makes the journal-replay model work (see [`./matching-engine-determinism.md`](./matching-engine-determinism.md)).

Every order type's specification reduces to: how do I compute `price_bound`, what do I do with `remaining_qty`, and which book do I rest on (lit, hidden, none)? The rest is bookkeeping.

## Market Order — Fast, Dangerous, Often Disabled

### Naive Semantics

A pure market order is `match(side, ±∞, qty, book)`: walk the opposite side from the top, taking each price level fully (or partially on the last) until `qty` is exhausted. It is the simplest order type to describe and the most dangerous one to ship.

### Slippage and Why Pure Market Orders Are Rare

In a thin book, a 10,000-share market buy can walk through twenty price levels before it is satisfied. The trader's "I want to buy at the market" intent does not include "even if the market moment is a 4% spike caused by a single fat-finger sell". Pure market orders during volatile open/close windows are the canonical cause of "flash crash" trades that get busted later.

For this reason, **NYSE and Nasdaq do not accept naked market orders during continuous trading on most equities** — they convert market orders to limit-with-protection orders at the gateway. Many international venues never allowed pure market orders at all. The Nasdaq Order Types reference and the NYSE Order Types pages document the specific protection bands per product.

### Market-with-Protection

The standard implementation:

1. At order arrival, snapshot the current NBBO (or last-trade price).
2. Compute a protection band — typically a percentage (e.g., 5%) or a fixed number of ticks away from the reference price.
3. Submit the order as a limit at `reference_price + protection` for buys, `reference_price − protection` for sells.
4. Apply IOC TIF (see below) — any unfilled remainder cancels rather than rests at the unrealistic protection price.

```text
on receive market_buy(qty):
    ref = nbbo.ask_top_price  # or last_trade
    protected_limit = ref * (1 + protection_pct)
    submit_limit(side=BUY, price=protected_limit, qty=qty, tif=IOC)
```

The trader gets "fast", they do not get "at any price." If the book is too thin to fill within the band, the unfilled remainder cancels and the trader receives an execution report with the partial fill plus a `OrdStatus=Canceled` for the residual.

This is the lesson: **the matching engine never sees a market order**. Market orders are a gateway concept that is rewritten before sequencing. The engine's vocabulary stays small.

## Limit Order — The Resting Workhorse

### Crossing vs Resting

A limit order is either:

- **Marketable / crossing** — its limit price is at or beyond the opposite-side top, so the engine can fill some or all of it immediately.
- **Non-marketable / resting** — its limit price is on its own side of the spread; nothing matches now, the order sits on the book at price-time priority.

The engine handles both with one code path: try to match; whatever remains, insert into the book at the limit price.

### Marketable Limit and Locked Markets

A "locked market" is one where bid == ask across venues; a "crossed market" is bid > ask. Both are unusual in regulated equities (Reg NMS forbids deliberately crossing the NBBO via standard orders) but possible during fast moves and across illiquid books. ISO orders (see below) are the explicit Reg NMS escape hatch for sweeping multiple venues that together would cross.

### Limit Matching Pseudocode

```text
fn match_limit(order, book):
    fills = []
    opposite = book.opposite_side(order.side)

    while order.qty_remaining > 0:
        top = opposite.top_level()
        if top is None:
            break
        if not crosses(order.price, top.price, order.side):
            break  # no more marketable price levels

        # Walk the FIFO at this level
        while order.qty_remaining > 0 and not top.empty():
            resting = top.head()
            trade_qty = min(order.qty_remaining, resting.qty_remaining)

            # Self-match prevention check (see SMP section)
            if smp_blocks(order, resting):
                handle_smp(order, resting, fills)
                continue

            fills.append(Fill(
                buy_id  = order.id  if order.side == BUY  else resting.id,
                sell_id = resting.id if order.side == BUY else order.id,
                price   = resting.price,            # taker pays maker's price
                qty     = trade_qty,
                ts      = engine.now(),
            ))
            order.qty_remaining   -= trade_qty
            resting.qty_remaining -= trade_qty
            if resting.qty_remaining == 0:
                top.pop()
            # Iceberg refresh fires here — see Iceberg section

        if top.empty():
            opposite.remove_level(top.price)

    if order.qty_remaining > 0 and order.tif != IOC and order.tif != FOK:
        book.same_side(order.side).insert(order)  # rests at price-time priority

    return fills, order.qty_remaining
```

A few invariants worth calling out:

- **Maker price wins.** The fill print is at the resting order's price, not the taker's limit. A buy limit at $10.05 hitting a resting sell at $10.00 prints at $10.00. The taker gets price improvement; the maker gets their advertised price. This is universal across price-time priority venues.
- **Time priority is FIFO within a price level.** The structure backing each level is a doubly-linked queue keyed on insertion time (sequence number, not wall-clock). See [`./order-book-data-structure.md`](./order-book-data-structure.md).
- **The `crosses` predicate is `taker_price >= maker_price` for buys, `<=` for sells.** That single sign flip is the only place "side" appears inside the matching loop.

## Stop and Stop-Limit — Triggered Orders

### Trigger Semantics — Last-Trade vs NBBO

A stop order is **not** on the lit book. It sits in a separate triggered-stop structure keyed by trigger price and side. Two trigger conventions exist:

- **Last-trade trigger** — the stop fires when a print at the trigger price (or beyond) is published. This is the dominant US equities convention.
- **NBBO trigger** — the stop fires when the NBBO bid (for sell stops) or ask (for buy stops) touches or crosses the trigger price.

Each exchange documents which it uses. CME futures and many international venues use last-trade. The engine must be explicit; ambiguity here causes traders to lose money on flash prints they did not consent to chase.

On trigger:

- A **stop** order becomes a market order — usually market-with-protection, per the gateway rule above.
- A **stop-limit** order becomes a limit order at the configured limit price.

### Stop Trigger Event Handler

```text
fn on_trade_print(symbol, price, qty):
    # Sell stops fire when price <= stop_price
    triggered_sells = stop_book.sells.pop_at_or_below(price)
    for stop in triggered_sells:
        engine.inject(stop.to_marketable_order())

    # Buy stops fire when price >= stop_price
    triggered_buys = stop_book.buys.pop_at_or_above(price)
    for stop in triggered_buys:
        engine.inject(stop.to_marketable_order())
```

`engine.inject` enqueues the converted order at the **current** sequence point, not at the original arrival time — once a stop triggers, it competes for time priority with whatever else is arriving. The conversion event must be journaled (input log) so replay reproduces the same triggers.

A subtle implementation note: the trade print that triggers stops can itself create stop-cascading sequences (a stop's market order walks the book, triggering more stops). The engine processes each injected order to completion before draining the next, but the journal records the trigger order strictly so replay matches. This is one of the reasons single-threaded determinism per symbol is non-negotiable.

### Stop-Limit Pitfalls

A stop-limit at trigger 100 / limit 99.50 on a sell will not fill if the price gaps through to 99.00 — the order rests at 99.50 in a market that has moved past it. This is by design ("I am willing to accept worse than 100 but no worse than 99.50") but surprises retail users who think "stop" means "guaranteed exit". The exchange docs are explicit; gateways often surface a confirmation dialog.

## Time-in-Force (TIF)

TIF is an orthogonal axis from order type. A limit order can be DAY, GTC, GTD, IOC, or FOK; a market order is essentially always IOC; an MOO order is by definition ATO TIF.

### DAY, GTC, GTD

- **DAY** — order is live for the current trading session; cancel-on-close at the auction.
- **GTC (Good-Til-Canceled)** — persists across sessions until filled or canceled. Most US exchanges no longer support pure GTC at the venue level (member firms simulate it by re-submitting DAY orders each morning). FIX `TimeInForce=1` is the field code.
- **GTD (Good-Til-Date)** — persists until a specified date/time, then auto-cancels. FIX `TimeInForce=6`.

### IOC — Immediate-or-Cancel

IOC means: match what is immediately available; cancel any remainder. It never rests on the book. This is the same shape as the market-with-protection rewrite from earlier — IOC is the TIF that makes the "no resting" guarantee.

```text
fills, remaining = match(order, book)
if remaining > 0:
    emit ExecutionReport(order_id=order.id, ord_status=CANCELED, leaves_qty=0,
                         cum_qty=order.qty - remaining, last_qty=0)
# Order is never inserted into book.same_side
```

IOC is the right TIF for: market-style orders, smart-order-router child orders that probe a venue, and aggressive sweeping orders that do not want to expose intent by resting.

### FOK — Fill-or-Kill

FOK is stricter: fill the **entire** quantity immediately, or cancel **everything**. Partial fills are forbidden. FOK is rare in modern equities (it discriminates against thin books) but standard in some block-trading and crypto venues.

The implementation is non-trivial — see [FOK Two-Phase Logic](#fok-two-phase-logic) below.

### ATO and ATC — Auction-Only TIFs

- **ATO (At-The-Open)** — only participates in the opening auction.
- **ATC (At-The-Close)** — only participates in the closing auction.

These are TIFs in some venues and order types (MOO/MOC) in others; semantically they are the same. The order is held off the continuous book and only enters the matching pool during the uncross.

## FOK Two-Phase Logic

A naive FOK implementation that just runs the limit-match loop and rolls back if `remaining > 0` is **not safe** — the moment you publish even one fill print, downstream consumers act on it (risk systems update positions, marketable-with-protection algos recompute their next move). You cannot un-print a trade.

The standard approach is **probe-then-commit**, executed atomically inside the engine's single-threaded core:

```text
fn match_fok(order, book):
    # Phase 1: probe — read-only walk through the opposite book
    probe_qty = 0
    probe_levels = []
    for level in book.opposite_side(order.side).levels_from_top():
        if not crosses(order.price, level.price, order.side):
            break
        for resting in level.queue_iter():
            if smp_blocks(order, resting):
                continue
            available = resting.qty_remaining
            take = min(order.qty_remaining - probe_qty, available)
            probe_levels.append((resting.id, level.price, take))
            probe_qty += take
            if probe_qty >= order.qty_remaining:
                break
        if probe_qty >= order.qty_remaining:
            break

    if probe_qty < order.qty_remaining:
        # Phase 2a: kill — emit a single Canceled report, no fills published
        emit ExecutionReport(order_id=order.id, ord_status=CANCELED,
                             leaves_qty=0, cum_qty=0)
        return [], order.qty_remaining

    # Phase 2b: commit — same walk, now mutating, guaranteed to fill exactly
    fills = []
    for (resting_id, price, take) in probe_levels:
        resting = book.find(resting_id)
        # Resting cannot have changed: we are single-threaded and no other
        # event has been processed since the probe (key invariant).
        execute_fill(order, resting, price, take, fills)
    return fills, 0
```

Three invariants make this work:

- **Single-threaded engine.** Between probe and commit, no other order is processed. The probe's view is the commit's view. This is one of the reasons matching engines do not multi-thread per symbol.
- **No allocation/IO in the probe.** The probe is a tight read-only loop; it must not block.
- **Deterministic SMP decisions.** The same SMP rule that filters during the probe must filter identically during the commit.

If your engine cannot guarantee single-threaded probe+commit, FOK is unsafe and should not be offered.

## Iceberg / Reserve Orders

### Display Slice vs Hidden Reserve

An iceberg order has two sizes:

- **Display size** (the "tip") — the quantity visible in the L2 market data feed.
- **Total size** (display + reserve) — the actual remaining quantity.

When a fill exhausts the display slice, a new slice is "refreshed" from the reserve. Each refresh is an L2 update — the level appears to receive a new resting order, which means it goes to the **back** of the time-priority queue at that price (per most exchanges' rules; some venues offer "priority preservation" but the default is time-priority loss on refresh).

The hidden reserve is held inside the engine's order record but is **not** published in L2. The L1/L2 feeds (per ITCH 5.0) only carry the display slice. The full size is visible to (a) the order owner, (b) the exchange's own systems, and (c) the regulator.

### Iceberg Refresh Logic

```text
fn on_fill(order, fill_qty):
    order.display_remaining -= fill_qty
    order.total_remaining   -= fill_qty

    if order.is_iceberg and order.display_remaining == 0 and order.total_remaining > 0:
        # Refresh: new display slice from reserve
        slice = min(order.display_size, order.total_remaining)
        order.display_remaining = slice

        # Per most exchanges: the refreshed slice loses time priority
        order.time_priority_seq = engine.next_seq()
        book.level_at(order.price).move_to_tail(order)

        # Publish the refresh as a new add at this level (display size only)
        publish_add(order.id_for_md, order.price, slice)
        # Note: the original "delete" of the exhausted slice was published
        # by the fill machinery already.
```

A subtle gotcha: if you publish the refresh as a **modify** instead of a **delete + add**, smart traders can detect icebergs by the modify-without-cancel pattern. Most exchanges document iceberg behavior precisely (Nasdaq, NYSE, CME) and ship the L2 events that match the documentation, not the events that hide the iceberg most effectively — that would itself be a regulatory issue.

## Hidden / Dark Orders

A hidden order is invisible in the L2 feed entirely — no display slice, no level update on insertion, no level update on cancel. It only becomes visible **at the moment of execution** (the trade print is mandatory; the fill cannot be hidden).

Hidden orders are the "dark liquidity" inside a lit exchange. They are different from a true dark pool (a separate venue), but related. The matching engine treats them exactly like any other resting order, with one wrinkle: at most exchanges, hidden orders are **deprioritized** behind displayed orders at the same price (price-display-time priority instead of price-time priority). The rationale is that displayed liquidity earns priority by exposing intent; hidden liquidity gives that up.

Regulator scrutiny on hidden orders is high. SEC Reg NMS, MiFID II, and equivalent regimes require:

- Pre-trade transparency exemptions to be explicit.
- The trade print at execution.
- Exchange disclosure of the percentage of hidden vs displayed flow.

Implementation-wise, hidden orders live in the same book structure as displayed orders, with a `hidden=true` flag and a comparator that breaks ties differently. The journal records them; the L2 publisher filters them; the trade-print publisher does not.

## Pegged Orders

A pegged order's limit price is a **function** of the NBBO that recomputes as the market moves:

- **Mid-peg** — sits at `(NBBO_bid + NBBO_ask) / 2`.
- **Bid-peg** (buy) — pegs to the NBBO bid.
- **Ask-peg** (sell) — pegs to the NBBO ask.
- **Primary peg** — pegs to the same-side top, often with an offset.
- **Market peg** — pegs to the opposite-side top.

A peg can also have an offset (`peg + 1 tick toward more aggressive`) and a cap (`peg, but never beyond limit price X`).

Implementation:

- The engine subscribes to the NBBO feed (internal or SIP).
- On NBBO change, recompute peg prices for all pegged orders on the book; for each whose price changed, cancel-and-reinsert at the new price (loses time priority) or, on some venues, in-place-modify (preserves priority within the new level).
- A cap, if hit, freezes the order at the cap price until the peg moves back inside.

Pegged orders are popular for execution algos that want to provide liquidity without managing the peg by hand. They are also a source of **"quote stuffing" amplification** — a fast NBBO that flips up and down can cause many cancel/insert events per peg per microsecond, which is one reason exchanges throttle peg recomputation cadence.

## Discretionary Orders

A discretionary order has two prices:

- **Display price** — what the L2 feed shows. The order rests here normally.
- **Discretion price** — a more aggressive price the order is **willing** to trade at, but does not advertise.

When an opposite-side order arrives that is marketable against the discretion price (but not the display price), the engine fills it at the discretion price using the hidden willingness. The L2 feed does not show the discretion; the trade print does.

This is conceptually a hybrid of pegged and hidden — display at X, willing to trade up to Y. It rewards traders who want to provide liquidity with a layered defense without giving up the entire range.

The matching engine implementation is a small wrapper: when comparing prices for a cross, use `discretion_price` for the marketable check but record the fill at the `display_price` (or vice versa, per exchange rule) — the exact convention is exchange-specific and must be documented in the trader-facing spec.

## MOO and MOC — Auction Orders

- **MOO (Market-on-Open)** — participates only in the opening auction at the exchange's official open price.
- **MOC (Market-on-Close)** — participates only in the closing auction at the official close price.

These orders are held off the continuous book entirely. They are accumulated during the auction collection window (e.g., Nasdaq Cross collects MOC orders until 3:50 p.m. ET; NYSE Closing Auction collects until 3:50 p.m. ET with imbalance dissemination from 3:50 onward).

The L2 feed during continuous trading does **not** include MOO/MOC orders. Imbalance feeds (Nasdaq Net Order Imbalance Indicator, NYSE Order Imbalance Information) publish aggregate buy/sell imbalances and the indicative auction price, but the individual orders stay private until the cross.

## Auction Mechanics — Opening, Closing, Periodic

### The Uncrossing Algorithm

An auction collects buy and sell orders during a defined window. At the cross moment, the engine computes the **clearing price** that maximizes executed volume, with secondary tiebreakers per exchange rule:

1. For each candidate price in the union of all order prices in the auction:
   - Compute total buy volume willing to trade at price ≥ candidate.
   - Compute total sell volume willing to trade at price ≤ candidate.
   - Executed volume at this candidate = `min(buy_vol, sell_vol)`.
2. Pick the candidate that maximizes executed volume.
3. If multiple candidates tie:
   - Tiebreaker 1: minimize imbalance (`|buy_vol − sell_vol|`).
   - Tiebreaker 2: pick price closest to the reference (last-trade or NBBO mid).
   - Tiebreaker 3: pick the lower of the two tied prices (Nasdaq) or apply a venue-specific rule (NYSE has its own).
4. At the chosen price, allocate fills using the venue's priority rule (typically pro-rata for the marginal level, full fill for inside orders).

```text
fn uncross(auction):
    candidates = sorted(set(o.price for o in auction.orders))
    best = None
    for px in candidates:
        buy_vol  = sum(o.qty for o in auction.buys  if o.price >= px)
        sell_vol = sum(o.qty for o in auction.sells if o.price <= px)
        exec_vol = min(buy_vol, sell_vol)
        imbalance = abs(buy_vol - sell_vol)
        score = (exec_vol, -imbalance, -abs(px - reference_price))
        if best is None or score > best.score:
            best = (px, exec_vol, score)
    cross_price = best[0]
    allocate_fills(auction, cross_price)
```

Auctions are deterministic given the input order set and tiebreak rules — the journal-replay model continues to apply. The auction is effectively a single matching event in the engine's input log, even though it processes thousands of orders at once.

Periodic auctions (e.g., MiFID II "frequent batch auctions") are the same algorithm running every few hundred milliseconds, between continuous matching windows, used as a defense against latency arbitrage.

## Self-Match Prevention (SMP)

Two orders from the same beneficial owner trading with each other is, depending on intent, either (a) a wash trade (illegal under most regulators) or (b) merely wasteful (member pays both sides of the spread plus exchange fees). Either way, exchanges block it.

The standard SMP rule set:

- **Cancel newest** — if the incoming order would match a resting order from the same owner, cancel the incoming order's marketable portion at this level. The resting order remains.
- **Cancel oldest** — cancel the resting order, let the incoming order continue.
- **Decrement and cancel** — reduce both orders by the would-have-traded quantity and cancel the smaller; the larger continues with the residual.
- **Reject** — reject the incoming order entirely (rare; harshest).

The decision must be **deterministic** and recorded in the journal. SMP violations are the kind of thing regulators sample-audit the journal for; a non-deterministic SMP rule is a guaranteed CAT discrepancy.

The check is keyed on a "self-match group" — typically the firm's MPID or a per-account identifier supplied on the order. Two orders share an SMP group if they share this identifier; orders from the same firm but different SMP groups can trade with each other (this is necessary so a firm's market-making desk and its agency desk can both rest orders without blocking each other).

```text
fn smp_blocks(taker, maker):
    if taker.smp_group is None or maker.smp_group is None:
        return False
    if taker.smp_group != maker.smp_group:
        return False
    return True  # decision rule (cancel-new, cancel-old, etc.) handled by handle_smp
```

## Cancel/Replace Semantics — When You Lose Time Priority

A FIX `OrderCancelReplaceRequest` (35=G) lets a member modify an existing order. The semantics question: does the modified order keep its time priority?

The universal rule: **changes that increase aggressiveness lose time priority. Changes that decrease aggressiveness preserve it.** Specifically:

| Change | Time priority |
|---|---|
| Price more aggressive (buy ↑, sell ↓) | **Lost** — replace = cancel + new at new sequence number. |
| Price less aggressive (buy ↓, sell ↑) | Most exchanges: **lost**, because the order is now sitting at a different price level (a different FIFO). Some exchanges allow in-place modify if it stays at the same level (rare). |
| Quantity decrease (size-down) | **Preserved** at most exchanges — same order, smaller. |
| Quantity increase | **Lost** — exchanges treat it as a new resting order to prevent gaming. |
| Other fields (time-in-force, customer ref) | **Preserved** — these do not affect matching priority. |

The implementation is straightforward — a price change or size-up triggers `cancel(old) + new(replacement)`, both journaled. A size-down is an in-place mutation of `qty_remaining` plus a `level.recompute_total()`. The L2 publisher emits the appropriate update.

The reason traders care so much: a hot price level might have a thousand orders ahead of you, accumulated over the morning. A "harmless" price change that bumps you to the back of the queue can cost real money. Member firms that do not understand this rule end up with surprised customers and recurring support tickets.

## Order ID Lineage — client\_order\_id, exchange\_order\_id, execution\_id

Three IDs travel with every order, and confusing them is a class of bug all its own:

- **`client_order_id`** (FIX `ClOrdID`, tag 11) — assigned by the **member firm**. It must be unique per firm per session per day. The exchange uses this to detect duplicates from the firm and to acknowledge orders back. A cancel/replace flow uses `OrigClOrdID` (tag 41) to reference the original and a fresh `ClOrdID` for the replacement.
- **`exchange_order_id`** (FIX `OrderID`, tag 37) — assigned by the **exchange** on order acknowledgment. It is the canonical ID inside the matching engine and across the audit trail. Once assigned, it is immutable for the life of the order — even across cancel/replace, the new replacement gets a **new** `OrderID` (the original is canceled).
- **`execution_id`** (FIX `ExecID`, tag 17) — assigned by the **exchange** per execution report (every fill, every ack, every cancel). It must be unique exchange-wide. A single order can have many `ExecID`s as it fills.

Lineage rules:

- **Replay must be reproducible by `OrderID`.** Replaying the input journal must regenerate the same `OrderID` mapping (this is why `OrderID` is assigned deterministically from the sequence number on first ack).
- **CAT submissions key on `OrderID` lineage**, plus the FIX-level `ClOrdID` chain. A regulator reconstructing an order's life from the audit trail follows `OrderID` for the engine view and `ClOrdID` chain for the firm view.
- **Self-match prevention groups are an additional axis** — they identify the *owner*, not the order or execution.

Anti-pattern: using `ClOrdID` as the engine's primary key. Two firms can use the same `ClOrdID` (it is only required unique per firm), so it is not globally unique. The exchange must assign its own ID.

## Routing and Sharded Matching Engines

A modern exchange runs **multiple matching engines** — one (or a small group) per symbol. The router at the gateway maps symbol → engine. Each engine is single-threaded and deterministic per its assigned symbols; the union of engines provides per-exchange capacity.

The routing table is small but critical:

- **Static map** — symbols are pre-assigned to engines based on liquidity profile (mega-active symbols like SPY get their own engine; less-active ones share).
- **Reload on rebalance** — moving a symbol between engines requires a coordinated halt, quiesce, and restart, journaled as part of the input log so replay is correct.
- **Failover** — each engine has a hot standby that tails the journal; on primary failure the standby takes the next sequence number (see [`./matching-engine-determinism.md`](./matching-engine-determinism.md) for the full failover protocol).

Cross-symbol order types (basket trades, spread trades) are coordinated by a separate orchestrator that splits the basket into per-symbol legs and reconciles the legs as a multi-step workflow. The matching engines themselves only ever see single-symbol orders.

## ISO — Intermarket Sweep Orders Under Reg NMS

Reg NMS Rule 611 (the "Order Protection Rule") forbids exchanges from executing trades at prices inferior to a protected quote on another venue. In normal flow, the gateway must route the order to the venue with the best price.

An **ISO** is the explicit escape hatch: a member firm marks an order ISO and represents (under regulatory penalty) that it is **simultaneously routing additional ISOs to all other venues with better prices**. The receiving exchange may then execute the ISO at its local price even if a better quote exists elsewhere — because the firm is sweeping all of those venues at once.

Mechanically:

- An ISO is a limit order with an `ExecInst=f` (FIX) flag.
- The receiving engine treats it like any other limit order, but bypasses the NBBO trade-through check that would normally reject or reroute the order.
- The firm is responsible for actually routing the parallel ISOs; the exchange does not verify this — the audit trail does, post hoc.

Wash-sale, manipulation, and trade-through enforcement under Reg NMS depend on the journal recording every ISO with its `ExecInst` flag and the corresponding parallel orders being traceable across the consolidated audit trail (CAT). Misuse of the ISO flag — claiming ISO without actually sweeping — is an enforcement matter.

The takeaway for exchange design: ISO is a small flag with large semantic weight. The journal must record it; the matching engine must respect it; the audit pipeline must pair it with the firm's other ISOs to validate the claim.

## Anti-Patterns

1. **Letting market orders into the engine without protection.** Pure market orders cause clearing-price disasters in thin books. Convert to limit-with-protection at the gateway. The engine's vocabulary stays small and safe.
2. **Partial fills on FOK.** FOK is "fill the whole thing or nothing." A partial fill is a regulatory violation, not a UX inconvenience. Implement probe-then-commit; if your engine cannot guarantee atomicity, do not offer FOK.
3. **Missing self-match prevention.** Two orders from the same beneficial owner crossing each other is a wash trade. SMP is not optional; the deterministic rule must be journaled and visible to the regulator.
4. **Time priority lost on innocuous modifies.** A "harmless" cancel/replace that bumps a member from #1 to #1000 in the queue is a 7-figure customer support escalation. Document the rule precisely; preserve priority for size-down; document everything else as priority-loss.
5. **Trusting wall-clock for time priority.** Use the sequence number from the journal. Wall-clock drifts; sequence numbers are gap-free and total-ordered.
6. **Iceberg refresh as an L2 modify.** Easy to detect ("modify without cancel = iceberg"). Most exchanges publish delete + add. Match the documented behavior; do not invent your own.
7. **Hidden orders without trade-print.** The trade print is mandatory regardless of pre-trade visibility. Hiding the print is fraud.
8. **Pegged orders without a recompute throttle.** A fast-flipping NBBO can cause millions of peg-modify events per second. Throttle the recompute cadence; bound the cancel/insert rate.
9. **Reusing `client_order_id` as the engine's primary key.** It is only unique per firm. Two firms can ship the same `ClOrdID` for completely different orders. The exchange assigns its own `OrderID`.
10. **ISO routing without parallel-sweep verification.** A firm claiming ISO must actually sweep the other venues. The exchange takes the claim on faith, but the consolidated audit trail must be reconcilable post hoc.
11. **Auction algorithm tiebreakers undocumented.** A trader cannot reproduce the cross price → loss of trust. Document the candidate set, the maximize-volume rule, the imbalance tiebreaker, and the reference-price tiebreaker exactly.
12. **Stop trigger semantics unspecified (last-trade vs NBBO).** Traders blow up on flash prints they did not consent to. The exchange spec must be unambiguous; the engine must implement it precisely; replay must reproduce it.
13. **MOO/MOC mixed into the continuous book.** Auction orders never participate in continuous matching. Hold them in a separate auction-collection structure; release them only at the cross.
14. **Routing decisions made inside the matching engine.** The matching engine processes single-symbol streams; routing is the gateway's job. Mixing them defeats the determinism property and complicates failover.

## Related

- [Designing a Stock Exchange](../design-stock-exchange.md) — parent case study; this doc expands the "Order Type Semantics" section with full implementation detail.
- [Order Book Data Structure](./order-book-data-structure.md) — the price-level + FIFO queue structure that backs every order type discussed here _(planned)_.
- [Matching Engine Determinism](./matching-engine-determinism.md) — single-thread per symbol, sequencer-led replay, and the failover protocol that underpins FOK probe-then-commit and stop trigger reproducibility _(planned)_.
- [Market Data Fan-out](./market-data-fan-out.md) — the L2 / ITCH-style publishing path that surfaces (or hides) the order types' visible semantics _(planned)_.
- [Designing a Payment System](../design-payment-system.md) — sister case study; the auth/settlement split mirrors the match/clearing split and the same idempotency discipline applies to fills.
- [API Design Styles](../../../building-blocks/api-design-styles.md) — REST vs streaming vs binary protocols; FIX and OUCH are the order-entry protocols that carry the order types described here.

## References

- [Nasdaq Order Types and Modifiers](https://www.nasdaqtrader.com/Trader.aspx?id=OptionsTradingChecklist) — Nasdaq's authoritative listing of supported order types, modifiers, and TIFs across products.
- [NYSE Pillar — Order Types and Modifiers](https://www.nyse.com/markets/nyse-arca/trading-info) — NYSE Arca order type reference; equivalent listings exist for NYSE and NYSE American.
- [SEC Final Rule — Regulation NMS (Release 34-51808)](https://www.sec.gov/rules/final/34-51808.pdf) — the foundational Reg NMS document, including Rule 611 (Order Protection) and the ISO definition.
- [CME Group — Introduction to Order Types](https://www.cmegroup.com/education/courses/intro-to-order-types.html) — futures-side reference; covers stop, stop-limit, GTC, GTD, MOO/MOC equivalents in the CME Globex context.
- [FIX 5.0 — OrdType (tag 40)](https://www.onixs.biz/fix-dictionary/5.0/tagNum_40.html) — the canonical enumeration of order types over FIX, with field codes used in production order-entry sessions.
- [FIX 5.0 — TimeInForce (tag 59)](https://www.onixs.biz/fix-dictionary/5.0/tagNum_59.html) — the TIF enumeration: DAY, GTC, OPG (ATO), IOC, FOK, GTX, GTD, ATC.
- [Nasdaq TotalView-ITCH 5.0 Specification](https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHspecification.pdf) — the binary multicast feed; describes which order types appear in L2 (displayed) vs are hidden, and the add/modify/delete event shapes that iceberg refresh and pegged-order modify map onto.
- [Joel Hasbrouck — *Empirical Market Microstructure*](https://global.oup.com/academic/product/empirical-market-microstructure-9780195301649) — the textbook reference on price formation, order flow, and the economic role of the order types described here.
- [SEC — Consolidated Audit Trail (CAT) Industry Plan](https://www.catnmsplan.com/) — the regulatory specification for daily order-lifecycle reporting; the destination for the journal data that captures every order type, ISO claim, and SMP decision discussed above.
- [LMAX Architecture — Martin Fowler](https://martinfowler.com/articles/lmax.html) — the canonical reference for the single-threaded deterministic matching model that makes FOK probe-then-commit, stop triggers, and auction uncross algorithms tractable.
