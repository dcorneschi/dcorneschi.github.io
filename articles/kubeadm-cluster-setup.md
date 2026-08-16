# Kubernetes Cluster Setup with kubeadm

## Prerequisites

| Requirement | Control Plane | Worker Nodes |
|-------------|---------------|--------------|
| CPU | 2 vCPU minimum | 1 vCPU minimum |
| RAM | 2 GB minimum | 2 GB minimum |
| OS | Ubuntu 22.04/24.04 | Ubuntu 22.04/24.04 |
| Network | Static IP, no overlap with pod CIDR | Static IP, no overlap with pod CIDR |

Ensure node IP range and pod IP range don't overlap (e.g., nodes on `192.168.x.x`, pods on `10.244.0.0/16`).

## Port Requirements

### Control Plane

| Port | Protocol | Component |
|------|----------|-----------|
| 6443 | TCP | Kubernetes API server |
| 2379-2380 | TCP | etcd server client API |
| 10250 | TCP | Kubelet API |
| 10259 | TCP | kube-scheduler |
| 10257 | TCP | kube-controller-manager |

### Worker Nodes

| Port | Protocol | Component |
|------|----------|-----------|
| 10250 | TCP | Kubelet API |
| 10256 | TCP | kube-proxy |
| 30000-32767 | TCP | NodePort Services |

Additionally, allow all UDP traffic between cluster nodes (required by Calico for DNS and pod communication).

## Step 1: Enable iptables Bridged Traffic (All Nodes)

```bash
# Disable swap permanently
sudo swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true

sudo apt-get update -y

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set sysctl params (persist across reboots)
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply without reboot
sudo sysctl --system
```

## Step 2: Install CRI-O Container Runtime (All Nodes)

```bash
CRIO_VERSION="v1.36"

sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates
sudo install -m 0755 -d /etc/apt/keyrings

# Add CRI-O repository
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update -y
sudo apt-get install -y cri-o

sudo systemctl daemon-reload
sudo systemctl enable crio --now
```

## Step 3: Install crictl (All Nodes)

```bash
CRICTL_VERSION="v1.36.0"
ARCH="amd64"  # or arm64

curl -LO "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-${ARCH}.tar.gz"
sudo tar zxvf "crictl-${CRICTL_VERSION}-linux-${ARCH}.tar.gz" -C /usr/local/bin
rm -f "crictl-${CRICTL_VERSION}-linux-${ARCH}.tar.gz"

# Configure crictl to use CRI-O
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/crio/crio.sock
image-endpoint: unix:///run/crio/crio.sock
timeout: 10
debug: false
EOF
```

## Step 4: Install kubeadm, kubelet, kubectl (All Nodes)

```bash
KUBERNETES_VERSION="v1.36"

# Add Kubernetes repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y

# Install specific version
KUBERNETES_INSTALL_VERSION="1.36.0-1.1"
sudo apt-get install -y kubelet="$KUBERNETES_INSTALL_VERSION" kubectl="$KUBERNETES_INSTALL_VERSION" kubeadm="$KUBERNETES_INSTALL_VERSION"

# Or install latest from repo
# sudo apt-get install -y kubelet kubeadm kubectl

# Hold packages to prevent unintended upgrades
sudo apt-mark hold kubelet kubeadm kubectl
```

Set the node IP for kubelet:

```bash
sudo apt-get install -y jq
local_ip="$(ip --json addr show eth0 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"

cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
EOF
```

## Step 5: Create kubeadm Config (Control Plane Only)

Replace `192.168.249.201` with your control plane node's IP.

```yaml
# kubeadm.config
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.249.201"
  bindPort: 6443
nodeRegistration:
  name: "controlplane"

---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: "v1.36.0"
controlPlaneEndpoint: "192.168.249.201:6443"
apiServer:
  extraArgs:
    - name: "enable-admission-plugins"
      value: "NodeRestriction"
    - name: "audit-log-path"
      value: "/var/log/kubernetes/audit.log"
controllerManager:
  extraArgs:
    - name: "node-cidr-mask-size"
      value: "24"
scheduler:
  extraArgs:
    - name: "leader-elect"
      value: "true"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: "cluster.local"

---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: "systemd"
syncFrequency: "1m"

---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
conntrack:
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: "1h"
  tcpEstablishedTimeout: "24h"
```

For cloud VMs with public IP access, set `controlPlaneEndpoint` to the public IP. Use a static public IP to avoid issues after VM restarts.

## Step 6: Initialize the Cluster (Control Plane Only)

```bash
sudo kubeadm init --config=kubeadm.config
```

After successful initialization, configure kubectl access:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify:

```bash
kubectl get po -n kube-system
kubectl get --raw='/readyz?verbose'
kubectl cluster-info
```

CoreDNS pods will be in `Pending` state until the CNI plugin is installed.

To schedule workloads on the control plane (single-node setup):

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## Step 7: Join Worker Nodes

Run the join command from `kubeadm init` output on each worker node:

```bash
sudo kubeadm join 192.168.249.201:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

If you lost the join command, regenerate it:

```bash
kubeadm token create --print-join-command
```

Label worker nodes:

```bash
kubectl label node node01 node-role.kubernetes.io/worker=worker
kubectl label node node02 node-role.kubernetes.io/worker=worker
```

Nodes will show `NotReady` until the network plugin is installed.

## Step 8: Install Calico Network Plugin

```bash
# Install Tigera operator and CRDs
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/tigera-operator.yaml

# Download custom resources
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/custom-resources.yaml
```

Edit `custom-resources.yaml` and change the CIDR to match your pod subnet:

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
```

Verify the CIDR matches your cluster:

```bash
kubectl -n kube-system get pod -l component=kube-controller-manager -o yaml | grep -i cluster-cidr
```

Apply:

```bash
kubectl apply -f custom-resources.yaml
```

Wait for all pods to reach Running state:

```bash
kubectl get po -A
kubectl get nodes    # Should now show Ready
```

## Step 9: Install Metrics Server

```bash
kubectl apply -f https://raw.githubusercontent.com/techiescamp/cka-certification-guide/refs/heads/main/lab-setup/manifests/metrics-server/metrics-server.yaml
```

For local/self-signed clusters, the manifest includes `--kubelet-insecure-tls`. Verify after ~60 seconds:

```bash
kubectl top nodes
kubectl top pod -n kube-system
```

## Step 10: Validate the Cluster

```bash
# Cluster info
kubectl cluster-info

# Node status
kubectl get nodes -o wide

# All system pods running
kubectl get po -n kube-system

# DNS resolution test
kubectl apply -f https://raw.githubusercontent.com/techiescamp/cka-certification-guide/refs/heads/main/lab-setup/manifests/utilities/dnsutils.yaml
kubectl exec -it dnsutils -- nslookup kubernetes.default
kubectl exec -it dnsutils -- nslookup google.com
kubectl delete pod dnsutils
```

## Step 11: Deploy a Test Application

```bash
# Create nginx deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
EOF

# Expose via NodePort
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32000
EOF

# Verify
kubectl get pods
kubectl get svc nginx-service
# Access: http://<node-ip>:32000
```

## Connect from Workstation

```bash
# Copy admin.conf from control plane to local machine
scp user@controlplane:/etc/kubernetes/admin.conf ~/kubeadm-config.yaml

# Backup existing kubeconfig
cp ~/.kube/config ~/.kube/config.bak

# Merge configs
export KUBECONFIG=~/.kube/config:~/kubeadm-config.yaml
kubectl config view --flatten > ~/.kube/merged_config.yaml
mv ~/.kube/merged_config.yaml ~/.kube/config

# Switch context
kubectl config get-contexts -o name
kubectl config use-context <cluster-name>
```

## Important File Locations

| Configuration | Path |
|---------------|------|
| Static pod manifests (etcd, apiserver, controller-manager, scheduler) | `/etc/kubernetes/manifests/` |
| TLS certificates (kubernetes-ca, etcd-ca, front-proxy-ca) | `/etc/kubernetes/pki/` |
| Admin kubeconfig | `/etc/kubernetes/admin.conf` |
| Kubelet configuration | `/var/lib/kubelet/config.yaml` |
| Kubelet service defaults | `/etc/default/kubelet` |

## Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| `[ERROR NumCPU]: available CPUs 1 < required 2` | Insufficient CPU | Use a VM with at least 2 vCPU |
| Nodes stuck in `NotReady` | CNI not installed | Install Calico or another network plugin |
| CoreDNS in `Pending` | No CNI available | Install network plugin |
| `timed out waiting for the condition` | kubelet can't start, wrong advertise address | Use private IP in `advertiseAddress`, public IP in `controlPlaneEndpoint` |
| `/etc/kubernetes/kubelet.conf already exists` | Previous join attempt | Run `sudo kubeadm reset` on worker first |
| `Port 10250 is in use` | Previous kubelet running | Run `sudo kubeadm reset` and retry |
| Calico pods crash/restart | Pod CIDR overlaps with node network | Ensure pod CIDR and node IPs don't overlap |
| DNS queries time out | UDP traffic blocked between nodes | Allow all UDP between cluster nodes |

## Reset a Node

```bash
# Reset kubeadm config (run on the node to reset)
sudo kubeadm reset

# Clean up iptables
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# Clean up IPVS tables
sudo ipvsadm --clear

# Remove CNI config
sudo rm -rf /etc/cni/net.d

# Remove kubelet data
sudo rm -rf /var/lib/kubelet/*
```

## Upgrade Cluster

```bash
# Check available versions
apt-cache madison kubeadm | tac

# Upgrade kubeadm on the control plane
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.37.0-1.1
sudo apt-mark hold kubeadm

# Verify upgrade plan
sudo kubeadm upgrade plan

# Apply the upgrade (control plane)
sudo kubeadm upgrade apply v1.37.0

# Drain the control plane node
kubectl drain controlplane --ignore-daemonsets --delete-emptydir-data

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.37.0-1.1 kubectl=1.37.0-1.1
sudo apt-mark hold kubelet kubectl

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Uncordon the control plane node
kubectl uncordon controlplane
```

For worker nodes, drain first, then upgrade:

```bash
# From control plane — drain the worker
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data

# On the worker node
sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get install -y kubeadm=1.37.0-1.1 kubelet=1.37.0-1.1 kubectl=1.37.0-1.1
sudo apt-mark hold kubeadm kubelet kubectl
sudo kubeadm upgrade node
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# From control plane — uncordon
kubectl uncordon node01
```

## etcd Backup and Restore

Frequently tested in CKA. The etcd pod runs as a static pod — check its manifest for cert paths.

### Find etcd Certificates

```bash
# From the static pod manifest
cat /etc/kubernetes/manifests/etcd.yaml | grep -E "cert|key|cacert"
```

Typical paths:
- `--cert-file=/etc/kubernetes/pki/etcd/server.crt`
- `--key-file=/etc/kubernetes/pki/etcd/server.key`
- `--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt`

### Backup

```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify the snapshot
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-table

# On etcd 3.5+, use etcdutl instead (etcdctl status is deprecated)
etcdutl --write-out=table snapshot status /opt/etcd-backup.db
```

### Restore

```bash
# Stop kube-apiserver (move manifest temporarily)
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Restore to a new data directory (etcdutl preferred on etcd 3.5+)
etcdutl --data-dir=/var/lib/etcd-restored snapshot restore /opt/etcd-backup.db

# Or with etcdctl (deprecated in 3.5+, removed in 3.6)
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# Update etcd manifest to use the new data directory
sudo sed -i 's|/var/lib/etcd|/var/lib/etcd-restored|g' /etc/kubernetes/manifests/etcd.yaml

# Also update the hostPath volume in the etcd manifest
# path: /var/lib/etcd → path: /var/lib/etcd-restored

# Restore kube-apiserver
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Wait for pods to restart
kubectl get po -n kube-system
```

## Certificate Management

kubeadm-generated certificates expire after 1 year. CKA may ask you to check or renew them.

```bash
# Check certificate expiration dates
sudo kubeadm certs check-expiration

# Renew all certificates
sudo kubeadm certs renew all

# Renew a specific certificate
sudo kubeadm certs renew apiserver

# After renewal, restart control plane components
sudo systemctl restart kubelet
```

### Certificate Locations

| Certificate | Path |
|-------------|------|
| Kubernetes CA | `/etc/kubernetes/pki/ca.crt` |
| API server cert | `/etc/kubernetes/pki/apiserver.crt` |
| API server kubelet client | `/etc/kubernetes/pki/apiserver-kubelet-client.crt` |
| etcd CA | `/etc/kubernetes/pki/etcd/ca.crt` |
| etcd server cert | `/etc/kubernetes/pki/etcd/server.crt` |
| Front proxy CA | `/etc/kubernetes/pki/front-proxy-ca.crt` |
| Service account key | `/etc/kubernetes/pki/sa.key` |

### Inspect a Certificate

```bash
# Check expiry and SANs
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A2 "Validity"
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A1 "Alternative"
```

## Troubleshooting the Cluster

### kubelet Issues

```bash
# Check kubelet status
sudo systemctl status kubelet

# View kubelet logs
sudo journalctl -xeu kubelet --no-pager | tail -50

# Check kubelet config
cat /var/lib/kubelet/config.yaml

# Restart kubelet
sudo systemctl restart kubelet
```

### Static Pod Issues

```bash
# List static pod manifests
ls /etc/kubernetes/manifests/

# Check for syntax errors in manifests
# If a static pod isn't starting, check kubelet logs:
sudo journalctl -xeu kubelet | grep -i "error\|fail"

# Verify static pods are running
crictl pods --namespace kube-system
```

### Container Runtime Debugging with crictl

```bash
# List running containers
crictl ps

# List all containers (including stopped)
crictl ps -a

# List pods
crictl pods

# View container logs
crictl logs <container-id>

# Inspect a container
crictl inspect <container-id>

# Pull an image manually
crictl pull registry.k8s.io/pause:3.9

# Check runtime info
crictl info
```

### Network and DNS Troubleshooting

```bash
# Check CoreDNS pods
kubectl get po -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS from a pod
kubectl run tmp-dns --image=busybox:1.36 --rm -it --restart=Never -- nslookup kubernetes.default

# Check kube-proxy mode
kubectl get cm kube-proxy -n kube-system -o yaml | grep mode

# Check service endpoints
kubectl get endpoints <service-name>
```

### Node Not Ready

```bash
# Check node conditions
kubectl describe node <node-name> | grep -A5 "Conditions"

# Common causes:
# - kubelet not running → systemctl start kubelet
# - CNI not installed → install network plugin
# - Disk/memory pressure → check df -h and free -m
# - Certificate expired → kubeadm certs check-expiration
```

## CKA Exam Quick Reference

| Task | Key Commands |
|------|-------------|
| Bootstrap cluster | `kubeadm init --config=kubeadm.config` |
| Join worker node | `kubeadm join <ip>:6443 --token <t> --discovery-token-ca-cert-hash sha256:<h>` |
| Generate join token | `kubeadm token create --print-join-command` |
| Upgrade cluster | `kubeadm upgrade plan` → `kubeadm upgrade apply v1.x.y` |
| Backup etcd | `etcdctl snapshot save` with certs |
| Restore etcd | `etcdctl snapshot restore --data-dir=<new-dir>` |
| Check certs | `kubeadm certs check-expiration` |
| Renew certs | `kubeadm certs renew all` |
| Drain node | `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` |
| Uncordon node | `kubectl uncordon <node>` |
| Reset node | `kubeadm reset` |
| Debug kubelet | `journalctl -xeu kubelet` |
| Debug containers | `crictl ps -a`, `crictl logs <id>` |
| Static pod path | `/etc/kubernetes/manifests/` |
| Kubelet config | `/var/lib/kubelet/config.yaml` |

## How kubeadm init Works

1. Runs preflight checks (CPU, memory, swap, ports, kernel modules)
2. Downloads cluster container images from `registry.k8s.io`
3. Generates TLS certificates in `/etc/kubernetes/pki/`
4. Generates kubeconfig files in `/etc/kubernetes/`
5. Starts kubelet service
6. Creates static pod manifests in `/etc/kubernetes/manifests/`
7. Starts control plane components (apiserver, controller-manager, scheduler, etcd)
8. Installs CoreDNS and kube-proxy
9. Generates bootstrap token for worker node joins
