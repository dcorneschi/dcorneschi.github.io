# How Kubernetes Ingress Controllers Work Internally

The watch-compute-reload loop that every Ingress controller follows — how it watches Ingress objects, computes configuration, updates the data plane, and handles reloads vs dynamic updates.

Note: For Ingress resource syntax, routing rules, and annotations, see the Ingress guide. This article focuses on how the controller itself works under the hood.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Ingress Controller (pod running in the cluster)                    │
│                                                                     │
│  ┌─────────────┐     ┌──────────────┐     ┌──────────────────────┐  │
│  │  Watch Loop │────▶│  Config      │────▶│  Data Plane          │  │
│  │  (informers)│     │  Builder     │     │  (proxy process)     │  │
│  │             │     │              │     │                      │  │
│  │  Watches:   │     │  Computes    │     │  NGINX / HAProxy /   │  │
│  │  - Ingress  │     │  desired     │     │  Envoy / Traefik     │  │
│  │  - Services │     │  proxy config│     │                      │  │
│  │  - Endpoints│     │  from K8s    │     │  Receives traffic    │  │
│  │  - Secrets  │     │  objects     │     │  and routes it       │  │
│  │  - ConfigMap│     │              │     │                      │  │
│  └─────────────┘     └──────────────┘     └──────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
         │                                           ▲
         │ Watch API server                          │ Traffic from LoadBalancer/NodePort
         ▼                                           │
┌───────────────┐                           ┌──────────────┐
│   API Server  │                           │  External LB │
│               │                           │  (cloud/bare │
│               │                           │   metal)     │
└───────────────┘                           └──────────────┘
```

## The Controller Loop

Every Ingress controller follows the same pattern regardless of which proxy it uses:

```
┌────────────────────────────────────────────────────────────────┐
│  Controller Loop                                               │
│                                                                │
│  1. WATCH: Receive events from API server                      │
│     - Ingress created/updated/deleted                          │
│     - Service endpoints changed                                │
│     - TLS Secret updated                                       │
│     - IngressClass changed                                     │
│                                                                │
│  2. COMPUTE: Build desired proxy configuration                 │
│     - Merge all Ingress objects for this IngressClass          │
│     - Resolve Services to backend pod IPs                      │
│     - Load TLS certificates from Secrets                       │
│     - Apply annotations (rate limiting, auth, rewrites)        │
│     - Validate configuration                                   │
│                                                                │
│  3. UPDATE: Apply configuration to data plane                  │
│     - Full reload (generate config file, send SIGHUP)          │
│     - OR dynamic update (API call to proxy, no reload)         │
│     - OR hot-reload (zero-downtime binary swap)                │
│                                                                │
│  4. STATUS: Update Ingress object status                       │
│     - Set status.loadBalancer.ingress[].hostname/ip            │
│     - So users can see the external endpoint                   │
│                                                                │
│  Loop time: typically 1-5 seconds from event to active config  │
└────────────────────────────────────────────────────────────────┘
```

## What the Controller Watches

```bash
# The controller creates informers for these resources:
# (same list-watch pattern described in the watches/informers article)

Ingress objects       → routing rules (host, path → service)
IngressClass objects  → which controller handles which Ingress
Service objects       → service port mapping
EndpointSlices        → actual pod IPs behind services
Secrets               → TLS certificates for HTTPS
ConfigMaps            → controller-wide configuration
Nodes                 → for topology-aware routing (some controllers)
```

### IngressClass — Controller Selection

Multiple Ingress controllers can run in the same cluster. IngressClass determines which controller handles which Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx    # Controller identifier
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
spec:
  ingressClassName: nginx              # Which controller handles this
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

The controller only processes Ingress objects matching its IngressClass. It ignores others.

## Configuration Generation

### NGINX Ingress Controller Example

The NGINX controller generates an `nginx.conf` from Kubernetes objects:

```
┌─────────────────────────────────────────────────────────────┐
│  Kubernetes Objects          →    Generated nginx.conf      │
│                                                             │
│  Ingress:                         server {                  │
│    host: app.example.com            server_name app.example.com;
│    path: /api                       location /api {         │
│    backend: api-svc:8080              proxy_pass http://upstream_api;
│                                     }                       │
│  Ingress:                           location / {            │
│    host: app.example.com              proxy_pass http://upstream_web;
│    path: /                          }                       │
│    backend: web-svc:80            }                         │
│                                                             │
│  EndpointSlice (api-svc):         upstream upstream_api {   │
│    10.244.1.5:8080                  server 10.244.1.5:8080; │
│    10.244.2.8:8080                  server 10.244.2.8:8080; │
│                                   }                         │
│                                                             │
│  Secret (tls-cert):               ssl_certificate /tmp/tls.crt;
│    tls.crt + tls.key              ssl_certificate_key /tmp/tls.key;
└─────────────────────────────────────────────────────────────┘
```

### HAProxy Ingress Controller

Similar pattern but generates HAProxy configuration:

```
Ingress objects → haproxy.cfg with frontends/backends
EndpointSlices → backend server lines with pod IPs
Annotations   → HAProxy-specific options (rate limiting, etc.)
```

### Envoy-Based Controllers (Contour, Emissary)

Instead of config files, these use Envoy's xDS API for dynamic updates:

```
Controller → computes desired state → pushes via gRPC xDS → Envoy picks up instantly
```

No config file, no reload needed.

## Reload vs Dynamic Update

### Full Reload (NGINX default for config changes)

```
Config change detected
    │
    ▼
Generate new nginx.conf
    │
    ▼
Validate: nginx -t
    │
    ▼
Signal: SIGHUP to master process
    │
    ▼
NGINX master spawns new workers with new config
Old workers finish existing connections, then exit
    │
    ▼
Zero-downtime reload complete (but takes ~1-2s)
```

**When full reload happens:**
- New Ingress rule added/removed
- Host or path changes
- TLS certificate changes
- Annotation changes that affect nginx.conf structure
- ConfigMap changes

### Dynamic Update (NGINX — endpoint changes only)

```
EndpointSlice changes (pod IP added/removed)
    │
    ▼
Controller updates NGINX's Lua-based upstream
via internal API (no config file change)
    │
    ▼
Instant update — no reload, no connection drop
```

The NGINX Ingress Controller uses Lua and an internal endpoint list that can be updated without reloading the entire config. This is critical for clusters with frequent pod scaling.

### Envoy xDS (No Reload Ever)

```
Any change (routes, backends, TLS)
    │
    ▼
Controller pushes new config via gRPC to Envoy
    │
    ▼
Envoy applies atomically (connections unaffected)
```

| Controller | Mechanism | Endpoint Update | Route Update |
|-----------|-----------|-----------------|--------------|
| NGINX Ingress | Config file + SIGHUP | Dynamic (Lua) | Full reload |
| HAProxy Ingress | Config file + reload | Dynamic (runtime API) | Full reload |
| Traefik | Internal (Go) | Dynamic | Dynamic |
| Contour/Envoy | xDS gRPC | Dynamic | Dynamic |
| AWS ALB Controller | Cloud API calls | Dynamic (target group) | API update |

## Backend Resolution — Services to Pod IPs

The controller resolves Services to actual pod IPs (it doesn't route through ClusterIP):

```
┌─────────────────────────────────────────────────────────────┐
│  Why controllers use pod IPs directly:                      │
│                                                             │
│  ClusterIP path:                                            │
│    Client → Controller → ClusterIP → kube-proxy → Pod       │
│    (double hop, extra NAT, lose client IP)                  │
│                                                             │
│  Direct pod IP path (what controllers actually do):         │
│    Client → Controller → Pod IP directly                    │
│    (single hop, preserves client IP, better performance)    │
│                                                             │
│  The controller watches EndpointSlices and builds           │
│  upstream lists with actual pod IPs — bypassing kube-proxy  │
└─────────────────────────────────────────────────────────────┘
```

```bash
# NGINX upstream shows pod IPs (not ClusterIP):
kubectl exec -n ingress-nginx <controller-pod> -- cat /etc/nginx/nginx.conf | grep "upstream"
# upstream upstream-default-my-app-80 {
#   server 10.244.1.5:8080;
#   server 10.244.2.8:8080;
# }
```

## TLS Certificate Handling

```
┌────────────────────────────────────────────────────────────────┐
│  TLS Flow                                                      │
│                                                                │
│  1. Ingress spec references a Secret:                          │
│     spec.tls[].secretName: my-tls-cert                         │
│                                                                │
│  2. Controller watches the Secret                              │
│     (reads tls.crt and tls.key)                                │
│                                                                │
│  3. Writes cert/key to filesystem (or memory)                  │
│     for the proxy process to use                               │
│                                                                │
│  4. On Secret update (cert rotation):                          │
│     → Controller detects change                                │
│     → Writes new cert                                          │
│     → Triggers reload (or dynamic TLS update)                  │
│                                                                │
│  NGINX: stores in /etc/ingress-controller/ssl/                 │
│  Envoy: receives via SDS (Secret Discovery Service)            │
└────────────────────────────────────────────────────────────────┘
```

## Status Updates

After configuring the proxy, the controller updates the Ingress object's status:

```yaml
status:
  loadBalancer:
    ingress:
    - hostname: abc123.us-east-1.elb.amazonaws.com
    # OR
    - ip: 203.0.113.50
```

This tells users (and external-dns) what address to use:

```bash
kubectl get ingress my-app
# NAME     CLASS   HOSTS             ADDRESS                                     PORTS
# my-app   nginx   app.example.com   abc123.us-east-1.elb.amazonaws.com          80, 443
```

## Health Checks

The controller performs its own backend health checks (separate from Kubernetes readiness probes):

```
┌────────────────────────────────────────────────────────────┐
│  Two layers of health checking:                            │
│                                                            │
│  Layer 1: Kubernetes (readiness probe)                     │
│    → Pod not Ready → removed from EndpointSlice            │
│    → Controller sees endpoint removal → removes backend    │
│                                                            │
│  Layer 2: Proxy-level (active health check)                │
│    → Proxy sends periodic health requests to backends      │
│    → Backend fails → proxy marks it down internally        │
│    → Faster detection than waiting for K8s endpoint update │
│                                                            │
│  Both work together for fastest failover                   │
└────────────────────────────────────────────────────────────┘
```

## Controller Deployment Patterns

### Single Replica (Simple)

```yaml
replicas: 1
# Simple but single point of failure
# Reload causes brief config gap
```

### Multiple Replicas + Leader Election

```yaml
replicas: 3
# All replicas serve traffic (data plane)
# Only leader writes status updates (control plane)
# Leader election via Lease object
```

### DaemonSet (One Per Node)

```yaml
kind: DaemonSet
# Controller on every node
# Combined with hostNetwork: true
# No LoadBalancer needed — clients hit nodes directly
# Used in bare-metal setups with external LB (MetalLB/F5)
```

## AWS ALB Ingress Controller — Different Model

The AWS ALB Controller doesn't run a proxy. It creates cloud resources:

```
Ingress object created
    │
    ▼
ALB Controller watches Ingress
    │
    ▼
Creates/updates AWS ALB via API:
  - ALB (Application Load Balancer)
  - Target Groups (with pod IPs)
  - Listeners (HTTP/HTTPS)
  - Rules (path/host routing)
    │
    ▼
Traffic: Client → ALB → Pod IPs directly
         (no proxy pod in the cluster)
```

No reload concept — changes are API calls to AWS.

## Performance Considerations

| Factor | Impact | Mitigation |
|--------|--------|------------|
| Frequent endpoint changes | Triggers reloads (NGINX) | Dynamic upstream update (Lua) |
| Many Ingress objects | Large config file, slow reload | Batch changes, use Envoy-based controllers |
| TLS cert rotation | Full reload per cert change | Dynamic TLS (Envoy SDS) |
| Large number of routes | Memory/CPU for config parsing | Split across multiple IngressClasses |
| Reload during traffic spike | Brief connection errors | Rolling restart, drain before reload |

## Debugging

```bash
# Check controller logs:
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=50

# Check controller pod is healthy:
kubectl get pods -n ingress-nginx

# See generated config (NGINX):
kubectl exec -n ingress-nginx <pod> -- cat /etc/nginx/nginx.conf

# Check if Ingress has an address:
kubectl get ingress -A

# Check controller's internal state:
kubectl exec -n ingress-nginx <pod> -- /nginx-ingress-controller --dump-config

# See backend resolution:
kubectl exec -n ingress-nginx <pod> -- curl -s localhost:10246/configuration/backends | jq .

# Check IngressClass:
kubectl get ingressclass

# Test from inside the cluster:
kubectl run curl --image=curlimages/curl --rm -it -- curl -H "Host: app.example.com" http://<controller-svc-ip>
```

## Quick Reference

```bash
# Controller loop: Watch → Compute config → Update data plane → Update status

# What it watches:
# Ingress, IngressClass, Services, EndpointSlices, Secrets, ConfigMaps

# Config update mechanisms:
# - Full reload: config file + SIGHUP (NGINX for route changes)
# - Dynamic update: internal API (NGINX for endpoint changes)
# - xDS push: gRPC (Envoy-based controllers)
# - Cloud API: no proxy (AWS ALB Controller)

# Backend resolution:
# Controllers use pod IPs directly (from EndpointSlices)
# They do NOT route through ClusterIP/kube-proxy

# IngressClass determines which controller handles which Ingress
# Multiple controllers can coexist in the same cluster

# Status update:
# Controller writes external address to ingress.status.loadBalancer

# Key debugging:
kubectl logs -n <controller-ns> -l <controller-label> --tail=50
kubectl get ingress -A  # Check ADDRESS column
kubectl describe ingress <name>  # Check events for errors
```
