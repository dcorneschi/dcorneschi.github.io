# Kubernetes allowPrivilegeEscalation Explained

## What It Does

The `allowPrivilegeEscalation` field is a Kubernetes security context setting that controls whether a container can gain more permissions than its parent process.

- When set to `true` (default), a container can use mechanisms like `setuid` or `setgid` binaries to escalate privileges
- When set to `false`, it prevents privilege escalation within the container, making it more secure

Under the hood, setting `allowPrivilegeEscalation: false` sets the `no_new_privs` flag on the container process via the Linux kernel. This flag is inherited by all child processes and cannot be unset.

## Where You Set It

In a Pod or Deployment's security context:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
```

## What It Prevents

| Mechanism | Without restriction | With allowPrivilegeEscalation: false |
|-----------|-------------------|--------------------------------------|
| setuid binaries (e.g., `sudo`, `passwd`) | Process gains root UID | Blocked — setuid bit ignored |
| setgid binaries | Process gains group privileges | Blocked — setgid bit ignored |
| Linux capabilities via execve | New capabilities can be gained | Blocked — no new capabilities |
| Kernel exploits that rely on privilege escalation | May succeed | Harder to exploit |

## Best Practices

- Set `allowPrivilegeEscalation: false` in production for all containers that don't explicitly need it
- Combine with `runAsNonRoot: true` to prevent running as root in the first place
- Required by the Pod Security Standards `restricted` profile
- If your application needs `sudo` or setuid binaries, you likely have a design issue — restructure to avoid them

## Complete Hardened Security Context

```yaml
securityContext:
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

## Related Settings

| Setting | Purpose |
|---------|---------|
| `runAsNonRoot: true` | Prevent container from running as UID 0 |
| `readOnlyRootFilesystem: true` | Prevent writes to the container filesystem |
| `capabilities.drop: ["ALL"]` | Remove all Linux capabilities |
| `seccompProfile.type: RuntimeDefault` | Apply default syscall filter |

These settings work together for defense-in-depth. `allowPrivilegeEscalation: false` is one layer — it ensures that even if an attacker gets code execution inside the container, they can't escalate beyond the container's current privilege level.
