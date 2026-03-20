# rk8s-single

Deploy a complete rk8s cluster on a single machine with SlayerFS persistent volume support.

## What it does

This skill guides Claude through the full deployment workflow:

1. **Asks** whether the target is local or a remote host via SSH
2. **Checks** if binaries are already compiled (skips the 15-20 min build if so)
3. **Configures** CNI networking and rkforge image registry
4. **Pulls** container images via rkforge (nginx, busybox, etc.)
5. **Fixes** the known pause:3.9 Docker-format layer issue
6. **Starts** Xline (3-node etcd cluster) → RKS (control plane) → RKL daemon (worker)
7. **Creates** a Pod with SlayerFS FUSE volume
8. **Verifies** by exec-ing into the container and testing volume read/write

## Trigger phrases

- "Deploy rk8s on the VM"
- "Start a container with SlayerFS volume"
- "Run nginx on slayerfs2"
- "Set up single-node rk8s cluster"

## Structure

```
rk8s-single/
├── SKILL.md                    # Main skill (10-step deployment guide)
└── references/
    ├── fix-pause-layer.md      # Detailed pause image fix script
    └── troubleshooting.md      # 8 common issues + full reset procedure
```

## Known issues

- **pause:3.9 Docker format**: The default registry serves pause with Docker media types that rkforge can't auto-unpack. The skill includes a manual fix step.
- **YAML command override**: The `command` field in pod YAML doesn't override the image's OCI entrypoint — image config takes precedence.
- **Registry bandwidth**: The default registry (`47.79.87.161:8968`) is slow for large images. nginx (~60MB) takes 5-10 minutes.
