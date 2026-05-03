---
title: "Distributed Transactions — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, distributed-systems, transactions, consistency, system-design]
---

# Distributed Transactions — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `distributed-systems` `transactions` `consistency` `system-design`

---

## Table of Contents
1. [How to use this doc](#how-to-use-this-doc)
2. [Easy (warm-up traps)](#1-easy-warm-up-traps)
3. [Subtle (review-passers)](#2-subtle-review-passers)
4. [Senior trap (production-only failures)](#3-senior-trap-production-only-failures)
5. [Solutions](#4-solutions)
6. [Related](#related)
7. [References](#references)

## Summary

Active-recall practice for distributed-transaction failure modes: dual writes, outbox/inbox plumbing, sagas, 2PC coordinator failures, idempotency, exactly-once myths, leader election, fencing tokens, split-brain, clock skew, and CRDT/quorum traps. Snippets are short and language-light (pseudo-code or Java-flavored) so the reader focuses on the concurrency story, not the syntax. Aim for one section per practice session; revisit the whole doc when reviewing any "transactional" cross-service work.

## How to use this doc
- Read the snippet, predict what breaks under crash/partition/retry, then peek at the hint.
- Hints are one-liners. Full root cause + fix lives in §4 Solutions, keyed by bug number.
- Sections are cumulative — Easy bugs still appear in production code review every week.

---

## 1. Easy (warm-up traps)

### Bug 1 — Dual write to DB then Kafka
```java
@Transactional
public void placeOrder(Order o) {
    orderRepo.save(o);            // committed when method returns
    kafka.send("orders", o.id);   // separate network call
}
```
<details><summary>Hint</summary>
Process can die between commit and send — or send can succeed and commit roll back.
</details>

### Bug 2 — Outbox without transactional insert
```java
public void createPayment(Payment p) {
    paymentRepo.save(p);                    // tx 1
    outboxRepo.save(new OutboxEvent(p));    // tx 2 (separate transaction)
}
```
<details><summary>Hint</summary>
The whole point of an outbox is the *single* local transaction.
</details>

### Bug 3 — Idempotency key = request UUID
```java
String idempotencyKey = UUID.randomUUID().toString();
httpClient.post("/charge", body, headers.with("Idempotency-Key", idempotencyKey));
```
<details><summary>Hint</summary>
What value does the *retry* send?
</details>

### Bug 4 — Distributed lock with no owner ID
```python
# Acquire
redis.set("lock:job-42", "1", nx=True, ex=30)
do_work()
redis.delete("lock:job-42")
```
<details><summary>Hint</summary>
Whose lock are you releasing?
</details>

### Bug 5 — "Exactly-once" claim
> "Our message bus guarantees exactly-once delivery, so consumers don't need dedup."

<details><summary>Hint</summary>
Read the broker docs again — what does it actually guarantee end-to-end?
</details>

### Bug 6 — Last-write-wins on a counter
```java
int balance = accountRepo.find(id).balance;
accountRepo.update(id, balance + delta);   // two concurrent callers
```
<details><summary>Hint</summary>
Lost update — and LWW won't save you.
</details>

### Bug 7 — Saga step with no compensation
```text
Saga: createOrder → reserveInventory → chargeCard → shipOrder
Compensations defined: cancelOrder, releaseInventory, refundCard
```
<details><summary>Hint</summary>
Count the steps and count the compensations.
</details>

---

## 2. Subtle (review-passers)

### Bug 8 — Outbox publisher without dedup downstream
```java
// Publisher polls outbox, sends to Kafka, marks row sent.
List<OutboxEvent> rows = outboxRepo.findUnsent(100);
rows.forEach(r -> { kafka.send(r); outboxRepo.markSent(r.id); });
```
<details><summary>Hint</summary>
What if the process dies between `kafka.send` and `markSent`?
</details>

### Bug 9 — Saga compensation that isn't commutative
```text
Step:    decrement(inventory, 5)
Compensate: increment(inventory, 5)
```
Retry of the original step happens *during* compensation.
<details><summary>Hint</summary>
What if the original is replayed after the compensation lands?
</details>

### Bug 10 — 2PC coordinator crash
Participant A and B have voted YES and are in the `prepared` state. Coordinator crashes before sending COMMIT or ABORT.
<details><summary>Hint</summary>
What state are A and B's locks in until a human shows up?
</details>

### Bug 11 — Distributed lock TTL with long work
```python
redis.set("lock:rebuild", owner, nx=True, ex=30)
rebuild_index()  # usually 10s, occasionally 90s
redis.delete_if_owner("lock:rebuild", owner)
```
<details><summary>Hint</summary>
The TTL fires while the work is still running.
</details>

### Bug 12 — Leader election with no fencing token
```text
1. Node A acquires leader lock.
2. A pauses (long GC).
3. Lock TTL expires; B becomes leader.
4. A resumes and writes to storage as "leader".
```
<details><summary>Hint</summary>
Storage has no idea A's lease is dead.
</details>

### Bug 13 — Quorum read without quorum write
```text
N=3, R=2, W=1
```
<details><summary>Hint</summary>
What's R + W vs N?
</details>

### Bug 14 — Inbox dedup with too-short TTL
```java
if (inbox.seenWithin(messageId, Duration.ofMinutes(10))) return;
inbox.record(messageId, Duration.ofMinutes(10));
process(message);
```
Producer retries an outage-buffered message 30 minutes later.
<details><summary>Hint</summary>
Measure the longest possible retry delay and compare.
</details>

### Bug 15 — TCC confirm assumed idempotent
```text
Try:    reserve $100 in escrow
Confirm: move $100 escrow → merchant
Cancel:  release $100 escrow
```
Confirm is invoked twice due to network retry.
<details><summary>Hint</summary>
What does the second `Confirm` do if the escrow is already empty?
</details>

### Bug 16 — Clock-based ordering across machines
```java
event.timestamp = System.currentTimeMillis();
events.sortBy(e -> e.timestamp);
```
<details><summary>Hint</summary>
Two machines, NTP skew of tens of milliseconds, possibly negative.
</details>

### Bug 17 — Read-your-writes after async replication
```text
POST /profile         → primary
GET  /profile (200ms) → replica (lag 800ms sometimes)
```
<details><summary>Hint</summary>
What does the user see right after editing their bio?
</details>

### Bug 18 — Vector clock dropped at service hop
```text
ServiceA --(v.clock=[A:3,B:1])--> ServiceB
ServiceB --(no v.clock header) --> ServiceC
```
<details><summary>Hint</summary>
ServiceC reasons about ordering using what?
</details>

### Bug 19 — CDC consumer reprocesses on rebalance
```java
// Kafka Connect → downstream that does:
upsert(row);
counter.increment();   // not idempotent
```
<details><summary>Hint</summary>
At-least-once delivery + non-idempotent side effect.
</details>

### Bug 20 — Counter CRDT used where set semantics needed
```text
"Likes" stored as a G-Counter.
User unlikes, then likes again.
```
<details><summary>Hint</summary>
Can a G-Counter decrement? What's the actual data model?
</details>

---

## 3. Senior trap (production-only failures)

### Bug 21 — Split-brain after partition heals
Two-node cluster, network partitions. Each side promotes itself to primary using a local timeout. Partition heals.
<details><summary>Hint</summary>
Whose writes win? More importantly — are *any* of them safe?
</details>

### Bug 22 — Saga with non-compensatable side effect
```text
Steps: createOrder → chargeCard → sendConfirmationEmail → reserveStock
```
Step 4 fails. Compensations run.
<details><summary>Hint</summary>
You can refund the card. Can you un-send the email?
</details>

### Bug 23 — "Strong consistency" claim with async replication
Vendor docs: "strongly consistent reads." Architecture: writes to leader, reads from any replica with `replication_lag_ms < 50`.
<details><summary>Hint</summary>
Strong consistency is not "almost up to date."
</details>

### Bug 24 — Read repair masks a convergence bug
Quorum reads trigger read repair on every divergence. Test suite is green because read repair always papers over the inconsistency before the assertion runs.
<details><summary>Hint</summary>
What does the bug look like when read-repair is disabled?
</details>

### Bug 25 — Partition tolerance silently abandoned
Service uses a CP store. Under load, the client switches to a "fast path" that reads from a local cache populated by an async stream. This switch is configured at the load balancer, not in code review.
<details><summary>Hint</summary>
The CAP claim of the *system* now depends on a flag nobody owns.
</details>

---

## 4. Solutions

### Bug 1 — Dual write to DB then Kafka
**Root cause:** No atomic boundary spans the DB commit and the Kafka publish. Crash between them produces a committed order with no event (lost event), or a published event with no row (phantom event). This is the canonical "dual-write problem."
**Fix:** Use the **Transactional Outbox** pattern: insert the event row into an `outbox` table inside the same DB transaction as the order. A separate relay (CDC like Debezium, or a poller) ships the row to Kafka and marks it sent.
**Reference:** Chris Richardson, *Pattern: Transactional outbox* — https://microservices.io/patterns/data/transactional-outbox.html

### Bug 2 — Outbox without transactional insert
**Root cause:** Outbox only works if the business write and the outbox write commit atomically. Two separate transactions reintroduce the dual-write race.
**Fix:** Single `@Transactional` boundary that includes both the entity persist and the outbox row.
```java
@Transactional
public void createPayment(Payment p) {
    paymentRepo.save(p);
    outboxRepo.save(new OutboxEvent(p)); // same tx
}
```
**Reference:** Kleppmann, *DDIA* Ch. 11 ("Stream Processing — Maintaining Derived State"); microservices.io/patterns/data/transactional-outbox.html.

### Bug 3 — Idempotency key = request UUID
**Root cause:** A fresh UUID per attempt defeats idempotency — every retry looks like a new request to the server. The key must be stable across retries of the *same logical operation*.
**Fix:** Generate the key once at the business-operation level (e.g., per checkout intent) and reuse it on every retry. Stripe's API guide explicitly calls this out.
**Reference:** Stripe API — *Idempotent Requests* — https://stripe.com/docs/api/idempotent_requests

### Bug 4 — Distributed lock with no owner ID
**Root cause:** `DEL lock:job-42` deletes the key regardless of who owns it. After a TTL-induced expiry+reacquire, the original holder's `DEL` releases the new owner's lock.
**Fix:** Store an owner token as the value and release with a Lua script that checks-and-deletes atomically.
```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else return 0 end
```
**Reference:** Redis docs, *Distributed locks with Redis* — https://redis.io/docs/latest/develop/use/patterns/distributed-locks/

### Bug 5 — "Exactly-once" claim
**Root cause:** End-to-end exactly-once delivery is impossible in an asynchronous network with failures (FLP, plus the well-known "two generals" intuition). Brokers like Kafka offer *exactly-once semantics* only within a transactional producer→broker→consumer chain on Kafka itself, and only for idempotent operations on the consumer side.
**Fix:** Design for at-least-once + idempotent consumers (dedup by message id with sufficient retention). Treat any vendor "exactly-once" badge as scoped marketing.
**Reference:** Mathias Verraes, *The Two Generals Problem* (and Kafka KIP-98 transactional spec); see also Tyler Treat, "You Cannot Have Exactly-Once Delivery" — https://bravenewgeek.com/you-cannot-have-exactly-once-delivery/

### Bug 6 — Last-write-wins on a counter
**Root cause:** Read-modify-write race. Two writers each read `100`, both write `110`, one increment lost.
**Fix:** Use atomic increment (`UPDATE ... SET balance = balance + ?`), optimistic concurrency with version column, or a CRDT counter for multi-master setups.
**Reference:** Kleppmann, *DDIA* Ch. 7 ("Transactions — Lost Updates").

### Bug 7 — Saga step with no compensation
**Root cause:** `shipOrder` has no compensation. If a later step fails (or shipOrder itself fails partway), the saga can't unwind. The original Sagas paper requires every step to have a compensating action *or* be designed as a pivot/non-revocable terminal step.
**Fix:** Either add a `cancelShipment` compensation or restructure the saga so shipping is the final, non-revocable step (a "pivot transaction").
**Reference:** Garcia-Molina & Salem, *Sagas* (1987) — https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf

### Bug 8 — Outbox publisher without dedup downstream
**Root cause:** Crash between `kafka.send` and `markSent` causes the next poll to resend the event. Consumer sees duplicates.
**Fix:** At-least-once is the contract; consumers must dedup using the outbox row id (a stable message key). Pair with idempotent handlers.
**Reference:** Debezium docs, *Outbox Event Router* — https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html

### Bug 9 — Saga compensation that isn't commutative
**Root cause:** If the saga retries the original step *after* the compensation runs (network reorder, redelivery), naive `+5/-5` arithmetic breaks: inventory ends up wrong or negative. Compensations must be **semantically commutative and idempotent** with respect to retried originals.
**Fix:** Tag every action with the saga instance id; compensations check whether the original was applied before reversing. Or use *semantic locks* / pivot transactions to prevent overlap.
**Reference:** Chris Richardson, *Pattern: Saga* — https://microservices.io/patterns/data/saga.html (countermeasures section).

### Bug 10 — 2PC coordinator crash
**Root cause:** Participants in the `prepared` phase hold locks awaiting a coordinator decision. With the coordinator dead, locks are held indefinitely — this is the classic 2PC blocking problem.
**Fix:** Use a coordinator with durable log and recovery (XA recovery), or move to a non-blocking protocol (3PC variants are theoretically possible but rarely used; in practice, prefer sagas or consensus-based commit).
**Reference:** Kleppmann, *DDIA* Ch. 9 ("Atomic Commit and Two-Phase Commit"); Pat Helland, *Life Beyond Distributed Transactions* — https://queue.acm.org/detail.cfm?id=3025012

### Bug 11 — Distributed lock TTL with long work
**Root cause:** TTL-bounded locks assume work duration < TTL. When work overruns, another worker acquires the lock and you have two workers in the critical section. The owner check from Bug 4 prevents accidental release but doesn't prevent concurrent execution.
**Fix:** Auto-renew (heartbeat) the lock from a watchdog thread *and* protect the protected resource with fencing tokens (Bug 12). Never rely on lock-only safety for correctness.
**Reference:** Kleppmann, *How to do distributed locking* — https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html

### Bug 12 — Leader election with no fencing token
**Root cause:** A paused leader resumes and writes after its lease has been reassigned. Storage has no way to know the writes are stale.
**Fix:** Lock service issues monotonically-increasing fencing tokens; storage rejects writes with a token lower than the highest seen.
```text
A acquires lease, gets token=33.
A pauses; B acquires lease, gets token=34, writes with 34.
A resumes, writes with 33 → storage rejects.
```
**Reference:** Kleppmann, *How to do distributed locking* — https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html

### Bug 13 — Quorum read without quorum write
**Root cause:** Dynamo-style consistency requires `R + W > N` to guarantee read-after-write. With `R=2, W=1, N=3`, the read quorum can miss the only replica that has the latest write.
**Fix:** Set `W=2` (so `R+W=4 > N=3`) or `R=3` for strong reads. Document the SLA.
**Reference:** Kleppmann, *DDIA* Ch. 5 ("Quorums for reading and writing"); DeCandia et al., *Dynamo* paper.

### Bug 14 — Inbox dedup with too-short TTL
**Root cause:** Dedup window must exceed the maximum possible redelivery delay (broker retention + producer retry + outage buffering). 10-minute window vs 30-minute redelivery → duplicate processed.
**Fix:** Size the dedup TTL to match the broker's max redelivery window plus a margin, or persist dedup state durably and trim by id watermark instead of wall-clock TTL.
**Reference:** Pat Helland, *Idempotence Is Not a Medical Condition* — https://queue.acm.org/detail.cfm?id=2187821

### Bug 15 — TCC confirm assumed idempotent
**Root cause:** A naive `Confirm` that "moves escrow→merchant" double-pays on retry if it doesn't check whether it's already been applied.
**Fix:** Persist a `(txId, state)` record. `Confirm` is a state transition: only `TRYING → CONFIRMED` performs the move; `CONFIRMED → CONFIRMED` is a no-op.
**Reference:** Pat Helland, *Life Beyond Distributed Transactions* (entity-state idempotence) — https://queue.acm.org/detail.cfm?id=3025012

### Bug 16 — Clock-based ordering across machines
**Root cause:** Wall-clock timestamps from different hosts can disagree by tens of ms (or jump backward on NTP correction). Sorting by them produces causally-wrong orders.
**Fix:** Use logical clocks (Lamport, vector) for causal ordering; or hybrid logical clocks (HLC) when you need approximate wall-clock + causality. For globally-ordered commits, see Spanner TrueTime.
**Reference:** Spanner paper — https://research.google/pubs/spanner-googles-globally-distributed-database-2/ ; Lamport, *Time, Clocks, and the Ordering of Events* (CACM 1978).

### Bug 17 — Read-your-writes after async replication
**Root cause:** Replica lag means the user reads the previous version of their own profile. RYW is a session-level consistency model that async replication does not provide by default.
**Fix:** Route the user's reads to the primary for a "stickiness window" after a write, or pass a write-version cookie/LSN that the replica must catch up to before serving.
**Reference:** Kleppmann, *DDIA* Ch. 5 ("Reading Your Own Writes").

### Bug 18 — Vector clock dropped at service hop
**Root cause:** Causal metadata must propagate end-to-end. A service that strips it from outgoing requests breaks downstream causal tracking and any conflict resolution that depends on it.
**Fix:** Treat the vector clock (or trace context, or `Last-Event-ID`) as a first-class header that frameworks propagate by default — same discipline as W3C `traceparent`.
**Reference:** Kleppmann, *DDIA* Ch. 5 ("Detecting Concurrent Writes" / "Version Vectors"); W3C Trace Context — https://www.w3.org/TR/trace-context/

### Bug 19 — CDC consumer reprocesses on rebalance
**Root cause:** Kafka rebalance can replay messages from the last committed offset. A non-idempotent side effect (counter increment) produces duplicates.
**Fix:** Make handlers idempotent (upsert by primary key, dedup table keyed by event id, or transactional consume-process-produce with Kafka EOS).
**Reference:** Confluent, *Exactly-Once Semantics in Apache Kafka* — https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/

### Bug 20 — Counter CRDT used where set semantics needed
**Root cause:** A G-Counter is grow-only; PN-Counter allows decrement but still represents a count, not membership. "Likes" by user is a set: it needs an OR-Set (or 2P-Set) so unlike+like is observably idempotent and commutative.
**Fix:** Model likes as `Set<UserId>` with OR-Set semantics; the displayed count is a derivation (`size`), not the source of truth.
**Reference:** Shapiro et al., *A comprehensive study of Convergent and Commutative Replicated Data Types* (INRIA, 2011) — https://hal.inria.fr/inria-00555588/document

### Bug 21 — Split-brain after partition heals
**Root cause:** Both halves elected leaders independently; both accepted writes; neither half's writes can be safely merged without a conflict-resolution policy. This is split-brain.
**Fix:** Require quorum-based election (Raft/Paxos) so a minority partition cannot elect a leader. Add fencing tokens so old leaders can't write after demotion. Plan a manual reconciliation policy for the (rare) case of true split-brain — usually involves picking a survivor and replaying the loser's writes.
**Reference:** Ongaro & Ousterhout, *Raft* paper — https://raft.github.io/raft.pdf ; Kleppmann, *DDIA* Ch. 8/9.

### Bug 22 — Saga with non-compensatable side effect
**Root cause:** Email send is irreversible. Once `sendConfirmationEmail` runs, no compensation can un-send it; "We're sorry, please ignore the previous email" is a poor user experience and a contract risk.
**Fix:** Reorder the saga so all compensatable steps come first and irreversible ones become *pivot transactions* — only execute after the saga is guaranteed to commit. Or split the email into "intent recorded" → defer actual send to a post-commit handler.
**Reference:** Chris Richardson, *Pattern: Saga* — pivot transactions section — https://microservices.io/patterns/data/saga.html ; Garcia-Molina & Salem, *Sagas* (1987).

### Bug 23 — "Strong consistency" claim with async replication
**Root cause:** "Strong consistency" (linearizability) means every read returns the latest committed value. Bounded-staleness reads can return values seconds old — that's a different (weaker) consistency model.
**Fix:** Either (a) route strong reads to the leader and accept the latency, or (b) honestly document the model as bounded-staleness / read-your-writes / monotonic-reads. Jepsen has repeatedly found vendors mis-marketing here.
**Reference:** Jepsen, *MongoDB 4.2.6 analysis* — https://jepsen.io/analyses/mongodb-4.2.6 (multiple snapshot-isolation violations despite "ACID" marketing).

### Bug 24 — Read repair masks a convergence bug
**Root cause:** Read repair runs on the read path and silently fixes divergence before the application sees it. Tests pass; under load with read repair throttled or anti-entropy disabled, the bug surfaces.
**Fix:** Add tests with read repair disabled and anti-entropy disabled. Use Jepsen-style fault-injection to verify the underlying replication is correct, not just the read path.
**Reference:** Kingsbury, *Jepsen* analyses (e.g., Cassandra 4.0.0) — https://jepsen.io/analyses/cassandra-4.0.0

### Bug 25 — Partition tolerance silently abandoned
**Root cause:** A "CP" architecture relied on a fast-path cache populated asynchronously. Under partition between cache and source-of-truth, the system continues serving stale-but-fast reads — silently degrading from CP to AP without telling consumers. The CAP claim is undefined when the failover path isn't audited.
**Fix:** Make consistency guarantees a system-level invariant tested under partition (chaos testing). Any "fast path" that bypasses the consistency guarantee must be explicit, opt-in, and surfaced in the SLA. Pat Helland's framing: durable state lives at the edge; everything inside is best-effort, and you must know which is which.
**Reference:** Pat Helland, *Standing on Distributed Shoulders of Giants* — https://queue.acm.org/detail.cfm?id=3056634 ; Kleppmann, *Please stop calling databases CP or AP* — https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html

---

## Related
- [Distributed transactions](./distributed-transactions.md) — concept reference for 2PC, sagas, TCC.
- [Consensus: Raft and Paxos](./consensus-raft-and-paxos.md) — quorum, leader election, fencing.
- [Idempotency and exactly-once](../communication/idempotency-and-exactly-once.md) — companion piece on dedup and retry semantics.
- [Network partitions and split-brain](../reliability/network-partitions-and-split-brain.md) — partition behaviors that drive most of these bugs.
- [Time and ordering](./time-and-ordering.md) — Lamport, vector, hybrid logical clocks.

## References

**Foundational papers**
- Garcia-Molina & Salem, *Sagas* (SIGMOD 1987) — https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf
- Lamport, *Time, Clocks, and the Ordering of Events in a Distributed System* (CACM 1978).
- Ongaro & Ousterhout, *In Search of an Understandable Consensus Algorithm (Raft)* — https://raft.github.io/raft.pdf
- Corbett et al., *Spanner: Google's Globally Distributed Database* — https://research.google/pubs/spanner-googles-globally-distributed-database-2/
- Shapiro et al., *A comprehensive study of CRDTs* (INRIA) — https://hal.inria.fr/inria-00555588/document

**Pat Helland (essential reading)**
- *Life Beyond Distributed Transactions* — https://queue.acm.org/detail.cfm?id=3025012
- *Idempotence Is Not a Medical Condition* — https://queue.acm.org/detail.cfm?id=2187821
- *Standing on Distributed Shoulders of Giants* — https://queue.acm.org/detail.cfm?id=3056634

**Books**
- Martin Kleppmann, *Designing Data-Intensive Applications* — Ch. 5 (Replication), Ch. 7 (Transactions), Ch. 8 (Trouble), Ch. 9 (Consistency & Consensus), Ch. 11 (Stream Processing).

**Patterns**
- Chris Richardson, *Saga pattern* — https://microservices.io/patterns/data/saga.html
- Chris Richardson, *Transactional outbox pattern* — https://microservices.io/patterns/data/transactional-outbox.html
- Debezium, *Outbox Event Router* — https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html

**Jepsen and analyses**
- Jepsen, *MongoDB 4.2.6* — https://jepsen.io/analyses/mongodb-4.2.6
- Jepsen, *Cassandra 4.0.0* — https://jepsen.io/analyses/cassandra-4.0.0
- Kleppmann, *How to do distributed locking* — https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
- Kleppmann, *Please stop calling databases CP or AP* — https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html

**Vendor docs**
- Stripe, *Idempotent Requests* — https://stripe.com/docs/api/idempotent_requests
- Redis, *Distributed locks* — https://redis.io/docs/latest/develop/use/patterns/distributed-locks/
- Confluent, *Exactly-Once Semantics in Kafka* — https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/
- W3C, *Trace Context* — https://www.w3.org/TR/trace-context/

**Other**
- Tyler Treat, *You Cannot Have Exactly-Once Delivery* — https://bravenewgeek.com/you-cannot-have-exactly-once-delivery/
