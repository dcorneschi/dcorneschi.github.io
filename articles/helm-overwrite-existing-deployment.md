# Replacing an Existing Deployment with a Helm Chart

## Scenario

You have an existing deployment (created with `kubectl` or a previous Helm
release) and want to replace it with a new Helm chart deployment. This guide
uses `metrics-server` in `kube-system` as the running example, but the approach
applies to any workload.

## 1. Check the Current Deployment

```bash
# Is metrics-server already deployed?
kubectl get deployment metrics-server -n kube-system
kubectl get all -n kube-system -l k8s-app=metrics-server

# Is there an existing Helm release managing it?
helm list -n kube-system
helm list --all-namespaces | grep metrics-server
```

Knowing *how* it was installed decides your path:

- **kubectl-managed** (no Helm release): delete the raw resources first (step 2).
- **Helm-managed**: upgrade or reinstall the release (steps 4–5).

## 2. Remove a kubectl-Managed Deployment

If the current deployment was created with `kubectl apply`, Helm won't adopt it
cleanly — remove the raw resources first:

```bash
kubectl delete deployment metrics-server -n kube-system
kubectl delete service metrics-server -n kube-system
kubectl delete serviceaccount metrics-server -n kube-system
kubectl delete clusterrole system:metrics-server
kubectl delete clusterrolebinding system:metrics-server

# Or delete everything matching the label
kubectl delete all -n kube-system -l k8s-app=metrics-server
```

> Deleting only the metrics-server resources is safe. Never delete the
> `kube-system` namespace itself — it holds core cluster components (CoreDNS,
> kube-proxy, and others) and removing it will break the cluster.

## 3. Install a Fresh Helm Release

```bash
helm install metrics-server ./helm/metrics-server-3.13.0 \
  -f ./helm/values.yaml \
  -f ./helm/values-dev.yaml \
  --namespace kube-system \
  --create-namespace
```

## 4. Upgrade an Existing Helm Release

If a Helm release already exists, upgrade it in place. `--reset-values` drops
any previously set values and applies only the ones you pass now:

```bash
helm upgrade metrics-server ./helm/metrics-server-3.13.0 \
  -f ./helm/values.yaml \
  -f ./helm/values-dev.yaml \
  --namespace kube-system \
  --reset-values
```

`helm upgrade --install` (aka `helm upgrade -i`) is the idempotent option: it
installs if the release is absent and upgrades if it exists.

## 5. Full Reinstall (Clean Slate)

When you want to discard all prior release state and start fresh:

```bash
# Remove the existing release
helm uninstall metrics-server -n kube-system

# Reinstall from scratch
helm install metrics-server ./helm/metrics-server-3.13.0 \
  -f ./helm/values.yaml \
  -f ./helm/values-dev.yaml \
  --namespace kube-system
```

## Useful Upgrade Flags

| Flag | Effect |
|------|--------|
| `--reset-values` | Ignore previously set values; use only chart defaults + files passed now |
| `--force` | Force resource updates through replacement (use with care — can cause downtime) |
| `--wait` | Block until resources are ready before reporting success |
| `--timeout <dur>` | How long `--wait` waits, e.g. `--timeout 300s` |
| `--atomic` | Roll back automatically if the upgrade fails (implies `--wait`) |

> In **Helm 4**, `--force` was renamed to `--force-replace` and `--atomic` to
> `--rollback-on-failure` (old names still work as deprecated aliases). See
> [What's New in Helm 4](articles/helm-4-whats-new.md).

## Environment-Specific Examples

Development:

```bash
helm upgrade metrics-server ./helm/metrics-server-3.13.0 \
  -f ./helm/values.yaml \
  -f ./helm/values-dev.yaml \
  --namespace kube-system
```

Production (block until ready, bounded timeout):

```bash
helm upgrade metrics-server ./helm/metrics-server-3.13.0 \
  -f ./helm/values.yaml \
  -f ./helm/values-prd.yaml \
  --namespace kube-system \
  --wait \
  --timeout 300s
```

PCI / regulated (roll back automatically on failure):

```bash
helm upgrade metrics-server ./helm/metrics-server-3.13.0 \
  -f ./helm/values.yaml \
  -f ./helm/values-pci.yaml \
  --namespace kube-system \
  --atomic \
  --wait
```

## Verification

```bash
# Helm release state
helm status metrics-server -n kube-system
helm get values metrics-server -n kube-system

# Kubernetes resources
kubectl get deployment metrics-server -n kube-system
kubectl get pods -n kube-system -l k8s-app=metrics-server

# Functional test
kubectl top nodes
kubectl top pods -n kube-system
```

## Troubleshooting

### The upgrade fails or hangs

```bash
# Look at what's blocking
kubectl describe deployment metrics-server -n kube-system
kubectl get events -n kube-system --sort-by=.lastTimestamp | tail -20

# Check for finalizers holding a resource
kubectl get deployment metrics-server -n kube-system -o yaml | grep -A3 finalizers
```

### A resource is stuck terminating

```bash
# Force-delete a single stuck resource (targeted, not the namespace)
kubectl delete deployment metrics-server -n kube-system --force --grace-period=0
```

### Ownership conflicts (Helm won't adopt existing resources)

If Helm errors with something like "exists and cannot be imported into the
current release," the resource lacks Helm's ownership metadata. Either delete the
conflicting resource (step 2) or annotate it so Helm can adopt it:

```bash
kubectl annotate deployment metrics-server -n kube-system \
  meta.helm.sh/release-name=metrics-server \
  meta.helm.sh/release-namespace=kube-system
kubectl label deployment metrics-server -n kube-system \
  app.kubernetes.io/managed-by=Helm
```

## Best Practices

1. **Back up** the current values and manifests before overwriting
   (`helm get values ... > backup.yaml`).
2. **Test in a development** environment first.
3. **Use `--wait`** (and `--atomic`/`--rollback-on-failure`) for production.
4. **Monitor** the rollout and have a **rollback plan** (`helm rollback`) ready.
5. **Verify functionality** after the change.

## Related

- [Installing metrics-server on Kubernetes](articles/metrics-server-install.md)
- [Installing Helm Charts Without helm repo add](articles/helm-install-without-repo-add.md)
- [What's New in Helm 4](articles/helm-4-whats-new.md)

## Skills Practiced

- Determining whether a workload is kubectl- or Helm-managed
- Replacing raw kubectl resources with a Helm release safely
- Choosing between upgrade, reinstall, and `upgrade --install`
- Using `--reset-values`, `--wait`, `--atomic`, and resolving ownership conflicts
