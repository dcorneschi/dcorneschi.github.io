# HAProxy on EKS with NLB

## Overview

When you deploy HAProxy on an EKS cluster and expose it via a Kubernetes Service of type `LoadBalancer`, the AWS Load Balancer Controller automatically provisions a Network Load Balancer (NLB) and supporting infrastructure.

### Traffic Flow

```
Internet
  → NLB (Layer 4, TCP passthrough)
    → Target Group (pod IPs registered)
      → HAProxy Pods (Layer 7, routing rules)
        → Backend K8s Services (ClusterIP)
          → Application Pods
```

HAProxy acts as the L7 reverse proxy inside the cluster, while the NLB provides the external entry point at Layer 4.

### Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                             AWS VPC                                  │
│                                                                      │
│  ┌────────────────────── Public Subnets ──────────────────────────┐ │
│  │                                                                 │ │
│  │   ┌─────────────────────────────────────────────────────────┐   │ │
│  │   │            Network Load Balancer (NLB)                  │   │ │
│  │   │                  Layer 4 - TCP                          │   │ │
│  │   │                                                         │   │ │
│  │   │   Listener :80 (TCP)       Listener :443 (TCP)          │   │ │
│  │   │        │                        │                       │   │ │
│  │   │   Target Group :80         Target Group :443            │   │ │
│  │   │   (Pod IPs registered)     (Pod IPs registered)         │   │ │
│  │   └────────┬────────────────────────┬───────────────────────┘   │ │
│  └────────────┼────────────────────────┼──────────────────────────┘ │
│               │                        │                             │
│  ┌────────────┼────── Private Subnets ─┼─────────────────────────┐  │
│  │            │                        │                          │  │
│  │   ┌────────▼────────────────────────▼────────────────────┐    │  │
│  │   │            HAProxy Pods (Deployment, replicas: 2)     │    │  │
│  │   │                                                       │    │  │
│  │   │   HAProxy Pod 1             HAProxy Pod 2             │    │  │
│  │   │   :80 (HTTP)                :80 (HTTP)                │    │  │
│  │   │   :443 (HTTPS/TCP)          :443 (HTTPS/TCP)          │    │  │
│  │   └────────┬─────────────────────────┬────────────────────┘    │  │
│  │            │                         │                         │  │
│  │            ▼                         ▼                         │  │
│  │   ┌──────────────────────────────────────────────────────┐    │  │
│  │   │          Backend Services (ClusterIP)                 │    │  │
│  │   │                                                       │    │  │
│  │   │  app-a:8080     app-b:8080     app-c:3000             │    │  │
│  │   │  (ClusterIP)    (ClusterIP)    (ClusterIP)            │    │  │
│  │   │      │               │              │                 │    │  │
│  │   │      ▼               ▼              ▼                 │    │  │
│  │   │  App A Pods     App B Pods     App C Pods             │    │  │
│  │   └──────────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘

Port Summary:
  Client → NLB:          :80/TCP, :443/TCP
  NLB → HAProxy Pods:    :80/TCP, :443/TCP (via Target Groups, direct pod IP)
  HAProxy → Backends:    :8080, :3000, etc. (whatever your apps listen on)
```

## How HAProxy Routes to Backends

### Via ClusterIP Services (Default)

1. HAProxy resolves `app-a-svc.default.svc.cluster.local` via CoreDNS → returns ClusterIP (e.g., `10.100.12.5`)
2. Packet hits the ClusterIP virtual IP
3. kube-proxy (iptables/IPVS) DNATs to one of the actual pod IPs
4. With VPC CNI, pod-to-pod traffic stays within the VPC (no overlay)

### Via Headless Service (Direct Pod IPs)

For full control over load balancing, use HAProxy's DNS-based service discovery with a headless service (`ClusterIP: None`):

```
resolvers k8s
  nameserver dns 10.100.0.10:53
  resolve_retries 3
  timeout resolve 1s
  timeout retry   1s
  hold valid      10s

backend app_a
  balance roundrobin
  server-template srv 5 _http._tcp.app-a-svc.default.svc.cluster.local resolvers k8s init-addr none
```

This bypasses kube-proxy entirely — HAProxy resolves individual pod IPs and load balances across them directly.

## Prerequisites — AWS Load Balancer Controller

```sh
# Create IAM policy
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json

# Create IRSA service account
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

## Deploy HAProxy

### ConfigMap (HAProxy Configuration)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: haproxy-config
  namespace: haproxy
data:
  haproxy.cfg: |
    global
      log stdout format raw local0
      maxconn 4096

    defaults
      log     global
      mode    http
      option  httplog
      option  dontlognull
      timeout connect 5s
      timeout client  30s
      timeout server  30s
      retries 3

    frontend http_front
      bind *:80
      bind *:443 ssl crt /etc/ssl/certs/haproxy.pem

      acl host_app_a  hdr(host) -i app-a.example.com
      acl host_app_b  hdr(host) -i app-b.example.com
      acl path_api    path_beg /api

      use_backend app_a  if host_app_a
      use_backend app_b  if host_app_b
      use_backend app_c  if path_api
      default_backend app_a

    backend app_a
      balance roundrobin
      option httpchk GET /health
      server srv1 app-a-svc.default.svc.cluster.local:8080 check

    backend app_b
      balance roundrobin
      option httpchk GET /health
      server srv1 app-b-svc.default.svc.cluster.local:8080 check

    backend app_c
      balance roundrobin
      option httpchk GET /health
      server srv1 app-c-svc.monitoring.svc.cluster.local:3000 check

    listen stats
      bind *:8404
      stats enable
      stats uri /stats
      stats refresh 10s
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: haproxy
  namespace: haproxy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: haproxy
  template:
    metadata:
      labels:
        app: haproxy
    spec:
      containers:
        - name: haproxy
          image: haproxy:2.9
          ports:
            - containerPort: 80
            - containerPort: 443
            - containerPort: 8404
          volumeMounts:
            - name: config
              mountPath: /usr/local/etc/haproxy/haproxy.cfg
              subPath: haproxy.cfg
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /stats
              port: 8404
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /stats
              port: 8404
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: config
          configMap:
            name: haproxy-config
```

## Expose HAProxy via Service (Triggers NLB Creation)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: haproxy
  namespace: haproxy
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-target-group-attributes: preserve_client_ip.enabled=true
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: haproxy
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
```

## What AWS Creates Automatically

When the controller reconciles the Service, it provisions:

| AWS Resource | Purpose |
|---|---|
| Network Load Balancer | Layer 4 entry point, one per Service |
| Target Group(s) | One per port (80, 443). Registers pod IPs directly |
| Listeners | TCP listeners on ports 80 and 443 forwarding to target groups |
| Security Group rules | Allows inbound traffic from NLB to nodes/pods |
| Elastic IPs (optional) | If annotated with `aws-load-balancer-eip-allocations` |
| DNS name | NLB gets a name like `xxxx.elb.<region>.amazonaws.com` |

## Health Checks

The NLB performs TCP health checks against HAProxy pods by default. Customize:

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: HTTP
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: /stats
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "3"
```

## TLS Termination Options

### Option 1: TLS at HAProxy (End-to-End in Cluster)

NLB passes TCP traffic through. HAProxy terminates TLS:

```
Client --TLS--> NLB (TCP passthrough) --TLS--> HAProxy (terminates) --HTTP--> Backends
```

Mount certificates in HAProxy pods via Secrets.

### Option 2: TLS at NLB (ACM Certificate)

NLB terminates TLS using an ACM certificate. HAProxy receives plain HTTP:

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:<region>:<account>:certificate/<cert-id>
  service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
  service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
```

```
Client --TLS--> NLB (terminates TLS) --HTTP--> HAProxy --HTTP--> Backends
```

### Option 3: TLS at Both (Re-Encryption)

NLB terminates client TLS, then re-encrypts to HAProxy:

```
Client --TLS--> NLB (terminates) --TLS--> HAProxy (terminates) --HTTP--> Backends
```

Less common, adds latency.

## Target Type: ip vs instance

| Target Type | Behavior | Pros | Cons |
|---|---|---|---|
| `ip` (recommended) | NLB routes directly to pod IPs | Faster, fewer hops, no SNAT, preserves client IP | Requires VPC CNI |
| `instance` | NLB routes to NodePort, kube-proxy forwards | Works without VPC CNI | Extra hop, SNAT, loses client IP |

Always use `ip` target type on EKS with VPC CNI.

## Preserving Client IP

With `ip` target type:

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-target-group-attributes: preserve_client_ip.enabled=true
```

HAProxy sees the real client IP in the connection. Without this, it sees the NLB's IP.

Alternatively, use PROXY protocol:

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
```

Then in HAProxy config:

```
frontend http_front
  bind *:80 accept-proxy
```

## Useful Annotations Reference

| Annotation | Description |
|---|---|
| `aws-load-balancer-type: external` | Use AWS LB Controller (not in-tree) |
| `aws-load-balancer-nlb-target-type: ip` | Route to pod IPs directly |
| `aws-load-balancer-scheme: internet-facing` | Public NLB (vs `internal`) |
| `aws-load-balancer-subnets` | Specify subnets for NLB placement |
| `aws-load-balancer-eip-allocations` | Assign static Elastic IPs |
| `aws-load-balancer-cross-zone-load-balancing-enabled: "true"` | Enable cross-AZ balancing |
| `aws-load-balancer-proxy-protocol: "*"` | Enable PROXY protocol v2 |
| `aws-load-balancer-ssl-cert` | ACM certificate ARN for TLS termination |
| `aws-load-balancer-ssl-ports` | Which ports to terminate TLS on |
| `aws-load-balancer-target-group-attributes` | Target group settings (stickiness, deregistration delay, etc.) |
| `aws-load-balancer-healthcheck-protocol` | Health check protocol (TCP/HTTP/HTTPS) |
| `aws-load-balancer-healthcheck-path` | Health check path (for HTTP/HTTPS) |

## Static Elastic IPs

For a fixed entry point (useful for DNS, firewalls):

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-eip-allocations: eipalloc-aaa,eipalloc-bbb,eipalloc-ccc
  service.beta.kubernetes.io/aws-load-balancer-subnets: subnet-aaa,subnet-bbb,subnet-ccc
```

One EIP per AZ/subnet.

## Internal NLB

For private services (not internet-facing):

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-type: external
  service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
  service.beta.kubernetes.io/aws-load-balancer-scheme: internal
  service.beta.kubernetes.io/aws-load-balancer-subnets: subnet-private-a,subnet-private-b
```

## Troubleshooting

```sh
# Check controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Verify target group registration
aws elbv2 describe-target-health --target-group-arn <arn> --output table

# Check Service events
kubectl describe svc haproxy -n haproxy

# Verify NLB was created
aws elbv2 describe-load-balancers --query "LoadBalancers[?contains(LoadBalancerName, 'haproxy')]" --output table

# Check HAProxy stats
kubectl port-forward -n haproxy svc/haproxy 8404:8404
# Open http://localhost:8404/stats
```

### Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| NLB not created | Controller not running or missing permissions | Check controller logs, verify IRSA |
| Targets unhealthy | HAProxy not listening or health check misconfigured | Check pods are running, verify health check path |
| 503 from NLB | No healthy targets | Check target group health, HAProxy logs |
| Client IP is NLB IP | `preserve_client_ip` not enabled | Add annotation or use PROXY protocol |
| Cross-AZ traffic charges | Cross-zone LB disabled | Enable `cross-zone-load-balancing-enabled` |
| NLB in wrong subnets | Subnet auto-discovery picks wrong ones | Explicitly set `aws-load-balancer-subnets` |

## Gotchas

- **NLB creation takes 2-3 minutes** — don't assume it's broken immediately after applying the Service.
- **One NLB per Service** — if you expose multiple Services as LoadBalancer, you get multiple NLBs ($$$). Use one HAProxy Service and route everything through it.
- **NLB doesn't do path-based routing** — that's HAProxy's job. NLB only does TCP/UDP forwarding.
- **Deregistration delay** — when a HAProxy pod is terminated, it stays in the target group for 300s by default. Reduce with `deregistration_delay.timeout_seconds=30`.
- **Pod readiness gates** — the LB controller uses readiness gates to prevent traffic to unready pods. Ensure pods have proper readiness probes.
- **Annotation changes require Service recreation** — some annotations (like `scheme`) can't be changed after NLB creation. Delete and recreate the Service.
