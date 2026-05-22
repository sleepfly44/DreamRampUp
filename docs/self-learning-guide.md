# Self-Learning Guide: Production-like Kubernetes on macOS

A complete record of building a local RKE2 cluster with Vault, ESO, a three-tier app stack, and Istio zero-trust networking — including every installation step, troubleshooting encounter, and verification command run along the way.

---

## What Was Built

```
Mac (64 GB RAM)
└── Multipass VMs
    ├── rke2-server   (4 CPU / 8 GB)  — control plane, etcd
    ├── rke2-agent1   (2 CPU / 6 GB)  — app workloads
    └── rke2-agent2   (2 CPU / 6 GB)  — Vault, ESO, Postgres
        │
        ├── HashiCorp Vault (dev mode, KV v2)
        ├── External Secrets Operator
        │
        ├── Frontend  (nginx, static HTML)
        ├── Backend   (Python / FastAPI)
        ├── Postgres  (Bitnami Helm chart)
        │
        └── Istio
            ├── STRICT mTLS mesh-wide (PeerAuthentication)
            └── AuthorizationPolicy (default deny-all + explicit allows)
```

---

## Mac Prerequisites

```bash
brew install multipass        # VM hypervisor
brew install kubectl          # cluster interaction
brew install helm             # chart installs
brew tap hashicorp/tap && brew install hashicorp/tap/vault  # Vault CLI
brew install istioctl         # Istio install + inspection
brew install gh               # GitHub CLI (optional, for PR workflow)
```

> **Note:** `vault` was removed from Homebrew core. Use the HashiCorp tap: `brew tap hashicorp/tap`.

---

## Phase 1 — RKE2 Cluster

### Goal
Three-node Kubernetes cluster running locally via Multipass VMs, fully reachable from Mac with `kubectl`.

### Installation

**1. Launch VMs**

```bash
multipass launch --name rke2-server --cpus 4 --memory 8G --disk 40G
multipass launch --name rke2-agent1 --cpus 2 --memory 6G --disk 20G
multipass launch --name rke2-agent2 --cpus 2 --memory 6G --disk 20G
```

**2. Install RKE2 on the server**

```bash
multipass exec rke2-server -- bash -c "
  curl -sfL https://get.rke2.io | sudo sh - &&
  sudo systemctl enable rke2-server.service &&
  sudo systemctl start rke2-server.service
"
```

Wait ~60 s for the control plane to initialise.

**3. Capture join credentials**

```bash
NODE_TOKEN=$(multipass exec rke2-server -- sudo cat /var/lib/rancher/rke2/server/node-token)
SERVER_IP=$(multipass info rke2-server | grep IPv4 | awk '{print $2}')
```

**4. Join agent nodes** (repeat for `rke2-agent1` and `rke2-agent2`)

```bash
multipass exec rke2-agent1 -- bash -c "
  curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE='agent' sudo sh - &&
  sudo mkdir -p /etc/rancher/rke2 &&
  cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
server: https://${SERVER_IP}:9345
token: ${NODE_TOKEN}
EOF
  sudo systemctl enable rke2-agent.service &&
  sudo systemctl start rke2-agent.service
"
```

**5. Configure kubectl on Mac**

```bash
multipass exec rke2-server -- sudo cat /etc/rancher/rke2/rke2.yaml > ~/.kube/rke2.yaml
SERVER_IP=$(multipass info rke2-server | grep IPv4 | awk '{print $2}')
sed -i '' "s/127.0.0.1/$SERVER_IP/" ~/.kube/rke2.yaml
export KUBECONFIG=~/.kube/rke2.yaml
# Make permanent:
echo 'export KUBECONFIG=~/.kube/rke2.yaml' >> ~/.zshrc
```

### Verification

```bash
multipass list                          # V1: all three VMs Running
multipass exec rke2-server -- sudo systemctl is-active rke2-server  # V2: active
kubectl get nodes                       # V3: all three Ready
kubectl get nodes -o wide | grep rke2-server   # V4: roles = control-plane,etcd
kubectl get nodes -o wide | grep rke2-agent    # V5: agents Ready, no roles
kubectl cluster-info                    # V6: live control plane URL
```

### Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Agent node stuck `NotReady` | Wrong IP or token in `config.yaml` | Re-check `/etc/rancher/rke2/config.yaml` on the agent; token must match exactly |
| `kubectl` connection refused | kubeconfig still points to `127.0.0.1` | Re-run the `sed` patch step |
| `kubectl get nodes` not found in new terminal | `KUBECONFIG` not exported | Add `export KUBECONFIG=~/.kube/rke2.yaml` to `~/.zshrc` |

---

## Phase 2 — HashiCorp Vault + External Secrets Operator

### Goal
Postgres credentials live in Vault. External Secrets Operator syncs them into a native Kubernetes Secret. No secrets ever appear in manifests.

### Installation

**1. Install Vault (dev mode)**

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com && helm repo update
helm install vault hashicorp/vault \
  --namespace vault --create-namespace \
  --set "server.dev.enabled=true"
kubectl -n vault wait --for=condition=Ready pod -l app.kubernetes.io/name=vault --timeout=120s
```

**2. Enable KV v2 and write credentials**

```bash
kubectl -n vault port-forward svc/vault 8200:8200 &
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=root

# Dev mode pre-mounts secret/ — enabling again returns "path already in use", which is fine
vault secrets list   # confirm secret/ is present

vault kv put secret/postgres \
  username=appuser \
  password=supersecret \
  database=appdb

vault kv get secret/postgres   # confirm all three fields
```

**3. Install External Secrets Operator**

```bash
helm repo add external-secrets https://charts.external-secrets.io && helm repo update
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace
kubectl -n external-secrets wait --for=condition=Ready pod \
  -l app.kubernetes.io/name=external-secrets --timeout=120s
```

**4. Create ClusterSecretStore**

```bash
kubectl -n external-secrets create secret generic vault-token --from-literal=token=root
kubectl apply -f specs/2026-05-20-phase-2-vault-eso/secret-store.yaml
```

`secret-store.yaml`:
```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        tokenSecretRef:
          name: vault-token
          namespace: external-secrets
          key: token
```

**5. Create ExternalSecret**

```bash
kubectl create namespace app
kubectl apply -f specs/2026-05-20-phase-2-vault-eso/external-secret.yaml
```

### Verification

```bash
kubectl -n vault get pods                        # V1: vault-0 1/1 Running
vault secrets list                               # V2: secret/ type=kv
vault kv get secret/postgres                     # V3: username/password/database present
kubectl -n external-secrets get pods             # V4: ESO controller 1/1 Running
kubectl get clustersecretstore vault-backend     # V5: READY=True
kubectl -n app get secret postgres-secret        # V6: Opaque, 3 keys
kubectl -n app get secret postgres-secret \
  -o jsonpath='{.data.username}' | base64 -d     # V7: prints "appuser"
```

### Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ClusterSecretStore` apply fails — `no matches for kind` | API version mismatch | Use `apiVersion: external-secrets.io/v1` (not `v1beta1`) with ESO 0.10+ |
| `vault secrets enable` returns "path already in use" | Dev mode pre-mounts `secret/` | Not an error — `secret/` is already KV v2, skip and continue |
| `ClusterSecretStore` READY=False | `vault-token` secret missing or wrong namespace | Secret must be in `external-secrets` namespace with key `token=root` |
| `ExternalSecret` stuck `SecretSyncedError` | Wrong Vault path or field name | `kubectl describe externalsecret -n app` shows exact error; check `remoteRef.key` and `property` |

---

## Phase 3 — App Stack

### Goal
Browser → nginx ingress → frontend (nginx) → backend (FastAPI) → Postgres. Credentials never in manifests.

### Installation

**1. Install local-path storage provisioner** *(RKE2 does not ship one by default)*

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.31/deploy/local-path-storage.yaml
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**2. Deploy Postgres via Bitnami chart**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update
helm install postgres bitnami/postgresql \
  --namespace app \
  --set auth.existingSecret=postgres-secret \
  --set auth.secretKeys.userPasswordKey=password \
  --set auth.database=appdb \
  --set auth.username=appuser
```

**3. Build and deploy backend / frontend images**

RKE2 uses its own containerd at `/run/k3s/containerd/containerd.sock` — separate from Docker's containerd at `/run/containerd/containerd.sock`. Images must be imported into the correct socket:

```bash
# Build on each agent VM (repeat for rke2-agent2)
multipass exec rke2-agent1 -- bash -c "
  sudo docker build -t backend:latest /tmp/ &&
  sudo docker save backend:latest | \
  sudo /var/lib/rancher/rke2/bin/ctr \
    --address /run/k3s/containerd/containerd.sock \
    -n k8s.io images import -
"
```

**4. Seed the database**

```bash
kubectl -n app exec postgres-postgresql-0 -- \
  env PGPASSWORD=supersecret psql -U appuser -d appdb \
  -c "CREATE TABLE IF NOT EXISTS items (id SERIAL PRIMARY KEY, name TEXT NOT NULL);
      INSERT INTO items (name) VALUES ('hello'), ('world');"
```

**5. Apply manifests**

```bash
kubectl apply -f backend/deployment.yaml
kubectl apply -f frontend/deployment.yaml
kubectl apply -f app-ingress.yaml
```

**6. Wire `/etc/hosts`**

```bash
# RKE2's ingress-nginx runs as a DaemonSet on host network — use any agent IP
echo "192.168.252.3  app.local" | sudo tee -a /etc/hosts
```

### Verification

```bash
kubectl get ns app                               # V1: Active
kubectl -n app get pods -l app.kubernetes.io/name=postgresql  # V2: 1/1 Running
kubectl -n app get pods -l app=backend           # V3: 1/1 Running
kubectl -n app exec deploy/backend -- python3 -c \
  "import urllib.request; print(urllib.request.urlopen('http://localhost:8000/api/items').read().decode())"
                                                 # V4: JSON array
kubectl -n app get pods -l app=frontend          # V5: 1/1 Running
kubectl -n kube-system get pods | grep rke2-ingress-nginx-controller  # V6: Running on all nodes
kubectl -n app get ingress                        # V7: ADDRESS shows node IPs
curl -s http://app.local                         # V8: HTML page returned
```

### Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Postgres pod stuck `Pending` — "unbound PVCs" | No default StorageClass | Install `rancher/local-path-provisioner` and patch it as default |
| `ErrImageNeverPull` despite image being present | Image imported into Docker's containerd, not RKE2's | Use `--address /run/k3s/containerd/containerd.sock` when running `ctr images import` |
| `dpkg` lock error during `apt install docker.io` | `unattended-upgrades` process held the lock | Run `sudo dpkg --configure -a` then retry |
| VM clock ahead of apt repo release date | VM clock drifted | Run `sudo chronyc makestep` on each VM |
| `curl` not found in backend pod | `python:3.11-slim` has no curl | Use `python3 -c "import urllib.request; ..."` instead |
| `http://app.local` connection refused | `/etc/hosts` entry missing | Must add with `sudo tee -a /etc/hosts` — the Bash sandbox cannot use `sudo` |
| ingress-nginx not in `ingress-nginx` namespace | RKE2 ships ingress-nginx in `kube-system` | Check `kubectl -n kube-system get pods | grep ingress` |

---

## Phase 4 — Istio + Zero-Trust Networking

### Goal
All in-mesh traffic encrypted with STRICT mTLS. No service can reach another without an explicit AuthorizationPolicy keyed to its SPIFFE identity.

### Installation

**1. Install Istio**

```bash
brew install istioctl
istioctl install --set profile=default -y
kubectl -n istio-system wait --for=condition=Ready pod -l app=istiod --timeout=120s
```

**2. Enable sidecar injection and restart workloads**

```bash
kubectl label namespace app istio-injection=enabled
kubectl rollout restart deployment -n app
kubectl rollout restart statefulset -n app
# Wait for all pods to show 2/2
kubectl -n app get pods
```

**3. Create dedicated service accounts** *(required for SPIFFE identity-based policies)*

```bash
kubectl -n app create serviceaccount frontend
kubectl -n app create serviceaccount backend
# Add serviceAccountName: frontend / backend to each Deployment spec
kubectl apply -f frontend/deployment.yaml
kubectl apply -f backend/deployment.yaml
```

**4. Apply mesh-wide STRICT mTLS**

```bash
kubectl apply -f istio/peer-authentication.yaml
```

```yaml
# peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

**5. Apply PERMISSIVE exception for the frontend** *(ingress-nginx has no sidecar)*

```bash
kubectl apply -f istio/peer-authentication-frontend.yaml
```

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: frontend-permissive
  namespace: app
spec:
  selector:
    matchLabels:
      app: frontend
  mtls:
    mode: PERMISSIVE
```

**6. Apply deny-all, then explicit allow policies** *(order matters)*

```bash
kubectl apply -f istio/deny-all.yaml                      # traffic breaks here
kubectl apply -f istio/allow-ingress-to-frontend.yaml     # ingress → frontend
kubectl apply -f istio/allow-frontend-to-backend.yaml     # frontend → backend (SPIFFE)
kubectl apply -f istio/allow-backend-to-postgres.yaml     # backend → postgres (SPIFFE)
```

### Verification

```bash
kubectl -n istio-system get pods                 # V1: istiod + ingressgateway Running
kubectl -n app get pods                          # V2: all pods 2/2 (app + envoy)
kubectl get peerauthentication -A                # V3: istio-system/default = STRICT
# V4 & V7: unauthorized pod must be denied
kubectl -n app run test --image=curlimages/curl --restart=Never --rm -i -- \
  curl -s --max-time 5 http://backend:8000/api/items
# → RBAC: access denied
curl -s http://app.local                         # V5: HTML returned
curl -s http://app.local/api/items               # V6: JSON from Postgres
```

### Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `502 Bad Gateway` after deny-all | ingress-nginx has no sidecar → STRICT mTLS blocks plaintext at transport layer, before AuthorizationPolicy runs | Add a workload-level `PERMISSIVE` PeerAuthentication for the frontend |
| All pods using same service account | Default SA used for all workloads → can't distinguish identities | Create dedicated SAs (`frontend`, `backend`); set `serviceAccountName` in each Deployment |
| `http://app.local` 502 even after allow-ingress policy | STRICT mTLS rejects connection before policy is evaluated | Frontend needs PERMISSIVE mode; the AuthorizationPolicy alone is not enough |
| Allow policy not taking effect | Policy propagation delay | Wait 5–10 s after `kubectl apply` before testing |
| V7 returns 200 instead of denied | Allow policy principal too broad | Ensure `principals` name specific service accounts, not `*` or `default` |

---

## Key Lessons Learned

### RKE2 has two containerd instances after installing Docker
Installing `docker.io` adds Docker's own containerd at `/run/containerd/containerd.sock`. RKE2's containerd runs at `/run/k3s/containerd/containerd.sock`. Images must be imported into the RKE2 socket explicitly:
```bash
sudo /var/lib/rancher/rke2/bin/ctr \
  --address /run/k3s/containerd/containerd.sock \
  -n k8s.io images import -
```

### Istio STRICT mTLS blocks non-mesh callers at the transport layer
AuthorizationPolicy evaluates identity — but if the caller has no sidecar (like ingress-nginx), the connection is rejected before policy runs. The fix is a workload-level PERMISSIVE override for the edge service only.

### Service account identity is the foundation of zero-trust
Istio AuthorizationPolicies use SPIFFE URIs (`cluster.local/ns/<ns>/sa/<sa>`). If multiple workloads share the `default` service account, you cannot distinguish them — any allow policy for one allows all. Always create dedicated service accounts per workload.

### ESO's API version changed
External Secrets Operator 0.10+ uses `apiVersion: external-secrets.io/v1`. Manifests written for older versions using `v1beta1` fail silently with "no matches for kind".

### RKE2 does not ship a default StorageClass
Unlike k3s, RKE2 does not include local-path provisioner. PVCs stay `Pending` until you install it manually:
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.31/deploy/local-path-storage.yaml
```

### Vault's `secret/` path is pre-mounted in dev mode
Dev mode automatically mounts a KV v2 engine at `secret/`. Running `vault secrets enable -path=secret kv-v2` returns an error ("path already in use") — this is not a failure; just skip it.

### VM clocks drift in Multipass
Ubuntu VMs can fall behind real time, causing `apt-get update` to fail with "Release file is not valid yet". Fix with:
```bash
multipass exec <vm-name> -- sudo chronyc makestep
```

---

## Quick Reference — Daily Commands

```bash
# Check cluster
kubectl get nodes
kubectl -n app get pods

# Check secrets pipeline
kubectl get clustersecretstore vault-backend
kubectl -n app get secret postgres-secret -o jsonpath='{.data.username}' | base64 -d

# Check Istio
kubectl get peerauthentication -A
kubectl get authorizationpolicy -n app

# Test full stack
curl -s http://app.local
curl -s http://app.local/api/items

# Restart VMs after Mac sleep
multipass start rke2-server rke2-agent1 rke2-agent2
# Then re-forward Vault if needed:
kubectl -n vault port-forward svc/vault 8200:8200 &
```
