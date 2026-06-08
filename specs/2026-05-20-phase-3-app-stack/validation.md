# Validation — Phase 3: App Stack

Run each check after completing all plan groups.
All 8 must pass before Phase 3 is considered done and this branch can be merged.

---

## Checks

| # | Command | Expected output |
|---|---|---|
| V1 | `kubectl get ns app` | Namespace `app` with `STATUS=Active` |
| V2 | `kubectl -n app get pods -l app=postgres` | Postgres pod `Running` with `1/1` ready |
| V3 | `kubectl -n app get pods -l app=backend` | Backend pod `Running` with `1/1` ready |
| V4 | `kubectl -n app exec deploy/backend -- curl -s localhost:8000/api/items` | Valid JSON response (e.g. `[]` or a list of items) |
| V5 | `kubectl -n app get pods -l app=frontend` | Frontend pod `Running` with `1/1` ready |
| V6 | `kubectl -n ingress-nginx get pods` | ingress-nginx controller pod `Running` with `1/1` ready |
| V7 | `kubectl -n ingress-nginx get svc ingress-nginx-controller` | `EXTERNAL-IP` column shows an IP address (not `<pending>`) |
| V8 | `curl -s http://app.local` | Returns HTML containing the items list rendered by the frontend |

---

## Pass bar

V4 (backend → Postgres) and V8 (browser → frontend → backend → Postgres) are the two end-to-end checks. All 8 must pass before merging. Do not proceed to Phase 4 until V8 passes.

---

## Common failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| Postgres pod `CrashLoopBackOff` | `postgres-secret` missing or key name mismatch | `kubectl -n app get secret postgres-secret -o yaml`; confirm `password` key exists |
| Backend pod `CrashLoopBackOff` | Image not found on the node | Verify image was built with `-n k8s.io` namespace on the correct agent VM |
| `/api/items` returns 500 | Backend can't reach Postgres | Check `DB_HOST` env var; confirm Postgres service name and namespace |
| `EXTERNAL-IP` stuck at `<pending>` | No load balancer in the cluster | RKE2 doesn't ship a cloud load balancer — use the node IP directly or install MetalLB |
| `http://app.local` connection refused | `/etc/hosts` entry missing or wrong IP | Re-check ingress controller IP and `/etc/hosts` entry |
| `http://app.local` returns 404 | Ingress not applied or host rule mismatch | `kubectl -n app get ingress`; confirm `host: app.local` matches |
