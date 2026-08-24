# HAProxy for Kubernetes on DigitalOcean

Setting up HAProxy Ingress Controller on DigitalOcean Kubernetes (DOKS) — Helm installation, Load Balancer integration, TLS, annotations, and troubleshooting.

## Architecture

```
Internet
  → DigitalOcean Load Balancer (Layer 4, TCP)
    → HAProxy Pods (Layer 7, routing rules)
      → Backend Services (ClusterIP)
        → Application Pods
```

DigitalOcean automatically provisions a cloud Load Balancer when you create a Kubernetes Service of type `LoadBalancer`. HAProxy handles L7 routing inside the cluster.

## Do I Need a Load Balancer?

You don't strictly need a DigitalOcean Load Balancer. Here are your options:

### Without Load Balancer (Cheaper)

**NodePort** — expose on a high port (30000-32767):

```yaml
controller:
  service:
    type: NodePort
    nodePorts:
      http: 30080
      https: 30443
```

Access via `<any-node-ip>:30080`. Point DNS to node IPs.

**HostNetwork** — HAProxy binds directly to node ports 80/443:

```yaml
controller:
  daemonset:
    useHostNetwork: true
    useHostPort: true
```

Access via node IPs on standard ports. Requires HAProxy on specific nodes.

**Floating IP** — assign a DigitalOcean Reserved IP to one node and point DNS there.

### Decision Factors

| Scenario | Recommendation | Cost |
|----------|---------------|------|
| Single-node cluster | NodePort or HostNetwork | $0 |
| Dev/test, budget-conscious | NodePort + Reserved IP | $4/mo |
| Multi-node, high availability | Load Balancer | $12/mo |
| Production | Load Balancer | $12/mo |

### With Load Balancer (Recommended for Production)

- Automatic failover between nodes
- Single static IP
- Built-in health checks
- DDoS protection at L4

## Installation

### Prerequisites

```bash
# Authenticate with doctl
doctl auth init

# Save kubeconfig for your cluster
doctl kubernetes cluster kubeconfig save <cluster-name>

# Verify connectivity
kubectl get nodes
```

### Install with Helm

```bash
helm repo add haproxytech https://haproxytech.github.io/helm-charts
helm repo update

helm install haproxy-ingress haproxytech/kubernetes-ingress \
  --namespace haproxy-ingress \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer \
  --set controller.ingressClass=haproxy
```

### values.yaml for DigitalOcean

```yaml
controller:
  replicaCount: 2

  ingressClass: haproxy
  ingressClassResource:
    enabled: true
    default: true
    name: haproxy

  service:
    type: LoadBalancer
    annotations:
      # DigitalOcean Load Balancer annotations
      service.beta.kubernetes.io/do-loadbalancer-name: "haproxy-lb"
      service.beta.kubernetes.io/do-loadbalancer-protocol: "tcp"
      service.beta.kubernetes.io/do-loadbalancer-size-slug: "lb-small"
      service.beta.kubernetes.io/do-loadbalancer-algorithm: "round_robin"
      service.beta.kubernetes.io/do-loadbalancer-healthcheck-protocol: "tcp"
      service.beta.kubernetes.io/do-loadbalancer-healthcheck-port: "80"

  resources:
    requests:
      cpu: 200m
      memory: 128Mi
    limits:
      memory: 256Mi

  config:
    timeout-connect: "5s"
    timeout-client: "60s"
    timeout-server: "60s"
    ssl-redirect: "true"

defaultBackend:
  enabled: true
```

### Install with values file

```bash
helm install haproxy-ingress haproxytech/kubernetes-ingress \
  --namespace haproxy-ingress \
  --create-namespace \
  -f values.yaml
```

## DigitalOcean Load Balancer Annotations

| Annotation | Description | Values |
|-----------|-------------|--------|
| `do-loadbalancer-name` | Custom name for the LB | Any string |
| `do-loadbalancer-protocol` | Protocol for LB listener | `tcp`, `http`, `https`, `http2` |
| `do-loadbalancer-size-slug` | LB size | `lb-small`, `lb-medium`, `lb-large` |
| `do-loadbalancer-algorithm` | LB algorithm | `round_robin`, `least_connections` |
| `do-loadbalancer-tls-ports` | Ports for TLS termination at LB | `443` |
| `do-loadbalancer-certificate-id` | DO-managed certificate for TLS | Certificate UUID |
| `do-loadbalancer-redirect-http-to-https` | Redirect HTTP to HTTPS | `true`, `false` |
| `do-loadbalancer-enable-proxy-protocol` | Enable PROXY protocol | `true`, `false` |
| `do-loadbalancer-healthcheck-protocol` | Health check protocol | `tcp`, `http` |
| `do-loadbalancer-healthcheck-port` | Health check target port | Port number |
| `do-loadbalancer-healthcheck-path` | Health check path (HTTP) | `/healthz` |
| `do-loadbalancer-disable-lets-encrypt-dns-records` | Don't auto-create DNS for cert | `true`, `false` |
| `do-loadbalancer-sticky-sessions-type` | Session affinity | `none`, `cookies` |
| `do-loadbalancer-sticky-sessions-cookie-name` | Cookie name for sticky sessions | Any string |
| `do-loadbalancer-sticky-sessions-cookie-ttl` | Cookie TTL in seconds | `300` |

## TLS Termination

### Option 1: TLS at DigitalOcean Load Balancer

Let the DO LB terminate TLS using a managed certificate. HAProxy receives plain HTTP:

```yaml
controller:
  service:
    annotations:
      service.beta.kubernetes.io/do-loadbalancer-protocol: "http"
      service.beta.kubernetes.io/do-loadbalancer-tls-ports: "443"
      service.beta.kubernetes.io/do-loadbalancer-certificate-id: "<certificate-uuid>"
      service.beta.kubernetes.io/do-loadbalancer-redirect-http-to-https: "true"
```

Get certificate ID:

```bash
doctl compute certificate list
```

### Option 2: TLS at HAProxy (TCP Passthrough)

DO LB passes TCP traffic through. HAProxy terminates TLS inside the cluster:

```yaml
controller:
  service:
    annotations:
      service.beta.kubernetes.io/do-loadbalancer-protocol: "tcp"
    # No certificate annotations — LB does TCP only
```

Create TLS secret for HAProxy:

```bash
kubectl create secret tls myapp-tls \
  --cert=fullchain.pem \
  --key=privkey.pem \
  -n default
```

Ingress resource:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    haproxy.org/ssl-redirect: "true"
spec:
  ingressClassName: haproxy
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 8080
```

### Option 3: TLS with cert-manager (Let's Encrypt)

```bash
# Install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

Create a ClusterIssuer:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: haproxy
```

Ingress with automatic certificate:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    haproxy.org/ssl-redirect: "true"
spec:
  ingressClassName: haproxy
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-auto
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 8080
```

## Proxy Protocol (Preserve Client IP)

By default, HAProxy sees the DO Load Balancer's IP instead of the client's IP. Enable PROXY protocol to preserve it:

```yaml
controller:
  service:
    annotations:
      service.beta.kubernetes.io/do-loadbalancer-enable-proxy-protocol: "true"
  config:
    proxy-protocol: "true"
```

HAProxy then reads the real client IP from the PROXY protocol header.

## DNS Setup

Point your domain to the Load Balancer IP:

```bash
# Get the LB external IP
kubectl get svc -n haproxy-ingress -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}'

# Create DNS A record via doctl
doctl compute domain records create example.com \
  --record-type A \
  --record-name myapp \
  --record-data <lb-ip> \
  --record-ttl 300
```

Or use a wildcard for all subdomains:

```bash
doctl compute domain records create example.com \
  --record-type A \
  --record-name "*.apps" \
  --record-data <lb-ip> \
  --record-ttl 300
```

## Example: Deploy an Application

### Deploy a sample app

```bash
kubectl create deployment web --image=nginx --replicas=3
kubectl expose deployment web --port=80 --target-port=80
```

### Create Ingress

```bash
kubectl create ingress web-ingress --class=haproxy --rule="web.example.com/*=web:80"
```

Or as YAML:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    haproxy.org/timeout-server: "30s"
spec:
  ingressClassName: haproxy
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

### Test

```bash
# Get LB IP
LB_IP=$(kubectl get svc -n haproxy-ingress -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')

# Test with Host header
curl -H "Host: web.example.com" http://$LB_IP/
```

## Multiple Services Behind One Load Balancer

Route different domains or paths to different services — all through a single DO Load Balancer:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-app
spec:
  ingressClassName: haproxy
  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 8080
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2
            port:
              number: 3000
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 8080
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 8080
```

This saves cost — one $12/mo Load Balancer handles all your domains.

## Common Annotations for Ingress Resources

```yaml
annotations:
  # Timeouts
  haproxy.org/timeout-server: "60s"
  haproxy.org/timeout-client: "60s"

  # Rate limiting
  haproxy.org/rate-limit-requests: "100"
  haproxy.org/rate-limit-period: "1s"

  # Load balancing
  haproxy.org/load-balance: "leastconn"

  # CORS
  haproxy.org/cors-enable: "true"
  haproxy.org/cors-allow-origin: "*"

  # Path rewrite
  haproxy.org/path-rewrite: "/api/(.*) /\\1"

  # Whitelist
  haproxy.org/whitelist: "10.0.0.0/8"

  # SSL redirect
  haproxy.org/ssl-redirect: "true"
```

## Monitoring

### HAProxy Stats

```bash
# Port-forward stats page
kubectl port-forward -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress 1024:1024

# Open http://localhost:1024/stats
```

### Prometheus Metrics

Enable in values.yaml:

```yaml
controller:
  config:
    prometheus: "true"
  serviceMonitor:
    enabled: true    # If Prometheus Operator is installed
```

```bash
# Scrape metrics
kubectl port-forward -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress 9101:9101
curl http://localhost:9101/metrics | grep haproxy
```

## Troubleshooting

### Load Balancer Not Created

```bash
# Check Service status
kubectl get svc -n haproxy-ingress

# Check events
kubectl describe svc -n haproxy-ingress haproxy-ingress-kubernetes-ingress

# Verify DOKS cluster can create LBs
doctl compute load-balancer list
```

Common issues:
- Account LB limit reached (default: 10)
- Invalid annotation values
- Cluster RBAC issue with cloud controller

### 503 Errors

```bash
# Check HAProxy pods are running
kubectl get pods -n haproxy-ingress

# Check endpoints exist for the backend service
kubectl get endpoints <service-name>

# Check Ingress is configured correctly
kubectl describe ingress <ingress-name>

# Check HAProxy logs
kubectl logs -n haproxy-ingress -l app.kubernetes.io/name=kubernetes-ingress --tail=50
```

### Client IP Shows Load Balancer IP

Enable PROXY protocol:

```yaml
annotations:
  service.beta.kubernetes.io/do-loadbalancer-enable-proxy-protocol: "true"
```

And in HAProxy config:

```yaml
controller:
  config:
    proxy-protocol: "true"
```

### Certificate Issues

```bash
# Check cert-manager certificates
kubectl get certificates -A
kubectl describe certificate <cert-name>

# Check cert-manager logs
kubectl logs -n cert-manager deploy/cert-manager --tail=50

# Verify HTTP-01 challenge is accessible
curl http://<domain>/.well-known/acme-challenge/test
```

### Health Check Failures

```bash
# Check DO LB health
doctl compute load-balancer list --format ID,Name,Status,IP

# Check what the LB is health-checking
doctl compute load-balancer get <lb-id> --format HealthCheck

# Verify HAProxy responds on health check port
kubectl exec -n haproxy-ingress <pod> -- curl -s localhost:80/healthz
```

## Upgrade

```bash
# Check current version
helm list -n haproxy-ingress

# Upgrade
helm repo update
helm upgrade haproxy-ingress haproxytech/kubernetes-ingress \
  --namespace haproxy-ingress \
  -f values.yaml

# Verify rollout
kubectl rollout status deployment -n haproxy-ingress
```

## Uninstall

```bash
# Remove HAProxy
helm uninstall haproxy-ingress -n haproxy-ingress

# Remove namespace (also removes the LB)
kubectl delete namespace haproxy-ingress

# Verify LB is deleted
doctl compute load-balancer list
```

## Quick Reference

```bash
# Install
helm install haproxy-ingress haproxytech/kubernetes-ingress -n haproxy-ingress --create-namespace -f values.yaml

# Get LB IP
kubectl get svc -n haproxy-ingress -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}'

# Create Ingress (quick)
kubectl create ingress myapp --class=haproxy --rule="myapp.example.com/*=myapp:8080"

# Logs
kubectl logs -n haproxy-ingress -l app.kubernetes.io/name=kubernetes-ingress --tail=100

# Stats page
kubectl port-forward -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress 1024:1024

# HAProxy config
kubectl exec -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress -- cat /etc/haproxy/haproxy.cfg

# Upgrade
helm upgrade haproxy-ingress haproxytech/kubernetes-ingress -n haproxy-ingress -f values.yaml

# Uninstall
helm uninstall haproxy-ingress -n haproxy-ingress
```
