How to set up NGINX Ingress with MetalLB on MicroK8s for bare-metal load balancing in a homelab.

### Overview

MicroK8s runs on bare metal (or VMs) where there's no cloud provider to hand out external IPs for LoadBalancer services. MetalLB fills that gap by assigning real LAN IPs to services, and pairing it with the NGINX Ingress controller gives you proper hostname-based routing — no port-forwarding or NodePort hacks needed.

### Prerequisites

- MicroK8s installed and running
- A range of unused IPs on your LAN reserved for MetalLB

### Enable the Addons

```bash
microk8s enable metallb ingress dns
```

When prompted for the IP range, provide a CIDR or range from your LAN that won't conflict with DHCP:

```
Enter each IP address range delimited by comma (e.g. 10.64.140.43-10.64.140.49,192.168.0.105-192.168.0.111): 192.168.1.200-192.168.1.210
```

### Verify MetalLB

```bash
# Check MetalLB pods are running
microk8s kubectl get pods -n metallb-system

# Check the ingress controller got an external IP
microk8s kubectl get svc -n ingress
```

The NGINX ingress controller service should now show an `EXTERNAL-IP` from your MetalLB pool instead of `<pending>`.

### MetalLB Configuration

MicroK8s configures MetalLB in L2 mode by default. To view or modify the IP pool:

```bash
microk8s kubectl get ipaddresspool -n metallb-system -o yaml
```

To update the pool after initial setup:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-addresspool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.200-192.168.1.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
```

```bash
microk8s kubectl apply -f metallb-pool.yaml
```

### Create an Ingress Resource

With MetalLB providing the external IP and NGINX handling routing, create Ingress resources as usual:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.homelab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-svc
                port:
                  number: 80
```

```bash
microk8s kubectl apply -f my-app-ingress.yaml
```

### DNS Setup

Point your hostnames to the MetalLB-assigned external IP. Options:

```bash
# Option 1: /etc/hosts for quick testing
echo "192.168.1.200 app.homelab.local" | sudo tee -a /etc/hosts

# Option 2: Local DNS server (Pi-hole, CoreDNS, dnsmasq)
# Add A record: app.homelab.local -> 192.168.1.200
```

### Testing

```bash
# Check external IP assignment
microk8s kubectl get svc -n ingress

# Check ingress resource
microk8s kubectl get ingress

# Test connectivity
curl -H "Host: app.homelab.local" http://192.168.1.200
```

### Common Annotations

| Annotation | Purpose |
|-----------|---------|
| `nginx.ingress.kubernetes.io/rewrite-target: /` | Strip path prefix before forwarding to backend |
| `nginx.ingress.kubernetes.io/ssl-redirect: "false"` | Disable HTTPS redirect for HTTP-only services |
| `nginx.ingress.kubernetes.io/proxy-body-size: "50m"` | Increase max upload size |
| `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"` | Forward to HTTPS backends |

### Troubleshooting

| Issue | Fix |
|-------|-----|
| Ingress service stuck on `<pending>` | MetalLB not running or IP pool exhausted — check `microk8s kubectl get pods -n metallb-system` |
| No route to external IP | MetalLB L2 mode needs the client on the same L2 network — check ARP with `arping 192.168.1.200` |
| 404 from ingress | Hostname mismatch — ensure the `Host` header matches the ingress rule |
| Multiple ingress controllers | Set `ingressClassName: nginx` explicitly to avoid conflicts |
| IP conflict with DHCP | Shrink your DHCP range or pick IPs outside it for the MetalLB pool |

### Architecture

```
Client (LAN)
    │
    ▼
MetalLB (L2 advertisement, assigns external IP)
    │
    ▼
NGINX Ingress Controller (routes by hostname/path)
    │
    ▼
Backend Service/Pods
```

### Related Commands

- `microk8s status` — check enabled addons
- `microk8s kubectl get ipaddresspool -n metallb-system` — view IP pool
- `microk8s kubectl describe ingress <name>` — debug ingress routing
- `microk8s kubectl logs -n ingress <nginx-pod>` — ingress controller logs
- `arping <metallb-ip>` — verify L2 advertisement is working
