# Deploy 2048 Game on EKS Auto Mode

A quick walkthrough to deploy the 2048 game on an EKS Auto Mode cluster — demonstrates scale-from-zero, automatic ALB provisioning via Ingress, topology spreading, and cleanup.

## Prerequisites

```bash
# Ensure kubeconfig is configured
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Verify cluster is ready
kubectl cluster-info
```

## Full Manifest

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: game-2048
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-2048
  namespace: game-2048
  labels:
    app.kubernetes.io/name: game-2048
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: game-2048
  template:
    metadata:
      labels:
        app.kubernetes.io/name: game-2048
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          minDomains: 2
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: game-2048
      containers:
        - name: game-2048
          image: public.ecr.aws/l6m2t8p7/docker-2048:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
          securityContext:
            allowPrivilegeEscalation: false
---
apiVersion: v1
kind: Service
metadata:
  name: game-2048
  namespace: game-2048
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: game-2048
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
---
apiVersion: eks.amazonaws.com/v1
kind: IngressClassParams
metadata:
  name: eks-auto-alb
spec:
  scheme: internet-facing
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: eks-auto-alb
spec:
  controller: eks.amazonaws.com/alb
  parameters:
    apiGroup: eks.amazonaws.com
    kind: IngressClassParams
    name: eks-auto-alb
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: game-2048
  namespace: game-2048
  annotations:
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: eks-auto-alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: game-2048
                port:
                  number: 80
```

### Key points

- **Service is `ClusterIP`** — the ALB routes traffic directly to pod IPs (`target-type: ip`), no NLB needed
- **`IngressClassParams`** with `scheme: internet-facing` — the EKS Auto Mode native way to configure ALB scheme
- **`IngressClass`** uses controller `eks.amazonaws.com/alb` — this is the Auto Mode built-in controller, not the standard AWS LBC
- **`topologySpreadConstraints`** — spreads pods across at least 2 AZs for high availability

## Deploy

```bash
# Apply the manifest
kubectl apply -f 2048-game.yaml

# Watch pods come up (triggers node provisioning from zero)
kubectl get pods -n game-2048 -w

# Watch nodes being provisioned
kubectl get nodes -w
```

## Verify

```bash
# Check deployment status
kubectl get deployment -n game-2048

# Get the ALB DNS (provisioned automatically — no controller install needed)
kubectl get ingress game-2048 -n game-2048 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Open the ALB URL in your browser to play the game. It may take 1-2 minutes for the ALB to become active.

## What This Demonstrates

| Feature | What Happens |
|---------|-------------|
| Scale from zero | Cluster had no nodes → pods go Pending → Karpenter provisions EC2 instances |
| Automatic ALB | `Ingress` with `ingressClassName: eks-auto-alb` triggers ALB creation — built into Auto Mode |
| Topology spreading | Pods spread across AZs for high availability |
| IP target mode | ALB routes directly to pod IPs — no NodePort or kube-proxy hop |
| Resource requests | Karpenter uses `requests` to pick the optimal instance type for bin-packing |
| Security context | `allowPrivilegeEscalation: false` — follows Auto Mode best practices |

## Scale Up / Down

```bash
# Scale to 10 replicas (may trigger additional nodes)
kubectl scale deployment game-2048 -n game-2048 --replicas=10

# Watch new nodes appear
kubectl get nodes -w

# Scale back down (idle nodes will be consolidated/terminated)
kubectl scale deployment game-2048 -n game-2048 --replicas=2
```

## Clean Up

```bash
kubectl delete namespace game-2048
```

This deletes the deployment, service, ingress, IngressClass, IngressClassParams, and triggers node scale-down (nodes terminate when empty after `consolidateAfter: 30s`).

> **Note:** The `IngressClass` and `IngressClassParams` are cluster-scoped resources. Delete them separately if not in the namespace:
> ```bash
> kubectl delete ingressclass eks-auto-alb
> kubectl delete ingressclassparams eks-auto-alb
> ```

## Troubleshooting

```bash
# 1. Are pods running?
kubectl get pods -n game-2048

# 2. Are pods healthy? (check events for image pull issues, resource problems)
kubectl describe pod -n game-2048 -l app.kubernetes.io/name=game-2048

# 3. Is the service routing correctly? (should show pod IPs as endpoints)
kubectl get endpoints game-2048 -n game-2048

# 4. Is the IngressClass created?
kubectl get ingressclass

# 5. Does the Ingress have an address? (empty = ALB not provisioned yet)
kubectl get ingress -n game-2048

# 6. Check Ingress events for errors (SG issues, subnet issues)
kubectl describe ingress game-2048 -n game-2048

# 7. Check all events in the namespace (most issues surface here)
kubectl get events -n game-2048 --sort-by='.lastTimestamp'

# 8. Are subnets tagged correctly for ALB discovery?
# Public subnets need: kubernetes.io/role/elb = 1
# Private subnets need: kubernetes.io/role/internal-elb = 1
aws ec2 describe-subnets --filters "Name=tag:kubernetes.io/role/elb,Values=1" \
  --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone}' --output table

# 9. Check node claims (is a node being provisioned?)
kubectl get nodeclaims -o wide

# 10. Pods not spreading across AZs — check topology
kubectl get pods -n game-2048 -o wide

# 11. Can you curl the ALB directly? (once address appears)
ALB=$(kubectl get ingress game-2048 -n game-2048 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -I http://$ALB
```

### Common Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Pods stuck `Pending` | No nodes provisioned yet | Wait 30-60s for Karpenter to launch a node; check `kubectl get nodeclaims` |
| Ingress has no address | ALB not created | Check `kubectl describe ingress` events — look for subnet tag or SG errors |
| `FailedDeployModel` / SG dependency | Stale security group from previous deploy | Wait 2-5 min for old LB to drain; or find/delete orphaned ENIs on the SG |
| ALB returns 502/503 | Targets not yet registered or unhealthy | Wait for target registration; check pod readiness and endpoint list |
| Pods not spreading across AZs | Not enough AZs available or node capacity | Check `topologySpreadConstraints` and available subnets |
| `IngressClass not found` | IngressClass resource not applied | Apply the `IngressClass` and `IngressClassParams` from the manifest |

## ALB vs NLB on EKS Auto Mode

Both are provisioned automatically — no controller installation needed.

| | ALB (Application Load Balancer) | NLB (Network Load Balancer) |
|---|---|---|
| **Triggered by** | `Ingress` resource | `Service type: LoadBalancer` |
| **Layer** | Layer 7 (HTTP/HTTPS) | Layer 4 (TCP/UDP) |
| **Routing** | Path-based, host-based, header-based | Port-based only |
| **Target type** | IP (pods directly) or instance | IP or instance |
| **TLS termination** | At the ALB | At the NLB or passthrough |
| **Use case** | Web apps, APIs, multiple services behind one LB | gRPC, TCP services, low-latency, static IP needed |
| **Controller** | `eks.amazonaws.com/alb` (built-in) | Built-in (no controller needed) |

### ALB approach (used in this article)

Requires `IngressClassParams` + `IngressClass` + `Ingress` — see the full manifest above.

### NLB approach (alternative)

Simply use a `Service type: LoadBalancer` — no Ingress, IngressClass, or IngressClassParams needed:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: game-2048
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-2048
  namespace: game-2048
  labels:
    app.kubernetes.io/name: game-2048
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: game-2048
  template:
    metadata:
      labels:
        app.kubernetes.io/name: game-2048
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          minDomains: 2
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: game-2048
      containers:
        - name: game-2048
          image: public.ecr.aws/l6m2t8p7/docker-2048:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
          securityContext:
            allowPrivilegeEscalation: false
---
apiVersion: v1
kind: Service
metadata:
  name: game-2048
  namespace: game-2048
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: game-2048
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

```bash
# Deploy
kubectl apply -f 2048-game-nlb.yaml

# Get the NLB DNS
kubectl get svc game-2048 -n game-2048 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

## References

- [Deploy a Sample Load Balancer Workload to EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/auto-elb.html)
