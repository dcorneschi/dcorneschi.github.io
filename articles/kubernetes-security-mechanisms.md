# Kubernetes Security Mechanisms

An overview of the security layers available in Kubernetes — from API authentication to pod-level hardening.

## Security Layers Overview

```
External Traffic → Network Policies → Ingress → Service → Pod Security → Container Security
                                                              │
API Requests → Authentication → Authorization (RBAC) → Admission Controllers → etcd
```

| Layer | What It Protects | Mechanism |
|-------|------------------|-----------|
| Authentication | Who can access the API | Certificates, tokens, OIDC |
| Authorization | What they can do | RBAC (Roles, ClusterRoles) |
| Admission Control | What resources are allowed | Validating/Mutating webhooks, Pod Security |
| Network Policies | Pod-to-pod communication | CNI-enforced firewall rules |
| Pod Security | Container runtime behavior | SecurityContext, Pod Security Standards |
| Secrets Management | Sensitive data | Secrets, external vaults |

## Authentication

Kubernetes doesn't have a built-in user database. It relies on external identity:

| Method | Use Case |
|--------|----------|
| X.509 client certificates | Service accounts, admin access via kubeconfig |
| Bearer tokens | Service account tokens (auto-mounted in pods) |
| OIDC (OpenID Connect) | Human users via identity providers (Google, Azure AD, Keycloak) |
| Webhook token authentication | Custom auth systems |

### Service Account Tokens

Every pod gets a service account token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token` unless you disable it:

```yaml
spec:
  automountServiceAccountToken: false
```

> **Best practice:** Disable auto-mounting for pods that don't need API access. Most application pods don't.

### Check who you are

```bash
# Current context identity
kubectl auth whoami

# What can I do?
kubectl auth can-i --list
```

## RBAC (Role-Based Access Control)

RBAC controls what authenticated users/service accounts can do.

### Core Objects

| Object | Scope | Purpose |
|--------|-------|---------|
| `Role` | Namespace | Grants permissions within a single namespace |
| `ClusterRole` | Cluster-wide | Grants permissions across all namespaces or on cluster resources |
| `RoleBinding` | Namespace | Binds a Role/ClusterRole to a user/group/SA in a namespace |
| `ClusterRoleBinding` | Cluster-wide | Binds a ClusterRole to a user/group/SA cluster-wide |

### Example: Read-only access to pods in a namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Example: ClusterRole for node access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

### RBAC Debugging

```bash
# Can a specific service account do something?
kubectl auth can-i get pods --as=system:serviceaccount:dev:my-app -n dev

# List all roles in a namespace
kubectl get roles -n <namespace>

# List all bindings
kubectl get rolebindings -n <namespace>

# Who has cluster-admin?
kubectl get clusterrolebindings -o json | jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'
```

## Admission Controllers

Admission controllers intercept API requests after authentication/authorization but before the object is persisted. They can validate or mutate requests.

### Built-in Controllers

| Controller | Purpose |
|------------|---------|
| `NamespaceLifecycle` | Prevents operations in non-existent or terminating namespaces |
| `LimitRanger` | Applies default resource requests/limits |
| `ResourceQuota` | Enforces namespace resource quotas |
| `PodSecurity` | Enforces Pod Security Standards (replaces PSP) |
| `MutatingAdmissionWebhook` | Calls external webhooks that can modify objects |
| `ValidatingAdmissionWebhook` | Calls external webhooks that can reject objects |

### Pod Security Admission (PSA)

Replaces the deprecated PodSecurityPolicy (removed in 1.25). Enforces predefined security profiles at the namespace level:

| Profile | Description |
|---------|-------------|
| `privileged` | No restrictions (default) |
| `baseline` | Prevents known privilege escalations (hostNetwork, hostPID, privileged containers) |
| `restricted` | Heavily restricted — must run as non-root, drop all capabilities, read-only root filesystem |

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

Modes:
- **enforce** — reject pods that violate the profile
- **audit** — allow but log violations
- **warn** — allow but show warnings to the user

> **Migration tip:** Start with `warn` and `audit` to see what would break, then switch to `enforce`.

## Network Policies

By default, all pods can communicate with all other pods. Network Policies restrict traffic at L3/L4.

> **Prerequisite:** Your CNI must support Network Policies (Calico, Cilium, Weave Net). Flannel does NOT.

### Default deny all ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}          # Applies to all pods in namespace
  policyTypes:
  - Ingress
  # No ingress rules = deny all
```

### Allow traffic only from specific pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Allow traffic from a specific namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090
```

### Deny all egress (then whitelist)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}     # Allow DNS (kube-system)
    ports:
    - protocol: UDP
      port: 53
```

## SecurityContext (Pod and Container Level)

Controls the security settings of pods and individual containers.

### Pod-level SecurityContext

```yaml
spec:
  securityContext:
    runAsNonRoot: true            # Prevent running as root
    runAsUser: 1000               # Run as specific UID
    runAsGroup: 3000              # Run as specific GID
    fsGroup: 2000                 # Group for volume mounts
    seccompProfile:
      type: RuntimeDefault        # Enable seccomp filtering
```

### Container-level SecurityContext

```yaml
spec:
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false   # Prevent setuid binaries
      readOnlyRootFilesystem: true      # Immutable container filesystem
      capabilities:
        drop:
          - ALL                          # Drop all Linux capabilities
        add:
          - NET_BIND_SERVICE             # Only add what's needed
```

### Hardened Pod Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
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
    emptyDir: {}     # Writable tmp since root is read-only
```

## Secrets

Kubernetes Secrets store sensitive data (passwords, tokens, keys). They're base64-encoded, not encrypted by default.

### Creating Secrets

```bash
# From literal values
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t

# From files
kubectl create secret tls app-tls \
  --cert=tls.crt \
  --key=tls.key
```

### Using Secrets in Pods

```yaml
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
    volumeMounts:
    - name: certs
      mountPath: /etc/certs
      readOnly: true
  volumes:
  - name: certs
    secret:
      secretName: app-tls
```

### Encryption at Rest

By default, Secrets are stored unencrypted in etcd. Enable encryption:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-key>
  - identity: {}
```

> **Best practice:** Use an external secrets manager (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault) with the External Secrets Operator or CSI Secrets Store Driver for production.

## Service Account Security

### Least-privilege Service Accounts

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
automountServiceAccountToken: false   # Don't mount token unless needed
```

### Bound Service Account Tokens (K8s 1.22+)

Modern Kubernetes uses short-lived, audience-bound tokens instead of long-lived secrets:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app
  containers:
  - name: app
    image: my-app
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/tokens
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600     # Short-lived (1 hour)
          audience: my-api            # Audience-bound
```

## Security Checklist

| Area | Action | Priority |
|------|--------|----------|
| RBAC | Use least-privilege roles, avoid cluster-admin for apps | High |
| Service Accounts | Disable auto-mount, use dedicated SAs per app | High |
| Pod Security | Enforce `restricted` profile in production namespaces | High |
| Network Policies | Default deny, then whitelist required traffic | High |
| SecurityContext | Run as non-root, drop all capabilities, read-only root FS | High |
| Secrets | Encrypt at rest, use external secrets manager | High |
| Images | Use minimal base images, scan for vulnerabilities, pin versions | Medium |
| API Server | Disable anonymous auth, enable audit logging | Medium |
| etcd | Encrypt, restrict network access, enable TLS | Medium |
| Nodes | Minimize installed software, apply OS patches, use CIS benchmarks | Medium |

## AppArmor

A Linux kernel security module (LSM) that restricts what individual programs can do — file access, network, capabilities.

Profiles are loaded on the node and applied per-container. Since Kubernetes 1.30, you can use the `securityContext.appArmorProfile` field directly:

```yaml
spec:
  containers:
  - name: app
    securityContext:
      appArmorProfile:
        type: Localhost
        localhostProfile: my-custom-profile
```

Before 1.30, use annotations:

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/my-custom-profile
```

> AppArmor profiles must be loaded on the node (`/etc/apparmor.d/`). Common on Ubuntu/Debian-based nodes.

## seccomp

Filters which syscalls a process can make. More granular than AppArmor for syscall-level control — you define a JSON profile listing allowed/denied syscalls.

```yaml
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault        # Use the container runtime's default profile
```

Or use a custom profile:

```yaml
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/my-seccomp.json   # Relative to kubelet's seccomp dir
```

Custom profile example (`/var/lib/kubelet/seccomp/profiles/my-seccomp.json`):

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "stat", "fstat", "mmap", "exit_group"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

> The default runtime profile (Docker/containerd) already blocks ~44 dangerous syscalls. `RuntimeDefault` is a good baseline — use custom profiles for further hardening.

## SELinux

Another Linux kernel security module (alternative/complement to AppArmor). Uses labels on processes and files to enforce mandatory access control (MAC). Common on RHEL/CentOS/Fedora-based nodes.

```yaml
spec:
  securityContext:
    seLinuxOptions:
      level: "s0:c123,c456"    # MCS label for multi-tenancy isolation
      type: "container_t"       # SELinux type for the process
```

> SELinux is more complex than AppArmor but more granular in multi-tenant scenarios. On EKS with Amazon Linux, SELinux is available but often in permissive mode by default.

## OPA / Gatekeeper / Kyverno

External admission controllers that go beyond what Pod Security Standards can do. They let you write custom policies.

| Tool | Policy Language | Complexity |
|------|----------------|------------|
| OPA/Gatekeeper | Rego | Higher — flexible but steeper learning curve |
| Kyverno | YAML-native | Lower — policies look like Kubernetes resources |

Example policies:
- All images must come from an approved registry
- No containers without resource limits
- All pods must have specific labels
- Prevent `latest` tag usage

Kyverno example — require resource limits:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-limits
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-limits
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "CPU and memory limits are required"
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"
                cpu: "?*"
```

## RuntimeClass and Sandboxed Runtimes

Run containers in a stronger isolation boundary than standard runc:

| Runtime | Approach | Trade-off |
|---------|----------|-----------|
| **gVisor** | Intercepts syscalls in userspace (application kernel) | Lower performance, stronger isolation |
| **Kata Containers** | Runs each pod in a lightweight VM | Higher resource overhead, VM-level isolation |

Applied via `runtimeClassName` on the pod spec:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
---
apiVersion: v1
kind: Pod
metadata:
  name: untrusted-workload
spec:
  runtimeClassName: gvisor
  containers:
  - name: app
    image: untrusted-image
```

> Useful when you don't trust the workload at all — multi-tenant clusters, running user-submitted code, CI/CD build containers.

## Falco (Runtime Detection)

Runtime threat detection — watches syscalls and Kubernetes audit logs for suspicious behavior. Complements enforcement tools by alerting on things they might miss:

- Shell spawned inside a container
- Sensitive file read (`/etc/shadow`, `/etc/passwd`)
- Unexpected outbound network connections
- Binary executed that wasn't part of the original image
- Privilege escalation attempts

Falco doesn't block — it detects and alerts. Pair it with enforcement (AppArmor, seccomp, Network Policies) for defense in depth.

## How Security Mechanisms Relate

| Tool | When | What |
|------|------|------|
| PSS / Gatekeeper / Kyverno | Admission time | Policy validation — reject non-compliant pods |
| AppArmor / SELinux | Runtime | Process access control — restrict file/network access |
| seccomp | Runtime | Syscall filtering — block dangerous kernel calls |
| Linux Capabilities | Runtime | Privilege scoping — fine-grained root decomposition |
| Network Policies | Runtime | Network traffic control — pod-to-pod firewall |
| gVisor / Kata | Runtime | Workload isolation — stronger process boundary |
| Falco | Runtime | Detection and alerting — watch for suspicious activity |

These layers complement each other — defense in depth. No single one covers everything.

## Useful Commands

```bash
# Check RBAC permissions for a service account
kubectl auth can-i --list --as=system:serviceaccount:<namespace>:<sa-name>

# Find pods running as root
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].securityContext.runAsUser == 0 or .spec.securityContext.runAsUser == 0) | .metadata.name'

# Find pods with privileged containers
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].securityContext.privileged == true) | "\(.metadata.namespace)/\(.metadata.name)"'

# Check network policies in a namespace
kubectl get networkpolicies -n <namespace>

# Describe a network policy
kubectl describe networkpolicy <name> -n <namespace>

# Check Pod Security Admission labels on namespaces
kubectl get namespaces --show-labels | grep pod-security

# Audit secrets access
kubectl get clusterrolebindings -o json | jq '.items[] | select(.roleRef.name | test("secret")) | .subjects'

# Check if etcd encryption is enabled
kubectl get apiservices -o json | jq '.items[] | select(.metadata.name | test("v1"))'
```
