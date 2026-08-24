# HAProxy Ingress Setup on EKS

Installing and configuring the HAProxy Ingress Controller on EKS — Helm charts, IngressClass, annotations, TLS, rate limiting, and operational one-liners.

## Installation with Helm

### Add the Helm Repository

```bash
helm repo add haproxytech https://haproxytech.github.io/helm-charts
helm repo update
```

### Install HAProxy Ingress Controller

```bash
helm install haproxy-ingress haproxytech/kubernetes-ingress \
  --namespace haproxy-ingress \
  --create-namespace \
  --set controller.replicaCount=3 \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"=internet-facing \
  --set controller.ingressClass=haproxy
```

### Full values.yaml Example

```yaml
controller:
  replicaCount: 3

  ingressClass: haproxy
  ingressClassResource:
    default: false
    enabled: true
    name: haproxy

  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
      service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
      service.beta.kubernetes.io/aws-load-balancer-target-type: ip

  resources:
    requests:
      cpu: 500m
      memory: 256Mi
    limits:
      memory: 512Mi

  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70

  config:
    timeout-connect: "5s"
    timeout-client: "60s"
    timeout-server: "60s"
    timeout-http-request: "10s"
    timeout-http-keep-alive: "60s"
    maxconn: "2000"
    nbthread: "4"
    ssl-redirect: "true"
    log-format: "%ci:%cp [%tr] %ft %b/%s %TR/%Tw/%Tc/%Tr/%Ta %ST %B %CC %CS %tsc %ac/%fc/%bc/%sc/%rc %sq/%bq %hr %hs %{+Q}r"

  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app.kubernetes.io/name
          operator: In
          values:
          - kubernetes-ingress
      topologyKey: kubernetes.io/hostname

  tolerations: []
  nodeSelector: {}

defaultBackend:
  enabled: true
  replicaCount: 2
```

### Install with values.yaml

```bash
helm install haproxy-ingress haproxytech/kubernetes-ingress \
  --namespace haproxy-ingress \
  --create-namespace \
  -f values.yaml
```

### Upgrade

```bash
helm upgrade haproxy-ingress haproxytech/kubernetes-ingress \
  --namespace haproxy-ingress \
  -f values.yaml
```

## IngressClass

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: haproxy
  annotations:
    ingressclass.kubernetes.io/is-default-class: "false"
spec:
  controller: haproxy.org/ingress-controller
```

Set `is-default-class: "true"` if HAProxy should handle all Ingress resources that don't specify an IngressClass.

## Basic Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    haproxy.org/timeout-server: "60s"
spec:
  ingressClassName: haproxy
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

## TLS Termination

### With a Kubernetes Secret

```bash
# Create TLS secret
kubectl create secret tls myapp-tls \
  --cert=fullchain.pem \
  --key=privkey.pem \
  -n default
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    haproxy.org/ssl-redirect: "true"
    haproxy.org/ssl-redirect-code: "301"
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

### With cert-manager (Automatic Certificates)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    haproxy.org/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
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

### TLS Passthrough (No Termination)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    haproxy.org/ssl-passthrough: "true"
spec:
  ingressClassName: haproxy
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
              number: 443
```

## Common Annotations

### Timeouts

```yaml
annotations:
  haproxy.org/timeout-connect: "5s"
  haproxy.org/timeout-server: "120s"
  haproxy.org/timeout-client: "120s"
  haproxy.org/timeout-http-request: "10s"
  haproxy.org/timeout-http-keep-alive: "60s"
  haproxy.org/timeout-queue: "30s"
  haproxy.org/timeout-tunnel: "3600s"     # WebSocket/long-lived connections
```

### Rate Limiting

```yaml
annotations:
  haproxy.org/rate-limit-requests: "100"          # Requests per period
  haproxy.org/rate-limit-period: "1s"             # Time period
  haproxy.org/rate-limit-status-code: "429"       # Response code when limited
```

### Load Balancing

```yaml
annotations:
  haproxy.org/load-balance: "roundrobin"          # roundrobin, leastconn, source
  haproxy.org/check: "true"                       # Enable health checks
  haproxy.org/check-interval: "5s"
  haproxy.org/check-http: "/health"               # HTTP health check path
```

### Request/Response Modification

```yaml
annotations:
  haproxy.org/request-set-header: "X-Forwarded-Proto https"
  haproxy.org/response-set-header: "Strict-Transport-Security max-age=31536000"
  haproxy.org/path-rewrite: "/api/(.*) /\\1"      # Strip /api prefix
  haproxy.org/server-proto: "h2"                  # Use HTTP/2 to backend
```

### Connection Limits

```yaml
annotations:
  haproxy.org/pod-maxconn: "250"                  # Max connections per pod
  haproxy.org/server-maxconn: "500"               # Max connections per server
```

### Whitelisting

```yaml
annotations:
  haproxy.org/whitelist: "10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16"
  haproxy.org/blacklist: "1.2.3.4"
```

### CORS

```yaml
annotations:
  haproxy.org/cors-enable: "true"
  haproxy.org/cors-allow-origin: "https://example.com"
  haproxy.org/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
  haproxy.org/cors-allow-headers: "Content-Type, Authorization"
  haproxy.org/cors-max-age: "86400"
```

### Sticky Sessions

```yaml
annotations:
  haproxy.org/cookie-persistence: "mycookie"      # Session affinity via cookie
```

## ConfigMap Global Settings

The controller's global config is set via the Helm `controller.config` map or a ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: haproxy-ingress
  namespace: haproxy-ingress
data:
  timeout-connect: "5s"
  timeout-client: "60s"
  timeout-server: "60s"
  timeout-http-request: "10s"
  maxconn: "2000"
  nbthread: "4"
  ssl-redirect: "true"
  syslog-server: "address:stdout, format:raw, facility:daemon"
  log-format: "%ci:%cp [%tr] %ft %b/%s %TR/%Tw/%Tc/%Tr/%Ta %ST %B %CC %CS %tsc %ac/%fc/%bc/%sc/%rc %sq/%bq %hr %hs %{+Q}r"
```

## Monitoring

### HAProxy Stats Page

Enable the stats endpoint:

```yaml
# In values.yaml
controller:
  config:
    stats: "true"
    stats-port: "1024"
```

Access via port-forward:

```bash
kubectl port-forward -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress 1024:1024
# Open http://localhost:1024/stats
```

### Prometheus Metrics

```yaml
controller:
  config:
    prometheus: "true"
  serviceMonitor:
    enabled: true
```

Metrics are exposed on port 9101 by default.

```bash
# Port-forward and scrape
kubectl port-forward -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress 9101:9101
curl http://localhost:9101/metrics
```

### Key Metrics

| Metric | Description |
|--------|-------------|
| `haproxy_frontend_current_sessions` | Active connections on frontend |
| `haproxy_backend_current_sessions` | Active connections per backend |
| `haproxy_server_response_time_average_seconds` | Avg backend response time |
| `haproxy_frontend_http_requests_total` | Total HTTP requests |
| `haproxy_backend_http_responses_total` | Responses by status code |
| `haproxy_backend_connection_errors_total` | Backend connection failures |
| `haproxy_frontend_bytes_in_total` | Inbound bytes |
| `haproxy_frontend_bytes_out_total` | Outbound bytes |

## One-Liners

```bash
# Check HAProxy Ingress pods
kubectl get pods -n haproxy-ingress -o wide

# Check the NLB external hostname
kubectl get svc -n haproxy-ingress -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'

# Create an Ingress resource quickly (no YAML needed)
kubectl create ingress myapp-ingress --class=haproxy --rule="myapp.example.com/*=myapp:8080" -n default

# Create Ingress with TLS
kubectl create ingress myapp-ingress --class=haproxy --rule="myapp.example.com/*=myapp:8080,tls=myapp-tls" -n default

# View all Ingress resources using HAProxy
kubectl get ingress -A -o json | jq '.items[] | select(.spec.ingressClassName=="haproxy") | {name: .metadata.name, ns: .metadata.namespace, hosts: [.spec.rules[].host]}'

# Check HAProxy controller logs
kubectl logs -n haproxy-ingress -l app.kubernetes.io/name=kubernetes-ingress --tail=100

# Follow logs for errors
kubectl logs -n haproxy-ingress -l app.kubernetes.io/name=kubernetes-ingress -f | grep -i "error\|warn"

# View generated HAProxy config
kubectl exec -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress -- cat /etc/haproxy/haproxy.cfg

# Check HAProxy config for a specific backend
kubectl exec -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress -- cat /etc/haproxy/haproxy.cfg | grep -A 20 "backend.*myapp"

# Reload HAProxy config (graceful)
kubectl exec -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress -- kill -USR2 1

# Check current connections
kubectl exec -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress -- sh -c 'echo "show stat" | socat /var/run/haproxy.sock -' | cut -d, -f1,2,5,8 | column -t -s,

# Show HAProxy info (version, uptime, connections)
kubectl exec -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress -- sh -c 'echo "show info" | socat /var/run/haproxy.sock -'

# Check which Helm values are set
helm get values haproxy-ingress -n haproxy-ingress

# Check Helm release status
helm status haproxy-ingress -n haproxy-ingress

# Test an Ingress endpoint
curl -H "Host: myapp.example.com" http://<nlb-hostname>/
```

## Troubleshooting

### 503 Service Unavailable

```bash
# Check if backend pods are running
kubectl get pods -l app=myapp

# Check if endpoints exist for the service
kubectl get endpoints myapp

# Verify the service port matches the Ingress backend port
kubectl get svc myapp -o jsonpath='{.spec.ports}'

# Check HAProxy backend status
kubectl exec -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress -- \
  sh -c 'echo "show stat" | socat /var/run/haproxy.sock -' | grep myapp
```

### 404 Not Found

```bash
# Check Ingress rules match the request host/path
kubectl get ingress myapp -o yaml

# Check IngressClass is correct
kubectl get ingress myapp -o jsonpath='{.spec.ingressClassName}'

# Check if HAProxy picked up the Ingress
kubectl logs -n haproxy-ingress -l app.kubernetes.io/name=kubernetes-ingress | grep myapp
```

### Timeouts

```bash
# Check timeout annotations on the Ingress
kubectl get ingress myapp -o jsonpath='{.metadata.annotations}' | jq

# Check global timeouts
kubectl get configmap -n haproxy-ingress -o yaml | grep timeout

# Check backend response time from HAProxy stats
kubectl exec -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress -- \
  sh -c 'echo "show stat" | socat /var/run/haproxy.sock -' | grep myapp | cut -d, -f1,2,61
```

### TLS Issues

```bash
# Verify TLS secret exists and has correct data
kubectl get secret myapp-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates -subject

# Check if HAProxy loaded the certificate
kubectl exec -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress -- \
  cat /etc/haproxy/haproxy.cfg | grep -A 5 "bind.*ssl"

# Test TLS externally
curl -vI https://myapp.example.com 2>&1 | grep -E "subject|expire|issuer"
```

### Controller Not Picking Up Ingress

```bash
# Check IngressClass
kubectl get ingressclass

# Verify controller is watching the right IngressClass
kubectl logs -n haproxy-ingress -l app.kubernetes.io/name=kubernetes-ingress | grep -i "ingress\|class"

# Check controller args
kubectl get deploy -n haproxy-ingress -o jsonpath='{.items[0].spec.template.spec.containers[0].args}'
```

## Tips and Tricks

### Use PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: haproxy-ingress
  namespace: haproxy-ingress
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: kubernetes-ingress
```

### Spread Across AZs

```yaml
controller:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: kubernetes-ingress
```

### Graceful Shutdown

Ensure in-flight connections complete during pod termination:

```yaml
controller:
  terminationGracePeriodSeconds: 60
  config:
    timeout-client-fin: "50s"
    timeout-server-fin: "50s"
```

### Custom Error Pages

```yaml
annotations:
  haproxy.org/custom-http-errors: "403,404,500,503"
  haproxy.org/custom-http-errors-backend: "error-pages"
```

### WebSocket Support

```yaml
annotations:
  haproxy.org/timeout-tunnel: "3600s"     # Keep WebSocket connections alive
```

### Proxy Protocol (Preserve Client IP)

When behind NLB with target type `instance`:

```yaml
controller:
  config:
    proxy-protocol: "true"

# NLB annotation
service:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
```

With target type `ip`, client IP is preserved without proxy protocol.

## Quick Reference

```bash
# Install
helm install haproxy-ingress haproxytech/kubernetes-ingress -n haproxy-ingress --create-namespace -f values.yaml

# Upgrade
helm upgrade haproxy-ingress haproxytech/kubernetes-ingress -n haproxy-ingress -f values.yaml

# Status
kubectl get pods -n haproxy-ingress
kubectl get svc -n haproxy-ingress

# NLB hostname
kubectl get svc -n haproxy-ingress -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'

# Logs
kubectl logs -n haproxy-ingress -l app.kubernetes.io/name=kubernetes-ingress --tail=100

# HAProxy config
kubectl exec -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress -- cat /etc/haproxy/haproxy.cfg

# Stats socket
kubectl exec -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress -- sh -c 'echo "show stat" | socat /var/run/haproxy.sock -'

# All HAProxy-managed Ingresses
kubectl get ingress -A --field-selector metadata.annotations.ingressClassName=haproxy 2>/dev/null || kubectl get ingress -A | grep haproxy

# Test endpoint
curl -H "Host: myapp.example.com" http://<nlb-hostname>/health
```
