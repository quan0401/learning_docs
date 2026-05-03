---
title: "OWASP Top 10 (2021) — The Vulnerability Landscape Every Backend Dev Must Know"
date: 2026-05-03
updated: 2026-05-03
tags: [security, owasp, vulnerabilities, web-security, appsec]
---

# OWASP Top 10 (2021) — The Vulnerability Landscape Every Backend Dev Must Know

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `security` `owasp` `vulnerabilities` `web-security` `appsec`

---

## Table of Contents

- [Summary](#summary)
- [1. How the OWASP Top 10 Is Built](#1-how-the-owasp-top-10-is-built)
- [2. A01:2021 — Broken Access Control](#2-a012021--broken-access-control)
- [3. A02:2021 — Cryptographic Failures](#3-a022021--cryptographic-failures)
- [4. A03:2021 — Injection](#4-a032021--injection)
- [5. A04:2021 — Insecure Design](#5-a042021--insecure-design)
- [6. A05:2021 — Security Misconfiguration](#6-a052021--security-misconfiguration)
- [7. A06:2021 — Vulnerable and Outdated Components](#7-a062021--vulnerable-and-outdated-components)
- [8. A07:2021 — Identification and Authentication Failures](#8-a072021--identification-and-authentication-failures)
- [9. A08:2021 — Software and Data Integrity Failures](#9-a082021--software-and-data-integrity-failures)
- [10. A09:2021 — Security Logging and Monitoring Failures](#10-a092021--security-logging-and-monitoring-failures)
- [11. A10:2021 — Server-Side Request Forgery (SSRF)](#11-a102021--server-side-request-forgery-ssrf)
- [12. What's Likely in OWASP Top 10 2025](#12-whats-likely-in-owasp-top-10-2025)
- [13. Mitigation Patterns That Cut Across Categories](#13-mitigation-patterns-that-cut-across-categories)
- [Related](#related)
- [References](#references)

---

## Summary

The OWASP Top 10 is a community-driven, evidence-based ranking of the most critical web application security risks. The 2021 edition is the current authoritative list as of 2026-05-03; the 2025 candidate data has been published but the final list is still being finalized at the time of writing. This document walks each of the ten categories with concrete Node and Spring Boot examples, then surfaces the cross-cutting mitigations (least privilege, defense in depth, schema validation, secure-by-default frameworks) that defeat several at once. Treat this as a quick-reference map, not a checklist — most categories deserve their own deep dive (and several are linked from the Tier 2 docs in this learning path).

The 2021 ranking was a meaningful structural shift from 2017: Broken Access Control jumped to #1 (it was #5), Cryptographic Failures replaced "Sensitive Data Exposure" as a more accurate root-cause framing, and SSRF was promoted to its own category from a footnote. Those changes reflect a hard truth — the bugs that take down companies are mostly authorization and configuration mistakes, not exotic memory corruption.

---

## 1. How the OWASP Top 10 Is Built

The list is **not a vote**. It combines:

- **Eight quantitative data sources** — bug bounty programs, scanning vendors, and corporate AppSec teams contribute aggregated incidence rates (what % of tested apps had this CWE) and average CVSS-like impact.
- **One community survey** — to surface forward-looking risks the data lags behind (e.g., insecure design, supply-chain integrity).
- **CWE mapping** — each Top 10 category is a bucket of related Common Weakness Enumeration IDs. A03 Injection covers 33 CWEs.

The ranking weight is roughly: incidence rate × (max test coverage) × (max impact). This is why "Broken Access Control" is #1: ~94% of tested apps had at least one access-control flaw, and the impact of the worst flaws is severe.

| 2021 Rank | Category | Key CWEs | Vs 2017 |
|-----------|----------|----------|---------|
| A01 | Broken Access Control | 200, 201, 352 | ↑ from #5 |
| A02 | Cryptographic Failures | 259, 327, 331 | ↑ from #3 (renamed) |
| A03 | Injection | 79, 89, 73 | ↓ from #1 (XSS folded in) |
| A04 | Insecure Design | 209, 256, 501 | NEW |
| A05 | Security Misconfiguration | 16, 611 | ↑ from #6 |
| A06 | Vulnerable & Outdated Components | 1104 | ↑ from #9 |
| A07 | Identification & Authentication Failures | 297, 384, 521 | ↓ from #2 |
| A08 | Software & Data Integrity Failures | 502, 829 | NEW |
| A09 | Security Logging & Monitoring Failures | 117, 223, 778 | ↑ from #10 |
| A10 | Server-Side Request Forgery (SSRF) | 918 | NEW |

---

## 2. A01:2021 — Broken Access Control

**Definition.** The application enforces business logic but fails to enforce that the calling user is authorized to invoke that logic on that data. Manifests as IDOR, missing function-level checks, forced browsing, and privilege escalation.

### 2.1 Real-World Patterns

- **IDOR (Insecure Direct Object Reference).** `GET /api/invoices/12345` returns invoice 12345 regardless of which user is logged in.
- **Missing authorization on admin endpoints.** `POST /api/admin/users` is hidden from the UI but reachable directly.
- **JWT/cookie tampering.** `role=user` field client-side is changed to `role=admin` and the server trusts it.
- **CORS misconfiguration.** `Access-Control-Allow-Origin: *` plus `Allow-Credentials: true` (which is invalid but historically accepted by some browsers) leaks authenticated responses to attacker origins.

### 2.2 Node Example — Vulnerable

```typescript
// Express handler that trusts the URL parameter
app.get('/api/invoices/:id', requireLogin, async (req, res) => {
  const invoice = await db.invoice.findUnique({ where: { id: req.params.id } });
  if (!invoice) return res.status(404).end();
  res.json(invoice); // BUG: doesn't check invoice.userId === req.user.id
});
```

Fix: scope every query by the authenticated principal.

```typescript
app.get('/api/invoices/:id', requireLogin, async (req, res) => {
  const invoice = await db.invoice.findFirst({
    where: { id: req.params.id, userId: req.user.id },
  });
  if (!invoice) return res.status(404).end();
  res.json(invoice);
});
```

### 2.3 Spring Example — Method-Level Authorization

```java
@RestController
@RequestMapping("/api/invoices")
public class InvoiceController {

    @GetMapping("/{id}")
    @PostAuthorize("returnObject.userId == authentication.principal.id")
    public Invoice get(@PathVariable UUID id) {
        return invoiceRepository.findById(id).orElseThrow();
    }
}
```

`@PostAuthorize` evaluates after the method runs. For high-traffic endpoints, prefer scoping the query (as in the Node example) so you don't fetch rows the caller can't see.

### 2.4 Notable CVEs and Incidents

- **First American Financial (2019)** — IDOR exposed 885M mortgage documents by incrementing a URL ID.
- **CVE-2021-22986** (F5 BIG-IP) — missing authentication on iControl REST allowed unauthenticated RCE; classic broken access control at the API gateway tier.

### 2.5 Cross-Reference

Deep dive: [Authentication vs Authorization](03-authentication-vs-authorization.md).

---

## 3. A02:2021 — Cryptographic Failures

**Definition.** Sensitive data is exposed because crypto is missing, misused, or implemented with broken primitives. Renamed from "Sensitive Data Exposure" to point at the root cause.

### 3.1 Common Failures

- **Plaintext at rest.** Passwords, API keys, PII stored without encryption.
- **Weak algorithms.** MD5, SHA-1, DES, RC4, ECB mode AES.
- **Hardcoded keys.** Keys in source, config files, or container images.
- **TLS misconfiguration.** Self-signed certs in prod, accepting any cert (`rejectUnauthorized: false` in Node), TLS 1.0/1.1, weak cipher suites.
- **Predictable randomness.** `Math.random()` for security tokens, `java.util.Random` for session IDs.

### 3.2 Password Storage

```typescript
// WRONG — fast hash, no salt
import { createHash } from 'node:crypto';
const hash = createHash('sha256').update(password).digest('hex');

// RIGHT — argon2id, memory-hard, salt baked in
import argon2 from 'argon2';
const hash = await argon2.hash(password, { type: argon2.argon2id });
const ok = await argon2.verify(hash, password);
```

```java
// Spring Security — BCrypt with cost factor
PasswordEncoder encoder = new BCryptPasswordEncoder(12);
String hash = encoder.encode(rawPassword);
boolean ok = encoder.matches(rawPassword, hash);
```

NIST SP 800-63B-3 recommends memory-hard functions (Argon2id, scrypt) over bcrypt for new deployments.

### 3.3 Notable CVEs

- **CVE-2014-3566 (POODLE)** — SSLv3 padding oracle attack. Drove industry-wide deprecation of SSLv3.
- **Adobe (2013)** — 153M user records leaked with passwords encrypted using ECB-mode 3DES, plus a "password hint" column. Adjacent identical hints revealed identical passwords.

### 3.4 Cross-Reference

Deep dive: [TLS Handshake and Certificates](06-tls-handshake-and-certificates.md). Tier 3 will cover crypto primitives.

---

## 4. A03:2021 — Injection

**Definition.** User-controlled input is interpreted as code or query syntax by a downstream interpreter. Includes SQLi, NoSQLi, LDAP injection, OS command injection, ORM injection, and XSS (folded in for 2021).

### 4.1 SQL Injection — Vulnerable Node

```typescript
// Concatenated SQL — vulnerable
const sql = `SELECT * FROM users WHERE email = '${req.query.email}'`;
const rows = await pool.query(sql);
```

Attacker sends `email=' OR 1=1 --` and dumps the table.

```typescript
// Parameterized — safe
const rows = await pool.query('SELECT * FROM users WHERE email = $1', [req.query.email]);
```

### 4.2 Spring Data — Safe by Default

```java
// JPQL named parameter — parameterized
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// Spring Data method derivation — also parameterized
Optional<User> findByEmail(String email);
```

Avoid `nativeQuery = true` with concatenation. If you must build dynamic SQL, use `JdbcTemplate` with placeholders or `Criteria API`.

### 4.3 NoSQL Injection (MongoDB)

```typescript
// User-controlled object passed straight to query — vulnerable
await db.collection('users').findOne({ email: req.body.email, password: req.body.password });
// Attacker sends { "email": "victim@x.com", "password": { "$ne": null } }
```

Validate types with a schema (Zod/Joi) before use; reject objects where strings are expected.

### 4.4 OS Command Injection

```typescript
// Vulnerable
import { exec } from 'node:child_process';
exec(`pdftotext ${userPath} -`); // attacker passes "; rm -rf /"

// Safe — execFile with arg array
import { execFile } from 'node:child_process';
execFile('pdftotext', [userPath, '-']);
```

### 4.5 XSS (Cross-Site Scripting)

Three flavors:

- **Reflected** — payload in URL parameter rendered without escaping.
- **Stored** — payload persisted (e.g., comment field) and rendered to other users.
- **DOM-based** — JS reads URL/`document.location` and writes to `innerHTML` without sanitizing.

Modern frameworks (React, Vue, Angular, Thymeleaf with default escaping) escape by default — XSS comes back via `dangerouslySetInnerHTML`, `v-html`, `[innerHTML]`, or rendered Markdown without sanitization.

```tsx
// React — safe (text node)
<div>{userInput}</div>

// React — dangerous unless sanitized
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />
```

Defense in depth: Content Security Policy (`Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-...'`) makes `<script>` injection harmless.

### 4.6 Notable CVEs

- **CVE-2017-5638 (Apache Struts 2)** — OGNL expression injection in `Content-Type` header. The Equifax breach exploit; ~147M records.
- **CVE-2022-22965 (Spring4Shell)** — class-loader manipulation via data binding; not a classic injection but in the same family of "user input reaches dangerous interpreter".

---

## 5. A04:2021 — Insecure Design

**Definition.** New category for 2021. The application is *implemented correctly* but its design omits a control that should exist — e.g., no rate limiting on password reset, no anti-automation on signup, business logic that allows negative quantities.

### 5.1 Examples

- **Password reset without rate limiting** — attacker enumerates email addresses by timing.
- **Coupon stacking** — applying the same percentage discount twice because the spec didn't say "max one discount".
- **Race conditions in payments** — two concurrent withdrawal requests both pass the balance check.
- **No re-authentication for sensitive actions** — change-email and change-password don't require current password.

### 5.2 Mitigation: Threat Modeling at Design Time

The fix is process: threat-model new features before coding. STRIDE, abuse cases, and "evil user stories" surface design gaps that no static analyzer can find.

### 5.3 Cross-Reference

Deep dive: [Threat Modeling with STRIDE](02-threat-modeling-stride.md).

---

## 6. A05:2021 — Security Misconfiguration

**Definition.** Default credentials, verbose error pages, unnecessary services enabled, missing security headers, S3 buckets readable by `*`, debug endpoints in production.

### 6.1 Common Misconfigurations

- **Spring Boot Actuator exposed.** `management.endpoints.web.exposure.include=*` on the public network — `/actuator/heapdump` leaks memory contents including credentials.
- **Express error stack traces in prod.** `app.use(errorHandler())` defaulting to verbose mode.
- **CORS `*` with credentials.** Allows any origin to make authenticated requests.
- **Missing security headers.** No `Strict-Transport-Security`, no `Content-Security-Policy`, no `X-Content-Type-Options: nosniff`.
- **S3 buckets `public-read`.** Capital One (CVE-2019-3719 era), Booz Allen, Verizon — all leaked customer data via misconfigured object storage.

### 6.2 Hardening Checklist

```yaml
# Spring Boot — restrict Actuator
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus
  endpoint:
    health:
      show-details: never
```

```typescript
// Express — helmet enables a sensible header baseline
import helmet from 'helmet';
app.use(helmet());
// Sets HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, etc.
// Configure CSP explicitly per app.
```

### 6.3 Notable Incidents

- **Capital One (2019)** — SSRF + IAM misconfiguration on a WAF instance let an attacker enumerate S3 buckets and exfiltrate ~106M records.
- **MongoDB ransomware sweep (2017)** — tens of thousands of internet-exposed MongoDB instances with default `bind 0.0.0.0` and no authentication were wiped and ransomed.

---

## 7. A06:2021 — Vulnerable and Outdated Components

**Definition.** Using a library, framework, or runtime with known CVEs. Failing to track what you depend on transitively.

### 7.1 The Reality

A 2024 Sonatype report found the average enterprise Java app pulls in ~150 transitive dependencies, and 1 in 8 downloads from Maven Central is a known-vulnerable version.

### 7.2 Practical Defenses

- **SCA (Software Composition Analysis).** Snyk, Dependabot, GitHub Advanced Security, OWASP Dependency-Check, Trivy.
- **Pin and update.** `package-lock.json`, `pnpm-lock.yaml`, Maven dependency management section. Renovate or Dependabot for automated PRs.
- **Generate an SBOM.** SPDX or CycloneDX. Required for federal procurement (US EO 14028) and increasingly for B2B contracts.

### 7.3 Notable CVEs

- **CVE-2021-44228 (Log4Shell)** — JNDI lookup in Log4j 2.x ≤ 2.14.1, CVSS 10.0. Exploitable via any logged user input. Forced industry-wide SBOM scrambles.
- **CVE-2014-0160 (Heartbleed)** — OpenSSL 1.0.1–1.0.1f memory disclosure. Leaked private keys from millions of servers.
- **event-stream incident (2018)** — npm package `event-stream` was handed off to a malicious maintainer who added a payload targeting bitcoin wallets in `copay`. Pure supply-chain.

### 7.4 Cross-Reference

Tier 4 will cover supply-chain and SBOM in depth.

---

## 8. A07:2021 — Identification and Authentication Failures

**Definition.** Weak credentials policy, no rate limiting on login, predictable session IDs, missing MFA, broken logout, credential stuffing.

### 8.1 Patterns

- **No login rate limiting.** Attacker tries 10K passwords against `bob@example.com`.
- **No credential-stuffing defense.** Attackers use leaked password databases against your login form.
- **Session fixation.** Server accepts attacker-supplied session ID.
- **Predictable tokens.** Email-confirmation tokens that are sequential, or use `Math.random()`.
- **Mishandled logout.** Server-side session not invalidated; JWT revocation not implemented.

### 8.2 NIST SP 800-63B Recommendations

NIST modernized password guidance in 2017 (and reaffirmed in subsequent revisions):

- **No periodic rotation** unless evidence of compromise.
- **No composition rules** (uppercase, special char). Length matters more.
- **Check against breach corpus.** HIBP Pwned Passwords API lets you check `k-anonymity`-style without sending the password.
- **Allow password managers** (don't disable paste).
- **Minimum 8 chars, support up to 64+**.

### 8.3 Cross-Reference

Deep dives: [Authentication vs Authorization](03-authentication-vs-authorization.md), [OAuth 2.0 and OIDC](04-oauth2-and-oidc-deep-dive.md), [JWT Design and Pitfalls](05-jwt-design-and-pitfalls.md).

---

## 9. A08:2021 — Software and Data Integrity Failures

**Definition.** New for 2021. Code or data is updated without integrity verification. CI/CD pipelines, auto-update mechanisms, and unsigned packages all fall here.

### 9.1 Examples

- **Unsigned auto-updates** — app fetches updates from a CDN over HTTPS but doesn't verify a signature; a CDN compromise lets attackers ship malware.
- **Insecure deserialization** — Java `ObjectInputStream`, Python `pickle.load`, Ruby `Marshal.load`, Node `node-serialize`. User-controlled bytes become objects with side-effectful constructors.
- **CI/CD pipeline tampering** — SolarWinds-style: build server compromised, malicious code injected at compile time, signed by the legitimate signing key.

### 9.2 Java Deserialization Example

```java
// VULNERABLE — never deserialize untrusted bytes
ObjectInputStream ois = new ObjectInputStream(request.getInputStream());
Object obj = ois.readObject(); // gadget chain RCE
```

Mitigations:
- Use JSON or Protobuf, not native Java serialization.
- If you must, use `ObjectInputFilter` (JEP 290) to allowlist classes.
- Never accept serialized objects from untrusted sources.

### 9.3 Notable CVEs

- **CVE-2017-9805 (Struts REST plugin)** — XStream deserialization RCE.
- **SolarWinds Orion (2020)** — supply-chain compromise of the build pipeline; ~18,000 customers received the trojaned update.
- **CVE-2018-1000525 (node-serialize)** — `unserialize()` of user input → RCE.

### 9.4 Defenses

- **Sigstore / cosign** for container images.
- **SLSA framework** levels 1–4 for build provenance.
- **In-toto** attestations.
- **Reproducible builds** so the same source produces the same binary.

---

## 10. A09:2021 — Security Logging and Monitoring Failures

**Definition.** Insufficient logging means breaches go undetected. The Verizon DBIR routinely shows median time-to-detect over 200 days.

### 10.1 What to Log

- **Authentication events** — successes and failures, with IP and user agent.
- **Authorization failures** — access denied, privilege escalation attempts.
- **Server-side validation failures** — bad input that should never come from a real client.
- **High-value actions** — admin operations, role changes, password resets, money movements.
- **Errors at integration boundaries** — DB, third-party API, payment processors.

### 10.2 What NOT to Log

- Full request/response bodies for sensitive endpoints (will leak passwords, tokens, PII).
- Authorization headers, session cookies, JWTs.
- PII beyond what's necessary for forensics.

### 10.3 Practical Setup

- **Structured logs** (JSON) with consistent fields: `timestamp`, `level`, `traceId`, `userId`, `event`.
- **Centralize** — Loki, ELK, Datadog, CloudWatch. Local logs are useless during a breach.
- **Alert on anomalies** — login spikes from new geos, surge in 4xx, sudden 5xx error class.
- **Retention** — at least 90 days hot, 1 year cold, longer for regulated workloads.

### 10.4 Cross-Reference

[Network Observability](../../networking/advanced/network-observability.md) covers the metrics-and-tracing complement.

---

## 11. A10:2021 — Server-Side Request Forgery (SSRF)

**Definition.** New for 2021. The server fetches a URL supplied (directly or indirectly) by the user, allowing the user to reach internal services the user can't reach directly.

### 11.1 Why It's Devastating in Cloud

Cloud workloads have a **metadata service** at `169.254.169.254` (AWS, GCP, Azure) that returns IAM credentials. SSRF that reaches the metadata service grants the attacker the workload's IAM role.

```
GET /api/fetch-url?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/my-role
```

### 11.2 The Capital One Pattern

The 2019 Capital One breach combined SSRF (a vulnerable WAF on EC2) with IAM credentials from the metadata service to enumerate and exfiltrate S3 buckets. ~106M records.

### 11.3 Defenses

- **Allowlist destination hostnames.** Don't allowlist by URL pattern — that's bypassable. Resolve the hostname, check the resolved IP against an allowlist.
- **Block private IP ranges and metadata IPs.** RFC 1918 (10/8, 172.16/12, 192.168/16), RFC 6598 (100.64/10), 127/8, 169.254/16, IPv6 fc00::/7 and fe80::/10.
- **Use IMDSv2** (AWS) — requires `PUT` to fetch a session token, blocking simple SSRF that only does GETs. AWS made IMDSv2 the default for new instances in 2024.
- **Network segmentation.** Outbound egress filtering at the VPC/subnet level so SSRF can't reach the metadata service or internal admin networks.
- **DNS rebinding protection.** Resolve the hostname *once* at the URL-validation step, then connect to that IP — not the hostname again. Otherwise an attacker controls a domain that resolves to a public IP for the validation request and `169.254.169.254` for the actual fetch.

### 11.4 Node Example

```typescript
import { lookup } from 'node:dns/promises';
import { isIP } from 'node:net';

const PRIVATE_RANGES = [
  /^10\./, /^172\.(1[6-9]|2\d|3[01])\./, /^192\.168\./,
  /^127\./, /^169\.254\./, /^0\./,
];

async function safeFetch(rawUrl: string): Promise<Response> {
  const url = new URL(rawUrl); // throws on garbage
  if (url.protocol !== 'https:') throw new Error('only https');

  const { address } = await lookup(url.hostname);
  if (PRIVATE_RANGES.some((r) => r.test(address))) {
    throw new Error('private IP refused');
  }

  // Connect by IP to avoid DNS rebinding; pass the original Host header
  const ipUrl = new URL(url);
  ipUrl.hostname = address;
  return fetch(ipUrl, { headers: { Host: url.hostname } });
}
```

### 11.5 Notable CVEs

- **CVE-2021-26855 (ProxyLogon)** — SSRF in Exchange Server pre-auth, chained with arbitrary file write for RCE.
- **CVE-2019-5418** (Rails) — file content disclosure via crafted `Accept` header; SSRF-adjacent.

---

## 12. What's Likely in OWASP Top 10 2025

OWASP publishes the data-call results before the final list. As of mid-2026, drafts circulating in the security community suggest the 2025 list will likely:

- **Promote a "Supply Chain" category** that absorbs A06 (Vulnerable Components) and A08 (Integrity Failures) into a unified bucket reflecting the post-Log4Shell, post-SolarWinds reality.
- **Add or elevate "AI/LLM-related" risks** — prompt injection, training-data poisoning, insecure output handling. OWASP already publishes a separate **Top 10 for LLM Applications** which is converging with mainstream AppSec.
- **Keep Broken Access Control at #1** — incidence rates have not improved.

Treat this section as forward-looking, not authoritative. Verify against `owasp.org/Top10` when the 2025 list ships.

---

## 13. Mitigation Patterns That Cut Across Categories

A handful of practices defeat several Top 10 categories at once.

### 13.1 Use Frameworks Securely by Default

- Spring Security, Express + helmet + express-validator, Rails, Django — they all have safe defaults. Don't bypass.
- React/Vue/Angular escape output by default. Don't reach for `dangerouslySetInnerHTML`.

### 13.2 Validate at the Boundary with a Schema

Zod (TS), Joi (TS), Bean Validation `@Valid` (Java), Pydantic (Python). Reject unknown fields, enforce types, reject objects where primitives are expected (kills NoSQLi).

```typescript
import { z } from 'zod';

const LoginSchema = z.object({
  email: z.string().email().max(254),
  password: z.string().min(8).max(128),
});

app.post('/login', (req, res) => {
  const parsed = LoginSchema.safeParse(req.body);
  if (!parsed.success) return res.status(400).json({ error: 'invalid' });
  // parsed.data is fully typed and validated
});
```

### 13.3 Principle of Least Privilege

- Database users with only the grants they need (no `SUPERUSER`).
- IAM roles scoped to specific resources, not `Resource: *`.
- Container security contexts: `runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`.

### 13.4 Defense in Depth

Multiple independent controls, so no single bypass causes a breach. WAF + framework escaping + parameterized queries + DB principle of least privilege all defend against SQLi.

### 13.5 Secure Defaults > Documented Best Practices

If your library requires the developer to remember to escape, validate, or set a flag, it will fail in production. Frameworks that default safe (React's text escaping, Spring's parameterized queries, helmet's headers) win in the long run.

### 13.6 Logging and Detection

You will not prevent every attack. Detect and respond. See [A09](#10-a092021--security-logging-and-monitoring-failures).

---

## Related

- [Threat Modeling with STRIDE](02-threat-modeling-stride.md)
- [Authentication vs Authorization](03-authentication-vs-authorization.md)
- [OAuth 2.0 and OIDC Deep Dive](04-oauth2-and-oidc-deep-dive.md)
- [JWT Design and Pitfalls](05-jwt-design-and-pitfalls.md)
- [TLS Handshake and Certificates](06-tls-handshake-and-certificates.md)
- [Secret Management](07-secret-management.md)
- [Spring Security Filter Chain](../../java/security/security-filter-chain.md)
- [System Design — Defense in Depth and Threat Modeling](../../system-design/security/defense-in-depth-and-threat-modeling.md)

---

## References

- **OWASP Top 10 2021** — https://owasp.org/Top10/
- **OWASP Top 10 for LLM Applications** — https://owasp.org/www-project-top-10-for-large-language-model-applications/
- **CWE — Common Weakness Enumeration** — https://cwe.mitre.org/
- **NIST SP 800-63B-3** — Digital Identity Guidelines: Authentication and Lifecycle Management. https://pages.nist.gov/800-63-3/sp800-63b.html
- **CVE-2021-44228 (Log4Shell)** — https://nvd.nist.gov/vuln/detail/CVE-2021-44228
- **CVE-2017-5638 (Struts/Equifax)** — https://nvd.nist.gov/vuln/detail/CVE-2017-5638
- **CVE-2014-0160 (Heartbleed)** — https://nvd.nist.gov/vuln/detail/CVE-2014-0160
- **CVE-2014-3566 (POODLE)** — https://nvd.nist.gov/vuln/detail/CVE-2014-3566
- **CVE-2021-26855 (ProxyLogon)** — https://nvd.nist.gov/vuln/detail/CVE-2021-26855
- **Capital One 2019 Breach Postmortem** — https://www.capitalone.com/digital/facts2019/
- **AWS IMDSv2 Documentation** — https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html
- **OWASP SSRF Prevention Cheat Sheet** — https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- **OWASP Cheat Sheet Series** (catalog) — https://cheatsheetseries.owasp.org/
- **Verizon Data Breach Investigations Report (DBIR)** — https://www.verizon.com/business/resources/reports/dbir/
