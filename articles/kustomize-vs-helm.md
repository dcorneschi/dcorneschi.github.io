# Kustomize vs Helm

Two approaches to managing Kubernetes manifests across environments — when to use each, how they work, and how to combine them.

## Overview

| | Kustomize | Helm |
|---|---|---|
| **Approach** | Patch and overlay plain YAML | Template engine with Go templates |
| **Philosophy** | "Keep YAML as YAML" — no templating | "Charts are packages" — parameterized templates |
| **Built into kubectl** | Yes (`kubectl apply -k`) | No (separate `helm` binary) |
| **Packaging** | No concept of packages | Charts (versioned, distributable) |
| **Dependencies** | None | Chart dependencies (subcharts) |
| **Lifecycle management** | None (just generates YAML) | Releases with install/upgrade/rollback/uninstall |
| **State tracking** | None | Release history stored as Secrets in cluster |
| **Complexity** | Low (overlays + patches) | Medium-High (templates, values, helpers, hooks) |
| **Learning curve** | Low | Medium |
| **Best for** | In-house apps, small variations between envs | Third-party software, complex parameterization |

## How Kustomize Works

Kustomize takes a base set of YAML files and applies overlays (patches, additions, transformations) to produce environment-specific manifests. No templating — just plain YAML in, modified YAML out.

```
base/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml

overlays/
├── dev/
│   ├── kustomization.yaml
│   └── replica-patch.yaml
├── staging/
│   ├── kustomization.yaml
│   └── resource-patch.yaml
└── prod/
    ├── kustomization.yaml
    ├── replica-patch.yaml
    └── hpa.yaml
```

### Base

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
```

### Overlay (Production)

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
- hpa.yaml

namespace: production

patches:
- path: replica-patch.yaml

images:
- name: my-app
  newTag: v1.2.3

commonLabels:
  env: production
```

```yaml
# overlays/prod/replica-patch.yaml
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
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
```

### Apply

```bash
# Preview what will be applied
kubectl kustomize overlays/prod

# Apply directly
kubectl apply -k overlays/prod

# Diff against live cluster
kubectl diff -k overlays/prod
```

## How Helm Works

Helm uses Go templates to generate YAML from parameterized charts. Charts are packages that can be versioned, shared, and installed as releases.

```
my-chart/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl
│   └── NOTES.txt
└── charts/           # dependencies
```

### Chart Template

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
```

### Values

```yaml
# values.yaml (defaults)
replicaCount: 1
image:
  repository: my-app
  tag: latest
service:
  port: 8080
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

```yaml
# values-prod.yaml (production overrides)
replicaCount: 5
image:
  tag: v1.2.3
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

### Install/Upgrade

```bash
# Install a release
helm install my-app ./my-chart -f values-prod.yaml -n production

# Upgrade
helm upgrade my-app ./my-chart -f values-prod.yaml -n production

# Rollback
helm rollback my-app 1 -n production

# Preview (dry-run)
helm template my-app ./my-chart -f values-prod.yaml

# Diff (requires helm-diff plugin)
helm diff upgrade my-app ./my-chart -f values-prod.yaml
```

## Key Differences in Detail

### Templating vs Patching

**Helm** — Templates with logic (if/else, loops, functions):

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-app.fullname" . }}
spec:
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
{{- end }}
```

**Kustomize** — Patches that modify existing YAML (no logic):

```yaml
# Just add/remove the HPA file in the overlay
# overlays/prod/kustomization.yaml
resources:
- ../../base
- hpa.yaml    # Include HPA only in prod
```

### Release Management

**Helm** tracks releases:

```bash
# List releases
helm list -A

# History of a release
helm history my-app -n production

# Rollback to previous version
helm rollback my-app 2 -n production

# Uninstall (removes all resources)
helm uninstall my-app -n production
```

**Kustomize** has no release concept. It just generates YAML — lifecycle is managed by `kubectl apply` or a GitOps tool (ArgoCD, Flux).

### Third-Party Software

**Helm** excels here — most open-source tools provide official Helm charts:

```bash
# Install nginx-ingress
helm install ingress-nginx ingress-nginx/ingress-nginx -f my-values.yaml

# Install Prometheus stack
helm install monitoring prometheus-community/kube-prometheus-stack -f monitoring-values.yaml

# Install ArgoCD
helm install argocd argo/argo-cd -f argocd-values.yaml
```

**Kustomize** can consume Helm charts but it's less natural:

```yaml
# kustomization.yaml
helmCharts:
- name: ingress-nginx
  repo: https://kubernetes.github.io/ingress-nginx
  version: 4.7.1
  releaseName: ingress-nginx
  valuesFile: values.yaml
```

## When to Use Each

### Use Kustomize When

- Your manifests are simple with small env-to-env differences (replicas, images, resource limits)
- You want to keep plain YAML readable without template syntax
- You're managing in-house applications (not distributing charts)
- You're using GitOps (ArgoCD/Flux support Kustomize natively)
- You want minimal tooling (built into kubectl)
- You need to patch third-party Helm output without forking the chart

### Use Helm When

- You're installing third-party software (nginx, Prometheus, ArgoCD, cert-manager)
- You need complex parameterization (conditional resources, loops, computed values)
- You want release lifecycle management (install, upgrade, rollback, history)
- You're distributing your app as a reusable package to others
- You need chart dependencies (app chart depends on Redis subchart)
- You need hooks (pre-install, post-upgrade jobs)

### Use Both Together

The most common production pattern:

```
# Third-party tools — install with Helm
helm install argocd argo/argo-cd -f argocd-values.yaml

# In-house apps — deploy with Kustomize (via ArgoCD)
argocd app create my-app \
  --repo https://github.com/org/k8s-manifests \
  --path overlays/prod \
  --dest-server https://kubernetes.default.svc
```

Or use Kustomize to patch Helm output:

```yaml
# kustomization.yaml
helmCharts:
- name: my-chart
  repo: https://my-registry.com/charts
  version: 1.0.0
  valuesFile: values.yaml

patches:
- target:
    kind: Deployment
    name: my-chart-app
  patch: |
    - op: add
      path: /spec/template/metadata/annotations/custom
      value: "added-by-kustomize"
```

## Comparison by Task

| Task | Kustomize | Helm |
|------|-----------|------|
| Change image tag per env | `images:` in kustomization.yaml | `image.tag` in values |
| Change replicas per env | Strategic merge patch | `replicaCount` in values |
| Add a resource in one env | Add to overlay `resources:` | `{{- if .Values.xxx }}` conditional |
| Change namespace | `namespace:` in kustomization.yaml | `--namespace` flag or values |
| Add labels/annotations | `commonLabels:` / `commonAnnotations:` | Template helpers |
| Generate ConfigMaps | `configMapGenerator:` | `tpl` function or values |
| Generate Secrets | `secretGenerator:` | Separate secret management |
| Conditional resources | Include/exclude files per overlay | `if/else` in templates |
| Loops | Not supported | `range` in templates |
| Append to an array | JSON patch only (`op: add, path: .../-`) | Template logic or values merge |
| Rollback | `git revert` + reapply | `helm rollback` |

## Kustomize Merge Behavior (Key Gotcha)

### Objects/Maps — MERGES fields

```yaml
# Base
metadata:
  labels:
    app: myapp
    version: v1

# Patch
metadata:
  labels:
    environment: dev

# Result: MERGED (all fields kept)
metadata:
  labels:
    app: myapp
    version: v1
    environment: dev
```

### Arrays/Lists — REPLACES entire array

```yaml
# Base
args:
  - --secure-port=10250
  - --cert-dir=/tmp
  - --metric-resolution=15s

# Patch
args:
  - --kubelet-insecure-tls

# Result: REPLACED (all original args lost!)
args:
  - --kubelet-insecure-tls
```

This is the most common Kustomize surprise. You can't append a single item to an array with a strategic merge patch — you must include the entire array.

### Solutions for Array Patching

**Option 1: Strategic merge — include the full array:**

```yaml
# deployment-patch.yaml (must repeat all args + add new one)
spec:
  template:
    spec:
      containers:
      - name: metrics-server
        args:
        - --secure-port=10250
        - --cert-dir=/tmp
        - --metric-resolution=15s
        - --kubelet-insecure-tls    # new arg added
```

**Option 2: JSON patch — append without repeating (recommended):**

```yaml
# kustomization.yaml
patches:
- target:
    kind: Deployment
    name: metrics-server
  patch: |-
    - op: add
      path: /spec/template/spec/containers/0/args/-
      value: --kubelet-insecure-tls
```

The `/-` at the end of the path means "append to the array." This is the closest Kustomize gets to Helm's flexibility for single-value changes.

**Option 3: Use Helm instead (if array manipulation is frequent):**

```bash
helm install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --set 'args={--kubelet-insecure-tls}'
```

### Why This Limitation Exists

| Aspect | Helm | Kustomize |
|--------|------|-----------|
| Philosophy | Generate YAML from templates | Patch existing YAML |
| Array handling | Custom template logic (loops, merge) | YAML merge semantics (replace) |
| Single value to array | Supported (template logic) | Only via JSON patch |
| Complexity | Higher (must author templates) | Lower (direct patches) |

Kustomize deliberately avoids templating to keep things simple. The trade-off is less flexibility for complex merging — particularly arrays.

## Common Kustomize Features

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

namespace: my-namespace

namePrefix: prod-
nameSuffix: -v2

commonLabels:
  team: platform
  env: production

commonAnnotations:
  owner: platform-team

images:
- name: my-app
  newName: registry.example.com/my-app
  newTag: v1.2.3

configMapGenerator:
- name: app-config
  literals:
  - DB_HOST=prod-db.example.com
  - LOG_LEVEL=warn

secretGenerator:
- name: db-creds
  literals:
  - password=supersecret

patches:
- path: increase-replicas.yaml
- target:
    kind: Deployment
    name: my-app
  patch: |
    - op: replace
      path: /spec/replicas
      value: 5
```

## Common Helm Features

```yaml
# values.yaml
global:
  env: production

replicaCount: 3

image:
  repository: my-app
  tag: v1.2.3
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  hosts:
  - host: app.example.com
    paths:
    - path: /
      pathType: Prefix

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilization: 70

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

## Useful Commands

### Kustomize

```bash
# Build (render) without applying
kubectl kustomize overlays/prod

# Apply
kubectl apply -k overlays/prod

# Diff against live
kubectl diff -k overlays/prod

# Delete resources managed by kustomize
kubectl delete -k overlays/prod

# Build with Helm chart support (requires kustomize binary, not kubectl)
kustomize build --enable-helm overlays/prod
```

### Helm

```bash
# Search for charts
helm search repo nginx
helm search hub prometheus

# Show chart values (defaults)
helm show values ingress-nginx/ingress-nginx

# Install
helm install <release> <chart> -f values.yaml -n <namespace> --create-namespace

# Upgrade
helm upgrade <release> <chart> -f values.yaml -n <namespace>

# Dry-run (preview)
helm template <release> <chart> -f values.yaml

# List releases
helm list -A

# Release history
helm history <release> -n <namespace>

# Rollback
helm rollback <release> <revision> -n <namespace>

# Uninstall
helm uninstall <release> -n <namespace>

# Diff (plugin)
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade <release> <chart> -f values.yaml
```
