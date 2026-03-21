---
name: rk8s-single
description: "Deploy rk8s container cluster on a single machine with SlayerFS volume support. Use this skill whenever the user wants to: start/deploy/run rk8s containers or pods on a machine, set up a single-node rk8s cluster (Xline + RKS + RKL), test SlayerFS volumes, run nginx/busybox or other containers via rk8s, or anything involving rkl/rks/rkforge deployment. Trigger even if the user just says 'start a container on the VM' or 'deploy rk8s' or 'run a pod with volume'."
---

# rk8s Single-Machine Cluster Deployment

This skill deploys a complete rk8s cluster on a single machine with 4 pods (1 nginx + 3 busybox), each with a SlayerFS persistent volume. Acceptance criteria: busybox pods ping each other and volume read/write works.

## Architecture

```
┌─ Single Machine ──────────────────────────────────┐
│  Xline (3 Docker containers)  ← state storage     │
│  RKS (control plane)          ← scheduler, CSI    │
│  RKL daemon (worker node)     ← container mgmt    │
│  libfuse overlay (per-container) ← rootfs mount   │
│  SlayerFS (FUSE, in-process)  ← volume backend    │
└────────────────────────────────────────────────────┘

Pods deployed:
  nginx     (nginx:latest)    + slayerfs volume at /data
  busybox-1 (busybox:latest)  + slayerfs volume at /data
  busybox-2 (busybox:latest)  + slayerfs volume at /data
  busybox-3 (busybox:latest)  + slayerfs volume at /data
```

## Step 0: Determine Target Environment

Before doing anything, ask the user:

1. **Target machine**: Is this the local machine or a remote host via SSH?
   - If remote: get the SSH alias or `user@host` and verify connectivity
   - If local: commands run directly with `sudo`
2. **Project path**: Where is the rk8s repo? (default: `~/rk8s/project` or `~/yrj/rk8s/project`)

Based on the answers, set these variables for all subsequent steps:
- `SSH_CMD`: empty for local, `ssh <host>` for remote
- `PROJECT_DIR`: path to rk8s/project
- `BIN`: `$PROJECT_DIR/target/release`

For remote hosts, prefix all commands with `$SSH_CMD`.

## Step 1: Check Prerequisites

Check these in parallel — skip any step where the prerequisite is already met:

```bash
# Check if binaries are compiled (CRITICAL: skip compilation if they exist, it takes 15-20 min)
ls $BIN/rkl $BIN/rks 2>/dev/null && echo "Binaries exist" || echo "NEED COMPILATION"

# Check CNI config
sudo ls /etc/cni/net.d/test.conflist 2>/dev/null && echo "CNI configured" || echo "NEED CNI SETUP"

# Check rkforge config
sudo cat ~/.config/rk8s/rkforge.toml 2>/dev/null | grep insecure || echo "NEED RKFORGE CONFIG"

# Check if services are already running
ps aux | grep -E 'rks.*start|rkl.*daemon' | grep -v grep
docker ps --format '{{.Names}}' | grep -E 'node[123]'
```

Only proceed with steps that are actually needed. Never recompile if binaries exist.

## Step 2: Compile (only if needed)

```bash
cd $PROJECT_DIR && cargo build --release
```

Timeout: 20 minutes. This produces `rkl`, `rks` in `target/release/`.

## Step 3: Configure CNI

CNI plugins are **embedded in the rkl binary** — no external binaries needed. Only config files:

```bash
sudo mkdir -p /etc/cni/net.d
sudo cp $PROJECT_DIR/test/test.conflist $PROJECT_DIR/test/subnet.env /etc/cni/net.d/
```

## Step 4: Configure rkforge Registry

The default registry is `47.79.87.161:8968` (the project's distribution server). It uses HTTP, but rkforge defaults to HTTPS. Fix with:

```bash
mkdir -p ~/.config/rk8s
cat > ~/.config/rk8s/rkforge.toml << 'EOF'
entries = []

[registry]
insecure-registries = ["47.79.87.161:8968"]

[image]
EOF
```

**Important**: When running rkl with `sudo`, rkforge reads the config of `$SUDO_USER` (the original non-root user), not root's config.

**Available images on the default registry**:
- `nginx:latest` (~60MB)
- `busybox:latest` (~2MB)
- `pause:3.9` (~300KB, used internally for pod sandbox)

**Note**: `busybox:1.36` is NOT available on the default registry. Always use `busybox:latest`.

## Step 5: Clean Up Previous State (if restarting)

**CRITICAL**: If restarting from a failed or previous deployment, you MUST clean all state before re-deploying. Leftover containers in `/run/youki/` cause "container already exists" errors that block pod creation permanently.

```bash
# Stop services
sudo killall rkl rks 2>/dev/null
sleep 2

# Unmount all rkl-related FUSE and overlay mounts (MUST do before rm)
mount | grep -E "fuse.*rkl|slayerfs|overlay.*rkl" | awk '{print $3}' | while read mnt; do
  sudo umount -l "$mnt" 2>/dev/null
done

# Clean container runtime state
sudo find /run/youki -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null

# Clean rkl data directories
sudo find /var/lib/rkl/bundle -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null
sudo find /var/lib/rkl/pods -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null
sudo find /var/lib/rkl/volumes -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null

# Clean CNI bridge
sudo ip link set cni0 down 2>/dev/null
sudo brctl delbr cni0 2>/dev/null
```

**Why `find -exec rm` instead of `rm -rf /path/*`**: Over SSH, glob expansion with `*` can silently fail. `find -exec` is reliable.

**Why unmount first**: FUSE mounts prevent `rm -rf` from deleting directories. Always unmount before cleaning.

## Step 6: Start Xline

Skip if Xline containers are already running (`docker ps | grep node1`).

If restarting from a failed deployment, **stop and remove** existing Xline containers first to clear stale pod data from etcd:

```bash
docker stop node1 node2 node3 2>/dev/null
docker rm node1 node2 node3 2>/dev/null
```

Then start fresh:

```bash
docker network create --subnet=172.20.0.0/24 xline_net 2>/dev/null || true
MEMBERS="node1=172.20.0.3:2380,node2=172.20.0.4:2380,node3=172.20.0.5:2380"
for i in 1 2 3; do
  IP="172.20.0.$((i+2))"
  docker run -d --name "node$i" --net xline_net --ip "$IP" --cap-add=NET_ADMIN \
    ghcr.io/xline-kv/xline:latest xline \
    --name "node$i" --members "$MEMBERS" \
    --data-dir /usr/local/xline/data-dir --storage-engine rocksdb \
    --initial-cluster-state new \
    --client-listen-urls "http://$IP:2379" --peer-listen-urls "http://$IP:2380" \
    --client-advertise-urls "http://$IP:2379" --peer-advertise-urls "http://$IP:2380"
done
sleep 5
```

Verify: `docker logs node1 2>&1 | tail -5` should show leader election.

**Note**: Alternatively, you can build Xline from the repo at `$PROJECT_DIR/Xline/Xline/` with `cargo build -p xline --release` and run it natively. The Docker approach is simpler for single-machine testing.

## Step 7: Start RKS

Skip if already running (`ps aux | grep 'rks.*start'`).

```bash
cat > /tmp/rks-config.yaml << 'EOF'
addr: "127.0.0.1:50051"
xline_config:
  endpoints:
    - "http://172.20.0.3:2379"
    - "http://172.20.0.4:2379"
    - "http://172.20.0.5:2379"
  prefix: "/coreos.com/network"
  subnet_lease_renew_margin: 60
network_config:
  Network: "10.1.0.0/16"
  SubnetMin: "10.1.1.0"
  SubnetMax: "10.1.254.0"
  SubnetLen: 24
tls_config:
  enable: false
  vault_folder: ""
  keep_dangerous_files: false
dns_config:
  Port: 5300
EOF

sudo nohup $BIN/rks start --config /tmp/rks-config.yaml > /tmp/rks.log 2>&1 &
sleep 5
```

Verify: `tail -5 /tmp/rks.log` should show subnet allocation and "local rks node registered".

## Step 8: Start RKL Daemon

Skip if already running (`ps aux | grep 'rkl.*daemon'`).

```bash
sudo sh -c "RKS_ADDRESS=127.0.0.1:50051 RKL_USE_LIBFUSE=1 nohup $BIN/rkl pod daemon > /tmp/rkl-daemon.log 2>&1 &"
sleep 5
```

`RKL_USE_LIBFUSE=1` enables the libfuse overlay backend for container rootfs. This causes each container to get a FUSE-based overlayfs mount at `<bundle>/merged/`, served by a dedicated `rkl mount --daemon --libfuse` child process. Without this flag, Linux native kernel overlayfs is used instead.

Verify: `tail -5 /tmp/rks.log` should show heartbeat messages from the worker node.

## Step 9: Deploy Pods

**CRITICAL**: Deploy pods ONE AT A TIME. Wait for each pod to reach `Running` status before deploying the next. Deploying multiple pods simultaneously causes the watcher to enter a "container already exists" loop that blocks all subsequent pod creation.

### Pod YAML files

**nginx-pod.yaml:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: web
      image: nginx:latest
      volumeMounts:
        - name: data-vol
          mountPath: /data
  volumes:
    - name: data-vol
      type: slayerFs
      capacity_bytes: 1073741824
```

**busybox-N.yaml** (create 3 files: busybox-1.yaml, busybox-2.yaml, busybox-3.yaml):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-N
spec:
  containers:
    - name: sh
      image: busybox:latest
      args:
        - "sleep"
        - "3600"
      volumeMounts:
        - name: data-vol
          mountPath: /data
  volumes:
    - name: data-vol
      type: slayerFs
      capacity_bytes: 1073741824
```

### Deploy sequence

Deploy each pod and wait for Running before proceeding:

```bash
# Deploy nginx first
sudo $BIN/rkl pod create /tmp/nginx-pod.yaml --cluster 127.0.0.1:50051
# Poll until Running (typically 5-15 seconds)
sudo $BIN/rkl pod list --cluster 127.0.0.1:50051  # wait for nginx: 1/1 Running

# Then busybox-1
sudo $BIN/rkl pod create /tmp/busybox-1.yaml --cluster 127.0.0.1:50051
# Poll until Running (typically 10-30 seconds for first busybox pull)
sudo $BIN/rkl pod list --cluster 127.0.0.1:50051

# Then busybox-2
sudo $BIN/rkl pod create /tmp/busybox-2.yaml --cluster 127.0.0.1:50051
sudo $BIN/rkl pod list --cluster 127.0.0.1:50051

# Then busybox-3
sudo $BIN/rkl pod create /tmp/busybox-3.yaml --cluster 127.0.0.1:50051
sudo $BIN/rkl pod list --cluster 127.0.0.1:50051
```

Expected final state:
```
NAME       READY  STATUS   RESTARTS  AGE
busybox-1  1/1    Running  0         1m
busybox-2  1/1    Running  0         30s
busybox-3  1/1    Running  0         10s
nginx      1/1    Running  0         1m
```

**Note on image pulling**: RKL automatically pulls images via rkforge when they are not cached locally. No manual `rkforge pull` step is needed. The first pull of nginx (~60MB) may take a few minutes; busybox (~2MB) is fast.

**Note on command override**: The YAML `command` field currently does NOT override the image's OCI entrypoint — the image config takes precedence. Use `args` to append arguments.

## Step 10: Verify

### 10a. Network verification (ping between busybox pods)

First, get pod IPs:
```bash
for pod in nginx busybox-1 busybox-2 busybox-3; do
  PID=$(sudo cat /run/youki/$pod/state.json | python3 -c "import sys,json; print(json.load(sys.stdin)['pid'])")
  IP=$(sudo nsenter -t $PID -n ip addr | grep "inet " | grep vethcni0 | awk '{print $2}')
  echo "$pod: $IP (PID=$PID)"
done
```

Then ping between busybox pods:
```bash
# busybox-1 -> busybox-2
sudo $BIN/rkl pod exec busybox-1 busybox-1-sh \
  -e PATH=/bin:/usr/bin -- ping -c 3 <busybox-2-ip>

# busybox-2 -> busybox-3
sudo $BIN/rkl pod exec busybox-2 busybox-2-sh \
  -e PATH=/bin:/usr/bin -- ping -c 3 <busybox-3-ip>

# busybox-3 -> busybox-1
sudo $BIN/rkl pod exec busybox-3 busybox-3-sh \
  -e PATH=/bin:/usr/bin -- ping -c 3 <busybox-1-ip>

# busybox-1 -> nginx
sudo $BIN/rkl pod exec busybox-1 busybox-1-sh \
  -e PATH=/bin:/usr/bin -- ping -c 3 <nginx-ip>
```

Expected: `0% packet loss` for all pings.

### 10b. Volume verification (SlayerFS read/write)

**Write from containers:**
```bash
# Write from each busybox
sudo $BIN/rkl pod exec busybox-1 busybox-1-sh \
  -e PATH=/bin:/usr/bin -- /bin/sh -c "echo hello-from-busybox-1 > /data/test.txt && cat /data/test.txt"

sudo $BIN/rkl pod exec busybox-2 busybox-2-sh \
  -e PATH=/bin:/usr/bin -- /bin/sh -c "echo hello-from-busybox-2 > /data/test.txt && cat /data/test.txt"

sudo $BIN/rkl pod exec busybox-3 busybox-3-sh \
  -e PATH=/bin:/usr/bin -- /bin/sh -c "echo hello-from-busybox-3 > /data/test.txt && cat /data/test.txt"
```

**Verify from host (bidirectional):**
```bash
# Check FUSE mounts exist
mount | grep "slayerfs.*pods"

# Find volume paths by matching content
for uid_dir in /var/lib/rkl/pods/*/volumes/data-vol; do
  CONTENT=$(sudo cat "$uid_dir/test.txt" 2>/dev/null)
  echo "$uid_dir: $CONTENT"
done

# Write from host -> read from container
BUSYBOX1_VOL=$(for d in /var/lib/rkl/pods/*/volumes/data-vol; do
  sudo grep -l "busybox-1" "$d/test.txt" 2>/dev/null && break
done | head -1 | sed 's|/test.txt||')
sudo sh -c "echo from-host > $BUSYBOX1_VOL/host.txt"
sudo $BIN/rkl pod exec busybox-1 busybox-1-sh \
  -e PATH=/bin:/usr/bin -- cat /data/host.txt
# Expected: "from-host"
```

### Container naming convention

The `rkl pod exec` command uses the format:
```
sudo $BIN/rkl pod exec <pod-name> <pod-name>-<container-name> -e PATH=... -- <command>
```

For the pods above:
- nginx pod: `sudo $BIN/rkl pod exec nginx nginx-web ...`
- busybox-1 pod: `sudo $BIN/rkl pod exec busybox-1 busybox-1-sh ...`
- busybox-2 pod: `sudo $BIN/rkl pod exec busybox-2 busybox-2-sh ...`
- busybox-3 pod: `sudo $BIN/rkl pod exec busybox-3 busybox-3-sh ...`

## Cleanup

```bash
# Delete all pods (one at a time)
for pod in nginx busybox-1 busybox-2 busybox-3; do
  sudo $BIN/rkl pod delete $pod --cluster 127.0.0.1:50051
done

# Stop services
sudo killall rkl rks 2>/dev/null

# Clean runtime state (required for clean re-deployment)
mount | grep -E "fuse.*rkl|slayerfs|overlay.*rkl" | awk '{print $3}' | while read mnt; do
  sudo umount -l "$mnt" 2>/dev/null
done
sudo find /run/youki -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null
sudo find /var/lib/rkl/bundle -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null
sudo find /var/lib/rkl/pods -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null
sudo find /var/lib/rkl/volumes -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null

# Clean CNI bridge (needed before restarting)
sudo ip link set cni0 down 2>/dev/null
sudo brctl delbr cni0 2>/dev/null

# Stop Xline (optional)
docker stop node1 node2 node3 2>/dev/null
docker rm node1 node2 node3 2>/dev/null
docker network rm xline_net 2>/dev/null
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `container already exists` loop | Stale containers in `/run/youki/` | Run Step 5 cleanup, restart Xline, RKS, RKL |
| Pod stuck in Pending forever | Watcher stuck after container conflict | Same as above: full cleanup + restart |
| `Failed to pull manifest: manifest unknown` | Image tag not on remote registry | Use `busybox:latest` (not `1.36`). Check `curl http://47.79.87.161:8968/v2/_catalog` |
| `Failed to pull manifest` (HTTPS error) | Registry HTTP/HTTPS mismatch | Check rkforge.toml insecure-registries |
| nginx-web container shows Stopped | Known issue: nginx process exits immediately | Pod still shows Running; nginx issue is cosmetic for now |
| Volume provisioning creates many volumes | Known watcher loop bug | Not harmful; extra volumes are cleaned up on pod delete |
| `rm -rf /var/lib/rkl/*` doesn't work | Active FUSE mounts prevent deletion | Unmount all FUSE mounts first (Step 5) |
| Pod stuck in Pending (no errors) | RKL daemon not connected | Check `tail /tmp/rks.log` for heartbeats |
| rkforge pull very slow / timeout | Registry bandwidth limited (~60MB takes 5-10 min) | Wait, or use a local registry |
| No FUSE mount on rootfs `merged/` | RKL started without `RKL_USE_LIBFUSE=1` | Restart RKL daemon with `RKL_USE_LIBFUSE=1` env var |
| `rkl mount --daemon --libfuse` processes not visible | Normal for SlayerFS volumes — they run as async tasks inside the rkl daemon, not separate processes | Only rootfs overlay gets dedicated `rkl mount` child processes; SlayerFS FUSE sessions are in-process (check via `/proc/<rkl-pid>/fd \| grep /dev/fuse`) |
