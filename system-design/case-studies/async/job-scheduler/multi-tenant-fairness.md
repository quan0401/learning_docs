---
title: "Multi-Tenant Fairness — Weighted Fair Queueing, Quotas, and Noisy-Neighbor Defense"
date: 2026-05-01
updated: 2026-05-01
tags: [system-design, deep-dive, scheduler, fairness, multi-tenancy]
---

# Multi-Tenant Fairness — Weighted Fair Queueing, Quotas, and Noisy-Neighbor Defense

**Date:** 2026-05-01 | **Updated:** 2026-05-01
**Tags:** `system-design` `deep-dive` `scheduler` `fairness` `multi-tenancy`

> **Parent case study:** [`../design-job-scheduler.md`](../design-job-scheduler.md). This is the deep-dive companion for **§7.3 Multi-tenant fairness and weighted fair queueing**. The parent doc explains the problem in two paragraphs and shows the WFQ formula. This doc gets concrete about how the algorithm works, what the cheaper approximations are, where they fit in a queue topology, what quotas do that fairness alone cannot, and what fails under load when you skip any of it.

## Table of Contents

- [Summary](#summary)
- [The Noisy-Neighbor Problem](#the-noisy-neighbor-problem)
- [Two Distinct Tools — Quotas and Fairness](#two-distinct-tools--quotas-and-fairness)
- [Per-Tenant Quotas](#per-tenant-quotas)
  - [Concurrency Cap](#concurrency-cap)
  - [Rate Cap (QPS)](#rate-cap-qps)
  - [CPU-Minutes-per-Hour Budget](#cpu-minutes-per-hour-budget)
  - [Reserved Capacity vs Burst Pool](#reserved-capacity-vs-burst-pool)
- [Weighted Fair Queueing — Virtual Time](#weighted-fair-queueing--virtual-time)
  - [Why Virtual Time Works](#why-virtual-time-works)
  - [WFQ Pseudocode](#wfq-pseudocode)
  - [Cost and Approximations](#cost-and-approximations)
- [Deficit Round Robin](#deficit-round-robin)
  - [DRR Pseudocode](#drr-pseudocode)
- [Stride Scheduling](#stride-scheduling)
- [Earliest Deadline First](#earliest-deadline-first)
- [Hierarchical Fairness](#hierarchical-fairness)
- [Priority Classes](#priority-classes)
  - [Strict Priority and Starvation](#strict-priority-and-starvation)
  - [Priority Inversion](#priority-inversion)
- [Where Fairness Lives — Queue Layer vs Worker Layer](#where-fairness-lives--queue-layer-vs-worker-layer)
- [Backpressure and Admission Control](#backpressure-and-admission-control)
- [Per-Tenant Token Bucket — Code](#per-tenant-token-bucket--code)
- [Tenant Tagging and Per-Tenant Lag](#tenant-tagging-and-per-tenant-lag)
- [Throttling Under SLA Breach](#throttling-under-sla-breach)
- [Worked Example — 3 Tenants, Weights 4:2:1](#worked-example--3-tenants-weights-421)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

A multi-tenant job scheduler has one resource — worker capacity — that every tenant wants to consume at once. Without explicit fairness, the queue is FIFO and the loudest submitter wins: a single tenant who enqueues 10,000 jobs at midnight starves every other tenant until those 10,000 drain. The fix is two tools applied together. **Per-tenant quotas** cap the absolute resource a tenant can consume — concurrent slots, queries-per-second, CPU-minutes-per-hour — and they reject or defer work above the cap regardless of who else is on the system. **Weighted fair queueing** governs how the remaining capacity is split among tenants who all have work and are all under their quotas; classic WFQ tags each job with a virtual finish time `(max(virtual_clock, last_finish[t]) + size / weight[t])` and dispatches the smallest virtual finish first, giving each tenant a share proportional to its weight. WFQ is `O(log n)` per dispatch with a heap; cheaper variants — Deficit Round Robin (`O(1)`), Stride Scheduling, and Lottery Scheduling — sacrifice small amounts of fairness for big constant-factor wins. Above this sit hierarchical fairness (per-org, then per-team, then per-job within team), priority classes (`critical` / `standard` / `best-effort` with strict ordering or weighted blends), and EDF when SLAs trump fairness. Around it sit backpressure (reject when the global queue exceeds threshold), tenant tagging (every job carries `tenant_id` so an aggregator can report per-tenant lag), and throttling (cut a tenant's share when a higher-priority tenant's SLA is at risk). Skipping any layer produces a familiar set of outages — single FIFO under load, per-tenant queue with no weighting, priority inversion where a low-priority job blocks a high-priority job, and unbounded growth of per-tenant queues that turns the scheduler into a memory bomb.

## The Noisy-Neighbor Problem

Concretely: at 23:59:59 the marketing team's nightly export job enqueues 1,000,000 row-level email jobs. The queue is a single FIFO. The on-call team's database-backup job, scheduled for 00:00:00, gets enqueued behind those million emails. Workers drain the queue in order. The backup waits 47 minutes, misses its SLA, and the next morning the on-call team files an incident.

The scheduler did exactly what FIFO promises. The problem is that FIFO is not the right contract for a multi-tenant system. Three properties matter:

1. **Per-tenant isolation.** A tenant must not be able to consume more resource than its quota, even if the rest of the system is idle.
2. **Fair share under contention.** When multiple tenants want resource, each gets a share proportional to its configured weight. No tenant is starved indefinitely.
3. **Bounded latency for high-priority work.** Critical jobs (security alerts, billing close) skip the line, but with bounded preemption so they cannot lock out the rest of the system.

A single FIFO queue gives none of these. A per-tenant FIFO with naive round-robin gives the first two but ignores priority. Real schedulers stack quotas, WFQ, priority, and EDF until all three properties hold.

## Two Distinct Tools — Quotas and Fairness

Conflating these two is the most common design mistake.

| Tool | Question it answers | Triggered by |
|---|---|---|
| **Quota** | "May tenant T have any more resource at all right now?" | Tenant's own consumption hitting a configured ceiling |
| **Fairness** | "Among tenants who currently want resource and are under their quotas, who goes next?" | Multiple eligible tenants competing for one slot |

Quotas are absolute — they reject or defer regardless of system load. Fairness is relative — it only matters when there is contention. A system with quotas only is fine when load is low but starves on busy days. A system with fairness only lets a tenant with high weight monopolize the system if no one else has work, then keeps starving the rest when they show up because they are still rebuilding their virtual-time deficit.

You need both. Quotas keep the worst case bounded; fairness keeps the average case proportional.

## Per-Tenant Quotas

### Concurrency Cap

The simplest cap: tenant `acme` cannot have more than `N` jobs running concurrently. Enforced at dispatch: when the loop picks a candidate row, it checks `running_count[acme] < N`. If false, the row is left as `WAITING` with `next_fire_at` bumped by a small jitter (10–30 s) to avoid hot-spinning the dispatcher.

This is what AWS Lambda calls **reserved concurrency** at the function level and **account concurrency** at the account level. Lambda's default account quota is 1,000 concurrent executions; functions with reserved concurrency get a guaranteed slice that no other function can consume, with the rest pooled as burst.

Concurrency cap is the most useful single quota because it tracks the dimension that actually maps to a finite resource (worker slots, DB connections, downstream-service capacity). A QPS cap doesn't help if each request takes 30 s — you still have hundreds in flight.

### Rate Cap (QPS)

Tenant `acme` cannot dispatch more than `M` jobs per second. Implemented as a per-tenant token bucket: tokens accrue at rate `M/sec`, capped at `burst`. Each dispatch consumes a token; if the bucket is empty, the row is deferred.

QPS caps protect downstream systems from rate-sensitive load — a third-party API with a 100 req/s ceiling, a database that thrashes above a write rate, a payment gateway with an enforced TPS contract. They do not protect against long-tail work; for that, use concurrency.

### CPU-Minutes-per-Hour Budget

A coarser, longer-window quota: tenant `acme` can consume at most `K` CPU-minutes in any rolling hour. Each completed job adds its measured CPU time to a per-tenant counter; once `K` is hit, dispatch defers all `acme` jobs until the rolling window relieves pressure.

This is what Kubernetes `ResourceQuota` does for namespaces — caps on CPU, memory, GPU, and pod count. It is the only quota that pushes back on jobs that are individually fine but expensive in aggregate. The trade-off is windowing complexity: rolling windows, fixed-window resets, and token-bucket-over-CPU all give different burst characteristics, and operators must pick one and document it.

### Reserved Capacity vs Burst Pool

A common pattern: split worker capacity into a **reserved** slice and a **burst** pool.

| Slice | Allocation | Behavior |
|---|---|---|
| Reserved | Fixed per-tenant; sum ≤ total capacity minus headroom | Each tenant always has its reserved slots available; never preempted |
| Burst | The remainder | Best-effort; allocated by WFQ among tenants with work |

A tenant with reserved 10 + access to burst gets 10 always plus its WFQ share of the rest. A tenant with reserved 0 only gets burst; if the system is busy, that tenant runs slowly or not at all.

Reserved capacity is what AWS Lambda's reserved concurrency does; the unreserved pool is the burst. Kubernetes uses this pattern via `ResourceQuota` (reserved per namespace) plus cluster-wide best-effort scheduling for jobs without quota.

## Weighted Fair Queueing — Virtual Time

WFQ comes from packet scheduling — Demers, Keshav, and Shenker (SIGCOMM 1989) introduced it to give each TCP flow on a router a fair share of bandwidth without head-of-line blocking. The math is identical for jobs.

Each tenant has a **weight** `w[t]` (an integer or rational; ratios are what matter). Each job is tagged with a **virtual finish time** when it is enqueued:

```text
virtual_finish_time(job_j) = max(virtual_clock, last_finish[t]) + size(job_j) / w[t]
```

where:
- `virtual_clock` is the global virtual time, advanced at the rate of work being completed.
- `last_finish[t]` is the virtual finish time of tenant `t`'s previously enqueued job.
- `size(job_j)` is the job's expected cost (1 if all jobs are unit-size; CPU-seconds or rows-processed in finer-grained models).

The dispatcher always picks the job with the **smallest virtual finish time** across all eligible tenants. A tenant with weight 2 has its virtual finish times advance half as fast per unit of work as a tenant with weight 1, so it gets twice the dispatch rate.

### Why Virtual Time Works

The classical fairness goal is **max-min fair share**: every tenant gets at least its proportional share, and no tenant gets more without first satisfying every tenant that wants less. Virtual finish time encodes this directly — the smallest virtual finish time is the tenant most "behind" its share.

The proof in Demers-Keshav-Shenker shows that WFQ approximates Generalized Processor Sharing (GPS) — an idealized scheduler that runs all flows simultaneously at proportional rates — with a bounded gap of one packet (one job). For our purposes: at any moment, no tenant is more than one job-time behind where the ideal would put it.

This is stronger than round-robin: round-robin gives each tenant equal turns, which is fair only if all tenants have equal weight and all jobs are equal size. WFQ handles weights and variable sizes correctly.

### WFQ Pseudocode

```pseudo
struct Tenant:
    weight: float        # configured share, e.g. 1.0, 2.0, 4.0
    last_finish: float   # virtual finish time of last enqueued job (init 0)

global virtual_clock: float = 0
global ready_heap: MinHeap<(virtual_finish_time, job_id)>

# On enqueue:
function enqueue(job, tenant):
    vft = max(virtual_clock, tenant.last_finish) + job.size / tenant.weight
    tenant.last_finish = vft
    job.virtual_finish_time = vft
    ready_heap.push((vft, job.id))

# On dispatch (when a worker is free):
function dispatch():
    if ready_heap.empty():
        return None
    (vft, job_id) = ready_heap.pop()
    job = load_job(job_id)
    # Advance virtual clock to this job's start (the smallest VFT in the heap).
    virtual_clock = vft - (job.size / tenant_of(job).weight)
    return job

# On completion (only matters if the worker pool is the bottleneck — virtual
# clock is driven by dispatch, not by completion, in most implementations).
```

`O(log n)` per enqueue and per dispatch with a heap. For a scheduler with 10⁵ ready jobs, that is ~17 comparisons per operation — negligible compared to the DB hop.

A subtle point: `virtual_clock` does not advance with wall-clock time. It advances **as work is dispatched**. A tenant that goes idle does not "build up credit" by accumulating wall-clock seconds while inactive; when it returns, its `last_finish` is reset to `max(virtual_clock, last_finish)`, which catches it up to the current virtual time. This is the property that prevents a returning idle tenant from monopolizing the system as it "catches up" — a real bug in naive implementations.

### Cost and Approximations

WFQ's `O(log n)` dispatch is fine at moderate scale. At very high scale (millions of ready jobs, millions of dispatches per second) the heap becomes a hot point. Three approximations:

| Algorithm | Cost | Fairness gap | Use case |
|---|---|---|---|
| WFQ (heap) | `O(log n)` | Within one job | Default; fine to ~10⁶ ready jobs |
| Deficit Round Robin (DRR) | `O(1)` amortized | Within one quantum | Network-line-rate switches; very high dispatch rates |
| Stride Scheduling | `O(log n)` | Within one stride | Process scheduling; deterministic |
| Lottery Scheduling | `O(1)` per ticket | Probabilistic; converges | Trivially mergeable; fault tolerant |

For job schedulers, WFQ or DRR is the right answer for almost everyone. Stride and lottery come from OS process scheduling (Waldspurger & Weihl, OSDI 1994); their main virtue in that domain is that they are deterministic without the heap.

## Deficit Round Robin

DRR (Shreedhar & Varghese, SIGCOMM 1995) approximates WFQ with `O(1)` per dispatch. Instead of computing virtual finish times, it gives each tenant a **deficit counter** that accumulates a **quantum** per round. A tenant can dispatch any job whose size fits in its current deficit; size is subtracted on dispatch.

The trick: tenants with higher weight get larger quanta, so they accumulate dispatch credit faster. Variable-size jobs are handled correctly because the deficit carries forward.

### DRR Pseudocode

```pseudo
struct Tenant:
    weight: int
    quantum: int           # bytes/jobs/cost-units per round; ∝ weight
    deficit: int = 0
    queue: FIFO<Job>

global active_list: List<Tenant>      # tenants with non-empty queues

function enqueue(job, tenant):
    tenant.queue.push(job)
    if tenant not in active_list:
        active_list.append(tenant)
        tenant.deficit = 0   # reset on (re-)activation

function dispatch():
    while active_list not empty:
        tenant = active_list[0]
        tenant.deficit += tenant.quantum

        while tenant.queue not empty:
            head = tenant.queue.peek()
            if head.size <= tenant.deficit:
                tenant.deficit -= head.size
                tenant.queue.pop()
                return head    # dispatch this job
            else:
                break          # not enough deficit; move on

        if tenant.queue.empty():
            tenant.deficit = 0
            active_list.remove(tenant)
        else:
            active_list.rotate_left(1)   # next tenant's turn

    return None
```

The quantum sizing matters: pick a quantum at least as large as the maximum job size, or DRR can dispatch zero jobs in a round and stall. With `quantum >= max_job_size`, every active tenant dispatches at least one job per round, and the long-run share converges to `weight[t] / sum(weights)`.

DRR's appeal in the scheduler context: a single integer (`deficit`) per tenant, plus a circular list of active tenants. No heap. Constant per-dispatch work. Cisco line cards have used DRR at line rate for decades; the scheduler use case is far easier.

The fairness gap is bounded by `max_job_size`. For a typical scheduler with jobs in the seconds-to-minutes range, the gap is comparable to one job, identical to WFQ in practice.

## Stride Scheduling

Waldspurger & Weihl's stride scheduler computes a **stride** per tenant: `stride[t] = K / weight[t]` for some constant `K`. Each tenant has a **pass** counter starting at its stride. The dispatcher picks the tenant with the smallest pass, dispatches its job, and advances its pass by its stride.

This is mathematically equivalent to WFQ with unit-size jobs, but explicitly deterministic — the same set of weights and arrivals always produces the same dispatch order. Useful in process scheduling where reproducibility matters; less of a draw in job schedulers, where DB persistence already gives reproducibility.

Lottery scheduling is the probabilistic sibling: each tenant gets `weight[t]` lottery tickets; the dispatcher draws one ticket uniformly at random and dispatches that tenant's next job. Long-run share converges to `weight[t] / sum(weights)`. The two algorithms compose naturally — currency, transferable tickets, and ticket inflation give a clean policy language.

## Earliest Deadline First

When **deadline matters more than fairness**, the right answer is EDF: dispatch the job with the earliest deadline. If a billing-close job must finish by 09:00 and a marketing-export job has no deadline, EDF picks the billing job regardless of tenant weights.

EDF is optimal for single-resource real-time scheduling: if any policy can meet all deadlines, EDF will. The downside: it ignores fairness. A tenant that always submits jobs with tight deadlines drains the system. The mitigation is **deadline policing** — reject or de-prioritize jobs whose deadline-tightness exceeds the tenant's quota.

The pragmatic blend: WFQ for the steady-state, with EDF override for jobs marked `priority=critical` and `deadline < threshold`. Most jobs flow through fairness; a small set bypass on deadline. This is what Linux's CFS does for `SCHED_DEADLINE` tasks layered over the normal fair scheduler.

## Hierarchical Fairness

Real systems have nested tenants: an organization contains teams; a team contains projects; a project contains job kinds. Single-level WFQ doesn't capture this — if `team-a` has 100 jobs and `team-b` has 1, flat WFQ gives equal share, but if both teams are inside `org-acme` and a different `org-globex` has 1 job, you want `acme` and `globex` to share equally first, then `team-a` / `team-b` to split `acme`'s share.

Hierarchical WFQ runs the algorithm at each level:

```
                     root
                   /      \
              org-acme    org-globex   (weights 1:1)
             /        \
        team-a    team-b               (weights 1:1)
        /    \
     job-x   job-y                     (weights 1:1)
```

Dispatch picks the smallest virtual finish time at the root, then at the org level, then at the team level, then the actual job. Each level maintains its own virtual clock and per-child `last_finish`.

Cost: one heap per internal node; `O(depth × log fan-out)` per dispatch. For trees with depth ≤ 4 and modest fan-out, this is still fast.

The alternative — flatten to a single level by computing effective weights from the tree (`effective_weight[job] = weight[org] × weight[team] × weight[job]`) — is simpler but loses the property that an idle sibling's share goes to its parent's other children, not to other organizations.

## Priority Classes

Priority is orthogonal to fairness. A tenant has a weight (its share); a job has a priority class (its lane).

Common shape:

| Class | Latency target | Resource share | Preemption |
|---|---|---|---|
| `critical` | seconds | Reserved capacity | May preempt `best-effort` |
| `standard` | minutes | Burst pool, WFQ-shared | No preemption |
| `best-effort` | hours | Whatever's left | Preemptable |

This is what Kubernetes `PriorityClass` encodes: pods of higher priority can preempt lower-priority pods, and the scheduler picks higher-priority work first.

### Strict Priority and Starvation

Strict priority — always run higher-priority before lower — is simple and starves the low end. A steady stream of `standard` work blocks every `best-effort` job forever.

The fix is **age-boost**: every minute, increment the priority of any `WAITING` job older than a threshold:

```sql
UPDATE jobs
SET priority = LEAST(priority + 1, MAX_PRIORITY)
WHERE status = 'WAITING'
  AND created_at < now() - interval '5 minutes';
```

Eventually the oldest `best-effort` job's effective priority equals `standard`, and it dispatches. The aging rate is the knob: too fast and priority is meaningless; too slow and `best-effort` jobs starve under sustained load. Tune to "the oldest best-effort job can wait at most N hours" and back-solve.

Linux's CFS solves this differently: there is no strict priority — `nice` values map to weights in a single WFQ-shaped scheduler. A `nice -19` task gets ~88% share against a `nice 0` task, but the `nice 0` task is never starved; it always gets a small slice. This is a more elegant model when "priority" really means "weight".

### Priority Inversion

A `critical` job is blocked waiting for a resource (a distributed lock, a row, a downstream service) held by a `best-effort` job. The `best-effort` job is itself starved by `standard` work. So the `critical` job waits indefinitely on a job that is being denied CPU.

This is **priority inversion**, and it is the most pernicious failure mode in priority-based schedulers. The classic example is the Mars Pathfinder bug — a low-priority task held a mutex that a high-priority task wanted, while a medium-priority task starved the low-priority one.

The fix is **priority inheritance**: when a high-priority job blocks on a lock held by a low-priority job, the low-priority job's effective priority is temporarily boosted to the highest waiter's priority. The boost lasts only while the lock is held.

In a job scheduler, priority inheritance applies to:
- Distributed locks (per-job locks; see [`./distributed-lock-per-job.md`](./distributed-lock-per-job.md)).
- Database row locks held during the dispatch transaction.
- Downstream rate-limit tokens (a `critical` job waiting on a tenant's token bucket should get priority on the next available token).

Implementing inheritance for distributed locks is hard — the boost has to flow across machines. A common simplification: critical jobs do not share locks with best-effort jobs (separate lock namespaces), so inversion cannot occur. This is the "isolation by class" approach: each priority class has its own locks, queues, and worker pool, and they only meet at shared external resources where contention is bounded.

## Where Fairness Lives — Queue Layer vs Worker Layer

There are two architectures for enforcing fairness:

**Queue-layer fairness.** Multiple physical or logical queues — one per tenant, or one per priority class. A dispatcher round-robins (or WFQs) across queues, pulling the next job from the chosen queue.

```
[tenant-A queue] ─┐
[tenant-B queue] ─┤── dispatcher (WFQ across queues) ── workers
[tenant-C queue] ─┘
```

This is what RabbitMQ + per-queue consumers, AWS SQS + per-queue lambda triggers, and Kafka + per-topic consumer groups give you. The fairness logic is in the dispatcher; queues are dumb FIFOs.

**Worker-layer fairness.** A single shared queue with tenant-tagged jobs. Workers pull, but the dispatch choice (which job to pick from the candidate set) embeds the WFQ decision.

```
[shared queue, tenant-tagged] ── dispatcher (WFQ over candidates) ── workers
```

This is what Sidekiq Pro/Enterprise's rate limiter pattern looks like, and what a SQL-backed scheduler does naturally — the dispatch query selects with `ORDER BY virtual_finish_time` and `LIMIT N`.

The two compose: queue-layer for coarse isolation (one queue per priority class), worker-layer WFQ within each queue for tenant fairness. This is a common production shape.

The trade-off:

| Architecture | Pros | Cons |
|---|---|---|
| Queue-layer | Each queue isolates failure (poison job in tenant-A doesn't block tenant-B); easy to reason about | Queue count grows with tenant count; head-of-line blocking within a queue if jobs are unevenly sized |
| Worker-layer | Single queue scales sublinearly with tenant count; precise fairness control | Tenant tagging must be reliable; one bad tenant can fill the queue's storage |

The hybrid — one queue per priority class, WFQ across tenants within each queue — is the right default for systems above ~100 tenants.

## Backpressure and Admission Control

Fairness governs **how** work is dispatched. Backpressure governs **whether** work is accepted at all.

When the global ready queue exceeds a threshold, the scheduler must push back. Three policies:

1. **Reject new submissions.** Return `429 Too Many Requests` (or equivalent) to the API. The submitter retries with backoff.
2. **Defer enqueue.** Accept the request but stamp `next_fire_at = now + delay`. The job will dispatch when the system has room.
3. **Tenant-scoped backpressure.** Reject only the tenant who is over their quota; other tenants continue to accept normally.

Tenant-scoped backpressure is the right default. A misbehaving tenant gets `429`s for its own submissions while the rest of the system runs fine. The trigger is the per-tenant quota, not the global queue depth — global thresholds catch only systemic overload.

Combined with a **per-tenant queue cap**: tenant `acme` can have at most 50,000 `WAITING` jobs at any moment. Beyond that, new submissions are rejected. This bounds memory growth — without it, a tenant's queue can grow unboundedly while waiting on a downstream rate limit, which is a memory bomb.

See [`../../../scalability/backpressure-bulkhead-circuit-breaker.md`](../../../scalability/backpressure-bulkhead-circuit-breaker.md) for the general pattern.

## Per-Tenant Token Bucket — Code

The simplest admission control: a token bucket per tenant. Each submission consumes a token; if the bucket is empty, reject.

```pseudo
struct TokenBucket:
    capacity: int            # max tokens (burst size)
    refill_rate: float       # tokens per second
    tokens: float
    last_refill: timestamp

global buckets: map<tenant_id, TokenBucket>

function try_consume(tenant_id, cost = 1) -> bool:
    bucket = buckets[tenant_id]
    now = wall_clock()

    # Refill based on time elapsed since last check.
    elapsed = (now - bucket.last_refill).seconds
    bucket.tokens = min(bucket.capacity, bucket.tokens + elapsed * bucket.refill_rate)
    bucket.last_refill = now

    if bucket.tokens >= cost:
        bucket.tokens -= cost
        return true
    return false

# At submission time:
function on_submit(job, tenant_id):
    if not try_consume(tenant_id, cost = job.size):
        return REJECT(reason = "tenant rate limit exceeded",
                      retry_after = estimate_refill_time(tenant_id))
    insert_into_db(job)
    return ACCEPT(job_id)
```

Distributed implementation: store `(tokens, last_refill)` in Redis with a Lua script that does the refill-and-consume atomically. The Lua script is what makes this safe across multiple submission API instances. See [`../../../building-blocks/rate-limiters.md`](../../../building-blocks/rate-limiters.md) for the production-grade Redis token-bucket implementation.

The cost parameter matters for variable-size jobs: a 1-row email costs 1 token; a 100,000-row export costs 100,000 tokens. This collapses concurrency limits and rate limits into a single mechanism — the bucket size is the burst, the refill is the steady-state rate.

## Tenant Tagging and Per-Tenant Lag

Every job row in the scheduler's `runs` table carries a `tenant_id`. This is non-optional — it is the foundation for every fairness, quota, and observability decision downstream.

The aggregator computes per-tenant lag continuously:

```sql
SELECT
  tenant_id,
  COUNT(*) FILTER (WHERE status = 'WAITING')                          AS waiting,
  COUNT(*) FILTER (WHERE status = 'RUNNING')                          AS running,
  EXTRACT(epoch FROM now() - MIN(next_fire_at) FILTER (WHERE status = 'WAITING'))
                                                                     AS oldest_wait_sec,
  AVG(EXTRACT(epoch FROM started_at - next_fire_at))
       FILTER (WHERE started_at > now() - interval '10 minutes')     AS avg_dispatch_lag
FROM runs
GROUP BY tenant_id;
```

This drives:

- **SLA dashboards.** Per-tenant `avg_dispatch_lag` and `oldest_wait_sec` are the operator's view of fairness in action.
- **Alerting.** If a high-priority tenant's `oldest_wait_sec` exceeds its SLA, page on-call. If a low-priority tenant's `waiting` exceeds its queue cap, the submission API should already be rejecting; alert if not.
- **Auto-throttling.** When tenant A's SLA is at risk, the scheduler can temporarily reduce tenant B's effective weight (next section).

The pattern is the standard observability triad: tag every event with the tenant, aggregate by tenant, surface per-tenant metrics. Without consistent tenant tagging, the rest of the multi-tenancy story is opaque.

## Throttling Under SLA Breach

When a high-priority tenant is missing its SLA because a low-priority tenant is consuming the burst pool, the scheduler can **dynamically throttle** the offender.

Mechanism:

1. The aggregator detects `tenant-A oldest_wait_sec > SLA[tenant-A]`.
2. Identify the largest consumers of burst capacity in the last window — say `tenant-X` is using 80% of burst.
3. Temporarily reduce `tenant-X`'s effective weight (e.g., halve it for 5 minutes).
4. Tenant-A's WFQ share rises proportionally; its lag should drop.
5. After 5 minutes, restore normal weight; if SLA still breached, escalate.

This is closed-loop control. The risk is oscillation: if the throttle is too aggressive, tenant-X's lag spikes and triggers its own SLA alert. Use a hysteresis band — throttle activates above one threshold, deactivates below a lower one — to prevent flapping.

A simpler mechanism is **priority promotion**: when tenant-A's oldest job ages past SLA - margin, boost its priority class from `standard` to `critical` for that job alone. The job jumps the line; tenant-X is not punished, only deferred for one slot. This avoids the throttling complexity but doesn't help if many of tenant-A's jobs are aging simultaneously.

## Worked Example — 3 Tenants, Weights 4:2:1

Three tenants — `acme`, `globex`, `initech` — with weights 4, 2, 1. Each enqueues jobs of unit size. The total capacity is `1 dispatch per virtual time unit`. We trace the first ~14 dispatches.

Initial state:
- `last_finish[acme] = 0`, `last_finish[globex] = 0`, `last_finish[initech] = 0`
- All three enqueue 100 jobs each at `virtual_clock = 0`.

Per the WFQ formula, `vft(acme.job_k) = k / 4`; `vft(globex.job_k) = k / 2`; `vft(initech.job_k) = k / 1`.

The first 100 jobs sorted by VFT:

| Dispatch # | Tenant | Job index k | VFT |
|---|---|---|---|
| 1 | acme | 1 | 0.25 |
| 2 | acme | 2 | 0.50 |
| 3 | globex | 1 | 0.50 |
| 4 | acme | 3 | 0.75 |
| 5 | acme | 4 | 1.00 |
| 6 | globex | 2 | 1.00 |
| 7 | initech | 1 | 1.00 |
| 8 | acme | 5 | 1.25 |
| 9 | acme | 6 | 1.50 |
| 10 | globex | 3 | 1.50 |
| 11 | acme | 7 | 1.75 |
| 12 | acme | 8 | 2.00 |
| 13 | globex | 4 | 2.00 |
| 14 | initech | 2 | 2.00 |

In 14 dispatches: `acme` got 8, `globex` got 4, `initech` got 2 — exactly 4:2:1. Tied VFTs (e.g., dispatches 5/6/7 all at VFT=1.00) are broken by a secondary key — typically tenant ID hash or arrival time — so the order is deterministic.

Continuing to 100 total dispatches: `acme` ≈ 57, `globex` ≈ 29, `initech` ≈ 14 (4:2:1 = 4/7 : 2/7 : 1/7 ≈ 57.1% : 28.6% : 14.3% of 100).

If `initech` then stops submitting, `acme` and `globex` split the freed share at 4:2 = 2:1, so `acme` gets ~67% and `globex` ~33% of subsequent capacity. When `initech` returns, its `last_finish` is reset to `max(virtual_clock, last_finish[initech])` — it does **not** get to "spend" the credit it accumulated while idle. Without this reset, an idle tenant returning would monopolize the system to "catch up" — the work-conservation bug WFQ explicitly avoids.

## Anti-Patterns

- **Single FIFO queue under multi-tenant load.** The textbook noisy-neighbor failure. One tenant's batch starves the rest. The minimum acceptable scheduler has at least per-tenant rate limits; FIFO without isolation is a beta-only design.
- **Per-tenant queue without weighting.** Round-robin across per-tenant queues gives equal share regardless of weight; a free tier and a paid tier with reserved capacity get the same dispatch rate. Add weights, or you have built equality without proportionality.
- **Priority inversion under shared locks.** A `critical` job blocked on a lock held by a `best-effort` job that is itself starved. Use priority inheritance, separate lock namespaces by priority class, or strict tier isolation.
- **Unbounded growth of per-tenant queues.** A tenant submits faster than its quota allows dispatch; the `WAITING` queue grows unboundedly. This is a memory bomb (in-memory queue) or a storage bomb (SQL-backed). Always cap queue depth per tenant; reject submissions above the cap.
- **Quota-only design with no fairness.** All tenants are under quota but compete for one slot. Without WFQ, dispatch is FIFO among the eligible — first-come wins, even if "first" is consistently the same tenant.
- **Fairness-only design with no quotas.** A tenant with weight 100 can monopolize the system if no one else has work, then when others arrive, the WFQ catch-up still gives that tenant a disproportionate slice for a while. Quotas cap absolute consumption; fairness governs relative shares.
- **Wall-clock-driven virtual time.** `virtual_clock` advancing with `time.now()` instead of with dispatch progress means an idle tenant accumulates "credit" while not running, and on return floods the system. Virtual time advances with work, not with wall time.
- **No age-boost on strict priority.** Best-effort jobs starve indefinitely under sustained higher-class load. Every priority scheduler must age low-priority jobs upward, or document an explicit "best-effort jobs may never run" SLA.
- **Tenant tag absent or unreliable.** If `tenant_id` can be missing, default-bucketed, or wrong on some jobs, the entire fairness story breaks. Tagging must be enforced at the API layer and validated; jobs without tenant ID are rejected, not assigned to a default bucket.
- **Burst pool sized larger than total capacity minus reserved.** Reserving 800 of 1,000 slots and a burst pool of 1,000 means oversubscription — when reserved tenants all show up, they cannot get their reservations because burst is consuming them. Burst = total - sum(reserved); never larger.
- **Hierarchical fairness flattened to product-of-weights.** `eff_weight = w_org × w_team × w_job` simplifies the dispatch but loses the property that an idle sibling's share goes to its parent's other children, not to other organizations. The flat model gives the wrong proportions in real trees.
- **Throttling without hysteresis.** Auto-throttling tenant X every time tenant A's SLA wobbles produces oscillation: throttle, X's lag spikes, unthrottle, A's lag returns. Use bands (high-water and low-water) and dwell times.
- **DRR with quantum smaller than max job size.** A round in which no tenant's deficit covers any pending job is a stall — the dispatcher loops without dispatching, burning CPU for no progress. Pick `quantum >= max_job_size` always.
- **Same priority class for "critical" infrastructure and "interactive" user actions.** They have different latency targets and different tolerance for preemption. A user-facing action waiting behind a critical-but-slow background job is a UX failure that looks like a scheduler bug.
- **Quota refill on a fixed window with synchronized boundary.** Token buckets that reset at the top of every minute give a thundering herd: every tenant that hit the cap submits at HH:00 and the dispatcher faces a cliff of work. Use rolling windows or jittered refill instead of fixed boundaries.
- **No SLO per priority class.** "We have priorities" is meaningless without "critical jobs dispatch within 10s p99, standard within 5min p99, best-effort within 4hr p99." The SLO is what drives weight tuning, alert thresholds, and autoscaling decisions.

## Related

- [`./retry-and-idempotency.md`](./retry-and-idempotency.md) — retries amplify load; a misbehaving handler that retries 10× is effectively a 10× weight multiplier and breaks fairness math unless retries count against quota.
- [`./distributed-lock-per-job.md`](./distributed-lock-per-job.md) — per-job locks are the resource where priority inversion shows up; the lock acquisition path needs priority inheritance or separated namespaces.
- [`./leader-election-for-scheduler.md`](./leader-election-for-scheduler.md) — a single dispatcher runs WFQ; if dispatch is sharded across leaders, fairness becomes per-shard and global proportions need a coordination layer.
- [`../design-job-scheduler.md`](../design-job-scheduler.md) — parent case study; §7.3 motivates this whole layer in two paragraphs.
- [`../../../building-blocks/rate-limiters.md`](../../../building-blocks/rate-limiters.md) — the production-grade token-bucket implementation used for per-tenant admission; the same primitive that backs the QPS quota.
- [`../../../scalability/backpressure-bulkhead-circuit-breaker.md`](../../../scalability/backpressure-bulkhead-circuit-breaker.md) — backpressure as a global-queue-depth control that complements per-tenant quotas; bulkheads as the worker-pool isolation that priority classes typically map to.

## References

- A. Demers, S. Keshav, S. Shenker, ["Analysis and Simulation of a Fair Queueing Algorithm"](https://dl.acm.org/doi/10.1145/75247.75248) — the original WFQ paper from SIGCOMM 1989; the virtual-time formulation used throughout this doc.
- M. Shreedhar, G. Varghese, ["Efficient Fair Queueing using Deficit Round Robin"](http://www.cs.unc.edu/~jeffay/courses/nidsS05/papers/Shreedhar-DRR.pdf) — SIGCOMM 1995; the `O(1)` WFQ approximation that production line-rate switches use, applicable to job dispatch at high rates.
- C. A. Waldspurger, W. E. Weihl, ["Lottery Scheduling: Flexible Proportional-Share Resource Management"](https://www.usenix.org/legacy/publications/library/proceedings/osdi/full_papers/waldspurger.pdf) — OSDI 1994; lottery and stride scheduling, with the policy-language properties (currency, transferable tickets) that translate cleanly to multi-tenant job scheduling.
- AWS, [Lambda concurrency](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html) — reserved vs unreserved concurrency, account quotas, and burst behavior; production reference for the reserved/burst split discussed in this doc.
- Linux kernel, [CFS — Completely Fair Scheduler](https://docs.kernel.org/scheduler/sched-design-CFS.html) — `vruntime` is virtual time, `nice` values map to weights; the OS kernel-side implementation of the same WFQ math, with red-black tree replacing the heap.
- Kubernetes, [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) — namespace-scoped quotas on CPU, memory, GPU, and object counts; the production reference for the CPU-minutes-per-hour budget pattern.
- Kubernetes, [PriorityClass and Pod Priority/Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) — priority classes with preemption semantics; the production reference for the `critical`/`standard`/`best-effort` shape.
- Sidekiq, [Pro/Enterprise Rate Limiting](https://github.com/sidekiq/sidekiq/wiki/Ent-Rate-Limiting) — concurrent and bucket-based limiters as a reference implementation of per-tenant quotas in a Ruby job-scheduler ecosystem.
- L. Kleinrock, _Queueing Systems, Vol. 1: Theory_ — textbook treatment of fair queueing, max-min fair share, and the GPS model that WFQ approximates; the formal foundation when the engineering questions get sharp.
