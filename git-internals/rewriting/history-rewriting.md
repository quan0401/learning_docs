---
title: "History Rewriting — Rebase, Cherry-Pick, and git filter-repo"
date: 2026-05-07
updated: 2026-05-07
tags: [git, git-internals, rebase, filter-repo, history]
---

# History Rewriting — Rebase, Cherry-Pick, and git filter-repo

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `git` `git-internals` `rebase` `filter-repo` `history`

---

## Table of Contents

1. [Interactive Rebase](#1-interactive-rebase)
2. [Cherry-Pick and Revert](#2-cherry-pick-and-revert)
3. [git filter-repo](#3-git-filter-repo)
4. [History Surgery in Production](#4-history-surgery-in-production)
5. [Plumbing Walkthrough](#5-plumbing-walkthrough)

## Summary

Git history is immutable in the sense that you cannot mutate a commit object — its SHA is a hash of its contents (tree, parents, author, committer, message). What you *can* do is build new commit objects that look almost like the old ones and point your refs at them instead. Every "rewrite" command in Git is really a **replay**: walk a list of commits, recompute each one against a new base or with a new patch, and stitch the resulting objects into a new chain.

This tier covers the four mainstream rewriting tools — interactive rebase, cherry-pick, revert, and `git filter-repo` — what each one actually does to your object database, and the production hazards that show up when you run them on shared history (force-push policy, secret rotation, GitHub's cached refs). We end with plumbing demos that replay a commit chain by hand and run a small `filter-repo` operation on a throwaway repo.

The cardinal rule, lifted directly from Pro Git: rewrite freely while history is local, but treat pushed commits as final unless you have a strong reason and a clear coordination story. ([Pro Git §7.6][progit-rewriting])

---

## 1. Interactive Rebase

`git rebase -i <upstream>` is the everyday tool for cleaning up a feature branch before opening a PR. The `-i` opens an editor with a **todo list** — one line per commit that would be replayed — and Git executes that script top-to-bottom after you save and quit.

### 1.1 What rebase actually does

The `git-rebase` man page describes the algorithm in four steps ([git-rebase docs][git-rebase]):

1. Build a list of commits on the current branch since it diverged from `<upstream>` that don't already have an equivalent commit upstream (Git uses patch-id equivalence here, not raw SHA).
2. `git checkout --detach <upstream>` — detach `HEAD` at the new base.
3. **Replay the commits one by one, in order**, similar to running `git cherry-pick <commit>` for each one.
4. `git checkout -B <branch>` — point the original branch ref at the final replayed commit.

So a "rebase" is mechanically a chain of cherry-picks plus a ref move. The original commits aren't deleted; they become unreachable from any branch and survive in the reflog and loose-object pool until garbage collection.

### 1.2 The todo list commands

When you run `git rebase -i HEAD~5`, the editor opens with five `pick` lines. You edit them to express what you want:

| Command | Action |
|---------|--------|
| `pick` | Replay the commit unchanged (default). |
| `reword` | Replay, then stop so you can edit the commit message. |
| `edit` | Replay, then stop so you can amend the snapshot or split. |
| `squash` | Fold this commit into the previous one; combine messages. |
| `fixup` | Like `squash`, but discard this commit's message. |
| `drop` | Skip this commit entirely (or just delete the line). |
| `exec` | Run a shell command; non-zero exit halts the rebase. |
| `break` | Stop here without replaying anything (useful with `--exec`). |
| `label` / `reset` / `merge` | Reconstruct merge topology with `--rebase-merges`. |

Two practical patterns:

```bash
# Run tests after every commit during rebase
git rebase -i --exec "pnpm test" main

# Reorder by editing the todo list — Git reads the SHAs, not the messages
pick deadbee Implement parser
pick fa1afe1 Wire up CLI
pick f1a5c00 Add fixtures
```

If you reorder commits whose patches touch overlapping hunks, Git will pause at a conflict — resolve it, `git add`, then `git rebase --continue`. `--abort` returns to the original branch tip; `--skip` drops the current commit.

### 1.3 Splitting a commit

Mark the commit `edit`. When the rebase pauses, undo the commit but keep the changes in the working tree, then commit pieces separately:

```bash
git reset HEAD^                      # uncommit, keep working-tree changes
git add src/parser.ts
git commit -m "parser: extract tokenizer"
git add src/cli.ts
git commit -m "cli: wire tokenizer flag"
git rebase --continue
```

This is the canonical Pro Git recipe for splitting a commit during rebase ([Pro Git §7.6][progit-rewriting]).

### 1.4 `--autosquash`, `fixup!`, `squash!`

The `--autosquash` flag lets you queue corrections during normal development and have them slotted into the right place automatically when you rebase. The `git-rebase` man page defines the behavior precisely ([git-rebase docs][git-rebase]):

> Automatically squash commits with specially formatted messages into previous commits being rebased. If a commit message starts with `"squash! "`, `"fixup! "` or `"amend! "`, the remainder of the title is taken as a commit specifier, which matches a previous commit if it matches the title or the hash of that commit.

The right way to create these commits is with `git commit --fixup` and `git commit --squash`, which fill in the prefix for you:

```bash
# Make a normal commit
git commit -m "Implement parser"

# Later, fix a typo in that commit's code
git add src/parser.ts
git commit --fixup HEAD~2          # creates "fixup! Implement parser"

# Tweak the original message too
git commit --fixup=reword:HEAD~2   # creates "amend! Implement parser"

# Run autosquash rebase — the fixup is moved next to its target and changed to `fixup`
git rebase -i --autosquash main
```

Set `git config --global rebase.autoSquash true` to make `--autosquash` the default. Pair it with `rebase.autoStash = true` if you frequently rebase with dirty working trees.

### 1.5 What gets garbage-collected

A rewritten commit becomes unreachable but survives in the **reflog** (default 90 days for reachable, 30 for unreachable per `gc.reflogExpire` / `gc.reflogExpireUnreachable`). To recover after a botched rebase:

```bash
git reflog                         # find the pre-rebase HEAD
git reset --hard HEAD@{4}          # restore it
```

If you've run `git gc --prune=now` or the reflog has expired, the objects are gone. This safety net is exactly why "rewriting" is forgiving locally and unforgiving once pushed.

---

## 2. Cherry-Pick and Revert

Both commands replay or invert single commits onto the current branch. They share the same conflict-resolution machinery and the same `--continue` / `--skip` / `--abort` sequencer subcommands.

### 2.1 Cherry-pick mechanics

`git cherry-pick <commit>` computes the **patch** between `<commit>` and its parent, then performs a three-way merge of that patch against your current `HEAD`. The merge bases for the three-way are: the parent of `<commit>` (base), `<commit>` itself (theirs), and `HEAD` (ours).

When the merge succeeds cleanly, Git creates a new commit with the same author, message, and tree-delta, but a different parent and committer — so the resulting SHA is different. This is **patch-identity**, not object identity: cherry-pick reuses the *change*, not the commit object.

When it conflicts, the man page describes the bookkeeping ([git-cherry-pick docs][git-cherry-pick]):

> The current branch and `HEAD` pointer stay at the last commit successfully made. The `CHERRY_PICK_HEAD` ref is set to point at the commit that introduced the change that is difficult to apply. Paths in which the change applied cleanly are updated both in the index file and in your working tree. No other modifications are made.

```bash
# Resolve the conflict, then continue
git status                         # see paths under "Unmerged paths"
edit src/conflicting.ts
git add src/conflicting.ts
git cherry-pick --continue
```

### 2.2 The `-x` flag

Use `-x` when cherry-picking between **publicly visible branches** (a backport from `main` to `release-1.2`). The man page is explicit ([git-cherry-pick docs][git-cherry-pick]):

> When recording the commit, append a line that says `(cherry picked from commit …)` to the original commit message in order to indicate which commit this change was cherry-picked from. This is done only for cherry picks without conflicts.

```bash
git checkout release-1.2
git cherry-pick -x abc1234
# → commit message ends with "(cherry picked from commit abc1234)"
```

For purely private rewrites — say, lifting a single commit out of a stash branch — `-x` adds noise. Use it when an auditor (human or automated) might need to trace the backport.

### 2.3 `--no-commit` (-n)

`-n` applies the patch to your index and working tree without creating a commit. This is how you stack multiple cherry-picks into a single commit:

```bash
git cherry-pick -n abc1234 def5678 9876fed
# → all three patches now in the index, no commits yet
git commit -m "backport parser fixes (abc1234, def5678, 9876fed)"
```

It also skips the index-clean precondition, so you can layer a cherry-picked patch on top of uncommitted local changes.

### 2.4 Revert as inverse cherry-pick

`git revert <commit>` is mechanically a cherry-pick with the patch direction flipped. The man page calls it out ([git-revert docs][git-revert]):

> Given one or more existing commits, revert the changes that the related patches introduce, and record some new commits that record them.

It does **not** delete the original commit or rewrite history — it adds a new commit whose patch is the inverse. This is the safe choice on shared branches because no one's clones become invalid.

```bash
git revert HEAD                    # inverse of HEAD as a new commit
git revert --no-commit HEAD~3..HEAD # stage inverses of last 3, one combined commit
git revert -m 1 <merge-sha>        # revert a merge by picking which mainline to keep
```

The `-m` flag is mandatory when reverting a merge commit because Git needs to know which parent to consider the "mainline" — the side whose changes you're keeping. Pick `-m 1` for the receiving branch, `-m 2` for the topic that was merged in.

### 2.5 Conflict resolution shape

For both commands:

```bash
# When a conflict halts the operation
git status                         # list unmerged paths
edit <files>                       # resolve markers
git add <files>
git cherry-pick --continue         # or git revert --continue
# Or bail out:
git cherry-pick --abort            # restore pre-cherry-pick state
git cherry-pick --skip             # drop the conflicting commit, continue with the next
```

If you have `rerere` enabled (`git config rerere.enabled true`), Git records resolutions and replays them on subsequent identical conflicts — invaluable when rebasing or cherry-picking the same patch across many branches.

---

## 3. git filter-repo

`git filter-repo` is the modern history-surgery tool. It is **not** included in core Git; install it via Homebrew (`brew install git-filter-repo`), pip (`pip install git-filter-repo`), or your distro's package manager.

### 3.1 Why it replaced `filter-branch`

The `git filter-branch` man page now opens with this warning, verbatim ([git-filter-branch docs][git-filter-branch]):

> `git filter-branch` has a plethora of pitfalls that can produce non-obvious manglings of the intended history rewrite (and can leave you with little time to investigate such problems since it has such abysmal performance). These safety and performance issues cannot be backward compatibly fixed and as such, **its use is not recommended**. Please use an alternative history filtering tool such as [git filter-repo](https://github.com/newren/git-filter-repo/).

The `git-filter-repo` README explains the rationale in more detail: `filter-branch` spawns a separate shell and a separate Git process **per commit**, is "extremely to unusably slow" on non-trivial repos, and is "riddled with gotchas that can silently corrupt your rewrite or at least thwart your cleanup efforts" ([filter-repo README][filter-repo]).

### 3.2 How filter-repo works under the hood

`filter-repo` reads the repository through `git fast-export`, applies your transformations to the resulting stream of commits and blobs **in-process** (Python), then pipes the modified stream into `git fast-import`. Because the rewrite is one streaming pass with no per-commit subprocess overhead, it runs orders of magnitude faster than `filter-branch` and avoids whole categories of shell-quoting bugs.

After import, it cleans up: removes the original refs, expires the reflog, and runs `git gc` so the rewritten objects become the only reachable history.

### 3.3 Safety: fresh-clone requirement

By default, `filter-repo` refuses to run unless it detects a fresh clone — no extra remotes, no stash, no work-in-progress. The README puts it bluntly: the tool will "detect and bail if we're not in a fresh clone" unless you override with `--force`.

The reason: rewriting history changes every commit SHA from the rewrite point forward. If you have local branches, stashes, or remotes that still reference the old SHAs, you'll be in for a confusing merge of two parallel histories. Best practice:

```bash
git clone --no-local <origin> repo-rewrite
cd repo-rewrite
git filter-repo <flags>
# verify, then push to a fresh remote (or force-push to original after coordinating)
```

### 3.4 Common operations

The flags below are documented in the `filter-repo` README and `--help` output ([filter-repo README][filter-repo]).

**Path filtering** — keep only files under a directory:

```bash
# Only keep history of src/ — everything else disappears
git filter-repo --path src/

# Multiple paths; combine with --invert-paths to remove instead of keep
git filter-repo --path docs/ --path README.md
git filter-repo --path secrets.env --invert-paths
```

`--path-glob` accepts wildcards (`*.log`), `--path-regex` accepts regex.

**Subdirectory promotion** — move files into a subdirectory or pull a subdirectory up:

```bash
# Move everything into a subdirectory (useful when merging a repo into a monorepo)
git filter-repo --to-subdirectory-filter packages/parser

# Promote a subdirectory to repo root
git filter-repo --subdirectory-filter src/parser
```

**Tag and ref renaming**:

```bash
git filter-repo --tag-rename '':'parser-'   # prefix all tags with "parser-"
git filter-repo --refs refs/heads/main      # only rewrite main, leave others untouched
```

**Blob redaction** — replace string contents (great for leaked secrets):

```bash
# Replace literal strings or regexes from a file (one rule per line)
cat > replacements.txt <<'EOF'
AKIAIOSFODNN7EXAMPLE==>REMOVED-AWS-KEY
regex:password\s*=\s*"[^"]*"==>password="REMOVED"
EOF
git filter-repo --replace-text replacements.txt
```

**Strip large blobs**:

```bash
git filter-repo --strip-blobs-bigger-than 10M
```

**Rewrite author/committer info**:

```bash
# mailmap format: "Proper Name <proper@email>" "Old Name <old@email>"
cat > .mailmap <<'EOF'
Quan Kento <quan@qode.world> <quan@old-email.example>
EOF
git filter-repo --mailmap .mailmap
```

GitHub's sensitive-data guide notes that recent versions ship a `--sensitive-data-removal` flag that loosens some safety checks for redaction workflows ([GitHub: removing sensitive data][gh-secrets]).

---

## 4. History Surgery in Production

The mechanical steps above are the easy part. The hard part is the social and operational fallout.

### 4.1 Splitting a monorepo

Goal: extract `packages/parser/` into its own repo with full history.

```bash
# Start from a fresh clone
git clone --no-local git@github.com:org/monorepo.git parser-extract
cd parser-extract

# Promote the subdirectory and drop everything else
git filter-repo --subdirectory-filter packages/parser

# Push to the new home
git remote add origin git@github.com:org/parser.git
git push -u origin main --tags
```

Reverse direction (absorbing a repo into a monorepo) uses `--to-subdirectory-filter packages/parser` followed by `git remote add` and `git merge --allow-unrelated-histories` from the monorepo side.

### 4.2 Removing leaked secrets

This is the scenario where every step matters. GitHub's official guide is explicit about the order of operations ([GitHub: removing sensitive data][gh-secrets]):

> if the sensitive data you need to remove is a secret (e.g. password/token/credential), as is often the case, then as a first step you need to revoke and/or rotate that secret. Once the secret is revoked or rotated, it can no longer be used for access, and that may be sufficient to solve your problem.

**Step 1: Rotate the secret first.** Generate a new credential, deploy it everywhere it's used, and disable the old one. The leaked value is now worthless.

**Step 2: Rewrite history.** Use `--invert-paths` to delete the offending file, or `--replace-text` to redact a string while keeping the file:

```bash
git clone --no-local git@github.com:org/repo.git repo-clean
cd repo-clean

# Option A: drop the file from every commit
git filter-repo --path config/prod.env --invert-paths

# Option B: redact specific strings
echo 'AKIAIOSFODNN7EXAMPLE==>REMOVED' > redact.txt
git filter-repo --replace-text redact.txt
```

**Step 3: Force-push.** Coordinate with collaborators first — every clone now has dangling history.

```bash
git push --force --all
git push --force --tags
```

**Step 4: Tell GitHub it cannot serve the old SHAs.** Even after rewrite and force-push, GitHub may still serve old commit SHAs from cached views. The official guide states the warning verbatim ([GitHub: removing sensitive data][gh-secrets]):

> If you only rewrite your history and force push it, the commits with sensitive data may still be accessible elsewhere: In any clones or forks of your repository; Directly via their SHA-1 hashes in cached views on GitHub; Through any pull requests that reference them

Practical implications:

- Open a GitHub Support ticket to request that they purge cached views and stale references.
- Audit forks. A `git clone` made before the rewrite still contains the secret.
- Audit pull requests that referenced the bad commits — close or rewrite them.
- **The rotated secret stays rotated.** Don't undo step 1 just because step 2 succeeded.

### 4.3 Rewriting author info

Common when an old commit was made under a personal email and you want to attribute it to a work email, or when an entire repo was committed as `root@buildbox`.

```bash
git clone --no-local <repo> rewrite
cd rewrite
cat > .mailmap <<'EOF'
Quan Kento <quan@qode.world> <quan@personal.example>
Quan Kento <quan@qode.world> root <root@buildbox>
EOF
git filter-repo --mailmap .mailmap
```

This rewrites every commit's author and committer fields. SHAs change accordingly. The `--commit-callback` flag accepts a Python snippet for arbitrary per-commit transformations if mailmap isn't expressive enough.

### 4.4 Force-push policy

For shared branches:

- **Default to forbidden.** Configure branch protection on `main` / `release/*` to reject force-pushes outright.
- **Carve out an exception process.** A documented runbook for "we leaked a secret and rewrote history" — including pre-announcement, scheduled window, and a checklist for collaborators to re-clone.
- **Use `--force-with-lease` for personal branches** to avoid clobbering work pushed by a collaborator (or a CI bot) since you last pulled:

```bash
git push --force-with-lease origin feature/parser
```

`--force-with-lease` rejects the push if the remote ref has moved since your last fetch — a cheap insurance policy.

### 4.5 Communication

After any rewrite of shared history, send the team:

```text
We rewrote main to remove a leaked AWS key (commit abc1234..def5678).
The key has already been rotated.
Please re-clone or run:
  git fetch origin
  git reset --hard origin/main
Old SHAs from before <date> are no longer reachable on origin.
```

Half the production pain from history rewrites comes from collaborators silently merging stale local branches back in. A clear "stop, re-clone" message prevents the rewrite from being undone in pieces.

---

## 5. Plumbing Walkthrough

Three demos, all runnable in a throwaway directory.

### 5.1 Setup: a tiny test repo

```bash
cd /tmp
rm -rf rewrite-demo && mkdir rewrite-demo && cd rewrite-demo
git init -q
git commit --allow-empty -m "root"

echo "alpha" > a.txt && git add a.txt && git commit -q -m "add a"
echo "beta"  > b.txt && git add b.txt && git commit -q -m "add b"
echo "gamma" > c.txt && git add c.txt && git commit -q -m "add c"

git log --oneline
# 4 commits: root, add a, add b, add c
```

### 5.2 Demo 1 — Manually replay a commit chain with cherry-pick

We'll simulate a rebase by detaching `HEAD` to `root` and cherry-picking commits one by one — this is exactly what `git rebase` does internally.

```bash
# Save the original tip
ORIG=$(git rev-parse HEAD)

# Capture the three commits in order (oldest first)
COMMITS=$(git rev-list --reverse main ^$(git rev-parse HEAD~3))
echo "$COMMITS"
# → SHAs of "add a", "add b", "add c"

# Detach to root and replay
git checkout --detach $(git rev-parse HEAD~3)
for C in $COMMITS; do
  git cherry-pick "$C"
done

# Move main to the new tip
git branch -f main HEAD
git checkout main

git log --oneline
# Same shape, but every SHA after "root" is new — they have new committer timestamps
```

Compare commit SHAs before and after: the trees are identical (same files), the messages are identical, but committer fields differ, so the SHAs differ. That difference is the entire reason force-push is necessary after a rebase.

### 5.3 Demo 2 — Build a commit object directly with `commit-tree`

`git commit-tree` is the plumbing primitive that `cherry-pick`, `rebase`, and `commit` all eventually call. It takes a tree SHA, optional parent SHAs, and a message, and writes a commit object. We'll rebuild "add c" from scratch.

```bash
# Reset for a clean demo
git checkout main
git reset --hard "$ORIG"

# Inspect the existing "add c" commit
git cat-file -p HEAD
# → tree <sha>
#   parent <sha>
#   author ... committer ...
#
#   add c

TREE=$(git rev-parse HEAD^{tree})
PARENT=$(git rev-parse HEAD^)

# Build a new commit object pointing at the same tree and parent
NEW=$(git commit-tree "$TREE" -p "$PARENT" -m "add c (rebuilt)")
echo "new commit: $NEW"

# Inspect — it's a real commit object, just not yet referenced by any branch
git cat-file -p "$NEW"

# Move a branch to it to make it reachable
git branch demo-rebuild "$NEW"
git log --oneline demo-rebuild
```

This is what every history-rewriting tool does at the bottom of the stack: walk the existing commits, recompute their trees if needed, and write fresh commit objects with `commit-tree`. The tool layer (rebase, cherry-pick, filter-repo) just orchestrates which trees and parents go in.

### 5.4 Demo 3 — A small `filter-repo --path` operation

Requires `git-filter-repo` installed and on `PATH`.

```bash
# Fresh demo repo with two top-level files
cd /tmp
rm -rf filter-demo && mkdir filter-demo && cd filter-demo
git init -q
echo "keep me" > keep.txt && git add keep.txt && git commit -q -m "add keep"
echo "secret" > leaked.txt && git add leaked.txt && git commit -q -m "add leaked"
echo "more"   >> keep.txt   && git add keep.txt   && git commit -q -m "update keep"
echo "shh"    >> leaked.txt && git add leaked.txt && git commit -q -m "update leaked"

git log --oneline
# 4 commits, both files present in the latest tree

# Drop leaked.txt from every commit in history
git filter-repo --path leaked.txt --invert-paths --force

git log --oneline
# Still 4 commits (filter-repo keeps commits even if they become empty by default,
# unless --prune-empty=always); but leaked.txt is gone everywhere
git log --all --full-history -- leaked.txt
# → empty: no commit in the rewritten history mentions the file

ls
# → only keep.txt
```

Inspect the underlying objects to confirm the rewrite:

```bash
# The commit SHAs are completely different from before
git rev-list --all

# The tree of each commit no longer references the leaked.txt blob
for C in $(git rev-list --all); do
  echo "=== $C ==="
  git ls-tree "$C"
done
```

`--force` is needed here because our demo isn't a fresh clone (we initialized in place); in real use, clone first.

### 5.5 What the demos prove

- Rebase = chain of cherry-picks + ref move (Demo 1).
- Cherry-pick / rebase / filter-repo all reduce to writing new commit objects with `commit-tree` and pointing refs at them (Demo 2).
- `filter-repo` operates as one streaming pass over the entire history, regardless of how many commits are affected (Demo 3).

The unified mental model: history rewriting is **append-only object creation plus ref reassignment**. The "rewrite" is in the refs, never in the existing objects.

---

## Related

- [git-internals/INDEX.md](../INDEX.md) — Tier index for the Git internals path.
- [git-internals/objects/](../objects/) — Tier 1 covering blob/tree/commit objects, the foundation `commit-tree` writes into.
- [git-internals/refs/](../refs/) — Tier 2 covering refs, reflogs, and how the recoverability of rewrites depends on reflog retention.
- [system-design/](../../system-design/INDEX.md) — Operational coordination patterns relevant when force-pushing on shared branches.
- [security/](../../security/INDEX.md) — Secret management and rotation; pair with the secret-removal workflow in §4.2.
- [observability/](../../observability/INDEX.md) — Audit logging for rewrites on shared infra (who rewrote what, when).

## References

- [git-rebase documentation][git-rebase] — interactive rebase mechanics, todo-list commands, `--autosquash`, `fixup!`/`squash!` semantics.
- [git-cherry-pick documentation][git-cherry-pick] — three-way merge, `-x` flag, `--no-commit`, conflict bookkeeping.
- [git-revert documentation][git-revert] — inverse-commit creation and `--no-commit` for batched reverts.
- [git-filter-branch documentation][git-filter-branch] — official deprecation notice pointing to filter-repo.
- [git-filter-repo README][filter-repo] — flag reference (`--path`, `--invert-paths`, `--replace-text`, `--mailmap`, `--to-subdirectory-filter`, `--subdirectory-filter`, `--tag-rename`, `--refs`, `--strip-blobs-bigger-than`) and rationale for replacing filter-branch.
- [GitHub Docs: Removing sensitive data from a repository][gh-secrets] — the cached-refs warning, secret-rotation-first guidance, and `--sensitive-data-removal` flag.
- [gitrevisions documentation][gitrevisions] — `^`, `~`, `..`, `...` syntax used throughout this doc.
- [Pro Git §7.6 Rewriting History][progit-rewriting] — `commit --amend`, splitting commits, the cardinal rule about pushed history.

[git-rebase]: https://git-scm.com/docs/git-rebase
[git-cherry-pick]: https://git-scm.com/docs/git-cherry-pick
[git-revert]: https://git-scm.com/docs/git-revert
[git-filter-branch]: https://git-scm.com/docs/git-filter-branch
[filter-repo]: https://github.com/newren/git-filter-repo
[gh-secrets]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository
[gitrevisions]: https://git-scm.com/docs/gitrevisions
[progit-rewriting]: https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History
