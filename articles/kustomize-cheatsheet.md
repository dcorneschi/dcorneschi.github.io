# Kustomize Cheatsheet

Commands, one-liners, and tips for working with Kustomize — building overlays, patching resources, and managing Kubernetes manifests without templates.

## Core Commands

```bash
# Build (render) kustomization to stdout
kubectl kustomize .
kubectl kustomize overlays/prod

# Apply directly
kubectl apply -k .
kubectl apply -k overlays/prod

# Diff against live cluster
kubectl diff -k overlays/prod

# Delete resources managed by kustomization
kubectl delete -k overlays/prod

# Build with standalone kustomize binary (supports more features)
kustomize build .
kustomize build overlays/prod

# Build with Helm chart support (requires kustomize binary, not kubectl)
kustomize build --enable-helm .

# Output to file
kubectl kustomize overlays/prod > rendered.yaml
```

## Project Structure

```
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patches/
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patches/
    └── prod/
        ├── kustomization.yaml
        └── patches/
```

## kustomization.yaml Reference

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Include base or other resources
resources:
- ../../base
- extra-resource.yaml
- https://github.com/org/repo/path?ref=v1.0.0   # remote base

# Set namespace for all resources
namespace: production

# Add prefix/suffix to all resource names
namePrefix: prod-
nameSuffix: -v2

# Add labels to all resources
commonLabels:
  env: production
  team: platform

# Add annotations to all resources
commonAnnotations:
  owner: platform-team
  managed-by: kustomize

# Override images
images:
- name: my-app
  newName: registry.example.com/my-app
  newTag: v1.2.3
- name: nginx
  newTag: 1.25-alpine

# Generate ConfigMaps
configMapGenerator:
- name: app-config
  literals:
  - DB_HOST=prod-db.example.com
  - LOG_LEVEL=warn
- name: nginx-config
  files:
  - nginx.conf
- name: env-config
  envs:
  - config.env

# Generate Secrets
secretGenerator:
- name: db-creds
  literals:
  - username=admin
  - password=supersecret
- name: tls-cert
  files:
  - tls.crt
  - tls.key
  type: kubernetes.io/tls

# Patches (multiple formats)
patches:
- path: patches/replicas.yaml          # file-based patch
- path: patches/resources.yaml
  target:                               # target-specific patch
    kind: Deployment
    name: my-app
- target:                               # inline JSON patch
    kind: Deployment
    name: my-app
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5

# Replace existing resources entirely
replacements:
- source:
    kind: ConfigMap
    name: app-config
    fieldPath: data.DB_HOST
  targets:
  - select:
      kind: Deployment
    fieldPaths:
    - spec.template.spec.containers.[name=app].env.[name=DB_HOST].value
```

## Image Overrides

```yaml
# kustomization.yaml
images:
# Change tag only
- name: my-app
  newTag: v2.0.0

# Change registry + tag
- name: my-app
  newName: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app
  newTag: abc123

# Change to digest
- name: my-app
  newName: my-app
  digest: sha256:abc123def456...
```

One-liner to set image from CLI:

```bash
cd overlays/prod && kustomize edit set image my-app=my-app:v2.0.0
```

## Patch Types

### Strategic Merge Patch (Default)

Merges fields into existing resources. Good for adding/modifying fields:

```yaml
# patches/increase-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: app
        resources:
          limits:
            cpu: "2"
            memory: 1Gi
```

### JSON Patch (Precise Operations)

Use for arrays, removing fields, or exact path operations:

```yaml
# kustomization.yaml
patches:
- target:
    kind: Deployment
    name: my-app
  patch: |-
    - op: add
      path: /spec/template/spec/containers/0/args/-
      value: --verbose
    - op: replace
      path: /spec/replicas
      value: 10
    - op: remove
      path: /spec/template/spec/containers/0/resources/limits
```

### JSON Patch Operations

| Operation | Path | Effect |
|-----------|------|--------|
| `add` | `/spec/template/metadata/annotations/key` | Add new field |
| `add` | `/spec/template/spec/containers/0/args/-` | Append to array |
| `replace` | `/spec/replicas` | Replace existing value |
| `remove` | `/spec/template/spec/containers/0/resources` | Delete field |
| `copy` | from + path | Copy value between paths |
| `move` | from + path | Move value between paths |

### Patch Targeting (Apply to Multiple Resources)

```yaml
patches:
# All Deployments
- target:
    kind: Deployment
  patch: |-
    - op: add
      path: /metadata/annotations/managed-by
      value: kustomize

# Specific name pattern (regex)
- target:
    kind: Deployment
    name: ".*-api"
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 3

# By label selector
- target:
    kind: Service
    labelSelector: "tier=frontend"
  patch: |-
    - op: replace
      path: /spec/type
      value: LoadBalancer
```

## ConfigMap and Secret Generators

### From Literals

```yaml
configMapGenerator:
- name: app-config
  literals:
  - DATABASE_URL=postgres://prod-db:5432/myapp
  - REDIS_HOST=redis.cache.svc
  - LOG_LEVEL=warn
```

### From Files

```yaml
configMapGenerator:
- name: nginx-config
  files:
  - configs/nginx.conf
  - configs/mime.types

# With custom key names
- name: app-scripts
  files:
  - entrypoint.sh=scripts/start.sh
```

### From Env File

```yaml
configMapGenerator:
- name: env-config
  envs:
  - .env.production
```

### Behavior Options

```yaml
configMapGenerator:
- name: app-config
  behavior: merge      # merge with existing ConfigMap from base
  literals:
  - NEW_KEY=new-value

# Options: create (default), replace, merge
```

### Disable Hash Suffix

By default, Kustomize appends a hash to generated ConfigMap/Secret names. To disable:

```yaml
generatorOptions:
  disableNameSuffixHash: true
```

## Common One-Liners

```bash
# Set image tag from CLI
cd overlays/prod && kustomize edit set image my-app=my-app:v2.0.0

# Set namespace from CLI
cd overlays/prod && kustomize edit set namespace production

# Add a resource from CLI
cd overlays/prod && kustomize edit add resource extra-deploy.yaml

# Add a label from CLI
cd overlays/prod && kustomize edit add label env:production

# Add an annotation from CLI
cd overlays/prod && kustomize edit add annotation owner:platform-team

# Add a patch from CLI
cd overlays/prod && kustomize edit add patch --path patches/replicas.yaml

# Add a configmap generator literal
cd overlays/prod && kustomize edit add configmap app-config --from-literal=KEY=value

# Build and apply in one shot
kustomize build overlays/prod | kubectl apply -f -

# Build, diff, then apply
kubectl diff -k overlays/prod && kubectl apply -k overlays/prod

# Render and validate with kubeconform
kustomize build overlays/prod | kubeconform -strict

# Count resources in output
kubectl kustomize overlays/prod | grep "^kind:" | sort | uniq -c

# Find which overlay changed an image
grep -r "newTag" overlays/
```

## Tips and Tricks

### Use Components for Shared Patches

Components are reusable patch sets that can be included in multiple overlays:

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

patches:
- target:
    kind: Deployment
  patch: |-
    - op: add
      path: /spec/template/metadata/annotations/prometheus.io~1scrape
      value: "true"
    - op: add
      path: /spec/template/metadata/annotations/prometheus.io~1port
      value: "9090"
```

```yaml
# overlays/prod/kustomization.yaml
components:
- ../../components/monitoring
- ../../components/security
```

### Override a Single Environment Variable

```yaml
patches:
- target:
    kind: Deployment
    name: my-app
  patch: |-
    - op: add
      path: /spec/template/spec/containers/0/env/-
      value:
        name: LOG_LEVEL
        value: debug
```

### Add a Sidecar Container

```yaml
# patches/add-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: log-shipper
        image: fluentd:v1.16
        volumeMounts:
        - name: logs
          mountPath: /var/log/app
      volumes:
      - name: logs
        emptyDir: {}
```

### Inline Patch to Add Tolerations

```yaml
patches:
- target:
    kind: Deployment
  patch: |-
    - op: add
      path: /spec/template/spec/tolerations
      value:
        - key: "dedicated"
          operator: "Equal"
          value: "app"
          effect: "NoSchedule"
```

### Remove a Field

```yaml
patches:
- target:
    kind: Deployment
    name: my-app
  patch: |-
    - op: remove
      path: /spec/template/spec/containers/0/resources/limits
```

### Use Remote Bases (Git)

```yaml
resources:
- https://github.com/org/k8s-base//deploy?ref=v1.5.0
```

### Conditional Inclusion (No If/Else, Use Overlays)

Kustomize has no conditionals. Instead, structure overlays to include/exclude files:

```yaml
# overlays/prod/kustomization.yaml (includes HPA)
resources:
- ../../base
- hpa.yaml

# overlays/dev/kustomization.yaml (no HPA)
resources:
- ../../base
```

### Validate Before Applying

```bash
# Dry-run against the cluster
kubectl apply -k overlays/prod --dry-run=server

# Validate YAML structure
kustomize build overlays/prod | kubeval

# Validate with kubeconform (faster, more up-to-date)
kustomize build overlays/prod | kubeconform -strict -summary
```

## Debugging

```bash
# See what kustomize generates (full rendered output)
kubectl kustomize overlays/prod

# Check which resources are included
kubectl kustomize overlays/prod | grep "^kind:" | sort | uniq -c

# Check which images are in the output
kubectl kustomize overlays/prod | grep "image:" | sort -u

# Diff between two overlays
diff <(kubectl kustomize overlays/dev) <(kubectl kustomize overlays/prod)

# Find syntax errors
kustomize build overlays/prod 2>&1 | head -20

# Check kustomization.yaml validity
kustomize cfg tree overlays/prod
```

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Array replacement instead of append | Original args/env/volumes disappear | Use JSON patch with `/-` to append |
| Wrong patch target name | Patch silently ignored | Check `metadata.name` matches exactly |
| Missing resource in kustomization.yaml | Resource not in output | Add to `resources:` list |
| Tab instead of spaces in YAML | Parse error | Use spaces only |
| Forgot `|-` on inline patches | YAML parsing error | Add `|-` after `patch:` |
| Hash suffix breaks references | Service can't find ConfigMap | Use `generatorOptions.disableNameSuffixHash: true` or let Kustomize update refs |
| Strategic merge replaces arrays | List items lost | Use JSON patch for arrays |
| Path with special chars in JSON patch | Patch fails | Escape `/` as `~1` and `~` as `~0` |
