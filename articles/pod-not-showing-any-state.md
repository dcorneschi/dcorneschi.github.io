# Troubleshooting Failing Workloads When No Pod Shows Up in Any State

Sometimes a workload is clearly broken — a Deployment isn't serving, a Job never runs — but
`kubectl get pods` shows **nothing**. No `Pending`, no `CrashLoopBackOff`, no `Failed`, no
pod at all.

This is confusing because most pod troubleshooting assumes a pod object exists to inspect.
When there is no pod, the failure is one of three shapes:

1. **The pod was never created** — the failure is one level up, at the controller or
   admission layer, so no pod object ever came into being.
2. **The pod was created, failed, and was deleted** — it flickered through a failed state
   and got garbage-collected or cleaned up before you looked.
3. **You are looking in the wrong place** — wrong namespace, selector, or cluster context.

The single most important mindset shift: **stop looking at pods and look at events and the
parent object.** That is where "never created" and "already gone" failures leave their trace.

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -50
```

> Events have a short TTL (~1 hour). For failures older than that, you need the controller's
> own logs or the cluster audit log (see the end of this article).

---

## Case 1: The pod was never created (controller/admission rejected it)

The controller (ReplicaSet, Job, DaemonSet, etc.) tried to create the pod and something
rejected the `CREATE` call, so no pod object exists. Common blockers:

- **Admission webhooks** — a validating or mutating webhook denies the pod (or the webhook
  is down and set to `failurePolicy: Fail`).
- **Pod Security Admission** or a policy engine (**OPA/Gatekeeper**, **Kyverno**) blocks the
  spec.
- **ResourceQuota** exceeded for the namespace.
- **Missing/forbidden ServiceAccount** or RBAC preventing pod creation.
- The **controller itself is broken** or not running.

The evidence is on the **controller**, not on a (non-existent) pod:

```bash
kubectl describe replicaset <rs>          # look for FailedCreate events
kubectl describe deployment <deploy>
kubectl describe job <job>
kubectl get events -A --field-selector reason=FailedCreate
```

A typical `FailedCreate` event names the blocker directly, e.g. an admission webhook denial
or a quota violation. Fix the blocker (relax/repair the webhook or policy, raise the quota,
grant RBAC) and the controller retries creation on its own.

---

## Case 2: The pod was created, failed, and was deleted (churn / GC)

The pod briefly existed in `Failed`/`Error`, then was removed before a point-in-time
`kubectl get pods` could catch it. Causes:

- **Jobs/CronJobs cleaning up fast** — a low `backoffLimit`, or `ttlSecondsAfterFinished`
  deleting finished/failed pods quickly.
- **Preemption** — a higher-priority pod preempted a **bare** pod (no controller), which
  deletes it permanently. (See `pod-pending-insufficient-resources.md`.)
- **Terminated-pod garbage collection** — once the number of terminated pods exceeds the
  cluster threshold, the controller manager reaps the oldest failed pods.
- **Fast restart loops** where the pod is recreated under a new name each time.

Catch the flicker by watching instead of snapshotting, and by widening the query to all
namespaces and all states:

```bash
kubectl get pods -A --watch
kubectl get pods -A -o wide | grep -Ei "error|failed|evicted"
kubectl get events -A --sort-by=.lastTimestamp
```

For Jobs specifically, the Job object retains the failure count even after its pods are gone:

```bash
kubectl describe job <job>     # Pods Statuses / Failed count / backoffLimit reached
```

---

## Case 3: Jobs and CronJobs that never spawned a pod

A `CronJob` can fail to create a `Job`, or a `Job` can fail to create its pod — so no pod is
ever produced. Watch for:

- A bad schedule, or a **missed `startingDeadlineSeconds`** so the run was skipped.
- **`concurrencyPolicy: Forbid`** with a previous run still active, so new runs are
  suppressed.
- The `Job` created but its pod template is rejected (falls back to Case 1).

```bash
kubectl get cronjob,job -A
kubectl describe cronjob <name>    # look for "missed schedule" / creation warnings
kubectl get events -A --field-selector involvedObject.kind=CronJob
```

---

## Case 4: Desired replicas is zero

There is genuinely nothing to show because nothing is desired.

- A Deployment/StatefulSet manually set to `replicas: 0`.
- A **HorizontalPodAutoscaler** or **KEDA `ScaledObject`** that scaled the workload **to
  zero** (scale-to-zero, or a broken metrics source pinning it at the minimum).

```bash
kubectl get deploy,statefulset,rs -A -o wide     # compare DESIRED vs CURRENT vs READY
kubectl get hpa -A
kubectl describe hpa <name>                        # check current/target metrics and events
```

---

## Case 5: The namespace is Terminating

A namespace stuck in `Terminating` (usually blocked finalizers) will not admit new pods, and
existing ones are being torn down — so `kubectl get pods` in it can look empty or transient.

```bash
kubectl get ns
kubectl get namespace <ns> -o yaml     # inspect .status and .spec.finalizers
```

If a namespace is stuck terminating, the workloads inside it will never start. Resolve the
blocking finalizers (identify what owns them) rather than force-removing finalizers blindly,
which can orphan resources.

---

## Case 6: Wrong namespace, selector, or cluster

The most mundane cause, and worth ruling out first.

- **`kubectl get pods` defaults to the `default` namespace.** A workload elsewhere shows
  nothing until you pass `-A` or `-n`.

  ```bash
  kubectl get pods -A
  kubectl get pods -n <namespace>
  ```

- **A label selector** on your command hides pods that don't match.

- **Wrong cluster/context** in multi-cluster setups — you're looking at cluster B while the
  failure is in cluster A.

  ```bash
  kubectl config current-context
  kubectl config get-contexts
  ```

---

## A general triage flow

1. **Confirm scope.** `kubectl get pods -A` and `kubectl config current-context` — rule out
   namespace/cluster mistakes.
2. **Check the parent object.** `kubectl describe deploy/rs/job/cronjob <name>` — look for
   `FailedCreate`, missed schedules, or `replicas: 0`.
3. **Read recent events cluster-wide.** `kubectl get events -A --sort-by=.lastTimestamp` —
   this surfaces admission denials, quota errors, and preemptions.
4. **Watch for churn.** `kubectl get pods -A --watch` for a minute to catch pods that appear
   and vanish.
5. **Check scalers.** `kubectl get hpa -A` and any KEDA `ScaledObject`s for scale-to-zero.
6. **Go older than the event TTL.** If the failure predates the ~1h event window, use
   controller logs or the audit log (below).

---

## When events have aged out: controller logs and audit logs

Events are ephemeral. For failures that happened earlier, go to the components that made the
decision:

```bash
# The controller that should have created the pod
kubectl -n kube-system logs deploy/<controller>          # operators, custom controllers
kubectl -n kube-system logs kube-controller-manager-<node>   # (where accessible)

# The scheduler, for placement/preemption decisions
kubectl -n kube-system logs kube-scheduler-<node>
```

On managed platforms (EKS/AKS/GKE) the control-plane component logs and the **API server
audit log** are exposed through the provider's logging service rather than `kubectl`. The
audit log is the definitive record of "was this pod ever requested, and what rejected it,"
which is exactly the question Case 1 raises.

---

## TL;DR

- No pod in any state = the pod was **never created**, was **created-then-deleted**, or you
  are **looking in the wrong place**.
- Stop inspecting pods; inspect **events and the parent object**:
  `kubectl get events -A --sort-by=.lastTimestamp` and
  `kubectl describe <deploy|rs|job|cronjob>`.
- Rule out the mundane first: `-A` for all namespaces, `current-context` for the right
  cluster, and `replicas`/HPA for scale-to-zero.
- `FailedCreate` events point at admission webhooks, policies, quotas, or RBAC.
- Watch (`--watch`) to catch fast churn; use **controller/scheduler logs and the audit log**
  for anything older than the ~1h event TTL.
