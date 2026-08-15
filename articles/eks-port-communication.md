# EKS Port Communication

How the EKS control plane and worker nodes communicate — ports, protocols, security group rules, and traffic flows.

## Port Communication Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    EKS Control Plane                        │
│                  (AWS Managed)                              │
│                                                             │
│  API Server (443), Controller Manager, Scheduler            │
└─────────────────────────────────────────────────────────────┘
                              │
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   OUTBOUND              OUTBOUND              OUTBOUND
   (Control →            (Kubelet)             (Kubelet)
    Workers)                                    Logs/Metrics
        │                     │                     │
        │ TCP 10250           │ TCP 10250           │ TCP 10250
        │ TCP 10251           │                     │
        │ TCP 10255           │                     │
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────────────────────────────────────────────────┐
│                    Worker Node                            │
│                                                           │
│  Kubelet (10250)                                          │
│  kube-proxy (10256)                                       │
│  Container Runtime                                        │
│  Pods (dynamic ports)                                     │
└───────────────────────────────────────────────────────────┘
        ▲                     ▲
        │                     │
   INBOUND               INBOUND
   (Kubelet API)        (Health Checks)
        │                     │
        │ TCP 10250           │ TCP 10250
        │                     │
        ▼                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    EKS Control Plane                        │
│                  (via Cross-Account ENIs)                   │
└─────────────────────────────────────────────────────────────┘
```

## Port Details

### Control Plane Ports

| Port | Protocol | Service | Purpose |
|:----:|:--------:|---------|---------|
| 443 | TCP | kube-apiserver | API server endpoint (kubectl, kubelet, controllers) |
| 10257 | TCP | kube-controller-manager | Health check endpoint |
| 10259 | TCP | kube-scheduler | Health check endpoint |

### Worker Node Ports

| Port | Protocol | Service | Purpose |
|:----:|:--------:|---------|---------|
| 10250 | TCP | kubelet | Kubelet API (logs, exec, port-forward, metrics) |
| 10255 | TCP | kubelet | Read-only port (deprecated, metrics) |
| 10256 | TCP | kube-proxy | Health check endpoint |
| 30000-32767 | TCP/UDP | NodePort | Services exposed as NodePort |

### Control Plane → Worker Nodes

| Port | Protocol | Direction | Purpose |
|:----:|:--------:|-----------|---------|
| 10250 | TCP | Control Plane → Node | Kubelet API — pod execution, logs, exec, port-forward |
| 443 | TCP | Control Plane → Node | Admission webhooks running on nodes |
| 1025-65535 | TCP | Control Plane → Node | Pod communication (webhooks, metrics-server) |

### Worker Nodes → Control Plane

| Port | Protocol | Purpose |
|:----:|:--------:|---------|
| 443 | TCP | API server communication (registration, heartbeat, pod updates) |

### Inter-Node Communication

| Port | Protocol | Purpose |
|:----:|:--------:|---------|
| All | TCP/UDP | Pod-to-pod communication (VPC CNI, routable IPs) |
| 10250 | TCP | kubelet-to-kubelet (metrics, health) |
| 10256 | TCP | kube-proxy health checks |
| 53 | TCP/UDP | CoreDNS (cluster DNS) |

## Security Group Rules

### EKS Cluster Security Group (Managed by EKS)

EKS creates a shared security group for control plane ENIs and worker nodes:

| Direction | Source/Destination | Ports | Purpose |
|-----------|-------------------|:-----:|---------|
| Inbound | Self (same SG) | All | Node-to-node + node-to-control-plane |
| Outbound | 0.0.0.0/0 | All | Control plane to nodes, nodes to internet |

### Worker Node Security Group (Minimum Required)

**Inbound:**

| Source | Ports | Purpose |
|--------|:-----:|---------|
| Cluster SG (control plane ENIs) | 10250 | Kubelet API (logs, exec, port-forward) |
| Cluster SG (control plane ENIs) | 443 | Webhooks |
| Cluster SG (control plane ENIs) | 1025-65535 | Pod communication |
| Worker node SG (self) | All | Inter-node pod traffic |
| VPC CIDR (or specific) | 30000-32767 | NodePort services (if used) |

**Outbound:**

| Destination | Ports | Purpose |
|-------------|:-----:|---------|
| Cluster SG (control plane) | 443 | API server |
| 0.0.0.0/0 | 443 | ECR, STS, S3, CloudWatch (HTTPS) |
| Worker node SG (self) | All | Inter-node pod communication |
| VPC DNS (x.x.x.2) | 53 | DNS resolution |

### Control Plane Security Group (What AWS Manages)

**Inbound:**

| Source | Ports | Purpose |
|--------|:-----:|---------|
| Worker node SG | 443 | kubelet → API server |
| Your CIDR (if public endpoint) | 443 | kubectl access |

**Outbound:**

| Destination | Ports | Purpose |
|-------------|:-----:|---------|
| Worker node SG | 10250 | API server → kubelet (logs, exec) |
| Worker node SG | 443 | API server → webhooks |
| Worker node SG | 1025-65535 | API server → pods |

## Key Communication Flows

### 1. Control Plane → Kubelet (TCP 10250)

Used for:
- Retrieving logs from running containers (`kubectl logs`)
- Executing commands in pods (`kubectl exec`)
- Port forwarding (`kubectl port-forward`)
- Attaching to containers (`kubectl attach`)
- Health checks and metrics collection

### 2. Kubelet → Control Plane (TCP 443)

Used for:
- Node registration and heartbeats (node lease)
- Pod status updates
- ConfigMap and Secret retrieval
- Watch for pod assignments

### 3. Control Plane → Webhooks (TCP 443 or custom)

Used for:
- Mutating admission webhooks (inject sidecars, defaults)
- Validating admission webhooks (policy enforcement)
- Conversion webhooks (CRD version conversion)
- Aggregated API servers

### 4. Inter-Node Communication

Used for:
- Pod-to-pod communication via VPC CNI (real VPC IPs)
- Service traffic between nodes (kube-proxy iptables/IPVS)
- CoreDNS queries (UDP/TCP 53)
- Metrics collection between nodes

## Verifying Connectivity

```sh
# Check if kubelet is reachable (from the node itself)
curl -sk https://localhost:10250/healthz

# Check if API server is reachable (from a node)
ENDPOINT=$(grep server /var/lib/kubelet/kubeconfig | awk '{print $2}')
curl -sk $ENDPOINT/healthz

# Check kube-proxy health
curl -s http://localhost:10256/healthz

# Verify the cluster security group
aws eks describe-cluster --name <cluster> \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text

# Check security group rules
CLUSTER_SG=$(aws eks describe-cluster --name <cluster> --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)
aws ec2 describe-security-groups --group-ids $CLUSTER_SG --output json | jq '.SecurityGroups[0].IpPermissions'

# Test port connectivity from a pod
kubectl run nettest --image=busybox --rm -it -- nc -zv <node-ip> 10250
```

## Troubleshooting Port Issues

| Symptom | Likely Blocked Port | Fix |
|---------|:-------------------:|-----|
| `kubectl logs` times out | 10250 (control plane → node) | Add inbound 10250 from cluster SG to node SG |
| `kubectl exec` hangs | 10250 (control plane → node) | Same as above |
| Node `NotReady` | 443 (node → control plane) | Add outbound 443 from node SG to cluster SG |
| Webhooks timeout | 443 or custom (control plane → pod) | Add inbound from cluster SG on webhook port |
| Pod can't reach API server | 443 (pod → control plane) | Check node SG outbound + VPC routing |
| DNS failures | 53 (pod → CoreDNS) | Check inter-node SG rules |
| NodePort not accessible | 30000-32767 | Add inbound from source to node SG |

## Important Notes

- The **control plane is managed by AWS** — you don't manage its security groups directly
- Worker nodes communicate with the control plane via **cross-account ENIs** placed in your VPC
- The VPC CNI handles pod-to-pod communication — pods get real VPC IPs, no overlay
- All communication is encrypted in transit (TLS for kubelet API, mTLS for etcd)
- **Port 10255** (kubelet read-only) is deprecated — use 10250 with auth instead
- **Port 10251/10252** (scheduler/controller-manager) are internal to the control plane — you never interact with them
- EKS Auto Mode manages all security groups automatically — no manual rules needed
