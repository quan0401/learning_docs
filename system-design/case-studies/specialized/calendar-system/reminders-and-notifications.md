---
title: "Reminders and Notifications — Delayed Queues, Per-Attendee Preferences, and Idempotent Delivery"
date: 2026-05-01
updated: 2026-05-01
tags: [system-design, deep-dive, calendar, reminders, notifications]
---

# Reminders and Notifications — Delayed Queues, Per-Attendee Preferences, and Idempotent Delivery

**Date:** 2026-05-01 | **Updated:** 2026-05-01
**Tags:** `system-design` `deep-dive` `calendar` `reminders` `notifications`

> **Parent case study:** [Design a Calendar System](../design-calendar-system.md). This deep-dive expands §6 "Reminders and Notifications".

## Table of Contents

- [Summary](#summary)
- [What a Reminder Actually Is](#what-a-reminder-actually-is)
- [Per-Attendee Semantics](#per-attendee-semantics)
- [Local-Time Display vs UTC Scheduling](#local-time-display-vs-utc-scheduling)
- [Scheduling Reminders at Event Create Time](#scheduling-reminders-at-event-create-time)
- [Storage: Materialize vs Compute on Fire](#storage-materialize-vs-compute-on-fire)
- [Delayed Queue Implementations](#delayed-queue-implementations)
- [Redis Sorted Set Pattern](#redis-sorted-set-pattern)
- [AWS EventBridge Scheduler Pattern](#aws-eventbridge-scheduler-pattern)
- [SQS Delay Queues, Cassandra Time Buckets, and Kafka Delay Brokers](#sqs-delay-queues-cassandra-time-buckets-and-kafka-delay-brokers)
- [Cross-DST Reminder Accuracy](#cross-dst-reminder-accuracy)
- [Cancellation on Event Edit or Delete](#cancellation-on-event-edit-or-delete)
- [Idempotent Delivery](#idempotent-delivery)
- [Snooze, Dismiss, and Multi-Device Fan-Out](#snooze-dismiss-and-multi-device-fan-out)
- [Quiet Hours, Do-Not-Disturb, and Channel Routing](#quiet-hours-do-not-disturb-and-channel-routing)
- [Recurring Events and Per-Occurrence Reminders](#recurring-events-and-per-occurrence-reminders)
- [Worked Example: 09:00 NYC Meeting, London Attendee](#worked-example-0900-nyc-meeting-london-attendee)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

The parent case study compresses reminders into one diagram and a few paragraphs: a delay queue or hierarchical timing wheel, a dispatcher, a Bloom filter for dedup. That summary is correct and elides the engineering required to make reminders feel right. A reminder is a **per-attendee, per-occurrence promise** with a contract that pins to UTC for scheduling but to local wall-clock time for *meaning*. The hard parts are: scheduling fan-out at create-time across N attendees × M reminder rules × K materialized occurrences; cancellation that must reach every queued message when an event is moved or deleted; idempotent delivery so a retry storm doesn't spam users; cross-DST reminders that must re-anchor to UTC after the zone definition changes; per-attendee preferences for channel (push, email, SMS), quiet hours, and DND that defer non-urgent reminders without dropping them; and multi-device fan-out where one tap on one phone dismisses the reminder across the user's other devices. This deep-dive walks the data model, four delayed-queue implementations (Redis ZADD, SQS DelaySeconds, Cassandra time-bucketed, AWS EventBridge Scheduler), the cancellation flow, the idempotency key, and the anti-patterns that turn "remind me 15 minutes before" into a 4 AM SMS for a meeting that was moved last Tuesday. The recurring theme: reminders are a fan-out write proportional to `attendees × reminder_rules × occurrences_in_window`, and the schedule must be cancellable at every link in that fan-out without leaking firing messages back to users.

## What a Reminder Actually Is

A reminder is a tuple — *not* a property of an event. The minimal identity is:

```text
Reminder = (event_id, attendee_id, recurrence_id, method, offset)
```

Where:

- `event_id` is the canonical series ID (or one-shot event).
- `attendee_id` is the user the reminder is for. Every attendee has their own reminder set; a reminder is not "for the event" but "for an attendee at the event."
- `recurrence_id` is the original local start of the specific occurrence (NULL for non-recurring events). RFC 5545 calls this `RECURRENCE-ID`. It pins the reminder to one occurrence; `THIS_AND_FOLLOWING` overrides each carry their own.
- `method` is the channel: `popup` (in-app push), `email`, `sms`, `desktop`, etc.
- `offset` is the time before the event start (e.g., `15M`, `1H`, `1D`). Day-of reminders may be expressed as offsets from local midnight rather than the event start.

The fire time is derived: `fire_at_utc = event_start_utc - offset`. The pair `(event_id, attendee_id, recurrence_id, fire_at_utc)` is the natural idempotency key — see [Idempotent Delivery](#idempotent-delivery) below.

RFC 5545 models this as a `VALARM` block nested inside `VEVENT`:

```text
BEGIN:VEVENT
UID:abc123@example.com
DTSTART;TZID=America/New_York:20260501T090000
DTEND;TZID=America/New_York:20260501T100000
SUMMARY:Project sync
BEGIN:VALARM
ACTION:DISPLAY
DESCRIPTION:Project sync starts in 15 minutes
TRIGGER:-PT15M
END:VALARM
BEGIN:VALARM
ACTION:EMAIL
SUMMARY:Project sync starts in 1 hour
ATTENDEE:mailto:bob@example.com
TRIGGER:-PT1H
END:VALARM
END:VEVENT
```

`TRIGGER:-PT15M` is "fire 15 minutes before DTSTART"; `TRIGGER:20260501T080000Z` is an absolute UTC trigger. Round-tripping VALARM through ICS export/import requires that the data model preserve method, offset, and any per-VALARM attendee scope.

## Per-Attendee Semantics

The single most common mistake when designing this surface is treating reminders as a property of the event. They are a property of the **(event, attendee)** pair. The implications:

- **Alice and Bob attend the same meeting.** Alice has "popup 15 min before" set as her default; Bob has "email 1 day before, popup 10 min before." Each attendee owns their reminder rules.
- **The organizer does not control attendee reminders.** When you create a meeting and add Bob as attendee, Bob's *default reminders* (configured per-calendar in his account) attach automatically. The organizer's "10 min before" reminder is the organizer's, not Bob's.
- **VALARM in iTIP messages is advisory.** When an `iTIP REQUEST` arrives over email (RFC 5546), the embedded VALARM is the *organizer's* alarm preference. Most clients (Apple Calendar, Outlook, Google Calendar) drop or ignore them and apply the receiver's default reminders instead, precisely because each attendee's preferences should win.
- **Per-attendee preferences live elsewhere.** Channels available, quiet hours, DND state, device tokens — all attached to the attendee, not the event.

The data model in the parent doc encodes this with a separate `reminders` table keyed by `(event_id, attendee_email, method, minutes_before)`. Attendee defaults can either be denormalized into that table at attach time or computed at fire time from the attendee's calendar default.

## Local-Time Display vs UTC Scheduling

A reminder fires at a UTC instant. The text it displays — "Project sync in 15 minutes" — is rendered for the attendee's *current* zone, which may differ from the event's organizer zone, which may differ from the attendee's home zone if they're traveling.

Three rules:

1. **Schedule in UTC.** The delay queue stores `fire_at_utc` because UTC is monotonic and zone-free. A queue keyed on local time would be incorrect across DST transitions and pointless across attendee zones.
2. **Display in local.** The notification template renders the event time in the attendee's *current* zone (from the device or the user's preference). "Meeting at 9 AM" is meaningless without a zone; "Meeting at 9 AM EDT (in 15 minutes)" is precise.
3. **Day-of reminders are special.** A "day-of" reminder ("today at 9 AM") fires at local midnight, not at `start_utc - 1d`. For a 09:00 New York event, "1 day before" *as offset* fires at 09:00 the previous day in the attendee's zone — but "morning of" fires at, say, 08:00 attendee-local that morning. Storing the rule as an offset is insufficient; store the *anchor* (`midnight_local` vs `event_start`) and the offset separately.

This is the same intent-vs-instant split that recurrence handles in [`./time-zone-correctness.md`](./time-zone-correctness.md): the user's intent is "morning of, in *my* zone"; the scheduler converts intent to instant at the latest safe moment.

## Scheduling Reminders at Event Create Time

The fan-out shape on event creation:

```text
on_event_create(event):
  occurrences = expand_recurrence(event, window=[now, now + 13 months])
  for each attendee in event.attendees:
      reminders = attendee.default_reminders ∪ event.organizer_reminders_for_attendee
      for each occurrence in occurrences:
          for each reminder in reminders:
              fire_at_utc = occurrence.start_utc - reminder.offset
              if fire_at_utc < now: continue
              enqueue_delayed(
                  key      = (event.id, attendee.id, occurrence.recurrence_id,
                              reminder.method, reminder.offset),
                  fire_at  = fire_at_utc,
                  payload  = compact_payload(event, attendee, occurrence, reminder),
              )
```

The fan-out is `attendees × occurrences_in_window × reminders_per_attendee`. Concrete numbers:

- 5 attendees, daily for a year, 2 reminders each → 5 × 365 × 2 = 3,650 queue entries.
- 50 attendees, weekly for 5 years, 3 reminders each → 50 × 260 × 3 = 39,000 entries.
- 500-person all-hands, weekly, 1 reminder each → 500 × 52 = 26,000 entries per year, recomputed on every series edit.

This is why the materialized window matters. Computing reminders for every occurrence forever is unbounded; computing them for the next 13 months is finite. A reminder roller (a daily job) extends the window forward as time passes:

```text
on_daily_window_extension():
  for each active recurring event:
      next_window = [last_materialized + 1d, last_materialized + 13 months + 1d]
      new_occurrences = expand_recurrence(event, window=next_window)
      enqueue reminders for each (attendee, occurrence, rule)
```

This pattern shows up in every case study with rolling materialized state — see [`./rrule-expansion.md`](./rrule-expansion.md) for the canonical version.

## Storage: Materialize vs Compute on Fire

Two storage strategies for reminders, with the same trade-off shape as RRULE expansion:

| Strategy | Storage | Fire latency | Edit cost | Notes |
|----------|---------|--------------|-----------|-------|
| **Materialize all upfront** | One row per (attendee, occurrence, rule) in the delay queue | O(1) — just pop | O(N) on series edit | The default for production. Fan-out is bounded by the materialization window. |
| **Compute on fire from rule** | One row per `(event_id, attendee_id, rule)` + a tickler that wakes every minute | O(rule eval) per fire | O(1) on edit | Simple; doesn't scale beyond a single-instance scheduler; tickler is the bottleneck. |
| **Hybrid** | Rules stored canonically + a rolling-window materializer + a fire-time re-validator | O(1) cache hit, O(rule eval) cold | O(K_window) on edit | Production answer for high-traffic systems. |

The hybrid is the right answer at scale because it lets the delay queue stay finite while still allowing edits to be cheap (you only recompute the affected window) and gives you a single late-bound check at fire time to catch drift (event moved, attendee removed, RSVP declined).

A second axis: where does the schedule itself live?

```text
events                  -- canonical event
  + rrule, dtstart, ...

reminder_rules          -- per (attendee, event) configuration
  (event_id, attendee_id, method, offset, anchor)
  PRIMARY KEY (event_id, attendee_id, method, offset, anchor)

reminder_queue          -- materialized fire schedule (the delay queue)
  (event_id, attendee_id, recurrence_id, method, offset, fire_at_utc, dedup_key)
  -- stored in Redis ZSET / SQS / EventBridge / Cassandra-time-bucket / etc.
```

`reminder_rules` is the source of truth. `reminder_queue` is a derived materialization. On an event edit, the rules survive; the queue is invalidated and recomputed for the affected window.

## Delayed Queue Implementations

A delayed queue is the abstraction every reminder system needs. The interface is small:

```text
enqueue(message, fire_at_utc, dedup_key)   // schedule a delivery
cancel(dedup_key)                          // un-schedule
poll(now)                                  // return all messages whose fire_at <= now
```

Five implementations are in widespread use; each has a different cost shape and operational profile.

| Implementation | Best for | Granularity | Cancellation | Operational complexity | Caveats |
|---|---|---|---|---|---|
| **Redis sorted set** (`ZADD score=fire_at`) | Sub-second precision, in-house deployment | ~1 ms | O(log N) by member key | Medium — you operate Redis | Memory bound; needs sharding past ~100M entries |
| **AWS SQS Delay Queues** | Bounded delays up to 15 minutes | 1 s | None (immutable once enqueued) | Low — fully managed | 15-minute max delay disqualifies most calendar reminders |
| **AWS EventBridge Scheduler** | Up to 1-year delays, point-in-time triggers | 1 minute | Yes (delete schedule) | Low — fully managed | 1-min minimum granularity; per-account quota |
| **Cassandra time-bucketed table** | Petabyte-scale, billions of pending reminders | 1 minute (bucket size) | Tombstone the row | High — schema and bucket sizing matter | Needs a poller per bucket; tombstone hygiene |
| **Kafka with delay-broker** | High throughput, ordered delivery within partition | Sub-second | Compact via tombstone or skip on consume | Medium — operate Kafka + delay topics | Consumer must skip pre-due messages or block |

Production calendars typically run a hierarchical timing wheel built on **Redis sorted sets** for the next ~24 hours (sub-second precision, fast cancellation) and **EventBridge Scheduler / Cassandra** for the long tail (everything beyond the wheel's horizon). A daily roller pulls long-tail entries into the wheel as their fire time approaches.

## Redis Sorted Set Pattern

The canonical implementation. Each entry is a member of a ZSET keyed by `fire_at_utc` as the score. A poller reads the lowest-score members below `now()` and dispatches them.

```python
import redis
import json
import time

r = redis.Redis()

# Enqueue a reminder fire
def enqueue_reminder(event_id, attendee_id, recurrence_id, method, offset_sec, fire_at_utc, payload):
    dedup_key = f"{event_id}:{attendee_id}:{recurrence_id}:{method}:{offset_sec}"
    member = json.dumps({
        "dedup_key": dedup_key,
        "event_id": event_id,
        "attendee_id": attendee_id,
        "recurrence_id": recurrence_id,
        "method": method,
        "payload": payload,
    })
    # Score = fire_at as Unix epoch seconds
    r.zadd("reminders:wheel", {member: fire_at_utc})
    # Index for cancellation
    r.hset("reminders:by_key", dedup_key, member)

# Cancel a reminder by key
def cancel_reminder(dedup_key):
    member = r.hget("reminders:by_key", dedup_key)
    if member:
        r.zrem("reminders:wheel", member)
        r.hdel("reminders:by_key", dedup_key)

# Poller — runs every second on N parallel workers, sharded by hash(member)
def poll_due_reminders():
    now = int(time.time())
    # ZRANGEBYSCORE returns members with score in [-inf, now]
    due = r.zrangebyscore("reminders:wheel", min="-inf", max=now, start=0, num=1000)
    for member in due:
        # Atomic claim — only one poller wins this member
        if r.zrem("reminders:wheel", member) > 0:
            entry = json.loads(member)
            r.hdel("reminders:by_key", entry["dedup_key"])
            dispatch_to_channel(entry)
```

Key operational notes:

- **Atomic claim via `ZREM`.** Multiple poller workers running concurrently must not double-dispatch. `ZREM` returning non-zero is the proof of ownership; if it returns zero, another worker already claimed it. This is the same pattern as `LMPOP` for FIFO queues.
- **Sharding past memory limits.** A single Redis node holds ~100M entries comfortably. Beyond that, shard by `hash(attendee_id) mod N` into N ZSETs. Each poller owns one shard.
- **Two indices.** The ZSET indexes by fire time; a separate hash indexes by dedup key. Cancellation is O(log N) on the ZSET via `ZREM` after a hash lookup.
- **Loss-of-Redis recovery.** Redis ZSET state is the queue. AOF + replication is mandatory. On a cold start, replay from `reminder_rules` for the affected window — the rules are the source of truth.
- **Lua script for batch dispatch.** The poll-then-claim loop above is two Redis round-trips per member. A Lua script that atomically pops up to N due members in one round-trip (`ZRANGEBYSCORE` + `ZREMRANGEBYSCORE` in a single script) cuts dispatch latency at scale.

## AWS EventBridge Scheduler Pattern

For long delays (days to a year out) where running Redis is overkill, AWS EventBridge Scheduler is a managed alternative. Each reminder is a one-shot schedule with a UTC `at()` expression:

```python
import boto3

scheduler = boto3.client("scheduler")

def schedule_reminder(event_id, attendee_id, recurrence_id, method, fire_at_utc, payload):
    # Schedule names must be unique per account; use the dedup key
    name = f"reminder-{event_id}-{attendee_id}-{recurrence_id}-{method}"
    # at() expression in UTC, format: at(yyyy-mm-ddThh:mm:ss)
    fire_at_iso = fire_at_utc.strftime("%Y-%m-%dT%H:%M:%S")

    scheduler.create_schedule(
        Name=name,
        ScheduleExpression=f"at({fire_at_iso})",
        ScheduleExpressionTimezone="UTC",
        FlexibleTimeWindow={"Mode": "OFF"},  # exact firing
        Target={
            "Arn": "arn:aws:lambda:us-east-1:123456789012:function:reminder-dispatcher",
            "RoleArn": "arn:aws:iam::123456789012:role/SchedulerRole",
            "Input": json.dumps({
                "dedup_key": name,
                "event_id": event_id,
                "attendee_id": attendee_id,
                "recurrence_id": recurrence_id,
                "method": method,
                "payload": payload,
            }),
        },
        ActionAfterCompletion="DELETE",  # auto-cleanup after fire
    )

def cancel_scheduled_reminder(event_id, attendee_id, recurrence_id, method):
    name = f"reminder-{event_id}-{attendee_id}-{recurrence_id}-{method}"
    try:
        scheduler.delete_schedule(Name=name)
    except scheduler.exceptions.ResourceNotFoundException:
        pass  # already fired or never existed
```

Operational notes:

- **`FlexibleTimeWindow.Mode = "OFF"`** means fire at the exact instant; `"FLEXIBLE"` accepts a tolerance window in exchange for delivery-rate flattening during scheduler load spikes. Calendars want "OFF" for headline reminders and may accept "FLEXIBLE" for less critical ones.
- **`ActionAfterCompletion = "DELETE"`** auto-cleans after the schedule fires. Without this, the account accumulates stale schedule definitions and hits quota.
- **Per-account quotas.** Default is around 1M schedules per account; raise via service quotas. For a billion-user calendar, partition across many accounts or fall back to a self-hosted ZSET wheel.
- **1-minute minimum granularity.** Don't use EventBridge Scheduler for "fire now" or "fire in 30 seconds"; use SQS or the Redis wheel.
- **Cancellation is fast.** `delete_schedule` is O(1); idempotent on `ResourceNotFoundException`.

The hybrid pattern: every reminder beyond the wheel's horizon (say, 24 hours) gets an EventBridge schedule. A daily roller runs at 00:00 UTC, queries reminders due in `[now, now+24h]`, deletes their EventBridge schedules, and inserts them into the Redis wheel for fine-grained dispatch.

## SQS Delay Queues, Cassandra Time Buckets, and Kafka Delay Brokers

Three more patterns worth knowing:

**SQS Delay Queues.** AWS SQS supports `DelaySeconds` per-message up to 900 seconds (15 minutes). For "in 5 minutes" tickler patterns, perfectly fine; for calendar reminders, the 15-minute cap disqualifies it. The pattern that *does* work: chain SQS-with-delay messages — when a 15-minute message fires, if the actual `fire_at` is still in the future, re-enqueue with another 15-minute delay. This is a poor man's delay queue and is frequently slower and more error-prone than just running Redis.

**Cassandra time-bucketed table.** Designed for petabyte-scale: shard reminders into per-minute partitions and have pollers read each minute's bucket as it goes due.

```sql
CREATE TABLE reminders_by_minute (
  minute_bucket  TIMESTAMP,                    -- truncated to 1-min boundary
  fire_at        TIMESTAMP,
  dedup_key      TEXT,
  payload        TEXT,
  PRIMARY KEY (minute_bucket, fire_at, dedup_key)
) WITH CLUSTERING ORDER BY (fire_at ASC, dedup_key ASC);
```

The poller reads the past minute's bucket, dispatches each row, then issues a tombstone delete. Cassandra tombstones are notoriously fiddly — a poll-then-tombstone pattern that misses a row leaves a permanent zombie. Use TTL on rows so they auto-expire even if the dispatcher loses a row, and use a separate audit log for "did this fire" rather than the queue itself. Discord runs a Cassandra-backed delay queue for its push reminder system at multi-trillion-message scale.

**Kafka with delay-broker.** Kafka has no native delay support; delays are layered on. The two patterns:

- **Per-delay topic.** Topics like `reminders.delay.5m`, `reminders.delay.1h`, `reminders.delay.24h`. A "delay broker" consumer reads each topic, sleeps until each message's `fire_at`, then forwards to the dispatcher topic. Simple but requires multiple topics for different delay buckets.
- **Single topic, consumer skips early messages.** All reminders go to one topic, sorted approximately by `fire_at` via a custom partitioner. The consumer compares each message's `fire_at` to `now()`; if early, it pauses the partition and sleeps. This is the LinkedIn pattern.

Kafka delay brokers shine when reminder dispatch needs **strict ordering within a partition** (e.g., partition by attendee so a single attendee's reminders are ordered) and **at-least-once delivery with exactly-once-ish semantics via idempotent consumers**. Most calendars don't need ordering and prefer the simpler ZSET + EventBridge hybrid.

## Cross-DST Reminder Accuracy

A reminder pinned to UTC is correct *until* the event's wall-clock-to-UTC mapping changes. Three ways the mapping can change after the reminder is queued:

1. **The IANA tzdb is updated.** A country abolishes DST or shifts the transition date with short notice. The future event's `dtstart_utc` recomputes; queued reminders pinned to the old UTC are now wrong.
2. **The user moves the event.** Organizer edits the start time. All reminders for all attendees reschedule.
3. **The user changes their reminder rule.** Attendee changes "15 min before" to "1 hour before" — the existing fire is wrong; cancel and re-enqueue.

For the tzdb case specifically:

```text
on_tzdb_update(from_version, to_version, affected_zones):
  # 1. Recompute dtstart_utc / dtend_utc for events in affected_zones
  for each event whose zone in affected_zones:
      old_start_utc = event.dtstart_utc
      new_start_utc = recompute_utc(event.dtstart_local, event.zone)
      if new_start_utc != old_start_utc:
          event.dtstart_utc = new_start_utc
          # 2. Cancel all queued reminders for affected occurrences
          for each (attendee, occurrence) in materialized_window:
              cancel_reminder(dedup_key(event.id, attendee.id, occurrence, ...))
          # 3. Re-enqueue with new fire_at
          enqueue_reminders_for_window(event)
          # 4. Notify the organizer (the meeting actually moved in UTC)
          notify_tzdb_drift(event, old_start_utc, new_start_utc)
```

Step 4 matters. A user who scheduled "2 PM Beirut time" expects 2 PM Beirut, not whatever 2 PM Beirut used to mean before the legislative change. The reminder fires at the new correct moment, but the *organizer should know* that the absolute UTC instant moved. For widely-attended events, also notify all attendees that the wall-clock didn't change but the UTC did — this matters for calendar federation (CalDAV peers will see the SEQUENCE bump).

For the deeper tzdb story, see [`./time-zone-correctness.md`](./time-zone-correctness.md).

## Cancellation on Event Edit or Delete

Cancellation is the most under-engineered part of most reminder systems. The mistake mode: an event is deleted, but its reminders remain queued; an hour later the user gets a "meeting in 15 minutes" notification for a meeting that doesn't exist.

Three concurrent strategies, all of which should be in place:

**Strategy 1: Delete from the queue at edit time.** When the event is deleted, iterate the materialized reminders and call `cancel_reminder(dedup_key)` for each. Simple, correct, works perfectly for Redis ZSET and EventBridge Scheduler. Fan-out cost: `attendees × occurrences × rules`.

```python
def on_event_deleted(event_id):
    # 1. Mark canonical event as deleted
    db.execute("UPDATE events SET status='cancelled' WHERE event_id=%s", (event_id,))

    # 2. Find all materialized reminders for this event
    pending = db.execute("""
        SELECT attendee_id, recurrence_id, method, offset, dedup_key
        FROM reminder_queue
        WHERE event_id = %s AND fire_at_utc > now()
    """, (event_id,))

    # 3. Cancel each in the queue backend
    for row in pending:
        cancel_reminder_in_redis(row["dedup_key"])
        # Or: scheduler.delete_schedule(Name=row["dedup_key"]) for EventBridge

    # 4. Delete from canonical reminder_queue
    db.execute("DELETE FROM reminder_queue WHERE event_id=%s AND fire_at_utc > now()", (event_id,))

    # 5. Notify attendees of cancellation (separate channel)
    notify_event_cancelled(event_id)
```

**Strategy 2: Late-bound re-validation at fire time.** The dispatcher, before sending, re-reads the canonical event:

```python
def dispatch_reminder(entry):
    event = db.fetch_event(entry["event_id"])
    if event is None or event["status"] == "cancelled":
        return  # silently drop
    if entry["recurrence_id"] is not None:
        occ = expand_occurrence(event, entry["recurrence_id"])
        if occ is None or occ["status"] == "cancelled":
            return  # this specific instance was deleted
    rsvp = db.fetch_rsvp(entry["event_id"], entry["attendee_id"], entry["recurrence_id"])
    if rsvp == "DECLINED":
        return  # respect the decline
    rule = db.fetch_reminder_rule(entry["event_id"], entry["attendee_id"],
                                   entry["method"], entry["offset"])
    if rule is None:
        return  # rule was deleted between queue and fire
    deliver(entry, event)
```

This is the safety net. Strategy 1 is cheaper but races with the dispatcher; Strategy 2 catches anything Strategy 1 missed. **Both must exist.**

**Strategy 3: Tombstone the dedup key.** When canceling, write a tombstone to a fast K/V (Redis SET) with TTL = max-future-fire. The dispatcher checks the tombstone before sending and skips. This handles the case where Strategy 1 deleted the queue entry but a duplicate copy lurked in a slow consumer; Strategy 2 reads the canonical event, but the tombstone is faster than a DB hit.

```python
def cancel_reminder(dedup_key, fire_at_utc):
    cancel_reminder_in_redis(dedup_key)            # Strategy 1
    ttl = max(0, fire_at_utc - now() + 300)         # 5-min grace
    redis.setex(f"tombstone:{dedup_key}", ttl, "1") # Strategy 3

def dispatch_reminder(entry):
    if redis.exists(f"tombstone:{entry['dedup_key']}"):
        return  # cancelled while in flight
    # ... Strategy 2 re-validation
```

For event *moves* (start time changed), the cancel-and-re-enqueue cycle is the same: cancel the old fire times, enqueue the new ones, tombstone the old dedup keys for the duration of the flight window.

## Idempotent Delivery

A retry storm in the queue (Redis network blip, EventBridge double-fires under load, dispatcher restart between dispatch and ack) can deliver the same reminder twice. The defense is an idempotency key plus a fast dedup store.

The natural key is the tuple that makes the reminder unique:

```text
dedup_key = sha256(event_id || attendee_id || recurrence_id || method || fire_at_utc)
```

Note that `fire_at_utc` is part of the key, not just `event_id + attendee_id`. If the event is moved, the new fire generates a new key — that's *correct*; the new fire is a different logical reminder.

The dispatch loop:

```python
def dispatch_reminder(entry):
    key = entry["dedup_key"]
    # SETNX: only the first dispatch wins
    if not redis.set(f"sent:{key}", "1", nx=True, ex=86400):
        return  # already sent within the last 24h, drop
    try:
        deliver_to_channel(entry)
    except DeliveryFailedNonRetryable:
        # Don't keep the dedup key — allow operator to manually retrigger
        redis.delete(f"sent:{key}")
        raise
    except DeliveryFailedRetryable:
        # Keep the dedup key with a short TTL so retry within window doesn't double-fire
        redis.expire(f"sent:{key}", 300)
        raise
```

Two operational notes:

- **TTL longer than the longest retry window.** If the channel retries for up to 1 hour after the initial dispatch, the dedup key must outlive that window or a successful retry races a fresh fire of the same reminder.
- **Per-channel idempotency keys downstream.** Push and SMS providers (FCM, APNs, Twilio) accept their own idempotency tokens; pass `dedup_key` as the provider-side `Idempotency-Key` header. This catches duplicates the queue layer missed.

For the deeper exactly-once vs at-least-once discussion, see [`../../../communication/exactly-once-and-idempotency.md`](../../../communication/exactly-once-and-idempotency.md).

## Snooze, Dismiss, and Multi-Device Fan-Out

**Snooze.** A snooze action reschedules the same reminder to fire later: `fire_at += snooze_duration`. The implementation is cancel-old + enqueue-new with the *same* logical identity but a new `fire_at`, so the dedup key changes:

```python
def snooze_reminder(dedup_key, snooze_seconds):
    entry = redis.hget("reminders:by_key", dedup_key)
    if not entry: return
    # Cancel the original
    cancel_reminder(dedup_key)
    # Enqueue at the new time
    new_fire_at = now() + snooze_seconds
    new_dedup_key = compute_key(entry, new_fire_at)
    enqueue_reminder(entry, new_fire_at, new_dedup_key)
    # Track the snooze chain for analytics
    db.insert_snooze_log(entry["event_id"], entry["attendee_id"], new_fire_at)
```

**Dismiss.** A dismiss is a permanent cancellation of *this firing* (not the rule). Drop the queue entry, write a tombstone, do not regenerate.

**Multi-device fan-out.** A user has 3 registered devices (phone, tablet, desktop). One reminder fires; all three should display it; tapping "dismiss" on any one should clear the others.

The pattern that works:

1. **Single dispatch event** to a "delivery coordinator" service per attendee.
2. The coordinator fans out to each registered device (FCM token, APNs token, web push subscription).
3. Each platform's silent push channel ("data message" in FCM, "background notification" in APNs) carries a `dedup_key` and a `display_at` instant.
4. The device displays the local notification.
5. On dismiss/snooze, the device sends an event back to the coordinator, which broadcasts a "dismissed" silent push to the other devices, which cancel their local notifications via `removeDeliveredNotifications` (APNs UNUserNotificationCenter) or `cancel()` (Android NotificationManager).

The cross-device dismissal hop is best-effort — if a device is offline, its local notification persists until it reconnects. The user usually finds this acceptable.

## Quiet Hours, Do-Not-Disturb, and Channel Routing

Channel routing is per-attendee, per-time-of-day, per-urgency. The data:

```text
attendee_preferences (
  attendee_id     BIGINT,
  channel_default JSONB,  -- e.g., {"work_hours": "push+email", "personal": "push"}
  quiet_hours     JSONB,  -- e.g., {"weekday": ["22:00","07:00"], "weekend": ["23:00","09:00"]}
  zone            TEXT,
  dnd_until       TIMESTAMPTZ,  -- temporary DND override
  PRIMARY KEY (attendee_id)
)
```

Three policies for what to do when a reminder fires inside quiet hours:

| Policy | Behavior | When to use |
|---|---|---|
| **Suppress entirely** | Drop the reminder. | Marketing, low-value reminders. |
| **Defer to end of quiet** | Reschedule for the moment quiet hours end. | Daily-summary reminders. |
| **Downgrade channel** | Switch from push to silent badge; hold email until morning. | Most calendar reminders — the user wants the meeting reminder, just not the buzz at 3 AM. |

The dispatcher applies the policy at fire time:

```python
def dispatch_reminder(entry):
    prefs = fetch_attendee_preferences(entry["attendee_id"])
    if in_quiet_hours(now(), prefs):
        if prefs.quiet_policy == "suppress":
            log_skipped(entry, reason="quiet_hours")
            return
        elif prefs.quiet_policy == "defer":
            new_fire = quiet_hours_end(prefs)
            re_enqueue(entry, new_fire)
            return
        elif prefs.quiet_policy == "downgrade":
            entry["method"] = downgrade_channel(entry["method"])  # push → badge, email stays
    # ... continue to channel-specific delivery
```

**DND override.** A user's "do not disturb" toggle is a hard short-term override. `dnd_until = now() + 2h` means suppress for 2 hours; reminders that should fire in that window get deferred to `dnd_until`. Calendar-event reminders with `<=10 min until start` may bypass DND (configurable per user) — the contract says "remind me right before; that's the whole point."

**Channel cost vs latency.** A rough hierarchy:

| Channel | Cost / message | Latency | Reliability | Use for |
|---|---|---|---|---|
| Local / popup (already on device) | $0 | <1 s | High when device online | Default |
| Push (FCM / APNs) | ~$0.0001 | <5 s | Medium (depends on connectivity) | Mobile attendee |
| Email (SES / SendGrid) | ~$0.0001 | 30 s – 5 min | High (asynchronous) | Long-lead reminders |
| SMS (Twilio) | $0.005 – $0.05 | <30 s | Very high | Critical, opt-in only |
| Voice call | $0.01 – $0.10 | <60 s | Very high | Healthcare, on-call |

A typical calendar's default routing: push first; email if no push delivered or for long-lead reminders; SMS only for explicitly opted-in critical reminders.

For the broader notification system architecture, see [`../../../building-blocks/message-queues-and-brokers.md`](../../../building-blocks/message-queues-and-brokers.md).

## Recurring Events and Per-Occurrence Reminders

Each occurrence of a recurring event has its own reminders. The fan-out implication: a daily standup with one reminder per attendee × 5 attendees × 365 occurrences in the rolling window = 1,825 queue entries.

The data model for recurring reminders:

```text
reminder_rules (
  event_id         BIGINT,
  attendee_id      BIGINT,
  method           TEXT,
  offset           INTERVAL,
  anchor           TEXT,           -- 'event_start' | 'midnight_local'
  -- Override-aware fields:
  applies_to       TEXT,           -- 'all' | 'this_only' | 'this_and_following'
  recurrence_id    TIMESTAMP,      -- the specific instance for 'this_only' / split point
  PRIMARY KEY (event_id, attendee_id, method, offset, recurrence_id)
)
```

Per-occurrence overrides work like RFC 5545 `RECURRENCE-ID`:

- **Override one occurrence's reminders.** "For *this* occurrence only, also send SMS." Insert a row with `applies_to='this_only'` and the specific `recurrence_id`.
- **Cancel reminders for one occurrence.** Insert an override row with `method='none'`. The materializer skips that occurrence.
- **Decline one occurrence.** RSVP = DECLINED for that `recurrence_id` cancels reminders for the attendee at that occurrence; series-level reminders remain for other occurrences.

The materializer iterates: for each (occurrence, attendee) pair, resolve the effective rule set (default ∪ series ∪ instance overrides − instance excludes) and enqueue.

A common bug: changing the master series's reminders doesn't propagate to already-queued occurrences. Fix: on rule edit, cancel all queued reminders for affected occurrences and re-materialize. This is the same fan-out as event move.

## Worked Example: 09:00 NYC Meeting, London Attendee

Event: weekly project sync, organized by Alice (NYC), `RRULE:FREQ=WEEKLY;BYDAY=TU`, `DTSTART;TZID=America/New_York:20260505T090000`. Bob (London, `Europe/London`) is an attendee. Bob's default reminder is "1 hour before, push notification."

**Step 1: First occurrence — May 5, 2026 (Tuesday).**

| Field | Value |
|---|---|
| Event local start (NYC) | 2026-05-05 09:00 EDT |
| `dtstart_local` | 2026-05-05 09:00 |
| Event zone | `America/New_York` |
| `dtstart_utc` | 2026-05-05 13:00:00 UTC |
| Bob's reminder offset | -1h (`PT1H`) |
| Reminder `fire_at_utc` | 2026-05-05 12:00:00 UTC |
| Bob's local at fire | 2026-05-05 13:00 BST (London) |

The reminder fires at noon UTC. Bob sees a push at 1 PM London time saying "Project sync at 14:00 BST (in 1 hour)." Display is in Bob's local zone; scheduling is in UTC; the offset is calculated against the event's UTC instant.

**Step 2: Second occurrence — May 12, 2026.**

Same arithmetic; nothing surprising. Both NYC and London are still in their summer time. Reminder fires 2026-05-12 12:00:00 UTC.

**Step 3: NYC drops to standard time on November 1, 2026.**

EDT (UTC-4) becomes EST (UTC-5). London dropped from BST to GMT a week earlier (October 25, 2026, EU rules). The pattern of who-is-on-DST-when matters.

| Date | NYC offset | London offset | Event local NYC | Event UTC | Bob's local at fire |
|---|---|---|---|---|---|
| 2026-10-27 | EDT (UTC-4) | GMT (UTC+0) | 09:00 | 13:00 UTC | 13:00 GMT |
| 2026-11-03 | EST (UTC-5) | GMT (UTC+0) | 09:00 | **14:00 UTC** | **14:00 GMT** |

Between October 27 and November 3, the *UTC instant* of the meeting moved by one hour. Why? Because NYC fell back; the local 09:00 NYC anchor stayed; the UTC computed from "09:00 in `America/New_York`" shifted from `09:00 - (-4h) = 13:00 UTC` to `09:00 - (-5h) = 14:00 UTC`. Bob's reminder, which fires "1 hour before in UTC," now fires at 13:00 UTC (= 13:00 GMT London) for the November 3 occurrence — *one hour later* in London than the October 27 reminder.

**This is correct behavior**, even though it can feel surprising. The user's intent is "9 AM in NYC"; the wall-clock anchor in NYC is preserved; the UTC instant moves; the reminder offset from UTC moves with it; Bob's display time in London moves to match.

**Step 4: London also falls back? No — already happened.** The EU completed its transition a week earlier, so London is on GMT for both occurrences. If Bob lived in continental Europe and his zone fell back on the same week as NYC, the *delta* between Alice's NYC time and Bob's local time would be unchanged across the transition; Bob's display would remain "13:00 local."

**Step 5: What if the materialized reminders were enqueued in October and not refreshed before November?**

If the queue carries `fire_at_utc = 2026-11-03T12:00:00Z` (computed under the EDT-still-valid assumption), the reminder fires an hour early (12:00 UTC = 12:00 GMT London = 07:00 EST NYC). The meeting *itself* was correctly recomputed (NYC organizer sees 09:00 NYC = 14:00 UTC), but Bob's reminder is wrong.

The fix is the materializer: when the event's UTC instant changes (DST transition for the organizer's zone, tzdb update, event move), cancel all queued reminders and re-enqueue. The most common production bug is to recompute the event's UTC but not the reminders' UTC.

**Step 6: What if Bob travels to Tokyo on November 3?**

Display shifts: 14:00 UTC = 23:00 JST. The reminder fires at 13:00 UTC = 22:00 JST. Bob sees "Project sync at 23:00 JST (in 1 hour)" on his Tokyo device. The dispatcher resolved the local time at fire time using Bob's *current* zone (advertised by the device or remembered preference), not the zone he was in when he was added to the meeting.

This is why the queue stores `fire_at_utc` and the device renders local at fire time: the user's zone is mutable; UTC is not.

## Anti-Patterns

**1. Per-event single delivery instead of per-attendee.** Designing the queue around `(event_id)` rather than `(event_id, attendee_id)` means everyone in a meeting gets the organizer's reminders, no one gets their personal preferences, and quiet hours can't be honored per-attendee. The reminder identity must include the attendee.

**2. Polling the database every minute for due reminders.** A `SELECT * FROM reminders WHERE fire_at_utc <= now()` against a billion-row table every 60 seconds is a self-inflicted DDOS. Use a delay queue (ZSET / EventBridge / Cassandra time-bucket) where the index *is* `fire_at_utc` and the read is bounded to the next minute's worth.

**3. No cancellation on event deletion.** The user deletes a meeting; the reminders fire anyway because the queue wasn't told. The dispatcher must re-validate against the canonical event at fire time; the queue should be told at edit time as a fast path; tombstones cover the race.

**4. Duplicates on retry without idempotency.** Two queue claimers, two dispatches; the user gets two notifications. Worse: the user's SMS provider charges per message. The dedup key keyed on `(event_id, attendee_id, recurrence_id, method, fire_at_utc)` plus a Redis SETNX before delivery solves this.

**5. Storing reminders as offsets without the anchor.** "1 day before" is ambiguous: 24 hours before the event start, or 09:00 the previous day in the attendee's zone? Both are reasonable; pick one per rule and store the anchor explicitly. Cron-style "morning of" reminders should anchor to attendee-local midnight + offset, not to event start − 24h.

**6. Pinning UTC at create time and never recomputing.** A future event whose zone gets a DST law change keeps firing at the wrong moment because nobody recomputed. The tzdb update job must scan affected events and re-anchor.

**7. Treating snooze as "fire again now."** Snooze must reschedule with a new `fire_at`; firing immediately and pretending it's snoozed creates a feedback loop (user snoozes again, reminder fires again immediately).

**8. No multi-device dismissal.** A user dismisses on their phone; the watch keeps buzzing. The dismiss must propagate to other registered devices via silent push.

**9. Treating quiet hours as "drop the reminder."** A meeting reminder dropped at 3 AM doesn't fire at 09:00 either — the moment passed. Defer or downgrade; don't suppress unless the user explicitly opted in.

**10. Reminders for declined occurrences.** A user declined the Tuesday standup three weeks running; reminders still fire because the dispatcher checks event existence, not attendee RSVP. Re-validate RSVP per-occurrence at fire time.

**11. Using SMS as a default channel.** SMS is expensive and intrusive. Default to push; let users opt into SMS for specific reminder types (medical appointments, on-call) with explicit consent flow.

**12. Recurring event reminders that materialize forever.** Naive expansion of `RRULE:FREQ=DAILY;UNTIL=99991231T235959Z` produces infinite reminders. The roller window (e.g., 13 months forward) bounds materialization; a daily extension job pushes the window forward.

**13. Cancellation that races the dispatcher.** The user deletes the event; the cancellation hasn't propagated to the queue; the dispatcher pops the entry and sends a stale reminder. Tombstone + late-bound re-validation defends this; without both, the user gets ghost notifications.

**14. Hardcoding timezone offsets in reminder math.** "Subtract 4 hours for EDT" is wrong half the year. Always go through tzdb via the IANA zone string.

**15. Not preserving `fire_time` semantics.** A catch-up dispatcher that fires a missed reminder with `fire_time = now()` instead of the original schedule writes the wrong context (if the reminder template includes "in 15 minutes," it should still say "in 15 minutes" — relative to the *original* fire time). For missed-fire policy detail, see [`../../async/job-scheduler/missed-fire-policies.md`](../../async/job-scheduler/missed-fire-policies.md).

**16. Single distributed lock for the whole queue.** Per-job (per-fire) locks scale; one lock for the whole reminder queue is a hot row that serializes all dispatch. See [`../../async/job-scheduler/distributed-lock-per-job.md`](../../async/job-scheduler/distributed-lock-per-job.md) for the right granularity.

## Related

- [RRULE Expansion](./rrule-expansion.md) — recurrence materialization within a rolling window; the same pattern that bounds reminder fan-out.
- [Time-Zone Correctness](./time-zone-correctness.md) — IANA tzdb, DST transitions, and the local-vs-UTC storage rules that reminders depend on.
- [Design a Calendar System (parent)](../design-calendar-system.md) — the full case study; this doc expands §6.
- [Missed-Fire Policies](../../async/job-scheduler/missed-fire-policies.md) — what to do when reminders weren't fired during an outage; per-reminder skip vs catch-up vs coalesce.
- [Distributed Lock Per Job](../../async/job-scheduler/distributed-lock-per-job.md) — preventing double-dispatch across queue claimers.
- [Exactly-Once and Idempotency](../../../communication/exactly-once-and-idempotency.md) — the broader idempotency story that the reminder dedup key participates in.
- [Message Queues and Brokers](../../../building-blocks/message-queues-and-brokers.md) — broader queue mechanics that delay queues specialize.

## References

- [RFC 5545 §3.6.6 — VALARM Component](https://datatracker.ietf.org/doc/html/rfc5545#section-3.6.6) — the iCalendar alarm component; ACTION, TRIGGER, DESCRIPTION semantics for popup, email, and audio reminders.
- [RFC 5546 — iCalendar Transport-Independent Interoperability Protocol (iTIP)](https://datatracker.ietf.org/doc/html/rfc5546) — invitation messages over which VALARM is conveyed; why receiver-side defaults usually win over organizer VALARMs.
- [Apple UNUserNotificationCenter](https://developer.apple.com/documentation/usernotifications/unusernotificationcenter) — iOS / macOS notification center API; scheduling local notifications, content, and removeDeliveredNotifications for cross-device dismissal.
- [Apple WWDC — User Notifications](https://developer.apple.com/documentation/usernotifications) — the broader local + remote notification programming model.
- [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging) — Android push delivery; data messages for silent dispatch and notification messages for displayed alerts.
- [AWS EventBridge Scheduler — Getting Started](https://docs.aws.amazon.com/scheduler/latest/UserGuide/getting-started.html) — managed long-delay scheduler with one-shot `at()` schedules, FlexibleTimeWindow, and ActionAfterCompletion=DELETE auto-cleanup.
- [AWS SQS Delay Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html) — message-level DelaySeconds up to 900 s; the chained-delay anti-pattern people reach for when they need longer.
- [Redis Sorted Sets — ZADD](https://redis.io/commands/zadd/) — the canonical primitive for delayed-job queues; ZRANGEBYSCORE for poll, ZREM for atomic claim, ZRANGEBYSCORE+ZREMRANGEBYSCORE Lua script for batch dispatch.
- [Redis Sorted Sets — ZRANGEBYSCORE](https://redis.io/commands/zrangebyscore/) — the read primitive that pops due entries below `now()`.
- [Discord — How Discord Stores Trillions of Messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) — Cassandra-backed time-bucketed scaling pattern referenced for billion-scale delay queues.
