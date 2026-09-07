# Finding the Real Image Version Behind a `latest` Tag

A mutable tag like `nginx:latest` tells you nothing about what's actually
running — two pods on the same tag can be different builds. This guide shows how
to resolve the real version and immutable digest of a running container.

## When Using the `nginx:latest` Tag

If your deployment uses `nginx:latest`, check the actual resolved version that's running.

### Get the full image ID with SHA digest

Shows the exact image pulled from the registry:

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].imageID}'
```

### Get the image tag being used

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].image}'
```

### Formatted output with container name and image

```bash
kubectl get pod <pod-name> -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.image}{"\t"}{.imageID}{"\n"}{end}'
```

## Check the Actual Nginx Version Inside the Container

### Run `nginx -v`

```bash
kubectl exec <pod-name> -- nginx -v
```

### Via an interactive shell

```bash
kubectl exec -it <pod-name> -- /bin/sh -c "nginx -v"
```

### Check the build configuration

```bash
kubectl exec <pod-name> -- nginx -V
```

## Find Nginx Pods First

### List all pods

```bash
kubectl get pods
```

### Filter by label

```bash
kubectl get pods -l app=<your-label>
```

### Get pods for a specific deployment

```bash
kubectl get pods -l app=nginx -o wide
```

## Check the Image from the Deployment Spec

### Get the image from a deployment

```bash
kubectl get deployment <deployment-name> -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### With a namespace

```bash
kubectl get deployment <deployment-name> -n <namespace> -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### All containers in a deployment

```bash
kubectl get deployment <deployment-name> -o jsonpath='{.spec.template.spec.containers[*].image}'
```

> **Spec image vs running image:** the deployment spec shows the *requested*
> image (e.g., `nginx:latest`), while `status.containerStatuses[].imageID` shows
> the *resolved digest* that was actually pulled. Compare both to detect drift.

## Complete Example Workflow

```bash
# Step 1: Find nginx pods
kubectl get pods -l app=nginx

# Step 2: Capture the pod name (e.g., nginx-deployment-abc123)
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')

# Step 3: Check the image ID (SHA digest)
kubectl get pod "$POD_NAME" -o jsonpath='{.status.containerStatuses[0].imageID}'

# Step 4: Check the actual nginx version running
kubectl exec "$POD_NAME" -- nginx -v

# Step 5: Get detailed info
kubectl get pod "$POD_NAME" -o jsonpath='{range .status.containerStatuses[*]}Name: {.name}{"\n"}Image: {.image}{"\n"}ImageID: {.imageID}{"\n"}{end}'
```

## Output Examples

Image ID:

```text
docker-pullable://nginx@sha256:0d17b565c37bcbd895e9d92315a05c1c3c9a29f762b011a10c54a66cd53c9b31
```

Nginx version:

```text
nginx version: nginx/1.25.3
```

Formatted pod info:

```text
Name: nginx
Image: nginx:latest
ImageID: docker-pullable://nginx@sha256:0d17b565c37bcbd895e9d92315a05c1c3c9a29f762b011a10c54a66cd53c9b31
```

## Additional Checks

### List all running images in a namespace

```bash
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'
```

### Check the image pull policy

```bash
kubectl get deployment <deployment-name> -o jsonpath='{.spec.template.spec.containers[0].imagePullPolicy}'
```

### Describe the pod for full details

```bash
kubectl describe pod <pod-name>
```

### Check the ReplicaSet image info

```bash
kubectl get rs -o wide | grep nginx
```

## Best Practices

1. **Avoid `latest` in production** — pin specific versions for reproducibility.
2. **Use SHA digests** — reference images by digest for immutable deployments.
3. **Understand the image pull policy** — know when Kubernetes re-pulls an image.
4. **Monitor image updates** — alert when base images change.
5. **Document versions** — track which image versions are deployed where.

## Troubleshooting

### Image pull errors

```bash
kubectl describe pod <pod-name> | grep -A 10 "Events:"
```

### Check image pull secrets

```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.imagePullSecrets[*].name}'
```

### Verify an image reference is valid

```bash
kubectl run test-nginx --image=nginx:latest --dry-run=client -o yaml
```

## Skills Practiced

- Distinguishing a mutable tag from the resolved image digest (`imageID`)
- Reading running-container image info from `status.containerStatuses`
- Confirming the in-container version with `nginx -v` / `nginx -V`
- Detecting image drift between the deployment spec and running pods
