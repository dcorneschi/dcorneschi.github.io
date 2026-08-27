# Kubernetes CSI Driver Architecture

How Container Storage Interface (CSI) drivers plug into Kubernetes — the sidecar containers, gRPC socket communication, controller vs node components, and the volume lifecycle from provisioning to mounting.

Note: For EBS CSI provisioning workflow and StorageClass configuration, see the EKS persistent volumes guide. This article covers the generic CSI specification and architecture.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Control Plane (CSI Controller — Deployment/StatefulSet)            │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │ external-        │  │ external-        │  │ CSI Driver       │   │
│  │ provisioner      │  │ attacher         │  │ (controller mode)│   │
│  │ (sidecar)        │  │ (sidecar)        │  │                  │   │
│  │                  │  │                  │  │ Implements:      │   │ 
│  │ Watches PVCs     │  │ Watches VA       │  │ CreateVolume     │   │
│  │ Calls            │  │ objects          │  │ DeleteVolume     │   │
│  │ CreateVolume     │  │ Calls            │  │ ControllerPublish│   │
│  │                  │  │ ControllerPublish│  │ ontrollerUnpub   │   │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘   │
│           │                     │                     │             │
│           └─────────────────────┼─────────────────────┘             │
│                                 │                                   │
│                          CSI Socket (gRPC)                          │
│                          /var/lib/csi/sockets/                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Worker Node (CSI Node — DaemonSet)                                 │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │ node-driver-     │  │ liveness-probe   │  │ CSI Driver       │   │
│  │ registrar        │  │ (sidecar)        │  │ (node mode)      │   │
│  │ (sidecar)        │  │                  │  │                  │   │
│  │                  │  │ Health checks    │  │  Implements:     │   │
│  │ Registers driver │  │ the CSI driver   │  │  NodeStageVolume │   │
│  │ with kubelet     │  │                  │  │  NodePublishVol  │   │
│  │                  │  │                  │  │  NodeGetInfo     │   │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘   │
│           │                     │                     │             │
│           └─────────────────────┼─────────────────────┘             │
│                                 │                                   │
│                          CSI Socket (gRPC)                          │
│                          /var/lib/kubelet/plugins/<driver>/csi.sock │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## The CSI Specification

CSI is a standard gRPC interface that storage vendors implement. Kubernetes doesn't know how to talk to AWS EBS, GCP PD, or Ceph — the CSI driver handles that:

```
┌───────────────┐                    ┌──────────────────┐
│  Kubernetes   │── gRPC calls ─────▶│  CSI Driver      │
│  (sidecars)   │                    │ (vendor-specific)│
│               │◀── gRPC responses ─│                  │
└───────────────┘                    └──────────────────┘
                                            │
                                            ▼
                                     ┌──────────────────┐
                                     │  Storage Backend │
                                     │  (AWS EBS, Ceph, │
                                     │   NFS, etc.)     │
                                     └──────────────────┘
```

### CSI Services (gRPC Interfaces)

| Service | Methods | Where It Runs |
|---------|---------|--------------|
| **Identity** | GetPluginInfo, GetPluginCapabilities, Probe | Both controller + node |
| **Controller** | CreateVolume, DeleteVolume, ControllerPublishVolume, ControllerUnpublishVolume, ListVolumes, CreateSnapshot, DeleteSnapshot | Controller pod only |
| **Node** | NodeStageVolume, NodeUnstageVolume, NodePublishVolume, NodeUnpublishVolume, NodeGetInfo, NodeGetCapabilities | Node DaemonSet only |

## Sidecar Containers — The Kubernetes-CSI Bridge

The CSI driver itself only implements the gRPC interface. It doesn't watch Kubernetes objects. That's the job of **sidecar containers** maintained by the Kubernetes CSI community:

### external-provisioner

```
┌──────────────────────────────────────────────────────────────┐
│  external-provisioner                                        │
│                                                              │
│  Watches: PersistentVolumeClaims (PVCs)                      │
│                                                              │
│  When: New PVC with matching StorageClass                    │
│  Action: Calls CSI CreateVolume()                            │
│  Then: Creates a PV object bound to the PVC                  │
│                                                              │
│  When: PV reclaim policy = Delete and PVC deleted            │
│  Action: Calls CSI DeleteVolume()                            │
│  Then: Deletes the PV object                                 │
└──────────────────────────────────────────────────────────────┘
```

### external-attacher

```
┌──────────────────────────────────────────────────────────────┐
│  external-attacher                                           │
│                                                              │
│  Watches: VolumeAttachment objects                           │
│                                                              │
│  When: New VolumeAttachment (pod scheduled, needs volume)    │
│  Action: Calls CSI ControllerPublishVolume()                 │
│  Result: Volume attached to the node (e.g., EBS attached)    │
│                                                              │
│  When: VolumeAttachment deleted (pod gone)                   │
│  Action: Calls CSI ControllerUnpublishVolume()               │
│  Result: Volume detached from node                           │
└──────────────────────────────────────────────────────────────┘
```

### external-resizer

```
┌──────────────────────────────────────────────────────────────┐
│  external-resizer                                            │
│                                                              │
│  Watches: PVCs with spec.resources.requests > status.capacity│
│                                                              │
│  When: User increases PVC size                               │
│  Action: Calls CSI ControllerExpandVolume()                  │
│  Result: Backend volume resized (e.g., EBS volume grows)     │
│  Then: Kubelet calls NodeExpandVolume() for filesystem resize│
└──────────────────────────────────────────────────────────────┘
```

### external-snapshotter

```
┌──────────────────────────────────────────────────────────────┐
│  external-snapshotter                                        │
│                                                              │
│  Watches: VolumeSnapshot objects                             │
│                                                              │
│  When: New VolumeSnapshot created                            │
│  Action: Calls CSI CreateSnapshot()                          │
│  Result: Storage backend takes a snapshot                    │
│  Then: Creates VolumeSnapshotContent object                  │
└──────────────────────────────────────────────────────────────┘
```

### node-driver-registrar

```
┌──────────────────────────────────────────────────────────────┐
│  node-driver-registrar (runs on every node)                  │
│                                                              │
│  On startup:                                                 │
│    Registers the CSI driver with the kubelet                 │
│    via the kubelet plugin registration mechanism             │
│    (/var/lib/kubelet/plugins_registry/)                      │
│                                                              │
│  Tells kubelet: "driver X is available at this socket path"  │
└──────────────────────────────────────────────────────────────┘
```

### liveness-probe

```
┌──────────────────────────────────────────────────────────────┐
│  liveness-probe                                              │
│                                                              │
│  Calls CSI Probe() method periodically                       │
│  Exposes /healthz HTTP endpoint for Kubernetes liveness probe│
│  If CSI driver is unresponsive → pod gets restarted          │
└──────────────────────────────────────────────────────────────┘
```

## The gRPC Socket

All communication between sidecars and the CSI driver happens via a Unix domain socket shared through an `emptyDir` volume:

```yaml
volumes:
- name: socket-dir
  emptyDir: {}

# Sidecars and driver all mount the same volume:
containers:
- name: csi-driver
  volumeMounts:
  - name: socket-dir
    mountPath: /csi
- name: external-provisioner
  args: ["--csi-address=/csi/csi.sock"]
  volumeMounts:
  - name: socket-dir
    mountPath: /csi
```

```bash
# On a node, see the CSI socket:
ls /var/lib/kubelet/plugins/ebs.csi.aws.com/csi.sock
```

## Volume Lifecycle — Complete Flow

### Provisioning (PVC → Volume Created)

```
1. User creates PVC
    │
    ▼
2. external-provisioner sees PVC
    │ (matches StorageClass provisioner name)
    ▼
3. Calls gRPC: CreateVolume(name, capacity, parameters)
    │
    ▼
4. CSI driver calls storage API (e.g., AWS ec2:CreateVolume)
    │
    ▼
5. Returns volume ID to provisioner
    │
    ▼
6. Provisioner creates PV object with volumeHandle = volume ID
    │
    ▼
7. PV controller binds PV ↔ PVC
```

### Attaching (Pod Scheduled → Volume Attached to Node)

```
1. Pod is scheduled to node-1
    │
    ▼
2. Attach/Detach controller creates VolumeAttachment object:
   {spec: {source: pv-001, nodeName: node-1}}
    │
    ▼
3. external-attacher sees VolumeAttachment
    │
    ▼
4. Calls gRPC: ControllerPublishVolume(volumeID, nodeID)
    │
    ▼
5. CSI driver attaches volume (e.g., AWS ec2:AttachVolume)
    │
    ▼
6. external-attacher updates VolumeAttachment status: attached=true
```

### Staging + Mounting (Kubelet → Volume Usable by Pod)

```
7. Kubelet sees volume is attached (VolumeAttachment.status.attached=true)
    │
    ▼
8. Kubelet calls gRPC: NodeStageVolume(volumeID, stagingPath)
    │ stagingPath: /var/lib/kubelet/plugins/kubernetes.io/csi/pv/<pv-name>/globalmount
    ▼
9. CSI driver: formats volume (if needed) and mounts to staging path
    │ (e.g., mkfs.ext4 /dev/xvdba && mount /dev/xvdba /staging/path)
    ▼
10. Kubelet calls gRPC: NodePublishVolume(volumeID, stagingPath, targetPath)
    │ targetPath: /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/<pv-name>/mount
    ▼
11. CSI driver: bind-mounts from staging path to target path
    │
    ▼
12. Kubelet mounts target path into pod's container filesystem
    │
    ▼
13. Pod container sees volume at its spec.containers[].volumeMounts[].mountPath
```

### Two-Stage Mount — Why?

```
┌────────────────────────────────────────────────────────────────┐
│  Why NodeStageVolume AND NodePublishVolume?                    │
│                                                                │
│  Stage (global mount):                                         │
│    - Format + mount the device ONCE per node                   │
│    - Even if multiple pods on the same node use the same PV    │
│    - Expensive operations (format, fsck) happen once           │
│                                                                │
│  Publish (per-pod mount):                                      │
│    - Bind-mount the staged volume into each pod                │
│    - Cheap operation (just a bind mount)                       │
│    - Each pod gets its own mount point                         │
│                                                                │
│  Result: ReadWriteMany volumes mounted by multiple pods on     │
│  the same node only stage once but publish multiple times      │
└────────────────────────────────────────────────────────────────┘
```

## VolumeAttachment Object

The bridge between "volume exists" and "volume is attached to a node":

```yaml
apiVersion: storage.k8s.io/v1
kind: VolumeAttachment
metadata:
  name: csi-att-abc123
spec:
  attacher: ebs.csi.aws.com
  source:
    persistentVolumeName: pv-001
  nodeName: ip-10-0-1-50.ec2.internal
status:
  attached: true
  attachmentMetadata:
    devicePath: /dev/xvdba
```

```bash
# List all volume attachments:
kubectl get volumeattachments

# Check if a specific volume is attached:
kubectl get volumeattachments -o json | jq '.items[] | select(.spec.source.persistentVolumeName == "pv-001")'
```

## CSIDriver Object

Registers the driver's capabilities with Kubernetes:

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: ebs.csi.aws.com
spec:
  attachRequired: true           # Needs ControllerPublish (attach to node)
  podInfoOnMount: false          # Don't pass pod info to NodePublishVolume
  fsGroupPolicy: File            # Apply fsGroup to mounted files
  volumeLifecycleModes:
  - Persistent                   # Supports PV/PVC
  - Ephemeral                    # Supports inline ephemeral volumes
  storageCapacity: true          # Reports capacity via CSIStorageCapacity
  requiresRepublish: false       # No need to call NodePublishVolume again
```

```bash
# List installed CSI drivers:
kubectl get csidrivers

# See driver capabilities:
kubectl describe csidriver ebs.csi.aws.com
```

## CSINode Object

One per node — reports which CSI drivers are registered on that node:

```yaml
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: ip-10-0-1-50.ec2.internal
spec:
  drivers:
  - name: ebs.csi.aws.com
    nodeID: i-0abc123def456      # Node identifier for the storage system
    topologyKeys:
    - topology.ebs.csi.aws.com/zone
    allocatable:
      count: 25                  # Max volumes attachable to this node
```

```bash
# Check what drivers a node has:
kubectl get csinode <node-name> -o yaml

# Check max volume count per node:
kubectl get csinode -o custom-columns=NODE:.metadata.name,DRIVERS:.spec.drivers[*].name,MAX:.spec.drivers[*].allocatable.count
```

## Common CSI Drivers

| Driver | Storage | Provisioner Name |
|--------|---------|------------------|
| AWS EBS CSI | Amazon EBS | `ebs.csi.aws.com` |
| AWS EFS CSI | Amazon EFS | `efs.csi.aws.com` |
| GCP PD CSI | Google Persistent Disk | `pd.csi.storage.gke.io` |
| Azure Disk CSI | Azure Managed Disk | `disk.csi.azure.com` |
| Azure File CSI | Azure Files (SMB/NFS) | `file.csi.azure.com` |
| Ceph RBD CSI | Ceph Block | `rbd.csi.ceph.com` |
| CephFS CSI | CephFS | `cephfs.csi.ceph.com` |
| NFS CSI | NFS shares | `nfs.csi.k8s.io` |
| Longhorn | Longhorn distributed | `driver.longhorn.io` |
| OpenEBS | OpenEBS | `cstor.csi.openebs.io` |

## Debugging CSI Issues

```bash
# Check CSI driver pods are running:
kubectl get pods -n kube-system -l app=ebs-csi-controller
kubectl get pods -n kube-system -l app=ebs-csi-node

# Check CSI driver registration:
kubectl get csidrivers
kubectl get csinodes

# Check VolumeAttachment status:
kubectl get volumeattachments
kubectl describe volumeattachment <name>

# CSI controller logs (provisioning/attaching):
kubectl logs -n kube-system -l app=ebs-csi-controller -c csi-provisioner --tail=20
kubectl logs -n kube-system -l app=ebs-csi-controller -c csi-attacher --tail=20
kubectl logs -n kube-system -l app=ebs-csi-controller -c ebs-plugin --tail=20

# CSI node logs (staging/mounting):
kubectl logs -n kube-system -l app=ebs-csi-node -c ebs-plugin --tail=20

# Check PVC events for provisioning errors:
kubectl describe pvc <name>

# Check pod events for mount errors:
kubectl describe pod <name> | grep -A5 "Events"

# Common errors:
# "driver name not found" → CSI driver DaemonSet not running on the node
# "FailedAttachVolume" → external-attacher can't call ControllerPublishVolume
# "FailedMount" → NodeStageVolume or NodePublishVolume failed
# "volume already attached to a different node" → Force detach needed (EBS is single-attach)
```

## Quick Reference

```bash
# CSI architecture:
# Controller (Deployment): external-provisioner + external-attacher + driver
# Node (DaemonSet): node-driver-registrar + liveness-probe + driver

# Communication: gRPC over Unix socket (emptyDir shared between containers)

# Volume lifecycle:
# CreateVolume → ControllerPublishVolume → NodeStageVolume → NodePublishVolume
# (provision)     (attach to node)          (format+mount)    (bind-mount to pod)

# Reverse:
# NodeUnpublishVolume → NodeUnstageVolume → ControllerUnpublishVolume → DeleteVolume
# (unbind from pod)      (unmount)           (detach from node)          (delete volume)

# Key objects:
kubectl get csidrivers              # Registered drivers
kubectl get csinodes                # Per-node driver info + max volumes
kubectl get volumeattachments       # Volume-to-node attachment state
kubectl get storageclasses          # Provisioner + parameters

# Debugging:
kubectl logs -n kube-system <csi-controller-pod> -c csi-provisioner
kubectl logs -n kube-system <csi-controller-pod> -c csi-attacher
kubectl logs -n kube-system <csi-node-pod> -c <driver-container>
kubectl describe pvc <name>         # Provisioning events
kubectl describe pod <name>         # Mount events
```
