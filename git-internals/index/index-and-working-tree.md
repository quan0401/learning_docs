---
title: "The Index and the Three-Tree Model"
date: 2026-05-07
updated: 2026-05-07
tags: [git, git-internals, index, merge, sparse-checkout]
---

# The Index and the Three-Tree Model

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `git` `git-internals` `index` `merge` `sparse-checkout`

---

## Table of Contents

1. The Index — More Than a Staging Area
2. The Three-Tree Model
3. Merge Conflicts and the Three-Way Merge
4. Sparse Checkout and skip-worktree
5. Plumbing Walkthrough

---

## Summary

Tier 2 covered Git's object model: blobs, trees, commits, and tags as content-addressed snapshots in `.git/objects/`. This tier covers the missing piece that turns those snapshots into a working development tool: the **index** (also called the **cache** or **staging area**).

The index is a single binary file at `.git/index`. It is not just a list of paths you have run `git add` on — it is a fully realized tree, ready to be hashed into a real tree object the moment you commit. Every Git porcelain command (`status`, `diff`, `commit`, `checkout`, `merge`, `reset`) is fundamentally a manipulation of the relationship between three trees: **HEAD**, the **index**, and the **working tree**.

Once you internalize the three-tree model, the rest of Git stops feeling like a grab-bag of half-overlapping commands. `git add` writes blobs to the object database and updates the index. `git commit` is literally `write-tree | commit-tree | update-ref` chained together. `git reset` is just "move HEAD plus optionally also update the index and working tree." Merge conflicts become "the index has stages 1, 2, and 3 for the same path."

This doc starts with the binary layout of `.git/index`, walks through every command in the three-tree dance, then drops into merge stages, sparse-checkout, and finally rebuilds `git commit` from plumbing primitives.

---

## 1. The Index — More Than a Staging Area

### 1.1 What the index actually is

Most introductions describe the index as "the staging area: a list of changes you intend to commit." That's true but underspecifies it. The index is:

- A flat list of every tracked file in your working tree (not just changed ones).
- For each file: the mode, blob SHA, stage number, path, and a cache of `stat()` data so Git can answer "is this file dirty?" without re-hashing.
- A **complete tree** in serialized form. The instant the index is fully merged, `git write-tree` can hash it into a real tree object with no extra input.

A freshly cloned repo with 5,000 tracked files has 5,000 index entries — the index is not "things waiting to be committed", it is the proposed next commit, pre-computed.

### 1.2 The binary header

The index is a binary file. From `gitformat-index` (the official spec):

```text
Header (12 bytes total):
  4 bytes  signature      'D' 'I' 'R' 'C'  (0x44 0x49 0x52 0x43, "dircache")
  4 bytes  version        big-endian uint32, one of {2, 3, 4}
  4 bytes  entry count    big-endian uint32, number of index entries
```

After the header come the entries themselves (sorted by path), then optional **extensions** (cache tree, untracked cache, sparse directory, etc.), then a trailing SHA-1 (or SHA-256) of everything before it.

You can verify this on any repo:

```bash
$ xxd -l 12 .git/index
00000000: 4449 5243 0000 0002 0000 0042  DIRC.......B
#         D I R C  ver = 2     entries = 0x42 = 66
```

`DIRC` is short for "directory cache" — the original name for the index. Some Git internals docs and source files still use `cache` and `index` interchangeably (`git update-index`, but `git ls-files --cached`).

### 1.3 What an entry looks like

Each entry is a packed C struct: ctime, mtime, dev, ino, mode, uid, gid, size, the 20-byte (SHA-1) or 32-byte (SHA-256) blob ID, flags (which include the **stage number** and a name length), then the path, then NUL-padding to align to 8 bytes. The `stat()` cache is what lets `git status` be fast: Git compares the cached `mtime`/`size`/`ino` against the current `lstat()` and only re-hashes the file if they disagree.

A consequence: if a backup tool or `tar` extraction stomps `mtime` without changing content, `git status` may suddenly think every file is dirty. The fix is `git update-index --refresh`, which re-hashes mismatched entries and updates the cached stat back to clean.

### 1.4 `git add -p` — the patch interface

`git add` writes blobs and updates the index. `git add -p` (patch mode) makes it interactive: Git shows each hunk and asks `Stage this hunk [y,n,q,a,d,e,?]?`. The `e` choice opens an editor where you can hand-edit the diff before staging, which is the only sane way to commit half of a file.

The mechanism underneath: Git calls `git diff` between the working tree and the index, splits the patch into hunks, and for each accepted hunk it applies the patch to a temporary blob, writes that blob, and updates the index entry — all without touching the working tree. The working tree still has all the unstaged changes.

```bash
$ git add -p src/auth.ts
@@ -10,4 +10,8 @@ export function login(user: string) {
+  // TODO: rate-limit this
+  if (failed > 5) throw new Error("locked");
   return token(user);
 }
Stage this hunk [y,n,q,a,d,j,J,g,/,e,?]? y
```

After answering `y`, the index has the new logic but `src/auth.ts` on disk still has whatever else you typed. `git diff` shows the unstaged remainder; `git diff --cached` shows what you just staged.

### 1.5 The assume-unchanged and skip-worktree bits

Two flag bits per entry change Git's behaviour:

- **assume-unchanged** — "Trust me, I haven't touched this; don't bother `lstat()`-ing it." Originally for slow filesystems. Set with `git update-index --assume-unchanged <path>`. Git is allowed to clobber the bit if it needs to.
- **skip-worktree** — "This entry exists in the index but the working tree copy is intentionally absent or out-of-date. Don't materialize it on checkout, don't treat absence as deletion." This is the bit sparse-checkout uses (see §4).

Per `git-update-index`, the difference is:

> assume-unchanged: file exists locally; Git omits checking it. skip-worktree: file may not exist; Git avoids updating/writing it and doesn't treat absence as intentional deletion.

People reach for `--assume-unchanged` to "ignore changes to a tracked config file" and then get surprised when a `git pull` later flips the bit off. The right tool for that case is usually `--skip-worktree`, but the idiomatic answer is "don't track the config file; use a template."

---

## 2. The Three-Tree Model

### 2.1 The three trees

Git operates on three trees simultaneously (Pro Git §7.7 *Reset Demystified*):

| Tree | What it is | Where it lives |
|---|---|---|
| **HEAD** | Last commit's snapshot | A commit object pointed to by `HEAD` |
| **Index** | The proposed next commit | `.git/index` (binary file) |
| **Working tree** | Files you can edit | Your filesystem |

Every tracked file exists in all three. Most of the time they agree. The interesting Git commands move data between them.

### 2.2 Reading state: status and diff

`git status` answers two questions by comparing pairs of trees:

- **Index vs HEAD** → "Changes to be committed" (staged).
- **Working tree vs Index** → "Changes not staged for commit" (unstaged).
- Files in working tree but not in index → "Untracked".

`git diff` shows working-tree-vs-index. `git diff --cached` (or `--staged`) shows index-vs-HEAD. `git diff HEAD` shows working-tree-vs-HEAD (the union of the other two).

```text
                         git diff --cached
                       (HEAD ←→ Index)
   ┌──────┐         ┌───────┐         ┌──────────────┐
   │ HEAD │ ←─────→ │ Index │ ←─────→ │ Working tree │
   └──────┘         └───────┘         └──────────────┘
                                         git diff
                                     (Index ←→ WT)
                ←──────  git diff HEAD  ──────→
```

### 2.3 Moving things into the index

- `git add <path>` — hash working-tree file → write blob → update index entry.
- `git rm <path>` — remove from index *and* working tree.
- `git rm --cached <path>` — remove from index only (keep on disk, mark untracked).
- `git mv a b` — combination of `rm` + `add` against the index.

### 2.4 Moving things out of the index — `git reset`

`git reset` does up to three things, in order, stopping based on the flag (Pro Git §7.7):

| Command | Move HEAD? | Update index? | Update working tree? |
|---|---|---|---|
| `git reset --soft <commit>` | yes | no | no |
| `git reset --mixed <commit>` (default) | yes | yes | no |
| `git reset --hard <commit>` | yes | yes | yes |

So:

- `git reset --soft HEAD~` — undo the last commit, keep everything staged. The index and working tree don't move; only the branch ref moves back one commit.
- `git reset HEAD~` (mixed) — undo the last commit, unstage the changes (they're still on disk).
- `git reset --hard HEAD~` — undo the last commit and discard all the changes. Destructive; the working tree is overwritten.

Path-form `git reset <commit> -- <path>` does *not* move HEAD. It just copies that path's entry from `<commit>` into the index. So `git reset HEAD -- file.ts` is the canonical "unstage this file" — copy HEAD's version of `file.ts` over the index entry. The working tree is untouched.

### 2.5 `git checkout` vs `git reset`

The two commands look similar but differ in what they move:

- `git reset <branch>` moves *the branch HEAD points to* and (per `--mixed`) the index.
- `git checkout <branch>` moves *HEAD itself* to point at the new branch; the previous branch ref is unchanged.

Per the Pro Git cheat sheet:

| | HEAD | Index | Workdir | Safe? |
|---|---|---|---|---|
| `reset --soft`  | REF  | no  | no  | yes |
| `reset` (mixed) | REF  | yes | no  | yes |
| `reset --hard`  | REF  | yes | yes | NO |
| `checkout <commit>` | HEAD | yes | yes | yes (refuses on conflict) |
| `reset <commit> <paths>` | no | yes | no | yes |
| `checkout <commit> <paths>` | no | yes | yes | NO |

`git checkout` does a trivial merge that aborts if your local changes would be lost. `git reset --hard` does not — it overwrites unconditionally. That's the asymmetry that makes `--hard` famous for eating uncommitted work.

Modern Git (2.23+) split `git checkout` into `git switch` (move branches) and `git restore` (restore files), because the original was overloaded. The mechanics are the same.

---

## 3. Merge Conflicts and the Three-Way Merge

### 3.1 Three-way merge intuition

A two-way merge ("yours" vs "theirs") is undecidable: if a line differs, you can't tell who changed it. A three-way merge adds the **merge base** — the most recent commit reachable from both sides. With three points, every change can be attributed to exactly one side, and only changes on *both* sides at the same hunk become conflicts.

```text
        merge base (O)
        /       \
   ours (A)   theirs (B)
        \       /
       merge result
```

For each hunk:
- O = A = B → no change, take A.
- O = A, B differs → take B (only theirs changed it).
- O = B, A differs → take A (only ours changed it).
- A = B → take either (both made the same change).
- All three differ → **conflict**, leave for human.

### 3.2 The index during a conflict

When `git merge` hits a conflict, the index gains **higher stages** for conflicted paths (Pro Git §7.8, `git-read-tree(1)`):

- **Stage 0** — merged (normal).
- **Stage 1** — common ancestor (base).
- **Stage 2** — ours (HEAD).
- **Stage 3** — theirs (incoming).

```bash
$ git ls-files --stage
100644 c3f7d8e... 1  src/config.ts   # base
100644 9a1b2c0... 2  src/config.ts   # ours
100644 5d4e6f8... 3  src/config.ts   # theirs
100644 0123456... 0  src/other.ts    # cleanly merged
```

`git write-tree` refuses to run while any stage > 0 exists. That's how Git enforces "you must resolve before you commit." Resolution means: edit the working-tree file, `git add` it (which writes a stage-0 entry and removes stages 1/2/3), then `git commit`.

### 3.3 Conflict markers

When Git can't auto-merge a hunk it writes both versions into the working-tree file with markers:

```text
<<<<<<< HEAD
const TIMEOUT = 5_000;
=======
const TIMEOUT = 30_000;
>>>>>>> feature/long-timeout
```

Set `merge.conflictStyle = diff3` (or `zdiff3` in Git 2.35+) and Git includes the base too:

```text
<<<<<<< HEAD
const TIMEOUT = 5_000;
||||||| merged common ancestors
const TIMEOUT = 10_000;
=======
const TIMEOUT = 30_000;
>>>>>>> feature/long-timeout
```

`zdiff3` is the same idea but with common leading/trailing context hoisted out of the conflict, producing smaller markers. The base hunk is enormously useful: it tells you *what each side started from*, which is the missing piece for understanding intent.

### 3.4 The merge-state refs

While a merge is in progress, Git writes a few extra refs into `.git/`:

- **`ORIG_HEAD`** — where HEAD pointed before the operation. Set by `merge`, `rebase`, `reset`, `pull`. Recovery: `git reset --hard ORIG_HEAD`.
- **`MERGE_HEAD`** — the commit being merged in. Exists only while a merge is in progress; `git commit` consumes it to make the second parent.
- **`CHERRY_PICK_HEAD`** — same idea for `git cherry-pick`. While set, `git commit` records authorship from the picked commit.
- **`REVERT_HEAD`** — same for `git revert`.

Plus `MERGE_MSG` (proposed commit message), `AUTO_MERGE` (Git 2.38+, the auto-merged tree), and the entry stages described above. `git merge --abort` undoes the merge by checking out `ORIG_HEAD` and clearing all of these.

### 3.5 `git rerere` — reuse recorded resolution

Long-lived feature branches that get rebased onto `main` repeatedly hit the same conflicts over and over. `git rerere` ("reuse recorded resolution") records how you resolved a conflict the first time, then replays it automatically next time the same conflict appears.

Enable globally:

```bash
git config --global rerere.enabled true
```

Once on, `git merge`, `git rebase`, and `git cherry-pick` automatically:

1. **Record** — when you commit a merge resolution, Git stores `(conflicted state, your resolution)` in `.git/rr-cache/<hash>/`.
2. **Replay** — when the same conflicted state appears again, Git does a three-way merge between (the earlier conflicted state, your earlier resolution, the current conflicted state). If it resolves cleanly, the resolution lands in the working tree automatically.

You still have to `git add` and commit — rerere doesn't auto-commit. But it removes the tedium of replaying the same hand-resolution across a dozen rebases. Combine with `git rerere status` and `git rerere diff` to inspect what's queued.

---

## 4. Sparse Checkout and skip-worktree

### 4.1 The problem

Monorepos can have hundreds of thousands of files. Cloning the repo and checking out the whole tree is wasteful when you only ever touch one service. Sparse checkout solves the "I want fewer files in my working tree" problem; partial clone (Tier 7) solves the related "I want fewer objects in `.git/objects/`" problem. They are independent and complementary.

### 4.2 How it works

The mechanism is the **skip-worktree** bit on each index entry. The index still lists every file in the repo. Git just refuses to materialize ones whose entry has skip-worktree set.

The list of paths to keep lives in `.git/info/sparse-checkout`. The config flag `core.sparseCheckout = true` switches the feature on. `core.sparseCheckoutCone = true` enables the simpler cone mode.

### 4.3 Cone mode vs non-cone mode

**Non-cone mode** is the original. The patterns file uses gitignore-style globs:

```text
/*
!/*/
/services/billing/
/lib/
```

This included-files-from-the-root, exclude-subdirs, include-billing approach lets you do anything but is, per `git-sparse-checkout(1)`, "slower (O(N*M)) and has confusing semantics". Discouraged.

**Cone mode** (default since 2.27, recommended) restricts the patterns to whole directories. You list directories; Git includes:

- All files at the repo root.
- All files in any directory on the path *down to* a listed directory.
- All files at any depth *under* a listed directory.

Internally, cone-mode patterns produce structurally simpler globs that Git matches in O(N).

### 4.4 The `git sparse-checkout` command

```bash
# Turn on cone-mode sparse checkout for a few subtrees.
git sparse-checkout set services/billing services/auth lib

# Add another path to the existing set.
git sparse-checkout add services/notifications

# What's currently included?
git sparse-checkout list

# Re-apply rules (after a merge brought in unwanted files).
git sparse-checkout reapply

# Restore the full working tree.
git sparse-checkout disable
```

Subcommand summary (per `git-sparse-checkout(1)`):

| Subcommand | Purpose |
|---|---|
| `set` | Enable sparse-checkout and replace the directory/pattern list. Updates the working tree. |
| `add` | Append to the existing list. |
| `list` | Print current entries. |
| `reapply` | Re-apply rules after a merge/rebase that may have materialized excluded files. |
| `disable` | Turn sparse-checkout off; restore all tracked files. |
| `init` | Deprecated alias for `set` with no paths. |

### 4.5 The sparse index

Sparse-checkout only narrows the *working tree*. The index still contains every file. In a monorepo with 500k files, every `git status` walks 500k entries. The **sparse index** (Git 2.33+, opt-in via `git sparse-checkout init --sparse-index` or `set --sparse-index`) compresses ranges of skip-worktree entries into a single tree-typed entry, shrinking the index from 500k entries to a few hundred. This is what makes sparse-checkout actually fast on giant repos.

---

## 5. Plumbing Walkthrough

The whole three-tree model can be driven directly by plumbing. This section rebuilds porcelain operations from primitives so the abstractions stop being magic.

### 5.1 Inspecting the index — `git ls-files --stage`

```bash
$ git ls-files --stage
100644 8a3c2d4f... 0       README.md
100644 1f2e3d4c... 0       src/main.ts
100755 9a8b7c6d... 0       scripts/build.sh
```

Output columns (per `git-ls-files(1)`): `<mode> <object-sha> <stage> <path>`. Mode `100644` is a regular file, `100755` is executable, `120000` is a symlink. Stage `0` means cleanly merged. During a conflict you'll see the same path repeated at stages 1, 2, 3 (see §3.2).

### 5.2 Building the index by hand

Start with an empty repo and put a file into the index without using `git add`:

```bash
$ git init demo && cd demo

$ echo "hello plumbing" > greet.txt

# Hash the file's contents into a blob; -w writes it to .git/objects.
$ blob=$(git hash-object -w greet.txt)
$ echo "$blob"
ce013625030ba8dba906f756967f9e9ca394464a

# Insert that blob into the index at path "greet.txt", mode 100644.
$ git update-index --add --cacheinfo 100644,$blob,greet.txt

$ git ls-files --stage
100644 ce013625030ba8dba906f756967f9e9ca394464a 0       greet.txt
```

`--cacheinfo` is the low-level "put exactly this entry into the index" knob. `git add` does this for you, plus the `hash-object -w` step.

### 5.3 Index → tree object: `git write-tree`

```bash
$ tree=$(git write-tree)
$ echo "$tree"
1cf42c4afe34a08a96d57a36a8fff1d2d9f8b6e2

$ git cat-file -p "$tree"
100644 blob ce013625030ba8dba906f756967f9e9ca394464a    greet.txt
```

`git write-tree` walks the index entries and produces a real tree object representing them. It refuses if the index has unmerged stages.

### 5.4 Tree → commit: `git commit-tree`

```bash
$ commit=$(echo "first commit" | git commit-tree "$tree")
$ echo "$commit"
4d7c5e6e1b2a3c4d5e6f78901234567890abcdef

$ git cat-file -p "$commit"
tree 1cf42c4afe34a08a96d57a36a8fff1d2d9f8b6e2
author You <you@example.com> 1714000000 +0000
committer You <you@example.com> 1714000000 +0000

first commit
```

`commit-tree` takes a tree and zero-or-more `-p <parent>` flags and a message on stdin, and writes a commit object pointing at that tree. It does **not** move any branch.

### 5.5 Updating the branch: `git update-ref`

```bash
$ git update-ref refs/heads/main "$commit"
$ git symbolic-ref HEAD refs/heads/main   # only needed first time
$ git log --oneline
4d7c5e6 first commit
```

`update-ref` sets a ref to a SHA. `symbolic-ref HEAD <ref>` makes HEAD point at a branch (an attached HEAD). After this you have a real, log-visible commit, built without ever calling `git add` or `git commit`.

### 5.6 `git commit` is just three plumbing calls

Putting §5.3–§5.5 together:

```bash
# What `git commit -m "msg"` is doing under the hood.
tree=$(git write-tree)
commit=$(echo "msg" | git commit-tree "$tree" -p HEAD)
git update-ref HEAD "$commit"
```

Read that twice. `git commit` runs `git write-tree` over the current index, calls `git commit-tree` with the current HEAD as parent, and `update-ref`s the current branch to the new commit. There is no separate "commit storage" — the index *was* the next commit, all along.

### 5.7 Reading a tree back into the index — `git read-tree`

The reverse direction. `git read-tree <tree>` overwrites the index with the contents of a tree, but **does not touch the working tree**:

```bash
$ git read-tree HEAD~     # index now matches the previous commit
$ git status
# all files staged for "deletion" or "modification" relative to current HEAD
```

To bring the working tree along, follow it with `git checkout-index`:

```bash
$ git checkout-index -a -f
```

`-a` means "all", `-f` means "overwrite existing files". Together, `read-tree + checkout-index` is what `git checkout <commit>` does internally.

### 5.8 Three-way merge via `read-tree -m`

`git read-tree` accepts up to three trees with `-m` (per `git-read-tree(1)`):

```bash
$ git read-tree -m <base> <ours> <theirs>
```

Git fills the index with stages 1/2/3 for paths that differ, auto-collapsing the trivial cases (where two of the three agree) to stage 0. Anything left at stages 1/2/3 is a conflict for you to resolve. This is exactly the merge state described in §3.2 — porcelain `git merge` is `read-tree -m` plus working-tree updates plus running merge drivers on conflicting blobs.

### 5.9 Putting it all together — a manual cherry-pick

```bash
# Cherry-pick the changes from <commit> on top of HEAD, by hand.

target=$(git rev-parse <commit>)
parent=$(git rev-parse "$target^")

# Three-way merge: base = parent of cherry, ours = HEAD, theirs = cherry.
git read-tree -m "$parent" HEAD "$target"
git checkout-index -a -f             # propagate to working tree

# (Resolve any conflicts here, then `git add` to collapse to stage 0.)

tree=$(git write-tree)
commit=$(git cat-file -p "$target" \
  | sed -n 's/^author //p; s/^.*//p' >/dev/null   # (real script reuses author)
  ; git log -1 --format='%B' "$target" \
  | git commit-tree "$tree" -p HEAD)
git update-ref HEAD "$commit"
```

Compare with `git cherry-pick <commit>`. Same primitives, same result.

---

## Related

- [git-internals/objects/blobs-trees-commits.md](../objects/blobs-trees-commits.md) — Tier 2 covers what tree and commit objects look like; this doc shows how `write-tree`/`commit-tree` build them from the index.
- [git-internals/refs/refs-and-head.md](../refs/refs-and-head.md) — `update-ref`, HEAD, branches, and the symbolic-ref dance referenced in §5.5.
- [low-level-design/foundations/composition-vs-inheritance.md](../../low-level-design/foundations/composition-vs-inheritance.md) — the index-as-staged-tree is a clean example of representing intent as data instead of process state.
- [operating-systems/syscalls/syscalls-fundamentals.md](../../operating-systems/syscalls/syscalls-fundamentals.md) — the `stat()` cache in §1.3 is why understanding `lstat()` matters for repo performance.

## References

- Pro Git, 2nd ed., §7.7 *Reset Demystified* — https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified
- `gitformat-index` (binary index file format) — https://git-scm.com/docs/index-format
- `git-update-index(1)` — https://git-scm.com/docs/git-update-index
- `git-read-tree(1)` (covers `-m`, three-way merge stages) — https://git-scm.com/docs/git-read-tree
- `git-write-tree(1)` — https://git-scm.com/docs/git-write-tree
- `git-ls-files(1)` (`--stage` output format) — https://git-scm.com/docs/git-ls-files
- `git-sparse-checkout(1)` (cone vs non-cone mode, sparse index) — https://git-scm.com/docs/git-sparse-checkout
- `git-rerere(1)` — https://git-scm.com/docs/git-rerere
