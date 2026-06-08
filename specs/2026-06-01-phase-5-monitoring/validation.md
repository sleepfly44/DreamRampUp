# Validation — Phase 5: Prometheus + Grafana Monitoring

Run each check after completing all plan groups.
All 8 must pass before Phase 5 is considered done and this branch can be merged.

---

## Checks

| # | Command | Expected output |
|---|---|---|
| V1 | `kubectl -n monitoring get pods` | Prometheus, Grafana, AlertManager, kube-state-metrics, node-exporter all `Running` |
| V2 | `kubectl -n monitoring get servicemonitor` | `backend-metrics` and `postgres-exporter` both listed |
| V3 | Port-forward Prometheus (`kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090`) then open `http://localhost:9090/targets` | `backend` and `postgres-exporter` targets show `State=UP` |
| V4 | In Prometheus UI: query `http_requests_total` | Returns time-series with `handler`, `method`, `status_code` labels from the backend |
| V5 | In Prometheus UI: query `pg_up` | Returns `1` |
| V6 | In Prometheus UI: query `pg_stat_database_tup_fetched` | Returns rows-fetched metrics per database |
| V7 | Port-forward Grafana (`kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80`) then open `http://localhost:3000` | Login succeeds; FastAPI dashboard visible with live request rate data |
| V8 | Grafana — open Postgres dashboard | Active connections and transaction rate panels show live data |

---

## Pass bar

V3 (`UP` targets), V4 (backend metrics flowing), V5+V6 (Postgres metrics flowing), and V7+V8 (dashboards with live data) are the critical checks. All 8 must pass before merging.

---

## Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Prometheus pods `Pending` | No PVC bound — StorageClass missing or not default | Verify `kubectl get storageclass` shows `local-path (default)` |
| Backend target `DOWN` in Prometheus | Istio mTLS blocking Prometheus scrape | Exclude metrics port from Istio interception via pod annotation `traffic.sidecar.istio.io/excludeInboundPorts` |
| `http_requests_total` metric not found | `prometheus-fastapi-instrumentator` not installed or `/metrics` not exposed | Check `curl http://localhost:8000/metrics` from inside the backend pod |
| `pg_up` returns `0` | Postgres exporter can't connect to DB | Check exporter pod logs; verify `DATA_SOURCE_NAME` env var uses correct credentials and service hostname |
| ServiceMonitor not picked up by Prometheus | Label selector mismatch | Ensure ServiceMonitor labels match Prometheus's `serviceMonitorSelector`; check `kubectl -n monitoring get prometheus -o yaml` |
| Grafana shows "No data" | Datasource not configured or dashboard query wrong | Verify Prometheus datasource URL in Grafana settings; check query matches actual metric names |
