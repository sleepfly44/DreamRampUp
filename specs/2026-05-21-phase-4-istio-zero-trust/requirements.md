# Requirements — Phase 4: Istio + Zero-Trust Networking

## Prerequisites

- Phase 3 complete: `http://app.local` loads and returns live data from Postgres
- All three app pods (frontend, backend, postgres) running and healthy in the `app` namespace
- `istioctl` installed locally (`brew install istioctl`)
- Verify the app is fully working **before** installing Istio — once deny-all is applied, traffic breaks until explicit allow policies are in place

---

## Key decisions

**Istio over Calico/Cilium network policies**
Kubernetes NetworkPolicies (Calico, Cilium) operate at L3/L4 — they can allow or deny by IP and port but cannot distinguish *which service* is calling (no identity). Istio operates at L7 with mutual TLS: each workload gets a SPIFFE identity (via its service account), and AuthorizationPolicies enforce access based on that identity. This means `frontend → backend` is allowed but an arbitrary pod in the same namespace is denied, even if it targets the same port.

**STRICT mTLS over PERMISSIVE**
PERMISSIVE mode allows both plaintext and mTLS traffic — useful during migration but provides no security guarantee. STRICT mode rejects any non-mTLS traffic, ensuring the entire mesh is encrypted and every caller is authenticated. For a learning lab focused on zero-trust patterns, STRICT is the only meaningful choice.

**Default deny-all first, then explicit allows**
The order matters: apply deny-all first, then add allow policies one at a time. This proves each policy is necessary — if you add allows before deny-all you can't tell what was already working by default. It also mirrors real production practice: never rely on implicit defaults.

**Why AuthorizationPolicy over NetworkPolicy**
AuthorizationPolicy is Istio-native and works on service account identity (SPIFFE), not pod IP ranges. This survives pod restarts and rescheduling without any IP management.

---

## Blast radius warning

**Group 4 (deny-all) will immediately break `http://app.local`.**
This is expected and intentional. The restoration sequence is:
1. Apply deny-all → traffic breaks
2. Apply frontend → backend allow → browser loads HTML but `/api/items` may fail
3. Apply backend → postgres allow → full stack restored

Do not panic when things break in Group 4. Follow the group order.

Additionally, enabling Istio sidecar injection (Group 2) restarts all app pods. There will be a brief outage while pods re-roll.

---

## Out of scope for Phase 4

- **Kiali** — service mesh visualisation dashboard; useful but not required to validate zero-trust
- **Jaeger / Zipkin** — distributed tracing; deferred
- **Prometheus / Grafana** — metrics; deferred
- **Istio ingress gateway** — we continue using RKE2's nginx ingress; Istio's gateway is a separate pattern
- **Egress gateways** — controlling outbound traffic from the mesh; out of scope
- **Multi-cluster mesh** — single cluster only
- **JWT / OIDC request authentication** — end-user auth at the mesh layer; deferred
