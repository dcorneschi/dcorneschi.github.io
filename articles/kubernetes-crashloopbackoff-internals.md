# What Happens When a Pod Crashes — CrashLoopBackOff Internals

How the kubelet detects container exits, applies exponential backoff, and transitions a pod into CrashLoopBackOff — the internal state machine, restart policies, and backoff reset logic.

Note: For troubleshooting CrashLoopBackOff when logs are empty (exit codes, entrypoint override, OOMKill detection), see the dedicated troubleshooting guide.

## High-Level Flow

```
Container exits (crash)
        │
        ▼
┌──────────────┐     ┌─────────────────┐     ┌───────────────────┐
│   Kubelet    │────▶│ Restart Policy  │────▶│  Backoff Timer    │
│  detects exit│     │ evaluation      │     │  (exponential)    │
└──────────────┘     └─────────────────┘     └───────────────────┘
                                                       │
                                                       ▼
                                              ┌──────────────────┐
                                              │  Wait backoff    │
                                              │  duration before │
                                              │  restarting      │
                                              └──────────────────┘
                                                       │
                                                       ▼
                                              ┌──────────────────┐
                                              │  CRI: Create +   │
                                              │  Start container │
                                              └──────────────────┘
```

## The Kubelet's Container State Machine

The kubelet maintains a state for each container in a pod:

```
┌──────────┐         ┌───────────┐         ┌────────────┐
│ Waiting  │────────▶│  Running  │────────▶│ Terminated │
│          │         │           │         │            │
│ reason:  │         │ startedAt │         │ exitCode   │
│ - Pending│         │           │         │ reason:    │
│ - Backoff│◀────────┼───────────┼─────────│ - Error    │
│          │  (if    │           │         │ - Completed│
│          │  restart│           │         │ - OOMKilled│
└──────────┘  policy)└───────────┘         └────────────┘
```

After a container terminates, the kubelet checks the restart policy:

| `restartPolicy` | Container Exit | Action |
|----------------|---------------|--------|
| `Always` | Any exit (0 or non-zero) | Restart with backoff |
| `OnFailure` | Non-zero exit code | Restart with backoff |
| `OnFailure` | Exit code 0 | Do not restart |
| `Never` | Any exit | Do not restart |

## Exponential Backoff Progression

The kubelet uses exponential backoff starting at 10 seconds, doubling each time, capping at 5 minutes:

```
Crash #1 → Restart after 10s
Crash #2 → Restart after 20s
Crash #3 → Restart after 40s
Crash #4 → Restart after 80s
Crash #5 → Restart after 160s
Crash #6 → Restart after 300s (5 min cap)
Crash #7 → Restart after 300s
Crash #8 → Restart after 300s
...
```

```
Backoff Duration
     ▲
300s │              ┌───────────────────────────
     │              │
160s │         ┌────┘
     │         │
 80s │     ┌───┘
     │     │
 40s │   ┌─┘
     │   │
 20s │  ┌┘
 10s │ ┌┘
     │─┘
     └──────────────────────────────────────────▶ Crash count
       1    2    3    4    5    6    7    8
```

### Backoff Reset

The backoff timer resets to 10s after a container runs successfully for **10 minutes** (the `stableRunningThreshold`). This means:

```
Crash → 10s wait → Start → runs for 12 minutes → Crash
                                                    │
                                                    ▼
                                            Backoff resets to 10s
                                            (container was stable)
```

If the container keeps crashing within 10 minutes of starting, the backoff continues escalating.

## CrashLoopBackOff State

CrashLoopBackOff is NOT a pod phase — it's a container `waiting.reason`. The pod phase stays `Running` (because the pod sandbox is alive):

```yaml
status:
  phase: Running                    # Pod phase — still "Running"
  containerStatuses:
  - name: app
    state:
      waiting:
        reason: CrashLoopBackOff    # Container state — waiting to restart
        message: "back-off 2m40s restarting failed container"
    lastState:
      terminated:
        exitCode: 1
        reason: Error
        startedAt: "2024-03-15T10:00:00Z"
        finishedAt: "2024-03-15T10:00:02Z"
    restartCount: 5
    ready: false
```

### What "CrashLoopBackOff" Means in kubectl Output

```bash
kubectl get pods
# NAME    READY   STATUS             RESTARTS      AGE
# app     0/1     CrashLoopBackOff   5 (2m ago)    10m
```

The status column shows `CrashLoopBackOff` when:
- The container has terminated with an error
- The kubelet is waiting for the backoff timer before restarting
- Once the backoff expires and the container starts again, status briefly shows `Running` (or `Error` if it crashes instantly)

## Internal Kubelet Logic

```
┌──────────────────────────────────────────────────────────────┐
│  Kubelet SyncPod Loop (runs every 10s or on watch event)     │
│                                                              │
│  for each container in pod.spec.containers:                  │
│    current_state = get container state from CRI              │
│                                                              │
│    if current_state == EXITED:                               │
│      if should_restart(restartPolicy, exitCode):             │
│        backoff = calculate_backoff(restartCount)             │
│        time_since_last_exit = now - lastExitTime             │
│                                                              │
│        if time_since_last_exit < backoff:                    │
│          // Still in backoff window                          │
│          set container.state = Waiting(CrashLoopBackOff)     │
│          return  // Will retry on next sync                  │
│                                                              │
│        else:                                                 │
│          // Backoff expired, restart                         │
│          CRI.CreateContainer(...)                            │
│          CRI.StartContainer(...)                             │
│          restartCount++                                      │
│          set container.state = Running                       │
│                                                              │
│    if current_state == RUNNING:                              │
│      if running_duration > 10 minutes:                       │
│        reset backoff to 0                                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Container Exit Codes

The exit code determines what happened:

| Exit Code | Meaning | Common Cause |
|-----------|---------|-------------|
| 0 | Success | Normal completion (only restarts with `Always` policy) |
| 1 | Application error | Unhandled exception, assertion failure |
| 2 | Shell misuse | Bad shell builtin usage |
| 126 | Permission denied | Binary not executable |
| 127 | Command not found | Wrong entrypoint/command path |
| 128+N | Signal N | Container killed by signal |
| 137 (128+9) | SIGKILL | OOMKilled or external kill |
| 139 (128+11) | SIGSEGV | Segmentation fault |
| 143 (128+15) | SIGTERM | Graceful shutdown (normal during pod delete) |

```bash
# Check exit code of last crash:
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'

# Check if OOMKilled:
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
```

## OOMKill Flow

When a container exceeds its memory limit, the kernel's OOM killer terminates it:

```
Container allocates memory beyond limit
        │
        ▼
┌──────────────────┐
│  Linux Kernel    │
│  cgroup memory   │
│  limit reached   │
├──────────────────┤
│  OOM Killer      │──── kills process with SIGKILL (exit 137)
└──────────────────┘
        │
        ▼
┌──────────────────┐
│  Container       │
│  Runtime detects │
│  OOMKilled       │
└──────────────────┘
        │
        ▼
┌──────────────────┐
│ Kubelet          │
│ reason: OOMKilled│
│ exitCode: 137    │
│ restarts with    │
│ backoff          │
└──────────────────┘
```

```bash
# Detect OOMKill from pod events:
kubectl describe pod <name> | grep -i oom

# Check node-level OOM events:
kubectl get events --field-selector reason=OOMKilling

# Check container last state:
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
```

## Pod Conditions During CrashLoopBackOff

```yaml
conditions:
- type: PodScheduled
  status: "True"           # Pod is scheduled to a node
- type: Initialized
  status: "True"           # Init containers completed
- type: ContainersReady
  status: "False"          # Main container is NOT ready
- type: Ready
  status: "False"          # Pod is NOT ready (removed from Service endpoints)
```

The pod stays on the node. The sandbox (pause container) keeps running. Only the application container keeps restarting.

## Impact on Services and Traffic

When a container is in CrashLoopBackOff:
- Pod `Ready` condition = False
- Endpoints controller removes the pod from EndpointSlices
- No traffic is routed to this pod via Services
- During the brief moments when the container starts and passes readiness probe, traffic may briefly flow before the next crash

## Multi-Container Pods

Each container has its own restart count and backoff timer:

```
Pod with containers: [app, sidecar]

app container:     crash → 10s → crash → 20s → crash → 40s → ...
sidecar container: running normally (unaffected by app crashes)
```

The pod's overall `Ready` condition is False if ANY container is not ready.

With init containers:
- Init containers use `restartPolicy: Always` → become sidecars (K8s 1.28+)
- Regular init containers: if they crash, the pod restarts ALL init containers from the beginning

## Restart Count in kubectl

```bash
kubectl get pods
# NAME    READY   STATUS             RESTARTS        AGE
# app     0/1     CrashLoopBackOff   142 (3m ago)    2d

# RESTARTS = total container restarts since pod was scheduled
# (3m ago) = time since last restart attempt
```

The restart count persists for the lifetime of the pod. It never resets unless the pod is deleted and recreated.

## Quick Reference

```bash
# See restart count and last exit info
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].restartCount}'
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'

# Watch pod status changes
kubectl get pod <name> -w

# Check events for context
kubectl describe pod <name> | tail -20

# See container state reason
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].state}'

# Calculate current backoff (approximate):
# min(300, 10 * 2^(restartCount-1))

# Check if it's OOMKilled vs application error
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# "OOMKilled" = memory limit exceeded
# "Error" = non-zero exit (application bug)
# "Completed" = exit 0 (only restarts if policy is Always)
```
