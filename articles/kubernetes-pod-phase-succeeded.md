# Kubernetes Pod Phases and the Succeeded Phase

## Overview

In Kubernetes, `status.phase` is a high-level summary of where a pod is in its
lifecycle. `status.phase=Succeeded` is one specific state. This article explains
the phases and what "Succeeded" means in practice.

## Pod Phases

A pod moves through these phases:

1. **Pending** — accepted by the cluster but not running yet (waiting on
   scheduling, image pulls, etc.).
2. **Running** — bound to a node with at least one container executing.
3. **Succeeded** — all containers terminated successfully (exit code 0) and
   will not be restarted.
4. **Failed** — all containers terminated and at least one exited non-zero.
5. **Unknown** — the pod's state can't be determined (usually a node
   communication issue).

## What "Succeeded" Means

When a pod is `Succeeded`:

- **All containers completed successfully** (exit code 0).
- **The pod will not restart** — it's considered done.
- **It's the expected end state** for run-to-completion workloads.

This only happens with a `restartPolicy` of `OnFailure` or `Never`. A pod with
`restartPolicy: Always` (the default for Deployments) never reaches `Succeeded` —
its containers are always restarted, so it stays `Running`.

## Common Sources of Succeeded Pods

- **Jobs** — one-time tasks: data processing, backups, migrations.
- **CronJobs** — scheduled tasks that run periodically.
- **Batch processing** — ETL jobs, report generation, and similar.

> Init containers don't produce a `Succeeded` *pod* — they run to completion
> inside a pod that then continues to its main containers. The pod phase reflects
> the main containers.

## Finding Succeeded Pods

```bash
# All succeeded pods across namespaces
kubectl get pods --all-namespaces --field-selector=status.phase=Succeeded

# In one namespace
kubectl get pods -n <namespace> --field-selector=status.phase=Succeeded
```

Useful for auditing completed jobs, confirming scheduled tasks finished, and
cleanup.

## Cleaning Up Succeeded Pods

```bash
# Delete all succeeded pods in a namespace
kubectl delete pods -n <namespace> --field-selector=status.phase=Succeeded

# Cluster-wide
kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded
```

> Better long-term: set `ttlSecondsAfterFinished` on Jobs so the control plane
> cleans up finished pods automatically, and tune CronJob
> `successfulJobsHistoryLimit` / `failedJobsHistoryLimit` to bound retained runs.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: report
spec:
  ttlSecondsAfterFinished: 3600   # delete 1 hour after completion
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: report
        image: my-report:latest
```

## Phase vs Condition vs Container State

`status.phase` is a coarse summary. For detail, look at:

- **`status.conditions`** — e.g. `PodScheduled`, `Ready`, `ContainersReady`.
- **`status.containerStatuses[].state`** — `waiting` / `running` / `terminated`,
  with the exit code and reason for terminated containers.

```bash
# Why a container terminated (exit code + reason)
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].state.terminated}'
```

A container's `state` is one of `waiting`, `running`, or `terminated`. The pod
phase is derived from the combination of its containers' states:

| Pod phase | Container states | Meaning |
|-----------|------------------|---------|
| Pending | All `waiting` | Containers not started yet |
| Running | Mix of `running`/`waiting` | At least one container running |
| Running | All `running` | All containers running |
| Succeeded | All `terminated`, exit 0 | Pod completed successfully |
| Failed | One+ `terminated`, exit ≠ 0 | Pod failed |
| Unknown | Any | Node communication lost |

Example status showing both levels:

```yaml
status:
  phase: Running
  conditions:
    - type: PodScheduled
      status: "True"
    - type: Initialized
      status: "True"
    - type: ContainersReady
      status: "True"
    - type: Ready
      status: "True"
  containerStatuses:
    - name: app
      ready: true
      restartCount: 0
      state:
        running:
          startedAt: "2024-01-10T10:00:08Z"
```

## ContainerCreating Is Not a Phase

`ContainerCreating` is one of the most-seen statuses in `kubectl get pods`, but
it's **not** a pod phase. It's the `reason` on a container's `waiting` state
while the pod is still in the `Pending` phase:

```text
Pending (scheduled) → image pull → ContainerCreating → Running
```

During `ContainerCreating`, the kubelet is:

- unpacking image layers and preparing the container filesystem,
- creating and mounting volumes (PVCs, Secrets, ConfigMaps, the SA token),
- configuring networking and DNS,
- generating the runtime spec (cgroups, limits) and starting the process.

Mapping:

- `status.phase`: `Pending`
- `status.containerStatuses[*].state.waiting.reason`: `ContainerCreating`

### Why it stalls

- Large image pulls or a slow/unreachable registry.
- Volume problems — missing PVC/Secret/ConfigMap or a mount failure.
- Node pressure (DiskPressure/MemoryPressure/PIDPressure) or no disk for image extraction.
- Runtime errors (cgroup/permission issues).

### Debugging

```bash
kubectl describe pod <pod>                 # events show the exact hold-up
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].state.waiting.reason}'
kubectl describe node <node>               # node conditions and allocatable
kubectl get pvc; kubectl get secret; kubectl get configmap   # referenced objects exist?
```

If setup succeeds, the container moves to `running`. If it fails, the waiting
reason changes to something like `CreateContainerConfigError`,
`CreateContainerError`, or `ImagePullBackOff` instead of `ContainerCreating`.

## Note

This differs from long-running services (web servers, APIs), which stay in the
`Running` phase indefinitely and never reach `Succeeded`.

## Related

- [Kubernetes Pod Conditions Flow](articles/kubernetes-pod-conditions-flow.md)
- [Kubernetes CronJob Examples & Reference](articles/kubernetes-cronjob-examples.md)
- [Kubernetes Pod Troubleshooting Guide](articles/kubernetes-pod-troubleshooting-guide.md)

## Skills Practiced

- Distinguishing the five pod phases and what triggers `Succeeded`
- Understanding how `restartPolicy` determines whether a pod can succeed
- Listing and cleaning up completed pods with `--field-selector=status.phase=Succeeded`
- Using `ttlSecondsAfterFinished` and CronJob history limits for automatic cleanup
