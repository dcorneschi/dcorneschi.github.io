# runAsNonRoot: true

How `runAsNonRoot` works in Kubernetes — what it enforces, why containers fail with it enabled, how to fix images, and the relationship with `runAsUser`.

## What runAsNonRoot Does

```yaml
securityContext:
  runAsNonRoot: true
```

This tells the kubelet: **reject the container if it would run as UID 0 (root)**.

It's a validation check, not a transformation. It doesn't change the user — it blocks the container from starting if the effective user is root.

## Where It Fails

```
Error: container has runAsNonRoot and image will run as root
```

This happens when:
1. The container image has `USER root` or no `USER` directive (defaults to root)
2. No `runAsUser` is set in the pod/container securityContext
3. `runAsNonRoot: true` is set

The kubelet checks the image's configured user. If it resolves to UID 0, the container is **not started**.

## How the Check Works

```
Pod starts
  │
  ├─ runAsNonRoot: true?
  │    ├─ No  → Start container (no check)
  │    └─ Yes → Check effective UID
  │
  ├─ runAsUser set in securityContext?
  │    ├─ Yes → Use that UID
  │    └─ No  → Use UID from image (USER directive)
  │
  ├─ Effective UID == 0?
  │    ├─ Yes → BLOCK: "container has runAsNonRoot and image will run as root"
  │    └─ No  → Start container
  │
  └─ No USER in image and no runAsUser?
       └─ Defaults to UID 0 → BLOCK
```

## Fix 1: Set runAsUser in SecurityContext

Override the image's default user without modifying the image:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
```

This works even if the image has `USER root` or no `USER` directive — the kubelet uses UID 1000 instead.

## Fix 2: Set USER in the Dockerfile

Build the image to run as non-root by default:

```dockerfile
FROM ubuntu:24.04

RUN groupadd -r appuser && useradd -r -g appuser -u 1000 appuser

# Install deps as root
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Switch to non-root
USER 1000:1000

WORKDIR /app
COPY --chown=1000:1000 . .

CMD ["./start.sh"]
```

Now the image reports UID 1000, and `runAsNonRoot: true` passes without needing `runAsUser` in the pod spec.

## Fix 3: Use a Numeric USER in Dockerfile

Some images use a username instead of a numeric UID:

```dockerfile
USER appuser
```

This can cause issues because the kubelet resolves the username to a UID by looking inside the container's `/etc/passwd`. If the resolution fails or resolves to 0, the pod won't start.

Best practice — always use numeric UIDs:

```dockerfile
# Good: explicit numeric UID
USER 1000

# Bad: relies on /etc/passwd resolution
USER appuser
```

## runAsNonRoot vs runAsUser

| Setting | What It Does |
|---------|-------------|
| `runAsNonRoot: true` | **Validates** that the effective UID is not 0. Blocks if it is. |
| `runAsUser: 1000` | **Sets** the UID to run as. Overrides the image's USER directive. |

### Combinations

| runAsNonRoot | runAsUser | Image USER | Result |
|:---:|:---:|:---:|------|
| true | not set | root (or none) | **BLOCKED** — would run as root |
| true | not set | 1000 | Starts as UID 1000 |
| true | 1000 | root (or none) | Starts as UID 1000 (override) |
| true | 0 | anything | **BLOCKED** — explicitly set to root |
| false/not set | not set | root | Starts as root (no check) |
| false/not set | 1000 | root | Starts as UID 1000 (no check, just override) |

### Best Practice: Use Both

```yaml
securityContext:
  runAsNonRoot: true   # Safety net — block if somehow root
  runAsUser: 1000      # Explicitly set the UID
```

Using both gives defense-in-depth: `runAsUser` sets the correct UID, and `runAsNonRoot` acts as a guardrail if something changes.

## Pod-Level vs Container-Level

SecurityContext can be set at both levels:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  # Pod-level: applies to all containers
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000

  containers:
  - name: app
    image: myapp:latest
    # Container-level: overrides pod-level for this container
    securityContext:
      runAsUser: 2000        # This container uses UID 2000
      allowPrivilegeEscalation: false

  - name: sidecar
    image: busybox
    # Inherits pod-level: UID 1000, runAsNonRoot: true
    securityContext:
      allowPrivilegeEscalation: false
```

Container-level settings override pod-level settings for that specific container.

## Common Images That Run as Root

Many popular images default to root:

| Image | Default UID | Fix |
|-------|:-----------:|-----|
| nginx | 0 (root) | Use `nginxinc/nginx-unprivileged` or set `runAsUser` |
| redis | 0 (root) | Set `runAsUser: 999` (redis user exists in image) |
| postgres | 0 (root) | Set `runAsUser: 999` (switches to postgres internally) |
| mysql | 0 (root) | Set `runAsUser: 999` (switches to mysql internally) |
| ubuntu | 0 (root) | Set `runAsUser: 1000` |
| alpine | 0 (root) | Set `runAsUser: 1000` |
| node | 0 (root) | Set `runAsUser: 1000` (node user exists at UID 1000) |
| python | 0 (root) | Set `runAsUser: 1000` |

### Images That Already Run as Non-Root

| Image | Default UID |
|-------|:-----------:|
| `nginxinc/nginx-unprivileged` | 101 |
| `bitnami/*` | 1001 |
| `gcr.io/distroless/*` | 65534 (nonroot) |
| `chainguard/*` | 65532 |

## File Permission Issues

Running as non-root can cause permission errors if the image expects root access to certain paths.

### Symptoms

```
Permission denied: /var/log/nginx/error.log
mkdir: cannot create directory '/data': Permission denied
```

### Fix with fsGroup

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000    # All mounted volumes get this group ownership
```

`fsGroup` sets the group ownership of all files in mounted volumes, so the non-root user can write to them.

### Fix with initContainer

```yaml
initContainers:
- name: fix-permissions
  image: busybox
  command: ["sh", "-c", "chown -R 1000:1000 /data"]
  securityContext:
    runAsUser: 0   # init container runs as root to fix perms
  volumeMounts:
  - name: data
    mountPath: /data
containers:
- name: app
  image: myapp
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  volumeMounts:
  - name: data
    mountPath: /data
```

### Fix in Dockerfile

```dockerfile
RUN mkdir -p /data /var/log/app && chown -R 1000:1000 /data /var/log/app
USER 1000
```

## Interaction with Pod Security Standards

`runAsNonRoot: true` is required by the **restricted** Pod Security Standard:

| PSS Level | runAsNonRoot Requirement |
|-----------|------------------------|
| Privileged | No requirement |
| Baseline | No requirement |
| Restricted | Must be `true` |

If your namespace enforces `restricted` PSS, all pods must have `runAsNonRoot: true`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

## Debugging

### Check What User the Image Uses

```bash
# Check image config
docker inspect <image> --format '{{.Config.User}}'

# Or with crane
crane config <image> | jq '.config.User'

# Or run it and check
docker run --rm <image> id
```

### Check What User the Container Is Running As

```bash
kubectl exec <pod-name> -- id
# uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)

kubectl exec <pod-name> -- whoami
# appuser
```

### Check the Security Context in Use

```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.securityContext}'
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].securityContext}'
```

## Full Hardened Pod Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

This passes the **restricted** Pod Security Standard:
- `runAsNonRoot: true` — no root
- `allowPrivilegeEscalation: false` — no privilege escalation
- `readOnlyRootFilesystem: true` — no writes to container filesystem
- `capabilities.drop: ALL` — no Linux capabilities
- `seccompProfile: RuntimeDefault` — syscall filtering
- Writable paths via emptyDir volumes

## Quick Reference

```bash
# Check image default user
docker inspect <image> --format '{{.Config.User}}'

# Check running pod user
kubectl exec <pod> -- id

# Verify security context
kubectl get pod <pod> -o yaml | grep -A 10 securityContext

# Find pods running as root
kubectl get pods -A -o json | jq '.items[] | select(.spec.securityContext.runAsNonRoot != true) | .metadata.name'
```

```yaml
# Minimum to enforce non-root
securityContext:
  runAsNonRoot: true
  runAsUser: 1000

# Full hardened context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```
