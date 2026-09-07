# Kubernetes Node Disk Pressure

## Overview

**Node Disk Pressure** is a node condition that indicates a node is running low on available disk space. When this happens, Kubernetes takes protective measures to prevent the node from running out of disk entirely.

## What Triggers Disk Pressure?

Disk pressure is triggered when:

- **Ephemeral storage** (used by containers, logs, emptyDir volumes) exceeds thresholds
- **Image filesystem** (where container images are stored) is running low
- **Node filesystem** (root filesystem) is running low

### Default Thresholds

- **Soft eviction**: 85% disk usage (configurable via `--eviction-soft`)
- **Hard eviction**: 90% disk usage (configurable via `--eviction-hard`)

## What Happens When Disk Pressure Occurs?

1. **Node is marked with a taint**: `node.kubernetes.io/disk-pressure:NoSchedule`
2. **No new pods are scheduled** on that node (unless they tolerate the taint)
3. **Pod eviction begins**: Kubelet starts evicting pods to free up disk space
   - Eviction order: BestEffort pods → Burstable pods → Guaranteed pods
   - Pods using the most disk space are evicted first

## Common Causes

- Container logs growing too large
- Too many container images stored locally
- Application writing excessive data to ephemeral storage
- Lack of log rotation
- Large emptyDir volumes
- Build artifacts or temporary files accumulating

## How to Check for Disk Pressure

### Check node conditions

```bash
kubectl describe node <node-name> | grep -A 5 Conditions
```

### Check all nodes for disk pressure

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="DiskPressure")].status}{"\n"}{end}'
```

### Check node disk usage details

```bash
kubectl describe node <node-name> | grep -A 10 "Allocated resources"
```

### SSH into the node and check disk usage (if accessible)

```bash
df -h
du -sh /var/lib/docker
du -sh /var/lib/kubelet
```

## How to Resolve Disk Pressure

### 1. Clean up unused images

```bash
# On the node
docker system prune -a

# Or using crictl (for containerd)
crictl rmi --prune
```

### 2. Clean up unused containers

```bash
docker container prune
# Or
crictl rmp
```

### 3. Configure log rotation

```yaml
# In kubelet configuration
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
containerLogMaxSize: "10Mi"
containerLogMaxFiles: 5
```

### 4. Set ephemeral storage limits in pod specs

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: nginx
    resources:
      limits:
        ephemeral-storage: "2Gi"
      requests:
        ephemeral-storage: "1Gi"
```

### 5. Use persistent volumes instead of emptyDir

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

### 6. Increase disk size

- Add more storage to the node
- Scale up to larger instance types with more disk
- Add additional volumes and configure kubelet to use them

### 7. Configure kubelet eviction thresholds

```yaml
# In kubelet configuration
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
evictionHard:
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"
evictionSoft:
  nodefs.available: "15%"
  imagefs.available: "20%"
evictionSoftGracePeriod:
  nodefs.available: "1m"
  imagefs.available: "1m"
```

## Prevention Best Practices

1. **Monitor disk usage**: Set up alerts before hitting thresholds.

   ```yaml
   # Prometheus alert example
   - alert: NodeDiskPressure
     expr: kube_node_status_condition{condition="DiskPressure",status="true"} == 1
     for: 5m
   ```

2. **Implement log aggregation**: Use centralized logging (Fluentd, Filebeat, etc.).

3. **Set resource quotas**: Limit ephemeral storage at the namespace level.

   ```yaml
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: storage-quota
   spec:
     hard:
       requests.ephemeral-storage: "50Gi"
       limits.ephemeral-storage: "100Gi"
   ```

4. **Regular cleanup jobs**: Schedule CronJobs to clean up old data.

5. **Use image pull policies wisely**: Set `imagePullPolicy: IfNotPresent` to avoid duplicate images.

## Troubleshooting Commands

```bash
# Check which pods are using the most ephemeral storage
kubectl get pods -A -o json | jq '.items[] | {name: .metadata.name, namespace: .metadata.namespace, ephemeral: .spec.containers[].resources.requests."ephemeral-storage"}'

# Check node allocatable resources
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, allocatable: .status.allocatable}'

# View kubelet logs for eviction events
journalctl -u kubelet | grep -i evict

# Check for disk pressure events
kubectl get events --all-namespaces --field-selector reason=NodeDiskPressure
```

## Related Node Conditions

- **MemoryPressure**: Node is running low on memory
- **PIDPressure**: Node is running low on available process IDs
- **NetworkUnavailable**: Node's network is not correctly configured

## References

- [Kubernetes Node Conditions](https://kubernetes.io/docs/concepts/architecture/nodes/#condition)
- [Kubelet Eviction Policies](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [Resource Management for Pods](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

## Skills Practiced

- Recognizing the DiskPressure node condition and its eviction behavior
- Diagnosing disk pressure with `kubectl describe node`, JSONPath, and node-level `df`/`du`
- Freeing space via image/container pruning and log rotation
- Preventing recurrence with ephemeral-storage limits, quotas, and eviction thresholds
