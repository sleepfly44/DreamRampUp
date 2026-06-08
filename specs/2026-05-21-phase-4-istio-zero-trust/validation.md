# Validation — Phase 4: Istio + Zero-Trust Networking

Run each check after completing all plan groups.
All 7 must pass before Phase 4 is considered done and this branch can be merged.

---

## Checks

| # | Command | Expected output |
|---|---|---|
| V1 | `kubectl -n istio-system get pods` | `istiod` pod Running; `istio-ingressgateway` pod Running |
| V2 | `kubectl -n app get pods` | All app pods show `2/2` containers (app + envoy sidecar) |
| V3 | `istioctl x describe service backend.app` | Output includes `mTLS: STRICT` |
| V4 | `kubectl -n app run test --image=curlimages/curl --restart=Never --rm -i -- curl -s http://backend:8000/api/items` (before allow policies) | `RBAC: access denied` or connection refused |
| V5 | `curl -s http://app.local` | Returns HTML page (frontend → backend allow is working) |
| V6 | `curl -s http://app.local/api/items` | Returns JSON array of items (backend → postgres allow is working) |
| V7 | `kubectl -n app run test --image=curlimages/curl --restart=Never --rm -i -- curl -s http://backend:8000/api/items` (after all allow policies) | `RBAC: access denied` — debug pod has no allow policy |

---

## Pass bar

V3 (STRICT mTLS confirmed), V5+V6 (full stack working through policies), and V7 (deny-all enforced for unknown callers) are the critical checks. All 7 must pass before merging.

---

## Common failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| istiod pod stuck in `Pending` | Insufficient resources on nodes | `kubectl describe pod -n istio-system` to check; verify agent VMs have headroom |
| Pods still show `1/1` after restart | Namespace label not applied | `kubectl get ns app --show-labels`; confirm `istio-injection=enabled` |
| `istioctl x describe` shows PERMISSIVE | PeerAuthentication not applied or wrong namespace | Must be in `istio-system` to be mesh-wide; check `kubectl get peerauthentication -A` |
| V5 fails after deny-all | Frontend → backend allow policy not applied or wrong service account name | `kubectl describe authorizationpolicy -n app`; verify `principals` match actual service account |
| V6 fails after allow | Backend → postgres allow policy missing or Postgres not in the mesh | Postgres pod must also show `2/2`; check sidecar injection |
| V7 returns 200 (not denied) | Allow policy is too broad (matching all sources) | Review `spec.rules[].from[].source.principals` — must name specific service accounts |
