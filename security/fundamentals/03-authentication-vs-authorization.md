---
title: "Authentication vs Authorization — Identity, Sessions, and Access Models"
date: 2026-05-03
updated: 2026-05-03
tags: [security, authentication, authorization, rbac, abac, identity]
---

# Authentication vs Authorization — Identity, Sessions, and Access Models

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `security` `authentication` `authorization` `rbac` `abac` `identity`

---

## Table of Contents

- [Summary](#summary)
- [1. Authentication vs Authorization — The Hard Line](#1-authentication-vs-authorization--the-hard-line)
- [2. Authentication Factors](#2-authentication-factors)
- [3. Session-Based Authentication](#3-session-based-authentication)
- [4. Token-Based Authentication](#4-token-based-authentication)
- [5. Authorization Models](#5-authorization-models)
  - [5.1 RBAC](#51-rbac)
  - [5.2 ABAC](#52-abac)
  - [5.3 ReBAC](#53-rebac)
  - [5.4 PBAC and Policy-as-Code](#54-pbac-and-policy-as-code)
  - [5.5 Choosing a Model](#55-choosing-a-model)
- [6. Principle of Least Privilege](#6-principle-of-least-privilege)
- [7. Common Authorization Failure Modes](#7-common-authorization-failure-modes)
- [8. Concrete Implementations](#8-concrete-implementations)
- [9. Multi-Tenancy Considerations](#9-multi-tenancy-considerations)
- [Related](#related)
- [References](#references)

---

## Summary

Authentication answers **"who are you?"** Authorization answers **"what may you do?"** Conflating them is one of the most consequential mistakes in backend systems — it's the root cause of OWASP A01 (Broken Access Control) and A07 (Authentication Failures). This document draws the line cleanly, walks the major access-control models (RBAC, ABAC, ReBAC, PBAC), and shows the failure modes that consistently produce CVEs: IDOR, BOLA, missing function-level checks, role-vs-attribute mismatches in distributed systems. The audience is a backend engineer who already implements authz on every endpoint but wants to understand *why* the chosen model fits, when to switch, and what the next breach in this category looks like.

The bottom line: authentication is increasingly delegated to identity providers (Auth0, Okta, AWS Cognito, Google, GitHub via OIDC) — most teams should not roll it themselves. Authorization is *not* delegated; it lives in your code and your data model. Treat authentication as solved infrastructure and authorization as core product logic.

---

## 1. Authentication vs Authorization — The Hard Line

| | Authentication (AuthN) | Authorization (AuthZ) |
|--|--|--|
| Question | Who are you? | What may you do? |
| When | Once per session/token | On every request that touches a protected resource |
| Output | A verified identity (subject, claims) | A yes/no decision (or a filtered result set) |
| Failure mode | Spoofing — STRIDE **S** | Privilege escalation — STRIDE **E** |
| Examples | Login, OAuth callback, JWT signature check | "Can user 42 read invoice 7?", "Can the `viewer` role mutate?" |
| Standards | NIST SP 800-63, OIDC, FIDO2/WebAuthn | XACML, OPA/Rego, Cedar, Spring Security `@PreAuthorize` |

A signed JWT proves who the bearer is — **nothing more**. The decision "this user can read this invoice" is authorization; it is *your* logic and *your* data, not the identity provider's.

A common confusion: scopes in OAuth (e.g., `read:invoices`) feel like authorization, but they describe *delegated authority* of a client app, not access decisions on a specific resource. Cross-references: [OAuth 2.0 and OIDC](04-oauth2-and-oidc-deep-dive.md).

---

## 2. Authentication Factors

NIST SP 800-63B groups proof of identity into three factor categories:

| Factor | Examples | Common attacks |
|--------|----------|----------------|
| **Knowledge** — something you know | Password, PIN, security question | Phishing, credential stuffing, brute force |
| **Possession** — something you have | TOTP code (Google Authenticator), hardware key (YubiKey), push notification, SMS code | SIM swap (SMS), MFA fatigue (push), OTP phishing |
| **Inherence** — something you are | Fingerprint, face, voice | Spoof (printed face), replay |

**Multi-Factor Authentication (MFA)** combines two or more *different* factor categories. Two passwords aren't MFA. Two devices both running TOTP arguably are.

### 2.1 Phishing-Resistant MFA

NIST and CISA now strongly recommend **phishing-resistant** factors:

- **WebAuthn / Passkeys** (FIDO2) — origin-bound, can't be relayed.
- **PIV smartcards** with PKI.

Anti-recommended for high-stakes: SMS OTP (SIM swap), email OTP (account takeover chain), push notifications without number-matching (MFA fatigue → 2022 Uber breach).

### 2.2 Passwordless

Passkeys are passwordless. The user authenticates to the device with biometric/PIN; the device signs a server challenge with a key pair tied to the relying party's origin. No phishable shared secret crosses the network. This is the direction the industry is moving.

### 2.3 Verifying Something the User Knows: Password Storage

If you do store passwords:

- **Hash** with argon2id (preferred) or bcrypt cost ≥ 12.
- **Never** plaintext, never reversible encryption.
- **Per-password salt**, generated by a CSPRNG, stored alongside the hash.
- **Compare in constant time** to prevent timing oracle.
- **Check against breach corpus** (HIBP Pwned Passwords k-anonymity API) on signup and password change.

```typescript
import argon2 from 'argon2';

// On signup
const hash = await argon2.hash(password, {
  type: argon2.argon2id,
  memoryCost: 19456,    // 19 MiB
  timeCost: 2,
  parallelism: 1,
});

// On login
const valid = await argon2.verify(stored, submitted);
```

```java
// Spring Security
PasswordEncoder enc = PasswordEncoderFactories.createDelegatingPasswordEncoder();
String hash = enc.encode(rawPassword); // {argon2}... or {bcrypt}...
boolean ok = enc.matches(rawPassword, hash);
```

---

## 3. Session-Based Authentication

The traditional model: server assigns a random session ID, stores session state server-side (in Redis, Postgres, or memory), and sends the ID to the client as a cookie.

```
POST /login         (username, password)
  → 200 OK
    Set-Cookie: SID=8f3c...; HttpOnly; Secure; SameSite=Lax; Path=/

GET /api/profile
  Cookie: SID=8f3c...
  → server looks up SID in store, finds user_id=42, evaluates request
```

### 3.1 Cookie Flags That Matter

| Flag | Purpose |
|------|---------|
| `Secure` | Only sent over TLS. Mandatory in production. |
| `HttpOnly` | Inaccessible to JS — defends against XSS exfiltration. |
| `SameSite=Lax` (default since Chrome 80) | Cross-site requests don't carry the cookie except top-level GET navigations. Defends against CSRF. |
| `SameSite=Strict` | No cross-site requests at all. Stronger but breaks "click email link → land logged in". |
| `SameSite=None; Secure` | Cross-site allowed (needed for some embedded flows). |
| `Path=/`, `Domain` | Scope. Avoid setting `Domain` unless you must — broader scope = bigger blast radius. |
| `__Host-` prefix | Browser enforces `Secure`, `Path=/`, no `Domain`. Strongest cookie. |

### 3.2 Session ID Properties

- **At least 128 bits of entropy** from a CSPRNG (Node `crypto.randomBytes`, Java `SecureRandom`).
- **Opaque** — don't put user ID in it; the server looks it up.
- **Rotated on privilege change** (login, role change) to prevent session fixation.
- **Invalidated on logout** server-side; clearing the cookie isn't enough.
- **Idle and absolute timeouts** — typically 15–30 min idle, 8–24 h absolute for normal apps.

### 3.3 Pros and Cons

**Pros**: revocation is trivial (delete the row), small over-the-wire footprint, server can update permissions immediately by mutating session state.

**Cons**: requires a session store with the latency and availability characteristics of your hottest path. For multi-region active-active, the session store is the critical-path coordination point.

---

## 4. Token-Based Authentication

Server issues a self-contained, signed token (typically JWT). Client sends it on every request. Server verifies signature and trusts the claims.

```
POST /login → { access_token: "eyJ...", refresh_token: "..." }
GET /api/profile
  Authorization: Bearer eyJ...
  → server verifies signature, parses claims, evaluates request
```

### 4.1 Pros and Cons

**Pros**: no server-side session lookup; easy to scale horizontally; works naturally with microservices and mobile/SPA clients.

**Cons**: revocation is hard (the whole point is the server doesn't store state); leaked tokens are valid until expiry; bigger over-the-wire size; many footguns (see [JWT Design and Pitfalls](05-jwt-design-and-pitfalls.md)).

### 4.2 When to Use Which

| Use sessions | Use tokens |
|--------------|------------|
| Single web app, single backend, single domain | SPA + API on different domains, mobile app + API, microservices |
| Need fine-grained immediate revocation | Want stateless API tier |
| Already have a fast session store | Already have an OIDC provider |
| Compliance requires audit log per session | OK with short-lived tokens + refresh rotation |

The myth that "JWT scales better" is overstated — a Redis lookup is microseconds. The real reason to use tokens is that *someone else has a great token-issuing IdP* (Auth0, Cognito, Okta, Keycloak, Google) and you should not build login.

---

## 5. Authorization Models

Authentication produces a verified identity. Authorization decides what that identity may do. The four mainstream models trade off expressiveness, performance, and operational cost.

### 5.1 RBAC

**Role-Based Access Control** (NIST RBAC, INCITS 359). The classic model.

- A user has one or more **roles**.
- A role has a set of **permissions** (action + resource type).
- Decision: `allow(user, action, resource) ⇔ ∃ role ∈ user.roles such that (action, type(resource)) ∈ role.permissions`.

```
Roles:    admin, editor, viewer
Perms:    admin   → notes:*
          editor  → notes:read, notes:write
          viewer  → notes:read

Alice has [admin]                   → can do anything to any note
Bob   has [editor]                  → can read/write any note
Carol has [viewer]                  → can read any note
```

**Strengths.** Easy to reason about. Easy to audit. Fits human intuition.

**Weaknesses.** Doesn't capture **resource-specific** permissions ("Carol can edit *her own* notes"). Adding "Carol owns this note" is outside the model — you bolt it on with extra checks, and inconsistency creeps in. With many resource-specific rules, RBAC explodes into "role per project per stage per environment".

**When to use.** Coarse permissions in admin tools, internal apps, the role of the *human user* against the application as a whole.

### 5.2 ABAC

**Attribute-Based Access Control** (NIST SP 800-162). Decisions are computed from arbitrary attributes of the subject, action, resource, and environment.

```
Allow read on Note if:
  subject.tenant_id == resource.tenant_id
  AND (resource.owner_id == subject.id
       OR resource.shared_with contains subject.id)
  AND environment.time within business_hours(subject.tenant)
```

**Strengths.** Expressive. Captures resource-specific and contextual rules naturally. Supports row-level/cell-level constraints.

**Weaknesses.** Hard to audit ("show me everyone who can read invoice 7" is a search problem, not a join). Performance: every decision is policy evaluation. Policies become a programming language and need testing.

**When to use.** Rich data models, multi-tenant SaaS, regulated workloads where access depends on classification, time, location, or device posture.

Standard: **XACML** (OASIS) for the formal spec; **OPA/Rego**, **Cedar** (AWS) for modern implementations.

### 5.3 ReBAC

**Relationship-Based Access Control**. Decisions follow paths in a graph of relationships. Popularized by **Google Zanzibar**, the system that powers Google Drive sharing.

```
Tuples:
  doc:report  owner       user:alice
  doc:report  viewer      group:eng#member
  group:eng   member      user:bob

Question: can user:bob view doc:report?
Walk: doc:report → viewer → group:eng → member → user:bob ✓
```

**Strengths.** Captures real-world sharing semantics: documents shared with groups, folders inheriting permissions, organization hierarchies. Sub-millisecond decisions even at Google's scale.

**Weaknesses.** Tooling is newer (OpenFGA, SpiceDB, AuthZed). Requires a separate authorization data store kept in sync with your domain data — that sync is the operational hard part.

**When to use.** Collaboration products (docs, design tools, codebases), social products with friend-of-friend sharing, anything that looks like "permissions inherit from container with overrides".

### 5.4 PBAC and Policy-as-Code

**Policy-Based Access Control** is umbrella terminology for "the decision is computed by a policy engine". Both ABAC and ReBAC are PBAC implementations.

The core idea: separate the **policy** (what is allowed) from the **policy enforcement point** (PEP — your application's authorization check) and the **policy decision point** (PDP — the engine that evaluates the policy).

```
Application code (PEP):
  if not opa.check("read", "note:42", request.user):
      return 403

OPA (PDP) evaluates Rego policy file:
  package notes.authz

  allow if {
      input.action == "read"
      input.resource.owner == input.subject.id
  }

  allow if {
      input.action == "read"
      input.resource.shared_with[_] == input.subject.id
  }
```

Modern policy engines:

- **OPA / Rego** — CNCF graduated; ubiquitous in cloud-native and Kubernetes.
- **Cedar** — AWS-developed, used by Verified Permissions; smaller language than Rego, easier to analyze.
- **OpenFGA / SpiceDB** — Zanzibar-style ReBAC.
- **Casbin** — RBAC/ABAC library available in many languages.

### 5.5 Choosing a Model

| Question | Direction |
|----------|-----------|
| Are roles enough? Few users, few resource types? | RBAC |
| Do permissions depend on resource attributes (owner, classification, time)? | ABAC |
| Do permissions follow sharing graphs (folders, groups, friends)? | ReBAC |
| Compliance demands externalized policies, audit, and policy review? | Policy-as-code (OPA/Cedar) |
| Are you a startup with three roles? | Stop reading; ship RBAC. |

In practice, mature systems combine: RBAC at the coarse level (admin/user/service-account), ABAC or ReBAC for resource-level rules.

---

## 6. Principle of Least Privilege

> Every program and every privileged user of the system should operate using the least amount of privilege necessary to complete the job. — Saltzer & Schroeder, 1975.

Operationally:

- **Database users.** App connects with a role that has only the grants it needs. Migrations run as a separate user with DDL rights. Reports run as a read-only user.
- **IAM roles.** No `Resource: *`, no `Action: *`. Use the AWS Access Analyzer or GCP Recommender to surface unused permissions.
- **Container `securityContext`.** `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, drop all Linux capabilities and add only what's needed.
- **API tokens.** Scope to specific repos / projects; expire short.
- **Humans.** Just-in-time elevation for admin work, not persistent admin roles.

The complement is **Privilege Bracketing**: elevate, do the work, drop privileges. Sudo is the original.

---

## 7. Common Authorization Failure Modes

These appear in CVE feeds and bug bounty payouts every week.

### 7.1 IDOR (Insecure Direct Object Reference)

`GET /api/invoices/12345` returns the invoice without checking ownership. The First American Financial 2019 breach (885M documents) was a single IDOR.

**Fix.** Scope the query by the authenticated principal: `WHERE id = ? AND user_id = ?`. Don't rely on UI hiding URLs.

### 7.2 BOLA (Broken Object-Level Authorization)

The OWASP API Security #1 risk. Same as IDOR but in the API context. Attackers iterate IDs in JSON bodies and path parameters.

**Fix.** Authorization at the data-access layer, not just in the controller. Hibernate filter / row-level security / repository wrappers that *can't be called* without a tenant scope.

### 7.3 Missing Function-Level Authorization

The UI hides the "Delete" button from non-admins, but the endpoint doesn't check.

**Fix.** Server-side check on **every** mutating endpoint. `@PreAuthorize` on every controller method. A test that asserts a non-admin gets 403 from each admin endpoint.

### 7.4 Mass Assignment

User submits `{ "name": "Bob", "isAdmin": true }`; the server `req.body` directly into the model. Now Bob is admin.

**Fix.** Schema-validate inputs into a typed DTO that contains only the fields users may set. Don't bind raw request bodies to entities.

### 7.5 Confused Deputy

A privileged service does something on behalf of a less-privileged caller without verifying the caller had the right.

**Fix.** Pass the caller's identity through, not the service's. In Spring: propagate `Authentication`. In OAuth: use the user's access token, or **Token Exchange** (RFC 8693).

### 7.6 Time-of-Check / Time-of-Use (TOCTOU)

Permission checked at request start; resource changes by the time the action runs.

**Fix.** Authorize at the same DB transaction as the action; use optimistic locking; recheck on each mutation.

### 7.7 Distributed Authz Skew

Microservice A trusts a JWT issued 8 minutes ago; service B revoked the role 2 minutes ago; the JWT is still valid for 5 more minutes.

**Fix.** Short token lifetimes; back-channel revocation lists for high-stakes; central PDP if requirements demand sub-minute consistency.

---

## 8. Concrete Implementations

### 8.1 Express + Custom Middleware

```typescript
import { Request, Response, NextFunction } from 'express';

interface AuthedRequest extends Request {
  user: { id: string; tenantId: string; roles: string[] };
}

const requireRole = (role: string) =>
  (req: AuthedRequest, res: Response, next: NextFunction) => {
    if (!req.user.roles.includes(role)) return res.status(403).end();
    next();
  };

// Object-level authorization at the data layer
async function getNoteForUser(noteId: string, userId: string) {
  return db.note.findFirst({
    where: { id: noteId, OR: [{ ownerId: userId }, { sharedWith: { has: userId } }] },
  });
}

app.get('/notes/:id', authenticate, async (req: AuthedRequest, res) => {
  const note = await getNoteForUser(req.params.id, req.user.id);
  if (!note) return res.sendStatus(404); // 404 not 403 — don't leak existence
  res.json(note);
});

app.delete('/admin/users/:id', authenticate, requireRole('admin'), deleteUserHandler);
```

### 8.2 Spring Security — Method Authorization

```java
@RestController
@RequestMapping("/api/notes")
public class NoteController {

    @GetMapping("/{id}")
    public Note get(@PathVariable UUID id, Authentication auth) {
        // Scoping at the data layer — preferred
        return noteService.findForUser(id, currentUserId(auth))
            .orElseThrow(() -> new ResourceNotFoundException(id));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("@noteAuthz.canDelete(#id, authentication)")
    public void delete(@PathVariable UUID id) {
        noteService.delete(id);
    }
}

@Component("noteAuthz")
public class NoteAuthz {
    private final NoteRepository repo;

    public boolean canDelete(UUID noteId, Authentication auth) {
        UUID userId = ((AppUser) auth.getPrincipal()).id();
        return repo.findById(noteId)
            .map(n -> n.ownerId().equals(userId))
            .orElse(false);
    }
}
```

### 8.3 OPA Sidecar Pattern

The application calls a local OPA sidecar over HTTP/gRPC for every authorization decision. Policies are versioned in Git, deployed to OPA, and changes are visible without a redeploy of the app.

```rego
# notes/authz.rego
package notes.authz
import future.keywords.if, future.keywords.in

default allow := false

allow if {
  input.action == "read"
  input.subject.tenant_id == input.resource.tenant_id
  some user_id in input.resource.viewers
  user_id == input.subject.id
}
```

```typescript
async function authz(action: string, resource: any, subject: any): Promise<boolean> {
  const r = await fetch('http://localhost:8181/v1/data/notes/authz/allow', {
    method: 'POST',
    body: JSON.stringify({ input: { action, resource, subject } }),
  });
  return (await r.json()).result === true;
}
```

---

## 9. Multi-Tenancy Considerations

The most violated trust boundary in SaaS is **between tenants**. Common designs:

- **Pool model** — shared schema, `tenant_id` column on every row. Cheapest. **Every** query must filter by `tenant_id`. PostgreSQL Row-Level Security (RLS) is a must-have safety net.
- **Bridge model** — schema per tenant, shared DB. Stronger isolation; more migrations to run.
- **Silo model** — DB per tenant. Strongest; expensive at scale.

Defense in depth for the pool model:

```sql
-- Postgres RLS as a backstop
ALTER TABLE notes ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON notes
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Application sets the GUC at the start of the request
SET LOCAL app.tenant_id = '...';
```

Even if the application forgets a `WHERE tenant_id`, RLS rejects cross-tenant access. Test it: write an integration test that sets a different tenant ID and asserts queries return zero rows.

---

## Related

- [OWASP Top 10 (2021)](01-owasp-top-10.md) — A01 and A07 cover the failure modes in this doc empirically
- [Threat Modeling with STRIDE](02-threat-modeling-stride.md) — STRIDE S and E directly map to authn and authz
- [OAuth 2.0 and OIDC Deep Dive](04-oauth2-and-oidc-deep-dive.md) — delegated authorization protocols
- [JWT Design and Pitfalls](05-jwt-design-and-pitfalls.md) — when tokens are the authn vehicle
- [Spring Security — Authentication and Authorization](../../java/security/authentication-authorization.md)
- [System Design — Authentication](../../system-design/security/authentication.md)
- [System Design — Authorization](../../system-design/security/authorization.md)
- [Kubernetes RBAC](../../kubernetes/security/rbac-and-service-accounts.md)

---

## References

- **NIST SP 800-63-3** — Digital Identity Guidelines (and -63B for authenticators, -63C for federation). https://pages.nist.gov/800-63-3/
- **NIST SP 800-162** — Guide to Attribute Based Access Control (ABAC) Definition and Considerations. https://csrc.nist.gov/pubs/sp/800/162/final
- **NIST RBAC standard (INCITS 359-2012)** — https://csrc.nist.gov/projects/role-based-access-control
- **Saltzer & Schroeder, "The Protection of Information in Computer Systems"** (1975) — origin of least privilege. https://www.cs.virginia.edu/~evans/cs551/saltzer/
- **Google Zanzibar paper** — Pang et al., USENIX ATC 2019. https://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/
- **OWASP API Security Top 10 (2023)** — BOLA is API1. https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- **OWASP Authentication Cheat Sheet** — https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- **OWASP Session Management Cheat Sheet** — https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- **OWASP Authorization Cheat Sheet** — https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- **OASIS XACML 3.0** — http://docs.oasis-open.org/xacml/3.0/xacml-3.0-core-spec-os-en.html
- **Open Policy Agent (OPA)** — https://www.openpolicyagent.org/
- **Cedar policy language** — https://www.cedarpolicy.com/
- **OpenFGA** — https://openfga.dev/
- **AuthZed / SpiceDB** — https://authzed.com/
- **WebAuthn Level 3** — https://www.w3.org/TR/webauthn-3/
- **CISA — Implementing Phishing-Resistant MFA** — https://www.cisa.gov/sites/default/files/publications/fact-sheet-implementing-phishing-resistant-mfa-508c.pdf
