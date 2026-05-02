---
title: "Missed-Fire Policies — Catch-Up, Skip, and Coalesce After Outages"
date: 2026-05-01
updated: 2026-05-01
tags: [system-design, deep-dive, scheduler, missed-fire, recovery]
---

# Missed-Fire Policies — Catch-Up, Skip, and Coalesce After Outages

**Date:** 2026-05-01 | **Updated:** 2026-05-01
**Tags:** `system-design` `deep-dive` `scheduler` `missed-fire` `recovery`

> **Parent case study:** [Design a Job Scheduler](../design-job-scheduler.md). This deep-dive expands §7.5 "Missed-fire policies".

## Table of Contents

- [Summary](#summary)
- [The Outage Problem](#the-outage-problem)
- [Three Classic Policies](#three-classic-policies)
- [Policy Semantics: Catch-Up, Skip, Coalesce](#policy-semantics-catch-up-skip-coalesce)
- [Idempotent vs Non-Idempotent Jobs](#idempotent-vs-non-idempotent-jobs)
- [Choosing a Policy by Job Shape](#choosing-a-policy-by-job-shape)
- [Detection: How Does the Scheduler Know It Missed Anything?](#detection-how-does-the-scheduler-know-it-missed-anything)
- [Watermark-Based Detection](#watermark-based-detection)
- [Per-Policy Fire-List Computation](#per-policy-fire-list-computation)
- [Catch-Up Rate Limiting](#catch-up-rate-limiting)
- [Multi-Region Schedulers](#multi-region-schedulers)
- [Worker Capacity During Catch-Up](#worker-capacity-during-catch-up)
- [Operator Override and Incident Controls](#operator-override-and-incident-controls)
- [Worked Example: 2-Hour Outage of an Hourly Job](#worked-example-2-hour-outage-of-an-hourly-job)
- [How Real Systems Implement This](#how-real-systems-implement-this)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

The parent case study introduces missed-fire policies in roughly a dozen lines: list the Quartz misfire instructions, mention a misfire threshold, note that the scheduler scans for missed fires on startup. That summary is correct but elides the engineering required to make catch-up safe in production. Real systems must decide, per job, whether the missed window's *meaning* was "do every occurrence I missed" (catch-up), "the moment passed, drop them" (skip), or "the missed occurrences collapse into a single fire" (coalesce). They must detect missed fires reliably across schedule edits, daylight-saving boundaries, and multi-region failovers. They must rate-limit replay so a 12-hour outage doesn't generate a thundering herd of 720 simultaneous hourly jobs. And they must survive the worst case — non-idempotent jobs with default catch-up — by making the policy a first-class per-job field, not a global toggle. This deep-dive walks the policy semantics, the detection mechanics (computed schedule vs. recorded history), the watermark approach Airflow uses, the rate-limit machinery Sidekiq and Temporal expose, and the anti-patterns that turn an outage into a self-inflicted incident. The recurring theme: missed-fire behavior is a product decision before it is a code decision, and the scheduler's job is to make every legitimate choice expressible cleanly.

## The Outage Problem

Concrete scenario. The scheduler cluster goes down at 23:00 UTC on day D and recovers at 01:00 UTC on day D+1. Across that two-hour gap, the following triggers were due to fire:

- An hourly metric scrape (`23:00`, `00:00`, `01:00` — three fires, one straddling the recovery edge).
- A daily report generator (`00:00` — one fire, missed by an hour).
- A push-notification fan-out for a "good morning" campaign in Asia/Tokyo (`23:30 UTC` = `08:30 JST` — one fire, time-sensitive).
- A retry sweeper that reaps stuck rows every 5 minutes (`23:00, 23:05, … 00:55` — twenty-four fires).

When the scheduler restarts and reads the database at 01:00, it sees four jobs whose `next_fire_at` is in the past. What should it do?

The answer is *not* "fire them all immediately." Doing so produces:

- The hourly scrape runs three times back-to-back at 01:00 — the metric ingestion service sees three writes for the 23:00, 00:00, 01:00 buckets, two of which carry stale snapshot data because they observe the world at 01:00, not at the original fire time.
- The daily report generates a "yesterday" report at 01:00, which is correct *if* it queries by date range.
- The good-morning push goes out at 01:00 UTC = 10:00 JST — too late to be a good-morning push at all. It is now an annoyance.
- The retry sweeper kicks off twenty-four parallel sweeps in the same second, all contending for the same row locks, half of them blocked on the others.

Different jobs need different policies. There is no single right answer because each job's *contract with the wall clock* is different. The scheduler's job is to honor those contracts after recovery.

## Three Classic Policies

Quartz nailed the vocabulary that the rest of the industry mostly inherits. The relevant misfire instructions for `CronTrigger` (and the equivalent for `SimpleTrigger`):

| Quartz constant | Behavior | Common name |
|---|---|---|
| `MISFIRE_INSTRUCTION_FIRE_NOW` (SimpleTrigger) / `MISFIRE_INSTRUCTION_FIRE_ONCE_NOW` (CronTrigger) | Fire one occurrence immediately, then resume the normal schedule. | **Coalesce** |
| `MISFIRE_INSTRUCTION_DO_NOTHING` | Skip everything missed. Advance `next_fire_at` to the next future scheduled occurrence. | **Skip** |
| `MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT` (SimpleTrigger) / `MISFIRE_INSTRUCTION_FIRE_ALL_WITH_REMAINING_COUNT` | For SimpleTriggers with a finite repeat count: re-fire all remaining occurrences. | **Catch-up bounded** |
| `MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY` | Replay every missed fire with no rate limit. | **Catch-up unbounded** |
| `MISFIRE_INSTRUCTION_SMART_POLICY` | Library picks: SimpleTriggers usually get a re-fire of remaining occurrences; CronTriggers usually get FIRE_ONCE_NOW. | **Smart default** |

The simplifying mental model used through the rest of this doc — and matching what Airflow, Sidekiq-Cron, and EventBridge Scheduler implement under different names — collapses these to three policies:

| Policy | What happens to missed fires | Equivalent in Quartz |
|---|---|---|
| **Catch-up (FIRE_ALL)** | Every missed occurrence runs, in order. | `IGNORE_MISFIRE_POLICY` (no limit) or `FIRE_ALL_WITH_REMAINING_COUNT` (bounded) |
| **Skip (IGNORE_MISFIRES)** | All missed occurrences are dropped; only future fires happen. | `DO_NOTHING` |
| **Coalesce (FIRE_LAST_WITH_REMAINING_COUNT)** | The whole missed window collapses into a single immediate fire; the schedule resumes. | `FIRE_ONCE_NOW` |

Different libraries lean on different vocabulary. Airflow speaks of `catchup=True` versus `catchup=False`; Sidekiq-Cron exposes `:fire_immediately` and a backfill window; EventBridge Scheduler offers `FlexibleTimeWindow` plus a delivery model that drops fires older than a threshold. The semantics map cleanly to **catch-up / skip / coalesce**.

## Policy Semantics: Catch-Up, Skip, Coalesce

Make the contracts precise.

**Catch-up.** "Every missed occurrence is a real, distinct execution. Run them all, ideally in original order, ideally with the original `fire_time` available to the job so it knows which window it represents."

- The job's `fire_time` argument is the *originally scheduled* time, not "now." This is what makes ETL-style catch-up sensible — the 02:00 partition fire really is for the 02:00 partition, even if it actually starts running at 03:30.
- The system replays in chronological order so dependent jobs and downstream state observe the original sequence.
- Total work scales with outage duration × fire frequency. A 12-hour outage of a 5-minute job produces 144 fires. The scheduler must cap concurrency or the worker fleet melts.

**Skip.** "Missed time is lost time. Pretend the fires never existed; the next legitimate execution is the next scheduled future fire."

- `next_fire_at` advances to the smallest future occurrence ≥ `now()`.
- Nothing runs immediately. There is no `fire_time` < now in any new run record.
- Operationally cheap; only safe when each fire is purely refresh-style — losing one is equivalent to skipping a heartbeat.

**Coalesce.** "There were N missed fires; semantically I only need one of them. Run a single fire now with `fire_time = now()` (or, in some implementations, `fire_time` = the last missed scheduled time), then resume the regular schedule."

- This is the default for most CronTriggers in Quartz precisely because it matches the most common need: "yes, I want yesterday's batch, but only one copy of it."
- The work executed at recovery represents the cumulative effect of the missed window, *not* a strict per-occurrence replay. The job's logic must be tolerant of "I'm running late and the time range is wider than my normal window."

These three semantics differ on **how many runs are produced** *and* **what `fire_time` each run carries**. That second axis matters because correct downstream behavior often depends on `fire_time` being the *originally scheduled* moment.

## Idempotent vs Non-Idempotent Jobs

The single most important question when picking a policy is: **does running the same fire twice cause damage?**

| Job type | Idempotent? | Safe default |
|---|---|---|
| Refresh a cache | Yes | Catch-up or Coalesce, doesn't matter |
| Recompute a daily aggregate over `fire_date` | Yes (deterministic over the date range) | Catch-up if you want the partition; Coalesce if you only want today |
| Send a marketing push notification | **No** | Skip (almost always) |
| Charge a subscription renewal | **No** (idempotency keyed externally) | Catch-up *only if* the run is keyed on `(customer_id, billing_period)` so a replay is a no-op |
| Generate an invoice PDF | Maybe (overwrites file) | Coalesce |
| Send a "you forgot something" reminder email | **No** | Skip |
| Reap stuck rows from a queue | Yes | Coalesce (one sweep covers the gap) |
| Sync inventory snapshot to a partner | Yes (last-write-wins) | Coalesce |
| Run a regulatory daily report | Yes (must produce one report per day) | Catch-up |
| Health-probe an external API | Yes but stale | Skip |

Two design principles fall out of the table:

1. **Default catch-up is dangerous.** It is the right answer for some jobs (idempotent ETL) and the wrong answer for many (notifications, communications, time-sensitive triggers). Pick the policy per-job; do not let the platform's default be `catch_up = true` for everything.
2. **Idempotency is a property of the job, not the scheduler.** The scheduler can replay safely *only* if the job is idempotent on `fire_time`. The retry-and-idempotency design (see [./retry-and-idempotency.md](./retry-and-idempotency.md)) is what unlocks safe catch-up. Without an idempotency key derived from `(job_id, fire_time)`, replay is gambling.

## Choosing a Policy by Job Shape

A pragmatic decision tree:

```text
Is the job effect time-sensitive?  (push, "good morning", flash sale)
  Yes → Skip (the moment passed; running late is worse than not running)
  No  → continue

Does each fire produce a unique, durable artifact keyed by fire_time?
  (e.g. partition, invoice, aggregate, regulatory report)
  Yes → Catch-up (with rate limit; preserve original fire_time)
  No  → continue

Is the job high-frequency (sub-minute or minute) and does it converge to a fixed-point state?
  (cache refresh, queue sweep, health check, metric scrape)
  Yes → Coalesce (one fire suffices)
  No  → continue

Is the job a daily-ish "deadline" job (rollup, batch, report) that should produce yesterday's output if missed?
  Yes → Catch-up (single missed fire at most; replay it)
  No  → consider Skip as the safest default and revisit when usage clarifies
```

Concrete mapping for the four jobs in the opening scenario:

| Job | Frequency | Policy | Reason |
|---|---|---|---|
| Hourly metric scrape | Hourly | Coalesce | Each scrape converges to the same view of the world; one fire at recovery captures the current state. |
| Daily report generator | Daily | Catch-up (max 1 missed) | Yesterday's report still has business value; replay with `fire_time = D-1 00:00`. |
| Good-morning push | Daily, time-of-day | Skip | A push at 10am local is annoying; the moment is gone. |
| Retry sweeper | Every 5 min | Coalesce | A single sweep at recovery time reaps everything that's stuck. |

## Detection: How Does the Scheduler Know It Missed Anything?

Two approaches to detecting missed fires; they have different failure modes.

**1. Recorded-schedule history (read `schedule_history`).** Every fire that *did* execute is recorded in a `runs` (or `schedule_history`) table with `(job_id, fire_time, status)`. On scheduler startup, for each job, find the largest `fire_time` for which a run exists and compare to the cron's expected schedule.

**2. Computed-schedule comparison (read `next_fire_at` from `schedules`).** The schedules table stores `next_fire_at` for each active job. On startup, any row with `next_fire_at < now() - misfire_threshold` indicates a missed fire window.

The two approaches are not equivalent.

| Property | Recorded history | Computed schedule |
|---|---|---|
| Detects fires that the scheduler enqueued but the worker didn't finish | Yes (status = ENQUEUED, RUNNING, FAILED) | No (only sees what hasn't been advanced) |
| Survives schedule edits (cron expression changed during outage) | Tricky — history was for old expression | Clean — `next_fire_at` is per the current expression |
| Survives a scheduler that crashed mid-write to `next_fire_at` | Yes (history is the source of truth) | Risky — `next_fire_at` may be stale or unwritten |
| Cost on startup | Read history per job + recompute schedule | Single index range scan: `WHERE next_fire_at < now() - threshold` |
| Production choice | For correctness audits | For startup recovery |

In practice, production systems use both: the computed schedule (`next_fire_at < now() - threshold`) drives the *recovery action*, and the recorded history is consulted to *deduplicate* — if a run already exists for that `fire_time` (perhaps another scheduler instance enqueued it before crashing), don't enqueue a second.

```sql
-- Find missed fires on scheduler startup
SELECT job_id, next_fire_at, misfire_policy, last_successful_fire_at
FROM schedules
WHERE status = 'ACTIVE'
  AND next_fire_at < now() - INTERVAL '60 seconds'    -- misfire_threshold
ORDER BY next_fire_at ASC;

-- Before enqueueing a missed fire, dedupe against history
SELECT 1 FROM runs
WHERE job_id = $1 AND fire_time = $2
  AND status IN ('ENQUEUED', 'RUNNING', 'SUCCESS');
```

**A subtle failure mode: the "phantom missed fire" after a long pause.** A job that has been intentionally paused for a week resumes; the scheduler sees `next_fire_at` was a week ago and computes 168 missed hourly fires. This is *correct* per the algorithm but *wrong* per intent — the operator paused the job; resume should mean "from now on," not "replay the entire pause window." Two defenses: (a) when pausing, also clear `next_fire_at` and recompute on resume; (b) have a separate `paused_until` column that the missed-fire detector ignores. The watermark approach handles this naturally because `last_successful_fire_at` doesn't move during the pause, but the operator has to decide whether to advance it manually on resume.

**Another subtle case: the just-edited-cron problem.** An operator changes a job from "every 5 minutes" to "hourly at :00" while it's actively running. Without care, the next missed-fire scan sees `next_fire_at` set under the old expression and decides nothing was missed; or, worse, sees a watermark that was last advanced under the old expression and enumerates dozens of hourly occurrences "missed" within the last hour. The cleanest defense: on schedule edits, recompute `next_fire_at` immediately under the new expression and *also* advance the watermark to "edit time" so catch-up enumeration starts fresh. Document this explicitly so operators understand that schedule edits cleanly demarcate the "before" and "after" eras.

## Watermark-Based Detection

Airflow's model — and the cleanest way to think about catch-up in general — is the **watermark**. Each job carries `last_successful_fire_at` (or, in Airflow's vocabulary, the latest successful `data_interval_end`). The scheduler computes catch-up from there:

```text
catch_up_window = ( last_successful_fire_at, now() ]
missed_fires    = enumerate_cron_in_window(cron, catch_up_window)
```

The watermark eliminates a class of bugs that pure `next_fire_at`-based detection has:

- **Schedule edited during outage.** If the cron expression changes from "hourly" to "every 30 minutes" while the scheduler is down, `next_fire_at` is stored under the new expression. Watermark-based catch-up enumerates the *new* schedule from the watermark forward — which is what most operators want, because they edited the schedule for a reason.
- **Multiple in-flight fires.** If the scheduler crashed mid-batch and three fires are partially recorded, the watermark only advances when a fire reaches `SUCCESS`. Catch-up enumerates everything strictly after the last *successful* moment, naturally including in-flight failures.
- **Idempotency contract is explicit.** The job-side dedup key is `(job_id, fire_time)`. Catch-up is safe whenever the job stores its output keyed on `fire_time`, because re-running for an already-completed `fire_time` is a no-op.

```pseudocode
function compute_missed_fires(job):
    cron       = job.cron_expression
    tz         = job.timezone
    watermark  = job.last_successful_fire_at  // null on first ever run
    threshold  = global_misfire_threshold     // e.g. 60s
    horizon    = now() - threshold            // anything past horizon is "current," not missed

    if watermark is null:
        // First run — only consider the most recent past occurrence as "missed"
        most_recent = previous_cron_occurrence(cron, tz, horizon)
        return [most_recent]  // skip policy will discard, catch-up will fire

    missed = []
    cursor = next_cron_occurrence(cron, tz, after=watermark)
    while cursor <= horizon:
        missed.append(cursor)
        cursor = next_cron_occurrence(cron, tz, after=cursor)

    return missed
```

The watermark, the cron evaluator, and the timezone resolver are the three primitives. Get them right and every policy is just a different reduction over `missed`.

For time-zone correctness in `next_cron_occurrence` (DST transitions are a frequent source of phantom missed fires), see [./time-zone-correctness.md](./time-zone-correctness.md).

## Per-Policy Fire-List Computation

Given the `missed` list from above, each policy reduces it differently:

```pseudocode
function fires_to_enqueue(job, missed):
    if missed is empty:
        return []

    switch job.misfire_policy:

        case CATCH_UP:
            // Every missed fire becomes a run. Preserve original fire_time.
            return [Run(job_id=job.id, fire_time=t, scheduled_for=now())
                    for t in missed]

        case SKIP:
            // Drop everything missed. Caller advances next_fire_at to next future occurrence.
            return []

        case COALESCE:
            // One fire that represents the whole window.
            // Convention: fire_time = the *latest* missed occurrence (so partition logic still works).
            return [Run(job_id=job.id,
                        fire_time=last(missed),
                        scheduled_for=now(),
                        coalesced_count=len(missed))]

        case CATCH_UP_BOUNDED:
            // Replay at most N most recent occurrences (e.g. last 24 hours of an hourly job).
            window_start = now() - job.max_catchup_lookback   // e.g. 24h
            recent = [t for t in missed if t >= window_start]
            return [Run(job_id=job.id, fire_time=t, scheduled_for=now()) for t in recent]
```

Three details worth calling out:

1. **`fire_time` versus `scheduled_for`.** `fire_time` is the logical/contract time the job represents; `scheduled_for` is the wall-clock time the scheduler actually wants the worker to start. Catch-up needs both because the job's idempotency key uses `fire_time` and the worker's "do not run before" gate uses `scheduled_for`.
2. **Coalesce keeps the count.** Recording `coalesced_count` lets the job log "this fire absorbed N missed occurrences" — useful for surveillance ("did the scheduler eat my fires?") and for the job to scale its workload (a coalesced sweeper might process more rows because the window was wider).
3. **Bounded catch-up is the safe default for catch-up.** Unbounded catch-up after a 12-hour outage of a per-minute job is 720 runs. A bounded window of, say, "at most the last hour" gives the operator a knob and prevents the worst tail.

## Catch-Up Rate Limiting

Catch-up replay must not stampede the worker fleet. Two strategies; they answer different questions.

**A. Replay at original cadence.** Rebuild the schedule's natural rhythm. If the job is hourly, fire one occurrence per hour starting at recovery; the system catches up over time but keeps load steady.

```pseudocode
function catch_up_at_original_cadence(job, missed):
    // Schedule each missed fire at intervals matching the cron's natural period
    cadence = estimate_cron_period(job.cron)   // e.g. 3600s for hourly
    base    = now()
    runs    = []
    for i, fire_time in enumerate(missed):
        runs.append(Run(
            job_id        = job.id,
            fire_time     = fire_time,
            scheduled_for = base + i * cadence,
        ))
    return runs
```

Pro: no thundering herd; load looks like the steady state. Con: takes as long to catch up as the outage itself — a 2-hour outage of an hourly job means catch-up finishes 2 hours after recovery. Rarely the right choice for short outages of low-frequency jobs; sometimes the right choice for long outages of high-frequency jobs where the worker fleet can't absorb the burst.

**B. Replay as fast as possible with a concurrency cap.** Fire missed occurrences immediately, but bound the number simultaneously running.

```pseudocode
function catch_up_with_concurrency_cap(job, missed, max_concurrent):
    // All runs are scheduled for `now()`, but the dispatcher enforces concurrency.
    runs = [Run(job_id=job.id,
                fire_time=t,
                scheduled_for=now(),
                catchup_priority=t)              // older fires go first
            for t in missed]
    return runs

// At the dispatcher
function dispatch_loop():
    while true:
        running = count_running_for_job(job_id)
        if running >= job.max_concurrent_catchup:    // e.g. 4
            sleep(small)
            continue
        next_run = pop_oldest_pending_for_job(job_id)
        if next_run is None: break
        worker_pool.submit(next_run)
```

Pro: outage recovers fast. Con: requires the dispatcher to know about per-job concurrency — many simple schedulers don't, so this is the place where Quartz's `@DisallowConcurrentExecution`, Airflow's `max_active_runs`, and Temporal's per-schedule overlap policy live.

A combined rule that works in practice:

```text
if num_missed > catchup_panic_threshold (e.g., 100):
    fall back to coalesce + alert operator
elif num_missed > burst_threshold (e.g., 10):
    catch up at cadence (Strategy A)
else:
    catch up with concurrency cap (Strategy B)
```

## Multi-Region Schedulers

When the scheduler runs in two or more regions for HA, missed-fire logic gets a coordination problem.

**Single active region (failover).** One region holds the leader lease; the other(s) are warm standbys. On failover, the new leader reads the shared database and runs the same missed-fire detection it would on a cold restart. Watermarks live in the database, so the new leader sees `last_successful_fire_at` from the old leader's last commit and computes catch-up from there. This is the cleanest model and what most production systems use; see [./leader-election-for-scheduler.md](./leader-election-for-scheduler.md).

**Active-active partitioned by job.** Each region owns a disjoint subset of jobs (sharded by `job_id`). On region failure, ownership migrates. Catch-up is the surviving region's responsibility for the migrated subset. The watermark is again shared; the only new concern is the migration handoff window — make sure ownership transitions atomically so two regions don't both "discover" the same missed fires and double-enqueue. A fencing token incremented on every ownership change defends this; see the broader pattern in [../../../reliability/disaster-recovery.md](../../../reliability/disaster-recovery.md).

**Active-active with shared journal.** All regions read the same schedule and use a per-job distributed lock (see [./distributed-lock-per-job.md](./distributed-lock-per-job.md)) to serialize fires. Catch-up detection runs in each region's loop; whichever region acquires the lock first does the catch-up work. The lock prevents double-enqueue. This is more code than failover-with-leader and is rarely worth it unless the regions are far apart and you need active-active for latency reasons.

In all three models the missed-fire policy itself is unchanged — it is a property of the job, applied wherever the scheduler is running. What changes is **which scheduler instance is responsible for applying the policy**, and that responsibility must be unambiguous.

## Worker Capacity During Catch-Up

A catch-up burst is a load event. The worker fleet should be sized — or scaled — to absorb it without falling over.

**Three levers.**

1. **Per-job concurrency cap.** "At most 4 instances of `daily_invoice_rollup` running at once." Prevents a single job from monopolizing workers during catch-up. Quartz uses `@DisallowConcurrentExecution`; Airflow uses `max_active_runs`; Sidekiq uses `unique` jobs; Temporal uses `WORKFLOW_ID_REUSE_POLICY` and schedule overlap policies.
2. **Global worker pool autoscaling.** If catch-up is queued, signal autoscaling to spin up additional workers. This requires the queue depth to be observable (a Prometheus metric is enough) and the autoscaler to react fast enough — usually 1–3 minutes — so a brief catch-up bursts into temporary capacity rather than starving steady-state work.
3. **Burst-capacity reservation.** Reserve a fraction of the worker pool (say, 20%) for catch-up traffic. Implement as a separate queue or a priority-aware dispatcher: catch-up runs go into a "burst" lane; steady-state runs continue uninterrupted.

The combination matters more than any single lever. Without per-job concurrency, autoscaling reacts to a flood that could have been throttled at the source. Without autoscaling, per-job concurrency caps slow catch-up to the same hours-long timeline as the outage. Without burst reservation, catch-up steals capacity from urgent steady-state jobs (an alert-handler that fires every minute has every right to keep running through a catch-up burst).

## Operator Override and Incident Controls

During a real incident, the operator usually wants to override per-job policies. The platform should expose this without requiring a deploy.

**Useful overrides.**

- **`disable_catchup_globally`** for the next N hours. Useful when recovery itself is fragile and any extra load is dangerous.
- **`force_skip` for job X**. The scheduler advances `next_fire_at` to the next future occurrence and discards the missed window. Useful when the operator knows the missed fires are now meaningless (the marketing window passed; the partner's API was down too).
- **`force_catch_up` for job X with explicit window**. "Run every hourly fire from 23:00 to 02:00 even though the policy was Coalesce." Useful for regulated jobs that *must* produce per-window output.
- **Pause the schedule entirely** for job X without losing the watermark. Resume later picks up from the last successful fire.

```sql
-- Example operator-facing controls
ALTER TABLE schedules ADD COLUMN paused_until timestamptz;
ALTER TABLE schedules ADD COLUMN catchup_override TEXT
  CHECK (catchup_override IN ('skip','catchup','coalesce') OR catchup_override IS NULL);
```

Make these controls reversible, audited, and time-bounded. A `catchup_override` left in place forever is a silent way to lose data. EventBridge Scheduler exposes a similar concept via `FlexibleTimeWindow` plus per-schedule pause; Airflow exposes `dag.is_paused` plus `clear` for catch-up; Temporal exposes schedule pause/unpause as first-class APIs.

## Worked Example: 2-Hour Outage of an Hourly Job

Walk through the three policies for a single hourly job. The scheduler is down 23:00 to 01:00 UTC; the cron is `0 * * * *` (top of every hour).

**State before outage.**
- `last_successful_fire_at = 22:00 UTC`
- `next_fire_at = 23:00 UTC`

**State at recovery (01:00 UTC).**
- `last_successful_fire_at = 22:00 UTC` (unchanged; nothing succeeded during outage)
- `next_fire_at = 23:00 UTC` (stale; the scheduler couldn't advance it)
- Missed fires: `[23:00, 00:00, 01:00]` (note: 01:00 is at the edge of the misfire threshold; assume threshold = 60s and recovery completes at 01:00:30, so 01:00 is "missed" by 30s — borderline; for this example assume yes)

**Policy: Catch-up (FIRE_ALL).**

| When | What runs | `fire_time` argument | Notes |
|---|---|---|---|
| 01:00:30 | Run 1 | 23:00 | Worker computes the 23:00 hourly bucket |
| 01:00:30 | Run 2 | 00:00 | Concurrency cap = 2; Run 3 waits |
| 01:00:35 | Run 3 starts when Run 1 finishes | 01:00 | |
| 02:00 | Steady-state Run 4 | 02:00 | Normal schedule resumes |

Total fires produced: 3 catch-up + 1 steady-state = 4 in the first hour after recovery. `last_successful_fire_at` ends up at 01:00 (or 02:00 after the next steady-state fire).

**Policy: Skip (DO_NOTHING).**

| When | What runs | `fire_time` |
|---|---|---|
| 01:00:30 | nothing | — |
| 02:00 | Steady-state Run 1 | 02:00 |

Total fires produced: 0 catch-up + 1 steady-state. `last_successful_fire_at` jumps from 22:00 directly to 02:00 with no intermediate fires recorded; the database carries an explicit `skipped_window` audit row noting `[23:00, 01:00]` was discarded.

**Policy: Coalesce (FIRE_ONCE_NOW).**

| When | What runs | `fire_time` | `coalesced_count` |
|---|---|---|---|
| 01:00:30 | Run 1 | 01:00 (or "now") | 3 |
| 02:00 | Steady-state Run 2 | 02:00 | — |

Total fires produced: 1 catch-up + 1 steady-state. The job sees `coalesced_count=3` and may widen its query window accordingly (e.g., a metric scraper grabs the last three hours instead of the last one). `last_successful_fire_at` advances to 01:00.

**What changes for shorter or longer outages?**

- **Outage of 5 minutes (sub-threshold).** Misfire threshold not crossed; the scheduler simply runs the late fire when it recovers. No policy applies because there is no misfire — just a delay.
- **Outage of 12 hours.** Catch-up produces 12 fires. Coalesce produces 1. The unbounded catch-up policy is dangerous here: if the worker fleet can run 4 hourly jobs concurrently and each takes 10 minutes, catching up takes 30 minutes. If the job takes longer than its period (1 hour), catch-up never completes. This is where bounded catch-up (`max_catchup_lookback = 6h`) becomes important.
- **Outage of 5 days.** Almost certainly want Skip or Coalesce regardless of the original policy. The semantic intent of "hourly" doesn't survive a multi-day gap; the operator should engage manually.

**Side-by-side accounting.** Concrete numbers for a single hourly job across the three policies, assuming the missed-fire scanner runs at 01:00:30 UTC:

| Aspect | Catch-up | Skip | Coalesce |
|---|---|---|---|
| Fires enqueued at recovery | 3 | 0 | 1 |
| Total work in first hour | 3 × hourly job duration | 0 | 1 × hourly job duration |
| `last_successful_fire_at` after recovery (assume all succeed) | 01:00 | 22:00 (unchanged until 02:00 fires) | 01:00 |
| Output partitions written | `[23:00, 00:00, 01:00]` | none for the gap | merged window `[22:00–01:00]` |
| Audit log entries | 3 `RUN` rows + 0 `SKIP` | 0 `RUN` rows + 1 `SKIPPED_WINDOW` for `[23:00, 01:00]` | 1 `RUN` row with `coalesced_count=3` |
| User-visible behavior | Three rapid fires | Silent gap (unless surfaced) | One fire, slightly heavier |

## How Real Systems Implement This

| System | Default | How to express catch-up | How to express skip | How to express coalesce |
|---|---|---|---|---|
| **Quartz** | `SMART_POLICY` (varies by trigger) | `MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY` or `FIRE_ALL_WITH_REMAINING_COUNT` | `MISFIRE_INSTRUCTION_DO_NOTHING` | `MISFIRE_INSTRUCTION_FIRE_ONCE_NOW` |
| **Apache Airflow** | `catchup_by_default = True` (DAG-level) | `catchup=True` plus `max_active_runs` | `catchup=False` | Not a first-class concept; closest equivalent is `catchup=False` with manual trigger |
| **Sidekiq-Cron** | Skip (no backfill by default) | Set `last_enqueue_time` in past + `enqueue` loop runs | Default behavior | N/A directly; emulate by single manual enqueue |
| **AWS EventBridge Scheduler** | Per-schedule `FlexibleTimeWindow` controls tolerance; missed deliveries beyond window are dropped | Schedule with `Mode=OFF` window and rely on at-least-once delivery within window | Default for past windows | Implicit: only one delivery per scheduled time |
| **Kubernetes CronJob** | Coalesce-ish (controlled by `startingDeadlineSeconds`) | Not native — set `startingDeadlineSeconds` very large + write idempotent jobs | `startingDeadlineSeconds: 0` or short | `concurrencyPolicy: Forbid` plus moderate `startingDeadlineSeconds` |
| **Temporal** | Schedule has `Catchup Window` plus `Overlap Policy` (`Skip`, `BufferOne`, `BufferAll`, `CancelOther`, `TerminateOther`, `AllowAll`) | `Overlap Policy = BufferAll` + long `Catchup Window` | Short `Catchup Window` (drops anything older) | `Overlap Policy = Skip` (only one runs; subsequent in-window are dropped) |

Two patterns to internalize from this table:

1. **Catch-up is opt-in or opt-out at different granularities.** Airflow makes it a DAG-level boolean (everything in the DAG inherits). Quartz makes it a per-trigger constant. Temporal makes it a richer per-schedule policy. Pick the granularity that matches how operators reason about your jobs; per-job is almost always right.
2. **The "delivery deadline" idea is widely useful.** EventBridge's `FlexibleTimeWindow`, Kubernetes' `startingDeadlineSeconds`, Temporal's `CatchupWindow` all encode "if we can't fire within X of the scheduled time, drop the fire." This collapses missed-fire policy to a single knob for many use cases — past-deadline is automatically Skip; within-deadline is automatically catch-up.

## Anti-Patterns

**1. Default catch-up for non-idempotent jobs.** A push-notification job set to `catch_up = true` because that's the platform default sends 24 hours of notifications in five minutes after an outage. Every notification is a separate user-visible event; users delete the app. Make Skip the default for any job that touches users; let operators opt into catch-up explicitly with an idempotency key in place.

**2. Unbounded catch-up rate.** Replaying 720 missed fires of a per-minute job in parallel takes the worker fleet down. Always cap concurrency; always offer bounded catch-up ("at most the last N occurrences"); always escalate to Coalesce if the missed count exceeds a sanity threshold.

**3. Skip without alerting.** A job is silently set to Skip; an outage drops 50 fires; nobody notices because the dashboard only shows successful runs. Skip is *fine* — silent skip is not. Emit an explicit `skipped_window` audit record for every skipped fire and an alert if the count exceeds a threshold (e.g., > 5 missed in any rolling 24h).

**4. Hardcoded global policy.** "Our scheduler always uses Coalesce" sounds clean until the regulated daily report is missed and the auditors ask why yesterday's report doesn't exist. Per-job policy is the right granularity.

**5. Catch-up that uses `now()` as `fire_time`.** A catch-up run for the 23:00 partition that sees `fire_time = 01:00` writes to the wrong partition. Always preserve the original scheduled time as `fire_time` for catch-up; pass a separate `actual_run_time` if the job needs both.

**6. Catch-up that ignores idempotency.** Replay without `(job_id, fire_time)` dedup means two scheduler instances at startup both decide to enqueue the catch-up fires; the worker pool runs each twice. The distributed lock per job (see [./distributed-lock-per-job.md](./distributed-lock-per-job.md)) defends this — but it must be in place *before* catch-up is enabled, not bolted on after a duplicate-charge incident.

**7. Misfire threshold tuned too low.** A 5-second threshold treats every transient GC pause or network hiccup as a misfire. The scheduler aggressively applies the misfire policy, generating spurious extra fires (catch-up) or spurious skips. Default to 60 seconds; tune up for low-frequency jobs that can tolerate looser timing.

**8. Misfire threshold tuned too high.** A 30-minute threshold means a real 10-minute outage of a per-minute job is treated as "running late," producing no catch-up and no skip — fires just lag indefinitely. Match the threshold to the natural job period: roughly 1× period for slow jobs, 5–10× period for fast jobs.

**9. Detection that ignores schedule edits.** A scheduler that stores `next_fire_at` and crashes; the operator edits the cron expression; the scheduler restarts and uses the *old* `next_fire_at` to compute missed fires. Use the watermark + current cron expression instead.

**10. Conflating retries with catch-up.** A run that fails and is retried by the scheduler is *not* a missed fire. Retries are governed by the retry policy (see [./retry-and-idempotency.md](./retry-and-idempotency.md)); missed-fire policy applies only to fires that never started.

**11. Catch-up storming the leader-election.** A scheduler that detects 500 missed fires, enqueues them all in a single transaction, blocks the database for 30 seconds, and loses its leader lease while doing so. Batch the enqueue (commit every 100 fires); checkpoint progress; renew the leader lease in between batches.

**12. Coalesce that loses the count.** A coalesced fire records `fire_time = now()` and `coalesced_count = null`. Downstream surveillance ("did we eat any fires?") can't tell whether one fire or one hundred were collapsed. Always record `coalesced_count` and the `[start, end]` window the coalesced fire represents.

**13. Cross-region catch-up double-fire.** Two regions independently detect missed fires after a partition heals; both enqueue. The per-job distributed lock catches the second enqueue, but only if it's keyed on `(job_id, fire_time)` — a lock keyed on `job_id` alone permits both enqueues to register and the worker fleet to fight over them. Key locks on `(job_id, fire_time)` for catch-up correctness.

**14. Operator override left enabled.** During an incident, ops sets `disable_catchup_globally`; incident resolves; nobody clears the flag. A real outage three weeks later silently drops missed fires. Time-bound the override (`disable_until = now() + 4h`); auto-clear when the timer expires; alert before clearing so ops can extend if needed.

## Related

- [Time-Zone Correctness](./time-zone-correctness.md) — DST transitions, IANA tzdata, and how `next_cron_occurrence` must work for catch-up enumeration to be correct.
- [Retry and Idempotency](./retry-and-idempotency.md) — the property that makes catch-up safe; the boundary between "the scheduler is replaying" and "the worker is retrying."
- [Distributed Lock Per Job](./distributed-lock-per-job.md) — the lock on `(job_id, fire_time)` that prevents catch-up from double-enqueuing across multiple scheduler instances.
- [Leader Election for Scheduler](./leader-election-for-scheduler.md) — how a single scheduler is responsible for catch-up at any one time; why failover triggers a fresh missed-fire scan.
- [Design a Job Scheduler (parent)](../design-job-scheduler.md) — full case study with all subsections.
- [Disaster Recovery](../../../reliability/disaster-recovery.md) — the broader pattern for cross-region recovery in which catch-up is one specific concern.

## References

- [Quartz — Misfire Instructions for CronTrigger (Tutorial Lesson 04)](https://www.quartz-scheduler.org/documentation/quartz-2.3.0/tutorials/tutorial-lesson-04.html) — the canonical write-up of `FIRE_ONCE_NOW`, `DO_NOTHING`, and the smart policy.
- [Quartz CronTrigger Javadoc — misfire constants](https://www.quartz-scheduler.org/api/2.3.0/org/quartz/CronTrigger.html) — formal constant definitions used by every Quartz-derived scheduler.
- [Apache Airflow — DAG runs and `catchup`](https://airflow.apache.org/docs/apache-airflow/stable/dag-run.html#catchup) — Airflow's per-DAG catch-up flag and how `max_active_runs` rate-limits replay.
- [Apache Airflow — Authoring and scheduling concepts](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/index.html) — the `data_interval_end` watermark model that drives Airflow's catch-up correctness.
- [Kubernetes CronJob — `startingDeadlineSeconds` and `concurrencyPolicy`](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) — Kubernetes' built-in delivery deadline plus concurrency control.
- [AWS EventBridge Scheduler — Getting started and `FlexibleTimeWindow`](https://docs.aws.amazon.com/scheduler/latest/UserGuide/getting-started.html) — managed scheduler with at-least-once delivery, flexible windows, and one-shot semantics.
- [Sidekiq Cron — scheduled jobs](https://github.com/sidekiq/sidekiq-cron) — Ruby/Sidekiq cron implementation; pragmatic examples of skip-by-default with optional manual backfill.
- [Temporal — Cron and schedule overlap policies](https://docs.temporal.io/cron) — Temporal's first-class schedule API with `Catchup Window` and `Overlap Policy` (`Skip`, `BufferOne`, `BufferAll`, `CancelOther`, `TerminateOther`, `AllowAll`).
