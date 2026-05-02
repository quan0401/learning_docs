---
title: "Leader Election for the Scheduler — Single Decider, HA Failover, and Sharded Schedulers"
date: 2026-05-01
updated: 2026-05-01
tags: [system-design, deep-dive, scheduler, leader-election, consensus]
---

# Leader Election for the Scheduler — Single Decider, HA Failover, and Sharded Schedulers

**Date:** 2026-05-01 | **Updated:** 2026-05-01
**Tags:** `system-design` `deep-dive` `scheduler` `leader-election` `consensus`

> **Parent case study:** [Design a Distributed Job Scheduler](../design-job-scheduler.md). This deep-dive expands §7.7 "Leader election for the scheduler itself."

## Table of Contents

- [Summary](#summary)
- [Why a Singleton Decider](#why-a-singleton-decider)
- [Active-Passive vs Active-Active](#active-passive-vs-active-active)
- [Election Backends: ZooKeeper, etcd, Consul, DynamoDB, Raft-Embedded](#election-backends-zookeeper-etcd-consul-dynamodb-raft-embedded)
- [Lease-Based Leadership and Renewal Cadence](#lease-based-leadership-and-renewal-cadence)
- [Split Brain: How Two Leaders Happen Anyway](#split-brain-how-two-leaders-happen-anyway)
- [Fencing Tokens: The Only Thing That Saves You](#fencing-tokens-the-only-thing-that-saves-you)
- [Code: ZooKeeper Sequential-Ephemeral Election](#code-zookeeper-sequential-ephemeral-election)
- [Code: etcd Lease + CompareAndSwap Election](#code-etcd-lease--compareandswap-election)
- [Code: Heartbeat Loop with Self-Fencing on Missed Renewal](#code-heartbeat-loop-with-self-fencing-on-missed-renewal)
- [Sharded Schedulers: One Leader per Shard](#sharded-schedulers-one-leader-per-shard)
- [Graceful Hand-Off vs Crash Failover](#graceful-hand-off-vs-crash-failover)
- [Failover SLO: How Fast Is Fast Enough?](#failover-slo-how-fast-is-fast-enough)
- [Decider vs Executor: Don't Stack Them on the Same Process](#decider-vs-executor-dont-stack-them-on-the-same-process)
- [Observability: Who Is the Leader Right Now?](#observability-who-is-the-leader-right-now)
- [Worked Example: Three-Instance Scheduler, Leader Dies](#worked-example-three-instance-scheduler-leader-dies)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

The parent case study mentions in roughly thirty lines that the scheduler tier "must not double-fire triggers" and that production stacks reach for ZooKeeper / etcd / Consul leases or Raft-embedded engines like Temporal. That sketch is correct but elides the engineering that makes the pattern work: choosing a lease TTL that survives realistic GC pauses without making failover unbearably slow, fencing every downstream side effect against a still-running deposed leader, partitioning the schedule space so 100 shards run 100 leaders without any of them losing single-writer semantics, distinguishing the *decider* (the dispatch loop) from the *executor* (the worker fleet) so the leader doesn't drown when scale arrives, and watching telemetry that can detect the silent failure mode where two instances both think they're leader for thirty seconds. The recurring theme: leader election is the easy part. **What makes a scheduler safe is fencing, not the lock service.** A correct lease holder with no fencing token is one GC pause away from duplicate fires; a stale lease holder with fencing is harmless. This deep-dive walks the failure modes, the primitives that prevent them, and the trade-offs you actually live with in production.

## Why a Singleton Decider

The scheduler tier has exactly one job that cannot be done concurrently: deciding *now is the time to fire trigger T*. Everything before that decision (storing the trigger definition) and after it (executing the job) parallelizes cleanly. The decision itself does not.

Consider what "fire now" means:

1. Read the next-due trigger from `schedules` table.
2. Check that no other instance has fired it (lock or row-version CAS).
3. Insert a `runs` row, advance `next_fire_at`.
4. Hand off to the executor (enqueue to worker queue).

If two scheduler instances run that loop simultaneously and the trigger's "fire window" matches a polling cycle on both, you have a race. With the per-job advisory lock from [`./distributed-lock-per-job.md`](./distributed-lock-per-job.md), step 2 may catch the duplicate — but only if both instances are honest about the lock. A bug, a clock skew, a partitioned network where instance A still believes it is sole owner, and the lock isn't enough.

The simpler, stronger invariant is *only one instance ever runs the dispatch loop*. That is what leader election buys: a coordinator that says "you (one specific process) own the dispatch loop until further notice; the rest of you are warm spares." Per-job locking remains, as defence in depth, but the load on it is one writer instead of N.

A useful framing: per-job locking solves the problem of *two attempts to fire the same trigger*; leader election solves the problem of *two dispatch loops looking at the same partition*. Both are needed because either alone has a hole. A leader can be ousted mid-fire while still holding open Postgres connections, and the new leader will scan the same row before the old leader's transaction commits. Without a per-job lock, you double-fire. Without leader election, you have N polling loops contending on every dispatch row, which is both correctness-fragile and wasteful.

## Active-Passive vs Active-Active

There are two coherent architectures, and one architecture that looks coherent but isn't.

**Active-passive.** One node holds the lease and runs the dispatch loop. The others sit idle, periodically attempting to acquire the lease. When the leader's lease expires (it crashed, was killed, network-partitioned), one of the standbys wins the next election round and takes over. This is the model used by Apache Airflow's HA scheduler (since 2.0), Quartz with `JobStoreCMT`, Kubernetes controller-manager, and most production schedulers.

- **Pros.** Simple. One writer means no coordination during normal operation. The standby's state is empty or a hot replica; promotion is fast.
- **Cons.** All dispatch throughput on one box. The failover gap is downtime for any fires that happen during it. CPU on the standby is wasted (90% idle).

**Active-active sharded.** Partition the schedule space (typically `hash(tenant_id) % N` or `hash(schedule_id) % N`) and run an independent active-passive election per shard. Each shard has one leader at a time, but the leaders are spread across the fleet. With 100 shards and a 10-instance fleet, each instance leads 10 shards on average; if an instance dies, its 10 shards fail over to the survivors. Throughput scales with N; no single box bottlenecks dispatch.

- **Pros.** Throughput scales horizontally. Failover blast radius is one shard's worth of work, not the entire scheduler.
- **Cons.** More moving parts. Rebalancing shards on resize is a real operational concern (see §[Sharded Schedulers](#sharded-schedulers-one-leader-per-shard)). Observability has to track N elections instead of 1.

**The wrong shape: active-active without partitioning.** Two scheduler instances both run the dispatch loop, both compete for per-job locks, "and the lock service handles deduplication." This works in steady state and breaks during failover, GC pauses, partitions, or any other moment the lock service's view of who-holds-what skews from reality. Don't build this. The per-job lock is defence in depth, not a substitute for sequencing the dispatch loop.

## Election Backends: ZooKeeper, etcd, Consul, DynamoDB, Raft-Embedded

The election primitive is the durable shared state where "I am the leader" is recorded. Five common choices:

| Backend | Mechanism | Failure detection | Typical use |
|---|---|---|---|
| **ZooKeeper** | Ephemeral sequential znode under `/leader/<shard>`; smallest sequence number wins | Session timeout (heartbeat-based) | Hadoop/HBase ecosystem, Solr, Kafka (legacy) |
| **etcd** | Lease + KV `Put` with `IfNotExists` (`PutOptions.WithLease`); holder renews lease | Lease TTL expiry | Kubernetes, CockroachDB, modern stacks |
| **Consul** | Session + KV `acquire` (lock-style) | Session TTL + health check | HashiCorp stack, service-mesh-adjacent systems |
| **AWS DynamoDB** | Conditional `PutItem`/`UpdateItem` with `attribute_not_exists` or version CAS | TTL attribute, periodic renewal | AWS-native systems, multi-region with global tables |
| **Raft-embedded** | The scheduler itself runs Raft (Hashicorp Raft, etcd Raft, Temporal's history shards) | Raft heartbeats | Temporal, Cadence, CockroachDB, anything that wants consensus on more than just leadership |

The first four are *external* coordination services: the scheduler is a client, the service holds the truth. Lightweight, well-understood, easy to reason about. The last is structurally different: the scheduler itself becomes a Raft cluster, and leadership is a free side effect of the consensus layer. Temporal and Cadence go this route because they need replicated state for workflow execution, not just for the dispatch decision. For most schedulers, embedding Raft is more weight than necessary — a separate etcd or ZooKeeper is operationally simpler.

**Kubernetes leader-election library.** A worth-special-mention case: Kubernetes ships a client-go library (`tools/leaderelection`) that implements lease-based election against the Kubernetes API server itself, using a `Lease` object as the coordination primitive. If your scheduler runs as a Kubernetes Deployment with multiple replicas, this is often the path of least resistance — the API server is already there, the library is well-tested, and you do not deploy a separate ZooKeeper / etcd cluster just for this. Behind the scenes, a `Lease` object is just a row in etcd that the API server writes; you are using etcd, you are just using it through the Kubernetes API.

**The thing they all share.** Every backend on this list provides the same logical contract: a *lease* — a time-bounded promise of exclusive access — plus the ability to *renew* the lease before it expires. Leadership is "I hold a current lease." Election is "the lease was free and I took it." Failover is "the lease expired and I took it." Different APIs, same shape.

## Lease-Based Leadership and Renewal Cadence

Two numbers define the safety/liveness trade-off: the lease TTL `T` and the renewal interval `R`. The leader writes a renewal every `R` seconds; the lease expires `T` seconds after the last successful renewal.

- **Renewal cadence `R`.** How often the leader proves liveness. Typically 1/3 to 1/5 of `T`. With `T = 30s` and `R = 10s`, you have three chances to renew before the lease lapses; one network blip is survivable.
- **Lease TTL `T`.** How long the lease is valid. Sets both the *failover gap* (after the leader dies, new leader cannot start until the lease expires) and the *split-brain window* (how long an old leader might still believe it's the leader after losing the lease).

Trade-offs:

- **Short TTL (e.g. 5s, R=1s).** Fast failover (5s of downtime). But under a 6-second JVM stop-the-world GC pause, the leader misses two renewals; the lease expires; a standby takes over; the original leader resumes after the GC and thinks it is still the leader for the brief window before its next renewal call returns "you no longer hold this." During that window — possibly seconds — both instances run the dispatch loop. Without fencing, duplicate fires.
- **Long TTL (e.g. 60s, R=15s).** Survives normal GC pauses easily. But the failover gap is up to 60 seconds: if the leader crashes hard, no triggers fire for nearly a minute. For a scheduler with sub-minute fire requirements, that is unacceptable. For a daily-batch scheduler, it is invisible.

A defensible default for a general-purpose scheduler: `T = 15s, R = 5s`. Three renewals per lease lifetime. Failover gap ≤ 15s. JVM GC pauses are typically <1s on a tuned heap and can be tightened further with G1/ZGC; if you see >5s pauses, fix the GC, do not lengthen the lease.

**The detection lag the formula misses.** The lock service's clock has to *notice* the missed renewal before declaring the lease expired. Most services scan periodically (etcd via lease TTL processing, ZooKeeper via session timeout). Add 1–2 seconds for detection. Total worst-case failover gap ≈ `T + detection_lag + new_leader_startup_cost`. For etcd with `T=15s` and `R=5s`, plan for ~20s P99 failover.

## Split Brain: How Two Leaders Happen Anyway

The seductive lie of leader election is "the lock service guarantees one leader." It does not. It guarantees *one current lease holder*. Whether that translates to "one process running the dispatch loop" depends entirely on whether deposed leaders notice they have been deposed *before* doing harm.

The canonical scenario, well-documented by Martin Kleppmann ("How to do distributed locking"):

1. Leader L1 acquires lease, TTL 30s.
2. L1 enters a long GC pause (35s). It does not run any code, including renewal.
3. Lock service expires the lease at T+30s.
4. Standby L2 acquires the lease at T+30s. L2 is now the leader.
5. L1 emerges from GC at T+35s. **L1 has no idea time passed.** Its in-process state still says "I am the leader; my next renewal is at T+10s."
6. L1's next instruction was, conveniently, a write to the database to mark a job as fired and enqueue it to the worker queue.
7. L1 issues that write before it gets around to calling `lockService.renew()` and discovering it has lost the lease.
8. The job has now been fired by both L1 (the deposed leader) and L2 (the new leader running its own dispatch loop in parallel).

Replace "GC pause" with "OS swap pressure," "VM live migration," "kernel scheduler starvation under noisy neighbour," "network partition that lets renewal succeed but blocks dispatch traffic," and you have the same problem. Split brain is not a rare event in cloud environments; it is a once-a-month event somewhere in a fleet of any meaningful size.

The naive fix is "make the lease longer." This trades a frequent split-brain risk for an infrequent-but-still-present one, while making failover slow. The lease TTL cannot be longer than the longest pause your runtime can ever experience, which is a quantity nobody actually knows.

The correct fix is fencing.

## Fencing Tokens: The Only Thing That Saves You

A fencing token is a monotonically-increasing number issued by the lock service every time the lease is granted. Each grant returns a strictly larger token than every prior grant. The lease holder includes the token in every subsequent write to a downstream system; the downstream system rejects any write whose token is smaller than the largest one it has previously seen.

- L1 acquires lease, gets fencing token `f=42`.
- L1 GC-pauses; lease expires.
- L2 acquires lease, gets fencing token `f=43`.
- L2 issues a dispatch write to Postgres with `WHERE last_fence_seen <= 43`; the row updates to `last_fence_seen = 43`.
- L1 wakes up and issues its dispatch write with `f=42`. The `WHERE last_fence_seen <= 42` clause fails because the row already has `last_fence_seen = 43`. The write is silently rejected. L1 then fails its renewal call, learns it has been deposed, and shuts down.

Critically, **the fencing check happens at the destination, not at the lock service.** The lock service cannot prevent L1 from sending a write — L1's network is fine, its database connection is open, it does not know it has lost. The destination, which sees the token, can refuse.

Implementation forms in a scheduler:

- **etcd `Revision`.** Every write to etcd is tagged with a cluster-wide monotonic revision. The leader reads its revision at acquisition and uses that as its fencing token.
- **ZooKeeper `czxid`.** Each znode has a creation transaction id; the leader's ephemeral znode's `czxid` is monotonic, usable as a token.
- **DynamoDB version attribute.** A `version` attribute on the lock row, incremented on each acquisition, used as a fencing token.
- **Application-side fencing column.** The `schedules` row carries a `last_fence_seen` column; every dispatch write to it includes `WHERE last_fence_seen <= :my_token`.

If you do not implement fencing on side effects, you are betting that no GC pause, no swap event, no live migration, no scheduler starvation will ever exceed your lease TTL on any node, ever. That bet has a known failure mode. Fencing makes the failure mode a no-op.

For a deeper treatment, see [Martin Kleppmann's "How to do distributed locking"](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — the canonical reference, and the source of the term "fencing token" as it is used here.

## Code: ZooKeeper Sequential-Ephemeral Election

The classic ZooKeeper recipe ([Apache ZooKeeper Recipes — Leader Election](https://zookeeper.apache.org/doc/current/recipes.html#sc_leaderElection)). Each candidate creates an ephemeral sequential znode under a parent path; the candidate whose znode has the smallest sequence number is the leader. Watching the predecessor (not the parent) avoids herd effects.

```python
import time
from kazoo.client import KazooClient
from kazoo.exceptions import NoNodeError

ELECTION_PATH = "/scheduler/leader"

class ZkLeaderElector:
    def __init__(self, zk: KazooClient, instance_id: str):
        self.zk = zk
        self.instance_id = instance_id
        self.my_node_path = None
        self.is_leader = False
        self.fencing_token = None  # czxid of our ephemeral node

    def join_election(self):
        self.zk.ensure_path(ELECTION_PATH)
        # Ephemeral + sequential: ZK appends a monotonic suffix and ties the
        # node's lifetime to our session. If our session dies, the node is
        # removed and we cease to be a candidate.
        self.my_node_path = self.zk.create(
            f"{ELECTION_PATH}/candidate-",
            value=self.instance_id.encode(),
            ephemeral=True,
            sequence=True,
        )
        # The czxid (creation transaction id) is monotonic across the cluster;
        # it works as our fencing token for any write we make as leader.
        stat = self.zk.exists(self.my_node_path)
        self.fencing_token = stat.czxid
        self._evaluate_leadership()

    def _evaluate_leadership(self):
        children = sorted(self.zk.get_children(ELECTION_PATH))
        my_name = self.my_node_path.split("/")[-1]
        my_index = children.index(my_name)
        if my_index == 0:
            self.is_leader = True
            self._on_become_leader()
        else:
            # Watch the immediate predecessor. When it goes away (its session
            # expired), we re-evaluate. Watching the parent would create a
            # thundering herd on every membership change.
            predecessor = children[my_index - 1]
            self.zk.exists(
                f"{ELECTION_PATH}/{predecessor}",
                watch=self._on_predecessor_change,
            )

    def _on_predecessor_change(self, event):
        # Predecessor went away; we may be the new leader.
        try:
            self._evaluate_leadership()
        except NoNodeError:
            # Race: someone deleted while we were checking. Retry.
            self._evaluate_leadership()

    def _on_become_leader(self):
        # Now safe to start the dispatch loop. Every write must include
        # self.fencing_token so downstream rejects stale-leader writes.
        start_dispatch_loop(self.fencing_token)

    def step_down(self):
        # Graceful: delete our node so the next candidate is promoted now,
        # not after our session times out.
        if self.my_node_path:
            self.zk.delete(self.my_node_path)
        self.is_leader = False
```

**What ZooKeeper gives you for free.** Session-based ephemeral nodes mean leader detection is built-in: when the leader's process dies, its TCP connection drops, ZooKeeper's session times out (configurable, typically 6–30s), the ephemeral node disappears, and the next candidate is promoted. No external heartbeat loop needed for liveness; the session *is* the heartbeat.

**The trap.** ZooKeeper sessions can survive process restarts if the client implementation reconnects within the session timeout. That is a feature, not a bug, but it means "instance restarted and immediately got its leadership back" is possible. Usually fine. Occasionally surprising.

## Code: etcd Lease + CompareAndSwap Election

etcd's election primitive is a *lease* combined with a *transactional put with `IfNotExists`*. The key holds the leader's identity; the lease's TTL governs liveness; renewal is a `LeaseKeepAlive` stream.

```go
package scheduler

import (
    "context"
    "errors"
    "log"
    "time"

    clientv3 "go.etcd.io/etcd/client/v3"
)

const (
    leaderKey  = "/scheduler/leader"
    leaseTTL   = 15 // seconds
)

type EtcdLeaderElector struct {
    client       *clientv3.Client
    instanceID   string
    leaseID      clientv3.LeaseID
    fencingToken int64 // etcd revision at acquisition
    cancel       context.CancelFunc
}

func (e *EtcdLeaderElector) Campaign(ctx context.Context) error {
    for {
        // Fresh lease per attempt; old leases are garbage-collected by etcd.
        lease, err := e.client.Grant(ctx, leaseTTL)
        if err != nil {
            return err
        }

        // Transaction: only put if the key does not already exist.
        // CreateRevision == 0 is etcd's way of saying "key absent".
        txn := e.client.Txn(ctx).
            If(clientv3.Compare(clientv3.CreateRevision(leaderKey), "=", 0)).
            Then(clientv3.OpPut(leaderKey, e.instanceID, clientv3.WithLease(lease.ID))).
            Else(clientv3.OpGet(leaderKey))

        resp, err := txn.Commit()
        if err != nil {
            return err
        }

        if resp.Succeeded {
            // We are leader. Capture revision as fencing token.
            e.leaseID = lease.ID
            e.fencingToken = resp.Header.Revision
            log.Printf("acquired leadership: instance=%s token=%d", e.instanceID, e.fencingToken)
            return e.serveAsLeader(ctx)
        }

        // Lost the race. Watch for current leader's lease to die, then retry.
        // Revoke our unused lease to free etcd resources.
        _, _ = e.client.Revoke(ctx, lease.ID)
        if err := e.waitForLeaderToVanish(ctx); err != nil {
            return err
        }
    }
}

func (e *EtcdLeaderElector) serveAsLeader(ctx context.Context) error {
    leaderCtx, cancel := context.WithCancel(ctx)
    e.cancel = cancel
    defer cancel()

    // KeepAlive returns a channel; each message is a renewal ack.
    // If the channel closes, we have lost the lease (network, expiry, etc).
    ch, kaErr := e.client.KeepAlive(leaderCtx, e.leaseID)
    if kaErr != nil {
        return kaErr
    }

    // Start dispatch loop in a goroutine; pass fencing token.
    dispatchDone := make(chan struct{})
    go func() {
        defer close(dispatchDone)
        runDispatchLoop(leaderCtx, e.fencingToken)
    }()

    for {
        select {
        case ka, ok := <-ch:
            if !ok {
                // Lease lost. Stop dispatching IMMEDIATELY.
                cancel()
                <-dispatchDone
                return errors.New("lease lost; stepping down")
            }
            _ = ka // ka.TTL tells us renewed TTL; useful for telemetry
        case <-leaderCtx.Done():
            <-dispatchDone
            return leaderCtx.Err()
        }
    }
}

func (e *EtcdLeaderElector) waitForLeaderToVanish(ctx context.Context) error {
    rch := e.client.Watch(ctx, leaderKey, clientv3.WithFilterPut())
    for resp := range rch {
        for _, ev := range resp.Events {
            if ev.Type == clientv3.EventTypeDelete {
                return nil // current leader's lease expired, retry campaign
            }
        }
    }
    return ctx.Err()
}

func (e *EtcdLeaderElector) StepDown(ctx context.Context) {
    if e.cancel != nil {
        e.cancel() // stop dispatch loop
    }
    if e.leaseID != 0 {
        // Revoking the lease deletes the key immediately, allowing the next
        // candidate to promote without waiting for TTL expiry.
        _, _ = e.client.Revoke(ctx, e.leaseID)
    }
}
```

**What this code gets right.**

- The fencing token is the etcd `Revision` at the moment of acquisition. Every subsequent write to Postgres / Redis / the worker queue must include this token; downstream rejects writes with a smaller token. Without the fencing column, this whole machinery is decorative.
- Lease loss propagates *into* the dispatch loop via `cancel()`. The loop must respect the context. A long-running dispatch tick that does not check `ctx.Done()` between database calls is a split-brain bug waiting for a GC pause.
- `StepDown` revokes the lease so the new leader can promote in milliseconds, not after `T` seconds of TTL expiry. Always graceful-shutdown if you can.

**What this code does not handle, but should.** Watching for *changes* to the lease key while you are leader, in case someone manually deletes it. (This is rare but happens with operator error.) Backoff on repeated lost-races to avoid election thrashing. Prometheus metrics on `lease_renewals_total`, `lease_renewals_failed_total`, `is_leader{instance=...}`. See [`./multi-tenant-fairness.md`](./multi-tenant-fairness.md) for how the dispatch loop *itself* needs telemetry once it is running.

## Code: Heartbeat Loop with Self-Fencing on Missed Renewal

Sometimes the lock-service library is opaque enough that you build the heartbeat yourself — particularly with DynamoDB conditional writes, which have no built-in lease primitive. Here is the pattern explicitly.

```python
import time
import threading
from dataclasses import dataclass

LEASE_TTL_SEC = 15
RENEW_INTERVAL_SEC = 5
GRACE_PERIOD_SEC = 2  # how long after a missed renewal we consider ourselves still leader

@dataclass
class Lease:
    holder: str
    fencing_token: int       # version number, monotonic
    expires_at_epoch: float  # absolute time

class DynamoLockClient:
    """Wraps DynamoDB conditional UpdateItem for lease semantics."""

    def acquire(self, key: str, holder: str) -> Lease | None:
        # UpdateItem with ConditionExpression:
        #   attribute_not_exists(holder) OR expires_at < :now
        # IF the condition holds, update holder, expires_at, fencing_token++.
        # Otherwise raise ConditionalCheckFailedException.
        ...

    def renew(self, key: str, holder: str, current_token: int) -> Lease | None:
        # UpdateItem with ConditionExpression:
        #   holder = :me AND fencing_token = :token
        # Sets expires_at = :now + TTL. Same fencing_token.
        ...

class LeaderHeartbeat:
    def __init__(self, lock: DynamoLockClient, instance_id: str):
        self.lock = lock
        self.instance_id = instance_id
        self.lease: Lease | None = None
        self.is_leader_atomic = False  # readable from dispatch loop
        self._stop = threading.Event()

    def run(self):
        while not self._stop.is_set():
            if self.lease is None or time.time() >= self.lease.expires_at_epoch:
                self._campaign()
            else:
                self._renew_or_step_down()
            time.sleep(RENEW_INTERVAL_SEC)

    def _campaign(self):
        result = self.lock.acquire("/scheduler/leader", self.instance_id)
        if result is not None:
            self.lease = result
            self.is_leader_atomic = True
            log.info("became leader: token=%d", result.fencing_token)
        else:
            self.is_leader_atomic = False

    def _renew_or_step_down(self):
        result = self.lock.renew(
            "/scheduler/leader",
            self.instance_id,
            self.lease.fencing_token,
        )
        now = time.time()
        if result is not None:
            self.lease = result
            return  # still leader

        # Renewal failed. Either:
        #   (a) lease expired and someone else took it (we are deposed)
        #   (b) network blip; we may still hold the lease
        # We cannot tell from this single failure. SELF-FENCE: assume the worst.
        time_since_last_known_good = now - (self.lease.expires_at_epoch - LEASE_TTL_SEC)
        if time_since_last_known_good > LEASE_TTL_SEC - GRACE_PERIOD_SEC:
            # We are too close to expiry to be confident. Step down NOW; the
            # dispatch loop will see is_leader_atomic=False and stop.
            log.warn("self-fencing: renewal failed and within grace window")
            self.is_leader_atomic = False
            self.lease = None

    def current_fencing_token(self) -> int | None:
        if self.is_leader_atomic and self.lease is not None:
            return self.lease.fencing_token
        return None
```

**The principle.** Don't wait for the lock service to tell you that you've been deposed. Watch the wall clock against your last-known-good renewal, and step down *yourself* if the gap exceeds a safety margin. Self-fencing is the only defence against the GC-pause split-brain when your runtime is unreliable about scheduler latency.

**The dispatch loop's responsibility.** Every iteration, before doing any side effect, the dispatch loop reads `current_fencing_token()`. If it returns `None`, the loop stops. If it returns a token, the loop includes that token in every downstream write. The loop never caches the token across iterations — it re-reads it each pass.

## Sharded Schedulers: One Leader per Shard

Active-passive scales to the throughput of one box. A single Postgres-backed dispatch loop on a tuned scheduler instance might handle 10k–100k triggers/minute; beyond that, you shard.

The standard partition is `shard_id = hash(tenant_id) % N` (or `hash(schedule_id) % N` if you don't have tenants). Each shard has its own election under `/scheduler/leader/<shard_id>`. With N=100 shards and a 10-instance fleet, each instance leads ~10 shards on average; if an instance dies, its shards' leases expire and the survivors pick them up.

**What you gain.**

- Throughput scales linearly with N (until you hit the database, which you should also shard — see the parent case study §6).
- Failover blast radius is *one shard's* dispatch backlog, not the entire scheduler.
- Per-shard leadership is a clean unit of work for telemetry, deploys, and chaos testing.

**What you have to design.**

- **Shard rebalance on resize.** If you grow the fleet from 10 to 20 instances, leases for some shards should migrate. The simplest path is to do nothing — leases are short-lived; new instances will pick up shards as existing leases expire on their normal cadence. For 100 shards with a 15s TTL, full rebalance happens within about 15 seconds of provisioning. If that is too slow, the fleet leader (a layer above the per-shard leaders) can issue "step down for shard X" hints; instances voluntarily release shards above their target count.

- **Skewed shard load.** Hash partitioning assumes uniform load. In reality, one tenant might own 80% of the schedule volume. Address with weighted hashing (one big tenant → multiple shards) or tenant-specific shard pools. See [`./multi-tenant-fairness.md`](./multi-tenant-fairness.md) for the fairness story above the shard layer.

- **Shard ownership in the data layer.** The dispatch query should be `WHERE shard_id = :my_shard` so a shard leader only scans its slice. If the schedules table is partitioned by `shard_id` (Postgres declarative partitioning, for instance), each shard's dispatch query hits one partition, no cross-shard lock contention.

- **Per-shard fencing.** Each shard's fencing token is independent. The `last_fence_seen` column lives on the schedule row, scoped to that shard's writes. A leader cannot use shard A's token to write to shard B's rows; it would not be the leader of B.

A worked picture:

```text
fleet = [scheduler-0, scheduler-1, scheduler-2, ..., scheduler-9]
shards = 100, leases = 15s

t=0:
  scheduler-0 leads shards: 0, 10, 20, 30, ..., 90
  scheduler-1 leads shards: 1, 11, 21, ..., 91
  ...
  scheduler-9 leads shards: 9, 19, 29, ..., 99

t=60: scheduler-3 crashes.
  Shards 3, 13, 23, 33, 43, 53, 63, 73, 83, 93 are leaderless.
  Their leases expire at t≤75.
  By t=80, surviving 9 instances have absorbed ~1.1 extra shards each.
  Maximum dispatch gap on any one shard: ~15s (the lease TTL).
```

That ~15s gap matters because for the duration of the gap, the shard's triggers do not fire. Whether that is acceptable depends on the shard's SLO. A daily-batch shard does not care about 15 seconds. A second-precision intraday-trigger shard might. Tune lease TTL per shard tier if needed; nothing requires all shards to share the same TTL.

## Graceful Hand-Off vs Crash Failover

Two cases. Both are common; they have different mechanics.

**Graceful hand-off (deploys, scale-down).** The leader is being shut down on purpose: a deploy rolling restart, a node drain, a manual intervention. The leader knows it is going away.

1. Service mesh / orchestrator sends `SIGTERM` to the scheduler instance.
2. Scheduler enters drain state: dispatch loop finishes the current tick (no new triggers fired) and then halts.
3. Scheduler explicitly *revokes* the lease (etcd `Revoke`, ZooKeeper `delete` of the ephemeral node, DynamoDB `UpdateItem` to clear the holder).
4. Lock service notifies / next candidate sees the lease is free; new leader acquires immediately.
5. Failover gap: ~tens of milliseconds (network round-trip + new candidate's acquire call), not the lease TTL.

The drain step is critical. Revoking the lease while the dispatch loop is still mid-tick means the new leader can promote and start dispatching the same triggers the old leader is in the middle of processing. With fencing, the old leader's writes will be rejected at the destination; without fencing, you get duplicate fires. Order matters: drain first, revoke second, exit third.

**Crash failover (segfault, OOM kill, network partition, kernel panic).** The leader does not know it is going away.

1. The lease silently stops being renewed.
2. Lock service detects expiry after `T` seconds.
3. Standby campaigns and acquires.
4. Failover gap: up to `T` seconds plus detection lag plus new-leader startup.

You cannot make this faster without lowering `T`, and lowering `T` makes split-brain more likely. Fencing is what makes the residual split-brain harmless.

The two cases compose during a deploy: if the orchestrator's `SIGTERM` doesn't reach the process (it crashed simultaneously), you fall back to crash failover. Always design for the crash case; the graceful case is a happy-path optimization.

## Failover SLO: How Fast Is Fast Enough?

Pick the SLO before tuning the lease, not after. Some workloads tolerate minutes; others can't tolerate seconds.

| Workload | Acceptable failover gap | Implications |
|---|---|---|
| Daily batch (e.g. nightly ETL at 02:00) | Minutes | Lease TTL 60s+ is fine; cheap to operate |
| Hourly aggregation | 10s of seconds | Lease TTL ~30s, tighter standby readiness |
| Per-minute triggers (cron `* * * * *`) | Seconds | Lease TTL 5–15s, requires GC tuning |
| Sub-second triggers | Milliseconds | Don't use a job scheduler — see the parent doc; you want a workflow engine like Temporal or a streaming system |
| Hard real-time (control loops) | Microseconds | Not a job-scheduler problem at all |

**The arithmetic.** Worst-case failover gap ≈ `lease_TTL + lock_service_detection_lag + new_leader_acquire_latency + new_leader_warmup`.

- `lease_TTL`: configurable, typically 5–60s.
- `lock_service_detection_lag`: 0–2s (etcd's lease processing) up to 30s (ZooKeeper default session timeout).
- `new_leader_acquire_latency`: typically <100ms on a healthy cluster.
- `new_leader_warmup`: seconds, depending on what state the new leader needs to load (in-memory caches, prepared statements, JIT warmup).

Practical defaults for a general-purpose scheduler: 15s lease, 5s renewal, 2s detection, 100ms acquire, ~3s warmup. Total: ~20s P99. If your SLO is 30s, this is comfortable. If your SLO is 5s, you have work to do — likely active-active sharding with pre-warmed standbys per shard.

**The standby's role in warmup.** If the standby keeps a tail of the dispatch state warm in memory (recent triggers, prepared connection pool, JIT-compiled hot paths), warmup approaches zero. If the standby is a cold process that loads state on promotion, warmup dominates. For tight SLOs, design the standby to be hot — running everything *except* the dispatch loop, ready to flip the switch the instant it acquires.

## Decider vs Executor: Don't Stack Them on the Same Process

A common scaling mistake: the scheduler instance that holds leadership also runs the worker pool that executes jobs. Looks fine on a whiteboard. Falls over in production for two reasons.

**Reason one: the leader becomes a bottleneck.** The dispatch loop should be lightweight (read next-due trigger, lock it, enqueue it). If the leader is also executing jobs, executing a slow job (a 10-minute report generation) starves the dispatch loop because they share CPU, memory pressure, GC, or thread pool. Fires get late.

**Reason two: leader churn becomes job churn.** When the leader fails over, in-flight jobs on that instance are abandoned mid-execution. A clean separation lets workers be a separate fleet (no leadership), surviving across leader transitions; only the dispatch decision is leadership-scoped.

The clean architecture:

```text
[Scheduler tier]                    [Worker tier]
 elects leader                       no leadership
 dispatch loop runs on leader        many independent workers
 enqueues to broker                  pull from broker
                                     execute idempotent handlers
                                     update runs table
```

The scheduler tier dispatches by writing to a durable queue (Kafka, SQS, RabbitMQ, or a Postgres `outbox` table — see [`../../distributed-infra/design-message-queue.md`](../../distributed-infra/design-message-queue.md)). Workers pull from the queue. Workers do not care who the scheduler leader is. The scheduler does not care which worker picks up the work. Failover of either tier is independent.

This is the architecture used by Airflow (scheduler ↔ Celery / Kubernetes executor), Temporal (history shards ↔ activity workers), and most production systems. The exception is in-process schedulers like Quartz running inside an application server — fine for low scale and single-tenant uses, but the leader-also-executes coupling caps the design.

## Observability: Who Is the Leader Right Now?

If you can't answer "who is the leader of shard 47 right now" in five seconds, your operations are blind. Required telemetry:

- **`scheduler_is_leader{instance, shard}`** — gauge, 1 if this instance currently holds the lease for this shard, 0 otherwise. Sum across the fleet must equal the number of shards. **Aleart on `sum(scheduler_is_leader{shard="X"}) > 1`** — that is a split-brain in progress; page immediately.
- **`scheduler_lease_renewals_total{instance, shard, outcome}`** — counter, labeled by `success | failed`. Renewal failure rate above ~1% indicates lock-service health issues or network problems.
- **`scheduler_lease_acquired_timestamp{instance, shard}`** — gauge, wall-clock time of last acquisition. Useful for "how long has this leader been leading?" and detecting flapping.
- **`scheduler_failover_duration_seconds{shard}`** — histogram, time between previous leader's lease expiry and new leader's first dispatch. Track P50/P95/P99 against SLO.
- **`scheduler_election_attempts_total{instance, shard, outcome}`** — counter, success/lost-race/error. Lost-race counts during steady-state suggest election thrashing.

**The "two leaders" detection.** The most dangerous failure mode is silent: two instances both believing they lead the same shard, neither with the lock service's blessing. Detect with:

1. Application-side metric `scheduler_is_leader` reported by every instance.
2. Server-side query against the lock service for ground truth (`who currently holds /scheduler/leader/<shard>`).
3. Compare. Any disagreement, alert.

This is also the rationale for fencing: if the two-leaders alert fires *and* fencing is properly wired, the harm is bounded — one of the two has stale tokens and its writes will be rejected. Without fencing, the alert fires and you're already shipping duplicate fires. With fencing, the alert fires and you have time to respond before the next dispatch tick.

## Worked Example: Three-Instance Scheduler, Leader Dies

Concrete walk-through. Three scheduler instances `S1`, `S2`, `S3` running etcd-based leader election with 15s lease TTL, 5s renewal cadence. One shard for simplicity. A pile of triggers due to fire across the next minute.

**Setup.**
- `S1` is leader. Fencing token at acquisition: `revision=10042`. Renews every 5s.
- `S2`, `S3` are watching `/scheduler/leader` for deletion.
- Schedules table has triggers `T1` (due 12:00:05), `T2` (due 12:00:15), `T3` (due 12:00:30).

**Timeline.**

| Wall time | S1 (current leader) | S2 / S3 (standby) | etcd state |
|---|---|---|---|
| 12:00:00 | renewing every 5s | watching key | lease alive, key=S1, rev=10042 |
| 12:00:05 | dispatches T1, includes fence=10042; advances `T1.next_fire_at` | idle | lease alive |
| 12:00:08 | **OOM-killed** by kernel | unaware yet | lease still alive (renewed at 12:00:05) |
| 12:00:13 | dead | watching | lease alive but no renewals |
| 12:00:20 | dead | watching | **lease expires** (T+15 from last renewal at 12:00:05) |
| 12:00:20.1 | dead | etcd deletes key on expiry | key=∅ |
| 12:00:20.2 | dead | both S2 and S3 see delete event; both campaign | |
| 12:00:20.3 | dead | S2 wins the CAS (created revision lower) | lease alive, key=S2, rev=10071 |
| 12:00:20.3 | dead | S2 captures fence=10071, starts dispatch loop, runs `SELECT … FOR UPDATE SKIP LOCKED` | |
| 12:00:20.4 | dead | S2 sees T2 was due at 12:00:15, applies missed-fire policy | |

**Missed-fire decision for T2.** Per [`./missed-fire-policies.md`](./missed-fire-policies.md), the schedule's policy is consulted:

- If `FIRE_ONCE_NOW`: T2 fires now (12:00:20.4) with fence=10071. The fact that it is 5.4s late is logged and emitted as a metric; the trigger advances normally.
- If `FIRE_ALL_MISSED`: same as above for T2 (only one missed fire).
- If `DO_NOTHING`: T2 is skipped; `next_fire_at` advances to whatever its cron schedule says comes after 12:00:15. The skip is logged.
- If the trigger has a tighter deadline (`misfire_threshold = 2s` and `now - due > threshold`), the skip path runs even if the policy says fire-now. This guards against firing yesterday's "send 9am notification" at 11pm.

**T3 fires normally at 12:00:30** with fence=10071. The dispatch loop is now fully under S2's leadership.

**Side-channel concern.** Suppose `S1` was not really dead but in a 12-second swap-induced pause. At 12:00:20, the kernel decides to schedule it. `S1` resumes, still believes it is leader, attempts to dispatch T2 with fence=10042. Postgres rejects the write because `last_fence_seen = 10071 > 10042`. The dispatch returns "not found / ignored." `S1` then reaches its renewal call, which fails (key holder is now S2 with rev=10071). `S1` learns it has been deposed, halts dispatch, and either restarts cleanly or shuts down. **No duplicate fire.**

If fencing were not implemented, `S1` would have inserted a duplicate `runs` row for T2, enqueued a duplicate worker job, and the worker queue would dispatch the same payload twice. With idempotency on the handler ([`../design-job-scheduler.md` §7.1](../design-job-scheduler.md)), the side effect is deduplicated; without idempotency, the email goes twice or the charge runs twice. Defence in depth: leader election + per-job locking + handler idempotency. The whole stack.

## Anti-Patterns

**1. No fencing on side effects.** The single most common mistake. Leader election is implemented, the dispatch loop respects `is_leader`, and the engineer thinks the system is safe. The first GC pause longer than the lease TTL produces duplicate fires that the team blames on "Postgres weirdness" or "the queue retrying." Fence every write; reject stale tokens at the destination.

**2. Lease TTL too long.** "We set it to 5 minutes for safety." Now any leader crash leaves the system idle for 5 minutes, missing every fire scheduled in that window. Daily-batch workloads can absorb this; minute-precision schedules cannot. Set TTL based on failover SLO, not on superstition.

**3. Lease TTL too short.** "We set it to 1 second for fast failover." Now every momentary GC blip causes a renewal failure and a re-election. The fleet thrashes; the dispatch loop spends more time in election handshakes than dispatching. Set TTL with realistic GC and network jitter in mind; 3–5x your worst-observed pause is a starting point.

**4. Ad-hoc election by file lock or DB row.** "We don't need ZooKeeper; we'll just use a Postgres row with `SELECT FOR UPDATE`." This works under cooperative load; it falls apart on partition (the holder's `KEEPALIVE` doesn't reach Postgres but Postgres's `idle_in_transaction_timeout` kicks in mid-fire), on connection pool exhaustion, or on Postgres failover (advisory lock is local to the primary; on failover, the lock vanishes and the new primary thinks no one holds it). Use a real coordination service, or accept that you're using Postgres advisory locks *as defence in depth* atop a real election.

**5. Leader does both deciding and executing.** Covered above. The dispatch loop and the worker pool belong to different tiers; bridging them with a leadership concern couples scaling, deploy cadence, and failure modes that should be independent.

**6. Election thrashing.** If the lease TTL is too short, or the lock service is itself unstable, instances flap between leader and not-leader on every minor disturbance. Each flap incurs a warmup cost on the new leader. Detect with `scheduler_election_attempts_total{outcome="lost-race"}` rate; if it's anything other than near-zero in steady state, your TTL is too tight or your renewal cadence is too sparse.

**7. Leader writes to multiple stores without atomicity.** "Leader updates Postgres, then publishes to Kafka." If the leader is deposed between the two, the new leader sees the Postgres row as fired but Kafka has no record. Use the [transactional outbox pattern](../../../communication/outbox-pattern.md) or accept that the worker queue lookups are part of the dispatch transaction.

**8. Trusting `is_leader` for long stretches.** A dispatch tick reads `is_leader` once and then runs for 30 seconds. During those 30 seconds, the leader is deposed and a new leader takes over. The first leader's tick continues, fencing writes are rejected, but the leader doesn't notice until the tick ends — and may have done some side-effect that doesn't go through the fenced path (a metric emission, a log entry interpreted as "this fired"). Re-check `is_leader` between every state-mutating step; abort the tick if the answer flips.

**9. Standby that doesn't watch the leader.** A standby that polls every 60 seconds for "is the leader still there?" has 60-second failover even if the lease TTL is 10s. Use the lock service's notification mechanism (etcd `Watch`, ZooKeeper watcher, Consul blocking query) so failover triggers as soon as the lease is freed.

**10. Mixing election with business logic.** Putting "if I am leader, also do X for tenant Y" in the election library results in a tangle where leadership semantics leak into application code. Keep the elector as a black box that exposes `is_leader` and `fencing_token`; build application logic that reads those two values.

**11. Forgetting to revoke on graceful shutdown.** A SIGTERM handler that exits before revoking the lease leaves the lock service waiting `T` seconds before it expires. Easy fix: defer `revoke()` in the shutdown path. Easier mistake to make than it sounds.

**12. Single-region election for multi-region scheduler.** If your scheduler runs across two regions for DR, but the election's lock service is in one region only, a region failure that takes out the lock service freezes elections globally. Either run the lock service multi-region (etcd cluster with quorum across regions, accepting the cross-region latency on every renewal) or accept that DR failover is a manual cutover and not an automatic election.

**13. Treating cloud-managed lock services as infinitely available.** etcd-as-a-service, DynamoDB, Consul Cloud — all have outages. The scheduler should degrade gracefully when the lock service is briefly unreachable: continue with the *current* leader for one or two missed renewals (within the lease TTL), then halt dispatch (do *not* assume leadership has held when you can't confirm). The halted state is correct; phantom-leadership during a lock-service outage is not.

**14. No "step-down" path for operators.** When the on-call needs to pin the leader to a known-good instance (because the current leader is misbehaving), they need a way to make the current leader release the lease *now*. Build a `kill -USR1` or admin endpoint that triggers `step_down()`. Without it, the operator's only recourse is to kill the process and wait for the lease to expire — slow, and the standby that promotes might be the same misbehaving binary.

## Related

- [`./distributed-lock-per-job.md`](./distributed-lock-per-job.md) — per-job locking; defence in depth above leader election.
- [`./missed-fire-policies.md`](./missed-fire-policies.md) — what the new leader does when it discovers triggers that fired-late.
- [`./multi-tenant-fairness.md`](./multi-tenant-fairness.md) — fairness within the dispatch loop the leader runs.
- [`./dependency-dag.md`](./dependency-dag.md) — dependency execution that the dispatch loop must respect.
- [`../design-job-scheduler.md`](../design-job-scheduler.md) — parent case study with the full scheduler design.
- [`../../../data-consistency/leader-election-and-coordination.md`](../../../data-consistency/leader-election-and-coordination.md) — the coordination primitive in general; covers the building blocks used here.
- [`../../../data-consistency/consensus-raft-and-paxos.md`](../../../data-consistency/consensus-raft-and-paxos.md) — Raft/Paxos consensus underlying ZooKeeper, etcd, and Raft-embedded schedulers like Temporal.

## References

- [Apache ZooKeeper Recipes — Leader Election](https://zookeeper.apache.org/doc/current/recipes.html#sc_leaderElection) — the canonical sequential-ephemeral-znode recipe.
- [etcd Concurrency / Election API (v3)](https://etcd.io/docs/v3.5/dev-guide/api_concurrency_reference_v3/) — etcd's lease + election primitives.
- [Kubernetes leader election (client-go)](https://github.com/kubernetes/client-go/tree/master/tools/leaderelection) — the library used by the kube-controller-manager and many Kubernetes operators.
- [HashiCorp Consul Sessions](https://developer.hashicorp.com/consul/docs/dynamic-app-config/sessions) — Consul's session + KV-acquire pattern for distributed locking.
- [Martin Kleppmann, "How to Do Distributed Locking" (2016)](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — the canonical treatment of fencing tokens; required reading.
- [AWS DynamoDB Conditional Writes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html) — the primitive behind DynamoDB-based lease implementations.
- [Diego Ongaro and John Ousterhout, "In Search of an Understandable Consensus Algorithm" (Raft)](https://raft.github.io/raft.pdf) — the consensus algorithm underneath etcd, Consul, and Raft-embedded schedulers.
- [Apache Airflow — Scheduler HA (Airflow 2.0+)](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/scheduler.html) — production scheduler with HA via row-level locking against the metadata DB; instructive comparison case.
