# Plan — Phase 3: App Stack

Each group maps to a roadmap step. Complete groups in order; each has an atomic done check.

---

## Group 1 — Confirm app namespace (roadmap 3.1)

1. Verify the `app` namespace exists (created in Phase 2):
   `kubectl get ns app`

**Done:** `kubectl get ns app` shows `app` with `STATUS=Active`.

---

## Group 2 — Deploy Postgres (roadmap 3.2)

1. Add the Bitnami Helm repo if not present:
   `helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update bitnami`
2. Install Postgres using the ESO-managed secret from Phase 2:
   ```
   helm install postgres bitnami/postgresql \
     --namespace app \
     --set auth.existingSecret=postgres-secret \
     --set auth.secretKeys.userPasswordKey=password \
     --set auth.database=appdb \
     --set auth.username=appuser
   ```
3. Wait for the Postgres pod to be Ready

**Done:** `kubectl -n app get pods` shows the postgres pod `Running` with `1/1` ready.

---

## Group 3 — Build and deploy FastAPI backend (roadmap 3.3)

1. Write `backend/main.py` (FastAPI app reading from Postgres)
2. Write `backend/Dockerfile`
3. Build the image inside each agent VM using `multipass exec` + `nerdctl build` (containerd-native, no Docker needed)
4. Apply `backend/deployment.yaml` and `backend/service.yaml`
5. Wait for the backend pod to be Ready

**Done:** `kubectl -n app get pods` shows the backend pod `Running` with `1/1` ready.

---

## Group 4 — Smoke test backend → Postgres (roadmap 3.4)

1. Exec into the backend pod and curl `localhost:8000/api/items`

**Done:** Response is valid JSON (empty list `[]` is fine — Postgres is reachable).

---

## Group 5 — Build and deploy nginx frontend (roadmap 3.5)

1. Write `frontend/index.html` (fetches `/api/items`, renders a list)
2. Write `frontend/nginx.conf` (proxies `/api/` to backend service)
3. Write `frontend/Dockerfile`
4. Build the image inside agent VMs (same pattern as Group 3)
5. Apply `frontend/deployment.yaml` and `frontend/service.yaml`
6. Wait for the frontend pod to be Ready

**Done:** `kubectl -n app get pods` shows the frontend pod `Running` with `1/1` ready.

---

## Group 6 — Install ingress-nginx controller (roadmap 3.6)

1. Add the ingress-nginx Helm repo:
   `helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && helm repo update ingress-nginx`
2. Install the controller:
   ```
   helm install ingress-nginx ingress-nginx/ingress-nginx \
     --namespace ingress-nginx \
     --create-namespace
   ```
3. Wait for the controller pod to be Ready and capture its external IP

**Done:** `kubectl -n ingress-nginx get pods` shows controller pod `Running`; `kubectl -n ingress-nginx get svc ingress-nginx-controller` shows an external IP.

---

## Group 7 — Apply Ingress and wire /etc/hosts (roadmap 3.7)

1. Apply `app-ingress.yaml` (routes `app.local` → frontend service)
2. Get the ingress controller's external IP:
   `kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'`
3. Add to `/etc/hosts`: `<INGRESS_IP>  app.local`
4. Open `http://app.local` in the browser

**Done:** `http://app.local` loads in the browser and displays the items list from Postgres.
