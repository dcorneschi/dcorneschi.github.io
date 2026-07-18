# Kubernetes imagePullPolicy

The `imagePullPolicy` field controls whether kubelet pulls a container image when launching a pod. It's defined per container in your pod spec.

## Quick Reference

| Value | Behavior |
|-------|----------|
| `IfNotPresent` | Pull only if the image is not already on the node |
| `Always` | Check the registry every time — pull only if the digest differs from what's cached locally |
| `Never` | Never pull — fail with `ErrImageNeverPull` if the image isn't on the node |

## Decision Flow

<p align="center">
  <img src="/articles/images/imagepullpolicy-flow.png" alt="imagePullPolicy decision flow diagram" width="800"/>
</p>

## Listing Pod Policies

```bash
kubectl get pod <pod-name> -o yaml | grep Policy | sort | uniq
```

Typical output:

```
dnsPolicy: ClusterFirst
imagePullPolicy: Always
preemptionPolicy: PreemptLowerPriority
restartPolicy: Always
terminationMessagePolicy: File
```

## IfNotPresent

Kubelet checks if the image already exists on the node. If it does, the pod starts immediately without contacting the registry. If not, kubelet pulls the image first.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3
        imagePullPolicy: IfNotPresent
```

### Tradeoffs

| Pros | Cons |
|------|------|
| Faster pod startup when image is cached | If the tag is overwritten in the registry, nodes may run stale versions |
| Reduces registry bandwidth | Different nodes can end up with different image digests for the same tag |

## Always

Kubelet contacts the registry on every pod start to resolve the image tag to a digest. If the local cached image matches that digest, no download happens. If the digest differs, kubelet pulls the new image.

```yaml
containers:
- name: nginx
  image: nginx:1.25.3
  imagePullPolicy: Always
```

### Tradeoffs

| Pros | Cons |
|------|------|
| Guarantees all pods run the image matching the current tag in the registry | Adds latency on every pod start due to registry lookup |
| Prevents stale image drift across nodes | Requires registry availability — pods won't start if the registry is down |

> **Warning:** If `imagePullPolicy` is set to `Always` and the image registry is offline, the container will not run even if the same image is already stored locally. A registry that is unavailable may therefore prevent your application from (re)starting.

## Never

Kubelet will not pull images under any circumstances. If the image is present on the node, the pod starts. Otherwise, it enters `ErrImageNeverPull` state.

```yaml
containers:
- name: nginx
  image: nginx:1.25.3
  imagePullPolicy: Never
```

### Use Cases

- Preloaded images on nodes for minimal startup latency
- Air-gapped environments without registry access
- Avoiding authentication overhead with private registries when images are pre-distributed

## Default Behavior

If you omit the `imagePullPolicy` field, Kubernetes applies these rules:

| Condition | Default Policy |
|-----------|---------------|
| Image tag is `:latest` or no tag specified | `Always` |
| Image has a specific tag (e.g., `:1.25.3`) | `IfNotPresent` |
| Image is referenced by digest (`@sha256:...`) | `IfNotPresent` |

### Example — no tag triggers Always

```yaml
containers:
- name: nginx
  image: nginx          # no tag = defaults to :latest = imagePullPolicy: Always
```

## Image Tags vs Digests

A tag is a mutable label — it can point to different image builds over time. A digest is an immutable SHA256 hash that uniquely identifies a specific image layer set.

| Concept | Mutable | Example |
|---------|---------|---------|
| Tag | Yes | `nginx:1.25.3` |
| Digest | No | `nginx@sha256:6c7be49d2a11c...` |

### Why Digests Matter

If someone pushes a new build under the same tag, nodes using `IfNotPresent` won't notice the change — they keep using the cached version. Nodes pulling fresh will get the new build. You end up with two different versions running under the same tag.

Pinning by digest eliminates this problem entirely:

```yaml
containers:
- name: nginx
  image: nginx@sha256:6c7be49d2a11c86a6a42397e7b4eac7b24a4428e0b562e6cba9a9bfcc6c87b5b
  imagePullPolicy: IfNotPresent
```

## Best Practices

| Practice | Reason |
|----------|--------|
| Pin images by digest in production | Prevents silent tag overwrites from causing inconsistencies |
| Enable tag immutability on your registry | Blocks pushes to existing tags at the registry level |
| Always set `imagePullPolicy` explicitly | Avoid relying on implicit defaults that change with tag format |
| Use `Always` in CI/CD staging environments | Ensures you test the exact image that was just built |
| Use `IfNotPresent` with digests in production | Combines consistency with fast startup |

## Registry Tag Immutability

Most container registries (ECR, GCR, ACR, Harbor) support a tag immutability setting. When enabled, pushing an image with an existing tag is rejected. This is usually off by default — enable it for production repositories.

| Registry | Setting |
|----------|---------|
| AWS ECR | Image tag mutability → `IMMUTABLE` |
| GCP Artifact Registry | Tag immutability policy |
| Azure ACR | Tag locking via retention policies |
| Harbor | Tag immutability rule per project |

## Pulling from Private Registries

When your images live in a private registry, kubelet needs credentials to pull them. Kubernetes handles this through `imagePullSecrets`.

### Create a Registry Secret

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server='registry.example.com' \
  --docker-username='my-user' \
  --docker-password='my-password' \
  --docker-email='user@example.com'
```

### Reference the Secret in Your Pod Spec

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: registry.example.com/my-org/my-app:1.0.0
    imagePullPolicy: IfNotPresent
  imagePullSecrets:
  - name: my-registry-secret
```

### Attach Secret to a Service Account

To avoid specifying `imagePullSecrets` on every pod, attach it to a service account:

```bash
# Create a dedicated service account (or use 'default')
kubectl create serviceaccount myserviceaccount

# Patch the service account to include the pull secret
kubectl patch serviceaccount myserviceaccount \
  -p '{"imagePullSecrets": [{"name": "my-registry-secret"}]}'
```

Then reference the service account in your pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: myserviceaccount
  containers:
  - name: app
    image: registry.example.com/my-org/my-app:1.0.0
```

All pods using that service account will automatically inherit the pull secret.

### Common Registry URLs

| Provider | Registry URL Format |
|----------|-------------------|
| AWS ECR | `<account_id>.dkr.ecr.<region>.amazonaws.com` |
| GCP Artifact Registry | `<region>-docker.pkg.dev/<project>/<repo>` |
| Azure ACR | `<registry_name>.azurecr.io` |
| Docker Hub (private) | `https://index.docker.io/v1/` |
| GitHub Container Registry | `ghcr.io` |

## ImagePullBackOff Explained

When kubelet fails to pull an image, the pod enters a `Waiting` state with `ImagePullBackOff` status. Kubernetes retries the pull with exponentially increasing delays (back-off) — starting at ~10 seconds and capping at 5 minutes.

### Common Causes

| Cause | Diagnostic |
|-------|-----------|
| Typo in image name or tag | `kubectl describe pod` → check the image field in Events |
| Missing `imagePullSecrets` | Events show `unauthorized` or `authentication required` |
| Wrong secret or expired credentials | Recreate the secret with fresh credentials |
| Image doesn't exist in the registry | Verify with `docker pull` or your registry's UI |
| Registry rate limiting (Docker Hub free tier) | Wait or authenticate to increase pull limits |

### Debugging Steps

```bash
# Check pod events for the exact error
kubectl describe pod <pod-name> | grep -A10 Events

# Verify the secret exists and is correctly formatted
kubectl get secret my-registry-secret -o yaml

# Test pull manually from a node
crictl pull registry.example.com/my-org/my-app:1.0.0
```

## Serial vs Parallel Image Pulling

By default, kubelet pulls container images one at a time (serially) on each node. If a pod has multiple containers (init containers, sidecars), they queue up.

### Enable Parallel Pulls

Edit the kubelet configuration file on the node (commonly `/var/lib/kubelet/config.yaml`):

```yaml
serializeImagePulls: false
```

Then restart kubelet:

```bash
sudo systemctl restart kubelet
```

### When It Helps

| Scenario | Benefit |
|----------|---------|
| Pod with many init containers or sidecars | Reduces total startup time |
| Images hosted on fast registries with low latency | Parallel downloads saturate bandwidth better |

### When It Doesn't Help

| Scenario | Why |
|----------|-----|
| Network bandwidth is the bottleneck | Parallel pulls share the same pipe — total time stays similar |
| Single-container pods | Nothing to parallelize |

> Note: This is a per-node setting. Each node in your cluster needs to be configured independently. Different nodes can already pull images concurrently for different pods regardless of this setting.

## Multi-Cloud, Hybrid & Air-Gapped Environments

Image pulling behavior becomes more critical when clusters span multiple environments.

| Environment | Challenge | Recommendation |
|-------------|-----------|----------------|
| Multi-cloud | High latency between cluster and registry in another cloud | Mirror images to a registry in the same cloud, or use `IfNotPresent` to reduce cross-cloud pulls |
| Edge / hybrid | Unreliable or slow links to central registry | Pre-pull images onto edge nodes, use `Never` or `IfNotPresent` |
| Air-gapped | No external network access at all | Host a local registry mirror inside the air-gapped network; use `Never` if images are baked into node images |

### Local Registry Mirror Pattern

For air-gapped or edge clusters, run a registry mirror inside your network:

```bash
# Run a local registry
docker run -d -p 5000:5000 --restart=always --name registry registry:2

# Tag and push images to local registry
docker tag nginx:1.25.3 localhost:5000/nginx:1.25.3
docker push localhost:5000/nginx:1.25.3
```

Then reference the local mirror in your pod specs:

```yaml
containers:
- name: nginx
  image: my-local-registry:5000/nginx:1.25.3
  imagePullPolicy: IfNotPresent
```

## File Permission Issues with Local Images

When using `imagePullPolicy: Never`, the image must be readable by the container runtime on the node. Possible issues:

| Issue | Diagnostic | Fix |
|-------|-----------|-----|
| File permissions too restrictive | `ls -l` on the image store path | `chmod` the image layers |
| SELinux denying access | `ausearch -m avc -ts recent` | Add appropriate SELinux context or set permissive mode for troubleshooting |
| AppArmor profile blocking reads | `dmesg \| grep DENIED` | Update the AppArmor profile |

> This is rare with standard container runtimes (containerd, CRI-O) since they manage their own storage. It's more common with custom setups or manually loaded images.

## Default Registry Behavior

If you don't include a registry hostname in the image field, Kubernetes assumes the image is on **Docker Hub**.

| Image Field | Resolved To |
|-------------|-------------|
| `nginx:1.25.3` | `docker.io/library/nginx:1.25.3` |
| `myorg/myapp:latest` | `docker.io/myorg/myapp:latest` |
| `ghcr.io/myorg/myapp:1.0` | `ghcr.io/myorg/myapp:1.0` (used as-is) |
| `registry.k8s.io/pause:3.9` | `registry.k8s.io/pause:3.9` (used as-is) |

To pull from a different registry, always include the full hostname in the image reference.

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| `ErrImageNeverPull` | Policy is `Never` and image isn't on the node | Preload the image or change policy |
| `ImagePullBackOff` | Registry auth failure or image doesn't exist | Check `imagePullSecrets` and image path |
| Pods running different versions with same tag | `IfNotPresent` with mutable tags | Use digests or switch to `Always` |
| Slow pod startup | `Always` policy hitting a slow registry | Consider `IfNotPresent` with pinned digests |

## Useful Commands

| Command | Description |
|---------|-------------|
| `kubectl get pod <pod> -o jsonpath='{.spec.containers[*].imagePullPolicy}'` | Check pull policy for a running pod |
| `kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].imageID}'` | Get the actual image digest running in the pod |
| `kubectl describe pod <pod> \| grep -A5 Events` | See pull events and timing |
| `crictl images` | List images cached on the node (containerd) |
| `docker images` | List images cached on the node (Docker runtime) |
