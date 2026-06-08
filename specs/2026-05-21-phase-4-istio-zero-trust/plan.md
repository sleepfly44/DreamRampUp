# Plan — Phase 4: Istio + Zero-Trust Networking

Each group maps to a roadmap step. Order matters — do not skip ahead.
Enabling deny-all (Group 4) will break traffic; Groups 5 and 6 restore it explicitly.

---

## Group 1 — Install Istio (roadmap 4.1)

1. Install `istioctl` if not present: `brew install istioctl`
2. Install Istio with the default profile:
   `istioctl install --set profile=default -y`
3. Wait for istiod and ingress gateway pods to be Ready

**Done:** `kubectl -n istio-system get pods` shows `istiod` and `istio-ingressgateway` Running.

---

## Group 2 — Enable sidecar injection and restart app pods (roadmap 4.2)

1. Label the `app` namespace for automatic sidecar injection:
   `kubectl label namespace app istio-injection=enabled`
2. Restart all deployments so existing pods get the Envoy sidecar:
   `kubectl rollout restart deployment -n app`
3. Wait for all pods to re-roll with 2/2 containers

**Done:** Every pod in the `app` namespace shows `2/2` containers (app + envoy proxy).

---

## Group 3 — Apply mesh-wide STRICT mTLS (roadmap 4.3)

1. Apply `istio/peer-authentication.yaml` (PeerAuthentication with `mode: STRICT` in `istio-system`)
2. Verify mTLS is enforced on the backend service

**Done:** `istioctl x describe service backend.app` reports mTLS as `STRICT`.

---

## Group 4 — Apply default deny-all AuthorizationPolicy (roadmap 4.4)

1. Apply `istio/deny-all.yaml` (empty-spec AuthorizationPolicy in the `app` namespace)
2. Verify that cross-service calls are now blocked

**Done:** A curl from inside a pod to another service returns 403 or connection refused.
Note: `http://app.local` will stop working after this step — expected.

---

## Group 5 — Allow frontend → backend (roadmap 4.5)

1. Apply `istio/allow-frontend-to-backend.yaml`
2. Verify `http://app.local` loads again (frontend can reach backend)

**Done:** `http://app.local` returns HTML and the items list loads in the browser.

---

## Group 6 — Allow backend → Postgres (roadmap 4.6)

1. Apply `istio/allow-backend-to-postgres.yaml`
2. Verify `/api/items` returns data from Postgres

**Done:** `curl http://app.local/api/items` (via nginx proxy) returns JSON with items.

---

## Group 7 — Verify no unintended paths are open (roadmap 4.7)

1. Launch a temporary debug pod in the `app` namespace (not frontend or backend)
2. Attempt to curl the backend directly from the debug pod
3. Confirm the request is denied

**Done:** Direct curl from the debug pod to `backend:8000/api/items` returns 403 RBAC denied.
