# Requirements — Phase 3: App Stack

## Prerequisites

- Phase 2 complete: `postgres-secret` exists in the `app` namespace with `username`, `password`, and `database` keys
- `app` namespace already created (Phase 2, Group 6)
- `KUBECONFIG` set to `~/.kube/rke2.yaml`
- Helm repos: Bitnami and ingress-nginx will be added during this phase

---

## Key decisions

**Bitnami chart for Postgres**
The Bitnami PostgreSQL chart has native support for `auth.existingSecret`, which lets it pull credentials from the ESO-managed `postgres-secret` without ever placing a password in a manifest or Helm values file. This is the whole point of Phase 2 — Phase 3 consumes it correctly.

**nginx for the frontend**
The frontend has no build step — it's a static HTML file served by nginx. This keeps the image small and the Dockerfile trivial. The nginx `proxy_pass` directive handles the `/api/` → backend routing, so the browser only ever talks to one origin (no CORS issues).

**No external image registry**
The cluster has no access to a private registry and we don't want to push to Docker Hub for a local lab. Images are built directly inside the agent VMs using `nerdctl` (containerd's CLI, pre-installed with RKE2). The image is then available to pods scheduled on that node. For multi-node scheduling, the image must be built on each agent VM or a local registry must be set up — we build on both agents to keep it simple.

**CORS**
The FastAPI backend uses `CORSMiddleware` with `allow_origins=["*"]` for simplicity. In this setup the frontend proxies all `/api/` calls through nginx, so CORS headers are not actually required — the browser sees all traffic as same-origin. The middleware is included for correctness if the backend is ever called directly.

---

## Image build strategy

RKE2 uses containerd, not Docker. The workflow per agent VM is:

```bash
# Copy source into the VM
multipass transfer <local-file> <vm-name>:<remote-path>

# Build with nerdctl (containerd-native, ships with RKE2)
multipass exec <vm-name> -- sudo /var/lib/rancher/rke2/bin/nerdctl \
  -n k8s.io build -t <image-name>:<tag> <build-context>
```

The `-n k8s.io` namespace flag makes the image visible to the kubelet without any import step.

---

## Out of scope for Phase 3

- TLS / HTTPS — ingress will serve plain HTTP on `app.local`
- Persistent volumes beyond the Bitnami chart's default (local-path) — data will be lost on pod restart
- Database migrations / schema management — the `items` table is created manually for the smoke test
- Authentication or authorization at the app layer — deferred to Phase 4 (Istio AuthorizationPolicy)
- Horizontal pod autoscaling or resource limits
- Health checks / readiness probes beyond Kubernetes defaults
- A container image registry — images are built directly on the VM nodes
