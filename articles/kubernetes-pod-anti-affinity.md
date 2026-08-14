# Kubernetes Pod Anti-Affinity

## What Is It?

Pod anti-affinity tells the Kubernetes scheduler to **avoid placing a pod on a node that already runs a matching pod**. It is the primary mechanism for spreading replicas across nodes, availability zones, or any other topology.

Without anti-affinity, the scheduler may place all replicas of a deployment on the same node — which means a single node failure or drain takes down all replicas simultaneously.

---

## Types

### requiredDuringSchedulingIgnoredDuringExecution (hard)

The scheduler **will not** place the pod on a node that violates the rule. If no suitable node exists, the pod stays `Pending`.

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname
```

### preferredDuringSchedulingIgnoredDuringExecution (soft)

The scheduler **tries to** avoid placing the pod on a node that violates the rule, but will do so if no other option exists. The pod will never be `Pending` because of this rule.

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: myapp
          topologyKey: kubernetes.io/hostname
```

---

## topologyKey

The `topologyKey` defines the scope of the anti-affinity rule. It maps to a node label.

| topologyKey | Scope |
|-------------|-------|
| `kubernetes.io/hostname` | Spread across individual nodes |
| `topology.kubernetes.io/zone` | Spread across availability zones |
| `topology.kubernetes.io/region` | Spread across regions |

---

## Examples

### Spread across nodes (hard)

Guarantees no two replicas of the same app land on the same node.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: kubernetes.io/hostname
      containers:
        - name: myapp
          image: myapp:latest
```

### Spread across availability zones (soft)

Prefers different AZs but won't block scheduling if only one AZ is available.

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: myapp
          topologyKey: topology.kubernetes.io/zone
```

### Combine both — AZ hard, node soft

Requires different AZs, but only prefers different nodes within the same AZ.

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: topology.kubernetes.io/zone
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: myapp
          topologyKey: kubernetes.io/hostname
```

---

## Real-World Case: ebs-csi-controller

Both replicas of `ebs-csi-controller` ended up on the same node. When that node was drained, both replicas had to be evicted — the PDB caused retries until one was rescheduled elsewhere.

Adding anti-affinity prevents this:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: ebs-csi-controller
          topologyKey: kubernetes.io/hostname
```

`preferred` is used here (not `required`) because `ebs-csi-controller` typically runs 2 replicas — using `required` would block scheduling if only 1 node is available.

---

## Namespace Scope

By default, anti-affinity matches pods **across all namespaces**. If your label selector is generic (e.g., `app: web`), you might unintentionally conflict with pods in other namespaces.

Scope it with `namespaces` or `namespaceSelector`:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname
        namespaces:
          - production
```

Or using `namespaceSelector` (Kubernetes 1.24+):

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname
        namespaceSelector:
          matchLabels:
            environment: production
```

---

## Using matchExpressions

For more flexible matching, use `matchExpressions` instead of `matchLabels`. This is useful when you want to spread all pods from a team or a group of related services.

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: team
                operator: In
                values:
                  - platform
                  - infra
          topologyKey: kubernetes.io/hostname
```

Available operators: `In`, `NotIn`, `Exists`, `DoesNotExist`.

---

## Performance Considerations

Anti-affinity with `requiredDuringScheduling` can be **expensive on large clusters**. The scheduler must evaluate all pods matching the label selector across all nodes to determine feasibility.

Guidelines:
- On clusters with **hundreds of nodes and thousands of pods**, prefer `preferred` over `required` when possible
- Use narrow label selectors to reduce the number of pods the scheduler must evaluate
- Consider `topologySpreadConstraints` as a lighter-weight alternative for even distribution
- Monitor scheduler latency (`scheduler_scheduling_algorithm_duration_seconds`) if you notice slow pod placement

---

## Common Pitfalls

### Overly broad label selectors

If your anti-affinity label selector matches pods you didn't intend to conflict with, unrelated deployments can block each other from scheduling.

```yaml
# Bad — too broad, matches any pod with role=backend across all namespaces
labelSelector:
  matchLabels:
    role: backend

# Good — specific to your app
labelSelector:
  matchLabels:
    app: myapp
    component: api
```

### Anti-Affinity + PDB drain deadlocks

When a node is drained, the following deadlock can occur:

1. Pod must be evicted from the node being drained
2. PDB blocks eviction because minimum availability would be violated
3. The replacement pod can't schedule on another node because anti-affinity blocks it (e.g., the only other node already has a replica)
4. Drain hangs indefinitely

Mitigations:
- Use `preferred` instead of `required` when replica count is close to node count
- Ensure you have **more schedulable nodes than replicas**
- Set PDB `maxUnavailable` instead of `minAvailable` when possible
- Use `--delete-emptydir-data` and appropriate drain timeouts

---

## Anti-Affinity vs Topology Spread Constraints

Kubernetes 1.19+ introduced `topologySpreadConstraints` as a more flexible alternative.

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule   # hard — like required
    labelSelector:
      matchLabels:
        app: myapp
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway  # soft — like preferred
    labelSelector:
      matchLabels:
        app: myapp
```

| | Pod Anti-Affinity | Topology Spread Constraints |
|---|---|---|
| Spread granularity | Binary (same node or not) | Numeric skew (max difference between nodes) |
| Multiple topologies | Separate rules | Multiple constraints in one block |
| Even distribution | Not guaranteed | Controlled via `maxSkew` |
| Kubernetes version | All versions | 1.19+ (stable) |

For most use cases, `topologySpreadConstraints` is the **preferred modern approach**.

---

## When to Use Hard vs Soft

| Scenario | Recommendation |
|----------|---------------|
| Replicas > available nodes | `preferred` — hard would leave pods `Pending` |
| Replicas ≤ available nodes | `required` — guarantees spread |
| Critical HA workloads | `required` across AZs |
| Best-effort spreading | `preferred` with high weight |
| Single-node clusters (dev) | Skip anti-affinity entirely |

---

## Verify Spread

```bash
# Check which nodes pods are running on
kubectl get pods -o wide -l app=myapp

# Check pod affinity rules
kubectl get pod <pod-name> -o jsonpath='{.spec.affinity}' | jq .

# Check why a pod is pending (often anti-affinity conflict)
kubectl describe pod <pod-name>
```

---

## References

- [Kubernetes Docs: Affinity and anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)
- [Kubernetes Docs: Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kubernetes Docs: Pod Disruption Budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
