# Kubeadm Cluster Upgrade

Related: [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) | [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

## Kubernetes Versioning

Kubernetes follows semver: `major.minor.patch` (e.g., `v1.35.2`).

| Upgrade type | Example          | Description                            |
|--------------|------------------|----------------------------------------|
| Patch        | 1.35.0 → 1.35.1  | Bug fixes and security patches         |
| Minor        | 1.34.0 → 1.35.0  | New features, API changes              |
| Major        | 1.x → 2.x        | Breaking changes (hasn't happened yet) |

The major version has been `1` since Kubernetes' first stable release. Going from `1.34` to `1.35` is a minor upgrade — and kubeadm only supports upgrading one minor version at a time.

## Package Repository

Since Kubernetes 1.28, each minor version has its own apt repository. Before upgrading, point to the new version's repo:

```bash
# Update the version in the existing apt source (e.g., v1.34 → v1.35)
sudo sed -i 's/v1.34/v1.35/g' /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
```

> On RHEL/CentOS, update the `baseurl` in `/etc/yum.repos.d/kubernetes.repo` to point to the new minor version instead.

## Upgrade Steps

These steps assume the package repository already points to the new minor version (see [Package Repository](#package-repository) above).

### 1. Find the exact package version

The apt/yum package version includes a suffix (e.g., `-1.1`) that changes between builds. List the available versions to get the exact string:

```bash
sudo apt-get update
apt-cache madison kubeadm
```

Use the resulting version (e.g., `1.35.0-1.1`) in the install commands below.

### 2. Upgrade the first control plane node

```bash
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.35.0-1.1
sudo apt-mark hold kubeadm

# Shows the target versions and a per-component upgrade table
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.35.0

kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.35.0-1.1 kubectl=1.35.0-1.1
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

kubectl uncordon <node-name>
```

### 3. Upgrade additional control plane nodes (if any)

```bash
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.31.0-1.1
sudo apt-mark hold kubeadm

sudo kubeadm upgrade node

kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.31.0-1.1 kubectl=1.31.0-1.1
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

kubectl uncordon <node-name>
```

### 4. Upgrade worker nodes (one at a time)

```bash
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.31.0-1.1
sudo apt-mark hold kubeadm

sudo kubeadm upgrade node

kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.31.0-1.1 kubectl=1.31.0-1.1
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

kubectl uncordon <worker-node>
```

### 5. Verify

```bash
kubectl get nodes
```

All nodes should show the new version and `Ready` status.

## Automatic etcd Backups

When `kubeadm upgrade apply` runs, it automatically creates backups before proceeding:

- `/etc/kubernetes/tmp/kubeadm-backup-etcd-<timestamp>` — etcd data snapshot
- `/etc/kubernetes/tmp/kubeadm-backup-manifests-<timestamp>` — static pod manifests

These can be used to restore the cluster if the upgrade fails.

## Important Notes

- Always upgrade one minor version at a time (e.g., 1.30 → 1.31, not 1.30 → 1.32).
- Upgrade control plane nodes before workers.
- Only run `kubeadm upgrade apply` on the first control plane node. Use `kubeadm upgrade node` on all others.
- Drain nodes before upgrading kubelet to avoid workload disruption.
- If you're on RHEL/CentOS, swap `apt` commands for `yum` and `apt-mark` for `yum versionlock`.
- Back up etcd before starting — `kubeadm upgrade` does this automatically, but a manual snapshot is good insurance.

## Skills Practiced

- Understanding Kubernetes semver and minor-version upgrade constraints
- Repointing package repositories for a new minor version
- Relying on kubeadm's automatic etcd and manifest backups
- Sequencing control plane and worker node upgrades safely
