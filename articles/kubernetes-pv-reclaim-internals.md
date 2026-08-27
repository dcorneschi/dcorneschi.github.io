# What Happens When a PersistentVolume Is Reclaimed

The lifecycle of a PersistentVolume after the PVC binding is released — how reclaim policies (Retain, Delete, Recycle) work, what the PV controller does, volume detachment, and data persistence.

Note: For EBS CSI provisioning workflow and StorageClass configuration, see the EKS persistent volumes guide. This article focuses on the reclaim/unbind phase.

## PV Lifecycle Overview

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────────────┐
│ Available│────▶│  Bound   │────▶│ Released │────▶│ Reclaim Policy   │
│          │     │ (PVC     │     │ (PVC     │     │ Applied:         │
│ (no PVC) │     │  bound)  │     │  deleted)│     │ Retain/Delete/   │
└──────────┘     └──────────┘     └──────────┘     │ Recycle          │
                                                   └──────────────────┘
```

### PV Phases

| Phase | Meaning |
|-------|---------|
| `Available` | PV exists but no PVC is bound to it |
| `Bound` | PV is bound to a PVC — pod can use it |
| `Released` | PVC was deleted, but PV still has data and the claimRef |
| `Failed` | Automatic reclamation failed |

```bash
# Check PV phase:
kubectl get pv
# NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM
# pv-001    10Gi       RWO            Retain           Bound       default/my-pvc
# pv-002    5Gi        RWO            Delete           Released    default/old-pvc
# pv-003    20Gi       RWO            Retain           Available
```

## What Triggers Reclaim

Reclaim happens when the **PVC is deleted** while the PV still exists:

```
Time ──────────────────────────────────────────────────────────────────▶

User              API Server        PV Controller       CSI/Cloud
  │                  │                  │                  │
  │ delete PVC ─────▶│                  │                  │
  │                  │ PVC removed      │                  │
  │                  │ from etcd        │                  │
  │                  │                  │                  │
  │                  │── watch event ──▶│                  │
  │                  │                  │ PV.status =      │
  │                  │                  │ Released         │
  │                  │                  │                  │
  │                  │                  │ Check reclaim    │
  │                  │                  │ policy:          │
  │                  │                  │ Retain/Delete/   │
  │                  │                  │ Recycle          │
  │                  │                  │                  │
  │                  │                  │── (if Delete) ──▶│ Delete volume
  │                  │                  │                  │ from cloud/storage
  │                  │                  │                  │
  │                  │                  │ Delete PV object │
  │                  │                  │ from API         │
```

## Reclaim Policy: Retain

```yaml
persistentVolumeReclaimPolicy: Retain
```

**What happens:**

```
PVC deleted
    │
    ▼
PV status → Released
    │
    ▼
Nothing else happens automatically.
PV keeps its data. Volume is NOT deleted from storage.
PV remains in "Released" state indefinitely.
    │
    ▼
Admin must manually:
  1. Back up data (if needed)
  2. Delete the PV object
  3. OR clean claimRef to make it Available again
```

The volume and its data persist on the underlying storage system. The PV object stays in the cluster with `status: Released` and the old `claimRef` still set.

### Re-Using a Retained PV

A Released PV cannot be automatically bound to a new PVC (the old claimRef blocks it). To reuse:

```bash
# Option 1: Remove claimRef to make it Available again
kubectl patch pv pv-001 --type=json \
  -p='[{"op":"remove","path":"/spec/claimRef"}]'
# PV status: Released → Available
# A new PVC can now bind to it

# Option 2: Delete the PV and create a new one pointing to the same volume
kubectl delete pv pv-001
# Then create a new PV with the same volumeHandle/volumeID

# Option 3: Create a PVC that explicitly references the PV
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: new-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
  volumeName: pv-001    # Explicitly bind to this PV
EOF
# Only works if claimRef is cleared first
```

## Reclaim Policy: Delete

```yaml
persistentVolumeReclaimPolicy: Delete
```

**What happens:**

```
PVC deleted
    │
    ▼
PV status → Released
    │
    ▼
PV Controller detects Released + Delete policy
    │
    ├── Calls CSI DeleteVolume() (or cloud API)
    │   → AWS: ec2:DeleteVolume (EBS disk gone)
    │   → GCP: compute.disks.delete
    │   → Azure: disk delete
    │
    └── Deletes the PV object from the API server
    │
    ▼
Both PV and underlying storage are GONE.
Data is permanently lost.
```

```bash
# Watch the deletion:
kubectl get pv -w
# NAME     STATUS    RECLAIM POLICY
# pv-001   Bound     Delete
# pv-001   Released  Delete          ← PVC deleted
# (pv-001 disappears)               ← PV and volume deleted
```

### Delete Policy — What Gets Destroyed

| Storage Type | What Gets Deleted |
|-------------|------------------|
| AWS EBS | The EBS volume (all data gone) |
| GCP PD | The persistent disk |
| Azure Disk | The managed disk |
| NFS | Nothing (NFS export still exists, but PV is gone) |
| Local volume | Nothing (files remain on node, PV object removed) |
| CSI driver | Whatever `DeleteVolume()` does in the driver |

## Reclaim Policy: Recycle (Deprecated)

```yaml
persistentVolumeReclaimPolicy: Recycle
```

**What happens:**

```
PVC deleted
    │
    ▼
PV status → Released
    │
    ▼
PV Controller runs: rm -rf /volume/*
    (scrubs the volume data)
    │
    ▼
PV status → Available
    (can be bound to a new PVC)
```

**Deprecated since Kubernetes 1.28.** Use dynamic provisioning with `Delete` policy instead, or `Retain` with manual cleanup.

Recycle only works with NFS and hostPath volumes. It literally runs `rm -rf` inside a recycler pod.

## StorageClass and Default Reclaim Policy

When PVs are dynamically provisioned, they inherit the reclaim policy from the StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete          # PVs created by this class use "Delete"
volumeBindingMode: WaitForFirstConsumer
```

| StorageClass reclaimPolicy | PVs created | On PVC deletion |
|----------------------------|-------------|-----------------|
| `Delete` (default) | Get `persistentVolumeReclaimPolicy: Delete` | Volume destroyed |
| `Retain` | Get `persistentVolumeReclaimPolicy: Retain` | Volume preserved |

```bash
# Check StorageClass reclaim policy:
kubectl get storageclass -o custom-columns=NAME:.metadata.name,RECLAIM:.reclaimPolicy

# Change an existing PV's reclaim policy (does not affect StorageClass):
kubectl patch pv pv-001 -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

## Volume Detachment Before Reclaim

Before the reclaim policy runs, the volume must be detached from the node:

```
Pod deleted (or node drained)
    │
    ▼
Kubelet unmounts the volume from the pod
    │
    ▼
CSI Node Driver: NodeUnpublishVolume + NodeUnstageVolume
    │
    ▼
Attach/Detach Controller: detaches volume from node
    │
    ▼
CSI Controller: ControllerUnpublishVolume
    │
    ▼
VolumeAttachment object deleted
    │
    ▼
PVC can now be deleted safely
    │
    ▼
Reclaim policy runs on the PV
```

```bash
# Check if a volume is still attached:
kubectl get volumeattachments | grep <pv-name>

# Stuck VolumeAttachment prevents reclaim:
kubectl describe volumeattachment <name>
```

## Finalizers — Preventing Premature Deletion

PVs and PVCs have finalizers that prevent deletion until safe:

```yaml
# PV finalizer:
metadata:
  finalizers:
  - kubernetes.io/pv-protection
  # Prevents PV deletion while bound to a PVC

# PVC finalizer:
metadata:
  finalizers:
  - kubernetes.io/pvc-protection
  # Prevents PVC deletion while mounted by a pod
```

### Protection Flow

```
kubectl delete pvc my-pvc
    │
    ├── Is the PVC mounted by any pod?
    │     YES → PVC enters Terminating but deletion is BLOCKED
    │           (finalizer prevents removal from etcd)
    │           Waits until no pod references it
    │     NO  → PVC deleted, PV enters Released
    │
    ▼

kubectl delete pv my-pv
    │
    ├── Is the PV bound to a PVC?
    │     YES → PV enters Terminating but deletion is BLOCKED
    │           Waits until PVC is deleted first
    │     NO  → PV deleted (and underlying volume if Delete policy)
```

```bash
# PVC stuck Terminating (still mounted):
kubectl get pvc my-pvc
# NAME     STATUS        VOLUME   CAPACITY
# my-pvc   Terminating   pv-001   10Gi

# Find which pod is still using it:
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName == "my-pvc") | .metadata.name'

# Force remove finalizer (data loss risk!):
kubectl patch pvc my-pvc -p '{"metadata":{"finalizers":null}}'
```

## Static vs Dynamic Provisioning Reclaim

| Scenario | PV Creation | Typical Reclaim | Data Safety |
|----------|------------|-----------------|-------------|
| Dynamic (StorageClass) | Automatic | Delete (default) | Data lost on PVC delete |
| Dynamic (StorageClass + Retain) | Automatic | Retain | Data preserved, manual cleanup |
| Static (admin creates PV) | Manual | Retain (typical) | Admin manages lifecycle |

## Common Patterns

### Protect Production Data (Retain + Backup)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: production-ebs
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain              # Never auto-delete volumes
parameters:
  type: gp3
```

### Dev/Test (Delete — Ephemeral)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: dev-ebs
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete              # Clean up automatically
parameters:
  type: gp3
```

### Patch Existing PVs to Retain (Emergency)

```bash
# Change all PVs in a namespace to Retain:
kubectl get pv -o name | xargs -I {} kubectl patch {} \
  -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

## Debugging

```bash
# Check PV status and reclaim policy:
kubectl get pv -o custom-columns=\
  NAME:.metadata.name,\
  STATUS:.status.phase,\
  RECLAIM:.spec.persistentVolumeReclaimPolicy,\
  CLAIM:.spec.claimRef.name

# Check for stuck Released PVs:
kubectl get pv --field-selector status.phase=Released

# Check VolumeAttachments (stuck detach blocks reclaim):
kubectl get volumeattachments

# Check PVC finalizers:
kubectl get pvc <name> -o jsonpath='{.metadata.finalizers}'

# Check PV finalizers:
kubectl get pv <name> -o jsonpath='{.metadata.finalizers}'

# PV controller logs:
kubectl logs -n kube-system -l component=kube-controller-manager --tail=50 | grep -i "pv\|volume\|reclaim"
```

## Quick Reference

```bash
# Reclaim policies:
# Retain  → PV stays Released, volume and data preserved, manual cleanup needed
# Delete  → PV deleted, underlying volume destroyed, data permanently lost
# Recycle → (deprecated) rm -rf on volume, PV becomes Available

# Lifecycle:
# Available → Bound (PVC binds) → Released (PVC deleted) → reclaim action

# Key protections:
# kubernetes.io/pvc-protection — blocks PVC delete while pod uses it
# kubernetes.io/pv-protection  — blocks PV delete while PVC is bound

# Re-use a Retained PV:
kubectl patch pv <name> --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'

# Change reclaim policy:
kubectl patch pv <name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# StorageClass sets default reclaim policy for dynamically provisioned PVs
# Default is "Delete" if not specified in StorageClass
```
