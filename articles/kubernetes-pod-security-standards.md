# Kubernetes Pod Security Standards (PSS)

Related: [Pod Security Standards Documentation](https://kubernetes.io/docs/concepts/security/pod-security-standards/) | [Kubernetes Security Mechanisms](articles/kubernetes-security-mechanisms.md)

## What Are Pod Security Standards?

Pod Security Standards (PSS) define three security profiles that cover the security spectrum for pods. They replace the deprecated PodSecurityPolicy (removed in Kubernetes 1.25) with a simpler, namespace-level enforcement model.

The enforcement is handled by the built-in **Pod Security Admission** controller — no external tools required.

## The Three Profiles

| Profile | Description | Use Case |
|---------|-------------|----------|
| `privileged` | No restrictions whatsoever | System-level workloads (CNI, CSI, monitoring agents) |
| `baseline` | Prevents known privilege escalations | General workloads that don't need special privileges |
| `restricted` | Heavily restricted, follows hardening best practices | Security-sensitive and untrusted workloads |

### What Each Profile Restricts

| Control | Privileged | Baseline | Restricted |
|---------|-----------|----------|------------|
| Privileged containers | Allowed | **Blocked** | **Blocked** |
| Host namespaces (hostPID, hostIPC, hostNetwork) | Allowed | **Blocked** | **Blocked** |
| Host ports | Allowed | **Blocked** | **Blocked** |
| HostPath volumes | Allowed | **Blocked** | **Blocked** |
| Privileged escalation (allowPrivilegeEscalation) | Allowed | Allowed | **Blocked** |
| Running as root | Allowed | Allowed | **Blocked** |
| Non-root groups | Allowed | Allowed | **Blocked** |
| Seccomp profile | Any | Any | **Must be RuntimeDefault or Localhost** |
| Capabilities | Any | Drops some dangerous ones | **Drop ALL, only add NET_BIND_SERVICE** |
| Volume types | Any | Any | Limited to core types (configMap, secret, emptyDir, PVC, etc.) |
| Writable root filesystem | Allowed | Allowed | Allowed (not enforced by PSS) |

## Enforcement Modes

Pod Security Admission supports three modes per namespace:

| Mode | Behavior |
|------|----------|
| `enforce` | Reject pods that violate the profile |
| `audit` | Allow pods but log violations in the API server audit log |
| `warn` | Allow pods but return warnings to the user (visible in kubectl output) |

You can mix modes — for example, enforce `baseline` but warn on `restricted` to see what would break:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

## Applying Pod Security Standards

### Per-Namespace (Labels)

Apply via namespace labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

```bash
# Or via kubectl
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

### Cluster-Wide Defaults (AdmissionConfiguration)

Set defaults for all namespaces via the API server configuration:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: baseline
      enforce-version: latest
      warn: restricted
      warn-version: latest
      audit: restricted
      audit-version: latest
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces:
      - kube-system              # Exempt system namespace
      - kube-node-lease
      - kube-public
```

## Version Pinning

You can pin enforcement to a specific Kubernetes version to avoid surprise breakages during upgrades:

```yaml
pod-security.kubernetes.io/enforce-version: "v1.30"
```

Using `latest` always applies the most current definition of the profile.

## Exemptions

Some workloads legitimately need elevated privileges. You can exempt:

- **Namespaces** — exclude entire namespaces (e.g., `kube-system`)
- **Users** — specific service accounts or users
- **RuntimeClasses** — pods using specific runtimes (e.g., sandboxed)

Exemptions are configured in the `AdmissionConfiguration` (cluster-wide), not per-namespace.

## Migration Guide: PodSecurityPolicy → PSS

### Step 1: Audit current state

```bash
# Find namespaces without PSS labels
kubectl get namespaces --show-labels | grep -v pod-security

# Check what would fail under baseline
kubectl label --dry-run=server namespace <name> pod-security.kubernetes.io/enforce=baseline

# Dry-run restricted
kubectl label --dry-run=server namespace <name> pod-security.kubernetes.io/enforce=restricted
```

### Step 2: Start with warn/audit

Apply `warn` and `audit` first to identify violations without breaking anything:

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

Check warnings when deploying:

```
Warning: would violate PodSecurity "restricted:latest": 
  allowPrivilegeEscalation != false (container "app" must set securityContext.allowPrivilegeEscalation=false)
```

### Step 3: Fix violations

Update pod specs to comply:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL
```

### Step 4: Enforce

Once clean, switch to enforce:

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  --overwrite
```

## A Pod That Passes restricted

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: my-app:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
    ports:
    - containerPort: 8080
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 256Mi
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

## A Pod That Fails restricted (and Why)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: non-compliant-pod
spec:
  containers:
  - name: app
    image: my-app:1.0
    # Missing: securityContext.allowPrivilegeEscalation: false
    # Missing: securityContext.capabilities.drop: ["ALL"]
    # Missing: pod-level seccompProfile
    # Missing: runAsNonRoot: true
    securityContext:
      privileged: true           # Blocked by baseline AND restricted
```

Violations:
1. `privileged: true` — blocked by baseline
2. No `allowPrivilegeEscalation: false` — required by restricted
3. No capabilities drop — restricted requires dropping ALL
4. No seccomp profile — restricted requires RuntimeDefault or Localhost
5. No `runAsNonRoot: true` — restricted requires non-root

## PSS vs OPA/Gatekeeper/Kyverno

| Aspect | Pod Security Standards | OPA/Gatekeeper/Kyverno |
|--------|----------------------|------------------------|
| Built-in | Yes (no install needed) | No (requires deployment) |
| Policy scope | Pod security only | Anything (labels, images, resources, etc.) |
| Customizable | No (fixed three profiles) | Fully customizable policies |
| Granularity | Namespace-level | Per-resource, per-field |
| Operational cost | Zero | Needs maintenance, testing, upgrades |
| Best for | Baseline security across the cluster | Custom organizational policies |

> **Recommendation:** Use PSS as your baseline (it's free and built-in), then layer Gatekeeper/Kyverno on top for organization-specific policies.

## PSA vs PodSecurityPolicy (Removed in 1.25)

| Feature              | PodSecurityPolicy (removed) | Pod Security Admission (current) |
|----------------------|-----------------------------|----------------------------------|
| Scope                | Cluster-wide via RBAC       | Namespace-level via labels       |
| Configuration        | Custom YAML resources       | Three predefined levels          |
| Flexibility          | Fine-grained field control  | Coarse-grained (3 levels only)   |
| Admission controller | Separate, must be enabled   | Built-in, enabled by default     |
| Maintenance          | Complex RBAC binding needed | Just label the namespace         |

## Common Mistakes

1. **Forgetting seccompProfile** — Restricted requires it at the pod or container level. Missing it causes rejection even if everything else is correct.
2. **Not dropping ALL capabilities** — Restricted requires `capabilities.drop: ["ALL"]`. Dropping only some specific capabilities is not enough.
3. **Running as root** — Even if the image runs as non-root internally, you must explicitly set `runAsNonRoot: true` for restricted. PSA checks the spec, not the runtime behavior.
4. **Assuming existing pods are affected** — Changing namespace labels only affects new pods. Running pods continue as-is until they're recreated.
5. **Confusing warn with enforce** — Warnings don't block pods. Only `enforce` mode actually rejects non-compliant pods. Teams often think `warn` provides protection — it doesn't.
6. **Baseline blocks more than expected** — Baseline blocks `hostPath` volumes, which some logging agents and monitoring tools rely on. Check before applying.
7. **Not versioning the labels** — Without `enforce-version`, the profile definition may change on cluster upgrade. Pin to a version for predictability in production.

## How runAsNonRoot Interacts with Container Images

A common source of confusion is how `runAsNonRoot` behaves depending on whether the container image was built with a non-root user.

### Image built with non-root user (e.g., USER appuser)

If the image already defines a non-root user and you set `runAsNonRoot: true` without specifying `runAsUser`, the container runs as the image's built-in user:

```yaml
spec:
  securityContext:
    runAsNonRoot: true        # Enforces non-root
  containers:
  - name: app
    image: my-app-nonroot:1.0  # Image has USER 1001
    # Container runs as UID 1001 — works fine
```

### Image built as root (default)

If the image runs as root (most base images like `nginx`, `redis`, `postgres`) and you set `runAsNonRoot: true`, the pod **fails to start**:

```
Error: container has runAsNonRoot and image will run as root
```

Fix by adding a specific `runAsUser`:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001           # Override the image's root user
  containers:
  - name: app
    image: nginx              # Image defaults to root
```

> **But this may not be enough.** If the application inside the image needs root privileges (e.g., binding to ports below 1024, modifying system files), it will crash with permission errors even though the pod starts. You need an image specifically built for non-root operation (e.g., [nginx-unprivileged](https://github.com/nginxinc/docker-nginx-unprivileged)).

### Recommendations

| Scenario | What to set |
|----------|-------------|
| Image already has a non-root user | `runAsNonRoot: true` (no need for `runAsUser`) |
| Need to enforce a specific UID (compliance) | `runAsNonRoot: true` + `runAsUser: <uid>` |
| Image requires root at startup | Use a non-root variant of the image, or don't use `restricted` profile |
| PSA enforce=restricted namespace | Must have `runAsNonRoot: true` — the container uses the image's UID if no `runAsUser` is set |

### Pod-level vs Container-level SecurityContext

```yaml
spec:
  securityContext:              # Pod-level — applies to ALL containers as default
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    securityContext:            # Container-level — overrides pod-level for this container
      runAsUser: 1001           # This container runs as 1001, not 1000
      allowPrivilegeEscalation: false
  - name: sidecar
    # No container-level securityContext — inherits pod-level (UID 1000)
```

Container-level settings override pod-level for that specific container. Pod-level acts as the default for all containers that don't specify their own.

Source: [Secure Kubernetes Pods With SecurityContext (DevOpsCube)](https://newsletter.devopscube.com/p/secure-kubernetes-pods-with-securitycontext)

## Useful Commands

```bash
# Check PSS labels on all namespaces
kubectl get namespaces -o custom-columns=\
NAME:.metadata.name,\
ENFORCE:.metadata.labels.pod-security\\.kubernetes\\.io/enforce,\
WARN:.metadata.labels.pod-security\\.kubernetes\\.io/warn,\
AUDIT:.metadata.labels.pod-security\\.kubernetes\\.io/audit

# Dry-run enforcement to see what would be rejected
kubectl label --dry-run=server namespace <name> \
  pod-security.kubernetes.io/enforce=restricted

# Check violations in audit logs (if audit logging is enabled)
kubectl get events --field-selector reason=FailedCreate -A

# Find pods that would violate restricted profile
kubectl get pods -A -o json | jq '.items[] | select(
  .spec.containers[].securityContext.allowPrivilegeEscalation != false or
  .spec.securityContext.runAsNonRoot != true
) | "\(.metadata.namespace)/\(.metadata.name)"'

# Remove PSS enforcement from a namespace
kubectl label namespace <name> pod-security.kubernetes.io/enforce-
```
