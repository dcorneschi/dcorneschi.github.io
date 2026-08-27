# Kubernetes Objects vs Resources vs Custom Resources

Understanding the difference between Kubernetes objects, API resources, and custom resources — what they are, how they relate, and how they map to the API server.

## What is a Kubernetes Object?

An object is anything a user creates and persists in the cluster. Pods, Deployments, Secrets, ConfigMaps, Namespaces — all objects.

Objects are stored in etcd under the `/registry` directory:

```
/registry/pods/default/nginx
/registry/deployments/production/my-app
/registry/secrets/kube-system/coredns-token
/registry/namespaces/default
```

You declare objects using a spec (YAML or JSON) that describes the desired state. Kubernetes continuously reconciles actual state to match.

### Object Specification Structure

Every object spec shares these common fields:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver-pod
  namespace: default
  labels:
    app: web
  annotations:
    description: "Frontend webserver"
spec:
  containers:
  - name: webserver
    image: nginx:latest
    ports:
    - containerPort: 80
```

| Field | Description |
|-------|-------------|
| `apiVersion` | API group and version (e.g., `v1`, `apps/v1`, `batch/v1`) |
| `kind` | Object type — Pod, Deployment, Service, ConfigMap, etc. |
| `metadata` | Identity: name, namespace, labels, annotations, finalizers, UID |
| `spec` | Desired state and characteristics of the object |

### Object UID

Every object gets a universally unique identifier (UID). No two objects in a cluster share the same UID.

Name uniqueness is scoped by kind + namespace:
- You **cannot** have two Pods named `webserver` in the same namespace
- You **can** have a Pod named `webserver` and a Deployment named `webserver` in the same namespace

```bash
# View an object's UID
kubectl get pod <pod-name> -o jsonpath='{.metadata.uid}'
```

## Native Kubernetes Object Categories

| Category | Objects |
|----------|---------|
| Workload | Pods, ReplicaSets, Deployments, StatefulSets, DaemonSets, Jobs, CronJobs, HPA, VPA |
| Service & Networking | Services, Ingress, IngressClasses, NetworkPolicies, Endpoints, EndpointSlices |
| Storage | PersistentVolumes, PersistentVolumeClaims, StorageClasses |
| Configuration & Management | ConfigMaps, Namespaces, ResourceQuotas, LimitRanges, PodDisruptionBudgets, PriorityClasses |
| Security | Secrets, ServiceAccounts, Roles, RoleBindings, ClusterRoles, ClusterRoleBindings |
| Metadata | Labels, Selectors, Annotations, Finalizers |

```bash
# List all object types available in your cluster
kubectl api-resources

# List with API group information
kubectl api-resources --api-group=apps
kubectl api-resources --api-group=batch
```

## What is a Kubernetes Resource?

A resource is an API endpoint that the API server exposes for a specific object type. It's the URL you hit to create, read, update, or delete objects.

**Object** = the thing stored in etcd  
**Resource** = the API endpoint used to manage that object

### API Resource Endpoints

```
GET    /api/v1/namespaces                              → list all namespaces
GET    /api/v1/pods                                    → list all pods (all namespaces)
GET    /api/v1/namespaces/{ns}/pods                    → list pods in a namespace
GET    /api/v1/namespaces/{ns}/pods/{name}             → get a specific pod
POST   /api/v1/namespaces/{ns}/pods                    → create a pod
DELETE /api/v1/namespaces/{ns}/pods/{name}             → delete a pod
GET    /apis/apps/v1/namespaces/{ns}/deployments       → list deployments
GET    /apis/apps/v1/namespaces/{ns}/deployments/{name} → get a specific deployment
```

### How kubectl Uses Resources

When you run `kubectl apply -f pod.yaml`:

1. kubectl reads the YAML and identifies `kind: Pod`, `apiVersion: v1`
2. kubectl converts the YAML to JSON
3. kubectl sends a POST request to `/api/v1/namespaces/default/pods`
4. The API server validates, admits, and persists the object in etcd

```bash
# See what API endpoint kubectl would hit (increase verbosity)
kubectl get pods -v=6

# Discover API resources and their endpoints
kubectl api-resources -o wide

# Direct API call via kubectl proxy
kubectl proxy &
curl http://localhost:8001/api/v1/namespaces/default/pods
```

### Resource vs Object — Key Distinction

| Term | Meaning | Example |
|------|---------|---------|
| Object | A record in etcd representing desired + actual state | A Pod named `nginx` running in `default` namespace |
| Resource | An API endpoint for managing objects of a given type | `/api/v1/namespaces/default/pods` |
| Kind | The object type as declared in YAML | `Pod`, `Deployment`, `Service` |

In casual usage people say "resource" and "object" interchangeably. But in RBAC rules and API machinery, the distinction matters:

```yaml
# RBAC example — "resources" here means API resources
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]        # <-- API resource name
  verbs: ["get", "list", "watch"]
```

## What are Custom Resources?

Custom Resources (CRs) extend the Kubernetes API with your own object types. You define a new kind, register it via a CustomResourceDefinition (CRD), and build a controller to act on it.

### Example: Backup Custom Resource

CRD definition:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.devopscube.com
spec:
  group: devopscube.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              etcdEndpoint:
                type: string
              s3Bucket:
                type: string
              s3Region:
                type: string
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    kind: Backup
    shortNames:
    - bk
```

Custom object using that CRD:

```yaml
apiVersion: devopscube.com/v1
kind: Backup
metadata:
  name: my-backup
spec:
  etcdEndpoint: http://etcd:2379
  s3Bucket: my-bucket
  s3Region: us-west-2
```

Once applied, Kubernetes creates a new API endpoint:

```
/apis/devopscube.com/v1/namespaces/{ns}/backups
```

A custom controller watches for Backup objects and performs the actual backup logic.

### Common CRD Examples in the Wild

| Project | CRD Kind | Purpose |
|---------|----------|---------|
| cert-manager | `Certificate`, `Issuer` | TLS certificate automation |
| ArgoCD | `Application`, `AppProject` | GitOps deployments |
| Prometheus Operator | `ServiceMonitor`, `PrometheusRule` | Monitoring configuration |
| Istio | `VirtualService`, `DestinationRule` | Service mesh routing |
| Karpenter | `NodePool`, `EC2NodeClass` | Node autoscaling |

```bash
# List all CRDs in your cluster
kubectl get crds

# Describe a specific CRD
kubectl describe crd certificates.cert-manager.io

# List custom objects
kubectl get backups
kubectl get certificates -A
```

## Kubernetes Manifests

A manifest is a YAML file containing one or more object specs. Multiple objects can be separated by `---`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: webserver
spec:
  selector:
    app: webserver
  ports:
  - port: 80
    targetPort: 80
```

- No hard limit on the number of objects per manifest
- etcd has a default 1.5 MB request size limit per object
- Keep manifests focused and manageable — group related objects together

```bash
# Apply a manifest
kubectl apply -f manifest.yaml

# Apply all manifests in a directory
kubectl apply -f ./k8s/

# Dry-run to see what would be created
kubectl apply -f manifest.yaml --dry-run=client -o yaml
```

## Quick Reference

```bash
# List all API resources (native + custom)
kubectl api-resources

# List only namespaced resources
kubectl api-resources --namespaced=true

# List only cluster-scoped resources
kubectl api-resources --namespaced=false

# Show API versions
kubectl api-versions

# Explain any object's spec fields
kubectl explain pod.spec
kubectl explain deployment.spec.strategy
kubectl explain --recursive pod.spec.containers

# Get object as YAML (see full spec + status)
kubectl get pod <name> -o yaml
kubectl get deployment <name> -o yaml

# List all CRDs
kubectl get crds

# Check what API group an object belongs to
kubectl api-resources | grep -i deployment
```
