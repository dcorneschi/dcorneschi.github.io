## Complete Guide: MicroK8s, kubeadm, EKS, K3s, kind, k3d, minikube, GKE, AKS, and more

---

## Quick Comparison: Log Locations by Distribution

### Container Logs
Container specific details: exceptions, crashes, misconfigurations

- **MicroK8s:** `/var/snap/microk8s/common/var/log/containers/*.log`
- **kubeadm:** `/var/log/containers/*.log`
- **K3s:** `/var/log/containers/*.log`
- **kind:** `/var/log/containers/*.log` (access via `docker exec <node>`)
- **k3d:** `/var/log/containers/*.log` (access via `docker exec <node>`)
- **minikube:** `/var/log/containers/*.log` (inside VM)
- **EKS/GKE/AKS:** `/var/log/containers/*.log` (on worker nodes)

### Pod Logs
Container interactions within a pod, multi-container issues

- **MicroK8s:** `/var/snap/microk8s/common/var/log/pods/<namespace>_<pod>_<uid>/<container>/`
- **kubeadm:** `/var/log/pods/<namespace>_<pod>_<uid>/<container>/`
- **K3s:** `/var/log/pods/<namespace>_<pod>_<uid>/<container>/`
- **kind:** `/var/log/pods/<namespace>_<pod>_<uid>/<container>/` (via docker exec)
- **k3d:** `/var/log/pods/<namespace>_<pod>_<uid>/<container>/` (via docker exec)
- **minikube:** `/var/log/pods/<namespace>_<pod>_<uid>/<container>/` (inside VM)
- **EKS/GKE/AKS:** `/var/log/pods/<namespace>_<pod>_<uid>/<container>/` (on worker nodes)

### Kubelet Logs
Pod lifecycle, resources, communication, scheduling issues

- **MicroK8s:** `journalctl -u snap.microk8s.daemon-kubelet`
- **kubeadm:** `journalctl -u kubelet`
- **K3s:** `journalctl -u k3s` or `journalctl -u k3s-agent`
- **kind:** `docker exec <node> journalctl -u kubelet`
- **k3d:** `docker exec <node> journalctl -u kubelet`
- **minikube:** `minikube ssh` then `journalctl -u kubelet`
- **EKS:** `journalctl -u kubelet` (on worker nodes via SSH/SSM)
- **GKE:** Cloud Logging (Stackdriver) or `journalctl -u kubelet` on nodes
- **AKS:** Azure Monitor Logs or `journalctl -u kubelet` on nodes

### API Server Logs
API server operations and client interaction issues

- **MicroK8s:** `journalctl -u snap.microk8s.daemon-apiserver`
- **kubeadm:** `kubectl logs -n kube-system kube-apiserver-<node>`
- **K3s:** `journalctl -u k3s | grep apiserver`
- **kind:** `kubectl logs -n kube-system kube-apiserver-<node>`
- **k3d:** `kubectl logs -n kube-system kube-apiserver-<node>`
- **minikube:** `kubectl logs -n kube-system kube-apiserver-minikube`
- **EKS:** CloudWatch: `/aws/eks/<cluster>/cluster/api`
- **GKE:** Cloud Logging: `resource.type="k8s_cluster"`
- **AKS:** Azure Monitor: Control Plane Logs

### Controller Manager Logs
Issues with controllers like ReplicaSets or deployments

- **MicroK8s:** `journalctl -u snap.microk8s.daemon-controller-manager`
- **kubeadm:** `kubectl logs -n kube-system kube-controller-manager-<node>`
- **K3s:** `journalctl -u k3s | grep controller`
- **kind:** `kubectl logs -n kube-system kube-controller-manager-<node>`
- **k3d:** `kubectl logs -n kube-system kube-controller-manager-<node>`
- **minikube:** `kubectl logs -n kube-system kube-controller-manager-minikube`
- **EKS:** CloudWatch: `/aws/eks/<cluster>/cluster/controllerManager`
- **GKE:** Cloud Logging (managed)
- **AKS:** Azure Monitor: Control Plane Logs

### Scheduler Logs
Pod scheduling issues, resource constraints, affinity rules

- **MicroK8s:** `journalctl -u snap.microk8s.daemon-scheduler`
- **kubeadm:** `kubectl logs -n kube-system kube-scheduler-<node>`
- **K3s:** `journalctl -u k3s | grep scheduler`
- **kind:** `kubectl logs -n kube-system kube-scheduler-<node>`
- **k3d:** `kubectl logs -n kube-system kube-scheduler-<node>`
- **minikube:** `kubectl logs -n kube-system kube-scheduler-minikube`
- **EKS:** CloudWatch: `/aws/eks/<cluster>/cluster/scheduler`
- **GKE:** Cloud Logging (managed)
- **AKS:** Azure Monitor: Control Plane Logs

### etcd Logs
Data consistency or leader elections issues

- **MicroK8s:** `journalctl -u snap.microk8s.daemon-etcd`
- **kubeadm:** `kubectl logs -n kube-system etcd-<node>`
- **K3s:** `journalctl -u k3s | grep etcd` (embedded)
- **kind:** `kubectl logs -n kube-system etcd-<node>`
- **k3d:** `kubectl logs -n kube-system etcd-<node>`
- **minikube:** `kubectl logs -n kube-system etcd-minikube`
- **EKS:** Managed by AWS (not directly accessible)
- **GKE:** Managed by Google (not directly accessible)
- **AKS:** Managed by Azure (not directly accessible)

### System Logs
Node-related issues, hardware failures, resource problems

- **MicroK8s:** `journalctl -k`
- **kubeadm:** `journalctl -k`
- **K3s:** `journalctl -k`
- **kind:** `docker exec <node> journalctl -k`
- **k3d:** `docker exec <node> journalctl -k`
- **minikube:** `minikube ssh` then `journalctl -k`
- **EKS/GKE/AKS:** `journalctl -k` (on worker nodes)

### Container Runtime Logs
Container runtime (containerd/docker) issues

- **MicroK8s:** `journalctl -u snap.microk8s.daemon-containerd`
- **kubeadm:** `journalctl -u containerd` or `journalctl -u docker`
- **K3s:** `journalctl -u k3s` (embedded)
- **kind:** `docker exec <node> journalctl -u containerd`
- **k3d:** `docker exec <node> journalctl -u containerd`
- **minikube:** `minikube ssh` then `journalctl -u docker` or `journalctl -u containerd`
- **EKS/GKE/AKS:** `journalctl -u containerd` (on worker nodes)

---

## Quick Access Commands by Distribution

### Container & Pod Logs (Universal)
```bash
# View container logs
kubectl logs <pod-name> -c <container-name> -f

# View all containers in pod
kubectl logs <pod-name> --all-containers=true -f

# Previous container logs (after crash)
kubectl logs <pod-name> --previous

# With timestamps
kubectl logs <pod-name> --timestamps=true

# Last 100 lines
kubectl logs <pod-name> --tail=100

# Since 1 hour ago
kubectl logs <pod-name> --since=1h
```

### Kubelet Logs

| Distribution | Command |
|--------------|---------|
| **MicroK8s** | `sudo journalctl -u snap.microk8s.daemon-kubelet -f` |
| **kubeadm** | `sudo journalctl -u kubelet -f` |
| **K3s** | `sudo journalctl -u k3s -f` or `sudo journalctl -u k3s-agent -f` |
| **kind** | `docker exec <node-name> journalctl -u kubelet -f` |
| **k3d** | `docker exec <node-name> journalctl -u kubelet -f` |
| **minikube** | `minikube ssh` then `sudo journalctl -u kubelet -f` |
| **EKS** | SSH/SSM to node: `sudo journalctl -u kubelet -f` |
| **GKE** | SSH to node: `sudo journalctl -u kubelet -f` or Cloud Logging |
| **AKS** | SSH to node: `sudo journalctl -u kubelet -f` or Azure Monitor |

### Control Plane Logs

| Distribution | API Server | Controller Manager | Scheduler |
|--------------|------------|-------------------|-----------|
| **MicroK8s** | `journalctl -u snap.microk8s.daemon-apiserver -f` | `journalctl -u snap.microk8s.daemon-controller-manager -f` | `journalctl -u snap.microk8s.daemon-scheduler -f` |
| **kubeadm** | `kubectl logs -n kube-system kube-apiserver-<node>` | `kubectl logs -n kube-system kube-controller-manager-<node>` | `kubectl logs -n kube-system kube-scheduler-<node>` |
| **K3s** | `journalctl -u k3s \| grep apiserver` | `journalctl -u k3s \| grep controller` | `journalctl -u k3s \| grep scheduler` |
| **kind** | `kubectl logs -n kube-system kube-apiserver-<node>` | `kubectl logs -n kube-system kube-controller-manager-<node>` | `kubectl logs -n kube-system kube-scheduler-<node>` |
| **k3d** | `kubectl logs -n kube-system kube-apiserver-<node>` | `kubectl logs -n kube-system kube-controller-manager-<node>` | `kubectl logs -n kube-system kube-scheduler-<node>` |
| **minikube** | `kubectl logs -n kube-system kube-apiserver-minikube` | `kubectl logs -n kube-system kube-controller-manager-minikube` | `kubectl logs -n kube-system kube-scheduler-minikube` |
| **EKS** | CloudWatch: `/aws/eks/<cluster>/cluster/api` | CloudWatch: `/aws/eks/<cluster>/cluster/controllerManager` | CloudWatch: `/aws/eks/<cluster>/cluster/scheduler` |
| **GKE** | Cloud Logging: `resource.type="k8s_cluster"` | Cloud Logging (managed) | Cloud Logging (managed) |
| **AKS** | Azure Monitor: Control Plane Logs | Azure Monitor: Control Plane Logs | Azure Monitor: Control Plane Logs |

---

## Distribution-Specific Features

| Feature | MicroK8s | kubeadm | K3s | kind | k3d | minikube | EKS | GKE | AKS |
|---------|----------|---------|-----|------|-----|----------|-----|-----|-----|
| **Installation** | Snap package | kubeadm tool | Single binary | Docker containers | Docker + k3s | VM (various drivers) | AWS managed | GCP managed | Azure managed |
| **Control Plane** | Snap services | Static pods | Embedded process | Static pods in containers | Static pods in containers | Static pods in VM | AWS managed | GCP managed | Azure managed |
| **Nodes** | Host/VM | Host/VM | Host/VM | Docker containers | Docker containers | VM | EC2 instances | GCE instances | Azure VMs |
| **etcd** | Snap service | Static pod | Embedded/external | Static pod in container | Static pod in container | Static pod in VM | AWS managed | GCP managed | Azure managed |
| **Runtime** | Containerd | Containerd/Docker | Containerd | Containerd | Containerd | Docker/Containerd | Containerd | Containerd | Containerd |
| **Log Access** | journalctl (snap) | journalctl + kubectl | journalctl (unified) | docker exec + kubectl | docker exec + kubectl | minikube ssh + kubectl | CloudWatch + kubectl | Cloud Logging | Azure Monitor |
| **Best For** | Dev, edge, IoT | Production, on-prem | Edge, IoT, lightweight | Local dev, CI/CD | Local dev, multi-cluster | Local dev, learning | Production (AWS) | Production (GCP) | Production (Azure) |
| **Multi-Node** | Yes | Yes | Yes | Yes | Yes | Yes (with effort) | Yes | Yes | Yes |
| **Resource Usage** | Low-Medium | Medium | Very Low | Low | Very Low | Medium | N/A | N/A | N/A |

---

## Docker-Based Distributions (kind & k3d)

### kind (Kubernetes in Docker)

**Access Node Shell:**
```bash
# List nodes (they're Docker containers)
docker ps --filter name=kind

# Access control plane node
docker exec -it kind-<cluster>-control-plane bash

# Access worker node
docker exec -it kind-<cluster>-worker bash
```

**View Logs:**
```bash
# Container logs (standard kubectl)
kubectl logs <pod-name> -c <container-name>

# Kubelet logs (inside node container)
docker exec kind-<cluster>-control-plane journalctl -u kubelet -f

# API server logs
kubectl logs -n kube-system kube-apiserver-kind-<cluster>-control-plane

# All control plane logs
kubectl logs -n kube-system -l tier=control-plane --all-containers=true

# Node system logs (inside node container)
docker exec kind-<cluster>-control-plane journalctl -k

# Docker logs of the node container itself
docker logs kind-<cluster>-control-plane

# Access logs on node filesystem
docker exec kind-<cluster>-control-plane cat /var/log/containers/<pod>_<ns>_<container>-<id>.log
```

**kind-Specific Commands:**
```bash
# Export all logs from cluster (very useful!)
kind export logs ./kind-logs

# This exports:
# - Container logs
# - Pod logs  
# - Control plane logs
# - Node system logs
# - Kubelet logs

# Get cluster info
kind get clusters
kind get nodes --name <cluster-name>
```

### k3d (k3s in Docker)

**Access Node Shell:**
```bash
# List nodes (they're Docker containers running k3s)
docker ps --filter name=k3d

# Access server node
docker exec -it k3d-<cluster-name>-server-0 sh

# Access agent node
docker exec -it k3d-<cluster-name>-agent-0 sh
```

**View Logs:**
```bash
# Container logs (standard kubectl)
kubectl logs <pod-name> -c <container-name>

# K3s server logs (all control plane components embedded)
docker logs k3d-<cluster-name>-server-0

# Real-time k3s logs
docker logs -f k3d-<cluster-name>-server-0

# Filter for specific components
docker logs k3d-<cluster-name>-server-0 2>&1 | grep apiserver
docker logs k3d-<cluster-name>-server-0 2>&1 | grep scheduler
docker logs k3d-<cluster-name>-server-0 2>&1 | grep controller

# Access logs on node filesystem
docker exec k3d-<cluster-name>-server-0 cat /var/log/containers/<pod>_<ns>_<container>-<id>.log

# Pod logs directory
docker exec k3d-<cluster-name>-server-0 ls /var/log/pods/
```

**k3d-Specific Commands:**
```bash
# List clusters
k3d cluster list

# List nodes
k3d node list

# Get specific node logs
docker logs k3d-<cluster-name>-server-0 --tail 100 --follow

# Stop/start cluster (preserves logs)
k3d cluster stop <cluster-name>
k3d cluster start <cluster-name>
```

---

## minikube

**Access Node:**
```bash
# SSH into minikube VM
minikube ssh

# Or use docker driver
docker exec -it minikube bash
```

**View Logs:**
```bash
# Container logs (standard kubectl)
kubectl logs <pod-name> -c <container-name>

# Kubelet logs (inside minikube VM)
minikube ssh
sudo journalctl -u kubelet -f
exit

# API server logs
kubectl logs -n kube-system kube-apiserver-minikube

# All control plane logs
kubectl logs -n kube-system -l tier=control-plane --all-containers=true

# System logs (inside VM)
minikube ssh
sudo journalctl -k
exit

# Docker/containerd logs (inside VM)
minikube ssh
sudo journalctl -u docker -f  # if using docker driver
sudo journalctl -u containerd -f  # if using containerd
exit

# Access container logs on filesystem
minikube ssh
sudo cat /var/log/containers/<pod>_<ns>_<container>-<id>.log
exit
```

**minikube-Specific Commands:**
```bash
# View minikube system logs
minikube logs

# Follow logs in real-time
minikube logs -f

# Last 100 lines
minikube logs --length=100

# Get node status
minikube status

# SSH into minikube VM
minikube ssh

# View addons logs
minikube addons list
kubectl logs -n kube-system <addon-pod>

# Check minikube version and driver
minikube version
minikube profile list
```

---

## Cloud Provider Specific Commands

### AWS EKS

**Enable Control Plane Logging:**
```bash
aws eks update-cluster-config \
  --name <cluster-name> \
  --logging '{"clusterLogging":[{
    "types":["api","audit","authenticator","controllerManager","scheduler"],
    "enabled":true
  }]}'
```

**View CloudWatch Logs:**
```bash
# Tail all cluster logs
aws logs tail /aws/eks/<cluster-name>/cluster --follow

# API server logs
aws logs tail /aws/eks/<cluster-name>/cluster/api --follow

# Audit logs
aws logs tail /aws/eks/<cluster-name>/cluster/audit --follow

# Filter logs
aws logs filter-log-events \
  --log-group-name /aws/eks/<cluster-name>/cluster \
  --filter-pattern "error"

# List log streams
aws logs describe-log-streams \
  --log-group-name /aws/eks/<cluster-name>/cluster
```

**Worker Node Logs:**
```bash
# SSH via SSM
aws ssm start-session --target <instance-id>

# View kubelet logs
sudo journalctl -u kubelet -f

# View container runtime logs
sudo journalctl -u containerd -f
```

### Google GKE

**Enable Cloud Logging:**
```bash
# Enable logging when creating cluster
gcloud container clusters create <cluster-name> \
  --enable-cloud-logging \
  --logging=SYSTEM,WORKLOAD

# Update existing cluster
gcloud container clusters update <cluster-name> \
  --enable-cloud-logging \
  --logging=SYSTEM,WORKLOAD
```

**View Logs:**
```bash
# View cluster logs
gcloud logging read "resource.type=k8s_cluster" --limit 50

# View pod logs
gcloud logging read "resource.type=k8s_pod" --limit 50

# View node logs
gcloud logging read "resource.type=gce_instance" --limit 50

# Filter by severity
gcloud logging read "severity>=ERROR" --limit 50

# Real-time logs
gcloud logging tail "resource.type=k8s_cluster"
```

**SSH to Node:**
```bash
# List nodes
kubectl get nodes

# SSH to node
gcloud compute ssh <node-name>

# View kubelet logs
sudo journalctl -u kubelet -f
```

### Azure AKS

**Enable Monitoring:**
```bash
# Enable Container Insights
az aks enable-addons \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --addons monitoring

# Enable control plane logs
az aks update \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --enable-azure-monitor-metrics
```

**View Logs:**
```bash
# View logs in Azure Monitor
az monitor log-analytics query \
  --workspace <workspace-id> \
  --analytics-query "KubePodInventory | limit 50"

# Control plane logs
az monitor log-analytics query \
  --workspace <workspace-id> \
  --analytics-query "AzureDiagnostics | where Category == 'kube-apiserver'"

# Container logs
az monitor log-analytics query \
  --workspace <workspace-id> \
  --analytics-query "ContainerLog | limit 50"
```

**SSH to Node:**
```bash
# Create debug pod
kubectl debug node/<node-name> -it --image=mcr.microsoft.com/dotnet/runtime-deps:6.0

# View kubelet logs
chroot /host
journalctl -u kubelet -f
```

---

## Log Aggregation Solutions

### EFK Stack (Elasticsearch, Fluentd, Kibana)
```bash
# Deploy Fluentd DaemonSet
kubectl apply -f https://raw.githubusercontent.com/fluent/fluentd-kubernetes-daemonset/master/fluentd-daemonset-elasticsearch.yaml

# Deploy Elasticsearch
helm repo add elastic https://helm.elastic.co
helm install elasticsearch elastic/elasticsearch

# Deploy Kibana
helm install kibana elastic/kibana
```

### Loki + Promtail + Grafana
```bash
# Install Loki stack
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --set grafana.enabled=true \
  --set prometheus.enabled=true \
  --set promtail.enabled=true
```

### Fluent Bit
```bash
# Install Fluent Bit
helm repo add fluent https://fluent.github.io/helm-charts
helm install fluent-bit fluent/fluent-bit
```

---

## Troubleshooting Tips

### Check Log Files Exist
```bash
# List container logs
ls -lh /var/log/containers/

# List pod logs
ls -lh /var/log/pods/

# Check disk space
df -h /var/log
```

### Find Crashed Pods
```bash
# Get all non-running pods
kubectl get pods -A | grep -v Running

# Get recent events
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# Describe pod
kubectl describe pod <pod-name> -n <namespace>

# Get previous logs
kubectl logs <pod-name> --previous
```

### Access Logs When kubectl Unavailable

**For kind/k3d:**
```bash
docker exec <node-name> crictl ps -a
docker exec <node-name> crictl logs <container-id>
```

**For minikube:**
```bash
minikube ssh
sudo crictl ps -a
sudo crictl logs <container-id>
```

**For cloud providers:**
```bash
# Use cloud provider's logging service
# AWS: CloudWatch
# GCP: Cloud Logging
# Azure: Azure Monitor
```

### Debug Container Runtime Issues
```bash
# Check containerd
sudo crictl ps -a
sudo crictl logs <container-id>
sudo crictl inspect <container-id>

# Check containerd service
sudo systemctl status containerd
sudo journalctl -u containerd -f
```

---

## Quick Reference Summary

### Universal kubectl Commands
```bash
kubectl logs <pod> -c <container> -f              # Follow logs
kubectl logs <pod> --all-containers=true          # All containers
kubectl logs <pod> --previous                     # Previous container (after crash)
kubectl logs <pod> --since=1h                     # Last hour
kubectl logs <pod> --tail=100                     # Last 100 lines
kubectl logs <pod> --timestamps=true              # Include timestamps
kubectl get events -A --sort-by='.lastTimestamp'  # Recent events
kubectl describe pod <pod>                        # Pod details
```

### Log File Locations (Standard Across Most Distributions)
```bash
# Container logs (symlinks)
/var/log/containers/<pod>_<namespace>_<container>-<container-id>.log

# Pod logs (organized by pod)
/var/log/pods/<namespace>_<pod-name>_<pod-uid>/<container-name>/

# Kubelet logs (systemd-based systems)
journalctl -u kubelet

# Container runtime logs
/var/lib/docker/containers/<container-id>/<container-id>-json.log  # Docker
/var/lib/containerd/...  # Containerd
```

### Distribution Quick Identification
```bash
# Check which distribution you're using
kubectl version --short
kubectl get nodes -o wide

# MicroK8s
microk8s status

# K3s
k3s --version

# kind
kind get clusters

# k3d
k3d cluster list

# minikube
minikube status

# EKS
aws eks describe-cluster --name <cluster>

# GKE
gcloud container clusters list

# AKS
az aks list
```

---

## Best Practices

1. **Enable logging early** - Configure control plane logging when creating clusters
2. **Use log aggregation** - Implement EFK, Loki, or cloud-native solutions for production
3. **Set retention policies** - Configure log rotation and retention to manage disk space
4. **Monitor disk usage** - Logs can fill up disk quickly, especially on worker nodes
5. **Use structured logging** - JSON format makes parsing and searching easier
6. **Include timestamps** - Always use `--timestamps=true` for debugging
7. **Check previous logs** - Use `--previous` flag to see logs from crashed containers
8. **Use labels and selectors** - Filter logs by labels for easier troubleshooting
9. **Enable audit logging** - Critical for security and compliance in production
10. **Test log access** - Verify you can access logs before you need them in an emergency
