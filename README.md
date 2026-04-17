# ☸️ Kubernetes Cluster — Ansible Automation
kubeadm-ansible-cluster
Build, upgrade, and remove a production-ready Kubernetes cluster using `kubeadm` and Ansible. Supports single-master and **highly available multi-master** setups with an  HAProxy load balancer.

> **Optional:** Use the [Initial Server Setup playbook](./server-setup/README.md) first to prepare your servers with a dedicated Ansible user before running this playbook.

---

## 📋 Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Inventory Configuration](#inventory-configuration)
- [Variables Reference](#variables-reference)
- [How It Works — Play by Play](#how-it-works--play-by-play)
- [Usage](#usage)
- [Upgrade the Cluster](#upgrade-the-cluster)
- [Remove the Cluster](#remove-the-cluster)
- [Troubleshooting](#troubleshooting)

---

## Architecture

```
                        ┌──────────────────────────┐
                        │       HAProxy LB         │
                        │       :6443              │
                        │  (control-plane endpoint)│
                        └──────────┬───────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
┌───────▼────────┐       ┌────────▼────────┐        ┌────────▼────────┐
│   master-1     │       │   master-2      │        │   master-3      │
│ (control-plane)│       │ (control-plane) │        │ (control-plane) │
└────────────────┘       └─────────────────┘        └─────────────────┘
        │                          │                          │
        └───────────────┬──────────┴──────────┬───────────────┘
                        │                     │
              ┌─────────▼────────┐   ┌────────▼────────┐
              │    worker-1      │   │    worker-2     │   ... more workers
              └──────────────────┘   └─────────────────┘
```

**Single-master setup:** Remove the load balancer and extra masters from the inventory — `kubeadm init` will use the bootstrap master IP as the control-plane endpoint directly.

**HA multi-master setup:** Keep the load balancer in inventory,
3 masters (odd number) → required for etcd quorum stability ,
All masters register behind it and `--control-plane-endpoint` points to the LB DNS name.

---

## Prerequisites

### Control node

| Requirement | Details |
|---|---|
| **Ansible** | 2.9 or later |
| **`ansible.posix` collection** | `ansible-galaxy collection install ansible.posix` |
| SSH access | Key-based auth to all nodes as the configured `ansible_user` |

### Target nodes

| Requirement | Details |
|---|---|
| **OS** | Ubuntu 22.04 / Debian 12 (or compatible) |
| **CPU** | 2+ cores per node |
| **RAM** | 2 GB+ per node (4 GB+ recommended for masters) |
| **Network** | All nodes must reach each other on their `node_IP` |
| **Hostnames** | Must match the inventory host names exactly — see note below |

> ⚠️ **Hostname requirement:** The names you define in the inventory (e.g. `master1`, `worker1`) must match the actual system hostnames of the servers. Mismatched hostnames cause node registration conflicts in Kubernetes.

---

## Project Structure

```
k8s-cluster/
├── site.yml                        # Main entry point — all plays defined here
├── inventory/
│   └── hosts                       # Host definitions and group assignments
├── group_vars/
│   ├── all.yml                     # Global vars (ansible_user, cluster_domain)
│   ├── kube_cluster.yml            # Kubernetes version, CNI, pod CIDR
│   └── load-balancer.yml           # HAProxy settings
└── roles/
    ├── common/                     # System prep applied to ALL nodes
    ├── ha_proxy_lb/                # HAProxy load balancer setup
    ├── swap_manage/                # Disables swap (required by Kubernetes)
    ├── containerd/                 # Container runtime installation
    ├── k8s_packages/               # kubeadm, kubelet, kubectl installation
    ├── cluster_init/               # kubeadm init on bootstrap master
    ├── cluster_join_workers/       # Join worker nodes to the cluster
    ├── cluster_join_masters/       # Join additional masters (HA mode)
    ├── k8s_admin/                  # kubeconfig setup for admin user
    ├── cluster_network/            # CNI plugin deployment (Flannel)
    ├── cluster_status/             # Verify cluster is healthy after build
    ├── upgrade_k8s_cluster/        # Rolling cluster upgrade
    └── remove_k8s_cluster/         # Full cluster teardown
```

---

## Inventory Configuration

### File: `inventory/hosts`

```ini
[bootstrap_master]
<MASTER_1_HOSTNAME>

[master-nodes]
<MASTER_1_HOSTNAME>
<MASTER_2_HOSTNAME>
<MASTER_3_HOSTNAME>
[worker-nodes]
<WORKER_1_HOSTNAME>

[load-balancer]
<LB_HOSTNAME>

[kube_cluster:children]
bootstrap_master
master-nodes
worker-nodes

[all]
<MASTER_1_HOSTNAME>    ansible_host=<MASTER_1_IP>    node_IP=<MASTER_1_IP>
<MASTER_2_HOSTNAME>    ansible_host=<MASTER_2_IP>    node_IP=<MASTER_2_IP>
<MASTER_3_HOSTNAME>    ansible_host=<MASTER_3_IP>    node_IP=<MASTER_3_IP>
<WORKER_1_HOSTNAME>    ansible_host=<WORKER_1_IP>    node_IP=<WORKER_1_IP>
<LB_HOSTNAME>          ansible_host=<LB_IP>          node_IP=<LB_IP>
```

### Host groups explained

| Group | Purpose |
|---|---|
| `bootstrap_master` | The **first** master — `kubeadm init` runs here, join tokens are generated here |
| `master-nodes` | **All** masters including the bootstrap one — used for join and kubeconfig setup |
| `worker-nodes` | Worker nodes that join the cluster after masters are ready |
| `load-balancer` | Optional HAProxy node — omit entirely for single-master setups |
| `kube_cluster` | Meta-group combining all Kubernetes nodes (masters + workers) |

### Single-master setup

Leave `master-nodes` with only one entry and remove the `load-balancer` group entirely:

```ini
[bootstrap_master]
<MASTER_HOSTNAME>

[master-nodes]
<MASTER_HOSTNAME>

[worker-nodes]
<WORKER_1_HOSTNAME>
<WORKER_2_HOSTNAME>

[kube_cluster:children]
bootstrap_master
master-nodes
worker-nodes

[all]
<MASTER_HOSTNAME>     ansible_host=<MASTER_IP>     node_IP=<MASTER_IP>
<WORKER_1_HOSTNAME>   ansible_host=<WORKER_1_IP>   node_IP=<WORKER_1_IP>
<WORKER_2_HOSTNAME>   ansible_host=<WORKER_2_IP>   node_IP=<WORKER_2_IP>
```

When no load balancer is present, `kubeadm init` automatically uses the bootstrap master `node_IP` as the control-plane endpoint.

---

## Variables Reference

### `group_vars/all.yml` — Global settings

```yaml
ansible_user: <USERNAME>       # User Ansible connects as on all nodes
cluster_domain: <DOMAIN>       # Internal cluster domain, e.g. k8s-cluster.local
```

### `group_vars/kube_cluster.yml` — Kubernetes settings

```yaml
HOME_USER: "{{ ansible_user }}"         # Non-root user who gets kubeconfig access
POD_NETWORK_CIDR: "10.244.0.0/16"       # Pod CIDR — must match your CNI plugin
kubernetes_version: "v1.X"             # Version to install during build
Updated_version: "v1.Y"                # Target version for upgrades
package_version: "1.Y.Z-1.1"           # APT package version string for upgrade
kubeadm_version: "v1.Y.Z"              # kubeadm binary version for upgrade
cni_manifest: "<CNI_MANIFEST_URL>"     # CNI plugin manifest URL (default: Flannel)
```

> **CNI note:** The default `POD_NETWORK_CIDR` (`10.244.0.0/16`) is configured for Flannel. If you switch to a different CNI plugin (e.g. Calico uses `192.168.0.0/16`), update both the CIDR and the manifest URL accordingly.

### `group_vars/load-balancer.yml` — HAProxy settings

```yaml
haproxy_bind_ip: "{{ node_IP }}"                       # IP HAProxy listens on
LOAD_BALANCER_DNS: "loadbalancer.{{ cluster_domain }}" # DNS name used as control-plane endpoint
LOAD_BALANCER_PORT: "6443"                             # kube-apiserver port
haproxy_maxconn: 2000
haproxy_timeout_connect: "10s"
haproxy_timeout_client: "1m"
haproxy_timeout_server: "1m"
```

---

## How It Works — Play by Play

`site.yml` runs plays in strict order. Each play depends on the previous completing successfully.

```
 Play 1 ── common ──────────────────────────────► ALL nodes
              System prep: packages, kernel modules, sysctl, etc.

 Play 2 ── ha_proxy_lb ─────────────────────────► load-balancer only
              Installs and configures HAProxy to front the API servers.
              Skipped automatically if the load-balancer group is empty.

 Play 3 ── swap_manage + containerd + k8s_packages ► kube_cluster (masters + workers)
              Disables swap, installs containerd runtime,
              installs kubeadm / kubelet / kubectl.

 Play 4 ── cluster_init ────────────────────────► bootstrap_master only
              Runs: kubeadm init
              Generates worker join token and master certificate key.
              Saves both to Ansible facts for subsequent plays.

 Play 5 ── cluster_join_workers ────────────────► worker-nodes
              Each worker runs the join command generated in Play 4.

 Play 6 ── cluster_join_masters ────────────────► master-nodes (excluding bootstrap)
              Each additional master joins with --control-plane flag
              and the certificate key from Play 4.

 Play 7 ── k8s_admin ───────────────────────────► master-nodes
              Sets up kubeconfig for the non-root admin user.

 Play 8 ── cluster_network + cluster_status ────► bootstrap_master
              Applies the CNI manifest (Flannel by default).
              Verifies all nodes show as Ready.
```

> `any_errors_fatal: true` is set on all join and network plays — if any node fails, the entire run halts immediately to prevent a partially built cluster.

---

## Usage

### 1. Configure your inventory

Edit `inventory/hosts` and replace all placeholders with your actual hostnames and IPs.

### 2. Configure your variables

Edit `group_vars/all.yml` and `group_vars/kube_cluster.yml` to set your Kubernetes version, CNI manifest, and admin user.

### 3. Build the cluster

```bash
ansible-playbook -i inventory/hosts site.yml --tags build
```

### Target a single node (for debugging)

```bash
ansible-playbook -i inventory/hosts site.yml --tags build --limit <HOSTNAME>
```

### Dry run (no changes applied)

```bash
ansible-playbook -i inventory/hosts site.yml --tags build --check
```

### Verbose output

```bash
ansible-playbook -i inventory/hosts site.yml --tags build -v
```

---

## Upgrade the Cluster

The upgrade play runs **one node at a time** (`serial: 1`) in **sorted inventory order** to perform a safe rolling upgrade with zero downtime.

### Steps

**1. Set the target version in `group_vars/kube_cluster.yml`:**

```yaml
Updated_version: "v1.Y"           # Target minor version, e.g. v1.33
package_version: "1.Y.Z-1.1"      # Full APT package string, e.g. 1.33.1-1.1
kubeadm_version: "v1.Y.Z"         # Full kubeadm version, e.g. v1.33.1
```

> ⚠️ Only upgrade **one minor version at a time** (e.g. 1.31 → 1.32, not 1.31 → 1.33). Skipping minor versions is not supported by kubeadm.

**2. Run the upgrade:**

```bash
ansible-playbook -i inventory/hosts site.yml --tags upgrade
```

The `upgrade_k8s_cluster` role handles drain, package upgrade, kubeadm upgrade, and uncordon for each node automatically.

---

## Remove the Cluster

> ⚠️ This is **destructive and irreversible.** All cluster state and workloads will be lost.

```bash
ansible-playbook -i inventory/hosts site.yml --tags remove
```

The `remove_k8s_cluster` role runs `kubeadm reset`, removes Kubernetes packages, cleans up CNI configuration, and resets iptables rules on every node in `kube_cluster`.

---

## Troubleshooting

### Node stuck in `NotReady` after joining

- Verify CNI pods are running: `kubectl get pods -A | grep -i flannel`
- Confirm `node_IP` in the inventory matches the node's actual primary network interface IP
- Check kubelet logs on the affected node: `journalctl -u kubelet -n 50`

### `kubeadm init` fails — port already in use

A previous partial init may have left state behind. Reset the bootstrap master and re-run:

```bash
kubeadm reset -f
ansible-playbook -i inventory/hosts site.yml --tags build
```

### Additional masters fail to join

- The certificate key generated during `cluster_init` expires after **2 hours** — re-run the full build if too much time passed between plays
- Confirm the load balancer is healthy and port `6443` is reachable from all master nodes

### HAProxy play is skipped or shows `skipping: no hosts matched`

- The `[load-balancer]` group must be defined **and populated** in the inventory
- An empty group causes the play to silently run on zero hosts

### `kubectl` not working for the admin user

- The `k8s_admin` role sets up kubeconfig only for the user defined in `HOME_USER`
- Confirm `HOME_USER` matches the actual login user on the masters
- Manual fix on any master:
  ```bash
  mkdir -p ~/.kube
  sudo cp /etc/kubernetes/admin.conf ~/.kube/config
  sudo chown $(id -u):$(id -g) ~/.kube/config
  ```

### Verify the cluster is healthy

```bash
kubectl get nodes -o wide       # All nodes should show Ready
kubectl get pods -A             # All system pods should be Running
kubectl cluster-info            # API server endpoint should respond
```