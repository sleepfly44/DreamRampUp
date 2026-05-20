# Roadmap

Phases mirror the README structure, decomposed into the smallest independently verifiable steps.
Each step has one clear done criterion — a command whose output confirms success.

---

## Phase 1 — RKE2 Cluster

**Goal:** Three-node cluster reachable from Mac via `kubectl`.

| Step | Task | Done when |
|---|---|---|
| 1.1 | Launch three Multipass VMs (`rke2-server`, `rke2-agent1`, `rke2-agent2`) | `multipass list` shows all three Running |
| 1.2 | Install and start RKE2 server on `rke2-server` | `sudo systemctl status rke2-server` is active |
| 1.3 | Capture node token and server IP from `rke2-server` | Values saved; no command fails |
| 1.4 | Join `rke2-agent1` to the cluster | Node appears in `kubectl get nodes` |
| 1.5 | Join `rke2-agent2` to the cluster | All three nodes show `Ready` in `kubectl get nodes` |
| 1.6 | Copy and patch kubeconfig to Mac | `kubectl get nodes` works from Mac terminal |

---

## Phase 2 — HashiCorp Vault + External Secrets Operator

**Goal:** Postgres credentials stored in Vault, synced as a Kubernetes Secret via ESO.

| Step | Task | Done when |
|---|---|---|
| 2.1 | Install Vault (dev mode) via Helm into `vault` namespace | `kubectl -n vault get pods` shows vault pod Running |
| 2.2 | Port-forward Vault and enable KV v2 secrets engine | `vault secrets list` shows `secret/` path |
| 2.3 | Write Postgres credentials to Vault at `secret/postgres` | `vault kv get secret/postgres` returns username + password |
| 2.4 | Install External Secrets Operator via Helm | `kubectl -n external-secrets get pods` shows ESO pod Running |
| 2.5 | Create `ClusterSecretStore` pointing at Vault | `kubectl get clustersecretstore vault-backend` shows READY=True |
| 2.6 | Create `ExternalSecret` for Postgres credentials | `kubectl -n app get secret postgres-secret` exists and has keys |

---

## Phase 3 — App Stack

**Goal:** Browser request reaches frontend, which proxies to backend, which queries Postgres.

| Step | Task | Done when |
|---|---|---|
| 3.1 | Create `app` namespace | `kubectl get ns app` exists |
| 3.2 | Deploy Postgres via Bitnami Helm chart using ESO-managed secret | `kubectl -n app get pods` shows postgres pod Running |
| 3.3 | Build and deploy FastAPI backend (image + Deployment + Service) | `kubectl -n app get pods` shows backend pod Running |
| 3.4 | Verify backend can reach Postgres | `curl` from inside backend pod returns JSON from `/api/items` |
| 3.5 | Build and deploy nginx frontend (image + Deployment + Service) | `kubectl -n app get pods` shows frontend pod Running |
| 3.6 | Install ingress-nginx controller | `kubectl -n ingress-nginx get pods` shows controller Running |
| 3.7 | Apply Ingress manifest and add `app.local` to `/etc/hosts` | `http://app.local` loads the items list in the browser |

---

## Phase 4 — Istio + Zero-Trust Networking

**Goal:** All in-mesh traffic is mTLS; no service can reach another without an explicit AuthorizationPolicy.

| Step | Task | Done when |
|---|---|---|
| 4.1 | Install Istio (default profile) via `istioctl` | `kubectl -n istio-system get pods` shows istiod Running |
| 4.2 | Enable sidecar injection on `app` namespace and restart deployments | Every app pod shows 2/2 containers (app + envoy proxy) |
| 4.3 | Apply mesh-wide `PeerAuthentication` (STRICT mTLS) | `istioctl x describe service backend.app` reports mTLS STRICT |
| 4.4 | Apply default deny-all `AuthorizationPolicy` in `app` namespace | All cross-service calls return 403/connection refused |
| 4.5 | Apply allow policy: frontend → backend | `http://app.local` loads successfully again |
| 4.6 | Apply allow policy: backend → postgres | `/api/items` returns data from Postgres |
| 4.7 | Verify no unintended paths are open | Direct curl from an unrelated pod to backend is denied |
