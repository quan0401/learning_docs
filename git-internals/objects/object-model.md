---
title: "The Git Object Model — Blobs, Trees, Commits, Tags"
date: 2026-05-07
updated: 2026-05-07
tags: [git, git-internals, objects, sha, plumbing]
---

# The Git Object Model — Blobs, Trees, Commits, Tags

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `git` `git-internals` `objects` `sha` `plumbing`

---

## Table of Contents

- [Summary](#summary)
- [1. The Four Object Types](#1-the-four-object-types)
  - [1.1 Why "content-addressable filesystem" is the right mental model](#11-why-content-addressable-filesystem-is-the-right-mental-model)
  - [1.2 Blob — file contents, nothing else](#12-blob--file-contents-nothing-else)
  - [1.3 Tree — a directory listing](#13-tree--a-directory-listing)
  - [1.4 Commit — a snapshot pointer with metadata](#14-commit--a-snapshot-pointer-with-metadata)
  - [1.5 Tag — a named, optionally-signed pointer](#15-tag--a-named-optionally-signed-pointer)
  - [1.6 The graph in one picture](#16-the-graph-in-one-picture)
- [2. SHA-1, SHA-256, and the Object ID Transition](#2-sha-1-sha-256-and-the-object-id-transition)
  - [2.1 How an object ID is computed](#21-how-an-object-id-is-computed)
  - [2.2 Why SHA-1 — and why it had to go](#22-why-sha-1--and-why-it-had-to-go)
  - [2.3 The SHA-256 transition plan](#23-the-sha-256-transition-plan)
  - [2.4 Today: which hash do you actually have?](#24-today-which-hash-do-you-actually-have)
- [3. The Object Database](#3-the-object-database)
  - [3.1 `.git/objects/` layout](#31-gitobjects-layout)
  - [3.2 Loose object on-disk format](#32-loose-object-on-disk-format)
  - [3.3 Packed objects and delta compression](#33-packed-objects-and-delta-compression)
  - [3.4 When does Git pack?](#34-when-does-git-pack)
- [4. Plumbing Walkthrough — Build a Repo from Scratch](#4-plumbing-walkthrough--build-a-repo-from-scratch)
  - [4.1 The cast of plumbing commands](#41-the-cast-of-plumbing-commands)
  - [4.2 Step 0 — initialize an empty repo](#42-step-0--initialize-an-empty-repo)
  - [4.3 Step 1 — write two blobs with `git hash-object`](#43-step-1--write-two-blobs-with-git-hash-object)
  - [4.4 Step 2 — assemble a tree with `git mktree`](#44-step-2--assemble-a-tree-with-git-mktree)
  - [4.5 Step 3 — create a commit with `git commit-tree`](#45-step-3--create-a-commit-with-git-commit-tree)
  - [4.6 Step 4 — point a branch at it with `git update-ref`](#46-step-4--point-a-branch-at-it-with-git-update-ref)
  - [4.7 Step 5 — sanity check with porcelain](#47-step-5--sanity-check-with-porcelain)
  - [4.8 Bonus — `git write-tree` from the index](#48-bonus--git-write-tree-from-the-index)
- [Related](#related)
- [References](#references)

---

## Summary

Git is a content-addressable filesystem layered with a small DAG: every file body, directory listing, snapshot, and named release is stored as one of four object types — **blob**, **tree**, **commit**, **tag** — keyed by the SHA-1 (or SHA-256) hash of its content plus a tiny header. Once you can read those four record shapes and run the plumbing commands that produce them, "what does Git actually do?" stops being mysterious. This doc dissects the object format, walks the SHA-1 → SHA-256 transition, and rebuilds a one-commit repository using only plumbing — no `git add`, no `git commit`.

## 1. The Four Object Types

### 1.1 Why "content-addressable filesystem" is the right mental model

The Pro Git book opens its internals chapter with the exact framing every backend engineer should latch onto: "Git is a content-addressable filesystem" — a key/value store where the key is the SHA-1 of the value plus a header, and the value is some bytes. Everything else (branches, tags, the index, `git log`, merges) is built on top of that store.

Four record types live in the store:

| Type | Stores | Points to |
|------|--------|-----------|
| `blob` | Raw file bytes (no filename, no metadata) | Nothing |
| `tree` | Directory listing — entries of `(mode, type, hash, name)` | Blobs and other trees |
| `commit` | Snapshot pointer + author + parent(s) + message | One tree, zero or more parent commits |
| `tag` | Annotated tag — name + tagger + message + signature | Any object (usually a commit) |

The objects are immutable. Editing a file produces a new blob with a new hash; that change ripples up to a new tree, then a new commit. History is a Merkle DAG: any tampering anywhere changes the commit hash, which is why `git log` over a healthy repo is integrity-checked end-to-end.

### 1.2 Blob — file contents, nothing else

A blob is *just bytes*. It does not record the filename, the mode, or where the file lives in the tree — those belong to the parent tree object. Two files with identical content (anywhere in the repo, anywhere in history) share one blob.

```bash
$ echo 'hello' | git hash-object --stdin
ce013625030ba8dba906f756967f9e9ca394464a
```

That hash is deterministic across every Git installation on Earth. The same six-byte input always produces `ce0136…` because the SHA-1 is computed over `blob 6\0hello\n`.

### 1.3 Tree — a directory listing

A tree is the analogue of one directory level in a filesystem. Each entry is `<mode> <type> <hash>\t<name>` when pretty-printed; on disk it is a binary record (mode in ASCII, name NUL-terminated, raw 20-byte SHA-1).

The mode is **not** a full POSIX mode — Git stores only a small whitelist:

| Mode | Type | Meaning |
|------|------|---------|
| `100644` | blob | Regular non-executable file |
| `100755` | blob | Regular executable file |
| `120000` | blob | Symbolic link (blob holds the link target) |
| `040000` | tree | Subdirectory |
| `160000` | commit | Submodule (gitlink) |

Read a tree with `git ls-tree` (or `git cat-file -p` if you want the pretty form):

```bash
$ git ls-tree HEAD
100644 blob f8051e05d7cc77b9aaeef09bc92709cfdead7d6d	README.md
040000 tree 4d5fcadc293a348e88f777dc0920f11e7d71441c	src
```

Trees have no notion of "modified time," "owner," or "ACL." That ruthless minimalism is why Git is so portable across operating systems.

### 1.4 Commit — a snapshot pointer with metadata

A commit is the smallest record that ties everything else together. Its body is plain text and trivially readable:

```text
tree c619f9369c7d7fb086a46a4dc566a576f9567aeb
parent 4e64b4d77362b08e0c0f4340fe51d457ab24beb4
author Demo <demo@example.com> 1778170824 +0700
committer Demo <demo@example.com> 1778170824 +0700

Initial commit from plumbing
```

Three things to internalize:

1. **A commit points to one tree** — the *whole snapshot*, not a diff. Diffs are computed on demand by comparing two trees.
2. **A commit has zero, one, or many parents.** Zero = root commit. One = ordinary commit. Two+ = merge commit. The order matters: the first parent is the "mainline."
3. **`author` and `committer` are different roles.** Author = who wrote the change. Committer = who applied it. Rebasing rewrites the committer but keeps the author. That distinction is why `git log --pretty=%an` vs `%cn` exist.

### 1.5 Tag — a named, optionally-signed pointer

There are two flavours of tag, and they are *very* different on disk:

- **Lightweight tag**: not an object at all. It's just a ref (`refs/tags/v1.0`) whose contents is a commit hash. Identical to a branch except branches are mutable and tags are conventionally not.
- **Annotated tag** (created with `git tag -a` or `-s`): a real object with a tagger, message, and optional GPG signature. The ref points at the tag object, which points at the target commit.

Annotated tag body looks like:

```text
object 4e64b4d77362b08e0c0f4340fe51d457ab24beb4
type commit
tag v1.0.0
tagger Demo <demo@example.com> 1778170824 +0700

Release 1.0.0
-----BEGIN PGP SIGNATURE-----
...
-----END PGP SIGNATURE-----
```

Signed commits and signed tags are how Git provides authenticated history without trusting the transport.

### 1.6 The graph in one picture

```text
                  refs/heads/main ──► commit C2 ──► tree T2
                                         │           ├─ blob B1  hello.txt
                                         │           └─ tree T2a
                                         │              └─ blob B3  src/index.ts
                                         │
                                         └─ parent ─► commit C1 ──► tree T1
                                                       │            ├─ blob B1  hello.txt
                                                       │            └─ blob B2  README.md
                  refs/tags/v1.0 ──► tag object T ──► commit C1
```

Every arrow is a SHA pointer. Every node is content-addressed. That's the whole model.

## 2. SHA-1, SHA-256, and the Object ID Transition

### 2.1 How an object ID is computed

Pro Git, chapter 10.2, walks through the algorithm in Ruby. Distilled:

```python
# Pseudocode — exact recipe from the Pro Git book, ch. 10.2
def object_id(obj_type: str, content: bytes) -> str:
    header = f"{obj_type} {len(content)}\0".encode("ascii")
    store  = header + content
    return sha1(store).hexdigest()
```

Three things to notice:

1. The **header is part of the hash input**. That's why a 6-byte blob `hello\n` and a 6-byte commit body with identical bytes hash to different IDs — the header disambiguates.
2. The size is in **bytes**, not characters. Multi-byte UTF-8 counts every byte.
3. The on-disk file is `zlib.deflate(store)` — but the hash is computed on the *uncompressed* `store`, so two different zlib levels produce the same hash but different file sizes.

### 2.2 Why SHA-1 — and why it had to go

SHA-1 was chosen in 2005 for speed and ubiquity. The cryptographic case against it has been building for two decades:

- **2005** — Wang, Yin, and Yu showed an attack finding SHA-1 collisions in fewer than 2⁶⁹ operations.
- **2015** — "The SHAppening" demonstrated a freestart collision around 2⁵⁷ operations.
- **2017-02-23** — **SHAttered**: CWI Amsterdam and Google produced two distinct PDF files with the same SHA-1 hash, the first practical full collision. They quoted "the equivalent processing power of 6,500 years of single-CPU computations and 110 years of single-GPU computations."
- **2020-01-05** — **SHAmbles**: a *chosen-prefix* collision at ~2⁶³·⁴ work, costing roughly $45,000 of cloud compute per collision. Chosen-prefix is the dangerous variant — an attacker can craft two semantically different documents (e.g., a benign and a malicious commit) sharing a hash.

Note that Git's threat model is more forgiving than a digital signature scheme. SHAttered-style identical-prefix collisions don't immediately let you swap a commit in someone else's repo: the colliding object has to *also* parse as a valid Git object, *also* match expected file content, *and* survive transport (Git refuses to replace an object it already has). But "the foundations of our content-addressing scheme are weakening" is reason enough to move.

### 2.3 The SHA-256 transition plan

Git formally selected SHA-256 in late 2018 and ships an experimental implementation. Key design choices:

- **Per-repo opt-in**, not a flag day. Each repository decides independently.
- **Dual naming**: a SHA-256 repo can keep a translation table to SHA-1 names so users and tooling can address objects either way.
- **Interop**: a SHA-256 client can push to / fetch from a SHA-1 server with on-the-fly translation.

Configuration looks like:

```ini
[core]
    repositoryFormatVersion = 1
[extensions]
    objectFormat = sha256
    compatObjectFormat = sha1
```

`repositoryFormatVersion = 1` is a guard rail — any Git old enough not to know about extensions will refuse to read the repo rather than corrupt it.

### 2.4 Today: which hash do you actually have?

In practice, as of writing:

```bash
$ git --version
git version 2.x.y

$ git init demo
$ git -C demo rev-parse --show-object-format
sha1
```

SHA-1 is still the default for `git init`. SHA-256 needs `git init --object-format=sha256`. Most hosting providers (GitHub, GitLab, Gitea) still serve SHA-1 by default; check current support before relying on SHA-256 for a shared repo. The transition is "production-ready in pieces, not yet rolled out to everyone."

The takeaway: understand both, use SHA-1 for everyday work, and don't be surprised when an object ID is 64 hex chars instead of 40.

## 3. The Object Database

### 3.1 `.git/objects/` layout

Pop a fresh repo and look:

```bash
$ git init demo && cd demo
$ ls -F .git/objects
info/  pack/
```

Two subdirectories matter:

- **`info/`** — auxiliary metadata (alternates, packed-refs hints). Mostly empty for local repos.
- **`pack/`** — compressed bundles, see below.

Add a single file and you'll see *loose* objects appear:

```bash
$ echo hi > a.txt && git add a.txt
$ ls .git/objects
45  info  pack
$ ls .git/objects/45
b983e8e5f5d4a0e6e4e6ebe1a2b3c4d5e6f7a8
```

The directory name is the **first 2 hex characters** of the object hash; the filename is the **remaining 38**. The split exists for a single boring reason: many filesystems get slow when one directory holds tens of thousands of files. Sharding by the first byte caps any directory at ~256 entries on average.

### 3.2 Loose object on-disk format

Each loose object file is `zlib.deflate("<type> <size>\0<content>")`. Reproducing it by hand is a great way to convince yourself there is no magic:

```python
# Pseudocode showing what `git hash-object -w` does internally
import hashlib, zlib
content = b"hello\n"
header  = f"blob {len(content)}\0".encode()
store   = header + content
oid     = hashlib.sha1(store).hexdigest()           # e.g. ce013625...
path    = f".git/objects/{oid[:2]}/{oid[2:]}"
open(path, "wb").write(zlib.compress(store))
```

The `<type>` prefix is one of `blob`, `tree`, `commit`, `tag`. The space, the decimal size, and the NUL separator are mandatory.

Read it back without Git:

```python
import zlib
raw = open(".git/objects/ce/013625030ba8dba906f756967f9e9ca394464a", "rb").read()
print(zlib.decompress(raw))   # b'blob 6\x00hello\n'
```

That's the whole storage format for a loose object. No checksum field, no version byte — the SHA itself is the integrity check, because verifying it requires reproducing the hash from the bytes you decompressed.

### 3.3 Packed objects and delta compression

Loose storage is honest but wasteful. A 200 MB repo with 50,000 objects might balloon to gigabytes if each version of every file is a fully-compressed standalone blob. Git's answer is **packfiles**.

```bash
$ ls .git/objects/pack
pack-978e03944f5c581011e6998cd0e9e30000905586.idx
pack-978e03944f5c581011e6998cd0e9e30000905586.pack
```

Two files travel together:

- **`.pack`** — many objects concatenated, with delta compression: similar objects (e.g., two versions of the same file) are stored as a *base object* plus a binary diff. The Pro Git example shows two ~22 KB files stored in ~7 KB total — a ~50% saving.
- **`.idx`** — a sorted index from object ID to byte offset inside the `.pack`, so any object can be located in O(log n) without scanning the whole pack.

Inspect a pack:

```bash
$ git verify-pack -v .git/objects/pack/pack-978e0394...idx
b042a60ef7dff760008df33cee372b945b6e884e blob   22054 5799 1463
033b4468fa6b2a9547a70d88d1bbe8bf3f9ed0d5 blob   9     20   7262 1 b042a60ef7dff760008df33cee372b945b6e884e
```

The second blob `033b4…` is stored as a 9-byte delta against `b042a…`. This is what makes `git clone` of a 10-year-old repo feasible.

### 3.4 When does Git pack?

Three triggers, paraphrased from the Pro Git "Packfiles" chapter:

1. **Automatically**, when too many loose objects accumulate (controlled by `gc.auto`).
2. **Manually**, when you run `git gc` or `git repack`.
3. **On `git push`**, the sender bundles the objects you don't have into a pack and streams it.

The result is that a long-lived repo is mostly packed, with a small head of recently-created loose objects. Both formats live side-by-side; Git looks up an object by trying loose first, then scanning the `.idx` of each pack.

## 4. Plumbing Walkthrough — Build a Repo from Scratch

This section is reproducible. Run it line-by-line and you'll end with a real one-commit Git repo whose first commit was authored by *no porcelain* — just plumbing.

### 4.1 The cast of plumbing commands

| Command | Reads | Writes | Outputs to stdout |
|---------|-------|--------|-------------------|
| `git hash-object [-w] [--stdin]` | A file or stdin | A blob (with `-w`) | The blob's hash |
| `git mktree` | `ls-tree`-formatted lines on stdin | A tree | The tree's hash |
| `git write-tree` | The index | A tree | The tree's hash |
| `git commit-tree <tree> [-p <parent>]` | A tree hash + message | A commit | The commit's hash |
| `git update-ref <ref> <hash>` | A hash | Nothing object-wise; updates a ref file | Nothing |
| `git cat-file -t/-s/-p <hash>` | An object | Nothing | Type / size / pretty body |
| `git ls-tree <tree-ish>` | A tree | Nothing | Tree entries |

### 4.2 Step 0 — initialize an empty repo

```bash
mkdir -p /tmp/git-plumbing && cd /tmp/git-plumbing
rm -rf .git
git init -q -b main
ls .git/objects   # only info/ and pack/, both empty
```

No commits exist. `cat .git/HEAD` shows `ref: refs/heads/main`, but `refs/heads/main` doesn't exist yet.

### 4.3 Step 1 — write two blobs with `git hash-object`

```bash
BLOB1=$(printf 'Hello, plumbing!\n' | git hash-object --stdin -w)
BLOB2=$(printf '# Repo\n'           | git hash-object --stdin -w)

echo "$BLOB1"   # cbbd742a6016d80ed8cde650aec0ba8b3be8555c
echo "$BLOB2"   # f8051e05d7cc77b9aaeef09bc92709cfdead7d6d
```

Two new loose objects exist on disk. Confirm:

```bash
git cat-file -t "$BLOB1"     # blob
git cat-file -s "$BLOB1"     # 17
git cat-file -p "$BLOB1"     # Hello, plumbing!
```

Reproduce the hash by hand to convince yourself nothing is hidden:

```bash
{ printf 'blob 17\0'; printf 'Hello, plumbing!\n'; } | shasum
# cbbd742a6016d80ed8cde650aec0ba8b3be8555c  -
```

Note the size is **17** because the trailing newline counts.

### 4.4 Step 2 — assemble a tree with `git mktree`

`mktree` reads `ls-tree` format from stdin. Tabs separate the hash from the name:

```bash
TREE=$(printf "100644 blob %s\thello.txt\n100644 blob %s\tREADME.md\n" \
       "$BLOB1" "$BLOB2" | git mktree)
echo "$TREE"   # c619f9369c7d7fb086a46a4dc566a576f9567aeb

git cat-file -p "$TREE"
# 100644 blob f8051e05d7cc77b9aaeef09bc92709cfdead7d6d    README.md
# 100644 blob cbbd742a6016d80ed8cde650aec0ba8b3be8555c    hello.txt
```

`mktree` re-sorted the entries — tree entries are sorted by name, and that sort is part of what makes the tree hash deterministic.

### 4.5 Step 3 — create a commit with `git commit-tree`

`commit-tree` takes a tree, optional parents (`-p`), and a message. We pin the timestamps and identity to env vars so the resulting commit hash is reproducible across machines:

```bash
COMMIT=$(echo "Initial commit from plumbing" | \
  GIT_AUTHOR_NAME=Demo    GIT_AUTHOR_EMAIL=demo@example.com    \
  GIT_COMMITTER_NAME=Demo GIT_COMMITTER_EMAIL=demo@example.com \
  GIT_AUTHOR_DATE='2026-05-07T00:00:00+0000' \
  GIT_COMMITTER_DATE='2026-05-07T00:00:00+0000' \
  git commit-tree "$TREE")

echo "$COMMIT"
git cat-file -p "$COMMIT"
# tree c619f9369c7d7fb086a46a4dc566a576f9567aeb
# author Demo <demo@example.com> 1778198400 +0000
# committer Demo <demo@example.com> 1778198400 +0000
#
# Initial commit from plumbing
```

For a child commit you'd pass `-p $COMMIT`. For a merge, pass two `-p` flags.

### 4.6 Step 4 — point a branch at it with `git update-ref`

So far the commit is *unreachable* — it exists in `.git/objects` but no ref points to it, and `git gc` would garbage-collect it. Fix that:

```bash
git update-ref refs/heads/main "$COMMIT"
```

That single command is what makes `main` "exist." It writes one file: `.git/refs/heads/main` containing the commit hash. (Or, if refs are packed, it appends to `.git/packed-refs`.)

### 4.7 Step 5 — sanity check with porcelain

Now that we have a ref, all the porcelain commands work:

```bash
git log --oneline
# 4e64b4d Initial commit from plumbing

git status
# On branch main
# nothing to commit, working tree clean   <-- if you also check out the tree

git ls-tree HEAD
# 100644 blob f8051e05d7cc77b9aaeef09bc92709cfdead7d6d    README.md
# 100644 blob cbbd742a6016d80ed8cde650aec0ba8b3be8555c    hello.txt
```

You built a Git commit without ever touching `git add`, the index, or the working tree.

### 4.8 Bonus — `git write-tree` from the index

The plumbing path the porcelain actually takes is slightly different. `git add` populates the **index** (a binary file at `.git/index`), and `git write-tree` produces a tree from the index instead of from stdin:

```bash
echo 'from porcelain' > c.txt
git update-index --add c.txt   # plumbing equivalent of `git add`
git write-tree
# <hash of a tree containing README.md, hello.txt, and c.txt>
```

`mktree` is what you reach for when you're scripting tree creation from external data. `write-tree` is what `git commit` calls under the hood.

---

If you only remember three things from this doc:

1. There are four object types. Each is keyed by `sha(<type> <size>\0<content>)`.
2. The object database is just `.git/objects/<XX>/<rest>` for loose objects, and packfiles (`.pack` + `.idx`) for the packed majority.
3. `hash-object → mktree → commit-tree → update-ref` is the entire commit pipeline. The porcelain is icing.

## Related

- [System Design — building blocks INDEX](../../system-design/building-blocks/) — Git is the canonical content-addressable store; compare with [object-and-blob-storage.md](../../system-design/building-blocks/object-and-blob-storage.md).
- [Database INDEX](../../database/INDEX.md) — Git's pack format is a content-addressed log; useful to compare against append-only storage engines.
- [Security INDEX](../../security/INDEX.md) — SHAttered/SHAmbles are textbook examples of why hash-function agility matters.

## References

- Pro Git, 2nd ed., ch. 10.2 — *Git Internals: Git Objects*. <https://git-scm.com/book/en/v2/Git-Internals-Git-Objects>
- Pro Git, 2nd ed., ch. 10.4 — *Git Internals: Packfiles*. <https://git-scm.com/book/en/v2/Git-Internals-Packfiles>
- `git-cat-file(1)` official documentation. <https://git-scm.com/docs/git-cat-file>
- `git-hash-object(1)` official documentation. <https://git-scm.com/docs/git-hash-object>
- `git-ls-tree(1)`, `git-mktree(1)`, `git-write-tree(1)`, `git-commit-tree(1)`, `git-update-ref(1)`. <https://git-scm.com/docs>
- *Hash function transition* design doc. <https://git-scm.com/docs/hash-function-transition>
- Stevens, Bursztein, Karpman, Albertini, Markov — *The first collision for full SHA-1* (CWI / Google, 2017). <https://shattered.io/static/shattered.pdf>
- Leurent & Peyrin — *SHA-1 is a Shambles: First Chosen-Prefix Collision on SHA-1* (2020). <https://eprint.iacr.org/2020/014>
