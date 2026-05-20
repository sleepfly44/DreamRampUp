# Plan — Phase 2: HashiCorp Vault + External Secrets Operator

Each group maps to a roadmap step. Complete groups in order; each has an atomic done check.

---

## Group 1 — Install Vault in dev mode (roadmap 2.1)

1. Add the HashiCorp Helm repo and update:
   `helm repo add hashicorp https://helm.releases.hashicorp.com && helm repo update`
2. Install Vault in dev mode into the `vault` namespace:
   ```
   helm install vault hashicorp/vault \
     --namespace vault \
     --create-namespace \
     --set "server.dev.enabled=true"
   ```
3. Wait for the Vault pod to reach Running state

**Done:** `kubectl -n vault get pods` shows the vault pod `Running` with `1/1` ready.

---

## Group 2 — Enable KV v2 secrets engine (roadmap 2.2)

1. Port-forward Vault to localhost:
   `kubectl -n vault port-forward svc/vault 8200:8200 &`
2. Export Vault env vars:
   `export VAULT_ADDR=http://127.0.0.1:8200 && export VAULT_TOKEN=root`
3. Enable KV v2 engine at the `secret` path:
   `vault secrets enable -path=secret kv-v2`

**Done:** `vault secrets list` shows `secret/` with type `kv`.

---

## Group 3 — Write Postgres credentials to Vault (roadmap 2.3)

1. Write credentials to `secret/postgres`:
   ```
   vault kv put secret/postgres \
     username=appuser \
     password=supersecret \
     database=appdb
   ```
2. Read back to verify

**Done:** `vault kv get secret/postgres` returns `username`, `password`, and `database` fields.

---

## Group 4 — Install External Secrets Operator (roadmap 2.4)

1. Add the ESO Helm repo and update:
   `helm repo add external-secrets https://charts.external-secrets.io && helm repo update`
2. Install ESO into its own namespace:
   ```
   helm install external-secrets external-secrets/external-secrets \
     --namespace external-secrets \
     --create-namespace
   ```
3. Wait for ESO pods to reach Running state

**Done:** `kubectl -n external-secrets get pods` shows the ESO controller pod `Running`.

---

## Group 5 — Create ClusterSecretStore (roadmap 2.5)

1. Create a Kubernetes Secret in the `external-secrets` namespace containing the Vault root token:
   `kubectl -n external-secrets create secret generic vault-token --from-literal=token=root`
2. Apply `secret-store.yaml` (ClusterSecretStore pointing at Vault's in-cluster address)
3. Verify the store reaches READY state

**Done:** `kubectl get clustersecretstore vault-backend` shows `READY=True`.

---

## Group 6 — Create ExternalSecret for Postgres credentials (roadmap 2.6)

1. Create the `app` namespace: `kubectl create namespace app`
2. Apply `external-secret.yaml` (ExternalSecret syncing `secret/postgres` into a K8s Secret named `postgres-secret` in the `app` namespace)
3. Wait for the sync interval (~1 min) or force a refresh

**Done:** `kubectl -n app get secret postgres-secret -o jsonpath='{.data}'` returns base64-encoded `username` and `password` keys.
