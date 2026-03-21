# rk8s Single-Machine Troubleshooting

## Common Issues

### 1. "container already exists" loop during pod creation

**Where**: Pod creation, pause container sandbox setup
**Cause**: Stale containers from a previous deployment exist in `/run/youki/`. When RKS tries to create a new pod with the same name, the pause container creation fails because the old container state is still registered.

**This is the most common issue.** It happens when:
- A previous deployment failed or was interrupted
- Pods were deleted from the control plane but containers weren't cleaned up
- Multiple pods were deployed simultaneously (see Step 9 in SKILL.md)

**Fix**: Full cleanup is required:
```bash
# Stop services
sudo killall rkl rks 2>/dev/null
sleep 2

# Unmount all FUSE mounts first (required before rm)
mount | grep -E "fuse.*rkl|slayerfs.*rkl" | awk '{print $3}' | while read mnt; do
  sudo umount -l "$mnt" 2>/dev/null
done

# Clean container runtime state
sudo find /run/youki -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null

# Clean rkl data
sudo find /var/lib/rkl/bundle -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null
sudo find /var/lib/rkl/pods -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null
sudo find /var/lib/rkl/volumes -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null

# Clean CNI bridge
sudo ip link set cni0 down 2>/dev/null
sudo brctl delbr cni0 2>/dev/null

# Restart Xline to clear stale pod data
docker stop node1 node2 node3 2>/dev/null
docker rm node1 node2 node3 2>/dev/null
# Then re-run Xline start from Step 6

# Restart RKS and RKL
# Re-run Steps 7 and 8
```

**Important**: Use `find -exec rm` instead of `rm -rf /path/*`. Over SSH, glob expansion with `*` can silently fail.

**Important**: Unmount FUSE mounts BEFORE trying to remove directories. Active FUSE mounts prevent `rm -rf` from working.

### 2. Pod stuck in Pending (no error in logs)

**Possible causes**:
- RKL daemon not connected to RKS
- Volume provisioning timeout

**Diagnosis**:
```bash
# Check RKS logs for heartbeats
tail -5 /tmp/rks.log | grep Heartbeat
# Should see: "Heartbeat from node '<hostname>' (ready)" every ~5 seconds

# Check for scheduling events
grep -E "assigned|create|error|ERROR" /tmp/rks.log | tail -20
```

### 3. "Failed to pull manifest: manifest unknown"

**Cause**: The requested image tag doesn't exist on the remote registry.

**Fix**: Check available tags:
```bash
curl -s http://47.79.87.161:8968/v2/_catalog
curl -s http://47.79.87.161:8968/v2/library/busybox/tags/list
```

Available images on the default registry:
- `nginx:latest`
- `busybox:latest` (NOT `busybox:1.36`)
- `pause:3.9`

### 4. "Failed to pull manifest" (connection error)

**Cause**: rkforge is trying HTTPS on an HTTP-only registry.

**Fix**: Ensure `~/.config/rk8s/rkforge.toml` has:
```toml
[registry]
insecure-registries = ["47.79.87.161:8968"]
```

Remember: with `sudo`, rkforge reads `$SUDO_USER`'s config, not root's.

### 5. nginx-web container shows Stopped

**Cause**: Known issue. The nginx entrypoint script (`/docker-entrypoint.sh`) starts but the nginx process exits immediately in the rk8s container environment. The pod still shows `1/1 Running` because the pause sandbox is alive.

**Workaround**: This is cosmetic. The nginx pod's network namespace and volume still work. For testing, use busybox pods which run `sleep 3600` reliably.

### 6. Volume provisioning creates many duplicate volumes

**Cause**: Known watcher loop bug. Each time the watcher re-processes a pod, it provisions a new volume. This creates many volume entries in Xline and many FUSE mounts.

**Impact**: Not harmful to pod operation. Extra volumes are cleaned up when the pod is deleted (though manual FUSE unmount may be needed).

### 7. `rm -rf` doesn't actually delete directories

**Cause**: Active FUSE mounts inside the directory tree prevent deletion.

**Fix**: Always unmount first:
```bash
mount | grep -E "fuse.*rkl|slayerfs.*rkl" | awk '{print $3}' | while read mnt; do
  sudo umount -l "$mnt" 2>/dev/null
done
```

### 8. Network issues after restart

**Fix**: Clean up the old CNI bridge before restarting services:
```bash
sudo ip link set cni0 down 2>/dev/null
sudo brctl delbr cni0 2>/dev/null
```

## Full Reset Procedure

When things are badly broken, do a full reset:

```bash
# 1. Kill services
sudo killall rkl rks 2>/dev/null
sleep 2

# 2. Unmount all FUSE mounts
mount | grep -E "fuse.*rkl|slayerfs.*rkl" | awk '{print $3}' | while read mnt; do
  sudo umount -l "$mnt" 2>/dev/null
done

# 3. Clean all runtime state
sudo find /run/youki -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null
sudo find /var/lib/rkl/bundle -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null
sudo find /var/lib/rkl/pods -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null
sudo find /var/lib/rkl/volumes -mindepth 1 -maxdepth 1 -exec rm -rf {} + 2>/dev/null

# 4. Clean CNI
sudo ip link set cni0 down 2>/dev/null
sudo brctl delbr cni0 2>/dev/null

# 5. Restart Xline (clears stale pod data)
docker stop node1 node2 node3 2>/dev/null
docker rm node1 node2 node3 2>/dev/null
# Re-run Step 6

# 6. Restart RKS and RKL
# Re-run Steps 7 and 8

# 7. Deploy pods one at a time
# Re-run Step 9
```
