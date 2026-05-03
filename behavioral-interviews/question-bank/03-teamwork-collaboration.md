---
title: "Teamwork & Collaboration — Cross-Team, Underperforming Teammate, Helping a Teammate, Mentoring"
date: 2026-05-03
updated: 2026-05-03
tags: [behavioral-interviews, question-bank, teamwork, collaboration, mentoring, influence]
---

# Teamwork & Collaboration — Cross-Team, Underperforming Teammate, Helping a Teammate, Mentoring

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `behavioral-interviews` `question-bank` `teamwork` `collaboration` `mentoring` `influence`

---

## Table of Contents

- [Summary](#summary)
- [1. "Tell Me About a Cross-Team Collaboration"](#1-tell-me-about-a-cross-team-collaboration)
- [2. "Tell Me About a Teammate Who Wasn't Contributing Enough"](#2-tell-me-about-a-teammate-who-wasnt-contributing-enough)
- [3. "Tell Me About a Time You Helped a Teammate Solve a Technical Challenge"](#3-tell-me-about-a-time-you-helped-a-teammate-solve-a-technical-challenge)
- [4. "Tell Me About Mentoring or Coaching Someone"](#4-tell-me-about-mentoring-or-coaching-someone)
- [Related](#related)
- [References](#references)

---

## Summary

Teamwork questions score *Influence Without Authority*, *Earn Trust*, and the "develop others" axis of leadership. The biggest failure in this category is **the rescue narrative**: candidates frame their "help" as having saved someone, which scores poorly on both humility and on the "develop the person, not the deliverable" axis the rubric actually wants. Strong answers in this category foreground *what changed about the teammate's capability* (not just what got shipped), use SBI for any feedback you describe, and stay scrupulously professional when the question invites complaint.

---

## 1. "Tell Me About a Cross-Team Collaboration"

### 1.1 What this question is really asking

Whether you can drive outcomes in a context where you have no formal authority and the other side has competing priorities. This is a senior-track signal — at staff level it's nearly always asked.

### 1.2 The competency signal

- **Influence Without Authority.** Aligning peers and adjacent teams without a manager backing you.
- **Communication.** Adapting your framing to a different team's context.
- **Earn Trust.** Building credibility across a team boundary.
- **Deal With Ambiguity.** Cross-team work usually has unowned grey zones.

### 1.3 STAR worked example

> **[S]** *Last summer the payments team I was on needed to coordinate with the iOS app team to roll out a new idempotency-key contract. The two teams reported up different VPs, had different release cadences (we shipped twice a week, they shipped weekly), and had different priorities — they were mid-launch on a separate feature.*
>
> **[T]** *I was the backend lead. My task was to land the contract change in their app within 6 weeks, on a timeline that worked for both teams.*
>
> **[A]** *I started by reading the iOS team's planning docs from the last quarter and identifying their actual release pressures. I came to my first meeting with their tech lead with a one-page proposal that named three options — the smallest, fastest version of the change they'd need to make. We agreed on the smallest version in 20 minutes; the meeting could have been a debate without that prep.*
>
> *Their team had a constraint I hadn't anticipated — they were freezing app changes for two weeks around their feature launch. Rather than push, I rescoped: I built a server-side fallback that handled both the old and new contract simultaneously, so the iOS change could land in their first post-freeze release without rushing. That added 3 days of work on my side; it bought them 4 weeks of breathing room.*
>
> *I wrote a 5-line weekly status email to both teams every Monday — what changed, what's blocked, what's next — so neither side had to chase. Their tech lead later told me this was the first cross-team project she'd been part of where she didn't have to chase status.*
>
> **[R]** *The iOS change shipped in their first post-freeze release, three days after their feature launch. The dual-contract server-side path stayed in place for one quarter as a safety margin and was later removed cleanly. The status-email pattern became my default for any cross-team work; my manager later asked me to write it up as a team practice.*

### 1.4 Common pitfalls

- **Adversarial framing.** "The iOS team was being difficult" — even if true, this scores poorly. Frame the other team as having legitimate constraints you adapted to.
- **No preparation step.** "I went to the meeting and we figured it out" — misses the influence signal.
- **Not naming the rescoping or trade-off.** Cross-team work always has compromise; if your answer has none, it sounds like you got everything you wanted.
- **Vague about who did what.** "We aligned" — but how, and what specifically did you do?

### 1.5 Follow-ups to expect

- "What did the other team's tech lead push back on?"
- "How did you find their planning docs? Was that a normal thing in your org?"
- "What if the rescoping hadn't worked?"
- "How did you handle the disagreement with $teammate inside your own team?"

### 1.6 Story-bank tags

Influence Without Authority · Earn Trust · Communication · Cross-Team

---

## 2. "Tell Me About a Teammate Who Wasn't Contributing Enough"

### 2.1 What this question is really asking

Whether you can address a performance issue with a peer **without becoming part of the problem**. The trap: this question is an invitation to complain, and most candidates take it. Trained interviewers score complaining as a red flag.

### 2.2 The competency signal

- **Earn Trust.** Did you handle this with discretion?
- **Develop the Best.** Did you try to help the teammate, or just escalate?
- **Conflict-handling.** Direct, private, professional vs gossip / triangulation.
- **Self-awareness.** Did you check whether the issue was them or also a system / context problem?

### 2.3 STAR worked example

> **[S]** *Q2 last year on my team, one of my peers — a mid-level engineer I'll call $teammate — had been falling behind on PRs. Reviews from him took 3–5 days when team norm was 24 hours, and his own PRs were sitting open for over a week.*
>
> **[T]** *I was not his manager. The deliverable impact landed on me — I had two PRs blocked on his review for a week. I needed to address it without going around him to my manager prematurely.*
>
> **[A]** *Before raising it, I checked whether the issue was specific to him or general — looked at the team's PR metrics — and confirmed it was specific. I also checked whether something was going on; I knew his partner had recently had a baby and he might have load I wasn't seeing.*
>
> *I asked him for a 1:1 — coffee, not a calendar invite — and used SBI: "When PRs sit for 3–5 days (Situation), my own work gets blocked (Behavior — wait, the behavior is his, but the* impact *is on me). The impact is that I'm context-switching while waiting." I asked what was going on and what I could do.*
>
> *He told me he was overwhelmed — he was on parental leave-adjacent load and hadn't said so to the team. We agreed: he'd flag PRs he wasn't going to get to within 24 hours and I'd reroute them to other reviewers; I'd also pick up two of the items he'd been planning to take that sprint.*
>
> *I did not go to my manager. The conversation handled it.*
>
> **[R]** *PR turnaround for him recovered to 24h within a week. Two weeks later, he raised the broader load issue with the team and our manager directly, which produced a temporary scope adjustment. The "flag PRs you can't review" pattern became a team norm. I later realized I'd been carrying assumptions about why he was slow that were wrong — the issue wasn't engagement, it was load. I keep that lesson present in similar situations.*

### 2.4 Common pitfalls

- **Bashing the teammate.** "He just wasn't pulling his weight" — scores low on Earn Trust.
- **Going to the manager first.** Some interviewers will probe "did you talk to him directly?" If the answer is no, the score takes a hit.
- **Not checking your own assumptions.** A strong answer includes "I considered whether this was a load issue, not an engagement issue."
- **Casting yourself as the rescuer.** "I picked up his work for him" without his agency in the resolution scores poorly.
- **No second-order learning.** What did *you* take away about how you assess teammates?

### 2.5 Follow-ups to expect

- "What if he had pushed back when you raised it?"
- "Why didn't you escalate to your manager?"
- "What if the issue had been engagement rather than load?"
- "Tell me about a time you *did* need to escalate."

### 2.6 Story-bank tags

Earn Trust · Conflict-handling · Develop · Self-awareness

---

## 3. "Tell Me About a Time You Helped a Teammate Solve a Technical Challenge"

### 3.1 What this question is really asking

Whether you can pair / debug / coach without taking over, and whether the teammate ended up *more* capable as a result. The trap: candidates often describe rescuing the teammate by writing the code themselves. That's the opposite of what the rubric scores.

### 3.2 The competency signal

- **Develop the Best.** Did the teammate end the interaction more capable?
- **Technical depth.** Did you understand the problem well enough to teach it?
- **Empathy.** Did you adjust to the teammate's actual gap?
- **Not the rescuer trap.** "I solved it for them" is the wrong answer.

### 3.3 STAR worked example

> **[S]** *Earlier this year, a junior on my team was stuck on a Postgres locking bug — a deadlock that was firing intermittently in their migration script. They'd been on it for two days and were frustrated.*
>
> **[T]** *They asked me for help. I had two options: take it from them and fix it myself, or pair with them and walk through the diagnosis. The deliverable pressure was low — there was no immediate deadline — so I picked the second.*
>
> **[A]** *I sat with them for about 90 minutes. I started by asking them to walk me through their hypothesis: what they thought was happening, what evidence they had. They had jumped straight to "Postgres has a bug" — which is almost always wrong; I redirected to the basics: what locks were being acquired, in what order, by what queries.*
>
> *We turned on `log_lock_waits = on` together and re-ran. They could see the actual lock contention in the log. From there, I asked them to draw the lock-acquisition order on the whiteboard. They saw the inversion themselves — the migration was acquiring locks in a different order than the runtime path, which was the textbook deadlock setup.*
>
> *I let them write the fix. It was a 4-line change to lock-acquisition order. I reviewed the PR with detail comments, not just an LGTM.*
>
> **[R]** *The fix shipped, the deadlock didn't recur. More importantly, two months later they hit a similar locking issue on a different surface and diagnosed it themselves in under 30 minutes — they told me afterwards. The "draw the lock graph" pattern is now in their toolkit. I started keeping a running doc of "diagnosis recipes" I share with juniors when they hit a class of bug for the first time; it's about 10 entries deep now.*

### 3.4 Common pitfalls

- **The rescue narrative.** "I took it over and fixed it." This scores low on develop-others.
- **Not naming what they learned.** "I helped them solve it" — but what changed in *them*?
- **The wrong gap diagnosis.** Helping someone with a syntax issue is different from helping with a mental-model issue. Strong answers identify *what kind* of gap they were filling.
- **Passive teammate.** Stories where you do all the talking and they just nod miss the develop signal.

### 3.5 Follow-ups to expect

- "What did you do *not* tell them?" — restraint signal
- "How did you decide between fixing it yourself and pairing?"
- "What if they'd been resistant to the pairing?"
- "Tell me about a time the teammate didn't grow from your help."

### 3.6 Story-bank tags

Develop the Best · Technical depth · Empathy · Mentor / Coach

---

## 4. "Tell Me About Mentoring or Coaching Someone"

### 4.1 What this question is really asking

Sustained development of another person, not a one-off help moment. The signal: did you have a sustained mentoring relationship, did you adapt your approach to the mentee, and did the mentee actually grow.

### 4.2 The competency signal

- **Develop the Best.**
- **Earn Trust.**
- **Empathy and adaptation.**
- **Patience over time.**

### 4.3 Mentoring vs coaching — the distinction matters

| Mentoring | Coaching |
|---|---|
| Teaching a skill or domain | Helping someone solve their own problem |
| You are the expert on the topic | You are not necessarily the expert |
| Driven by knowledge transfer | Driven by questions, not answers |
| Often longer arc, often informal | Often shorter arc, often structured |

If the question says "mentoring," lean toward the knowledge-transfer story. If it says "coaching," lean toward the question-driven story. Many candidates conflate them — a small specific framing wins points.

### 4.4 STAR worked example

> **[S]** *Over the last year I've had a regular mentoring relationship with $name, a new-grad on my team. We started in her first month; she's now mid-Q4 of her first year.*
>
> **[T]** *Her manager asked me to be her informal mentor — code reviews, project unblocking, career-conversation cover. I treated this as: my job is to help her become independent on the team's tech stack and to develop her judgement on the kinds of decisions seniors make routinely.*
>
> **[A]** *I set a recurring 30-minute weekly coffee — no agenda, but with a rolling set of topics on a shared doc. Early on the topics were tactical: "how should I structure this PR?", "I'm stuck on the test setup." Mid-year they shifted: "how do I push back on a tech-lead suggestion I disagree with?", "how do I scope a project I'm being given for the first time?"*
>
> *I deliberately did not solve her problems. I asked her what she'd already tried, what she thought the trade-offs were, and where her uncertainty was. When she had a real knowledge gap I taught it directly — locking, transactions, the specifics of our service mesh. When she had a judgement gap I asked questions instead.*
>
> *Two specific moments. First: she had a disagreement with a senior on the team about a design choice. I role-played the senior with her so she could rehearse the disagree-and-commit conversation; she went into the actual meeting more confident. Second: at her 6-month review, her manager said her growth had outpaced peer hires; I'd given her honest feedback throughout the year about the gaps so the review wasn't a surprise.*
>
> **[R]** *She's now leading her own first project — a small one, scoped appropriately — and is the on-call mentor for the next new-grad joining the team. The "coffee with rolling topics" pattern is something I've kept and now do with two other juniors. The biggest thing for me: I started the year thinking I'd teach her the tech; the higher-leverage work was teaching her how to make the kinds of decisions seniors make, which I'd been doing implicitly without naming.*

### 4.5 Common pitfalls

- **One-off help dressed as mentoring.** "I helped a teammate that one time" — too thin for this question.
- **Mentoring that produced no growth.** If you can't name what changed in the mentee, the story doesn't land.
- **Self-aggrandizing.** "I taught her everything she knows" — scores poorly.
- **No second-order change in *you*.** A strong mentor learned something about mentoring from the mentee.

### 4.6 Follow-ups to expect

- "What's something you'd do differently in mentoring her?"
- "Have you mentored someone where it didn't work?"
- "How did you give her the feedback that landed hardest?"
- "What did *you* learn from mentoring her?"

### 4.7 Story-bank tags

Develop the Best · Earn Trust · Patience · Empathy · Mentor / Coach

---

## Related

- [Conflict & Disagreement](04-conflict-disagreement.md) — adjacent: when teammate issues become genuine conflicts
- [Leadership & Initiative](08-leadership-initiative.md) — adjacent: leadership stories often overlap with mentoring
- [Failure & Feedback](09-failure-and-feedback.md) — for the "feedback I gave" moments inside teamwork stories
- [Communication Skills](10-communication.md) — SBI feedback pattern referenced here

## References

- Lara Hogan, *Resilient Management*, 2019 — the SBI (Situation-Behavior-Impact) feedback pattern, the BICEPS needs framework, and the feedback equation
- Camille Fournier, *The Manager's Path*, 2017 — chapters on mentoring and on the IC-to-Tech-Lead transition
- Will Larson, *An Elegant Puzzle*, 2019 — sustained-development practice and the difference between mentoring and coaching
- Kim Scott, *Radical Candor*, 2nd ed., 2019 — care-personally-challenge-directly framework relevant to the underperforming-teammate question
- Tanya Reilly, *The Staff Engineer's Path*, 2022 — multiplier patterns at IC level (mentoring, glue work, lifting peers)
- Amazon, ["Leadership Principles"](https://www.amazon.jobs/content/en/our-workplace/leadership-principles) — *Hire and Develop the Best*, *Earn Trust*
