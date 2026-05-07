# Git Internals Documentation Index — Learning Path

A progressive path through how Git actually works under the porcelain commands every backend dev runs daily. Object model first, then refs, then the index, then history rewriting, then packfiles and GC, then the distributed protocol. The goal is to make `git` legible enough that incidents (a corrupted repo, a force-push that ate work, a slow `clone`) become diagnosable rather than mysterious.

Cross-references to the [System Design learning path](../system-design/INDEX.md) (content-addressable storage as a building block — Git is the canonical example), the [Database learning path](../database/INDEX.md) (Merkle trees, immutable storage, B-trees vs Git trees), the [Linux learning path](../linux/INDEX.md) (`.git/` filesystem layout, hooks as scripts), the [Security learning path](../security/INDEX.md) (signed commits, secret leak detection, supply-chain integrity), the [Java learning path](../java/INDEX.md) and [TypeScript learning path](../typescript/INDEX.md) (build/CI workflows that depend on Git semantics), and the [Behavioral Interviews learning path](../behavioral-interviews/INDEX.md) (the "tell me about a time you recovered lost work" story bank) where topics overlap.

**Markers:** **★** = core must-learn (everyday work and incident response). **○** = supporting deep-dive (specialized internals or operator concerns). Internalize all ★ before going deep on ○.

---

## Tier 1 — The Object Model

Blobs, trees, commits, tags. The four object types that compose every repo, and the SHA-1/SHA-256 addressing that makes Git content-addressable.

- ★ [The Git Object Model — Blobs, Trees, Commits, Tags](objects/object-model.md) _(2026-05-07)_
  - Covers: the four object types, SHA-1/SHA-256 transition, `.git/objects/` loose vs packed, plumbing walkthrough (`hash-object`, `cat-file`, `ls-tree`, `mktree`, `commit-tree`, `update-ref`)

---

## Tier 2 — Refs and Reflog

What a branch actually is. Why `HEAD` is a special pointer. How the reflog saves you from yourself.

- ★ [Refs and Reflog — How Branches, Tags, and HEAD Actually Work](refs/refs-and-reflog.md) _(2026-05-07)_
  - Covers: refs as filesystem pointers, symbolic refs and `packed-refs`, reflog mechanics + recovery + expiry, remote-tracking refs and the fetch refspec

---

## Tier 3 — The Index and Working Tree

The three trees model: HEAD, index, working tree. The bit that distinguishes Git from every other VCS the user has touched.

- ★ [The Index and the Three-Tree Model](index/index-and-working-tree.md) _(2026-05-07)_
  - Covers: `.git/index` binary format (DIRC + version), three-tree model and `reset` semantics, three-way merge with `ORIG_HEAD`/`MERGE_HEAD`/`CHERRY_PICK_HEAD`, sparse checkout and `skip-worktree`

---

## Tier 4 — History Rewriting

Rebase, cherry-pick, amend, and the modern `git filter-repo`. When safe, when not, and what the reflog promises.

- ★ [History Rewriting — Rebase, Cherry-Pick, and git filter-repo](rewriting/history-rewriting.md) _(2026-05-07)_
  - Covers: interactive rebase mechanics (squash/fixup/reorder, autosquash), cherry-pick and revert patch-identity, `git filter-repo` over `filter-branch`, secret-removal aftermath (force-push, cached refs, secret rotation)

---

## Tier 5 — Packfiles, Delta Compression, and Garbage Collection

Why `git gc` exists. Pack format, delta chains, multi-pack indexes, and the operational story behind big-repo performance.

- ○ [Packfiles, Delta Compression, and Garbage Collection](packfiles/packfiles-and-gc.md) _(2026-05-07)_
  - Covers: `.pack` and `.idx` v2 layout, delta heuristics (`pack.window`, `pack.depth`), `git gc` triggers and `--aggressive` tradeoff, commit-graph generation numbers, multi-pack-index

---

## Tier 6 — Distributed Protocol and Workflows

How `clone`, `fetch`, `push` actually move bytes. Hooks, partial clone, and the protocols that make GitHub/GitLab fast.

- ○ [The Distributed Protocol — Wire Protocol, Hooks, and Signing](protocol/distributed-protocol.md) _(2026-05-07)_
  - Covers: wire protocols v0/v1/v2 with capability advertisement, smart HTTP and SSH transport, partial/shallow clone (`--filter=blob:none`, `--depth`), client/server hooks, submodules vs subtrees, signed commits (GPG, SSH, sigstore `gitsign`)

---

## Quick Reference by Topic

### Object Model

- [Object Types](objects/object-model.md#1-the-four-object-types) _(2026-05-07)_
- [SHA Transition](objects/object-model.md#2-sha-1-sha-256-and-the-object-id-transition) _(2026-05-07)_
- [Object Database](objects/object-model.md#3-the-object-database) _(2026-05-07)_
- [Plumbing Commands](objects/object-model.md#4-plumbing-walkthrough--build-a-repo-from-scratch) _(2026-05-07)_

### Refs

- [Refs and HEAD](refs/refs-and-reflog.md#1-refs-as-file-system-pointers) _(2026-05-07)_
- [packed-refs](refs/refs-and-reflog.md#2-symbolic-refs-and-packed-refs) _(2026-05-07)_
- [Reflog](refs/refs-and-reflog.md#3-reflog--the-safety-net) _(2026-05-07)_
- [Remote-Tracking Refs](refs/refs-and-reflog.md#4-remote-tracking-refs-and-the-fetch-refspec) _(2026-05-07)_

### Index

- [The Index](index/index-and-working-tree.md#1-the-index--more-than-a-staging-area) _(2026-05-07)_
- [Three Trees](index/index-and-working-tree.md#2-the-three-tree-model) _(2026-05-07)_
- [Merge Conflicts](index/index-and-working-tree.md#3-merge-conflicts-and-the-three-way-merge) _(2026-05-07)_
- [Sparse Checkout](index/index-and-working-tree.md#4-sparse-checkout-and-skip-worktree) _(2026-05-07)_

### History Rewriting

- [Interactive Rebase](rewriting/history-rewriting.md#1-interactive-rebase) _(2026-05-07)_
- [Cherry-Pick and Revert](rewriting/history-rewriting.md#2-cherry-pick-and-revert) _(2026-05-07)_
- [git filter-repo](rewriting/history-rewriting.md#3-git-filter-repo) _(2026-05-07)_
- [History Surgery](rewriting/history-rewriting.md#4-history-surgery-in-production) _(2026-05-07)_

### Packfiles and GC

- [Pack Format](packfiles/packfiles-and-gc.md#1-pack-format) _(2026-05-07)_
- [Delta Compression](packfiles/packfiles-and-gc.md#2-delta-compression) _(2026-05-07)_
- [git gc](packfiles/packfiles-and-gc.md#3-git-gc-and-repack-strategy) _(2026-05-07)_
- [Commit-Graph](packfiles/packfiles-and-gc.md#4-commit-graph-and-multi-pack-index) _(2026-05-07)_

### Distributed Protocol

- [Wire Protocols](protocol/distributed-protocol.md#1-wire-protocols-v0-v1-v2) _(2026-05-07)_
- [Smart HTTP and SSH](protocol/distributed-protocol.md#2-smart-http-and-ssh-transport) _(2026-05-07)_
- [Partial and Shallow Clone](protocol/distributed-protocol.md#3-partial-shallow-and-sparse-clones) _(2026-05-07)_
- [Hooks](protocol/distributed-protocol.md#4-hooks--client-and-server) _(2026-05-07)_
- [Submodules and Subtrees](protocol/distributed-protocol.md#5-submodules-and-subtrees) _(2026-05-07)_
- [Signed Commits](protocol/distributed-protocol.md#6-signed-commits-and-tags) _(2026-05-07)_

---

## Bug Spotting

Active-recall practice docs will land here once the tiers above are populated. Pattern matches the rest of the repo: 22+ broken snippets organized by difficulty (Easy / Subtle / Senior trap), one-line `<details>` hints inline, full root-cause + fix in a Solutions section. Every bug cites a real reference (CVE, git mailing-list thread, GitHub/GitLab incident, official-doc gotcha).

---

## Related Learning Paths

- [System Design — content-addressable storage, Merkle trees](../system-design/INDEX.md) — Git as the canonical example
- [Database — immutable storage, B-trees vs Merkle trees](../database/INDEX.md) — adjacent storage models
- [Linux — `.git/` layout, hooks as shell scripts](../linux/INDEX.md) — the operator's view
- [Security — signed commits, secret scanning](../security/INDEX.md) — supply-chain integrity
- [Behavioral Interviews — incident recovery stories](../behavioral-interviews/INDEX.md) — "tell me about a time you recovered lost work"
