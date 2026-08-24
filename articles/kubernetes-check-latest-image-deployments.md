# Check If Deployments Run the Latest Image

## List All Deployments With Their Images

```bash
kubectl get deployments -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: {range .spec.template.spec.containers[*]}{.image}{" "}{end}{"\n"}{end}'
```

This outputs every deployment across all namespaces alongside its container image(s) and tags.

## Filter Deployments Using the `:latest` Tag

```bash
kubectl get deployments -A -o json | jq -r '
  .items[] |
  .metadata.namespace + "/" + .metadata.name + ": " +
  ([.spec.template.spec.containers[].image] | join(", "))
' | grep -i "latest"
```

Shows only deployments explicitly using the `:latest` tag.

## Detect Image Drift (Running vs. Desired)

A pod might still run an older image even if the deployment spec references `:latest` (cached pull). Compare the actual running digest:

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: {range .status.containerStatuses[*]}{.imageID}{" "}{end}{"\n"}{end}'
```

Compare the `imageID` (sha256 digest) against what your registry currently reports as `latest` using:

```bash
# Using skopeo
skopeo inspect docker://<registry>/<image>:latest | jq '.Digest'

# Using crane
crane digest <registry>/<image>:latest
```

## Force Re-Pull With Rollout Restart

If deployments use `imagePullPolicy: Always`, a rollout restart forces Kubernetes to pull the newest image:

```bash
# Single deployment
kubectl rollout restart deployment/<name> -n <namespace>

# All deployments in a namespace
kubectl get deployments -n <namespace> -o name | xargs -I {} kubectl rollout restart {} -n <namespace>
```

## Find Deployments NOT Using `:latest` (Pinned Tags)

```bash
kubectl get deployments -A -o json | jq -r '
  .items[] |
  select(.spec.template.spec.containers[].image | test(":latest") | not) |
  .metadata.namespace + "/" + .metadata.name + ": " +
  ([.spec.template.spec.containers[].image] | join(", "))
'
```

Useful to audit which workloads use pinned/versioned tags (generally the safer practice).

## Automated Solutions

There is no built-in Kubernetes mechanism to check "is this the newest image in the registry." For continuous enforcement consider:

| Tool | Description |
|------|-------------|
| [Keel](https://keel.sh) | Lightweight automatic image updates |
| [FluxCD Image Automation](https://fluxcd.io/flux/components/image/) | GitOps-native image tracking and updates |
| [Argo CD Image Updater](https://argocd-image-updater.readthedocs.io/) | Works alongside Argo CD to update images |

## Summary

| Method | Pros | Cons |
|--------|------|------|
| `:latest` + `imagePullPolicy: Always` + restart | Simple | No auditability, requires restart |
| Compare digests (skopeo/crane) | Accurate, scriptable | Requires registry access |
| Keel / Flux / Argo Image Updater | Fully automated | Additional infrastructure to manage |

**Recommendation:** Pin image tags in production and use a GitOps tool to detect and propose updates via PRs. Use `:latest` only in dev/staging environments.
