# Requirements — Phase 2: HashiCorp Vault + External Secrets Operator

## Prerequisites

- Phase 1 complete: all three nodes `Ready` in `kubectl get nodes`
- `KUBECONFIG` set to `~/.kube/rke2.yaml` (or exported in shell)
- `vault` CLI installed: `brew install vault`
- Helm repos will be added during this phase — no pre-install needed

---

## Key decisions

**Vault dev mode (not production mode)**
Dev mode starts Vault unsealed, with a fixed root token (`root`), and stores all data in memory. This means secrets are lost on pod restart. The trade-off is zero operational overhead — no unseal keys to manage, no persistent volume to configure — which is appropriate for a learning lab focused on the integration pattern, not Vault operations.

**ESO over native Kubernetes Secrets**
Kubernetes Secrets can be created manually with `kubectl create secret`, but that requires secrets to live in version-controlled manifests or be applied by hand. ESO automates the sync from Vault to K8s Secrets on a refresh interval, which is the production pattern. Learning ESO here pays off in Phase 3 when the app consumes secrets without ever seeing them in a manifest.

**KV v2 over KV v1**
KV v2 adds secret versioning and soft-delete. It's the current Vault default and what ESO's `version: "v2"` field targets. No meaningful downside for this use case.

**Token auth over AppRole / Kubernetes auth**
Vault supports several auth methods for machine identities (AppRole, Kubernetes service account JWT, etc.). Token auth with the root token is the simplest — one less moving part while learning the ESO integration. AppRole or Kubernetes auth would be the next step in a hardening exercise.

---

## Security notes

Dev mode is explicitly unsafe for production:

- The root token (`root`) is hardcoded and has full Vault access
- All data is in-memory and lost on pod restart
- Vault is not TLS-terminated inside the cluster
- The Vault UI is exposed without authentication controls

None of this matters for a local learning lab, but the pattern established here (SecretStore → ExternalSecret → K8s Secret) is identical to what a production setup uses — only the auth method and Vault configuration change.

---

## Out of scope for Phase 2

The following are explicitly deferred:

- Vault HA (multi-replica, Raft storage) — deferred indefinitely for this lab
- Auto-unseal (AWS KMS, GCP KMS, etc.) — not applicable in dev mode
- TLS between ESO and Vault — deferred; dev mode Vault is HTTP only
- Vault Agent / Vault Agent Injector — alternative secret delivery pattern, not needed here
- AppRole or Kubernetes auth methods — token auth is sufficient for this phase
- Secrets for anything other than Postgres — Phase 3 will consume what's set up here
- Vault backup / snapshot — dev mode data is ephemeral by design
