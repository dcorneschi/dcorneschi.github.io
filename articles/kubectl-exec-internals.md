# How kubectl exec Works

The connection lifecycle of `kubectl exec` — how kubectl upgrades HTTP to a streaming protocol, the API server proxies to the kubelet, the kubelet attaches to the container runtime, and how TTY/stdin/stdout are multiplexed.

## High-Level Flow

```
┌──────────┐     ┌───────────────┐     ┌──────────┐     ┌───────────────┐     ┌───────────┐
│  kubectl │────▶│   API Server  │────▶│  Kubelet │────▶│   Container   │────▶│  Process  │
│  exec    │     │  (proxy)      │     │ (on node)│     │   Runtime     │     │  (in pod) │
│          │◀────│               │◀────│          │◀────│  (containerd) │◀────│           │
└──────────┘     └───────────────┘     └──────────┘     └───────────────┘     └───────────┘
     │                  │                   │                   │                    │
     └── SPDY/WebSocket multiplexed streams (stdin, stdout, stderr, resize) ─────────┘
```

## Step 1: kubectl Sends the Exec Request

```bash
kubectl exec -it my-pod -- /bin/bash
```

kubectl constructs a POST request to the pod's `exec` subresource:

```
POST /api/v1/namespaces/default/pods/my-pod/exec?
  command=/bin/bash&
  stdin=true&
  stdout=true&
  stderr=true&
  tty=true
```

### Query Parameters

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `command` | (required) | Command to execute in the container |
| `container` | first container | Which container to exec into |
| `stdin` | false | Attach stdin stream |
| `stdout` | true | Attach stdout stream |
| `stderr` | true | Attach stderr stream (false if tty=true) |
| `tty` | false | Allocate a TTY (interactive mode) |

```bash
# Non-interactive (no stdin, no tty):
kubectl exec my-pod -- ls /tmp

# Interactive with TTY:
kubectl exec -it my-pod -- /bin/bash

# Specific container in multi-container pod:
kubectl exec -it my-pod -c sidecar -- /bin/sh
```

## Step 2: Protocol Upgrade (HTTP → SPDY/WebSocket)

The exec endpoint doesn't return a normal HTTP response. Instead, it upgrades the connection to a streaming protocol:

```
┌──────────┐                              ┌───────────────┐
│  kubectl │                              │   API Server  │
│          │── POST /exec (Upgrade hdr) ─▶│               │
│          │                              │               │
│          │◀─ 101 Switching Protocols ───│               │
│          │                              │               │
│          │══ Multiplexed streams ═══════│               │
│          │   (stdin, stdout, stderr,    │               │
│          │    resize signals)           │               │
└──────────┘                              └───────────────┘
```

### SPDY vs WebSocket

| Protocol | When Used | Notes |
|----------|-----------|-------|
| SPDY/3.1 | Default (kubectl ≤ 1.29) | Multiple named streams over one TCP connection |
| WebSocket | Default (kubectl 1.30+) | Standards-based, works through more proxies |

The upgrade headers:

```http
POST /api/v1/namespaces/default/pods/my-pod/exec?... HTTP/1.1
Host: api.example.com:6443
Upgrade: SPDY/3.1
Connection: Upgrade
Authorization: Bearer <token>
```

Response:

```http
HTTP/1.1 101 Switching Protocols
Upgrade: SPDY/3.1
Connection: Upgrade
```

After the 101 response, the connection carries multiplexed binary streams.

### Stream Channels

The SPDY/WebSocket connection multiplexes these channels:

| Channel | ID | Direction | Purpose |
|---------|----|-----------|---------|
| stdin | 0 | client → server | Keyboard input to the process |
| stdout | 1 | server → client | Process standard output |
| stderr | 2 | server → client | Process standard error |
| resize | 3 | client → server | Terminal window size changes (JSON: `{"Width":120,"Height":40}`) |
| error | 3/4 | server → client | Error status on exit |

When `tty=true`:
- stdout and stderr are merged (TTY mixes them)
- stderr channel carries the exit status instead

## Step 3: API Server Proxies to Kubelet

The API server doesn't run commands directly. It proxies the entire stream to the kubelet running on the pod's node:

```
┌───────────────┐                         ┌──────────┐
│   API Server  │                         │  Kubelet │
│               │── SPDY/WS connection ──▶│  :10250  │
│               │   POST /exec            │          │
│               │                         │          │
│  Acts as a    │                         │  Receives│
│  transparent  │                         │  streams │
│  proxy        │                         │  and     │
│               │◀── streams ─────────────│  relays  │
└───────────────┘                         └──────────┘
```

The API server:
1. Authenticates and authorizes the request (needs `pods/exec` create permission)
2. Looks up which node the pod is on (`pod.spec.nodeName`)
3. Opens a connection to that node's kubelet (port 10250)
4. Relays all streams bidirectionally

```bash
# RBAC needed for exec:
# resources: ["pods/exec"]
# verbs: ["create"]
```

## Step 4: Kubelet Calls the Container Runtime

The kubelet uses CRI (Container Runtime Interface) to execute a command in the container:

```
┌──────────┐                              ┌───────────────┐
│  Kubelet │── CRI: ExecSync/Exec() ─────▶│  containerd   │
│          │                              │               │
│          │   Parameters:                │  Creates exec │
│          │   - container ID             │  process in   │
│          │   - command                  │  container's  │
│          │   - tty (bool)               │  namespace    │
│          │   - stdin/stdout/stderr      │               │
│          │                              │               │
│          │◀── stream back ──────────────│               │
└──────────┘                              └───────────────┘
```

### What the Container Runtime Does

```
1. Find the container's PID namespace and mount namespace
2. Use nsenter (or equivalent) to enter the container's namespaces
3. Execute the command as a new process inside those namespaces
4. Connect stdin/stdout/stderr to the SPDY/WS streams
5. If tty=true: allocate a pseudo-terminal (pty)
```

The executed command runs:
- In the container's filesystem (mount namespace)
- With the container's network (network namespace)
- As the container's user (UID/GID from SecurityContext)
- With the container's environment variables
- But as a NEW process (not replacing the main process)

## Step 5: TTY Handling

When `-t` (tty) is specified:

```
┌──────────────────────────────────────────────────────────┐
│  TTY Allocation                                          │
│                                                          │
│  kubectl (client):                                       │
│    - Puts local terminal in raw mode                     │
│    - Sends window size on resize signal (SIGWINCH)       │
│    - Forwards all keystrokes (including Ctrl+C)          │
│                                                          │
│  Container runtime:                                      │
│    - Creates a pty (pseudo-terminal) pair                │
│    - Master side: connected to the stream                │
│    - Slave side: process stdin/stdout/stderr             │
│    - Process thinks it's running in a real terminal      │
│                                                          │
│  Result:                                                 │
│    - Line editing works (arrow keys, history)            │
│    - Programs like vim, top, less work correctly         │
│    - Ctrl+C sends SIGINT to the process (not kubectl)    │
│    - Terminal colors and cursor positioning work         │
└──────────────────────────────────────────────────────────┘
```

### Terminal Resize Flow

```
User resizes terminal window
    │
    ▼
kubectl detects SIGWINCH
    │
    ▼
Sends resize event on channel 3: {"Width":120,"Height":40}
    │
    ▼
API Server → Kubelet → Container Runtime
    │
    ▼
Runtime calls ioctl(TIOCSWINSZ) on the pty
    │
    ▼
Process receives SIGWINCH, redraws (vim, top, etc.)
```

## Connection Lifecycle

```
Time ──────────────────────────────────────────────────────────────────▶

kubectl         API Server       Kubelet          Runtime         Process
   │               │               │               │               │
   │ POST /exec ──▶│               │               │               │
   │               │ auth + authz  │               │               │
   │               │               │               │               │
   │ ◀─ 101 ────── │               │               │               │
   │               │── connect ───▶│               │               │
   │               │               │── CRI Exec ──▶│               │
   │               │               │               │── nsenter ───▶│
   │               │               │               │               │ /bin/bash
   │               │               │               │               │
   │═══ stdin ════▶│══════════════▶│══════════════▶│══════════════▶│
   │◀══ stdout ═══ │◀══════════════│◀══════════════│◀══════════════│
   │               │               │               │               │
   │  ... interactive session ...  │               │               │
   │               │               │               │               │
   │═══ "exit" ══▶ │               │               │               │ exits
   │               │               │               │◀── exit(0) ───│
   │               │               │◀── done ──────│               │
   │               │◀── done ──────│               │               │
   │◀── close ──── │               │               │               │
   │               │               │               │               │
   │ exit code: 0  │               │               │               │
```

## Why kubectl exec Hangs — Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Hangs immediately | API server can't reach kubelet (port 10250 blocked) | Check security groups, network policies between control plane and nodes |
| Hangs after "Upgrade" | Proxy/LB doesn't support connection upgrade | Use WebSocket protocol, check ALB/nginx config |
| Hangs on EKS | Private endpoint, kubectl outside VPC | Connect via VPN/bastion, or enable public endpoint |
| Works but slow | High latency between kubectl and API server | Normal for cross-region, nothing to fix |
| "container not found" | Container crashed or restarted | Check pod status, use correct container name |
| "command not found" | Binary doesn't exist in container image | Use a different shell (`/bin/sh`), or install tools |
| Disconnects after idle | TCP timeout on proxy/LB | Set keepalive, or use `--pod-running-timeout` |

```bash
# Debug connectivity:
kubectl exec -it my-pod -- /bin/sh -v=6

# Check kubelet port is reachable (from API server network):
# Port 10250 must be open from control plane to worker nodes

# Check if WebSocket upgrade is supported:
kubectl exec -it my-pod -- /bin/sh --v=8 2>&1 | grep -i upgrade
```

## Security Considerations

```bash
# Exec requires specific RBAC:
# verbs: ["create"] on resource: ["pods/exec"]

# This is effectively root shell access to the container!
# Audit exec usage:
kubectl get events -A --field-selector reason=Exec

# In audit logs, look for:
# requestURI: /api/v1/namespaces/.../pods/.../exec
```

### Restricting Exec Access

```yaml
# RBAC that allows exec only in specific namespace:
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-exec
  namespace: development
rules:
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get"]   # Needed to resolve pod to node
```

## exec vs attach

| Feature | `kubectl exec` | `kubectl attach` |
|---------|---------------|-----------------|
| Process | Creates NEW process | Connects to EXISTING PID 1 |
| Command | Any command | Whatever container's main process is |
| Multiple calls | Each exec is independent | Only one attach per container |
| Use case | Run debugging commands | Interact with main process stdin |

```bash
# exec: new process
kubectl exec my-pod -- ps aux    # Shows your process + container processes

# attach: connect to PID 1's stdin/stdout
kubectl attach -it my-pod        # Connects to whatever CMD/ENTRYPOINT is
```

## Quick Reference

```bash
# Basic exec:
kubectl exec my-pod -- ls /tmp
kubectl exec my-pod -c container-name -- whoami

# Interactive:
kubectl exec -it my-pod -- /bin/bash
kubectl exec -it my-pod -- /bin/sh    # If bash not available

# Non-TTY with stdin (pipe data in):
echo "hello" | kubectl exec -i my-pod -- cat

# Connection path:
# kubectl → API server (:6443) → kubelet (:10250) → containerd → process

# Protocol: SPDY/3.1 (legacy) or WebSocket (1.30+)
# Multiplexed channels: stdin(0), stdout(1), stderr(2), resize(3)

# RBAC required:
# resources: ["pods/exec"], verbs: ["create"]
# resources: ["pods"], verbs: ["get"]

# Common fixes for hangs:
# - Check port 10250 is open (control plane → nodes)
# - Check proxy/LB supports WebSocket/SPDY upgrade
# - Use -v=8 to see where it stalls
```
