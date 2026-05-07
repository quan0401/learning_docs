# Security Documentation Index — Learning Path

A progressive path through application and infrastructure security for backend engineers. Practical and protocol-grounded — connects threat models, identity, transport security, and secret handling to what you actually ship as a TypeScript/Node or Java/Spring developer.

Cross-references to the [Networking learning path](../networking/INDEX.md), the [Java learning path](../java/INDEX.md) (Spring Security tier), the [TypeScript learning path](../typescript/INDEX.md), the [Go learning path](../golang/INDEX.md) (`crypto/tls`, `crypto/subtle`, `net/http` security headers), the [Kubernetes learning path](../kubernetes/INDEX.md) (Pod Security, RBAC), the [System Design learning path](../system-design/INDEX.md) (security tier), the [Behavioral Interviews learning path](../behavioral-interviews/INDEX.md) (the interview-stage counterpart — STAR, story banking, and the question-by-category bank), and the [Web Scraping learning path](../web-scraping/INDEX.md) (OSINT and external attack-surface mapping — recon as the offensive mirror of OWASP/STRIDE) where topics overlap.

**Markers:** **★** = core must-learn (everyday backend security work, common in interviews and incident response). **○** = supporting deep-dive (specialized or compliance-driven). Internalize all ★ before going deep on ○.

---

## Tier 1 — Fundamentals: The Backend Security Baseline

The minimum every backend engineer needs to reason about threats, identity, transport, and secrets without fabricating their own crypto or auth scheme.

1. [★ OWASP Top 10 (2021) — The Vulnerability Landscape](fundamentals/01-owasp-top-10.md) — A01 Broken Access Control through A10 SSRF, real-world examples in Node and Spring, mitigation patterns, 2025 update notes _(2026-05-03)_
2. [★ Threat Modeling with STRIDE — Designing Security In](fundamentals/02-threat-modeling-stride.md) — STRIDE categories, data-flow diagrams, trust boundaries, when to threat model, comparison with PASTA and LINDDUN _(2026-05-03)_
3. [★ Authentication vs Authorization — Identity, Sessions, and Access Models](fundamentals/03-authentication-vs-authorization.md) — authn factors, session vs token auth, RBAC vs ABAC vs ReBAC vs PBAC, IDOR/BOLA, principle of least privilege _(2026-05-03)_
4. [★ OAuth 2.0 and OpenID Connect — Delegated Authorization and Federated Identity](fundamentals/04-oauth2-and-oidc-deep-dive.md) — RFC 6749 grant types, PKCE (RFC 7636), ID token vs access token, scopes vs claims, JWKS, state/CSRF pitfalls _(2026-05-03)_
5. [★ JWT Design and Pitfalls — Claims, Algorithms, and Storage](fundamentals/05-jwt-design-and-pitfalls.md) — RFC 7519, alg confusion, kid injection, JWS vs JWE, cookie vs localStorage, revocation, refresh rotation _(2026-05-03)_
6. [★ TLS Handshake and Certificates — Encryption in Transit, Done Right](fundamentals/06-tls-handshake-and-certificates.md) — TLS 1.3 vs 1.2, ECDHE, X.509 chain, SNI, ALPN, OCSP stapling, CT logs, ACME, mTLS, common failure modes _(2026-05-03)_
7. [★ Secret Management — Stores, Rotation, and Leak Detection](fundamentals/07-secret-management.md) — env vars vs secret stores, Vault, AWS/GCP Secrets Manager, K8s Secrets, SOPS/age, KMS envelope, rotation, gitleaks _(2026-05-03)_

---

## Tier 2 — Application-Layer Attacks

Deeper coverage of the injection and parsing-class vulnerabilities the OWASP Top 10 only sketches. Each gets its own document with exploit walkthroughs and language-specific defenses.

1. [★ SQL Injection Deep Dive — Parameterized Queries, ORMs, and Second-Order SQLi](web-attacks/sql-injection-deep-dive.md) — JDBC/Prisma/Sequelize/JPA/MyBatis safe patterns, ORDER BY column injection, IN-list arity, second-order SQLi, blind SQLi, NoSQL `$where` (Mongoose CVE-2024-53900 / CVE-2025-23061), defense in depth, `pg_stat_statements` detection _(2026-05-07)_
- _XSS — Reflected, Stored, DOM-based, and Modern CSP (planned)_
- _SSRF — Cloud Metadata, DNS Rebinding, and Allowlists (planned)_
- _XXE and Unsafe XML/YAML Parsing (planned)_
- _Insecure Deserialization — Java, Node, and Pickle (planned)_
- _CSRF — SameSite, Tokens, and Double-Submit Cookies (planned)_

---

## Tier 3 — Cryptography for Backend Devs (planned)

Just enough crypto to stop misusing it. No new ciphers, no rolling-your-own, but a clear mental model for what each primitive guarantees.

- _Symmetric Encryption — AES-GCM, ChaCha20-Poly1305, AEAD (planned)_
- _Asymmetric Crypto — RSA vs ECDSA vs Ed25519, Hybrid Encryption (planned)_
- _Hashing and Password Storage — bcrypt, scrypt, argon2id (planned)_
- _Random Numbers and Key Derivation — CSPRNG, HKDF, PBKDF2 (planned)_
- _MACs and Signatures — HMAC, Signing vs Encrypting (planned)_

---

## Tier 4 — Supply-Chain and SBOM (planned)

The dependencies you didn't write are still your attack surface.

- _Dependency Risk — npm, Maven Central, and Typosquatting (planned)_
- _SBOM — SPDX, CycloneDX, and SLSA Levels (planned)_
- _Sigstore and Signed Builds (planned)_
- _Container Image Provenance — Cosign, Notary v2 (planned)_

---

## Tier 5 — Cloud and Infrastructure Security (planned)

The control plane your code runs on.

- _IAM Fundamentals — AWS, GCP, Azure Identity Models (planned)_
- _Network Segmentation — VPC, Security Groups, and Private Service Connect (planned)_
- _Pod and Container Security — Seccomp, AppArmor, gVisor (planned)_
- _Audit Logging and Detection — CloudTrail, GuardDuty, Falco (planned)_
- _Compliance Baselines — SOC 2, ISO 27001, PCI-DSS (planned)_

---

## Quick Reference by Topic

### Fundamentals

- [OWASP Top 10 (2021)](fundamentals/01-owasp-top-10.md)
- [Threat Modeling (STRIDE)](fundamentals/02-threat-modeling-stride.md)
- [Authentication vs Authorization](fundamentals/03-authentication-vs-authorization.md)
- [OAuth 2.0 and OIDC](fundamentals/04-oauth2-and-oidc-deep-dive.md)
- [JWT Design and Pitfalls](fundamentals/05-jwt-design-and-pitfalls.md)
- [TLS Handshake and Certificates](fundamentals/06-tls-handshake-and-certificates.md)
- [Secret Management](fundamentals/07-secret-management.md)

### Application-Layer Attacks

- [SQL Injection Deep Dive](web-attacks/sql-injection-deep-dive.md) _(2026-05-07)_
- _XSS (planned)_
- _SSRF (planned)_
- _XXE (planned)_
- _Insecure Deserialization (planned)_
- _CSRF (planned)_

### Cryptography

- _Symmetric Encryption (planned)_
- _Asymmetric Crypto (planned)_
- _Password Hashing (planned)_
- _Random and KDFs (planned)_
- _MACs and Signatures (planned)_

### Supply Chain

- _Dependency Risk (planned)_
- _SBOM (planned)_
- _Sigstore (planned)_
- _Container Image Provenance (planned)_

### Cloud and Infrastructure

- _IAM Fundamentals (planned)_
- _Network Segmentation (planned)_
- _Pod and Container Security (planned)_
- _Audit Logging (planned)_
- _Compliance Baselines (planned)_

---

## Bug Spotting

Active-recall practice docs. Each presents 22+ broken snippets organized by difficulty (Easy / Subtle / Senior trap), with one-line `<details>` hints inline and full root-cause + fix in a Solutions section. Every bug cites a real reference (RFC, CVE, official-doc gotcha, postmortem, library issue). Use these to pressure-test concept knowledge after working through the tiers above.

- [JWT — Bug Spotting](fundamentals/jwt-bug-spotting.md) ★ — _(2026-05-03)_

---

## Related Learning Paths

- [Spring Security tier in the Java path](../java/INDEX.md) — concrete `SecurityFilterChain`, OAuth2 resource server, OIDC, and Spring Cloud Vault implementations
- [TLS & Certificates in the Networking path](../networking/application-layer/tls-and-certificates.md) — the protocol view of what Tier 1 doc 6 covers from the engineer view
- [Zero-Trust Networking](../networking/advanced/zero-trust.md) — BeyondCorp, identity-aware proxies, microsegmentation
- [Kubernetes RBAC and Pod Security](../kubernetes/security/) — cluster-level identity and workload isolation
- [System Design — Security tier](../system-design/security/) — architecture-level treatment of authn, authz, encryption, and zero trust
