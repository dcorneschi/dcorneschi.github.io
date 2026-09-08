# EKS Fargate

**EKS Fargate** is a serverless compute engine for Kubernetes that runs your
containers without you managing EC2 instances. AWS handles the underlying
infrastructure automatically.

## Key Differences

| Feature | EC2 Nodes | Fargate |
|---------|-----------|---------|
| **Management** | You manage EC2 instances, scaling, patching | AWS manages everything |
| **Cost** | Pay for instances even if unused | Pay only for resources used (vCPU/memory) |
| **Scaling** | Manual or Auto Scaling groups | Automatic |
| **Best for** | Sustained, cost-optimized workloads | Bursty workloads, minimal ops |

## How Fargate Profiles Work

A **Fargate profile** defines which pods run on Fargate. It includes:

- **Namespace selector** — which Kubernetes namespaces
- **Label selector** — which pod labels match
- **Subnets and security groups** — where pods run (private subnets only)

When a pod matches a profile's selectors, the Fargate scheduler places it on
Fargate instead of an EC2 node.

## Example Fargate Profile

```bash
aws eks create-fargate-profile \
  --cluster-name my-cluster \
  --fargate-profile-name my-profile \
  --pod-execution-role-arn arn:aws:iam::ACCOUNT:role/AmazonEKSFargatePodExecutionRole \
  --selectors namespace=default,labels.app=web \
  --subnets subnet-12345 subnet-67890
```

This runs all pods in the `default` namespace with label `app=web` on Fargate.

> The pod execution role is an IAM role Fargate assumes to pull images and write
> logs on the pod's behalf — distinct from any Kubernetes RBAC the workload uses.

## Real-World Examples

### 1. Temporary batch jobs

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  namespace: batch-jobs
spec:
  template:
    metadata:
      labels:
        workload: batch
    spec:
      containers:
      - name: processor
        image: my-processor:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "256m"
```

With a Fargate profile for `namespace=batch-jobs`, this scales up when needed and
down to zero when done — no idle EC2 capacity.

### 2. Development / testing environments

```bash
aws eks create-fargate-profile \
  --cluster-name dev-cluster \
  --fargate-profile-name dev-profile \
  --pod-execution-role-arn arn:aws:iam::ACCOUNT:role/AmazonEKSFargatePodExecutionRole \
  --selectors namespace=dev
```

Dev pods run on Fargate with no EC2 capacity planning.

## Limitations

- **No DaemonSets** — Fargate runs one pod per micro-VM; DaemonSets aren't supported (use sidecars for per-pod agents).
- **No privileged containers** — limited security context options.
- **Resource sizing** — only specific vCPU/memory combinations (0.25–16 vCPU on current Fargate; each vCPU has a fixed memory range).
- **No GPU support.**
- **Port 10250 reserved** — Fargate reserves 10250, so tools like Metrics Server must use an alternate port (e.g. 10251).
- **Private subnets only** — Fargate pods can't run in public subnets.

## Pricing

You pay for the vCPU and memory requested per pod, billed per second (with a
one-minute minimum). Roughly: 1 vCPU + 2 GB ≈ $0.05/hour per pod, varying by
region. Check current AWS Fargate pricing for exact figures.

## When to Use Fargate

Use Fargate for:

- Bursty or intermittent workloads
- Development/testing clusters
- Cost-sensitive, low-traffic services
- Environments that need minimal ops overhead

Avoid Fargate for:

- High-throughput, sustained workloads (EC2 is usually cheaper)
- Workloads needing DaemonSets
- GPU-accelerated applications
- Workloads requiring privileged containers or strict low-level security control

## Checking Fargate Status

```bash
# List all Fargate profiles for a cluster
aws eks list-fargate-profiles --cluster-name YOUR_CLUSTER --region YOUR_REGION

# Details of a specific profile
aws eks describe-fargate-profile \
  --cluster-name YOUR_CLUSTER --fargate-profile-name PROFILE_NAME --region YOUR_REGION

# Quick check: just the profile names
aws eks list-fargate-profiles --cluster-name YOUR_CLUSTER --region YOUR_REGION \
  --query 'fargateProfileNames' --output text

# Just the status of a profile
aws eks describe-fargate-profile \
  --cluster-name YOUR_CLUSTER --fargate-profile-name PROFILE_NAME --region YOUR_REGION \
  --query 'fargateProfile.status' --output text
```

Status values: **ACTIVE** (enabled and working), **CREATING**, **DELETING**,
**CREATE_FAILED** / **DELETE_FAILED** (error states).

## Metrics Server on Fargate

Because **port 10250 is reserved on Fargate**, Metrics Server must serve on a
different port when it runs on Fargate nodes.

### EKS add-on (recommended)

The community add-on is pre-configured to use port 10251:

```bash
aws eks create-addon --cluster-name YOUR_CLUSTER --addon-name metrics-server
```

(Also available in the console: EKS cluster → Add-ons → Get more add-ons →
Community add-ons → Metrics Server.)

### Manual deployment

If deploying from the manifest, replace port `10250` with `10251` in
`components.yaml`, then apply:

```bash
# Download, edit the --secure-port and containerPort from 10250 to 10251, then:
kubectl apply -f components.yaml
```

### Security group requirements

- **Port 10251** (or your chosen alternate) — between metrics-server pods and all nodes.
- **Port 10250** — still needed for Metrics Server to scrape kubelet endpoints on non-Fargate nodes.

If you have **no Fargate nodes**, the default port 10250 works without changes.
See [Installing metrics-server on Kubernetes](articles/metrics-server-install.md)
for the general install.

## Related

- [Installing metrics-server on Kubernetes](articles/metrics-server-install.md)
- [EKS Node Groups Explained](articles/eks-node-groups-explained.md)
- [EKS Auto Mode](articles/eks-auto-mode.md)

## Skills Practiced

- Understanding Fargate's serverless model and how it differs from EC2 nodes
- Creating Fargate profiles with namespace/label selectors and a pod execution role
- Recognizing Fargate limitations (no DaemonSets/GPU, reserved port 10250)
- Running Metrics Server on Fargate with an alternate port
