# Persistent Volumes on EKS with EBS CSI Driver

How to provision persistent storage on AWS EKS using the EBS CSI Driver — covering StorageClass configuration, dynamic provisioning, default storage class setup, and CSI driver installation.

## Prerequisites

- EKS Cluster v1.30 or later
- `eksctl` and AWS CLI installed
- EBS CSI Driver add-on enabled on the cluster

## EBS CSI Driver Workflow

```
Developer creates PVC → API Server → EBS CSI Controller → AWS EBS API (provisions volume)
                                                              ↓
                                        PV created with EBS Volume ID (PV ↔ PVC bound)
                                                              ↓
                            Pod created with PVC → CSI Node Driver → attaches EBS to node → Pod starts
```

1. Developer creates a PVC referencing a StorageClass
2. The EBS CSI Controller receives the request and provisions an EBS volume via the AWS API
3. The controller gets temporary AWS credentials via the EKS Pod Identity Agent
4. A PV is automatically created and bound to the PVC (dynamic provisioning)
5. The EBS volume is **not** attached to a node until a pod using the PVC is scheduled
6. When the pod starts, the CSI Node driver (DaemonSet on each node) attaches the EBS volume to the node where the pod runs

## EBS Volume Types

| Type | Category | IOPS | Use Case |
|------|----------|------|----------|
| `gp3` | General Purpose SSD | Up to 16,000 | Default choice for most workloads |
| `gp2` | General Purpose SSD | Up to 16,000 | Legacy default (use gp3 instead) |
| `io2` | Provisioned IOPS SSD | Up to 256,000 | Databases, latency-sensitive workloads |
| `io1` | Provisioned IOPS SSD | Up to 64,000 | Legacy provisioned IOPS |
| `st1` | Throughput Optimized HDD | 500 | Big data, log processing |
| `sc1` | Cold HDD | 250 | Infrequent access, archival |

> **Key constraint:** EBS volumes are zone-specific. The node and the EBS volume must be in the same availability zone.

## Creating a StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: io2-volume
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iopsPerGB: "50"
  fsType: ext4
volumeBindingMode: Immediate
```

```bash
kubectl apply -f ebs-io2-volume.yaml
kubectl get storageclass
```

### volumeBindingMode Options

| Mode | Behavior |
|------|----------|
| `Immediate` | PV is provisioned and bound to PVC as soon as the PVC is created |
| `WaitForFirstConsumer` | PV provisioning is delayed until a pod using the PVC is scheduled (ensures volume is in the same AZ as the pod) |

> **Recommendation:** Use `WaitForFirstConsumer` for most workloads. It prevents AZ mismatch issues where a volume is provisioned in one AZ but the pod gets scheduled in another.

## Creating a PVC (Dynamic Provisioning)

With dynamic provisioning, you don't need to create a PV manually. The CSI driver creates it automatically when the PVC is submitted.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: io2-volume
  resources:
    requests:
      storage: 12Gi
```

```bash
kubectl apply -f io2-pvc.yaml
kubectl get pvc
```

### Access Modes

| Mode | Description |
|------|-------------|
| `ReadWriteOnce` | Single node can read/write (standard for EBS) |
| `ReadOnlyMany` | Multiple nodes can read (not supported by EBS) |
| `ReadWriteMany` | Multiple nodes can read/write (not supported by EBS — use EFS instead) |
| `ReadWriteOncePod` | Single pod can read/write (K8s 1.27+) |

> EBS only supports `ReadWriteOnce` and `ReadWriteOncePod`. For multi-node read/write access, use EFS with the EFS CSI Driver.

## Attaching a Volume to a Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: io2-volume-deployment
  labels:
    app: io2-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: io2-app
  template:
    metadata:
      labels:
        app: io2-app
    spec:
      containers:
      - name: io2-container
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: io2-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: io2-volume
        persistentVolumeClaim:
          claimName: my-pvc
```

```bash
kubectl apply -f deployment.yaml
```

When the pod is created, the EBS volume state changes from `Available` to `In-use` in the AWS console.

## Setting a Default StorageClass

Starting with EKS 1.30, Amazon no longer sets a default StorageClass automatically. You need to annotate one yourself — otherwise PVCs without an explicit `storageClassName` won't get provisioned.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
```

```bash
kubectl apply -f gp3-storageclass.yaml

# Verify default is set
kubectl get storageclass
# The default class shows (default) next to its name
```

> **Tip:** Set `allowVolumeExpansion: true` so you can resize PVCs later without recreating them. Set `encrypted: "true"` for encryption at rest.

## Installing EBS CSI Driver on Existing Clusters

### Step 1: Ensure OIDC Provider is associated

```bash
export REGION=us-west-2
export CLUSTER_NAME=my-cluster

# Associate OIDC provider (skip if already done)
eksctl utils associate-iam-oidc-provider --region=$REGION --cluster=$CLUSTER_NAME --approve
```

### Step 2: Install the EBS CSI Driver add-on

```bash
eksctl create addon --cluster $CLUSTER_NAME --region $REGION --name aws-ebs-csi-driver
```

This creates the add-on with an IAM role and policy automatically. Verify it's running:

```bash
kubectl get pods -n kube-system | grep ebs-csi
```

You should see:
- **ebs-csi-controller** pods (typically 2 replicas) — handle provisioning
- **ebs-csi-node** pods (one per worker node, DaemonSet) — handle attach/mount

### Step 3: Install Pod Identity Agent and migrate from IRSA

IRSA (IAM Roles for Service Accounts) is deprecated. Migrate to Pod Identity:

```bash
# Install Pod Identity Agent if not present
aws eks create-addon --cluster-name $CLUSTER_NAME --addon-name eks-pod-identity-agent

# Migrate from IRSA to Pod Identity
eksctl utils migrate-to-pod-identity --cluster $CLUSTER_NAME --approve

# Verify Pod Identity associations
eksctl get podidentityassociation --cluster $CLUSTER_NAME
```

### Using eksctl cluster config (new clusters)

If creating a new cluster, include the add-ons in your eksctl config:

```yaml
addons:
- name: aws-ebs-csi-driver
  version: latest
- name: eks-pod-identity-agent
  version: latest

addonsConfig:
  autoApplyPodIdentityAssociations: true
```

## Reclaim Policies

The `reclaimPolicy` on a StorageClass determines what happens to the EBS volume when the PVC is deleted:

| Policy | Behavior |
|--------|----------|
| `Delete` | PV and the underlying EBS volume are both deleted (default for dynamic provisioning) |
| `Retain` | PV is released but the EBS volume is kept in AWS — manual cleanup required |

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-retain
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain          # Keep the EBS volume after PVC deletion
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  fsType: ext4
```

> **Production recommendation:** Use `Retain` for databases and stateful workloads. If someone accidentally deletes a PVC, the data is still on the EBS volume and can be recovered by creating a new PV pointing to the existing volume ID.

### Recovering a Retained Volume

After a PVC is deleted with `Retain` policy, the PV goes to `Released` state. To reuse it:

```bash
# Find the released PV and its EBS volume ID
kubectl get pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,VOLUME:.spec.csi.volumeHandle

# Remove the old claimRef so a new PVC can bind to it
kubectl patch pv <pv-name> -p '{"spec":{"claimRef":null}}'
```

Then create a new PVC that matches the PV's storageClass and capacity, or bind it explicitly.

## Volume Snapshots and Restore

The EBS CSI Driver supports VolumeSnapshots for backing up and restoring EBS volumes. This requires the CSI snapshot controller and CRDs.

### Prerequisites

```bash
# Install snapshot CRDs (if not already present)
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

# Install snapshot controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```

### Create a VolumeSnapshotClass

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Retain           # Keep the AWS snapshot even if VolumeSnapshot is deleted
```

### Take a Snapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-pvc-snapshot
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: my-pvc    # The PVC to snapshot
```

```bash
kubectl apply -f snapshot.yaml

# Check snapshot status
kubectl get volumesnapshot
kubectl describe volumesnapshot my-pvc-snapshot
```

The snapshot is `readyToUse: true` once the EBS snapshot completes in AWS.

### Restore from a Snapshot

Create a new PVC that references the snapshot as its data source:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 12Gi
  dataSource:
    name: my-pvc-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

```bash
kubectl apply -f restored-pvc.yaml
kubectl get pvc restored-pvc
```

The CSI driver creates a new EBS volume from the snapshot and binds it to the PVC. You can then attach it to a pod like any other PVC.

> **Use cases:** Pre-upgrade database backups, cloning production data to staging, disaster recovery, and point-in-time restores.

## Troubleshooting

### PVC stuck in Pending

```bash
# Check PVC events
kubectl describe pvc <pvc-name>

# Common causes:
# - EBS CSI Driver not installed
# - StorageClass doesn't exist
# - IAM permissions missing (CSI controller can't call AWS API)
# - volumeBindingMode: WaitForFirstConsumer (waits for pod)
```

### Pod stuck in Pending (volume won't attach)

```bash
# Check pod events
kubectl describe pod <pod-name>

# Common causes:
# - AZ mismatch: volume in us-west-2a, pod scheduled in us-west-2b
# - EBS volume limit: EC2 instances have a max number of attachable volumes
# - Node has no CSI node driver pod running
```

### Check CSI driver health

```bash
# Controller pods
kubectl get pods -n kube-system -l app=ebs-csi-controller

# Node driver pods (should be one per node)
kubectl get pods -n kube-system -l app=ebs-csi-node

# CSI driver logs
kubectl logs -n kube-system -l app=ebs-csi-controller --tail=50

# Verify StorageClass references the correct provisioner
kubectl get storageclass -o custom-columns=NAME:.metadata.name,PROVISIONER:.provisioner
```

## Useful Commands

```bash
# List all storage classes
kubectl get storageclass

# List PVCs in all namespaces
kubectl get pvc -A

# List PVs with their bound PVCs and storage class
kubectl get pv -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage,STATUS:.status.phase,CLAIM:.spec.claimRef.name,STORAGECLASS:.spec.storageClassName

# Check EBS CSI Driver service account
kubectl -n kube-system describe sa ebs-csi-controller-sa

# Delete a PVC (also deletes the EBS volume if reclaimPolicy is Delete)
kubectl delete pvc <pvc-name>

# Resize a PVC (requires allowVolumeExpansion: true on StorageClass)
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```
