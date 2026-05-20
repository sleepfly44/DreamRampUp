# Validation — Phase 1: RKE2 Cluster

Run each check from your Mac terminal after completing all plan groups.
All six must pass before Phase 1 is considered done and this branch can be merged.

---

## Checks

| # | Command | Expected output |
|---|---|---|
| V1 | `multipass list` | Three entries — `rke2-server`, `rke2-agent1`, `rke2-agent2` — all in `Running` state |
| V2 | `multipass exec rke2-server -- sudo systemctl is-active rke2-server` | `active` |
| V3 | `kubectl get nodes` | Three nodes listed, all with `STATUS = Ready` |
| V4 | `kubectl get nodes -o wide \| grep rke2-server` | Node has `ROLES` containing `control-plane,etcd,master` |
| V5 | `kubectl get nodes -o wide \| grep rke2-agent` | Both agent nodes show `<none>` roles and `Ready` status |
| V6 | `kubectl cluster-info` | Prints a live `Kubernetes control plane` URL (not a connection error) |

---

## Pass bar

All six checks return the expected output with no errors. A single failure means the phase is not done — do not proceed to Phase 2.

---

## Common failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| Agent node stuck in `NotReady` | Wrong server IP or token in `config.yaml` | Re-check `/etc/rancher/rke2/config.yaml` on the agent VM; token must match exactly |
| `kubectl` connection refused | kubeconfig still points to `127.0.0.1` | Re-run the `sed` patch in Group 6 step 2 |
| `rke2-server` service fails to start | Not enough disk space or RAM | Check `multipass info rke2-server`; verify VM was launched with correct specs |
| `multipass shell` hangs | VM is not Running | `multipass start <vm-name>` then retry |
