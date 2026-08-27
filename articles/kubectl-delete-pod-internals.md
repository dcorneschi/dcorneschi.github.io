# What Happens When You Run kubectl delete pod

The full lifecycle of pod deletion — from the API call through graceful termination to container cleanup on the node.

## High-Level Flow

```
┌──────────┐     ┌───────────────┐     ┌───────────┐     ┌──────────┐     ┌───────────┐
│  kubectl │────▶│   API Server  │────▶│   etcd    │────▶│  Kubelet │────▶│ Container │
│  delete  │     │  (set deletion│     │ (store    │     │ (on node)│     │  Runtime  │
│          │     │   timestamp)  │     │  update)  │     │          │     │           │
└──────────┘     └───────────────┘     └───────────┘     └──────────┘     └───────────┘
```

## Detailed Step-by-Step

### Step 1: kubectl Sends DELETE Request

```bash
kubectl delete pod nginx --namespace default
```

kubectl sends:

```
DELETE /api/v1/namespaces/default/pods/nginx
```

With a body containing `gracePeriodSeconds` (default: from pod spec's `terminationGracePeriodSeconds`, typically 30s).

```bash
# See the HTTP request:
kubectl delete pod nginx -v=8

# Force delete (skip graceful):
kubectl delete pod nginx --grace-period=0 --force
```

### Step 2: API Server Sets deletionTimestamp

The API server does NOT immediately remove the pod from etcd. Instead it:

1. Sets `metadata.deletionTimestamp` on the pod object
2. Sets `metadata.deletionGracePeriodSeconds`
3. The pod enters the **Terminating** state

```
┌───────────────────────────────────────────────┐
│  Pod object in etcd (after DELETE request)     │
│                                               │
│  metadata:                                    │
│    deletionTimestamp: "2024-03-15T10:00:00Z"  │
│    deletionGracePeriodSeconds: 30             │
│  status:                                      │
│    phase: Running  (still Running!)           │
└───────────────────────────────────────────────┘
```

The pod is now in a transitional state:
- `kubectl get pods` shows `Terminating`
- The pod still exists in etcd
- Controllers and kubelet react to the `deletionTimestamp`

### Step 3: Endpoint Removal (Parallel)

The moment `deletionTimestamp` is set, multiple things happen in parallel:

```
                    deletionTimestamp set
                           │
              ┌────────────┼────────────────┐
              ▼            ▼                ▼
     Endpoints controller  Kubelet         kube-proxy
     removes pod IP from   starts          removes iptables
     EndpointSlices        termination     rules for pod
              │            sequence
              ▼                │
     Services stop             │
     routing to pod            ▼
                          (Step 4 below)
```

The endpoints controller watches for pods with `deletionTimestamp` and removes them from EndpointSlices immediately. This means:
- New traffic stops being routed to the pod
- Existing connections may still be active (TCP established sessions)

```bash
# Watch endpoints being updated:
kubectl get endpoints <service-name> -w
kubectl get endpointslices -l kubernetes.io/service-name=<service-name> -w
```

### Step 4: Kubelet Executes Termination Sequence

The kubelet watches for pods with `deletionTimestamp` assigned to its node and starts the shutdown:

```
┌────────────────────────────────────────────────────────────────┐
│  Kubelet Termination Sequence                                  │
│                                                                │
│  T+0s:   deletionTimestamp detected                            │
│           │                                                    │
│  T+0s:   Run preStop hook (if defined)                         │
│           │   - Runs to completion OR until grace period ends   │
│           │   - Blocks SIGTERM until done                       │
│           │                                                    │
│  T+Xs:   preStop completes (or times out)                      │
│           │                                                    │
│  T+Xs:   Send SIGTERM to PID 1 in all containers               │
│           │   - Container should catch and handle gracefully    │
│           │   - Close connections, flush buffers, save state    │
│           │                                                    │
│  T+30s:  Grace period expires                                  │
│           │   (terminationGracePeriodSeconds total, including   │
│           │    preStop time)                                    │
│           │                                                    │
│  T+30s:  Send SIGKILL to all remaining processes               │
│           │   - Cannot be caught or ignored                     │
│           │   - Immediate process death                         │
│           │                                                    │
│  T+30s:  Container runtime cleans up                           │
│           │   - Remove container filesystem                    │
│           │   - Release network namespace                      │
│           │   - Unmount volumes                                │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**Critical detail:** The grace period is a TOTAL budget. preStop hook time is subtracted from it:

```
terminationGracePeriodSeconds = 30

If preStop takes 10s:
  → Container gets SIGTERM at T+10s
  → Container has 20s to shut down before SIGKILL

If preStop takes 29s:
  → Container gets SIGTERM at T+29s
  → Container has 1s before SIGKILL
```

### Step 5: Container Stops

The container runtime (containerd/CRI-O) handles the actual process kill:

```
┌──────────────┐
│   Kubelet    │
│              │──── CRI StopContainer() ────▶ containerd
│              │                                    │
│              │                              SIGTERM → PID 1
│              │                                    │
│              │                              (grace period)
│              │                                    │
│              │──── CRI StopContainer()     SIGKILL (if still alive)
│              │     (with timeout=0)               │
│              │                              Process dead
│              │                                    │
│              │──── CRI RemoveContainer() ──▶ Cleanup
└──────────────┘
```

### Step 6: Kubelet Reports Back, Pod Removed from etcd

Once all containers are stopped:

1. Kubelet updates pod status (`phase: Succeeded` or `phase: Failed`)
2. Kubelet sends a status update to API server
3. API server removes the pod object from etcd (finalizers permitting)
4. Pod no longer appears in `kubectl get pods`

```bash
# If a pod is stuck Terminating, it usually means:
# - Finalizers haven't been cleared
# - The node is unreachable (kubelet can't confirm termination)
# - A preStop hook is hanging

# Check finalizers:
kubectl get pod <name> -o jsonpath='{.metadata.finalizers}'

# Check node status:
kubectl get node <node-name> -o jsonpath='{.status.conditions[?(@.type=="Ready")]}'
```

## The Complete Timeline

```
Time ─────────────────────────────────────────────────────────────▶

kubectl          API Server        Endpoints       Kubelet          Container
   │                │                │               │                │
   │ DELETE ────────▶│                │               │                │
   │                │ set            │               │                │
   │                │ deletionTS     │               │                │
   │                │                │               │                │
   │ ◀── 200 OK ───│                │               │                │
   │ (returns       │                │               │                │
   │  immediately)  │                │               │                │
   │                │  watch event ──▶│               │                │
   │                │                │ remove from   │                │
   │                │                │ EndpointSlice │                │
   │                │                │               │                │
   │                │  watch event ──┼───────────────▶│                │
   │                │                │               │ preStop hook    │
   │                │                │               │────────────────▶│
   │                │                │               │                │ (hook runs)
   │                │                │               │◀───────────────│
   │                │                │               │                │
   │                │                │               │ SIGTERM ───────▶│
   │                │                │               │                │ (graceful
   │                │                │               │                │  shutdown)
   │                │                │               │                │
   │                │                │               │ (grace period   │
   │                │                │               │  expires)       │
   │                │                │               │                │
   │                │                │               │ SIGKILL ───────▶│ (dead)
   │                │                │               │                │
   │                │                │               │ status update ─▶│
   │                │ ◀─────────────┼───────────────│                │
   │                │ remove from   │               │                │
   │                │ etcd          │               │                │
```

## Finalizers — Why Pods Get Stuck Terminating

Finalizers are keys in `metadata.finalizers` that prevent the API server from removing the object from etcd until they're cleared:

```yaml
metadata:
  finalizers:
  - example.com/my-cleanup
```

The API server won't delete the pod until all finalizers are removed. This allows controllers to perform cleanup before the object disappears.

```bash
# See what's blocking deletion:
kubectl get pod <name> -o yaml | grep -A5 finalizers

# Remove a stuck finalizer (use carefully):
kubectl patch pod <name> -p '{"metadata":{"finalizers":null}}'
```

## Force Delete

```bash
# Force delete — skips graceful termination
kubectl delete pod nginx --grace-period=0 --force
```

What this does differently:
1. Sets `gracePeriodSeconds: 0` in the DELETE request
2. API server immediately removes the pod from etcd
3. Kubelet still tries to clean up containers, but the pod object is already gone
4. If the node is unreachable, the pod vanishes from the API but containers may still run on the node

**When to use force delete:**
- Node is permanently gone (decommissioned, terminated)
- Pod is stuck Terminating due to unreachable node
- Testing/development cleanup

**Never force delete in production** if the node is still running — you'll have orphaned containers.

## Multi-Container Pods

For pods with multiple containers (including sidecars):

```
┌──────────────────────────────────────────┐
│  Multi-container termination             │
│                                          │
│  1. ALL containers receive preStop hooks │
│     simultaneously                       │
│                                          │
│  2. ALL containers receive SIGTERM       │
│     simultaneously (after preStop)       │
│                                          │
│  3. Grace period is shared across ALL    │
│     containers (not per-container)       │
│                                          │
│  4. SIGKILL sent to any container still  │
│     alive when grace period expires      │
└──────────────────────────────────────────┘
```

With native sidecars (Kubernetes 1.28+, `restartPolicy: Always` in initContainers):
- Sidecar containers are terminated AFTER main containers exit
- They get their own SIGTERM after all regular containers are gone

## Debugging Stuck Terminating Pods

```bash
# Check pod age vs termination time
kubectl get pod <name> -o jsonpath='{.metadata.deletionTimestamp}'

# Check if finalizers are blocking
kubectl get pod <name> -o jsonpath='{.metadata.finalizers}'

# Check if node is reachable
kubectl get node $(kubectl get pod <name> -o jsonpath='{.spec.nodeName}') \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'

# Check container states
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[*].state}'

# Nuclear option — remove from API (containers may still run!)
kubectl delete pod <name> --grace-period=0 --force
```

### Common Causes of Stuck Terminating

| Cause | Fix |
|-------|-----|
| Finalizer not cleared | Identify the controller, fix it, or patch finalizers to null |
| Node unreachable | Wait for node recovery, or force delete if node is gone |
| preStop hook hanging | Fix the hook, or reduce `terminationGracePeriodSeconds` |
| Volume unmount stuck | Check CSI driver logs, force detach the volume |
| Container ignores SIGTERM | Fix signal handling in application code |

## Signals Reference

| Signal | When Sent | Can Be Caught | Purpose |
|--------|-----------|:-------------:|---------|
| SIGTERM (15) | After preStop completes | Yes | Polite shutdown request |
| SIGKILL (9) | After grace period expires | No | Forced immediate death |

Your application MUST handle SIGTERM for graceful shutdown:

```bash
# Common pattern in shell scripts:
#!/bin/bash
trap 'echo "SIGTERM received"; cleanup; exit 0' SIGTERM

# Common pattern in entrypoint wrappers:
exec "$@"  # Use exec so PID 1 is YOUR process, not bash
```

If PID 1 is a shell that doesn't `exec`, SIGTERM goes to the shell, not your application.

## Quick Reference

```bash
# Normal delete (graceful, uses terminationGracePeriodSeconds)
kubectl delete pod <name>

# Custom grace period
kubectl delete pod <name> --grace-period=60

# Force delete (immediate API removal, use with caution)
kubectl delete pod <name> --grace-period=0 --force

# Delete all pods in a namespace
kubectl delete pods --all -n <namespace>

# Delete pods by label
kubectl delete pods -l app=nginx

# Watch termination progress
kubectl get pod <name> -w

# Check why pod is stuck Terminating
kubectl get pod <name> -o yaml | grep -E "deletionTimestamp|finalizers|nodeName"
```
