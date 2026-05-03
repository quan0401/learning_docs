---
title: "TLS Handshake and Certificates — Encryption in Transit, Done Right"
date: 2026-05-03
updated: 2026-05-03
tags: [security, tls, certificates, pki, x509, mtls, acme]
---

# TLS Handshake and Certificates — Encryption in Transit, Done Right

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `security` `tls` `certificates` `pki` `x509` `mtls` `acme`

---

## Table of Contents

- [Summary](#summary)
- [1. What TLS Provides](#1-what-tls-provides)
- [2. TLS 1.2 Handshake](#2-tls-12-handshake)
- [3. TLS 1.3 Handshake](#3-tls-13-handshake)
- [4. Cipher Suites and Key Exchange](#4-cipher-suites-and-key-exchange)
- [5. X.509 Certificates and the Chain](#5-x509-certificates-and-the-chain)
- [6. SNI and ALPN](#6-sni-and-alpn)
- [7. Revocation — CRL, OCSP, OCSP Stapling, Must-Staple](#7-revocation--crl-ocsp-ocsp-stapling-must-staple)
- [8. Certificate Transparency](#8-certificate-transparency)
- [9. ACME and Let's Encrypt](#9-acme-and-lets-encrypt)
- [10. Mutual TLS (mTLS)](#10-mutual-tls-mtls)
- [11. Common Failure Modes](#11-common-failure-modes)
- [12. Backend Configuration Cheat Sheet](#12-backend-configuration-cheat-sheet)
- [Related](#related)
- [References](#references)

---

## Summary

TLS is the protocol that turns a TCP connection into a confidential, integrity-protected, authenticated channel. Every HTTPS request, every internal gRPC call worth its salt, every database connection from your app to RDS — TLS underneath. This document covers what TLS actually provides, how the 1.2 and 1.3 handshakes differ (1.3 is faster, simpler, and dropped a generation of footguns), the cipher-suite vocabulary you need to read a config file, the X.509 trust chain and SNI, certificate revocation including the OCSP-stapling mess, Certificate Transparency, ACME-based automation, and mutual TLS for service-to-service.

The audience is a backend engineer who already curls HTTPS endpoints daily but needs to debug `unable to verify the first certificate`, configure a reverse proxy with sane cipher suites, and understand why an mTLS rollout went sideways. The companion document in the networking path ([TLS & Certificates](../../networking/application-layer/tls-and-certificates.md)) covers the protocol mechanics in more depth; this one focuses on the security-engineering view.

---

## 1. What TLS Provides

Per RFC 8446 (TLS 1.3), TLS is designed to give the application three properties:

- **Confidentiality.** A passive attacker on the wire learns nothing about the cleartext.
- **Integrity.** An active attacker can't modify bytes without detection (the connection breaks).
- **Authentication.** At minimum, the server is authenticated to the client. With mTLS, the client is authenticated too.

What TLS does **not** provide:

- **Endpoint security.** A compromised server still hands data to attackers.
- **Hidden metadata.** Hostnames in SNI are visible to the network (until ECH ships broadly).
- **Application-layer authorization.** TLS auths the channel; your app still owes authn and authz on top.
- **Replay resistance for application data**, except for 0-RTT data in TLS 1.3 (which is replay-vulnerable by design — see [Section 3](#3-tls-13-handshake)).

---

## 2. TLS 1.2 Handshake

```
Client                                                   Server

ClientHello (versions, cipher suites, extensions, random) →
                                                       ← ServerHello (chosen version, suite, random)
                                                       ← Certificate (server's chain)
                                                       ← ServerKeyExchange (ECDHE params, signed)
                                                       ← ServerHelloDone
ClientKeyExchange (ECDHE pub) →
ChangeCipherSpec →
Finished (encrypted, MACs the handshake) →
                                                       ← ChangeCipherSpec
                                                       ← Finished

  Application data (encrypted)
```

Two round trips before any application data. Plenty of fields exposed in the clear (server certificate, SNI). All cipher suites and parameters negotiable.

The `RSA` key-exchange variant of 1.2 is **not forward-secret** — if the server's private key leaks later, all past recorded traffic decrypts. Modern deployments use `ECDHE` (ephemeral key agreement); the handshake signs the ephemeral key with the long-term certificate key.

---

## 3. TLS 1.3 Handshake

TLS 1.3 (RFC 8446, 2018) cuts the handshake to one round trip and removes a generation of dangerous options.

```
Client                                                   Server

ClientHello (key_share, signature_algorithms, ALPN, SNI) →
                                                       ← ServerHello (key_share, encrypted from here)
                                                       ← {EncryptedExtensions}
                                                       ← {Certificate}
                                                       ← {CertificateVerify}
                                                       ← {Finished}
{Finished} →
  Application data (encrypted)
```

What changed:

- **One RTT** (or zero with 0-RTT — see below).
- **Forward secrecy mandatory** — only (EC)DHE key agreement, no RSA key transport.
- **Mandatory AEAD** — only authenticated-encryption ciphers (AES-GCM, ChaCha20-Poly1305). No more MAC-then-encrypt, no CBC.
- **Signature scheme cleanup** — RSA-PSS, ECDSA, EdDSA. No MD5, no SHA-1.
- **Compression and renegotiation removed** — both were the source of high-profile attacks (CRIME, BEAST).
- **Encrypted server certificate** — middleboxes can no longer inspect.

### 3.1 0-RTT and Replay

TLS 1.3 supports sending application data in the *first* flight when reconnecting to a previously-known server (using a pre-shared key from the previous session). This saves an RTT on warm connections.

**Hazard.** 0-RTT data is **replayable** — there's no nonce yet. An attacker who captures a 0-RTT request can replay it. Use 0-RTT only for **idempotent** requests (GETs, retried-safe POSTs). HTTP/2 over TLS 1.3 with 0-RTT requires application-level idempotency reasoning. Most servers default 0-RTT off; turn it on deliberately.

### 3.2 Encrypted Client Hello (ECH)

The remaining metadata leak in TLS 1.3 is the SNI in `ClientHello` (the hostname). ECH (RFC 9460 + drafts) wraps the inner `ClientHello` in a public-key encryption to a frontend certificate, making the actual hostname invisible to network observers. Cloudflare, Firefox, and Chrome have shipped ECH; deployment is uneven but accelerating.

---

## 4. Cipher Suites and Key Exchange

A cipher suite for TLS 1.2 looks like:

```
ECDHE-RSA-AES256-GCM-SHA384
└──┬───┘└┬┘└────┬────┘└──┬──┘
key      auth   bulk     MAC/PRF
exchange         encryption
```

| Component | What it does |
|-----------|--------------|
| Key exchange (ECDHE / DHE / RSA) | Agree on a shared secret; ECDHE provides forward secrecy |
| Authentication (RSA / ECDSA / EdDSA) | Sign the key exchange so the client knows which server it's talking to |
| Bulk cipher (AES-GCM / ChaCha20-Poly1305 / AES-CBC) | Encrypt application data (post-handshake) |
| MAC / PRF (SHA256 / SHA384) | Integrity / handshake derivation |

In TLS 1.3, cipher suites only specify the AEAD and hash:

```
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_GCM_SHA256
```

Key exchange and signature are negotiated separately via the `key_share` and `signature_algorithms` extensions.

### 4.1 What to Allow

Mozilla's SSL Configuration Generator publishes maintained recommendations: https://ssl-config.mozilla.org

**Modern profile (2026)** — TLS 1.3 only:

```
ssl_protocols TLSv1.3;
ssl_ciphers <empty — TLS 1.3 picks from its built-in suites>;
ssl_prefer_server_ciphers off;
```

**Intermediate profile** — TLS 1.2 + 1.3, broad client support:

```
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:
            ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:
            ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers off;
```

Disable: SSLv2, SSLv3, TLS 1.0, TLS 1.1 (deprecated by RFC 8996), 3DES, RC4, anything CBC, anything with `MD5` or `SHA1` (HMAC), `EXPORT` and `NULL` suites.

---

## 5. X.509 Certificates and the Chain

A TLS certificate is an X.509 v3 certificate (RFC 5280) — a structured document signed by a Certificate Authority asserting:

- Subject (the entity being certified — usually a hostname).
- Subject's public key.
- Validity period.
- Extensions: Subject Alternative Names (SANs), Key Usage, Extended Key Usage (`serverAuth`, `clientAuth`), CRL/OCSP URLs, etc.
- Issuer (the CA).
- A signature by the issuer's private key.

### 5.1 The Chain

A **chain of trust** typically has three certs:

```
Root CA  →  Intermediate CA  →  Leaf (your server cert)
```

- **Root CA** — self-signed, trusted out-of-band (shipped with OS / browser trust stores). Its private key is kept offline and rarely used.
- **Intermediate CA** — signed by the root, used to sign leafs. CAs use intermediates so the root can stay offline.
- **Leaf** — your server's actual certificate.

The TLS handshake delivers **leaf + intermediates**, and the client builds a chain to a trusted root in its store.

### 5.2 The Most Common Bug: Missing Intermediate

Symptoms: works in browsers, fails in Node `fetch`, in Java `HttpClient`, in `curl`. Browsers cache and fetch missing intermediates; non-browser clients often don't.

```
unable to verify the first certificate    (Node)
PKIX path building failed                  (Java)
unable to get local issuer certificate     (curl/openssl)
```

Fix: configure your server to send the full chain (leaf + intermediates). Test with:

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts < /dev/null
```

If you see only one cert, you're missing intermediates.

### 5.3 SAN, CN, and Hostname Validation

Modern clients validate the hostname against the certificate's **Subject Alternative Name (SAN)** extension. Common Name (CN) in the Subject DN is **deprecated** for hostname validation (Chrome stopped honoring it in 2017).

A cert for `api.example.com` and `*.example.com` has SANs:

```
DNS:api.example.com, DNS:*.example.com
```

Wildcards match exactly one label (`a.example.com` ✓, `a.b.example.com` ✗). They never match the apex (`example.com` requires its own SAN).

### 5.4 Certificate Lifetimes

The industry has been ratcheting maximum lifetimes down:

- 2015: 39 months
- 2018: 825 days
- 2020: 398 days
- 2026 (CA/Browser Forum baseline): moving toward 47–90 days for public certs

Short lifetimes force automation, which is the only reliable defense against expired certs in production.

---

## 6. SNI and ALPN

### 6.1 Server Name Indication (RFC 6066)

Multiple HTTPS sites on the same IP need a way for the client to say "I'm asking for `a.example.com`" before the server picks a certificate. SNI is that mechanism: the client sends the hostname in the `ClientHello` extension. The server uses it to choose the right cert and (in HTTP/2+) the right vhost.

SNI is **plaintext** in TLS 1.2 and 1.3 (without ECH), so network observers see which hostname you're connecting to.

### 6.2 Application-Layer Protocol Negotiation (RFC 7301)

ALPN lets client and server agree on the protocol (HTTP/1.1, HTTP/2, gRPC) inside the TLS handshake. Required for HTTP/2 over TLS.

```
ClientHello extension: alpn = [h2, http/1.1]
ServerHello extension: alpn = h2
```

Without ALPN, you'd need a separate port for each protocol, or a wasteful upgrade dance.

---

## 7. Revocation — CRL, OCSP, OCSP Stapling, Must-Staple

When a certificate's private key is compromised before expiry, the CA needs to tell the world. Three mechanisms:

### 7.1 CRL (Certificate Revocation List)

The CA publishes a signed list of revoked certs at a well-known URL. Clients download it. Problems: large, infrequent updates, latency, privacy (every check tells the CA which sites you visit).

### 7.2 OCSP (Online Certificate Status Protocol)

Client asks the CA's OCSP responder about a specific cert. Smaller and fresher than CRLs. Same privacy issue. Also: OCSP responders go down, and clients face the question "fail open or fail closed?" Most browsers fail open — meaning OCSP doesn't actually defend against an attacker who can block traffic to the responder.

### 7.3 OCSP Stapling (RFC 6066)

The **server** queries OCSP, gets a recent signed response, and **staples** it to the TLS handshake. The client doesn't query the CA at all. Better for privacy, latency, and CA load.

Configuring on Nginx:

```nginx
ssl_stapling on;
ssl_stapling_verify on;
resolver 1.1.1.1 valid=300s;
```

### 7.4 OCSP Must-Staple (RFC 7633)

A cert can carry an extension saying "always require a stapled OCSP response." If the server doesn't staple, the client must reject. Closes the fail-open hole. Adoption is uneven; check that your hosting/CDN supports it.

### 7.5 The Reality

Major browser vendors (Google, Mozilla) have largely given up on OCSP and shifted to **proprietary revocation feeds** (CRLite, CRLSets) baked into the browser. For non-browser clients, OCSP stapling remains the practical answer, plus short-lived certs (90 days at Let's Encrypt) that limit the damage of a leak.

---

## 8. Certificate Transparency

After Symantec, Comodo, and DigiNotar incidents in the early 2010s (CAs issuing certs they shouldn't have, including for `*.google.com` to nation-state actors), the ecosystem mandated **Certificate Transparency** (RFC 6962, RFC 9162).

Every publicly trusted certificate must be submitted to multiple **CT logs** (append-only, signed Merkle trees). Browsers reject certs without sufficient SCTs (Signed Certificate Timestamps).

For defenders this means:

- Use **CT log monitoring** (`crt.sh`, Censys, Cert Spotter) to alert on certs issued for your domains by anyone.
- An attacker who tricks a CA into mis-issuing now leaves a public, immutable trail.

```bash
# What certs were issued for example.com lately?
curl 'https://crt.sh/?q=example.com&output=json' | jq '.[].name_value' | sort -u
```

For your own services: monitor CT for your domains. A cert you didn't expect is a sign of CA compromise or insider issue.

---

## 9. ACME and Let's Encrypt

Manual cert provisioning (paste a CSR into a CA web form, paste the cert back, restart Nginx) doesn't scale to short-lifetime certs. **ACME** (RFC 8555) automates the workflow: prove you control a domain, get a cert, renew automatically.

### 9.1 The Two Common Challenge Types

- **HTTP-01.** ACME server tells you "put a specific token at `http://yourdomain/.well-known/acme-challenge/<token>`". Server fetches it. Proves you control HTTP on the domain.
- **DNS-01.** ACME server tells you "publish a TXT record at `_acme-challenge.yourdomain` with this value". Server resolves it. Proves you control DNS. Required for wildcard certs.

### 9.2 Tooling

- **certbot** — original Let's Encrypt client, mature, pluggable.
- **acme.sh** — pure shell, vast DNS-provider support.
- **Caddy** — web server with ACME built in. Auto-provisions certs out of the box.
- **cert-manager** — Kubernetes operator. Manages cert lifecycle for ingresses.
- **Traefik** — reverse proxy with ACME integration.

### 9.3 Operational Notes

- **Stagger renewals.** If 100 services renew at midnight UTC on the 1st of the month, your CA will see a thundering herd and you may hit rate limits.
- **Monitor expiry.** Even with automation, alert on certs nearing expiry. Automation breaks.
- **Have a manual fallback.** You should be able to provision a cert by hand if your automation is the thing that's broken.
- **Don't rely on a single CA.** Have a backup. The 2020 Sectigo intermediate expiry took down many sites that hadn't tested fallbacks.

---

## 10. Mutual TLS (mTLS)

Standard TLS authenticates the **server** to the client. mTLS additionally authenticates the **client** to the server using a client certificate.

### 10.1 Where mTLS Fits

- **Service-to-service auth in microservices.** Service mesh (Istio, Linkerd) enables mTLS between every pod transparently.
- **B2B APIs.** Open Banking, healthcare interchanges, payment networks (FAPI, NACHA).
- **Device identity.** IoT devices with embedded certs.
- **Bastion access.** SSH-style "you can connect only with your enterprise-issued client cert".

### 10.2 mTLS Handshake

```
Client                                                   Server

ClientHello →
                                                       ← ServerHello, Certificate, ...
                                                       ← CertificateRequest (server asks for client cert)
                                                       ← ServerHelloDone
Certificate (client's chain) →
ClientKeyExchange →
CertificateVerify (signature proving private key) →
ChangeCipherSpec, Finished →
                                                       ← ChangeCipherSpec, Finished
```

The server's `CertificateRequest` includes which CAs it trusts for client certs. The client picks an appropriate cert (or fails). The client's `CertificateVerify` is a signature over the handshake transcript proving they hold the private key.

### 10.3 Client Cert Issuance

You need an internal PKI: a private CA that signs short-lived client certs. Options:

- **Smallstep CA** (`step-ca`) — open source, modern, ACME-capable.
- **HashiCorp Vault PKI secrets engine.**
- **AWS Private CA** (managed).
- **CFSSL** (Cloudflare).
- **Service mesh-issued.** Istio's Citadel issues SPIFFE-format certs to every pod.

### 10.4 Operational Hard Parts

- **Renewal.** Workloads need to refresh certs before expiry without downtime. SPIFFE/SPIRE solves this for K8s.
- **Revocation.** Same problem as public TLS, but you control the CA. Short lifetimes (hours) are the practical answer.
- **Debugging.** "TLS handshake failed" with no further info. Tools: `openssl s_client`, Wireshark pre-master-secret dump (`SSLKEYLOGFILE`).
- **Cert binding to identity.** Map cert subject → application identity; service mesh does this via SPIFFE IDs.

---

## 11. Common Failure Modes

### 11.1 Expired Certificate

The most common production TLS incident. Detection: monitoring with alerts at T-30, T-14, T-7 days. Root causes: manual provisioning, expired automation creds, DNS/HTTP challenge changed without updating the ACME client.

### 11.2 Hostname Mismatch

Cert is for `www.example.com`, client connects to `example.com`. Or cert is for an internal hostname, but the client uses the IP. Fix: SAN list covers all hostnames; clients use hostnames, not IPs.

### 11.3 Weak Cipher Suites

Old config still allowing TLS 1.0, 3DES, RC4. Test with:

```bash
sslyze example.com
testssl.sh example.com
nmap --script ssl-enum-ciphers -p 443 example.com
```

### 11.4 Self-Signed Cert in Production

Often a misconfiguration that someone "temporarily" worked around with `rejectUnauthorized: false` (Node) or `TrustAllStrategy` (Java). Now permanent. Removes all server authentication. Search your codebase regularly.

### 11.5 Trust Store Drift

You upgraded from Java 8 to Java 17 and a major CA isn't trusted, or vice versa. Or your container image's `ca-certificates` package is two years old and doesn't trust the cert your partner just deployed.

### 11.6 SNI Required But Not Sent

Java versions before 8u131 had quirks with SNI on `HttpsURLConnection`. Modern clients are fine but legacy code can still trip on this against multi-tenant servers.

### 11.7 Notable CVEs and Incidents

- **CVE-2014-0160 (Heartbleed)** — OpenSSL out-of-bounds read in heartbeat extension. Leaked memory including private keys. Triggered industry-wide patching and many cert reissues.
- **CVE-2014-3566 (POODLE)** — SSLv3 padding oracle. Drove SSLv3 deprecation.
- **DigiNotar 2011** — Dutch CA compromised; mis-issued `*.google.com` certs used to surveil Iranian Gmail users. Led to CT.
- **Symantec PKI 2017–2018** — multiple mis-issuance incidents; Google/Mozilla distrusted Symantec roots; DigiCert acquired.
- **Let's Encrypt CAA bug 2020** — LE revoked ~3M certs after discovering a CAA-checking bug. Massive scramble. Reminder: short-lived certs limit damage from CA bugs.

---

## 12. Backend Configuration Cheat Sheet

### 12.1 Node — HTTPS server

```typescript
import https from 'node:https';
import fs from 'node:fs';

const server = https.createServer({
  key: fs.readFileSync('/etc/letsencrypt/live/example.com/privkey.pem'),
  cert: fs.readFileSync('/etc/letsencrypt/live/example.com/fullchain.pem'),
  minVersion: 'TLSv1.2',
  ciphers: undefined,    // let Node pick TLS 1.3 + intermediate TLS 1.2 defaults
  honorCipherOrder: false,
}, app);

server.listen(443);
```

### 12.2 Node — outbound HTTPS (don't disable verification)

```typescript
// NEVER do this
fetch(url, { agent: new https.Agent({ rejectUnauthorized: false }) });

// Do this if you need a private CA
import https from 'node:https';
const agent = new https.Agent({
  ca: fs.readFileSync('/etc/internal-ca/root.pem'),
});
fetch(url, { agent });
```

### 12.3 Spring Boot — HTTPS

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    enabled-protocols: TLSv1.3,TLSv1.2
```

For mTLS resource server:

```yaml
server:
  ssl:
    client-auth: need
    trust-store: classpath:client-truststore.p12
    trust-store-password: ${TRUSTSTORE_PASSWORD}
```

### 12.4 Nginx — Modern TLS

```nginx
listen 443 ssl http2;
ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
ssl_session_timeout 1d;
ssl_session_cache shared:MozSSL:10m;
ssl_session_tickets off;

ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers off;

ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

add_header Strict-Transport-Security "max-age=63072000" always;
```

### 12.5 Curl Debugging

```bash
# Show full handshake including cert chain
openssl s_client -connect example.com:443 -servername example.com -showcerts

# Check cert expiry
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -dates

# Test with specific TLS version
curl --tlsv1.3 --tls-max 1.3 -v https://example.com/

# Test mTLS
curl --cert client.crt --key client.key https://api.example.com/
```

---

## Related

- [TLS & Certificates (Networking path)](../../networking/application-layer/tls-and-certificates.md) — the protocol-mechanics view; complements this engineering view
- [HTTP Evolution (Networking path)](../../networking/application-layer/http-evolution.md) — ALPN and HTTP/2 require TLS
- [Zero-Trust Networking](../../networking/advanced/zero-trust.md) — mTLS as identity in zero-trust meshes
- [OAuth 2.0 and OIDC](04-oauth2-and-oidc-deep-dive.md) — DPoP, mTLS sender-constrained tokens
- [JWT Design and Pitfalls](05-jwt-design-and-pitfalls.md) — alternative authn vehicle
- [System Design — Encryption at Rest and in Transit](../../system-design/security/encryption-at-rest-and-in-transit.md)

---

## References

- **RFC 8446** — The Transport Layer Security (TLS) Protocol Version 1.3. https://www.rfc-editor.org/rfc/rfc8446
- **RFC 5246** — The TLS Protocol Version 1.2. https://www.rfc-editor.org/rfc/rfc5246
- **RFC 8996** — Deprecating TLS 1.0 and TLS 1.1. https://www.rfc-editor.org/rfc/rfc8996
- **RFC 5280** — Internet X.509 Public Key Infrastructure Certificate and CRL Profile. https://www.rfc-editor.org/rfc/rfc5280
- **RFC 6066** — TLS Extensions: SNI, OCSP stapling. https://www.rfc-editor.org/rfc/rfc6066
- **RFC 7301** — TLS Application-Layer Protocol Negotiation (ALPN). https://www.rfc-editor.org/rfc/rfc7301
- **RFC 6962** — Certificate Transparency. https://www.rfc-editor.org/rfc/rfc6962
- **RFC 9162** — Certificate Transparency Version 2.0. https://www.rfc-editor.org/rfc/rfc9162
- **RFC 7633** — TLS Feature Extension (OCSP Must-Staple). https://www.rfc-editor.org/rfc/rfc7633
- **RFC 8555** — Automatic Certificate Management Environment (ACME). https://www.rfc-editor.org/rfc/rfc8555
- **RFC 9460** — Service Binding and Parameter Specification via the DNS (SVCB and HTTPS RRs); foundation for ECH. https://www.rfc-editor.org/rfc/rfc9460
- **CA/Browser Forum Baseline Requirements** — https://cabforum.org/baseline-requirements-documents/
- **Mozilla SSL Configuration Generator** — https://ssl-config.mozilla.org/
- **Let's Encrypt** — https://letsencrypt.org/
- **smallstep `step-ca`** — https://smallstep.com/docs/step-ca/
- **SPIFFE / SPIRE** — https://spiffe.io/
- **CVE-2014-0160 (Heartbleed)** — https://nvd.nist.gov/vuln/detail/CVE-2014-0160
- **CVE-2014-3566 (POODLE)** — https://nvd.nist.gov/vuln/detail/CVE-2014-3566
- **testssl.sh** — https://testssl.sh/
- **Bulletproof TLS — Hardening Guide** — https://www.feistyduck.com/library/openssl-cookbook/
