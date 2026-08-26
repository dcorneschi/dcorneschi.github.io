# EKS Load Balancers: ALB vs NLB

How to expose services on EKS using Application Load Balancers (ALB) and Network Load Balancers (NLB) — covering both EKS Auto Mode (built-in) and Standard EKS (AWS Load Balancer Controller).

## Quick Comparison

| | ALB (Application Load Balancer) | NLB (Network Load Balancer) |
|---|---|---|
| **OSI Layer** | Layer 7 (HTTP/HTTPS) | Layer 4 (TCP/UDP/TLS) |
| **Triggered by** | `Ingress` resource | `Service type: LoadBalancer` |
| **Routing** | Path, host, header, query string | Port-based only |
| **Protocols** | HTTP, HTTPS, gRPC (via HTTP/2) | TCP, UDP, TLS |
| **WebSocket** | Supported | Supported (TCP) |
| **Static IP** | No (use Global Accelerator for fixed IPs) | Yes (one per AZ, or Elastic IPs) |
| **TLS termination** | At the ALB | At the NLB or passthrough to pods |
| **Target types** | IP (pod direct) or instance (NodePort) | IP (pod direct) or instance (NodePort) |
| **Health checks** | HTTP/HTTPS (path-based) | TCP, HTTP, HTTPS |
| **Cost** | Per hour + LCU (request-based) | Per hour + NLCU (connection/bandwidth-based) |
| **Latency** | Slightly higher (L7 processing) | Lower (L4, no content inspection) |
| **Multiple services** | Yes (path/host routing behind one ALB) | One service per LB (or multiple ports) |
| **Use case** | Web apps, APIs, microservices, gRPC | TCP services, gRPC (passthrough), low-latency, static IP, high throughput |

## When to Use ALB

- HTTP/HTTPS web applications
- Multiple services behind a single load balancer (path-based or host-based routing)
- Need WAF integration
- Need OIDC/Cognito authentication at the LB level
- gRPC services (via HTTP/2)
- Canary / weighted routing

## When to Use NLB

- TCP/UDP services (databases, message queues, custom protocols)
- Need static IP addresses or Elastic IPs
- Need to preserve source IP (client IP)
- Ultra-low latency requirements
- TLS passthrough (terminate at the pod, not the LB)
- High throughput / high connection count workloads
- Services that need PrivateLink (NLB is required for VPC endpoint services)

---

## EKS Auto Mode (Built-in — No Controller Install)

### ALB on EKS Auto Mode

EKS Auto Mode has a built-in load balancing capability — no AWS Load Balancer Controller installation needed.

**Required resources:** `IngressClassParams` + `IngressClass` + `Ingress`

```yaml
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
  name: my-app
  namespace: default
  annotations:
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
spec:
  ingressClassName: eks-auto-alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

**Key points:**
- Controller is `eks.amazonaws.com/alb` (not the standard `ingress.k8s.aws/alb`)
- `IngressClassParams` sets the ALB scheme (`internet-facing` or `internal`)
- Service must be `ClusterIP` when using `target-type: ip`

### NLB on EKS Auto Mode

Simply use `Service type: LoadBalancer`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: default
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

No `IngressClass` or `IngressClassParams` needed — the NLB is provisioned automatically.

---

## Standard EKS (AWS Load Balancer Controller)

On standard EKS clusters, you need the [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) installed (via Helm or as an EKS add-on).

### ALB on Standard EKS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: default
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/abc123
spec:
  ingressClassName: alb
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

### NLB on Standard EKS

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: default
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 443
      targetPort: 8443
      protocol: TCP
```

> **Note:** On standard EKS, if you don't set `aws-load-balancer-type: external`, Kubernetes may create a Classic Load Balancer (CLB) instead of an NLB.

---

## Common Annotations Reference

### ALB Annotations (Ingress)

| Annotation | Values | Description |
|-----------|--------|-------------|
| `alb.ingress.kubernetes.io/scheme` | `internet-facing`, `internal` | Public or private ALB |
| `alb.ingress.kubernetes.io/target-type` | `ip`, `instance` | Route to pod IPs or NodePorts |
| `alb.ingress.kubernetes.io/listen-ports` | JSON array | Ports the ALB listens on |
| `alb.ingress.kubernetes.io/certificate-arn` | ACM ARN | TLS certificate for HTTPS |
| `alb.ingress.kubernetes.io/ssl-redirect` | `443` | Redirect HTTP to HTTPS |
| `alb.ingress.kubernetes.io/healthcheck-path` | `/health` | Custom health check path |
| `alb.ingress.kubernetes.io/healthcheck-interval-seconds` | `15` | Health check interval |
| `alb.ingress.kubernetes.io/success-codes` | `200-299` | HTTP codes for healthy |
| `alb.ingress.kubernetes.io/group.name` | `my-group` | Share one ALB across multiple Ingresses |
| `alb.ingress.kubernetes.io/group.order` | `1` | Priority within the group |
| `alb.ingress.kubernetes.io/actions.*` | JSON | Advanced routing (weighted, fixed-response, redirect) |
| `alb.ingress.kubernetes.io/conditions.*` | JSON | Advanced matching (headers, query strings) |
| `alb.ingress.kubernetes.io/wafv2-acl-arn` | WAF ARN | Attach WAFv2 Web ACL |
| `alb.ingress.kubernetes.io/security-groups` | SG IDs | Custom security groups for the ALB |
| `alb.ingress.kubernetes.io/subnets` | Subnet IDs | Specific subnets (overrides auto-discovery) |

### NLB Annotations (Service)

| Annotation | Values | Description |
|-----------|--------|-------------|
| `service.beta.kubernetes.io/aws-load-balancer-scheme` | `internet-facing`, `internal` | Public or private NLB |
| `service.beta.kubernetes.io/aws-load-balancer-nlb-target-type` | `ip`, `instance` | Route to pod IPs or NodePorts |
| `service.beta.kubernetes.io/aws-load-balancer-type` | `external` | Forces NLB (required on standard EKS) |
| `service.beta.kubernetes.io/aws-load-balancer-subnets` | Subnet IDs/names | Specific subnets |
| `service.beta.kubernetes.io/aws-load-balancer-eip-allocations` | EIP alloc IDs | Assign Elastic IPs |
| `service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled` | `true` | Enable cross-AZ balancing |
| `service.beta.kubernetes.io/aws-load-balancer-proxy-protocol` | `*` | Enable PROXY protocol v2 |
| `service.beta.kubernetes.io/aws-load-balancer-ssl-cert` | ACM ARN | TLS termination at NLB |
| `service.beta.kubernetes.io/aws-load-balancer-ssl-ports` | `443` | Which ports use TLS |
| `service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy` | Policy name | TLS security policy |
| `service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol` | `TCP`, `HTTP` | Health check protocol |
| `service.beta.kubernetes.io/aws-load-balancer-healthcheck-path` | `/health` | HTTP health check path |
| `service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval` | `10` | Health check interval (seconds) |
| `service.beta.kubernetes.io/aws-load-balancer-target-group-attributes` | key=value pairs | TG attributes (stickiness, deregistration delay) |

---

## Examples

### ALB with HTTPS and HTTP-to-HTTPS redirect

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-https
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/abc123
spec:
  ingressClassName: alb
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

### ALB with multiple services (path-based routing)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-service
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 3000
```

### ALB shared across multiple Ingresses (IngressGroup)

```yaml
# Ingress 1 — api
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    alb.ingress.kubernetes.io/group.name: shared-alb
    alb.ingress.kubernetes.io/group.order: "1"
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
---
# Ingress 2 — frontend (same ALB)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  annotations:
    alb.ingress.kubernetes.io/group.name: shared-alb
    alb.ingress.kubernetes.io/group.order: "2"
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 3000
```

### NLB with TLS termination

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-tls
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456789012:certificate/abc123
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - name: https
      port: 443
      targetPort: 8080
      protocol: TCP
```

### NLB with Elastic IPs (static IPs)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-static-ip
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-eip-allocations: eipalloc-aaa,eipalloc-bbb,eipalloc-ccc
    service.beta.kubernetes.io/aws-load-balancer-subnets: subnet-aaa,subnet-bbb,subnet-ccc
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

### NLB with PROXY protocol (preserve client IP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-proxy-protocol
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

> Your application must support PROXY protocol v2 to parse the client IP from the PROXY header.

### Internal NLB (private, no internet access)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: internal-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internal
spec:
  type: LoadBalancer
  selector:
    app: internal-app
  ports:
    - port: 443
      targetPort: 8443
      protocol: TCP
```

---

## Subnet Tagging Requirements

For automatic subnet discovery, tag your subnets:

| Subnet Type | Required Tag | Value |
|------------|-------------|-------|
| Public (internet-facing LBs) | `kubernetes.io/role/elb` | `1` |
| Private (internal LBs) | `kubernetes.io/role/internal-elb` | `1` |

Both also need: `kubernetes.io/cluster/<cluster-name>` = `shared` or `owned`

```bash
# Verify public subnet tags
aws ec2 describe-subnets \
  --filters "Name=tag:kubernetes.io/role/elb,Values=1" \
  --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}' --output table

# Verify private subnet tags
aws ec2 describe-subnets \
  --filters "Name=tag:kubernetes.io/role/internal-elb,Values=1" \
  --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}' --output table
```

---

## Troubleshooting

```bash
# ALB: check Ingress events
kubectl describe ingress <name>

# NLB: check Service events
kubectl describe svc <name>

# Verify target group health (find TG ARN from AWS console or CLI)
aws elbv2 describe-target-health --target-group-arn <tg-arn>

# Check if pods are registered as targets
kubectl get endpoints <service-name>

# Find load balancers created by the controller
aws elbv2 describe-load-balancers --query 'LoadBalancers[].{Name:LoadBalancerName,DNS:DNSName,Scheme:Scheme,Type:Type}' --output table

# Check security groups on the LB
aws elbv2 describe-load-balancers --names <lb-name> --query 'LoadBalancers[0].SecurityGroups'
```

## Useful Links

- [AWS Load Balancer Controller Docs](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [ALB Ingress Annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/)
- [NLB Service Annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/service/annotations/)
- [EKS Auto Mode Load Balancing](https://docs.aws.amazon.com/eks/latest/userguide/auto-elb.html)
- [Subnet Tagging for EKS](https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html)
