---
title: "Packfiles, Delta Compression, and Garbage Collection"
date: 2026-05-07
updated: 2026-05-07
tags: [git, git-internals, packfiles, delta-compression, gc]
---

# Packfiles, Delta Compression, and Garbage Collection

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `git` `git-internals` `packfiles` `delta-compression` `gc`

---

## Table of Contents

1. Pack Format
2. Delta Compression
3. `git gc` and Repack Strategy
4. Commit-Graph and Multi-Pack-Index
5. Plumbing Walkthrough
6. Related
7. References

## Summary

Earlier tiers showed Git as a content-addressable store of zlib-compressed
"loose" objects: one file per blob/tree/commit/tag in `.git/objects/ab/cdef…`.
That model is correct, but it scales badly. A clone of a real-world repo
would otherwise contain hundreds of thousands of tiny zlib-compressed files
sharing huge amounts of structural similarity. Pack files are Git's answer:
a single binary file (`.pack`) holding many objects, with random access via
a sibling index (`.idx`), and aggressive delta compression between similar
objects. Garbage collection (`git gc`) is the maintenance loop that pushes
loose objects into packs, repacks existing packs, prunes unreachable junk,
and writes auxiliary indexes (commit-graph, multi-pack-index) that make
reachability and history queries cheap on top of the packed store.

For a TypeScript/Node backend developer, the analogy is: loose objects are
your raw rows in a table, the pack file is a compressed columnar/clustered
format, the `.idx` is the B-tree index over that file, and `git gc` is the
nightly VACUUM/ANALYZE that keeps everything fast. This doc walks the
binary format, the heuristics behind delta compression, what `git gc`
actually does (including `--aggressive`), why commit-graph and multi-pack-
index exist, and finishes with a runnable plumbing walkthrough that lets
you see a real delta chain inside a real pack.

---

## 1. Pack Format

A pack lives at `.git/objects/pack/pack-<sha>.pack` with a sidecar index
`pack-<sha>.idx` (and, optionally, `.rev`, `.bitmap`, `.keep`, `.promisor`).
Together, the `.pack` plus its `.idx` give Git O(log N) lookup of any
contained object by SHA, plus streaming reads of the raw bytes.

### 1.1 `.pack` layout

The on-disk layout is documented in `gitformat-pack`:

```text
+----------+----------+----------+----------+
| 'P''A''C''K' (4 B)  | version  | objCount |   <- 12-byte header
| signature           | (4 B BE) | (4 B BE) |
+----------+----------+----------+----------+
| object 1 (variable, zlib- or delta-encoded)|
| object 2                                   |
| ...                                        |
| object N                                   |
+--------------------------------------------+
| trailer: hash of all bytes above (20 / 32 B)|
+--------------------------------------------+
```

The signature is the literal ASCII bytes `P A C K`. The version is currently
2 (Git accepts 2 or 3, generates only 2). The object count is unsigned
big-endian; `gitformat-pack` notes the practical implication: "we cannot
have more than 4G versions ;-) and more than 4G objects in a pack."

The trailer is "a pack checksum of all of the above" — a SHA-1 (or SHA-256
in modern Git) over the literal preceding bytes. Verifying a pack means
re-streaming it and comparing the trailer to the freshly computed digest.

### 1.2 Object entry encoding

Each object entry begins with a variable-length header that packs the
object's type and inflated size into one or more bytes:

```text
byte 0:  1ttt ssss   <- MSB=1 means "more length bytes follow"
                       ttt = 3-bit type
                       ssss = low 4 bits of size
byte 1:  1sss ssss   <- 7 more size bits, MSB=continuation
byte 2:  0sss ssss   <- last size byte (MSB=0)
... then either:
   - zlib-compressed object data (for non-delta types), or
   - delta header + zlib-compressed delta instructions
```

The 3-bit type field is one of:

| Value | Mnemonic           | Meaning                                         |
|-------|--------------------|-------------------------------------------------|
| 1     | `OBJ_COMMIT`       | A commit object, stored whole                   |
| 2     | `OBJ_TREE`         | A tree object, stored whole                     |
| 3     | `OBJ_BLOB`         | A blob, stored whole                            |
| 4     | `OBJ_TAG`          | An annotated tag, stored whole                  |
| 5     | (reserved)         | Invalid; reserved for future expansion          |
| 6     | `OBJ_OFS_DELTA`    | Delta against another object in the same pack, base addressed by negative offset |
| 7     | `OBJ_REF_DELTA`    | Delta against another object addressed by full SHA |

Type 0 is invalid. Types 1–4 are the same four object types you already know
from loose-object land. Types 6 and 7 are the heart of pack compression:
the bytes that follow are not the object itself but a *delta program* that,
when applied to a base object, reconstructs the target.

`OBJ_OFS_DELTA` is preferred inside a single pack because the offset is a
small varint and stays valid forever; `OBJ_REF_DELTA` is used when the base
lives in a different pack (so the offset is meaningless) — this is what
"thin packs" sent over the wire use, and a receiver "fattens" them via
`git index-pack --fix-thin`.

### 1.3 The `.idx` v2 file

The pack on its own is opaque: to find object `<sha>` you would have to
scan from the start. The `.idx` fixes that. v2 was introduced to support
packs larger than 4 GiB and adds CRC32s for safe pack-to-pack object copy:

```text
+--------------------------------------------+
| magic       \377 t O c     (4 B)           |
| version     2              (4 B BE)        |
+--------------------------------------------+
| fan-out[0..255]   (256 * 4 B BE)           |   cumulative count of SHAs
|                                            |   whose first byte <= i
+--------------------------------------------+
| sorted SHA names    (N * hashlen)          |
+--------------------------------------------+
| CRC32 per object    (N * 4 B)              |   v2 only
+--------------------------------------------+
| 4-byte offsets      (N * 4 B BE)           |
|   high bit = 0: this is the offset         |
|   high bit = 1: index into 8-byte table    |
+--------------------------------------------+
| 8-byte large offsets (M * 8 B BE)          |   only for >2 GiB offsets
+--------------------------------------------+
| pack checksum  (hashlen)                   |   copy of pack trailer
| idx checksum   (hashlen)                   |
+--------------------------------------------+
```

The fan-out table is the trick that makes lookup fast: to find a SHA, take
its first byte `b`, read `fanout[b-1]` and `fanout[b]`, and you have the
sub-range to binary-search. That's two table reads to narrow N objects to
roughly N/256, then `log2(N/256)` comparisons.

The CRC32 column was added in v2 specifically "so compressed data can be
copied directly from pack to pack during repacking without undetected data
corruption" (`gitformat-pack`). That is why `git repack` can splice byte
ranges between packs without re-inflating them.

---

## 2. Delta Compression

A pack does two things: it removes per-file overhead by concatenating
objects, and it replaces redundant content with deltas against similar
objects. The first wins about 50% with no cleverness. The second is where
real Git repos go from gigabytes of loose objects to tens or hundreds of
megabytes packed.

### 2.1 What a delta actually is

Git's delta is a small bytecode (copy and insert opcodes) that, given a base
object's bytes, reproduces a target object's bytes. A delta header records
the inflated sizes of base and target, then a sequence of instructions:

```text
0x80 | nybble : COPY  base[offset .. offset+size]
0x00 < n     : INSERT next n literal bytes
```

So the on-disk encoding for an `OBJ_OFS_DELTA` is: type+size header → base
offset (negative varint, pointing to the base object earlier in the pack) →
zlib-compressed delta program. To materialize the object, Git seeks back
to the base, recursively resolves *its* delta if needed, runs the program,
and emits the target bytes.

This recursion is what `pack.depth` bounds.

### 2.2 The candidacy heuristic

Git does not try to delta every pair (that would be O(N²) for a pack with
millions of objects). Instead, `pack-objects` uses a *sliding window* over
a sorted object list. The official `git-pack-objects` docs are explicit:

> "The objects are first internally sorted by type, size and optionally
> names and compared against the other objects within `--window` to see if
> using delta compression saves space."

In practice the sort key is roughly:
1. Type (commits with commits, trees with trees, blobs with blobs).
2. The "preferred base name hash" — Git hashes pathnames so that the same
   path in different commits is likely adjacent (this is why renames hurt
   pack ratio).
3. Size, descending — bigger objects considered first.

Within the window, every candidate is tried as a base for the current
target; the smallest delta wins, subject to the depth limit. If no delta
shrinks the object enough, it's stored whole.

### 2.3 The four knobs that matter

| Config                     | Default | What it controls                                            |
|----------------------------|---------|-------------------------------------------------------------|
| `pack.window`              | 10      | How many candidate bases to consider per object             |
| `pack.depth`               | 50      | Maximum delta-chain length (also `--depth`, max 4095)       |
| `pack.windowMemory`        | 0       | Hard memory cap for the window; 0 = no cap                  |
| `core.bigFileThreshold`    | 512m    | Files >= this are streamed and *not* delta-compressed       |

Defaults from `git-pack-objects` and `git-config` documentation. `git gc
--aggressive` overrides these with `gc.aggressiveWindow` (default 250) and
`gc.aggressiveDepth` (default 50), and forces `--no-reuse-delta` so every
delta is recomputed from scratch.

`core.bigFileThreshold` is the escape hatch for repositories that contain
large binaries (PDFs, MP4s, model weights). Above the threshold the file
is stored without trying to delta against neighbors and without loading
it whole into memory. For Node devs: this is roughly the equivalent of
"do not gzip files over 512 MiB; just stream them through."

### 2.4 Why blobs sort by size and type

Two operational reasons:

1. **Type adjacency keeps deltas useful.** A commit's bytes and a blob's
   bytes share almost nothing structurally; comparing them across the
   window wastes CPU. Sorting by type so similar objects are neighbors
   fills the window with realistic candidates.

2. **Size descending puts likely *bases* first.** Pro Git notes Git's
   preference: "the second version of the file is the one that is stored
   intact, whereas the original version is stored as a delta — this is
   because you're most likely to need faster access to the most recent
   version of the file." Storing the bigger/newer one whole and the
   smaller/older one as a delta against it means the hot path is fast.

The Pro Git example: a 22 054-byte `repo.rb` exists in two near-identical
versions. After `git gc`, the older revision is stored as a 9-byte delta
against the new full copy. Combined pack size: ~7 KiB versus the original
~15 KiB pair of loose objects.

---

## 3. `git gc` and Repack Strategy

`git gc` is a façade over a pipeline of plumbing commands. It is what runs
when Git decides "too much loose state has accumulated."

### 3.1 What `git gc` actually does

Quoting `git-gc`: `gc` "Runs a number of housekeeping tasks within the
current repository, such as compressing file revisions (to reduce disk
space and increase performance), removing unreachable objects which may
have been created from prior invocations of `git add`, packing refs,
pruning reflog, rerere metadata or stale working trees."

The pipeline, in order:

1. `git pack-refs --all --prune` — collapses `refs/heads/*` and
   `refs/tags/*` from per-file into `.git/packed-refs`.
2. `git reflog expire --expire-unreachable=…` — drops stale reflog entries
   so unreachable objects can later be pruned.
3. `git repack -d -l -A` (typically) — packs loose objects, removes
   redundant packs, and (because of `-A` instead of `-a`) demotes
   unreachable objects in old packs back to loose so they aren't
   immediately deleted.
4. `git prune --expire 2.weeks.ago` — deletes loose unreachable objects
   older than the grace window.
5. `git worktree prune` — drops dead linked worktrees.
6. (If enabled) `git commit-graph write --reachable --split` — writes the
   commit-graph for fast reachability.

### 3.2 When it auto-runs

`git gc --auto` is invoked from inside common porcelain commands (`git
commit`, `git merge`, `git fetch`, …). From `git-gc`: "When common
porcelain operations that create objects are run, they will check whether
the repository has grown substantially since the last maintenance, and if
so run `git gc` automatically."

The threshold knobs:

| Config             | Default | Meaning                                                        |
|--------------------|---------|----------------------------------------------------------------|
| `gc.auto`          | 6700    | Loose-object count that triggers an `auto` packing pass        |
| `gc.autoPackLimit` | 50      | Pack-file count that triggers consolidation into one pack      |
| `gc.autoDetach`    | true    | If true, `--auto` runs in the background so foreground is fast |

(Defaults stated in `git-gc` docs.) Set `gc.auto=0` to disable automatic
GC entirely — sometimes desirable on CI builders where you want
deterministic timing and run `git gc` explicitly.

### 3.3 `git repack -ad` versus `--geometric`

`git gc` invokes `git repack`. The relevant flags from `git-repack`:

- `-a` (all) — "pack everything referenced into a single pack… use with `-d`."
- `-d` — "After packing, if the newly created packs make some existing
  packs redundant, remove the redundant packs."
- `-A` — like `-a`, but with `-d` "any unreachable objects in a previous
  pack become loose, unpacked objects, instead of being left in the old
  pack." This is what gives you the two-week grace window.
- `-f` — passes `--no-reuse-delta`, forcing recomputation.
- `--geometric=<factor>` — "Arrange resulting pack structure so that each
  successive pack contains at least `<factor>` times the number of objects
  as the next-largest pack."

Geometric repack is the modern default in `git maintenance run --task=
incremental-repack`. Instead of one giant repack that touches every byte,
it keeps a few ordered packs in a *power-of-`<factor>*` hierarchy and only
rewrites the tail. For a multi-GB repo this is the difference between a
30-second background task and a 20-minute full repack.

### 3.4 `git gc --aggressive`: cost and tradeoff

`--aggressive` does three things differently:

1. Passes `--no-reuse-delta` — every delta recomputed from scratch.
2. Uses `gc.aggressiveWindow` (default **250**, vs. the normal 10).
3. Uses `gc.aggressiveDepth` (default **50**).

The window-of-250 alone is a 25× increase in delta candidate comparisons
per object. On a real repo this can take *hours*. The official guidance
from `git-gc` is unambiguous:

> "It's probably not worth it to use this option on a given repository
> without running tailored performance benchmarks on it. It takes a lot
> more time, and the resulting space/delta optimization may or may not be
> worth it. Not using this at all is the right trade-off for most users
> and their repositories."

Practical rule: do not run `--aggressive` reflexively. Run it once after a
big `git filter-repo` or after importing from another VCS, then never
again. For routine maintenance, `git maintenance start` with the default
incremental-repack task is the right answer.

---

## 4. Commit-Graph and Multi-Pack-Index

Even with packs, two operations stay slow on big repos: walking commit
parents (used by `git log --graph`, `git merge-base`, `git tag --merged`,
fetch negotiation) and locating an object when there are dozens of packs.
Two auxiliary files solve this without changing the on-disk object model.

### 4.1 Commit-graph file

`.git/objects/info/commit-graph` (or a chain of split files) caches every
commit's parent SHAs and "generation number" — a topological depth
calculated once at write time. From `git-commit-graph`: it provides
"significant performance gains for getting history of a directory or a
file with `git log -- <path>`" and accelerates reachability queries.

Generation numbers turn `is X reachable from Y?` from a graph walk that
loads each commit object into a comparison: if `gen(X) > gen(Y)`, Y can't
reach X, full stop. The default generation version is **2** (corrected
commit dates), per `commitGraph.generationVersion`.

`--reachable` is the build mode you almost always want: "Generate the new
commit graph by walking commits starting at all refs."

### 4.2 Multi-pack-index (MIDX)

`.git/objects/pack/multi-pack-index` is a single index across *all* packs
in the directory. Without it, looking up a SHA means checking each pack's
`.idx` in turn (O(packs)). With it, one binary search resolves SHA to
`(pack, offset)` directly. The MIDX docs: "We check `<dir>/packs/multi-
pack-index` for the current MIDX file, and `<dir>/packs` for the pack-
files to index."

MIDX pairs naturally with geometric repack: you keep many packs around for
write performance, and the MIDX hides the cost on read.

### 4.3 How they work together

```text
git log --graph --oneline    git rev-list --count HEAD   git fetch (negotiation)
        |                              |                            |
        v                              v                            v
 +------------------- commit-graph (parents + generation #) ------------+
        |
        v
 +------------------- multi-pack-index (sha -> pack, offset) -----------+
        |
        v
 +------------------- pack-N.idx (sha -> offset)  ---------------------+
        |
        v
 +------------------- pack-N.pack (compressed objects + deltas) -------+
```

Each layer caches the layer below. You can rebuild any of them at any time
without changing repository semantics; they are pure performance.

Enable both with:

```bash
git config core.commitGraph true
git config core.multiPackIndex true
git config gc.writeCommitGraph true
git config fetch.writeCommitGraph true   # write/update on every fetch
```

Or just `git maintenance start`, which sets these and registers a cron/
launchd job to keep them fresh.

---

## 5. Plumbing Walkthrough

Everything above is observable. Set up a tiny repo, force a pack, and
inspect.

### 5.1 Build a repo with delta-friendly content

```bash
mkdir /tmp/packdemo && cd /tmp/packdemo
git init -q

# 50 versions of the same large-ish file with small mutations.
# Delta compression will love this.
seq 1 5000 | tr '\n' ' ' > data.txt
git add data.txt && git commit -qm "v1"
for i in $(seq 2 50); do
  printf "\nrev %d\n" "$i" >> data.txt
  git commit -qam "v$i"
done

# Force everything into a single pack (this is what `git gc` does).
git repack -adf
```

After `git repack -adf` you should see a single `.pack` and `.idx`:

```bash
ls -lh .git/objects/pack/
# pack-<sha>.idx
# pack-<sha>.pack
```

### 5.2 `git verify-pack -v`: see the delta chain

```bash
PACK=$(ls .git/objects/pack/*.pack | head -1)
git verify-pack -v "$PACK" | head -40
```

Expected shape of a row:

```text
<sha>  blob   <size>  <packed-size>  <offset> <chainlen> <base-sha>
<sha>  commit <size>  <packed-size>  <offset>
non delta: 53 objects
chain length = 1: 49 objects
chain length = 2: ...
<pack-sha>
```

The columns are the SHA, type, inflated size, on-disk size, byte offset
in the pack, and (for deltas) chain length plus the base SHA. The trailing
histogram tells you exactly how the delta chains are distributed. With
the demo data above you should see the largest blob stored whole and ~49
deltified versions chained off it.

### 5.3 Inspect a single object's storage

```bash
# Pick the most recent blob.
LATEST=$(git ls-tree HEAD -- data.txt | awk '{print $3}')

# How is it stored? (cached/loose/packed, size in pack, delta base)
git cat-file --batch-check='%(objectname) %(objecttype) %(objectsize) %(objectsize:disk) %(deltabase)' <<< "$LATEST"
# <sha> blob 27839 12 <base-sha>
```

If the printed `(objectsize:disk)` is much smaller than `(objectsize)`,
that object is stored as a delta. `(deltabase)` gives you the base SHA;
follow it with another `cat-file --batch-check` to walk the chain.

### 5.4 Build a pack by hand: `git pack-objects --stdout`

```bash
# Stream every object reachable from HEAD into a fresh pack on stdout.
git rev-list --objects --all \
  | git pack-objects --stdout --revs > /tmp/handmade.pack

# Re-create a matching .idx from the .pack alone:
git index-pack -v --stdin < /tmp/handmade.pack
# writes pack-<sha>.idx (and a copy of the pack)
```

`--stdout` (per `git-pack-objects`) "Write the pack contents (what would
have been written to .pack file) out to the standard output." This is
literally what `git push` and `git fetch` do over the wire — and `git
index-pack` is what the receiving side runs.

### 5.5 Force a thicker pack and watch deltas grow

```bash
git -c pack.window=50 -c pack.depth=100 repack -adf
git verify-pack -v .git/objects/pack/*.pack | tail -10
```

Compare `chain length = …` lines before and after. With the wider window,
more objects find usable deltas, the average chain gets longer, and total
pack size shrinks — at the cost of more CPU during repack and slower
checkout (more delta application on read).

### 5.6 Commit-graph and MIDX

```bash
git commit-graph write --reachable
ls .git/objects/info/
# commit-graph

git multi-pack-index write
ls .git/objects/pack/
# multi-pack-index
# pack-<sha>.idx
# pack-<sha>.pack

git commit-graph verify
git multi-pack-index verify
```

Then time a graph walk before/after to feel the difference on a real
repo:

```bash
# In a large repo (e.g. linux.git)
git -c core.commitGraph=false rev-list --count HEAD
time git -c core.commitGraph=false rev-list --count HEAD

git commit-graph write --reachable
time git -c core.commitGraph=true rev-list --count HEAD
```

On the kernel repo this is typically a 5–20× speedup for reachability
queries.

### 5.7 What `git gc` actually invokes

To see the pipeline live:

```bash
GIT_TRACE=1 git gc --auto 2>&1 | grep -E 'pack-refs|repack|prune|reflog|commit-graph'
```

You'll see the sequence from §3.1 — `pack-refs`, `reflog expire`,
`repack`, `prune`, `commit-graph write` — each invoked as a separate
process. `git gc` is not magic; it's a shell-level orchestrator over
plumbing you can run yourself.

### 5.8 Cleanup checklist for a "weird and slow" clone

When a working repository is sluggish, run these in order and stop when
performance returns:

```bash
git fsck --full                              # confirm integrity first
git pack-refs --all --prune                  # collapse loose refs
git commit-graph write --reachable --split   # cheap, big win
git multi-pack-index write                   # only if many packs
git repack -ad --geometric=2                 # consolidate packs cheaply
git gc --prune=now                           # only when sure no work is salvageable
```

Do not jump to `git gc --aggressive`. It is rarely the answer and almost
always more expensive than the alternatives above.

---

## Related

- [../objects/loose-objects-and-zlib.md](../objects/loose-objects-and-zlib.md) — what packfiles compress *from*
- [../refs/refs-and-packed-refs.md](../refs/refs-and-packed-refs.md) — the "packed-refs" sibling concept for refs
- [../rewriting/filter-repo-and-history-rewriting.md](../rewriting/filter-repo-and-history-rewriting.md) — when `--aggressive` is actually justified (post-rewrite)
- [../../performance/01-latency-and-percentiles.md](../../performance/01-latency-and-percentiles.md) — why "average is fast" hides the tail-latency cost of full repack
- [../../operating-systems/01-process-and-thread-model.md](../../operating-systems/01-process-and-thread-model.md) — `gc.autoDetach` and what "in the background" means at the OS level

## References

- Git project, "gitformat-pack" — pack and idx binary format. <https://git-scm.com/docs/gitformat-pack>
- Git project, `git-pack-objects(1)` — `--window`, `--depth`, `--stdout`, defaults. <https://git-scm.com/docs/git-pack-objects>
- Git project, `git-gc(1)` — auto thresholds, `--aggressive` cost. <https://git-scm.com/docs/git-gc>
- Git project, `git-repack(1)` — `-a`, `-d`, `-A`, `-f`, `--geometric`. <https://git-scm.com/docs/git-repack>
- Git project, `git-verify-pack(1)`. <https://git-scm.com/docs/git-verify-pack>
- Git project, `git-commit-graph(1)` — `--reachable`, generation numbers. <https://git-scm.com/docs/git-commit-graph>
- Git project, `git-multi-pack-index(1)` — MIDX layout and location. <https://git-scm.com/docs/git-multi-pack-index>
- Scott Chacon and Ben Straub, *Pro Git*, 2nd ed., §10.4 "Packfiles". <https://git-scm.com/book/en/v2/Git-Internals-Packfiles>
