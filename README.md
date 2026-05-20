# RKE2 Local Cluster — Full Stack + Vault + Istio

> A production-like Kubernetes environment on macOS using Multipass + RKE2, with HashiCorp Vault, External Secrets Operator, a full app stack, and Istio mTLS.

---

## Architecture

```
Mac (64GB RAM)
└── Multipass VMs
    ├── rke2-server   (4 CPU / 8GB)  — control plane, etcd
    ├── rke2-agent1   (2 CPU / 6GB)  — app workloads
    └── rke2-agent2   (2 CPU / 6GB)  — Vault + ESO
        │
        ├── HashiCorp Vault (dev → prod)
        ├── External Secrets Operator (ESO)
        │
        ├── Frontend  (nginx serving React/HTML)
        ├── Backend   (Python / FastAPI)
        ├── Postgres  (Bitnami Helm chart)
        │
        └── [Phase 2] Istio
            ├── mTLS PeerAuthentication (mesh-wide)
            └── AuthorizationPolicy (default deny-all)
```

---

## Prerequisites

```bash
brew install multipass
brew install kubectl
brew install helm
brew install vault          # CLI only
brew install istioctl       # Phase 2
```

---

## Phase 1 — RKE2 Cluster

### 1.1 Spin up VMs

```bash
multipass launch --name rke2-server --cpus 4 --memory 8G --disk 40G
multipass launch --name rke2-agent1 --cpus 2 --memory 6G --disk 20G
multipass launch --name rke2-agent2 --cpus 2 --memory 6G --disk 20G
```

### 1.2 Install RKE2 on server

```bash
multipass shell rke2-server

curl -sfL https://get.rke2.io | sudo sh -
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service

# Grab token and IP (needed for agents)
sudo cat /var/lib/rancher/rke2/server/node-token
ip addr show eth0 | grep 'inet ' | awk '{print $2}' | cut -d/ -f1
```

### 1.3 Join agents

Run on **each** agent VM (`rke2-agent1`, `rke2-agent2`):

```bash
multipass shell rke2-agent1

curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sudo sh -

sudo mkdir -p /etc/rancher/rke2
cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
server: https://<SERVER_IP>:9345
token: <NODE_TOKEN>
EOF

sudo systemctl enable rke2-agent.service
sudo systemctl start rke2-agent.service
```

### 1.4 Configure kubectl on Mac

```bash
multipass exec rke2-server -- sudo cat /etc/rancher/rke2/rke2.yaml > ~/.kube/rke2.yaml

SERVER_IP=$(multipass info rke2-server | grep IPv4 | awk '{print $2}')
sed -i '' "s/127.0.0.1/$SERVER_IP/" ~/.kube/rke2.yaml

export KUBECONFIG=~/.kube/rke2.yaml
kubectl get nodes
```

**Expected output:**
```
NAME          STATUS   ROLES                       AGE   VERSION
rke2-server   Ready    control-plane,etcd,master   3m    v1.35.x+rke2r1
rke2-agent1   Ready    <none>                      1m    v1.35.x+rke2r1
rke2-agent2   Ready    <none>                      1m    v1.35.x+rke2r1
```

---

## Phase 2 — HashiCorp Vault + External Secrets Operator

### 2.1 Install Vault (dev mode)

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set "server.dev.enabled=true"

kubectl -n vault get pods
```

### 2.2 Configure Vault

```bash
# Port-forward to access Vault UI/CLI
kubectl -n vault port-forward svc/vault 8200:8200 &

export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=root  # dev mode default

# Enable KV secrets engine
vault secrets enable -path=secret kv-v2

# Store Postgres credentials
vault kv put secret/postgres \
  username=appuser \
  password=supersecret \
  database=appdb
```

### 2.3 Install External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

### 2.4 Connect ESO to Vault

```yaml
# secret-store.yaml
apiVersion: external-secrets.io/v1beta1
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

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: postgres-credentials
  namespace: app
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: postgres-secret
  data:
    - secretKey: username
      remoteRef:
        key: secret/postgres
        property: username
    - secretKey: password
      remoteRef:
        key: secret/postgres
        property: password
```

```bash
kubectl apply -f secret-store.yaml
kubectl apply -f external-secret.yaml

# Verify secret was created
kubectl -n app get secret postgres-secret
```

---

## Phase 3 — App Stack

### Stack

| Component | Image | Notes |
|---|---|---|
| Frontend | `nginx:alpine` | Serves static HTML, proxies `/api` |
| Backend | `python:3.11-slim` + FastAPI | REST API, reads from Postgres |
| Postgres | Bitnami Helm chart | Credentials from Vault via ESO |

### 3.1 Deploy Postgres

```bash
kubectl create namespace app

helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgres bitnami/postgresql \
  --namespace app \
  --set auth.existingSecret=postgres-secret \
  --set auth.secretKeys.userPasswordKey=password \
  --set auth.database=appdb \
  --set auth.username=appuser
```

### 3.2 Backend (FastAPI)

```python
# main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import psycopg2, os

app = FastAPI()
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"])

def get_db():
    return psycopg2.connect(
        host=os.environ["DB_HOST"],
        database=os.environ["DB_NAME"],
        user=os.environ["DB_USER"],
        password=os.environ["DB_PASSWORD"]
    )

@app.get("/api/items")
def get_items():
    conn = get_db()
    cur = conn.cursor()
    cur.execute("SELECT id, name FROM items;")
    rows = cur.fetchall()
    conn.close()
    return [{"id": r[0], "name": r[1]} for r in rows]
```

```yaml
# backend deployment — credentials injected from ESO-managed secret
env:
  - name: DB_HOST
    value: postgres-postgresql.app.svc.cluster.local
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: postgres-secret
        key: username
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres-secret
        key: password
```

### 3.3 Frontend (nginx)

```html
<!-- index.html -->
<script>
  fetch('/api/items')
    .then(r => r.json())
    .then(items => {
      document.getElementById('list').innerHTML =
        items.map(i => `<li>${i.name}</li>`).join('');
    });
</script>
<ul id="list">Loading...</ul>
```

```nginx
# nginx.conf — proxy /api to backend service
location /api/ {
    proxy_pass http://backend.app.svc.cluster.local:8000/api/;
}
```

### 3.4 Ingress

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: app
spec:
  rules:
    - host: app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

Add to `/etc/hosts`:
```
<INGRESS_IP>  app.local
```

Open `http://app.local` in your browser.

---

## Phase 4 — Istio + Zero Trust Networking

### 4.1 Install Istio

```bash
istioctl install --set profile=default -y

# Enable sidecar injection for app namespace
kubectl label namespace app istio-injection=enabled
kubectl rollout restart deployment -n app
```

### 4.2 Enable mTLS mesh-wide

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

### 4.3 Default deny-all

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: app
spec: {}  # empty spec = deny all
```

### 4.4 Explicit allow policies

```yaml
# Allow frontend → backend
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: app
spec:
  selector:
    matchLabels:
      app: backend
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/app/sa/frontend"]
---
# Allow backend → postgres
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-backend-to-postgres
  namespace: app
spec:
  selector:
    matchLabels:
      app: postgres
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/app/sa/backend"]
```

### 4.5 Verify mTLS

```bash
# Check all services are in STRICT mTLS
istioctl x describe service frontend.app
istioctl x describe service backend.app

# View traffic in Kiali (optional)
istioctl dashboard kiali
```

---

## Resource Summary

| VM | CPU | RAM | Role |
|---|---|---|---|
| rke2-server | 4 | 8GB | Control plane, etcd |
| rke2-agent1 | 2 | 6GB | Frontend, Backend |
| rke2-agent2 | 2 | 6GB | Vault, ESO, Postgres |
| **Total** | **8** | **20GB** | — |

---

## Troubleshooting

```bash
# Check RKE2 server logs
multipass exec rke2-server -- sudo journalctl -u rke2-server -f

# Check ESO sync status
kubectl get externalsecret -n app

# Check Vault status
vault status

# Check Istio proxy status
istioctl proxy-status

# Check mTLS enforcement
istioctl authn tls-check <pod> backend.app.svc.cluster.local
```

---

## Project Phases Checklist

- [ ] Phase 1 — RKE2 cluster (3 nodes, kubectl working)
- [ ] Phase 2 — Vault + ESO (secrets syncing to K8s)
- [ ] Phase 3 — App stack (browser → frontend → backend → Postgres)
- [ ] Phase 4 — Istio mTLS + default deny-all AuthorizationPolicy
