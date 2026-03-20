---
name: rk8s-single
description: "Deploy rk8s container cluster on a single machine with SlayerFS volume support. Use this skill whenever the user wants to: start/deploy/run rk8s containers or pods on a machine, set up a single-node rk8s cluster (Xline + RKS + RKL), test SlayerFS volumes, run nginx/busybox or other containers via rk8s, or anything involving rkl/rks/rkforge deployment. Trigger even if the user just says 'start a container on the VM' or 'deploy rk8s' or 'run a pod with volume'."
---

# rk8s Single-Machine Cluster Deployment

This skill guides the deployment of a complete rk8s cluster on a single machine, including container creation with SlayerFS persistent volumes.

## Architecture

```
┌─ Single Machine ──────────────────────────────┐
│  Xline (3 Docker containers) ← state storage  │
│  RKS (control plane)         ← scheduler, CSI │
│  RKL daemon (worker node)    ← container mgmt │
│  rkforge                     ← image registry │
│  SlayerFS (FUSE)             ← volume backend │
└────────────────────────────────────────────────┘
```

## Step 0: Determine Target Environment

Before doing anything, ask the user:

1. **Target machine**: Is this the local machine or a remote host via SSH?
   - If remote: get the SSH alias or `user@host` and verify connectivity
   - If local: commands run directly with `sudo`
2. **Project path**: Where is the rk8s repo? (default: `~/rk8s/project` or `~/yrj/rk8s/project`)
3. **What to deploy**: Which container image? (default: `nginx:latest`, alternatives: `busybox:1.36`, or a custom image from the registry)
4. **Volume needed?**: Should the pod have a SlayerFS volume? (default: yes)

Based on the answers, set these variables for all subsequent steps:
- `SSH_CMD`: empty for local, `ssh <host>` for remote
- `PROJECT_DIR`: path to rk8s/project
- `BIN`: `$PROJECT_DIR/target/release`
- `IMAGE`: the container image to deploy
- `VOLUME_ENABLED`: whether to include SlayerFS volume

For remote hosts, prefix all commands with `$SSH_CMD`. Wrap multi-line scripts in heredocs or scripts uploaded via `scp`/`ssh`.

## Step 1: Check Prerequisites

Check these in parallel — skip any step where the prerequisite is already met:

```bash
# Check if binaries are compiled (CRITICAL: skip compilation if they exist, it takes 15-20 min)
ls $BIN/rkl $BIN/rks $BIN/rkforge 2>/dev/null && echo "Binaries exist" || echo "NEED COMPILATION"

# Check CNI config
sudo ls /etc/cni/net.d/test.conflist 2>/dev/null && echo "CNI configured" || echo "NEED CNI SETUP"

# Check rkforge config
cat ~/.config/rk8s/rkforge.toml 2>/dev/null | grep insecure || echo "NEED RKFORGE CONFIG"

# Check if services are already running
ps aux | grep -E 'rks.*start|rkl.*daemon' | grep -v grep
docker ps --format '{{.Names}}' | grep -E 'node[123]|xline'
```

Only proceed with steps that are actually needed. Never recompile if binaries exist.

## Step 2: Compile (only if needed)

```bash
cd $PROJECT_DIR && cargo build --release
```

Timeout: 20 minutes. This produces `rkl`, `rks`, `rkforge` in `target/release/`.

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

## Step 5: Pull Images via rkforge

```bash
sudo $BIN/rkforge pull $IMAGE          # e.g. nginx:latest
sudo $BIN/rkforge pull pause:3.9       # required for pod sandbox
sudo $BIN/rkforge images               # verify
```

**Warning**: The default registry bandwidth is limited. Large images (nginx ~60MB) may take 5-10 minutes. If it times out or hangs, inform the user and ask if they want to try a different image or registry.

### Fix pause:3.9 Layer Format (Required)

The `pause:3.9` image on the default registry uses Docker media types (`tar.gzip`) which rkforge cannot auto-unpack. The layers remain as gzip files instead of directories, causing `cp command failed` errors during pod creation.

Fix by manually unpacking the layer. Refer to `references/fix-pause-layer.md` for the detailed script.

Quick version:
```bash
# Find the layer hash
LAYER_HASH=$(curl -s http://47.79.87.161:8968/v2/library/pause/manifests/3.9 \
  -H "Accept: application/vnd.oci.image.manifest.v1+json, application/vnd.docker.distribution.manifest.v2+json" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['layers'][0]['digest'].split(':')[1])")

# Unpack gzip file to directory
LAYER_PATH="/var/lib/rkforge/layers/$LAYER_HASH"
sudo mv "$LAYER_PATH" "${LAYER_PATH}.gz"
sudo mkdir -p "$LAYER_PATH"
sudo tar xzf "${LAYER_PATH}.gz" -C "$LAYER_PATH"
sudo rm "${LAYER_PATH}.gz"
```

**Root cause**: `rkforge/src/pull/media.rs` checks `ends_with("tar+gzip")` (OCI standard) but Docker format uses `tar.gzip`. Images pushed in OCI format (like nginx) don't need this fix.

## Step 6: Start Xline

Skip if Xline containers are already running (`docker ps | grep node1`).

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
sudo sh -c "RKS_ADDRESS=127.0.0.1:50051 nohup $BIN/rkl pod daemon > /tmp/rkl-daemon.log 2>&1 &"
sleep 5
```

Verify: `tail -5 /tmp/rks.log` should show heartbeat messages from the worker node.

## Step 9: Create Pod

Generate the pod YAML based on user's choices:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
  labels:
    app: <app-label>
spec:
  containers:
    - name: <container-name>
      image: <IMAGE>
      args:
        - "sleep"
        - "3600"
      ports:
        - containerPort: 80
      volumeMounts:          # only if VOLUME_ENABLED
        - name: data-vol
          mountPath: /data
  volumes:                   # only if VOLUME_ENABLED
    - name: data-vol
      type: slayerFs
      capacity_bytes: 1073741824
```

**Note on command override**: The YAML `command` field currently does NOT override the image's OCI entrypoint — the image config takes precedence. Use `args` to append arguments.

```bash
sudo $BIN/rkl pod create /tmp/pod.yaml --cluster 127.0.0.1:50051
sleep 20
sudo $BIN/rkl pod list --cluster 127.0.0.1:50051
```

Expected: `1/1 Running`.

## Step 10: Verify

### Container verification
```bash
# List containers
sudo $BIN/rkl container list

# Exec into the container (format: <pod-name>-<container-name>)
sudo $BIN/rkl pod exec <pod-name> <pod-name>-<container-name> \
  -e PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  -- <command>

# Example: check nginx version
sudo $BIN/rkl pod exec nginx-vol nginx-vol-nginx \
  -e PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  -- nginx -v
```

### Volume verification (if SlayerFS enabled)
```bash
# Check FUSE mount exists
mount | grep slayerfs | grep pods

# Write from container, read from host
sudo $BIN/rkl pod exec <pod-name> <pod-name>-<container-name> \
  -e PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  -- /bin/sh -c "echo 'test data' > /data/test.txt && cat /data/test.txt"

VOL_PATH=$(mount | grep 'slayerfs.*pods.*volumes' | tail -1 | awk '{print $3}')
sudo cat "$VOL_PATH/test.txt"

# Write from host, read from container
sudo sh -c "echo 'from host' > $VOL_PATH/host.txt"
sudo $BIN/rkl pod exec <pod-name> <pod-name>-<container-name> \
  -e PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  -- cat /data/host.txt
```

## Cleanup

```bash
# Delete pod
sudo $BIN/rkl pod delete <pod-name> --cluster 127.0.0.1:50051

# Stop services (optional)
sudo killall rkl rks 2>/dev/null

# Clean CNI bridge (needed before restarting)
sudo ip link set cni0 down 2>/dev/null
sudo brctl delbr cni0 2>/dev/null

# Stop Xline (optional)
docker stop node1 node2 node3 client 2>/dev/null
docker rm node1 node2 node3 client 2>/dev/null
docker network rm xline_net 2>/dev/null
```

## Troubleshooting

Read `references/troubleshooting.md` for known issues and solutions. Key ones:

| Symptom | Cause | Fix |
|---------|-------|-----|
| `cp command failed with exit code: Some(1)` | pause layer not unpacked (Docker format) | Run Step 5 pause fix |
| `executable '/pause' does not have correct permissions` | pause binary missing +x | `chmod +x` before building image |
| Pod stuck in Pending | RKL daemon not connected | Check `tail /tmp/rks.log` for heartbeats |
| `Failed to pull manifest` | Registry HTTPS/HTTP mismatch | Check rkforge.toml insecure-registries |
| rkforge pull very slow / timeout | Registry bandwidth limited (~60MB takes 5-10 min) | Wait, or try `--url <alt-registry>` |
