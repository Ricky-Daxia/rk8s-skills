# rk8s-single

Deploy a complete rk8s cluster on a single machine with 4 pods and SlayerFS persistent volumes.

## What it does

This skill guides Claude through the full deployment workflow:

1. **Asks** whether the target is local or a remote host via SSH
2. **Checks** if binaries are already compiled (skips the 15-20 min build if so)
3. **Configures** CNI networking and rkforge image registry
4. **Cleans** any stale state from previous deployments (critical for reliability)
5. **Starts** Xline (3-node etcd cluster) -> RKS (control plane) -> RKL daemon (worker)
6. **Deploys** 4 pods one at a time: 1 nginx + 3 busybox, each with a SlayerFS volume
7. **Verifies** by pinging between busybox pods and testing volume read/write

## Acceptance criteria

- All 4 pods reach `1/1 Running` status
- busybox pods can ping each other (validates CNI networking)
- SlayerFS volumes support read/write from containers and host (validates CSI)

## Trigger phrases

- "Deploy rk8s on the VM"
- "Start containers with SlayerFS volumes"
- "Run nginx and busybox on slayerfs2"
- "Set up single-node rk8s cluster"

## Structure

```
rk8s-single/
├── SKILL.md                    # Main skill (10-step deployment guide)
└── references/
    ├── fix-pause-layer.md      # Legacy: pause image fix (no longer needed)
    └── troubleshooting.md      # Common issues + full reset procedure
```

## Key learnings

- **Image auto-pull**: RKL automatically pulls images via rkforge — no manual `rkforge pull` step needed
- **Deploy one at a time**: Deploying multiple pods simultaneously causes "container already exists" loops
- **Clean state matters**: `/run/youki/` must be cleaned before re-deploying. FUSE mounts must be unmounted before `rm -rf`
- **Use `busybox:latest`**: `busybox:1.36` is NOT on the default registry
- **Use `find -exec rm`**: `rm -rf /path/*` silently fails over SSH due to glob expansion issues
