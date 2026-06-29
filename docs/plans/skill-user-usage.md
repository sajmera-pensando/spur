# Plan: user-usage skill

## Problem

There is no single command to answer "who has consumed the most resources this week?" or "how much GPU time did team X use last month?". `sacct` returns raw per-job rows that require aggregation. VictoriaMetrics has live allocation data but no per-user breakdown in the current metric set.

## Goal

A ranked, per-user (and optionally per-account) resource consumption report over a configurable time window. Answers: who ran what, for how long, consuming how much CPU and GPU.

## Data sources

| Source | How | What it gives |
|--------|-----|---------------|
| `sacct` | SSH | Per-job: user, account, CPUs, elapsed time, state, start/end |
| VictoriaMetrics | HTTP | Live allocation snapshot per cluster |
| `scontrol show partition` | SSH | Partition limits for context |

## Computed metrics (from sacct aggregation)

- **CPU-hours** = CPUs × elapsed seconds / 3600, summed per user
- **GPU-hours** = GPUs × elapsed seconds / 3600 (from GRES field if present)
- **Job count** by state (completed, failed, cancelled, timeout)
- **Success rate** = completed / total finished
- **Avg job duration**
- **Avg queue wait time** = start time − submit time

## Output sections

### [1] Summary Header
Time window, total jobs, total CPU-hours, total GPU-hours cluster-wide.

### [2] Per-User Ranked Table
Sorted by CPU-hours (or GPU-hours if GPUs present). Columns:
```
USER       JOBS  CPU-HRS  GPU-HRS  SUCCESS%  AVG_WAIT  AVG_RUNTIME
alice        42   1204.3     96.0     97.6%     2m 14s     28m 42s
bob          18    312.1      0.0     88.9%     0m 45s     17m 21s
```

### [3] Per-Account Rollup (if accounts configured)
Same table grouped by account — useful for chargebacks.

### [4] Efficiency Flags
Call out users with high failure rates (>20%), very long queue waits (>1h avg), or jobs that ran for <1 min (likely crashed fast).

## Args

| Arg | Default | Description |
|-----|---------|-------------|
| `window` | `7d` | Time window: `24h`, `7d`, `30d`, or `YYYY-MM-DD:YYYY-MM-DD` |
| `user` | all | Filter to a specific user |
| `account` | all | Filter to a specific account |
| `sort` | `cpu-hours` | Sort by: `cpu-hours`, `gpu-hours`, `jobs`, `success-rate` |
| `top` | 20 | Show top N users |

## Open questions to discuss

- **sacct availability**: `spurdbd` (accounting daemon) was not running on the test VM (warned at startup). Need to confirm if sacct works or falls back gracefully.
- **GPU tracking**: does `sacct` include GRES/GPU fields in Spur v0.3? Need to verify the output format.
- **Per-user metric from VictoriaMetrics**: current Spur metrics have no `user` label — all per-user data must come from sacct, not VictoriaMetrics.
