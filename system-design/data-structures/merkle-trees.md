---
title: "Merkle Trees"
date: 2026-04-25
updated: 2026-04-25
tags: [system-design, data-structures, hashing, anti-entropy]
---

# Merkle Trees — Efficient Diff at Scale

**Date:** 2026-04-25 | **Updated:** 2026-04-25
**Tags:** `system-design` `data-structures` `hashing` `anti-entropy`

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [Key Concepts](#key-concepts)
  - [Construction — Bottom-Up Hashing](#construction--bottom-up-hashing)
  - [Proof of Inclusion in O(log n)](#proof-of-inclusion-in-olog-n)
  - [Anti-Entropy Protocol — Compare Roots, Descend Mismatches](#anti-entropy-protocol--compare-roots-descend-mismatches)
  - [Sparse Merkle Trees](#sparse-merkle-trees)
  - [Merkle DAG — Beyond Trees](#merkle-dag--beyond-trees)
  - [Verkle Trees — The Successor](#verkle-trees--the-successor)
- [Trade-offs](#trade-offs)
- [Code Examples](#code-examples)
- [Real-World Uses](#real-world-uses)
  - [Bitcoin and Ethereum Block Headers](#bitcoin-and-ethereum-block-headers)
  - [Cassandra and DynamoDB Anti-Entropy Repair](#cassandra-and-dynamodb-anti-entropy-repair)
  - [IPFS — Merkle DAG as Filesystem](#ipfs--merkle-dag-as-filesystem)
  - [Git — Tree and Commit Objects](#git--tree-and-commit-objects)
  - [ZFS, Btrfs, and S3 Inventory](#zfs-btrfs-and-s3-inventory)
  - [rsync — Rolling Hash Plus Block Hash](#rsync--rolling-hash-plus-block-hash)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

A **Merkle tree** is a binary tree of cryptographic hashes where leaves hash data blocks and every interior node hashes the concatenation of its children. The root is a single fixed-size hash that **uniquely fingerprints** the entire dataset — change one byte anywhere and the root changes. Two properties make Merkle trees indispensable in distributed systems: any single leaf can be **proven authentic against the root in O(log n) hashes**, and two replicas can **find every divergent block in O(k log n)** time where `k` is the number of differences. This is the engine behind Git, Bitcoin and Ethereum block headers, Dynamo and Cassandra anti-entropy repair, IPFS content addressing, ZFS data integrity, and rsync-style efficient diff. This doc covers the construction, the inclusion proof, the anti-entropy protocol, sparse and DAG variants, the Verkle successor, and the systems where you actually meet these trees in production.

## Overview

You have two replicas of a 1 TB dataset. They have drifted — some writes were lost, some were applied out of order. You need to find the differences and only ship the differing blocks across the network. Naive solutions:

- **Hash the whole dataset** — O(1) compare, O(n) work to do the comparison, and tells you nothing about *where* differences are.
- **Hash every block and ship the list** — O(n) bytes on the wire just to compare. For 1 TB at 4 KB blocks and 32-byte hashes that's 8 GB of metadata.
- **Stream both datasets and diff** — O(n) bytes on the wire — exactly what you wanted to avoid.

Merkle trees give you the third option: a **logarithmic comparison protocol**. Each side computes a tree once. Comparing two replicas costs `O(k log n)` work where `k` is the number of differing leaves, regardless of dataset size. Most blocks match, and the protocol prunes matching subtrees immediately at the root and never descends them.

The same structure also gives you **succinct membership proofs**: given the root and a logarithmic-size sibling path, anyone can verify that a specific data block is part of the tree without holding the rest of the data. This is what makes Merkle roots valuable in block headers, light clients, and verifiable data structures.

```text
                root = H( H1 || H2 )
                        /        \
              H1 = H(Ha||Hb)   H2 = H(Hc||Hd)
                /   \           /   \
              Ha    Hb         Hc    Hd     <- leaf hashes
              |     |          |     |
              A     B          C     D      <- data blocks
```

Change `C` and only `Hc`, `H2`, and `root` change — `Ha`, `Hb`, `H1` are untouched. Two replicas compare roots first; if they match, done. If not, they compare children, and only descend into mismatched subtrees.

## Key Concepts

### Construction — Bottom-Up Hashing

A Merkle tree over `n` data blocks is built bottom-up:

1. **Hash each block** — `Hi = H(blocki)` for some cryptographic hash `H` (SHA-256, BLAKE3, Keccak-256).
2. **Pair adjacent hashes** — at each level, concatenate two siblings and hash: `parent = H(left || right)`.
3. **Promote up the tree** — repeat until a single root hash remains.

If `n` is not a power of two, implementations differ. Bitcoin **duplicates the last hash** at each odd level (a quirk that opened the CVE-2012-2459 malleability bug). Most modern systems either pad with a known sentinel, use an unbalanced tree where the lone node promotes unchanged, or fix `n` by construction (e.g. fixed-depth sparse trees).

```text
Level 0:  data blocks       A    B    C    D    E
Level 1:  leaf hashes       Ha   Hb   Hc   Hd   He
Level 2:  internal hashes   H(Ha||Hb)   H(Hc||Hd)   He'   <- He promoted (or duplicated)
Level 3:  internal hashes   H(H(Ha||Hb) || H(Hc||Hd))     H(He'||He')
Level 4:  root              H( ... )
```

The hash function must be **collision-resistant**: finding two distinct inputs with the same hash should be infeasible. Otherwise a forged block could match the tree. SHA-256 is the historical default; BLAKE3 is faster and increasingly common; Keccak-256 (a SHA-3 variant) is used by Ethereum.

**Domain separation** is a subtle but real concern: prepend a tag byte to leaf hashes (`H(0x00 || data)`) and a different tag to internal hashes (`H(0x01 || left || right)`). Without it, an attacker who chooses internal hash values can create a leaf whose data equals an interior node, confusing membership proofs. RFC 6962 (Certificate Transparency) specifies exactly this scheme.

### Proof of Inclusion in O(log n)

Given the root, you can prove that a specific block is in the tree by sending only the **sibling hashes along the path from the leaf to the root** — `O(log n)` hashes total. The verifier rebuilds the path:

```text
                       root
                      /    \
                    H1      [H2]      <- sibling at level 2
                   /   \
                [Ha]    Hb            <- sibling at level 1
                        |
                        B             <- the block being proven
```

To prove `B` is in the tree, send `[Ha, H2]`. Verifier computes:
- `H1' = H(Ha || H(B))`
- `root' = H(H1' || H2)`
- Check `root' == root`

For a tree of `n = 1,000,000` leaves the proof is 20 sibling hashes — about 640 bytes with SHA-256. This is the foundation of:

- **SPV (Simplified Payment Verification)** in Bitcoin — light wallets verify a transaction is in a block by holding only block headers and asking a full node for the Merkle path.
- **Certificate Transparency** — proves a TLS certificate was logged.
- **Light clients** in Ethereum, Cosmos, Filecoin.
- **Storage proofs** in S3 inventory and inclusion-style audit logs.

### Anti-Entropy Protocol — Compare Roots, Descend Mismatches

Two replicas want to find every divergent block. Each holds its own Merkle tree over the same key range. The protocol:

```pseudocode
function compareSubtree(localNode, peer):
  remoteHash = peer.getHash(localNode.path)
  if localNode.hash == remoteHash:
    return []                                   // entire subtree matches, prune
  if localNode.isLeaf():
    return [localNode.path]                     // base case: a divergent leaf
  diffs = []
  diffs += compareSubtree(localNode.left, peer)
  diffs += compareSubtree(localNode.right, peer)
  return diffs
```

If the trees are identical, exactly **one** hash is exchanged — the root. If `k` leaves differ, the protocol traverses at most `O(k log n)` nodes. The matching subtrees are pruned at the highest common ancestor, which is why **infrequent drift is cheap to detect** even on enormous datasets.

This is the core of **Dynamo-style anti-entropy repair**:

- Each node maintains a Merkle tree per **key range** (per virtual node / token range).
- During repair, replicas of the same range exchange roots, then descend mismatched subtrees.
- Only the actual divergent keys are streamed.

Cassandra calls this **read repair** (lazy, online) and **anti-entropy repair / `nodetool repair`** (scheduled, full). DynamoDB does similar comparisons internally but Amazon does not publish the algorithm in detail. See the original Dynamo paper for the canonical design.

### Sparse Merkle Trees

A **sparse Merkle tree (SMT)** has a leaf for every possible key — a fixed depth (typically 256) so every key maps deterministically to a leaf path via its hash. Most leaves are empty (a known zero value). To make this practical, implementations exploit the fact that any subtree containing only empty leaves has a precomputed, well-known hash. You only store the non-empty leaves and the path nodes that touch them.

**Why this matters:**

- **Proof of non-inclusion** — you can prove a key is *absent* by showing the empty leaf and its path to the root. A standard Merkle tree cannot do this without sorting and ranges.
- **Authenticated maps** — etcd-style key-value stores, ZK-rollup state trees, and verifiable credential registries use SMTs because every key has a well-defined location.
- **Deterministic structure** — two SMTs with the same key set always produce the same root, regardless of insertion order. Plain Merkle trees over a list depend on order.

Ethereum's state trie is a related structure — a **Merkle Patricia Trie** — which is a hybrid of a radix trie and a Merkle tree. It uses path compression to avoid the O(256) depth per key while preserving authenticated-map properties.

### Merkle DAG — Beyond Trees

A **Merkle DAG** generalizes the tree: nodes can have multiple parents and can reference each other by hash, so long as the graph remains acyclic (cycles are impossible because a node's hash depends on its children's hashes, which would have to exist before it).

**IPFS** is built on Merkle DAGs:

- Files are split into chunks; each chunk is a leaf with a CID (content identifier).
- A directory or large file is an internal node whose CID hashes references to child CIDs.
- Identical sub-files are deduplicated naturally — they have the same CID.
- The DAG is content-addressed: the address *is* the cryptographic identity.

**Git** is also a Merkle DAG (covered below): blobs, trees, and commits are nodes; a commit's hash depends on its tree's hash, parent commits, and metadata.

The DAG generalization unlocks **structural sharing**: if two large datasets share a 99% common prefix, the DAG nodes for that prefix are referenced by both roots and stored once. This is the basis of efficient versioning, incremental backups, and content-addressed storage.

### Verkle Trees — The Successor

Merkle proofs are `O(log n)` hashes — but the *constant* is large because each level of an `n`-ary tree requires `n-1` sibling hashes per step. Ethereum's Merkle Patricia Trie has `n=16`, so each step ships 15 sibling hashes; full state proofs are tens of kilobytes.

A **Verkle tree** replaces hash siblings with a **vector commitment** (typically a KZG or IPA polynomial commitment). The proof is a single short cryptographic object that authenticates the entire path, regardless of the branching factor. This lets Ethereum increase `n` to 256 (shorter trees) while shrinking proofs from tens of KB to hundreds of bytes.

Verkle trees are not a replacement for Merkle trees in general — vector commitments require trusted setup or pairings and are far more expensive to compute than SHA-256. They make sense when **proof size dominates** (light clients, ZK-rollups, stateless clients) and you can afford the prover cost.

## Trade-offs

| Concern | Plain Merkle Tree | Sparse Merkle Tree | Merkle DAG | Verkle Tree |
|---|---|---|---|---|
| Build cost | O(n) hashes | O(n + k log N) where N is key space | O(n) hashes | O(n) commitments (slow) |
| Inclusion proof size | O(log n) hashes | O(log N) hashes | O(log n) hashes per path | O(1) commitment (~200 B) |
| Non-inclusion proof | Requires sorted leaves | Native | Native via DAG path | Native |
| Deterministic root for same set | No (order matters) | Yes | Yes (content-addressed) | Yes |
| Hash function | SHA-256 / BLAKE3 / Keccak | Same | Same | KZG / IPA commitment |
| Implementation complexity | Low | Medium | Medium | High |
| Best fit | Anti-entropy, light proofs | Authenticated maps | Versioned content, IPFS, Git | Stateless Ethereum, ZK |

**Memory vs latency for anti-entropy:**

- A Merkle tree over a 1 TB key range with 4 MB leaves has 250,000 leaves — about 8 MB of internal hashes plus 8 MB of leaf hashes. Cheap.
- Make the leaves smaller (per-key) and the tree explodes — billions of leaf hashes.
- In practice, Cassandra builds the tree **per token range** with a configurable depth (default 15 levels = 32,768 leaf buckets per range). Each leaf hashes a *bucket* of keys, not a single key. Repair finds the divergent bucket, then streams the keys in it.

**Hash bucket granularity is the knob:**

- Smaller leaves → finer divergence detection → fewer wasted bytes streamed → bigger tree.
- Larger leaves → coarser detection → more keys re-streamed unnecessarily → smaller tree.

**Re-hashing on every write:**

- A simple Merkle tree must re-hash `O(log n)` nodes per write. For high write rates this is expensive.
- Cassandra recomputes the tree **only at repair time** (or with incremental repair, only over changed ranges). It is not continuously maintained.
- Git computes tree and commit hashes only at commit time, not per file save.

## Code Examples

### Build a Merkle Tree and Verify Inclusion (Python)

```python
import hashlib
from typing import List, Tuple

def H(data: bytes) -> bytes:
    return hashlib.sha256(data).digest()

# Domain-separated leaf and internal hashes (RFC 6962 style)
def leaf_hash(data: bytes) -> bytes:
    return H(b"\x00" + data)

def internal_hash(left: bytes, right: bytes) -> bytes:
    return H(b"\x01" + left + right)

def build_merkle_root(blocks: List[bytes]) -> Tuple[bytes, List[List[bytes]]]:
    """Returns (root, levels) where levels[0] are leaf hashes and levels[-1] = [root]."""
    if not blocks:
        raise ValueError("empty input")
    levels = [[leaf_hash(b) for b in blocks]]
    while len(levels[-1]) > 1:
        cur = levels[-1]
        nxt = []
        for i in range(0, len(cur), 2):
            left = cur[i]
            right = cur[i + 1] if i + 1 < len(cur) else cur[i]   # duplicate odd node
            nxt.append(internal_hash(left, right))
        levels.append(nxt)
    return levels[-1][0], levels

def inclusion_proof(levels: List[List[bytes]], index: int) -> List[Tuple[str, bytes]]:
    """Returns sibling hashes from leaf to root, tagged with 'L' or 'R' for the sibling side."""
    proof = []
    for level in levels[:-1]:
        is_right = index % 2 == 1
        sibling_idx = index - 1 if is_right else index + 1
        if sibling_idx >= len(level):
            sibling_idx = index   # odd-tail self-pair
        proof.append(("L" if is_right else "R", level[sibling_idx]))
        index //= 2
    return proof

def verify_inclusion(root: bytes, block: bytes, proof: List[Tuple[str, bytes]]) -> bool:
    h = leaf_hash(block)
    for side, sibling in proof:
        if side == "L":
            h = internal_hash(sibling, h)
        else:
            h = internal_hash(h, sibling)
    return h == root

# Example
blocks = [f"block-{i}".encode() for i in range(8)]
root, levels = build_merkle_root(blocks)
proof = inclusion_proof(levels, 5)
assert verify_inclusion(root, blocks[5], proof)
assert not verify_inclusion(root, b"forged", proof)
```

### Anti-Entropy Diff Between Two Trees

```python
def diff_subtree(local_levels, remote_levels, level_idx, node_idx):
    """Returns a list of differing leaf indices."""
    if level_idx == 0:
        # leaf level — disagreement here is a real divergence
        if node_idx >= len(local_levels[0]) or node_idx >= len(remote_levels[0]):
            return [node_idx]
        if local_levels[0][node_idx] != remote_levels[0][node_idx]:
            return [node_idx]
        return []

    # compare hashes at this level — prune matching subtrees
    if (node_idx < len(local_levels[level_idx])
        and node_idx < len(remote_levels[level_idx])
        and local_levels[level_idx][node_idx] == remote_levels[level_idx][node_idx]):
        return []

    # descend into both children
    left = diff_subtree(local_levels, remote_levels, level_idx - 1, node_idx * 2)
    right = diff_subtree(local_levels, remote_levels, level_idx - 1, node_idx * 2 + 1)
    return left + right

def diff_trees(local_levels, remote_levels):
    top = len(local_levels) - 1
    return diff_subtree(local_levels, remote_levels, top, 0)

# Example: simulate two replicas that disagree on blocks 2 and 5
blocks_a = [f"v1-{i}".encode() for i in range(8)]
blocks_b = list(blocks_a)
blocks_b[2] = b"v2-2"
blocks_b[5] = b"v2-5"

_, levels_a = build_merkle_root(blocks_a)
_, levels_b = build_merkle_root(blocks_b)

print(diff_trees(levels_a, levels_b))   # -> [2, 5]
```

In a real distributed system the two sides do not share `levels` directly — they exchange hashes incrementally over the network, requesting only the children of nodes whose hashes disagree. The protocol structure is identical; the implementation just streams nodes lazily.

## Real-World Uses

### Bitcoin and Ethereum Block Headers

**Bitcoin** stores a Merkle root of all transactions in each block header (80 bytes total). The full transaction list lives in the block body. A **Simplified Payment Verification (SPV)** wallet downloads only headers (~50 MB for the full chain at the time of writing) and asks a full node for a Merkle path to confirm a transaction is included in a block. Verification is `O(log n)` hashes — a few hundred bytes per proof.

The Bitcoin Merkle tree duplicates the last hash on odd levels, which led to **CVE-2012-2459** (a malleability bug allowing two distinct transaction lists to yield the same Merkle root). Bitcoin Core mitigated this by also requiring no duplicate transactions at the leaf level.

**Ethereum** uses *three* Merkle Patricia Tries per block, all rooted in the block header:

- **State trie** — every account's balance, nonce, storage, and code.
- **Transactions trie** — the block's transactions.
- **Receipts trie** — execution receipts (logs, gas used).

Light clients verify any storage slot or log emission with a Merkle Patricia proof against the relevant root. The migration to **Verkle trees** (planned for Ethereum's "Verge" upgrade) shrinks proofs by ~10x to enable stateless clients.

### Cassandra and DynamoDB Anti-Entropy Repair

**Cassandra** uses Merkle trees in `nodetool repair`:

1. The coordinator asks each replica of a token range to build a Merkle tree.
2. The trees are streamed to the coordinator.
3. The coordinator compares roots, descends mismatched subtrees, and identifies the differing buckets.
4. Cassandra streams the keys in those buckets between replicas (entire ranges, not individual keys, due to per-bucket granularity).

The default tree depth is 15 (32,768 leaf buckets per range). Repair is expensive because every replica has to re-read its data to recompute the tree — Cassandra has incremental repair (track which sstables have been repaired) and reaper-style scheduled repair to amortize the cost.

**Amazon Dynamo** (the original 2007 paper, not DynamoDB the service) describes Merkle trees as the anti-entropy mechanism between replicas of a key range. The published Dynamo design is the textbook reference: per-range trees, root-first comparison, descend on mismatch. DynamoDB the service is closed-source; Amazon does not publish the current internal repair algorithm.

See [`design-key-value-store.md`](../case-studies/distributed-infra/design-key-value-store.md) and [`quorum-and-tunable-consistency.md`](../data-consistency/quorum-and-tunable-consistency.md) for how anti-entropy fits with quorum reads/writes and read repair.

### IPFS — Merkle DAG as Filesystem

**IPFS** addresses every piece of content by its hash (a CID). A file is split into chunks; each chunk is a leaf in a Merkle DAG. A directory node references its children's CIDs. The CID *is* the address, so:

- Identical content has identical CID — automatic deduplication.
- A small change re-hashes only the affected path to the root, leaving the rest of the DAG untouched.
- Verifying any block is trivial: hash it and compare to the CID you fetched it under.

UnixFS (IPFS's directory format) and IPLD (the DAG schema language) formalize the Merkle DAG abstraction. Filecoin builds storage incentives on top of IPFS-style content addressing.

### Git — Tree and Commit Objects

Every Git object is content-addressed by its SHA-1 (transitioning to SHA-256). The object types form a Merkle DAG:

- **Blob** — file content. Its hash is `H("blob " + len + "\0" + content)`.
- **Tree** — a directory listing. Each entry is `(mode, name, hash_of_blob_or_subtree)`. Its hash is over the serialized listing.
- **Commit** — references one tree (the root of the snapshot) plus parent commit(s), author, message. Its hash is over those fields.

Two consequences:

- **Identical files in different commits share storage** — they have the same blob hash.
- **A commit hash uniquely identifies the entire repository state at that point** — change one byte anywhere and every ancestor hash up to the commit changes.

`git fetch` and `git push` use a Merkle-style protocol to figure out which objects the other side already has: walk the DAG, compare commit hashes, transfer only what's missing.

### ZFS, Btrfs, and S3 Inventory

**ZFS** and **Btrfs** are copy-on-write filesystems that build a Merkle tree over every block:

- Every block has a checksum stored in its parent.
- The superblock holds the root of the tree.
- A scrub walks the tree and detects silent corruption — the whole point of CoW filesystems.
- Snapshots and clones share unchanged subtrees.

**S3 inventory comparisons** between buckets, regions, or backups commonly use Merkle-style folder hashes: hash each object's metadata, hash each prefix as a function of its children, and compare prefix hashes between the source and destination to find what needs to sync. This is the same pattern as `rsync` extended to object storage. See [`design-object-storage.md`](../case-studies/distributed-infra/design-object-storage.md).

### rsync — Rolling Hash Plus Block Hash

`rsync` is *not* strictly a Merkle tree — it is the related "rolling hash plus strong hash" algorithm:

- Receiver splits its file into fixed-size blocks and computes a weak rolling hash + a strong (MD5/MD4) hash for each.
- Sends the list to the sender.
- Sender slides a window through its file computing the rolling hash; on a match, verifies with the strong hash.
- Sends only the literal bytes between matched blocks.

Spiritually it is the same idea as Merkle anti-entropy: hash blocks, send hashes, only ship divergent data. The difference is rsync uses a flat block list (no tree), so the metadata cost is O(n) and there is no logarithmic shortcut. For large datasets, modern protocols (zsync, BitTorrent, IPFS) use Merkle DAGs to recover the logarithmic property.

## Anti-Patterns

- **Hashing without domain separation.** Hashing leaves and internal nodes with the same scheme allows second-preimage attacks where an internal hash value is presented as a leaf. Always tag (`H(0x00 || data)` for leaves, `H(0x01 || left || right)` for internal nodes) — RFC 6962.
- **Using a non-cryptographic hash for "performance".** MurmurHash, xxHash, and CRC are not collision-resistant. They are fine for hash tables and cache keys, never for Merkle trees that protect against adversarial input. A Merkle tree built on a weak hash is worthless — anyone can forge a sibling.
- **Per-key leaves at scale.** A Merkle tree with one leaf per key over a billion keys is 1 billion leaf hashes. Always bucket: each leaf hashes a *range* of keys. Cassandra's depth-15 default gives 32,768 buckets per range — pick a granularity that keeps leaf hashes in the millions, not billions.
- **Continuous tree maintenance for write-heavy workloads.** Re-hashing `O(log n)` nodes on every write costs more than the write itself. Build the tree on demand (at repair time) or maintain it incrementally only over changed ranges.
- **Forgetting that order matters in plain Merkle trees.** `[A, B, C]` and `[B, A, C]` produce different roots. Two replicas with the same key set but different insertion order disagree at the root. For order-independent fingerprints, use a Sparse Merkle Tree or sort first.
- **Naive "duplicate the odd leaf" without checking.** The Bitcoin CVE-2012-2459 transaction-list malleability bug came from this. Either pad with a known sentinel, use a fixed depth, or explicitly reject duplicate leaves.
- **Treating Merkle inclusion proofs as authentication.** A Merkle proof shows a leaf is in a tree with a given root. It says **nothing** about whether the root is current, signed, or trusted. The root must come from an authenticated source (a block header with proof-of-work, a signed certificate transparency log entry, an etcd consensus value).
- **Ignoring proof-size growth in deep trees.** A 256-deep sparse Merkle tree with SHA-256 has 256-hash proofs = 8 KB. For light clients on mobile, that adds up. This is the motivation for Verkle trees.

## Related

- [Bloom and Cuckoo Filters](./bloom-and-cuckoo-filters.md) — probabilistic membership; Merkle trees are the deterministic, authenticated counterpart
- [HyperLogLog](./hyperloglog.md) — cardinality sketch; another fingerprint-the-set primitive but for counting, not authentication
- [Quorum Reads/Writes (NWR) and Tunable Consistency](../data-consistency/quorum-and-tunable-consistency.md) — anti-entropy repair complements quorum protocols when replicas drift
- [Consensus — Raft and Paxos](../data-consistency/consensus-raft-and-paxos.md) — the strongly-consistent alternative; Merkle anti-entropy is for eventually-consistent systems
- [Design a Key-Value Store](../case-studies/distributed-infra/design-key-value-store.md) — Dynamo-style design where Merkle trees drive replica repair
- [Design an Object Storage Service](../case-studies/distributed-infra/design-object-storage.md) — Merkle-style inventory comparison across regions and tiers

## References

- Ralph C. Merkle, ["Protocols for Public Key Cryptosystems" (1980)](https://www.merkle.com/papers/Protocols.pdf) — the original paper introducing the hash tree as a way to authenticate many items with one root
- Ralph C. Merkle, ["A Digital Signature Based on a Conventional Encryption Function" (CRYPTO '87)](https://link.springer.com/chapter/10.1007/3-540-48184-2_32) — the formal Merkle signature scheme that gives the tree its name
- Giuseppe DeCandia et al., ["Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — Section 4.7 describes Merkle-tree-based anti-entropy between replicas
- Satoshi Nakamoto, ["Bitcoin: A Peer-to-Peer Electronic Cash System" (2008)](https://bitcoin.org/bitcoin.pdf) — Section 7 ("Reclaiming Disk Space") and Section 8 ("Simplified Payment Verification") describe Merkle roots in block headers
- Juan Benet, ["IPFS - Content Addressed, Versioned, P2P File System" (2014)](https://github.com/ipfs/papers/raw/master/ipfs-cap2pfs/ipfs-p2p-file-system.pdf) — the IPFS whitepaper, describing the Merkle DAG model
- [Apache Cassandra — Anti-Entropy Repair Documentation](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/repair.html) — how Merkle trees drive `nodetool repair`
- Ben Laurie, Adam Langley, Emilia Kasper, ["Certificate Transparency" (RFC 6962)](https://datatracker.ietf.org/doc/html/rfc6962) — Merkle tree construction for CT logs, including the leaf/internal domain separation scheme
- Vitalik Buterin, ["Verkle trees" (2021)](https://vitalik.eth.limo/general/2021/06/18/verkle.html) — the canonical introduction to Verkle trees as a successor to Merkle Patricia tries in Ethereum
- Andrew Tridgell, ["Efficient Algorithms for Sorting and Synchronization" (PhD thesis, 1999)](https://www.samba.org/~tridge/phd_thesis.pdf) — the rsync algorithm, Chapters 3-4
- Rasmus Dahlberg, Tobias Pulls, Roel Peeters, ["Efficient Sparse Merkle Trees: Caching Strategies and Secure (Non-)Membership Proofs" (NordSec 2016)](https://eprint.iacr.org/2016/683.pdf) — modern reference on SMT construction and proofs
