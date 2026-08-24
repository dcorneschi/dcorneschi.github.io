# Kubernetes Deployment Strategies

An overview of deployment strategies for releasing new versions of applications in Kubernetes — from simple recreate to advanced traffic-splitting approaches.

## Strategy Comparison

| Strategy | Downtime | Rollback Speed | Resource Cost | Risk | Complexity |
|----------|----------|----------------|---------------|------|------------|
| Recreate | Yes | Slow (redeploy) | Low (1x) | High | Low |
| Rolling Update | No | Fast (automatic) | Low (1x + surge) | Low-Medium | Low |
| Blue-Green | No | Instant (switch) | High (2x) | Low | Medium |
| Canary | No | Fast (shift traffic) | Medium (1x + small) | Low | Medium-High |
| Shadow | No | N/A (no user impact) | High (2x) | Very Low | High |
| A/B Testing | No | Fast | Medium | Low | High |

## Recreate

Kill all V1 pods, then start V2 pods. Simple but causes downtime.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: Recreate
  template:
    spec:
      containers:
      - name: app
        image: my-app:v2
```

**How it works:**
1. All existing pods (V1) are terminated
2. Once all are gone, new pods (V2) are created
3. Users experience downtime between termination and readiness

**When to use:**
- Database migrations that require the old version to be fully stopped
- Applications that can't run two versions simultaneously (shared state conflicts)
- Development/staging environments where downtime is acceptable

## Rolling Update

Gradually replace V1 pods with V2 pods. The Kubernetes default strategy.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1            # Max pods above desired count during update
      maxUnavailable: 1      # Max pods that can be unavailable during update
  template:
    spec:
      containers:
      - name: app
        image: my-app:v2
```

**How it works:**
1. A new V2 pod is created (respecting `maxSurge`)
2. Once V2 pod is Ready, an old V1 pod is terminated (respecting `maxUnavailable`)
3. Repeats until all pods are V2
4. Both V1 and V2 serve traffic simultaneously during the rollout

**Key parameters:**

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `maxSurge` | 25% | How many extra pods can exist during update |
| `maxUnavailable` | 25% | How many pods can be down during update |

```bash
# Trigger a rolling update
kubectl set image deployment/my-app app=my-app:v2

# Watch the rollout
kubectl rollout status deployment/my-app

# Rollback if something goes wrong
kubectl rollout undo deployment/my-app

# Rollback to a specific revision
kubectl rollout undo deployment/my-app --to-revision=2

# View rollout history
kubectl rollout history deployment/my-app
```

**When to use:**
- Most stateless applications
- When zero-downtime is required with minimal complexity
- Default choice for production deployments

## Blue-Green

Run two identical environments (Blue = current, Green = new). Switch all traffic at once.

```yaml
# Blue deployment (current - V1)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
  labels:
    app: my-app
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
      - name: app
        image: my-app:v1
---
# Green deployment (new - V2)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
  labels:
    app: my-app
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
      - name: app
        image: my-app:v2
---
# Service pointing to blue (switch selector to green to cutover)
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
    version: blue          # Change to "green" to switch
  ports:
  - port: 80
    targetPort: 8080
```

**How it works:**
1. Deploy V2 alongside V1 (both running, only V1 receives traffic)
2. Test V2 internally
3. Switch the Service selector from `blue` to `green`
4. All traffic instantly moves to V2
5. Keep V1 running for quick rollback (switch selector back)

```bash
# Switch traffic to green
kubectl patch service my-app -p '{"spec":{"selector":{"version":"green"}}}'

# Rollback to blue
kubectl patch service my-app -p '{"spec":{"selector":{"version":"blue"}}}'
```

**When to use:**
- When you need instant rollback capability
- When you want to test the new version in production before exposing it to users
- When you can afford double the resources

## Canary

Route a small percentage of traffic to V2 while V1 handles the rest. Gradually increase if V2 is healthy.

### Using Replica Count (Simple)

```yaml
# V1 — 3 replicas (75% of traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-stable
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      track: stable
  template:
    metadata:
      labels:
        app: my-app
        track: stable
    spec:
      containers:
      - name: app
        image: my-app:v1
---
# V2 — 1 replica (25% of traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      track: canary
  template:
    metadata:
      labels:
        app: my-app
        track: canary
    spec:
      containers:
      - name: app
        image: my-app:v2
---
# Service selects both (based on shared "app" label)
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app            # Matches both stable and canary pods
  ports:
  - port: 80
    targetPort: 8080
```

Traffic split is proportional to pod count: 3 stable + 1 canary = 75%/25%.

### Using Gateway API (Precise Weights)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-canary
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - backendRefs:
    - name: my-app-stable
      port: 80
      weight: 90
    - name: my-app-canary
      port: 80
      weight: 10
```

**How it works:**
1. Deploy V2 with minimal replicas
2. Route a small percentage of real traffic to V2
3. Monitor error rates, latency, and logs
4. If healthy, gradually increase V2 traffic (scale up canary, scale down stable)
5. If unhealthy, route all traffic back to V1 and delete canary

**When to use:**
- High-risk deployments where you want to validate with real traffic
- When you need fine-grained control over rollout speed
- Combined with observability to auto-promote or auto-rollback

## Shadow (Dark Launch)

Send a copy of production traffic to V2 without affecting users. V2's responses are discarded.

```
Client → LB → V1 (responses sent to client)
              └──→ V2 (responses discarded, only for testing)
```

**How it works:**
1. Deploy V2 alongside V1
2. Mirror/duplicate incoming traffic to V2
3. V1 serves all real responses to users
4. V2 processes the same requests but its responses are thrown away
5. Monitor V2 for errors, latency, and correctness

**Implementation:** Requires a service mesh (Istio, Linkerd) or traffic mirroring at the ingress level. Not natively supported by Kubernetes alone.

```yaml
# Istio VirtualService example
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
  - my-app
  http:
  - route:
    - destination:
        host: my-app-v1
    mirror:
      host: my-app-v2
    mirrorPercentage:
      value: 100
```

**When to use:**
- Testing performance under real production load
- Validating V2 handles real data correctly before any user sees it
- Load testing without synthetic traffic

**Caution:** Shadow traffic causes side effects if V2 writes to databases or sends notifications. Only use for read-heavy or idempotent operations, or isolate V2's write targets.

## A/B Testing

Route traffic based on user attributes (device, location, headers, cookies) rather than random percentages.

```
Mobile users → V2 (new mobile experience)
Desktop users → V1 (existing experience)
```

**How it works:**
1. Deploy V1 and V2 simultaneously
2. Route traffic based on request attributes (User-Agent, headers, cookies, geolocation)
3. Measure business metrics (conversion rate, engagement) per variant
4. Promote the winning variant

**Implementation:** Requires header/cookie-based routing — use Gateway API, Istio, or an ingress controller with advanced routing:

```yaml
# Gateway API HTTPRoute — route by header
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ab-test
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - headers:
      - name: x-user-group
        value: beta
    backendRefs:
    - name: my-app-v2
      port: 80
  - backendRefs:
    - name: my-app-v1
      port: 80
```

**When to use:**
- Testing UX changes with specific user segments
- Feature flags at the infrastructure level
- When the decision to rollout depends on business metrics, not just technical health

## Tools for Advanced Strategies

| Tool | Strategies Supported | How It Works |
|------|---------------------|--------------|
| **Argo Rollouts** | Canary, Blue-Green, Analysis | CRD that replaces Deployment with progressive delivery |
| **Flagger** | Canary, Blue-Green, A/B | Works with Istio/Linkerd/Gateway API, automated promotion |
| **Istio** | Canary, Shadow, A/B | Service mesh with VirtualService traffic splitting |
| **Gateway API** | Canary, A/B | Native weighted routing via HTTPRoute |

## Choosing a Strategy

```
Is downtime acceptable?
  └─ Yes → Recreate
  └─ No →
      Do you need instant rollback?
        └─ Yes → Blue-Green
        └─ No →
            Do you need to test with real traffic first?
              └─ Yes, without user impact → Shadow
              └─ Yes, with a small % of users → Canary
              └─ Yes, based on user attributes → A/B Testing
              └─ No, just roll it out gradually → Rolling Update
```
