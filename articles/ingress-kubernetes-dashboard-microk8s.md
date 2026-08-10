<img src="/articles/images/kubernetes-logo.svg" alt="Kubernetes" width="150">

How to expose the Kubernetes Dashboard through an Ingress resource on MicroK8s.

### Overview

MicroK8s bundles the Kubernetes Dashboard as an addon, but by default it's only accessible via `kubectl proxy` or port-forwarding. Setting up an Ingress lets you reach the dashboard through a proper hostname — useful in a homelab where you're already running an ingress controller.

### Prerequisites

- MicroK8s with the following addons enabled:
  - `dashboard`
  - `ingress` (or your own ingress controller)
  - `dns`

```bash
microk8s enable dashboard ingress dns
```

### Create the Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: dashboard.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kubernetes-dashboard
                port:
                  number: 443
```

Save as `dashboard-ingress.yaml` and apply:

```bash
microk8s kubectl apply -f dashboard-ingress.yaml
```

### Key Annotations

| Annotation | Purpose |
|-----------|---------|
| `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"` | Dashboard backend serves HTTPS, tell the ingress to use HTTPS when forwarding |
| `nginx.ingress.kubernetes.io/ssl-passthrough: "true"` | Pass TLS directly to the dashboard without terminating at the ingress |

### DNS Setup

Point the hostname to your MicroK8s node IP. For local testing, add an entry to `/etc/hosts`:

```bash
echo "192.168.1.100 dashboard.local" | sudo tee -a /etc/hosts
```

Replace `192.168.1.100` with your node's actual IP.

### Get the Login Token

```bash
microk8s kubectl create token default -n kube-system
```

Or retrieve an existing service account token:

```bash
microk8s kubectl -n kube-system describe secret \
  $(microk8s kubectl -n kube-system get secret | grep default-token | awk '{print $1}')
```

### Verify

```bash
# Check the ingress was created
microk8s kubectl get ingress -n kube-system

# Test connectivity
curl -k https://dashboard.local
```

Open `https://dashboard.local` in your browser and use the token to log in.

### Troubleshooting

| Issue | Fix |
|-------|-----|
| 502 Bad Gateway | Ensure `backend-protocol: "HTTPS"` annotation is set — the dashboard only speaks HTTPS |
| Connection refused | Verify the ingress controller pod is running: `microk8s kubectl get pods -n ingress` |
| Dashboard service not found | Check the service name and namespace: `microk8s kubectl get svc -n kube-system` |
| Certificate warning | Expected with self-signed certs — accept or configure a real cert with cert-manager |

### Related Commands

- `microk8s status` — check which addons are enabled
- `microk8s kubectl get ingress -A` — list all ingress resources
- `microk8s kubectl logs -n ingress <nginx-pod>` — ingress controller logs
- `microk8s dashboard-proxy` — alternative built-in proxy access (no ingress needed)
