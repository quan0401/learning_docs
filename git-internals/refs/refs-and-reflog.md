---
title: "Refs and Reflog — How Branches, Tags, and HEAD Actually Work"
date: 2026-05-07
updated: 2026-05-07
tags: [git, git-internals, refs, reflog, recovery]
---

# Refs and Reflog — How Branches, Tags, and HEAD Actually Work

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `git` `git-internals` `refs` `reflog` `recovery`

---

## Table of Contents

- [Summary](#summary)
- [1. Refs as File-System Pointers](#1-refs-as-file-system-pointers)
  - [1.1 The mental model](#11-the-mental-model)
  - [1.2 Loose refs on disk](#12-loose-refs-on-disk)
  - [1.3 Branches vs tags vs HEAD](#13-branches-vs-tags-vs-head)
  - [1.4 Why `update-ref` instead of `echo >`](#14-why-update-ref-instead-of-echo-)
- [2. Symbolic Refs and packed-refs](#2-symbolic-refs-and-packed-refs)
  - [2.1 Symbolic refs](#21-symbolic-refs)
  - [2.2 `packed-refs` — when and why](#22-packed-refs--when-and-why)
  - [2.3 File layout and lookup order](#23-file-layout-and-lookup-order)
- [3. Reflog — The Safety Net](#3-reflog--the-safety-net)
  - [3.1 What gets logged, and where](#31-what-gets-logged-and-where)
  - [3.2 `HEAD@{N}` vs `branch@{N}`](#32-headn-vs-branchn)
  - [3.3 Expiry rules](#33-expiry-rules)
  - [3.4 Recovery scenarios](#34-recovery-scenarios)
- [4. Remote-Tracking Refs and the Fetch Refspec](#4-remote-tracking-refs-and-the-fetch-refspec)
  - [4.1 The `refs/remotes/` namespace](#41-the-refsremotes-namespace)
  - [4.2 Reading the default refspec](#42-reading-the-default-refspec)
  - [4.3 The leading `+` and forced updates](#43-the-leading--and-forced-updates)
- [5. Plumbing Walkthrough](#5-plumbing-walkthrough)
  - [5.1 Build a tiny repo with plumbing](#51-build-a-tiny-repo-with-plumbing)
  - [5.2 `git for-each-ref` — list refs programmatically](#52-git-for-each-ref--list-refs-programmatically)
  - [5.3 `git pack-refs` — collapsing loose refs](#53-git-pack-refs--collapsing-loose-refs)
  - [5.4 Reproducible "lost commit" recovery](#54-reproducible-lost-commit-recovery)
- [Related](#related)
- [References](#references)

## Summary

A Git "ref" is not magic. It is a file under `.git/refs/` whose contents are
either a 40-character hex object ID or the string `ref: <other-ref>`. Branches,
tags, and `HEAD` are all the same primitive — they only differ by *namespace*
(`refs/heads/`, `refs/tags/`, top-level) and convention. Once that mental model
clicks, three operational questions become tractable:

1. **Where does Git look first?** Loose file under `.git/refs/`, then
   `.git/packed-refs` (per `gitrepository-layout`).
2. **Why does the file ever change without me typing `git`?** Most porcelain
   commands (`commit`, `merge`, `reset`, `fetch`, `pull`, `rebase`) call
   `git update-ref` internally, which writes both the new value *and* a reflog
   entry to `.git/logs/refs/...`.
3. **What can I get back after a destructive operation?** Anything still
   reachable via the reflog, until `git reflog expire` plus `git gc` finally
   prune it (defaults: 90 days for reachable, 30 days for unreachable, per
   `git-reflog` docs).

This doc walks through the on-disk reality, then drives it directly with
`git update-ref`, `git symbolic-ref`, `git pack-refs`, `git reflog`, and
`git for-each-ref`, ending with a runnable lost-commit recovery.

---

## 1. Refs as File-System Pointers

### 1.1 The mental model

A ref is a *named pointer to an object*, almost always a commit. Pro Git puts
it directly:

> In Git, these simple names are called "references" or "refs"; you can find
> the files that contain those SHA-1 values in the `.git/refs` directory.

That is the whole abstraction. Everything else — fast-forward checks,
remote-tracking branches, tag annotations — is bookkeeping built on top of "a
file containing an object ID."

### 1.2 Loose refs on disk

Inspect a fresh repo:

```bash
git init demo-refs && cd demo-refs
echo "hello" > a.txt
git add a.txt
git -c user.email=q@x -c user.name=q commit -m "first"

ls .git/refs
# heads  tags

cat .git/refs/heads/main
# 5b4e... (40-hex commit OID)
```

The file is plain ASCII: 40 hex characters plus a trailing newline. Per
`gitrepository-layout`:

- `refs/heads/<name>` — "records tip-of-the-tree commit objects of branch
  `<name>`."
- `refs/tags/<name>` — "records any object name (not necessarily a commit
  object, or a tag object that points at a commit object)."
- `refs/remotes/<name>` — "records tip-of-the-tree commit objects of branches
  copied from a remote repository."

Notice the asymmetry: tag refs may point at *any* object kind. A lightweight
tag points directly at a commit; an annotated tag points at a tag object,
which itself points at the commit and carries metadata (author, message, GPG
signature).

### 1.3 Branches vs tags vs HEAD

| Ref kind        | File path                            | Pointer to     | Moves on commit?     |
|-----------------|--------------------------------------|----------------|----------------------|
| Branch          | `.git/refs/heads/<name>`             | commit         | Yes (if checked out) |
| Lightweight tag | `.git/refs/tags/<name>`              | commit         | No                   |
| Annotated tag   | `.git/refs/tags/<name>`              | tag object     | No                   |
| Remote-tracking | `.git/refs/remotes/origin/<name>`    | commit         | Yes, on `fetch`      |
| `HEAD`          | `.git/HEAD`                          | ref or commit  | Yes (the active one) |

`HEAD` is the special case. Per `gitrepository-layout`, it is "a symref… to
the `refs/heads/` namespace describing the currently active branch." When
detached, `HEAD` "can also record a specific commit directly, instead of
being a symref."

### 1.4 Why `update-ref` instead of `echo >`

Pro Git shows you can write a ref by hand:

```bash
echo 1a410efbd13591db07496601ebc7a059dd55cfe9 > .git/refs/heads/master
```

…and immediately recommends *not* doing that:

> However, the safer approach is using `git update-ref`:

`git update-ref` does three things `echo >` does not:

1. Locks the ref (writes `<ref>.lock`, renames atomically) so concurrent
   writers cannot corrupt it.
2. Optionally checks the *old* value (`git update-ref <ref> <new> <old>`),
   giving you a CAS-style "only update if it still points at `<old>`."
3. Writes a reflog entry to `.git/logs/refs/...` — without this, recovery
   tools have nothing to walk.

`update-ref` is what `commit`, `merge`, `reset`, `fetch`, `branch -d`, and
friends call under the hood.

---

## 2. Symbolic Refs and packed-refs

### 2.1 Symbolic refs

A symbolic ref is a regular file whose contents start with `ref:`. The
canonical example is `HEAD`:

```bash
cat .git/HEAD
# ref: refs/heads/main
```

`git symbolic-ref` reads, sets, and deletes symrefs:

```bash
git symbolic-ref HEAD
# refs/heads/main

git symbolic-ref --short HEAD
# main

# Point HEAD at a different branch without touching the working tree.
# (This is what `git checkout` does internally for the ref part.)
git symbolic-ref HEAD refs/heads/feature-x
```

Symbolic refs are not limited to `HEAD`. `refs/remotes/origin/HEAD` is the
classic second example — it usually points at `refs/remotes/origin/main` and
is what makes `git log origin` resolve to the remote's default branch.

You can chain symrefs, but keep them shallow. `git symbolic-ref --no-recurse`
exists precisely to stop after one level of dereferencing.

### 2.2 `packed-refs` — when and why

A repository with thousands of tags creates thousands of files under
`.git/refs/tags/`. That hurts `readdir` and slows ref iteration. `git
pack-refs` collapses them into one file:

> This command is used to solve the storage and performance problem by
> storing the refs in a single file, `$GIT_DIR/packed-refs`. When a ref is
> missing from the traditional `$GIT_DIR/refs` directory hierarchy, it is
> looked up in this file and used if found.

Two flags worth memorizing, straight from the docs:

- **`--all`**: "The command by default packs all tags and refs that are
  already packed, and leaves other refs alone… This option causes all refs to
  be packed as well, with the exception of hidden refs, broken refs, and
  symbolic refs."
- **`--no-prune`**: "The command usually removes loose refs under
  `$GIT_DIR/refs` hierarchy after packing them. This option tells it not to."

`git pack-refs` itself is *not* automatic by default; the docs recommend "to
pack its refs with `--all` once, and occasionally run `git pack-refs`." `git
gc` invokes it as part of its routine, which is why most developers never run
it directly.

### 2.3 File layout and lookup order

```text
# .git/packed-refs (sample)
# pack-refs with: peeled fully-peeled sorted
5b4e2c... refs/heads/main
9f30c1... refs/tags/v1.0.0
abc123... refs/tags/v1.0.1
^def456...
```

A few mechanical details:

- The header comment names the capabilities. `sorted` means entries are
  ordered by ref name; `fully-peeled` means every annotated tag is followed
  by its peeled commit on a `^...` line so consumers do not need to hit the
  object database.
- A `^<oid>` line *peels* the previous tag entry — the previous line is a tag
  object, the `^` line is the commit it ultimately resolves to.
- Lookup order: Git checks the loose file `.git/refs/<...>` first; if absent,
  it scans `packed-refs`. A loose ref always wins, which is how branch
  updates after packing still work — the new tip lands as a loose file even
  while older tags remain packed.

You can see both layers in action:

```bash
git pack-refs --all
ls .git/refs/heads          # may now be empty
cat .git/packed-refs        # contains main and tags
git update-ref refs/heads/main HEAD^   # creates a loose file again
ls .git/refs/heads          # main is back as a loose ref
```

---

## 3. Reflog — The Safety Net

### 3.1 What gets logged, and where

Every time a ref moves through `update-ref` (which is to say, almost always),
Git appends a line to `.git/logs/<ref-path>`. From `gitrepository-layout`:

> `logs/` Records of changes made to refs are stored in this directory.

Inspect one:

```bash
cat .git/logs/HEAD
# 0000000000000000000000000000000000000000 5b4e... q <q@x> 1714665600 +0000  commit (initial): first
# 5b4e... 9f30... q <q@x> 1714665900 +0000  commit: second
```

Format per line: `<old-oid> <new-oid> <committer> <timestamp> <tz>\t<message>`.
`HEAD`'s reflog tracks every position you have been at; each branch's
reflog (e.g. `.git/logs/refs/heads/main`) only tracks updates to that
specific branch.

### 3.2 `HEAD@{N}` vs `branch@{N}`

These selectors look similar and are not. Per `git-reflog`:

- **`HEAD@{N}`** — "where HEAD used to be N moves ago." Counts every reflog
  entry on `HEAD`: commits, checkouts, resets, rebases, merges.
- **`branch@{N}`** — "where a specific branch pointed N moves ago." Counts
  only entries on that branch's reflog.

Concrete consequence: after `git checkout other-branch`, `HEAD@{1}` is
"wherever I was before the checkout," but `main@{1}` is unchanged because the
`main` ref itself did not move.

There is also a time-based form, e.g. `master@{one.week.ago}` — "where master
used to point to one week ago in this local repository." Useful, but this is
*local* state; pushing or cloning does not transmit reflogs.

### 3.3 Expiry rules

The defaults, straight from `git reflog --help`:

- `--expire=<time>`: "Removes entries older than specified time (default:
  90 days from `gc.reflogExpire`)."
- `--expire-unreachable=<time>`: "Removes unreachable entries older than
  specified time (default: 30 days from `gc.reflogExpireUnreachable`)."

Two things worth internalizing:

1. **Reachable** entries (still in some ref's history) are kept longer
   (90 days). **Unreachable** entries — the typical "I just hard-reset and
   the old tip is dangling" case — get the shorter window (30 days).
2. Expiry only removes *reflog entries*. The objects themselves survive
   until `git gc` runs and finds them unreachable from any ref *and* any
   live reflog entry. That is why "lost" commits often come back: the object
   is still on disk, you just need to re-attach a ref to it.

You can adjust per-ref via `gc.reflogExpire.<pattern>` and
`gc.reflogExpireUnreachable.<pattern>`, e.g. set them to `never` for
`refs/heads/*` if you want indefinite undo on branch tips.

### 3.4 Recovery scenarios

| Scenario                            | First place to look                | Tool                                 |
|-------------------------------------|------------------------------------|--------------------------------------|
| `git reset --hard` ate uncommitted… | nothing — reflog only logs commits | none (lost). Use `git stash` next time |
| `git reset --hard` ate commits      | `git reflog` on the branch         | `git update-ref refs/heads/<b> <oid>` |
| Force-pushed over a teammate's tip  | their local reflog (not yours!)    | they run `git reflog show <branch>`   |
| Deleted branch                      | `HEAD`'s reflog (entries before delete) | `git branch <name> <oid>`        |
| Detached HEAD then checked out away | `HEAD@{N}`                         | `git branch rescue HEAD@{N}`          |

---

## 4. Remote-Tracking Refs and the Fetch Refspec

### 4.1 The `refs/remotes/` namespace

After `git clone`, your local `refs/heads/` contains your branches and
`refs/remotes/origin/` contains *snapshots of the remote's branches as of
your last fetch.* They are ordinary refs — files under `.git/refs/remotes/`
or rows in `packed-refs`. Nothing on the remote is touched by reading them;
that is just a directory in your own `.git`.

```bash
ls .git/refs/remotes/origin
# HEAD  main  feature-x

cat .git/refs/remotes/origin/HEAD
# ref: refs/remotes/origin/main
```

### 4.2 Reading the default refspec

A fresh clone configures, in `.git/config`:

```text
[remote "origin"]
    url = git@github.com:user/repo.git
    fetch = +refs/heads/*:refs/remotes/origin/*
```

That last line is the *fetch refspec*. The general shape, per `git-fetch`,
is `[+]<src>:<dst>`. Decoded:

- `<src>` = `refs/heads/*` — the pattern to read on the *remote*. The
  asterisk is a glob; for any ref on the remote that matches, Git captures
  whatever `*` matched.
- `<dst>` = `refs/remotes/origin/*` — the pattern to write *locally*, with
  `*` substituted by the captured value.

So a remote branch `refs/heads/feature/login` lands locally at
`refs/remotes/origin/feature/login`. The slash in the captured portion is
fine; the glob is a single replacement, not shell-style.

A pattern refspec must have exactly one `*` on each side. From the docs:

> A pattern \<refspec\> must have one and only one `*` in both the \<src\>
> and \<dst\>. It will map refs to the destination by replacing the `*` with
> the contents matched from the source.

### 4.3 The leading `+` and forced updates

The `+` matters. Without it, `git fetch` refuses to overwrite a ref with a
non-fast-forward update. With it, the fetch proceeds anyway — which is
what you want for remote-tracking refs, because the remote can legitimately
rebase or force-push, and your `refs/remotes/origin/<b>` is just a mirror.
From `git-fetch`:

> all of the rules described above about what's not allowed as an update can
> be overridden by adding an optional leading `+` to a refspec (or using the
> `--force` command line option).

A practical example:

```bash
# Fetch only one branch, mapping it to a custom local name.
git fetch origin +refs/heads/release-2026:refs/remotes/origin/release-2026

# Mirror tags too (they live in their own namespace).
git fetch origin '+refs/tags/*:refs/tags/*'
```

The second form is interesting: `<dst>` is `refs/tags/*`, not
`refs/remotes/...`. Tags are not normally namespaced per-remote, which is
why two remotes producing tags with the same name will clobber each other
unless you rewrite the destination.

---

## 5. Plumbing Walkthrough

Everything below is reproducible. Run each block in order in a clean
directory.

### 5.1 Build a tiny repo with plumbing

```bash
mkdir refs-demo && cd refs-demo
git init -q

# Configure for reproducibility.
git config user.email q@x
git config user.name q

# Three commits on main.
echo a > a.txt && git add a.txt && git commit -q -m "A"
echo b > b.txt && git add b.txt && git commit -q -m "B"
echo c > c.txt && git add c.txt && git commit -q -m "C"

# Create a branch by writing the ref directly via plumbing.
TIP=$(git rev-parse HEAD)
git update-ref refs/heads/feature "$TIP"

# Inspect both refs.
cat .git/refs/heads/main
cat .git/refs/heads/feature
# Same OID — feature was created at main's tip.

# HEAD is a symref.
cat .git/HEAD
# ref: refs/heads/main

# Move HEAD to feature without touching the working tree.
git symbolic-ref HEAD refs/heads/feature
cat .git/HEAD
# ref: refs/heads/feature
```

### 5.2 `git for-each-ref` — list refs programmatically

`git for-each-ref` is the scriptable, format-controllable version of
`git branch -l` and `git tag -l`. It hits both loose and packed refs.

```bash
git for-each-ref --format='%(refname) %(objectname) %(objecttype)'
# refs/heads/feature 9f30... commit
# refs/heads/main    9f30... commit

# Just branches, sorted by latest commit.
git for-each-ref --sort=-committerdate refs/heads/ \
    --format='%(refname:short) %(committerdate:short)'

# All refs that point at a specific commit.
git for-each-ref --points-at "$TIP"
```

### 5.3 `git pack-refs` — collapsing loose refs

```bash
ls .git/refs/heads
# feature  main

git pack-refs --all

ls .git/refs/heads
# (often empty — refs are now in packed-refs)

cat .git/packed-refs
# # pack-refs with: peeled fully-peeled sorted
# 9f30... refs/heads/feature
# 9f30... refs/heads/main

# Update a packed ref. Git writes a new loose file; packed-refs is unchanged
# until the next pack.
echo d > d.txt && git add d.txt && git commit -q -m "D"
ls .git/refs/heads
# main          <-- new loose file shadows the packed entry
```

### 5.4 Reproducible "lost commit" recovery

Now the headline scenario. We hard-reset over real work and recover it.

```bash
# Set up: two commits we care about, then a "bad" reset.
git checkout -q main
echo important1 > w1.txt && git add w1.txt && git commit -q -m "important work 1"
echo important2 > w2.txt && git add w2.txt && git commit -q -m "important work 2"
LOST=$(git rev-parse HEAD)

# Disaster.
git reset --hard HEAD~2
git log --oneline
# (no sign of "important work")

# Step 1: reflog shows we have not actually lost anything.
git reflog show main
# 5b4e... main@{0}: reset: moving to HEAD~2
# 9f30... main@{1}: commit: important work 2
# 8a21... main@{2}: commit: important work 1
# ...

# Step 2: confirm the "lost" commit is still in the object store.
git cat-file -t "$LOST"
# commit
git cat-file -p "$LOST" | head -3
# tree ...
# parent ...
# author ...

# Step 3: re-attach. update-ref takes raw OIDs; no porcelain needed.
git update-ref refs/heads/rescue "$LOST"
git log --oneline rescue
# 9f30... important work 2
# 8a21... important work 1
# ...

# Step 4: optional — fast-forward main back if you want.
git update-ref refs/heads/main "$LOST" $(git rev-parse main)
git log --oneline main
```

A few things this exercise reveals:

- The reflog stayed intact even though `reset --hard` blew away the
  working-tree state. That is because `reset` calls `update-ref`, which
  appends to the reflog *before* moving the pointer.
- `git update-ref refs/heads/main <new> <old>` is the safe form: if `main`
  is no longer at `<old>` (someone else moved it), the update aborts. This
  is the CAS primitive that powers concurrent ref updates.
- Recovery does not require `git reset` or `git checkout`. A new branch
  pointing at the rescued OID is enough; the porcelain commands are
  conveniences over `update-ref` and `symbolic-ref`.

You can also experiment with deleting a branch and recovering from `HEAD`'s
reflog (which records every `HEAD` movement, including the position before
you switched away):

```bash
git checkout -q -b throwaway
echo x > x.txt && git add x.txt && git commit -q -m "throwaway work"
THROW=$(git rev-parse HEAD)
git checkout -q main
git branch -D throwaway

# The branch ref is gone; the commit is not.
git reflog show HEAD | head
# ... HEAD@{1}: checkout: moving from throwaway to main
# ... HEAD@{2}: commit: throwaway work
git update-ref refs/heads/throwaway "$THROW"
git log --oneline throwaway
```

Finally, to feel the expiry rules with your own eyes — without waiting 30
days — run a dry-run expire with an aggressive window:

```bash
git reflog expire --expire-unreachable=now --dry-run --all
# prints which entries would be pruned
```

`--dry-run` is your friend here. The real command without it, plus a
follow-up `git gc --prune=now`, is what eventually loses the commit for
good.

---

## Related

- [`../objects-and-storage/objects-and-storage.md`](../objects-and-storage/objects-and-storage.md) — the object database that refs point into _(if/when written; not yet present)_

(No other sibling docs in `git-internals/` exist at this writing; cross-path
links will be added as adjacent tiers populate.)

## References

- Git, *git-update-ref*. <https://git-scm.com/docs/git-update-ref>
- Git, *git-symbolic-ref*. <https://git-scm.com/docs/git-symbolic-ref>
- Git, *git-pack-refs*. <https://git-scm.com/docs/git-pack-refs>
- Git, *git-reflog*. <https://git-scm.com/docs/git-reflog>
- Git, *git-for-each-ref*. <https://git-scm.com/docs/git-for-each-ref>
- Git, *git-fetch* (refspec syntax). <https://git-scm.com/docs/git-fetch>
- Git, *gitrepository-layout*. <https://git-scm.com/docs/gitrepository-layout>
- Chacon & Straub, *Pro Git*, 2nd ed., §10.3 "Git References."
  <https://git-scm.com/book/en/v2/Git-Internals-Git-References>
