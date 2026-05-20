# Tech Stack

## Infrastructure

| Tool | Role |
|---|---|
| **Multipass** | Spins up lightweight Ubuntu VMs on macOS — the hypervisor layer |
| **RKE2** | Rancher's hardened Kubernetes distribution; runs on the Multipass VMs |

### VM topology

| VM | CPU | RAM | Responsibilities |
|---|---|---|---|
| `rke2-server` | 4 | 8 GB | Control plane, etcd |
| `rke2-agent1` | 2 | 6 GB | App workloads (frontend, backend) |
| `rke2-agent2` | 2 | 6 GB | Vault, ESO, Postgres |

## Secrets management

| Tool | Role |
|---|---|
| **HashiCorp Vault** | Secret store; runs in dev mode inside the cluster (KV v2 engine) |
| **External Secrets Operator (ESO)** | Syncs Vault secrets into native Kubernetes Secrets on a refresh interval |

## Application stack

| Component | Technology | Notes |
|---|---|---|
| **Frontend** | nginx (alpine) | Serves static HTML; proxies `/api` to backend |
| **Backend** | Python 3.11 + FastAPI | REST API; reads from Postgres; credentials injected via ESO |
| **Database** | PostgreSQL (Bitnami Helm chart) | Credentials sourced from Vault via ESO-managed Secret |

## Networking & security

| Tool | Role |
|---|---|
| **Istio** | Service mesh; enforces mTLS and AuthorizationPolicies |
| **ingress-nginx** | Cluster ingress controller; routes external traffic to the frontend |

## Tooling (local Mac)

| Tool | Purpose |
|---|---|
| `kubectl` | Cluster interaction |
| `helm` | Chart installs (Vault, ESO, Postgres, ingress-nginx) |
| `vault` CLI | Interacting with the Vault API |
| `istioctl` | Istio install, inspection, and mTLS verification |
