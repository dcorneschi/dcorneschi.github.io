# How Pod Security Admission Enforces Policies

How the built-in Pod Security Admission (PSA) controller evaluates pods against security profiles — namespace labels, enforcement modes, profile levels, and what gets checked at admission time.

Note: For PSS profile details, runAsNonRoot behavior, and migration from PSP, see the Pod Security Standards guide. This article focuses on the enforcement mechanism itself.

## High-Level Flow

```
Pod CREATE/UPDATE request
        │
        ▼
┌───────────────┐     ┌──────────────────────────┐
│   API Server  │────▶│  Pod Security Admission  │
│               │     │  (built-in admission     │
│               │     │   controller)            │
│               │     │                          │
│               │     │ 1. Read namespace labels │
│               │     │ 2. Determine profile     │
│               │     │ 3. Evaluate pod spec     │
│               │     │ 4. Enforce/audit/warn    │
│               │◀────│                          │
└───────────────┘     └──────────────────────────┘
        │
        ▼
   Allow / Deny (with message)
```

## How PSA Is Configured — Namespace Labels

PSA uses **namespace labels** to declare policy. No separate policy objects — just labels on the namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Format: pod-security.kubernetes.io/<MODE>: <LEVEL>
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

    # Optional: pin to a specific K8s version's profile definition
    pod-security.kubernetes.io/enforce-version: v1.30
    pod-security.kubernetes.io/audit-version: v1.30
    pod-security.kubernetes.io/warn-version: v1.30
```

```bash
# Apply PSA labels to a namespace:
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Check namespace labels:
kubectl get namespace production --show-labels
```

## Three Modes

| Mode | Behavior | User Sees |
|------|----------|-----------|
| `enforce` | Rejects pods that violate the profile | Pod creation fails with error message |
| `audit` | Allows pods but logs violations in audit log | Nothing visible to user, appears in audit logs |
| `warn` | Allows pods but shows a warning to the user | Warning message in kubectl output |

You can use different profiles per mode — common pattern:

```
enforce: baseline     ← Block the dangerous stuff
audit:   restricted   ← Log everything that isn't best practice
warn:    restricted   ← Show warnings for non-ideal configs
```

This lets you enforce a minimum bar while nudging toward stricter compliance.

### Mode Behavior Example

```bash
# With enforce=baseline, audit=restricted, warn=restricted:

kubectl apply -f privileged-pod.yaml
# Error: pods "evil" is forbidden: violates PodSecurity "baseline:v1.30":
# privileged (container "app" must not set securityContext.privileged=true)

kubectl apply -f non-root-no-seccomp.yaml
# Warning: would violate PodSecurity "restricted:v1.30":
# seccompProfile (pod or container "app" must set securityContext.seccompProfile.type)
# pod created successfully (only enforcing baseline)
```

## Three Profile Levels

| Level | Description | What It Allows |
|-------|-------------|----------------|
| `privileged` | No restrictions | Everything (opt-out of PSA) |
| `baseline` | Prevents known privilege escalations | Most workloads without modification |
| `restricted` | Best practices — heavily locked down | Requires explicit security settings |

### What Each Profile Checks

#### Privileged (allows everything)

No checks applied. Equivalent to no policy.

#### Baseline (blocks dangerous configurations)

| Check | What's Blocked |
|-------|---------------|
| HostProcess | Windows HostProcess containers |
| Host Namespaces | `hostNetwork`, `hostPID`, `hostIPC` |
| Privileged Containers | `securityContext.privileged: true` |
| Capabilities | Adding capabilities beyond the allowed list |
| HostPath Volumes | `hostPath` volume mounts |
| Host Ports | `hostPort` usage |
| AppArmor | Custom profiles that aren't `runtime/default` or `localhost/*` |
| SELinux | Custom SELinux options (type must be allowed) |
| /proc Mount Type | `procMount` must be `Default` |
| Seccomp | Must be `RuntimeDefault` or `Localhost` (NOT `Unconfined`) — K8s 1.25+ |
| Sysctls | Only safe sysctls allowed |

#### Restricted (all baseline checks plus)

| Check | What's Required |
|-------|----------------|
| Volume Types | Only `configMap`, `emptyDir`, `projected`, `secret`, `downwardAPI`, `csi`, `pvc`, `ephemeral` |
| Privilege Escalation | `allowPrivilegeEscalation: false` required |
| Running as Non-Root | `runAsNonRoot: true` required |
| Seccomp Profile | Must set `RuntimeDefault` or `Localhost` explicitly |
| Capabilities | Must drop ALL, may only add `NET_BIND_SERVICE` |
| Run as User | Must not run as UID 0 (if `runAsUser` is set) |

## Evaluation Logic

```
┌──────────────────────────────────────────────────────────────────┐
│  PSA Evaluation (on pod CREATE or UPDATE)                        │
│                                                                  │
│  1. Get the pod's namespace                                      │
│  2. Read namespace labels:                                       │
│     pod-security.kubernetes.io/enforce = <level>                 │
│     pod-security.kubernetes.io/audit = <level>                   │
│     pod-security.kubernetes.io/warn = <level>                    │
│                                                                  │
│  3. For each mode (enforce, audit, warn):                        │
│     a. Load the profile checks for that level                    │
│     b. Evaluate the pod spec against each check                  │
│     c. Collect violations                                        │
│                                                                  │
│  4. Apply decisions:                                             │
│     enforce violations → REJECT the request                      │
│     audit violations   → ALLOW, but write to audit log           │
│     warn violations    → ALLOW, but return warning header        │
│                                                                  │
│  5. If no enforce violations → pod is admitted                   │
└──────────────────────────────────────────────────────────────────┘
```

### What Gets Evaluated

PSA evaluates:
- Pod spec (`spec.containers`, `spec.initContainers`, `spec.ephemeralContainers`)
- Pod-level security settings (`spec.securityContext`)
- Volume types (`spec.volumes`)
- Host settings (`spec.hostNetwork`, `spec.hostPID`, etc.)

PSA does NOT directly evaluate Deployments/StatefulSets/Jobs — it evaluates the **pod** that they create. The check happens when the controller creates the pod.

### Workload Resource Warnings

When you create a Deployment (not a pod directly), PSA still warns at Deployment creation time by doing a dry-run check on the pod template:

```bash
kubectl apply -f deployment.yaml
# Warning: would violate PodSecurity "restricted:v1.30": ...
# deployment.apps/my-app created
```

The Deployment itself isn't blocked (only `enforce` on actual pods blocks). But you get early feedback via `warn`.

## Exemptions

PSA supports exemptions to skip checks for specific users, namespaces, or runtime classes:

```yaml
# In the API server configuration (AdmissionConfiguration):
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
      audit: restricted
      audit-version: latest
      warn: restricted
      warn-version: latest
    exemptions:
      usernames:
      - system:serviceaccount:kube-system:*    # System components
      runtimeClasses:
      - untrusted                              # Sandboxed runtime
      namespaces:
      - kube-system                            # Control plane namespace
      - kube-public
```

### Exemption Types

| Exemption | What It Skips |
|-----------|--------------|
| `usernames` | Requests by these users skip PSA (e.g., system controllers) |
| `runtimeClasses` | Pods with these RuntimeClasses skip PSA |
| `namespaces` | All pods in these namespaces skip PSA |

Exemptions apply to all modes (enforce, audit, warn).

## Version Pinning

```yaml
labels:
  pod-security.kubernetes.io/enforce-version: v1.30
```

Without version pinning, the `latest` profile definition is used. Pinning ensures that upgrading your cluster doesn't suddenly change what's allowed.

```bash
# See available versions:
# Profiles are versioned per K8s minor version (v1.25, v1.26, ..., v1.30)
# "latest" = current cluster version's profile
```

## How It Differs from PSP (Deprecated)

| Feature | PodSecurityPolicy (removed 1.25) | Pod Security Admission |
|---------|--------------------------------|----------------------|
| Configuration | Cluster-scoped objects + RBAC binding | Namespace labels |
| Complexity | High (objects, bindings, ordering) | Low (just labels) |
| Mutation | Could modify pod spec (add defaults) | Read-only (no mutation) |
| Granularity | Per-policy, per-user | Per-namespace, three profiles |
| Custom policies | Arbitrary field rules | Fixed three-level profiles only |
| Audit mode | No | Yes (audit + warn modes) |

**Key limitation of PSA**: It cannot mutate pods. If a pod doesn't set `runAsNonRoot: true`, PSA rejects it — it doesn't add the field for you. Use a mutating webhook (Kyverno, Gatekeeper) if you need auto-remediation.

## Namespace-Level Dry Run

Before enforcing, test what would break:

```bash
# Dry-run: what pods in a namespace would violate restricted?
kubectl label namespace production \
  pod-security.kubernetes.io/warn=restricted \
  --dry-run=server

# Or check specific pods:
kubectl get pods -n production -o json | \
  kubectl label --dry-run=server -f - \
  pod-security.kubernetes.io/enforce=restricted
```

### Gradual Rollout Strategy

```
Phase 1: warn=restricted (see warnings, nothing breaks)
    ↓ Fix warnings
Phase 2: audit=restricted (log violations, track in audit logs)
    ↓ Verify no critical violations in audit
Phase 3: enforce=baseline (block dangerous, warn on restricted)
    ↓ Fix remaining
Phase 4: enforce=restricted (full lockdown)
```

## Common Violations and Fixes

| Violation | Message | Fix |
|-----------|---------|-----|
| Privileged | `must not set securityContext.privileged=true` | Remove `privileged: true` |
| RunAsNonRoot | `must set securityContext.runAsNonRoot=true` | Add `runAsNonRoot: true` |
| Privilege escalation | `must set allowPrivilegeEscalation=false` | Add `allowPrivilegeEscalation: false` |
| Seccomp | `must set seccompProfile.type` | Add `seccompProfile: {type: RuntimeDefault}` |
| Capabilities | `must not include in add` | Drop ALL, add only NET_BIND_SERVICE if needed |
| HostPath | `hostPath volumes are not allowed` | Use PVC/emptyDir/projected instead |
| HostNetwork | `must not set hostNetwork=true` | Remove `hostNetwork: true` |
| RunAsUser 0 | `must not run as UID 0` | Set `runAsUser: 1000` (or any non-zero) |

### Minimal Restricted-Compliant Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: my-app:latest
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
```

## Debugging

```bash
# Check what labels a namespace has:
kubectl get namespace <ns> -o jsonpath='{.metadata.labels}' | jq .

# See PSA violations in audit logs:
# Look for annotations: "pod-security.kubernetes.io/audit-violations"

# Test if a pod would be admitted:
kubectl run test --image=nginx --dry-run=server -n <namespace> -o yaml

# Check which namespaces enforce what:
kubectl get namespaces -o custom-columns=\
  NAME:.metadata.name,\
  ENFORCE:.metadata.labels.pod-security\.kubernetes\.io/enforce,\
  AUDIT:.metadata.labels.pod-security\.kubernetes\.io/audit,\
  WARN:.metadata.labels.pod-security\.kubernetes\.io/warn

# Events when pods are rejected:
kubectl get events -n <namespace> --field-selector reason=FailedCreate | grep -i security
```

## Quick Reference

```bash
# Configuration: namespace labels (no separate policy objects)
# pod-security.kubernetes.io/<MODE>: <LEVEL>
# Modes: enforce, audit, warn
# Levels: privileged, baseline, restricted

# Apply PSA to a namespace:
kubectl label namespace <ns> \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted

# Remove PSA from a namespace:
kubectl label namespace <ns> \
  pod-security.kubernetes.io/enforce- \
  pod-security.kubernetes.io/warn-

# Evaluation:
# - Checks pod spec at admission time (CREATE/UPDATE)
# - enforce: rejects violating pods
# - audit: logs violations (allows pod)
# - warn: returns warning (allows pod)

# PSA cannot mutate — only accept or reject
# Use Kyverno/Gatekeeper for auto-remediation

# Exemptions: configured in API server AdmissionConfiguration
# (usernames, runtimeClasses, namespaces)

# Key difference from PSP: no mutation, simpler config, namespace-scoped
```
