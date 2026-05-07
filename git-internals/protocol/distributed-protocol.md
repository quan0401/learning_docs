---
title: "The Distributed Protocol — Wire Protocol, Hooks, and Signing"
date: 2026-05-07
updated: 2026-05-07
tags: [git, git-internals, protocol, hooks, signing]
---

# The Distributed Protocol — Wire Protocol, Hooks, and Signing

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `git` `git-internals` `protocol` `hooks` `signing`

---

## Table of Contents

1. [Wire Protocols v0, v1, v2](#1-wire-protocols-v0-v1-v2)
2. [Smart HTTP and SSH Transport](#2-smart-http-and-ssh-transport)
3. [Partial, Shallow, and Sparse Clones](#3-partial-shallow-and-sparse-clones)
4. [Hooks — Client and Server](#4-hooks--client-and-server)
5. [Submodules and Subtrees](#5-submodules-and-subtrees)
6. [Signed Commits and Tags](#6-signed-commits-and-tags)
7. [Plumbing Walkthrough](#7-plumbing-walkthrough)

## Summary

Tiers 1–5 stayed inside one `.git/` directory. This tier crosses the network. When you run `git clone`, `git fetch`, `git push` — bytes move between two `git` processes that speak a framed line-oriented protocol over a pipe. The pipe might be SSH, HTTPS, or a local socket; the framing (pkt-line) and the conversation (capability advertisement → ref discovery → want/have negotiation → packfile) are the same.

This doc covers the three protocol versions Git supports today, the two transports (Smart HTTP and SSH) that run them in production, the partial/shallow/sparse clone optimizations that v2 enables, the hooks that fire on either end, the two strategies for embedding repos in repos (submodules vs subtrees), and the three ways to sign commits (GPG, SSH-key signing in Git 2.34+, and keyless OIDC via sigstore's `gitsign`). The walkthrough at the end runs `GIT_TRACE_PACKET=1 git fetch` and reads the wire conversation by hand.

The mental model: every Git transport operation is a conversation between an **`upload-pack`** process (server-side, when the client is fetching) and a client, or between a **`receive-pack`** process (server-side, when the client is pushing) and a client. Both sides speak pkt-line. Capability negotiation determines what features they agree on. The conversation either ends with a packfile flowing one direction, or with a `report-status` block flowing the other.

---

## 1. Wire Protocols v0, v1, v2

Git has three wire protocol versions. All three are **framed in pkt-line format**: a 4-byte ASCII hex length prefix followed by length minus 4 bytes of payload, with three special short packets reserved (`0000` flush, `0001` delim, `0002` response-end).

### 1.1 pkt-line framing

```text
0008abcd          → 4 bytes of length "0008", 4 bytes of payload "abcd"
0006a\n           → 4 bytes of length "0006", 2 bytes "a\n"
0000              → flush packet — end of message
0001              → delim packet — separator between sections (v2 only)
0002              → response-end packet (v2 stateless connections)
```

The 4-byte length includes itself. So the minimum data packet is `0005x` (5 bytes total, 1 byte of payload). `0004` is forbidden; `0000`/`0001`/`0002` are the special control packets.

The pkt-line spec lives in [`Documentation/gitprotocol-common.txt`](https://git-scm.com/docs/gitprotocol-common) and underpins both upload-pack and receive-pack. Everything else in this section is just _what we put inside pkt-lines_.

### 1.2 Protocol v0 — the original

v0 is the protocol from Git 1.0. The server, on connect, **immediately blasts every ref** in the repository with capabilities tacked onto the first ref line after a NUL byte:

```text
00887217a7c7e582c46cec22a130adf4b9d7d950fba0 HEAD\0multi_ack thin-pack side-band side-band-64k ofs-delta
00441d3fcd5ced445d1abc402225c0b8a1299641f497 refs/heads/integration
003f7217a7c7e582c46cec22a130adf4b9d7d950fba0 refs/heads/master
0000
```

For a repository with 200,000 refs (every PR branch on a busy monorepo), the server transmits megabytes of refs the client doesn't care about before negotiation can start. This is **the v0 problem**. v1 didn't fix it; v2 did.

### 1.3 Protocol v1 — version-tagged v0

v1 is essentially v0 with one extra opening pkt-line:

```text
000eversion 1
```

The point of v1 was to give clients and servers a way to negotiate "are we both willing to upgrade?" before paying any v0 transmission cost. v1 itself fixes nothing about the ref-advertisement problem. In practice, you only see v1 in the wild when something is misconfigured between client and server.

### 1.4 Protocol v2 — capability-advertised, command-driven

v2 inverts the model. The server **first advertises capabilities** and waits. The client then sends an explicit command (`ls-refs` for ref discovery, `fetch` for the packfile exchange, `object-info` for metadata). Everything is request-driven.

Capability advertisement (verbatim shape from `gitprotocol-v2.txt`):

```text
000eversion 2\n
0014agent=git/2.40.0\n
000cls-refs\n
0010fetch=shallow\n
0012object-format=sha1\n
0000
```

Reading the capability list:

| Capability | Meaning |
|------------|---------|
| `agent=...` | Server's `git` build (informational, optional) |
| `ls-refs` | Server supports the `ls-refs` command |
| `fetch=shallow` | Server supports `fetch`, with the `shallow` argument set |
| `object-format=sha1` | Server's hash algorithm — SHA-1 default, SHA-256 in transition repos |

The client then sends a command request — capability-list, then `0001` delim-pkt, then arguments, then `0000` flush:

```text
000ecommand=ls-refs\n
0012agent=git/2.40.0\n
0001
000bref-prefix HEAD\n
0014ref-prefix refs/heads/\n
0000
```

`ref-prefix` is the v2 superpower: the client filters server-side, so a `git fetch origin main` in a 200,000-ref repo only transfers refs matching `refs/heads/main`, not all 200,000.

### 1.5 Why v2 is filterable and on-demand

Three properties make v2 the foundation of every modern large-repo workflow:

1. **Server-side ref filtering** via `ref-prefix` — the ref advertisement scales with what you asked for, not what the server has.
2. **Capability-gated features** — partial clone (`filter`), shallow (`shallow`/`deepen`), bundle URIs, promisor remotes are advertised as fetch arguments. The client only enables what the server supports.
3. **Stateless-friendly framing** — `0002` response-end-pkt and the explicit `command=` per-request shape let v2 work cleanly over stateless HTTPS without long-lived sessions.

To enable v2 globally on the client:

```bash
git config --global protocol.version 2
# verify
git config --get protocol.version
```

Default has been v2 since Git 2.26 (released 2020). You almost never need to flip this manually now; you _do_ sometimes need to flip it _back_ to v0 when working against an ancient on-prem Stash/Gerrit that hasn't been patched.

---

## 2. Smart HTTP and SSH Transport

The wire protocol is transport-agnostic. Two transports dominate in practice: Smart HTTP and SSH. (`git://` exists, is unauthenticated, and is mostly historical at this point.)

### 2.1 Smart HTTP

Smart HTTP runs the pkt-line protocol over two HTTPS request/response pairs.

**Step 1 — info/refs (capability advertisement):**

```text
GET https://github.com/git/git.git/info/refs?service=git-upload-pack HTTP/1.1
Git-Protocol: version=2

HTTP/1.1 200 OK
Content-Type: application/x-git-upload-pack-advertisement

000eversion 2\n
0014agent=git/2.40.0\n
000cls-refs\n
0010fetch=shallow\n
0012object-format=sha1\n
0000
```

The `service` query param tells the server which backend to spawn — `git-upload-pack` for fetches, `git-receive-pack` for pushes. The `Git-Protocol: version=2` header is how the client requests v2 over HTTP (otherwise the server falls back to v0).

**Step 2 — POST the command:**

```text
POST https://github.com/git/git.git/git-upload-pack HTTP/1.1
Content-Type: application/x-git-upload-pack-request
Git-Protocol: version=2

000ecommand=ls-refs\n
...
0000
```

The server's HTTP response is the pkt-line response. For `command=fetch`, the response body contains the packfile. The whole conversation is two HTTP round-trips for v2 (sometimes a few more if the negotiation needs additional rounds).

### 2.2 SSH

SSH is a single bidirectional channel. The client opens an SSH connection to the host and runs `git-upload-pack '/repo.git'` (or `git-receive-pack`) as a remote command. To request v2 over SSH, the client sets the `GIT_PROTOCOL` environment variable on the remote side:

```text
GIT_PROTOCOL=version=2 git-upload-pack '/repo.git'
```

OpenSSH only forwards env vars listed in the server's `AcceptEnv` directive. Most Git hosting providers configure their SSH frontends to accept `GIT_PROTOCOL`. For your own SSH server, add to `/etc/ssh/sshd_config`:

```text
AcceptEnv GIT_PROTOCOL
```

Without this, the remote `git-upload-pack` never sees the variable and falls back to v0.

### 2.3 `git-upload-pack` vs `git-receive-pack`

Two server-side processes, one for each direction of data flow:

| Aspect | `git-upload-pack` | `git-receive-pack` |
|--------|-------------------|--------------------|
| Direction | Server → client (fetch/clone) | Client → server (push) |
| Client command | `git fetch`, `git clone`, `git pull`, `git ls-remote` | `git push` |
| Negotiation | Want/have, then packfile to client | Old/new oid pairs, packfile from client |
| Server hooks | None | `pre-receive`, `update`, `post-receive`, `post-update` |
| Failure modes | Slow clone, ref advertisement timeout | Hook rejection, non-fast-forward refused |

The split matters for permissions: a "read-only deploy key" in your CI is just an SSH key whose `command=` forced-command in `authorized_keys` only allows `git-upload-pack`. A push-capable key allows both.

### 2.4 Transport differences in practice

- **HTTP is stateless.** The server doesn't remember the client between the `info/refs` GET and the command POST. v2's `command=` framing is what makes this clean.
- **SSH is a single long-lived stream.** v0/v1 negotiation can interleave more freely; v2 still uses request/response shape but doesn't pay HTTP framing overhead.
- **HTTP integrates with corporate proxies and load balancers.** SSH usually doesn't — large-scale Git hosting (GitHub, GitLab) front HTTPS, route to backend pools, and treat SSH as a separate fleet.
- **Auth differs.** HTTPS uses Basic/token auth or OAuth-shaped headers. SSH uses keys (or, for `gh`, SSH cert auth). Both ultimately gate which `upload-pack`/`receive-pack` invocation runs.

---

## 3. Partial, Shallow, and Sparse Clones

Three orthogonal optimizations to the "clone fetches everything" default. They compose: a shallow + partial + sparse clone is the standard recipe for monorepo CI.

### 3.1 Shallow clone — `--depth=N`

```bash
git clone --depth=1 https://github.com/git/git.git
```

The server walks N commits back from each requested ref's tip and stops. The client gets just those commits, plus the trees and blobs they reach. Older history is _not_ on disk.

```bash
# Inspect the shallow boundary:
cat .git/shallow
# → 7c4e5e3... (the SHAs at which history was cut)
```

Shallow clones are great for CI and bad for archaeology. `git log` only sees the depth-N window. `git blame` may stop at the shallow boundary. To "unshallow" later:

```bash
git fetch --unshallow
```

Related options from `git-clone(1)`:

- `--shallow-since=<date>` — cut at a specific time
- `--shallow-exclude=<ref>` — cut at the tip of a ref
- `--shallow-submodules` — cascade to submodules
- `--single-branch` — implied by `--depth`; clone only one branch's history

### 3.2 Partial clone — `--filter=blob:none`

Partial clone fetches commits and trees normally but defers blob (file content) downloads until they're needed. The clone is a **promisor remote**: it stands in for "I'll provide the missing objects on demand."

```bash
git clone --filter=blob:none https://github.com/git/git.git
```

Common filter specs (from `git-clone(1)`):

| Filter | Behavior |
|--------|----------|
| `blob:none` | No blobs initially. All trees + commits. Blobs fetched on access. |
| `tree:0` | No blobs, no trees. Only commits. Trees + blobs fetched as needed. |
| `blob:limit=10m` | Blobs ≥ 10 MB excluded; smaller ones included. Useful for repos with large binaries. |
| `auto` | Honor the filter the server advertises via the promisor-remote protocol. |

`tree:0` is the most aggressive ("treeless clone"). Even a `git log -- some/path` will trigger network fetches for the trees needed to resolve the path. Use only when you're sure CI will only run a few targeted commands.

`blob:none` is the practical sweet spot: history navigation is fast and offline, file _contents_ stream on demand.

### 3.3 Sparse checkout — only check out a subtree

Partial clone controls what's _on disk in `.git/objects`_. Sparse checkout controls what's _in the working tree_. They compose:

```bash
git clone --filter=blob:none --sparse https://github.com/git/git.git
cd git
git sparse-checkout set Documentation/
```

`--sparse` initializes the sparse-checkout to only the top-level files. `git sparse-checkout set` widens it. The `skip-worktree` bit on each index entry tells Git "don't materialize this in the working tree." Combined with `blob:none`, this is the configuration most monorepo CI uses.

### 3.4 What ends up on disk vs fetched on demand

A useful table for reasoning about it:

| Configuration | Commits on disk | Trees on disk | Blobs on disk | Working-tree files |
|---------------|-----------------|---------------|----------------|---------------------|
| Default clone | All reachable | All reachable | All reachable | All paths checked out |
| `--depth=1` | Just the tip + N back | Just those reachable | Just those reachable | All paths checked out |
| `--filter=blob:none` | All reachable | All reachable | **Lazy** | All paths checked out (on demand) |
| `--filter=tree:0` | All reachable | **Lazy** | **Lazy** | Lazy |
| `--sparse` | All reachable | All reachable | All reachable | Only sparse cone |
| All three combined | Just N back | All reachable for those | Lazy | Only sparse cone |

GitHub's "Get up to speed with partial clone" blog post is the canonical narrative tour; the reference data above is from `git-clone(1)`.

### 3.5 Common pitfalls

- **Older Git on the server.** Partial clone needs server-side support (`uploadpack.allowFilter=true`). On-prem GitLab/Stash from before ~2019 may reject the filter.
- **Disconnected operation breaks lazy fetches.** A `--filter=blob:none` clone on a plane: `git checkout some-old-branch` may try to fetch missing blobs and fail.
- **`git gc` won't delete promised objects.** Promisor remotes mark fetched objects so GC keeps them. The flip side: a botched promisor remote config can leak object IDs that can never be resolved locally.

---

## 4. Hooks — Client and Server

Hooks are executable scripts (or binaries) that Git invokes at specific points in the lifecycle. They live in `.git/hooks/` by default. Each hook has a fixed name; if a file with that name exists and has the executable bit, Git runs it. If exit status is non-zero (and the hook is one that can abort), Git aborts the operation.

`core.hooksPath` overrides the location, so `dotfiles` repos and team-wide hook policies can live outside any one repo:

```bash
git config --global core.hooksPath ~/.config/git/hooks
```

### 4.1 Client-side hooks

Client-side hooks fire on the developer's machine. They are **advisory** — anyone can bypass with `--no-verify`, and they are not transmitted with `git clone` (so a fresh clone has none of your team's hooks). Treat them as helpful, not as policy enforcement.

| Hook | Trigger | Stdin | Argv | Can abort? |
|------|---------|-------|------|------------|
| `pre-commit` | `git commit`, before message prompt | None | None | Yes |
| `prepare-commit-msg` | `git commit`, after message prepared | None | message file, source, [oid] | Yes |
| `commit-msg` | `git commit`, after edit | None | message file | Yes |
| `pre-push` | `git push`, before transfer | `<local-ref> <local-oid> <remote-ref> <remote-oid>` per line | remote name, URL | Yes |
| `post-commit` | After commit lands | None | None | No (notification) |
| `post-checkout` | After `checkout`/`switch`/`clone` | None | prev-head, new-head, branch-flag | No |
| `post-merge` | After merge completes | None | squash-flag | No |
| `post-rewrite` | After `commit --amend`/`rebase` | rewritten oid pairs | command | No |

A real-world `pre-commit` example that blocks commits with `console.log`:

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit
set -e
if git diff --cached --name-only --diff-filter=ACM | xargs -r grep -nH 'console\.log' ; then
  echo "pre-commit: refused — console.log found in staged changes" >&2
  exit 1
fi
```

For team-wide enforcement, projects pair hooks with [pre-commit.com](https://pre-commit.com/), [Husky](https://typicode.github.io/husky/), or [Lefthook](https://lefthook.dev/). These tools install a single `pre-commit` hook that delegates to a checked-in config file, so hooks _are_ versioned with the repo (since the hook itself is generic glue).

### 4.2 Server-side hooks

Server-side hooks fire inside `git-receive-pack` on the remote. They _can_ enforce policy because the server controls them and clients can't bypass. From `githooks(5)`:

| Hook | When | Stdin | Argv | Per-ref? |
|------|------|-------|------|----------|
| `pre-receive` | Once per push, before any ref update | `<old-oid> <new-oid> <ref-name>` per line | None | No (one call) |
| `update` | After `pre-receive`, before each ref update | None | ref-name, old-oid, new-oid | Yes (one per ref) |
| `post-receive` | After all refs updated | Same as `pre-receive` | None | No (one call) |
| `post-update` | After all refs updated | None | List of updated refs | No |

Typical splits:

- **`pre-receive`** is where you reject the whole push: enforce signed commits, run a license scan, block force-push to `main`.
- **`update`** is where you reject one ref while letting others through: per-branch protection rules.
- **`post-receive`** is where you trigger CI, send notifications, kick a deploy.

A minimal `pre-receive` that blocks force-pushes to `main`:

```bash
#!/usr/bin/env bash
# hooks/pre-receive on the server
while read old_oid new_oid ref_name; do
  if [ "$ref_name" = "refs/heads/main" ]; then
    # Non-fast-forward = $old_oid is not an ancestor of $new_oid
    if ! git merge-base --is-ancestor "$old_oid" "$new_oid" 2>/dev/null; then
      echo "pre-receive: refused — non-fast-forward push to main" >&2
      exit 1
    fi
  fi
done
exit 0
```

Hosted Git providers (GitHub, GitLab, Bitbucket) implement branch protection on top of `pre-receive`/`update` semantics. You don't write these hooks directly on GitHub — you configure rulesets, and GitHub's `pre-receive` enforces them. On self-hosted GitLab Enterprise / Bitbucket Data Center / Gerrit, you _can_ install custom server-side hooks.

### 4.3 `core.hooksPath` for shared hook policy

The `core.hooksPath` config key (since Git 2.9) lets you point all of a user's repos at one shared hook directory:

```bash
# In a corporate dotfiles install:
git config --global core.hooksPath /opt/corp-git-hooks
```

Now every repo on this machine fires `/opt/corp-git-hooks/pre-commit` before each commit. Combined with a managed-endpoint policy, this is a clean way to enforce things like "no committing files matching `**/.env`."

---

## 5. Submodules and Subtrees

Two strategies for embedding one Git repository inside another. They solve overlapping problems with different trade-offs.

### 5.1 Submodules — gitlinks + `.gitmodules`

A submodule is a **gitlink object** in the parent's tree: a special tree entry whose mode is `160000` and whose value is a commit OID, not a blob OID. The parent commit pins the child to a specific commit.

```bash
git submodule add https://github.com/example/lib.git vendor/lib
```

This creates two artifacts:

- `vendor/lib/` — a separate clone of the submodule
- `.gitmodules` — a config file in the parent's working tree (and committed) that maps each submodule path to its URL and optional branch:

```ini
[submodule "vendor/lib"]
        path = vendor/lib
        url = https://github.com/example/lib.git
```

When someone clones the parent:

```bash
git clone https://github.com/example/parent.git
git submodule update --init --recursive
# or in one step:
git clone --recurse-submodules https://github.com/example/parent.git
```

The parent records `vendor/lib`'s commit OID. To bump the submodule:

```bash
cd vendor/lib
git fetch
git checkout v2.0.0
cd ../..
git add vendor/lib            # stages the new gitlink OID in the parent
git commit -m "bump lib to v2.0.0"
```

**Foot-guns:**

- **Detached HEAD by default.** `git submodule update` checks out the recorded commit, leaving the submodule in detached HEAD. Easy to lose work in the submodule if you commit there without thinking.
- **Two-step pushes.** A change in the submodule must be pushed to the submodule's remote before the parent's gitlink is meaningful — otherwise other clones can't resolve the OID.
- **`git submodule sync`** is needed if upstream URLs change.
- **Custom `submodule.<name>.update` commands** in `.gitmodules` are intentionally _not_ copied to `.git/config` for security reasons (`!shell-command` would otherwise be drive-by code execution on `submodule init`).

### 5.2 Subtrees — `git subtree split` and `git subtree merge`

A subtree _merges_ another repo's history into a path inside your repo. There's no special object type; everything is normal commits and trees. The other repo just becomes a subdirectory.

```bash
# Add an external repo as a subtree at vendor/lib:
git subtree add --prefix=vendor/lib https://github.com/example/lib.git v2.0.0 --squash
```

The `--squash` flag collapses the imported history into a single commit so the parent doesn't gain thousands of upstream commits.

To pull updates:

```bash
git subtree pull --prefix=vendor/lib https://github.com/example/lib.git v2.1.0 --squash
```

To extract changes you've made under `vendor/lib` back as commits against the upstream:

```bash
git subtree split --prefix=vendor/lib --branch=lib-extracted
git push https://github.com/example/lib.git lib-extracted:main
```

`git subtree split` rewrites history for the prefix only, producing a parallel history of just commits that touched `vendor/lib`. That branch can be pushed back to the upstream.

### 5.3 Submodule vs subtree decision matrix

| Concern | Submodule | Subtree |
|---------|-----------|---------|
| Clone simplicity | Two-step, easy to forget `--recurse-submodules` | One step — looks like one repo |
| History size | Submodule history stays in submodule | Subtree commits inflate parent history (mitigate with `--squash`) |
| Update workflow | `cd` in, fetch, `git add` the gitlink | `git subtree pull` from parent root |
| Contributing back upstream | Natural: just push from the submodule | Awkward: `git subtree split` then push |
| CI/CD integration | All CI must know about submodules | Transparent — looks like normal directories |
| Best when | Vendoring a library you mostly track upstream | Vendoring a library you actively patch and rarely upstream |

In practice: **submodules** for "I track this dependency at exact pinned versions" (think a Helm chart in a Kubernetes manifest repo), **subtrees** for "I forked this and our changes have diverged enough that contributing back is rare."

---

## 6. Signed Commits and Tags

Git lets you cryptographically sign commits and tags so verifiers can confirm "this commit was authored by the holder of key K." Three implementations are in production use.

### 6.1 GPG signing — the original

Configure once:

```bash
git config --global user.signingkey 4B7C9F18A1B2C3D4
git config --global commit.gpgSign true
git config --global tag.gpgSign true
```

`commit.gpgSign=true` makes every `git commit` invoke `gpg` to sign the resulting commit. The signature is stored in a `gpgsig` header inside the commit object itself. `git verify-commit <oid>` runs `gpg --verify` against the stored signature.

```bash
git verify-commit HEAD
# gpg: Signature made Mon 04 May 2026 ...
# gpg: Good signature from "Alice <alice@example.com>"
```

Pain points: GPG key management (which keyserver?), expiring keys, agent issues on macOS, and onboarding new contributors who've never used GPG. This is why the next two options exist.

### 6.2 SSH signing — Git 2.34+

Since Git 2.34 (Nov 2021), Git can sign with SSH keys instead of GPG. You're already using SSH keys to push; reuse them for signing.

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgSign true
```

For verification, Git needs to know which SSH keys belong to which committers. The `gpg.ssh.allowedSignersFile` config points at an `allowed_signers` file (the OpenSSH format `ssh-keygen -Y verify` uses):

```bash
git config --global gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers
```

```text
# ~/.config/git/allowed_signers
alice@example.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIN...
bob@example.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAID...
```

`git verify-commit HEAD` now checks the SSH signature against this file. GitHub also supports SSH commit signing — uploading an SSH key as a "signing key" (separate from "auth key") gets the green "Verified" badge.

### 6.3 `gitsign` — keyless OIDC via sigstore

[`sigstore/gitsign`](https://github.com/sigstore/gitsign) is the third option. Per its README, it provides "Keyless Git signing with Sigstore." Instead of holding any long-lived key:

1. You run `git commit`.
2. `gitsign` opens a browser for OIDC auth via your identity provider (GitHub, Google, etc.).
3. It fetches a short-lived X.509 cert from sigstore's Fulcio CA.
4. It signs the commit with that ephemeral cert.
5. The signature is recorded in sigstore's Rekor transparency log.

Configuration:

```bash
git config --global gpg.x509.program gitsign
git config --global gpg.format x509
git config --global commit.gpgSign true
```

Verification calls Rekor to confirm the signature was logged at commit time and that the OIDC identity matches. There's no key to lose, no key to rotate, no key to revoke — the cert is one-shot.

### 6.4 When to use which

| Method | Strength | Weakness |
|--------|----------|----------|
| GPG | Universal, the longest-supported option | Painful UX, key management overhead |
| SSH (2.34+) | Reuses keys you already manage; great UX | Newer; requires `allowed_signers` for verification at scale |
| `gitsign` | Keyless; auditable via transparency log | Requires sigstore infra; OIDC must be configured |

For a single contributor's laptop in 2026, SSH signing is the easiest. For a CI bot signing release tags, `gitsign` is increasingly the standard.

---

## 7. Plumbing Walkthrough

Time to read the wire. All commands below are runnable against any public Git repo.

### 7.1 `git ls-remote` — ref discovery without cloning

```bash
git ls-remote https://github.com/git/git.git
# 7c4e5e3...        HEAD
# 7c4e5e3...        refs/heads/master
# 4b7d68a...        refs/heads/maint
# ...
# 1a2b3c4...        refs/tags/v2.40.0
# 5d6e7f8...        refs/tags/v2.40.0^{}
```

`ls-remote` runs only the ref-discovery half of the protocol (under v2 it issues `command=ls-refs`). No packfile is sent. This is the cheapest way to ask "what's on the remote right now?" — and it's the basis of `git fetch`'s decision about what's new.

`refs/tags/v2.40.0^{}` is the **peeled tag**: the commit OID that the annotated tag object points at. Servers advertise both so the client can resolve "want this tag's underlying commit" without an extra fetch.

### 7.2 `GIT_TRACE_PACKET=1 git fetch` — read the wire

```bash
cd /tmp
git clone https://github.com/git/git.git --filter=blob:none --depth=1
cd git
GIT_TRACE_PACKET=1 git fetch --depth=2 origin master 2>&1 | head -40
```

Realistic output shape (lines abbreviated):

```text
20:14:01.123 packet:        clone< version 2
20:14:01.123 packet:        clone< agent=git/github-...
20:14:01.123 packet:        clone< ls-refs=unborn
20:14:01.123 packet:        clone< fetch=shallow wait-for-done filter
20:14:01.123 packet:        clone< server-option
20:14:01.123 packet:        clone< object-format=sha1
20:14:01.123 packet:        clone< 0000
20:14:01.124 packet:      fetch> command=ls-refs
20:14:01.124 packet:      fetch> agent=git/2.40.0
20:14:01.124 packet:      fetch> object-format=sha1
20:14:01.124 packet:      fetch> 0001
20:14:01.124 packet:      fetch> peel
20:14:01.124 packet:      fetch> symrefs
20:14:01.124 packet:      fetch> ref-prefix refs/heads/master
20:14:01.124 packet:      fetch> 0000
20:14:01.225 packet:        clone< 7c4e5e3... refs/heads/master
20:14:01.225 packet:        clone< 0000
20:14:01.226 packet:      fetch> command=fetch
20:14:01.226 packet:      fetch> agent=git/2.40.0
20:14:01.226 packet:      fetch> object-format=sha1
20:14:01.226 packet:      fetch> 0001
20:14:01.226 packet:      fetch> thin-pack
20:14:01.226 packet:      fetch> ofs-delta
20:14:01.226 packet:      fetch> deepen 2
20:14:01.226 packet:      fetch> want 7c4e5e3...
20:14:01.226 packet:      fetch> done
20:14:01.226 packet:      fetch> 0000
20:14:01.412 packet:        clone< shallow-info
20:14:01.412 packet:        clone< shallow 4b7d68a...
20:14:01.412 packet:        clone< 0001
20:14:01.412 packet:        clone< packfile
20:14:01.412 packet:        clone< [PACK ...binary...]
```

Read it as a story:

1. Server advertises v2 capabilities (`<` is server → client).
2. Client sends `command=ls-refs` with `ref-prefix refs/heads/master` — server-side filter.
3. Server returns just one ref.
4. Client sends `command=fetch` with `want <oid>`, `deepen 2`, `done`.
5. Server returns `shallow-info` (which commits become the new shallow boundary), then the packfile.

`GIT_TRACE_PACKET=1` is the single most useful flag for debugging "why is `git fetch` slow?" or "why is the server rejecting my push?" The pkt-line transcript shows exactly what was negotiated.

### 7.3 `git upload-pack` over a local socket

You can run a Git server in two lines using `git daemon` or `git upload-pack` directly. The crudest way: pipe stdin/stdout.

```bash
# In one terminal:
mkdir /tmp/repo && cd /tmp/repo && git init --bare

# Push a commit to it from another clone:
cd /tmp/some-other-clone
git push /tmp/repo main

# Now read refs from /tmp/repo using upload-pack directly:
git upload-pack /tmp/repo < /dev/null | head
# → pkt-line v0/v1/v2 capability advertisement on stdout
```

For a closer-to-production demo with a Unix socket, use `socat`:

```bash
# Server:
socat UNIX-LISTEN:/tmp/git.sock,fork EXEC:'git upload-pack /tmp/repo'

# Client:
git -c protocol.version=2 ls-remote 'ext::socat - UNIX-CONNECT:/tmp/git.sock'
```

The `ext::` transport is Git's escape hatch for arbitrary command-line transports — very useful for testing custom proxies and instrumentation.

### 7.4 `git verify-commit` — check a signature

```bash
# Sign a commit:
git -c commit.gpgSign=true commit --allow-empty -m "signed empty commit"

# Verify:
git verify-commit HEAD
# (For SSH signing, with allowed_signers configured:)
# Good "git" signature for alice@example.com with ED25519 key SHA256:...

# View the raw signature header in the commit object:
git cat-file -p HEAD
# tree ...
# parent ...
# author Alice ...
# committer Alice ...
# gpgsig -----BEGIN SSH SIGNATURE-----
#  U1NIU0lHAAAAA...
#  -----END SSH SIGNATURE-----
#
# signed empty commit
```

The signature is a header in the commit object. Verification re-serializes the commit _without_ the `gpgsig` header and runs the configured backend (`gpg`, OpenSSH's `ssh-keygen -Y verify`, or `gpg.x509.program` for `gitsign`) against it.

### 7.5 Install a `pre-commit` hook from scratch

```bash
cd /tmp
git init demo-hooks && cd demo-hooks

# Write a hook that blocks commits whose message contains "WIP":
cat > .git/hooks/commit-msg <<'EOF'
#!/usr/bin/env bash
msg_file="$1"
if grep -qiE '^WIP\b' "$msg_file"; then
  echo "commit-msg: refused — message starts with WIP" >&2
  exit 1
fi
EOF
chmod +x .git/hooks/commit-msg

# Try it:
echo content > file && git add file
git commit -m "WIP fix the bug"
# → commit-msg: refused — message starts with WIP

git commit -m "fix: stop the bug"
# → succeeds
```

For a team-wide version, store the hook under `scripts/hooks/commit-msg` in the repo and tell every developer to:

```bash
git config core.hooksPath scripts/hooks
```

Or use a managed runner (pre-commit.com / Husky / Lefthook) so the install step is `npm install` or `pre-commit install`.

---

## Related

- [Tier 5 — Pack format and delta compression (planned)](../INDEX.md#tier-5--packfiles-delta-compression-and-garbage-collection-planned) — what travels inside a `packfile` pkt-line block.
- [Networking — TCP and TLS](../../networking/transport/tcp-deep-dive.md) — what's underneath SSH and HTTPS transports.
- [Security — Supply-chain integrity](../../security/INDEX.md) — signed commits as a control in the SLSA framework.
- [Linux — fds and processes](../../linux/INDEX.md) — `git-upload-pack` is just a process talking pkt-lines on stdio.
- [System Design — content-addressable storage](../../system-design/INDEX.md) — Git's wire protocol is the canonical example of "exchange Merkle DAG diffs efficiently."
- [Observability — structured logs and tracing](../../observability/INDEX.md) — `GIT_TRACE_PACKET` is the same shape of insight as application-level tracing.

## References

- Git documentation: [`gitprotocol-v2`](https://git-scm.com/docs/gitprotocol-v2) — the v2 wire protocol, capability advertisement, `ls-refs`, `fetch`.
- Git documentation: [`gitprotocol-pack`](https://git-scm.com/docs/gitprotocol-pack) — pkt-line framing and v0/v1 negotiation.
- Git documentation: [`gitprotocol-http`](https://git-scm.com/docs/gitprotocol-http) — Smart HTTP transport.
- Git documentation: [`githooks`](https://git-scm.com/docs/githooks) — full list of client/server hooks, arguments, abort semantics.
- Git documentation: [`git-clone`](https://git-scm.com/docs/git-clone) — `--filter`, `--depth`, `--sparse`, `--single-branch`.
- Git documentation: [`git-submodule`](https://git-scm.com/docs/git-submodule) — `.gitmodules`, gitlinks, subcommands.
- Git documentation: [`git-config`](https://git-scm.com/docs/git-config) — `commit.gpgSign`, `tag.gpgSign`, `gpg.format`, `gpg.ssh.allowedSignersFile`, `core.hooksPath`.
- Git documentation: [`git-verify-commit`](https://git-scm.com/docs/git-verify-commit) — signature verification semantics.
- [`sigstore/gitsign`](https://github.com/sigstore/gitsign) — keyless OIDC commit signing via Fulcio + Rekor.
- GitHub Engineering: ["Get up to speed with partial clone and shallow clone"](https://github.blog/open-source/git/get-up-to-speed-with-partial-clone-and-shallow-clone/) — practical narrative tour of the on-demand fetch model.
