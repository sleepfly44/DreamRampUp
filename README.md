# DreamRampUp — Local Kubernetes Learning Lab

A production-like Kubernetes environment built on macOS using Multipass + RKE2.
Covers secrets management, a three-tier app stack, and zero-trust networking — all four phases complete.

---

## Architecture

```
Mac (64 GB RAM)
└── Multipass VMs
    ├── rke2-server   (4 CPU / 8 GB)  — control plane, etcd
    ├── rke2-agent1   (2 CPU / 6 GB)  — frontend, backend
    └── rke2-agent2   (2 CPU / 6 GB)  — Vault, ESO, Postgres
        │
        ├── HashiCorp Vault (dev mode, KV v2)
        ├── External Secrets Operator
        │
        ├── Frontend  (nginx — static HTML + /api proxy)
        ├── Backend   (Python / FastAPI)
        ├── Postgres  (Bitnami Helm chart)
        │
        └── Istio
            ├── STRICT mTLS mesh-wide (PeerAuthentication)
            ├── Default deny-all AuthorizationPolicy
            └── Explicit allow: ingress→frontend, frontend→backend, backend→postgres
```

---

## Status

| Phase | Description | Status |
|---|---|---|
| 1 | RKE2 cluster — 3 nodes, kubectl working | ✅ Complete |
| 2 | Vault + ESO — secrets syncing to K8s | ✅ Complete |
| 3 | App stack — browser → frontend → backend → Postgres | ✅ Complete |
| 4 | Istio mTLS + zero-trust AuthorizationPolicies | ✅ Complete |

---

## Repository Layout

```
├── backend/                  FastAPI app, Dockerfile, K8s manifests
├── frontend/                 nginx HTML, nginx.conf, Dockerfile, K8s manifests
├── istio/                    PeerAuthentication + AuthorizationPolicy manifests
├── app-ingress.yaml          Ingress for app.local
├── docs/
│   └── self-learning-guide.md  Full journey: install steps, troubleshooting, verifications
└── specs/
    ├── mission.md            Project purpose and success criteria
    ├── tech-stack.md         Tools and why they were chosen
    ├── roadmap.md            All phases and steps (all ✅)
    ├── 2026-05-20-phase-1-rke2-cluster/
    ├── 2026-05-20-phase-2-vault-eso/
    ├── 2026-05-20-phase-3-app-stack/
    └── 2026-05-21-phase-4-istio-zero-trust/
        └── plan.md, requirements.md, validation.md  (per phase)
```

---

## Mac Prerequisites

```bash
brew install multipass kubectl helm
brew tap hashicorp/tap && brew install hashicorp/tap/vault
brew install istioctl
```

---

## Quick Verification

```bash
export KUBECONFIG=~/.kube/rke2.yaml

# Cluster
kubectl get nodes

# Secrets pipeline
kubectl get clustersecretstore vault-backend
kubectl -n app get secret postgres-secret -o jsonpath='{.data.username}' | base64 -d

# App stack
curl -s http://app.local
curl -s http://app.local/api/items

# Istio
kubectl get peerauthentication -A
kubectl get authorizationpolicy -n app
kubectl -n app get pods   # all should show 2/2
```

---

## After Mac Sleep / Restart

```bash
multipass start rke2-server rke2-agent1 rke2-agent2

# Re-forward Vault if needed
kubectl -n vault port-forward svc/vault 8200:8200 &
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=root
```

---

## Documentation

- **[Self-Learning Guide](docs/self-learning-guide.md)** — complete installation walkthrough, every troubleshooting issue encountered, and all verification commands
- **[Mission](specs/mission.md)** — what this lab is and isn't
- **[Tech Stack](specs/tech-stack.md)** — tools used and key decisions
- **[Roadmap](specs/roadmap.md)** — all phases and steps with completion status
