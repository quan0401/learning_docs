---
title: "Dashboards, Runbooks, and On-Call"
date: 2026-04-26
updated: 2026-04-26
tags: [system-design, observability, on-call, sre]
---

# Dashboards, Runbooks, and On-Call

**Date:** 2026-04-26 | **Updated:** 2026-04-26
**Tags:** `system-design` `observability` `on-call` `sre`

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [Key Concepts](#key-concepts)
  - [Dashboard Tiers — One Purpose per Surface](#dashboard-tiers--one-purpose-per-surface)
  - [Dashboard Design Principles](#dashboard-design-principles)
  - [SLO Burn-Down Dashboards](#slo-burn-down-dashboards)
  - [Runbook Anatomy — Symptom to Post-Mortem](#runbook-anatomy--symptom-to-post-mortem)
  - [Auto-Remediation Hooks — PagerDuty, Rundeck, GitOps](#auto-remediation-hooks--pagerduty-rundeck-gitops)
  - [On-Call Ergonomics — Rotation, Page Budget, Handoff](#on-call-ergonomics--rotation-page-budget-handoff)
  - [Post-Incident Review — The Blameless Post-Mortem](#post-incident-review--the-blameless-post-mortem)
- [Trade-offs](#trade-offs)
- [Examples](#examples)
  - [A Sample Tiered Dashboard Structure](#a-sample-tiered-dashboard-structure)
  - [A Sample Runbook — High p99 Latency on Checkout](#a-sample-runbook--high-p99-latency-on-checkout)
- [Real-World Uses](#real-world-uses)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

Dashboards, runbooks, and on-call are the human-facing surfaces of a reliability program. Telemetry only matters when an operator can answer "is this broken?" in 30 seconds, then "what do I do about it?" in 5 minutes, then "did we actually learn anything?" in the days that follow. This doc covers how to design dashboards that answer one question each, runbooks that lead from symptom to remediation without "investigate" being a step, auto-remediation hooks that cut humans out of the loop where it's safe, SLO burn-down views that tie alerts to error budget, and the on-call ergonomics — rotation length, page budget, handoff, and post-mortem process — that keep the humans on the other end of the pager from quitting. The Google SRE Book chapters on on-call (ch. 11) and post-mortems (ch. 15) underpin most of the patterns here.

## Overview

A reliable system is not a system without incidents — it's a system where incidents are **detected fast, diagnosed faster, and remembered**. Three artifacts carry that load:

- **Dashboards** — the standing view. A first-responder lands on one, and the page either answers their question or points to the next dashboard.
- **Runbooks** — the procedural view. When an alert fires, the responder follows a runbook, not their intuition. Intuition is for the third page of a long incident, not the first.
- **On-call rotation** — the human view. People page each other in a sustainable rhythm. The schedule, the page budget, the handoff, and the post-mortem are the ergonomics that keep the system humane.

These three surfaces sit on top of the metrics, logs, and traces covered in [tracing-metrics-logs.md](tracing-metrics-logs.md), and use the signal taxonomy from [monitoring-red-use-golden-signals.md](monitoring-red-use-golden-signals.md). Without them, telemetry is noise — high-cardinality, low-decision noise.

## Key Concepts

### Dashboard Tiers — One Purpose per Surface

Dashboards fail when they try to be everything. A dashboard with 60 panels is not a dashboard, it's a parking lot. The fix is to **tier dashboards by audience and question**, with each tier answering one type of question:

| Tier | Audience | Question it answers | Typical content |
|------|----------|---------------------|------------------|
| **Tier 1 — Status / Top-of-funnel** | On-caller, exec | "Is anything user-visible broken right now?" | Per-service RED rollups, SLO burn rate, active incidents |
| **Tier 2 — Service** | Service owner, on-caller drilling in | "What is wrong with *this* service?" | RED + dependencies, deploy markers, error breakdowns |
| **Tier 3 — Subsystem / Resource** | Operator debugging | "What is happening inside this component?" | USE on the host or container, GC, queue depth, pool stats |
| **Tier 4 — Business / Product** | PM, exec, customer success | "Are users getting what they paid for?" | Conversion funnels, sign-ups, revenue, support ticket rate |

The flow during an incident is **top-down**: Tier 1 says "checkout is unhealthy" → Tier 2 for `checkout-svc` shows "p99 doubled, error rate flat, dependency `payments-api` is slow" → Tier 3 for `payments-api` shows "DB connection pool exhausted, GC pauses spiking". The responder never wanders. They walk a tree.

The "is X broken?" question must be answerable in **30 seconds** on Tier 1. Three things make that work:

- A single status indicator per service (green / yellow / red, plus a number that explains why).
- The same axis on every panel (left-to-right time, last 1h by default, click to expand to 24h or 7d).
- A glance-test: open the page on a phone screen at 2 AM. Can you tell what's wrong?

### Dashboard Design Principles

Honeycomb, Datadog, and Grafana all converge on roughly the same principles. The Grafana team calls this "dashboard hygiene":

- **One purpose per dashboard.** If you cannot write the question it answers in one sentence, split it.
- **Top-down layering.** Highest-level signal at the top (RED rollup, SLO), drill paths going down (per-endpoint, per-host, per-query). Eyes scan top-left to bottom-right; reserve that path for the most decision-critical signals.
- **RED and USE on the same view, where the audience overlaps.** RED (Rate, Errors, Duration) tells you about user-visible behavior. USE (Utilization, Saturation, Errors) tells you about resources. A service dashboard that shows only RED leaves the operator blind to "we're hitting CPU saturation"; a dashboard with only USE leaves them blind to "users are seeing 500s." See [monitoring-red-use-golden-signals.md](monitoring-red-use-golden-signals.md).
- **Annotate deploys, config changes, feature flag flips, and known incidents directly on the timelines.** Half of incidents are correlated with a recent change; if the dashboard does not show changes, the responder reconstructs them from chat scrollback.
- **Same time range, same colors, same units everywhere.** Latency in milliseconds in one panel and seconds in another panel two rows down is a real production trap.
- **Dashboards are not art.** Resist the urge to make every panel a different visualization. Stat + line + heatmap covers 95% of operational use.
- **Every dashboard has an owner**, listed at the top, and a "last reviewed" date. Stale dashboards lie.
- **Dashboards-as-code.** Define dashboards in JSONnet, Terraform, or the Grafana provisioning API. They live in Git, get reviewed, get version history, and can be templated across services. Hand-edited dashboards rot; generated dashboards stay consistent.
- **The 30-second test.** Open Tier 1 fresh. Without scrolling or clicking, can you say which services are healthy and which are not? If yes, the dashboard works. If you have to read a legend or hover for tooltips, redesign.

A useful test: pick a recent incident. Could a stranger to your team, given only the Tier 1 → Tier 2 → Tier 3 dashboards, have figured out what was wrong? If not, the dashboards have a gap.

```text
   Tier 1 — "Is anything broken?"            ← 30 sec answer
        │
        ▼
   Tier 2 — "What's wrong with this service?" ← 5 min answer
        │
        ▼
   Tier 3 — "What's happening inside this box?" ← deep debugging
        │
        ▼
   Tier 4 — "Are users getting value?"        ← business view
```

### SLO Burn-Down Dashboards

An SLO ("99.9% of checkout requests succeed in under 500 ms over a 28-day window") gives you an **error budget** — the inverse of the SLO. A 99.9% target over 28 days allows roughly 40 minutes of error per month. A burn-down dashboard turns that budget into a falling line.

```text
  100% +─────────╮
       │         ╰─────────╮
       │                   ╰─────╮
       │                         ╰─────╮         ← burn-down
       │                               ╰─────────────
   0%  +──────────────────────────────────────────────→
       start                                        end of window
```

Two kinds of alerts come off this:

- **Slow burn** — over 24h, you've consumed >2% of your monthly budget. Page during business hours.
- **Fast burn** — in the last 1h, the burn rate is high enough to exhaust the budget in under 6h if it continues. Page immediately.

Multi-window, multi-burn-rate alerts (described in the SRE Workbook ch. 5) avoid both noisy short-window pages and silent slow leaks. The burn-down dashboard makes this visible: the responder sees not just "an error happened" but "we will be out of budget by Thursday."

When budget is exhausted, the SRE Book prescribes a **freeze on risky changes** — feature flag-only deploys, no schema changes, no new infra rollouts — until the service stabilizes and the budget recovers. The dashboard is the tool that makes that policy enforceable.

A typical SLO burn-down dashboard has these panels:

- Headline number: "current budget remaining: 47%, will be exhausted in 9 days at current rate."
- Burn-down curve: budget over the rolling window, with a projected line.
- Burn rate by hour: a heatmap showing when budget is being consumed (often a single hour during a deploy is responsible for most of the month's burn).
- Top contributors: which endpoints, regions, or customers are consuming the most budget.
- Alert state: are slow-burn or fast-burn alerts firing right now?

The math: with a 28-day window and a 99.9% target, the budget is `0.001 * total_requests`. A fast-burn alert might trigger at "consuming budget at 14.4× the sustainable rate over the last hour" — meaning you'd exhaust the month in roughly 50 hours. A slow-burn alert at 6× over 6 hours catches the slow leak.

### Runbook Anatomy — Symptom to Post-Mortem

A runbook is a step-by-step procedure for handling a specific alert. It is **not** generic ops documentation. It is "this alert fired; here is what to do." Good runbooks have five parts in this exact order:

1. **Symptom** — what the alert says, and what users likely see. Plain language. Include a link to the alert definition.
2. **Diagnosis steps** — concrete, ordered checks. Each step is a query, a CLI command, or a dashboard URL. No "investigate" or "look at the logs."
3. **Remediation** — the fix. Often a single command, a runbook in another tool, or an escalation. List rollback, restart, scale-up, drain-and-retry — whichever applies.
4. **Escalation** — if remediation doesn't work, who to page next, with their handle and the criteria. Time-boxed: "if not resolved in 15 min, page database-team."
5. **Post-mortem link** — once the incident is over, link the post-mortem here. Future readers see what happened last time.

The fastest test of a runbook is: hand it to someone outside your team, page them at 3 AM, and see if they can run it. If they have to ask "what does *this* mean?" before step 3, the runbook needs editing.

A few practical rules:

- **Every page-able alert has a runbook.** No exceptions. The alert links to it directly. PagerDuty, Opsgenie, and Grafana all support a `runbook_url` field; populate it.
- **Diagnosis steps are commands or links, not prose.** A `kubectl` command, a SigNoz/Grafana URL with the relevant filters pre-filled, a SQL query against the metrics store. Copy-pasteable.
- **Runbooks live next to code**, in the same repo, version-controlled, reviewed in PRs. Confluence runbooks rot; runbooks in `runbooks/` next to `services/` get updated when behavior changes.
- **Update the runbook in the post-mortem.** Every incident either confirms the runbook was right or shows what was missing. Adding to the runbook is a post-mortem action item.
- **CI-check the runbook.** Lint links, validate command syntax, run the diagnostic queries against a non-production environment. A runbook that points to a deleted dashboard is worse than no runbook — it costs trust.
- **Time-box every step.** "Try restart, wait 3 minutes, if not recovered, escalate." Without time boxes, an on-caller spends 40 minutes on step 2 of a 6-step runbook because the script "almost worked."

### Auto-Remediation Hooks — PagerDuty, Rundeck, GitOps

Some classes of alerts have a known, safe, deterministic fix. For those, paging a human is wasteful — let the system fix itself. This is where **auto-remediation** lives.

The auto-remediation spectrum, from softest to strongest:

- **Self-healing infrastructure.** Kubernetes restarts crashed pods. Auto-scaling groups replace unhealthy nodes. ELBs drain dead targets. These are passive — they happen without an alert.
- **GitOps reconciliation.** Argo CD or Flux reconciles cluster state to a Git source of truth on a loop. Drift triggers a re-apply. A "config got changed manually" alert is auto-resolved by reconciliation, with an audit trail in Git.
- **Event-driven runbooks.** Rundeck, AWS SSM Automation, or PagerDuty Process Automation execute a scripted runbook on alert. Examples: "if disk > 90% on log host, run log rotation"; "if connection pool exhausted, restart the worker"; "if a queue is backing up, scale consumers."
- **Workflow engines + LLMs.** Some teams now use LLM-driven incident triage to summarize state and propose runbook steps. Treat as an assistant, not as the actor — keep the human in the loop on anything destructive.

Three rules for auto-remediation:

1. **Idempotent and bounded.** The action must be safe to run twice and must not loop forever. Cap retries; back off; alert on action failure.
2. **Observable.** Every auto-remediation produces an audit log entry, an annotation on the dashboard, and a slack/incident message. Silent fixes hide patterns.
3. **The runbook still exists.** If auto-remediation fails or is disabled, a human follows the same runbook. The script is just the human steps codified.

Auto-remediation reduces page volume without reducing visibility. A page that auto-resolves in 30 seconds with a remediation log is a different artifact from "the system fixed itself silently and we have no idea what happened."

A common pipeline:

```text
   alert fires
        │
        ▼
   PagerDuty event rule classifies → "known auto-remediable"?
        │                       │
       no                      yes
        │                       │
        ▼                       ▼
   page on-caller         trigger Rundeck job (idempotent, capped retries)
        │                       │
        │                       ▼
        │                  job logs to incident timeline + Slack
        │                       │
        │                       ▼
        │                  did alert clear within N minutes?
        │                       │           │
        │                      yes         no
        │                       │           │
        │                       ▼           ▼
        │                  resolve     fall through to page on-caller
        ▼                  silently
   on-caller follows runbook
```

Track `auto-remediation success rate` as a metric. If it dips, your remediation script is decaying or the underlying problem has shifted.

### On-Call Ergonomics — Rotation, Page Budget, Handoff

Google's SRE Book (ch. 11) is explicit about what makes on-call sustainable. The key levers:

- **Rotation length.** Weekly rotations are the dominant pattern. Less than a week creates churn — every shift starts cold. More than a week burns people out. A handoff at the same wall-clock time each week (e.g., Tuesday 10 AM) creates a natural rhythm.
- **Follow-the-sun.** If your team spans timezones, hand off the pager at end-of-day rather than running 24-hour shifts. A US-EU-APAC rotation means each person carries 8–10 hours of pager rather than 24. The cost is a more complex handoff and the risk of context loss across timezones — pay for that with a written handoff document.
- **Page budget per shift.** SRE Book sets a target of **no more than 2 pages per 12-hour shift on average**. Above that, the team is in alert-overload mode and should stop feature work to fix the alerts (or the underlying reliability problems). This is enforceable with a dashboard counting pages per shift; trending pages over time reveals when an alert has decayed into noise.
- **Primary / secondary.** Primary takes the page; secondary takes pages the primary misses, and is the first escalation. Two layers absorb a missed page without waking the entire team.
- **Compensation.** Whether time off, monetary, or both — on-call has a real cost and should be paid. Unpaid on-call burns out fast.
- **Onboarding and shadowing.** A new on-caller shadows two rotations before going primary. They watch how a senior responds, ask questions in real time, and get used to the tools without the pressure of being the responder.
- **Handoff document.** Every shift change includes a 10-line handoff: open incidents, ongoing investigations, recently flapping alerts, deploys in progress, anything to watch. Without this, every Tuesday morning starts with the new on-caller reverse-engineering the previous week from chat.
- **Acknowledged-vs-resolved metrics.** Track time-to-acknowledge and time-to-resolve per page, per service. Acknowledge times above 5 minutes mean the pager isn't loud enough or the on-caller is overwhelmed. Resolve times that climb over a quarter mean runbooks or remediation tooling are decaying.
- **Sustainable load-shedding.** When the on-caller is in the middle of an active incident, low-priority pages should be silenced (auto-snoozed, deferred to ticket queue) for the duration. Pages-during-pages multiply cognitive load and miss critical signals.

A simple rotation health dashboard answers four questions per shift: how many pages, how many during sleep hours, time-to-ack distribution, and how many auto-resolved. Trending these over months reveals when an on-call rotation is becoming unsustainable, ideally before someone quits.

The SRE Book's hard-won lesson: **on-call is part of the job, not on top of it.** The on-caller is doing reliability work. They are not also expected to ship features that week. If they are, the on-call is an unfunded mandate, and the team will erode.

### Post-Incident Review — The Blameless Post-Mortem

The post-mortem is where incidents become learning. Google SRE Book ch. 15 is the standard reference. The structure that works:

- **Blameless framing.** The post-mortem describes systems and decisions, not individuals' mistakes. "Engineer A pushed a bad config" becomes "the deploy pipeline allowed a config change without canary; the canary stage existed but was bypassed by a flag set during the previous incident's response." Blame removes information; blameless framing reveals it.
- **Timeline.** Minute-by-minute, with timestamps from logs and chat. Who did what, what alerted, what was tried.
- **Impact.** Users affected, requests dropped, error budget consumed, money lost or refunded. Concrete numbers.
- **Root causes (plural).** Almost no real incident has a single root cause. Use the "5 whys" or fishbone analysis. Document multiple contributing factors.
- **What went well.** Do not skip this. The team needs to see what their preparation paid for.
- **What went wrong.** Where the system or response failed.
- **Action items.** Each one has an owner, a JIRA/Linear ticket, and a target date. Items without owners are wishes.

Action items most often fall into a few buckets:

- Improve detection (better alert, lower threshold, faster propagation).
- Improve diagnosis (better dashboard, better logging, better runbook).
- Improve remediation (auto-remediation, faster rollback, better tooling).
- Prevent recurrence (canary, schema check, rate limit, circuit breaker, capacity headroom).

Post-mortems are **published to the org**, not buried in a Confluence corner. A culture of public post-mortems is what turns "we had an incident" into "we got smarter." They are also the link target from runbooks: a future on-caller hits the same alert, opens the runbook, sees three prior post-mortems linked, and walks in armed.

A post-mortem-readiness check during the incident itself: assign a **scribe role** at the start. The scribe captures timestamps, decisions, and observations in a shared doc as the incident unfolds. After resolution, that doc becomes 70% of the post-mortem draft. Without a scribe, the team reconstructs the timeline from chat scrollback days later, and detail is lost.

Tracked across incidents, post-mortem action items become their own SLO. "Action item completion rate" — what percentage of post-mortem TODOs are closed within 30 days — is a reliability KPI. A team with a 20% action-item completion rate is generating learning artifacts and discarding them.

## Trade-offs

| Choice | Trade-off |
|--------|------------|
| Many dashboards (one per question) vs few mega-dashboards | Many = clearer purpose, more navigation; few = less navigation, lower signal density. Many wins for incident response. |
| Per-team runbooks vs central knowledge base | Per-team = fresher, owned, code-reviewed; central = discoverable, harder to keep current. Per-team in a discoverable index wins. |
| Auto-remediation aggressively vs page-the-human | Aggressive = lower toil, risk of hiding regressions; page-the-human = full visibility, alert fatigue. Auto-remediate the deterministic, page on the ambiguous. |
| Long rotations (1 week+) vs short (24h) | Long = context retention, fatigue risk; short = rested responders, constant cold starts. Weekly with daytime/overnight split if traffic warrants. |
| Strict page budget vs ship-all-the-alerts | Strict = forces alert hygiene, may miss low-signal events; permissive = catches more, drowns the team. Strict wins; alerts that don't pass the bar become tickets, not pages. |
| Public post-mortems vs internal-only | Public (within the org) = broad learning, more discipline; internal = candor on sensitive incidents. Default public, with redacted versions for sensitive cases. |
| One global on-call vs per-service rotations | Global = small teams, simple; per-service = deep expertise, harder to staff. Per-service once the team is large enough, with a global escalation tier above. |
| RED + USE on the same dashboard vs separate | Same = immediate correlation between user-visible and resource symptoms; separate = less crowded, more navigation. Same wins for service-level Tier 2. |

## Examples

### A Sample Tiered Dashboard Structure

For an e-commerce platform with `web`, `checkout`, `payments`, and `search` services:

**Tier 1 — Platform Status** (URL: `/dashboards/status`)

- One row, four panels: per-service RED rollup (request rate, error rate, p50/p95/p99 latency, status color).
- One row: SLO burn rate per service (last 1h fast burn, last 24h slow burn) — see [SLO burn-down](#slo-burn-down-dashboards).
- One row: active incidents (PagerDuty embed), recent deploys (last 24h, color-coded by service).
- Top-right corner: "last updated" + page owner.

**Tier 2 — Per-Service** (e.g. `/dashboards/checkout`)

- Row 1: RED for `checkout-svc` — total + per-endpoint, deploy markers overlaid.
- Row 2: USE — CPU, memory, network, JVM heap, GC pauses, thread pool saturation.
- Row 3: Dependency health — call rate, error rate, latency to `payments`, `inventory`, `cart-cache`, `db-checkout`.
- Row 4: Top errors (log-scale heatmap or count by class).
- Row 5: Business — orders/min, conversion rate, cart abandonment.

**Tier 3 — Subsystem** (e.g. `/dashboards/checkout-db`)

- DB-level USE: connections, query rate, slow queries, replication lag, lock waits.
- Per-query latency heatmap.
- Storage saturation, IOPS.

**Tier 4 — Business** (e.g. `/dashboards/business`)

- Sign-ups/hour, paid conversions/hour, revenue/hour, refund rate, support ticket creation rate.
- Owned by product/finance, not engineering, but linked from Tier 1 so an exec can land on it directly.

The flow: page fires → Tier 1 (which service?) → Tier 2 (what's wrong with it?) → Tier 3 (what subsystem?) → fix.

### A Sample Runbook — High p99 Latency on Checkout

Stored at `runbooks/checkout/high-p99-latency.md`. Linked from the alert definition.

```markdown
# Checkout — High p99 Latency

**Alert:** `checkout_p99_latency_above_2s`
**Severity:** SEV-2 (paging, not waking the world)
**Owner:** checkout-team

## Symptom

p99 latency of `POST /checkout/submit` is above 2 seconds for 5 consecutive minutes.
Users likely see the spinner hang or a timeout error on the order confirmation step.

## Diagnosis steps

1. Open the [Checkout Tier 2 dashboard](https://grafana/d/checkout) — confirm the p99
   spike, check error rate (is it pure latency or are 5xx also climbing?), check the
   deploy marker (was there a deploy in the last 30 min?).
2. Check dependency latency panel. If `payments-api` p99 is also elevated, this is
   probably downstream — jump to [payments-api runbook](../payments/high-p99-latency.md).
3. Run:
   ```sh
   kubectl -n checkout top pods
   ```
   Look for any pod above 80% CPU or hitting memory limits.
4. Check the DB connection pool panel. If saturation > 90%, this is pool exhaustion.
5. Check the GC pause panel. If GC pauses > 500 ms, this is a heap issue.

## Remediation

| Diagnosis | Fix |
|-----------|-----|
| Recent bad deploy | `kubectl rollout undo deployment/checkout-svc -n checkout` |
| Single bad pod | Delete it: `kubectl delete pod <pod> -n checkout` (replicaset will replace it) |
| Pool exhaustion | Scale up: `kubectl scale deployment/checkout-svc --replicas=+2` |
| Downstream `payments-api` slow | Follow [payments runbook](../payments/high-p99-latency.md), do not roll back checkout |
| GC / heap pressure | Scale up; open ticket for heap profiling post-incident |

## Escalation

- If p99 not recovering 15 min after remediation, page **payments-team** secondary.
- If an active deploy is in progress and rolling back fails, page **platform-team**.
- If revenue impact > $10k/min (visible on Tier 4 business dashboard), declare SEV-1
  and page incident commander.

## Post-mortems

- 2026-02-14 — pool exhaustion from N+1 in cart enrichment. [post-mortem](../../postmortems/2026-02-14-checkout-pool.md)
- 2025-11-08 — payments-api timeout cascade. [post-mortem](../../postmortems/2025-11-08-payments-cascade.md)
```

A first-responder paged at 3 AM does not improvise. They open this file and start at step 1.

A few traits make this runbook usable:

- Symptom is in the user's language, not the metric's name.
- Diagnosis is a numbered list of concrete actions, not "investigate."
- Remediation is a table indexed by diagnosis, not a wall of prose.
- Escalation has a time box and a named target.
- Post-mortems are linked, with one-line summaries — the next on-caller sees prior context immediately.

Compare to the anti-pattern: a Confluence page titled "Checkout latency troubleshooting" with three paragraphs of background, a list of commands without their context, and "if all else fails, escalate to the team." That page exists in many organizations, and on-callers stop reading it after the second incident because it costs more time than it saves.

## Real-World Uses

- **Google SRE.** The original on-call discipline. Weekly rotations, 2-page-per-shift target, mandatory blameless post-mortems, error budgets that gate feature work. Documented in the SRE Book (ch. 11, 15) and SRE Workbook.
- **Netflix.** Service ownership model — the team that builds the service is on call for it. No separate ops team. Strong runbook culture, public post-mortems, "you build it, you run it."
- **Etsy.** Pioneered the blameless post-mortem culture (John Allspaw, "Blameless PostMortems and a Just Culture", 2012). The "why does this keep happening?" framing is now standard.
- **Stripe.** Heavy investment in dashboards-as-code (Veneur, then Grafana JSONnet). Every service has a generated standard dashboard so on-callers see the same shape across services.
- **PagerDuty.** Their own incident-response process is published. Time-bound escalation, named roles (incident commander, scribe, subject matter expert), and a post-incident review on every SEV-1/2.
- **Honeycomb.** Public production engineering blog showing dashboards driven by their own observability tool (high-cardinality, BubbleUp for outlier detection). Their guidance: "every chart should answer one question."
- **GitLab.** Public runbooks repository (`runbooks/`) — a real-world example of code-versioned, peer-reviewed runbooks at scale.

## Anti-Patterns

- **Dashboards-as-art.** Sixty panels in nine colors with sparkline gauges and donut charts that nobody reads. The dashboard exists; nobody uses it. Fix by deleting panels until the page answers a single question, then renaming the dashboard after that question.
- **Runbooks that say "investigate".** "Step 1: investigate the issue" is not a runbook. It is a vibe. Replace with concrete commands and dashboard links.
- **24-hour on-call shifts.** Sleep deprivation makes incidents worse. A tired on-caller introduces additional incidents during recovery. Cap at 12 hours, prefer 8.
- **Pager budget creep.** When the team accepts "we just get a lot of pages," the team is on a path to attrition. Fix by treating page count as a feature-blocking SLO of its own.
- **Hero culture.** Celebrating the engineer who handled the incident solo at 4 AM. Hero culture means the system depends on heroes. Celebrate the post-mortem action items, the runbook updates, and the auto-remediation that made the next incident routine.
- **Confluence runbooks that nobody updates.** A runbook last edited 14 months ago that still references a service that was renamed 6 months ago. Move runbooks into the same repo as the service code, require PR review, and CI-check links for 404s.
- **Post-mortems that blame an individual.** Once people are blamed, they hide information from the next post-mortem. The result is a worse system. Blameless framing is not soft — it is the only structure that gets honest information.
- **Auto-remediation without observability.** Scripts that silently restart services without logging. Six months later, nobody remembers the script exists, and a real outage is being papered over by a quiet restart loop. Every auto-action emits an event.
- **One mega-dashboard for everything.** "Operations" with 14 service rollups, infra, business KPIs, and a Slack feed. Unfocused. Split it.
- **Alerts without runbooks.** A page that fires with no `runbook_url`. The on-caller searches Slack history for context. Forbid alerts without runbooks at the alert-definition layer.
- **No SLO, just thresholds.** Alerting on "CPU > 80%" with no connection to user impact. Replace with SLO-based alerts (burn-rate) where the alert means "users are being hurt at rate X."
- **Skipping the post-mortem on small incidents.** Small incidents are the cheap learning. Skipping them means the org only learns from disasters.
- **Symptom-only alerting with no SLO connection.** "Latency over 1 second" without asking "for whom, on which path, with what budget consequence." Alert noise without the why.
- **Runbook copy-paste rot.** A new service is created by copying an old runbook, changing the service name, and shipping. The thresholds, escalation contacts, and diagnostic queries remain set to the old service. Treat runbook generation as code generation, with templates and validation.

## Related

- [Monitoring — RED, USE, and Golden Signals](monitoring-red-use-golden-signals.md) — the metric taxonomy that drives what goes on dashboards
- [Tracing, Metrics, and Logs — The Three Pillars of Observability](tracing-metrics-logs.md) — the underlying telemetry layer
- [Chaos Engineering and Game Days](../reliability/chaos-engineering-and-game-days.md) — game days are runbook drills; chaos exercises validate dashboards and on-call response in advance of real incidents
- [Case Study — Designing Monitoring and Alerting for an Async Pipeline](../case-studies/async/design-monitoring-alerting.md) — applied example of dashboard tiers, runbooks, and SLO alerting in a real system

## References

- Betsy Beyer, Chris Jones, Jennifer Petoff, Niall Richard Murphy (eds.), ["Site Reliability Engineering — How Google Runs Production Systems"](https://sre.google/sre-book/table-of-contents/), Chapter 11, "Being On-Call" — the canonical reference on on-call ergonomics, rotation length, page budget, and primary/secondary structure.
- Betsy Beyer et al., ["Site Reliability Engineering"](https://sre.google/sre-book/postmortem-culture/), Chapter 15, "Postmortem Culture: Learning from Failure" — the blameless post-mortem template and the cultural argument for it.
- Betsy Beyer et al., ["The Site Reliability Workbook"](https://sre.google/workbook/alerting-on-slos/), Chapter 5, "Alerting on SLOs" — multi-window, multi-burn-rate alerting and SLO burn-down design.
- John Allspaw, ["Blameless PostMortems and a Just Culture"](https://www.etsy.com/codeascraft/blameless-postmortems/) (Etsy Code as Craft, 2012) — the foundational essay on blameless framing.
- PagerDuty, ["Incident Response Documentation"](https://response.pagerduty.com/) — public, opinionated process for incident roles, severity levels, and post-incident review.
- Grafana Labs, ["Common observability and dashboard design pitfalls"](https://grafana.com/blog/2022/06/06/common-observability-and-dashboard-design-pitfalls-to-avoid/) — practical dashboard hygiene from the Grafana team.
- Datadog, ["Dashboard Best Practices"](https://docs.datadoghq.com/dashboards/guide/dashboard-best-practices/) — vendor guidance on top-down layering, consistent units, and one-purpose dashboards.
- Charity Majors et al. (Honeycomb), ["Observability Engineering"](https://www.honeycomb.io/wp-content/uploads/2022/04/observability-engineering-honeycomb.pdf) — high-cardinality observability and the case for question-driven dashboards.
- GitLab, ["Runbooks"](https://gitlab.com/gitlab-com/runbooks) — public, version-controlled, peer-reviewed runbooks repository at production scale.
