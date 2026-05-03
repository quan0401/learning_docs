---
title: "Handling Failure & Feedback — Critical Feedback Received, Feedback Given, Delivering Under Challenge, A Time You Failed"
date: 2026-05-03
updated: 2026-05-03
tags: [behavioral-interviews, question-bank, failure, feedback, self-awareness]
---

# Handling Failure & Feedback — Critical Feedback Received, Feedback Given, Delivering Under Challenge, A Time You Failed

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `behavioral-interviews` `question-bank` `failure` `feedback` `self-awareness`

---

## Table of Contents

- [Summary](#summary)
- [1. "Tell Me About Critical Feedback You Received"](#1-tell-me-about-critical-feedback-you-received)
- [2. "Tell Me About Feedback You Gave Someone"](#2-tell-me-about-feedback-you-gave-someone)
- [3. "Tell Me About a Time You Had to Deliver Under Challenging Circumstances"](#3-tell-me-about-a-time-you-had-to-deliver-under-challenging-circumstances)
- [4. "Tell Me About a Time You Failed"](#4-tell-me-about-a-time-you-failed)
- [Related](#related)
- [References](#references)

---

## Summary

This is the highest-rubric-density category. *A time you failed* is asked at almost every loop and is one of the strongest signals of self-awareness. *Critical feedback received* probes whether you actually integrate feedback or get defensive. *Feedback given* probes whether you can deliver hard truths without making the other person defensive. *Delivering under challenge* probes resilience and judgement under constraint. The traps are pervasive: throwaway failures, deflected feedback, "blunt truth-telling" that's actually rude, and challenge stories where the candidate worked themselves to exhaustion. The strongest answers in all four share a single property — **calibrated honesty** about what was hard, what didn't go well, and what specifically changed afterward.

---

## 1. "Tell Me About Critical Feedback You Received"

### 1.1 What this question is really asking

Whether you can hear feedback without flinching, integrate it, and change behavior. The interviewer is also checking whether you treat feedback as data or as personal attack.

### 1.2 The competency signal

- **Earn Trust.**
- **Self-awareness.**
- **Learn and Be Curious.**
- **Resilience.**

### 1.3 STAR worked example

> **[S]** *In my mid-year review two years ago, my manager gave me feedback that surprised me: she said I was sometimes hard for less-senior teammates to bring questions to. The specific behavior she'd seen was that during code review, my comments tended to assume context I had but the reviewee didn't.*
>
> **[T]** *I had two ways to take this. I could push back — "I'm trying to be efficient, my comments are technically correct" — or I could take it as data. I picked data.*
>
> **[A]** *I asked her for two specific examples — what comment, what context she thought was missing, who the reviewee was. She gave them. Both examples I remembered, and on re-reading them I could see what she meant: the comments assumed familiarity with our service-mesh setup that a 6-month-tenured engineer wouldn't have.*
>
> *I asked the two reviewees directly — separately, with low ceremony — whether the comments had landed. Both confirmed: technically correct but they'd had to spend 30+ minutes piecing together the assumed context. One of them had been quiet about it because she didn't want to seem like she didn't know.*
>
> *I made three changes. First, on every code-review comment I wrote going forward, I asked myself "what context does the reviewee already have?" before posting. Second, when I gave a comment that involved system context, I started linking the relevant doc or runbook directly in the comment. Third, I started using a "naive question test" — if a senior engineer reading my comment would understand it but a 6-month engineer wouldn't, I rewrote it.*
>
> **[R]** *Six months later, the same manager noted in my next review that the issue had improved. One of the reviewees from the original examples specifically thanked me a year later when she onboarded her own first new hire — she said she'd modeled her review style on the way mine had changed. The bigger thing for me: I'd been treating my reviews as a transactional speed metric, and the feedback reframed them as a teaching surface. That has stuck across other parts of my work too.*

### 1.4 Common pitfalls

- **Throwaway feedback.** "I was told my code could be cleaner" — too thin to be a real feedback story.
- **Deflection.** "I was told X but I didn't agree because Y." Strong answers acknowledge the legitimate part.
- **No specific behavior change.** "I tried to be more aware." What specifically changed?
- **The "I integrated it perfectly" narrative.** Some candidates describe feedback as if they accepted it instantly and never struggled. Trained interviewers know that's implausible; calibrated honesty about the initial sting scores higher.
- **The "I asked for feedback" humble-brag.** "I told my manager I wanted critical feedback" — the question is about feedback you *got*, not feedback you solicited.

### 1.5 Follow-ups to expect

- "What was your first reaction to the feedback?"
- "Has feedback ever been wrong?"
- "Tell me about feedback you didn't integrate."
- "How do you give yourself critical feedback?"

### 1.6 Story-bank tags

Earn Trust · Self-awareness · Learn and Be Curious · Resilience

---

## 2. "Tell Me About Feedback You Gave Someone"

### 2.1 What this question is really asking

Whether you can deliver hard feedback in a way the recipient can hear it. The trap: candidates often describe "blunt" feedback as if directness alone were the signal. The rubric wants *direct AND care-personally* — the Kim Scott "Radical Candor" pattern.

### 2.2 The competency signal

- **Earn Trust.**
- **Develop the Best.**
- **Communication.**
- **Empathy.**

### 2.3 SBI inside the answer

The strongest answers use SBI (Situation–Behavior–Impact) when describing the actual feedback delivery. SBI separates *what happened* from *how you felt about it*, which is what makes feedback land rather than provoke.

### 2.4 STAR worked example

> **[S]** *Last quarter, a peer engineer on my team — same level as me — had taken on a project that was running into design issues. I'd seen her design doc and noticed she was committing to a pattern (eventual consistency on what was actually a strict-consistency requirement) that I thought would cause real problems at launch.*
>
> **[T]** *She hadn't asked for my feedback. We were peers; I had no formal authority. I needed to flag the issue without overstepping or coming across as territorial.*
>
> **[A]** *I asked her for a 30-minute coffee — out of band, not in the team channel. I started by asking her to walk me through the design, including the parts I'd already read. I wanted to understand her reasoning, not just react to the doc.*
>
> *Her reasoning was sound for the part of the system she'd been thinking about. The issue was that she hadn't been deeply involved in the part of the system that needed strict consistency — that surface had history I knew but she didn't.*
>
> *I gave the feedback in SBI format:* **Situation** *— "in the doc you posted Tuesday";* **Behavior** *— "the section on read-after-write semantics specifies eventual consistency";* **Impact** *— "the downstream payment-confirmation path requires strict consistency, and I think the eventual model would surface as duplicate confirmations under retry, which we've seen on the older system."*
>
> *I named what I knew that she might not — the history of the strict-consistency requirement — and offered to send her the previous postmortem that had established it. I also explicitly said: "I'm flagging this because I'd want someone to flag it for me if it were my project. I'm not trying to take over the design."*
>
> *She read the postmortem, agreed the requirement was real, and revised the design over the following week. We reviewed the revision together; it was cleaner than her original would have been but also cleaner than I would have produced — she'd found a third approach I hadn't considered.*
>
> **[R]** *The project shipped with the revised design and there were no consistency-related issues at launch. She and I have a stronger working relationship; she'll now flag things to me proactively, and I do the same with her. The "out-of-band, ask first, then SBI" pattern is now my default for peer feedback.*

### 2.5 Common pitfalls

- **Bluntness as virtue.** "I just told her the truth." Directness without care is rudeness.
- **Public feedback.** Strong feedback is private and out of band. Public feedback in a team channel or PR is rarely a strong story.
- **No empathy step.** Asking the person to walk through their reasoning *before* delivering feedback is a strong signal.
- **No outcome for them.** What did the feedback actually change? If nothing, the story doesn't land.
- **Feedback that wasn't actually feedback.** "I gave a teammate a code review." That's a code review, not a feedback story.

### 2.6 Follow-ups to expect

- "What if she had pushed back?"
- "Has feedback you gave ever damaged a relationship?"
- "Tell me about feedback you held back."
- "How do you decide whether to give feedback at all?"

### 2.7 Story-bank tags

Earn Trust · Develop the Best · Communication · Empathy

---

## 3. "Tell Me About a Time You Had to Deliver Under Challenging Circumstances"

### 3.1 What this question is really asking

Resilience and judgment under constraint. The trap: candidates describe heroic effort, but the rubric wants *making the right calls under pressure*, not *working the most hours under pressure*.

### 3.2 The competency signal

- **Resilience.**
- **Bias for Action.**
- **Judgment under pressure.**
- **Communication during constraint.**

### 3.3 STAR worked example

> **[S]** *Q4 of last year, two weeks before our biggest holiday traffic peak, our primary capacity-planning lead — the engineer who owned the load-test framework — went on emergency medical leave. The framework had a known issue that would surface under peak traffic; he had been mid-fix.*
>
> **[T]** *Someone needed to land the fix and re-run the load test before peak. I was the closest peer; I had familiarity with the framework but had never owned it. The deadline was a hard external one — peak traffic would happen whether we were ready or not.*
>
> **[A]** *I made an early call: do not try to do this alone. I asked my manager to backfill 30% of my regular sprint commitments to other peers — written, with specifics — so I could focus. She agreed.*
>
> *I started by reading every commit my colleague had made in the prior month and his half-written design doc. I wanted to know what *he* would have done, not just what *I* would do; otherwise the fix would be a different fix in disguise. The half-written doc was clear enough that I could continue from where he'd stopped.*
>
> *I built the fix in 4 days. I deliberately chose a smaller, more conservative scope than his original plan — fix the immediate issue, defer two adjacent improvements he'd been planning. I named that scope reduction in writing so when he came back he wouldn't be surprised.*
>
> *Ran the load test. Found two issues — one I expected (the original known issue), one I hadn't (a new issue surfaced by the fix itself). Fixed both within 2 days.*
>
> *Re-ran the load test 5 days before peak. Clean. Wrote a brief to my manager and the team, including: what I changed, what I deferred, what we'd validated, and a list of things I was *not* confident about that should be watched during peak.*
>
> *I worked normal hours. The "no hero shifts" rule was deliberate — I'd seen enough on-call burnout that I wanted to model that constraint plus delegation produces results, not constraint plus exhaustion.*
>
> **[R]** *Peak went smoothly. No load-related incidents. My colleague returned three weeks later; we walked through what I'd changed, what I'd deferred, and the two things I'd been uncertain about. He picked the work back up cleanly. He told me later that the deferral note was the thing that made the handback easy. The "scope-down + name what you deferred" pattern is something I now use any time I'm picking up someone else's in-progress work.*

### 3.4 Common pitfalls

- **Hero shifts as the answer.** "I worked nights and weekends." Doesn't read as judgement.
- **No delegation or rescoping.** The strongest answers under challenge involve *scoping down*, not *doing more*.
- **No communication of the trade-offs.** Strong answers always include what was deferred.
- **No calibration of confidence.** Strong answers include "what I was *not* confident about" — a key resilience-and-honesty signal.

### 3.5 Follow-ups to expect

- "What if your manager hadn't backfilled your other work?"
- "Walk me through the things you were not confident about."
- "How did you decide what to defer vs ship?"
- "Tell me about a challenge you didn't deliver on."

### 3.6 Story-bank tags

Resilience · Bias for Action · Judgment · Communication

---

## 4. "Tell Me About a Time You Failed"

### 4.1 What this question is really asking

Three things, in order: **was this an actual failure**, **did you own it**, and **what specifically changed in your practice afterward**. This is the single most rubric-loaded question in the loop.

### 4.2 The competency signal

- **Self-awareness.**
- **Ownership.**
- **Learn and Be Curious.**
- **Earn Trust.**

### 4.3 The three sub-checks

| Sub-check | What it measures | Common failure |
|---|---|---|
| **Was this an actual failure?** | Self-awareness | Throwaway "failure" (not actually a failure) |
| **Did you own it?** | Ownership | Blame distributed elsewhere |
| **What changed afterward?** | Learning | "I learned to be more careful" (generic) |

A story that fails any one sub-check fails the question.

### 4.4 STAR worked example

> **[S]** *Three years ago, I led a small project to migrate our caching layer from Memcached to Redis. I was about 18 months into the role. The project was scoped to 6 weeks; I committed to it.*
>
> **[T]** *Land the migration in 6 weeks.*
>
> **[A]** *Two failures, in sequence.*
>
> *Failure one: I underestimated the data-migration path. I had assumed we could do a "warm the new cache from the old" approach and cut over. In practice, the warming took 3x longer than I'd planned because of how our access pattern worked — we weren't reading the keys we were warming until users hit them, so the warm wasn't actually populating things in time. I noticed this in week 3, and I should have surfaced it then. I didn't, because I thought I could recover.*
>
> *Failure two: in week 5, with the warming approach clearly not working, I had to switch strategies under time pressure to a dual-write-with-backfill pattern. I wrote the backfill script in a hurry. It had a bug — it was idempotent for the keys that existed but didn't handle the case where Memcached had already evicted a key. About 0.3% of keys ended up not migrated correctly. We didn't catch it until day 3 of running, when a user-facing latency spike traced back to cache-misses on those specific keys.*
>
> *I owned both. I wrote the postmortem myself rather than having my manager write it. The postmortem named the actual root causes: my optimism in week 3 (delayed surfacing) and my hurry in week 5 (the bug). I did not blame the original 6-week estimate or anyone else.*
>
> *I made three specific changes to my practice afterward.*
>
> *First: any project longer than 4 weeks gets a status post at the 30% mark, which forces me to ask whether I'd take the bet that I'll hit the original date. If the answer is no, I surface that day.*
>
> *Second: any data-migration script — no matter how time-pressured — gets a separate review from a peer before running. The bug in the backfill would have been caught by a 20-minute review.*
>
> *Third: I now run a small dry-run of any migration logic against 1% of production data before running against the full set. The cost is a few hours; the recovery cost of a bad migration is days.*
>
> **[R]** *We finished the migration 2 weeks late, with a small amount of data corruption that we cleaned up over the next week. The 30% status-check rule has caught two real slips early since then. The peer-review-of-migration-scripts rule has caught two non-trivial bugs. The 1% dry-run rule is now a team norm — I wrote it up after one of my colleagues suggested it generalize.*

### 4.5 Common pitfalls

- **Throwaway failures.** "I once stayed late to fix a bug I should have caught earlier." Not a real failure.
- **Externalizing blame.** "We failed because the deadline was unreasonable." The question is about *your* failure.
- **The humble-brag.** "I failed by working too hard." This is the "weakness disguised as a strength" pattern — instantly recognized by trained interviewers.
- **Generic learning.** "I learned to communicate more." What specifically?
- **No evidence the lesson stuck.** Strong answers include evidence the change has paid off since (caught a slip, caught a bug).
- **Picking too small a failure.** A failure that didn't matter doesn't score.
- **Picking too large / catastrophic a failure.** A failure that suggests you shouldn't be hired ("I leaked customer data") is too much. Calibrate.

### 4.6 Follow-ups to expect

- "What if you had surfaced the warming issue in week 3 — what would have happened?"
- "Has the 30%-mark check ever been wrong?"
- "Tell me about a failure that's older than this one."
- "What's the failure you're most likely to repeat?"
- "Have you ever shipped a fix and then realized the fix had its own bug?"

### 4.7 Story-bank tags

Self-awareness · Ownership · Learn and Be Curious · Earn Trust · Resilience

---

## Related

- [Common Myths and Misconceptions](../foundations/02-myths-and-misconceptions.md) — including the "pick a small failure" myth this doc directly addresses
- [Time Management](07-time-management.md) — adjacent: missed-deadline stories overlap with failure
- [Conflict & Disagreement](04-conflict-disagreement.md) — adjacent: feedback-given stories sometimes touch conflict
- [Story Banking](../foundations/04-story-banking.md) — every bank should include a real failure story

## References

- Kim Scott, *Radical Candor*, 2nd ed., 2019 — care-personally / challenge-directly framing for the feedback-given question
- Lara Hogan, *Resilient Management*, 2019 — the SBI feedback model used in the worked example
- Carol S. Dweck, *Mindset: The New Psychology of Success*, Random House, 2006 — growth-mindset framing for the failure question
- John Allspaw, ["Blameless PostMortems and a Just Culture"](https://www.etsy.com/codeascraft/blameless-postmortems/), Etsy Code as Craft, 2012
- Marcus Buckingham & Ashley Goodall, "The Feedback Fallacy," *Harvard Business Review*, March–April 2019 — modern critique of feedback-as-correction
- Brené Brown, *Dare to Lead*, Random House, 2018 — vulnerability-as-leadership in the failure-narrative space
- Amazon, ["Leadership Principles"](https://www.amazon.jobs/content/en/our-workplace/leadership-principles) — *Earn Trust* (the LP that explicitly mentions self-criticism)
