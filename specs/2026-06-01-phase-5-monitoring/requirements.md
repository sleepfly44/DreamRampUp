# Requirements — Phase 5: Prometheus + Grafana Monitoring

## Prerequisites

- Phase 4 complete: all app pods running `2/2` with Istio sidecars
- `local-path` StorageClass available and set as default (installed in Phase 3) — Prometheus and Grafana need PVCs
- `KUBECONFIG` set to `~/.kube/rke2.yaml`
- Vault port-forward running if ESO needs to refresh secrets during this phase

---

## Key decisions

**kube-prometheus-stack over standalone Prometheus + Grafana**
The kube-prometheus-stack Helm chart bundles Prometheus, Grafana, AlertManager, kube-state-metrics, and node-exporter in a single install. It also ships with pre-built dashboards for Kubernetes cluster metrics and handles RBAC, ServiceMonitors, and PodMonitors automatically. Installing each component separately would require wiring them together manually — no benefit for a learning lab.

**prometheus-fastapi-instrumentator over manual metrics**
`prometheus-fastapi-instrumentator` wraps the FastAPI app and auto-instruments all routes with RED metrics (Rate, Errors, Duration) via a single two-line change. Writing manual `prometheus_client` counters/histograms would be more educational in a production context but adds unnecessary boilerplate here.

**Pull model (Prometheus scrapes) over push (Pushgateway)**
Prometheus uses a pull model: it scrapes `/metrics` endpoints on a schedule. This is the standard Kubernetes pattern and requires no changes to how the app runs — just exposing the endpoint and creating a ServiceMonitor. Pushgateway is only needed for short-lived jobs that can't be scraped.

**ServiceMonitor over static scrape config**
`ServiceMonitor` is a Prometheus Operator CRD that declaratively defines what to scrape. Editing `prometheus.yml` directly would work but is overwritten on Helm upgrades. ServiceMonitors are the Kubernetes-native way and survive upgrades.

---

## Istio considerations

Prometheus scraping through the Istio mesh requires care:

- **App metrics (`/metrics`)**: The backend pod has an Envoy sidecar. Prometheus (outside the `app` namespace, no `app` service account) cannot reach the backend's `/metrics` port through the mTLS mesh without an AuthorizationPolicy allow rule.
- **Envoy sidecar metrics**: Each Envoy proxy exposes its own metrics on port `15090`. These are scraped by Prometheus automatically via annotations if configured correctly.
- **Solution**: Add an `AuthorizationPolicy` allowing Prometheus's service account to reach the backend's metrics port, OR annotate pods to tell Envoy to bypass mTLS for the metrics port (`traffic.sidecar.istio.io/excludeInboundPorts`).

The simplest approach for this lab: annotate the backend pod to exclude port `9090` (metrics) from Istio interception, keeping mTLS only on the app traffic port `8000`.

---

## Out of scope for Phase 5

- **AlertManager rules** — no notifications to Slack, PagerDuty, or email configured
- **Long-term storage** — Prometheus retention is default (15 days in-memory); no Thanos or Cortex
- **Distributed tracing** — Jaeger/Zipkin deferred
- **Custom recording rules** — no precomputed aggregation rules
- **Grafana alerting** — Grafana alert rules and notification channels deferred
- **Prometheus federation** — single Prometheus instance only
- **Ingress for Grafana** — accessed via port-forward only; no `grafana.local` ingress
