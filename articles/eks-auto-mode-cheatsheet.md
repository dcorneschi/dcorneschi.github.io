# EKS Auto Mode Cheatsheet

EKS Auto Mode manages Karpenter, CSI, CoreDNS, kube-proxy, and load balancing in a hidden control plane. You won't see controller pods — only the CRDs and resources they create.

## Create an EKS Auto Mode Cluster

```bash
# eksctl — single command
eksctl create cluster --name=my-cluster --enable-auto-mode

# AWS CLI — create then enable Auto Mode
aws eks create-cluster \
  --name my-cluster \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/eks-cluster-role \
  --resources-vpc-config subnetIds=subnet-xxx,subnet-yyy,securityGroupIds=sg-zzz \
  --compute-config '{
    "enabled": true,
    "nodePools": ["general-purpose", "system"],
    "nodeRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/eks-auto-node-role"
  }' \
  --kubernetes-network-config '{"elasticLoadBalancing":{"enabled":true}}' \
  --storage-config '{"blockStorage":{"enabled":true}}'

# Enable Auto Mode on an existing cluster
aws eks update-cluster-config \
  --name my-cluster \
  --compute-config '{
    "enabled": true,
    "nodePools": ["general-purpose", "system"],
    "nodeRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/eks-auto-node-role"
  }' \
  --kubernetes-network-config '{"elasticLoadBalancing":{"enabled":true}}' \
  --storage-config '{"blockStorage":{"enabled":true}}'

# Update kubeconfig to interact with the cluster
aws eks update-kubeconfig --name my-cluster --region us-east-1
```

> **Migration note:** Existing clusters must be on Karpenter v1.1+ before enabling Auto Mode to avoid NodePool/NodeClaim API conflicts.

### Terraform

```hcl
resource "aws_eks_cluster" "auto" {
  name     = "my-cluster"
  role_arn = aws_iam_role.cluster.arn

  vpc_config {
    subnet_ids = var.subnet_ids
  }

  # Enable Auto Mode compute
  compute_config {
    enabled       = true
    node_pools    = ["general-purpose", "system"]
    node_role_arn = aws_iam_role.auto_node.arn
  }

  # Native load balancing (no AWS LBC Helm chart needed)
  kubernetes_network_config {
    elastic_load_balancing {
      enabled = true
    }
  }

  # Native EBS storage (no EBS CSI driver Helm chart needed)
  storage_config {
    block_storage {
      enabled = true
    }
  }
}
```

> No Node Groups to define. No Helm charts for LBC or EBS CSI. Just the cluster resource with capability flags.

### eksctl Config File

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1
  version: "1.30"

autoModeConfig:
  enabled: true
```

### AWS Console

1. Open EKS Console → **Add cluster** → **Create**
2. Under **Configuration options**, select **Custom configuration**
3. Toggle **Use EKS Auto Mode** to ON
4. Configure VPC, subnets, and cluster IAM role
5. Review and create

### Migration from Standard to Auto Mode

```bash
# 1. Enable Auto Mode on existing cluster
aws eks update-cluster-config --name my-cluster --region us-east-1 \
  --compute-config '{
    "enabled": true,
    "nodePools": ["general-purpose", "system"],
    "nodeRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/eks-auto-node-role"
  }' \
  --kubernetes-network-config '{"elasticLoadBalancing":{"enabled":true}}' \
  --storage-config '{"blockStorage":{"enabled":true}}'

# 2. Wait for Auto Mode to be active
aws eks describe-cluster --name my-cluster --query "cluster.computeConfig"

# 3. Taint old node groups (shift workloads to Auto Mode nodes)
kubectl taint nodes -l eks.amazonaws.com/nodegroup=old-ng migration=true:PreferNoSchedule

# 4. Verify workloads running on Auto Mode nodes
kubectl get pods -A -o wide | grep -v <old-node-prefix>

# 5. Scale down old node groups
aws eks update-nodegroup-config --cluster-name my-cluster --nodegroup-name old-ng \
  --scaling-config minSize=0,desiredSize=0,maxSize=0

# 6. Delete old node groups
aws eks delete-nodegroup --cluster-name my-cluster --nodegroup-name old-ng
```

> Existing managed node groups continue to work alongside Auto Mode nodes. You can migrate workloads gradually.

## Architecture: NodePool → NodeClass → NodeClaim → Node

```
┌─────────────────────────────────────────────────────────────────────┐
│                        EKS Auto Mode                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐         ┌──────────────────┐                      │
│  │  NodePool    │────────▶│   NodeClass      │                      │
│  │              │  refs   │   (default)      │                      │
│  │ - constraints│         │                  │                      │
│  │ - limits     │         │ - role (IAM)     │                      │
│  │ - disruption │         │ - subnets        │                      │
│  │              │         │ - security groups│                      │
│  └──────┬───────┘         │ - storage config │                      │
│         │                 │ - instance prof. │                      │
│         │ creates         └──────────────────┘                      │
│         ▼                                                           │
│  ┌──────────────┐         ┌──────────────────┐                      │
│  │  NodeClaim   │────────▶│   EC2 Instance   │                      │
│  │              │launches │   (Node)         │                      │
│  │ - instance   │         │                  │                      │
│  │   type       │         │ - joins cluster  │                      │
│  │ - zone       │         │ - runs pods      │                      │
│  │ - capacity   │         │                  │                      │
│  └──────────────┘         └──────────────────┘                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Flow:
  Pod Pending → NodePool matches → NodeClaim created → NodeClass provides
  config (subnets, SG, IAM) → EC2 launched → Node joins → Pod scheduled
```

## Compute (Karpenter)

### Built-in NodePools

EKS Auto Mode provides two built-in NodePools:

| NodePool | Purpose | Architectures | Notes |
|----------|---------|---------------|-------|
| `general-purpose` | General workloads | amd64 only | Default for most pods |
| `system` | Cluster-critical add-ons | amd64, arm64 | Has `CriticalAddonsOnly` taint; CoreDNS runs here |

Both built-in NodePools:
- Use the `default` NodeClass
- Use only on-demand EC2 capacity
- Use C, M, and R instance families (generation 5+)

**Default disruption settings (general-purpose):**

| Setting | Value | Meaning |
|---------|-------|---------|
| `consolidationPolicy` | `WhenEmptyOrUnderutilized` | Remove/replace nodes that are empty or underused |
| `consolidateAfter` | `30s` | Wait 30s after a node becomes empty/underutilized before acting |
| `budgets.nodes` | `10%` | Disrupt at most 10% of nodes at a time |
| `expireAfter` | `336h` (14 days) | Force node replacement after 14 days |
| `terminationGracePeriod` | `24h` | Allow up to 24h for pods to gracefully terminate during disruption |

### Consolidation Methods

EKS Auto Mode continuously evaluates the cluster for cost optimization using two strategies:

| Method | When Applied | What Happens |
|--------|-------------|-------------|
| **Node deletion** | All pods on a node can fit on available capacity of other nodes | Node is drained and terminated |
| **Node replacement** | Pods can be redistributed across existing nodes + one cheaper replacement | Old node replaced with a lower-cost instance |

### Node Disruption Budgets (Maintenance Windows)

Beyond `budgets.nodes` (percentage of nodes disrupted at once), you can control **when** nodes are updated using `disruption.budgets[].schedule`:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: production
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
    budgets:
      # During business hours: no disruptions
      - nodes: "0"
        schedule: "0 9 * * 1-5"   # Mon-Fri 9am
        duration: 8h              # Until 5pm
      # Outside business hours: up to 10% at a time
      - nodes: 10%
  template:
    spec:
      expireAfter: 336h
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
```

> **Note:** If PDBs or NodePool disruption controls prevent a node from being replaced before the 21-day maximum node lifetime, the node is disrupted **regardless**. A misconfigured PDB cannot indefinitely block security patches.

### Automatic Node Health Repair

EKS Auto Mode includes built-in health monitoring that can automatically repair nodes for certain failure modes:

- Unresponsive kubelet
- Exhausted process IDs (PID pressure)
- Other failure modes identified by AWS from operating EKS at scale

Unhealthy nodes are replaced (not repaired in-place) to minimize disruption. Health issues are reported via Kubernetes events and node conditions.

```bash
# Check node conditions for health issues
kubectl get nodes -o wide
kubectl describe node <node-name> | grep -A5 Conditions

# Check for node health events
kubectl get events --field-selector reason=NodeUnhealthy --sort-by='.lastTimestamp'
```

> Disabling all built-in NodePools deletes the `default` NodeClass. You'd need a custom NodeClass.
> Removing a NodePool drains and terminates all its nodes.

```bash
# List node pools
kubectl get nodepools

# Describe a node pool
kubectl describe nodepool general-purpose

# List node classes (instance profile, subnets, security groups config)
kubectl get nodeclasses

# Describe node class (check readiness: instance profile, subnets, SGs)
kubectl describe nodeclass default

# List node claims (pending/active node requests)
kubectl get nodeclaims

# List node claims with extra details
kubectl get nodeclaims -o wide

# Describe a node claim (see why provisioning is stuck)
kubectl describe nodeclaim <name>

# Watch node claims in real time
kubectl get nodeclaims -w

# Delete a stuck nodeclaim (forces re-provisioning)
kubectl delete nodeclaim <name>
```

### NodeClaim Lifecycle

A NodeClaim represents a request for a single node. It goes through these states:

| READY | Meaning |
|-------|---------|
| `Unknown` | Instance launched, waiting to join the cluster |
| `True` | Node joined and is Ready |
| `False` | Failed to provision or join |

**Common reasons a NodeClaim stays `Unknown`:**
- Unsupported instance type (e.g. `t` family — burstable not supported in Auto Mode)
- Network connectivity issue (node can't reach API server)
- IAM/instance profile issue

**What `kubectl get nodeclaims` columns mean:**

| Column | Description |
|--------|-------------|
| NAME | NodeClaim name (prefixed by nodepool name) |
| TYPE | EC2 instance type selected (e.g. `c6a.large`) |
| CAPACITY | `on-demand` or `spot` |
| ZONE | Availability zone |
| NODE | Instance ID (empty until launched) |
| READY | `True`, `False`, or `Unknown` |

```bash
# Check nodes provisioned by Auto Mode
kubectl get nodes -L karpenter.sh/nodepool

# See node capacity and allocatable resources
kubectl describe nodes

# Show which nodepool each node belongs to
kubectl get nodes -L karpenter.sh/nodepool,type

# Show pods with their node assignment
kubectl get pods -o wide

# Get pods by nodepool label
kubectl get pods -l type=app
kubectl get pods -l type=core

# Get pods on a specific nodepool's nodes
kubectl get pods -o wide --field-selector spec.nodeName=$(kubectl get nodes -l karpenter.sh/nodepool=app -o jsonpath='{.items[0].metadata.name}')
```

### Manage Built-in NodePools (AWS CLI)

```bash
# Enable both built-in NodePools (requires all three config flags)
aws eks update-cluster-config \
  --name demo-cluster-auto --region us-east-1 \
  --compute-config '{
    "enabled": true,
    "nodePools": ["general-purpose", "system"],
    "nodeRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/demo-cluster-auto-auto-node-role"
  }' \
  --kubernetes-network-config '{"elasticLoadBalancing":{"enabled":true}}' \
  --storage-config '{"blockStorage":{"enabled":true}}'

# Disable both (use custom NodePools only)
aws eks update-cluster-config \
  --name demo-cluster-auto --region us-east-1 \
  --compute-config '{
    "enabled": true,
    "nodePools": []
  }' \
  --kubernetes-network-config '{"elasticLoadBalancing":{"enabled":true}}' \
  --storage-config '{"blockStorage":{"enabled":true}}'

# Enable only system pool (for cluster add-ons)
aws eks update-cluster-config \
  --name demo-cluster-auto --region us-east-1 \
  --compute-config '{
    "enabled": true,
    "nodePools": ["system"],
    "nodeRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/demo-cluster-auto-auto-node-role"
  }' \
  --kubernetes-network-config '{"elasticLoadBalancing":{"enabled":true}}' \
  --storage-config '{"blockStorage":{"enabled":true}}'
```

### Custom NodePool & NodeClass

When built-in NodePools don't fit your needs (spot instances, specific instance families, GPU, custom taints), create custom ones.

### Do I Need a Custom NodeClass?

> **The `default` NodeClass is immutable** — it's managed by EKS Auto Mode and any edits get reconciled back.
> To change storage, subnets, security groups, or IAM role, you must create a custom NodeClass.

**No custom NodeClass needed** — use the `default` when you only want to control:
- Instance types and families
- Capacity type (spot/on-demand)
- Taints and labels
- CPU/memory limits

Your custom NodePools just reference `nodeClassRef: name: default` and that's it.

**Yes, create a custom NodeClass** when you need:
- Different ephemeral storage size/IOPS (default is 80Gi)
- Different subnets (isolate workloads to specific AZs)
- Different security groups
- A different IAM role for nodes
- Custom network policy settings

**Custom NodeClass** — defines EC2 launch config (subnets, security groups, IAM role, storage):

```yaml
apiVersion: eks.amazonaws.com/v1
kind: NodeClass
metadata:
  name: custom
spec:
  ephemeralStorage:
    size: "20Gi"    # Range: 1-59000Gi (default is 80Gi)
    iops: 3000      # Range: 3000-16000
    throughput: 125  # Range: 125-1000
  role: demo-cluster-auto-auto-node-role
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: demo-cluster-auto
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: demo-cluster-auto
```

**Custom NodePool** — defines scheduling constraints, instance types, capacity type:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-pool
spec:
  template:
    metadata:
      labels:
        workload-type: batch
    spec:
      taints:
        - key: workload-type
          value: batch
          effect: NoSchedule
      nodeClassRef:
        name: custom
        kind: NodeClass
        group: eks.amazonaws.com
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: eks.amazonaws.com/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: eks.amazonaws.com/instance-cpu
          operator: In
          values: ["4", "8", "16"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
  limits:
    cpu: "1000"
```

**Deploy to custom NodePool** — use `nodeAffinity` + `tolerations` to target it:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-job
spec:
  replicas: 2
  selector:
    matchLabels:
      app: batch-job
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      tolerations:
        - key: workload-type
          operator: Equal
          value: batch
          effect: NoSchedule
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: workload-type
                    operator: In
                    values: ["batch"]
                  - key: karpenter.sh/capacity-type
                    operator: In
                    values: ["spot"]
      containers:
        - name: app
          image: nginx:latest
```

**Apply and verify:**

```bash
kubectl apply -f nodeclass.yaml
kubectl apply -f nodepool.yaml
kubectl apply -f deployment.yaml

# Check pods land on the right nodes
kubectl get pods -o wide
kubectl get nodes -L workload-type,karpenter.sh/capacity-type
```

### NodePool Requirement Keys Reference

| Key | Description | Example Values |
|-----|-------------|----------------|
| `karpenter.sh/capacity-type` | On-demand or spot | `on-demand`, `spot` |
| `eks.amazonaws.com/instance-category` | Instance category | `t`, `c`, `m`, `r`, `g`, `p` |
| `eks.amazonaws.com/instance-family` | Instance family | `t3a`, `c6i`, `m6i`, `r5a` |
| `eks.amazonaws.com/instance-cpu` | vCPU count | `4`, `8`, `16`, `32` |
| `eks.amazonaws.com/instance-generation` | Minimum generation | `5`, `6`, `7` |
| `kubernetes.io/arch` | CPU architecture | `amd64`, `arm64` |
| `kubernetes.io/os` | Operating system | `linux` |

### Best Practices

- Separate NodePools per workload type (general, GPU, batch, memory-optimized)
- Use taints + labels to control pod placement
- Limit instance types for predictability
- Use spot for non-critical / batch workloads
- Monitor scaling: `kubectl get events -A` and CloudWatch metrics
- Use Karpenter consolidation to optimize idle nodes
- **Always set `resources.requests` and `resources.limits`** — Karpenter uses requests for bin-packing and instance selection. Missing requests = poor cost optimization
- **Use `eks.amazonaws.com/compute-type: auto` nodeSelector** in mixed clusters (Auto Mode + Managed Node Groups) to ensure pods land on Auto Mode nodes
- **Apply AWS cost allocation tags** on pods and namespaces for cost visibility in Cost Explorer
- **Use `securityContext`** — nodes enforce SELinux, so set `runAsUser`, `runAsGroup`, and `allowPrivilegeEscalation: false`

**Minimal pod spec for Auto Mode (mixed cluster):**

```yaml
spec:
  nodeSelector:
    eks.amazonaws.com/compute-type: auto
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
    - name: app
      image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
      resources:
        requests:
          cpu: "500m"
          memory: "256Mi"
        limits:
          cpu: "1"
          memory: "512Mi"
      securityContext:
        allowPrivilegeEscalation: false
```

## Storage (EBS CSI)

### Local Instance Storage

For instance types with local NVMe storage (e.g. `c5d`, `m5d`, `i3`, `r5d`), EKS Auto Mode **automatically configures** the local drives for:
- Container images
- Pod ephemeral data
- Pod logs

No manual configuration needed — Bottlerocket bootstrap commands handle the setup. This gives workloads fast local SSD performance without any DaemonSets or host-level scripts.

### Persistent Volumes

```bash
# List storage classes
kubectl get storageclass

# List CSI drivers
kubectl get csidrivers

# Check persistent volumes
kubectl get pv

# Check persistent volume claims
kubectl get pvc -A

# Describe a stuck PVC
kubectl describe pvc <name> -n <namespace>

# Check volume attachments
kubectl get volumeattachments
```

### Custom StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "5000"
  throughput: "500"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
```

## Networking & Load Balancing

In EKS Auto Mode, the **AWS Load Balancer Controller is built-in** — no Helm chart installation needed.
- Creating a `Service` of `type: LoadBalancer` automatically provisions an NLB
- Creating an `Ingress` resource automatically provisions an ALB
- No IAM role setup for the LB controller — it's handled by the control plane

```bash
# List ingress classes
kubectl get ingressclass

# Check services of type LoadBalancer
kubectl get svc -A --field-selector spec.type=LoadBalancer

# List ingresses
kubectl get ingress -A

# Check VPC CNI config
kubectl describe nodeclass default | grep -A5 "Network Policy"

# View pod networking (CIDR, service CIDR)
aws eks describe-cluster --name demo-cluster-auto --region us-east-1 --query 'cluster.kubernetesNetworkConfig'
```

## DNS (CoreDNS)

In EKS Auto Mode, CoreDNS runs as a **node-level system service** — not as pods or a managed add-on.
There are no CoreDNS pods, no `kube-dns` service, and no CoreDNS ConfigMap visible in the cluster.
DNS queries from pods are intercepted at the node networking layer automatically.

```bash
# Test DNS resolution (use FQDN — busybox nslookup doesn't always append search domains)
kubectl run dnstest --rm -it --restart=Never --image=busybox -- nslookup kubernetes.default.svc.cluster.local

# Check what's actually in kube-system (expect only eks-extension-metrics-api)
kubectl get all -n kube-system

# View pod DNS config to confirm cluster DNS IP
kubectl run dnstest --rm -it --restart=Never --image=busybox -- cat /etc/resolv.conf
```

> **Note:** If you mix Auto Mode with other compute (Managed Node Groups, Fargate, self-managed),
> you'll need to install the CoreDNS add-on for those non-Auto nodes.

### DNS Troubleshooting Workflow

**Step 1: Verify DNS from inside the application pod**

```bash
# Exec into the pod
kubectl exec -it <pod-name> -- sh

# Check resolv.conf — nameserver should be the cluster DNS IP (e.g. 172.20.0.10)
cat /etc/resolv.conf

# Test internal service resolution
nslookup kubernetes.default 172.20.0.10

# Test FQDN resolution
nslookup <service>.<namespace>.svc.cluster.local 172.20.0.10

# Test external resolution
nslookup amazon.com 172.20.0.10

# Test connectivity
telnet google.com 80
```

**Step 2: Check for Network Policies blocking DNS**

```bash
# List all network policies (might block UDP/53 to cluster DNS)
kubectl get networkpolicies -A

# Describe to see ingress/egress rules
kubectl describe networkpolicy -A
```

**Step 3: View CoreDNS logs from the node**

Since CoreDNS runs as a systemd service, use `kubectl debug node` to access its logs:

```bash
# Launch a debug container on the node
kubectl debug node/<NODE_NAME> -it --profile=sysadmin --image=public.ecr.aws/amazonlinux/amazonlinux:2023

# Inside the debug container:
yum install -y util-linux-core

# Stream CoreDNS logs
nsenter -t 1 -m journalctl -f -u coredns
```

**Step 4: Verify node-level DNS resolution**

```bash
# Check node's own resolv.conf (uses systemd-resolved at 127.0.0.53)
nsenter -t 1 -m cat /etc/resolv.conf

# Test node DNS resolution
yum install -y bind-utils
nsenter -t 1 -n nslookup amazon.com 127.0.0.53
```

### CoreDNS Response Codes

| Code | Meaning | Action |
|------|---------|--------|
| `NOERROR` | Query succeeded | No issue |
| `NXDOMAIN` | Domain/service doesn't exist | Verify service exists: `kubectl get svc -A \| grep <name>` |
| `SERVFAIL` | Server failure during resolution | Check upstream DNS, network policies, or VPC DNS configuration |

### DNS Test Pod (for reproducing issues on specific nodes)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-test-pod
  labels:
    app: dns-test
spec:
  nodeName: <NODE_NAME>
  containers:
  - name: dns-tester
    image: curlimages/curl:latest
    command: ["/bin/sh"]
    args:
      - -c
      - |
        while true; do
          echo "Testing DNS resolution..."
          curl -Is --connect-timeout 5 http://google.com || echo "google.com failed"
          sleep 2
          curl -Is --connect-timeout 5 http://amazon.com || echo "amazon.com failed"
          sleep 2
          curl -Is --connect-timeout 5 http://my-service.default.svc.cluster.local || echo "internal failed"
          sleep 10
        done
  restartPolicy: Always
```

```bash
# Deploy to a specific node and watch logs
kubectl apply -f dns-test-pod.yaml
kubectl logs -f dns-test-pod
```

### NodeDiagnostic Resource (alternative to debug containers)

EKS Auto Mode includes a **Node Monitoring Agent** that can retrieve node logs without debug containers:

```bash
# Create a NodeDiagnostic resource to retrieve logs via S3
kubectl apply -f - <<EOF
apiVersion: eks.amazonaws.com/v1alpha1
kind: NodeDiagnostic
metadata:
  name: <NODE_NAME>
spec: {}
EOF

# Check status (logs are uploaded to S3)
kubectl get nodediagnostic <NODE_NAME> -o yaml
```

> See AWS docs: [Retrieve node logs for a managed node using kubectl and S3](https://docs.aws.amazon.com/eks/latest/userguide/node-logs-diagnostic.html)

## General Cluster Health

```bash
# All resources in kube-system
kubectl get all -n kube-system

# Cluster info
kubectl cluster-info

# Check API server health
kubectl get --raw='/healthz'

# Events (useful for debugging Auto Mode scaling)
kubectl get events --sort-by='.lastTimestamp'

# Check EKS add-ons via AWS CLI
aws eks list-addons --cluster-name demo-cluster-auto --region us-east-1
aws eks describe-addon --cluster-name demo-cluster-auto --addon-name <addon-name> --region us-east-1
```

## Debugging Auto Mode Scaling

```bash
# Why is my pod pending?
kubectl describe pod <pod-name>

# Check nodeclass readiness (instance profile, subnets, security groups)
kubectl describe nodeclass default

# Check node claims for provisioning status
kubectl describe nodeclaim <name>

# List all node claims with status
kubectl get nodeclaims -o wide

# Watch nodes being provisioned in real time
kubectl get nodes -w

# Check if pod disruption budgets are blocking scale-down
kubectl get pdb -A

# Check karpenter-related events
kubectl get events --field-selector source=karpenter --sort-by='.lastTimestamp'

# Check all recent events
kubectl get events --sort-by='.lastTimestamp' | head -30

# Verify node role trust policy (must include eks.amazonaws.com)
aws iam get-role --role-name <node-role-name> --query 'Role.AssumeRolePolicyDocument'

# Check if EKS can see the node role
aws eks describe-cluster --name demo-cluster-auto --region us-east-1 --query 'cluster.computeConfig'
```

## Quick Deploy (trigger node provisioning)

```bash
# Deploy a workload to trigger scale-from-zero
kubectl create deployment nginx --image=nginx --replicas=1

# Watch until a node appears
kubectl get nodes -w

# Clean up
kubectl delete deployment nginx
```

## GPU / Accelerated Workloads

EKS Auto Mode automatically manages NVIDIA, Trainium, and Inferentia drivers — no DaemonSets needed.

**GPU NodePool example:**

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu
spec:
  disruption:
    budgets:
      - nodes: 10%
    consolidateAfter: 1h
    consolidationPolicy: WhenEmpty
  template:
    spec:
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: eks.amazonaws.com/instance-family
          operator: In
          values: ["g6e", "g6"]
      taints:
        - key: nvidia.com/gpu
          effect: NoSchedule
      terminationGracePeriod: 24h0m0s
```

**GPU pod example:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nvidia-smi
spec:
  nodeSelector:
    eks.amazonaws.com/compute-type: auto
  restartPolicy: OnFailure
  containers:
    - name: nvidia-smi
      image: public.ecr.aws/amazonlinux/amazonlinux:2023-minimal
      args: ["nvidia-smi"]
      resources:
        requests:
          memory: "30Gi"
          cpu: "3500m"
          nvidia.com/gpu: 1
        limits:
          memory: "30Gi"
          nvidia.com/gpu: 1
  tolerations:
    - key: nvidia.com/gpu
      effect: NoSchedule
      operator: Exists
```

```bash
# Apply and validate
kubectl apply -f nodepool-gpu.yaml
kubectl apply -f pod.yaml
kubectl logs nvidia-smi

# Clean up (terminates GPU node)
kubectl delete -f nodepool-gpu.yaml
kubectl delete -f pod.yaml
```

> GPU instance types (p*, g*) often have a default service quota of 0. Request a quota increase via AWS Service Quotas console (can take 24-48h).

## Troubleshooting

### Node Not Joining Cluster

```bash
# Check NodeClaims for Ready=False
kubectl get nodeclaims

# Describe a stuck nodeclaim
kubectl describe nodeclaim <name>

# Get EC2 console output for boot issues
kubectl get pod <pod-name> -o wide   # find node/instance ID
aws ec2 get-console-output --instance-id <instance-id> --latest --output text

# Debug a node directly (launches a pod on it)
kubectl debug node/<instance-id> -it --profile=sysadmin --image=public.ecr.aws/amazonlinux/amazonlinux:2023

# Inside debug container: view kubelet logs
yum install -y util-linux-core
nsenter -t 1 -m journalctl -f -u kubelet
```

### NodeClass Not Ready

A healthy NodeClass goes through these status conditions (all must be `True`):

| Condition | Meaning |
|-----------|---------|
| `SubnetsReady` | Subnets matching `subnetSelectorTerms` were found |
| `SecurityGroupsReady` | Security groups matching `securityGroupSelectorTerms` were found |
| `InstanceProfileReady` | IAM instance profile was created/found for the node role |
| `ValidationSucceeded` | NodeClass spec passed validation |
| `Ready` | All conditions passed — NodeClass can provision nodes |

```bash
# Check all conditions
kubectl describe nodeclass default

# Common reasons:
# - InstanceProfileCreationFailed → cluster role trust policy missing sts:TagSession
# - UnauthorizedNodeRole → node role missing EKS access entry or IAM policies
# - SubnetsNotReady → subnets not tagged correctly
# - SecurityGroupsNotReady → SG not found

# THE FIX for InstanceProfileCreationFailed:
# Cluster role trust policy MUST include both sts:AssumeRole AND sts:TagSession:
#   {
#     "Effect": "Allow",
#     "Principal": { "Service": "eks.amazonaws.com" },
#     "Action": ["sts:AssumeRole", "sts:TagSession"]
#   }

# Verify cluster role trust policy
aws iam get-role --role-name <cluster-role-name> --query 'Role.AssumeRolePolicyDocument'

# Verify node role trust policy (must include eks.amazonaws.com)
aws iam get-role --role-name <node-role-name> --query 'Role.AssumeRolePolicyDocument'

# Check instance profiles for the role
aws iam list-instance-profiles-for-role --role-name <node-role-name>
```

### Karpenter Events in CloudWatch Logs

If control plane logging is enabled, query Karpenter decisions in CloudWatch Logs Insights:

```
fields @timestamp, @message
| filter @logStream like /kube-apiserver-audit/
| filter @message like 'DisruptionBlocked'
  or @message like 'DisruptionLaunching'
  or @message like 'DisruptionTerminating'
  or @message like 'FailedScheduling'
  or @message like 'NoCompatibleInstanceTypes'
  or @message like 'InsufficientCapacityError'
  or @message like 'NodeClassNotReady'
  or @message like 'Unconsolidatable'
  or @message like 'FailedDraining'
| sort @timestamp desc
```

Filter by specific node:
```
| filter @message like /i-01234567890123456/
```

### IAM Errors

```bash
# Check CloudTrail for IAM permission issues
# Navigate to CloudTrail > Event History > filter by Event source: eks.amazonaws.com
# Look for: AccessDenied, UnauthorizedOperation, InvalidClientTokenId

# Verify cluster role policies
aws iam list-attached-role-policies --role-name <cluster-role-name>

# Verify node role policies
aws iam list-attached-role-policies --role-name <node-role-name>
```

### Pod Stuck Pending

```bash
# Check pod events
kubectl describe pod <pod-name>

# Ensure pod targets Auto Mode nodes (if using nodeSelector)
# Must use: eks.amazonaws.com/compute-type: auto

# Check if NodePool can satisfy requirements
kubectl get nodepools -o yaml

# Force reconcile on nodeclass
kubectl annotate nodeclass default eks.amazonaws.com/force-reconcile=$(date +%s) --overwrite
```

## SELinux & Volume Sharing

EKS Auto Mode nodes run SELinux in enforcing mode. Pods get unique MCS labels, which prevents cross-pod volume access by default.

To share a volume between pods, assign the same SELinux label:

```yaml
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456,c789"
```

## Pod Identity (Replaces IRSA)

EKS Auto Mode defaults to **EKS Pod Identity** instead of IRSA (IAM Roles for Service Accounts via OIDC).

| | IRSA (Standard EKS) | EKS Pod Identity (Auto Mode) |
|---|---|---|
| **Setup** | Create OIDC provider, IAM role with trust policy referencing the OIDC issuer | Create Pod Identity Association via API/CLI |
| **How it works** | Service account annotation → webhook mutates pod → STS AssumeRoleWithWebIdentity | Pod Identity Agent on node intercepts AWS API calls → exchanges token for credentials |
| **Terraform/IaC** | Complex (OIDC provider + IAM role + trust policy + SA annotation) | Simple (`aws_eks_pod_identity_association` resource) |
| **Credential delivery** | Projected service account token → STS | Local agent on node handles token exchange |

```bash
# Create a Pod Identity Association
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace my-app \
  --service-account my-sa \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/my-pod-role

# List Pod Identity Associations
aws eks list-pod-identity-associations --cluster-name my-cluster

# Delete a Pod Identity Association
aws eks delete-pod-identity-association \
  --cluster-name my-cluster \
  --association-id a-xxxxxxxxxxxxxxx
```

> **Note:** The Pod Identity Agent runs as a systemd service on Auto Mode nodes. On Standard EKS clusters, install the `eks-pod-identity-agent` add-on to use Pod Identity.

## Cost & Limitations

| Factor | Detail |
|--------|--------|
| Management fee | 12% on top of EC2 instance costs (only for Auto Mode managed nodes) |
| Pod limit | 110 pods per node (not configurable) — can lead to more nodes than self-managed Karpenter |
| No mixed capacity in same pool | Cannot mix Spot and On-Demand in the same NodePool — separate pools required |
| Customization | Limited compared to self-managed Karpenter (no custom AMI, no SSH, no host-level changes) |
| Debugging | Harder — no node SSH, no SSM, no public IPv4, read-only root filesystem. Must use `kubectl debug node` |
| Control plane upgrades | Still manual — node OS/components auto-update, but control plane K8s version requires `aws eks update-cluster-version` |
| Version mismatch risk | Automated node updates can drift ahead of stateful workloads that expect a specific version |
| Migration prerequisite | Existing clusters must be on Karpenter v1.1+ to avoid NodePool/NodeClaim API conflicts |
| Availability | All commercial AWS regions except China |

## Managed EC2 Instances

Nodes in EKS Auto Mode are **Managed EC2 Instances** with these restrictions:

| Attribute | Traditional EKS Node | EKS Auto Mode Node |
|-----------|---------------------|---------------------|
| AMI | Amazon Linux 2, Bottlerocket, or custom | Bottlereck only (locked-down, immutable) |
| Public IPv4 | Assigned (configurable) | Not assigned |
| SSH access | Supported | Not supported |
| SSM access | Supported | Not supported |
| Root filesystem | Read-write | Read-only |
| SELinux | Optional | Enforcing |
| Node rotation | Manual (or via node group update) | Automatic every 21 days |
| Core add-ons | Run as pods in the cluster | Run as systemd services in the AMI |
| Visible in cluster | kubelet, kube-proxy pods, CoreDNS pods | Only metrics-server pods |

**Systemd services running on each Auto Mode node:**
- kubelet
- kube-proxy
- CoreDNS
- VPC CNI
- Node Monitor Agent
- Pod Identity Agent

## Auto Mode vs Fargate vs Self-Managed Karpenter

| | EKS Auto Mode | EKS with Fargate | Self-Managed Karpenter |
|---|---|---|---|
| **Compute** | EC2 (managed by AWS) | Serverless (no instances) | EC2 (you manage) |
| **Management fee** | 12% on EC2 | Fargate pricing (vCPU + memory/s) | None |
| **DaemonSets** | Supported | Not supported | Supported |
| **Privileged containers** | Supported | Not supported | Supported |
| **Persistent storage** | EBS (auto-managed CSI) | EFS only | EBS/EFS (you install CSI) |
| **GPU / accelerators** | Supported (auto driver install) | Not supported | Supported (manual driver setup) |
| **Custom AMI** | No (Bottlerocket only) | N/A | Yes |
| **SSH to nodes** | No | N/A | Yes |
| **Pod limit per node** | 110 (fixed) | 1 pod per Fargate instance | Configurable |
| **Autoscaling** | Built-in Karpenter | Per-pod scaling | You install & configure |
| **OS patching** | Automatic (21-day rotation) | AWS-managed | You manage |
| **Add-on management** | AWS-managed (CoreDNS, kube-proxy, VPC CNI, LB controller) | AWS-managed | You install & update |
| **Spot instances** | Supported (separate pool) | Not supported | Supported (flexible config) |
| **Operational overhead** | Low | Lowest | Highest |
| **Control / flexibility** | Medium | Low | High |
| **Best for** | Teams wanting managed infra with K8s flexibility | Simple stateless workloads | Teams needing full control |

> **How does EKS Auto Mode compare to similar offerings on other clouds?** See the comparison table below.

## EKS Auto Mode vs GKE Autopilot vs AKS Automatic

| | EKS Auto Mode | GKE Autopilot | AKS Automatic |
|---|---|---|---|
| **Provider** | AWS | Google Cloud | Microsoft Azure |
| **Compute** | Managed EC2 (Bottlerocket) | Managed GCE (COS) | Managed VMSS |
| **Pricing model** | EC2 cost + 12% management fee | Flat cluster fee + per-pod (vCPU/mem/s) | Standard AKS pricing |
| **Autoscaler** | Karpenter (built-in) | GKE NAP (built-in) | Karpenter + Azure workload autoscaling |
| **Node access** | No SSH, no SSM, no public IP | No SSH | Limited (Azure Serial Console) |
| **Custom AMI/image** | No (Bottlerocket only) | No (COS only) | No |
| **DaemonSets** | Supported | Supported (with restrictions) | Supported |
| **Privileged containers** | Supported | Not supported | Supported (with PSS) |
| **GPU / accelerators** | Supported (auto driver install) | Supported | Supported |
| **Pod limit per node** | 110 (fixed) | 110 (default, configurable) | Configurable |
| **Network policy** | VPC CNI (custom config limited) | GKE Dataplane V2 (enforced) | Azure CNI (Cilium) |
| **Control plane upgrades** | Manual (`aws eks update-cluster-version`) | Automatic (with maintenance windows) | Automatic (with maintenance windows) |
| **Node OS upgrades** | Automatic (21-day rotation) | Automatic | Automatic |
| **Add-on management** | AWS-managed (systemd services) | Google-managed | Azure-managed |
| **Spot/preemptible** | Supported (separate NodePool) | Supported (Spot pods) | Supported |
| **Persistent storage** | EBS (auto-managed CSI) | PD CSI (auto-managed) | Azure Disk CSI (auto-managed) |
| **Security model** | Immutable AMI, SELinux enforcing, read-only rootfs | Hardened COS, Workload Identity | Azure Policy, Entra ID integration |
| **Maturity** | GA since re:Invent 2024 | GA since 2021 | GA since 2024 |

**Key takeaways:**
- **GKE Autopilot** is the most opinionated — pod-level billing, enforced network policies, no privileged containers
- **AKS Automatic** has the broadest automation scope — control plane + nodes + networking + security policies
- **EKS Auto Mode** sits in the middle — strong node automation while preserving Kubernetes API flexibility and Spot support

## Key Facts

| Topic | Detail |
|-------|--------|
| AMI | Bottlerocket only (no custom AMI support) |
| SSH | Not supported — use `kubectl debug node` or `NodeDiagnostic` |
| Max node lifetime | 21 days, then auto-replaced |
| Managed components | Karpenter, LB Controller, VPC CNI, CoreDNS, kube-proxy, EBS CSI, Pod Identity Agent, Node Monitoring Agent |
| Visible in cluster | Only metrics-server pods; everything else runs off-cluster |
| EC2 pricing | Standard EC2 + Auto Mode management fee (only for managed nodes) |
| Mixed mode | Can run managed node groups alongside Auto Mode nodes |
| Scale to zero | Supported — nodes terminate when no workloads run |
| Custom tooling on host | Deploy as DaemonSet (no AMI customization) |
| Instance types | Built-in pools use C, M, R families (gen 5+); custom pools can use any |
| Local NVMe storage | Auto-configured for container images, pod ephemeral data, and logs on supported instance types |
| Cluster deletion | Deleting the EKS cluster forces termination of all associated EC2 managed instances |
| Min K8s version | Requires Kubernetes 1.29 or greater |

## When to Use Auto Mode vs Standard EKS

### Use EKS Auto Mode when:

- **Greenfield projects** — start here, avoid building node management tech debt
- **Dynamic/bursty workloads** — AI/ML training, CI/CD runners, batch jobs that scale 0→100 nodes
- **Small Platform teams** — spend time on developer experience, not node patching
- **Day 2 operations are painful** — if your backlog is full of upgrades, AMI rotations, and add-on version conflicts
- **Cost optimization without tuning** — let Karpenter bin-pack and consolidate automatically

### Use EKS Standard when:

- **Custom kernel requirements** — proprietary kernel modules, custom sysctl parameters, specific hardened AMIs (CIS benchmarks)
- **Instance store NVMe** — workloads mounting local NVMe drives in ways the CSI driver doesn't support
- **Node-level debugging required** — compliance frameworks requiring SSH/SSM for forensic analysis
- **Legacy applications** — apps needing host-level configuration, specific OS packages, or writable root filesystem
- **Full Karpenter control** — need custom consolidation logic, pod limits >110, or unsupported instance configurations

### Hybrid approach

You can run **both modes on the same cluster** — Managed Node Groups alongside Auto Mode nodes. Use `eks.amazonaws.com/compute-type: auto` nodeSelector to target Auto Mode, and standard node labels for your managed groups.

## Useful Links

- [EKS Auto Mode Overview](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)
- [Create a Node Class](https://docs.aws.amazon.com/eks/latest/userguide/create-node-class.html)
- [Create a Node Pool](https://docs.aws.amazon.com/eks/latest/userguide/create-node-pool.html)
- [Deploy Accelerated Workloads](https://docs.aws.amazon.com/eks/latest/userguide/auto-accelerated.html)
- [Deploy a Sample Load Balancer Workload](https://docs.aws.amazon.com/eks/latest/userguide/auto-elb.html)
- [Deploy a Sample Stateful Workload](https://docs.aws.amazon.com/eks/latest/userguide/auto-storage.html)
- [Migrate from Karpenter to EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/auto-migrate-karpenter.html)
- [Migrate from Managed Node Groups to EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/auto-migrate-mng.html)
- [Troubleshoot EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/auto-troubleshoot.html)
- [EKS Auto Mode Best Practices](https://docs.aws.amazon.com/eks/latest/best-practices/automode.html)
- [Cost Optimization](https://docs.aws.amazon.com/eks/latest/userguide/auto-cost-control.html)
- [EKS Blueprints - Custom NodePools (Terraform)](https://aws-ia.github.io/terraform-aws-eks-blueprints/patterns/eks-automode/eks-automode-custom-nodepools/)
- [AWS Samples - EKS Auto Mode](https://github.com/aws-samples/sample-aws-eks-auto-mode)
