# ArgoCD Access Methods on EKS

How to access the ArgoCD UI and CLI on an EKS cluster — port-forwarding, Ingress, LoadBalancer, and SSO integration.

## ArgoCD Server Service Types

After installing ArgoCD, the `argocd-server` Service defaults to `ClusterIP` — not accessible from outside the cluster. You need to expose it.

| Method | Use Case | External Access | TLS |
|--------|----------|----------------|-----|
| Port-forward | Development, quick testing | No (localhost only) | Built-in (self-signed) |
| LoadBalancer Service | Simple external access | Yes (via AWS NLB/CLB) | ACM or ArgoCD self-signed |
| Ingress (ALB) | Production with path/host routing | Yes (via ALB) | ACM certificate |
| Ingress (nginx/HAProxy) | Existing ingress controller | Yes (via existing LB) | cert-manager or controller TLS |
| VPN + ClusterIP | Secure internal access | Via VPN only | Built-in |

## Method 1: Port-Forward (Development)

The simplest approach — no infrastructure changes needed:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Access at `https://localhost:8080`. The CLI also works through this:

```bash
argocd login localhost:8080 --insecure
```

> **Limitation:** Only accessible from your workstation. Not suitable for teams or CI/CD.

## Method 2: LoadBalancer Service (Simple External Access)

Patch the ArgoCD server service to type LoadBalancer:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Or set it during Helm installation:

```yaml
# argocd-values.yaml
server:
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
```

```bash
helm install argocd argo/argo-cd -n argocd --create-namespace -f argocd-values.yaml
```

Get the LoadBalancer URL:

```bash
kubectl get svc argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### Internal-Only LoadBalancer

For access within VPC only (via VPN or peered networks):

```yaml
server:
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-scheme: internal
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
```

## Method 3: Ingress with AWS ALB

Use the AWS Load Balancer Controller to create an ALB for ArgoCD:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/abc-123
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/backend-protocol: HTTPS
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/conditions.argocd-server-grpc: |
      [{"field":"http-header","httpHeaderConfig":{"httpHeaderName": "Content-Type", "values":["application/grpc"]}}]
    alb.ingress.kubernetes.io/actions.argocd-server-grpc: |
      {"type":"forward","forwardConfig":{"targetGroups":[{"serviceName":"argocd-server","servicePort":"443"}]}}
spec:
  ingressClassName: alb
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
```

### Disable ArgoCD TLS (Let ALB Handle It)

If ALB terminates TLS, tell ArgoCD to serve HTTP:

```bash
kubectl -n argocd patch configmap argocd-cmd-params-cm --patch '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deployment argocd-server
```

Then change the Ingress backend protocol to HTTP:

```yaml
annotations:
  alb.ingress.kubernetes.io/backend-protocol: HTTP
```

## Method 4: Ingress with NGINX/HAProxy Controller

If you already have an ingress controller running:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-server-tls
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
```

> **SSL Passthrough:** ArgoCD uses gRPC which requires HTTP/2. SSL passthrough lets the ArgoCD server handle TLS directly, avoiding HTTP/2 issues at the ingress controller level.

### Alternative: Disable TLS on ArgoCD and Terminate at Ingress

```yaml
# argocd-cmd-params-cm
data:
  server.insecure: "true"
```

Then use standard TLS termination at the ingress:

```yaml
annotations:
  nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

## Getting the Initial Admin Password

```bash
# ArgoCD 2.x+
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

Login:

```bash
# Via CLI
argocd login argocd.example.com --username admin --password <password>

# Change the password
argocd account update-password
```

> **Best practice:** Delete the initial admin secret after setting up SSO or changing the password:
> ```bash
> kubectl -n argocd delete secret argocd-initial-admin-secret
> ```

## gRPC Access (CLI and CI/CD)

The ArgoCD CLI uses gRPC. This requires special handling depending on your access method:

| Access Method | CLI Connection | Notes |
|---------------|---------------|-------|
| Port-forward | `argocd login localhost:8080 --insecure` | Works directly |
| NLB (TCP) | `argocd login <nlb-dns>:443` | Works (NLB is L4, passes gRPC) |
| ALB (HTTP/2) | `argocd login <alb-dns>:443 --grpc-web` | ALB needs gRPC routing or use grpc-web |
| NGINX (ssl-passthrough) | `argocd login <domain>:443` | Works (passthrough preserves gRPC) |

### ALB gRPC Workaround

ALBs don't natively support gRPC in all configurations. Use `--grpc-web` flag:

```bash
argocd login argocd.example.com --grpc-web
```

Or set it globally:

```bash
argocd login argocd.example.com --grpc-web
# All subsequent commands will use grpc-web
```

## DNS Configuration

After exposing ArgoCD, point your DNS to the load balancer:

```bash
# Get the LB hostname
LB_DNS=$(kubectl get svc argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
# or for Ingress:
LB_DNS=$(kubectl get ingress argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "Create a CNAME or Route53 alias for argocd.example.com -> $LB_DNS"
```

Route 53 example:

```bash
aws route53 change-resource-record-sets --hosted-zone-id Z12345 --change-batch '{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "argocd.example.com",
      "Type": "CNAME",
      "TTL": 300,
      "ResourceRecords": [{"Value": "'$LB_DNS'"}]
    }
  }]
}'
```

## SSO Integration (Production)

For production, integrate ArgoCD with an identity provider instead of using the admin password.

### OIDC (Keycloak, Okta, Azure AD, Google)

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://argocd.example.com
  oidc.config: |
    name: Okta
    issuer: https://your-org.okta.com/oauth2/default
    clientID: 0oa1234567890abcdef
    clientSecret: $oidc.okta.clientSecret
    requestedScopes:
      - openid
      - profile
      - email
      - groups
```

Store the client secret in the argocd-secret:

```bash
kubectl -n argocd patch secret argocd-secret --patch '{"stringData": {"oidc.okta.clientSecret": "your-client-secret"}}'
```

### RBAC Policy for SSO Groups

```yaml
# argocd-rbac-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    p, role:org-admin, applications, *, */*, allow
    p, role:org-admin, clusters, get, *, allow
    p, role:org-admin, repositories, *, *, allow
    p, role:org-readonly, applications, get, */*, allow
    p, role:org-readonly, clusters, get, *, allow
    g, platform-team, role:org-admin
    g, developers, role:org-readonly
  policy.default: role:org-readonly
```

## Security Considerations

| Recommendation | Why |
|---------------|-----|
| Use internal LB + VPN for non-public clusters | Reduces attack surface |
| Enable SSO, disable admin account | No shared passwords |
| Use ACM certificates (not self-signed) | Trusted TLS, auto-renewal |
| Set `server.insecure: false` with ssl-passthrough | End-to-end encryption |
| Restrict Ingress to specific IPs (WAF or SG) | Limit who can reach the UI |
| Enable ArgoCD audit logging | Track who deployed what |
| Use IRSA for pod authentication | No long-lived credentials |
| Enforce TLS 1.2+ | Block weak ciphers |

### Disable the Default Admin User

After configuring SSO, disable the built-in admin account:

```bash
kubectl patch configmap argocd-cm -n argocd -p '{"data":{"admin.enabled":"false"}}'
kubectl -n argocd rollout restart deployment argocd-server
```

### Network Policy for ArgoCD

Restrict which pods can reach the ArgoCD server:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: argocd-server-ingress
  namespace: argocd
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: argocd-server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
    - protocol: TCP
      port: 8443
```

### Private ALB + WAF (Most Secure Pattern)

```
Private ALB (TLS 1.2+, WAF, internal subnets)
  → Ingress Controller
    → ArgoCD Server
      → OIDC/SSO Authentication
        → RBAC Authorization
```

```yaml
# Private ALB with WAF
server:
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-scheme: internal
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
  ingress:
    annotations:
      alb.ingress.kubernetes.io/scheme: internal
      alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:us-east-1:123456789012:regional/webacl/argocd-waf/abc123
      alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
```

### Secrets Management with IRSA

Use IAM Roles for Service Accounts instead of storing AWS credentials in secrets:

```bash
# Create IRSA for ArgoCD repo-server (to access private ECR, S3, etc.)
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace argocd \
  --name argocd-repo-server \
  --attach-policy-arn arn:aws:iam::123456789012:policy/ArgoCD-RepoAccess \
  --approve
```

## Quick Reference

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d

# Port-forward for quick access
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Expose via LoadBalancer
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Get LB address
kubectl get svc argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Login via CLI
argocd login argocd.example.com --username admin --password <password>

# Login via CLI (ALB with grpc-web)
argocd login argocd.example.com --grpc-web --username admin --password <password>

# Check ArgoCD server status
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server
```

## ArgoCD and Helm Charts

ArgoCD renders Helm chart templates and applies the manifests directly to Kubernetes. It does **not** create a Helm release by default — charts deployed by ArgoCD won't appear in `helm list -A`.

### Why helm list Shows Nothing

ArgoCD uses `helm template` (client-side rendering), not `helm install`. The rendered YAML is applied via the Kubernetes API, bypassing Helm's release tracking.

### Get the Helm Values ArgoCD Used

```bash
# View the full Application spec (includes helm values)
kubectl get application <app-name> -n argocd -o yaml

# Extract just the values to a file
kubectl get application <app-name> -n argocd -o jsonpath='{.spec.source.helm.values}' > values-used.yaml

# Check valuesObject (structured format, ArgoCD 2.6+)
kubectl get application <app-name> -n argocd -o jsonpath='{.spec.source.helm.valuesObject}'
```

### View Deployed Resources from a Helm-Based App

```bash
# Check what ArgoCD actually deployed
kubectl get deploy,svc,cm -n <namespace> -l app.kubernetes.io/name=<chart-name> -o yaml
```

### The Application CRD

`Application` is a Custom Resource that ArgoCD installs. Each one represents a deployed app:

```bash
# List all ArgoCD applications
kubectl get application -n argocd

# Detailed view
kubectl get application <name> -n argocd -o yaml

# Verify the CRD exists
kubectl get crd applications.argoproj.io
```

> **Note:** If you need `helm list` compatibility (e.g., for other tools that read Helm releases), enable the `helm.passCredentials` option or set `spec.source.helm.passCredentials: true` and use ArgoCD's native Helm sync mode.
