# Plan: queue-summary skill

## Problem

`squeue` gives a point-in-time snapshot with no historical context. Operators cannot answer "has the queue been growing for the last 3 hours?" or "which reason has been blocking jobs the most today?" without external tooling.

## Goal

A queue dashboard combining live per-job detail (from CLI) with historical reason-level trends (from VictoriaMetrics), giving operators both the current state and the context to understand whether it is normal.

---

## Prerequisite: enriched pending metrics (tracked separately)

The skill assumes that `spur_jobs_pending` will be exported with a `reason` label, one series per `PendingReason` variant. For example:

```
spur_jobs_pending{cluster="gpu-cluster", reason="Resources"}   8
spur_jobs_pending{cluster="gpu-cluster", reason="Dependency"}  3
spur_jobs_pending{cluster="gpu-cluster", reason="JobHeldUser"} 1
spur_jobs_pending{cluster="gpu-cluster", reason="Priority"}    0
...
```

This change is tracked separately and needs to be implemented in
`crates/spur-metrics/src/export/jobs.rs` by grouping the pending job snapshot
by `pending_reason` before registering gauges.

Until this lands, the historical sections of the skill degrade gracefully to
showing the unlabeled `spur_jobs_pending` total only.

---

## Data sources

| Source | How | What it gives |
|--------|-----|---------------|
| `squeue -l` | SSH | Live job list: ID, name, user, state, partition, nodes, elapsed, time limit |
| `scontrol show job <id>` | SSH (pending jobs only) | `pending_reason` string per job |
| `sinfo` | SSH | Node states and availability per partition |
| `sdiag` | SSH | Scheduler cycle count, RPC stats |
| VictoriaMetrics `query` | HTTP | Current per-reason pending counts (snapshot cross-check) |
| VictoriaMetrics `query_range` | HTTP | Historical per-reason trends over the requested window |

---

## Pending reason groups

Reasons are grouped for display. Source: `PendingReason` enum in
`spur-core/src/job.rs`. All actively emitted by the scheduler in v0.3:

| Display group | Reason strings |
|---------------|----------------|
| Waiting for resources | `Resources`, `Priority`, `NodeDown`, `Reservation`, `Licenses`, `BurstBufferResources` |
| Dependency | `Dependency`, `DependencyNeverSatisfied` |
| Held | `JobHeldUser`, `JobHeldAdmin` |
| Partition issues | `PartitionConfig`, `PartitionInactive` |
| Staging | `BurstBufferStageIn` |
| Terminal / failed | `DeadLine`, `OutOfMemory`, `RaisedSignal`, `NonZeroExitCode` |

QOS reasons (`QOSMaxCpuPerJobLimit`, `QOSGrpCpuLimit`, etc.) are defined in the
enum but have no live emission path in v0.3. Include them in the grouping logic
so the skill handles them automatically once they are wired up.

---

## VictoriaMetrics queries

### Current snapshot (instant queries)

```promql
# Total pending by reason
spur_jobs_pending{cluster="gpu-cluster"}

# Running and resource allocation
spur_jobs_running{cluster="gpu-cluster"}
spur_jobs_cpus_alloc{cluster="gpu-cluster"}
spur_jobs_gpus_alloc{cluster="gpu-cluster"}
spur_nodes_cpus{cluster="gpu-cluster"}
spur_nodes_gpus{cluster="gpu-cluster"}
```

### Historical trends (range queries, step=15m)

```promql
# Pending by reason over time — the core historical view
spur_jobs_pending{cluster="gpu-cluster", reason="Resources"}
spur_jobs_pending{cluster="gpu-cluster", reason="Dependency"}
spur_jobs_pending{cluster="gpu-cluster", reason="Priority"}
spur_jobs_pending{cluster="gpu-cluster", reason="JobHeldUser"}
# ... one query per reason, or use regex:
spur_jobs_pending{cluster="gpu-cluster"}  # returns all reason series

# Running trend for context
spur_jobs_running{cluster="gpu-cluster"}

# Total pending trend (fallback if reason label not yet available)
sum(spur_jobs_pending{cluster="gpu-cluster"})
```

From range data, compute:
- **Peak pending** — max value in window per reason
- **Avg pending** — average over window per reason
- **Dominant reason** — which reason had highest average over the window
- **Trend direction** — compare last 30m avg vs window avg: growing / draining / steady

---

## Output sections

### [1] Live Queue Snapshot

```
Queue Summary  .  gpu-cluster  .  2026-06-29 11:00
════════════════════════════════════════════════════════════════
  Running    3  |  Pending  12  |  Completing  0
  CPUs       18 / 24 allocated  (75.0%)
  GPUs       N/A (no GPUs registered)

  Pending trend  (last 6h, 15m buckets):
    Total        ▁▁▂▃▅▆▇█  peak: 14  now: 12  ↑ growing
    Resources    ▁▁▂▃▄▅▆▇  peak: 10  now:  8
    Dependency   ░░░░░▁▁▂  peak:  3  now:  3
    Held         ░░░░░░░▁  peak:  1  now:  1

  Dominant reason last 6h: Resources (avg 5.2 jobs)
```

### [2] Pending Jobs — Grouped by Reason

```
PENDING JOBS  (12 total)
────────────────────────────────────────────────────────────────
  Waiting for Resources  (8 jobs)
    JOBID   NAME          USER    CPUS   MEM     WAITED
    101     train-gpt     alice   16     64 GB   2h 14m
    102     train-gpt     alice   16     64 GB   2h 13m
    108     eval-run      bob      4     16 GB   1h 02m
    ...

  Dependency  (3 jobs)
    JOBID   NAME          USER    DEPENDS ON   WAITED
    110     postproc      bob     106          45m
    111     postproc      bob     107          45m
    114     aggregate     carol   110,111      44m

  Held  (1 job)
    JOBID   NAME          USER    HELD BY      WAITED
    115     debug-run     carol   user         10m
```

### [3] Running Jobs

```
RUNNING JOBS  (3 total)
────────────────────────────────────────────────────────────────
  JOBID   NAME        USER    NODES         CPUS   ELAPSED    LIMIT
  90      pretrain    alice   gpu-node-1    8      1h 42m     4h 00m
  91      eval        bob     gpu-node-2    4      0h 18m     1h 00m
  92      preproc     carol   gpu-node-1    6      0h 05m     2h 00m
```

### [4] Partition Availability

```
PARTITIONS
  NAME      STATE   NODES   IDLE   ALLOC   DOWN   TIMELIMIT
  default   up      4       1      3       0      72:00:00
  gpu       up      2       0      2       0      24:00:00
```

---

## Historical window analysis

When `hours` arg is provided (default 6), add a summary section:

```
QUEUE ANALYSIS — LAST 6h
────────────────────────────────────────────────────────────────
  Reason             Avg jobs   Peak jobs   % of window with >0
  Resources          5.2        10          84%
  Dependency         0.8         3          42%
  Held               0.1         1          12%
  Priority           0.0         0           0%

  Interpretation:
    ● Resources was the dominant blocker (84% of the window)
    ● Queue is growing — pending up 60% vs 3h ago
    ● 3 dependency chains active; oldest waiting 45m
```

Trend direction logic:
- **Growing**: last 30m avg > window avg by >20%
- **Draining**: last 30m avg < window avg by >20%
- **Steady**: within 20%

---

## Args

| Arg | Default | Description |
|-----|---------|-------------|
| `host` | `vm@10.11.99.151` | SSH target (controller node) |
| `hours` | `6` | Historical window for trend analysis |
| `user` | all | Filter live job list to a specific user |
| `partition` | all | Filter live job list to a specific partition |

---

## Graceful degradation

If per-reason metrics are not yet available (label `reason` absent):

- Historical section shows total pending trend only (no per-reason breakdown)
- Print a note: `(per-reason history unavailable — spur_jobs_pending reason label not yet exported)`
- Live grouping by reason still works (sourced from `scontrol show job`)

---

## Implementation notes

- `scontrol show job` called once per pending job to get reason — fine for queues
  up to ~200 jobs. For larger queues, check if `squeue` REASON column suffices
  (avoids N+1 SSH calls).
- All Victoria `query_range` calls use `step=15m` for 6h window (24 data points).
- Sparklines rendered with 8-level block chars `▁▂▃▄▅▆▇█`, padded with `░`.
- Script delivered via `scp` + SSH `python3`, same pattern as `cluster-utilization`.
