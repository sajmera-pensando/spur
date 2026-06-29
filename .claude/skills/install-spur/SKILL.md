---
name: install-spur
description: Install and start a single-node Spur cluster on a remote VM via SSH
user_invocable: true
arguments:
  - name: host
    description: VM IP address or hostname
    required: true
  - name: user
    description: SSH username (default vm)
    required: false
  - name: version
    description: Spur version to install — latest, nightly, or a specific tag like v0.1.0 (default nightly)
    required: false
  - name: tarball
    description: Local path to a pre-downloaded Spur tarball (use when the VM has no GitHub access)
    required: false
  - name: cluster_name
    description: Cluster name in spur.conf (default gpu-cluster)
    required: false
---

# /install-spur — Single-Node Spur Installation

Install Spur on a remote VM and bring up a single-node cluster (controller + agent).

## Usage

```
/install-spur host=10.11.99.151
/install-spur host=10.11.99.151 user=ubuntu version=nightly
/install-spur host=10.11.99.151 tarball=/mnt/c/Users/user/Downloads/spur-nightly-20260629-linux-amd64.tar.gz
```

---

## Prerequisites

Verify before proceeding:

| Check | Command | Requirement |
|-------|---------|-------------|
| SSH connectivity | `ssh {user}@{host} 'echo OK'` | Must succeed |
| OS | `uname -s` | Linux only |
| Arch | `uname -m` | x86_64 only |
| glibc | `ldd --version` | >= 2.28 (Ubuntu 20.04+, RHEL 8+) |
| sudo | `sudo -n true` | Required for WireGuard and config |

If any prerequisite fails, report and stop.

Also inventory what is already running (`systemctl list-units --state=running`, `ps aux`) to avoid touching existing services.

---

## Step 1 — Install Binaries

### Option A: GitHub release available

```bash
ssh {user}@{host} 'curl -fsSL https://raw.githubusercontent.com/ROCm/spur/main/install.sh | bash -s -- {version}'
```

If this returns a 403 (GitHub API rate limit), fall back to Option B.

### Option B: Tarball provided or downloaded manually

The tarball may be at a Windows path like `C:\Users\...\Downloads\spur-nightly-*.tar.gz`.
On WSL, Windows paths are accessible under `/mnt/c/Users/...`.

```bash
# Copy tarball to VM
scp -o StrictHostKeyChecking=no {tarball} {user}@{host}:/tmp/spur.tar.gz

# Extract and install
ssh {user}@{host} '
  TMPDIR=$(mktemp -d)
  tar xzf /tmp/spur.tar.gz -C "$TMPDIR"
  EXTRACTED=$(find "$TMPDIR" -maxdepth 1 -type d -name "spur-*" | head -1)
  INSTALL_DIR="$HOME/.local/bin"
  mkdir -p "$INSTALL_DIR"
  cp -f "$EXTRACTED"/bin/* "$INSTALL_DIR/"
  chmod +x "$INSTALL_DIR/spur" "$INSTALL_DIR/spurctld" "$INSTALL_DIR/spurd" "$INSTALL_DIR/spurdbd"
  for sym in sbatch srun squeue scancel sinfo sacct scontrol sdiag; do
    ln -sf "$INSTALL_DIR/spur" "$INSTALL_DIR/$sym"
  done
'
```

### Add to PATH

```bash
ssh {user}@{host} 'grep -q "\.local/bin" ~/.bashrc || echo "export PATH=\"\$HOME/.local/bin:\$PATH\"" >> ~/.bashrc'
```

Verify: `bash -lc "spur --version"` — should print the version.

---

## Step 2 — Install WireGuard Tools

```bash
ssh {user}@{host} 'sudo apt-get install -y wireguard-tools 2>&1 | tail -3'
```

---

## Step 3 — Initialize WireGuard Mesh

```bash
ssh {user}@{host} 'sudo /home/{user}/.local/bin/spur net init --cidr 10.44.0.0/16 --port 51820'
```

Note the printed public key — needed if adding worker nodes later.

Use the full binary path with `sudo` since `~/.local/bin` is not in root's PATH.

---

## Step 4 — Create /etc/spur/spur.conf

Detect node hardware first:

```bash
HOSTNAME=$(ssh {user}@{host} hostname)
CPUS=$(ssh {user}@{host} nproc)
MEM_MB=$(ssh {user}@{host} 'awk "/MemTotal/{print int(\$2/1024)}" /proc/meminfo')
```

Write config:

```bash
ssh {user}@{host} "sudo mkdir -p /etc/spur && sudo tee /etc/spur/spur.conf > /dev/null << 'EOF'
cluster_name = \"{cluster_name}\"

[controller]
listen_addr = \"[::]:6817\"
hosts = [\"10.44.0.1\"]
state_dir = \"/var/spool/spur\"

[scheduler]
plugin = \"backfill\"
interval_secs = 1

[network]
wg_enabled = true
wg_interface = \"spur0\"
agent_port = 6818

[[partitions]]
name = \"default\"
default = true
nodes = \"{HOSTNAME}\"
max_time = \"72:00:00\"

[[nodes]]
names = \"{HOSTNAME}\"
cpus = {CPUS}
memory_mb = {MEM_MB}
EOF"
```

---

## Step 5 — Start spurctld (Controller)

```bash
ssh {user}@{host} 'sudo bash -c "mkdir -p /var/spool/spur && nohup /home/{user}/.local/bin/spurctld -f /etc/spur/spur.conf > /var/log/spurctld.log 2>&1 &"'
sleep 2
ssh {user}@{host} 'ss -tlnp | grep -E "6817|6821"'
```

Both ports must appear in the output before continuing.

---

## Step 6 — Start spurd (Node Agent)

```bash
ssh {user}@{host} 'sudo bash -c "nohup /home/{user}/.local/bin/spurd -f /etc/spur/spur.conf --controller http://10.44.0.1:6817 --hostname $(hostname) --listen \"[::]:6818\" > /var/log/spurd.log 2>&1 &"'
sleep 2
ssh {user}@{host} 'ss -tlnp | grep 6818'
```

---

## Step 7 — Verify

```bash
# Node registered
ssh {user}@{host} 'bash -lc "spur nodes"'
# Expected: node appears as idle

# Diagnostics
ssh {user}@{host} 'bash -lc "sdiag"'

# Metrics (controller exposes on 127.0.0.1:6822)
ssh {user}@{host} 'curl -sf http://127.0.0.1:6822/metrics | head -20'
```

---

## Ports Used by Spur

| Port | Service |
|------|---------|
| 6817 | gRPC API (spurctld) |
| 6818 | Node agent (spurd) |
| 6820 | REST API |
| 6821 | Raft internal gRPC |
| 6822 | Metrics HTTP (localhost only) |
| 51820 | WireGuard (UDP) |

---

## Known Issues

- **GitHub API 403**: unauthenticated rate limit. Use `tarball=` to provide a pre-downloaded file, or pass a `GITHUB_TOKEN` via `curl -H "Authorization: token ..."`.
- **`sudo spur` not found**: `~/.local/bin` is not in root's PATH. Always use the full path `/home/{user}/.local/bin/spur` with sudo commands.
- **Port 6821 already in use**: a previous spurctld didn't fully exit. Kill it with `sudo pkill -f spurctld` and wait a second before restarting.
- **`-D` daemonize flag drops SSH session**: use `nohup ... &` instead of the `-D` flag when starting over SSH.
- **`sdiag` not found**: create the symlink manually: `ln -sf ~/.local/bin/spur ~/.local/bin/sdiag`. The `install.sh` in this repo includes `sdiag` in its `SYMLINKS` list from v0.3.0+.

---

## Output Format

```
Installing Spur {version} on {user}@{host}
══════════════════════════════════════════

✓ SSH connectivity confirmed (Linux x86_64, glibc 2.35)
✓ Existing services inventoried — will not touch Docker, Postgres
✓ Binaries installed: spur spurctld spurd spurdbd
✓ Symlinks created: sbatch srun squeue scancel sinfo sacct scontrol sdiag
✓ PATH updated in ~/.bashrc
✓ wireguard-tools installed
✓ WireGuard mesh initialized (spur0, 10.44.0.1/16, pubkey: xxxx)
✓ /etc/spur/spur.conf written (24 CPUs, 32136 MB, hostname ubuntu2204)
✓ spurctld started (PID 12345, listening on :6817 :6821)
✓ spurd started (PID 12346, listening on :6818)

Cluster status:
  PARTITION  AVAIL  NODES  STATE    NODELIST
  default*   up     1      idle     ubuntu2204

Metrics: http://127.0.0.1:6822/metrics ✓
```
