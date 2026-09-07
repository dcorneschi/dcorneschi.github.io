# Installing metrics-server on Kubernetes

`metrics-server` collects resource usage (CPU/memory) from kubelets and exposes
it through the Metrics API (`metrics.k8s.io`). It powers `kubectl top` and is a
prerequisite for the Horizontal Pod Autoscaler.

> **Note on `--kubelet-insecure-tls`:** In many clusters the kubelet serving
> certificate is self-signed and not trusted by metrics-server, so it fails to
> scrape with an x509 error. Adding `--kubelet-insecure-tls` skips that
> verification. It's fine for labs and clusters without properly signed kubelet
> certs, but in production prefer configuring the kubelet with certificates
> signed by the cluster CA instead.

## Installation

### Helm

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update

# Basic install/upgrade
helm upgrade --install metrics-server metrics-server/metrics-server

# Install into kube-system with insecure kubelet TLS
helm install metrics-server metrics-server/metrics-server \
  --set args[0]=--kubelet-insecure-tls \
  --namespace kube-system
```

Install directly from a release chart URL:

```bash
helm install metrics-server https://github.com/kubernetes-sigs/metrics-server/releases/download/metrics-server-helm-chart-3.13.0/metrics-server-3.13.0.tgz

helm install metrics-server https://github.com/kubernetes-sigs/metrics-server/releases/download/metrics-server-helm-chart-3.13.0/metrics-server-3.13.0.tgz \
  --set args="{--kubelet-insecure-tls}"
```

### Apply the manifest directly

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Edit the manifest and then apply

```bash
wget -O metrics-server.yaml https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Add the `--kubelet-insecure-tls` argument to the container args:

```yaml
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
```

```bash
kubectl apply -f metrics-server.yaml
```

### High-availability manifest (from GitHub)

For clusters with 2+ nodes, the project publishes a high-availability manifest
that runs multiple replicas with a PodDisruptionBudget and anti-affinity:

```bash
# Latest release
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability.yaml

# Pin a specific version instead of latest
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.7.2/high-availability.yaml
```

> HA requires at least 2 nodes; the anti-affinity rule keeps replicas on
> separate nodes. If your kubelet certs aren't signed by the cluster CA, you
> still need to add `--kubelet-insecure-tls` — download the file, edit the
> container args as shown above, then apply it.

### Argo CD (GitOps)

If you run Argo CD, install metrics-server declaratively as an `Application`. It
points at the same Helm chart and passes `--kubelet-insecure-tls` as a chart
parameter:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metrics-server
  namespace: argo
spec:
  project: default
  source:
    repoURL: https://kubernetes-sigs.github.io/metrics-server/
    targetRevision: 3.12.1
    helm:
      parameters:
        - name: args[0]
          value: '--kubelet-insecure-tls'
    chart: metrics-server
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
# Apply the Application (in the namespace where Argo CD runs)
kubectl apply -f metrics-server-application.yaml

# Watch it sync
kubectl get application metrics-server -n argo
argocd app get metrics-server
```

Notes:

- `metadata.namespace` (`argo` here) is where the Argo CD `Application` object
  lives — match it to your Argo CD install namespace (often `argocd`).
- `destination.namespace` (`kube-system`) is where metrics-server is deployed,
  and `CreateNamespace=true` creates it if missing.
- `automated` with `prune` and `selfHeal` keeps the cluster reconciled to the
  manifest and reverts drift automatically.

## Verify the status

```bash
kubectl get deployment metrics-server -n kube-system

kubectl get apiservice v1beta1.metrics.k8s.io
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml

kubectl describe apiservices v1beta1.metrics.k8s.io
```

The APIService should report `Available: True`. Once ready, `kubectl top nodes`
and `kubectl top pods` return data.

## Query the Metrics API

```bash
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq .
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods" | jq .
```

Or through `kubectl proxy`:

```bash
kubectl proxy

curl -o - http://localhost:8001/apis/metrics.k8s.io/v1beta1/pods/
curl -o - http://localhost:8001/apis/metrics.k8s.io/v1beta1/nodes/
```

## Links

- [metrics-server project site](https://kubernetes-sigs.github.io/metrics-server)
- [metrics-server Helm chart (Artifact Hub)](https://artifacthub.io/packages/helm/metrics-server/metrics-server)
- [Install and troubleshoot metrics-server on EKS](https://repost.aws/knowledge-center/eks-metrics-server-install-troubleshoot)

## Skills Practiced

- Installing metrics-server via Helm and via the raw manifest
- Understanding when and why `--kubelet-insecure-tls` is required
- Verifying the `v1beta1.metrics.k8s.io` APIService is available
- Querying the Metrics API directly with `kubectl get --raw` and `kubectl proxy`
