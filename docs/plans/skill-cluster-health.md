# Plan: cluster-health skill

## Problem

Checking cluster health requires querying multiple systems manually: `sinfo` for node states, `sdiag` for scheduler stats, VictoriaMetrics for metrics liveness, and log files for errors. There is no single command that gives a pass/fail health verdict with actionable detail.

## Goal

A fast, opinionated health check that returns GREEN / YELLOW / DEGRADED / RED with specific reasons. Designed to be run as a morning sanity check or as a first step before any maintenance operation.

## Data sources

| Source | Check |
|--------|-------|
| VictoriaMetrics | Metrics scrape alive, not degraded |
| `sdiag` | Controller reachable, Raft leader elected, scheduler cycling |
| `sinfo` | Node states — any down/error/unknown nodes |
| `squeue` | Stuck jobs (running > time limit, pending > threshold) |
| SSH to nodes | `spurd` process alive on each compute node |
| Log files | Recent ERROR lines in `/var/log/spurctld.log`, `/var/log/spurd.log` |

## Health levels

| Level | Meaning |
|-------|---------|
| ✅ GREEN | All checks pass |
| ⚠️ YELLOW | Minor issues — cluster functional but attention needed (e.g. 1 node drained, high queue) |
| 🔶 DEGRADED | Cluster running but impaired (e.g. node down, metrics not scraping) |
| 🔴 RED | Cluster non-functional (controller unreachable, no nodes idle, Raft has no leader) |

## Checks

### Controller
- [ ] `sdiag` responds within 2s
- [ ] Raft leader is elected (from sdiag output)
- [ ] Scheduler cycle count is incrementing (not frozen)
- [ ] REST API responds on port 6820

### Nodes
- [ ] No nodes in DOWN or ERROR state
- [ ] At least one node is IDLE or MIXED (cluster can accept jobs)
- [ ] `spurd` process running on each registered node (via SSH)
- [ ] No nodes stuck in DRAINING > 30 min with no running jobs

### Jobs
- [ ] No jobs stuck in RUNNING beyond their time limit + 5 min grace
- [ ] Pending queue depth < configurable threshold (default 100)
- [ ] No jobs stuck in COMPLETING > 10 min

### Metrics
- [ ] `lab_monitoring_spur_metrics_alive == 1`
- [ ] `lab_monitoring_spur_degraded == 0`
- [ ] Last scrape timestamp < 2 min ago

### Logs (last 15 min)
- [ ] No ERROR lines in spurctld.log
- [ ] No ERROR lines in spurd.log on compute nodes

## Output

```
Spur Cluster Health  .  gpu-cluster  .  2026-06-29 11:00:00
════════════════════════════════════════════════════

Overall: ✅ GREEN  (12 checks passed, 0 warnings, 0 failures)

CONTROLLER
  ✅ sdiag responding         (42ms)
  ✅ Raft leader elected      (node_id=1)
  ✅ Scheduler cycling        (1842 cycles)
  ✅ REST API responding      (:6820)

NODES  (1 total)
  ✅ No nodes DOWN/ERROR
  ✅ ubuntu2204 — idle, spurd running

JOBS
  ✅ No jobs over time limit
  ✅ Queue depth: 0 pending
  ✅ No jobs stuck in COMPLETING

METRICS
  ✅ Metrics alive, not degraded
  ✅ Last scrape: 18s ago

LOGS
  ✅ No errors in spurctld.log (last 15m)
  ✅ No errors in spurd.log (last 15m)
════════════════════════════════════════════════════
```

## Args

| Arg | Default | Description |
|-----|---------|-------------|
| `host` | `vm@10.11.99.151` | SSH target (controller node) |
| `pending_threshold` | `100` | Warn if pending jobs exceed this |
| `log_window_min` | `15` | Minutes of logs to scan for errors |

## Open questions to discuss

- **Multi-node**: SSH health check to each compute node requires knowing their IPs. Source from `sinfo` node list + known SSH credentials, or skip and just check via controller?
- **Log scanning**: `/var/log/spurctld.log` path is convention from our deployment — should be configurable.
- **Raft leader check**: `sdiag` output format — does it expose Raft leader info in v0.3? Need to verify.
- **Alerting hook**: should the skill optionally post a Slack message if RED/DEGRADED?
