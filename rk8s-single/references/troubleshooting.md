# rk8s Single-Machine Troubleshooting

## Common Issues

### 1. "cp command failed with exit code: Some(1)"

**Where**: Pod creation, pause container sandbox setup
**Cause**: The pause:3.9 image layer is stored as a gzip file (not unpacked to a directory) in `/var/lib/rkforge/layers/`. This happens because the default registry serves pause with Docker media types (`application/vnd.docker.image.rootfs.diff.tar.gzip`) which rkforge's media type detector doesn't recognize as tar+gzip.

**Fix**: See `fix-pause-layer.md` for the full script. Quick fix:
```bash
# Find the bad layer
LAYER_HASH=$(curl -s http://47.79.87.161:8968/v2/library/pause/manifests/3.9 \
  -H "Accept: application/vnd.oci.image.manifest.v1+json, application/vnd.docker.distribution.manifest.v2+json" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['layers'][0]['digest'].split(':')[1])")
LAYER_PATH="/var/lib/rkforge/layers/$LAYER_HASH"

# Convert file to directory
sudo mv "$LAYER_PATH" "${LAYER_PATH}.gz"
sudo mkdir -p "$LAYER_PATH"
sudo tar xzf "${LAYER_PATH}.gz" -C "$LAYER_PATH"
sudo rm "${LAYER_PATH}.gz"
```

**Prevention**: After fixing, don't clear the rkforge cache (`/var/lib/rkforge/layers/`), or you'll need to fix again.

### 2. "executable '/pause' does not have correct permissions"

**Where**: Pod creation, pause container start
**Cause**: The `pause` binary in the unpacked layer lacks execute permission.

**Fix**:
```bash
# Find the pause binary in the layer
find /var/lib/rkforge/layers/ -name "pause" -type f 2>/dev/null
# Add execute permission
sudo chmod +x /var/lib/rkforge/layers/<hash>/pause
```

### 3. Pod stuck in Pending

**Possible causes**:
- RKL daemon not connected to RKS
- Volume provisioning timeout
- Image pull failure

**Diagnosis**:
```bash
# Check RKS logs for errors
sudo tail -30 /tmp/rks.log | grep -E 'Error|error|fail|Heartbeat'

# Check if daemon is sending heartbeats
sudo tail -5 /tmp/rks.log | grep Heartbeat
# Should see: "Heartbeat from node '<hostname>' (ready)" every 5 seconds

# Check for CSI errors
sudo grep 'CSI\|volume\|Error' /tmp/rks.log | tail -10
```

### 4. "Failed to pull manifest"

**Cause**: rkforge is trying HTTPS on an HTTP-only registry.

**Fix**: Ensure `~/.config/rk8s/rkforge.toml` has the registry in `insecure-registries`:
```toml
[registry]
insecure-registries = ["47.79.87.161:8968"]
```

Remember: with `sudo`, rkforge reads `$SUDO_USER`'s config, not root's. Check `echo $SUDO_USER` and look at that user's `~/.config/rk8s/rkforge.toml`.

### 5. rkforge pull very slow or times out

**Cause**: The default registry (`47.79.87.161:8968`) has limited bandwidth. Large images like nginx (~60MB) take 5-10 minutes.

**Workaround**: Use `--url` to specify an alternative registry, or try smaller images like `busybox:1.36` (~2MB) for testing.

### 6. "container already exists" during pod creation

**Cause**: Stale containers from a previous failed attempt.

**Fix**:
```bash
# List and delete stale containers
sudo rkl container list
sudo rkl container delete <container-name>

# Also clean CNI bridge
sudo ip link set cni0 down 2>/dev/null
sudo brctl delbr cni0 2>/dev/null
```

### 7. Container status is "Stopped" but pod shows "Running"

**Cause**: The app container exited (possibly entrypoint script failure) but the pause sandbox container is still running. The pod status reflects the sandbox.

**Diagnosis**:
```bash
# Check the container's OCI config for what command ran
sudo cat /var/lib/rkl/bundle/<bundle-id>/config.json | python3 -c "
import sys,json
c=json.load(sys.stdin)
print('args:', c['process']['args'])
"
```

**Note**: The YAML `command` field does NOT override the image's OCI entrypoint — the image config takes precedence. Some images (like nginx) have complex entrypoint scripts that may fail in the rk8s container environment. Using `args: ["sleep", "3600"]` can help keep the container alive for debugging.

### 8. Network issues after Xline restart

**Fix**: Clean up the old CNI bridge before restarting services:
```bash
sudo ip link set cni0 down
sudo brctl delbr cni0
```

## Full Reset Procedure

When things are badly broken, do a full reset:

```bash
# 1. Kill services
sudo killall rkl rks 2>/dev/null

# 2. Clean CNI
sudo ip link set cni0 down 2>/dev/null
sudo brctl delbr cni0 2>/dev/null

# 3. Clean volume mounts
sudo umount /var/lib/rkl/volumes/*/globalmount 2>/dev/null
sudo umount /var/lib/rkl/pods/*/volumes/* 2>/dev/null

# 4. Clean runtime state (WARNING: destroys all containers and volumes)
sudo rm -rf /var/lib/rkl/pods/* /var/lib/rkl/volumes/* /var/lib/rkl/slayerfs/*
sudo rm -rf /var/lib/rkl/bundle/*

# 5. Optionally clear image cache (will need to re-pull + re-fix pause)
# sudo rm -rf /var/lib/rkforge/layers/* /var/lib/rkforge/metadata/*

# 6. Restart Xline (optional, only if Xline has issues)
docker stop node1 node2 node3 client 2>/dev/null
docker rm node1 node2 node3 client 2>/dev/null
# Then re-run Step 6 from SKILL.md

# 7. Restart RKS and RKL daemon
# Re-run Steps 7 and 8 from SKILL.md
```
