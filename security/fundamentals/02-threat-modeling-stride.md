---
title: "Threat Modeling with STRIDE — Designing Security In"
date: 2026-05-03
updated: 2026-05-03
tags: [security, threat-modeling, stride, design, secure-design]
---

# Threat Modeling with STRIDE — Designing Security In

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `security` `threat-modeling` `stride` `design` `secure-design`

---

## Table of Contents

- [Summary](#summary)
- [1. Why Threat Model](#1-why-threat-model)
- [2. STRIDE Categories](#2-stride-categories)
- [3. Data-Flow Diagrams and Trust Boundaries](#3-data-flow-diagrams-and-trust-boundaries)
- [4. Worked Example — A Notes Service](#4-worked-example--a-notes-service)
- [5. STRIDE Per Element](#5-stride-per-element)
- [6. From Threats to Mitigations](#6-from-threats-to-mitigations)
- [7. When to Threat Model](#7-when-to-threat-model)
- [8. STRIDE vs PASTA vs LINDDUN vs Attack Trees](#8-stride-vs-pasta-vs-linddun-vs-attack-trees)
- [9. Tooling](#9-tooling)
- [10. Common Anti-Patterns](#10-common-anti-patterns)
- [Related](#related)
- [References](#references)

---

## Summary

Threat modeling is the practice of systematically asking "what could go wrong, and what are we doing about it?" *before* shipping a feature. STRIDE is the most widely used taxonomy for asking that question — six threat categories (Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege) mapped to the four security properties they violate. This document teaches STRIDE through a worked example: a multi-tenant notes service. You'll learn to draw a data-flow diagram, mark trust boundaries, walk each element with STRIDE, and translate threats into concrete mitigations. The goal isn't to find every theoretical threat — it's to make the team's mental model of the system explicit enough that the obvious ones stop slipping through.

STRIDE was created at Microsoft in 1999 by Loren Kohnfelder and Praerit Garg, and codified in *The Security Development Lifecycle* by Howard and Lipner. It pairs naturally with data-flow diagrams: walk every process, every data store, every flow, and every external entity, and ask the six questions. Done seriously, a one-hour STRIDE session on a non-trivial design surfaces 5–15 issues that the team had not considered.

---

## 1. Why Threat Model

Most production security incidents trace back to a missing control that nobody thought to build, not to an exotic vulnerability in a library. OWASP A04 (Insecure Design) exists because static analyzers can't find what isn't there. Threat modeling is how teams find what isn't there.

The Microsoft SDL data shows that bugs caught at design time cost roughly **10× less** to fix than the same bug caught in QA, and **100× less** than one caught in production. The math gets worse if "production" means "post-incident".

Concrete payoffs:

- **Authorization gaps surface immediately.** Drawing the DFD makes the "who calls this?" question unavoidable.
- **Trust boundaries become explicit.** Every arrow that crosses a boundary needs authentication, authorization, and validation.
- **Architectural mistakes are still cheap to fix.** A wrong choice of "do auth at the gateway" vs "do auth at the service" is a meeting at design time and a quarter-long migration in production.
- **Onboarding artifact.** A current threat model is the single most useful onboarding document for a new engineer joining a service.

---

## 2. STRIDE Categories

STRIDE is an acronym for six threat classes. Each maps to a violated security property.

| Letter | Threat | Violates | Example |
|--------|--------|----------|---------|
| **S** | **Spoofing** identity | Authentication | Attacker logs in as someone else |
| **T** | **Tampering** with data | Integrity | Attacker modifies a request, file, or row in transit or at rest |
| **R** | **Repudiation** | Non-repudiation | User denies an action that the system can't prove they took |
| **I** | **Information disclosure** | Confidentiality | Attacker reads data they shouldn't |
| **D** | **Denial of service** | Availability | Attacker prevents legitimate users from using the system |
| **E** | **Elevation of privilege** | Authorization | Attacker gains capabilities they shouldn't have |

A cleaner mnemonic: STRIDE walks you through **AINCAA** — Authentication, Integrity, Non-repudiation, Confidentiality, Availability, Authorization — in scrambled order.

### 2.1 Why These Six and Not Others?

The categories are exhaustive and mostly orthogonal — every classic web vulnerability lands in at least one. SQL injection that reads other users' data is **I**; if it lets you become admin it's also **E**. Rate-limit bypass is **D**. Stolen session token is **S**. The taxonomy is useful because each category has a *different family of mitigations*.

| Mitigation family | Defeats |
|-------------------|---------|
| Strong authentication (MFA, mTLS, OAuth) | S |
| Cryptographic integrity (HMAC, signatures, TLS) | T |
| Audit logs with tamper evidence | R |
| Encryption, access control | I |
| Rate limiting, quotas, capacity planning | D |
| Authorization checks, least privilege | E |

---

## 3. Data-Flow Diagrams and Trust Boundaries

STRIDE is most useful with a **Data-Flow Diagram (DFD)**. DFDs use four element types:

| Symbol | Element | Examples |
|--------|---------|----------|
| Rectangle | **External entity** | User, third-party API, OAuth provider |
| Circle | **Process** | Service, microservice, function |
| Two parallel lines | **Data store** | Database, cache, S3 bucket, KMS |
| Arrow | **Data flow** | HTTP request, queue message, DB query |
| Dashed line | **Trust boundary** | Network perimeter, process boundary, tenant boundary |

A **trust boundary** is where the assumed level of trust changes. Crossing a trust boundary always requires:

- **Authentication** of the caller.
- **Authorization** of the action.
- **Input validation** at the receiving side.

Examples of trust boundaries:

- Internet ↔ your edge.
- Application ↔ database.
- Web tier ↔ admin tier.
- Tenant A's data ↔ tenant B's data.
- User-mode process ↔ kernel.
- Container ↔ host.

---

## 4. Worked Example — A Notes Service

A multi-tenant notes service. Users sign in via an OIDC provider, create notes, share notes with other users in their org, and search across their notes. Notes live in PostgreSQL; full-text search uses Elasticsearch; attachments live in S3.

### 4.1 The DFD

```
                 ┌────────────── Internet trust boundary ────────────┐
                 │                                                   │
   ┌────────┐    │   ┌──────────────┐        ┌──────────────────┐    │
   │  User  │────┼──▶│  CDN / WAF   │───────▶│   API Gateway    │    │
   └────────┘    │   └──────────────┘        │  (TLS terminate, │    │
                 │                           │   OIDC validate) │    │
   ┌────────┐    │                           └────────┬─────────┘    │
   │  OIDC  │◀──┐│                                    │              │
   │ (Auth0)│   ││                                    ▼              │
   └────────┘   ││                          ┌──────────────────┐     │
                ││                          │  Notes Service   │     │
                ││                          │  (Spring Boot)   │     │
                ││                          └────────┬─────────┘     │
                │└─── OIDC trust boundary ───────────┼───────────────┘
                │                                    │
                │     ┌──────── Application trust boundary ────┐
                │     │                                        │
                │     │   ┌───────────┐  ┌──────────────┐      │
                │     │   │ Postgres  │  │ Elasticsearch│      │
                │     │   │  (notes)  │  │   (search)   │      │
                │     │   └───────────┘  └──────────────┘      │
                │     │                                        │
                │     │            ┌──────┐                    │
                │     │            │  S3  │                    │
                │     │            │(blob)│                    │
                │     │            └──────┘                    │
                │     └────────────────────────────────────────┘
                │
                └─────── Tenant boundary inside the database ──
                          (logical, enforced by row-level security)
```

Trust boundaries:

- **Internet ↔ edge.** All inbound traffic is hostile until proven otherwise.
- **OIDC ↔ application.** ID tokens cross this boundary; JWKS is fetched across it.
- **Application ↔ data.** DB credentials are sensitive; queries cross.
- **Tenant ↔ tenant.** Logical inside the DB. The most violated boundary in multi-tenant SaaS.

### 4.2 What's *Not* on the Diagram (and Should Be)

Common omissions worth checking for:

- **Logs and metrics flows.** Where do logs go? Who can read them? Can a tenant's PII end up in shared logs?
- **Build/CI pipeline.** It pushes images and signs them — that's a trust boundary too.
- **Secrets management.** Where do DB credentials, OIDC client secrets, and S3 keys come from?
- **Backup and restore paths.** A backup with full data and weak ACLs is the same as the prod DB.

---

## 5. STRIDE Per Element

Walk the DFD systematically. For brevity we hit the high-value elements.

### 5.1 The User → API Gateway flow

| Threat | Concrete instance | Mitigation |
|--------|-------------------|------------|
| **S** | Attacker presents an expired or forged ID token | API gateway validates JWT signature against JWKS, checks `iss`, `aud`, `exp`, `nbf` |
| **T** | Attacker MitMs the connection and modifies request body | Enforce TLS 1.2+, HSTS preload, certificate pinning if mobile |
| **R** | User claims they didn't make a destructive request | Log request with authenticated user ID + correlation ID; immutable log store |
| **I** | Eavesdropper reads the JWT in transit | TLS; never put secrets in URL query params; `Cache-Control: no-store` on auth-bearing responses |
| **D** | Attacker floods the gateway | WAF rate limiting per IP and per token; CDN-level DDoS scrubbing; concurrency caps |
| **E** | Token replay across services with different audiences | Tight `aud` claim; short-lived access tokens (5–15 min); refresh-token rotation |

### 5.2 The Notes Service Process

| Threat | Concrete instance | Mitigation |
|--------|-------------------|------------|
| **S** | Internal service forges requests as a privileged user | mTLS between services; signed service-to-service tokens; never trust headers like `X-User-Id` |
| **T** | Note content is tampered with at rest | Postgres transaction integrity + row-level checks; if tamper evidence is required, sign rows with HMAC keyed by app secret |
| **R** | Admin changes a note and denies it | Application-level audit log keyed by actor ID; write-once log store (e.g., separate DB user with only INSERT) |
| **I** | Tenant A reads tenant B's notes via IDOR | Every query scoped by `tenant_id` *and* `user_id`; PostgreSQL Row-Level Security as defense in depth; integration tests for cross-tenant access |
| **D** | Long-running search query exhausts DB connections | Statement timeouts; connection pool limits; bulkhead pattern; query-level rate limits |
| **E** | Bug in role check lets a viewer become editor | `@PreAuthorize` on every method; centralized policy (OPA, Cedar); deny-by-default |

### 5.3 The Postgres Data Store

| Threat | Concrete instance | Mitigation |
|--------|-------------------|------------|
| **S** | Attacker connects to DB with a stolen credential | Network-level isolation (private subnet, no public IP); IAM-based auth (RDS) or short-lived credentials from Vault |
| **T** | DB compromise modifies notes silently | Backups to a separate account; periodic checksums; row-level signed columns for high-stakes data |
| **R** | DBA modifies a row with `psql` | Audit `pg_audit` extension; segregated DBA accounts; break-glass procedure with approval and log |
| **I** | Backup snapshot exfiltrated | Encryption at rest via KMS; CMK with key rotation; audit `kms:Decrypt` calls |
| **D** | Attacker fills disk with attachments references | Per-tenant quotas at app layer; storage alerts; automatic backpressure |
| **E** | App DB user has `SUPERUSER` | App user with minimum grants (`SELECT, INSERT, UPDATE, DELETE` on its tables only); separate migration user |

### 5.4 The S3 Bucket (Attachments)

| Threat | Concrete instance | Mitigation |
|--------|-------------------|------------|
| **S** | Attacker uploads malware as someone else | Pre-signed URLs scoped to user + content-type; virus scan on upload |
| **T** | Object replaced after upload | Object Lock for high-stakes data; checksum stored in Postgres |
| **R** | Object deleted, no record | S3 versioning + MFA delete; CloudTrail data events |
| **I** | Bucket public-readable (Capital One 2019) | Block Public Access at account + bucket level; SCP that denies `s3:PutBucketPolicy` allowing `*`; access via CloudFront with SigV4 |
| **D** | Cost-DoS: attacker uploads 100 GB | Per-user upload quota at app layer; max object size; lifecycle policy to delete orphaned uploads |
| **E** | App role has `s3:*` | Scoped role: `s3:GetObject` and `s3:PutObject` on the specific prefix only |

---

## 6. From Threats to Mitigations

A threat without a mitigation is a TODO. Each STRIDE finding should produce one of four outcomes:

- **Mitigate.** Implement a control. Most common.
- **Eliminate.** Redesign so the threat doesn't apply (e.g., don't accept user-supplied URLs at all → no SSRF).
- **Transfer.** Push the risk to a provider with stronger controls (e.g., hand off auth to an OIDC provider rather than rolling your own).
- **Accept.** Document the residual risk explicitly with sign-off. Acceptable for low-impact, low-likelihood threats.

The output of a session is a table with **threat → mitigation → owner → due date**. Without an owner and a date, mitigations rot.

### 6.1 DREAD (Optional) for Prioritization

DREAD scores threats on Damage, Reproducibility, Exploitability, Affected users, Discoverability. Microsoft retired DREAD internally because the scores were subjective and inconsistent across teams. CVSS v3/v4 is the modern equivalent and has the advantage of being widely understood.

Pragmatic alternative: a 2×2 of **likelihood × impact** is enough to triage a STRIDE backlog.

---

## 7. When to Threat Model

The pragmatic answer: **at design time**, and again whenever the design changes meaningfully.

### 7.1 Triggers

- **New service or new external surface.** Always.
- **New trust boundary.** Adding a third-party integration, opening an internal endpoint to the internet, federating with a new IdP.
- **New data classification.** Storing payment info, health records, or PII for the first time.
- **Architectural changes.** Moving auth from the gateway into each service, or vice versa.
- **Post-incident.** A real attack always reveals a missing or wrong mental model.

### 7.2 Cadence for Existing Services

Annual review for high-risk services. Light revisit each major release. Most teams don't do this and the threat model goes stale within 18 months. A stale threat model is worse than none — it gives false confidence.

### 7.3 Process Recipe (1-Hour Session)

1. **5 min** — Stakeholders state the asset (what we're protecting) and the goal of the feature.
2. **10 min** — Draw or update the DFD on a whiteboard / Miro / Excalidraw. Mark trust boundaries.
3. **30 min** — Walk every element through STRIDE. One person scribes, one drives. Don't filter — capture, then triage.
4. **10 min** — Triage the list. Each item gets owner + outcome (mitigate/eliminate/transfer/accept).
5. **5 min** — Schedule a follow-up to verify mitigations landed.

---

## 8. STRIDE vs PASTA vs LINDDUN vs Attack Trees

STRIDE isn't the only methodology. Each has a sweet spot.

| Method | Focus | Best For | Effort |
|--------|-------|----------|--------|
| **STRIDE** | Threat categories per element of a DFD | General-purpose application threat modeling; quick design-time reviews | Low–medium |
| **PASTA** | Process for Attack Simulation and Threat Analysis — 7 stages, business-driven | High-stakes systems where business impact analysis matters; regulated industries | High |
| **LINDDUN** | Privacy threats: Linking, Identifying, Non-repudiation, Detecting, Data disclosure, Unawareness, Non-compliance | GDPR/HIPAA/CCPA-impacted systems, where privacy is a first-class concern | Medium–high |
| **Attack Trees** | Goal-oriented: root = attacker goal, branches = ways to achieve it | Specific high-value goals (e.g., "exfiltrate customer DB"); pairs well with STRIDE | Variable |
| **VAST** | Visual, Agile, Simple Threat modeling — scales to many services | Large enterprises with many threat models to maintain | Medium |
| **OCTAVE** | Asset-driven, organizational | Long-horizon enterprise risk | High |

For most backend teams: STRIDE for design reviews, attack trees for the one or two scariest scenarios, LINDDUN added on if you process personal data at scale.

---

## 9. Tooling

### 9.1 Threat Dragon (OWASP)

Open-source, web-based, draws DFDs and walks STRIDE. Reasonable starting point. https://owasp.org/www-project-threat-dragon/

### 9.2 Microsoft Threat Modeling Tool

Windows-only, free, mature. Generates STRIDE threats automatically based on element types. https://aka.ms/threatmodelingtool

### 9.3 pytm (Python-based threat modeling as code)

Define the model in Python; generate the DFD and threat report. Plays well with code review.

```python
from pytm import TM, Server, Datastore, Boundary, Actor, Dataflow

tm = TM("Notes Service")

internet = Boundary("Internet")
appNet = Boundary("App Network")

user = Actor("User"); user.inBoundary = internet
gw = Server("API Gateway"); gw.inBoundary = appNet
notes = Server("Notes Service"); notes.inBoundary = appNet
db = Datastore("Postgres"); db.inBoundary = appNet

Dataflow(user, gw, "HTTPS Request")
Dataflow(gw, notes, "Internal RPC")
Dataflow(notes, db, "SQL")

tm.process()
```

### 9.4 Plain Markdown + Excalidraw

For most teams, the threat model lives best as a Markdown doc in the repo with an Excalidraw or Mermaid diagram. Reviewable in PRs, diffable, kept current.

---

## 10. Common Anti-Patterns

### 10.1 "We Don't Need to Threat Model — We Use Spring Security"

Frameworks defeat *implementation* mistakes (XSS escaping, parameterized queries). They don't defeat *design* mistakes (no rate limit on password reset, IDOR in business logic, missing tenant scoping). Most production breaches are design.

### 10.2 Exhaustive Boil-the-Ocean Sessions

Six engineers in a room for four hours generating a 200-row threat list nobody triages. Better: focused 1-hour sessions on one feature at a time, output triaged the same day.

### 10.3 Threat Modeling as a Compliance Checkbox

A doc generated to satisfy SOC 2 that nobody updates. The auditor is happy; the security posture is unchanged. Fix: tie the threat model to the design doc and require the reviewer to read both.

### 10.4 Skipping Trust Boundaries

If your DFD has no dashed lines, you've drawn a system diagram, not a threat model. The boundaries are where you do the work.

### 10.5 Mitigations Without Owners

"We should rate-limit the login endpoint" is a wish. "Owner: alice; ETA: end of sprint; tracked in JIRA-1234" is a commitment.

### 10.6 Treating STRIDE as a Strict Taxonomy

A single threat may belong in two categories (SQLi can be I and E). That's fine. STRIDE is a *prompt* to think systematically, not an ontology to argue about.

---

## Related

- [OWASP Top 10 (2021)](01-owasp-top-10.md) — the empirical complement to design-time STRIDE
- [Authentication vs Authorization](03-authentication-vs-authorization.md) — concrete authn/authz patterns that mitigate S and E
- [System Design — Defense in Depth and Threat Modeling](../../system-design/security/defense-in-depth-and-threat-modeling.md)
- [System Design — Zero-Trust Architecture](../../system-design/security/zero-trust-architecture.md)
- [Spring Security Filter Chain](../../java/security/security-filter-chain.md) — a canonical mitigation surface

---

## References

- **The Security Development Lifecycle** — Howard, Lipner. Microsoft Press, 2006. The original SDL/STRIDE reference.
- **Threat Modeling: Designing for Security** — Adam Shostack, 2014. Wiley. The definitive modern STRIDE book.
- **OWASP Threat Modeling Process** — https://owasp.org/www-community/Threat_Modeling_Process
- **OWASP Threat Dragon** — https://owasp.org/www-project-threat-dragon/
- **Microsoft Threat Modeling Tool** — https://aka.ms/threatmodelingtool
- **Microsoft STRIDE reference** — https://learn.microsoft.com/en-us/azure/security/develop/threat-modeling-tool-threats
- **pytm — threat modeling as code** — https://github.com/izar/pytm
- **PASTA: Process for Attack Simulation and Threat Analysis** — UcedaVelez, Morana. Wiley, 2015.
- **LINDDUN** privacy threat modeling — https://linddun.org/
- **NIST SP 800-154** — Guide to Data-Centric System Threat Modeling. https://csrc.nist.gov/pubs/sp/800/154/ipd
- **CWE — Common Weakness Enumeration** — https://cwe.mitre.org/
- **Threat Modeling Manifesto** — https://www.threatmodelingmanifesto.org/
