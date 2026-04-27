---
title: "WhatsApp Deep Dive — Per-Conversation Ordering"
date: 2026-04-27
updated: 2026-04-27
tags: [system-design, case-study, whatsapp, deep-dive, ordering, distributed-systems]
---

# WhatsApp Deep Dive — Per-Conversation Ordering

**Date:** 2026-04-27 | **Updated:** 2026-04-27
**Tags:** `system-design` `case-study` `whatsapp` `deep-dive` `ordering` `distributed-systems`

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [What "Ordering" Means in Chat](#what-ordering-means-in-chat)
- [Server-Assigned Monotonic IDs](#server-assigned-monotonic-ids)
- [Client Clock Skew](#client-clock-skew)
- [Lamport Timestamps and Vector Clocks](#lamport-timestamps-and-vector-clocks)
- [Hybrid Logical Clocks](#hybrid-logical-clocks)
- [Out-of-Order Delivery and Reorder Buffers](#out-of-order-delivery-and-reorder-buffers)
- [Idempotent Message Inserts](#idempotent-message-inserts)
- [Group Chat Ordering](#group-chat-ordering)
- [Operational Pitfalls](#operational-pitfalls)
- [Reference Architectures](#reference-architectures)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

The parent case study calls out per-conversation ordering as a "hard requirement" but glosses over the engineering that makes it real: server-assigned monotonic sequence numbers per shard, client-generated UUIDs for retry idempotency, reorder buffers that hide network reordering, and a deliberate refusal to trust client wall-clock timestamps. This deep dive expands the four-paragraph subsection into a senior-engineer treatment: why total order within a conversation is the only ordering guarantee users actually perceive, why global ordering is unnecessary and expensive, how Lamport and HLC techniques apply (and where they don't fit chat), how multi-device send forces a Lamport-style merge in the protocol, and what Signal/WhatsApp/Telegram/Slack do differently. The recurring lesson: pick a single sequencer per conversation, write idempotently, display defensively, and never sort UI by `Date.now()`.

## Overview

Chat ordering looks trivial until you list the failure modes:

- Two messages from the same sender, sent 10 ms apart on a flaky 4G connection, arrive at the server in reverse order.
- A user sends from their phone and laptop simultaneously, both offline, then both reconnect.
- A group has 1,000 members across continents; everyone must see the same ordering or quoting/replies break.
- A retry storm causes the same client message to arrive twice; the duplicate must not appear as two bubbles.
- A message sent at 11:59:59 PM displays under "Yesterday" on the sender's device and "Today" on the recipient's device because of timezone parsing.

Each of these is a different problem, and each has a textbook fix. The naive instinct — "just sort by `created_at`" — fails every one of them. The correct architecture separates **the sequencer that establishes order** (server-side, per conversation) from **the identifier that survives retries** (client-generated UUID) from **the wall-clock display** (which is allowed to be approximate). Get those three layers right and most chat-ordering bugs disappear.

The parent case study (see [`design-whatsapp.md`](../design-whatsapp.md)) lands on per-shard sequence numbers as the answer. This document explains why, what the alternatives look like, and where the boundary lies between "use a sequencer" and "use a logical clock."

## What "Ordering" Means in Chat

Before picking an algorithm, be precise about the guarantee users want.

**Total order per conversation.** Within a single 1:1 or group chat, every participant must see the same sequence of messages. If Alice sees "Hi → How are you?" then Bob and Carol must too. Anything else looks like a bug.

**FIFO per sender.** Within messages sent by the same person, original send order must be preserved on every recipient. "Wait, scratch that — I meant X" must arrive after the original message it amends, not before. This is implied by total order but worth naming, because some delivery substrates (UDP, multicast) don't give it for free.

**Causal order across senders.** If Bob's reply quotes Alice's question, every viewer must see Alice's question first. Quoting and threading make the causal arrows visible, so violating them is immediately wrong. Inside a single conversation, total order is strictly stronger than causal order, so a per-conversation sequencer satisfies it automatically.

**What chat does NOT need.**

- **Global total order.** There is no user-visible reason to order Alice-and-Bob's messages relative to Carol-and-Dave's. Conflating them costs a global sequencer (single point of contention) and buys nothing.
- **Wall-clock fairness.** If two messages were composed at the same instant on different continents, users do not care which one "really" came first — they care that both endpoints agree. A purely logical sequencer is fine.
- **Real-time synchronization.** Chat is not a market data feed. Latency budgets are 100 ms — 1 s, not microseconds.

This distinction is the entire reason chat systems use **per-conversation** sequencers instead of database-wide ones: scoping the order to what users actually see lets you shard horizontally without coordination.

## Server-Assigned Monotonic IDs

The clean answer is a **per-conversation monotonic counter**, owned by exactly one server (the shard leader for that conversation) at any given time. Three ways to implement it:

### 1. Per-conversation `next_seq` column

```sql
-- Postgres, one row per conversation
CREATE TABLE conversation_state (
  conversation_id  UUID PRIMARY KEY,
  next_seq         BIGINT NOT NULL DEFAULT 0,
  shard_id         INT    NOT NULL,
  -- fencing token for shard handoff: increments on every leadership change
  epoch            INT    NOT NULL DEFAULT 1
);

-- Atomic seq assignment + insert in a single transaction
BEGIN;
  UPDATE conversation_state
     SET next_seq = next_seq + 1
   WHERE conversation_id = $1
   RETURNING next_seq, epoch INTO :seq, :epoch;

  INSERT INTO messages (conversation_id, server_seq, epoch, client_msg_id, sender_id, body, created_at)
       VALUES ($1, :seq, :epoch, $2, $3, $4, NOW());
COMMIT;
```

Pros: simple, transactional, single source of truth. Cons: every write hits the same row — fine at WhatsApp's typical conversation throughput (a few messages/second per chat) but a hotspot if the same row is updated thousands of times per second (very large groups, broadcast channels).

### 2. Snowflake-style ID, scoped per conversation

A 64-bit ID layout that encodes time, shard, and a per-conversation counter. WhatsApp publicly uses Erlang and never disclosed its exact ID format, but Twitter's Snowflake is the canonical reference and the pattern is identical.

```text
+-------------------+--------------+--------------+----------------+
|  41 bits          |  10 bits     |  6 bits      |  7 bits        |
|  ms since epoch   |  shard id    |  conv hash   |  seq within ms |
+-------------------+--------------+--------------+----------------+
```

```python
# Pseudocode — NOT for direct copy; tune bit layout for your scale
class ConversationSnowflake:
    EPOCH_MS = 1_577_836_800_000  # 2020-01-01 UTC

    def __init__(self, shard_id: int):
        self.shard_id = shard_id
        self.last_ms = 0
        self.seq_per_conv: dict[str, int] = {}  # conv_id -> last seq in current ms

    def next_id(self, conv_id: str) -> int:
        now_ms = current_ms()
        if now_ms != self.last_ms:
            self.last_ms = now_ms
            self.seq_per_conv.clear()  # new ms — reset all per-conv counters
        seq = self.seq_per_conv.get(conv_id, -1) + 1
        if seq >= 128:  # 7-bit overflow
            spin_until_next_ms()
            return self.next_id(conv_id)
        self.seq_per_conv[conv_id] = seq
        conv_hash = hash(conv_id) & 0x3F  # 6 bits
        return (
            ((now_ms - self.EPOCH_MS) << 23)
            | (self.shard_id << 13)
            | (conv_hash << 7)
            | seq
        )
```

This gives roughly time-sortable IDs that are unique cluster-wide and monotonic per conversation **as long as a single shard owns that conversation**. The big caveats: clock regressions on the issuer break monotonicity (mitigate with `last_ms` clamp), and shard handoff requires fencing.

### 3. UUIDv7 (RFC 9562) for the message ID, separate `server_seq` for ordering

A modern compromise: use UUIDv7 (time-ordered UUID) as the primary key — globally unique, no central coordination required, naturally roughly time-sorted — and keep a smaller `server_seq` per conversation purely for ordering inside the chat.

```sql
CREATE TABLE messages (
  message_id    UUID    PRIMARY KEY,        -- UUIDv7, generated server-side
  conversation_id UUID  NOT NULL,
  server_seq    BIGINT  NOT NULL,           -- monotonic per conversation
  client_msg_id UUID    NOT NULL,           -- sent by client for idempotency
  sender_id     UUID    NOT NULL,
  body          BYTEA   NOT NULL,           -- ciphertext for E2EE
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (conversation_id, client_msg_id),  -- dedup
  UNIQUE (conversation_id, server_seq)      -- ordering invariant
);

CREATE INDEX messages_conv_seq ON messages (conversation_id, server_seq DESC);
```

This is the layout most modern chat systems converge on: UUIDv7 for `message_id` (no contention, easy partitioning), `server_seq` for ordering, `client_msg_id` for retry safety. The trade-off is one extra BIGINT per message.

### Why per-conversation, not global

A global sequencer (one Snowflake worker for the whole company) seems simpler — single counter, single place — but it fails the moment you scale:

- **Hot single point.** Every send routes through one node. That's an availability and throughput cap.
- **Wasted ordering.** Global order across unrelated chats has no user-visible value.
- **Cross-region pain.** A global sequencer in `us-east-1` adds cross-region RTT to every send originating elsewhere.

Per-conversation sequencers shard the work along the same axis as the data, which is what makes WhatsApp's "billions of users, millions of conversations" architecture tractable.

### Shard handoff and fencing

When a chat shard fails over to a new owner, the old owner might still have in-flight writes. Without fencing, you can produce two messages with the same `(conversation_id, server_seq)` from two different processes. Two defenses:

1. **Fence with epoch.** Every write includes the current epoch (incremented on each leader election). Any write whose epoch is less than the row's stored epoch is rejected.
2. **Drain before takeover.** New leader waits for the old leader's lease to expire (e.g., 30 seconds) before accepting writes, ensuring no concurrent writers.

Most production systems do both.

## Client Clock Skew

**Rule: never use the client's wall-clock for ordering.** The client's clock can be:

- Wrong by hours (user manually set time, broken NTP).
- Wrong by days (factory reset, dead battery RTC).
- Set to the future to bypass paywalls or trial expirations.
- Adjusted backward by NTP mid-session.
- In a different timezone, with the user expecting timestamps in their local time.

So the order of events is **always** decided by the server's `server_seq`. The client's `created_at_local` is at most a hint — used only for display, never for sorting.

### Display vs. ordering: a separation that saves bugs

| Concern | Source of truth | Used for |
|---|---|---|
| Position in conversation list | `server_seq` | Sort order, gap detection |
| "What time was this sent?" | `server_received_at` (server's clock) | UI label, "Yesterday at 3 PM" |
| "Did the user see this before now?" | client receipt event | Read receipts, unread counts |

The client may temporarily display an outgoing message at the bottom of the chat with `created_at_local` while it's still pending. As soon as the server `ACK { server_seq, server_received_at }` arrives, the message snaps into its correct position by `server_seq`, and the displayed time is replaced with `server_received_at`.

### Catching up after offline

When a user comes back online after being offline for hours, two things must NOT happen:

1. **Their offline-composed messages must not jump to the top.** They must be inserted at their correct `server_seq` position, which is wherever the server placed them when they finally landed.
2. **Their offline-received messages must not be reordered by client clock.** The server provides a fully-ordered batch; the client renders in `server_seq` order regardless of any local timestamp.

The implementation is straightforward: the client maintains a `last_seen_seq` per conversation and on reconnect requests `messages WHERE conversation_id = ? AND server_seq > last_seen_seq ORDER BY server_seq`. Whatever ordering the server has decided, the client adopts.

## Lamport Timestamps and Vector Clocks

The parent case study explicitly says: "Avoid Lamport clocks here." That deserves explanation, because Lamport and vector clocks are exactly the kind of tool a beginner reaches for when they hear "ordering in distributed systems."

### Why Lamport is overkill for 1:1 and small groups

Lamport timestamps solve **causal ordering across independent processes that exchange messages**. In WhatsApp's path, every send for a given conversation goes through the same shard, which means:

- There is exactly one sequencer.
- There are no concurrent writers needing to agree on order without coordination.
- The total order produced by `server_seq` is strictly stronger than what Lamport gives.

Adding Lamport on top would mean every client and server tracks a per-conversation logical clock, attaches it to messages, and merges on receive — for no additional ordering guarantee. The shard already gives a total order. Lamport is a tool you reach for when **you cannot have a single sequencer**, e.g., in a Dynamo-style leaderless replicated store. Chat with a leader-per-conversation does not need it.

### Where Lamport-style merging DOES appear: multi-device send

The exception is the boundary between **a single user's multiple devices** (phone, laptop, tablet). When Alice sends from her phone while her laptop is offline, then her laptop reconnects with messages it composed offline, both devices must arrive at a common ordering of "messages Alice sent". Two strategies:

1. **Server-arbitrated.** Each device tags its outgoing message with a client logical counter (per-device, monotonic). Server uses receipt order plus the per-device counter as a tiebreaker. This is essentially Lamport collapsed into a (device_id, counter) tuple.
2. **Echoed merge.** The first device to reach the server wins; the second device's offline draft arrives later and gets a larger `server_seq`. The user sees them in the order they reached the server, which may differ from the order they were composed. This is what WhatsApp/Signal effectively do — and users tolerate it because intra-user reordering is rare and visually distinguishable.

```pseudocode
# Lamport-style per-device counter for multi-device send
# Each device i maintains L_i, monotonic per device.
on send_at(device_i):
    L_i = L_i + 1
    msg = {
      client_msg_id: uuid7(),
      device_id: i,
      device_seq: L_i,
      body: ciphertext,
      created_at_local: wall_clock(),
    }
    enqueue_outbound(msg)

# On receipt at the server:
on receive(msg):
    if (conversation_id, client_msg_id) already exists:
        return existing.server_seq        # idempotent
    server_seq = next_seq(conversation_id)
    persist(msg, server_seq, epoch)
    fanout(msg)
    return server_seq
```

The server's `server_seq` is what every recipient sees. The (device_id, device_seq) tuple is metadata — useful for debugging "why did my draft from yesterday end up after today's reply?" but invisible to ordering UI.

### Vector clocks: definitely overkill

Vector clocks would give you concurrency detection — "these two messages were sent without one knowing about the other" — but in chat, **all messages within a conversation are concurrent under that definition**, because the senders do not exchange messages outside the chat itself. The vector clock collapses to "did I see message X before sending Y?" which is precisely the read-modify-write ordering the server already enforces. Skip it.

For the deeper treatment of Lamport, vector clocks, and HLC, see [`time-and-ordering.md`](../../../data-consistency/time-and-ordering.md).

## Hybrid Logical Clocks

HLC is the right tool when you want **server-side ordering that is also human-readable**. WhatsApp does not publicly use HLC, but if you were designing a chat system today and wanted to:

- Use the same timestamp for ordering AND for the "10:42 AM" label,
- Tolerate small clock skew between chat-service replicas without coordination,
- Make logs and traces sortable across machines,

…you would use HLC. The 64-bit format is compact, lexicographically sortable, and degrades gracefully under skew.

```go
// HLC update on local event or send
package hlc

import (
    "sync"
    "time"
)

type HLC struct {
    mu sync.Mutex
    pt int64 // physical time, ms since epoch
    l  int32 // logical counter
}

func (h *HLC) Tick() (int64, int32) {
    h.mu.Lock()
    defer h.mu.Unlock()
    now := time.Now().UnixMilli()
    if now > h.pt {
        h.pt = now
        h.l = 0
    } else {
        h.l++
    }
    return h.pt, h.l
}

// HLC update on receiving a remote (pt_m, l_m)
func (h *HLC) Receive(ptM int64, lM int32) (int64, int32) {
    h.mu.Lock()
    defer h.mu.Unlock()
    now := time.Now().UnixMilli()
    newPt := max3(h.pt, ptM, now)
    var newL int32
    switch {
    case newPt == h.pt && newPt == ptM:
        newL = max32(h.l, lM) + 1
    case newPt == h.pt:
        newL = h.l + 1
    case newPt == ptM:
        newL = lM + 1
    default:
        newL = 0
    }
    h.pt = newPt
    h.l = newL
    return h.pt, h.l
}

func max3(a, b, c int64) int64 {
    m := a
    if b > m { m = b }
    if c > m { m = c }
    return m
}

func max32(a, b int32) int32 { if a > b { return a }; return b }
```

In a chat system you would store `(hlc_pt, hlc_l)` alongside `server_seq`. Within a single conversation, `server_seq` is the source of truth (because one shard owns the order). HLC adds value across:

- **Audit logs** spanning multiple shards.
- **Cross-conversation queries** ("show me all messages this user sent in the last hour" where messages live in different shards).
- **Replication and snapshots** — same purpose HLC serves in CockroachDB and MongoDB.

For WhatsApp-scale 1:1 chat, plain `server_seq` per conversation plus a wall-clock `server_received_at` is enough. HLC is the upgrade if you want the two columns merged into one with stronger guarantees.

## Out-of-Order Delivery and Reorder Buffers

The server assigns a clean `server_seq` to every message, but the network does not guarantee they arrive at the recipient in `server_seq` order. Causes:

- WebSocket frames can be reordered if the server fans out via multiple goroutines/threads without synchronization.
- A retry of an earlier delivery can race a newer one.
- Push notification + WebSocket double-delivery can interleave.
- Multi-region replication lag: a message saved in `eu-west-1` may reach a recipient in `us-east-1` after a later message that originated `us-east-1`-local.

The defense is a **reorder buffer on the client**.

```js
// client-side reorder buffer
class ReorderBuffer {
  constructor({ maxWaitMs = 800, maxBufferSize = 32 } = {}) {
    this.lastDisplayedSeq = 0;
    this.pending = new Map();          // server_seq -> message
    this.maxWaitMs = maxWaitMs;
    this.maxBufferSize = maxBufferSize;
    this.flushTimer = null;
  }

  receive(msg) {
    if (msg.server_seq <= this.lastDisplayedSeq) {
      // already displayed (duplicate); idempotent dedupe
      return [];
    }
    this.pending.set(msg.server_seq, msg);
    return this.drain();
  }

  drain() {
    const toDisplay = [];
    while (this.pending.has(this.lastDisplayedSeq + 1)) {
      const next = this.lastDisplayedSeq + 1;
      toDisplay.push(this.pending.get(next));
      this.pending.delete(next);
      this.lastDisplayedSeq = next;
    }
    if (this.pending.size > 0) {
      this.scheduleForceFlush();
    } else if (this.flushTimer) {
      clearTimeout(this.flushTimer);
      this.flushTimer = null;
    }
    if (this.pending.size > this.maxBufferSize) {
      // overflow: request a gap fill from the server
      this.requestGapFill(this.lastDisplayedSeq + 1);
    }
    return toDisplay;
  }

  scheduleForceFlush() {
    if (this.flushTimer) return;
    this.flushTimer = setTimeout(() => {
      this.flushTimer = null;
      // force-display whatever is buffered, in seq order, accepting a gap
      const sorted = [...this.pending.values()].sort(
        (a, b) => a.server_seq - b.server_seq,
      );
      this.pending.clear();
      if (sorted.length) {
        this.lastDisplayedSeq = sorted[sorted.length - 1].server_seq;
        this.onForceDisplay(sorted);            // hook for UI
      }
    }, this.maxWaitMs);
  }

  requestGapFill(fromSeq) {
    // emits an HTTP/WebSocket request to fetch missing messages
    this.onGapFill(fromSeq);
  }
}
```

Three knobs to tune:

- **`maxWaitMs`** — how long to hold a message hoping its predecessor arrives. Too low: visible reordering jumps. Too high: feels laggy. WhatsApp-style apps use 500–1000 ms.
- **`maxBufferSize`** — protects against pathological cases where the gap message will never arrive. When exceeded, fetch the gap from the server.
- **Force-display behavior** — when the timer fires, you must display in `server_seq` order regardless of the gap. The next message that arrives may have a smaller `server_seq`, in which case you've made a permanent ordering choice. Some clients (Slack) prefer this; others (Telegram) prefer to fetch and re-render.

### Gap fill: the recovery path

If a client has displayed up to `server_seq = 100` and then receives `103` and `104`, it has a gap. After `maxWaitMs`, it requests:

```
GET /messages?conversation_id=X&from_seq=101&to_seq=102
```

This request hits a read-replica or message store. The server returns either the missing messages (which the client inserts in place) or a "no such messages exist" response (in which case the gap was a phantom and the client moves on). This is the same mechanism used after offline reconnect — and it's why **gap detection requires monotonic, dense `server_seq`**. If sequences had holes by design (e.g., Snowflake IDs that skip values), the client could not distinguish "gap I should fill" from "expected hole".

## Idempotent Message Inserts

Retries are inevitable. The client sends, the network drops, the client retries — but the original send may have succeeded. Without idempotency, the message appears twice.

The fix: every client message gets a **client-generated UUID** (`client_msg_id`) attached at send time, before the first network attempt. The server uses it as the dedup key.

```sql
-- Idempotent insert: returns existing row if client_msg_id already present
INSERT INTO messages (
    message_id, conversation_id, client_msg_id, server_seq, sender_id, body, created_at
) VALUES (
    gen_random_uuid_v7(),
    $1,
    $2,                                        -- client_msg_id (idempotency key)
    nextval_for_conversation($1),
    $3,
    $4,
    NOW()
)
ON CONFLICT (conversation_id, client_msg_id)
DO UPDATE SET conversation_id = EXCLUDED.conversation_id  -- no-op
RETURNING message_id, server_seq, created_at;
```

Two subtleties:

1. **The dedup window must be larger than the retry window.** If the dedup state expires after 1 hour but a client retries after 6 hours of being offline, you get a duplicate. WhatsApp keeps `client_msg_id` indexed for the lifetime of the message. Cheap because messages are small.
2. **The `server_seq` must NOT be assigned twice.** The `ON CONFLICT DO UPDATE` clause must NOT call `nextval`, or you'll burn sequence numbers on duplicates, leaving gaps. Use `ON CONFLICT DO NOTHING` and a `RETURNING` from a `SELECT` fallback if your DB doesn't return rows on conflict.

A safer pattern in PostgreSQL:

```sql
WITH ins AS (
  INSERT INTO messages (
      message_id, conversation_id, client_msg_id, server_seq, sender_id, body
  )
  SELECT
      gen_random_uuid_v7(),
      $1,
      $2,
      next_seq.v,
      $3,
      $4
  FROM (SELECT next_seq($1) AS v) next_seq
  ON CONFLICT (conversation_id, client_msg_id) DO NOTHING
  RETURNING message_id, server_seq
)
SELECT message_id, server_seq FROM ins
UNION ALL
SELECT message_id, server_seq FROM messages
 WHERE conversation_id = $1 AND client_msg_id = $2
LIMIT 1;
```

This guarantees: first call assigns a new `server_seq`; second call returns the existing one without burning a sequence.

### Client-side rendering during retry

The client UI shows a single bubble during the entire retry cycle:

1. Compose: render bubble with `client_msg_id`, status = "sending".
2. Network failure: keep bubble, status = "queued".
3. Retry succeeds: server returns `server_seq` for the same `client_msg_id`. Client matches by `client_msg_id` and updates the existing bubble — never creates a second one.
4. Eventually: status = "delivered" or "read" via separate receipt events, also matched by `client_msg_id` or `server_seq`.

This is the same reason form submissions on the web should use idempotency keys: retries are normal, and the UI must converge to a single canonical artifact.

## Group Chat Ordering

Groups complicate ordering because the same message must be delivered to N recipients in a way they all agree on.

### Single broker per conversation

WhatsApp's approach (per the public engineering talks): the same shard owns the conversation regardless of group size. All sends route to that shard, get a single `server_seq`, and fanout copies the ordered envelope to every recipient. There is exactly one ordering authority per group, and every recipient just adopts it.

```mermaid
sequenceDiagram
    participant A as Alice
    participant B as Bob
    participant C as Carol
    participant S as Shard owning Group X

    A->>S: send(group=X, client_msg_id=u1)
    Note over S: assigns server_seq=42
    B->>S: send(group=X, client_msg_id=u2)
    Note over S: assigns server_seq=43
    S->>A: ack(seq=42)
    S->>B: ack(seq=43)
    S-->>A: deliver(seq=43, sender=Bob)
    S-->>B: deliver(seq=42, sender=Alice)
    S-->>C: deliver(seq=42); deliver(seq=43)
```

Trade-off: the shard is a hot spot for very active groups. For 256-member groups (WhatsApp's default cap), the throughput is a non-issue. For 1024-member groups or broadcast channels, you may shard the conversation across multiple shards and accept a weaker ordering — but this is rare and almost always solved with a different product (Channels, Communities) rather than a stronger algorithm.

### Per-recipient queue (Slack, Telegram channels)

Telegram channels and Slack channels (the high-volume kind) sometimes use per-recipient queues for fanout. Each recipient has their own ordered queue; the sender's write goes through a fanout service that enqueues into N queues. Ordering across queues is still guaranteed by the source `server_seq` — the queues do not reorder.

The win is operational: a single hot conversation does not bottleneck on a single shard's write throughput. The cost is N writes per send, which is fine if reads dominate (channel: 1 sender, 1M readers) and bad if writes dominate (active group chat).

### Signal Sender Keys: encryption fanout vs ordering fanout

Signal's Sender Keys protocol is about encryption fanout, not message ordering. The ordering still comes from a server sequencer. What Sender Keys solves: instead of encrypting a 1 KB message N times pairwise (one ciphertext per recipient — expensive for large groups), the sender derives one symmetric "sender key", distributes it pairwise once, and encrypts each subsequent group message **once**. The server fans out the same ciphertext to all members.

WhatsApp uses Sender Keys (it adopted the Signal Protocol in 2016). This means the message has a single canonical ciphertext, which simplifies ordering: there is one row in `messages`, one `server_seq`, and N delivery pointers in `user_inbox`.

### Multi-device groups

The hard case: Alice has phone + laptop, both in a group with Bob and Carol. When Bob sends, the server must deliver to Alice's phone AND Alice's laptop. They must each apply the message at the same `server_seq` position. This works trivially because `server_seq` is shared — both Alice devices request `messages WHERE server_seq > last_seen` and get the same ordered batch. The only thing that differs is what "last_seen" each device has.

## Operational Pitfalls

A non-exhaustive list of bugs that have shipped in real chat systems.

**1. "Everyone sees a different order" from clock-based sorting.** A team uses `created_at_server` (millisecond-precision `NOW()`) and sorts by it. Two writes within the same millisecond on different replicas collide; replicas resolve with their own clock, which differs by ~5 ms. Different clients get different orderings. Fix: sort by `(server_seq)` only.

**2. Missing messages between gaps.** Client receives `seq=100` and `seq=102`. Without a reorder buffer or gap-fill mechanism, the client never realizes `101` exists. `101` was a real message that was lost in transit. Fix: gap detection on `server_seq`, request fill on suspected gaps.

**3. Daylight-saving and 24-hour clock display.** Server stores `server_received_at` as UTC. Client formats with `toLocaleString()` using device locale. In countries that observe DST, a message sent at 1:30 AM on the day of fall-back can display ambiguously. Fix: store and transmit UTC; format defensively; never store timezone-naive timestamps.

**4. Client-generated `created_at` used for sort.** Common bug in the early days of chat apps. Symptom: a user sets their phone clock to 1995 and all their messages appear at the top of every chat. Fix: client wall-clock is a hint only; sort by `server_seq`.

**5. UUIDv4 instead of UUIDv7 for `message_id`.** UUIDv4 is random — fine for uniqueness but causes catastrophic write amplification on B-tree indexes (every insert lands in a random page). UUIDv7 is time-prefixed, sorts roughly chronologically, and keeps the index hot at the right end. RFC 9562 finalized UUIDv7 in 2024; use it.

**6. Sequence gaps from rollback.** Postgres sequences (`SERIAL`, `BIGSERIAL`) are not transactional — a rolled-back transaction still consumes a value, leaving a hole. Clients that detect "gap" then waste effort polling for a non-existent message. Fix: use the `next_seq` column update inside the transaction (atomic with the insert) instead of a global sequence.

**7. Out-of-order writes during shard handoff.** Old shard leader processes a write; new leader takes over; old leader's write lands after new leader has assigned `server_seq=101` to a different message. Now there are two messages with `server_seq=101`. Fix: epoch fencing on every write.

**8. Push notification arrival before WebSocket message.** Bob's phone gets the FCM push wakeup. It opens the app, fetches via REST API, displays the new message. Then the WebSocket reconnects and delivers the same message. Without dedup, two bubbles. Fix: dedup by `(conversation_id, client_msg_id)` AND `server_seq` on the client.

**9. Multi-device and pinning ordering.** Pinned messages need a separate ordering (pin-time, by whom) that is layered ON TOP of `server_seq`. Conflating them means the pin order changes when someone edits a pinned message. Fix: model pin as a separate event with its own `pin_seq`.

## Reference Architectures

What real systems do.

### WhatsApp

Public information is sparse — most of what's known comes from old Erlang Factory talks (2014, 2017) by Rick Reed and Anton Lavrik. Key points:

- Erlang/OTP chat servers, with one process per active connection.
- Per-conversation ordering is enforced server-side. The exact ID format is undocumented but is widely believed to be a Snowflake-style 64-bit ID encoding shard, time, and a per-conversation counter.
- Mnesia (Erlang's distributed database) and later FreeBSD-based custom storage for the message log.
- E2EE via Signal Protocol since 2016. Group chats use Sender Keys.
- See: [Erlang Factory talks](https://www.erlang-factory.com/) (search WhatsApp), and Reed's "Scaling to Millions of Simultaneous Connections" (2012).

### Signal

Signal is open source ([github.com/signalapp](https://github.com/signalapp)) and the cleanest reference for the cryptographic protocol.

- Messages have a `(timestamp, message_id)` envelope. The `timestamp` is the sender's clock (used for the Double Ratchet, NOT for ordering UI).
- Server-side ordering uses an atomic counter per recipient queue.
- Group ordering uses Sender Keys for encryption; the server provides a single ordered queue per group.
- Reference: [Signal Protocol documentation](https://signal.org/docs/), specifically the [Sender Keys spec](https://signal.org/docs/specifications/sesame/) and the [X3DH key agreement](https://signal.org/docs/specifications/x3dh/).

### Telegram

Telegram uses an MTProto custom binary protocol. It exposes `message_id` as a globally-monotonic-per-conversation 32-bit integer (per the public API docs). Channels (broadcast) use per-channel monotonic IDs that all clients adopt.

- Reference: [MTProto schema](https://core.telegram.org/schema), specifically [Working with messages](https://core.telegram.org/api/messages).

### Slack

Slack messages have a `ts` field that looks like a Unix timestamp but is actually a tuple (`<seconds>.<microseconds>`) where the microseconds component is used as an in-second sequence number for tiebreaking. This is functionally an HLC-lite — wall-clock readable, but with a logical component for per-channel ordering. Slack's API uses `ts` as the message identifier and as the cursor for pagination.

- Reference: [Slack API: Conversations history](https://api.slack.com/methods/conversations.history), and [Working with timestamps](https://api.slack.com/messaging/retrieving).

### iMessage

Apple's iMessage uses Apple's APNs infrastructure for delivery and a server-side ordering on top. The exact algorithm is proprietary, but Apple has occasionally published clues — notably, iMessage uses per-conversation server timestamps with sub-millisecond precision for ordering, plus a client UUID for dedup.

## Anti-Patterns

**1. Sorting UI by client `created_at`.** Always wrong. Use `server_seq`. Client wall-clock is a display hint, never an ordering source.

**2. Global sequencer across all conversations.** A single hot row killing your write throughput. Shard by conversation.

**3. Using a vanilla DB sequence (`SERIAL`) for `server_seq`.** Rollbacks leave gaps; clients waste effort polling for missing messages that never existed. Use a per-conversation counter row in a transaction.

**4. UUIDv4 as message ID with index on `(conversation_id, message_id)`.** Random UUIDs cause B-tree write amplification. Use UUIDv7 (RFC 9562) for natural time-ordering.

**5. Lamport clocks for chat ordering.** Lamport solves a problem chat doesn't have (concurrent writers without a single sequencer). The shard is the sequencer; Lamport adds metadata for no benefit.

**6. Vector clocks per conversation.** Same problem as Lamport, plus O(N) metadata per message. Skip.

**7. Re-using `server_seq` on retry.** If the second arrival of a duplicate burns a fresh sequence number, you get gaps. Idempotent inserts must NOT advance the counter on conflict.

**8. Storing `created_at` without timezone.** Eventually you ship a feature that displays it in a different timezone, and DST + a forgotten `TIMESTAMP` column produces hour-shifted bugs. Always use `TIMESTAMPTZ` (or equivalent).

**9. Ignoring the multi-device send race.** If Alice sends from phone and laptop simultaneously while offline, both must arrive at a deterministic ordering on Bob's screen. The simplest and most correct rule: server-arrival-order wins; the device that reaches the server first gets the lower `server_seq`. Document this explicitly.

**10. Force-displaying after a tiny `maxWaitMs`.** Setting the reorder buffer flush at 50 ms means routine network jitter causes visible reordering jumps. 500–1000 ms is the right ballpark for user-visible chat.

**11. Trying to order across conversations for "global activity feed".** The activity feed is a different product. Build it on a separate stream (Kafka, Kinesis) keyed by user, not by attempting to order all messages globally.

**12. Conflating delivery receipts with message ordering.** Read receipts (`server_seq=42 was read at T`) are a separate event stream. Mixing them into the message log produces bugs where reads "appear before" the message. Keep them in a separate table indexed by `(conversation_id, recipient, server_seq)`.

## Related

- [WhatsApp Case Study (parent)](../design-whatsapp.md) — full system design with all subsections
- [Time and Ordering in Distributed Systems](../../../data-consistency/time-and-ordering.md) — Lamport, vector clocks, HLC, TrueTime
- [URL Shortener — Code Generation Strategies](../../basic/url-shortener/code-generation-strategies.md) — Snowflake, base62, counter-based ID strategies (similar problem space)
- [Real-Time Channels](../../../communication/real-time-channels.md) — WebSocket, SSE, push delivery patterns underlying chat
- [Consensus, Raft, and Paxos](../../../data-consistency/consensus-raft-and-paxos.md) — how shards elect a single leader for the per-conversation sequencer

## References

- [Lamport, "Time, Clocks, and the Ordering of Events in a Distributed System" (1978)](https://amturing.acm.org/p558-lamport.pdf) — foundational paper introducing happens-before and Lamport timestamps; ACM Turing Award reprint.
- [Kulkarni, Demirbas et al., "Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases" (2014)](https://cse.buffalo.edu/tech-reports/2014-04.pdf) — the Hybrid Logical Clock paper.
- [RFC 9562 — Universally Unique IDentifiers (UUIDs)](https://datatracker.ietf.org/doc/html/rfc9562) — standardizes UUIDv6, v7, v8; v7 is the time-ordered variant suitable for chat message IDs.
- [Twitter Engineering Blog — "Announcing Snowflake" (archived)](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake) — original Snowflake post; the [GitHub repo (archived)](https://github.com/twitter-archive/snowflake) has the bit-layout and source.
- [Signal Protocol documentation — Sender Keys (Sesame)](https://signal.org/docs/specifications/sesame/) — group encryption protocol used by WhatsApp/Signal/Messenger.
- [Telegram MTProto — Working with Messages](https://core.telegram.org/api/messages) — public API documentation describing per-channel monotonic message IDs.
- [Slack API — Working with Timestamps](https://api.slack.com/messaging/retrieving) — `ts` field format and pagination cursors.
- [Rick Reed (WhatsApp) — "That's 'Billion' with a B: Scaling to the Next Level at WhatsApp" (Erlang Factory 2014)](https://www.youtube.com/watch?v=c12cYAUTXXs) — public talk on WhatsApp's Erlang architecture.
- [Martin Kleppmann, _Designing Data-Intensive Applications_, Ch. 8 — "The Trouble with Distributed Systems"](https://dataintensive.net/) — book-length practitioner perspective on clock skew and ordering.
