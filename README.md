# rk8s-skills

[Claude Code](https://claude.ai/claude-code) skills for the [rk8s](https://github.com/r2cn-dev/rk8s) container orchestration system.

## Skills

| Skill | Description |
|-------|-------------|
| [rk8s-single](./rk8s-single/) | Deploy a single-node rk8s cluster with SlayerFS volume support |

## Installation

```bash
# One-line install (all supported agents: Claude Code, Codex, Cursor, Copilot, Cline, etc.)
npx skills add Ricky-Daxia/rk8s-skills -g -y
```

Other ways to install:

```bash
# Full GitHub URL
npx skills add https://github.com/Ricky-Daxia/rk8s-skills -g -y

# Only install to Claude Code
npx skills add Ricky-Daxia/rk8s-skills -g -y -a claude

# Install a specific skill
npx skills add Ricky-Daxia/rk8s-skills -g -y -s rk8s-single
```

## What is rk8s?

rk8s is a Kubernetes-like container orchestration system built in Rust, featuring:

- **RKL** — Worker node runtime + CLI (kubelet + kubectl)
- **RKS** — Control plane (API server + scheduler + controllers)
- **RKForge** — OCI image management (pull/push/build)
- **SlayerFS** — FUSE-based distributed storage (CSI backend)
- **Xline** — etcd-compatible distributed KV store

## License

MIT
