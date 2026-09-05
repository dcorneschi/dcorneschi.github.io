# In-Place Pod Resize with the Vertical Pod Autoscaler (VPA)

For most of Kubernetes' history, changing a Pod's CPU or memory meant **recreating the
Pod**. That's disruptive: connections drop, caches cold-start, and stateful or
long-running workloads take a real hit. The Vertical Pod Autoscaler (VPA) inherited that
limitation — to apply a new recommendation it had to **evict** the Pod and let the
controller recreate it.

Two features change that:

- **In-Place Pod Resize** — a core Kubernetes feature (GA in **v1.35**) that makes a
  container's CPU/memory requests and limits **mutable on a running Pod**, via a dedicated
  `resize` subresource, often without a restart.
- **VPA `InPlaceOrRecreate` mode** — a newer VPA update mode that applies recommendations
  by patching the running Pod through that `resize` subresource, falling back to eviction
  only when in-place isn't possible.

Together they let VPA right-size Pods with far less disruption. This article explains how
each piece works, how to wire them up, and the sharp edges to watch.

---

## 1. In-Place Pod Resize: the core mechanism

### Feature state

- **Alpha in 1.27 → Beta in 1.33 → Stable (GA) in 1.35.**
- The feature gate is `InPlacePodVerticalScaling`. As of 1.35 it is **locked on** — you
  can't disable it, and explicitly setting the gate is ignored (no error).
- The `kubectl` client must be **≥ v1.32** to use the `--subresource resize` flag.

### Desired vs actual resources

Two fields matter:

- **Desired:** `spec.containers[*].resources` — now **mutable** for CPU and memory. This is
  what you want.
- **Actual:** `status.containerStatuses[*].resources` — what the container is **currently**
  running with, as confirmed by the kubelet. This is what you check to see if a resize
  landed.

When desired ≠ actual, the kubelet attempts to reconcile (resize) the container. (There's
also `status.containerStatuses[*].allocatedResources`, used internally for scheduling —
for monitoring, focus on `.resources`.)

Mechanically, the kubelet applies the change by **updating the container's Linux cgroup
directly** — it does not delete the Pod or restart the container unless the resize policy
(or the runtime) requires it. The **Pod UID stays the same**, the process keeps running,
and the only visible evidence is the updated resource values. That unchanged UID is a
clean way to prove a change was truly in-place rather than a recreate.

> This is why cgroup **v2** is required on the nodes (next section) — the live cgroup
> update relies on it.

### Triggering a resize: the `resize` subresource

You change resources by patching the Pod's **`resize` subresource** — not the normal Pod
spec path:

```bash
kubectl patch pod resize-demo --subresource resize --patch \
  '{"spec":{"containers":[{"name":"app",
     "resources":{"requests":{"cpu":"800m"},"limits":{"cpu":"800m"}}}]}}'

# Also valid:
# kubectl edit pod resize-demo --subresource resize
# kubectl apply -f updated.yaml --subresource resize --server-side
```

Then verify by comparing spec vs status:

```bash
kubectl get pod resize-demo -o yaml \
  | yq '.status.containerStatuses[0].resources, .status.containerStatuses[0].restartCount'
```

### Resize policy: restart or not, per resource

Each container can declare, **per resource type**, whether applying a change requires a
restart, via `resizePolicy`:

```yaml
spec:
  containers:
  - name: app
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired      # default — apply live, no restart
    - resourceName: memory
      restartPolicy: RestartContainer # restart to apply
    resources:
      requests: { cpu: "700m", memory: "200Mi" }
      limits:   { cpu: "700m", memory: "200Mi" }
```

- **`NotRequired`** (default): apply the change to the running container without restarting.
- **`RestartContainer`**: restart the container to apply — commonly needed for **memory**,
  because many runtimes (notably the JVM and other GC'd/heap-preallocating stacks) can't
  adjust their memory ceiling live.

Interaction rules:

- If **only CPU** changes (and CPU is `NotRequired`), the container resizes in place, no
  restart. `restartCount` stays the same.
- If **memory** changes and memory is `RestartContainer`, the container restarts;
  `restartCount` increments.
- If **both** change and *either* policy is `RestartContainer`, the container restarts.
- If the Pod's overall `restartPolicy: Never`, then every `resizePolicy` **must** be
  `NotRequired` — you can't configure a restart-requiring resize on such a Pod.

### Resize status conditions

The kubelet reports progress via Pod status conditions:

- **`PodResizePending`** — can't grant yet:
  - `reason: Infeasible` — impossible on this node (e.g. more CPU than the node has). Actual
    resources stay at the old values.
  - `reason: Deferred` — not possible right now but might be later (e.g. once another Pod
    leaves); the kubelet retries.
- **`PodResizeInProgress`** — accepted and being applied; usually brief. Actuation errors
  show up here with `reason: Error`.

You can also read `status.observedGeneration` (stable in 1.35) to see which spec generation
the kubelet has acknowledged.

### Hard limitations (important)

In-place resize does **not** do everything. As of recent releases:

- **CPU and memory only.** No other resource types.
- **QoS class is fixed at creation.** A resize must keep the Pod in its original class:
  - *Guaranteed*: requests must still equal limits.
  - *Burstable*: can't make requests==limits for both CPU and memory (that'd become
    Guaranteed).
  - *BestEffort*: can't add requests/limits at all.
- **Memory decrease is best-effort.** With memory `NotRequired`, if current usage exceeds
  the new limit the resize is skipped and stays "in progress" — there's a race, so no hard
  guarantee against OOM.
- **You can't remove** requests/limits once set — only change the values.
- **Not supported:** Windows Pods; Pods under **static CPU/Memory manager** policies;
  non-restartable init containers and ephemeral containers (sidecar containers **can** be
  resized); memory resizes on swap-using Pods unless memory policy is `RestartContainer`.

---

## 2. VPA and why it needed this

The **Vertical Pod Autoscaler** observes historical CPU/memory usage and recommends (and
optionally applies) better requests. Historically its `updateMode` options were:

- **`Off`** — recommend only; apply nothing. (Read recommendations from the VPA object.)
- **`Initial`** — set requests only at Pod **creation**; never touch running Pods.
- **`Recreate`** — **evict** running Pods to apply new recommendations. Disruptive.
- **`Auto`** — historically behaved like `Recreate`. **Deprecated since VPA 1.4.0** because
  its meaning is changing as in-place matures; prefer an explicit mode.

The pain: `Recreate`/`Auto` meant every recommendation change caused a Pod restart. That's
exactly what In-Place Pod Resize fixes.

There's also a structural reason you need VPA (or some controller) rather than just
patching by hand: the `resize` subresource only exists **at the Pod level**. You cannot
resize a **Deployment** in place — editing a Deployment's Pod template rolls the
ReplicaSet and **recreates** the Pods. VPA bridges that gap by patching each running Pod's
`/resize` subresource directly, so the Deployment's Pods get resized without a rollout.

### How the components fit together

The in-place flow through VPA is:

1. **metrics-server** continuously tracks each Pod's CPU/memory usage.
2. The **VPA Recommender** analyzes that history and computes new target requests when a
   Pod is over- or under-provisioned.
3. The **VPA Updater**, instead of evicting the Pod, sends a **patch to the `/resize`
   subresource** via the API server.
4. The **kubelet** on the node receives it and **updates the container's cgroups directly**.

The result: the new resources land on the running Pod, same UID, no restart (unless
`resizePolicy` forces one).

### The `InPlaceOrRecreate` update mode

Newer VPA (the `InPlaceOrRecreate` mode, gated by VPA's own
`InPlaceOrRecreate` feature flag on the VPA components) applies recommendations by patching
the running Pod through the Kubernetes `/resize` subresource:

- It **first tries an in-place resize**. If the underlying cluster supports it and the
  change is feasible, the Pod is updated without recreation.
- It **falls back to eviction + recreation** when in-place is infeasible or stalls (e.g. an
  `Infeasible` condition, or a resize stuck in progress).

Critically, `InPlaceOrRecreate` **reduces** disruption; it does not promise
eviction-free operation. Treat eviction as always still possible.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: InPlaceOrRecreate          # try in-place, fall back to recreate
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed: { cpu: 100m, memory: 128Mi }   # always set floors
      maxAllowed: { cpu: "2",  memory: 2Gi  }     # ...and ceilings
      controlledResources: [cpu, memory]
```

---

## 3. Wiring it up end to end

### Prerequisites

1. **Kubernetes ≥ 1.33** for the feature broadly (GA at **1.35**; on 1.35+ the gate is
   locked on, nothing to enable). On 1.33/1.34 ensure `InPlacePodVerticalScaling` is
   enabled on control plane and all nodes.
2. **Nodes on cgroup v2.** The live cgroup update requires cgroup v2 — confirm per node:

   ```bash
   stat -fc %T /sys/fs/cgroup/
   # Expect: cgroup2fs (or "cgroup2"). "tmpfs" means cgroup v1 → in-place resize won't work.
   ```

3. **metrics-server installed.** VPA reads Pod usage from it; install it before VPA if it
   isn't already running (`kubectl get pods -n kube-system | grep metrics-server`).
4. **VPA installed** at a version that supports `InPlaceOrRecreate`, with the corresponding
   VPA feature flag enabled on the VPA controllers (updater/admission). Install from the
   `kubernetes/autoscaler` repo:

   ```bash
   git clone https://github.com/kubernetes/autoscaler.git
   cd autoscaler/vertical-pod-autoscaler/
   ./hack/vpa-up.sh      # brings up vpa-recommender, vpa-updater, vpa-admission-controller
   # tear down later with ./hack/vpa-down.sh
   ```

5. **`kubectl` ≥ 1.32** if you'll patch manually.
6. **Set initial requests/limits AND a `resizePolicy`** on your containers. The initial
   requests matter for QoS (see the trap below); `memory: RestartContainer` is what you
   want for runtimes that can't grow their heap live.

> **The BestEffort → Burstable trap.** If a container has **no** initial requests/limits,
> its Pod is **BestEffort**. When VPA later adds a request, that would make the Pod
> *Burstable* — but **QoS class is immutable for a running Pod**, so the cluster can't
> resize in place; it's forced to **delete and recreate** the Pod instead. Always set
> initial requests/limits so the Pod starts in the QoS class it will stay in.

### Verify in-place works at all (without VPA)

Before handing control to VPA, confirm the cluster can resize a Pod in place:

```bash
kubectl patch pod resize-demo --subresource resize --patch \
  '{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"800m"},"limits":{"cpu":"800m"}}}]}}'
kubectl get pod resize-demo -o jsonpath='{.status.containerStatuses[0].resources}{"\n"}'
kubectl get pod resize-demo -o jsonpath='{.status.conditions[?(@.type=="PodResizeInProgress")]}{"\n"}'
```

### Then let VPA drive it

Apply the `InPlaceOrRecreate` VPA above. Watch what VPA recommends and whether it applied
in place:

```bash
# Recommendations VPA computed
kubectl describe vpa web-vpa

# Did the running pods get resized without restarts?
kubectl get pods -l app=web \
  -o custom-columns='POD:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount,CPU:.status.containerStatuses[0].resources.requests.cpu,MEM:.status.containerStatuses[0].resources.requests.memory'
```

`restartCount` staying flat while the CPU/memory columns change is the signal that in-place
resize did its job.

---

## 4. Operational gotchas

- **Don't run VPA and HPA on the same metric.** The classic failure is a **death spiral**:
  VPA lowers per-Pod requests, HPA's percentage-based math shifts underneath it, replica
  count swings, per-Pod usage histograms skew, and the loop amplifies. Keep VPA and HPA on
  **different signals** (e.g. HPA on a custom/business metric, VPA on CPU/memory), or don't
  co-manage the same resource.
- **Always set `minAllowed` / `maxAllowed`.** Without bounds, a bad recommendation can
  starve or balloon a workload. Floors and ceilings are your safety rail.
- **Start in `Off` mode.** Observe recommendations for a while before enabling enforcement.
  VPA's recommender is **reactive** and context-blind — it reacts to observed usage, not to
  your knowledge of an upcoming spike.
- **Downsizing: CPU is safe, memory is not.** CPU is **compressible** — the kernel just
  throttles the process, nothing dies. Memory is **non-compressible** — if a container is
  already using more than the new limit, shrinking it in place risks an **OOMKill**. The
  kubelet's protection here is best-effort only (it skips the resize if usage exceeds the
  target, but there's still a race). For memory downsizing, prefer
  `resizePolicy: memory → RestartContainer` so the container comes back cleanly at the
  lower ceiling.
- **QoS class is immutable.** If you rely on *Guaranteed*, every VPA-applied value must keep
  requests==limits, or the resize is rejected.
- **Eviction is still on the table.** `InPlaceOrRecreate` falls back to recreation. Protect
  availability with **PodDisruptionBudgets** and multiple replicas, same as with `Recreate`.
- **Node capacity governs feasibility.** An `Infeasible` `PodResizePending` condition means
  the node can't fit the request; you need headroom, a bigger node, or (on newer versions)
  scheduler preemption for deferred resizes.

---

## 5. Quick reference

| Goal | Command / field |
|---|---|
| Resize a Pod manually | `kubectl patch pod <p> --subresource resize --patch '{...}'` |
| See actual (applied) resources | `status.containerStatuses[*].resources` |
| See desired resources | `spec.containers[*].resources` |
| Control restart-on-resize | `spec.containers[*].resizePolicy[].restartPolicy` (`NotRequired`/`RestartContainer`) |
| Watch resize state | Pod conditions `PodResizePending` (Infeasible/Deferred) / `PodResizeInProgress` |
| VPA update mode for in-place | `spec.updatePolicy.updateMode: InPlaceOrRecreate` |
| VPA bounds | `resourcePolicy.containerPolicies[].minAllowed` / `maxAllowed` |
| View VPA recommendations | `kubectl describe vpa <name>` |
| Confirm no restart happened | `status.containerStatuses[*].restartCount` unchanged |
| Confirm resize was in-place (not recreate) | Pod `metadata.uid` unchanged |
| Verify node supports it | `stat -fc %T /sys/fs/cgroup/` → `cgroup2fs` |
| Install / remove VPA | `./hack/vpa-up.sh` / `./hack/vpa-down.sh` |

---

## Summary

- **In-Place Pod Resize** (GA in Kubernetes **1.35**) makes container CPU/memory mutable on
  a live Pod through the **`resize` subresource**, with per-resource **`resizePolicy`**
  deciding whether a restart is needed (`NotRequired` vs `RestartContainer`).
- Watch **`status.containerStatuses[*].resources`** and the **`PodResizePending` /
  `PodResizeInProgress`** conditions to see what actually happened.
- **VPA `InPlaceOrRecreate`** applies recommendations in place first and only **evicts as a
  fallback**, replacing the old always-disruptive `Recreate`/`Auto` behavior (`Auto` is
  deprecated since VPA 1.4.0).
- It reduces disruption but doesn't eliminate it: **QoS is fixed**, **memory shrink is
  best-effort**, **eviction can still happen**, and the **VPA+HPA death spiral** is real.
  Set `minAllowed`/`maxAllowed`, start in `Off`, keep VPA and HPA on separate metrics, and
  use PDBs.

---

### Sources

- [Kubernetes 1.35: In-Place Pod Resize Graduates to Stable (Kubernetes Blog)](https://kubernetes.io/blog/2025/12/19/kubernetes-v1-35-in-place-pod-resize-ga/)
- [Resize CPU and Memory Resources assigned to Containers (Kubernetes docs)](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)
- [Resize CPU and Memory Resources assigned to Pods (Kubernetes docs)](https://kubernetes.io/docs/tasks/configure-pod-container/resize-pod-resources/)
- [VPA in-place updates support enhancement (kubernetes/autoscaler)](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/enhancements/4016-in-place-updates-support/README.md)
- [VPA quickstart — update modes including InPlaceOrRecreate (kubernetes/autoscaler)](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/docs/quickstart.md)
