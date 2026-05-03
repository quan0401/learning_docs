# Behavioral Interviews Documentation Index — Learning Path

A structured path through behavioral / competency interviewing for backend engineers. Treats interviews as a **signal-extraction process** rather than a memorization exercise: the interviewer is collecting evidence of behaviors against a competency rubric, and your job is to deliver that evidence in a form they can score. Grounded in the original Behavioral Event Interviewing literature (McClelland, Spencer & Spencer, DDI) and in publicly documented modern rubrics (Amazon Leadership Principles, Google's Project Oxygen findings, the kind of leveling guides Lara Hogan and Will Larson have published).

Cross-references to the [Low-Level Design learning path](../low-level-design/INDEX.md) (which covers OOD interview *technique* — entity modeling, pattern selection, six-step framework), the [System Design learning path](../system-design/INDEX.md) (HLD interview frameworks and case studies), the [Java learning path](../java/INDEX.md), the [TypeScript learning path](../typescript/INDEX.md), and the [Go learning path](../golang/INDEX.md). This path is deliberately **interview-stage orthogonal** to the technical paths: a complete loop typically pairs an HLD round, an LLD/coding round, and a behavioral round, and most candidates lose offers on the third even though they prepare only for the first two.

**Markers:** **★** = core must-learn (every backend engineer interviewing at IC4 and above will be asked these). **○** = supporting deep-dive (company-specific frameworks, senior/staff signal, reverse interviewing). Internalize all ★ before going deep on ○.

---

## Tier 1 — Foundations: How Behavioral Interviews Actually Work

The mental model first. What the interviewer is scoring, the framework that organizes the answer (STAR), the asset you build *before* the interview (story bank), and the techniques that separate a competent answer from a memorable one.

1. [★ What Are Behavioral Interviews — Signal, Calibration, and What Interviewers Actually Score](foundations/01-what-are-behavioral-interviews.md) — Behavioral Event Interviewing origin (McClelland 1973), past-behavior-predicts-future-behavior premise, competency rubrics, calibrated scoring, what trained interviewers literally write down _(2026-05-03)_
2. [★ Common Myths and Misconceptions — What This Round Is Not](foundations/02-myths-and-misconceptions.md) — "soft skills don't matter for ICs", "memorize five answers", "senior-only concern", "be authentic = be unprepared", "the interviewer wants the right answer" _(2026-05-03)_
3. [★ The STAR Framework — Situation, Task, Action, Result](foundations/03-star-framework.md) — origin (DDI's Behavioral Interviewing), each component's job, the action-heavy ratio (S+T ≤ 25%, A ≥ 60%, R ≥ 15%), SBI/CAR variants, common failure modes _(2026-05-03)_
4. [★ Story Banking — Mining and Maintaining Your Story Bank](foundations/04-story-banking.md) — the story-mining process (project-by-project audit), 6–10 stories that map to many competencies, the competency-coverage matrix, story versioning, story refresh cadence _(2026-05-03)_
5. [★ Advanced Answering Techniques — Show Don't Tell, Unexpected Questions, Follow-ups](foundations/05-advanced-answering-techniques.md) — show-don't-tell (specifics over adjectives), the recovery script for blank-mind moments, the follow-up tree (5 Whys directed at *you*), redirection without dodging _(2026-05-03)_

---

## Tier 2 — Question Bank: Worked Answers by Category

One doc per category. Each doc covers 2–4 related prompts with the same internal template: what the question is really asking, the competency signal, a STAR-formatted worked answer, common pitfalls, follow-ups to expect, and the story-bank tags the answer fits.

6. [★ Career Oriented Questions — Tell Me About Yourself, Why Leaving, Why This Company, Next-Role Priorities](question-bank/01-career-oriented.md) — the 90-second self-pitch, the diplomatic departure narrative, doing-your-homework on the target company, alignment between your next-role criteria and the role on offer _(2026-05-03)_
7. [★ Project Based Questions — Most-Proud Project, Choosing Technologies, Leading a Project](question-bank/02-project-based.md) — picking a project that scales to follow-ups, the technology-decision narrative (constraints → options → trade-offs → outcome), what "leading" actually means at IC level _(2026-05-03)_
8. [★ Teamwork & Collaboration — Cross-Team, Underperforming Teammate, Helping a Teammate, Mentoring](question-bank/03-teamwork-collaboration.md) — influence without authority, giving feedback that lands, the difference between mentoring and coaching, when "I helped" actually means "I rescued" (and why that's a red flag) _(2026-05-03)_
9. [★ Conflict & Disagreement — Conflict with a Teammate, Disagreement with Manager, Difficult Colleague](question-bank/04-conflict-disagreement.md) — disagree-and-commit (Amazon LP #13), the difference between conflict and dysfunction, escalation hygiene, the "difficult colleague" trap (don't bash) _(2026-05-03)_
10. [★ Problem Solving — Toughest Technical Problem, Production Incident, Decision Under Ambiguity, Trade-off Made](question-bank/05-problem-solving.md) — picking a problem with a non-obvious solution, postmortem-quality incident narratives, decision-making under incomplete information, naming the trade-off explicitly _(2026-05-03)_
11. [★ Adaptability & Learning — Quickly Learning New Tech, Outside Comfort Zone, Enhancing Technical Knowledge](question-bank/06-adaptability-learning.md) — the "I learned X in two weeks" answer that doesn't sound like a lie, the genuinely-new-territory story, your structured learning practice (not just "I read blogs") _(2026-05-03)_
12. [★ Time Management & Prioritization — Multiple Deadlines, Missed a Deadline](question-bank/07-time-management.md) — explicit prioritization frameworks (RICE, MoSCoW, eisenhower), the missed-deadline story (a real one, owned, with a system fix) _(2026-05-03)_
13. [★ Leadership & Initiative — Leading Without Being Asked, Going Above and Beyond](question-bank/08-leadership-initiative.md) — IC-level leadership signals (technical direction, unblocking peers, codifying practice), the difference between "above and beyond" and "scope creep" _(2026-05-03)_
14. [★ Handling Failure & Feedback — Critical Feedback Received, Feedback Given, Delivering Under Challenge, A Time You Failed](question-bank/09-failure-and-feedback.md) — accepting feedback without flinching, SBI for delivering feedback, choosing a *real* failure (not a humble-brag), what changed afterwards _(2026-05-03)_
15. [★ Communication Skills — Explaining to Non-Technical, Presenting to Leadership](question-bank/10-communication.md) — audience-aware explanation (analogy ladder, Feynman test), the BLUF/pyramid principle for leadership audiences, handling the hostile question _(2026-05-03)_

---

## Tier 3 — Company-Specific Frameworks _(planned)_

Not every loop scores against a generic competency model. Several large employers ship public, internally-calibrated rubrics. Mapping your stories to the rubric the interviewer is literally consulting is the highest-leverage prep you can do.

- ○ Amazon Leadership Principles — The 16 LPs, "examples in the form of stories," the bar raiser, written-narrative-style answers _(planned)_
- ○ Google "Googleyness" and Project Oxygen — what re:Work actually publishes, the Hiring Committee's rubric, GCA / Role-Related Knowledge / Leadership / Googleyness _(planned)_
- ○ Meta Execution / Drive / Direction — the four-pillar rubric, how E5/E6/E7 levels read in the same question _(planned)_
- ○ Microsoft Model-Coach-Care — the manager-track lens, individual-contributor adaptation _(planned)_
- ○ Stripe / Netflix / Airbnb — published values where they substitute for a behavioral rubric _(planned)_

---

## Tier 4 — Senior & Staff Signal _(planned)_

The same questions, asked of an L6+/staff candidate, score on a different rubric. Scope, ambiguity, and organizational impact dominate; "I shipped a feature" stops being enough.

- ○ Scope and impact at staff level — the multi-team, multi-quarter story; how to talk about influence vs delivery _(planned)_
- ○ Ambiguity questions — "tell me about a time the requirements weren't clear" answered at staff scope _(planned)_
- ○ Dealing with VPs and execs — disagreement upward, presenting trade-offs to a non-technical CFO, surviving steering committee _(planned)_
- ○ Org-design and headcount — "tell me about a time you scaled a team / split a team / hired against a gap" _(planned)_
- ○ Technical strategy — RFC-driving, multi-quarter platform decisions, deprecation campaigns _(planned)_

---

## Tier 5 — Reverse Interviewing _(planned)_

The 5–10 minutes at the end where *you* ask the questions. Most candidates waste it. It is also the strongest signal you have on whether to accept an offer.

- ○ Questions to ask the interviewer at each stage — recruiter, hiring manager, peer IC, skip-level, bar raiser _(planned)_
- ○ Reading red flags from interviewer answers — vague tenure data, evasive on-call answers, no examples of growth _(planned)_
- ○ Compensation discussion mechanics — when to discuss numbers, what to ask the recruiter, levelling questions _(planned)_
- ○ Negotiation after the offer — competing offers, the "what would change your mind" script, declining gracefully _(planned)_

---

## Quick Reference by Topic

### Foundations

- [What Are Behavioral Interviews](foundations/01-what-are-behavioral-interviews.md)
- [Common Myths and Misconceptions](foundations/02-myths-and-misconceptions.md)
- [The STAR Framework](foundations/03-star-framework.md)
- [Story Banking](foundations/04-story-banking.md)
- [Advanced Answering Techniques](foundations/05-advanced-answering-techniques.md)

### Question Bank — Career & Projects

- [Career Oriented Questions](question-bank/01-career-oriented.md)
- [Project Based Questions](question-bank/02-project-based.md)

### Question Bank — People

- [Teamwork & Collaboration](question-bank/03-teamwork-collaboration.md)
- [Conflict & Disagreement](question-bank/04-conflict-disagreement.md)
- [Communication Skills](question-bank/10-communication.md)

### Question Bank — Execution

- [Problem Solving](question-bank/05-problem-solving.md)
- [Adaptability & Learning](question-bank/06-adaptability-learning.md)
- [Time Management & Prioritization](question-bank/07-time-management.md)

### Question Bank — Ownership

- [Leadership & Initiative](question-bank/08-leadership-initiative.md)
- [Handling Failure & Feedback](question-bank/09-failure-and-feedback.md)

### Competency Tag Matrix

| Competency | Best-fit docs |
|---|---|
| Ownership | Leadership & Initiative · Failure & Feedback · Problem Solving |
| Bias for Action | Leadership & Initiative · Time Management · Adaptability |
| Influence Without Authority | Teamwork · Conflict · Communication |
| Customer / User Obsession | Project Based · Communication |
| Deal With Ambiguity | Problem Solving · Adaptability · Project Based |
| Earn Trust / Feedback | Failure & Feedback · Conflict · Teamwork |
| Deliver Results | Project Based · Time Management · Problem Solving |
| Hire and Develop / Mentor | Teamwork (mentoring) · Failure & Feedback (giving) |
| Are Right, A Lot | Problem Solving · Project Based · Conflict (disagree-and-commit) |

### Company-Specific Frameworks _(planned)_

- Amazon LPs, Google "Googleyness" / Project Oxygen, Meta Execution-Drive-Direction, Microsoft Model-Coach-Care

### Senior / Staff Signal _(planned)_

- Scope at staff level, ambiguity, exec-facing communication, org design, technical strategy

### Reverse Interviewing _(planned)_

- Questions to ask, reading red flags, compensation mechanics, negotiation

---

## Bug Spotting

Active-recall practice docs. Each presents 22+ broken snippets organized by difficulty (Easy / Subtle / Senior trap), with one-line `<details>` hints inline and full root-cause + fix in a Solutions section. Every bug cites a real reference (book chapter, posted rubric, talk, blog post, or HR/IO-psych research) — fabrication is forbidden. Use these to pressure-test STAR-answer hygiene after working through the tiers above.

- [Behavioral Answers — Bug Spotting](practice/behavioral-answer-bug-spotting.md) ★ — _(2026-05-03)_

---

## Cross-Path Reading

- [Low-Level Design — Interview Method](../low-level-design/INDEX.md) — the technical-OOD-interview counterpart: entity modeling, pattern selection, the six-step framework
- [System Design — Interview Framework](../system-design/INDEX.md) — the HLD-interview counterpart: requirements gathering, capacity estimation, deep dives, trade-off justification
- [Java](../java/INDEX.md), [TypeScript](../typescript/INDEX.md), [Go](../golang/INDEX.md) — your story bank's raw material lives here; the projects you draw STAR stories from are usually in your primary stack

---

## References — Path-Level Sources

(Each individual doc cites its own primary sources. Listed here are the umbrella works underlying the path.)

- David C. McClelland, "Testing for Competence Rather Than for Intelligence," *American Psychologist* 28(1), 1973 — the paper that started competency-based interviewing
- Lyle M. Spencer & Signe M. Spencer, *Competence at Work: Models for Superior Performance*, Wiley, 1993 — the foundational competency catalogue still echoed in modern rubrics
- Development Dimensions International (DDI), *Behavioral Interviewing® Targeted Selection®* materials — origin of the STAR acronym in industry use
- Tom Janz, *Behavior-Based Interviewing*, 1986 — adjacent methodology widely referenced in HR practice
- Google re:Work, ["Project Oxygen"](https://rework.withgoogle.com/guides/managers-identify-what-makes-a-great-manager/) and ["Project Aristotle"](https://rework.withgoogle.com/guides/understanding-team-effectiveness/) — what Google publicly published about manager and team effectiveness, which feeds the "Googleyness" rubric
- Amazon, ["Leadership Principles"](https://www.amazon.jobs/content/en/our-workplace/leadership-principles) — the 16 publicly listed LPs and how the bar raiser uses them
- Will Larson, *Staff Engineer: Leadership Beyond the Management Track*, 2021 — the staff-level archetypes and signal targets
- Gergely Orosz, *The Software Engineer's Guidebook*, 2024 — modern levelling and interview-loop reality
- Lara Hogan, *Resilient Management*, 2019 — feedback-equation and BICEPS frameworks frequently cited in modern behavioral rubrics
