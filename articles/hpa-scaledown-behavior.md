# HPA with scaleDown Behavior

Create a Horizontal Pod Autoscaler targeting CPU utilization with a custom
`scaleDown` stabilization window using `autoscaling/v2`.

## Setup

### 1. Create the namespace

```bash
kubectl create namespace scaling-lab
```

### 2. Create the deployment

```bash
kubectl create deployment web-frontend \
  --image=nginx:1.27 \
  --replicas=2 \
  -n scaling-lab

kubectl set resources deployment web-frontend \
  --requests=cpu=100m \
  --limits=cpu=500m \
  -n scaling-lab
```

> **Important:** HPA calculates CPU percentage based on `resources.requests.cpu`. Without a CPU request, HPA shows `<unknown>` and never scales.

### 3. Expose the deployment

```bash
kubectl expose deployment web-frontend \
  --port=80 --target-port=80 \
  --type=ClusterIP \
  -n scaling-lab
```

Service FQDN: `web-frontend.scaling-lab.svc.cluster.local`

```bash
# Test DNS resolution
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -n scaling-lab -- nslookup web-frontend
```

### 4. Generate the HPA manifest

```bash
kubectl autoscale deployment web-frontend \
  --cpu=60% \
  --min=2 \
  --max=8 \
  -n scaling-lab \
  --dry-run=client -o yaml > hpa.yaml
```

> **Note:** `--dry-run=client` doesn't contact the API server, so the generated YAML won't include the `namespace` field in metadata. You'll need to add `namespace: scaling-lab` manually. Using `--dry-run=server` instead will include the namespace automatically, but requires the namespace and deployment to already exist in the cluster.

### 5. Edit hpa.yaml — add scaleDown behavior

Open the generated file and add the `behavior` block so the final result
looks like this:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-frontend
  namespace: scaling-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-frontend
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 45
```

> **Why this matters:** Without a custom `scaleDown` behavior, HPA uses the default 300s (5 min) stabilization window. Setting it to 45s means replicas are removed faster after load drops — useful in dev/test to save resources. In production, a longer window prevents flapping when traffic is spiky.

### 6. Validate the manifest

```bash
kubectl apply -f hpa.yaml --dry-run=server
```

### 7. Apply

```bash
kubectl apply -f hpa.yaml
```

## Verification

```bash
kubectl get hpa web-frontend -n scaling-lab
kubectl get hpa web-frontend -n scaling-lab -o yaml | grep -A3 scaleDown
kubectl describe hpa web-frontend -n scaling-lab
```

## Load test

```bash
# Generate load (option A — using hey)
kubectl run load-generator \
  --image=williamyeh/hey:latest \
  --restart=Never \
  -n scaling-lab \
  -- -c 50 -q 100 -z 4m http://web-frontend

# Generate load (option B — using busybox wget loop)
kubectl run -i --tty load-generator --rm --image=busybox:1.36 -n scaling-lab \
  -- /bin/sh -c "while true; do wget -q -O- http://web-frontend; done"

# Watch HPA react
kubectl get hpa web-frontend -n scaling-lab -w

# Stop the load generator
kubectl delete pod load-generator -n scaling-lab

# Watch scale-down (should happen within ~45s)
kubectl get hpa web-frontend -n scaling-lab -w
```

## Cleanup

```bash
kubectl delete pod load-generator -n scaling-lab
kubectl delete hpa web-frontend -n scaling-lab
kubectl delete svc web-frontend -n scaling-lab
kubectl delete deployment web-frontend -n scaling-lab
kubectl delete namespace scaling-lab
```

## Skills Practiced

- Creating an HPA with the `autoscaling/v2` API
- Setting CPU resource requests so HPA can compute utilization
- Adding a custom `scaleDown` stabilization window via the `behavior` block
- Validating manifests with `--dry-run=server` and load-testing autoscaling
