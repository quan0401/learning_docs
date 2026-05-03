---
title: "Conflict & Disagreement — Conflict with a Teammate, Disagreement with Manager, Difficult Colleague"
date: 2026-05-03
updated: 2026-05-03
tags: [behavioral-interviews, question-bank, conflict, disagreement, earn-trust]
---

# Conflict & Disagreement — Conflict with a Teammate, Disagreement with Manager, Difficult Colleague

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `behavioral-interviews` `question-bank` `conflict` `disagreement` `earn-trust`

---

## Table of Contents

- [Summary](#summary)
- [1. "Tell Me About a Conflict with a Teammate"](#1-tell-me-about-a-conflict-with-a-teammate)
- [2. "Tell Me About a Disagreement with Your Manager"](#2-tell-me-about-a-disagreement-with-your-manager)
- [3. "Tell Me About a Difficult Colleague"](#3-tell-me-about-a-difficult-colleague)
- [Related](#related)
- [References](#references)

---

## Summary

The conflict question is asked in nearly every behavioral round. It is also one of the easiest places to disqualify yourself — bashing the other person, escalating prematurely, or claiming you've never had a conflict are all reliable no-hire signals. Strong answers in this category share three properties: **a real disagreement with technical or product substance**, **direct private resolution that didn't require manager involvement until it had to**, and **respect for the other party preserved through the answer**. The Amazon LP "Disagree and Commit" is the cleanest mental model — disagree fully and visibly while the decision is open, commit fully once it's made, even if you lost.

---

## 1. "Tell Me About a Conflict with a Teammate"

### 1.1 What this question is really asking

Whether you can hold technical or design disagreements without making them personal, and whether you can resolve them without manager intervention. The interviewer is also screening for whether you'd be one of the people they have to manage relationships around if you join.

### 1.2 The competency signal

- **Earn Trust.** Direct, private, professional resolution.
- **Disagree and Commit.** Held the disagreement, committed once decided.
- **Have Backbone, Disagree and Commit** (Amazon LP).
- **Self-awareness.** Did you check whether you might be wrong?

### 1.3 STAR worked example

> **[S]** *Earlier this year my tech lead and I disagreed about the rollout strategy for the idempotency change. He wanted a fast cutover with a feature flag at 100% within a week; I wanted a 4-week staged rollout with shadow comparison.*
>
> **[T]** *I was the project lead. The disagreement was real — both approaches had legitimate trade-offs, and I genuinely wasn't sure I was right. My task was to either change his mind or change mine, and to do it without dragging the team into a side-taking situation.*
>
> **[A]** *I asked him for a 30-minute 1:1, separately from the team standup. I came in with what I thought were the three failure modes my staged rollout was designed to catch — and I asked him what failure modes he thought the fast cutover was acceptable for.*
>
> *Two of his three reasons were strong. He pointed out that the dual-write path I was proposing had its own failure surface — clock skew between the API and DB, which I'd hand-waved. The third reason was weaker — he wanted to clear the project from the sprint, which was process-driven not risk-driven.*
>
> *I went away, ran a small load test that surfaced the clock-skew issue I'd dismissed, and came back the next day with a revised plan: a 2-week staged rollout (compromise on speed) with explicit clock-skew handling (his concern, addressed). I also acknowledged that the original 4-week timeline had been more conservative than the data justified.*
>
> *He agreed. We shipped on the revised timeline. Importantly: I never went to my manager about this, and the team standup remained a coordination forum, not a debate forum.*
>
> **[R]** *Shipped successfully. The clock-skew handling caught two real issues during shadow — both would have been customer-visible without it. Afterwards I added a "what failure modes are we accepting?" question to my own design-review checklist. My tech lead and I worked together more cleanly on the next two projects; he later told me the way I'd handled this one had built trust.*

### 1.4 Common pitfalls

- **No real disagreement.** "We disagreed about whether to use a hash map" — too small to be a real conflict, signals you don't have one.
- **Escalating to the manager too early.** Bar raisers will probe; if your first move is escalation, the score drops.
- **Casting the other person as wrong.** Strong answers acknowledge the legitimate part of their position.
- **Not changing your mind on anything.** If you "won" the entire disagreement, the story sounds rehearsed.
- **No team-protection move.** A good IC keeps the disagreement out of the team channel and the standup.

### 1.5 Follow-ups to expect

- "What if he hadn't agreed to the revised plan?"
- "When would you have escalated?"
- "What did you change your mind about?"
- "Tell me about a time you held the disagreement and lost."

### 1.6 Story-bank tags

Earn Trust · Disagree and Commit · Self-awareness · Have Backbone

---

## 2. "Tell Me About a Disagreement with Your Manager"

### 2.1 What this question is really asking

Whether you can push back upward without burning the relationship. The trap: candidates either (a) avoid this question by claiming they've never disagreed with their manager (low signal — implausible), or (b) describe a disagreement they "won" by going around the manager (poor signal — disloyalty risk).

### 2.2 The competency signal

- **Have Backbone.** Will you push back on a decision you think is wrong?
- **Disagree and Commit.** Will you commit fully if the manager keeps the call?
- **Earn Trust.** Did you handle the disagreement in a way that preserved the relationship?
- **Judgment.** Was the issue worth disagreeing about?

### 2.3 STAR worked example

> **[S]** *Two years ago my engineering manager wanted to ship a new feature behind a binary on/off flag. I thought we needed staged rollout (5%, 25%, 50%, 100%) with explicit gates. The feature touched the payments path and a regression would mean lost revenue — and a binary flag gives no mid-rollout signal.*
>
> **[T]** *He was my manager and the decision was technically his to make. My task was to make the strongest case I could, then commit either way.*
>
> **[A]** *I scheduled a 1:1 — the next regular one, not an emergency. I came in with two artifacts: a one-page note describing two real recent incidents on the team where staged rollout had caught issues a binary flag would have missed, and a rough estimate of the engineering cost (about 1 day of work) of adding the gates.*
>
> *I made the case directly. He pushed back on two points: he was concerned about adding 1 day to a tight timeline, and he didn't want me to build infrastructure we'd only use once.*
>
> *On the first, I named that 1 day vs the cost of a regression on the payments path was an asymmetric trade. On the second, I argued the staged-rollout pattern was generally useful and we'd reuse it — and offered to extract it as a reusable piece if I was building it anyway.*
>
> *He thought about it for a day and said yes. He told me later that the case I made — specifically the asymmetric trade-off framing — was what tipped it.*
>
> **[R]** *Shipped behind staged rollout. The 25% gate caught a real issue — a 0.4% increase in payment-failure rate that would have been a measurable revenue loss at 100%. We rolled back, fixed, and re-rolled within 48h. The staged-rollout helper became a reusable piece on the team, used 5+ times since. My manager and I had a stronger working relationship on hard calls afterward.*

### 2.4 The "I disagreed and committed" story

If you did *not* prevail in the disagreement, the answer's structure is the same up to the action — and then you commit cleanly.

> *"He kept the original plan. I went back to the team and didn't reopen the debate; my own role was to make the case, and once the decision was made I executed the original plan as if I'd designed it. The feature shipped on the binary flag. We did have a small issue at 100% rollout that staged would have caught — I noted it in the postmortem without saying 'I told you so.' The next time we had a similar choice, my manager came to me first and asked what I thought."*

This scores better than a story where you "won," because it shows the discipline rubrics actually want.

### 2.5 Common pitfalls

- **Going over the manager's head.** "I went to his manager" — disloyalty signal.
- **No commit after the loss.** "I kept arguing" or "I quietly sandbagged the project" — both score poorly.
- **No artifacts.** Pure rhetorical persuasion is weaker than data-backed.
- **Trivial disagreement.** "We disagreed on a variable name" — too small.
- **Claiming no disagreements.** "My manager and I always agree" — implausible, scores low on backbone.

### 2.6 Follow-ups to expect

- "What if he hadn't agreed?"
- "Have you disagreed with him and lost?" — be ready with the loss story
- "Did he ever push back on the way you raised it?"
- "How did this affect your relationship long-term?"

### 2.7 Story-bank tags

Have Backbone · Disagree and Commit · Earn Trust · Judgment

---

## 3. "Tell Me About a Difficult Colleague"

### 3.1 What this question is really asking

Whether you can describe a hard interpersonal situation without bashing, and whether you have the empathy to consider what made the colleague difficult from their side. **This question is a trap.** The interviewer is largely listening for whether you complain. Most candidates do.

### 3.2 The competency signal

- **Earn Trust.** Discretion and respect for the other party.
- **Empathy.** Can you put yourself in the other person's position?
- **Self-awareness.** Did you consider how *you* might have contributed?
- **Have Backbone (limited).** Did you address it directly when needed?

### 3.3 The reframe — don't take the bait

The strongest answers reframe the question. The colleague isn't *difficult*; they have a *style or constraint that didn't mesh well with yours*, and you adapted.

### 3.4 STAR worked example

> **[S]** *In my last role I worked closely with a senior engineer on an adjacent team — I'll call him $name. $name had a reputation on the team for being hard to work with: he was extremely direct in code reviews, often left blunt comments, and was known to push back hard on design proposals.*
>
> **[T]** *I needed his review on a design doc that touched both teams' systems, and I'd heard from peers that they avoided his reviews because the feedback stung. My task was to get the doc through review and to actually get value from his input.*
>
> **[A]** *I started by reading three of his recent code-review threads — both ones he'd been blunt on and ones he hadn't. I noticed his pattern: when he was blunt, he was almost always technically right and the bluntness was about specificity, not hostility. He just didn't soften his language.*
>
> *I went into the review with that frame. I asked him for a 30-minute walk-through rather than async comments — partly because in person his tone was less prone to seeming hostile, and partly because I genuinely wanted to learn from him.*
>
> *He gave me hard feedback. Two things were genuinely wrong with my design and I changed them. One thing he objected to I held my ground on — I had context he didn't, and I explained it; he agreed once he saw it.*
>
> *Afterward I sent him a short note: "thanks, the locking issue you flagged was the kind of thing I wouldn't have caught — the doc is much stronger because of your review." This wasn't sucking up; it was true and worth saying.*
>
> **[R]** *The design shipped. I now go to $name proactively when I want a hard review, and several other engineers on my team started doing the same once I described the experience. I learned that "difficult" usually means "communication-style mismatch" or "hard truths I don't want to hear" — and that the technical signal in those reviews was the strongest available.*

### 3.5 Common pitfalls

- **Bashing.** "He was just rude all the time." Doesn't matter if true — scores poorly.
- **No adaptation.** "I just put up with him." Misses the empathy and self-awareness signal.
- **No technical substance.** "He was difficult and we never got anything done" — what was actually at stake?
- **Casting yourself as the victim.** Even if true, doesn't score well.
- **Generic complaints.** "He was negative" — what specifically did he do?
- **Outing the person too clearly.** Don't name the person, the team, or details that identify them; the interviewer might know them.

### 3.6 Follow-ups to expect

- "What if his bluntness *had* been hostile rather than just direct?" — willingness to draw a line
- "When would you have escalated?"
- "Have you been difficult to work with? In what way?" — strong test of self-awareness
- "What did you change about your own approach?"

### 3.7 Story-bank tags

Earn Trust · Empathy · Self-awareness · Adaptation

---

## Related

- [Teamwork & Collaboration](03-teamwork-collaboration.md) — adjacent: the underperforming-teammate question is conflict-adjacent
- [Failure & Feedback](09-failure-and-feedback.md) — adjacent: receiving and giving hard feedback
- [Communication Skills](10-communication.md) — SBI feedback pattern referenced
- [The STAR Framework](../foundations/03-star-framework.md) — conflict stories particularly need explicit trade-off framing

## References

- Amazon, ["Leadership Principles"](https://www.amazon.jobs/content/en/our-workplace/leadership-principles) — LP #13: *Have Backbone; Disagree and Commit*
- Lara Hogan, *Resilient Management*, 2019 — the BICEPS needs framework (Belonging, Improvement, Choice, Equality, Predictability, Significance) — useful for diagnosing the human side of "difficult colleague"
- Kim Scott, *Radical Candor*, 2nd ed., 2019 — care-personally-challenge-directly; the conflict-handling baseline behind much of the Amazon LP
- Susan Scott, *Fierce Conversations*, 2002 — direct-conversation patterns
- Roger Fisher & William Ury, *Getting to Yes*, 3rd ed., 2011 — interest-based negotiation; the "go to the balcony" move applicable to manager disagreement
- Camille Fournier, *The Manager's Path*, 2017 — chapter on managing up, applicable to disagreement-with-manager stories
