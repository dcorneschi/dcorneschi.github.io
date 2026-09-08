# Dynamic PV/PVC Provisioning with a StorageClass

This guide explains how Kubernetes automatically creates a PersistentVolume (PV)
when you create a PersistentVolumeClaim (PVC) that references a StorageClass.

## Do PVs Get Created Automatically?

Yes. If you don't pre-create a PV, one is provisioned automatically as long as a
StorageClass with a working provisioner exists and your PVC references it. You
create the PVC; the provisioner creates the PV.

## Dynamic Provisioning Flow

1. You create only a **PVC** (e.g. via a Helm chart) with `storageClassName: nfs-client`.
2. Kubernetes sees the PVC needs storage and it's `Pending`.
3. The StorageClass provisioner (here `nfs-subdir-external-provisioner`) creates a **PV**.
4. The PVC binds to the new PV, and the pod mounts it.

## What You Specify vs What Gets Created

You provide only the PVC (often through Helm values):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: portainer-pvc
spec:
  storageClassName: nfs-client   # references the StorageClass
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
```

Kubernetes automatically creates the matching PV:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvc-abc123def-456-789          # auto-generated
spec:
  capacity:
    storage: 10Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain # inherited from the StorageClass
  storageClassName: nfs-client
  nfs:
    server: 192.168.50.9
    path: /volume1/docker/microk8s/portainer-portainer-pvc-abc123def  # auto-generated
```

## Requirements for Dynamic Provisioning

- A **StorageClass exists** (e.g. `nfs-client`).
- Its **provisioner is running** (e.g. `nfs-subdir-external-provisioner`).
- The **PVC references that StorageClass**.

You do **not** need to create the PersistentVolume or (for the NFS provisioner)
the backing directory — both are created for you.

## Real Example: Portainer on MicroK8s

Helm values (`portainer-values-simple.yaml`):

```yaml
persistence:
  enabled: true
  storageClass: "nfs-client"   # triggers dynamic provisioning
  accessMode: ReadWriteOnce
  size: 10Gi
```

Install:

```bash
microk8s helm3 install portainer portainer/portainer \
  --namespace portainer \
  --values portainer-values-simple.yaml
```

What happens:

1. Helm creates a PVC with `storageClassName: nfs-client`.
2. The NFS provisioner sees the pending PVC.
3. It creates a PV (auto-generated name) and the NFS directory
   `/volume1/docker/microk8s/portainer-portainer-pvc-<random-id>`.
4. The PVC binds to the new PV.
5. The Portainer pod mounts the volume.

## Verification

```bash
# PVC (created by Helm)
microk8s kubectl get pvc -n portainer

# PV (created automatically by the provisioner)
microk8s kubectl get pv

# The NFS directory (created automatically) — run on the NFS server
ls -la /volume1/docker/microk8s/
```

## Dynamic vs Static

| Approach | PV creation | PVC creation | NFS path |
|----------|-------------|--------------|----------|
| Dynamic | Automatic | By Helm/user | Auto-generated |
| Static | Manual | Manual | Fixed |

Benefits of dynamic provisioning: no PV/PVC YAML to hand-write, each app gets its
own uniquely named storage, and it scales without path conflicts.

## When It Happens

- A PVC is created with a `storageClassName`.
- That StorageClass exists and has a provisioner.
- The provisioner is running and healthy.
- The PVC is `Pending`, waiting for storage.

## When It Doesn't Happen

- No StorageClass specified, or `storageClassName: ""` (explicitly disables dynamic provisioning).
- The StorageClass doesn't exist.
- The provisioner isn't running.
- The PVC is already bound to an existing PV.

> If a PVC stays `Pending`, check `kubectl describe pvc <name>` for provisioner
> events, and confirm the provisioner pod is healthy and the StorageClass name
> matches exactly.

## Related

- [Persistent Volumes on EKS with EBS CSI Driver](articles/eks-persistent-volumes-ebs-csi.md)
- [NFS Storage for MicroK8s](articles/nfs-microk8s-installation.md)
- [Resource Quotas & LimitRanges](articles/kubernetes-resource-quotas-limitranges.md)

## Skills Practiced

- Understanding how a PVC + StorageClass triggers automatic PV provisioning
- Reading what the provisioner fills in (name, path, reclaim policy) on the PV
- Configuring dynamic storage through Helm values instead of raw PV/PVC YAML
- Diagnosing why a PVC stays `Pending` when provisioning doesn't fire
