---
title: "Live Comments Deep Dive — Fan-Out Architecture"
date: 2026-04-29
updated: 2026-04-29
tags: [system-design, case-study, live-comments, deep-dive, fanout, real-time]
---

# Live Comments Deep Dive — Fan-Out Architecture

**Date:** 2026-04-29 | **Updated:** 2026-04-29
**Tags:** `system-design` `case-study` `live-comments` `deep-dive` `fanout` `real-time`

## Summary

A live comment system is not a chat system with more users — it is a fan-out engine that happens to carry chat-shaped payloads. With 1M concurrent viewers on a single stream and only 1% of them posting, every accepted comment must be replicated to roughly a million sockets within a second, and the *cost of one extra delivery* dominates every other line on the bill. This deep dive sits underneath the [Design Live Comments](../design-live-comments.md) case study and zooms in on the fan-out subsystem alone: tree-shaped vs star-shaped topologies, what each pub/sub broker buys you at the edge, regional bridges that keep messages in-region until they have to leave, the per-pod subscription tables that decide who gets what, in-process LRU vs Redis for membership data, what backpressure looks like when 8 Gbps of egress runs out of bandwidth, how Twitch / YouTube / Facebook actually solve this, and the sampling and sharding strategies that exist purely because human attention is bounded at ~5 msg/sec. Everything here is a deliberate trade between fan-out cost, ordering, and graceful degradation; the goal is to recognise the choices, not to memorise one stack.

## Table of Contents

- [Summary](#summary)
- [Overview — Why Live Comments Is a Fan-Out Problem](#overview--why-live-comments-is-a-fan-out-problem)
- [Tree vs Star Fanout](#tree-vs-star-fanout)
- [Edge Pub/Sub Choices](#edge-pubsub-choices)
- [Regional Bridges](#regional-bridges)
- [Subscription Tables Per Pod](#subscription-tables-per-pod)
- [In-Process Caching — LRU vs Redis](#in-process-caching--lru-vs-redis)
- [Backpressure When Fan-Out Saturates](#backpressure-when-fan-out-saturates)
- [Industry Examples](#industry-examples)
- [Sampling and Sharding](#sampling-and-sharding)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Overview — Why Live Comments Is a Fan-Out Problem

Begin from the asymmetry: writers are scarce, readers are everyone. The number that drives architecture is `writers × readers`, not `writers`.

```
Concurrent viewers       V = 1,000,000
Writer ratio             w = 1%
Per-writer post rate     r = 1 msg/sec
Ingest                   = V·w·r        =       10,000 msg/sec
Delivery                 = (V·w·r)·V    = 10,000,000,000 msg/sec
```

Ten billion deliveries per second is the worst case on a single hot stream. The stream is not "one stream"; it is a multiplier of 100,000× between the message that enters the system and the messages that leave it. Every architectural lever in this document — trees over stars, regional bridges, sampling, multiplexing — exists to attack that 10⁵ multiplier.

Three properties have to survive that attack:

1. **Order.** Replies must not appear before parents; bans must beat the spam they ban. Order is per stream only, never global. The sequencer in the parent doc owns this; fan-out must not reorder.
2. **Sub-second p95 end-to-end.** Anything slower stops feeling live. The clock starts at `POST` and stops at the last viewer's render.
3. **Graceful degradation.** When the bandwidth bill cannot be paid, the system samples or sheds — it does not freeze, OOM, or reorder.

Fan-out architecture is the part of the live-comments system that takes one ordered message off the sequencer's mailbox and turns it into a million writes onto a million sockets, ideally without any of those properties breaking.

```
┌──────────┐     1     ┌──────────┐    100    ┌────────────┐  10K  ┌─────────┐
│Sequencer ├──────────▶│Bus/Relay ├──────────▶│ Edge Pods  ├──────▶│ Sockets │
└──────────┘           └──────────┘           └────────────┘       └─────────┘
   1 msg/s                ~100 fan-out          ~10K fan-out         ~1M total
```

Three hops, each multiplying by ~100. The work at each hop is bounded; the trick is choosing the topology at each hop so the hops compose.

## Tree vs Star Fanout

The single most important choice in this subsystem is whether the sequencer talks to every consumer directly (star) or talks to relays that talk to consumers (tree).

### Star fan-out

```
       ┌──▶ Edge1
       ├──▶ Edge2
SEQ ───┼──▶ Edge3
       ├──▶ ...
       └──▶ Edge200
```

- **Pro:** simple, one hop, lowest latency.
- **Con:** the sequencer does N sends per message, where N = number of Edge pods. At 200 pods × 10K msg/sec, that is 2M sends/sec from one process.
- **Con:** TCP congestion control is per-connection; one slow Edge pod blocks the sequencer's writer thread (head-of-line) unless every send is async with a per-pod queue, and now the sequencer holds 200 queues plus their backpressure logic.
- **When it works:** small clusters (≤ 50 pods), short-lived bursts, or when the bus itself is a managed broker that fans out internally (Kafka with a partition per pod, NATS subject hierarchy, Redis Pub/Sub channel).

### Tree fan-out

```
                  ┌─▶ Relay1 ─┬─▶ Edge1
                  │           ├─▶ Edge2
                  │           └─▶ Edge3
SEQ ──▶ TopRelay ─┼─▶ Relay2 ─┬─▶ Edge4
                  │           ├─▶ Edge5
                  │           └─▶ Edge6
                  └─▶ Relay3 ─┬─▶ Edge7
                              └─▶ Edge8
```

- **Pro:** the sequencer makes K sends (K = relay fan-out, often 10–32). Each relay then makes K sends. The total work is the same, but it is *parallelised* and *amortised* across nodes that can fail independently.
- **Pro:** a slow Edge pod stalls only its relay's queue, not the sequencer.
- **Pro:** you can colocate relays near their consumers (per-region, per-AZ), turning long links into short ones.
- **Con:** extra hop, +1–5 ms p50 in practice.
- **Con:** more components to operate; relay placement matters.

**The pattern Discord calls Manifold** is the canonical version: when one message must reach N nodes, do not call `send` N times; group destinations by node, send once per node, let the receiving node do the local fan-out. This collapses N×M work to N+M and is the reason Discord could scale Elixir to 5M concurrent users on commodity hardware ([How Discord Scaled Elixir to 5,000,000 Concurrent Users](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)).

**Hybrid** is the realistic answer: a *star* between the sequencer and a small relay tier (because there are few relays), then a *tree* beneath each relay (because there are many edges). This is essentially what Twitch's "Pubsub fans out internally to Edge nodes, Edge fans out from there to clients" describes ([Twitch Engineering: An Introduction and Overview](https://blog.twitch.tv/en/2015/12/18/twitch-engineering-an-introduction-and-overview-a23917b71a25/)).

### Branching factor

The optimal branching factor `K` per relay tier is the one that balances per-node CPU/network against tree depth. A few observations:

- A modern Linux box with epoll and a 25 Gbps NIC handles 50–100K outbound TCP writes/sec at small payload sizes without breaking sweat. Pick `K` so each relay does ≤ 50K writes/sec at peak.
- Each extra level adds latency roughly equal to one network RTT plus the queue latency at the relay. Deep trees (4+ levels) are rare in real systems; 2–3 is the sweet spot for a single-stream fan-out.
- For 200 Edge pods, two levels of tree with K=15 gets you there with ~32 sends from the sequencer/top relay and ~14 from each second-tier relay.

### When star is still right

For all the criticism above, star fan-out is correct in narrow cases:

- **Small clusters.** Below ~50 edges, the sequencer can issue async sends without breaking a sweat, and the extra relay hop is pure latency cost.
- **Managed broker as the tree.** If your pub/sub broker (Kafka, NATS, Redis cluster) already does internal fan-out across its own nodes, the sequencer's "star" is conceptual — the broker itself runs a tree internally.
- **Bursty workloads with cheap reconnects.** A short-lived event with weak ordering needs (status updates, presence pings) can tolerate the sequencer-as-star simplicity.

The decision rule is: if `N_edges × msg_rate > N_edges + relay_overhead × msg_rate`, use a tree. For 200 edges and 10K msg/sec, the right side wins; for 20 edges and 100 msg/sec, the left side does.

## Edge Pub/Sub Choices

The bus between sequencer and Edge pods is where most of the message-rate work happens. The four real options at scale, with the trade-offs that matter for live comments:

### Redis Pub/Sub

`PUBLISH stream:42 "{...}"` arrives at every server that has `SUBSCRIBE stream:42` active. Latency is sub-millisecond intra-cluster. There is **no durability**: a subscriber that disconnects loses messages, and there is no replay.

- **Use it for:** the live path, where missing a message in flight is acceptable (the cold-start cache covers reconnects). Many production live-comment paths run Redis Pub/Sub on the hot path and Kafka in parallel as the durable spine.
- **Watch out for:** the `PUBSUB NUMSUB` cliff — a single Redis node spends O(subscribers) time per publish; with 200 subscribers per channel and 10K publishes/sec, that's 2M dispatch ops/sec on one node. Cluster-shard by stream_id.
- **Reference:** [Redis Pub/Sub](https://redis.io/docs/latest/develop/interact/pubsub/).

### NATS / NATS JetStream

NATS core is a subject-routed, in-memory bus with sub-millisecond latency and a wildcard subject hierarchy: `live.stream.42`, `live.stream.42.mod`, `live.stream.42.tombstones`. JetStream layers durability, replay, and exactly-once on top.

- **Use it for:** when you want the simplicity of Redis Pub/Sub plus a sane subject hierarchy and an upgrade path to durability without changing your client. NATS handles "everyone subscribes to `live.stream.>`" cleanly without the per-subscriber dispatch cliff that Redis hits.
- **Strength:** subject filtering at the broker — Edge pods subscribe only to streams whose viewers they currently hold, not to a firehose.
- **References:** [NATS Subject-Based Messaging](https://docs.nats.io/nats-concepts/subjects), [JetStream](https://docs.nats.io/nats-concepts/jetstream).

### Kafka

Partitioned, durable, replayable log. Latency is 5–50 ms typical. Kafka is the wrong tool for the live path — sub-millisecond viewers can feel an extra 30 ms — but it is the right tool as a **durable spine**: the sequencer publishes to Redis (live path) and to Kafka (durable spine, async, off-critical-path) in parallel. Downstream consumers (VOD indexer, moderation, analytics, archive) read from Kafka without affecting the live path.

- **Use it for:** the durable backbone, not the hot fan-out.
- **Watch out for:** partition count limits — at very high stream counts you cannot have one partition per stream forever; group by stream-shard.

### MQTT brokers (HiveMQ, EMQX, Mosquitto)

MQTT was built for IoT, but the topic-tree model and QoS levels translate directly to live-comment fan-out:

- `live/stream/42/comments` is a topic; clients subscribe to a tree.
- QoS 0 = fire-and-forget (lowest latency, no ack).
- QoS 1 = at-least-once with ack.
- QoS 2 = exactly-once via a four-step handshake (high overhead).
- Retained messages give you a free "last value" cache for cold-start.
- Shared subscriptions (`$share/group/topic`) load-balance one topic across N consumers — useful for Edge pod groups.

For live comments QoS 0 over WebSocket-MQTT is competitive with Redis on latency, plus you get a topic tree and last-will-and-testament for cleaner disconnect handling. Facebook Messenger publicly attributed its mobile chat path to MQTT for years for this reason.

- **Reference:** [MQTT 5.0 OASIS Standard](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html).

### Picking one

| Bus | Latency | Durability | Subject filter | Topic count | Where it fits |
|-----|---------|------------|----------------|-------------|---------------|
| Redis Pub/Sub | <1 ms | none | by channel | high | hot path, simple |
| NATS core | <1 ms | none | wildcards | very high | hot path, hierarchy |
| NATS JetStream | 1–5 ms | yes | wildcards | very high | hot+spine |
| Kafka | 5–50 ms | yes | by partition | bounded | durable spine |
| MQTT QoS 0 | <2 ms | none | topic tree | very high | mobile-friendly hot path |

Production answer for a serious live-comment system: **two buses**. A low-latency in-memory bus on the hot path (Redis or NATS or MQTT) plus Kafka as the durable spine. The sequencer writes to both; the live path reads from the fast one; analytics and VOD read from Kafka. The cost is one more system to operate; the win is independent failure domains and the ability to replay history without re-publishing onto the live path.

## Regional Bridges

A 1M-viewer global stream is not 1M sockets in one region. It is 300K in NA, 250K in EU, 200K in APAC, 250K elsewhere. Three rules of thumb shape the cross-region design:

1. **Viewers connect to the nearest region.** Anycast or geo-DNS routes each viewer to its closest Edge tier.
2. **A poster is also nearest-region.** The post lands at a regional API, hits the regional sequencer (or, for global ordering, a global one — see below), and fans out *intra-region* immediately.
3. **Cross-region traffic is summarised, not duplicated.** A regional bridge replicates the canonical message once per remote region, where it is fanned out locally.

```
            ┌────────────────── Region NA ──────────────────┐
            │  Sequencer ─▶ Bus ─▶ Relays ─▶ Edge Pods ─▶ V │
            └─────────────────────│─────────────────────────┘
                                  │ (1 link, 1 copy per stream)
            ┌─────────────────────▼─────────────────────────┐
            │ Region EU                                     │
            │  Bridge ─▶ Bus ─▶ Relays ─▶ Edge Pods ─▶ V    │
            └───────────────────────────────────────────────┘
```

**Bridge implementations**:

- **Kafka MirrorMaker 2** (or Confluent Replicator). Reliable, durable, partition-preserving — ideal when the durable spine is Kafka anyway. ~100–500 ms cross-region replication latency depending on geography.
- **NATS leaf nodes / superclusters.** A NATS supercluster connects regions with bidirectional gateways; subjects are routed by interest, so messages flow only when there is a subscriber in the remote region. Often <100 ms cross-region.
- **Redis Pub/Sub bridge** (custom): a single subscriber per region forwards to the local Redis. Simple, lossy if the bridge restarts, requires re-subscription tracking.
- **Wormhole-style binlog tap** (Facebook): a publisher reads the canonical log and streams to consumers in remote DCs, with checkpointing and rewind. Designed for "trillion messages a day" inter-DC fan-out — see [Wormhole pub/sub system, moving data through space and time](https://engineering.fb.com/2013/06/13/data-infrastructure/wormhole-pub-sub-system-moving-data-through-space-and-time/).

**Where to enforce ordering**:

- **Per-region sequencer.** Each region has its own sequencer for streams whose owner is in that region; bridges replicate elsewhere. Global order is not preserved, but per-stream order is. Cheap and almost always good enough.
- **Global sequencer.** All writes for a stream go to one region. Posters in remote regions pay the cross-region RTT on POST (~80–250 ms). Most live-comment systems pick this when the stream is regionally bound (a US sports stream's audience is mostly US) and accept the latency for the rare remote poster.

**Cross-region latency budget** (for a global stream):

```
NA poster → NA sequencer        :   5 ms
NA sequencer → NA Edge pods     :  10 ms
NA sequencer → bridge           :   2 ms
bridge → EU bus                 :  80 ms (transatlantic)
EU bus → EU Edge pods           :  10 ms
EU Edge pod → EU viewer         :  20 ms
                                 -------
Total NA poster → EU viewer     : 127 ms
```

That fits inside a 1-second p95 with room to spare. The thing that breaks it is bridge restarts (cold buffer, head-of-line on backlog) and cross-region link saturation during incidents.

## Subscription Tables Per Pod

Each Edge pod holds a slice of the world: ~50K–200K WebSocket connections, each subscribed to one or more streams. The pod's job is to take an inbound message from the bus and deliver it to exactly the sockets on this pod that care.

The data structure that makes that O(subscribers-on-this-pod) instead of O(everyone) is the **subscription table**:

```
streamId → Set<connectionId>
connectionId → { socket, streamIds, lastSeenSeq, ... }
```

When a message arrives for `stream:42`:

```
for conn_id in subscriptions["stream:42"]:
    conn = connections[conn_id]
    conn.write(message)        # non-blocking; queue if socket is slow
```

This is where every wasted byte hurts. A few patterns:

### Co-locate subscribers

Edge load balancing must consider stream affinity. If `stream:42`'s subscribers are spread evenly across all 200 pods, every pod receives every message — bandwidth scales O(messages × pods). If `stream:42`'s subscribers are concentrated on 20 pods (and the bus knows this), only those 20 pods receive the message.

Two ways to achieve concentration:

1. **Stream-aware routing at the LB.** The Edge LB hashes `stream_id` to a pod group; clients connecting for `stream:42` go to one of those pods. Trade-off: a viewer in many streams gets routed by their first-stream, then has to multiplex.
2. **Pod-driven subscriptions.** Each pod subscribes to the bus only for streams where it currently has subscribers. When a viewer joins `stream:42` and this pod has no other subscribers for it, the pod calls `SUBSCRIBE stream:42`. When the last subscriber leaves, `UNSUBSCRIBE`. The bus handles concentration automatically.

Production systems do both: routing concentrates *new* viewers, and pod-driven subscriptions handle the long tail.

### Tracking subscription deltas

The subscription table is mutable on every connect/disconnect. Two updates per second per pod (across 100K connections, 10% join/leave per minute) is ~170 ops/sec — easy. The watch-out is during *cascading reconnects* (an Edge pod dies, 100K clients reconnect to neighbours within seconds): subscription churn can dwarf message rate. Bound this by making subscribe/unsubscribe cheap (in-memory hash) and batching bus subscribe calls.

### Per-stream multicast trees

Some pods build per-stream multicast trees inside the process: when a message arrives, walk a tree of connection groups (e.g., by tier: free / sub / mod) and skip groups whose viewers have all left. This is overkill for chat unless your single stream's subscriber list per pod exceeds ~50K.

## In-Process Caching — LRU vs Redis

The subscription table is one of several pieces of state the Edge pod consults on every message. Others:

- Per-stream **slow-mode** state (rarely changes; once per stream lifetime usually).
- Per-user **rate-limit** counters (per-message, per-user).
- **Recent message history** for cold-start (last N messages per stream).
- **Block lists** (banned users per stream).

For each, the choice is *where* the state lives:

| State | Hot per-message? | Mutability | Best home |
|-------|------------------|------------|-----------|
| Subscription table | yes | high | in-process |
| Slow-mode setting | no (per-stream) | low | in-process LRU + Redis |
| Rate-limit counters | yes | high | in-process LRU + Redis backstop |
| Recent message buffer | medium | high | in-process ring + Redis |
| Block list | yes | low | in-process LRU |

### In-process LRU

A bounded LRU cache (e.g., 100K entries) in the pod's address space. Lookups are nanoseconds. Misses fall through to Redis. TTL on entries means stale data self-evicts.

- **Pro:** zero network on hit. Predictable.
- **Con:** stale until TTL or pub/sub invalidation. Cold on pod startup.

### Redis (or memcached) for shared state

Slow-mode and block-list updates need to be visible to all pods. Two patterns:

1. **Read-through**: pod misses LRU → Redis GET → populate LRU. Simple, but the first request after a setting change pays Redis RTT.
2. **Subscribe-and-cache**: pod subscribes to a pub/sub channel for invalidations (`stream:42:slowmode_changed`) and updates its LRU on receipt. Reads are always local. The cache is eventually consistent but bounded by network jitter.

For live-comment fan-out, the right answer is almost always:

- Subscription table: in-process only (it's pod-local by definition).
- Slow-mode: in-process LRU with subscribe-and-cache invalidation.
- Rate limit: in-process token bucket per (user, stream); Redis as a backstop only for cross-pod leak detection.
- Recent messages: in-process ring buffer (last 5 minutes) per stream this pod has, plus Redis sorted set as the canonical source for cold-start by viewers on other pods.
- Block list: in-process LRU keyed by user_id, refreshed on stream join.

The general principle: **the pod's hot path must not call out to Redis on every message.** A 1 ms Redis RTT × 10K msg/sec/pod = 10 seconds of latency-equivalent CPU per second per pod. It would melt.

## Backpressure When Fan-Out Saturates

The whole fan-out chain has bounded capacity at every hop. Saturation can happen at:

- **Sequencer outbound queue.** Bus is slow.
- **Bus broker.** Network saturated, broker CPU pegged.
- **Relay outbound queue.** A specific Edge pod is slow.
- **Edge pod outbound queue.** A specific socket is slow.
- **Socket TCP buffer.** Client network is bad.

Each hop needs a strategy. The strategies, in increasing severity:

### 1. Drop oldest

Each subscriber queue is bounded (e.g., 1000 messages or 1 MB). On overflow, drop the oldest message. Send the client a `sampled_burst` event with the dropped count so the UI can show "47 messages skipped".

- **Why oldest, not newest?** New messages are the most relevant. If the client is 30 seconds behind, the old messages are dead anyway.
- **Where:** every queue in the chain — sequencer→bus, bus→relay, relay→Edge, Edge→socket.

### 2. Sample more aggressively

If the relay sees outbound queue depth growing, increase the sampling cutoff (from 5 msg/sec visible to 2 msg/sec). The fan-out cost per message is the same; sampling reduces the messages, not the per-message cost.

- **Where:** at the fan-out relay (after persistence, before delivery). Crucially, sampling here does not affect VOD or moderation; the durable spine has the full record.

### 3. Disconnect slow consumers

A WebSocket whose TCP receive window has been zero for 5 seconds is not coming back. Close it. The client reconnects and cold-starts from the hot cache.

- **Why:** holding the socket open keeps the slow client's queue allocated, eats memory on the Edge pod, and starves the queue's neighbours.
- **Heuristic:** queue depth > 50% of bound for > 5 seconds OR socket has been zero-window for > 3 seconds → close.

### 4. Shed load at the sequencer

If the bus is wedged or the relay tier is degraded, the sequencer refuses new posts (`503 Try Again`). Upstream `429`s and circuit breakers carry the bad news to clients.

- **When:** last resort. This breaks the poster's flow and produces visible incident behaviour. But it is better than buffering messages that will never deliver.

### 5. Stream sharding mid-incident

If one stream's outbound bandwidth saturates a relay, split that stream's fan-out across more relays mid-flight. Effectively, hot-rebalance. Modern systems with consistent-hash-ring placement do this automatically when a relay's CPU exceeds threshold.

### Bandwidth ceiling math

```
Comment payload (compressed)    = ~150 bytes
Viewers                         = 1,000,000
Messages per second visible     = 5 (after sampling)
Per-stream egress               = 150 × 1M × 5 = 750 MB/s = 6 Gbps
```

That 6 Gbps is the lower bound for a single 1M-viewer stream after sampling. Without sampling, at 10K msg/sec, it would be 1.2 TB/s — physically impossible without truly massive Edge fleets, and pointless because no human can read 10K msg/sec.

Backpressure is therefore not optional: at the 1M-viewer scale it is the *primary lever* for staying within the bandwidth budget without dropping the system.

### Failure semantics for the client

Backpressure decisions made server-side need a contract with the client so the UX is honest:

- `sampled_burst { dropped_count, window_ms }` — the server dropped messages by policy. Client shows "47 messages skipped" rather than pretending nothing happened.
- `rate_limit { retry_after_ms }` — the poster's writes are being shed; disable the post button visibly.
- `reconnect { reason: "backpressure" }` — the server proactively closed the socket; client reconnects via the cold-start path, which guarantees no duplicates (Snowflake-ID dedup) and bounded gap (the hot cache covers the last 5 minutes).
- `cooldown { until_ts }` — the stream is in a global cool-down (mod-imposed slow-mode 2.0).

Without these explicit signals the client invents incorrect explanations ("the chat is broken", "the streamer muted me"), users churn, and the operator gets paged for what is actually a normal degradation.

## Industry Examples

### Twitch — Edge + Pubsub + Clue + Room

Twitch's chat infrastructure ([Twitch Engineering: An Introduction and Overview](https://blog.twitch.tv/en/2015/12/18/twitch-engineering-an-introduction-and-overview-a23917b71a25/)) is the canonical reference for tree fan-out. Four components:

- **Edge** speaks IRC over TCP and WebSocket; holds the persistent client connections.
- **Pubsub** is the internal fan-out: Edge nodes subscribe to channel topics, Pubsub broadcasts.
- **Clue** is the moderation/signals service feeding ban / sub status / abuse decisions into the path.
- **Room** keeps viewer lists.

The architecture is "Edge fans in to Pubsub for ingest; Pubsub fans out to Edge for delivery; Edge fans out to clients." The IRC bridge means third-party bots (Streamlabs, Nightbot) can connect over a standard protocol; the bridge translates IRC messages into Twitch's internal protocol. Today the system carries hundreds of billions of messages per day. The original 2015 number was 10 billion/day — that growth shows where the design's headroom came from: it scaled by adding more Edge / Pubsub nodes, not by re-architecting.

### YouTube Live Chat

YouTube uses HTTP long-polling for live chat, not WebSocket — a deliberate trade-off for compatibility with the YouTube embed iframe and corporate proxies. The client polls `liveChatMessages.list` with a `nextPageToken`; the server holds the request open until new messages arrive or a timeout fires. The fan-out is implicit: the request hits a chat shard, which reads from a local message buffer.

- **Strength:** infinite client compatibility (any HTTP/HTTPS client works).
- **Weakness:** per-poll headers are larger than the comment payload at high message rates; this is one reason YouTube samples aggressively on popular streams and surfaces "Top chat" filters.
- **Reference:** [YouTube Live Streaming API: Getting Started](https://developers.google.com/youtube/v3/live/getting-started). The chat-message endpoints (`liveChatMessages`) are part of the same product surface.

### Facebook Live Comments — Wormhole

Facebook's live comments ride on top of the same publication graph that powers the rest of the product. The infrastructure piece worth studying is [Wormhole](https://engineering.fb.com/2013/06/13/data-infrastructure/wormhole-pub-sub-system-moving-data-through-space-and-time/), the inter-DC pub/sub that propagates user-data changes (and, by extension, comment events) across regions. Wormhole's three-component pipeline (Producer → Publisher → Consumer) plus checkpointing and rewind is a textbook regional bridge: at-least-once ordered delivery with ~seconds latency between data centers, handling > 1 trillion messages/day. Live comments specifically use the same fan-out spine plus an in-region distribution tier optimised for low latency.

### Discord — Manifold

Discord ([How Discord Scaled Elixir to 5,000,000 Concurrent Users](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)) is a chat system, not a live-comment system, but the fan-out problem is identical at the guild/channel level. Their `Manifold` library is the explicit implementation of "group destinations by node before sending" that turns N×M sends into N+M. The `GenServer` per guild owns its message ordering and pushes through Manifold to remote nodes; remote nodes do the local fan-out into their connection pools. This is the cleanest published reference for why tree fan-out beats star at scale.

### Slack — Channel Server / Gateway Server

Slack runs millions of concurrent WebSockets. The architecture splits **Channel Server** (writes, ordering, persistence) from **Gateway Server** (the WebSocket front-end). Kafka sits between them as the durable spine. Each Gateway Server holds connections; each Channel Server owns a slice of channels. Cross-server fan-out goes via Kafka topics partitioned by channel. The Slack model is the closest production analogue to the live-comment hot path with a durable spine.

### Comparative table

| System | Hot bus | Durable spine | Ordering owner | Cross-region | Delivery |
|--------|---------|---------------|----------------|--------------|----------|
| Twitch | Pubsub (custom) | (separate analytics path) | Edge per channel | regional Edge tiers | IRC/WebSocket |
| YouTube Live | sharded chat backend | YouTube data infra | per-chat shard | region-pinned typically | HTTPS long-poll |
| Facebook Live | in-DC pub/sub | Wormhole | per-post | Wormhole inter-DC | WebSocket |
| Discord | Erlang BEAM + Manifold | Cassandra (history) | guild GenServer | leaf nodes | WebSocket |
| Slack | Channel/Gateway + Kafka | Kafka | Channel Server | regional clusters | WebSocket |

The shape is the same in all five: one logical owner per stream/channel, a fan-out tier that does the multiplication, an Edge tier that holds connections, and a durable spine that is *not* on the live path. The implementations differ in language, broker, and persistence, but the topology is one design.

## Sampling and Sharding

Even with perfect fan-out, you cannot deliver 10K msg/sec to a human eye. Two reduction techniques exist, and they solve different problems:

### Sampling — pick a representative subset

Sampling reduces the *visible rate* without touching the *ingest rate* or persistence. It runs at the relay, after the message is durably committed.

**Strategies, composable:**

1. **Reservoir sampling per N-second window.** Keep a uniform random sample of `k` messages per second from the messages received in that second; flush. Unbiased, stateless across windows.
2. **Weighted by user signal.** Subscribers, longtime users, moderators get higher weight. Twitch and YouTube both apply this; it surfaces "the chat the streamer probably wants their audience to see" without explicit moderation.
3. **Per-viewer client-side throttle.** Server delivers more than the eye reads; client renders top-K, ages out the rest. Doubles as a survivability mechanism on slow devices.
4. **Trending burst aggregation.** When 5K viewers post "POG" in 200 ms, surface "POG ×5,000" as one aggregate event. This is qualitatively different from sampling because it preserves the *signal* (the audience erupted) while collapsing the *messages*.

**Adaptive thresholds.** Sample only when `incoming_rate > visible_rate_target`. A 50-viewer stream at 2 msg/sec passes everything through; a 1M-viewer stream at 10K msg/sec samples down to 5 visible. The transition is automatic and graceful — no operator flips a flag.

**Where to sample.** At the fan-out relay, never at the sequencer. Sampling at the sequencer would make the sample the only record; VOD replay would be broken; moderators could not audit. The invariant: **the sequencer accepts and persists 100%; the relay decides what reaches the live UI.**

### Sharding — split the stream itself

When a single stream's *write* rate exceeds what one sequencer process can handle, the stream is split into N sub-channels. Viewers see only their shard's chat. This is what Twitch has historically done for very large channels.

- **Hash by user_id** so a given viewer always sees the same shard's chat (consistent experience).
- **Hash by random** if the goal is statistical reduction (the viewer sees a random slice).
- **Loss:** the "shared experience" of a single chat is gone; two viewers in adjacent shards never see the same messages.
- **Win:** scales horizontally without architectural change; adds capacity by adding shards.

Sharding kicks in when ingest exceeds the single-process limit (~tens of thousands of msg/sec). For the typical 1M-viewer / 10K msg/sec stream, sharding is not strictly required — the sequencer can handle it — but combining shard + sample is the only realistic path beyond ~50K msg/sec ingest.

### Combining

```
                     ┌─ Shard 1 (10K writers) ─┐
1M viewers, 10K w/s ─┼─ Shard 2 (10K writers) ─┼─▶ each shard runs its
                     ├─ Shard 3 (10K writers) ─┤    own sequencer + relay
                     └─ Shard 4 (10K writers) ─┘    + sampling at 5 msg/sec
                                                    visible per shard
```

A 4-shard, 5-msg/sec-visible config delivers 20 msg/sec total to the UI, still readable, with each shard scaled independently.

## Anti-Patterns

1. **Star fan-out at the sequencer for hundreds of edges.** The sequencer becomes a connection management problem first and a sequencing problem second. Use a relay tier.
2. **Putting Kafka on the hot path.** Latency budget is shot before the message reaches the relay. Kafka is the spine, not the bus.
3. **Synchronous Redis call per message for slow-mode or rate-limit.** 1 ms × 10K msg/sec = 10 seconds of CPU-equivalent per second per pod. Cache locally with subscribe-and-cache invalidation.
4. **Forwarding every cross-region message every time, regardless of subscribers.** A regional bus must replicate only when the remote region has subscribers (interest-based), not blindly.
5. **One subscription table entry per (stream × user) instead of per stream.** Memory blows up; updates are expensive. Index by stream, fan out within the entry.
6. **No bounded queues anywhere.** One slow consumer eats memory until the pod OOMs. Every queue is bounded; overflow drops or disconnects.
7. **Sampling at the sequencer.** Loses data forever. Sample at the relay; persist 100% on the spine.
8. **Treating reactions/likes as comments.** Multiplies fan-out load 10× for zero user value. Counter pipeline.
9. **Long-polling at default.** At 1M viewers, header overhead alone dominates payload. Use long-poll only as a fallback for proxies that block WebSocket.
10. **One WebSocket per stream per user.** A user in 5 streams = 5 sockets = 5× connection cost. Multiplex on a single WS.
11. **Cross-region writes for streams whose audience is regional.** Most streams' audiences are concentrated in one region; don't pay cross-region RTT on every post for the rare remote viewer.
12. **Hot-rebalancing without idempotency.** When a stream is sharded mid-flight, in-flight messages can deliver twice. Idempotent client-side dedup (by Snowflake ID) is mandatory.
13. **Letting Edge pods subscribe to every stream.** O(streams × pods) message cost. Pods subscribe only to streams whose viewers they currently hold.
14. **No backpressure plan.** Eventually one Edge pod falls behind. Decide in advance: drop oldest, sample, disconnect. Don't OOM.
15. **Treating MQTT QoS 2 as "the right way".** The four-step handshake doubles latency for marginal benefit on a stream where dropped messages are recoverable from cold-start.

## Related

- [Design Live Comments](../design-live-comments.md) — the parent case study; this doc deep-dives on its Fan-Out Architecture section.
- [WhatsApp Group Chat Fan-Out](../whatsapp/group-chat-fanout.md) — sibling deep dive; smaller fan-out (groups ≤ 1024) but full duplex per member.
- [Real-Time Channels — WebSocket, SSE, Long Poll](../../../communication/real-time-channels.md) — the transport choices that drive the Edge tier.
- [Push vs Pull Architecture](../../../communication/push-vs-pull-architecture.md) — why push wins over poll for fan-out.
- [Event-Driven Architecture](../../../communication/event-driven-architecture.md) — the bus pattern at the heart of fan-out.
- [Stream Processing](../../../communication/stream-processing.md) — moderation, analytics, and VOD indexers consume the durable spine downstream.

## References

- [Twitch Engineering: An Introduction and Overview](https://blog.twitch.tv/en/2015/12/18/twitch-engineering-an-introduction-and-overview-a23917b71a25/) — Edge / Pubsub / Clue / Room split; IRC bridge over TCP and WebSocket; "10 billion messages a day" baseline.
- [Twitch State of Engineering 2023](https://blog.twitch.tv/en/2023/09/28/twitch-state-of-engineering-2023/) — current scale and operational evolution.
- [How Discord Scaled Elixir to 5,000,000 Concurrent Users](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users) — `Manifold` relay, BEAM fan-out, the tree-vs-star case study in production.
- [How Discord Stores Billions of Messages](https://discord.com/blog/how-discord-stores-billions-of-messages) — Cassandra-backed durable spine; Snowflake IDs.
- [Wormhole pub/sub system, moving data through space and time](https://engineering.fb.com/2013/06/13/data-infrastructure/wormhole-pub-sub-system-moving-data-through-space-and-time/) — Facebook's inter-DC publication system underpinning Live Comments fan-out at trillion-message-per-day scale.
- [Slack Engineering — Real-Time Messaging](https://slack.engineering/real-time-messaging/) — Channel Server / Gateway Server architecture; Kafka spine.
- [How Slack Supports Billions of Daily Messages — ByteByteGo](https://blog.bytebytego.com/p/how-slack-supports-billions-of-daily) — the fan-out pipeline in production.
- [YouTube Live Streaming API: Getting Started](https://developers.google.com/youtube/v3/live/getting-started) — `liveBroadcast`, `liveStream`, `cuepoint`; the surface that hosts live-chat features.
- [MQTT Version 5.0 OASIS Standard](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html) — topic tree, QoS levels, retained messages, shared subscriptions.
- [NATS Subject-Based Messaging](https://docs.nats.io/nats-concepts/subjects) — subject hierarchy, wildcards, interest-based routing.
- [NATS JetStream](https://docs.nats.io/nats-concepts/jetstream) — durability layer over NATS core; replay, retention, consumer types.
- [Redis Pub/Sub](https://redis.io/docs/latest/develop/interact/pubsub/) — `PUBLISH`, `SUBSCRIBE`, `PSUBSCRIBE` semantics; ephemeral, fire-and-forget delivery.
- [Cloudflare Durable Objects](https://developers.cloudflare.com/durable-objects/) — one-object-per-room model with WebSocket Hibernation; isolation by design.
- [Phoenix Channels](https://hexdocs.pm/phoenix/channels.html) — pub/sub topology, transport fallback, Phoenix.PubSub Redis adapter.
