# Requirements — Phase 1: RKE2 Cluster

## Prerequisites

Local Mac tools that must be installed before starting:

```bash
brew install multipass   # VM hypervisor
brew install kubectl     # cluster interaction
brew install helm        # needed in later phases; install now
```

No cloud account, container registry, or network beyond local LAN is required.

---

## Hardware constraints

The Mac has 64 GB RAM total. Phase 1 reserves 20 GB across three VMs:

| VM | CPU | RAM | Disk |
|---|---|---|---|
| `rke2-server` | 4 | 8 GB | 40 GB |
| `rke2-agent1` | 2 | 6 GB | 20 GB |
| `rke2-agent2` | 2 | 6 GB | 20 GB |
| **Total** | **8** | **20 GB** | **80 GB** |

`rke2-agent2` is sized identically to `rke2-agent1` now but will host Vault, ESO, and Postgres in Phase 2–3, where the heavier workloads land.

---

## Key decisions

**RKE2 over k3s or kubeadm**
RKE2 ships with hardened defaults (CIS benchmark alignment, built-in etcd, no external datastore required). For a security-focused learning lab this is a better fit than k3s (which trades security defaults for minimal footprint) or kubeadm (which requires more manual assembly).

**Multipass over Docker Desktop / colima**
Multipass runs genuine Ubuntu VMs with their own IP addresses, systemd, and network interfaces — the same environment as a bare-metal or cloud node. Docker Desktop and colima run containers, not VMs, which hides the node-level concerns (network interfaces, kubelet, systemd units) that this lab is designed to expose.

---

## Out of scope for Phase 1

The following are explicitly deferred to later phases. Do not attempt to install or configure them here:

- HashiCorp Vault and External Secrets Operator (Phase 2)
- Application deployments — backend, frontend, Postgres (Phase 3)
- Istio, mTLS, AuthorizationPolicies (Phase 4)
- Ingress controller
- Any Helm chart beyond RKE2 itself
- Persistent storage classes beyond the RKE2 default local-path provisioner
- DNS or TLS for the cluster API endpoint
