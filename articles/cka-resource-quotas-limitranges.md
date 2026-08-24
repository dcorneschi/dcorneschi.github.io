# CKA Resource - Quotas & LimitRanges

## Step 1: Create a Namespace

```bash
kubectl create namespace quota-lab
```

## Step 2: Apply a ResourceQuota

Create a file `quota.yaml` with the following content:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: quota-lab
spec:
  hard:
    pods: "3"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

Apply it:

```bash
kubectl apply -f quota.yaml -n quota-lab
kubectl describe quota -n quota-lab
```

This quota restricts:

- Max 3 pods
- CPU requests total ≤ 1 core
- Memory requests total ≤ 1Gi
- CPU limits total ≤ 2 cores
- Memory limits total ≤ 2Gi

## Step 3: Apply a LimitRange

Create a file `limitrange.yaml` with the following content:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: quota-lab
spec:
  limits:
  - default:
      cpu: "500m"
      memory: 512Mi
    defaultRequest:
      cpu: "200m"
      memory: 256Mi
    type: Container
```

Apply it:

```bash
kubectl apply -f limitrange.yaml -n quota-lab
kubectl describe limitrange -n quota-lab
```

This defines:

- Default request: 200m CPU, 256Mi memory
- Default limit: 500m CPU, 512Mi memory

## Step 4: Test with a Pod Without Resources

Create `pod-noresources.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-noresources
  namespace: quota-lab
spec:
  containers:
  - name: nginx
    image: nginx
```

Apply it:

```bash
kubectl apply -f pod-noresources.yaml
kubectl describe pod pod-noresources -n quota-lab
```

You should see default CPU/Memory values injected from the LimitRange.

## Step 5: Try to Exceed the Quota

Create `pod-large.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-large
  namespace: quota-lab
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "2"
        memory: 2Gi
      limits:
        cpu: "3"
        memory: 3Gi
```

Apply it:

```bash
kubectl apply -f pod-large.yaml
```

You should see a failure because it exceeds the namespace ResourceQuota.

## Skills Practiced

- Enforcing quotas with ResourceQuota
- Applying default requests/limits with LimitRange
- Debugging pod scheduling errors due to exceeded quotas
