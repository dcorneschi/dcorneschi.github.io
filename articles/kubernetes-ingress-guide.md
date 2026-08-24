# Ingress

Related: [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) | [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) | [IngressClass](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class)

## Background

An Ingress is a Kubernetes API object that manages external access to services
in a cluster, typically HTTP/HTTPS. It provides:

| Feature            | Description                                                    |
|--------------------|----------------------------------------------------------------|
| Host-based routing | Route `app.example.com` to service A, `api.example.com` to B  |
| Path-based routing | Route `/app` to service A, `/api` to service B                |
| TLS termination    | Terminate HTTPS at the ingress and forward HTTP to backends    |
| Name-based virtual hosting | Multiple hostnames on a single IP address             |

## Ingress vs Service (LoadBalancer/NodePort)

| Aspect                | Ingress                              | LoadBalancer Service              |
|-----------------------|--------------------------------------|-----------------------------------|
| Layer                 | L7 (HTTP/HTTPS)                      | L4 (TCP/UDP)                      |
| External IPs needed   | One (shared by all Ingress rules)    | One per service                   |
| Routing               | Host/path-based                      | Port-based only                   |
| TLS termination       | Built-in                             | Requires app-level TLS            |
| Cost (cloud)          | One load balancer for many services  | One load balancer per service     |

## Ingress Controller

An Ingress resource does nothing on its own. You need an Ingress Controller —
a pod that watches Ingress objects and configures a reverse proxy (nginx,
Traefik, HAProxy, etc.) accordingly.

Common controllers:

- **ingress-nginx** — the most widely used, maintained by the Kubernetes community
- **Traefik** — built-in to k3s
- **HAProxy Ingress**
- **Cloud-native** — AWS ALB Ingress Controller, GCE Ingress

> **Important:** If no Ingress Controller is running, creating an Ingress
> resource has no effect. The resource is stored in the API server but nothing
> acts on it.

## Ingress Resource Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:                          # controller-specific settings
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx               # which controller handles this
  defaultBackend:                       # catch-all (optional)
    service:
      name: fallback-svc
      port:
        number: 80
  tls:                                  # TLS termination (optional)
  - hosts:
    - app.example.com
    secretName: app-tls                 # must be type kubernetes.io/tls
  rules:                                # routing rules
  - host: app.example.com              # omit for all hosts
    http:
      paths:
      - path: /
        pathType: Prefix               # Exact or Prefix
        backend:
          service:
            name: my-svc
            port:
              number: 80
```

## IngressClass

IngressClass tells Kubernetes which controller should handle an Ingress. If
you have multiple controllers (e.g., nginx + traefik), each Ingress must
specify which one to use via `ingressClassName`.

```bash
kubectl get ingressclass
```

To set a default IngressClass (so you can omit `ingressClassName`):

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
```

## Common nginx-ingress Annotations

| Annotation                                          | Purpose                                    |
|-----------------------------------------------------|--------------------------------------------|
| `nginx.ingress.kubernetes.io/rewrite-target: /`     | Rewrite the URL path before forwarding     |
| `nginx.ingress.kubernetes.io/ssl-redirect: "false"` | Disable automatic HTTP→HTTPS redirect      |
| `nginx.ingress.kubernetes.io/proxy-body-size: "0"`  | Remove request body size limit             |
| `nginx.ingress.kubernetes.io/proxy-read-timeout`    | Backend read timeout in seconds            |
| `nginx.ingress.kubernetes.io/affinity: "cookie"`    | Enable sticky sessions                     |
| `nginx.ingress.kubernetes.io/auth-type: basic`      | Enable basic auth                          |

## Debugging Ingress

```bash
# Check if Ingress has an ADDRESS assigned (empty = controller not processing it)
kubectl get ingress

# Detailed status — events, rules, TLS, and backend endpoints
kubectl describe ingress <name>

# Verify the controller pods are running
kubectl get pods -n ingress-nginx

# Check controller logs for routing errors
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50

# Verify backend service has ready endpoints
kubectl get endpoints <service-name>

# Test from inside the cluster (bypass Ingress)
kubectl run curl --image=curlimages/curl --rm -it -- curl http://<service-name>.<namespace>.svc.cluster.local
```

> **Key indicator:** If `kubectl get ingress` shows no ADDRESS, either the controller isn't running or the `ingressClassName` doesn't match any installed controller.

## Common Mistakes

1. **No Ingress Controller installed** — The Ingress resource is created but nothing happens. No ADDRESS is assigned, no traffic is routed. Always verify the controller is running first.
2. **Wrong `ingressClassName`** — If the class doesn't match any controller, the Ingress is ignored. Check `kubectl get ingressclass` to see available classes.
3. **Service port mismatch** — The `port.number` in the Ingress backend must match the Service's `port`, not the container's `targetPort`. The Ingress talks to the Service, not directly to the pod.
4. **Missing TLS Secret** — If the Secret referenced in `tls.secretName` doesn't exist, the controller falls back to its default (fake) certificate. The Ingress still works but with an untrusted cert.
5. **Forgetting `rewrite-target`** — Without it, a request to `/one` is forwarded as `/one` to the backend. If the backend only serves on `/`, you get a 404 from the backend, not from the Ingress.
6. **Empty endpoints** — The Ingress returns 503 if the backend service has no ready endpoints. Check `kubectl get endpoints <svc-name>` to verify.
7. **Confusing Prefix path matching** — `Prefix` matches on `/` boundaries. `/app` matches `/app` and `/app/foo` but NOT `/application`. This trips people up.

## Links

- [Migrate to Gateway API (Datadog Blog)](https://www.datadoghq.com/blog/migrate-to-gateway-api)
- [Ingress Explained (YouTube)](https://www.youtube.com/watch?v=GhZi4DxaxxE)
- [EKS Gateway API Workshop (GitHub)](https://github.com/csonel/eks-gateway-api-workshop)
- [Gateway API Documentation](https://gateway-api.sigs.k8s.io/) — the successor to Ingress, supports traffic splitting, header-based routing, and multi-tenancy
