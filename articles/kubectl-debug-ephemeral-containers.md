# kubectl debug — Ephemeral Containers and Node Debugging

How to use `kubectl debug` to troubleshoot running pods (without restarting them), debug distroless/minimal images, access node-level tools, and copy pods with modified settings.

## What kubectl debug Does

`kubectl debug` creates debugging sessions in three modes:

```
┌────────────────────────────────────────────────────────────────┐
│  Mode 1: Ephemeral Container (inject into running pod)         │
│  → Adds a debug container to an existing pod's namespace       │
│  → Shares PID, network, and optionally filesystem              │
│                                                                │
│  Mode 2: Pod Copy (clone pod with modifications)               │
│  → Creates a new pod copied from an existing one               │
│  → Can change image, command, or add containers                │
│                                                                │
│  Mode 3: Node Debug (run a pod on a specific node)             │
│  → Creates a privileged pod on the node                        │
│  → Mounts host filesystem at /host                             │
│  → Access to node's PID/network namespace via nsenter          │
└────────────────────────────────────────────────────────────────┘
```

## Mode 1: Ephemeral Containers

Ephemeral containers are special containers added to a running pod without restarting it. They exist for debugging only.

### Basic Usage

```bash
# Add a debug container to a running pod:
kubectl debug -it my-pod --image=busybox

# With a specific container name:
kubectl debug -it my-pod --image=ubuntu --container=debugger

# Target a specific container's namespaces (share PID namespace):
kubectl debug -it my-pod --image=busybox --target=app-container
```

### How It Works Internally

```
kubectl debug -it my-pod --image=busybox --target=app
    │
    ▼
┌───────────────┐
│   API Server  │
│               │
│  PATCH pod:   │
│  add to       │
│ spec.ephemeral│
│  Containers[] │
└───────────────┘
    │
    ▼
┌───────────────────────┐
│  Kubelet              │
│                       │
│  Creates container in │
│  same pod sandbox     │
│  (shared netns, IPC)  │
│                       │
│  If --target: shares  │
│  PID namespace too    │
└───────────────────────┘
```

The pod spec gets an ephemeralContainers entry:

```yaml
spec:
  ephemeralContainers:
  - name: debugger-abc12
    image: busybox
    stdin: true
    tty: true
    targetContainerName: app   # Share PID namespace with this container
```

### What --target Gives You

Without `--target`: debug container gets its own PID namespace (can only see its own processes).

With `--target=app`: debug container shares PID namespace with the target container:

```bash
# Inside the debug container (with --target):
ps aux
# PID 1 is the app container's main process
# You can see all processes in the target container

# Access the target container's filesystem via /proc:
ls /proc/1/root/    # Target container's root filesystem
cat /proc/1/root/etc/config.yaml

# Attach strace to the target's process:
strace -p 1
```

### Ephemeral Container Constraints

| Feature | Ephemeral Container | Regular Container |
|---------|:------------------:|:-----------------:|
| Added to running pod | Yes | No (requires pod restart) |
| Resource requests/limits | No | Yes |
| Probes (liveness/readiness) | No | Yes |
| Ports | No | Yes |
| Restart on crash | No (runs once) | Yes (per restartPolicy) |
| Removed after use | No (stays in spec, but terminated) | N/A |
| Volume mounts | Yes (existing pod volumes) | Yes |

```bash
# Mount existing pod volumes in the debug container:
kubectl debug -it my-pod --image=busybox -- sh
# Volumes from the pod spec are available to mount

# You cannot currently specify volume mounts via kubectl debug CLI
# The debug container inherits the pod's volumes
```

### Debugging Distroless Images

Distroless containers have no shell, no package manager, no tools. Ephemeral containers solve this:

```bash
# App container is distroless (gcr.io/distroless/static):
kubectl exec my-pod -- sh
# Error: OCI runtime exec failed: exec: "sh": not found

# Instead, inject a debug container:
kubectl debug -it my-pod --image=busybox --target=app-container

# Now you can:
# - See the app's processes (ps)
# - Read its files (via /proc/1/root/)
# - Use networking tools (wget, nc, nslookup)
# - Run strace, tcpdump, etc.
```

## Mode 2: Pod Copy

Creates a new pod cloned from an existing one, with modifications:

### Copy with Different Image

```bash
# Copy pod but replace the container image (for debugging):
kubectl debug my-pod -it --copy-to=my-pod-debug --image=ubuntu

# Copy and change the command (override entrypoint):
kubectl debug my-pod -it --copy-to=my-pod-debug --container=app -- sh

# Copy with shared process namespace (see all containers' processes):
kubectl debug my-pod -it --copy-to=my-pod-debug --share-processes
```

### When to Use Pod Copy

- App crashes immediately (can't exec or attach)
- Need to test with a different image
- Need to add environment variables or change the command
- Don't want to affect the running pod

```bash
# Pod keeps crashing — copy it with a shell entrypoint:
kubectl debug my-crashing-pod -it --copy-to=debug-pod \
  --container=app -- /bin/sh

# Inside: inspect filesystem, check configs, run the app manually
```

### Pod Copy Behavior

The copied pod:
- Gets a new name (from `--copy-to`)
- Runs in the same namespace
- Has the same spec (labels, volumes, serviceAccount, etc.)
- Does NOT inherit the original pod's status or restarts
- Must be deleted manually when done

```bash
# Clean up:
kubectl delete pod debug-pod
```

## Mode 3: Node Debugging

Creates a privileged pod on a specific node with access to the host:

```bash
# Debug a node:
kubectl debug node/node-1 -it --image=ubuntu
```

### What Gets Created

```yaml
# The pod kubectl creates:
apiVersion: v1
kind: Pod
metadata:
  name: node-debugger-node-1-abc12
spec:
  hostNetwork: true
  hostPID: true
  hostIPC: true
  nodeName: node-1
  containers:
  - name: debugger
    image: ubuntu
    stdin: true
    tty: true
    securityContext:
      privileged: true
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /
  tolerations:
  - operator: Exists   # Tolerate ALL taints (can land on any node)
```

### What You Can Do from a Node Debug Pod

```bash
# Inside the debug pod:

# Access host filesystem:
ls /host/etc/
cat /host/var/log/syslog
cat /host/var/log/messages

# Enter host namespaces (become the node):
chroot /host

# Or use nsenter for specific namespaces:
nsenter -t 1 -m -u -i -n -p -- bash
# Now you're effectively on the node as root

# View kubelet logs:
nsenter -t 1 -m -- journalctl -u kubelet --since "5 minutes ago"

# View container runtime logs:
nsenter -t 1 -m -- journalctl -u containerd --since "5 minutes ago"

# Network debugging from node perspective:
nsenter -t 1 -n -- ss -tlnp
nsenter -t 1 -n -- iptables-save | head -50

# Check disk usage:
df -h /host
du -sh /host/var/lib/containerd/
```

## Debug Profiles

Kubernetes 1.27+ supports `--profile` to pre-configure the debug pod's security settings:

```bash
# General purpose (default):
kubectl debug -it my-pod --image=busybox --profile=general

# Restricted (no privilege escalation):
kubectl debug -it my-pod --image=busybox --profile=baseline

# Network admin (CAP_NET_ADMIN for tcpdump, etc.):
kubectl debug -it my-pod --image=nicolaka/netshoot --profile=netadmin

# Sysadmin (privileged, for node debugging):
kubectl debug node/node-1 -it --image=ubuntu --profile=sysadmin
```

| Profile | Capabilities | Use Case |
|---------|-------------|----------|
| `general` | None extra | Basic inspection (ps, cat, ls) |
| `baseline` | Restricted (PSS baseline) | Safe debugging in hardened clusters |
| `restricted` | Minimal (PSS restricted) | Most constrained, least access |
| `netadmin` | CAP_NET_ADMIN, CAP_NET_RAW | tcpdump, iptables, network tools |
| `sysadmin` | Privileged | Full node access, kernel tools |

## Useful Debug Images

| Image | Size | Tools Included |
|-------|------|----------------|
| `busybox:1.36` | 4 MB | sh, wget, nc, ps, ls, cat, vi |
| `alpine:3.19` | 7 MB | sh, apk (can install tools) |
| `ubuntu:24.04` | 78 MB | bash, apt (full package manager) |
| `nicolaka/netshoot` | 350 MB | tcpdump, dig, nslookup, curl, iperf, netstat, ss, ip, strace |
| `curlimages/curl` | 15 MB | curl, sh |
| `registry.k8s.io/e2e-test-images/agnhost:2.39` | 50 MB | Multi-tool K8s test image |

```bash
# Network debugging:
kubectl debug -it my-pod --image=nicolaka/netshoot --target=app -- bash
tcpdump -i eth0 port 80
curl -v localhost:8080
dig my-service.default.svc.cluster.local

# DNS debugging:
kubectl debug -it my-pod --image=busybox --target=app -- nslookup kubernetes.default

# Filesystem inspection:
kubectl debug -it my-pod --image=alpine --target=app -- sh
ls /proc/1/root/app/
cat /proc/1/root/app/config.yaml
```

## RBAC Requirements

```yaml
# For ephemeral containers (Mode 1):
rules:
- apiGroups: [""]
  resources: ["pods/ephemeralcontainers"]
  verbs: ["patch"]    # Or "update"
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get"]

# For pod copy (Mode 2):
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "create"]

# For node debug (Mode 3):
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "delete"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get"]    # To resolve node name
```

## Cleanup

Ephemeral containers cannot be removed from a pod — they stay in the spec (terminated state) until the pod is deleted. Debug pods from node/copy modes must be deleted manually:

```bash
# List debug pods (they have a generated name pattern):
kubectl get pods | grep "node-debugger\|debug"

# Delete node debug pod:
kubectl delete pod node-debugger-node-1-abc12

# Delete copied debug pod:
kubectl delete pod my-pod-debug
```

## Limitations

| Limitation | Explanation |
|-----------|-------------|
| Ephemeral containers can't be removed | They persist (terminated) in pod spec |
| No resource limits on ephemeral containers | Can't restrict CPU/memory of debug container |
| No auto-cleanup | Debug pods from copy/node mode must be deleted manually |
| Feature gate required (older clusters) | `EphemeralContainers` must be enabled (GA since 1.25) |
| PSS may block debug pods | `restricted` namespace policy rejects privileged debug pods |
| Node debug requires privileged | Won't work in clusters that block privileged containers |

## Quick Reference

```bash
# Mode 1: Ephemeral container in running pod
kubectl debug -it <pod> --image=<image>
kubectl debug -it <pod> --image=<image> --target=<container>

# Mode 2: Copy pod with modifications
kubectl debug <pod> -it --copy-to=<new-name> --image=<image>
kubectl debug <pod> -it --copy-to=<new-name> --container=<container> -- sh
kubectl debug <pod> -it --copy-to=<new-name> --share-processes

# Mode 3: Node debug
kubectl debug node/<node-name> -it --image=ubuntu
kubectl debug node/<node-name> -it --image=ubuntu --profile=sysadmin

# Profiles (1.27+):
# general, baseline, restricted, netadmin, sysadmin

# Inside node debug pod:
chroot /host                           # Become the node
nsenter -t 1 -m -u -i -n -p -- bash   # Enter all host namespaces

# Inside ephemeral container with --target:
ps aux                    # See target's processes
ls /proc/1/root/          # Target's filesystem
strace -p 1              # Trace target's main process

# Cleanup:
kubectl delete pod <debug-pod-name>
```
