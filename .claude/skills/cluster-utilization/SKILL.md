---
name: cluster-utilization
description: Show a terminal utilization dashboard for a Spur cluster, querying VictoriaMetrics for current snapshot and 24h historical metrics
user_invocable: true
arguments:
  - name: host
    description: SSH target for the VictoriaMetrics host, e.g. vm@10.11.99.151 (default vm@10.11.99.151)
    required: false
  - name: victoria_url
    description: VictoriaMetrics base URL (default http://localhost:8428, resolved on the host)
    required: false
  - name: hours
    description: Historical window in hours (default 24)
    required: false
---

# /cluster-utilization — Spur Cluster Dashboard

Query VictoriaMetrics and render a terminal utilization dashboard covering cluster health, current resource snapshot, historical utilization, and per-node breakdown.

## Defaults

- `host`: `vm@10.11.99.151`
- `victoria_url`: `http://localhost:8428`
- `hours`: `24`

---

## Step 0 — Discover clusters and ask the user

Before rendering, query VictoriaMetrics for all available cluster label values:

```bash
ssh -o StrictHostKeyChecking=no {user}@{host} \
  "curl -sf 'http://localhost:8428/api/v1/label/cluster/values'" \
| python3 -c "import sys,json; print('\n'.join(json.load(sys.stdin)['data']))"
```

Present the list to the user and ask:

> "The following Spur clusters are reporting into VictoriaMetrics:
>   [1] gpu-cluster
>   [2] gpu-cluster-2
>   ...
> Which would you like to view? Enter a number, comma-separated numbers, or 'all'."

Wait for the user's response, then build a cluster filter:
- Single cluster → `cluster="gpu-cluster"`
- Multiple clusters → render one dashboard section per cluster, or combine with `cluster=~"a|b"`
- All → omit the cluster filter (queries return across all clusters)

Append the cluster selector to every PromQL expression before running Step 1. For example:
- Single: `spur_jobs_running{cluster="gpu-cluster"}`
- All: `spur_jobs_running` (no filter)
- Multi: `spur_jobs_running{cluster=~"gpu-cluster|gpu-cluster-2"}`

---

## Execution

Write the script below to `/tmp/spur_dashboard.py` (substituting `VMURL`, `HOURS`, and `CLUSTER_FILTER` from the above), `scp` it to the host, then run it and print the output.

```bash
scp -o StrictHostKeyChecking=no /tmp/spur_dashboard.py {user}@{host}:/tmp/spur_dashboard.py
ssh -o StrictHostKeyChecking=no {user}@{host} 'python3 /tmp/spur_dashboard.py'
```

### Dashboard script

```python
import subprocess, json, datetime, time

VMURL = "http://localhost:8428"   # override from victoria_url arg
HOURS = 24                        # override from hours arg
CLUSTER_FILTER = ""               # e.g. 'cluster="gpu-cluster"' or 'cluster=~"a|b"' or "" for all

def _inject(expr):
    """Inject CLUSTER_FILTER into a PromQL metric selector."""
    if not CLUSTER_FILTER:
        return expr
    # append filter inside existing {} or add new {}
    if "{" in expr:
        return expr.replace("{", "{" + CLUSTER_FILTER + ",", 1)
    # handle bare metric name or metric{...} with label selector
    import re
    return re.sub(r'(\w+)(\b)', r'\1{' + CLUSTER_FILTER + r'}\2', expr, count=1)

def q(expr):
    try:
        r = subprocess.run(
            ["curl", "-sf", "--get", "--data-urlencode", f"query={_inject(expr)}", f"{VMURL}/api/v1/query"],
            capture_output=True, text=True, timeout=5
        )
        d = json.loads(r.stdout)
        res = d["data"]["result"]
        return res[0]["value"][1] if res else "N/A"
    except:
        return "N/A"

def qr(expr, hours=24):
    """Range query — returns list of float values, one per hour bucket."""
    try:
        now = int(time.time())
        step = max(3600, hours * 3600 // 24)
        r = subprocess.run(
            ["curl", "-sf", "--get",
             "--data-urlencode", f"query={_inject(expr)}",
             "--data-urlencode", f"start={now - hours*3600}",
             "--data-urlencode", f"end={now}",
             "--data-urlencode", f"step={step}",
             f"{VMURL}/api/v1/query_range"],
            capture_output=True, text=True, timeout=10
        )
        d = json.loads(r.stdout)
        res = d["data"]["result"]
        return [float(v[1]) for v in res[0]["values"]] if res else []
    except:
        return []

def sparkline(vals, width=24, min_range=0.01):
    """Render an 8-level ASCII bar chart, padded to `width` chars."""
    bars = "▁▂▃▄▅▆▇█"
    if not vals:
        return "░" * width
    lo, hi = min(vals), max(vals)
    # Don't draw variation for ranges smaller than min_range (default 1%)
    # — avoids misleading sparklines on near-zero metrics
    if hi == lo or (hi - lo) < min_range:
        return ("░" if hi == 0 else "█") * len(vals)
    s = "".join(bars[int((v - lo) / (hi - lo) * 7)] for v in vals)
    return s.rjust(width, "░")

def stats(vals):
    """Return (avg%, peak%, now%) strings from a list of 0-1 fractions."""
    if not vals:
        return "N/A", "N/A", "N/A"
    avg  = f"{sum(vals)/len(vals)*100:5.1f}%"
    peak = f"{max(vals)*100:5.1f}%"
    now  = f"{vals[-1]*100:5.1f}%"
    return avg, peak, now

def util_row(label, vals, na_note=""):
    if na_note and not vals:
        print(f"  {label:<32}  {'N/A':>6}  {'N/A':>6}  {'N/A':>6}  {na_note}")
        return
    avg, peak, now = stats(vals)
    spark = sparkline(vals)
    print(f"  {label:<32}  {avg:>6}  {peak:>6}  {now:>6}  {spark}")

def gb(v):
    try:
        return f"{float(v)/1024**3:.1f} GB"
    except:
        return "N/A"

def pct(num, den):
    try:
        n, d = float(num), float(den)
        return "N/A" if d == 0 else f"{n/d*100:.1f}%"
    except:
        return "N/A"

def fmt_uptime(v):
    try:
        s = int(float(v))
        d2, s = divmod(s, 86400)
        h2, s = divmod(s, 3600)
        m2, _ = divmod(s, 60)
        parts = []
        if d2: parts.append(f"{d2}d")
        if h2: parts.append(f"{h2}h")
        if m2: parts.append(f"{m2}m")
        return " ".join(parts) or "< 1m"
    except:
        return "N/A"

# CLUSTER_DISPLAY is set by the caller before writing this script:
#   single cluster  → the cluster name
#   multiple        → "gpu-cluster, gpu-cluster-2"
#   all             → "all clusters"
CLUSTER_DISPLAY = CLUSTER_FILTER or "all clusters"

# --- Instant queries ---
try:
    _r = subprocess.run(["curl","-sf","--get","--data-urlencode",
        'query=lab_monitoring_spur_metrics_alive{source="spur_metrics"}',
        f"{VMURL}/api/v1/query"], capture_output=True, text=True, timeout=5)
    alive = json.loads(_r.stdout)["data"]["result"][0]["value"][1]
except:
    alive = "N/A"
degraded      = q("lab_monitoring_spur_degraded")
nodes_total   = q("spur_nodes")
nodes_idle    = q("spur_nodes_idle")
nodes_alloc   = q("spur_nodes_alloc")
nodes_mixed   = q("spur_nodes_mixed")
nodes_down    = q("spur_nodes_down")
nodes_drain   = q("spur_nodes_drain")
uptime        = q("lab_host_uptime_seconds")
jobs_run      = q("spur_jobs_running")
jobs_pend     = q("spur_jobs_pending")
jobs_compl    = q("spur_jobs_completing")
jobs_done     = q("spur_jobs_completed")
jobs_fail     = q("spur_jobs_failed")
jobs_canc     = q("spur_jobs_cancelled")
jobs_tout     = q("spur_jobs_timeout")
jobs_oom      = q("spur_jobs_out_of_memory")
cpus_total    = q("spur_nodes_cpus")
cpus_alloc    = q("spur_jobs_cpus_alloc")
mem_total     = q("spur_nodes_memory_bytes")
mem_alloc     = q("spur_jobs_memory_alloc_bytes")
gpus_total    = q("spur_nodes_gpus")
gpus_alloc    = q("spur_jobs_gpus_alloc")
host_cpu      = q("lab_host_cpu_util_percent")
host_mem_used = q("lab_host_memory_used_bytes")
host_mem_tot  = q("lab_host_memory_total_bytes")
host_mem_avl  = q("lab_host_memory_available_bytes")
load1         = q("lab_host_load1")
load5         = q("lab_host_load5")
load15        = q("lab_host_load15")
disk_used     = q("lab_host_disk_usage_bytes")
disk_total    = q("lab_host_disk_total_bytes")

# --- Range queries (VictoriaMetrics aggregates server-side) ---
r_nodes_alloc = qr(f"spur_nodes_alloc{{{CLUSTER_FILTER}}} / spur_nodes{{{CLUSTER_FILTER}}}", HOURS)
r_cpua  = qr("spur_jobs_cpus_alloc / spur_nodes_cpus", HOURS)
# effective CPU = allocation ratio * actual host burn — not just reserved slots
r_cpue  = qr(f"spur_jobs_cpus_alloc{{{CLUSTER_FILTER}}} / spur_nodes_cpus{{{CLUSTER_FILTER}}} * avg(lab_host_cpu_util_percent{{{CLUSTER_FILTER}}}) / 100", HOURS)
r_mema  = qr("spur_jobs_memory_alloc_bytes / spur_nodes_memory_bytes", HOURS)
no_gpu  = gpus_total in ("0", "0.0", "N/A")
r_gpua  = [] if no_gpu else qr("spur_jobs_gpus_alloc / spur_nodes_gpus", HOURS)
r_gpue  = [] if no_gpu else qr(f"avg(lab_gpu_util_percent{{{CLUSTER_FILTER}}}) / 100", HOURS)

# --- Status ---
if alive in ("N/A", "0"):
    status = "✗ Metrics Unavailable"
elif degraded == "1":
    status = "● Degraded"
else:
    status = "● Healthy"

# --- Success rate ---
try:
    total_fin = sum(float(v) for v in [jobs_done, jobs_fail, jobs_canc, jobs_tout, jobs_oom] if v != "N/A")
    done_n = float(jobs_done) if jobs_done != "N/A" else 0
    succ_rate = f"{done_n/total_fin*100:.1f}%" if total_fin > 0 else "100.0%"
except:
    succ_rate = "N/A"

W = 80
now_str = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

print("=" * W)
print(f"  Spur Cluster Utilization  .  {CLUSTER_DISPLAY}  .  {now_str}")
print("=" * W)
print()
print("CLUSTER HEALTH  (instant snapshot)")
print(f"  Status    {status}")
print(f"  Nodes     {nodes_total} total  |  {nodes_idle} idle  |  {nodes_alloc} alloc  |  {nodes_mixed} mixed  |  {nodes_down} down  |  {nodes_drain} drained  (now)")
print(f"  Uptime    {fmt_uptime(uptime)}")
print()
print("CURRENT SNAPSHOT")
print(f"  Jobs      {jobs_run} running  |  {jobs_pend} pending  |  {jobs_compl} completing")
print(f"            {jobs_done} completed  |  {jobs_fail} failed  |  {jobs_canc} cancelled  |  {jobs_tout} timeout  |  {jobs_oom} OOM")
print(f"  CPUs      {cpus_alloc} / {cpus_total} allocated  ({pct(cpus_alloc, cpus_total)})")
print(f"  Memory    {gb(mem_alloc)} / {gb(mem_total)} allocated  ({pct(mem_alloc, mem_total)})")
gpu_line = "N/A (no GPUs registered)" if no_gpu else f"{gpus_alloc} / {gpus_total} allocated  ({pct(gpus_alloc, gpus_total)})"
print(f"  GPUs      {gpu_line}")
try:
    free_nodes = int(float(nodes_idle)) if nodes_idle != "N/A" else "N/A"
    free_cpus  = int(float(cpus_total)) - int(float(cpus_alloc)) if cpus_total != "N/A" and cpus_alloc != "N/A" else "N/A"
    free_gpus  = int(float(gpus_total)) - int(float(gpus_alloc)) if gpus_total != "N/A" and gpus_alloc != "N/A" and not no_gpu else "N/A"
    gpu_free_str = f" | {free_gpus} free GPUs" if free_gpus != "N/A" else ""
    print(f"  Free      {free_nodes} idle nodes  |  {free_cpus} free CPUs{gpu_free_str}  (now)")
except:
    pass
try:
    print(f"  Host CPU  {float(host_cpu):.1f}%  load avg: {load1} / {load5} / {load15}")
except:
    print(f"  Host CPU  {host_cpu}  load avg: {load1} / {load5} / {load15}")
print(f"  Host Mem  {gb(host_mem_used)} used  |  {gb(host_mem_avl)} free  |  {gb(host_mem_tot)} total")
print(f"  Disk      {gb(disk_used)} / {gb(disk_total)}  ({pct(disk_used, disk_total)})")
print()
print(f"UTILIZATION -- LAST {HOURS}h  (historical trends)")
print(f"  {'Metric':<32}  {'avg':>6}  {'peak':>6}  {'now':>6}  Trend (oldest->now)")
print(f"  {'-'*32}  {'-'*6}  {'-'*6}  {'-'*6}  {'-'*24}")
util_row("Nodes -- allocated",         r_nodes_alloc)
util_row("CPU  -- allocated",          r_cpua)
util_row("CPU  -- effective",          r_cpue)
util_row("Mem  -- allocated",          r_mema)
if no_gpu:
    util_row("GPU  -- allocated",      [],  "N/A (no GPUs registered)")
    util_row("GPU  -- effective",      [],  "N/A (no GPUs registered)")
else:
    util_row("GPU  -- allocated",      r_gpua)
    util_row("GPU  -- effective",      r_gpue)
print()
print(f"  Success rate    {succ_rate}  ({jobs_done} completed / {jobs_fail} failed / {jobs_canc} cancelled)")
print()

# --- Capacity insights ---
print("CAPACITY INSIGHTS")
try:
    now_ts = int(time.time())
    step_s = max(3600, HOURS * 3600 // 24)

    if r_nodes_alloc:
        free_frac = [1.0 - v for v in r_nodes_alloc]
        avg_free_nodes = sum(free_frac) / len(free_frac)
        peak_free_frac = max(free_frac)
        min_free_frac  = min(free_frac)

        start_ts = now_ts - HOURS * 3600
        ts_list  = [start_ts + i * step_s for i in range(len(free_frac))]

        peak_free_idx  = free_frac.index(peak_free_frac)
        min_free_idx   = free_frac.index(min_free_frac)
        peak_free_time = datetime.datetime.fromtimestamp(ts_list[peak_free_idx]).strftime("%m-%d %H:%M")
        min_free_time  = datetime.datetime.fromtimestamp(ts_list[min_free_idx]).strftime("%m-%d %H:%M")

        try:
            n_total = int(float(nodes_total))
            avg_free_n  = avg_free_nodes * n_total
            peak_free_n = peak_free_frac * n_total
            min_free_n  = min_free_frac  * n_total
        except:
            avg_free_n = peak_free_n = min_free_n = None

        node_str = f" ({avg_free_n:.1f} nodes avg)" if avg_free_n is not None else ""
        print(f"  Avg free capacity   {avg_free_nodes*100:.1f}%{node_str}  over last {HOURS}h")

        if avg_free_n is not None:
            print(f"  Most available      {peak_free_time}  →  {peak_free_frac*100:.1f}% free  ({peak_free_n:.0f} nodes)")
            print(f"  Least available     {min_free_time}  →  {min_free_frac*100:.1f}% free  ({min_free_n:.0f} nodes)")
        else:
            print(f"  Most available      {peak_free_time}  →  {peak_free_frac*100:.1f}% free")
            print(f"  Least available     {min_free_time}  →  {min_free_frac*100:.1f}% free")

        if r_gpua and not no_gpu:
            gpu_free_frac = [1.0 - v for v in r_gpua]
            avg_gpu_free  = sum(gpu_free_frac) / len(gpu_free_frac)
            try:
                g_total = int(float(gpus_total))
                print(f"  Avg free GPUs       {avg_gpu_free*100:.1f}%  ({avg_gpu_free*g_total:.1f} GPUs avg)")
            except:
                print(f"  Avg free GPUs       {avg_gpu_free*100:.1f}%")

        if HOURS < 168:
            print(f"  Tip  Run with hours=168 to reveal weekly patterns and day-of-week capacity trends.")
    else:
        print("  N/A (insufficient historical data)")
except Exception as e:
    print(f"  N/A ({e})")
print()

# --- GPU Hardware section ---
if not no_gpu:
    CF = CLUSTER_FILTER
    gpu_util_avg  = q(f"avg(lab_gpu_util_percent{{{CF}}})")
    gpu_util_max  = q(f"max(lab_gpu_util_percent{{{CF}}})")
    gpu_pwr_avg   = q(f"avg(lab_gpu_power_watts{{{CF}}})")
    gpu_pwr_sum   = q(f"sum(lab_gpu_power_watts{{{CF}}})")
    gpu_temp_avg  = q(f"avg(lab_gpu_temp_celsius{{{CF}}})")
    gpu_temp_max  = q(f"max(lab_gpu_temp_celsius{{{CF}}})")
    gpu_ecc_total = q(f"sum(lab_gpu_ecc_errors_total{{{CF}}})")
    try:
        gpu_util_avg_s = f"{float(gpu_util_avg):.1f}%"
        gpu_util_max_s = f"{float(gpu_util_max):.1f}%"
    except:
        gpu_util_avg_s = gpu_util_max_s = "N/A"
    try:
        gpu_pwr_avg_s = f"{float(gpu_pwr_avg):.0f} W"
        gpu_pwr_sum_s = f"{float(gpu_pwr_sum):.0f} W"
    except:
        gpu_pwr_avg_s = gpu_pwr_sum_s = "N/A"
    try:
        gpu_temp_avg_s = f"{float(gpu_temp_avg):.1f} °C"
        gpu_temp_max_s = f"{float(gpu_temp_max):.0f} °C"
    except:
        gpu_temp_avg_s = gpu_temp_max_s = "N/A"
    ecc_s = gpu_ecc_total if gpu_ecc_total != "N/A" else "N/A"
    print("GPU HARDWARE  (cluster aggregate)")
    print(f"  Utilization   avg {gpu_util_avg_s}  |  max {gpu_util_max_s}")
    print(f"  Power         avg {gpu_pwr_avg_s} per GPU  |  total {gpu_pwr_sum_s}")
    print(f"  Temperature   avg {gpu_temp_avg_s}  |  max {gpu_temp_max_s}")
    print(f"  ECC errors    {ecc_s} (cumulative across all GPUs)")
    print()

# --- NIC section ---
CF = CLUSTER_FILTER
nic_links_up    = q(f"sum(lab_nic_link_up{{{CF}}})")
nic_links_total = q(f"count(lab_nic_link_up{{{CF}}})")
net_rx_rate     = q(f"sum(irate(lab_host_net_rx_bytes_total{{{CF}}}[5m]))")
net_tx_rate     = q(f"sum(irate(lab_host_net_tx_bytes_total{{{CF}}}[5m]))")
def mbps(v):
    try:
        return f"{float(v)*8/1e6:.1f} Mbps"
    except:
        return "N/A"
print("NIC  (cluster aggregate, iface=eth2)")
print(f"  Links         {nic_links_up} / {nic_links_total} up")
print(f"  Throughput    RX {mbps(net_rx_rate)}  |  TX {mbps(net_tx_rate)}  (5m avg)")
print()

# --- Per-node breakdown ---
try:
    r2 = subprocess.run(
        ["curl", "-sf", "--get", "--data-urlencode", f"query=spur_node_cpu_load{{{CLUSTER_FILTER}}}",
         f"{VMURL}/api/v1/query"],
        capture_output=True, text=True, timeout=5
    )
    nodes_data = json.loads(r2.stdout)["data"]["result"]
except:
    nodes_data = []

# Fetch per-node GPU aggregates (avg util, avg power, max temp, ecc) in batch
try:
    gpu_util_by_node  = {s["metric"].get("serial","?"): float(s["value"][1])
        for s in json.loads(subprocess.run(["curl","-sf","--get","--data-urlencode",
            f"query=avg by (serial) (lab_gpu_util_percent{{{CLUSTER_FILTER}}})",
            f"{VMURL}/api/v1/query"],capture_output=True,text=True,timeout=5).stdout)["data"]["result"]}
    gpu_pwr_by_node   = {s["metric"].get("serial","?"): float(s["value"][1])
        for s in json.loads(subprocess.run(["curl","-sf","--get","--data-urlencode",
            f"query=avg by (serial) (lab_gpu_power_watts{{{CLUSTER_FILTER}}})",
            f"{VMURL}/api/v1/query"],capture_output=True,text=True,timeout=5).stdout)["data"]["result"]}
    gpu_temp_by_node  = {s["metric"].get("serial","?"): float(s["value"][1])
        for s in json.loads(subprocess.run(["curl","-sf","--get","--data-urlencode",
            f"query=max by (serial) (lab_gpu_temp_celsius{{{CLUSTER_FILTER}}})",
            f"{VMURL}/api/v1/query"],capture_output=True,text=True,timeout=5).stdout)["data"]["result"]}
    gpu_ecc_by_node   = {s["metric"].get("serial","?"): int(float(s["value"][1]))
        for s in json.loads(subprocess.run(["curl","-sf","--get","--data-urlencode",
            f"query=sum by (serial) (lab_gpu_ecc_errors_total{{{CLUSTER_FILTER}}})",
            f"{VMURL}/api/v1/query"],capture_output=True,text=True,timeout=5).stdout)["data"]["result"]}
except:
    gpu_util_by_node = gpu_pwr_by_node = gpu_temp_by_node = gpu_ecc_by_node = {}

# Fetch cpus_alloc history for all nodes in one range query to compute idle durations
NODE_STEP = 300  # 5-minute resolution
_now = int(time.time())
try:
    r3 = subprocess.run(
        ["curl", "-sf", "--get",
         "--data-urlencode", f"query=spur_node_cpus_alloc{{{CLUSTER_FILTER}}}",
         "--data-urlencode", f"start={_now - HOURS*3600}",
         "--data-urlencode", f"end={_now}",
         "--data-urlencode", f"step={NODE_STEP}",
         f"{VMURL}/api/v1/query_range"],
        capture_output=True, text=True, timeout=15
    )
    alloc_history = {
        s["metric"].get("node", "?"): s["values"]
        for s in json.loads(r3.stdout)["data"]["result"]
    }
except:
    alloc_history = {}

def node_idle_info(node):
    """Return (state, idle_since_str, idle_dur_str, times_used) for a node."""
    vals = alloc_history.get(node, [])
    if not vals:
        return "unknown", "N/A", "N/A", 0

    times_used = 0
    prev_alloc = None
    for ts, val in vals:
        alloc = float(val) > 0
        if prev_alloc is True and not alloc:
            times_used += 1
        prev_alloc = alloc

    current_alloc = float(vals[-1][1]) > 0
    if current_alloc:
        return "alloc", "—", "—", times_used

    # Walk backwards to find last transition from alloc→idle
    idle_since_ts = None
    for i in range(len(vals) - 1, -1, -1):
        if float(vals[i][1]) > 0:
            if i + 1 < len(vals):
                idle_since_ts = int(vals[i + 1][0])
            break

    if idle_since_ts is None:
        idle_since_ts = int(vals[0][0])

    idle_dur_s = _now - idle_since_ts
    d2, r2 = divmod(idle_dur_s, 86400)
    h2, r2 = divmod(r2, 3600)
    m2 = r2 // 60
    parts = []
    if d2: parts.append(f"{d2}d")
    if h2: parts.append(f"{h2}h")
    if m2: parts.append(f"{m2}m")
    dur_str = " ".join(parts) or "< 1m"
    since_str = datetime.datetime.fromtimestamp(idle_since_ts).strftime("%m-%d %H:%M")
    return "idle", since_str, dur_str, times_used

NW = max((len(nd["metric"].get("node", "?")) for nd in nodes_data), default=4)
NW = max(NW, 4)
print("PER-NODE BREAKDOWN")
print(f"  {'NODE':<{NW}} {'STATE':<6} {'CPUS':>4} {'CPU_LOAD':>8} {'MEM_ALLOC':>10} {'GPUS':>6} {'GPU_UTIL':>8} {'PWR/GPU':>7} {'TEMP':>5} {'ECC':>4} {'IDLE_SINCE':>11} {'IDLE_DUR':>10} {'USED':>4}")
print(f"  {'-'*NW} {'-'*6} {'-'*4} {'-'*8} {'-'*10} {'-'*6} {'-'*8} {'-'*7} {'-'*5} {'-'*4} {'-'*11} {'-'*10} {'-'*4}")
for nd in nodes_data:
    node = nd["metric"].get("node", "?")
    # spur_node_cpu_load is reported in millicores — divide by 1000 for cores
    cpu_load = float(nd["value"][1]) / 1000
    nc  = q(f'spur_node_cpus{{node="{node}",{CLUSTER_FILTER}}}')
    nca = q(f'spur_node_cpus_alloc{{node="{node}",{CLUSTER_FILTER}}}')
    nma = q(f'spur_node_memory_alloc_bytes{{node="{node}",{CLUSTER_FILTER}}}')
    nga = q(f'spur_node_gpus_alloc{{node="{node}",{CLUSTER_FILTER}}}')
    ng  = q(f'spur_node_gpus{{node="{node}",{CLUSTER_FILTER}}}')
    state, idle_since, idle_dur, times_used = node_idle_info(node)
    if state == "unknown":
        state = "idle" if nca in ("0", "0.0", "N/A") else "alloc"
    mem_str = gb(nma)
    gpu_str = "N/A" if ng in ("0", "0.0", "N/A") else f"{nga}/{ng}"
    util_s = f"{gpu_util_by_node[node]:.0f}%"  if node in gpu_util_by_node else "N/A"
    pwr_s  = f"{gpu_pwr_by_node[node]:.0f}W"   if node in gpu_pwr_by_node  else "N/A"
    temp_s = f"{gpu_temp_by_node[node]:.0f}°C" if node in gpu_temp_by_node else "N/A"
    ecc_s  = str(gpu_ecc_by_node.get(node, "N/A"))
    print(f"  {node:<{NW}} {state:<6} {nc:>4} {cpu_load:>8.2f} {mem_str:>10} {gpu_str:>6} {util_s:>8} {pwr_s:>7} {temp_s:>5} {ecc_s:>4} {idle_since:>11} {idle_dur:>10} {times_used:>4}")

print()
print("=" * W)
print(f"  Data: VictoriaMetrics @ {VMURL}  |  window: last {HOURS}h")
if HOURS < 168:
    print(f"  Tip  Use hours=168 to see weekly capacity patterns and day-of-week utilization trends.")
print("=" * W)
print()
print("TERMINOLOGY")
print("  idle        Node is registered and healthy but has 0 CPUs/GPUs allocated.")
print("  alloc       Node has CPUs/GPUs assigned to one or more running jobs.")
print("  GPU_UTIL    Average utilization % across all GPUs on the node (from hardware exporter).")
print("  PWR/GPU     Average power draw per GPU in watts.")
print("  TEMP        Maximum GPU temperature across the node's GPUs (°C).")
print("  ECC         Cumulative ECC memory error count across the node's GPUs.")
print("  IDLE_SINCE  Timestamp when the node last transitioned from allocated → idle.")
print("  IDLE_DUR    How long the node has been continuously idle since IDLE_SINCE.")
print("  USED        Number of alloc→idle transitions in the window (not individual jobs).")
print("  Success %   completed / (completed + failed + cancelled + timeout + OOM).")
```

---

## Error handling

| Scenario | Action |
|----------|--------|
| SSH unreachable | Print `ERROR: cannot reach {host}` and stop |
| VictoriaMetrics unreachable | Print `ERROR: VictoriaMetrics at {victoria_url} not responding` and stop |
| Individual metric returns N/A | Show `N/A` in that field, continue rendering |
| Division by zero (gpus_total=0, etc.) | Show `N/A`, do not crash |
| Range query returns fewer than 24 points | Pad sparkline left with `░` |

---

## Examples

```
/cluster-utilization
/cluster-utilization host=vm@10.11.99.151
/cluster-utilization hours=48
/cluster-utilization victoria_url=http://10.11.99.151:8428
```
