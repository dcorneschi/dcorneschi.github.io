<img src="/articles/images/kubernetes-logo.svg" alt="Kubernetes" width="150">

How to set up NFS storage for MicroK8s using the NFS CSI driver for persistent volumes in a homelab.

### Overview

MicroK8s doesn't ship with NFS support out of the box. By installing the NFS CSI driver you can provision PersistentVolumes backed by an NFS server on your LAN — ideal for a homelab where you want shared, network-attached storage across nodes without cloud provider block storage.

### Prerequisites

- MicroK8s with `helm3` and `dns` addons enabled
- An NFS server accessible from the MicroK8s node(s)
- NFS client utilities installed on each node

### Install NFS Client Utilities

Every MicroK8s node needs the NFS client packages to mount shares:

```bash
# Ubuntu/Debian
sudo apt install nfs-common

# Fedora/RHEL
sudo dnf install nfs-utils

# Arch Linux
sudo pacman -S nfs-utils
```

### Set Up the NFS Server (if needed)

If you don't already have an NFS server, set one up on a separate machine:

```bash
# Install NFS server
sudo apt install nfs-kernel-server

# Create the export directory
sudo mkdir -p /srv/nfs/k8s
sudo chown nobody:nogroup /srv/nfs/k8s
sudo chmod 0777 /srv/nfs/k8s

# Export the directory
echo "/srv/nfs/k8s 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Apply and start
sudo exportfs -rav
sudo systemctl enable --now nfs-kernel-server
```

Verify the export:

```bash
showmount -e localhost
```

### Install the NFS CSI Driver

```bash
microk8s enable helm3

microk8s helm3 repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
microk8s helm3 repo update

microk8s helm3 install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --set kubeletDir=/var/snap/microk8s/common/var/lib/kubelet
```

The `kubeletDir` override is required because MicroK8s uses a non-standard kubelet path.

### Verify the Driver

```bash
# Check CSI driver pods
microk8s kubectl get pods -n kube-system | grep csi-nfs

# Check the CSI driver is registered
microk8s kubectl get csidrivers
```

You should see `nfs.csi.k8s.io` in the CSI drivers list.

### Create a StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.1.50
  share: /srv/nfs/k8s
reclaimPolicy: Retain
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=4.1
```

```bash
microk8s kubectl apply -f nfs-storageclass.yaml
```

Optionally set it as the default StorageClass:

```bash
microk8s kubectl patch storageclass nfs-csi -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Test with a PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-nfs-pvc
spec:
  storageClassName: nfs-csi
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: test-nfs-pod
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - mountPath: /data
          name: nfs-volume
  volumes:
    - name: nfs-volume
      persistentVolumeClaim:
        claimName: test-nfs-pvc
```

```bash
microk8s kubectl apply -f test-nfs.yaml

# Verify the PVC is bound
microk8s kubectl get pvc test-nfs-pvc

# Write a test file
microk8s kubectl exec test-nfs-pod -- sh -c "echo 'hello nfs' > /data/test.txt"

# Verify on the NFS server
cat /srv/nfs/k8s/pvc-*/test.txt
```

### Clean Up Test Resources

```bash
microk8s kubectl delete pod test-nfs-pod
microk8s kubectl delete pvc test-nfs-pvc
```

### Troubleshooting

| Issue | Fix |
|-------|-----|
| PVC stuck in `Pending` | Check CSI driver pods are running: `microk8s kubectl get pods -n kube-system \| grep csi-nfs` |
| Mount failed: no such file | NFS client not installed on the node — install `nfs-common` |
| Permission denied | Check NFS export options — ensure the node IP is in the allowed range and `no_root_squash` is set |
| Stale file handle | NFS server was restarted — delete and recreate the pod, or remount |
| Timeout mounting | Verify connectivity: `showmount -e <nfs-server-ip>` from the node |
| Wrong kubelet path | Must set `kubeletDir=/var/snap/microk8s/common/var/lib/kubelet` during helm install |

### Related Commands

- `showmount -e <server-ip>` — list NFS exports from a server
- `microk8s kubectl get pv` — list persistent volumes
- `microk8s kubectl get pvc -A` — list all persistent volume claims
- `microk8s kubectl describe pvc <name>` — debug PVC binding issues
- `microk8s kubectl get csidrivers` — verify CSI driver registration
- `exportfs -rav` — reload NFS exports on the server
