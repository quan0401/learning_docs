---
title: "Behavioral Answers — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, behavioral-interviews, star, practice]
---

# Behavioral Answers — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `behavioral-interviews` `star` `practice`

---

## Table of Contents
1. [How to use this doc](#how-to-use-this-doc)
2. [Easy (warm-up traps)](#1-easy-warm-up-traps)
3. [Subtle (review-passers)](#2-subtle-review-passers)
4. [Senior trap (bar-raiser-only failures)](#3-senior-trap-bar-raiser-only-failures)
5. [Solutions](#4-solutions)
6. [Related](#related)
7. [References](#references)

## Summary

24 broken STAR-style answer snippets across the question categories in this learning path. Coverage spans the rubric-failure catalogue: pronoun discipline, action-heavy ratio, missing trade-offs, generic learning, throwaway failures, hypothetical drift, blame distribution, fabricated specifics, scope-level mismatch, "rescue" framing, and the bluffed follow-up that bar raisers explicitly probe for. Each bug cites a real public source — book, talk, posted rubric, HR/IO-psych research, or a published interviewer guide. Aim for one pass before any loop.

## How to use this doc
- Read each broken snippet and identify the rubric-failure mode before opening `<details>`.
- The hint inside `<details>` is one line; the full diagnosis + fixed answer is in §4 Solutions, keyed by bug number.
- Difficulty sections are cumulative — don't skip Easy unless you've nailed those failure modes in mock interviews.

---

## 1. Easy (warm-up traps)

### Bug 1 — pronoun collapse

> *"Tell me about a time you owned a project end-to-end."*
>
> *"We had a problem with our deployment pipeline last year. We decided to migrate it to a new system. We chose Kubernetes because it was modern. We worked together on the migration, and we shipped it on time. We learned a lot."*

<details><summary>Hint</summary>Count the first-person singular verbs.</details>

### Bug 2 — the throwaway failure

> *"Tell me about a time you failed."*
>
> *"Honestly, I work hard to avoid failure, but I'd say one time I didn't comment my code well enough on a PR and the reviewer asked me to add comments. I learned to be more thorough with my comments."*

<details><summary>Hint</summary>Is this an actual failure, or a moment of being asked to revise?</details>

### Bug 3 — hypothetical drift

> *"Tell me about a time you disagreed with your manager."*
>
> *"If my manager and I disagreed about something, I'd probably write a one-page doc and ask for time to discuss it privately. I think being prepared with data is important. I'd never go around them."*

<details><summary>Hint</summary>What tense is the answer in?</details>

### Bug 4 — claim without evidence

> *"Tell me about yourself."*
>
> *"I'm a passionate engineer who loves solving hard problems. I'm a strong communicator and a team player. I think technology can change the world and I want to be part of that. I'm hardworking and detail-oriented."*

<details><summary>Hint</summary>Adjectives without behaviors. The interviewer cannot score "passionate."</details>

### Bug 5 — no quantification

> *"What's a project you're most proud of?"*
>
> *"I worked on a project to improve our caching layer. It made things faster and reduced errors. The team was happy with the result and customers benefited. It was a great project to lead."*

<details><summary>Hint</summary>Faster than what, by how much, with what error reduction?</details>

### Bug 6 — blame the deadline

> *"Tell me about a time you missed a deadline."*
>
> *"There was a project last year where the deadline wasn't realistic from the start. The PM committed us to 6 weeks but it should have been 10. We missed it because the original estimate was bad."*

<details><summary>Hint</summary>Whose ownership is on display?</details>

### Bug 7 — bashing the colleague

> *"Tell me about a difficult colleague."*
>
> *"I worked with someone who was just rude in code reviews. He'd leave snarky comments and put people down. The whole team avoided his reviews. Eventually management had to step in."*

<details><summary>Hint</summary>The interviewer is checking whether you'll do the same to a future colleague — them.</details>

### Bug 8 — implausible speed

> *"Tell me about a time you quickly learned a new technology."*
>
> *"I learned Rust in a weekend. By Monday I was deploying production code in it. It really helped me realize I can pick up anything fast."*

<details><summary>Hint</summary>What's the bar raiser's first follow-up going to be?</details>

---

## 2. Subtle (review-passers)

### Bug 9 — humble-brag failure

> *"Tell me about a time you failed."*
>
> *"I think my biggest failure is that I sometimes care too much about quality and end up holding up a project. Last year I refused to ship a feature until I'd added 100% test coverage, and we shipped two days late because of it. I've learned that I need to balance my perfectionism with practical delivery."*

<details><summary>Hint</summary>This is the "weakness disguised as a strength" pattern.</details>

### Bug 10 — front-loaded answer

> *"Tell me about a project you led."*
>
> *"I joined the team in 2021. We were a 12-person team working on the order-management service, which had been built originally in 2018 by a different team. The original team had used a microservices architecture with about 8 services, written in a mix of Java and Python. The product itself was a B2B platform serving about 200 customers, ranging from mid-size SMBs to a few enterprise accounts. The team's culture was…* [continues for 90 seconds before describing any action]"

<details><summary>Hint</summary>The S+T to A ratio.</details>

### Bug 11 — no trade-off named

> *"How did you choose the technology for the project?"*
>
> *"We chose Postgres because it's reliable and the team knew it well. We've been happy with the choice. It's been working great for us."*

<details><summary>Hint</summary>What was *given up* by choosing Postgres? What would have flipped the decision?</details>

### Bug 12 — generic learning

> *"What did you learn from the failure?"*
>
> *"I learned to communicate more proactively, to be more careful with estimates, and to make sure the team is aligned. I think these are important lessons for any engineer."*

<details><summary>Hint</summary>Three claims, zero specifics. What did "communicate more proactively" actually look like the next time?</details>

### Bug 13 — story-question mismatch

> *"Tell me about a time you handled a conflict."*
>
> *"I led a migration project last year. We had to move from Memcached to Redis. The team was excited about it. I designed the dual-write approach and implemented it over 6 weeks. The migration was successful and we…"* [no conflict in the answer]

<details><summary>Hint</summary>The candidate is telling a different story.</details>

### Bug 14 — passive voice

> *"Tell me about a project you're proud of."*
>
> *"A migration was performed last year on our payment service. The runbook was written, the dual-write path was designed, and the cutover was executed over a 6-week window. The result was achieved with no customer impact."*

<details><summary>Hint</summary>Where is the actor?</details>

### Bug 15 — no second-order change

> *"Tell me about a time you missed a deadline."*
>
> *"I committed to 6 weeks and it took 8. I owned it, told my manager honestly, and re-planned with the downstream team. We shipped 2 weeks late. It was a learning experience."*

<details><summary>Hint</summary>The story is honest. What's missing is what changed in the candidate's *practice* afterward.</details>

### Bug 16 — the rescue narrative

> *"Tell me about a time you helped a teammate."*
>
> *"A junior on my team was stuck on a Postgres locking issue for two days. He came to me for help. I sat down at his computer, debugged it, and fixed it for him. He was really grateful."*

<details><summary>Hint</summary>What changed in the junior's capability?</details>

### Bug 17 — bluffed precision

> *"What was the impact?"*
>
> *"We reduced p99 latency by exactly 27.4%. The error rate dropped by 14.2 basis points. Customer satisfaction scores increased by 11 points on our internal survey."*

<details><summary>Hint</summary>The numbers feel manufactured.</details>

### Bug 18 — silently substituted question

> *"Tell me about a time you led a multi-team initiative."*
>
> *"There was a project last year where I led the migration of our caching layer. I worked closely with my own team — three other engineers — and we shipped it in 6 weeks…"* [the project was single-team]

<details><summary>Hint</summary>The question asked about *multi-team*. The answer is single-team without a flag.</details>

---

## 3. Senior trap (bar-raiser-only failures)

### Bug 19 — fabrication that fails follow-up

> *"Tell me about a production incident you led."*
>
> *"We had an outage last year affecting 50,000 users for 90 minutes. I led the incident response."*
>
> *Follow-up: "Walk me through the timeline — what time did you detect, what was your first hypothesis, and what did you try?"*
>
> *"It was sometime in the afternoon. We tried a few things. Eventually we figured out it was a database issue and rolled back. I don't remember the exact details."*

<details><summary>Hint</summary>The opening is precise; the follow-up is vague. Trained interviewers note the pattern.</details>

### Bug 20 — scope-level mismatch (staff candidate, IC4 story)

> *"Tell me about a time you drove organizational change."* &nbsp; *(staff-engineer interview)*
>
> *"In Q2 I noticed our team's PR review times were creeping up. I started a 'review buddies' program where each engineer was paired with another for accountability. PR turnaround came down 30%. The program is still running on the team."*

<details><summary>Hint</summary>Same story would score great at IC4. At staff, what's the scope?</details>

### Bug 21 — the "I won the disagreement" with no commit

> *"Tell me about a time you disagreed with your manager."*
>
> *"He wanted to ship without staged rollout. I made the case for it, and when he didn't agree, I went to his manager and got the call escalated. The skip-level overruled him and we shipped with staged rollout. It caught a real issue, so I was right."*

<details><summary>Hint</summary>The candidate "won." What's the trust signal?</details>

### Bug 22 — confident bluff on the unknown

> *"Tell me about a time you presented to leadership."* &nbsp; *During the answer, asked: "How did your VP handle the question about cost?"*
>
> *"She was really impressed with the cost analysis. She said it was the most rigorous she'd seen and she immediately approved the budget."*

<details><summary>Hint</summary>The follow-up asked how the *VP* handled it. The answer pivots to praise.</details>

### Bug 23 — no honest gap (the polished learning narrative)

> *"How do you enhance your technical knowledge?"*
>
> *"I read a paper a week, finish two technical books a month, attend three conferences a year, contribute to open source on weekends, and write a blog post each month. I find time for all of it because I'm passionate about learning."*

<details><summary>Hint</summary>This is implausible at sustained cadence. Trained interviewers will probe, and the depth won't be there.</details>

### Bug 24 — feedback story that's actually a code review

> *"Tell me about feedback you gave someone."*
>
> *"In a code review last week, I left a comment that the approach the reviewee was using would have a race condition under high concurrency. I gave a detailed explanation in the comment with a link to the documentation. They updated the PR and we merged it."*

<details><summary>Hint</summary>This isn't the kind of feedback the question is asking about.</details>

---

## 4. Solutions

### Bug 1 — pronoun collapse

**Failure:** Five "we" verbs, zero "I" verbs. The rubric scores *individual actions*; this answer has none.

**Source:** DDI's *Targeted Selection®* trainer notes; Lara Hogan's *Resilient Management* (2019) on the I-statement practice.

**Fix:**
> *"My team had a deployment pipeline that took 45 minutes per deploy and was fragile. **I** drafted a one-page proposal to migrate to Kubernetes, named two alternatives we considered, and led the design review. **I** owned the dual-environment cutover plan; another engineer owned the manifests. **I** ran the staged rollout — three environments over two weeks — and kept the rollback path warm. We shipped on the original timeline; deploy time dropped to 6 minutes; the rollback we used twice in week one was clean."*

The "we" applied to shared decisions is fine; "I" replaces every "we" attached to an individual action.

### Bug 2 — the throwaway failure

**Failure:** Not a real failure. The rubric's three sub-checks (was it real, did you own it, what changed) score zero on the first.

**Source:** Carol Dweck, *Mindset* (2006) on the difference between performance-goal and learning-goal framings; the Amazon LP *Earn Trust* explicitly mentions self-criticism.

**Fix:** see [Failure & Feedback §4](../question-bank/09-failure-and-feedback.md#4-tell-me-about-a-time-you-failed) for a full worked answer with a real failure.

### Bug 3 — hypothetical drift

**Failure:** Behavioral questions ask for past behavior. "I would" answers produce zero ticks in the actions column.

**Source:** Frank L. Schmidt & John E. Hunter, *Psychological Bulletin* 124(2), 1998 — the meta-analysis showing past-behavior questions outperform hypothetical questions in predictive validity.

**Fix:** Stay in past tense. "Last year my manager and I disagreed about $specific. *I* did $specific_action…"

### Bug 4 — claim without evidence

**Failure:** Adjectives without behaviors. The rubric cannot tick "passionate."

**Source:** Lewis C. Lin, *Decode and Conquer* (2020), Chapter 4 on show-don't-tell; Lara Hogan, *Resilient Management* (2019).

**Fix:** Replace each adjective with a specific behavior. "I'm a strong communicator" becomes "*Last quarter I gave a 30-minute talk to ~100 engineers on the dedup-design trade-offs; the talk became part of the team's onboarding materials.*"

### Bug 5 — no quantification

**Failure:** The "impact" sub-score on the rubric requires numbers.

**Source:** Nancy Duarte, *DataStory* (2019); the Amazon LP *Deliver Results* explicitly references quantified outcomes.

**Fix:** Find the numbers before the interview. "*p99 dropped from 800ms to 220ms; error rate from 2.1% to 0.3%; we reduced the on-call page count by 60% over the next quarter.*"

### Bug 6 — blame the deadline

**Failure:** Blame externalized to the PM and the original estimate. The rubric's *Ownership* sub-score requires the candidate to own the miss.

**Source:** Steve McConnell, *Software Estimation: Demystifying the Black Art* (2006) on the cone-of-uncertainty — engineers are responsible for their own estimates; "the PM committed us" without push-back is itself an ownership failure.

**Fix:** "*I committed to 6 weeks. In retrospect I should have pushed back at the time — my estimate had no margin for unknown-unknowns. I missed by 2 weeks. I now bake 25% margin into any estimate longer than 4 weeks; the practice has caught two real slips early since.*"

### Bug 7 — bashing the colleague

**Failure:** The interviewer scores *Earn Trust* and *Empathy*; bashing fails both. Also signals you'd describe a future colleague (them) similarly.

**Source:** Kim Scott, *Radical Candor* (2019) — the difference between candor and rudeness; the Amazon LP *Earn Trust*.

**Fix:** see [Conflict & Disagreement §3](../question-bank/04-conflict-disagreement.md#3-tell-me-about-a-difficult-colleague) — the reframe is "communication-style mismatch I adapted to," not "bad person I tolerated."

### Bug 8 — implausible speed

**Failure:** Bar raiser will probe. "Show me a non-trivial program you wrote that weekend" or "what's the difference between `Box<dyn Trait>` and `&dyn Trait`?" — depth probably isn't there.

**Source:** Anders Ericsson, *Peak* (2016) on the empirical limits of skill acquisition; deliberate-practice research consistently shows real proficiency takes structured weeks-to-months, not a weekend.

**Fix:** Calibrated honesty. "*I had three days of structured ramp on gRPC streaming — read X, built Y, validated Z. I came out with a real-but-bounded mental model: I knew streaming and cancellation; I had not learned the wire format or the experimental load-balancing options.*"

### Bug 9 — humble-brag failure

**Failure:** Trained interviewers recognize the "weakness disguised as a strength" pattern instantly. Scores zero on self-awareness.

**Source:** Gayle Laakmann McDowell, *Cracking the Coding Interview*, 6th ed. (2015), Chapter 7 explicitly warns against this pattern. Brené Brown, *Dare to Lead* (2018) on vulnerability-as-honesty.

**Fix:** Pick a real failure. See [Failure & Feedback §4](../question-bank/09-failure-and-feedback.md#4-tell-me-about-a-time-you-failed).

### Bug 10 — front-loaded answer

**Failure:** S+T over 50%. By the time the action phase starts, the interviewer has not ticked anything yet and the budget is half-gone.

**Source:** DDI's *Targeted Selection®* trainer notes recommend a roughly 25/65/10 S+T/A/R split. Will Larson, *An Elegant Puzzle* (2019), on writing-for-leadership analogously emphasizes leading with the action.

**Fix:** Cut Situation to 1–2 sentences. "*Q3 last year on the order-management team — 12 engineers, ~3k RPS service. I led the migration project. Here's what I did: …*"

### Bug 11 — no trade-off named

**Failure:** The decision-quality sub-score requires explicit trade-off articulation. "Reliable and team knew it" names benefits without naming what was given up.

**Source:** Donella Meadows, *Thinking in Systems* (2008) on leverage-point thinking; Will Larson, *An Elegant Puzzle* (2019) on the "what we're not doing and why" doc-section pattern.

**Fix:** "*Postgres bought us durability and observability at the cost of raw throughput; our peak rate was within Postgres territory, so throughput wasn't the binding constraint. If peak had been 30k+ RPS rather than 3k we'd have invested in the Kafka path.*"

### Bug 12 — generic learning

**Failure:** "Communicate more proactively" is unfalsifiable. The learning sub-score requires a *specific* second-order change.

**Source:** Donald Schön, *The Reflective Practitioner* (1983) on reflection-on-action; the Amazon LP *Learn and Be Curious*.

**Fix:** "*I added a 30%-mark check to any project longer than 4 weeks: at 30% elapsed I ask 'would I take the bet I'll hit the original date?'. If no, I surface it that day. The check has caught two real slips early since.*"

### Bug 13 — story-question mismatch

**Failure:** Memorized story bent to fit the question. The conflict signal isn't in the answer.

**Source:** DDI's *Targeted Selection®* trainer notes warn that interviewers should redirect to the missing competency; the redirection itself is recorded as needing remedial probing.

**Fix:** Pick a story that *contains the conflict*. If you don't have one, say so honestly: "*I haven't had a major conflict on a migration project. The closest is a disagreement with my tech lead on the cache-vs-load-test question. May I tell you about that instead?*"

### Bug 14 — passive voice

**Failure:** Removes the actor entirely. The rubric scores individual actions; "the runbook was written" assigns the action to nobody.

**Source:** Strunk & White, *The Elements of Style* (4th ed., 2000); Lara Hogan's I-statement practice in *Resilient Management* (2019).

**Fix:** Re-write with "I" as the subject of every action verb. "*I wrote the runbook, designed the dual-write path, and executed the cutover.*"

### Bug 15 — no second-order change

**Failure:** The learning sub-score requires evidence the lesson stuck. "It was a learning experience" doesn't anchor anything.

**Source:** Carol Dweck, *Mindset* (2006); Donald Schön, *The Reflective Practitioner* (1983).

**Fix:** Name the specific practice change *and* an instance where it has paid off since. "*The 30%-mark check has caught two slips early since.*"

### Bug 16 — the rescue narrative

**Failure:** *Develop the Best* scores capability change in the other person. "I fixed it for him" produces zero ticks on develop-others.

**Source:** Camille Fournier, *The Manager's Path* (2017), Chapter on coaching; Tanya Reilly, *The Staff Engineer's Path* (2022) on multiplier patterns.

**Fix:** see [Teamwork §3](../question-bank/03-teamwork-collaboration.md#3-tell-me-about-a-time-you-helped-a-teammate-solve-a-technical-challenge). "*I sat with him for 90 minutes. I asked him to walk through his hypothesis. We turned on lock-wait logging together; he saw the lock inversion himself. He wrote the fix. Two months later he diagnosed a similar locking issue independently in 30 minutes.*"

### Bug 17 — bluffed precision

**Failure:** Suspiciously precise numbers without context. Trained interviewers will probe ("how was the customer-satisfaction score measured?") and the answer collapses.

**Source:** Daniel Kahneman, *Thinking, Fast and Slow* (2011), Chapters 21–22 on calibration. Bar-raiser interviewer training at Amazon explicitly checks for confidence-calibration mismatches.

**Fix:** Round honestly. "*p99 dropped from roughly 200ms with spikes to a flat ~180ms. Error rate dropped meaningfully — I'd have to check the dashboard for the exact number, but it was about 0.3% to under 0.05%.*"

### Bug 18 — silently substituted question

**Failure:** The candidate substituted a single-team story for a multi-team question without flagging it. The rubric's specific competency (cross-team influence) gets zero ticks.

**Source:** DDI's *Targeted Selection®* trainer notes; the explicit "may I tell you a different story?" handoff pattern.

**Fix:** Flag the substitution. "*I haven't led a formal multi-team initiative. The closest I have is leading a migration that involved coordination with one adjacent team — the iOS team — for a contract change. May I describe that instead?*"

### Bug 19 — fabrication that fails follow-up

**Failure:** Surface answer is precise; follow-up is vague. The pattern is the strongest signal of fabrication, and bar raisers are trained to spot it.

**Source:** Schmidt & Hunter, *Psychological Bulletin* 124(2), 1998 on consistency-across-probes as a validity check; structured-interview training programs at Amazon and Google explicitly note this pattern.

**Fix:** Tell real stories. If you don't have a real production-incident story at the scale you're claiming, scale the claim down to what you actually have. "*The largest incident I've personally led was a 22-minute checkout-service degradation affecting ~18k requests. Here's the timeline: …*"

### Bug 20 — scope-level mismatch

**Failure:** The story is great at IC4 and weak at staff. Staff-level *organizational change* scores at multi-team scope, not within-team-process scope.

**Source:** Will Larson, *Staff Engineer* (2021), on staff-engineer scope and impact; Tanya Reilly, *The Staff Engineer's Path* (2022).

**Fix:** Pick a story with multi-team or organizational scope. "*Last year I drove the cross-org adoption of a standardized incident-response template. Started with my own team; partnered with two adjacent teams; ran a write-up that the head of engineering circulated to the whole org; six months later the template is the default in our incident-response tooling.*"

### Bug 21 — "I won" with no commit

**Failure:** Going around the manager fails *Earn Trust* and *Disagree and Commit*. Even though "I was right," the trust signal is broken.

**Source:** Amazon Leadership Principle #13, *Have Backbone; Disagree and Commit*; Camille Fournier, *The Manager's Path* (2017), Chapter on managing up.

**Fix:** see [Conflict & Disagreement §2](../question-bank/04-conflict-disagreement.md#2-tell-me-about-a-disagreement-with-your-manager). "*I made the strongest case I could. He kept the original plan. I executed it as designed. The feature did have an issue at 100% rollout that staged would have caught; I noted it in the postmortem without saying 'I told you so.' The next time we had a similar choice, he came to me first.*"

### Bug 22 — confident bluff on the unknown

**Failure:** The follow-up specifically asked about the *VP's* response; the answer pivots to praise without engaging with what was actually asked. Trained bar raisers will re-ask the same question and the bluff becomes obvious.

**Source:** Kahneman, *Thinking, Fast and Slow* (2011), on the "answering an easier question" substitution; bar-raiser training programs flag this as a calibration check.

**Fix:** Honest engagement. "*She pushed back on the cost number — said our engineering-hour estimate seemed low for a 2-quarter migration. I named the assumptions behind it; one of them she challenged — I hadn't budgeted for the operational learning curve. I revised the estimate up by 30% in a follow-up doc and she signed off.*"

### Bug 23 — no honest gap

**Failure:** Implausibly polished cadence. Bar raisers will probe ("what's the most recent paper you read?") and the depth won't be there. The lack of any abandoned practice is itself a tell.

**Source:** Cal Newport, *Deep Work* (2016) — sustainable depth requires bounded scope; Anders Ericsson, *Peak* (2016) on the limits of sustained deliberate practice.

**Fix:** see [Adaptability & Learning §3](../question-bank/06-adaptability-learning.md#3-how-do-you-enhance-your-technical-knowledge). Include a practice you tried and dropped. "*I tried a one-paper-a-week cadence for six months and the marginal value dropped — I was reading without retaining. I switched to one-paper-a-month with explicit notes; that's what's actually sustained.*"

### Bug 24 — feedback story that's actually a code review

**Failure:** The "feedback you gave someone" question is asking about *interpersonal* feedback (capability, behavior, growth), not technical code review. Wrong category entirely.

**Source:** Lara Hogan, *Resilient Management* (2019), on the SBI feedback model and the distinction between technical critique and interpersonal feedback; Kim Scott, *Radical Candor* (2019).

**Fix:** see [Failure & Feedback §2](../question-bank/09-failure-and-feedback.md#2-tell-me-about-feedback-you-gave-someone) for an SBI-formatted interpersonal-feedback worked answer.

---

## Related

- [The STAR Framework](../foundations/03-star-framework.md) — the structural rubric most of these bugs violate
- [Common Myths and Misconceptions](../foundations/02-myths-and-misconceptions.md) — including the myths that produce many of these failure modes
- [Story Banking](../foundations/04-story-banking.md) — fabrication and silent substitution often trace back to a thin bank
- [Advanced Answering Techniques](../foundations/05-advanced-answering-techniques.md) — show-don't-tell, follow-up survival, recovery scripts
- [Failure & Feedback question bank](../question-bank/09-failure-and-feedback.md) — covers the failure-question rubric in detail

## References

- David C. McClelland, "Testing for Competence Rather Than for Intelligence," *American Psychologist* 28(1), 1973
- Frank L. Schmidt & John E. Hunter, "The Validity and Utility of Selection Methods in Personnel Psychology," *Psychological Bulletin* 124(2), 1998
- Daniel Kahneman, *Thinking, Fast and Slow*, FSG, 2011 — Chapters 21–22 on structured-interview rationale and the "answering an easier question" substitution
- Carol S. Dweck, *Mindset: The New Psychology of Success*, Random House, 2006
- Anders Ericsson & Robert Pool, *Peak: Secrets from the New Science of Expertise*, Houghton Mifflin Harcourt, 2016
- Donald Schön, *The Reflective Practitioner*, Basic Books, 1983
- Steve McConnell, *Software Estimation: Demystifying the Black Art*, Microsoft Press, 2006
- Donella Meadows, *Thinking in Systems*, Chelsea Green, 2008
- Lara Hogan, *Resilient Management*, A Book Apart, 2019
- Camille Fournier, *The Manager's Path*, O'Reilly, 2017
- Will Larson, *An Elegant Puzzle*, Stripe Press, 2019, and *Staff Engineer*, 2021
- Tanya Reilly, *The Staff Engineer's Path*, O'Reilly, 2022
- Kim Scott, *Radical Candor*, 2nd ed., St. Martin's Press, 2019
- Brené Brown, *Dare to Lead*, Random House, 2018
- Cal Newport, *Deep Work*, Grand Central, 2016
- Gayle Laakmann McDowell, *Cracking the Coding Interview*, 6th ed., CareerCup, 2015
- Lewis C. Lin, *Decode and Conquer*, 4th ed., 2020
- Strunk & White, *The Elements of Style*, 4th ed., Pearson, 2000
- Amazon, ["Leadership Principles"](https://www.amazon.jobs/content/en/our-workplace/leadership-principles)
- Google re:Work, ["Hiring" guides](https://rework.withgoogle.com/subjects/hiring/)
- Development Dimensions International, *Targeted Selection®* trainer materials (DDI publishes excerpts at `ddiworld.com`)
