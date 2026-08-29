# hetzner-k3s Cheatsheet

[hetzner-k3s](https://github.com/vitobotta/hetzner-k3s) is a CLI that spins up production-ready [k3s](https://k3s.io/) Kubernetes clusters on Hetzner Cloud in minutes — no Terraform, no management cluster, and your Hetzner token never leaves your machine. You describe the cluster in a YAML file and the tool creates the servers, private network, firewall, load balancer, and installs k3s. To manage the underlying Hetzner resources directly, pair it with the [hcloud CLI](articles/hcloud-cheatsheet.md).

> Commands and the config schema below follow hetzner-k3s v2. The tool moves quickly — check `hetzner-k3s --version` and the [official docs](https://vitobotta.github.io/hetzner-k3s) for the exact options in your release.

## Installation

```bash
# macOS / Linux (Homebrew)
brew install vitobotta/tap/hetzner_k3s

# Linux binary (amd64) — adjust the version tag as needed
wget https://github.com/vitobotta/hetzner-k3s/releases/download/v2.4.5/hetzner-k3s-linux-amd64
chmod +x hetzner-k3s-linux-amd64
sudo mv hetzner-k3s-linux-amd64 /usr/local/bin/hetzner-k3s

# Verify
hetzner-k3s --version
```

## Core Commands

```bash
# Create (or update) a cluster from a config file
hetzner-k3s create --config cluster.yaml

# Delete the entire cluster
hetzner-k3s delete --config cluster.yaml

# List available k3s releases
hetzner-k3s releases

# Upgrade to a specific k3s version (the version flag is required)
hetzner-k3s upgrade --config cluster.yaml --new-k3s-version v1.33.7+k3s3

# Run a command or script on nodes
hetzner-k3s run --config cluster.yaml --command "systemctl status k3s"
hetzner-k3s run --config cluster.yaml --instance master-1 --command "kubectl get nodes"
hetzner-k3s run --config cluster.yaml --script ./maintenance.sh
hetzner-k3s run --config cluster.yaml --instance worker-1 --script ./check-disk.sh
```

Most commands accept:

- `--force` — skip confirmation prompts (useful in automation)
- `--quiet` — suppress the sponsor message

```bash
hetzner-k3s create  --config cluster.yaml --force --quiet
hetzner-k3s delete  --config cluster.yaml --force
hetzner-k3s upgrade --config cluster.yaml --new-k3s-version v1.33.7+k3s3 --force
```

## Configuration File

### Minimal HA cluster

```yaml
hetzner_token: <your-hetzner-api-token>
cluster_name: my-cluster
kubeconfig_path: "./kubeconfig"
k3s_version: v1.32.0+k3s1

networking:
  ssh:
    port: 22
    use_agent: false
    public_key_path: "~/.ssh/id_ed25519.pub"
    private_key_path: "~/.ssh/id_ed25519"
  allowed_networks:
    ssh:
      - 0.0.0.0/0
    api:
      - 0.0.0.0/0

masters_pool:
  instance_type: cpx22
  instance_count: 3
  locations:
    - fsn1
    - hel1
    - nbg1

worker_node_pools:
  - name: workers
    instance_type: cpx32
    instance_count: 3
    location: hel1
```

> Use an **odd** number of masters (1, 3, or 5) so etcd can form a quorum. Spreading 3 masters across `fsn1`/`hel1`/`nbg1` survives a single-location outage.

### Production cluster with autoscaling and ARM workers

```yaml
hetzner_token: <your-token>
cluster_name: prod-cluster
kubeconfig_path: "./kubeconfig"
k3s_version: v1.32.0+k3s1

networking:
  ssh:
    port: 22
    use_agent: false
    public_key_path: "~/.ssh/id_ed25519.pub"
    private_key_path: "~/.ssh/id_ed25519"
  allowed_networks:
    ssh:
      - 10.0.0.0/8      # restrict SSH to the private network
    api:
      - 0.0.0.0/0
  private_network:
    enabled: true
    subnet: 10.0.0.0/16

masters_pool:
  instance_type: cpx22
  instance_count: 3
  locations:
    - fsn1
    - hel1
    - nbg1

worker_node_pools:
  - name: general
    instance_type: cpx32
    instance_count: 3
    location: hel1
    autoscaling:
      enabled: true
      min_instances: 3
      max_instances: 10

  - name: arm-workers
    instance_type: cax21   # ARM (Ampere)
    instance_count: 2
    location: fsn1
    labels:
      - "node.kubernetes.io/instance-type=arm64"
    taints:
      - "arch=arm64:NoSchedule"

# Optional CNI (default: flannel)
cni: cilium

# Optional extra k3s args
additional_k3s_server_args:
  - "--disable=traefik"
  - "--disable=servicelb"

additional_k3s_agent_args:
  - "--node-label=environment=production"
```

## Common Workflows

### Initial setup

```bash
# 1. Create a Hetzner API token at https://console.hetzner.cloud/ (Security -> API Tokens)
export HETZNER_TOKEN="your-token-here"

# 2. Generate an SSH key if you don't have one
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "hetzner-k3s"

# 3. Write cluster.yaml (see above), then create the cluster
hetzner-k3s create --config cluster.yaml

# 4. Use the cluster
export KUBECONFIG=./kubeconfig
kubectl get nodes -o wide
```

### Scale a worker pool

`create` is idempotent — bump `instance_count` (or autoscaling bounds) in `cluster.yaml` and re-run it to reconcile.

```bash
hetzner-k3s create --config cluster.yaml
```

### Upgrade k3s

Upgrades are driven by the [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller): `hetzner-k3s upgrade` creates upgrade **Plans** that spawn **Jobs** in the `system-upgrade` namespace, which drain and upgrade nodes one at a time. Roll out and verify as follows.

**1. Initiate the upgrade**

```bash
hetzner-k3s releases                                    # pick a target version
hetzner-k3s upgrade --config cluster.yaml --new-k3s-version v1.35.0+k3s3
```

**2. Monitor the rollout**

```bash
# Node versions change as the controller works through them
watch kubectl get nodes -o wide

# Upgrade plans, jobs, and pods
watch kubectl get jobs,pods -n system-upgrade
kubectl get plan -n system-upgrade -o wide

# Follow a specific upgrade job's logs
kubectl logs -n system-upgrade -f job/<upgrade-job-name>
```

**3. Verify completion**

```bash
kubectl get jobs -n system-upgrade                      # all should be Complete
kubectl get nodes -o wide                               # all Ready on the new version

# Catch any failed or still-running jobs
kubectl get jobs -n system-upgrade --field-selector status.successful!=1
```

**4. Re-run create (important)**

After the k3s version has rolled out, run `create` again so hetzner-k3s reconciles its own state and configuration files to the new version. Skipping this can leave the tool's tracked version out of sync.

```bash
hetzner-k3s create --config cluster.yaml
```

**5. Clean up the upgrade plans and jobs**

Once everything is healthy, remove the completed plans/jobs so the next upgrade starts clean.

```bash
kubectl -n system-upgrade delete job --all
kubectl -n system-upgrade delete plan --all
```

> Update the `k3s_version` in `cluster.yaml` to match the new version so future `create`/`upgrade` runs stay consistent.

### Update the node OS

`upgrade` only touches k3s — it doesn't patch the underlying OS. Use `run` to apply OS package updates across all nodes:

```bash
hetzner-k3s run --config cluster.yaml --command "sudo apt update && sudo apt upgrade -y"
```

For updates that need a reboot, roll nodes one at a time (drain, reboot, uncordon) to avoid disrupting workloads.

### Access and inspect

```bash
export KUBECONFIG=./kubeconfig
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
kubectl top nodes          # requires metrics-server
```

### Back up

```bash
cp cluster.yaml cluster.yaml.backup
cp kubeconfig kubeconfig.backup
kubectl get all -A -o yaml > cluster-state-backup.yaml
```

## Instance Types

x86 (Intel/AMD):

| Type | vCPU | RAM | Disk | Use case |
|------|------|-----|------|----------|
| `cx22` | 2 | 4 GB | 40 GB | dev/test |
| `cpx22` | 3 | 4 GB | 80 GB | small masters |
| `cpx32` | 4 | 8 GB | 160 GB | general workers |
| `cpx42` | 8 | 16 GB | 240 GB | large workers |

ARM (CAX — cheaper per core):

| Type | vCPU | RAM | Disk | Use case |
|------|------|-----|------|----------|
| `cax21` | 4 | 8 GB | 80 GB | budget workers |
| `cax31` | 8 | 16 GB | 160 GB | ARM workloads |

> ARM nodes need ARM-built container images. Use `labels`/`taints` on ARM pools so only compatible workloads land there.

## Locations

| Code | Location | Region |
|------|----------|--------|
| `fsn1` | Falkenstein | Germany |
| `nbg1` | Nuremberg | Germany |
| `hel1` | Helsinki | Finland |
| `ash` | Ashburn | USA |
| `hil` | Hillsboro | USA |
| `sin` | Singapore | Asia |

## Troubleshooting

### Cluster status

```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl get pods -n kube-system
kubectl logs -n kube-system <pod-name>
```

### SSH to a node and check k3s

```bash
kubectl get nodes -o wide            # find the node IP
ssh -i ~/.ssh/id_ed25519 root@<node-ip>
systemctl status k3s
journalctl -u k3s -f
```

### Common issues

Cluster creation fails — verify the token and SSH access:

```bash
curl -H "Authorization: Bearer $HETZNER_TOKEN" https://api.hetzner.cloud/v1/servers
ssh-add -l
```

Nodes not joining — check the firewall, private network, and k3s logs on the node:

```bash
ssh root@<node-ip> "journalctl -u k3s -n 100"
```

Autoscaling not working — inspect the cluster-autoscaler:

```bash
kubectl get pods -n kube-system | grep autoscaler
kubectl logs -n kube-system -l app=cluster-autoscaler
kubectl get configmap -n kube-system cluster-autoscaler-status -o yaml
```

## Best Practices

**High availability**
- 3 or 5 masters (odd for etcd quorum), spread across locations.
- At least 2 workers per pool.

**Security**
- Restrict `allowed_networks.ssh` to known IPs/ranges rather than `0.0.0.0/0`.
- Enable `private_network` so cluster traffic stays off the public internet.
- Use a separate SSH key per cluster and keep k3s current.

**Cost**
- Prefer ARM (CAX) for compatible workloads.
- Enable autoscaling to shed workers during quiet periods.
- Use small types for dev/test and delete idle clusters.

**Performance**
- For large clusters (>100 nodes), give etcd/masters more resources.
- Consider Cilium CNI for advanced networking and policy.
- Deploy metrics-server to see `kubectl top` data.

## Sample Monthly Costs

Rough guidance; verify current Hetzner pricing.

| Configuration | Masters | Workers | Approx. cost |
|---------------|---------|---------|--------------|
| Dev | 1× cx22 | 2× cx22 | ~€16 |
| Small prod | 3× cpx22 | 3× cpx32 | ~€58 |
| Medium prod | 3× cpx22 | 10× cpx32 | ~€135 |
| Large prod | 3× cpx42 | 50× cpx32 | ~€615 |

Includes a load balancer (~€5.50/mo). hetzner-k3s itself is free and adds no management fees.

## Resources

- Repository: [github.com/vitobotta/hetzner-k3s](https://github.com/vitobotta/hetzner-k3s)
- Documentation: [vitobotta.github.io/hetzner-k3s](https://vitobotta.github.io/hetzner-k3s)
- Hetzner Cloud Console: [console.hetzner.cloud](https://console.hetzner.cloud/)
- k3s docs: [docs.k3s.io](https://docs.k3s.io/)

Content rephrased for compliance with licensing restrictions, based on the [hetzner-k3s documentation](https://github.com/vitobotta/hetzner-k3s).
