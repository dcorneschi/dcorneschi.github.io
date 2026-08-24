# Kubernetes Gateway API Guide

Related: [Gateway API Documentation](https://gateway-api.sigs.k8s.io/) | [Ingress](articles/kubernetes-ingress-guide.md)

## What Is the Gateway API?

The Gateway API is the successor to Kubernetes Ingress. It provides a more expressive, extensible, and role-oriented way to manage external traffic entering your cluster.

| Feature | Ingress | Gateway API |
|---------|---------|-------------|
| Protocol support | HTTP/HTTPS only (L7) | HTTP, gRPC, TCP, UDP, TLS (L4 + L7) |
| Routing | Path and host-based only | Path, host, header, method, weighted |
| Configuration | Single Ingress object + annotations | Dedicated resources per concern (Gateway, HTTPRoute, etc.) |
| Cross-namespace routing | Complex, not straightforward | Native via ReferenceGrant |
| Portability | Relies on vendor-specific annotations | Standardized spec, portable across controllers |
| Traffic splitting | Not supported natively | Built-in weighted routing, canary, blue-green |
| Service mesh integration | Limited | First-class support (Istio, Linkerd, etc.) |

## Core Resources

| Resource | Scope | Purpose |
|----------|-------|---------|
| `GatewayClass` | Cluster | Selects which controller manages Gateway resources (like IngressClass) |
| `Gateway` | Namespace | Defines how external traffic enters the cluster (listeners, ports, protocols) |
| `HTTPRoute` | Namespace | Routes HTTP/HTTPS traffic to backend services |
| `GRPCRoute` | Namespace | Routes gRPC traffic to backend services |
| `ReferenceGrant` | Namespace | Enables cross-namespace routing |

## Architecture (NGINX Gateway Fabric Example)

```
┌──────────────────────────────────────────────────────────────┐
│                    Gateway API Controller                    │
│                                                              │
│  ┌───────────────────┐         ┌──────────────────────────┐  │
│  │  Control Plane    │  gRPC   │       Data Plane         │  │
│  │ Fabric Controller) ──────▶  │  (NGINX Proxy Pod)       │  │
│  │  Watches Gateway  │         │  Handles actual traffic  │  │
│  │  API resources    │         │  + NGINX Agent           │  │
│  └───────────────────┘         └──────────────────────────┘  │
│                                          │                   │
└──────────────────────────────────────────┼───────────────────┘
                                           │
                                    ┌──────┴──────┐
                                    │LoadBalancer │
                                    │  / NodePort │
                                    └──────┬──────┘
                                           │
                                      Client Traffic
```

**Control Plane:** Watches Gateway API resources, converts them into NGINX config, sends config to the data plane via gRPC.

**Data Plane:** Runs NGINX with an agent. Created automatically when you create a Gateway resource. Handles actual traffic routing.

## Traffic Flow

```
Client → Load Balancer → Data Plane (NGINX Proxy) → Backend Service → Pod
                              ▲
                              │ config
                     Control Plane (watches Gateway + HTTPRoute)
```

1. Install the Gateway API Controller → deploys the Control Plane only
2. Create a Gateway resource → Control Plane creates a Data Plane pod + LoadBalancer service
3. Create an HTTPRoute → Control Plane converts rules into NGINX config and pushes to Data Plane
4. Client traffic hits the LoadBalancer → routed by Data Plane to the correct backend

## Setup

### Step 1: Install Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml

# Verify CRDs
kubectl get crds | grep gateway

# Check API resources
kubectl api-resources --api-group=gateway.networking.k8s.io
```

### Step 2: Install a Gateway API Controller

Using NGINX Gateway Fabric with Helm:

```bash
helm install ngf . -n nginx-gateway --create-namespace

# Verify
kubectl -n nginx-gateway get all
```

Other supported controllers:
- **Envoy Gateway**
- **Kong Gateway**
- **Traefik**
- **Istio**

See the full list at [Gateway API Implementations](https://gateway-api.sigs.k8s.io/implementations/).

### Step 3: Create a GatewayClass

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-gateway-class
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
```

```bash
kubectl apply -f gatewayclass.yaml
kubectl get gatewayclasses
```

## Creating a Gateway

The Gateway defines listeners (port + protocol combinations) that accept external traffic:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: webserver
spec:
  gatewayClassName: nginx-gateway-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

```bash
kubectl apply -f web-gateway.yaml
kubectl -n webserver get gateway
kubectl -n webserver describe gateway web-gateway
```

> When you create a Gateway, the controller provisions a Data Plane pod and a LoadBalancer service automatically.

### Wildcard Hostname Listener

Accept traffic for any subdomain:

```yaml
spec:
  gatewayClassName: nginx-gateway-class
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    hostname: "*.example.com"
```

## HTTPRoute — Routing Traffic

### Simple Route (All Traffic to One Service)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: webserver-httproute
  namespace: webserver
spec:
  parentRefs:
  - name: web-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: nginx-service
      port: 80
```

### Path-Based Routing

Route different paths to different services:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: colors-httproute
  namespace: colors
spec:
  parentRefs:
  - name: colors-gateway
    sectionName: http
  hostnames:
  - "dev.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /green
    backendRefs:
    - name: green-app-service
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /orange
    backendRefs:
    - name: orange-app-service
      port: 80
```

### Header-Based Routing

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: header-route
spec:
  parentRefs:
  - name: web-gateway
  rules:
  - matches:
    - headers:
      - name: x-env
        value: canary
    backendRefs:
    - name: canary-service
      port: 80
  - backendRefs:
    - name: stable-service
      port: 80
```

### Weighted Traffic Splitting (Canary)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-route
spec:
  parentRefs:
  - name: web-gateway
  rules:
  - backendRefs:
    - name: stable-service
      port: 80
      weight: 90
    - name: canary-service
      port: 80
      weight: 10
```

## Cross-Namespace Routing with ReferenceGrant

By default, an HTTPRoute can only reference services in its own namespace. Use ReferenceGrant to allow cross-namespace references:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-from-frontend
  namespace: backend          # Namespace where the target service lives
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: frontend       # Namespace where the HTTPRoute lives
  to:
  - group: ""
    kind: Service
```

## NodePort Setup (Without Cloud Load Balancer)

For bare-metal or local clusters, expose via NodePort instead of LoadBalancer:

```yaml
# custom-values.yaml for Helm
nginx:
  service:
    type: NodePort
    nodePorts:
    - port: 32000
      listenerPort: 80
    - port: 32443
      listenerPort: 443
```

```bash
helm install ngf . -n nginx-gateway --create-namespace -f custom-values.yaml
```

Access the application via `http://<node-ip>:32000`.

## Gateway API vs Ingress — When to Use What

| Use Case | Recommendation |
|----------|---------------|
| Simple HTTP path/host routing | Either works (Ingress is simpler) |
| TCP/UDP/gRPC routing | Gateway API (Ingress doesn't support L4) |
| Canary/weighted deployments | Gateway API |
| Cross-namespace routing | Gateway API (native support) |
| Header/method-based routing | Gateway API |
| Existing Ingress setup that works fine | Keep Ingress (no need to migrate immediately) |
| Service mesh integration | Gateway API |

> **Migration path:** Gateway API and Ingress can coexist. You don't need to migrate everything at once. Start new services with Gateway API and migrate existing ones gradually.

## Useful Commands

```bash
# List GatewayClasses
kubectl get gatewayclasses

# List Gateways in all namespaces
kubectl get gateways -A

# List HTTPRoutes
kubectl get httproutes -A

# Describe a Gateway (shows listeners, attached routes, addresses)
kubectl describe gateway <name> -n <namespace>

# Describe an HTTPRoute (shows rules, backend refs, parent gateway)
kubectl describe httproute <name> -n <namespace>

# Check Gateway API CRDs
kubectl get crds | grep gateway

# Check controller pods
kubectl get pods -n nginx-gateway

# View the NGINX config generated by the controller
kubectl -n <namespace> exec -it <data-plane-pod> -- nginx -T
```

## Links

- [Gateway API Official Documentation](https://gateway-api.sigs.k8s.io/)
- [Gateway API Implementations](https://gateway-api.sigs.k8s.io/implementations/)
- [NGINX Gateway Fabric](https://docs.nginx.com/nginx-gateway-fabric/)
- [Migrate to Gateway API (Datadog Blog)](https://www.datadoghq.com/blog/migrate-to-gateway-api)

Source: [Kubernetes Gateway API Tutorial (DevOpsCube)](https://devopscube.com/kubernetes-gateway-api/)
