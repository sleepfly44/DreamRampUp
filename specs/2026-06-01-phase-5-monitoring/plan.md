# Plan — Phase 5: Prometheus + Grafana Monitoring

Each group builds on the previous. Complete in order.

---

## Group 1 — Install kube-prometheus-stack

1. Add the Prometheus community Helm repo:
   `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update`
2. Install kube-prometheus-stack into a dedicated `monitoring` namespace:
   ```
   helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
     --namespace monitoring --create-namespace \
     --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
     --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
   ```
3. Wait for all monitoring pods to be Ready

**Done:** `kubectl -n monitoring get pods` shows Prometheus, Grafana, and AlertManager all `Running`.

---

## Group 2 — Instrument FastAPI backend (/metrics)

1. Add `prometheus-fastapi-instrumentator` to `backend/requirements.txt`
2. Update `backend/main.py` to expose a `/metrics` endpoint
3. Rebuild the backend image on both agent VMs and import into RKE2 containerd
4. Restart the backend deployment
5. Apply `monitoring/backend-servicemonitor.yaml` so Prometheus scrapes the backend

**Done:** `kubectl -n monitoring get servicemonitor backend-metrics` exists; Prometheus UI shows `backend` as a scrape target with `UP` status.

---

## Group 3 — PostgreSQL exporter + scrape config

1. Deploy `prometheus-postgres-exporter` via Helm into the `app` namespace, pointing at the Postgres service and using credentials from `postgres-secret`
2. Apply `monitoring/postgres-servicemonitor.yaml` so Prometheus scrapes the exporter

**Done:** Prometheus UI shows `postgres-exporter` as a scrape target with `UP` status; `pg_up` metric returns `1`.

---

## Group 4 — Grafana dashboards

1. Port-forward Grafana to localhost: `kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80 &`
2. Log in (default: `admin` / `prom-operator`)
3. Import or build a FastAPI dashboard showing: request rate, error rate, p99 latency
4. Import or build a Postgres dashboard showing: active connections, transaction rate, cache hit ratio

**Done:** Both dashboards visible in Grafana with live data points.
