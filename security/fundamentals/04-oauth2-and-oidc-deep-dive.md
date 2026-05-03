---
title: "OAuth 2.0 and OpenID Connect — Delegated Authorization and Federated Identity"
date: 2026-05-03
updated: 2026-05-03
tags: [security, oauth2, oidc, identity, federation, pkce]
---

# OAuth 2.0 and OpenID Connect — Delegated Authorization and Federated Identity

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `security` `oauth2` `oidc` `identity` `federation` `pkce`

---

## Table of Contents

- [Summary](#summary)
- [1. The Problem OAuth Solves](#1-the-problem-oauth-solves)
- [2. Roles and Vocabulary](#2-roles-and-vocabulary)
- [3. Grant Types in Practice](#3-grant-types-in-practice)
  - [3.1 Authorization Code + PKCE](#31-authorization-code--pkce)
  - [3.2 Client Credentials](#32-client-credentials)
  - [3.3 Refresh Token](#33-refresh-token)
  - [3.4 Device Authorization](#34-device-authorization)
  - [3.5 Deprecated: Implicit and ROPC](#35-deprecated-implicit-and-ropc)
- [4. PKCE — RFC 7636](#4-pkce--rfc-7636)
- [5. OpenID Connect](#5-openid-connect)
  - [5.1 ID Tokens vs Access Tokens](#51-id-tokens-vs-access-tokens)
  - [5.2 Discovery and JWKS](#52-discovery-and-jwks)
  - [5.3 Scopes and Claims](#53-scopes-and-claims)
- [6. Common Pitfalls](#6-common-pitfalls)
- [7. Spring Security Examples](#7-spring-security-examples)
- [8. Node Examples](#8-node-examples)
- [9. OAuth 2.1 — What's Changing](#9-oauth-21--whats-changing)
- [Related](#related)
- [References](#references)

---

## Summary

OAuth 2.0 (RFC 6749) is a **delegated authorization** protocol: it lets a *user* grant a *client application* the right to call a *resource server* on their behalf, without sharing their password with the client. OpenID Connect (OIDC) is an authentication layer on top of OAuth 2.0 that adds a standardized way to identify the user via an ID token. They are not interchangeable — OAuth alone does not log the user in, and OIDC without OAuth wouldn't have the access-token machinery to call APIs.

This document walks the modern grant types (authorization code with PKCE, client credentials, device code, refresh), explains why implicit and ROPC are deprecated, and surfaces the recurring pitfalls that produce real CVEs and breach reports: missing `state` parameter (CSRF), open redirect on the callback, mixed-up token audiences, refresh-token theft, and the confused-deputy problem at the resource server. The audience is a backend engineer who will integrate with Auth0, Okta, Keycloak, AWS Cognito, Google, Microsoft Entra ID, or GitHub — and who needs to spot the gap between "the example works" and "the implementation is secure".

---

## 1. The Problem OAuth Solves

Before OAuth (and during the brief, dark era of "OAuth 1.0 with three-legged signature dances"), the canonical pattern for "let app X read your Gmail contacts" was:

> Give app X your Gmail password.

This is the **password antipattern**. The app has full account access forever, can change your password, and you have no way to revoke just that app.

OAuth 2.0 fixes this by introducing an **authorization server** that issues short-lived **access tokens** to clients after the user explicitly consents. The client never sees the password; the user can revoke the token; the resource server only ever sees a token.

Use cases:

- **"Sign in with Google"** for a third-party app (this is OIDC on top of OAuth).
- **GitHub Actions calls AWS** — Actions presents an OIDC token, AWS exchanges it for short-lived credentials.
- **Mobile app calling its own API** — the app uses authorization code + PKCE, gets an access token, calls the API.
- **Service-to-service** — backend service uses client credentials grant.

---

## 2. Roles and Vocabulary

| Role | Definition | Example |
|------|------------|---------|
| **Resource Owner** | The user who owns the data | A Gmail user |
| **Client** | The app that wants access | A photo-printing service |
| **Authorization Server (AS)** | Issues tokens after authenticating the user and getting consent | Google's `accounts.google.com` |
| **Resource Server (RS)** | Hosts the protected API | Gmail API |
| **User Agent** | The browser the user uses | Chrome |

OAuth tokens come in two flavors and you must know which is which:

| Token | Purpose | Audience | Lifetime |
|-------|---------|----------|----------|
| **Access Token** | Bearer credential to call the resource server | The RS (`aud` claim) | Short — 5–60 min typical |
| **Refresh Token** | Used to get new access tokens without bothering the user | The AS only | Long — hours to months |
| **ID Token** (OIDC) | Proves *who the user is* to the client | The client (`aud` claim) | Short — used at login |

### 2.1 Public vs Confidential Clients

- **Confidential client.** Can keep a secret. Backend services. Authenticates to the AS with a client secret or signed JWT.
- **Public client.** Cannot keep a secret. SPAs and mobile apps. Must use PKCE to compensate.

This distinction drives which grant type you use.

---

## 3. Grant Types in Practice

### 3.1 Authorization Code + PKCE

The default grant for any user-facing flow. Used by SPAs, mobile apps, and traditional server-rendered web apps.

```
1. Client → User-Agent → AS:
   GET /authorize?
     response_type=code
     &client_id=abc
     &redirect_uri=https://app.example.com/callback
     &scope=openid profile email
     &state=<random>
     &code_challenge=<S256(verifier)>
     &code_challenge_method=S256
     &nonce=<random>          (OIDC)

2. AS authenticates the user, gets consent.

3. AS → User-Agent → Client:
   GET https://app.example.com/callback?code=AUTHCODE&state=<same random>

4. Client → AS (back-channel):
   POST /token
     grant_type=authorization_code
     &code=AUTHCODE
     &redirect_uri=https://app.example.com/callback
     &client_id=abc
     &code_verifier=<verifier>      (the original, AS hashes and compares)
     [client_secret if confidential]

5. AS → Client:
   { access_token, id_token, refresh_token, expires_in, token_type: "Bearer" }
```

Why two round trips (front-channel for the code, back-channel for the token)?

- The code is **short-lived** (≤10 min, often ≤30 s) and **single-use**, so even if intercepted via referrer leakage or browser history, it's nearly useless without the matching code verifier.
- The token exchange happens server-to-server, so tokens never touch the URL bar, browser history, or referer headers.

### 3.2 Client Credentials

Service-to-service. No user, no consent.

```
POST /token
  grant_type=client_credentials
  &client_id=svc-A
  &client_secret=...     (or signed JWT assertion — preferred)
  &scope=billing.read
```

Use cases:

- Backend job calls another internal API.
- CI pipeline calls a deployment API.
- Webhook receiver authenticates outbound calls.

The access token represents the **client's** authority, not a user's. The resource server should authorize based on the client's allowed scopes, not on a user identity.

Best practice: replace `client_secret` with a **JWT client assertion** (RFC 7523) signed by a private key. Rotating signing keys is easier than rotating shared secrets, and the secret never crosses the wire.

### 3.3 Refresh Token

```
POST /token
  grant_type=refresh_token
  &refresh_token=...
  &client_id=abc
```

The AS responds with a new access token (and optionally a new refresh token).

**Refresh-token rotation** (modern best practice): every refresh issues a new refresh token and invalidates the old one. If an attacker steals a refresh token and uses it, the legitimate client's next refresh fails — the AS detects reuse and revokes the entire token family. This is the standard defense for SPAs that store refresh tokens in browser storage.

### 3.4 Device Authorization

For devices without a browser (TVs, CLIs, IoT). Defined in RFC 8628.

```
1. Device → AS: POST /device_authorization (with client_id, scope)
2. AS → Device: { device_code, user_code, verification_uri, interval, expires_in }
3. Device shows: "Visit example.com/device and enter code ABCD-1234"
4. User opens browser on phone, completes auth.
5. Device polls /token with grant_type=urn:ietf:params:oauth:grant-type:device_code
6. AS eventually returns access_token.
```

The CLI flow for `gh auth login`, `aws sso login`, `kubectl oidc-login` all use this.

### 3.5 Deprecated: Implicit and ROPC

**Implicit grant** (`response_type=token`) — access token returned in URL fragment from the authorize endpoint. Created for SPAs before CORS made the back-channel feasible. **Don't use it.** Token leaks via browser history, referer headers, browser extensions. Authorization code + PKCE is the replacement.

**Resource Owner Password Credentials (ROPC)** (`grant_type=password`) — client collects username/password and sends them to the AS. The exact pattern OAuth was created to *avoid*. **Don't use it** except for first-party migration scenarios where the legacy app already has the password.

OAuth 2.1 (in progress) formally removes both.

---

## 4. PKCE — RFC 7636

**Proof Key for Code Exchange.** Originally for mobile apps, now mandatory for any public client and recommended for confidential clients too.

### 4.1 The Problem

Without PKCE, an attacker who intercepts the authorization code (by registering a malicious app with the same redirect URI on a mobile OS, or via a compromised browser extension) can exchange it for a token.

### 4.2 The Mechanism

1. Client generates a random `code_verifier` (43–128 chars, high entropy).
2. Computes `code_challenge = BASE64URL(SHA256(code_verifier))`.
3. Sends the **challenge** in the `/authorize` request.
4. Sends the **verifier** in the `/token` request.
5. AS recomputes `SHA256(verifier)` and compares to the stored challenge. If different, rejects.

```typescript
import { randomBytes, createHash } from 'node:crypto';

function pkce() {
  const verifier = randomBytes(32).toString('base64url');
  const challenge = createHash('sha256').update(verifier).digest('base64url');
  return { verifier, challenge };
}
```

PKCE binds the code to the specific client instance that started the flow. An attacker who steals the code can't redeem it without the verifier, which only the legitimate client knows.

`code_challenge_method=plain` exists for legacy. Always use `S256`.

---

## 5. OpenID Connect

OIDC (specs published by the OpenID Foundation, 2014) is a thin authentication layer on top of OAuth 2.0. The core addition: the **ID token**.

### 5.1 ID Tokens vs Access Tokens

| | ID Token | Access Token |
|--|----------|--------------|
| Format | JWT (always) | Anything (JWT, opaque, etc.) |
| Audience | The client (`aud = client_id`) | The resource server |
| Purpose | Tells the client *who logged in* | Bearer credential to call APIs |
| Validated by | The client | The resource server |
| Contains user identity claims | Yes — `sub`, `email`, `name`, ... | Sometimes (provider-specific) |
| Sent to APIs? | **No** — it's not for that | Yes |

**Critical rule.** The client validates the ID token to learn the user's identity. The resource server validates the access token to authorize API calls. Sending an ID token in `Authorization: Bearer ...` is a common bug; the resource server should reject any token whose `aud` doesn't match its identifier.

### 5.2 Discovery and JWKS

OIDC providers publish a **discovery document** at `https://issuer/.well-known/openid-configuration`:

```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "response_types_supported": ["code", "token", "id_token", "code token", ...],
  "id_token_signing_alg_values_supported": ["RS256"]
}
```

The `jwks_uri` returns a JSON Web Key Set — the public keys used to verify ID tokens (and JWT access tokens, when the AS issues them). Clients fetch and cache these. Each key has a `kid` (key ID) that matches the `kid` in the JWT header.

Key rotation: the AS publishes the new key in `jwks_uri` first, signs new tokens with it, then eventually retires old keys. Clients should refresh JWKS on `kid` miss, with a small jitter to avoid stampedes.

### 5.3 Scopes and Claims

**Scopes** are the strings the client requests authority for: `openid profile email read:invoices`.

**Claims** are the fields in the ID token / userinfo response that the AS returns based on those scopes. `openid` is mandatory for OIDC; `profile` returns standard profile claims (`name`, `picture`, `locale`, ...); `email` returns `email` and `email_verified`.

```json
{
  "iss": "https://accounts.google.com",
  "aud": "1234.apps.googleusercontent.com",
  "sub": "117654321987654321",
  "iat": 1714723200,
  "exp": 1714726800,
  "nonce": "abc123",
  "email": "alice@example.com",
  "email_verified": true,
  "name": "Alice Example"
}
```

The client must verify, at minimum:

- **Signature** against the JWKS for that issuer.
- `iss` matches the configured issuer string exactly.
- `aud` includes (or equals) the client's own `client_id`.
- `exp` is in the future, `iat`/`nbf` is sane (small clock skew tolerance).
- `nonce` matches the one the client included in `/authorize`.

---

## 6. Common Pitfalls

### 6.1 Missing `state` (CSRF on Login)

Without a `state` parameter that the client generates and verifies on callback, an attacker can initiate a flow with their own account, capture the resulting code, and trick a victim's browser into hitting the callback URL — logging the victim into the attacker's account. The attacker can then read what the victim does.

**Fix.** Generate a CSPRNG `state`, store it in the user's session (or as a signed cookie), and verify it on callback. Reject if mismatched.

### 6.2 Open Redirect on `redirect_uri`

Many ASes accept partial-match redirect URIs (`https://app.example.com/*`). Combined with an open redirect on the client (`/callback?next=//evil.com`), an attacker exfiltrates the code.

**Fix.** Register exact, full redirect URIs at the AS. No wildcards. No path manipulation. RFC 8252 (OAuth for native apps) and the Security Best Current Practice draft both insist.

### 6.3 Token Audience Confusion (Confused Deputy)

Service A trusts service B. An attacker obtains a token issued for service C (legitimately, because they have an account on C) and presents it to A. If A doesn't validate `aud`, it accepts.

**Fix.** Always validate `aud` exactly. Use **Token Exchange** (RFC 8693) when one service legitimately needs to call another with a different audience.

### 6.4 alg Confusion in JWT Validation

If the resource server is configured with the AS's RSA public key but accepts tokens with `alg=HS256`, an attacker can forge a token using the public key as the HMAC secret. Real CVE: **CVE-2018-0114** (Cisco node-jose library), and many similar bugs in jsonwebtoken-style libraries.

**Fix.** Pin the expected algorithm. Reject anything else, especially `alg=none`. See [JWT Design and Pitfalls](05-jwt-design-and-pitfalls.md).

### 6.5 Refresh Token Theft

SPA stores refresh token in localStorage; XSS exfiltrates it; attacker uses it for months.

**Fix.** Refresh-token rotation with reuse detection. Even better: use `BFF` (Backend-for-Frontend) pattern — the SPA holds an HttpOnly session cookie, and the BFF holds the OAuth tokens server-side.

### 6.6 PKCE Not Enforced

The AS supports PKCE but doesn't *require* it. A misconfigured client skips it; the security guarantee is gone.

**Fix.** Enforce PKCE on the AS for all public clients. OAuth 2.1 makes this mandatory.

### 6.7 Mixing Up Code, Token, and ID Token

Using an ID token to call an API; using an access token to identify a user; sending tokens in URLs. All seen in real bug reports.

**Fix.** Per role: clients consume ID tokens, resource servers consume access tokens, neither goes in URLs.

---

## 7. Spring Security Examples

### 7.1 OAuth2 Login (the relying-party / OIDC client)

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
        provider:
          google:
            issuer-uri: https://accounts.google.com
```

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a
                .requestMatchers("/", "/login**").permitAll()
                .anyRequest().authenticated())
            .oauth2Login(o -> o
                .userInfoEndpoint(u -> u.oidcUserService(this::oidcUserService)));
        return http.build();
    }

    private OidcUser oidcUserService(OidcUserRequest req) {
        OidcUser raw = new OidcUserService().loadUser(req);
        // Hook to load app-specific roles by sub
        return raw;
    }
}
```

Spring Security validates the ID token (signature via JWKS, `iss`, `aud`, `exp`, `nonce`) automatically.

### 7.2 Resource Server (validating access tokens)

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://accounts.google.com
          audiences: my-api
```

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a
                .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .requestMatchers("/api/**").authenticated())
            .oauth2ResourceServer(rs -> rs.jwt(Customizer.withDefaults()));
        return http.build();
    }
}
```

Spring Security caches JWKS, validates signature, claims, and converts scopes to `SCOPE_*` authorities for `@PreAuthorize`.

---

## 8. Node Examples

### 8.1 OIDC Client with `openid-client`

```typescript
import { Issuer, generators } from 'openid-client';

const issuer = await Issuer.discover('https://accounts.google.com');
const client = new issuer.Client({
  client_id: process.env.GOOGLE_CLIENT_ID!,
  client_secret: process.env.GOOGLE_CLIENT_SECRET!,
  redirect_uris: ['https://app.example.com/callback'],
  response_types: ['code'],
});

// Login route
app.get('/login', (req, res) => {
  const codeVerifier = generators.codeVerifier();
  const codeChallenge = generators.codeChallenge(codeVerifier);
  const state = generators.state();
  const nonce = generators.nonce();

  // Persist verifier, state, nonce server-side (signed cookie or session)
  req.session.pkce = { codeVerifier, state, nonce };

  const url = client.authorizationUrl({
    scope: 'openid profile email',
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
    state,
    nonce,
  });
  res.redirect(url);
});

// Callback
app.get('/callback', async (req, res) => {
  const { codeVerifier, state, nonce } = req.session.pkce;
  const params = client.callbackParams(req);
  const tokenSet = await client.callback(
    'https://app.example.com/callback',
    params,
    { code_verifier: codeVerifier, state, nonce },
  );
  // tokenSet has { access_token, id_token, refresh_token, expires_at }
  // tokenSet.claims() returns the validated ID token claims
  req.session.user = tokenSet.claims();
  res.redirect('/');
});
```

The library validates the ID token. It does *not* validate that you're using PKCE — you must request it.

### 8.2 Resource Server (JWT validation with `jose`)

```typescript
import { createRemoteJWKSet, jwtVerify } from 'jose';

const JWKS = createRemoteJWKSet(new URL('https://accounts.google.com/.well-known/jwks.json'));

async function authenticate(req: Request, res: Response, next: NextFunction) {
  const auth = req.headers.authorization;
  if (!auth?.startsWith('Bearer ')) return res.sendStatus(401);
  try {
    const { payload } = await jwtVerify(auth.slice(7), JWKS, {
      issuer: 'https://accounts.google.com',
      audience: 'my-api',
      algorithms: ['RS256'],   // pin the algorithm
    });
    (req as any).user = payload;
    next();
  } catch (e) {
    res.sendStatus(401);
  }
}
```

`jose` enforces the `algorithms` allowlist; if you forget it, the library still refuses `alg=none`.

---

## 9. OAuth 2.1 — What's Changing

OAuth 2.1 is a consolidation of OAuth 2.0 plus the Security Best Current Practice (BCP). Highlights:

- **PKCE mandatory** for all clients (public and confidential).
- **Implicit grant removed.**
- **ROPC removed.**
- **Exact-string redirect URI matching.** No partial matches.
- **Refresh tokens** must be sender-constrained (DPoP or mTLS) or rotated.
- **Bearer token usage in URL query string forbidden.** Always use `Authorization: Bearer ...` header.

If you're starting fresh in 2026, target OAuth 2.1 and the FAPI 2.0 profile if you're in financial services.

### 9.1 DPoP — Demonstrating Proof of Possession

DPoP (RFC 9449) binds a token to a key pair held by the client. Even if a token leaks, the attacker can't use it without the private key. Mitigates token theft on public clients.

```
Authorization: DPoP eyJ...           (the token)
DPoP: eyJhbGciOiJFUzI1NiIsImpr...     (a JWT signed by the client's key, scoped to method+URL+timestamp)
```

DPoP is increasingly supported by major IdPs and is the recommended path for SPAs that can't use mTLS.

---

## Related

- [JWT Design and Pitfalls](05-jwt-design-and-pitfalls.md) — JWT validation rules used throughout this doc
- [Authentication vs Authorization](03-authentication-vs-authorization.md) — where OAuth fits in the broader picture
- [TLS Handshake and Certificates](06-tls-handshake-and-certificates.md) — mTLS sender-constrained tokens
- [OWASP Top 10 (2021)](01-owasp-top-10.md) — A07 covers many of these pitfalls
- [Spring Security — OAuth2 and JWT](../../java/security/oauth2-jwt.md)
- [Spring Security — OIDC and Modern Auth](../../java/security/oidc-and-modern-auth.md)
- [System Design — Single Sign-On](../../system-design/security/single-sign-on.md)

---

## References

- **RFC 6749** — The OAuth 2.0 Authorization Framework. https://www.rfc-editor.org/rfc/rfc6749
- **RFC 6750** — Bearer Token Usage. https://www.rfc-editor.org/rfc/rfc6750
- **RFC 7636** — Proof Key for Code Exchange (PKCE). https://www.rfc-editor.org/rfc/rfc7636
- **RFC 7523** — JWT Profile for OAuth 2.0 Client Authentication and Authorization Grants. https://www.rfc-editor.org/rfc/rfc7523
- **RFC 8252** — OAuth 2.0 for Native Apps. https://www.rfc-editor.org/rfc/rfc8252
- **RFC 8628** — OAuth 2.0 Device Authorization Grant. https://www.rfc-editor.org/rfc/rfc8628
- **RFC 8693** — OAuth 2.0 Token Exchange. https://www.rfc-editor.org/rfc/rfc8693
- **RFC 9449** — DPoP: Demonstrating Proof of Possession. https://www.rfc-editor.org/rfc/rfc9449
- **OAuth 2.0 Security Best Current Practice (draft)** — https://datatracker.ietf.org/doc/draft-ietf-oauth-security-topics/
- **OAuth 2.1 (draft)** — https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
- **OpenID Connect Core 1.0** — https://openid.net/specs/openid-connect-core-1_0.html
- **OpenID Connect Discovery 1.0** — https://openid.net/specs/openid-connect-discovery-1_0.html
- **Financial-grade API (FAPI) 2.0** — https://openid.net/specs/fapi-2_0-security-02.html
- **CVE-2018-0114** — alg confusion in node-jose. https://nvd.nist.gov/vuln/detail/CVE-2018-0114
- **Spring Security OAuth2 reference** — https://docs.spring.io/spring-security/reference/servlet/oauth2/index.html
- **node openid-client** — https://github.com/panva/node-openid-client
- **jose (node JWT/JWS/JWE)** — https://github.com/panva/jose
- **OWASP Authentication Cheat Sheet** — https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
