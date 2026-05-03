---
title: "Spring Security — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, spring-security, auth, jwt, oauth2, csrf, java]
---

# Spring Security — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `spring-security` `auth` `jwt` `oauth2` `csrf` `java`

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
This doc is an active-recall practice surface for the Spring Security 6 / Spring Boot 3 failure surface. It collects 22 broken `SecurityFilterChain`, `@PreAuthorize`, JWT/OAuth2, CSRF, password-encoding, and session-management snippets covering the bugs that production reviewers actually catch: matcher ordering, `ROLE_` prefix confusion, `@EnableMethodSecurity` omissions, JWT authority extraction, CORS-vs-filter-chain ordering, `HttpFirewall` rejecting encoded slashes, session-fixation defaults, `@Async` losing the `SecurityContext`, reactive `ReactiveSecurityContextHolder` propagation, plus a few real CVEs (CVE-2022-22978, CVE-2023-20862, CVE-2024-22257). Try the bug, peek at the one-line hint, then jump to §4 for the full root cause and reference. Treat this as a 30-minute warm-up before reviewing any PR that touches `HttpSecurity`, an `AuthenticationProvider`, or a JWT decoder.

## How to use this doc
- Try to spot the bug before opening the `<details>` hint.
- The hint is one line; the full root cause and fix are in §4 Solutions, keyed by bug number.
- Skip Easy only if you've already nailed those traps in code review.

## 1. Easy (warm-up traps)

### Bug 1 — `permitAll()` catch-all before `authenticated()`

```java
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/**").permitAll()
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated())
        .build();
}
```
<details><summary>Hint</summary>
Spring Security evaluates matcher rules top-down and short-circuits on the first match.
</details>

### Bug 2 — `hasRole("ROLE_ADMIN")` with the prefix already there

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ROLE_ADMIN")
    .anyRequest().authenticated());
```
<details><summary>Hint</summary>
`hasRole` and `hasAuthority` differ in exactly one thing — and it's already in the string.
</details>

### Bug 3 — `@PreAuthorize` on a class with no `@EnableMethodSecurity`

```java
@SpringBootApplication
public class App { public static void main(String[] a) { SpringApplication.run(App.class, a); } }

@Service
public class ReportService {
    @PreAuthorize("hasRole('ADMIN')")
    public Report generate(Long id) { /* ... */ }
}
```
<details><summary>Hint</summary>
Method security is opt-in — and the annotation moved in Spring Security 6.
</details>

### Bug 4 — `@PreAuthorize` on a private method

```java
@Service
public class AdminService {
    public void deleteTenant(Long id) {
        doDelete(id);
    }
    @PreAuthorize("hasRole('ADMIN')")
    private void doDelete(Long id) { /* drops tables */ }
}
```
<details><summary>Hint</summary>
Same proxy-visibility rule as `@Transactional`.
</details>

### Bug 5 — Plain-text password compare in a custom `UserDetailsService`

```java
@Component
public class DbAuthProvider implements AuthenticationProvider {
    @Override public Authentication authenticate(Authentication a) {
        User u = userRepo.findByEmail(a.getName()).orElseThrow();
        if (!u.getPassword().equals(a.getCredentials().toString())) {
            throw new BadCredentialsException("bad creds");
        }
        return new UsernamePasswordAuthenticationToken(u, null, u.getAuthorities());
    }
    @Override public boolean supports(Class<?> t) { return UsernamePasswordAuthenticationToken.class.isAssignableFrom(t); }
}
```
<details><summary>Hint</summary>
Two problems: comparing plaintext, and timing.
</details>

### Bug 6 — BCrypt strength `4` shipped to production

```java
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(4); // "fast for tests"
}
```
<details><summary>Hint</summary>
Strength is the work-factor exponent — 4 means 16 iterations.
</details>

### Bug 7 — CSRF disabled globally on a stateful app

```java
@Bean
SecurityFilterChain web(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.disable()) // "easier for our React app"
        .formLogin(Customizer.withDefaults())
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
```
<details><summary>Hint</summary>
Form login + session cookies + disabled CSRF = the textbook attack.
</details>

## 2. Subtle (review-passers)

### Bug 8 — `JwtAuthenticationConverter` not extracting authorities

```java
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
        .build();
}
// JWT contains: { "sub": "alice", "roles": ["ADMIN", "AUDITOR"] }
// /admin/** returns 403 for alice. Why?
```
<details><summary>Hint</summary>
Spring Security's default converter only reads one specific claim — and it isn't `roles`.
</details>

### Bug 9 — `OAuth2ResourceServer` with no issuer validation

```java
@Bean
JwtDecoder jwtDecoder() {
    // jwks-uri configured, signature checked, but...
    return NimbusJwtDecoder.withJwkSetUri("https://idp.example.com/.well-known/jwks.json")
        .build();
}
```
<details><summary>Hint</summary>
A signed token from a *different* tenant of the same IdP would still verify. What claim isn't being checked?
</details>

### Bug 10 — CORS configured but Spring Security blocks the preflight

```java
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .build();
}

@Bean
CorsFilter corsFilter() { /* configured with allowed origins, methods, headers */ }
```
<details><summary>Hint</summary>
The browser sends `OPTIONS` with no `Authorization` header — and Spring Security's filter chain runs first.
</details>

### Bug 11 — `RegexRequestMatcher` with a dot pattern

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers(RegexRequestMatcher.regexMatcher("/api/admin/.*")).hasRole("ADMIN")
    .anyRequest().permitAll());
// running on Spring Security 5.5.6
```
<details><summary>Hint</summary>
There is a real CVE for this exact configuration on certain servlet containers.
</details>

### Bug 12 — Concurrent session control "enforced" on a stateless JWT API

```java
http
    .sessionManagement(s -> s
        .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        .maximumSessions(1).maxSessionsPreventsLogin(true))
    .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()));
```
<details><summary>Hint</summary>
What state does `maximumSessions` track, and where?
</details>

### Bug 13 — `formLogin()` and `httpBasic()` both enabled, returning HTML to the API

```java
@Bean
SecurityFilterChain all(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults())
        .httpBasic(Customizer.withDefaults())
        .build();
}
// curl -u alice:wrong /api/users → 302 redirect to /login (HTML)
```
<details><summary>Hint</summary>
Two `AuthenticationEntryPoint`s registered — which one wins on a 401?
</details>

### Bug 14 — `@Async` losing the `SecurityContext`

```java
@Service
public class ExportService {
    @Async
    public CompletableFuture<Path> exportForCurrentUser() {
        String user = SecurityContextHolder.getContext().getAuthentication().getName(); // NPE
        return CompletableFuture.completedFuture(write(user));
    }
}
```
<details><summary>Hint</summary>
`SecurityContextHolder` is thread-local by default.
</details>

### Bug 15 — Reactive: `ReactiveSecurityContextHolder` lost across `subscribeOn`

```java
@GetMapping("/me")
public Mono<String> me() {
    return Mono.fromCallable(() -> "compute")
        .subscribeOn(Schedulers.boundedElastic())
        .flatMap(x -> ReactiveSecurityContextHolder.getContext()
            .map(c -> c.getAuthentication().getName())); // empty in some setups
}
```
<details><summary>Hint</summary>
Reactor context propagation depends on operator order — and this isn't WebFlux's thread-local.
</details>

### Bug 16 — Custom servlet filter added *after* `SecurityFilterChain` but reading the auth

```java
@Component
public class TenantHeaderFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain) {
        String name = SecurityContextHolder.getContext().getAuthentication().getName(); // NPE on permitAll endpoints
        MDC.put("tenant", resolveTenant(name));
        chain.doFilter(req, res);
    }
}
// registered as a plain @Component with no @Order
```
<details><summary>Hint</summary>
Where in the filter chain does this register, and what runs first for `permitAll` endpoints?
</details>

### Bug 17 — `DelegatingPasswordEncoder` upgrade flow not wired

```java
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(); // legacy users still have raw SHA-1 hashes in DB
}
```
<details><summary>Hint</summary>
You added BCrypt last year but old hashes are now unverifiable — and silently fail login.
</details>

## 3. Senior trap (production-only failures)

### Bug 18 — Logout left vulnerable on Spring Security 6.0.0–6.0.2 (CVE-2023-20862)

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
  <version>3.0.4</version> <!-- pulls Spring Security 6.0.2 -->
</dependency>
```
```java
http.logout(l -> l.logoutSuccessUrl("/"));
// Users observe: after POST /logout, the session id is rotated but the
// previous Authentication can still be read on the *next* request.
```
<details><summary>Hint</summary>
There is a real CVE for the serialized `SecurityContext` not being cleared on logout in this version range.
</details>

### Bug 19 — `HttpFirewall` rejecting `%2F` after a path-decoding refactor

```java
@Bean
HttpFirewall allowEncodedSlash() {
    StrictHttpFirewall fw = new StrictHttpFirewall();
    fw.setAllowUrlEncodedSlash(true); // "we have artifact paths with /"
    return fw;
}

http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/artifacts/**").authenticated()
    .anyRequest().authenticated());
// /artifacts/group%2Fartifact/1.0/file.jar — what does the matcher see?
```
<details><summary>Hint</summary>
`StrictHttpFirewall` is doing exactly what its name says, and even when relaxed, the *matcher* sees decoded paths differently from the request.
</details>

### Bug 20 — Direct `AuthenticatedVoter#vote` call (CVE-2024-22257)

```java
// custom voter chain in a legacy app
AuthenticatedVoter voter = new AuthenticatedVoter();
int decision = voter.vote(null, invocation, attributes); // null auth on a permit-all path
if (decision == AccessDecisionVoter.ACCESS_GRANTED) { /* allow */ }
// running Spring Security 6.1.5
```
<details><summary>Hint</summary>
Spring Security 6.2.3 has a published advisory for exactly this call shape.
</details>

### Bug 21 — JWT `kid` taken straight from token header against a multi-key JWKS

```java
@Bean
JwtDecoder jwtDecoder() {
    return NimbusJwtDecoder.withJwkSetUri(jwksUri)
        .jwtProcessorCustomizer(p -> p.setJWSKeySelector((header, ctx) -> {
            String kid = header.getKeyID(); // attacker-controlled
            return jwks.getKeys().stream()
                .filter(k -> k.getKeyID().equals(kid))
                .map(JWK::toRSAKey)
                .map(k -> { try { return k.toRSAPublicKey(); } catch (Exception e) { throw new RuntimeException(e); } })
                .toList();
        }))
        .build();
}
```
<details><summary>Hint</summary>
Trusting the `kid` is fine when the JWKS is your IdP's, but the algorithm header is a different story — and the `alg` isn't pinned anywhere here.
</details>

### Bug 22 — `RememberMe` with the default key

```java
http.rememberMe(rm -> rm.tokenValiditySeconds(60 * 60 * 24 * 14));
// no .key("...") supplied
```
<details><summary>Hint</summary>
The default key is generated at startup — what happens across a rolling restart of three replicas?
</details>

### Bug 23 — Session fixation default vs explicit `migrateSession` after a custom auth filter

```java
@Bean
SecurityFilterChain web(HttpSecurity http) throws Exception {
    return http
        .addFilterBefore(new SamlAssertionFilter(), UsernamePasswordAuthenticationFilter.class)
        .sessionManagement(s -> s.sessionFixation(Customizer.withDefaults()))
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
// SamlAssertionFilter calls SecurityContextHolder.getContext().setAuthentication(token) directly
```
<details><summary>Hint</summary>
The session-fixation strategy is wired into the standard authentication filters — your custom one isn't on that pathway.
</details>

## 4. Solutions

### Bug 1 — `permitAll()` catch-all before `authenticated()`
**Root cause:** `authorizeHttpRequests` evaluates rules in declaration order and stops at the first match. `requestMatchers("/**").permitAll()` matches every request, including `/admin/**`, before the admin rule is ever consulted. This is a classic security bypass.
**Fix:**
```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/", "/health", "/static/**").permitAll()
    .anyRequest().authenticated());
```
Order specific matchers from most-restrictive to least, and end with `anyRequest()`.
**Reference:** [Spring Security Reference — Authorize HttpServletRequests](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html)

### Bug 2 — `hasRole("ROLE_ADMIN")` with the prefix already there
**Root cause:** `hasRole(X)` is implemented as `hasAuthority("ROLE_" + X)`. Passing `"ROLE_ADMIN"` produces a check for `"ROLE_ROLE_ADMIN"`, which never matches. The endpoint becomes effectively closed to everyone, or — combined with a fallthrough — open.
**Fix:** Pick one form and stick to it:
```java
.requestMatchers("/admin/**").hasRole("ADMIN")            // expects authority ROLE_ADMIN
// or
.requestMatchers("/admin/**").hasAuthority("ROLE_ADMIN")  // explicit
```
**Reference:** [Spring Security Reference — Authorities (`hasRole` vs `hasAuthority`)](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html#_request_matchers)

### Bug 3 — `@PreAuthorize` on a class with no `@EnableMethodSecurity`
**Root cause:** Spring Security's method security is opt-in. Without `@EnableMethodSecurity` on a `@Configuration` class, `@PreAuthorize` is a no-op annotation. Spring Security 6 also dropped the older `@EnableGlobalMethodSecurity(prePostEnabled = true)` in favor of the new annotation.
**Fix:**
```java
@Configuration
@EnableMethodSecurity // prePostEnabled=true by default in SS6
public class MethodSecurityConfig { }
```
**Reference:** [Spring Security Reference — Method Security (`@EnableMethodSecurity`)](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html)

### Bug 4 — `@PreAuthorize` on a private method
**Root cause:** Spring AOP advice is applied via a proxy. Private methods are dispatched directly on the target instance — the proxy never sees the call, so `@PreAuthorize` is silently ignored. Same failure mode as `@Transactional` on a private method.
**Fix:** Make the method public (or move it to another bean and call through the proxy), or switch to AspectJ load-time weaving with `@EnableMethodSecurity(mode = AdviceMode.ASPECTJ)`.
**Reference:** [Spring Security Reference — Method Security (proxying limitations)](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html)

### Bug 5 — Plain-text password compare in a custom `UserDetailsService`
**Root cause:** Two failures: storing/comparing plaintext passwords (a credential-storage anti-pattern), and using `String.equals` which is not constant-time, leaking timing information. Spring Security ships a `PasswordEncoder` interface specifically to avoid this.
**Fix:**
```java
@Bean PasswordEncoder pwd() { return PasswordEncoderFactories.createDelegatingPasswordEncoder(); }
// in the provider:
if (!passwordEncoder.matches(rawPassword, u.getPassword())) {
    throw new BadCredentialsException("bad creds");
}
```
Hash with a salted, slow KDF (BCrypt, Argon2, scrypt). Never store or compare plaintext.
**Reference:** [OWASP — Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html); [Spring Security Reference — Password Storage](https://docs.spring.io/spring-security/reference/features/authentication/password-storage.html)

### Bug 6 — BCrypt strength `4` shipped to production
**Root cause:** `BCryptPasswordEncoder(4)` runs `2^4 = 16` rounds, completing in microseconds. An attacker with a leaked hash table can crack passwords at GPU rates. Spring's default strength is `10`; OWASP currently recommends `bcrypt cost ≥ 10` (and Argon2id where available). Lower strengths are appropriate only in test fixtures.
**Fix:**
```java
@Bean PasswordEncoder pwd() { return new BCryptPasswordEncoder(12); }
```
Profile-segregate test config so the weak encoder never reaches `prod`.
**Reference:** [OWASP — Password Storage Cheat Sheet (BCrypt cost)](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html#bcrypt); [Spring Security Reference — BCryptPasswordEncoder](https://docs.spring.io/spring-security/reference/features/authentication/password-storage.html#authentication-password-storage-bcrypt)

### Bug 7 — CSRF disabled globally on a stateful app
**Root cause:** CSRF protection should be disabled only for *stateless* APIs that authenticate via tokens in a header (which browsers won't auto-attach cross-origin). With session cookies + form login, the browser sends the cookie on any cross-origin form POST, and a malicious site can submit `/transfer?to=attacker`. Disabling CSRF here re-opens the textbook attack.
**Fix:**
```java
http
    // for cookie-session UI:
    .csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
    // *separate* SecurityFilterChain for /api/** with stateless JWT may disable CSRF
    ;
```
Disable CSRF only on a dedicated stateless filter chain matching `/api/**`.
**Reference:** [Spring Security Reference — Cross Site Request Forgery (CSRF)](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html); [OWASP — CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)

### Bug 8 — `JwtAuthenticationConverter` not extracting authorities
**Root cause:** The default `JwtGrantedAuthoritiesConverter` reads from the `scope` / `scp` claim and prefixes authorities with `SCOPE_`. Custom IdPs that put roles under `roles` (or `realm_access.roles` for Keycloak) produce a JWT with no Spring authorities, so `hasRole("ADMIN")` always fails.
**Fix:**
```java
@Bean
JwtAuthenticationConverter jwtAuthConverter() {
    JwtGrantedAuthoritiesConverter g = new JwtGrantedAuthoritiesConverter();
    g.setAuthoritiesClaimName("roles");
    g.setAuthorityPrefix("ROLE_");
    JwtAuthenticationConverter c = new JwtAuthenticationConverter();
    c.setJwtGrantedAuthoritiesConverter(g);
    return c;
}
http.oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(jwtAuthConverter())));
```
**Reference:** [Spring Security Reference — OAuth 2.0 Resource Server JWT (Authorization Extraction)](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html#oauth2resourceserver-jwt-authorization-extraction)

### Bug 9 — `OAuth2ResourceServer` with no issuer validation
**Root cause:** `withJwkSetUri` only verifies the signature against the JWKS. It does *not* validate `iss`, `aud`, or `exp` against your tenant. A token signed by the same IdP for a different audience or tenant would still authenticate. RFC 7519 §4.1.1 requires that consumers validate `iss` when present.
**Fix:**
```java
@Bean
JwtDecoder jwtDecoder() {
    NimbusJwtDecoder d = NimbusJwtDecoder.fromIssuerLocation("https://idp.example.com/realms/prod");
    OAuth2TokenValidator<Jwt> validators = new DelegatingOAuth2TokenValidator<>(
        JwtValidators.createDefaultWithIssuer("https://idp.example.com/realms/prod"),
        new JwtAudienceValidator("my-api"));
    d.setJwtValidator(validators);
    return d;
}
```
Use `fromIssuerLocation` (which calls `/.well-known/openid-configuration`) and add an audience validator.
**Reference:** [Spring Security Reference — OAuth2 Resource Server JWT (Issuer & Audience Validation)](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html#oauth2resourceserver-jwt-validation); [RFC 7519 §4.1.1 (`iss`)](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.1)

### Bug 10 — CORS configured but Spring Security blocks the preflight
**Root cause:** Browser CORS preflight is `OPTIONS` with no auth header. Spring Security's filter chain runs before MVC, sees an unauthenticated request, and returns 401 — the preflight fails, the actual request never happens. The `CorsFilter` must be invoked *before* the security chain, which happens automatically only when you wire CORS via `HttpSecurity`.
**Fix:**
```java
http
    .cors(Customizer.withDefaults())                // engages the registered CorsConfigurationSource
    .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());

@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration c = new CorsConfiguration();
    c.setAllowedOrigins(List.of("https://app.example.com"));
    c.setAllowedMethods(List.of("GET","POST","PUT","DELETE","OPTIONS"));
    c.setAllowedHeaders(List.of("Authorization","Content-Type"));
    UrlBasedCorsConfigurationSource s = new UrlBasedCorsConfigurationSource();
    s.registerCorsConfiguration("/**", c);
    return s;
}
```
**Reference:** [Spring Security Reference — CORS](https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html)

### Bug 11 — `RegexRequestMatcher` with a dot pattern (CVE-2022-22978)
**Root cause:** **CVE-2022-22978** (CVSS 9.8). On Spring Security 5.5.x prior to 5.5.7 (and 5.4.x prior to 5.4.11, 5.6.x prior to 5.6.4), `RegexRequestMatcher` configured with a regex containing `.` could be bypassed on certain servlet containers because URL components containing newlines were not handled. The pattern `/api/admin/.*` could be evaded with crafted input.
**Fix:** Upgrade Spring Security to a patched version, *and* prefer `MvcRequestMatcher` / `AntPathRequestMatcher` for path-based authorization unless you have a real regex requirement:
```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/admin/**").hasRole("ADMIN"));
```
**Reference:** [NVD — CVE-2022-22978](https://nvd.nist.gov/vuln/detail/CVE-2022-22978); [Spring Security Advisory CVE-2022-22978](https://spring.io/security/cve-2022-22978)

### Bug 12 — Concurrent session control "enforced" on a stateless JWT API
**Root cause:** `maximumSessions` is enforced by the `SessionRegistry`, which tracks `HttpSession` ids. With `SessionCreationPolicy.STATELESS` no session is ever created, so the registry is empty and the limit is never enforced. The configuration silently does nothing.
**Fix:** Either make the chain stateful (and accept the cookie overhead), or enforce a token-revocation/active-token store outside Spring Security:
```java
// on the stateful side only:
http.sessionManagement(s -> s.maximumSessions(1).maxSessionsPreventsLogin(true));
```
For JWT APIs, track active refresh tokens in a database and reject re-use on issuance.
**Reference:** [Spring Security Reference — Session Management (Concurrent Sessions)](https://docs.spring.io/spring-security/reference/servlet/authentication/session-management.html#concurrent-sessions)

### Bug 13 — `formLogin()` and `httpBasic()` both enabled, returning HTML to the API
**Root cause:** When multiple authentication mechanisms are registered on one `SecurityFilterChain`, the *last-configured* `AuthenticationEntryPoint` wins on a 401. With `formLogin` after `httpBasic`, the entry point is `LoginUrlAuthenticationEntryPoint` and a `curl` against an API gets a 302 to `/login` (HTML). The fix is to split chains by `securityMatcher`.
**Fix:**
```java
@Bean @Order(1)
SecurityFilterChain api(HttpSecurity http) throws Exception {
    return http.securityMatcher("/api/**")
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults())
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .csrf(c -> c.disable())
        .build();
}
@Bean @Order(2)
SecurityFilterChain web(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults())
        .build();
}
```
**Reference:** [Spring Security Reference — Multiple SecurityFilterChain](https://docs.spring.io/spring-security/reference/servlet/configuration/java.html#jc-httpsecurity-multiple)

### Bug 14 — `@Async` losing the `SecurityContext`
**Root cause:** `SecurityContextHolder` defaults to `MODE_THREADLOCAL`. `@Async` dispatches to a different thread which has no security context, so `getAuthentication()` returns `null` and `.getName()` NPEs. The fix is to use a `DelegatingSecurityContextAsyncTaskExecutor` (or `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL` for simple cases — but that breaks with thread pools).
**Fix:**
```java
@Bean(name = "securityTaskExecutor")
TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor t = new ThreadPoolTaskExecutor();
    t.initialize();
    return new DelegatingSecurityContextAsyncTaskExecutor(t);
}
// then
@Async("securityTaskExecutor")
public CompletableFuture<Path> exportForCurrentUser() { /* ... */ }
```
**Reference:** [Spring Security Reference — `DelegatingSecurityContextExecutor`](https://docs.spring.io/spring-security/reference/servlet/integrations/concurrency.html)

### Bug 15 — Reactive: `ReactiveSecurityContextHolder` lost across `subscribeOn`
**Root cause:** `ReactiveSecurityContextHolder` reads from the *Reactor Context*, which propagates downstream through operators but is not implicitly attached to arbitrary `Mono.fromCallable` chains. `subscribeOn` doesn't strip the context, but if you build a `Mono` outside the request pipeline (e.g., in a `@Bean` or a static helper) it has no context to begin with.
**Fix:** Build the `Mono` from inside the request flow so the context is in scope, or explicitly read auth at the entry point and pass it down:
```java
@GetMapping("/me")
public Mono<String> me() {
    return ReactiveSecurityContextHolder.getContext()
        .map(c -> c.getAuthentication().getName())
        .flatMap(name -> Mono.fromCallable(() -> compute(name))
            .subscribeOn(Schedulers.boundedElastic()));
}
```
**Reference:** [Spring Security Reference — Reactive `ReactiveSecurityContextHolder`](https://docs.spring.io/spring-security/reference/reactive/authentication/reactive-authentication-manager.html)

### Bug 16 — Custom servlet filter added *after* `SecurityFilterChain` but reading the auth
**Root cause:** Spring Boot auto-registers any `@Component` that implements `Filter` into the main servlet filter chain. Without an `@Order`, ordering is unspecified — and Spring Security's `FilterChainProxy` is itself one of those filters. If your filter runs first, `SecurityContextHolder` is empty. If it runs on a `permitAll` endpoint, the context is also empty (no authentication ever happens).
**Fix:** Either register the filter *inside* the `SecurityFilterChain` so it sees the auth, or guard the read:
```java
http.addFilterAfter(new TenantHeaderFilter(), AuthorizationFilter.class);
// or, if it must stay a global filter:
@Bean FilterRegistrationBean<TenantHeaderFilter> reg(TenantHeaderFilter f) {
    FilterRegistrationBean<TenantHeaderFilter> r = new FilterRegistrationBean<>(f);
    r.setOrder(SecurityProperties.DEFAULT_FILTER_ORDER + 1);
    return r;
}
```
**Reference:** [Spring Security Reference — Adding a Custom Filter to the Filter Chain](https://docs.spring.io/spring-security/reference/servlet/configuration/java.html#adding-custom-filter)

### Bug 17 — `DelegatingPasswordEncoder` upgrade flow not wired
**Root cause:** `BCryptPasswordEncoder` only knows BCrypt. Hashes stored with another scheme (or with no `{id}` prefix) cannot be matched, so legacy users silently fail login. Spring Security ships `DelegatingPasswordEncoder` (built by `PasswordEncoderFactories.createDelegatingPasswordEncoder()`) which reads a `{bcrypt}…`, `{noop}…`, `{pbkdf2}…` prefix and dispatches to the right algorithm — and supports re-encoding on successful login.
**Fix:**
```java
@Bean PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```
On successful login, check `passwordEncoder.upgradeEncoding(stored)` and re-hash with the current default if true.
**Reference:** [Spring Security Reference — DelegatingPasswordEncoder](https://docs.spring.io/spring-security/reference/features/authentication/password-storage.html#authentication-password-storage-dpe)

### Bug 18 — Logout left vulnerable on Spring Security 6.0.0–6.0.2 (CVE-2023-20862)
**Root cause:** **CVE-2023-20862** (CVSS 6.3). In Spring Security 5.7.x prior to 5.7.8, 5.8.x prior to 5.8.3, and 6.0.x prior to 6.0.3, the `logout` support did not properly clean a *serialized* `SecurityContext`. After a logout, a subsequent request could still observe the previous authentication. Spring Boot 3.0.4 transitively pulls Spring Security 6.0.2, which is vulnerable.
**Fix:** Upgrade Spring Boot to 3.0.5+ (which pulls Spring Security 6.0.3+). After upgrade, verify by hitting a `permitAll` endpoint after `POST /logout` and confirming `SecurityContextHolder.getContext().getAuthentication()` is anonymous.
**Reference:** [NVD — CVE-2023-20862](https://nvd.nist.gov/vuln/detail/CVE-2023-20862); [Spring Security Advisory CVE-2023-20862](https://spring.io/security/cve-2023-20862)

### Bug 19 — `HttpFirewall` rejecting `%2F` after a path-decoding refactor
**Root cause:** `StrictHttpFirewall` rejects encoded slashes by default to prevent matcher-bypass attacks where `/admin%2F..` could match a `permitAll("/static/**")` matcher. Allowing them via `setAllowUrlEncodedSlash(true)` is dangerous because the `RequestMatcher` may decode differently from your downstream code, opening a path-confusion bypass. The Spring docs explicitly warn against relaxing these defaults.
**Fix:** Don't relax the firewall. Encode artifact identifiers differently (e.g., as a path parameter `/artifacts/{group}/{artifact}/{version}/{file}`) so encoded slashes aren't needed. If you must, audit every authorization matcher to confirm it interprets the decoded path the same way your app does.
**Reference:** [Spring Security Reference — `HttpFirewall` (StrictHttpFirewall)](https://docs.spring.io/spring-security/reference/servlet/exploits/firewall.html)

### Bug 20 — Direct `AuthenticatedVoter#vote` call (CVE-2024-22257)
**Root cause:** **CVE-2024-22257**. In Spring Security 5.7.x prior to 5.7.12, 5.8.x prior to 5.8.11, 6.0.x prior to 6.0.10, 6.1.x prior to 6.1.8, and 6.2.x prior to 6.2.3, calling `AuthenticatedVoter#vote` *directly* with a `null` `Authentication` could return `ACCESS_GRANTED` instead of throwing or denying. Most apps don't do this, but legacy custom voter chains can.
**Fix:** Upgrade Spring Security past the patched versions. Better: stop calling deprecated `AccessDecisionVoter` APIs entirely. The recommended Spring Security 6 model is `AuthorizationManager`, which is null-checked at the framework level and integrates with `@EnableMethodSecurity`.
**Reference:** [Spring Security Advisory CVE-2024-22257](https://spring.io/security/cve-2024-22257); [Spring Security Reference — Migrating from `AccessDecisionVoter` to `AuthorizationManager`](https://docs.spring.io/spring-security/reference/migration-7/authorization.html)

### Bug 21 — JWT `kid` taken straight from token header against a multi-key JWKS
**Root cause:** Trusting the `kid` to pick a JWKS entry is fine — JWKS is your trusted source. The trap is *not pinning the algorithm*. If your custom selector returns an RSA key but the token's `alg` header is `HS256`, some JWT libraries will use the public key as an HMAC secret and verify the attacker-signed token. Even when not exploitable, accepting `none` or unexpected algs is a long-running source of bypasses.
**Fix:** Use Spring Security's built-in `NimbusJwtDecoder.withJwkSetUri(...).jwsAlgorithm(SignatureAlgorithm.RS256).build()` (repeat `.jwsAlgorithm` per accepted alg). Reject everything else. Don't write a custom `JWSKeySelector` unless you also pin algs.
```java
NimbusJwtDecoder.withJwkSetUri(jwksUri)
    .jwsAlgorithm(SignatureAlgorithm.RS256)
    .build();
```
**Reference:** [Spring Security Reference — JWT Algorithm Selection](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html#oauth2resourceserver-jwt-decoder-algorithm); [RFC 7519 §6.1 (alg confusion)](https://datatracker.ietf.org/doc/html/rfc7519); [OWASP — JWT Cheat Sheet (alg confusion)](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)

### Bug 22 — `RememberMe` with the default key
**Root cause:** Without `.key("...")`, Spring Security generates a per-instance random key at startup. Across a horizontally-scaled app, *every replica has a different key*, so a remember-me cookie issued by replica A is rejected by B (random user logout). Worse, a single-replica restart silently invalidates every persisted remember-me cookie. And TokenBasedRememberMeServices uses the key as an HMAC secret — leaking it via heap dump compromises every cookie ever issued.
**Fix:** Set a stable, secret key from configuration:
```java
http.rememberMe(rm -> rm
    .key(env.getRequiredProperty("app.remember-me.key"))
    .tokenValiditySeconds(60 * 60 * 24 * 14));
```
Store the key in a secret manager, rotate via blue/green. For higher security, use `PersistentTokenBasedRememberMeServices` with a database-backed token store.
**Reference:** [Spring Security Reference — Remember-Me Authentication](https://docs.spring.io/spring-security/reference/servlet/authentication/rememberme.html)

### Bug 23 — Session fixation default vs explicit `migrateSession` after a custom auth filter
**Root cause:** Session-fixation protection (`SessionAuthenticationStrategy`) is invoked by Spring Security's standard authentication filters (e.g., `UsernamePasswordAuthenticationFilter`). A custom filter that calls `SecurityContextHolder.getContext().setAuthentication(token)` *directly* bypasses the strategy entirely, so the pre-authentication session id is never rotated. An attacker who set the victim's `JSESSIONID` before login keeps a valid post-auth session.
**Fix:** Invoke the strategy from the custom filter:
```java
@Autowired SessionAuthenticationStrategy sessionStrategy;

protected void doFilter(...) {
    Authentication auth = buildAuth(req);
    sessionStrategy.onAuthentication(auth, req, res);  // rotates session id
    SecurityContextHolder.getContext().setAuthentication(auth);
    new HttpSessionSecurityContextRepository().saveContext(
        SecurityContextHolder.getContext(), req, res);
    chain.doFilter(req, res);
}
```
Better: extend `AbstractAuthenticationProcessingFilter`, which wires the strategy automatically.
**Reference:** [Spring Security Reference — Session Fixation](https://docs.spring.io/spring-security/reference/servlet/authentication/session-management.html#ns-session-fixation); [OWASP — Session Management Cheat Sheet (fixation)](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#renew-the-session-id-after-any-privilege-level-change)

## Related
- [Security Filter Chain](security/security-filter-chain.md)
- [Authentication & Authorization](security/authentication-authorization.md)
- [OAuth2 & JWT](security/oauth2-jwt.md)
- [OIDC and Modern Auth](security/oidc-and-modern-auth.md)
- [Secrets Management](security/secrets-management.md)

## References
- Spring Security Reference (6.x): https://docs.spring.io/spring-security/reference/
- Spring Security Reference — Authorize HttpServletRequests: https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html
- Spring Security Reference — Method Security: https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html
- Spring Security Reference — OAuth 2.0 Resource Server JWT: https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html
- Spring Security Reference — CSRF: https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html
- Spring Security Reference — CORS: https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html
- Spring Security Reference — Password Storage: https://docs.spring.io/spring-security/reference/features/authentication/password-storage.html
- Spring Security Reference — Session Management: https://docs.spring.io/spring-security/reference/servlet/authentication/session-management.html
- Spring Security Reference — Remember-Me: https://docs.spring.io/spring-security/reference/servlet/authentication/rememberme.html
- Spring Security Reference — HttpFirewall: https://docs.spring.io/spring-security/reference/servlet/exploits/firewall.html
- Spring Security Reference — DelegatingSecurityContextExecutor: https://docs.spring.io/spring-security/reference/servlet/integrations/concurrency.html
- NVD — CVE-2022-22978 (RegexRequestMatcher bypass): https://nvd.nist.gov/vuln/detail/CVE-2022-22978
- Spring Security Advisory — CVE-2022-22978: https://spring.io/security/cve-2022-22978
- NVD — CVE-2023-20862 (logout SecurityContext not cleared): https://nvd.nist.gov/vuln/detail/CVE-2023-20862
- Spring Security Advisory — CVE-2023-20862: https://spring.io/security/cve-2023-20862
- Spring Security Advisory — CVE-2024-22257 (AuthenticatedVoter null auth): https://spring.io/security/cve-2024-22257
- OWASP — Password Storage Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- OWASP — CSRF Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- OWASP — JWT for Java Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html
- OWASP — Session Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- RFC 7519 — JSON Web Token: https://datatracker.ietf.org/doc/html/rfc7519
- RFC 6749 — OAuth 2.0 Authorization Framework: https://datatracker.ietf.org/doc/html/rfc6749
