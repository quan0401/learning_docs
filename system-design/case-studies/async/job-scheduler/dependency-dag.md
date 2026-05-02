---
title: "Job Dependency DAG — Topological Scheduling, Branching, and Failure Propagation"
date: 2026-05-01
updated: 2026-05-01
tags: [system-design, deep-dive, scheduler, dag, workflows]
---

# Job Dependency DAG — Topological Scheduling, Branching, and Failure Propagation

**Date:** 2026-05-01 | **Updated:** 2026-05-01
**Tags:** `system-design` `deep-dive` `scheduler` `dag` `workflows`

## Table of Contents

- [Summary](#summary)
- [Why a DAG and Not a List](#why-a-dag-and-not-a-list)
- [The Two-Layer Model — Definition vs Run](#the-two-layer-model--definition-vs-run)
  - [DAG Definition (Static)](#dag-definition-static)
  - [DAG Run (Per Fire Time)](#dag-run-per-fire-time)
  - [Per-Run Instances Multiply Fast](#per-run-instances-multiply-fast)
- [Storage — Nodes, Edges, and Versions](#storage--nodes-edges-and-versions)
  - [Schema](#schema)
  - [Immutability per Version](#immutability-per-version)
  - [Versioning Rules at Runtime](#versioning-rules-at-runtime)
- [Topological Scheduling](#topological-scheduling)
  - [The Core Algorithm](#the-core-algorithm)
  - [Worked Code — DAG Topological Sort with State Propagation](#worked-code--dag-topological-sort-with-state-propagation)
- [Edge Trigger Rules](#edge-trigger-rules)
  - [The Six Useful Rules](#the-six-useful-rules)
  - [Worked Code — Edge Rule Evaluator](#worked-code--edge-rule-evaluator)
- [Branching and Skipped Nodes](#branching-and-skipped-nodes)
- [Cycle Detection on Submission](#cycle-detection-on-submission)
  - [Why DFS, Not Kahn](#why-dfs-not-kahn)
  - [Worked Code — DFS Cycle Detection](#worked-code--dfs-cycle-detection)
- [Fan-Out and Fan-In — Dynamic Tasks](#fan-out-and-fan-in--dynamic-tasks)
  - [Static vs Dynamic Mapping](#static-vs-dynamic-mapping)
  - [Gather Step Semantics](#gather-step-semantics)
- [Sensors and External Triggers](#sensors-and-external-triggers)
- [Resource Pools and Concurrency Limits](#resource-pools-and-concurrency-limits)
- [Backfill — Walking Historical Date Ranges](#backfill--walking-historical-date-ranges)
- [Sub-DAGs and Reusable Patterns](#sub-dags-and-reusable-patterns)
- [Worked Example — ETL DAG with a Failure in Transform](#worked-example--etl-dag-with-a-failure-in-transform)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

A general-purpose job scheduler eventually grows the same feature its users keep asking for: "run job B only after job A finishes, and job D only when both B and C succeed, and run E if anything failed." That is a directed acyclic graph of jobs. The mental model that scales is a **two-layer system** — a static, versioned **DAG definition** (nodes and edges in a database) and a **DAG run** (per fire-time execution that materializes each node into a `task_instance` with its own state machine). The scheduler walks the DAG topologically: a node fires when its incoming edges are satisfied per their **trigger rules** (`ALL_SUCCESS`, `ONE_SUCCESS`, `ALL_DONE`, `ONE_FAILED`, `ALL_FAILED`, `NONE_FAILED`). Cycles are rejected at submission via DFS coloring. Versions are immutable — running DAGs continue on the version they started on; new triggers pick up the latest. Fan-out (dynamic tasks) and fan-in (gather) compose with the same model. Sensors are first-class nodes that block on external state. Resource pools cap per-job concurrency. Backfill replays the DAG over a historical date range. The anti-patterns are familiar from any state-machine system: implicit timing dependencies, shared mutable state across nodes, non-idempotent steps, infinite per-node retries that block the run, and mutating a DAG mid-run. Apache Airflow is the reference implementation; Argo Workflows, Dagster, Prefect, AWS Step Functions, and Temporal occupy adjacent points in the same design space.

## Why a DAG and Not a List

A linear pipeline (`A → B → C → D`) is the easy case and a list-of-jobs encoding works. The interesting cases are:

- **Parallelism with a join.** Extract from three sources in parallel, transform each, then join. Five jobs, three of them concurrent.
- **Conditional branches.** Validate inputs; if validation fails, run the cleanup and notify branch instead of the main path.
- **Fan-out over data.** Process one partition per region — but the regions are not known until you query the metadata table at run time.
- **Cleanup that always runs.** Whatever happens — success, failure, partial — emit a final report and tear down temp tables.

Each of these is awkward in a list and natural in a DAG. The cost of generalizing to a DAG is small (an edge table and a topological walker); the payoff is that every workflow above is the same code path with different graph shapes.

The DAG also gives you **a definition you can reason about statically**: lint it, visualize it, search for cycles, count critical path length, predict resource usage. None of those tools work on imperative code that builds the workflow at runtime.

## The Two-Layer Model — Definition vs Run

The single most important distinction in this design — and the one most home-grown schedulers fumble — is **the DAG definition is not the DAG run**.

### DAG Definition (Static)

A DAG definition is a record describing the graph:

- A `dag_id` (e.g., `daily_revenue_etl`).
- A version (`v3`, content-hashed or monotonic).
- A set of nodes (each a job spec — image, command, retries, timeout, resource pool).
- A set of directed edges (with optional trigger rule per edge).
- A schedule (cron expression or external trigger).
- Catchup, backfill, default trigger rule, max active runs.

This is the artifact code review sees, the artifact CI deploys, and the artifact you draw on a whiteboard. It has no state of its own beyond "is this version current?"

### DAG Run (Per Fire Time)

When the scheduler decides to fire the DAG (cron tick, manual trigger, sensor signal), it materializes a **DAG run**:

```text
dag_run_id = (dag_id, dag_version, fire_time)
```

The run owns:

- One `task_instance` per node — its own state (`SCHEDULED`, `QUEUED`, `RUNNING`, `SUCCESS`, `FAILED`, `SKIPPED`, `UPSTREAM_FAILED`, `RETRY`).
- Per-run context (the `fire_time`, the parameters, the resolved fan-out cardinality).
- A run-level state derived from the task instances (`RUNNING`, `SUCCESS`, `FAILED`, `PARTIAL`).

The same DAG definition can have many concurrent runs (different fire times, or backfill catching up on history) and they do not interfere — each has its own task instances.

### Per-Run Instances Multiply Fast

A DAG with 30 nodes and a daily schedule running for a year produces **30 × 365 ≈ 11,000 task instances**. Add a fan-out node that explodes to 50 partitions and the count balloons to over half a million. Storage and indexing for `task_instance` is the table you must size — see [`design-job-scheduler.md`](../design-job-scheduler.md) §5 for the partitioning strategy.

The corollary: never embed task-instance state into the DAG definition. The definition is small (kilobytes); the runs are large (gigabytes per active DAG over its lifetime).

## Storage — Nodes, Edges, and Versions

### Schema

```sql
-- Definition
CREATE TABLE dag_versions (
  dag_id        TEXT NOT NULL,
  version       INT  NOT NULL,
  schedule      TEXT,            -- cron expr or 'event'
  default_rule  TEXT NOT NULL DEFAULT 'all_success',
  catchup       BOOLEAN NOT NULL DEFAULT FALSE,
  max_active    INT NOT NULL DEFAULT 1,
  body_hash     BYTEA NOT NULL,  -- content hash of node + edge set
  created_at    TIMESTAMPTZ NOT NULL,
  is_current    BOOLEAN NOT NULL,
  PRIMARY KEY (dag_id, version)
);

CREATE UNIQUE INDEX one_current_per_dag ON dag_versions (dag_id) WHERE is_current;

CREATE TABLE dag_nodes (
  dag_id     TEXT NOT NULL,
  version    INT  NOT NULL,
  node_id    TEXT NOT NULL,
  spec       JSONB NOT NULL,    -- image, cmd, retries, pool, timeout
  PRIMARY KEY (dag_id, version, node_id),
  FOREIGN KEY (dag_id, version) REFERENCES dag_versions
);

CREATE TABLE dag_edges (
  dag_id        TEXT NOT NULL,
  version       INT  NOT NULL,
  upstream_id   TEXT NOT NULL,
  downstream_id TEXT NOT NULL,
  trigger_rule  TEXT,           -- nullable: inherit dag default
  PRIMARY KEY (dag_id, version, upstream_id, downstream_id),
  FOREIGN KEY (dag_id, version) REFERENCES dag_versions
);

-- Run
CREATE TABLE dag_runs (
  run_id      UUID PRIMARY KEY,
  dag_id      TEXT NOT NULL,
  version     INT  NOT NULL,
  fire_time   TIMESTAMPTZ NOT NULL,
  state       TEXT NOT NULL,
  started_at  TIMESTAMPTZ,
  ended_at    TIMESTAMPTZ,
  UNIQUE (dag_id, fire_time, version)
);

CREATE TABLE task_instances (
  run_id    UUID NOT NULL REFERENCES dag_runs,
  node_id   TEXT NOT NULL,
  try_no    INT  NOT NULL DEFAULT 1,
  state     TEXT NOT NULL,       -- scheduled/queued/running/success/failed/skipped/upstream_failed/retry
  started_at TIMESTAMPTZ,
  ended_at   TIMESTAMPTZ,
  worker     TEXT,
  PRIMARY KEY (run_id, node_id, try_no)
);

CREATE INDEX ti_open_state ON task_instances (run_id, state)
  WHERE state IN ('scheduled', 'queued', 'running', 'retry');
```

The partial index on `task_instances.state` keeps the scheduler's "what is ready to run" query cheap — it scans only open work, not the historical mass of `success` rows.

### Immutability per Version

A `dag_versions` row is **immutable** once written. To "change" a DAG you write a new row with `version + 1`, copy/modify the nodes and edges, flip `is_current` atomically. The unique partial index `WHERE is_current` enforces exactly one current version per `dag_id` — the swap is a single transaction:

```sql
BEGIN;
UPDATE dag_versions SET is_current = FALSE
  WHERE dag_id = $1 AND is_current;
INSERT INTO dag_versions (dag_id, version, ..., is_current)
  VALUES ($1, $newv, ..., TRUE);
-- ... insert nodes, edges for the new version ...
COMMIT;
```

### Versioning Rules at Runtime

- **In-flight runs continue on their pinned version.** A run created at version 3 reads `dag_nodes` and `dag_edges` with `(dag_id, version=3)` for its entire life — even if version 4 lands while the run is mid-flight. This is the same lesson as the trip state machine: never mutate the contract a running workflow depends on.
- **New fires pick up `is_current`.** The cron trigger reads `dag_versions WHERE is_current` at fire time, materializes a run pinned to that version, and proceeds.
- **Manual reruns are explicit.** A user re-running a historical fire-time can pick "use the version that was current then" or "use the latest" — both are valid; both are explicit. Inferring it silently is the bug.

## Topological Scheduling

### The Core Algorithm

A node fires when, for each incoming edge, the edge's trigger rule evaluates `True` against the upstream node's task-instance state.

The scheduler does not need to compute a full topological order eagerly — it only needs to answer "given the current run state, which task instances just became eligible?" That is a local check after every state change:

1. Some task `t_u` (upstream) just transitioned to a terminal state (`SUCCESS`, `FAILED`, `SKIPPED`).
2. Look up downstream nodes via `dag_edges WHERE upstream_id = t_u.node_id`.
3. For each downstream `t_d`, evaluate every incoming edge of `t_d`. If all evaluate `True`, `t_d` is eligible — enqueue.
4. If any edge evaluates `False` and is unsatisfiable (e.g., `ALL_SUCCESS` and an upstream FAILED), mark `t_d` as `UPSTREAM_FAILED` and propagate.

The algorithm is **event-driven, not poll-driven**. Each terminal transition triggers a bounded local walk — no full graph scan. For very large DAGs (thousands of nodes) this matters; the cost per state change is proportional to the fan-out of the changed node, not the DAG size.

### Worked Code — DAG Topological Sort with State Propagation

This is the in-memory reference implementation; the production version uses the same logic but reads/writes via the schema above.

```python
from collections import defaultdict, deque
from dataclasses import dataclass
from enum import Enum

class TaskState(Enum):
    SCHEDULED = "scheduled"
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    SKIPPED = "skipped"
    UPSTREAM_FAILED = "upstream_failed"

@dataclass(frozen=True)
class Edge:
    upstream: str
    downstream: str
    rule: str  # "all_success", "one_success", etc.

class DAGRun:
    def __init__(self, nodes: list[str], edges: list[Edge]):
        self.nodes = set(nodes)
        self.edges = edges
        self.upstreams: dict[str, list[Edge]] = defaultdict(list)
        self.downstreams: dict[str, list[str]] = defaultdict(list)
        for e in edges:
            self.upstreams[e.downstream].append(e)
            self.downstreams[e.upstream].append(e.downstream)
        self.state: dict[str, TaskState] = {n: TaskState.SCHEDULED for n in nodes}

    def roots(self) -> list[str]:
        return [n for n in self.nodes if not self.upstreams[n]]

    def on_terminal(self, node: str) -> list[str]:
        """Called when `node` enters a terminal state.
        Returns the list of downstream nodes newly eligible to run
        (or newly marked UPSTREAM_FAILED)."""
        eligible: list[str] = []
        for d in self.downstreams[node]:
            if self.state[d] != TaskState.SCHEDULED:
                continue  # already decided
            decision = evaluate_node(self.upstreams[d], self.state)
            if decision == "fire":
                eligible.append(d)
            elif decision == "skip":
                self.state[d] = TaskState.SKIPPED
                eligible.extend(self.on_terminal(d))  # propagate
            elif decision == "upstream_failed":
                self.state[d] = TaskState.UPSTREAM_FAILED
                eligible.extend(self.on_terminal(d))  # propagate
            # else "wait" — leave it
        return eligible

    def is_done(self) -> bool:
        terminal = {TaskState.SUCCESS, TaskState.FAILED,
                    TaskState.SKIPPED, TaskState.UPSTREAM_FAILED}
        return all(s in terminal for s in self.state.values())
```

Two properties matter:

- **The walk is incremental.** Each `on_terminal` only touches direct downstreams; deeper propagation happens via the recursive call only when a node becomes `SKIPPED` or `UPSTREAM_FAILED`.
- **The state is the single source of truth.** No "queued for evaluation" side-list — the `state` dict captures everything; the walker is stateless beyond the graph structure.

## Edge Trigger Rules

A trigger rule is a predicate on the **set of upstream task states**. Airflow's vocabulary became the de-facto standard; the names below match its [Trigger Rules](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#trigger-rules) page.

### The Six Useful Rules

| Rule | Fires when | Skips when | Typical use |
|------|-----------|-----------|-------------|
| `ALL_SUCCESS` (default) | every upstream is `SUCCESS` | any upstream `FAILED`/`SKIPPED`/`UPSTREAM_FAILED` | the normal case |
| `ALL_FAILED` | every upstream is `FAILED` | any upstream `SUCCESS` | "everything broke; do the disaster-recovery branch" |
| `ALL_DONE` | every upstream is in any terminal state | never (always evaluates once) | cleanup, "always run" notifications |
| `ONE_SUCCESS` | at least one upstream `SUCCESS` (others may still be running) | all upstreams reached terminal without any success | branch joins where any success suffices |
| `ONE_FAILED` | at least one upstream `FAILED` | all upstreams ended without any failure | "notify on failure" branches |
| `NONE_FAILED` | every upstream `SUCCESS` or `SKIPPED`; none `FAILED`/`UPSTREAM_FAILED` | any upstream failed | tolerate skipped branches |
| `NONE_SKIPPED` | no upstream `SKIPPED` | any upstream skipped | strict "all paths must execute" |

The truth table is small but the corner cases are real. In particular:

- **`ALL_DONE` should still propagate `UPSTREAM_FAILED` carefully.** A cleanup task with `ALL_DONE` typically *should* run even when upstreams failed — that's the whole point. But if the cleanup itself depends on data the upstream produced, you've miscoded the dependency. The lesson: `ALL_DONE` is a structural cleanup hook, not a "ignore failures" cheat.
- **`ONE_SUCCESS` can fire early.** It does not wait for the slower upstreams. If you actually wanted "wait for all, succeed if any did," use a custom rule or split into two stages.

The default trigger rule for an edge is `ALL_SUCCESS` because it is the only rule that maps `success` of upstreams to `eligible` of downstream with no surprises. Every other rule should be opted into explicitly.

### Worked Code — Edge Rule Evaluator

```python
def evaluate_node(in_edges: list[Edge],
                  state: dict[str, TaskState]) -> str:
    """Return one of: 'fire', 'wait', 'skip', 'upstream_failed'."""
    if not in_edges:
        return "fire"  # root

    # Group upstream states once.
    states = [state[e.upstream] for e in in_edges]
    rules = [e.rule for e in in_edges]

    # Per-edge evaluation, then combine.
    decisions = [eval_one(rule, states[i], states)
                 for i, rule in enumerate(rules)]

    if any(d == "wait" for d in decisions):
        return "wait"
    if all(d == "fire" for d in decisions):
        return "fire"
    if any(d == "upstream_failed" for d in decisions):
        return "upstream_failed"
    return "skip"

def eval_one(rule: str, _own: TaskState,
             upstream_states: list[TaskState]) -> str:
    """For a single edge's rule, evaluated against ALL upstream states.

    Returns 'fire' / 'wait' / 'skip' / 'upstream_failed'."""
    s = upstream_states
    terminal = {TaskState.SUCCESS, TaskState.FAILED,
                TaskState.SKIPPED, TaskState.UPSTREAM_FAILED}

    if any(t not in terminal for t in s) and rule != "one_success" and rule != "one_failed":
        return "wait"

    if rule == "all_success":
        if all(t == TaskState.SUCCESS for t in s):
            return "fire"
        if any(t in (TaskState.FAILED, TaskState.UPSTREAM_FAILED) for t in s):
            return "upstream_failed"
        return "skip"

    if rule == "all_failed":
        if all(t == TaskState.FAILED for t in s):
            return "fire"
        return "skip"

    if rule == "all_done":
        return "fire"  # we already gated on terminal above

    if rule == "one_success":
        if any(t == TaskState.SUCCESS for t in s):
            return "fire"
        if all(t in terminal for t in s):
            return "skip"
        return "wait"

    if rule == "one_failed":
        if any(t == TaskState.FAILED for t in s):
            return "fire"
        if all(t in terminal for t in s):
            return "skip"
        return "wait"

    if rule == "none_failed":
        bad = {TaskState.FAILED, TaskState.UPSTREAM_FAILED}
        if any(t in bad for t in s):
            return "upstream_failed"
        if all(t in (TaskState.SUCCESS, TaskState.SKIPPED) for t in s):
            return "fire"
        return "wait"

    if rule == "none_skipped":
        if any(t == TaskState.SKIPPED for t in s):
            return "skip"
        if all(t in terminal for t in s):
            return "fire" if all(t == TaskState.SUCCESS for t in s) else "upstream_failed"
        return "wait"

    raise ValueError(f"unknown rule: {rule}")
```

The shape is verbose for a reason — every rule has slightly different "wait vs decide now" semantics, and writing it as a lookup table hides that. The verbosity is a feature when debugging "why didn't task X fire?" at 2 a.m.

## Branching and Skipped Nodes

Branching means an upstream node decides at runtime which of its downstreams to take. Two encodings:

- **Branch operator (Airflow `BranchPythonOperator` style).** The upstream returns the `node_id` of the next node to run. The scheduler marks all *other* downstreams as `SKIPPED` immediately. Downstream-of-skipped nodes propagate `SKIPPED` per their trigger rules.
- **Conditional gates on edges.** Each edge carries a predicate (e.g., `"$.upstream.output.kind == 'large'"`). The scheduler evaluates the predicate when the upstream succeeds. Edges that do not match are treated as if their upstream had been `SKIPPED`.

Branch operators are simpler to reason about (one decision, one place); edge conditions compose better for complex graphs but produce the "where did the decision come from" puzzle.

The critical detail is **skipped is not failed**. A `SKIPPED` task is a normal terminal state; downstreams with `NONE_FAILED` or `ALL_DONE` see it as fine; downstreams with `ALL_SUCCESS` see it as a non-success and skip themselves. A `SKIPPED` propagation that is *not* what you want is a sign you should use `NONE_FAILED_MIN_ONE_SUCCESS` (Airflow's name for the rule that fires if at least one upstream succeeded and none failed) — common around branch joins.

## Cycle Detection on Submission

A DAG with a cycle is not a DAG; it is a deadlock generator. Reject at submission time, not at runtime.

### Why DFS, Not Kahn

Two well-known algorithms:

- **Kahn's algorithm** (BFS over nodes with zero in-degree). Linear, simple, and gives a topological order as a byproduct. Useful when you need the order.
- **DFS three-coloring**. White (unvisited), gray (on the current DFS stack), black (done). A back-edge to a gray node = cycle. Linear, doesn't need to track in-degree, and the gray-stack at detection is the cycle path itself — nice for error messages.

For submission validation we do not need a topological order (the runtime walker is event-driven). We just need a yes/no plus, on no, the offending cycle. DFS three-coloring is the right pick.

### Worked Code — DFS Cycle Detection

```python
def detect_cycle(nodes: list[str],
                 edges: list[tuple[str, str]]) -> list[str] | None:
    """Returns None if acyclic. Returns the cycle as a list of node_ids
    [a, b, c, a] if a cycle is found."""
    WHITE, GRAY, BLACK = 0, 1, 2
    color = {n: WHITE for n in nodes}
    parent: dict[str, str | None] = {n: None for n in nodes}
    adj: dict[str, list[str]] = {n: [] for n in nodes}
    for u, v in edges:
        adj[u].append(v)

    def dfs(start: str) -> list[str] | None:
        # Iterative DFS with explicit stack of (node, iterator).
        stack = [(start, iter(adj[start]))]
        color[start] = GRAY
        while stack:
            node, it = stack[-1]
            try:
                nxt = next(it)
            except StopIteration:
                color[node] = BLACK
                stack.pop()
                continue
            if color[nxt] == WHITE:
                parent[nxt] = node
                color[nxt] = GRAY
                stack.append((nxt, iter(adj[nxt])))
            elif color[nxt] == GRAY:
                # Reconstruct cycle.
                cycle = [nxt]
                cur = node
                while cur is not None and cur != nxt:
                    cycle.append(cur)
                    cur = parent[cur]
                cycle.append(nxt)
                cycle.reverse()
                return cycle
            # BLACK: cross-edge or forward-edge — fine in a DAG.
        return None

    for n in nodes:
        if color[n] == WHITE:
            cycle = dfs(n)
            if cycle:
                return cycle
    return None
```

The iterative form matters because real DAG submissions can have thousands of nodes; a recursive DFS would blow the Python stack on long chains. The iterative version uses an explicit `stack` list and is bounded by graph size only.

Wire this into the API:

```python
def submit_dag(dag_id: str, nodes: list, edges: list):
    cycle = detect_cycle([n.id for n in nodes],
                         [(e.upstream, e.downstream) for e in edges])
    if cycle:
        raise InvalidDAG(f"cycle detected: {' -> '.join(cycle)}")
    # ... write a new dag_version, nodes, edges in one transaction
```

## Fan-Out and Fan-In — Dynamic Tasks

Some node counts are not known at definition time. "Process one batch per region" depends on a metadata query that runs at fire time. Two patterns:

### Static vs Dynamic Mapping

- **Static mapping.** The DAG definition lists every region explicitly. Simple, but redeploying the DAG is now a prerequisite to onboarding a new region. Often a non-starter.
- **Dynamic mapping (Airflow `expand`, Argo `withSequence` / `withItems`).** A node returns a list at runtime; the scheduler materializes one task instance per element. The mapped tasks share a parent node id but are distinguished by `map_index` (`partition-0`, `partition-1`, …). The downstream **gather** node has trigger rule `ALL_SUCCESS` against the parent — it waits for every mapped instance.

Dagster expresses this as **dynamic outputs**; Argo as **`withParam`** flowing into a child template. Same idea, different surface.

The runtime cost: the `task_instances` table count multiplies by the fan-out. Plan storage and the scheduler's eligibility-check query for the case where a single fan-out emits 10,000 children — that's 10,000 rows for one parent, not pathological but worth indexing.

### Gather Step Semantics

The gather step is not magic — it is a regular node with `ALL_SUCCESS` against the fan-out parent. What changes is how the scheduler resolves "all upstreams succeeded" when the upstream is itself a set of mapped instances:

- The scheduler stores the materialized `map_index` set when the parent expands.
- It tracks each child's terminal state.
- The gather is eligible when **every** child in that set is `SUCCESS` (or per the gather's chosen trigger rule).

If the rule is `ONE_SUCCESS`, the gather can fire while other mapped children are still running — useful for "process partitions; succeed if at least one partition produced output."

## Sensors and External Triggers

A sensor is a node whose work is "wait until X is true." Examples:

- File appears at `s3://bucket/path/`.
- HTTP endpoint returns `200`.
- Database row appears matching a predicate.
- A Kafka topic partition reaches a given offset.

Sensors come in two flavors:

- **Poke sensors.** A worker holds the slot, polls every N seconds, releases the slot only when the condition is met or it times out. Simple but burns a worker for the duration — bad if you have hundreds of cheap waits.
- **Reschedule / deferrable sensors.** The sensor checks once, returns "not yet," and the scheduler reschedules a check N seconds later. The slot is released between checks. This is the right default for long waits; Airflow's `mode='reschedule'` and the newer **deferrable operators** (Triggerer-based) implement this.

In a DAG, a sensor is just a regular node with `ALL_SUCCESS` downstream — its `SUCCESS` is the signal that the gate has opened. This unifies "external readiness" with "upstream complete" under one model: every gate is an upstream task in `SUCCESS`.

## Resource Pools and Concurrency Limits

Pools are named integer counters that gate node execution. Examples:

- A `s3_extract` pool with 5 slots — limits concurrent S3 ingest to avoid hitting account-level rate limits.
- A `gpu` pool with 2 slots matching the cluster's GPU count.
- A `db_writer` pool with 10 slots so a backfill cannot DoS the production OLTP database.

Each node spec names a pool (defaults to a global pool if unset); the scheduler will only enqueue a task instance when the named pool has a free slot. Multiple DAG runs share pools — that is the whole point. Without pools, parallel DAG runs amplify each other's load and the cluster melts.

In schema:

```sql
CREATE TABLE pools (name TEXT PRIMARY KEY, capacity INT NOT NULL);
CREATE TABLE pool_holds (
  pool_name  TEXT NOT NULL REFERENCES pools,
  run_id     UUID NOT NULL,
  node_id    TEXT NOT NULL,
  try_no     INT  NOT NULL,
  acquired_at TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (pool_name, run_id, node_id, try_no)
);
```

Acquire is a transactional "count holds; if < capacity then INSERT" — see [`./distributed-lock-per-job.md`](./distributed-lock-per-job.md) for the lease-with-fencing variant when workers can crash holding a slot.

## Backfill — Walking Historical Date Ranges

Backfill = "run the DAG for fire times in the past." Why: a new pipeline needs to populate history; a bug fix in transform logic needs to be re-applied to last month's runs; a regulator asks for a recomputation.

Backfill is a first-class operation, not a hack:

- **Materialize one DAG run per fire time** in the requested range. Each run is independent; scheduler treats them as ordinary runs, not a single mega-run.
- **Concurrency limit.** A `max_active_runs` cap (e.g., 4) keeps backfills from saturating the cluster. The scheduler launches 4 historical fires in parallel, then the next 4, until the range is exhausted.
- **Resource pools still apply.** A backfill that ignores pools is a load test on production.
- **Idempotency at the node level.** Each node must be re-runnable for a given `fire_time` without doubling effects. See [`./retry-and-idempotency.md`](./retry-and-idempotency.md) — the same idempotency that protects retries protects backfills.
- **Catchup vs explicit.** A new schedule with `catchup=True` auto-backfills from the start date; with `catchup=False`, only the next future fire counts. Default to `False`; require explicit operator action to backfill.

## Sub-DAGs and Reusable Patterns

Two reuse mechanisms:

- **Sub-DAG / Task Group.** A named subgraph embedded into a parent. The parent treats it as a single node from the outside (one upstream, one downstream); inside it has its own structure. Airflow has `TaskGroup` (declarative, lightweight); the older `SubDagOperator` is deprecated for performance reasons.
- **DAG-of-DAGs (TriggerDagRunOperator).** A node in DAG A triggers a run of DAG B and (optionally) waits for it. This is heavier but cleaner when DAG B has its own schedule, owners, or SLAs.

The trap with sub-DAGs is **scheduling them as if they were one task** — they aren't; they have N internal tasks competing for slots. Task Groups are visual-only in Airflow precisely to avoid that confusion: the grouping is for humans, the scheduler still sees individual nodes.

For Argo Workflows, the equivalents are **templates** (`steps:` and `dag:` template kinds with `templateRef:` for cross-workflow reuse). For Temporal, the equivalent is **child workflows**.

## Worked Example — ETL DAG with a Failure in Transform

Concrete graph:

```text
extract_orders ─┐
                ├─► transform ─► load ─► notify_success
extract_users  ─┘                  │
                                   └─► export_to_warehouse
                ⌡ (rule ALL_DONE) ─────► cleanup_temp
                ⌡ (rule ONE_FAILED) ──► notify_failure
```

Plain English:

- `extract_orders` and `extract_users` run in parallel (no dependency between them).
- `transform` depends on **both** extracts (`ALL_SUCCESS`).
- `load` depends on `transform` (`ALL_SUCCESS`).
- `notify_success` and `export_to_warehouse` both depend on `load` (`ALL_SUCCESS`); they run in parallel after load.
- `cleanup_temp` depends on `load` with rule `ALL_DONE` — runs whether load succeeded or not.
- `notify_failure` depends on `transform` and `load` with rule `ONE_FAILED` — runs if either failed.

Now `transform` fails after retries are exhausted. Walk:

1. `extract_orders` → `SUCCESS`. `transform` evaluates: one upstream `SUCCESS`, one still `RUNNING`. `wait`.
2. `extract_users` → `SUCCESS`. `transform` evaluates: both `SUCCESS`. `fire`. Enters `RUNNING`.
3. `transform` → `FAILED` (after retry exhaustion). On-terminal walk:
   - `load`: rule `ALL_SUCCESS` against `transform`. Upstream `FAILED` → `upstream_failed` → mark `load = UPSTREAM_FAILED`. Recurse.
     - `notify_success`: rule `ALL_SUCCESS` against `load`. Upstream `UPSTREAM_FAILED` → `upstream_failed` → mark `notify_success = UPSTREAM_FAILED`. Recurse (no downstream).
     - `export_to_warehouse`: same path → `UPSTREAM_FAILED`. Recurse (no downstream).
     - `cleanup_temp`: rule `ALL_DONE` against `load`. Upstream `UPSTREAM_FAILED` is terminal → `fire`. Enters `RUNNING`. Eventually `SUCCESS`.
   - `notify_failure`: rule `ONE_FAILED` against `(transform, load)`. `transform = FAILED` → at least one failed → `fire`. Enters `RUNNING`. Eventually `SUCCESS` (the alert goes out).

End state of the DAG run:

| Node | State |
|------|-------|
| `extract_orders` | `SUCCESS` |
| `extract_users` | `SUCCESS` |
| `transform` | `FAILED` |
| `load` | `UPSTREAM_FAILED` |
| `notify_success` | `UPSTREAM_FAILED` |
| `export_to_warehouse` | `UPSTREAM_FAILED` |
| `cleanup_temp` | `SUCCESS` |
| `notify_failure` | `SUCCESS` |

Run-level state: `FAILED` (because at least one node failed) — but `cleanup_temp` and `notify_failure` did their jobs. That asymmetry is the whole point of having distinct trigger rules: the unhappy path is also a path, and the DAG describes both.

## Anti-Patterns

- **Implicit timing dependencies.** "Job B runs at 10:05; A finishes by 10:00" is a race condition with extra steps. Make the dependency explicit in the DAG; a clock is not a synchronization primitive.
- **Shared mutable state across nodes.** Node A writes to `/tmp/state.json`; node B reads it. Works on one host, fails on a multi-worker cluster. Use durable storage (object store, message queue, DB) and pass explicit references between nodes (XComs in Airflow, outputs in Argo).
- **Non-idempotent nodes.** A node that doubles a counter every time it runs is a node that fails the first retry. Every node must be re-runnable for the same `(run_id, node_id, try_no)` without doubling effects. See [`./retry-and-idempotency.md`](./retry-and-idempotency.md).
- **Infinite retries on a single node.** `retries=999` blocks the entire run forever; the scheduler keeps holding the pool slot, the alert pages get suppressed, and the on-call engineer learns about it from the data team. Cap retries; on exhaustion, fail and let the DAG's failure path run.
- **Mutating a DAG mid-run.** Adding or removing a node while a run is in flight produces "task X has no row in `dag_nodes` for version Y" errors. The fix is hard versioning: in-flight runs read their pinned version.
- **Treating sensors as cheap.** A poke sensor that polls S3 every 30 seconds for 8 hours is 960 useless calls; a deferrable sensor is one. Pick the right kind.
- **No cycle check on submission.** Eventually a refactor wires `D → A` "just for the cleanup notification" and the next morning the scheduler is in a tight loop. The cycle check is one DFS call — there is no excuse to skip it.
- **Conflating DAG state with task state.** "The DAG ran" and "every task in the DAG succeeded" are different statements. Track both and surface the difference in alerting.
- **Sub-DAG performance traps.** Nested DAGs that schedule themselves as one task cause severe scheduler-load spikes. Use Task Groups (visual only) or explicit DAG-of-DAGs (heavier but isolated).
- **Letting the scheduler also run the work.** The scheduler decides; workers execute. A scheduler that runs tasks inline becomes the bottleneck and the single point of failure. Same lesson as [`./leader-election-for-scheduler.md`](./leader-election-for-scheduler.md): keep the control plane small and lean.
- **Backfill with no concurrency cap.** A 365-day backfill that fires 365 runs in parallel is a denial-of-service attack on your own database. Cap `max_active_runs`; treat backfill as a controlled flood, not a faucet.

## Related

- [Designing a Job Scheduler](../design-job-scheduler.md) — the parent case study; this doc expands §7.6.
- [Distributed Lock per Job](./distributed-lock-per-job.md) — how a single task instance ensures at-most-one execution despite scheduler retries.
- [Retry and Idempotency](./retry-and-idempotency.md) — what every node must guarantee for the DAG model to be safe under retries and backfill.
- [Leader Election for the Scheduler](./leader-election-for-scheduler.md) — keeping a single planner alive at the control plane so DAG runs are not double-fired.
- [ETL/ELT and Pipelines](../../../batch-and-stream/etl-elt-and-pipelines.md) — the most common use case for a DAG scheduler.
- [Event-Driven Architecture as a Style](../../../architectural-styles/event-driven-architecture-style.md) — sensors and triggers blur into event-driven processing; the DAG is the orchestration layer on top.

## References

- [Apache Airflow — DAGs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html) — the canonical reference for DAG concepts; the vocabulary in this doc tracks Airflow's.
- [Apache Airflow — Trigger Rules](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#trigger-rules) — the named rules (`ALL_SUCCESS`, `ONE_FAILED`, `NONE_FAILED_MIN_ONE_SUCCESS`, …).
- [Argo Workflows — DAG Template](https://argo-workflows.readthedocs.io/en/latest/walk-through/dag/) — the Kubernetes-native DAG runner; templates, dependencies, and `withParam` fan-out.
- [Dagster — Concepts](https://docs.dagster.io/concepts) — assets, ops, jobs; software-defined assets reframe the DAG around outputs rather than tasks.
- [Prefect — Write Flows](https://docs.prefect.io/3.0/develop/write-flows) — flows and tasks; dynamic mapping and result caching.
- [AWS Step Functions — State Types](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-states.html) — Choice, Parallel, Map; the managed-service formulation of branching and fan-out.
- [Temporal — Workflows](https://docs.temporal.io/workflows) — workflow-as-code with durable history; the "run the DAG as a program" school, complementary to declarative DAG runners.
- Donald E. Knuth, _The Art of Computer Programming, Volume 1: Fundamental Algorithms_ (3rd ed.) — §2.2.3 on topological sorting, the canonical algorithmic reference.
