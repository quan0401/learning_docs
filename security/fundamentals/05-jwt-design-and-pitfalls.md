---
title: "JWT Design and Pitfalls — Claims, Algorithms, and Storage"
date: 2026-05-03
updated: 2026-05-03
tags: [security, jwt, jws, jwe, tokens, claims]
---

# JWT Design and Pitfalls — Claims, Algorithms, and Storage

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `security` `jwt` `jws` `jwe` `tokens` `claims`

---

## Table of Contents

- [Summary](#summary)
- [1. JWT Anatomy](#1-jwt-anatomy)
- [2. Standard Claims](#2-standard-claims)
- [3. JWS vs JWE](#3-jws-vs-jwe)
- [4. Algorithm Pitfalls](#4-algorithm-pitfalls)
  - [4.1 alg=none](#41-algnone)
  - [4.2 HS vs RS Confusion](#42-hs-vs-rs-confusion)
  - [4.3 kid Header Injection](#43-kid-header-injection)
  - [4.4 jku and x5u Header Injection](#44-jku-and-x5u-header-injection)
  - [4.5 Embedded Public Key (jwk)](#45-embedded-public-key-jwk)
- [5. Storage on the Client](#5-storage-on-the-client)
- [6. Revocation Strategies](#6-revocation-strategies)
- [7. JWT vs Opaque Tokens](#7-jwt-vs-opaque-tokens)
- [8. Refresh Token Rotation](#8-refresh-token-rotation)
- [9. Validation Checklist](#9-validation-checklist)
- [10. Concrete Examples](#10-concrete-examples)
- [Related](#related)
- [References](#references)

---

## Summary

JSON Web Tokens (RFC 7519) are everywhere — OIDC ID tokens, OAuth access tokens (in many providers), session tokens for SPAs, service-to-service auth. They're also a graveyard of CVEs. JWTs are deceptively easy to use and brutally easy to misuse: the signature algorithm is *negotiated by the token itself*, the validation interface is full of footguns, and storing them safely on a browser is a multi-layer puzzle.

This document covers the structure (header / payload / signature), standard claims (`iss`/`sub`/`aud`/`exp`/`nbf`/`iat`/`jti`), the difference between JWS (signed) and JWE (encrypted), the well-documented algorithm-confusion attacks, where to store tokens on web/mobile clients, how to handle revocation despite JWT's stateless premise, refresh-token rotation, and the conditions under which you should reach for opaque tokens instead. Throughout, the emphasis is on what to actually configure in `jsonwebtoken`, `jose`, `nimbus-jose-jwt`, and Spring Security's resource server — and what to refuse.

---

## 1. JWT Anatomy

A JWT is three Base64URL-encoded segments separated by dots:

```
HEADER.PAYLOAD.SIGNATURE
```

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjJjNGUifQ
.eyJpc3MiOiJodHRwczovL2F1dGguZXhhbXBsZS5jb20iLCJzdWIiOiJ1c2VyXzQyIiwiYXVkIjoibXktYXBpIiwiZXhwIjoxNzE0NzI2ODAwLCJpYXQiOjE3MTQ3MjMyMDAsInNjb3BlIjoicmVhZDppbnZvaWNlcyJ9
.<binary signature, base64url>
```

Decoded:

```json
// Header
{ "alg": "RS256", "typ": "JWT", "kid": "2c4e" }

// Payload (claims)
{
  "iss": "https://auth.example.com",
  "sub": "user_42",
  "aud": "my-api",
  "exp": 1714726800,
  "iat": 1714723200,
  "scope": "read:invoices"
}

// Signature: RSA-SHA256 over BASE64URL(header) + "." + BASE64URL(payload)
```

**Critical property:** anyone with the token can read the claims. Base64URL is *encoding*, not encryption. Anything secret must go in JWE (encrypted) or stay out of the token entirely.

---

## 2. Standard Claims

RFC 7519 reserves these names:

| Claim | Meaning | Validation |
|-------|---------|------------|
| `iss` | Issuer | String-equal to expected issuer |
| `sub` | Subject (the principal — usually user ID) | Use as opaque identifier |
| `aud` | Audience | Must include this resource server's identifier |
| `exp` | Expiration time (epoch seconds) | Reject if `now > exp` (with small skew tolerance, e.g., 30 s) |
| `nbf` | Not Before | Reject if `now < nbf` |
| `iat` | Issued At | Sanity check; reject far-future values |
| `jti` | JWT ID — unique per token | For replay detection / revocation lists |

OIDC adds `nonce`, `auth_time`, `acr`, `amr`, `azp`. Custom claims are allowed but should be namespaced (e.g., `https://example.com/roles`) to avoid collision.

### 2.1 Don't Stuff the Payload

Common mistake: putting an entire user profile into the JWT. Drawbacks:

- Larger over-the-wire size on every request.
- Stale data — the JWT lives until `exp`, but the user's profile may have changed.
- More PII spread across logs and CDN caches.

Put just enough to authorize: `sub`, `roles`/`scope`, and tenant identifier. Look up the rest from the database.

---

## 3. JWS vs JWE

| | JWS (RFC 7515) | JWE (RFC 7516) |
|--|----------------|----------------|
| Provides | Integrity + authenticity | Confidentiality + integrity |
| Body | Visible (base64url, not encrypted) | Encrypted |
| Use | Authentication tokens, ID tokens | Tokens carrying secrets you can't put in plaintext |

The vast majority of "JWTs" you encounter are JWS — signed and integrity-protected, but **not** encrypted.

If your token must carry sensitive data the resource server needs (e.g., decrypted on its way through a service mesh), use JWE — but first ask whether you should put that data anywhere at all. Most of the time the right answer is "store the data server-side, reference it by ID".

A **nested JWT** is JWS-inside-JWE: the outer layer encrypts so untrusted middleboxes can't read it, the inner layer is signed so the recipient knows who issued it. Standard pattern in some FAPI 2.0 deployments.

---

## 4. Algorithm Pitfalls

The single most exploited surface in JWT libraries is the algorithm negotiation in the header. The token says how it was signed, and naïve validators believe it.

### 4.1 alg=none

RFC 7519 includes an `alg=none` value for "unsigned JWT". A library that accepts whatever the header says will accept a token with no signature.

```
{ "alg": "none", "typ": "JWT" }.{ "sub": "admin" }.
```

**Mitigation.** Never call `verify(token)` without specifying allowed algorithms. Reject `none`. All modern libraries refuse `none` unless explicitly enabled, but some old code (and some hand-rolled implementations) doesn't.

### 4.2 HS vs RS Confusion

The classic JWT bug, captured by **CVE-2015-9235** (jsonwebtoken), **CVE-2016-10555** (node-jsonwebtoken auth0), **CVE-2018-0114** (Cisco node-jose), and many siblings.

The setup: the resource server has the AS's **RSA public key** to verify `RS256` tokens. The library's `verify(token, key)` accepts the algorithm from the token header. An attacker:

1. Takes the public key (it's public — published at the JWKS URL).
2. Crafts a token with `alg=HS256`.
3. Signs it with HMAC-SHA256, using the *public key bytes* as the HMAC secret.
4. Sends it.

The library reads `alg=HS256` from the header, treats the configured key as a symmetric secret, and HMAC-validates the token. **Pass.**

**Mitigation.** Pin algorithms.

```typescript
// jose — algorithms allowlist required
await jwtVerify(token, JWKS, {
  algorithms: ['RS256'],
});
```

```javascript
// jsonwebtoken
jwt.verify(token, publicKey, { algorithms: ['RS256'] });  // mandatory
```

```java
// Spring Security resource server — pin via decoder
NimbusJwtDecoder decoder = NimbusJwtDecoder
    .withJwkSetUri("https://issuer/.well-known/jwks.json")
    .jwsAlgorithm(SignatureAlgorithm.RS256)
    .build();
```

### 4.3 kid Header Injection

The `kid` (key ID) field tells the validator which key in the JWKS to use. If the validator uses `kid` to query a database (`SELECT key FROM keys WHERE kid = ?`), an attacker passes `kid='; DELETE FROM keys; --` for SQLi, or `kid=../../etc/passwd` for path traversal, or `kid=<URL of attacker-controlled key>`.

**Mitigation.** Treat `kid` as an untrusted key identifier. Lookup by exact string in a known set; never use it in dynamic SQL or file paths. Most JWKS-based libraries do this correctly; bespoke implementations frequently don't.

### 4.4 jku and x5u Header Injection

JWS headers can include `jku` (JWK Set URL) or `x5u` (X.509 URL) pointing to where the validator should fetch the verification key. If the validator fetches whatever URL the token says, an attacker hosts their own JWKS and signs a token with their own key.

**Mitigation.** Ignore `jku`/`x5u` from the token. Configure the validator with the issuer's expected JWKS URI, period.

### 4.5 Embedded Public Key (jwk)

JWS allows embedding the public key directly in the header. A naïve validator uses the embedded key. Same outcome: attacker provides the key, signs the token, validator says yes.

**Mitigation.** Same as `jku` — never use the token-supplied key. Library defaults vary; verify yours rejects this.

---

## 5. Storage on the Client

The browser-storage debate has produced more bad blog posts than perhaps any other security topic. The honest version:

| Storage | XSS exposure | CSRF exposure | Notes |
|---------|--------------|---------------|-------|
| `localStorage` | **Read by XSS** | None | Easy but XSS = total compromise |
| `sessionStorage` | **Read by XSS** | None | Same as localStorage but cleared on tab close |
| **HttpOnly cookie** | Not readable by JS | Possible (mitigated by SameSite) | The token isn't accessible to your JS, only sent automatically |
| In-memory variable | Lost on refresh | None | Used briefly for short-lived access tokens; combine with refresh in cookie |

The realistic threat model: **XSS is more common than CSRF in modern apps with same-site defaults**. SameSite=Lax (default in Chromium since 2020) blocks classic CSRF. XSS is what eats your tokens.

### 5.1 Backend-for-Frontend (BFF) Pattern

The most defensible SPA architecture in 2026:

```
Browser → SPA → BFF → API
            ↑
         HttpOnly session cookie
                ↓
          BFF holds OAuth tokens server-side
```

The browser holds only an opaque session cookie. The BFF performs the OAuth dance, stores tokens in a server-side store, and proxies API calls. XSS still hurts (the attacker can issue authed requests through your origin), but the tokens themselves are not exfiltratable — you can rotate the session, see all activity in BFF logs, and apply rate limits and anomaly detection at the BFF.

Spring Cloud Gateway, Auth.js (NextAuth), and Keycloak all support BFF patterns.

### 5.2 If You Must Use localStorage

If your architecture forces tokens in JS (e.g., a static SPA without a BFF):

- **Short access-token lifetime** (5–15 min).
- **Refresh-token rotation with reuse detection.**
- **Strict Content Security Policy** to make XSS injection harder.
- **No third-party scripts in the SPA origin.**
- Consider **DPoP** (RFC 9449) so a stolen token isn't usable from another machine.

### 5.3 Mobile Apps

Use the platform's secure storage:

- iOS: Keychain.
- Android: EncryptedSharedPreferences / Keystore.
- React Native: react-native-keychain.

Don't put tokens in plain `AsyncStorage` or `SharedPreferences`.

---

## 6. Revocation Strategies

JWT's headline feature is statelessness — and statelessness is the enemy of revocation. You can't tell a "no longer valid" JWT from a "still valid" one without state somewhere.

### 6.1 Just Live With Short Lifetimes

For most applications, access tokens with `exp` 5–15 minutes is the answer. Compromised tokens expire quickly. Pair with refresh-token rotation.

### 6.2 Token Blocklist on `jti`

Each token has a unique `jti`. On revoke (logout, role change, password reset, "sign out everywhere"), add `jti` (or `sub` + `iat`) to a Redis set with TTL = token lifetime. Resource servers check the set on every request.

```typescript
// On logout
await redis.set(`blocklist:${jti}`, '1', 'EX', secondsUntilExp);

// On request
if (await redis.get(`blocklist:${jti}`)) return res.sendStatus(401);
```

This re-introduces a per-request lookup, but it's a single Redis read.

### 6.3 Versioned Subjects

Add a `tokenVersion` claim. Bump the user's version in the DB on logout-everywhere or password change. Resource servers compare. Cheap, but requires a DB lookup unless cached.

### 6.4 Short Access + Long Refresh + Refresh-Side Revocation

The most popular pragmatic approach in OAuth/OIDC:

- Access tokens: short, no revocation needed.
- Refresh tokens: server-side state at the AS. Revoke the refresh, and within `exp` of the access token, the user is fully out.

### 6.5 OAuth 2.0 Token Revocation Endpoint (RFC 7009)

A standard endpoint to tell the AS "this token is dead". Most providers expose `/revoke`. The AS handles the state; resource servers benefit indirectly via short access-token lifetimes.

---

## 7. JWT vs Opaque Tokens

| | JWT | Opaque |
|--|-----|--------|
| What it is | Self-describing, signed JSON | Random string, lookup-only |
| Validation | Local (signature + claims) | Remote (call AS introspection endpoint) |
| Revocation | Hard | Trivial (delete the row) |
| Per-request cost | CPU (signature verify) | Network (introspection call) |
| When to use | Internal microservices, short-lived access tokens, OIDC ID tokens | High-stakes systems where instant revocation matters; small-scale apps |

**RFC 7662** defines an introspection endpoint where the resource server posts a token to the AS and gets `{ active: true, sub, scope, ... }` back. The AS can cache, the resource server can cache shortly. This is the path some financial APIs choose for their access-token transport: short opaque tokens that introspect to the AS for every request.

**Common hybrid.** ID tokens (OIDC) are JWT (the client validates locally). Access tokens are *either* JWT or opaque, depending on provider. Auth0 issues JWT access tokens by default; Google issues opaque access tokens that are introspected at `/tokeninfo`.

---

## 8. Refresh Token Rotation

A refresh token's job is to be longer-lived than an access token, used once to get a new access token, and ideally never seen by anything except the AS.

**Rotation pattern:**

1. Client uses refresh token R1 to get access token A2 + refresh token R2.
2. AS marks R1 as used.
3. Client discards R1, stores R2.
4. If R1 is ever used again, the AS treats it as token theft and revokes the entire token family — both R2 and any access tokens derived from it.

This is the standard behavior in Auth0, Okta, AWS Cognito, Keycloak's "refresh token rotation" mode.

```
Client     AS
  |  refresh(R1)            |
  |───────────────────────▶|
  |  { A2, R2 }             |
  |◀───────────────────────|
  ...
  |  refresh(R1) (replay!)  |
  |───────────────────────▶|
  | revoke family. 401.     |
  |◀───────────────────────|
```

Combined with short access tokens and HttpOnly storage, this is a strong baseline for SPAs without a BFF.

---

## 9. Validation Checklist

Every JWT consumer must:

- [ ] **Verify signature** against the expected public key (or HMAC secret).
- [ ] **Pin algorithms** to an allowlist (e.g., `['RS256']`). Never accept whatever the header says.
- [ ] **Validate `iss`** — exact string match.
- [ ] **Validate `aud`** — must contain this service's identifier.
- [ ] **Validate `exp`** — current time < exp, with small skew tolerance.
- [ ] **Validate `nbf`** if present.
- [ ] **Validate `iat`** is reasonable (not far in the future).
- [ ] **Validate `nonce`** for OIDC ID tokens — matches the nonce sent in `/authorize`.
- [ ] **Reject `alg=none`** explicitly.
- [ ] **Ignore token-supplied keys** (`jwk`, `jku`, `x5u`).
- [ ] **Check revocation** if your model demands it (blocklist, version, introspection).
- [ ] **Cap token size** to prevent JWS-bomb DoS.

In Node, `jose` enforces most of this if you pass the right options. In Java, Spring Security's resource server with JWKS does the right thing by default. Don't roll your own.

---

## 10. Concrete Examples

### 10.1 Issuing a Token (Node)

```typescript
import { SignJWT } from 'jose';

const secret = new TextEncoder().encode(process.env.JWT_SECRET);

export async function issueToken(userId: string, scopes: string[]) {
  return new SignJWT({ scope: scopes.join(' ') })
    .setProtectedHeader({ alg: 'HS256', typ: 'JWT' })
    .setIssuer('https://auth.example.com')
    .setSubject(userId)
    .setAudience('my-api')
    .setIssuedAt()
    .setExpirationTime('15m')
    .setJti(crypto.randomUUID())
    .sign(secret);
}
```

For production, use `RS256` with a key pair so resource servers don't need the symmetric secret.

### 10.2 Validating a Token (Node, RS256 + JWKS)

```typescript
import { createRemoteJWKSet, jwtVerify } from 'jose';

const JWKS = createRemoteJWKSet(new URL('https://auth.example.com/.well-known/jwks.json'));

export async function verifyToken(token: string) {
  const { payload } = await jwtVerify(token, JWKS, {
    issuer: 'https://auth.example.com',
    audience: 'my-api',
    algorithms: ['RS256'],
    clockTolerance: 30,
  });
  // Optional revocation check
  if (await redis.get(`blocklist:${payload.jti}`)) {
    throw new Error('revoked');
  }
  return payload;
}
```

### 10.3 Spring Security — Resource Server with JWT

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
          audiences: my-api
```

```java
@Configuration
@EnableMethodSecurity
public class ResourceServerConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a
                .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .requestMatchers("/api/**").authenticated()
                .anyRequest().denyAll())
            .oauth2ResourceServer(rs -> rs.jwt(jwt -> jwt
                .jwtAuthenticationConverter(this::convertJwt)));
        return http.build();
    }

    private AbstractAuthenticationToken convertJwt(Jwt jwt) {
        Collection<GrantedAuthority> auths = JwtGrantedAuthoritiesConverter.builder()
            .authoritiesClaimName("scope")
            .authorityPrefix("SCOPE_")
            .build()
            .convert(jwt);
        return new JwtAuthenticationToken(jwt, auths, jwt.getSubject());
    }
}
```

Spring Security validates signature, `iss`, `aud`, `exp`, `nbf`, `iat`, and clock skew. It pins `RS256` (or whatever the JWKS publishes) automatically and ignores `jku`/`x5u`/`jwk` from the token.

### 10.4 What Bad Validation Looks Like

```typescript
// DANGEROUS — accepts any algorithm including none
const decoded = jwt.verify(token, key);

// DANGEROUS — no aud/iss check
jwt.verify(token, key, { algorithms: ['RS256'] });

// DANGEROUS — uses the kid from the token to load a key from disk
const key = fs.readFileSync(`/keys/${decoded.header.kid}.pem`);

// DANGEROUS — manually decodes without verifying
const claims = JSON.parse(Buffer.from(token.split('.')[1], 'base64url').toString());
```

If you see any of the above in a code review, block the PR.

---

## Related

- [OAuth 2.0 and OIDC Deep Dive](04-oauth2-and-oidc-deep-dive.md) — JWT is the primary token format used there
- [Authentication vs Authorization](03-authentication-vs-authorization.md) — JWT is one authn vehicle among several
- [TLS Handshake and Certificates](06-tls-handshake-and-certificates.md) — sender-constrained tokens via mTLS
- [OWASP Top 10 (2021)](01-owasp-top-10.md) — A02 (Cryptographic Failures) and A07 (Auth Failures)
- [Spring Security — OAuth2 and JWT](../../java/security/oauth2-jwt.md)

---

## References

- **RFC 7519** — JSON Web Token (JWT). https://www.rfc-editor.org/rfc/rfc7519
- **RFC 7515** — JSON Web Signature (JWS). https://www.rfc-editor.org/rfc/rfc7515
- **RFC 7516** — JSON Web Encryption (JWE). https://www.rfc-editor.org/rfc/rfc7516
- **RFC 7517** — JSON Web Key (JWK). https://www.rfc-editor.org/rfc/rfc7517
- **RFC 7518** — JSON Web Algorithms (JWA). https://www.rfc-editor.org/rfc/rfc7518
- **RFC 7009** — OAuth 2.0 Token Revocation. https://www.rfc-editor.org/rfc/rfc7009
- **RFC 7662** — OAuth 2.0 Token Introspection. https://www.rfc-editor.org/rfc/rfc7662
- **RFC 9449** — DPoP: OAuth 2.0 Demonstrating Proof of Possession. https://www.rfc-editor.org/rfc/rfc9449
- **RFC 8725** — JSON Web Token Best Current Practices. https://www.rfc-editor.org/rfc/rfc8725
- **CVE-2018-0114** — Cisco node-jose alg confusion. https://nvd.nist.gov/vuln/detail/CVE-2018-0114
- **CVE-2015-9235** — jsonwebtoken alg confusion. https://nvd.nist.gov/vuln/detail/CVE-2015-9235
- **CVE-2016-10555** — Auth0 jsonwebtoken algorithm confusion. https://nvd.nist.gov/vuln/detail/CVE-2016-10555
- **OWASP JWT Cheat Sheet** — https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html
- **Auth0 — Refresh Token Rotation** — https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation
- **jose (Node)** — https://github.com/panva/jose
- **Nimbus JOSE+JWT (Java)** — https://connect2id.com/products/nimbus-jose-jwt
- **Spring Security Resource Server** — https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/index.html
