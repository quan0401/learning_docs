---
title: "Project Based Questions — Most-Proud Project, Choosing Technologies, Leading a Project"
date: 2026-05-03
updated: 2026-05-03
tags: [behavioral-interviews, question-bank, projects, technical-decision, leadership]
---

# Project Based Questions — Most-Proud Project, Choosing Technologies, Leading a Project

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `behavioral-interviews` `question-bank` `projects` `technical-decision` `leadership`

---

## Table of Contents

- [Summary](#summary)
- [1. "Tell Me About a Project You're Most Proud Of"](#1-tell-me-about-a-project-youre-most-proud-of)
- [2. "How Did You Choose the Technology / Approach for $project?"](#2-how-did-you-choose-the-technology--approach-for-project)
- [3. "Tell Me About a Project You Led"](#3-tell-me-about-a-project-you-led)
- [Related](#related)
- [References](#references)

---

## Summary

Project-based questions are where most engineering candidates spend the bulk of their behavioral time. They are also where bar raisers extract the deepest follow-up signal: a memorized project narrative dies fast under "*walk me through the trade-off you made on $decision*". The three questions covered here look similar but score differently. *Most-proud* probes ownership and quantified impact; *technology choice* probes decision quality and trade-off articulation; *leading a project* probes scope of personal influence at the IC level. Pick a different story for each if possible — recycling a single project across all three weakens each answer.

---

## 1. "Tell Me About a Project You're Most Proud Of"

### 1.1 What this question is really asking

What does *you* value, what does *good work* look like in your model, and how scoreable is the project as a STAR. Bar raisers also use the answer as a launchpad for follow-up: the project you pick will get probed for 10+ minutes, so pick one that has depth.

### 1.2 The competency signal

- **Ownership.** Was the outcome attributable to your individual contribution?
- **Delivery.** Did the project actually ship and have measurable impact?
- **Self-awareness.** Are you proud of the right reasons (impact, growth, hard problem solved) rather than the wrong reasons (it was a fun stack to work with)?
- **Communication.** Can you tell the story of a complex project in 3 minutes?

### 1.3 Picking the project

The right project to pick has all of:

- **You owned the execution end-to-end** (not just one slice)
- **There was a real decision point** with a non-obvious answer
- **The outcome is quantified** with at least one specific number
- **It survives 3 layers of follow-up** (you remember the design doc, the disagreements, the post-launch metrics)

The wrong projects to pick:

- **Pure-success stories with no setbacks** — less scoreable than recovery stories
- **Projects where you were one of many** — hard to separate your contribution
- **Projects you can't talk about specifically** (NDA-restricted, legal-sensitive)
- **Projects where your role was minor** but you'd like credit for the team's outcome

### 1.4 STAR worked example

> **[S]** *In Q3 last year on the order-processing team, we had a long-standing problem with duplicate orders. Customers occasionally hit "submit" twice during slow network conditions, and our deduplication relied on a 5-minute Redis-keyed lock — which was both fragile and overspecified for the problem. The team had been talking about fixing it for two quarters but no one had owned it.*
>
> **[T]** *I volunteered to design and lead the fix. My task was to land a correct, durable, and migrationally-safe replacement for the dedup layer.*
>
> **[A]** *I started by mining a month of duplicate-order data from logs and reproducing four real failure modes. I wrote a one-page design doc proposing an idempotency-key-based approach: clients send a UUID with each request, and the server uses that as the dedup key with a 24-hour window in Postgres rather than 5-minute Redis.*
>
> *I considered three alternatives in the doc — keep Redis with longer TTL, move to Kafka with consumer-side dedup, and the Postgres-keyed approach — and named the trade-offs explicitly. The Postgres approach lost on raw write throughput but won on durability and observability, which were what we actually needed.*
>
> *I shared the doc, ran a 30-minute review with the team and SRE, and accepted three rounds of comments before locking the design. I built the implementation in 10 days, wrote a backwards-compatible migration that ran the new and old paths in parallel for two weeks, and ran shadow comparison to validate.*
>
> *Two issues surfaced during shadow: a clock-skew bug between the API and DB hosts that caused 0.02% of keys to look stale, and a retry-storm pattern from the iOS client that I hadn't anticipated. I fixed the first; for the second, I worked with the iOS team to add jittered retries.*
>
> **[R]** *We rolled out cleanly with no rollback. Duplicate-order incidents went from ~40 a month to under 5; the under-5 are now traceable to genuine client bugs rather than infrastructure. The design doc became the team's template for similar work — used twice since for similar dedup problems on other surfaces. After the project, I started a quarterly review of every "fragile-tolerated" issue on the team's runbook so the same two-quarter delay doesn't repeat.*

### 1.5 Common pitfalls

- **Picking a project from too long ago.** A 2-year-old project will have decayed details; the follow-up tree breaks.
- **Picking a project with no setbacks.** Pure-success stories don't show recovery and decision-making.
- **Conflating team success with personal contribution.** "We shipped X" is not the right voice; "I owned Y inside the team's X" is.
- **No quantified impact.** "It went well" instead of "from 40/month to under 5".
- **No second-order learning.** "I'd do it the same way" — this is a missed scoring opportunity.

### 1.6 Follow-ups to expect

- "Walk me through the design doc — what were the three alternatives?"
- "How did you decide on the 24-hour window?"
- "What were the three rounds of comments about?"
- "Tell me about the clock-skew bug specifically — how did you find it?"
- "What would you do differently?"
- "How did the iOS team react to the change you needed from them?"

### 1.7 Story-bank tags

Ownership · Delivery · Decision-making · Communication · Cross-team

---

## 2. "How Did You Choose the Technology / Approach for $project?"

### 2.1 What this question is really asking

Decision quality. The interviewer wants to see whether you make technical choices through structured reasoning or by hype. The strongest candidates can name (a) the constraints, (b) the alternatives considered, (c) the trade-off that decided it, (d) what would have flipped the decision.

### 2.2 The competency signal

- **Decision-making.** Structured trade-off reasoning vs cargo-culting.
- **Are Right, A Lot** (Amazon LP).
- **Engineering judgement.** Awareness of cost, complexity, durability.
- **Communication.** Can you articulate the trade-off cleanly?

### 2.3 The decision narrative — structure

| Beat | Content |
|---|---|
| Constraints | What were the actual requirements? Latency budget, scale, durability, team familiarity, deadline |
| Alternatives | 2–3 real options you considered |
| The trade-off axis | The dimension on which the choice turned |
| The choice | Which one and why |
| What would flip it | Counterfactual: under what different constraints would you have chosen differently |

### 2.4 Worked example

> *"For the dedup project, the constraints were: handle ~3k order RPS at peak, keep the dedup window long enough to cover client retry behavior (~24h), survive single-node failure, and finish before holiday peak.*
>
> *I considered three options.*
>
> *Option 1 — Keep Redis, increase TTL to 24h. Pro: zero migration cost. Con: Redis durability story is thin under failover, and we'd already had one incident where the cluster lost ~10 minutes of data.*
>
> *Option 2 — Push dedup to Kafka with consumer-side idempotency. Pro: durable by construction. Con: significantly more moving pieces, the team had limited Kafka experience, and we'd be on the critical path for shipping which had a hard deadline.*
>
> *Option 3 — Postgres with an idempotency_key table and unique-constraint enforcement. Pro: durable, observable through normal SQL tooling, team highly familiar. Con: lower raw write throughput than Redis.*
>
> *The deciding axis was* **durability + team familiarity vs raw throughput**. *Our peak write rate (~3k/sec) was well within Postgres territory; we'd already verified that with a load test. Throughput wasn't the binding constraint. Durability was.*
>
> *We picked option 3.*
>
> *What would have flipped it: if the peak RPS had been 30k+ rather than 3k, Postgres throughput would have become the binding constraint and we'd have invested in the Kafka path. Or, if the deadline had been further out, we'd have considered a hybrid. The decision was specific to our actual numbers."*

### 2.5 Common pitfalls

- **No alternatives named.** "We chose Postgres because it's reliable" — no trade-off visible.
- **The trade-off is unrelated to the actual decision.** "We picked X because it's faster" when the decision actually rode on team familiarity.
- **Hype-driven justification.** "We picked Kafka because it's the modern way." The interviewer's note will read "hype-driven" and the score reflects it.
- **No counterfactual.** "We'd choose the same way again" is acceptable in passing but the absence of counterfactual reasoning is a tell.

### 2.6 Follow-ups to expect

- "What was the load test you ran?" — bar raisers love this follow-up; have the numbers
- "What was the team's Kafka experience exactly?" — be ready to be specific
- "Have you used the Postgres-idempotency pattern before? Where?" — lineage of the idea
- "What did you not consider?" — willingness to name blind spots

### 2.7 Story-bank tags

Decision-making · Engineering judgement · Communication · Are Right A Lot

---

## 3. "Tell Me About a Project You Led"

### 3.1 What this question is really asking

What does *leading* mean in your model, and is it scope-appropriate for the level you're applying for. The trap: the candidate either (a) picks a project where they were a contributor and overstates their leadership role, or (b) picks a project where they were the manager and undershoots IC-level signal. Either fails.

### 3.2 The competency signal

- **Leadership at IC level.** Technical direction, peer alignment, unblocking, codifying practice.
- **Influence Without Authority.** Did you align peers without being their manager?
- **Ownership.** Did you take responsibility for the outcome?
- **Delivery.** Did the project actually ship?

### 3.3 What "leading" means at IC level

For an IC4–IC5 role, "leading a project" usually means one or more of:

- **Driving the design.** You wrote the design doc, ran the review, owned the trade-off discussion.
- **Coordinating people.** You ran the standups, kept the timeline, surfaced blockers.
- **Unblocking peers.** When teammates got stuck, you debugged with them.
- **Owning the outcome.** When something went wrong, you were the one in the war room.

It usually does *not* mean: writing performance reviews, hiring decisions, headcount allocation. Those are manager-track signals.

### 3.4 Worked example — picking up where the dedup story left off

> **[S]** *Same Q3 dedup project I just described. After my design doc was approved, the project shifted from "I designed it" to "I led the team's implementation."*
>
> **[T]** *Three engineers including me on the implementation, with two reviewers from SRE and the iOS team. My job was to drive the project to launch on the holiday-peak timeline.*
>
> **[A]** *I broke the work into 6 vertical slices and assigned them — keeping the trickiest two (the dual-write migration and the shadow-comparison harness) for myself. I ran a 15-minute standup three times a week and held the timeline visible in a single Linear tracking view.*
>
> *Two specific moments:*
>
> *One of my teammates was new to the codebase and got blocked on the migration's transactional semantics. Rather than just unblock them, I paired with them for two hours and walked through the actual rollback semantics. They later picked up the failover path and ran it themselves — that part of the project ended up being theirs.*
>
> *Two weeks before launch, the iOS team surfaced the retry-storm issue. I rescoped the project to add a coordination doc with the iOS team — added two days to the timeline but eliminated a class of bugs that would have shown up post-launch. I made the call myself and brought it to my manager after, not before.*
>
> **[R]** *We shipped on the original timeline plus the two-day coordination buffer, with no P0 during peak. The teammate I paired with was promoted that cycle, and my manager cited the way I'd developed her on the project specifically. The Linear tracking view became the team's default for project leadership.*

### 3.5 Common pitfalls

- **Manager-flavored answer at IC level.** "I assigned headcount and ran the OKR review" — wrong register for an IC role.
- **No specific people named.** "I led the team" — but who? What did you specifically do for them?
- **Conflating leading with doing all the work.** A good IC lead unblocks others; doing 100% of the implementation is the *opposite* of leading.
- **No outcome for the people, only for the project.** "We shipped on time" is fine, but "we shipped on time and $teammate grew into the on-call lead role" is better.

### 3.6 Follow-ups to expect

- "What did you specifically do for the teammate who got stuck?"
- "How did you decide which slices to take yourself?"
- "What did you escalate and what did you handle yourself?"
- "Tell me about a teammate who pushed back on your direction."
- "How would your teammates describe your leadership?"

### 3.7 Story-bank tags

Leadership · Ownership · Influence Without Authority · Mentor / Develop · Delivery

---

## Related

- [Career Oriented Questions](01-career-oriented.md) — TMAY usually pivots into a Project Based answer
- [Problem Solving](05-problem-solving.md) — adjacent: technical-problem stories often share material with project stories
- [Leadership & Initiative](08-leadership-initiative.md) — adjacent: leadership stories without the "project" framing
- [Story Banking](../foundations/04-story-banking.md) — pick the right project, not the most recent one

## References

- Will Larson, *Staff Engineer*, 2021 — IC-level leadership archetypes (Tech Lead, Architect, Solver, Right-Hand)
- Will Larson, *An Elegant Puzzle: Systems of Engineering Management*, 2019 — what "leading" means at the IC/manager boundary
- Tanya Reilly, *The Staff Engineer's Path*, 2022 — IC technical leadership signals
- Camille Fournier, *The Manager's Path*, 2017 — Tech Lead chapter, applicable to senior IC questions
- Amazon, ["Leadership Principles"](https://www.amazon.jobs/content/en/our-workplace/leadership-principles) — *Ownership*, *Bias for Action*, *Are Right A Lot* mapping to project answers
- Gayle Laakmann McDowell, *Cracking the PM Interview*, CareerCup, 2017 — though PM-flavored, the project-decision narrative pattern transfers cleanly
