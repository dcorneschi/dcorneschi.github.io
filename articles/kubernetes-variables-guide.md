# Kubernetes Variables Guide

How to pass configuration to containers in Kubernetes — environment variables, ConfigMaps, Secrets, and the Downward API.

## Environment Variables in Pod Specs

The simplest way to pass variables to a container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: my-app:1.0
    env:
    - name: APP_ENV
      value: "production"
    - name: LOG_LEVEL
      value: "info"
    - name: MAX_RETRIES
      value: "3"
```

> **Note:** All `value` fields must be strings. Numbers and booleans need quotes: `"3"`, `"true"`.

## Variable Expansion

Kubernetes supports `$(VAR_NAME)` syntax for referencing other environment variables:

```yaml
env:
- name: BASE_URL
  value: "https://api.example.com"
- name: HEALTH_URL
  value: "$(BASE_URL)/health"
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: LOG_PREFIX
  value: "$(POD_NAME)-app"
```

> Variables are expanded in the order they're defined. A variable can only reference variables defined above it in the list.

## ConfigMaps

ConfigMaps store non-sensitive configuration data as key-value pairs.

### Creating ConfigMaps

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_RETRIES=3

# From a file
kubectl create configmap nginx-config --from-file=nginx.conf

# From an env file
kubectl create configmap app-config --from-env-file=app.env

# From a directory (each file becomes a key)
kubectl create configmap configs --from-file=./config-dir/
```

### ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  MAX_RETRIES: "3"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      pool_size: 10
```

### Using ConfigMaps as Environment Variables

#### Single key:

```yaml
env:
- name: APP_ENV
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_ENV
```

#### All keys at once:

```yaml
envFrom:
- configMapRef:
    name: app-config
```

This creates environment variables for every key in the ConfigMap. The variable names match the key names.

#### With a prefix:

```yaml
envFrom:
- configMapRef:
    name: app-config
  prefix: CFG_
```

Creates `CFG_APP_ENV`, `CFG_LOG_LEVEL`, etc.

### Using ConfigMaps as Volume Mounts

Mount ConfigMap data as files in the container:

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

Each key becomes a file in `/etc/config/`. The file content is the value.

#### Mount a specific key as a file:

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
    items:
    - key: config.yaml
      path: app-config.yaml    # File name inside the mount
```

> **Auto-reload:** Volume-mounted ConfigMaps are updated automatically when the ConfigMap changes (with a delay of ~60s). Environment variables from ConfigMaps are NOT updated — the pod must be restarted.

## Secrets

Secrets store sensitive data (passwords, tokens, keys). They work almost identically to ConfigMaps but with base64 encoding and tighter access controls.

### Creating Secrets

```bash
# From literal values
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t

# From files
kubectl create secret generic tls-cert \
  --from-file=cert.pem \
  --from-file=key.pem

# TLS secret (special type)
kubectl create secret tls app-tls \
  --cert=tls.crt \
  --key=tls.key
```

### Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
data:
  username: YWRtaW4=          # base64 encoded "admin"
  password: czNjcjN0          # base64 encoded "s3cr3t"
```

Or use `stringData` to avoid base64 encoding yourself:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
stringData:
  username: admin
  password: s3cr3t
```

### Using Secrets as Environment Variables

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-creds
      key: password

# Or all keys at once
envFrom:
- secretRef:
    name: db-creds
```

### Using Secrets as Volume Mounts

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-creds
      defaultMode: 0400       # Restrictive file permissions
```

## Downward API

The Downward API exposes pod and container metadata as environment variables or files — no ConfigMap/Secret needed.

### Available Fields

| Source | Field | Example Value |
|--------|-------|---------------|
| Pod | `metadata.name` | `my-app-7c5ddbdf54-x2k9p` |
| Pod | `metadata.namespace` | `production` |
| Pod | `metadata.uid` | `f47ac10b-58cc-...` |
| Pod | `metadata.labels['app']` | `my-app` |
| Pod | `metadata.annotations['version']` | `1.0.0` |
| Pod | `spec.nodeName` | `ip-10-0-1-42` |
| Pod | `spec.serviceAccountName` | `my-app-sa` |
| Pod | `status.podIP` | `10.244.1.5` |
| Pod | `status.hostIP` | `10.0.1.42` |
| Container | `requests.cpu` | `100m` |
| Container | `requests.memory` | `128Mi` |
| Container | `limits.cpu` | `500m` |
| Container | `limits.memory` | `256Mi` |

### Using Downward API as Environment Variables

```yaml
env:
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: POD_NAMESPACE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace
- name: POD_IP
  valueFrom:
    fieldRef:
      fieldPath: status.podIP
- name: NODE_NAME
  valueFrom:
    fieldRef:
      fieldPath: spec.nodeName
- name: CPU_REQUEST
  valueFrom:
    resourceFieldRef:
      containerName: app
      resource: requests.cpu
- name: MEM_LIMIT
  valueFrom:
    resourceFieldRef:
      containerName: app
      resource: limits.memory
```

### Using Downward API as Volume Files

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "cpu_request"
        resourceFieldRef:
          containerName: app
          resource: requests.cpu
```

## Comparison: When to Use What

| Method | Use Case | Updates Without Restart |
|--------|----------|----------------------|
| `env` with `value` | Simple, static values | No |
| ConfigMap (env) | Shared config across pods | No |
| ConfigMap (volume) | Config files, large values | Yes (~60s delay) |
| Secret (env) | Passwords, tokens in env vars | No |
| Secret (volume) | Certificates, key files | Yes (~60s delay) |
| Downward API | Pod identity, resource info | N/A (reflects current state) |

## Optional and Required References

By default, referencing a missing ConfigMap or Secret prevents the pod from starting. Make it optional:

```yaml
env:
- name: OPTIONAL_VAR
  valueFrom:
    configMapKeyRef:
      name: maybe-exists
      key: some-key
      optional: true          # Pod starts even if ConfigMap doesn't exist

envFrom:
- configMapRef:
    name: maybe-exists
    optional: true
```

## Immutable ConfigMaps and Secrets

Mark a ConfigMap or Secret as immutable to prevent accidental changes and improve cluster performance (kubelet stops watching for updates):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
immutable: true
```

> Once set to `immutable: true`, you can't change it back or modify the data. You must delete and recreate it.

## Useful Commands

```bash
# List ConfigMaps
kubectl get configmaps -n <namespace>

# View ConfigMap contents
kubectl get configmap <name> -o yaml

# Edit a ConfigMap in-place
kubectl edit configmap <name>

# View Secret contents (decoded)
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 -d

# View all keys in a Secret
kubectl get secret <name> -o json | jq '.data | keys'

# Check what env vars a running pod sees
kubectl exec <pod-name> -- env | sort

# Check if a pod references a missing ConfigMap/Secret
kubectl describe pod <pod-name> | grep -A 5 "Warning"
```
