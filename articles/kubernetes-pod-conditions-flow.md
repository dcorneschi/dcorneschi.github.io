# Kubernetes Pod Conditions Flow

How a pod transitions from creation to running — the conditions, phases, and container states that Kubernetes tracks at each step.

## Pod Conditions Flow Diagram

```
                     Kubernetes Pod Conditions Flow


             ┌──────────────────────────────┐
             │          Pod Created         │
             └──────────────┬───────────────┘
                            │
                            ▼
             ┌──────────────────────────────┐         ┌──────────────────────────────────────────┐
             │     PodScheduled: True       │         │ • Evaluates resource requirements        │
             │                              │         │ • Checks node selectors, affinity rules  │
             │  Scheduler assigns to node   │────────►│ • Considers taints/tolerations           │
             │                              │         │ • Assigns pod to specific node           │
             └──────────────┬───────────────┘         │ • Occurs within seconds if resources     │
                            │                         │   available                              │
                            │                         │ ⚠ Remains TRUE throughout pod lifetime   │
                            │                         └──────────────────────────────────────────┘
                            ▼
             ┌──────────────────────────────┐         ┌──────────────────────────────────────────┐
             │ PodReadyToStartContainers:   │         │ • Kubelet works with container runtime   │
             │           True               │         │ • Creates pod sandbox (isolated          │
             │                              │────────►│   environment)                           │
             │   Sandbox created,           │         │ • Configures networking (CNI)            │
             │   network configured         │         │ • Assigns IP address                     │
             │                              │         │ • Mounts required volumes                │
             └──────────────┬───────────────┘         │ • Infrastructure ready for containers    │
                            │                         └──────────────────────────────────────────┘
                            │
                            ▼
             ┌──────────────────────────────┐         ┌──────────────────────────────────────────┐
             │       Initialized: True      │         │ • Init containers run sequentially       │
             │                              │         │ • Used for setup, pre-loading data       │
             │   Init containers complete   │────────►│ • Wait for dependencies                  │
             │                              │         │ • Run security/compliance checks         │
             └──────────────┬───────────────┘         │ • Main containers won't start until      │
                            │                         │   complete                               │
                            │                         │ ⚠ Remains TRUE throughout pod lifetime   │
                            │                         └──────────────────────────────────────────┘
                            ▼
             ┌──────────────────────────────┐         ┌──────────────────────────────────────────┐
             │    ContainersReady: True     │         │ • Container images pulled                │
             │                              │         │ • All containers started                 │
             │   All containers running,    │────────►│ • Readiness probes configured            │
             │   probes pass                │         │ • HTTP/TCP/exec checks pass              │
             │                              │         │ • Application ready to handle requests   │
             └──────────────┬───────────────┘         │ • Tracks individual container readiness  │
                            │                         │    Can change multiple times during      │
                            │                         │    lifetime                              │
                            │                         └──────────────────────────────────────────┘
                            ▼
             ┌──────────────────────────────┐         ┌──────────────────────────────────────────┐
             │         Ready: True          │         │ • Pod fully operational                  │
             │                              │         │ • Services route traffic to this pod     │
             │   Pod accepts traffic        │────────►│ • Includes custom readiness gates        │
             │                              │         │ • Load balancers add to endpoint pool    │
             └──────────────────────────────┘         │ • Most critical condition for production │
                                                      │    Can change multiple times during      │
                                                      │    lifetime                              │
                                                      └──────────────────────────────────────────┘
```

### Legend

- ⚠ = Remains TRUE once set
- 🔄 = Can fluctuate TRUE/FALSE
- All 4 conditions must be TRUE for pod to serve traffic

### Pod Phases (only one at a time)

| Phase | Meaning |
|-------|---------|
| Pending | Initial phase — waiting for scheduling, image pull, or init containers |
| Running | At least 1 container running |
| Succeeded | All containers completed successfully |
| Failed | At least 1 container failed |
| Unknown | Node communication issues |

## Pod Conditions

Every pod has four built-in conditions:

| Condition | Meaning | Set By |
|-----------|---------|--------|
| `PodScheduled` | Pod has been assigned to a node | Scheduler |
| `Initialized` | All init containers have completed | kubelet |
| `ContainersReady` | All containers are running and have started | kubelet |
| `Ready` | Pod is ready to serve traffic (probes pass + readiness gates) | kubelet + readiness gates |

### Condition Structure

```yaml
status:
  conditions:
    - type: PodScheduled
      status: "True"
      lastTransitionTime: "2024-01-15T10:00:00Z"
    - type: Initialized
      status: "True"
      lastTransitionTime: "2024-01-15T10:00:05Z"
    - type: ContainersReady
      status: "True"
      lastTransitionTime: "2024-01-15T10:00:15Z"
    - type: Ready
      status: "True"
      lastTransitionTime: "2024-01-15T10:00:20Z"
```

### The Timeline

```
T+0s     Pod created → Phase: Pending
         PodScheduled: False
         Initialized: False
         ContainersReady: False
         Ready: False

T+2s     Scheduler assigns node
         PodScheduled: True ✓

T+5s     Init containers complete (if any)
         Initialized: True ✓

T+10s    kubelet pulls images, starts containers
         ContainersReady: True ✓

T+15s    Readiness probe passes (if configured)
         Ready: True ✓
         → Pod added to Service endpoints
         → Receives traffic
```

## Pod Phases

The pod's overall lifecycle state:

| Phase | Meaning |
|-------|---------|
| `Pending` | Accepted by the cluster, not yet running (waiting for scheduling, image pull, init containers) |
| `Running` | At least one container is running (or starting/restarting) |
| `Succeeded` | All containers exited with code 0, won't restart |
| `Failed` | All containers terminated, at least one exited non-zero |
| `Unknown` | State cannot be determined (usually node communication lost) |

```sh
# Check pod phase
kubectl get pod <pod> -o jsonpath='{.status.phase}'
```

## Container States

Each container within a pod has its own state:

| State | Meaning |
|-------|---------|
| `Waiting` | Not yet running (image pull, init container dependency, CrashLoopBackOff) |
| `Running` | Executing normally |
| `Terminated` | Finished execution (either success or failure) |

```sh
# Check container states
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].state}' | jq

# Common waiting reasons
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}'
```

### Common Waiting Reasons

| Reason | Meaning |
|--------|---------|
| `ContainerCreating` | Pulling image or setting up volumes |
| `CrashLoopBackOff` | Container keeps crashing, exponential backoff before restart |
| `ImagePullBackOff` | Can't pull the image (wrong name, no auth, registry down) |
| `ErrImagePull` | Initial pull failure |
| `CreateContainerConfigError` | ConfigMap/Secret missing or invalid |
| `PodInitializing` | Init containers still running |

### Common Terminated Reasons

| Reason | Meaning |
|--------|---------|
| `Completed` | Exited with code 0 (success) |
| `Error` | Exited with non-zero code |
| `OOMKilled` | Killed by kernel due to memory limit exceeded |
| `DeadlineExceeded` | `activeDeadlineSeconds` reached |
| `Evicted` | kubelet evicted the pod (node pressure) |

## Condition: PodScheduled

Transitions `False → True` when the scheduler assigns the pod to a node.

### When It Stays False

```sh
# Check why scheduling failed
kubectl describe pod <pod> | grep -A 5 "Events"
```

| Event Message | Cause |
|---------------|-------|
| `0/N nodes are available: N Insufficient cpu` | Not enough CPU capacity |
| `0/N nodes are available: N Insufficient memory` | Not enough memory capacity |
| `didn't match Pod's node affinity/selector` | No node matches scheduling constraints |
| `N node(s) had untolerated taint` | Tainted nodes, missing toleration |
| `didn't satisfy pod topology spread constraints` | Can't maintain even distribution |
| `N node(s) had volume node affinity conflict` | PVC zone doesn't match available nodes |

### How Long Before Timeout

There is no timeout for `PodScheduled=False`. The pod stays `Pending` indefinitely until:
- A matching node becomes available
- Preemption frees resources
- The pod is manually deleted
- A higher-priority pod evicts something

## Condition: Initialized

Transitions `False → True` when **all init containers** have completed successfully (exit code 0).

### Init Container Execution

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db-svc 5432; do sleep 2; done']
    - name: migrate
      image: myapp:migrate
      command: ['./migrate.sh']
  containers:
    - name: app
      image: myapp:latest
```

Execution order:
1. `wait-for-db` runs (blocks until DB is reachable)
2. `wait-for-db` exits 0 → `migrate` starts
3. `migrate` exits 0 → `Initialized=True` → main containers start

### When It Stays False

- Init container is stuck (waiting for a dependency that's down)
- Init container keeps crashing (`CrashLoopBackOff`)
- Init container can't pull its image

```sh
# Check init container status
kubectl get pod <pod> -o jsonpath='{.status.initContainerStatuses}' | jq
```

## Condition: ContainersReady

Transitions `False → True` when ALL containers in the pod have started.

This means:
- Images are pulled
- Containers are created
- Processes are running

This does NOT mean:
- Readiness probes have passed
- The application is actually ready to serve traffic

## Condition: Ready

Transitions `False → True` when:
1. `ContainersReady = True` (all containers running)
2. All readiness probes pass (if configured)
3. All readiness gates are satisfied (if configured)

### Ready = Traffic Flows

When `Ready=True`:
- Pod is added to Service endpoints
- Load balancer target group registers the pod
- Ingress routes traffic to the pod

When `Ready=False`:
- Pod is removed from Service endpoints
- No traffic is routed to the pod
- Pod stays in the cluster but doesn't serve

### Readiness Probe

```yaml
spec:
  containers:
    - name: app
      readinessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
        failureThreshold: 3
        successThreshold: 1
```

The pod won't become `Ready` until the probe succeeds `successThreshold` consecutive times.

If the probe later fails `failureThreshold` consecutive times, `Ready` transitions back to `False` and the pod is removed from Service endpoints.

## Readiness Gates (Pod Ready++)

Custom conditions that must be `True` for the pod to be considered `Ready`, in addition to the standard checks.

### Use Case: AWS Load Balancer Controller

The AWS LB Controller uses readiness gates to prevent traffic before the target group registration is complete:

```yaml
spec:
  readinessGates:
    - conditionType: target-health.elbv2.k8s.aws
```

The pod won't be `Ready` until the ALB/NLB target group confirms the pod is healthy.

### How Readiness Gates Appear in Conditions

```yaml
status:
  conditions:
    - type: PodScheduled
      status: "True"
    - type: Initialized
      status: "True"
    - type: ContainersReady
      status: "True"
    - type: target-health.elbv2.k8s.aws    # Custom readiness gate
      status: "True"
    - type: Ready
      status: "True"                        # Only True when ALL above are True
```

If the readiness gate is `False`, the pod stays `Ready=False` even though `ContainersReady=True`.

## Common Troubleshooting Patterns

### Pod Stuck: Pending (PodScheduled=False)

```sh
kubectl describe pod <pod> | grep -A 5 "Events"
# Look for FailedScheduling events

# Check cluster capacity
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check if nodes match the pod's constraints
kubectl get nodes --show-labels
```

### Pod Stuck: Pending (Initialized=False)

```sh
# Check init container logs
kubectl logs <pod> -c <init-container-name>

# Check init container status
kubectl get pod <pod> -o jsonpath='{.status.initContainerStatuses}' | jq
```

### Pod Stuck: Running (ContainersReady=False)

```sh
# Check container states
kubectl describe pod <pod> | grep -A 10 "State:"

# Common: CrashLoopBackOff — check logs
kubectl logs <pod> --previous
```

### Pod Running But Ready=False

```sh
# Check readiness probe
kubectl describe pod <pod> | grep -A 5 "Readiness"

# Check readiness gate conditions
kubectl get pod <pod> -o jsonpath='{.status.conditions}' | jq '.[] | select(.status=="False")'

# Manually test the health endpoint
kubectl exec <pod> -- curl -s localhost:8080/health
```

### Pod Was Ready, Then Became NotReady

```sh
# Readiness probe started failing
kubectl describe pod <pod> | grep -A 3 "Warning  Unhealthy"

# Check recent events
kubectl get events --field-selector involvedObject.name=<pod> --sort-by='.lastTimestamp'

# Check if it's a resource issue
kubectl top pod <pod>
```

## Probes: The Three Types

| Probe | When Checked | Action on Failure |
|-------|-------------|-------------------|
| `startupProbe` | During container startup only | Kill + restart container |
| `livenessProbe` | Continuously after startup | Kill + restart container |
| `readinessProbe` | Continuously after startup | Remove from Service endpoints (no restart) |

### Startup → Liveness → Readiness

```yaml
spec:
  containers:
    - name: app
      startupProbe:           # First: wait for app to start
        httpGet:
          path: /health
          port: 8080
        failureThreshold: 30  # 30 × 10s = 5 minutes to start
        periodSeconds: 10
      livenessProbe:          # Then: restart if app hangs
        httpGet:
          path: /health
          port: 8080
        periodSeconds: 10
        failureThreshold: 3
      readinessProbe:         # Always: control traffic routing
        httpGet:
          path: /ready
          port: 8080
        periodSeconds: 5
        failureThreshold: 2
```

Flow:
1. `startupProbe` runs until it passes (or container is killed after `failureThreshold`)
2. Once startup passes, `livenessProbe` and `readinessProbe` begin
3. `livenessProbe` failure → container restart
4. `readinessProbe` failure → pod removed from endpoints (but container keeps running)

## Condition Transition Events

Kubernetes generates events when conditions change:

```sh
# Watch condition transitions
kubectl get events --field-selector involvedObject.name=<pod> --sort-by='.lastTimestamp'

# Common events:
# Scheduled       — PodScheduled became True
# Pulling         — Pulling image
# Pulled          — Image pulled successfully
# Created         — Container created
# Started         — Container started
# Unhealthy       — Probe failed
# Killing         — Container being terminated
# BackOff         — Back-off restarting failed container
```

## Pod Deletion Flow

When a pod is deleted, conditions transition in reverse:

```
kubectl delete pod <pod>
    │
    ▼
1. Pod set to "Terminating" (still in endpoints briefly)
2. Endpoints controller removes pod from Service
3. kubelet sends SIGTERM to containers
4. preStop hook runs (if configured)
5. Grace period countdown (default 30s)
6. If still running after grace period → SIGKILL
7. Pod object removed from API server
```

The `Ready=False` transition happens at step 2 — traffic stops flowing before the container is killed.

### Graceful Shutdown Gap

There's a race between the endpoints update (step 2) and the SIGTERM (step 3). Some traffic may still arrive after SIGTERM. Use a `preStop` hook with a small sleep to handle this:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "5"]   # Wait for endpoints to propagate
```

## kubectl Quick Reference

```sh
# All conditions
kubectl get pod <pod> -o jsonpath='{.status.conditions}' | jq

# Specific condition
kubectl get pod <pod> -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'

# Phase
kubectl get pod <pod> -o jsonpath='{.status.phase}'

# Container states
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].state}' | jq

# Pods not ready
kubectl get pods --field-selector status.phase=Running -o json | jq '.items[] | select(.status.conditions[] | select(.type=="Ready" and .status=="False")) | .metadata.name'

# All pending pods
kubectl get pods -A --field-selector status.phase=Pending

# Pod restart count
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}'
```

## Gotchas

- **`Ready` ≠ `Running`**: A pod can be `Running` (phase) but `Ready=False` (readiness probe failing). It won't receive traffic.
- **`ContainersReady` ≠ `Ready`**: If readiness gates are configured, `ContainersReady=True` doesn't mean `Ready=True`.
- **Readiness probes don't restart containers**: Only liveness probes do. A pod stuck in `Ready=False` will stay that way forever unless the probe starts passing or the pod is manually deleted.
- **Init containers block everything**: If an init container hangs, the main containers never start. There's no timeout — it blocks indefinitely (unless the container itself has a timeout).
- **Pod can go from Ready=True back to Ready=False**: If a readiness probe starts failing after the pod was ready, it's removed from endpoints. This is by design for graceful degradation.
- **Events are garbage collected**: Kubernetes only keeps events for 1 hour by default. If you're debugging an old issue, events may be gone.
- **`Unknown` phase means node is unreachable**: The API server can't confirm the pod's state. The node may be down or network-partitioned.
