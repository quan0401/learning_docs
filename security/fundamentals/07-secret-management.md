---
title: "Secret Management — Stores, Rotation, and Leak Detection"
date: 2026-05-03
updated: 2026-05-03
tags: [security, secrets, vault, kms, kubernetes, rotation, twelve-factor]
---

# Secret Management — Stores, Rotation, and Leak Detection

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `security` `secrets` `vault` `kms` `kubernetes` `rotation` `twelve-factor`

---

## Table of Contents

- [Summary](#summary)
- [1. What Counts as a Secret](#1-what-counts-as-a-secret)
- [2. The Spectrum of Storage](#2-the-spectrum-of-storage)
- [3. Environment Variables — Twelve-Factor Reality Check](#3-environment-variables--twelve-factor-reality-check)
- [4. HashiCorp Vault](#4-hashicorp-vault)
- [5. Cloud-Managed Secret Stores](#5-cloud-managed-secret-stores)
- [6. Kubernetes Secrets](#6-kubernetes-secrets)
- [7. Secrets-in-Git: Sealed Secrets, SOPS, and age](#7-secrets-in-git-sealed-secrets-sops-and-age)
- [8. KMS-Backed Envelope Encryption](#8-kms-backed-envelope-encryption)
- [9. Dynamic Secrets and Short-Lived Credentials](#9-dynamic-secrets-and-short-lived-credentials)
- [10. Secret Rotation in Practice](#10-secret-rotation-in-practice)
- [11. Leak Detection and Response](#11-leak-detection-and-response)
- [12. Decision Guide](#12-decision-guide)
- [Related](#related)
- [References](#references)

---

## Summary

Secrets are the credentials your application uses to talk to *other systems* — database passwords, API keys, OAuth client secrets, signing keys, TLS private keys. Mishandling them is the most common path from "no breach" to "front-page breach." This document covers the realistic options: environment variables (and why they're a default, not an answer), centralized secret stores (Vault, AWS/GCP/Azure managed services), Kubernetes Secrets and the encryption-at-rest gap, secrets-in-Git approaches (SOPS, Sealed Secrets, age), KMS-backed envelope encryption as the universal building block, dynamic secrets that don't need rotation because they don't outlive the request, rotation patterns, and detection (gitleaks, trufflehog, vendor-side push protection).

The goal is to make the right choice the easy choice for new services. Wrong choices (committing `.env`, hardcoding tokens, storing JWT signing keys in `application.properties`) produce CVEs and bug-bounty payouts. Right choices fade into the background.

---

## 1. What Counts as a Secret

A secret is anything where leakage gives an adversary a meaningful capability. Not exhaustive but representative:

- **Database credentials.** Connection strings, usernames, passwords.
- **API keys and OAuth client secrets.** Stripe, Twilio, OpenAI, GitHub apps.
- **Signing/encryption keys.** JWT HMAC secrets, JWE private keys, application-level encryption keys.
- **TLS private keys.** Server certs, mTLS client certs.
- **SSH private keys.** Deploy keys, jump-box access.
- **Webhook signing secrets.** Stripe, GitHub, Slack — the secret that proves a webhook is real.
- **Cloud credentials.** AWS access keys, GCP service account keys (avoid; use workload identity).
- **Encryption-at-rest keys.** DB encryption keys, S3 SSE keys.

What is **not** a secret (and shouldn't be treated as one):

- Public IPs, hostnames, region identifiers.
- OAuth client IDs (the public half — but the secret half is).
- Public keys.
- Configuration knobs that don't grant access.

Treating non-secrets as secrets is a low-grade tax that erodes the practice — engineers stop respecting the boundary because it's pointlessly large.

---

## 2. The Spectrum of Storage

| Approach | Strength | Weakness | Typical use |
|----------|----------|----------|-------------|
| Hardcoded in source | None — leaks on git push | Always wrong | (don't) |
| `.env` file in repo | None | Leaks on git push | (don't) |
| `.env` ignored by git, loaded at boot | Out of source | Lives in plaintext on dev machines, CI secrets jumble | Dev only |
| Environment variables from CI/CD secret store | Decoupled from source | Visible in process listings, dumped on crash | OK for many cases; pair with rotation |
| Mounted file from secret store | Not in `env`, can be re-read on rotation | Still on disk in container | Better |
| Fetched at runtime from Vault / Secrets Manager | Fresh on each call, audited | Adds dependency to startup path | Production default |
| Dynamic, just-in-time issued credentials | No long-lived secret to leak | Highest setup cost | High-stakes systems |

There's no single right answer; the answer depends on threat model, scale, and operational tolerance.

---

## 3. Environment Variables — Twelve-Factor Reality Check

The Twelve-Factor App says *config* (including credentials) lives in the environment. This is **right** about the *interface* and **mostly fine** about the *storage* — provided you're aware of the gaps.

### 3.1 Why Env Vars

- Decoupled from code; same image runs in any environment.
- Trivially inspected by every language and framework.
- Easy to override per-environment.

### 3.2 The Gaps

- **Process inspection.** `/proc/<pid>/environ`, `ps eauxww`, debugger attach. On a multi-tenant host, anyone with the same UID sees them.
- **Crash dumps.** Java heap dumps, Node `--report-on-fatalerror`, core dumps — often include env.
- **Subprocess inheritance.** A spawned child process inherits the env. If the child is user-influenced (a shell-out for image processing), that's a problem.
- **CI logs.** Many CI systems "mask" secrets in logs but it's regex-based and frequently bypassed (e.g., base64-encoded forms).
- **Verbose error messages.** Frameworks logging "config" objects sometimes include secrets.

### 3.3 Practical Hardening

- **Don't print env at startup.** Tempting for debugging, fatal in production.
- **Configure logging to scrub.** Redact based on key name patterns (`*PASSWORD*`, `*SECRET*`, `*TOKEN*`).
- **Use mounted files instead of env** for high-stakes secrets (TLS keys, JWT signing keys). They can be re-read on rotation, don't appear in `/proc/.../environ`, and aren't inherited by subprocesses.
- **Don't pass secrets via CLI args.** `ps` shows them.

### 3.4 .env in Development

`.env` files are fine for dev provided:

- The file is in `.gitignore` (verify in CI).
- Pre-commit hooks run a leak scanner.
- Developers know not to copy production secrets into dev.

`direnv`, `dotenvx`, `dotenv-vault` reduce friction without weakening the boundary.

---

## 4. HashiCorp Vault

Vault is the de facto open-source secrets manager. Worth knowing even if you end up on a managed service.

### 4.1 Core Concepts

- **Secret engines.** Backends that store or generate secrets: KV, Database (issues short-lived DB users), AWS (issues short-lived IAM creds), PKI (acts as a CA), Transit (encryption-as-a-service).
- **Auth methods.** How clients prove identity: AppRole, Kubernetes Service Account JWT, AWS IAM, OIDC, GitHub.
- **Policies.** ACLs in HCL: which paths a token may read/write/list.
- **Tokens.** What clients present. Short-lived, scoped to policies.
- **Lease + renewal.** Most secrets have a TTL; clients renew before expiry or get fresh ones.

### 4.2 Typical Flow for a Service

```
1. Pod starts → reads its K8s ServiceAccount JWT.
2. Calls Vault: POST /v1/auth/kubernetes/login with the JWT.
3. Vault verifies with the cluster's TokenReview API.
4. Vault returns a short-lived Vault token bound to a policy.
5. Service reads /v1/database/creds/myrole → fresh DB username + password (TTL 1h).
6. Service connects to DB with those creds.
7. Service renews lease before TTL or re-reads.
8. When the service exits, the DB user is auto-revoked.
```

The DB never sees a long-lived password. There's no secret to rotate by hand. There's a per-pod audit trail of every credential issuance.

### 4.3 Spring Cloud Vault Example

```yaml
spring:
  cloud:
    vault:
      uri: https://vault.internal:8200
      authentication: KUBERNETES
      kubernetes:
        role: my-app
      database:
        enabled: true
        role: my-app-db
        backend: database
  datasource:
    url: jdbc:postgresql://db.internal:5432/app
    # username/password injected from Vault dynamically
```

Spring Cloud Vault refreshes the credentials on lease expiry. Your code doesn't know the password rotated.

### 4.4 Operational Notes

- Vault is critical-path. Plan for HA (3+ nodes), backups (Raft snapshots), and a sealed-key recovery procedure.
- Audit every secret read. Vault writes structured audit logs; ship to your SIEM.
- Use namespaces (Enterprise) or carefully scoped policies for multi-tenant.

---

## 5. Cloud-Managed Secret Stores

If you're already on a cloud, the managed service usually wins on operational simplicity.

### 5.1 AWS Secrets Manager

- Secrets are JSON blobs encrypted with a KMS key.
- IAM-controlled access (`secretsmanager:GetSecretValue` on a specific ARN).
- Built-in rotation via Lambda functions for RDS, Redshift, DocumentDB.
- Integrates with ECS/EKS via the Secrets Store CSI Driver and Pods Identity.
- Cost is per-secret, per-month + per-API-call.

```typescript
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManagerClient({ region: 'us-east-1' });
const out = await client.send(new GetSecretValueCommand({ SecretId: 'prod/db/credentials' }));
const { username, password } = JSON.parse(out.SecretString!);
```

### 5.2 AWS Systems Manager Parameter Store

- Cheaper alternative for simple cases; SecureString parameters use KMS too.
- Less feature-rich (no built-in rotation), fine for most app secrets.

### 5.3 GCP Secret Manager

- Versioned secrets, IAM access, encryption with Cloud KMS.
- Rotation via Cloud Scheduler + Cloud Run jobs.
- Workload Identity Federation lets GKE pods authenticate without a key file.

```typescript
import { SecretManagerServiceClient } from '@google-cloud/secret-manager';
const client = new SecretManagerServiceClient();
const [version] = await client.accessSecretVersion({
  name: 'projects/my-project/secrets/db-password/versions/latest',
});
const password = version.payload?.data?.toString();
```

### 5.4 Azure Key Vault

- Keys, secrets, and certs in one service.
- HSM-backed keys (Premium tier).
- Managed Identities for AKS / VMs / Functions integrate without secrets.

### 5.5 Comparison

| | AWS SM | GCP SM | Azure KV | Vault (self-host) |
|--|--------|--------|----------|--------------------|
| Built-in DB rotation | ✓ (RDS, etc.) | Limited | ✓ | ✓ (every supported DB) |
| Dynamic credentials | Lambda-based, custom | Limited | Custom | ✓ Native |
| HSM | KMS-backed | Cloud HSM tier | ✓ | Auto-Unseal with cloud HSM |
| Multi-cloud | No | No | No | ✓ |
| Operational burden | Low | Low | Low | High |
| Cost at small scale | Low | Low | Low | Free + ops time |

Choose managed unless you have a strong reason (multi-cloud, regulatory) for self-hosting Vault.

---

## 6. Kubernetes Secrets

The most misunderstood object in Kubernetes. By default, K8s Secrets are **base64-encoded plaintext** in etcd, *not* encrypted. Many security incidents trace back to that single fact.

### 6.1 What K8s Secrets Are

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-password
type: Opaque
data:
  password: c2VjcmV0   # base64 of "secret"
```

They're a key-value store with the same RBAC as other API resources. Mounted into pods as files (preferred) or env vars.

### 6.2 The Encryption-at-Rest Gap

Out of the box, etcd stores Secrets in plaintext. Anyone with etcd access (and on many clusters, anyone with `cluster-admin` Kubernetes access) reads them.

**Fix.** Enable the `EncryptionConfiguration` API:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - kms:
          name: cloud-kms
          endpoint: unix:///var/run/kmsplugin/socket.sock
          cachesize: 1000
      - identity: {}
```

The KMS provider plugs into AWS KMS / GCP KMS / Azure Key Vault. The data encryption key is wrapped by the cloud KMS root key. Even etcd backups can't be decrypted without KMS access.

EKS/GKE/AKS expose this as a one-click option. Use it.

### 6.3 RBAC on Secrets

`get`/`list` on `secrets` is the most dangerous verb in Kubernetes RBAC. A compromised pod with that permission reads every secret in its namespace. Audit:

```bash
kubectl auth can-i list secrets --as system:serviceaccount:default:my-app
```

Default to *no* secret access from workload service accounts. Use the **Secrets Store CSI Driver** to mount secrets directly from Vault / AWS SM into pods without storing them as K8s Secrets.

### 6.4 Pod-Level Hardening

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false   # if not needed
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
    - name: app
      image: my-app:1.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities: { drop: [ALL] }
      volumeMounts:
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: my-app-secrets
```

---

## 7. Secrets-in-Git: Sealed Secrets, SOPS, and age

GitOps wants the desired state in Git. Plaintext secrets in Git are unacceptable. Three popular reconciliations:

### 7.1 Sealed Secrets (Bitnami)

A controller in the cluster holds a private key. You encrypt secrets with the public key (using `kubeseal`) and commit the encrypted `SealedSecret` to Git. The controller decrypts and creates a `Secret` in the cluster.

```bash
echo -n s3cr3t | kubeseal --raw --name db-password --namespace prod \
  > sealed-password.bin
```

Pros: tight integration with K8s, simple model.
Cons: cluster-bound (sealed secrets can't be re-applied to a different cluster without the same key); key rotation is non-trivial.

### 7.2 SOPS (Mozilla)

Encrypt structured files (YAML, JSON, env, ini). Per-key encryption — you can diff what changed. Backed by AWS KMS, GCP KMS, Azure Key Vault, age, or PGP.

```yaml
db:
    password: ENC[AES256_GCM,data:...,iv:...,tag:...,type:str]
```

Pros: works for any config file, not just K8s; great review experience (only encrypted values change in diffs).
Cons: developers need access to the underlying key.

### 7.3 age

Modern replacement for PGP. Small (< 1 MB binary), tied to short ed25519 keys, simple CLI. Often used as a backend for SOPS or directly via `age` for blob encryption.

```bash
age -r age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p \
    -o secrets.yaml.age secrets.yaml
```

Pros: simpler than PGP, no web of trust theater.
Cons: smaller ecosystem; still emerging.

### 7.4 What to Pick

- **K8s-only, GitOps**: Sealed Secrets if cluster-bound is fine, SOPS+KMS if you want portability and audit.
- **App configs broadly**: SOPS.
- **Just blob encryption**: age.

Avoid: PGP. The UX is awful and enthusiasm has migrated to age and SOPS.

---

## 8. KMS-Backed Envelope Encryption

The pattern underneath every cloud secret store and most file-encryption tools: **envelope encryption.**

```
1. Generate a Data Encryption Key (DEK) — random AES-256 key.
2. Encrypt the data with DEK (AES-GCM).
3. Encrypt DEK with the Key Encryption Key (KEK) held in KMS.
4. Store [encrypted_data, encrypted_DEK] together.

To decrypt:
1. Send encrypted_DEK to KMS, get plaintext DEK back.
2. Decrypt data with DEK.
3. Forget DEK.
```

Why this is good:

- The KEK never leaves the HSM/KMS. Even an admin can't extract it.
- DEK is held in process memory only briefly.
- Re-encrypting the data means just re-wrapping the DEK with a new KEK — no need to re-encrypt all data.
- KMS can audit every `Decrypt` call.

This is how AWS KMS, GCP KMS, Azure Key Vault, Vault Transit, and HSMs work under the hood. Most secret stores expose envelope encryption transparently.

```typescript
// Example with AWS KMS
import { KMSClient, EncryptCommand, DecryptCommand } from '@aws-sdk/client-kms';
import { randomBytes, createCipheriv, createDecipheriv } from 'node:crypto';

const kms = new KMSClient({ region: 'us-east-1' });
const KEY_ID = 'arn:aws:kms:us-east-1:123:key/...';

async function encrypt(plaintext: Buffer): Promise<Buffer> {
  const dek = randomBytes(32);
  const iv = randomBytes(12);
  const cipher = createCipheriv('aes-256-gcm', dek, iv);
  const encrypted = Buffer.concat([cipher.update(plaintext), cipher.final()]);
  const tag = cipher.getAuthTag();

  const wrapped = await kms.send(new EncryptCommand({ KeyId: KEY_ID, Plaintext: dek }));
  // Output format: [wrapped DEK length][wrapped DEK][iv][tag][ciphertext]
  const wrappedDek = Buffer.from(wrapped.CiphertextBlob!);
  const len = Buffer.alloc(4); len.writeUInt32BE(wrappedDek.length);
  return Buffer.concat([len, wrappedDek, iv, tag, encrypted]);
}
```

In production, use `@aws-crypto/client-node` (the AWS Encryption SDK) which handles framing, key context, and rotation correctly.

---

## 9. Dynamic Secrets and Short-Lived Credentials

The best secret is the one you don't have to rotate because it doesn't outlive its use.

### 9.1 Vault Database Secret Engine

```bash
vault read database/creds/my-app-role
# Key                Value
# lease_id           database/creds/my-app-role/abc123
# lease_duration     1h
# username           v-token-my-app-role-9aZ
# password           A1b2C3...
```

Vault asked PostgreSQL to `CREATE ROLE` with a fresh username/password and `GRANT` based on the role's SQL template. After 1h the lease expires and Vault `DROP`s the user.

### 9.2 AWS STS

`sts:AssumeRole` returns short-lived credentials (15 min – 12 h) for an IAM role. Pods authenticate with their service account (IRSA on EKS, Workload Identity on GKE) and get role credentials at runtime. **No long-lived keys** ever exist.

### 9.3 GitHub Actions OIDC → AWS / GCP

Actions present an OIDC token signed by GitHub. AWS / GCP exchange it for short-lived credentials. The repo never holds an `AWS_ACCESS_KEY_ID`. This is how mature CI integrates with cloud in 2026.

### 9.4 SPIFFE / SPIRE

Workload identity for service mesh. Each workload gets a short-lived SVID (SPIFFE Verifiable Identity Document) — an X.509 cert signed by the SPIRE Server. mTLS terminates with proof of identity, no shared secret.

---

## 10. Secret Rotation in Practice

### 10.1 Why Rotate

- Limit blast radius of an undetected leak (the leaked secret stops working).
- Force the system to handle rotation continuously (so when emergency rotation happens, it's not the first time).
- Compliance (PCI, SOC 2 type II, HIPAA all expect rotation).

### 10.2 Rotation Modes

- **Hot rotation.** Two secrets valid simultaneously for a window. Issue new, propagate, retire old. Required for any secret used by multiple processes that don't restart together.
- **Restart rotation.** Update the secret in the store, restart the consumer, which picks up the new value. Acceptable for stateless apps with rolling deployments.
- **Just-in-time (no rotation needed).** Dynamic secrets per-request — the secret never lives long enough to need rotation.

### 10.3 What's Hard

- **Long-lived clients** (mobile apps with cached creds). Plan for graceful migration paths.
- **Symmetric keys used across services** (HMAC for webhooks). Use key IDs in messages so multiple keys validate during transition.
- **Database users.** Rolling rotation requires the app to support two passwords during the cutover (some libraries don't).
- **Vendor-side rotation** (Stripe, Twilio). Have a runbook; some vendors require notice; some don't.

### 10.4 Webhook Signing Key Rotation

```
Sender computes: hmac(key_v2, body), header X-Sig-V2

Receiver:
  current_keys = { "v1": "...", "v2": "..." }
  for kid, key in current_keys:
    if hmac(key, body) == provided[kid]:
      accept
```

Adding a key version is the safe way to rotate without dropping in-flight requests.

---

## 11. Leak Detection and Response

### 11.1 Pre-Push Detection

- **gitleaks** — fast, regex-based; runs locally and in CI.
- **trufflehog** — verifies leaked credentials are still active by hitting the vendor API.
- **GitHub Push Protection** — built-in for many vendors (AWS, Stripe, Slack, etc.). Blocks the push if a known token format is detected.
- **Pre-commit hooks** — `pre-commit` framework runs gitleaks/detect-secrets on every commit.

### 11.2 Continuous Detection

- **GitHub Secret Scanning** (free for public repos, paid for private). Vendors auto-revoke leaked tokens; e.g., AWS revokes leaked access keys within minutes.
- **GitGuardian, Spectral, Datadog Sensitive Data Scanner.**
- **Cloud-side**: AWS GuardDuty has a Credential Exfiltration finding type; GCP Security Command Center has equivalents.

### 11.3 Incident Response Runbook

When a secret leaks:

1. **Rotate immediately.** Don't wait for forensics. The window of exposure starts at leak time.
2. **Revoke at the vendor.** Don't just remove from the secret store; explicitly revoke at the issuing service.
3. **Audit usage.** Pull access logs for the credential — when was it last used legitimately? Anything after that is suspect.
4. **Search broadly.** If one secret leaked from a config file, others probably did too. Scan with trufflehog over Git history.
5. **Post-incident.** What process failure put the secret in a leakable place? Add the corresponding pre-commit / CI / push-protection control.

### 11.4 Real Incidents

- **Uber 2016** — AWS keys committed to a private GitHub repo accessed by an attacker who scraped via a former engineer's credentials. ~57M records.
- **Codecov 2021** — bash uploader script tampered with; CI environments leaked their env vars (which contained secrets) to attacker-controlled infrastructure.
- **Toyota 2023** — source code accidentally posted to GitHub for years exposed access keys to customer data.

The pattern: secrets in Git, secrets in CI env, secrets shipped to third-party services as a side effect.

---

## 12. Decision Guide

### 12.1 By Stage

| Stage | Reasonable choice |
|-------|-------------------|
| Local dev | `.env` file in `.gitignore`, pre-commit gitleaks |
| Single-server side project | Env vars from systemd / `.env.local` outside repo |
| Small team, one cloud | Cloud-managed secret store (AWS SM / GCP SM / Key Vault) |
| Multi-cloud / regulated | Vault + KMS-backed encryption |
| Kubernetes | Secrets Store CSI Driver + cloud secret manager OR Vault Agent injector |
| GitOps | SOPS + KMS for application config; Sealed Secrets or External Secrets Operator for K8s Secrets |
| High-stakes | Dynamic secrets via Vault DB engine + STS + workload identity; no long-lived secrets |

### 12.2 By Secret Type

| Secret | Best home |
|--------|-----------|
| DB password | Vault dynamic creds, or rotated managed-store secret |
| API key (3rd party) | Managed secret store; rotate quarterly |
| TLS private key | Filesystem + restricted perms; or KMS-wrapped if stored in app |
| JWT signing key | KMS for signing operations; never in app memory |
| Webhook signing secret | Managed secret store; key versioning header |
| SSH key for deploy | Short-lived OIDC-issued cert, not a long-lived key |
| AWS credentials in CI | OIDC federation, not access keys |
| Encryption-at-rest key | Cloud KMS; use envelope encryption |

### 12.3 What to Avoid

- Hardcoded secrets in source.
- Secrets in `application.properties` / `application.yml` *committed to the repo*.
- Secrets in container image layers (someone will `docker save` and tar the layers).
- Secrets in build args (visible in `docker history`).
- Long-lived static cloud credentials when workload identity is available.
- "Just use env vars" without thinking about what process inspection looks like.

---

## Related

- [TLS Handshake and Certificates](06-tls-handshake-and-certificates.md) — TLS private keys are the canonical secret
- [JWT Design and Pitfalls](05-jwt-design-and-pitfalls.md) — signing keys management
- [OAuth 2.0 and OIDC Deep Dive](04-oauth2-and-oidc-deep-dive.md) — client secrets, JWT client assertions
- [OWASP Top 10 (2021)](01-owasp-top-10.md) — A02 Cryptographic Failures
- [Spring Security — Secrets Management](../../java/security/secrets-management.md)
- [Kubernetes Secrets and Supply Chain](../../kubernetes/security/secrets-and-supply-chain.md)
- [System Design — Secrets Management and Key Rotation](../../system-design/security/secrets-management-and-key-rotation.md)

---

## References

- **The Twelve-Factor App — Config** — https://12factor.net/config
- **HashiCorp Vault Documentation** — https://developer.hashicorp.com/vault/docs
- **Spring Cloud Vault** — https://docs.spring.io/spring-cloud-vault/docs/current/reference/html/
- **AWS Secrets Manager** — https://docs.aws.amazon.com/secretsmanager/
- **AWS Encryption SDK** — https://docs.aws.amazon.com/encryption-sdk/
- **GCP Secret Manager** — https://cloud.google.com/secret-manager/docs
- **Azure Key Vault** — https://learn.microsoft.com/en-us/azure/key-vault/
- **Kubernetes — Encrypting Secret Data at Rest** — https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- **Secrets Store CSI Driver** — https://secrets-store-csi-driver.sigs.k8s.io/
- **Bitnami Sealed Secrets** — https://github.com/bitnami-labs/sealed-secrets
- **Mozilla SOPS** — https://github.com/getsops/sops
- **age** — https://github.com/FiloSottile/age
- **SPIFFE / SPIRE** — https://spiffe.io/
- **NIST SP 800-57** — Recommendation for Key Management. https://csrc.nist.gov/projects/key-management/key-management-guidelines
- **NIST SP 800-131A** — Transitioning the Use of Cryptographic Algorithms and Key Lengths. https://csrc.nist.gov/pubs/sp/800/131/a/r2/final
- **gitleaks** — https://github.com/gitleaks/gitleaks
- **trufflehog** — https://github.com/trufflesecurity/trufflehog
- **GitHub Secret Scanning** — https://docs.github.com/en/code-security/secret-scanning
- **OWASP Secrets Management Cheat Sheet** — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- **AWS — Use IAM Roles for Service Accounts (IRSA)** — https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html
- **GCP — Workload Identity Federation** — https://cloud.google.com/iam/docs/workload-identity-federation
