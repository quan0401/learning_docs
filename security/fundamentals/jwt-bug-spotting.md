---
title: "JWT — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, jwt, auth, security]
---

# JWT — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `jwt` `auth` `security`

---

## Table of Contents
1. [How to use this doc](#how-to-use-this-doc)
2. [Easy (warm-up traps)](#1-easy-warm-up-traps)
3. [Subtle (review-passers)](#2-subtle-review-passers)
4. [Senior trap (production-only failures)](#3-senior-trap-production-only-failures)
5. [Solutions](#4-solutions)
6. [Related](#related)
7. [References](#references)

## Summary

26 broken JWT snippets across `jsonwebtoken` (Node), `jose`, `java-jwt`, `PyJWT`, and `go-jose`. Coverage spans the RFC 8725 BCP catalogue: `alg=none`, RS256↔HS256 confusion, `kid`/`jku`/`x5u` header injection, weak HS256 secrets, missing `aud`/`iss`/`exp`/`nbf` validation, unsafe storage, refresh rotation gaps, JWE misuse, and the Java 15–18 ECDSA `(0, 0)` signature trap (CVE-2022-21449). Aim for one pass weekly until the failure modes are reflexive.

## How to use this doc
- Try to spot the bug before opening `<details>`.
- The hint inside `<details>` is one line; the full root cause + fix is in §4 Solutions, keyed by bug number.
- Difficulty sections are cumulative — don't skip Easy unless you've nailed those traps in code review.

---

## 1. Easy (warm-up traps)

### Bug 1 — verify without algorithms list (Node `jsonwebtoken`)
```ts
import jwt from "jsonwebtoken";

export function authenticate(token: string, secret: string) {
  // RS256 issuer, but…
  return jwt.verify(token, secret);
}
```
<details><summary>Hint</summary>
What does the verifier do when the token's header says <code>alg: none</code>?
</details>

### Bug 2 — accepting `alg=none` (PyJWT pre-2.x style)
```python
import jwt

def decode(token: str) -> dict:
    return jwt.decode(token, options={"verify_signature": False})
```
<details><summary>Hint</summary>
The function name is <code>decode</code> but the call is doing something stronger — read the options.
</details>

### Bug 3 — short HS256 secret
```ts
import jwt from "jsonwebtoken";
const SECRET = "s3cret";
export const sign = (payload: object) =>
  jwt.sign(payload, SECRET, { algorithm: "HS256" });
```
<details><summary>Hint</summary>
Hashcat eats this for breakfast. RFC 7518 §3.2 has a minimum.
</details>

### Bug 4 — JWT in `localStorage`
```ts
// after login
localStorage.setItem("access_token", res.data.token);

// on every API call
fetch("/api/orders", {
  headers: { Authorization: `Bearer ${localStorage.getItem("access_token")}` },
});
```
<details><summary>Hint</summary>
One stored XSS anywhere on the origin and the token walks out the door.
</details>

### Bug 5 — cookie missing security flags
```ts
res.cookie("session", token, { maxAge: 3600_000 });
```
<details><summary>Hint</summary>
Three flags: one stops JS access, one stops plaintext leak, one stops cross-site send.
</details>

### Bug 6 — sensitive data in payload
```ts
const token = jwt.sign(
  { sub: user.id, email: user.email, ssn: user.ssn, plan: user.plan },
  SECRET,
);
```
<details><summary>Hint</summary>
JWS is signed, not encrypted. <code>jwt.io</code> can read it.
</details>

### Bug 7 — token in URL
```ts
res.redirect(`https://app.example.com/callback?token=${signedJwt}`);
```
<details><summary>Hint</summary>
Where do URLs end up? Access logs, Referer headers, browser history, proxy logs.
</details>

---

## 2. Subtle (review-passers)

### Bug 8 — algorithm confusion: RS256 token verified as HS256
```ts
import jwt from "jsonwebtoken";
import { readFileSync } from "node:fs";

const publicKey = readFileSync("./rs256-public.pem", "utf8");

export function verify(token: string) {
  // We "support both" RS256 and HS256:
  return jwt.verify(token, publicKey);
}
```
<details><summary>Hint</summary>
Attacker re-signs with HS256 using the public key as the shared secret.
</details>

### Bug 9 — `kid` header used as a file path
```ts
import jwt from "jsonwebtoken";
import { readFileSync } from "node:fs";

export function verify(token: string) {
  const decoded = jwt.decode(token, { complete: true });
  const kid = decoded?.header.kid as string;
  const key = readFileSync(`/etc/app/keys/${kid}.pem`, "utf8");
  return jwt.verify(token, key, { algorithms: ["RS256"] });
}
```
<details><summary>Hint</summary>
What if <code>kid</code> contains <code>../../../../dev/null</code> or a path to a file the attacker controls?
</details>

### Bug 10 — `kid` header used in raw SQL
```python
import jwt, sqlite3

def verify(token, conn):
    header = jwt.get_unverified_header(token)
    kid = header["kid"]
    cur = conn.execute(f"SELECT pem FROM keys WHERE kid = '{kid}'")
    pem = cur.fetchone()[0]
    return jwt.decode(token, pem, algorithms=["RS256"])
```
<details><summary>Hint</summary>
String interpolation into SQL plus an attacker-chosen header value.
</details>

### Bug 11 — trusting `jku` to fetch a JWKS
```ts
import jwt from "jsonwebtoken";
import jwksClient from "jwks-rsa";

export async function verify(token: string) {
  const { header } = jwt.decode(token, { complete: true })!;
  const client = jwksClient({ jwksUri: header.jku as string }); // attacker-controlled
  const key = await client.getSigningKey(header.kid);
  return jwt.verify(token, key.getPublicKey(), { algorithms: ["RS256"] });
}
```
<details><summary>Hint</summary>
The token tells you where to fetch its own verification key. Spot the trust problem.
</details>

### Bug 12 — missing `audience` check
```ts
jwt.verify(token, publicKey, {
  algorithms: ["RS256"],
  issuer: "https://idp.example.com",
});
```
<details><summary>Hint</summary>
Same IdP issues tokens for the billing API, the admin API, and your service.
</details>

### Bug 13 — missing `issuer` check
```python
import jwt
payload = jwt.decode(token, public_key, algorithms=["RS256"], audience="api")
```
<details><summary>Hint</summary>
Anyone with a valid key for the right algorithm can mint a token your code accepts.
</details>

### Bug 14 — `exp` not enforced (Java `java-jwt`)
```java
import com.auth0.jwt.JWT;
import com.auth0.jwt.interfaces.DecodedJWT;

public DecodedJWT parse(String token) {
    // Note: no .verify(), no algorithm, no clock check.
    return JWT.decode(token);
}
```
<details><summary>Hint</summary>
<code>JWT.decode</code> is a parser, not a verifier. Read the Javadoc.
</details>

### Bug 15 — unbounded leeway/clock skew
```ts
jwt.verify(token, publicKey, {
  algorithms: ["RS256"],
  clockTolerance: 60 * 60 * 24, // "fixes" the flaky NTP issue
});
```
<details><summary>Hint</summary>
The unit is seconds. How long is the token effectively valid past <code>exp</code>?
</details>

### Bug 16 — `nbf` ignored
```python
jwt.decode(token, key, algorithms=["RS256"],
           options={"verify_nbf": False})
```
<details><summary>Hint</summary>
Replay a future-dated token now — what stops you?
</details>

### Bug 17 — refresh token reused with no rotation/jti tracking
```ts
app.post("/refresh", async (req, res) => {
  const claims = jwt.verify(req.body.refresh, REFRESH_SECRET) as JwtPayload;
  const access = jwt.sign({ sub: claims.sub }, ACCESS_SECRET, { expiresIn: "15m" });
  res.json({ access });
});
```
<details><summary>Hint</summary>
The same refresh string works forever once stolen. Where's the family/jti bookkeeping?
</details>

### Bug 18 — long-lived access tokens, no revocation
```go
// Issued at login
claims := jwt.MapClaims{
    "sub": user.ID,
    "exp": time.Now().Add(30 * 24 * time.Hour).Unix(), // 30 days
}
tok := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
signed, _ := tok.SignedString(secret)
```
<details><summary>Hint</summary>
User clicks "log out everywhere" — what actually invalidates this token?
</details>

### Bug 19 — symmetric secret shared across services
```yaml
# helm values shared by 8 microservices
jwt:
  secret: ${JWT_HS256_SECRET}   # same env var, same value, every service
```
<details><summary>Hint</summary>
Verifier and signer are the same key. One compromised pod = forge tokens for every service.
</details>

### Bug 20 — Bearer token logged
```ts
app.use((req, _res, next) => {
  console.log(`${req.method} ${req.url} auth=${req.headers.authorization}`);
  next();
});
```
<details><summary>Hint</summary>
Splunk/Datadog/CloudWatch will happily index that for years.
</details>

---

## 3. Senior trap (production-only failures)

### Bug 21 — ECDSA `(r=0, s=0)` accepted on Java 15–18 (CVE-2022-21449)
```java
import com.auth0.jwt.JWT;
import com.auth0.jwt.algorithms.Algorithm;
import java.security.interfaces.ECPublicKey;

public boolean verify(String token, ECPublicKey pub) {
    Algorithm algorithm = Algorithm.ECDSA256(pub, null);
    JWT.require(algorithm).build().verify(token); // throws on failure
    return true;
}
```
<details><summary>Hint</summary>
On vulnerable JDKs, what does <code>Signature.verify</code> return for an all-zero signature?
</details>

### Bug 22 — no `kid` in JWS, single key, no rotation path
```ts
// signer
const token = jwt.sign(payload, privatePem, { algorithm: "RS256" });
// verifier
jwt.verify(token, publicPem, { algorithms: ["RS256"] });
```
<details><summary>Hint</summary>
You're forced to roll a new key. How do you accept old + new tokens during the cutover?
</details>

### Bug 23 — JWE `alg: dir` with a 16-byte "key"
```ts
import { CompactEncrypt } from "jose";

const key = new TextEncoder().encode("super-secret-key");        // 16 bytes
const jwe = await new CompactEncrypt(new TextEncoder().encode(JSON.stringify(claims)))
  .setProtectedHeader({ alg: "dir", enc: "A256GCM" })
  .encrypt(key);
```
<details><summary>Hint</summary>
A256GCM needs how many bytes of key material? What does <code>dir</code> actually do?
</details>

### Bug 24 — gateway verifies, downstream re-trusts blindly
```ts
// API gateway
const claims = jwt.verify(req.headers.authorization!.slice(7), publicKey, {
  algorithms: ["RS256"], issuer: "idp", audience: "gateway",
});
proxyTo("orders-svc", { headers: { "X-User": claims.sub, "X-Roles": claims.roles } });

// orders-svc — no JWT check, just trusts the gateway
app.get("/orders", (req, res) => {
  if (req.headers["x-roles"]?.includes("admin")) return adminView(res);
  return userView(res, req.headers["x-user"]);
});
```
<details><summary>Hint</summary>
What stops a teammate from <code>kubectl port-forward</code>'ing orders-svc and curling it directly?
</details>

### Bug 25 — refresh reuse not detected (no jti family)
```ts
// /refresh
const claims = jwt.verify(refresh, secret) as JwtPayload;
const newRefresh = jwt.sign({ sub: claims.sub }, secret, { expiresIn: "30d" });
const newAccess  = jwt.sign({ sub: claims.sub }, secret, { expiresIn: "15m" });
// We mark the old one as "used" by… not doing anything.
return { newAccess, newRefresh };
```
<details><summary>Hint</summary>
Attacker steals refresh, both you and they call <code>/refresh</code>. RFC 6819 §5.2.2.3 has the detection pattern.
</details>

### Bug 26 — `jsonwebtoken` `verify(token, secret)` accepts both HS* and RS* (CVE-2015-9235 family)
```ts
// Versions of jsonwebtoken < 4.2.2 — and any modern version called this way:
import jwt from "jsonwebtoken";
const publicKey = readFileSync("./rsa-public.pem", "utf8");

export function verify(token: string) {
  return jwt.verify(token, publicKey); // no algorithms whitelist
}
```
<details><summary>Hint</summary>
This is Bug 1 + Bug 8 fused. Auth0 patched the historical case; the modern footgun is the same call shape.
</details>

---

## 4. Solutions

### Bug 1 — verify without algorithms list
**Root cause:** Without `algorithms`, older `jsonwebtoken` versions honored `alg` from the header — including `none` — and modern versions still let an attacker pick any algorithm the key is interpretable as. RFC 8725 §3.1 says verifiers MUST whitelist accepted algorithms.
**Fix:**
```ts
return jwt.verify(token, secret, { algorithms: ["RS256"] });
```
**Reference:** [RFC 8725 §3.1](https://datatracker.ietf.org/doc/html/rfc8725#section-3.1), CVE-2015-9235 (jsonwebtoken).

### Bug 2 — accepting `alg=none`
**Root cause:** `verify_signature: False` disables every check that makes a JWT a JWT. RFC 8725 §3.2 forbids verifying with `alg: none`.
**Fix:**
```python
return jwt.decode(token, key, algorithms=["RS256"],
                  audience="api", issuer="https://idp")
```
**Reference:** [RFC 8725 §3.2](https://datatracker.ietf.org/doc/html/rfc8725#section-3.2). Inversoft prime-jwt had the same shape: [CVE-2018-1000531](https://nvd.nist.gov/vuln/detail/CVE-2018-1000531).

### Bug 3 — short HS256 secret
**Root cause:** RFC 7518 §3.2 requires HS256 keys to be at least 256 bits. `"s3cret"` is 6 ASCII bytes; brute-forceable in seconds with hashcat mode 16500.
**Fix:**
```ts
const SECRET = process.env.JWT_HS256_SECRET; // 32+ random bytes, base64
```
**Reference:** [RFC 7518 §3.2](https://datatracker.ietf.org/doc/html/rfc7518#section-3.2); [PortSwigger — brute-forcing weak secrets](https://portswigger.net/web-security/jwt#brute-forcing-secret-keys).

### Bug 4 — JWT in `localStorage`
**Root cause:** `localStorage` is reachable from any JS on the origin. One XSS exfiltrates the token and there's no way to mark it HttpOnly.
**Fix:** Store in an `HttpOnly; Secure; SameSite=Lax|Strict` cookie or in memory (with silent refresh via cookie-bound refresh token).
**Reference:** [OWASP JWT for Java Cheat Sheet — Token Storage on the Client Side](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html#token-storage-on-the-client-side).

### Bug 5 — cookie missing security flags
**Root cause:** Without `httpOnly` JS reads it; without `secure` it leaks over HTTP; without `sameSite` it rides along with CSRF.
**Fix:**
```ts
res.cookie("session", token, {
  httpOnly: true, secure: true, sameSite: "lax", path: "/", maxAge: 3600_000,
});
```
**Reference:** [OWASP Session Management Cheat Sheet — Cookies](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#cookies).

### Bug 6 — sensitive data in payload
**Root cause:** JWS is base64url, not encrypted. Anyone holding the token reads the claims.
**Fix:** Keep PII server-side; reference by `sub`. If you truly need encrypted claims, use JWE (RFC 7516) with `alg: RSA-OAEP-256` + `enc: A256GCM`.
**Reference:** [RFC 7519 §6](https://datatracker.ietf.org/doc/html/rfc7519#section-6), [RFC 8725 §3.11](https://datatracker.ietf.org/doc/html/rfc8725#section-3.11).

### Bug 7 — token in URL
**Root cause:** URLs are logged in access logs, Referer headers, browser history, proxy caches, and SaaS log aggregators.
**Fix:** Return tokens in JSON response bodies or `Set-Cookie`. For OAuth callbacks, use the `form_post` response mode or PKCE-bound code exchange.
**Reference:** [RFC 6750 §5.3](https://datatracker.ietf.org/doc/html/rfc6750#section-5.3) ("Don't Pass Bearer Tokens in Page URLs").

### Bug 8 — RS256 token verified as HS256
**Root cause:** When the verifier doesn't pin the algorithm, an attacker re-signs the token with `alg: HS256` using the RSA **public key** as the HMAC secret. The library happily HMAC-verifies with the public key it has on file.
**Fix:**
```ts
return jwt.verify(token, publicKey, { algorithms: ["RS256"] });
```
**Reference:** [Auth0 — Critical vulnerabilities in JSON Web Token libraries (Tim McLean)](https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/), [PortSwigger — JWT algorithm confusion](https://portswigger.net/web-security/jwt/algorithm-confusion-attacks).

### Bug 9 — `kid` as file path
**Root cause:** The `kid` header is attacker-controlled. RFC 7515 §4.1.4 doesn't constrain its content — your code does.
**Fix:** Look `kid` up in a fixed allowlist or JWKS map; never concatenate into filesystem paths.
```ts
const KEY_BY_KID: Record<string, string> = { "2026-q2": pem1, "2026-q3": pem2 };
const key = KEY_BY_KID[decoded.header.kid as string];
if (!key) throw new Error("unknown kid");
```
**Reference:** [PortSwigger — kid header path traversal](https://portswigger.net/web-security/jwt#injection-via-the-kid-parameter), [RFC 8725 §3.5](https://datatracker.ietf.org/doc/html/rfc8725#section-3.5).

### Bug 10 — `kid` as raw SQL
**Root cause:** Same as Bug 9 in a different sink. `kid` is attacker-supplied; concatenating it into SQL is classic injection. A `kid` of `' UNION SELECT 'attacker_pem` returns whatever PEM the attacker wants.
**Fix:** Parameterize and validate against an allowlist.
```python
cur = conn.execute("SELECT pem FROM keys WHERE kid = ?", (kid,))
```
**Reference:** [PortSwigger — kid SQLi](https://portswigger.net/web-security/jwt#injection-via-the-kid-parameter).

### Bug 11 — trusting `jku`
**Root cause:** `jku`/`x5u` (RFC 7515 §4.1.2/§4.1.5) point to a JWKS URL. If you fetch whatever the token says, the attacker hosts the JWKS and signs with the matching private key.
**Fix:** Ignore `jku` from untrusted tokens. Configure JWKS URLs out-of-band per issuer; if you must accept `jku`, allowlist hostnames.
**Reference:** [RFC 8725 §3.5](https://datatracker.ietf.org/doc/html/rfc8725#section-3.5), [PortSwigger — jku injection](https://portswigger.net/web-security/jwt#jwt-header-parameter-injections).

### Bug 12 — missing `audience`
**Root cause:** Same IdP issues tokens for many resource servers. Without `aud`, a token minted for the billing API is accepted by yours. RFC 7519 §4.1.3 says the verifier MUST reject if `aud` doesn't list it.
**Fix:**
```ts
jwt.verify(token, publicKey, {
  algorithms: ["RS256"],
  issuer: "https://idp.example.com",
  audience: "https://orders.example.com",
});
```
**Reference:** [RFC 7519 §4.1.3](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.3).

### Bug 13 — missing `issuer`
**Root cause:** Multi-tenant IdPs and shared JWKS infrastructure mean any party with a key your verifier accepts can mint tokens. RFC 7519 §4.1.1 + RFC 8725 §3.4 require checking `iss`.
**Fix:**
```python
jwt.decode(token, public_key, algorithms=["RS256"],
           audience="api", issuer="https://idp.example.com")
```
**Reference:** [RFC 8725 §3.4](https://datatracker.ietf.org/doc/html/rfc8725#section-3.4).

### Bug 14 — `JWT.decode` instead of `JWT.require().verify()`
**Root cause:** `com.auth0.jwt.JWT.decode` only parses; it doesn't check signature, `exp`, or anything. The Javadoc is explicit.
**Fix:**
```java
Algorithm alg = Algorithm.RSA256((RSAPublicKey) publicKey, null);
DecodedJWT decoded = JWT.require(alg)
    .withIssuer("https://idp.example.com")
    .withAudience("api")
    .acceptLeeway(30) // seconds
    .build()
    .verify(token);
```
**Reference:** [java-jwt JavaDoc — `JWT.decode`](https://javadoc.io/doc/com.auth0/java-jwt/latest/com/auth0/jwt/JWT.html#decode(java.lang.String)).

### Bug 15 — unbounded leeway
**Root cause:** `clockTolerance` is in **seconds**. 86 400 = one extra day past `exp`. RFC 8725 §3.10 recommends "no more than a few minutes" of skew.
**Fix:**
```ts
jwt.verify(token, publicKey, { algorithms: ["RS256"], clockTolerance: 30 });
```
**Reference:** [RFC 8725 §3.10](https://datatracker.ietf.org/doc/html/rfc8725#section-3.10).

### Bug 16 — `nbf` ignored
**Root cause:** `nbf` (RFC 7519 §4.1.5) is the lower bound; disabling it lets future-dated tokens be replayed early. Pair with poor clock hygiene and you have a long replay window.
**Fix:** Default options enforce `nbf`. Don't disable it.
```python
jwt.decode(token, key, algorithms=["RS256"], audience="api", issuer="idp")
```
**Reference:** [RFC 7519 §4.1.5](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.5).

### Bug 17 — refresh with no rotation
**Root cause:** Stateless refresh = stolen refresh works until natural expiry. RFC 6749 §10.4 + RFC 6819 §5.2.2.3 require rotation with reuse detection.
**Fix:** Issue a new refresh token each call, mark the previous `jti` consumed in a server-side store, and revoke the entire refresh family on detected reuse.
**Reference:** [RFC 6819 §5.2.2.3](https://datatracker.ietf.org/doc/html/rfc6819#section-5.2.2.3), [Auth0 — Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation).

### Bug 18 — long-lived access, no revocation
**Root cause:** A pure JWT has no server-side state to revoke. 30-day lifetimes mean "log out everywhere" is a lie unless you also keep a denylist or use short access + rotated refresh.
**Fix:** Cap access tokens at minutes, push state to a refresh token + revocation list, or use opaque tokens with token introspection (RFC 7662) for sessions you must be able to kill.
**Reference:** [OWASP JWT Cheat Sheet — Token Sidejacking & Revocation](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html#token-explicit-revocation-by-the-user).

### Bug 19 — symmetric secret shared across services
**Root cause:** With HS256 the verifier IS the signer. Every service that can verify can mint. One leaked pod = forge tokens for every audience.
**Fix:** Use asymmetric (RS256/ES256/EdDSA) so only the IdP signs; resource servers hold only the public JWKS. If you must use HMAC, give each audience its own key and pin `aud`.
**Reference:** [RFC 8725 §3.5](https://datatracker.ietf.org/doc/html/rfc8725#section-3.5), [OWASP API Security Top 10 — API2:2023 Broken Authentication](https://owasp.org/API-Security/editions/2023/en/0xa2-broken-authentication/).

### Bug 20 — Bearer token logged
**Root cause:** `Authorization` headers in access logs end up in long-retention SIEMs. Anyone with log read perms gets every active session.
**Fix:** Redact in middleware before logging; configure your log shipper to drop the header by default.
```ts
const auth = req.headers.authorization ? "Bearer [REDACTED]" : "-";
```
**Reference:** [OWASP Logging Cheat Sheet — Data to exclude](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html#data-to-exclude).

### Bug 21 — ECDSA `(0, 0)` accepted on Java 15–18 (CVE-2022-21449)
**Root cause:** On affected JDKs (17.0.0–17.0.2, 18) the `SunEC` provider didn't reject signatures where `r = s = 0`. An attacker submits a token with the all-zero signature and `verify` returns `true`. Neil Madden's writeup ("Psychic Signatures") is the definitive reference.
**Fix:** Patch to JDK 17.0.3 / 18.0.1+. Independently, validate `r` and `s` are in `[1, n-1]` before calling `Signature.verify`, or prefer EdDSA where the malleability surface is smaller.
**Reference:** [NVD — CVE-2022-21449](https://nvd.nist.gov/vuln/detail/CVE-2022-21449), [Neil Madden — Psychic Signatures in Java](https://neilmadden.blog/2022/04/19/psychic-signatures-in-java/).

### Bug 22 — no `kid`, no rotation path
**Root cause:** Without a `kid` in the JWS header, the verifier can't tell which of several active keys signed a token, so two keys can't coexist during cutover. You're forced into a hard flip with mass logouts.
**Fix:** Always emit `kid`; verifier picks the matching key from a JWKS that holds *both* current and next keys during rotation.
```ts
jwt.sign(payload, privatePem, { algorithm: "RS256", keyid: "2026-q2" });
```
**Reference:** [RFC 7517 §4.5](https://datatracker.ietf.org/doc/html/rfc7517#section-4.5), [RFC 8725 §3.7](https://datatracker.ietf.org/doc/html/rfc8725#section-3.7).

### Bug 23 — JWE `alg: dir` with too-short key
**Root cause:** `alg: dir` means the supplied key IS the content-encryption key. `enc: A256GCM` requires 32 bytes; 16 bytes either errors or (in some libs historically) silently zero-pads. `jose` rejects it, but copy-pasted code with another lib may not.
**Fix:** Generate the right size or let `jose` do it:
```ts
const key = await generateSecret("A256GCM"); // 32 random bytes
```
**Reference:** [RFC 7518 §4.5](https://datatracker.ietf.org/doc/html/rfc7518#section-4.5) (Direct Encryption), [RFC 7518 §5.3](https://datatracker.ietf.org/doc/html/rfc7518#section-5.3) (A256GCM key size).

### Bug 24 — gateway verifies, downstream re-trusts
**Root cause:** Defense at the edge only. Anyone inside the cluster (compromised pod, port-forward, sidecar) can hit `orders-svc` directly with crafted `X-User`/`X-Roles`. OWASP API Security calls this "broken function-level authorization" + missing zero-trust.
**Fix:** Re-verify the JWT in every service (cheap with cached JWKS) **or** use mTLS-bound short-lived service tokens (SPIFFE/SPIRE) so a forged header alone isn't sufficient. Never trust headers stripped/added by an upstream you can bypass.
**Reference:** [OWASP API Security Top 10 — API5:2023 BFLA](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/), [NIST SP 800-207 Zero Trust Architecture](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-207.pdf).

### Bug 25 — refresh reuse not detected
**Root cause:** Without a `jti`-keyed family table, you can't tell legitimate-rotation from stolen-token-replay. Both calls succeed and now both parties have valid sessions.
**Fix:** Persist `{family_id, current_jti}` per refresh chain. On `/refresh`: if the presented `jti` is not the current one for its family → revoke the entire family and force re-login. RFC 6819 §5.2.2.3 ("Refresh Token Rotation with Reuse Detection") is the canonical pattern; Auth0 ships it as a feature.
**Reference:** [RFC 6819 §5.2.2.3](https://datatracker.ietf.org/doc/html/rfc6819#section-5.2.2.3), [Auth0 — Automatic Reuse Detection](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation#automatic-reuse-detection).

### Bug 26 — `verify(token, secret)` with no algorithm pin
**Root cause:** The historical bug (CVE-2015-9235) was the library defaulting to whatever `alg` the header said. Modern `jsonwebtoken` requires `algorithms` for keys whose type is ambiguous, but old call sites and forked code paths still slip through code review. The shape of the call — `verify(token, key)` with no options — is the smell.
**Fix:**
```ts
return jwt.verify(token, publicKey, {
  algorithms: ["RS256"],
  issuer: "https://idp.example.com",
  audience: "https://orders.example.com",
});
```
**Reference:** [NVD — CVE-2015-9235](https://nvd.nist.gov/vuln/detail/CVE-2015-9235), [Auth0 — Critical vulnerabilities in JWT libraries](https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/), [RFC 8725 §3.1](https://datatracker.ietf.org/doc/html/rfc8725#section-3.1).

---

## Related

- [JWT Design and Pitfalls](05-jwt-design-and-pitfalls.md) — concept doc this practice deck drills against
- [OAuth 2.0 and OIDC Deep Dive](04-oauth2-and-oidc-deep-dive.md) — where refresh tokens, PKCE, and `aud`/`iss` come from
- [Authentication vs Authorization](03-authentication-vs-authorization.md) — sessions vs token auth tradeoffs
- [TLS Handshake and Certificates](06-tls-handshake-and-certificates.md) — cert pinning context for JWKS fetch trust
- [OWASP Top 10](01-owasp-top-10.md) — A07 Identification and Authentication Failures, A02 Cryptographic Failures

## References

- **RFCs:** [RFC 7519 (JWT)](https://datatracker.ietf.org/doc/html/rfc7519), [RFC 7515 (JWS)](https://datatracker.ietf.org/doc/html/rfc7515), [RFC 7516 (JWE)](https://datatracker.ietf.org/doc/html/rfc7516), [RFC 7517 (JWK)](https://datatracker.ietf.org/doc/html/rfc7517), [RFC 7518 (JWA)](https://datatracker.ietf.org/doc/html/rfc7518), [RFC 8725 (JWT BCP)](https://datatracker.ietf.org/doc/html/rfc8725), [RFC 6749 (OAuth 2.0)](https://datatracker.ietf.org/doc/html/rfc6749), [RFC 6819 (OAuth Threat Model)](https://datatracker.ietf.org/doc/html/rfc6819), [RFC 6750 (Bearer Tokens)](https://datatracker.ietf.org/doc/html/rfc6750), [RFC 7662 (Token Introspection)](https://datatracker.ietf.org/doc/html/rfc7662)
- **OWASP:** [JWT for Java Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html), [API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x00-header/), [Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- **PortSwigger Web Security Academy:** [JWT attacks](https://portswigger.net/web-security/jwt), [Algorithm confusion](https://portswigger.net/web-security/jwt/algorithm-confusion-attacks)
- **CVEs (verified on NVD):**
  - [CVE-2015-9235](https://nvd.nist.gov/vuln/detail/CVE-2015-9235) — `jsonwebtoken < 4.2.2`, asymmetric → symmetric algorithm substitution
  - [CVE-2018-1000531](https://nvd.nist.gov/vuln/detail/CVE-2018-1000531) — Inversoft prime-jwt ≤ 1.3.0, `alg: none` accepted by `JWTDecoder.decode`
  - [CVE-2022-21449](https://nvd.nist.gov/vuln/detail/CVE-2022-21449) — Oracle Java SE 17.0.0–17.0.2 / 18, ECDSA `(r=0, s=0)` "psychic signature"
- **Disclosures and writeups:** [Tim McLean / Auth0 — Critical vulnerabilities in JSON Web Token libraries](https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/), [Neil Madden — Psychic Signatures in Java (CVE-2022-21449)](https://neilmadden.blog/2022/04/19/psychic-signatures-in-java/)
- **NIST:** [SP 800-207 Zero Trust Architecture](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-207.pdf)
