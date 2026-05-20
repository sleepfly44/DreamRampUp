# Plan — Phase 1: RKE2 Cluster

Each group maps to a roadmap step. Complete groups in order; each has an atomic done check.

---

## Group 1 — Launch Multipass VMs (roadmap 1.1)

1. Install Multipass if not present: `brew install multipass`
2. Launch `rke2-server`: 4 CPU, 8 GB RAM, 40 GB disk
3. Launch `rke2-agent1`: 2 CPU, 6 GB RAM, 20 GB disk
4. Launch `rke2-agent2`: 2 CPU, 6 GB RAM, 20 GB disk
5. Confirm all three VMs are running

**Done:** `multipass list` shows all three VMs in `Running` state.

---

## Group 2 — Install RKE2 on the server node (roadmap 1.2)

1. Shell into `rke2-server`: `multipass shell rke2-server`
2. Download and install RKE2: `curl -sfL https://get.rke2.io | sudo sh -`
3. Enable the systemd service: `sudo systemctl enable rke2-server.service`
4. Start the service: `sudo systemctl start rke2-server.service`
5. Wait ~60 s for the control plane to initialize

**Done:** `sudo systemctl status rke2-server` reports `active (running)`.

---

## Group 3 — Capture join credentials (roadmap 1.3)

1. Inside `rke2-server`, read the node token:
   `sudo cat /var/lib/rancher/rke2/server/node-token`
2. Read the server's internal IP:
   `ip addr show eth0 | grep 'inet ' | awk '{print $2}' | cut -d/ -f1`
3. Save both values somewhere accessible (notes, env vars) before exiting the shell

**Done:** Token and IP are recorded; no step produced an error.

---

## Group 4 — Join rke2-agent1 (roadmap 1.4)

1. Shell into `rke2-agent1`: `multipass shell rke2-agent1`
2. Install RKE2 agent: `curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sudo sh -`
3. Create config dir: `sudo mkdir -p /etc/rancher/rke2`
4. Write `/etc/rancher/rke2/config.yaml` with `server` and `token` from Group 3
5. Enable and start the agent service
6. Exit and verify from Mac

**Done:** `kubectl get nodes` (after Group 6) shows `rke2-agent1` with `Ready`.

---

## Group 5 — Join rke2-agent2 (roadmap 1.5)

1. Repeat Group 4 steps for `rke2-agent2` (same RKE2 agent install, same config pattern)

**Done:** All three nodes show `Ready` in `kubectl get nodes`.

---

## Group 6 — Configure kubectl on Mac (roadmap 1.6)

1. Copy kubeconfig from the server VM:
   `multipass exec rke2-server -- sudo cat /etc/rancher/rke2/rke2.yaml > ~/.kube/rke2.yaml`
2. Patch the server address (replace `127.0.0.1` with the VM's actual IP):
   `SERVER_IP=$(multipass info rke2-server | grep IPv4 | awk '{print $2}')` then
   `sed -i '' "s/127.0.0.1/$SERVER_IP/" ~/.kube/rke2.yaml`
3. Export: `export KUBECONFIG=~/.kube/rke2.yaml`
4. (Optional) Append to `~/.zshrc` so it persists across sessions

**Done:** `kubectl get nodes` from the Mac terminal returns all three nodes as `Ready`.
