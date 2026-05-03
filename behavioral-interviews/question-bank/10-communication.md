---
title: "Communication Skills — Explaining to Non-Technical, Presenting to Leadership"
date: 2026-05-03
updated: 2026-05-03
tags: [behavioral-interviews, question-bank, communication, presentation, audience-adaptation]
---

# Communication Skills — Explaining to Non-Technical, Presenting to Leadership

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `behavioral-interviews` `question-bank` `communication` `presentation` `audience-adaptation`

---

## Table of Contents

- [Summary](#summary)
- [1. "Tell Me About Explaining a Technical Concept to a Non-Technical Audience"](#1-tell-me-about-explaining-a-technical-concept-to-a-non-technical-audience)
- [2. "Tell Me About Presenting to Senior Leadership"](#2-tell-me-about-presenting-to-senior-leadership)
- [Related](#related)
- [References](#references)

---

## Summary

Communication questions score *audience-adaptation* and *Bottom-Line-Up-Front (BLUF) discipline*. The two questions covered here look similar but score different signals: the non-technical-explanation question scores whether you can build an analogy ladder and meet the audience where they are; the leadership-presentation question scores whether you can compress to the decision the audience needs to make and survive a hostile question without getting defensive. Strong answers in this category share three properties: a *named audience* (not just "non-technical people"), a *specific framing technique* (analogy, BLUF, the pyramid principle), and a *measured outcome* (the audience took the action you wanted, the leader made the decision, the question that was meant to derail you did not).

---

## 1. "Tell Me About Explaining a Technical Concept to a Non-Technical Audience"

### 1.1 What this question is really asking

Whether you can adapt your communication to the listener's frame, and whether you respect the audience enough to do that work. The trap: candidates pick a story about explaining "what an API is" to a customer-success person, which is too easy to score well. Pick a story where the explanation actually mattered — a compliance person, a CFO, a marketing lead — and where you had to meet a real comprehension gap.

### 1.2 The competency signal

- **Communication.**
- **Empathy.** Did you adapt to the audience?
- **Customer / User Obsession.** Often the non-technical audience is internal customer-equivalent.
- **Earn Trust.**

### 1.3 The analogy-ladder pattern

The strongest non-technical explanations build an *analogy ladder* — start with something the audience already knows, draw a structural analogy to the technical concept, then add the technical detail incrementally.

| Layer | What it does | Example (explaining a database lock) |
|---|---|---|
| **Layer 1: ground analogy** | Connect to something familiar | "It's like a meeting room — only one team can be inside at a time" |
| **Layer 2: extend the analogy** | Add structure | "If two teams want it, the second one waits in line" |
| **Layer 3: introduce the failure mode** | The thing you're trying to explain | "If the team inside forgets to leave, everyone waiting backs up" |
| **Layer 4: technical anchor** | Optional, only if needed | "We call that 'lock contention' — the database equivalent of the meeting-room queue" |

The ladder lets the audience choose how deep to go. Most stop at layer 3 — that's enough for them to make the decision they need to make.

### 1.4 STAR worked example

> **[S]** *Last year, my team had an incident that affected ~12,000 customers — a checkout slowness for about 40 minutes. The next day, our customer-success lead — a non-engineer with 8 years of customer-facing experience but no technical background — needed to explain what had happened to three large enterprise customers, on calls she'd be running personally. She came to me asking for the engineering-side explanation she could draw from.*
>
> **[T]** *I needed to give her an explanation she could actually use — accurate enough that the customer's technical contact (often an engineer on their side) would believe her, simple enough that she could deliver it confidently and answer follow-ups without overstepping.*
>
> **[A]** *I started by asking her two things: who specifically would be on the calls, and what they were most likely to ask. The customers' contacts varied — some were engineering-side, some were operations-side. The most likely follow-up question was "could this happen again?" — which is the customer-success question disguised as a technical one.*
>
> *I built the explanation in three layers, on a one-page doc.*
>
> *Layer 1 (everyone): "On Tuesday afternoon, our checkout service was slower than usual for about 40 minutes. Some customers' orders that involved 5+ items took 3–5 seconds to process instead of the normal 200 milliseconds. We've identified the cause and put a fix in place. We've also added monitoring to catch this specific pattern earlier next time."*
>
> *Layer 2 (if the audience asks "what was the cause?"): "We rely on a service that checks inventory before completing an order. That service got slower than usual for orders with 5+ items, and our timeout was set tight enough that it sometimes gave up. We've raised the timeout while the inventory team optimizes their side."*
>
> *Layer 3 (if the audience asks "could this happen again?"): "The specific pattern won't recur — we've fixed it. Similar patterns could happen with other services we depend on. We've added a class of monitoring (per-cart-size SLO tracking) that would have caught this 30 minutes earlier; we're rolling that pattern out across our other downstream dependencies."*
>
> *I role-played the calls with her — I played the customer's engineering contact and asked the hardest technical follow-ups I could think of. We did this twice; the second pass she answered without referring to the doc. She was ready.*
>
> **[R]** *All three calls went well — the customers were satisfied and one specifically commented that the explanation was unusually clear. The customer-success lead has come to me twice since for similar incident-explanation prep, and the layer-1/2/3 pattern has become a reusable template — we now produce that document for every customer-impacting incident as part of the postmortem process.*

### 1.5 Common pitfalls

- **Too-easy audience.** "I explained APIs to my mother." Not a credible signal.
- **Talking down.** "I dumbed it down for them." Disrespectful framing reads through.
- **No analogy.** Going straight to the technical content with shorter words isn't adaptation, it's compression.
- **No follow-up preparation.** Strong answers anticipate what the audience will *ask back*.
- **No measured outcome.** Did the explanation work? What did the audience do with it?

### 1.6 Follow-ups to expect

- "What if a customer's engineering contact had pushed back on the timeout explanation?"
- "How did you decide what to put in layer 2 vs layer 3?"
- "Tell me about a time the explanation didn't work."
- "What's a concept you've found particularly hard to explain?"

### 1.7 Story-bank tags

Communication · Empathy · Customer Obsession · Earn Trust

---

## 2. "Tell Me About Presenting to Senior Leadership"

### 2.1 What this question is really asking

Whether you can compress to what the leader needs to decide, whether you survive a hostile question without getting defensive, and whether you respect the audience's time. The trap: candidates describe presenting *information* to leadership, when leadership doesn't want information — they want a *decision frame*.

### 2.2 The competency signal

- **Communication.**
- **Bias for Action.** Did you ask for a decision, not just present?
- **Judgment.** Did you frame the decision well?
- **Resilience.** Did you handle the hostile question?

### 2.3 The BLUF / pyramid-principle move

Senior leaders read top-to-bottom and stop reading when they have the answer. Strong communication respects that.

**BLUF (Bottom-Line-Up-Front):** lead with the recommendation, then the data. Not the journey through the data with the recommendation at the end.

**Pyramid principle (Barbara Minto):** structure communication as recommendation → 2–4 supporting reasons → evidence under each reason. Each layer should make sense without the layer below it.

> **Bad opening:** "I'd like to walk through the analysis we did over the last 4 weeks. We started by looking at..."

> **Strong opening:** "I'm recommending we migrate our caching layer to Redis over the next two quarters, with a $X investment. Three reasons: durability, observability, and team readiness. I'll spend 5 minutes on each, then I'd like a decision today."

### 2.4 STAR worked example

> **[S]** *Q3 last year I presented the Postgres-vs-Redis cache decision to our VP of engineering and the staff engineers across the org — about 12 people in the room. I had a 20-minute slot. The decision had been made at the team level; the VP wanted to ratify it before the migration started.*
>
> **[T]** *Get the decision ratified, and use the time to surface any concerns the broader org had that I hadn't heard yet.*
>
> **[A]** *I used the BLUF structure throughout.*
>
> *Slide 1 (the recommendation): "Recommend migrating the cache layer to Redis over Q4–Q1. $X investment, 2-quarter timeline. Asking for sign-off today."*
>
> *Slides 2–4: three reasons — durability, observability, team readiness — each with two supporting data points and one anticipated counter-argument addressed up front.*
>
> *Slide 5: what we considered and didn't pick (Memcached, Postgres-as-cache, "do nothing"), with a brief why-not for each.*
>
> *Slide 6: risks I was naming (operational learning curve, cost surprise) and the mitigations I was committing to.*
>
> *Slide 7: the ask — explicit decision, sign-off on the budget.*
>
> *I rehearsed twice the day before. I expected three hostile-question categories: "is this just chasing the new shiny?", "what's the actual cost in engineering hours?", "what if we're wrong?"*
>
> *Two of those came up. The "actual cost in engineering hours" question I had a number for — 6 engineer-weeks — and I named the assumptions behind it. The "what if we're wrong" question I had pre-prepared: I named the reversibility (we could roll back to Postgres in a 2-week window during the first month) and the early-warning signals I'd be watching.*
>
> *One question I hadn't prepped: a staff engineer asked whether the Redis decision would lock us out of an experimental sharding architecture they were prototyping. I didn't know. I said so honestly — "I haven't looked at that; let me check and follow up tomorrow." I followed up the next day with a short note. The fact that I didn't try to bluff was, I think, what tipped the room from skeptical to ratifying.*
>
> **[R]** *The VP signed off in the meeting. The migration shipped on schedule. Two months in, I sent a follow-up note to the same group with the actual progress against the plan — a practice the VP has since asked other teams to adopt for major decisions. The pattern of front-loading the recommendation, naming what we considered and didn't pick, and answering "what if we're wrong" up front is something I now use in any leadership-facing deck.*

### 2.5 Common pitfalls

- **Information dump.** "I walked them through the analysis." Leaders don't have time for the analysis; they need the decision.
- **No ask.** Strong leadership presentations end with a specific decision request.
- **No anticipated objections.** Pre-empting the obvious counter-arguments saves time and signals preparation.
- **Bluffing the unknown question.** When you don't know, say so and follow up. Bluffing is recoverable from a small bluff but not from a confident one that turns out to be wrong.
- **Defensive on the hostile question.** Treat the hostile question as legitimate (often it is). The hostile question is usually pointing at a real risk you should engage with.

### 2.6 Follow-ups to expect

- "Tell me about a hostile question you didn't handle well."
- "How do you decide what to put in the executive summary?"
- "Have you presented to leadership and lost the decision?"
- "What's the difference between presenting to your team and presenting to leadership?"

### 2.7 Story-bank tags

Communication · Bias for Action · Judgment · Resilience

---

## Related

- [Career Oriented Questions](01-career-oriented.md) — adjacent: TMAY is a 90-second leadership-style pitch
- [Project Based Questions](02-project-based.md) — adjacent: technology-choice presentations
- [Conflict & Disagreement](04-conflict-disagreement.md) — adjacent: managing-up moments overlap with leadership presentation
- [Failure & Feedback](09-failure-and-feedback.md) — adjacent: hostile-question handling overlaps with feedback-receiving

## References

- Barbara Minto, *The Pyramid Principle: Logic in Writing and Thinking*, 3rd ed., Pearson, 2009 — the canonical reference for the leadership-presentation structure
- Chip Heath & Dan Heath, *Made to Stick*, Random House, 2007 — the SUCCES framework (Simple, Unexpected, Concrete, Credible, Emotional, Stories)
- Nancy Duarte, *Resonate*, Wiley, 2010, and *DataStory*, IDEAPRESS, 2019 — narrative-arc and data-presentation patterns for leadership audiences
- Lewis C. Lin, *Decode and Conquer*, 4th ed., 2020 — BLUF discipline for product-and-engineering audiences
- Will Larson, *An Elegant Puzzle*, 2019 — managing-up and the "writing for leadership" patterns
- Lara Hogan, [Demystifying Public Speaking](https://abookapart.com/products/demystifying-public-speaking), A Book Apart, 2017 — hostile-question handling and presentation prep
- Amazon, ["Leadership Principles"](https://www.amazon.jobs/content/en/our-workplace/leadership-principles) — *Insist on the Highest Standards* — applied to leadership-presentation rigor
