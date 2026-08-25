# HPA with scaleDown Behavior

Related: [HPA Documentation](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)

## Prerequisites

The [metrics-server](https://github.com/kubernetes-sigs/metrics-server) must be running in the cluster. HPA fetches CPU/memory metrics from it — without it, HPA can't make scaling decisions.

```bash
# Verify metrics-server is running
kubectl get pods -n kube-system | grep metrics-server

# Confirm it's returning data
kubectl top nodes
```

## How HPA Works

```
┌────────────────┐
│ metrics-server │
└───────┬────────┘
        │ metrics
        ▼
┌───────────────────────────┐
│  HorizontalPodAutoscaler  │
└───────────┬───────────────┘
            │ scale
            ▼
┌───────────────────────────┐
│       Deployment          │
│                           │
│  ┌───────┐ ┌───────┐      │
│  │ Pod 1 │ │ Pod 2 │ ...  │
│  └───────┘ └───────┘      │
└───────────────────────────┘
```

The HPA controller checks metrics every 15 seconds, compares current usage against the target, and adjusts the replica count on the Deployment.

## Key Concepts

The HPA controller runs every 15 seconds (configurable via `--horizontal-pod-autoscaler-sync-period`).

Scaling formula:

```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / desiredMetricValue))
```

Default behavior:

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300    # 5 min — wait before removing pods
    policies:
    - type: Percent
      value: 100
      periodSeconds: 15
  scaleUp:
    stabilizationWindowSeconds: 0      # immediate
    policies:
    - type: Percent
      value: 100
      periodSeconds: 15
    - type: Pods
      value: 4
      periodSeconds: 15
    selectPolicy: Max
```

Limit scale-down rate to 10% per minute:

```yaml
behavior:
  scaleDown:
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60
```

> This limits how fast HPA removes pods. With 20 replicas, HPA can remove at most 2 pods per minute (10% of 20). Next minute at 18 replicas, it removes 1 (10% of 18, rounded down). Gradual ramp-down instead of dropping all at once — prevents sudden capacity loss when traffic is spiky.

Completely disable scale-down:

```yaml
behavior:
  scaleDown:
    selectPolicy: Disabled
```

> This tells the HPA to never remove pods automatically — replicas only go up, never down. Useful in production where you'd rather pay for idle pods than risk dropping capacity during unpredictable traffic. To manually scale down: `kubectl scale deployment <name> --replicas=<N>`

Tolerance (Kubernetes v1.35+):

```yaml
behavior:
  scaleUp:
    tolerance: 0.05
```

> The default cluster-wide tolerance is 10% (0.1). This means HPA ignores metric fluctuations within 10% of the target — no scaling happens for small variations. For example, with a CPU target of 60%, HPA won't scale up unless usage exceeds 66% (60% × 1.1). Setting `tolerance: 0.05` (5%) makes HPA more sensitive: it would scale up at 63% instead.
>
> The default tolerance is a hardcoded value in the kube-controller-manager, set via the `--horizontal-pod-autoscaler-tolerance` flag. It's not visible in `kubectl describe hpa`. To check the current value:
>
> ```bash
> kubectl get pods -n kube-system -l component=kube-controller-manager -o yaml | grep horizontal-pod-autoscaler-tolerance
> ```
>
> If nothing is returned, the default `0.1` is in effect. Before v1.35, this value is cluster-wide and cannot be configured per-HPA — only by modifying the controller-manager startup flags (not possible on managed clusters like EKS).

Asymmetric tolerance — react fast on scale-up, be conservative on scale-down:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      tolerance: 0.01    # 1% — scale up aggressively (triggers at 70.7%)
    scaleDown:
      tolerance: 0.05    # 5% — scale down conservatively (triggers at 66.5%)
```

> With a 70% CPU target:
> - `scaleUp tolerance: 0.01` → HPA adds pods as soon as usage exceeds 70.7% (70% × 1.01). Very responsive to traffic spikes.
> - `scaleDown tolerance: 0.05` → HPA removes pods only when usage drops below 66.5% (70% × 0.95). Avoids flapping during temporary dips.
>
> This asymmetry is the key insight: you almost always want to scale up faster than you scale down. Scaling up too slow loses requests; scaling down too fast causes repeated up/down cycles.

Hands-on test with a stress container:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stress-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stress-test
  template:
    metadata:
      labels:
        app: stress-test
    spec:
      containers:
      - name: stress
        image: polinux/stress
        command: ["stress"]
        args: ["--cpu", "1", "--timeout", "600"]
        resources:
          requests:
            cpu: 100m
          limits:
            cpu: 200m
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: stress-test
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: stress-test
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 10
  behavior:
    scaleUp:
      tolerance: 0.02    # 2% — triggers at just above 10.2% of 100m
    scaleDown:
      tolerance: 0.1     # 10% — conservative scale-down
```

> The stress command generates ~100m CPU load. With requests at 100m, that's 100% utilization — well above the 10% target. With a 2% scaleUp tolerance the HPA triggers almost immediately. Watch it scale: `kubectl get hpa stress-test -w`

## Multi-Metric HPA (CPU + Memory)

HPA supports scaling on both CPU and memory simultaneously using `autoscaling/v2`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-frontend
  namespace: scaling-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

When multiple metrics are specified, HPA evaluates each one independently and picks the highest replica count. If CPU says "scale to 4" and memory says "scale to 6", you get 6 replicas.

> **Gotcha — memory-based scaling:** Many runtimes don't release memory after load drops (JVM, Python, Go GC). This means scale-up works fine, but scale-down may never trigger because memory usage stays high even when traffic is gone. CPU-based scaling is generally more predictable for scale-down.

> **Important:** Pods must have `resources.requests` set for both `cpu` and `memory` for utilization-based targets to work. Without requests, HPA can't calculate the percentage and shows `<unknown>`.

Beyond CPU and memory, HPA v2 also supports:

- **Custom metrics** (`type: Pods` or `type: Object`) — app-level metrics like requests per second, exposed via a custom metrics adapter (e.g., Prometheus Adapter).
- **External metrics** (`type: External`) — metrics from outside the cluster, like a message queue depth from a cloud provider.

## Test HPA with Load Generator

The official Kubernetes HPA walkthrough using php-apache:

```bash
# 1. Deploy test application
kubectl apply -f https://k8s.io/examples/application/php-apache.yaml

# 2. Create HPA (scale at 50% CPU, 1-10 replicas)
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

# 3. Verify HPA is created
kubectl get hpa

# 4. Generate load (in a separate terminal)
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never \
  -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

# 5. Watch HPA scale up
kubectl get hpa php-apache -w

# 6. Stop load generator (Ctrl+C) and watch scale down

# 7. Cleanup
kubectl delete deployment.apps/php-apache service/php-apache horizontalpodautoscaler.autoscaling/php-apache
```

## Useful Commands

```bash
# Watch HPA decisions in real time
kubectl get hpa -w

# Detailed HPA status — shows current/target metrics, conditions, and events
kubectl describe hpa <name>

# Check HPA events for scaling decisions and errors
kubectl events --for hpa/<name>

# View current replicas vs desired
kubectl get hpa -o custom-columns=NAME:.metadata.name,MIN:.spec.minReplicas,MAX:.spec.maxReplicas,CURRENT:.status.currentReplicas,DESIRED:.status.desiredReplicas

# Check metrics-server is returning pod metrics (required for HPA)
kubectl top pods -n <namespace>
```

> **Tip:** HPA events show exactly why it scaled (or didn't). Look for `SuccessfulRescale`, `FailedGetResourceMetric`, or `FailedComputeMetricsReplicas` events.

## KEDA — Event-Driven Alternative

[KEDA](https://keda.sh/) (Kubernetes Event-Driven Autoscaling) extends HPA with 60+ scalers for event sources like message queues, databases, and custom metrics — without needing a custom metrics adapter.

Key differences from HPA:

| | HPA | KEDA |
|---|---|---|
| **Scale to zero** | No (minimum 1 replica) | Yes |
| **Metric sources** | CPU, memory, custom/external metrics API | 60+ built-in scalers (Kafka, SQS, Redis, Prometheus, etc.) |
| **Setup** | Built-in, no extra install | Requires KEDA operator |
| **Under the hood** | Standalone controller | Creates and manages HPA objects for you |

KEDA is a good fit when you need to scale based on queue depth, schedule, or external events rather than just CPU/memory.

Source: [HPA Documentation](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
