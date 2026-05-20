# Validation — Phase 2: HashiCorp Vault + External Secrets Operator

Run each check after completing all plan groups.
All 7 must pass before Phase 2 is considered done and this branch can be merged.

---

## Checks

| # | Command | Expected output |
|---|---|---|
| V1 | `kubectl -n vault get pods` | Vault pod shows `Running` with `1/1` ready |
| V2 | `vault secrets list` | Output includes a row with `secret/` and type `kv` |
| V3 | `vault kv get secret/postgres` | Returns `username`, `password`, and `database` fields with their values |
| V4 | `kubectl -n external-secrets get pods` | ESO controller pod shows `Running` with `1/1` ready |
| V5 | `kubectl get clustersecretstore vault-backend` | `READY` column shows `True` |
| V6 | `kubectl -n app get secret postgres-secret` | Secret exists with `TYPE=Opaque` |
| V7 | `kubectl -n app get secret postgres-secret -o jsonpath='{.data.username}' \| base64 -d` | Prints `appuser` |

---

## Pass bar

All 7 checks return the expected output. Do not proceed to Phase 3 until V5 (ClusterSecretStore READY) and V7 (secret value correct) both pass — these confirm the full Vault → ESO → K8s Secret pipeline is working, not just that components are running.

---

## Common failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| Vault pod stuck in `Pending` | No schedulable node with available resources | `kubectl describe pod -n vault` to check events; verify agent nodes are Ready |
| `vault secrets list` connection refused | Port-forward not running | Re-run `kubectl -n vault port-forward svc/vault 8200:8200 &` |
| ClusterSecretStore `READY=False` | Vault token secret missing or wrong namespace | Check `kubectl -n external-secrets get secret vault-token`; token value must be `root` |
| ExternalSecret stuck in `SecretSyncedError` | Vault path or field name mismatch | `kubectl describe externalsecret postgres-credentials -n app` for the exact error |
| `postgres-secret` missing `username` key | Wrong `property` field in ExternalSecret spec | Confirm `remoteRef.property` matches the field name written with `vault kv put` |
