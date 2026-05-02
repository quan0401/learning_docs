---
title: "RRULE Expansion — Materialize vs Compute on Demand for Recurring Events"
date: 2026-05-01
updated: 2026-05-01
tags: [system-design, deep-dive, calendar, rrule, recurrence]
---

# RRULE Expansion — Materialize vs Compute on Demand for Recurring Events

**Date:** 2026-05-01 | **Updated:** 2026-05-01
**Tags:** `system-design` `deep-dive` `calendar` `rrule` `recurrence`

> **Parent case study:** [Design a Calendar System](../design-calendar-system.md). This deep-dive expands "§1 RRULE Expansion".

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [RFC 5545 RRULE — The Tiny DSL Behind Every Recurring Event](#rfc-5545-rrule--the-tiny-dsl-behind-every-recurring-event)
- [The Expansion Problem — One Rule, Infinite Occurrences](#the-expansion-problem--one-rule-infinite-occurrences)
- [Strategy 1 — Materialize-All Within a Bounded Window](#strategy-1--materialize-all-within-a-bounded-window)
- [Strategy 2 — Materialize-on-Read with LRU Cache](#strategy-2--materialize-on-read-with-lru-cache)
- [Strategy 3 — Compute-on-Demand Per Query](#strategy-3--compute-on-demand-per-query)
- [The Hybrid Production Choice](#the-hybrid-production-choice)
- [EXDATE and RDATE — Exception Dates Added and Removed](#exdate-and-rdate--exception-dates-added-and-removed)
- [Single-Instance Overrides — RECURRENCE-ID](#single-instance-overrides--recurrence-id)
- [Detached Instances — "This and Future" Edits](#detached-instances--this-and-future-edits)
- [Storage Schema — Master Plus Materialized View](#storage-schema--master-plus-materialized-view)
- [Indexing for Range Queries](#indexing-for-range-queries)
- [Open-Ended Series — COUNT and UNTIL](#open-ended-series--count-and-until)
- [Library Choice — Don't Roll Your Own](#library-choice--dont-roll-your-own)
- [Performance Numbers](#performance-numbers)
- [Worked Example — Weekly Engineering Sync](#worked-example--weekly-engineering-sync)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

A recurring calendar event is a tiny program. The user types "every weekday at 9 AM until end of year" and the system stores a single 60-character `RRULE` line — `FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR;UNTIL=20271231T235959Z` — that, when *evaluated*, expands into 260-ish virtual instances, any of which can be individually cancelled (`EXDATE`), individually overridden (`RECURRENCE-ID`), or split off into a new series ("change this and all future occurrences"). Storing each occurrence as its own row would be both wasteful and wrong: wasteful because a daily standup over five years is 1,825 rows where one rule plus a handful of overrides expresses everything more honestly, and wrong because editing the master rule (move the standup to 9:30) would require a bulk-update job over thousands of rows that are about to be regenerated anyway. The opposite extreme — *never* materialize, evaluate the rule on every query — is also wrong, because a calendar grid view for one user across one month involves merging dozens of recurring series, and re-running an RRULE parser on every page load is CPU-bound work that scales linearly with event count and quadratically with attendee count for free-busy. The production answer is **the hybrid**: keep the master event row with its RRULE blob and override rows in the canonical store; precompute a rolling materialized view (often [-13 months, +13 months] from now) into a per-user, per-month partitioned table that the read path queries directly; refresh the window nightly and on every series edit; fall through to on-demand expansion only for cold reads outside the window. This deep-dive walks the three strategies, the three flavors of exception (`EXDATE`, `RDATE`, `RECURRENCE-ID`), the data-model split on a "this and future" edit (terminate the original series with `UNTIL`, create a new series at the split point, copy forward overrides whose recurrence-id is past the split), the indexing shape (B-tree on `(user_id, dtstart_utc)` for materialized rows; partial index on `recurring=true` for the master scan that drives the materializer), and the worked example of a weekly meeting expanded for the next quarter. The anti-patterns at the end are the bugs that actually ship: storing 365 rows for a daily event, expanding the rule on every query, missing EXDATE during expansion (the cancelled instance shows up anyway), and forgetting that an open-ended `RRULE` with no `COUNT` or `UNTIL` is an unbounded sequence — materialize without a window cap and storage grows forever.

## Overview

The parent case study (`../design-calendar-system.md`) introduces RRULE expansion in §1 and lists three strategies in a comparison table. This document opens the *why* behind those choices and walks through the operational consequences a production calendar must handle.

The questions answered here:

1. **Why is the RRULE so small and the expansion so large?** Because RRULE is a *generative* expression, not enumerated data. `FREQ=DAILY` with no termination is mathematically infinite; the storage cost is one row, the read cost is bounded by the query window, and the edit cost is one row.
2. **Why is materialize-all wrong even when it seems simple?** Because open-ended series exist (`UNTIL=99991231T235959Z` is a real iCal idiom for "forever"), and because every series edit invalidates every materialized row. A `THIS_AND_FOLLOWING` edit on a 5-year series invalidates 1,200 rows; doing this synchronously on the user's edit blocks the UI.
3. **Why is compute-on-demand wrong at scale?** Because a user viewing their April calendar has 50+ recurring series and the read path must call the RRULE expander 50 times per page load, each call parsing the rule fresh, each call iterating from the series start to find occurrences in April. A single user's month view becomes a CPU bottleneck.
4. **What is the hybrid?** Master event row + override rows + a per-user materialized view of [-13 months, +13 months] kept warm by a background materializer job. Reads in the window are O(1); reads outside fall through to on-demand. Edits invalidate the affected window for the affected user(s), enqueueing a background recompute.
5. **What about EXDATE and RECURRENCE-ID?** Two different mechanisms. `EXDATE` says "the rule would have produced an instance on this date but skip it"; `RECURRENCE-ID` says "the rule produced an instance on this date but its details are overridden by a separate row."
6. **What does "this and future" actually mean in storage?** Split the master series at the edit point: set `UNTIL` on the original to just before the edit's `recurrence_id`, create a new master starting at `recurrence_id` with the new properties. Copy forward any future overrides.
7. **Which library?** Use a vetted one. `python-dateutil.rrule`, `ical4j`, `rrule.js` (`node-rrule`), `libical`. RRULE has corner cases (`BYSETPOS` interaction with `BYDAY`, week-numbering at year boundaries, leap-year `BYYEARDAY=366`) that bite handwritten implementations.

```mermaid
flowchart TB
    user[User: weekly Monday/Wednesday/Friday<br/>at 9 AM until 2027-12-31]
    user --> master[(events row<br/>id=evt-1<br/>rrule='FREQ=WEEKLY;BYDAY=MO,WE,FR;UNTIL=...'<br/>dtstart=2026-04-06T09:00:00<br/>tz=America/New_York)]
    master --> strategies{Expansion strategy}

    strategies -->|all upfront| matAll[Materialize all<br/>~270 rows in user_calendar_view<br/>O(1) read · O(N) edit]
    strategies -->|on read| matRead[LRU cache<br/>per-user-per-month<br/>O(1) hit · O(K) miss]
    strategies -->|never| compute[Evaluate RRULE per query<br/>O(K) every read]
    strategies -->|hybrid prod| hybrid[Materialize +/- 13 months<br/>fall through outside]

    master --> exdate[(event_exdates<br/>cancelled instances<br/>e.g. 2026-12-25)]
    master --> overrides[(event_overrides<br/>per-instance edits keyed<br/>by recurrence_id)]

    hybrid --> read[Read path:<br/>SELECT FROM user_calendar_view<br/>WHERE user_id=? AND yyyy_mm=?]
    hybrid --> edit[Edit path:<br/>invalidate window for affected users<br/>enqueue materializer job]

    style master fill:#cfe
    style hybrid fill:#fec
    style exdate fill:#fcc
    style overrides fill:#ccf
```

The general calendar architecture lives in `../design-calendar-system.md`; this doc specifically handles the *recurrence-expansion* dimension — how a calendar produces the right list of instances from a generative rule plus a list of exceptions.

## RFC 5545 RRULE — The Tiny DSL Behind Every Recurring Event

RFC 5545 §3.3.10 specifies `RRULE` (recurrence rule) as a semicolon-separated list of name/value pairs. The grammar is small but the corners are sharp.

The required first part is `FREQ`, taking one of `SECONDLY`, `MINUTELY`, `HOURLY`, `DAILY`, `WEEKLY`, `MONTHLY`, `YEARLY`. Production calendars typically constrain to `DAILY` and above — a `SECONDLY` recurrence is theoretically valid but operationally hostile.

The other parts are filters and modifiers:

| Part | Meaning | Example |
|------|---------|---------|
| `INTERVAL` | Multiplier on `FREQ` | `FREQ=WEEKLY;INTERVAL=2` = every two weeks |
| `BYDAY` | Day-of-week selector, optionally numbered | `BYDAY=MO,WE,FR` or `BYDAY=1MO` (first Monday) |
| `BYMONTHDAY` | Day-of-month selector, can be negative | `BYMONTHDAY=-1` = last day of month |
| `BYMONTH` | Month selector | `BYMONTH=1,4,7,10` = quarterly months |
| `BYYEARDAY` | Day-of-year selector | `BYYEARDAY=100` |
| `BYWEEKNO` | ISO week-number selector (only with `FREQ=YEARLY`) | `BYWEEKNO=1,26` |
| `BYHOUR` / `BYMINUTE` | Time-of-day selectors | `BYHOUR=9,17` |
| `BYSETPOS` | Pick the Nth element of the BYxxx-filtered set | `BYSETPOS=-1` = last element |
| `WKST` | Week-start day (Monday vs Sunday — affects `BYWEEKNO`) | `WKST=MO` (default in EU), `WKST=SU` (US) |
| `COUNT` | Stop after N occurrences | `COUNT=10` |
| `UNTIL` | Stop at or before this UTC instant | `UNTIL=20271231T235959Z` |

`COUNT` and `UNTIL` are mutually exclusive in RFC 5545. If neither is present, the series is open-ended.

Examples worth keeping in your head:

```text
RRULE:FREQ=WEEKLY;BYDAY=MO,WE,FR;UNTIL=20271231T235959Z
  → every Monday/Wednesday/Friday until end of 2027

RRULE:FREQ=MONTHLY;BYDAY=1MO;COUNT=12
  → first Monday of each month, 12 times

RRULE:FREQ=YEARLY;BYMONTH=11;BYDAY=4TH
  → US Thanksgiving (fourth Thursday in November)

RRULE:FREQ=MONTHLY;BYDAY=MO,TU,WE,TH,FR;BYSETPOS=-1
  → last weekday of every month

RRULE:FREQ=DAILY;INTERVAL=2;COUNT=10
  → every other day, 10 times

RRULE:FREQ=DAILY
  → every day, forever (open-ended)
```

The two filters that bite are **`BYSETPOS`** (it operates on the *set produced by the BYxxx filters within one FREQ period*, not on the whole series) and **`BYDAY=1MO` vs `BYDAY=MO`** (the first selects only the first Monday in the period; the second selects every Monday). RRULE library bugs cluster here. The full grammar lives in RFC 5545 §3.3.10 and is required reading before designing storage.

`DTSTART` (the seed instant) is part of the event record, not the RRULE. The expansion is anchored at `DTSTART`. Crucially, the `BYxxx` filters are evaluated against the calendar of `DTSTART` — so a `FREQ=YEARLY;BYMONTHDAY=29;BYMONTH=2` event with a non-leap-year `DTSTART` produces no occurrences (a famously confusing case).

## The Expansion Problem — One Rule, Infinite Occurrences

The defining property of RRULE is that it is *generative*: a small expression that, when evaluated, can produce an unbounded sequence of instants. `FREQ=DAILY` with no `COUNT` or `UNTIL` is mathematically infinite. `FREQ=YEARLY;BYMONTH=2;BYMONTHDAY=29` produces one instance every 4 years, forever.

This creates an asymmetry between **storage** and **read**:

- **Storage cost** is one row regardless of series length.
- **Read cost** depends on the query window: a one-month view of a daily series enumerates ~30 instances; a 10-year view enumerates ~3,650.

The naive "store one row per occurrence" model collapses on three failure modes:

1. **Open-ended series.** `RRULE:FREQ=DAILY` has no terminator. A materializer that tries to enumerate "all instances" loops until it runs out of memory. Even a soft cap ("materialize up to year 2099") produces 27,000 rows for a daily series starting today — for one event, on one calendar.
2. **Edit storms.** Every edit to the master series invalidates every materialized row. A `THIS_AND_FOLLOWING` edit on a 5-year daily series invalidates ~1,800 rows. A bulk import of 500 recurring events from a CalDAV migration produces millions of materialized rows synchronously.
3. **Free-busy fan-out.** A scheduling assistant that fans out across 12 attendees + 3 candidate rooms = 15 free-busy queries. Each query scans materialized instances. A single user with thousands of events × 15 fans-out × P99 contention is a hot read path.

The materialization choice is therefore not "should I materialize?" but "**within what window, refreshed how often, invalidated by what events?**"

## Strategy 1 — Materialize-All Within a Bounded Window

Precompute every instance for every recurring series within a fixed horizon — say, 1 year forward and 1 year back from now — and store one row per instance.

**Storage shape:**

```sql
CREATE TABLE materialized_instances (
  user_id        UUID NOT NULL,
  event_id       UUID NOT NULL,                  -- master event id
  recurrence_id  TIMESTAMPTZ NOT NULL,           -- the original instant the rule produced
  dtstart_utc    TIMESTAMPTZ NOT NULL,
  dtend_utc      TIMESTAMPTZ NOT NULL,
  override_id    UUID,                            -- NULL unless this row was overridden
  is_cancelled   BOOLEAN NOT NULL DEFAULT FALSE,
  PRIMARY KEY (user_id, dtstart_utc, event_id)
);

CREATE INDEX idx_materialized_user_range
  ON materialized_instances(user_id, dtstart_utc, dtend_utc);
```

**Refresh cadence:** A nightly job scans every recurring master event, expands it from `now() - 1y` to `now() + 1y`, and replaces the rows for that event in the materialized table. Daily refresh keeps the window rolling forward.

**Read path:**

```sql
SELECT *
FROM materialized_instances
WHERE user_id = ?
  AND dtstart_utc < ?     -- window end
  AND dtend_utc   > ?     -- window start
ORDER BY dtstart_utc;
```

A single B-tree range scan. P99 < 10 ms for typical month views.

**When it's right:**

- Calendars where reads dominate writes 100:1 or more.
- Series counts in the hundreds, not millions.
- The 1-year horizon is acceptable to users (events 14 months out are rare anyway).

**When it's wrong:**

- Series with `UNTIL=99991231T235959Z` (the iCal idiom for "forever"). The horizon caps it at 1 year, but the user expectation is "forever." Reads at year+2 fall off a cliff.
- Edit-heavy workloads. A `THIS_AND_FOLLOWING` edit on a daily 5-year series rewrites ~365 rows. A bulk import rewrites millions.
- Multi-attendee events where the same instance is materialized once per attendee. A 100-person company-wide weekly all-hands materialized for each attendee for 1 year is `100 × 52 = 5,200` rows for one logical event.

The pure materialize-all is rarely the production answer. It is, however, a useful baseline to compare against.

## Strategy 2 — Materialize-on-Read with LRU Cache

Don't materialize until someone asks. When a query for a specific window arrives, expand all relevant series into the window and cache the result in an LRU keyed by `(user_id, year_month)`.

**Cache shape:**

```text
cache_key:  ("user-123", "2026-04")
cache_val:  [list of instance records for that user in April 2026]
ttl:        24h or "until any of these events is edited"
```

**Read path on cache hit:** O(1) — return the cached list.

**Read path on cache miss:**

1. Load all recurring master events whose effective range overlaps `[2026-04-01, 2026-05-01)` for `user_id=123`.
2. For each, expand the RRULE within the window using the library.
3. Subtract `EXDATE` instances. Apply `RDATE` additions. Apply `RECURRENCE-ID` overrides.
4. Merge with non-recurring events in the window.
5. Sort by `dtstart_utc`.
6. Cache the result; return.

**Eviction:** LRU on cache size; explicit invalidation on event create/update/delete that affects the user's window.

**When it's right:**

- Sparse access patterns where most users look at "this month" and "next month" only — the cache stays warm for the hot windows and cold windows pay the miss cost rarely.
- Storage is constrained and the materialize-all blob would be large.

**When it's wrong:**

- Free-busy workloads. The scheduling assistant queries arbitrary windows for arbitrary attendees; cache hit rate is low.
- Cold-start after a deployment — the cache is empty, every read is a miss, the RRULE expander becomes the system's bottleneck for the first 10 minutes.

The on-read approach is a useful *layer* in the hybrid (Redis or Memcached LRU sitting in front of the materialized view), but rarely the only strategy.

## Strategy 3 — Compute-on-Demand Per Query

Never materialize. Every query — every page load, every free-busy probe — re-evaluates every relevant RRULE for the requested window.

**Storage shape:**

```sql
-- master event row only, no materialized table
CREATE TABLE events (
  id          UUID PRIMARY KEY,
  user_id     UUID NOT NULL,
  rrule       TEXT,                       -- nullable; NULL = single instance
  dtstart     TIMESTAMP NOT NULL,
  dtend       TIMESTAMP NOT NULL,
  schedule_tz TEXT NOT NULL
);

CREATE TABLE event_exdates (
  event_id    UUID NOT NULL,
  exdate      TIMESTAMP NOT NULL,
  PRIMARY KEY (event_id, exdate)
);

CREATE TABLE event_overrides (
  event_id        UUID NOT NULL,
  recurrence_id   TIMESTAMP NOT NULL,
  override_data   JSONB NOT NULL,
  PRIMARY KEY (event_id, recurrence_id)
);
```

**Read path:** for each candidate event whose `[dtstart, until-or-end]` range overlaps the query window, instantiate an RRULE iterator, advance it to the window start, enumerate instances within the window, subtract `EXDATE`, apply overrides.

**Storage:** minimal. One row per series. No background jobs.

**Edit cost:** O(1). One row updated.

**Read cost:** O(K · S) where K is the average instances per series in the window and S is the number of recurring series the user has. For a user with 200 recurring series and a one-month window with ~5 instances per series on average, that's 1,000 RRULE iterations per read.

**When it's right:**

- Single-user, low-traffic systems. Personal calendar apps with one user have plenty of CPU budget.
- Storage is the binding constraint. Compute is free, disk is not.

**When it's wrong:**

- Free-busy at scale. 12 attendees × 3 candidate rooms × 200 recurring series each × RRULE expansion = 10K+ iterations per scheduling query. P99 explodes.
- Cold reads after process restart. The RRULE library has internal caches (compiled rules, parsed BYxxx state); a cold process re-parses every rule on first access.

The on-demand approach is the *fallback* in the hybrid for cold reads outside the materialized window, not the primary read path.

## The Hybrid Production Choice

The hybrid combines the strengths of all three:

1. **Canonical store:** master `events` rows with RRULE blobs, `event_exdates`, `event_overrides`. This is the source of truth and is small.
2. **Materialized view:** per-user-per-month partitioned table holding expanded instances for `[now - 13 months, now + 13 months]`. Refreshed nightly; invalidated on edit.
3. **LRU cache (optional):** hot-window memoization in front of the materialized view for high-traffic users.
4. **On-demand fallback:** for queries outside the materialized window, evaluate the RRULE on the fly and don't bother caching.

```sql
-- Canonical store
CREATE TABLE events (
  id              UUID PRIMARY KEY,
  user_id         UUID NOT NULL,
  rrule           TEXT,
  dtstart         TIMESTAMP NOT NULL,
  dtend           TIMESTAMP NOT NULL,
  schedule_tz     TEXT NOT NULL,
  master_event_id UUID,                   -- non-null for "this and future" splits
  recurring       BOOLEAN GENERATED ALWAYS AS (rrule IS NOT NULL) STORED,
  CHECK (rrule IS NULL OR (rrule LIKE 'FREQ=%' OR rrule LIKE 'RRULE:FREQ=%'))
);

CREATE INDEX idx_events_user_recurring ON events(user_id) WHERE recurring;

-- Materialized view, partitioned by (user_id, year_month)
CREATE TABLE user_calendar_view (
  user_id         UUID NOT NULL,
  yyyy_mm         CHAR(7) NOT NULL,                -- '2026-04'
  event_id        UUID NOT NULL,
  recurrence_id   TIMESTAMPTZ,                     -- NULL for single instance
  dtstart_utc     TIMESTAMPTZ NOT NULL,
  dtend_utc       TIMESTAMPTZ NOT NULL,
  override_id     UUID,
  PRIMARY KEY (user_id, yyyy_mm, dtstart_utc, event_id)
) PARTITION BY LIST (yyyy_mm);

CREATE INDEX idx_view_range
  ON user_calendar_view(user_id, dtstart_utc, dtend_utc);
```

**Read path (the hot path):**

```sql
-- "Show me events for April 2026"
SELECT v.*, e.title, e.location, o.override_data
FROM user_calendar_view v
JOIN events e ON e.id = v.event_id
LEFT JOIN event_overrides o
       ON o.event_id = v.event_id
      AND o.recurrence_id = v.recurrence_id
WHERE v.user_id = ?
  AND v.yyyy_mm = '2026-04'
ORDER BY v.dtstart_utc;
```

A single index range scan with a left-join lookup for overrides. P99 < 20 ms.

**Edit path:**

1. Update the canonical `events` row (or insert an `event_overrides` row, depending on edit mode).
2. Compute the window of affected `(user_id, yyyy_mm)` partitions.
3. Enqueue an asynchronous *materializer job* per affected partition.
4. The materializer expands the RRULE for the affected event in the affected partition, applies EXDATE/RDATE/overrides, and replaces the rows for that event in `user_calendar_view`.

**Materializer job pseudocode:**

```python
def materialize_event_partition(event_id: UUID, user_id: UUID, yyyy_mm: str):
    """
    Recompute user_calendar_view rows for one event in one partition.
    Idempotent: deletes existing rows for (user_id, yyyy_mm, event_id) and replaces.
    """
    event = events.find_by_id(event_id)
    window_start = parse_month_start(yyyy_mm)        # e.g., 2026-04-01T00:00Z
    window_end   = window_start + relativedelta(months=1)

    if not event.is_recurring():
        instances = [event] if event.dtstart_utc < window_end and event.dtend_utc > window_start else []
    else:
        rule = rrulestr(event.rrule, dtstart=event.dtstart_local, tzinfos=tz(event.schedule_tz))
        # Bound the iterator to the window plus enough overflow for events that span a boundary
        raw_instances = rule.between(window_start - DEFAULT_EVENT_DURATION,
                                     window_end,
                                     inc=True)

        exdates = set(event_exdates.find(event_id))
        instances = [i for i in raw_instances if i not in exdates]

        rdates = event_rdates.find(event_id, window_start, window_end)
        instances = sorted(instances + rdates)

    # Apply per-instance overrides
    overrides = event_overrides.find(event_id, window_start, window_end)
    override_by_recid = {o.recurrence_id: o for o in overrides}

    rows = []
    for inst in instances:
        ovr = override_by_recid.get(inst)
        if ovr and ovr.is_cancelled:
            continue
        rows.append({
            "user_id":       user_id,
            "yyyy_mm":       yyyy_mm,
            "event_id":      event_id,
            "recurrence_id": inst,
            "dtstart_utc":   ovr.dtstart_utc if ovr else inst.astimezone(UTC),
            "dtend_utc":     ovr.dtend_utc   if ovr else (inst + event.duration).astimezone(UTC),
            "override_id":   ovr.id if ovr else None,
        })

    with transaction():
        user_calendar_view.delete_where(user_id=user_id, yyyy_mm=yyyy_mm, event_id=event_id)
        user_calendar_view.bulk_insert(rows)
```

The job is per-partition because:

- A `THIS_AND_FOLLOWING` edit invalidates only the partitions from the split point forward, not the historical ones.
- Multiple workers can materialize different partitions of the same event in parallel.
- The blast radius of a buggy edit is one partition, not the whole calendar.

**Materializer scheduler:** a daily cron at low-traffic hours (typically 02:00 user-local for hosted SaaS, or staggered by tenant) that re-runs materialization for all `(user_id, yyyy_mm)` partitions in the rolling window. Catches drift caused by tzdata updates, rule changes that didn't enqueue invalidations, and corrupted partitions.

## EXDATE and RDATE — Exception Dates Added and Removed

RFC 5545 lets a series carry exception lists alongside the RRULE.

**`EXDATE`** removes specific instances that the RRULE would have produced. If the RRULE says "every Monday at 9 AM" and `EXDATE:20261225T090000` is set, the December 25 instance is suppressed. Common use case: cancelling a single occurrence of a weekly meeting without editing the master rule.

**`RDATE`** adds specific instances that the RRULE would *not* have produced. If the RRULE says "first Monday of each month" but the user wants to add a one-off on a specific Friday, `RDATE:20260417T090000` adds that instance to the series.

`EXDATE` and `RDATE` are stored as separate rows in production calendars rather than embedded in the master RRULE blob — they're naturally append-only and queried by date range.

```sql
CREATE TABLE event_exdates (
  event_id    UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  exdate      TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (event_id, exdate)
);

CREATE TABLE event_rdates (
  event_id    UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  rdate       TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (event_id, rdate)
);
```

The expander applies them in this order:

1. Generate raw instances from `RRULE` within the query window.
2. Subtract `EXDATE` set.
3. Add `RDATE` set.
4. Apply `RECURRENCE-ID` overrides (next section).

**The merging logic in pseudocode:**

```python
def merge_with_exceptions(
    rrule_instances: list[datetime],
    exdates: set[datetime],
    rdates: set[datetime],
    overrides: dict[datetime, EventOverride],
) -> list[InstanceRow]:
    """
    Apply EXDATE / RDATE / RECURRENCE-ID overrides in the canonical order.
    """
    # Step 1: subtract EXDATE
    surviving = [i for i in rrule_instances if i not in exdates]

    # Step 2: add RDATE (sorted-merge to keep ordering)
    surviving = sorted(set(surviving) | rdates)

    # Step 3: apply per-instance overrides
    result = []
    for inst in surviving:
        ovr = overrides.get(inst)
        if ovr is None:
            result.append(InstanceRow.from_master(inst))
            continue
        if ovr.is_cancelled:
            continue                          # equivalent to a late-bound EXDATE
        result.append(InstanceRow.from_override(inst, ovr))
    return result
```

A common bug is to evaluate the overrides as `recurrence_id == dtstart_after_override`, which fails when an override moves the instance to a different time — the link to the original instance is lost. The override is keyed by the **original** `recurrence_id`, never the post-override `dtstart`.

The other common bug is to forget that `EXDATE` matches against the rule-produced instant, not the override-produced one. If a user moves the December 25 instance to December 26 (override) and then cancels it, the cancellation `EXDATE` is `20261225` (the original recurrence-id), not `20261226`.

## Single-Instance Overrides — RECURRENCE-ID

`RECURRENCE-ID` is RFC 5545's mechanism for "this one instance is different." The override row carries:

- `recurrence_id` — the instant the rule would have produced (the "anchor" to the original instance).
- Replacement values for any property: `dtstart`, `dtend`, `summary`, `location`, `attendees`, etc.

The `recurrence_id` is *immutable* and identifies the instance forever, even if the override moves the actual time. Editors must never recompute `recurrence_id` from the override's new `dtstart`.

**Storage:**

```sql
CREATE TABLE event_overrides (
  id              UUID PRIMARY KEY,
  event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  recurrence_id   TIMESTAMPTZ NOT NULL,            -- original rule-produced instant
  dtstart_utc     TIMESTAMPTZ,                     -- NULL = unchanged from master
  dtend_utc       TIMESTAMPTZ,
  summary         TEXT,
  location        TEXT,
  is_cancelled    BOOLEAN NOT NULL DEFAULT FALSE,
  override_data   JSONB,                            -- attendee changes, description, etc.
  UNIQUE (event_id, recurrence_id)
);
```

**Read path:** the materializer or expander does a left-join on `(event_id, recurrence_id)` to fold overrides in. A cancelled override (`is_cancelled = TRUE`) is equivalent to a late-bound `EXDATE` — the instance is suppressed from the output but the override record is preserved for audit.

**The "cancel one instance" UX flow:**

1. User clicks "delete this occurrence" on a recurring event.
2. App can either insert an `EXDATE` or insert an `event_overrides` row with `is_cancelled = TRUE`.
3. Most apps prefer `event_overrides` because it preserves the audit trail (who cancelled, when, with what reason).

**The "move one instance" UX flow:**

1. User drags a recurring event instance to a new time.
2. App inserts an `event_overrides` row with `recurrence_id = original_instant` and `dtstart_utc = new_time`.
3. Future expansions of the master RRULE will produce the original instant; the materializer joins it with the override and emits the moved time.

## Detached Instances — "This and Future" Edits

`THIS_AND_FOLLOWING` is the trickiest update mode. The user opens a weekly meeting, picks "this and all future occurrences," and changes the time from 9 AM to 9:30 AM. The data-model operation is a **series split**:

1. Set `UNTIL` on the original master to just before the edit's `recurrence_id`. The original series is now bounded.
2. Create a new master event with the modified properties, `DTSTART = recurrence_id`, and the same `RRULE` body (or a new one if the user changed the recurrence).
3. Link the new master to the old via `master_event_id` for forensic chaining.
4. Copy forward any future overrides whose `recurrence_id >= split_point` from the old master to the new one (or move them, depending on calendar UX semantics).
5. Materializer: invalidate `[split_point, +13 months]` for the old master; materialize the new master in `[split_point, +13 months]`.

**SQL sketch:**

```sql
BEGIN;

-- Step 1: bound the original series
UPDATE events
SET rrule = regexp_replace(rrule, 'UNTIL=[^;]+;?', '')        -- strip existing UNTIL
         || ';UNTIL=' || to_iso8601(:split_point - INTERVAL '1 second')
WHERE id = :original_event_id;

-- Step 2: create the new series
INSERT INTO events (id, user_id, rrule, dtstart, dtend, schedule_tz, master_event_id)
VALUES (
  :new_event_id,
  :user_id,
  :new_rrule,
  :new_dtstart,
  :new_dtend,
  :schedule_tz,
  :original_event_id
);

-- Step 3: copy forward overrides past the split point
INSERT INTO event_overrides (id, event_id, recurrence_id, dtstart_utc, dtend_utc, override_data, is_cancelled)
SELECT gen_random_uuid(), :new_event_id, recurrence_id, dtstart_utc, dtend_utc, override_data, is_cancelled
FROM event_overrides
WHERE event_id = :original_event_id
  AND recurrence_id >= :split_point;

DELETE FROM event_overrides
WHERE event_id = :original_event_id
  AND recurrence_id >= :split_point;

COMMIT;
```

**Why the split, not just an in-place edit?** Because the user explicitly asked for two semantically different intervals: "before the edit" and "after the edit." Storing them as one row would lose the property change at the split point, which matters for:

- Audit trails ("when did the meeting move from 9 AM to 9:30?").
- ICS export fidelity (other clients see a series that ended and a new series that started).
- Free-busy correctness (the busy-bitmap before the split reflects 9 AM; after, 9:30 AM).

The pattern lives in every production calendar: Google Calendar, Outlook, Apple Calendar all do this split. The `master_event_id` link is the chain that lets you reconstruct "this is the third generation of the engineering all-hands."

## Storage Schema — Master Plus Materialized View

The full schema collected:

```sql
-- Canonical events (recurring or single)
CREATE TABLE events (
  id              UUID PRIMARY KEY,
  user_id         UUID NOT NULL,
  master_event_id UUID REFERENCES events(id),       -- non-null after THIS_AND_FOLLOWING split
  title           TEXT NOT NULL,
  location        TEXT,
  description     TEXT,
  rrule           TEXT,                              -- NULL for single-instance events
  dtstart         TIMESTAMP NOT NULL,                -- local time at series start
  dtend           TIMESTAMP NOT NULL,
  schedule_tz     TEXT NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  recurring       BOOLEAN GENERATED ALWAYS AS (rrule IS NOT NULL) STORED
);

CREATE INDEX idx_events_user           ON events(user_id);
CREATE INDEX idx_events_recurring      ON events(user_id) WHERE recurring;
CREATE INDEX idx_events_master_chain   ON events(master_event_id) WHERE master_event_id IS NOT NULL;

-- Excluded dates (cancelled instances of an RRULE)
CREATE TABLE event_exdates (
  event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  exdate          TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (event_id, exdate)
);

-- Added dates (one-off instances grafted onto an RRULE)
CREATE TABLE event_rdates (
  event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  rdate           TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (event_id, rdate)
);

-- Per-instance overrides
CREATE TABLE event_overrides (
  id              UUID PRIMARY KEY,
  event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  recurrence_id   TIMESTAMPTZ NOT NULL,
  dtstart_utc     TIMESTAMPTZ,
  dtend_utc       TIMESTAMPTZ,
  override_data   JSONB,
  is_cancelled    BOOLEAN NOT NULL DEFAULT FALSE,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (event_id, recurrence_id)
);

-- Materialized read-side view
CREATE TABLE user_calendar_view (
  user_id         UUID NOT NULL,
  yyyy_mm         CHAR(7) NOT NULL,
  event_id        UUID NOT NULL,
  recurrence_id   TIMESTAMPTZ,
  dtstart_utc     TIMESTAMPTZ NOT NULL,
  dtend_utc       TIMESTAMPTZ NOT NULL,
  override_id     UUID,
  PRIMARY KEY (user_id, yyyy_mm, dtstart_utc, event_id)
) PARTITION BY LIST (yyyy_mm);
```

The split between the canonical store (small, normalized, source of truth) and the materialized view (denormalized, eventually consistent, read-optimized) is the same pattern as CQRS in general: writes target the canonical model, reads target the projection. The materializer is the projection-update process.

## Indexing for Range Queries

Calendar queries are almost always *range queries* — "events overlapping this window." The classical SQL pattern for interval overlap is:

```sql
WHERE dtstart_utc < :window_end
  AND dtend_utc   > :window_start
```

A B-tree on `(user_id, dtstart_utc, dtend_utc)` covers most read patterns:

- Per-user listing: `WHERE user_id = ?` uses the leading column.
- Range filter: `WHERE user_id = ? AND dtstart_utc < ?` uses the first two columns.
- The `dtend_utc > ?` filter is applied at index-scan time without a second seek.

For the materializer scan that drives precomputation, a partial index on `recurring = TRUE`:

```sql
CREATE INDEX idx_events_recurring_user
  ON events(user_id, dtstart)
  WHERE recurring;
```

The materializer iterates this index once per user per refresh, expanding only the recurring series — non-recurring events are inserted directly into `user_calendar_view` without RRULE processing.

For free-busy specifically, see `./free-busy-queries.md`. The approach there is a per-user busy bitmap rather than a row-per-instance scan; the bitmap is built from `user_calendar_view` via a Kafka projector.

For very large calendars (10K+ events per user), partition `user_calendar_view` by `yyyy_mm` so that month queries hit a single partition. PostgreSQL declarative partitioning works well; one partition per month for the active window, with old partitions archived after a year.

## Open-Ended Series — COUNT and UNTIL

RFC 5545 allows three termination forms:

1. **`COUNT=N`** — exactly N occurrences. Mathematically bounded.
2. **`UNTIL=<UTC instant>`** — stops at or before the given UTC instant. Mathematically bounded.
3. **Neither** — open-ended. Mathematically unbounded.

Open-ended series are common in real life: "the team standup, every weekday, indefinitely." Production calendars handle them by:

1. **Capping the materialization window.** Never expand beyond `[now - 13 months, now + 13 months]`. The materializer sees an unbounded RRULE and stops at the window edge.
2. **Recurring the materialization daily.** The window slides forward; new instances enter the materialized view as old ones age out.
3. **Falling through for far-future queries.** If a user navigates to "April 2030," the materialized view has no rows; the read path falls through to on-demand RRULE expansion for that single read.

The dangerous variant is `UNTIL=99991231T235959Z` — syntactically bounded but practically open-ended. Treat it as if it had no `UNTIL`. Same window-cap policy.

The other dangerous variant is a `COUNT` so large it might as well be open-ended: `COUNT=10000` for a daily series is 27+ years of instances. Same policy: cap at the window edge.

The policy lives in the materializer:

```python
HORIZON_MONTHS_FORWARD = 13
HORIZON_MONTHS_BACK = 13

def materialize_event(event):
    horizon_start = now() - relativedelta(months=HORIZON_MONTHS_BACK)
    horizon_end   = now() + relativedelta(months=HORIZON_MONTHS_FORWARD)

    rule = rrulestr(event.rrule, dtstart=event.dtstart_local, tzinfos=...)

    # The library's `between(after, before, inc=True)` bounds the iterator.
    # Without this, a FREQ=DAILY with no UNTIL never returns.
    instances = rule.between(
        max(horizon_start, event.dtstart_utc),
        horizon_end,
        inc=True,
    )
    # ... apply EXDATE/RDATE/overrides, write to user_calendar_view
```

`dateutil.rrule.between` is bounded — pass it a date range and it returns finitely. Without explicit bounds (`rule.list()` or naive iteration), an open-ended rule never terminates.

## Library Choice — Don't Roll Your Own

RRULE has corner cases that bite handwritten implementations. Use a vetted library and pin the version.

**Python:** [`python-dateutil`](https://dateutil.readthedocs.io/en/stable/rrule.html) is the standard. The `dateutil.rrule` module covers all of RFC 5545 and exposes `rrulestr` for parsing iCal-format strings. Battle-tested, widely-deployed.

```python
from datetime import datetime
from dateutil.rrule import rrulestr
from dateutil.tz import gettz

# Weekly meeting Mon/Wed/Fri at 9 AM America/New_York, until end of 2027
rule_text = "DTSTART;TZID=America/New_York:20260406T090000\n" \
            "RRULE:FREQ=WEEKLY;BYDAY=MO,WE,FR;UNTIL=20271231T235959Z"

rule = rrulestr(rule_text)

# Get all instances in April 2026
window_start = datetime(2026, 4, 1, tzinfo=gettz("America/New_York"))
window_end   = datetime(2026, 5, 1, tzinfo=gettz("America/New_York"))

for inst in rule.between(window_start, window_end, inc=True):
    print(inst.isoformat())
# 2026-04-06T09:00:00-04:00
# 2026-04-08T09:00:00-04:00
# 2026-04-10T09:00:00-04:00
# ... (13 instances total)
```

**Java:** [`ical4j`](https://github.com/ical4j/ical4j) for full iCalendar handling, or `java.time.temporal` plus the built-in JSR-310 types for simpler cases. `ical4j` includes `Recur` for RRULE.

**JavaScript / TypeScript:** [`rrule` (`node-rrule`)](https://github.com/jakubroztocil/rrule). API mirrors `python-dateutil` closely. Used in major browser-side calendar apps.

**C / C++:** [`libical`](https://github.com/libical/libical) — the reference implementation. Underpins macOS Calendar, Mozilla Thunderbird, and many others.

**Don't roll your own.** The frequent footguns:

- `BYSETPOS` operates on the BYxxx-filtered set within one FREQ period. Naive implementations apply it to the whole series.
- Week numbering at year boundaries is `WKST`-dependent and ISO-8601-defined. Naive implementations pick Sunday or Monday inconsistently.
- `BYMONTHDAY=-1` (last day of month) interacts with month-length variability. Naive implementations get February wrong.
- Leap-year handling for `BYYEARDAY=366` and `BYMONTHDAY=29;BYMONTH=2` requires the rule library to *skip non-leap years* rather than emit a non-existent date.
- Time-zone interaction: the RRULE expansion happens in the event's local time, but `UNTIL` is specified in UTC. The library handles the mixing; handwritten code conflates them.

Pin the library version. RRULE library upgrades have, occasionally, fixed bugs in ways that change instance lists for existing rules. Run a regression test that re-expands a representative set of RRULEs after every upgrade and diffs against the previous library version.

## Performance Numbers

Concrete numbers worth keeping in mind:

| Series | Window | Instances |
|--------|--------|-----------|
| `FREQ=DAILY` for 10 years | 10 years | 3,650 |
| `FREQ=DAILY` for 10 years | 1 month | 30 |
| `FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR` for 5 years | 1 month | ~22 |
| `FREQ=MONTHLY;BYDAY=1MO` for 10 years | 1 month | 1 |
| `FREQ=YEARLY;BYMONTH=11;BYDAY=4TH` (Thanksgiving) | 1 month | 0 or 1 |

Storage costs in the materialized view, assuming a row is ~200 bytes:

| Profile | Rows | Storage |
|---------|------|---------|
| User with 50 daily series, 1-year window | 50 × 365 = 18,250 | 3.6 MB |
| User with 50 daily series, 26-month window | 50 × 791 = 39,550 | 7.9 MB |
| 10,000 users with 50 daily series each, 26-month window | 395 M rows | 79 GB |

A 79 GB materialized view is small by data-warehouse standards but real by OLTP standards. Partitioning by `yyyy_mm` keeps individual partitions small; old partitions can be archived to cheaper storage after they age out of the window.

CPU costs for on-demand expansion with `python-dateutil`:

- Single-rule expansion of a 1-month window: ~50 µs.
- Single-rule expansion of a 26-month window: ~1 ms.
- 50 rules expanded for a 1-month view: ~2.5 ms.
- 50 rules × 12 attendees expanded for a free-busy probe: ~30 ms (linear in attendees × rules).

The materialized view amortizes this CPU cost: pay it once at materialization time, recover it across thousands of subsequent reads.

## Worked Example — Weekly Engineering Sync

A complete trace of a recurring event from creation through expansion through one-off cancellation through "this and future" edit.

**Initial creation.**

Senior engineer creates "Engineering Sync" — every Monday and Wednesday at 14:00 America/New_York, starting 2026-04-06, no end date.

```sql
INSERT INTO events (id, user_id, title, rrule, dtstart, dtend, schedule_tz)
VALUES (
  'evt-eng-sync',
  'user-eng-lead',
  'Engineering Sync',
  'FREQ=WEEKLY;BYDAY=MO,WE',
  '2026-04-06T14:00:00',
  '2026-04-06T15:00:00',
  'America/New_York'
);
```

The materializer enqueues partition jobs for `('user-eng-lead', '2026-04')` through `('user-eng-lead', '2027-05')` (13 months forward). Each partition job expands the RRULE into its window:

```text
2026-04 partition for evt-eng-sync:
  2026-04-06T14:00:00-04:00 → UTC 2026-04-06T18:00:00Z
  2026-04-08T14:00:00-04:00 → UTC 2026-04-08T18:00:00Z
  2026-04-13T14:00:00-04:00 → UTC 2026-04-13T18:00:00Z
  2026-04-15T14:00:00-04:00 → UTC 2026-04-15T18:00:00Z
  2026-04-20T14:00:00-04:00 → UTC 2026-04-20T18:00:00Z
  2026-04-22T14:00:00-04:00 → UTC 2026-04-22T18:00:00Z
  2026-04-27T14:00:00-04:00 → UTC 2026-04-27T18:00:00Z
  2026-04-29T14:00:00-04:00 → UTC 2026-04-29T18:00:00Z
  (8 instances)
```

**Quarter view (April-June 2026):**

```sql
SELECT v.dtstart_utc, v.recurrence_id, e.title, o.override_data
FROM user_calendar_view v
JOIN events e ON e.id = v.event_id
LEFT JOIN event_overrides o
       ON o.event_id = v.event_id
      AND o.recurrence_id = v.recurrence_id
WHERE v.user_id = 'user-eng-lead'
  AND v.yyyy_mm IN ('2026-04', '2026-05', '2026-06')
  AND v.event_id = 'evt-eng-sync'
ORDER BY v.dtstart_utc;

-- 26 rows (8 + 9 + 9), no overrides, all clean
```

**One-off cancellation (Memorial Day, 2026-05-25).**

User cancels Memorial Day 2026 (a Monday). UI calls "delete this occurrence."

```sql
INSERT INTO event_exdates (event_id, exdate)
VALUES ('evt-eng-sync', '2026-05-25T18:00:00Z');
```

Materializer invalidates `('user-eng-lead', '2026-05')` and re-runs:

```text
2026-05 partition (after EXDATE):
  2026-05-04T18:00:00Z (Mon)
  2026-05-06T18:00:00Z (Wed)
  2026-05-11T18:00:00Z (Mon)
  2026-05-13T18:00:00Z (Wed)
  2026-05-18T18:00:00Z (Mon)
  2026-05-20T18:00:00Z (Wed)
  -- 2026-05-25 SKIPPED (EXDATE)
  2026-05-27T18:00:00Z (Wed)
  (7 instances, was 9)
```

**One-instance reschedule (June 3, 2026 moved from 14:00 to 15:00).**

A specific Wednesday gets bumped because of an offsite. UI calls "edit this occurrence."

```sql
INSERT INTO event_overrides (id, event_id, recurrence_id, dtstart_utc, dtend_utc)
VALUES (
  'ovr-001',
  'evt-eng-sync',
  '2026-06-03T18:00:00Z',                  -- the original rule-produced instant
  '2026-06-03T19:00:00Z',                  -- moved to 15:00 NY = 19:00Z
  '2026-06-03T20:00:00Z'
);
```

Materializer invalidates `('user-eng-lead', '2026-06')`. The expansion produces the original recurrence_id (18:00Z); the join with `event_overrides` substitutes the new dtstart/dtend. The output row:

```text
recurrence_id  = 2026-06-03T18:00:00Z   (anchor — never changes)
dtstart_utc    = 2026-06-03T19:00:00Z   (moved time)
dtend_utc      = 2026-06-03T20:00:00Z
override_id    = ovr-001
```

**"This and future" edit (starting 2026-09-07, switch from MO/WE to TU/TH).**

The team decides Tuesday/Thursday works better. User picks "this and all future occurrences" and changes days.

```sql
BEGIN;

-- Bound the original series at 2026-09-06 (Sunday before)
UPDATE events
SET rrule = 'FREQ=WEEKLY;BYDAY=MO,WE;UNTIL=20260906T235959Z'
WHERE id = 'evt-eng-sync';

-- Create the new series
INSERT INTO events (id, user_id, master_event_id, title, rrule, dtstart, dtend, schedule_tz)
VALUES (
  'evt-eng-sync-v2',
  'user-eng-lead',
  'evt-eng-sync',                          -- chain to original
  'Engineering Sync',
  'FREQ=WEEKLY;BYDAY=TU,TH',
  '2026-09-08T14:00:00',                   -- first Tuesday after split
  '2026-09-08T15:00:00',
  'America/New_York'
);

-- Copy forward overrides past split (none in this case; the June 3 override is before the split)

COMMIT;
```

Materializer invalidates and recomputes from September 2026 forward, both for the bounded original (now empty after September 6) and the new series.

The audit trail: anyone querying `evt-eng-sync-v2.master_event_id` finds `evt-eng-sync`, traces back, and sees the rule-and-day change at 2026-09-07.

## Anti-Patterns

1. **Storing "every day" as 365 separate event rows.** A daily standup expanded eagerly into 365 rows for the year (or 1,825 for five years) bloats storage, makes edits O(N), and breaks the moment the user wants to extend the series. Always store the master event with an RRULE blob and let the materializer produce instances within a window.

2. **Expanding the full series at query time.** "On every page load, evaluate every recurring event's RRULE from `DTSTART` forward" forces the iterator to run from the series origin (potentially years ago) for every read. Pin the iterator to the query window with `between(after, before)` or equivalent.

3. **No upper bound on series length.** `RRULE:FREQ=DAILY` with no `COUNT` and no `UNTIL` is unbounded. A materializer that doesn't cap at a horizon (13 months forward is the standard) loops until OOM.

4. **Treating `UNTIL=99991231T235959Z` as different from open-ended.** It isn't, in any practical sense. Apply the same window-cap policy.

5. **Missing `EXDATE` during expansion.** The cancelled instance shows up in the materialized view because the expander generated it from the rule and didn't subtract the exception list. Symptom: "I cancelled Christmas but it's still on my calendar." Fix: always apply `EXDATE` after generation.

6. **Keying overrides by current `dtstart` instead of `recurrence_id`.** When a user moves an instance, the override's `dtstart_utc` is the new time but the *anchor* to the original is `recurrence_id` (the rule-produced instant). Keying by the new `dtstart` loses the link and double-counts the instance (once at the original time from the rule, once at the new time from the override).

7. **Recomputing `recurrence_id` from the override's modified `dtstart`.** `recurrence_id` is immutable. It identifies *which* instance was overridden, even if the override moves the time. Editors that recompute it break the join with the master series.

8. **In-place edit instead of series split for "this and future".** Mutating the master series for a "this and future" edit loses the history of the property change. Future ICS exports and external sync miss the split point. Always do the bounded-original-plus-new-master split.

9. **Synchronous materialization on edit.** A user editing a 5-year daily series triggers materialization of ~1,800 partitions; doing this in the request thread blocks the UI for seconds. Always enqueue materializer jobs and return success after writing the canonical row.

10. **No materialized-view partitioning.** A single `user_calendar_view` table for all users grows without bound and has worse range-scan locality. Partition by `yyyy_mm` so month queries hit one partition; archive old partitions after a year.

11. **Forgetting that the RRULE is in the event's local time but `UNTIL` is in UTC.** Mixing the two produces off-by-one errors at DST boundaries. Most libraries handle this; handwritten code does not.

12. **Rolling your own RRULE expander.** `BYSETPOS` semantics, `WKST`-dependent week numbering, leap-year `BYMONTHDAY=29;BYMONTH=2` interactions, negative `BYMONTHDAY` for "last day of month" — the corner cases bite. Use `python-dateutil`, `ical4j`, `rrule.js`, or `libical`. Pin the version.

13. **Pinning to a buggy library version forever.** RRULE library upgrades have, occasionally, fixed bugs in ways that change instance lists. Run a regression suite that re-expands a representative set of rules and diffs against the previous version before any library upgrade.

14. **Materializing across users with no per-user partition.** A company-wide all-hands materialized once globally seems efficient, but free-busy queries are per-user and benefit from per-user locality. Materialize per-user even for shared events; the duplication is small relative to the read-path win.

15. **Treating the materialized view as the source of truth.** The canonical store is `events` + `event_exdates` + `event_overrides`. The materialized view is a projection that can be rebuilt from canonical. If you find yourself updating `user_calendar_view` directly without going through canonical, the projection has rotted into a parallel write path; fix it before the divergence becomes load-bearing.

## Related

- [`./time-zone-correctness.md`](./time-zone-correctness.md) — sibling deep-dive on tz-aware event semantics, IANA tzdata updates, and DST policies. Closely interacts with RRULE expansion: the rule iterates in the event's local time and must apply DST policies for skipped or ambiguous instances.
- [`./free-busy-queries.md`](./free-busy-queries.md) — sibling deep-dive on the busy-bitmap projector that consumes from `user_calendar_view`. Free-busy correctness depends on the materializer being current.
- [`./conflict-detection.md`](./conflict-detection.md) — sibling deep-dive on overlap detection across calendars; built on top of free-busy and therefore on top of the materialized view.
- [`../design-calendar-system.md`](../design-calendar-system.md) — parent case study; this doc expands §1.
- [`../../async/job-scheduler/time-zone-correctness.md`](../../async/job-scheduler/time-zone-correctness.md) — analogous deep-dive for the cron-style scheduler problem; shares the IANA-name / DST-policy / run-history schema patterns. Many of the time-handling lessons translate directly.
- [`../../../data-consistency/time-and-ordering.md`](../../../data-consistency/time-and-ordering.md) — foundation; the general treatment of POSIX time, civil time, ordering, and clocks across distributed systems. Required reading for the distinctions used here.

## References

- IETF — [*RFC 5545 — Internet Calendaring and Scheduling Core Object Specification (iCalendar)*](https://datatracker.ietf.org/doc/html/rfc5545). The canonical specification. §3.3.10 covers RRULE; §3.8.5 covers EXDATE/RDATE; §3.8.4.4 covers RECURRENCE-ID. Required reading.
- IETF — [*RFC 7529 — Non-Gregorian Recurrence Rules in iCalendar*](https://datatracker.ietf.org/doc/html/rfc7529). Extension that adds non-Gregorian calendar systems (Hijri, Hebrew, etc.) to RRULE via the `RSCALE` parameter. Relevant for international calendaring.
- python-dateutil — [*`dateutil.rrule`*](https://dateutil.readthedocs.io/en/stable/rrule.html). The reference Python implementation. `rrulestr` for parsing iCal strings; `rruleset` for combining RRULE / EXDATE / RDATE; `between(after, before)` for bounded iteration.
- ical4j — [*`org.mnode.ical4j`*](https://github.com/ical4j/ical4j). The reference Java implementation. Full iCalendar parsing including RRULE via `Recur`; integrates with `java.time` types in recent versions.
- node-rrule — [*`rrule` on GitHub*](https://github.com/jakubroztocil/rrule). The reference JavaScript/TypeScript implementation. Used by major browser-side calendar UIs. Mirrors the python-dateutil API.
- libical — [*`libical/libical`*](https://github.com/libical/libical). The reference C/C++ implementation. Underpins macOS Calendar, Thunderbird, and many embedded calendar clients.
- Google — [*Google Calendar API — Events resource*](https://developers.google.com/calendar/api/concepts/events-calendars). Production reference for how a major calendar models `recurrence`, `recurringEventId`, `originalStartTime`, and update modes (`sendUpdates`, `instances` endpoint).
- Apple — [*EKRecurrenceRule — EventKit*](https://developer.apple.com/documentation/eventkit/ekrecurrencerule). The Apple Calendar SDK abstraction. Lower-level than RFC 5545 in some ways (no direct EXDATE) but shows how a major OS-level calendar models recurrence.
- Microsoft — [*PatternedRecurrence resource (Microsoft Graph)*](https://learn.microsoft.com/en-us/graph/api/resources/patternedrecurrence). Outlook/Exchange's recurrence model. Notable for not being RFC 5545 directly — uses a different shape (`pattern` + `range`) that maps to RRULE concepts but with different field names. Useful contrast point.
- IANA — [*Time Zone Database*](https://www.iana.org/time-zones). Authoritative source for tz rules; required for correct expansion of events whose `schedule_tz` is an IANA name. Subscribe to `tz-announce@iana.org` for releases that may shift recurring instances.
