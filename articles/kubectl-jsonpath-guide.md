<p align="center">
  <img src="/articles/images/kubernetes-logo.svg" alt="Kubernetes logo" width="200"/>
</p>

<h1 align="center">kubectl JSONPath Guide</h1>

JSONPath is built into kubectl — no extra tools needed. Use `-o jsonpath='{expression}'` to extract and format data from Kubernetes resources.

## Syntax

```bash
kubectl get <resource> -o jsonpath='{<expression>}'
```

### Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `$` | Root object (implicit in kubectl) | `{.items}` |
| `@` | Current object (in filters) | `{?(@.field)}` |
| `.field` | Child field | `{.metadata.name}` |
| `['field']` | Bracket notation | `{['metadata']['name']}` |
| `[*]` | All array elements | `{.items[*].metadata.name}` |
| `[0]` | First element | `{.items[0]}` |
| `[0,1]` | Multiple elements | `{.items[0,1]}` |
| `[?(@.expr)]` | Filter expression | `{.items[?(@.status.phase=="Running")]}` |
| `..field` | Recursive descent | `{..image}` |
| `{"\n"}` | Newline | Used inside `range` |
| `{"\t"}` | Tab | Used inside `range` |
| `range`/`end` | Iterate and format | `{range .items[*]}...{end}` |

### Filter Expressions

```bash
[?(@.field=="value")]       # Field equals value
[?(@.field!="value")]       # Field not equals
[?(@.field>5)]              # Numeric comparison
[?(@.field)]                # Field exists
[?(!@.field)]               # Field doesn't exist
```

## Formatting Output

### Default (Space-Separated)

```bash
# All names on one line, space-separated
kubectl get pods -n <namespace> -o jsonpath='{.items[*].metadata.name}'
# pod1 pod2 pod3

# With trailing newline (cleaner terminal output)
kubectl get pods -n <namespace> -o jsonpath='{.items[*].metadata.name}{"\n"}'
```

### One Per Line (range)

```bash
# One name per line
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# Name and status (tab-separated)
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

### Custom Headers

```bash
# Headers + data using $'' syntax
kubectl get nodes -o jsonpath=$'NAME\tSTATUS\tVERSION\n{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'
```

### Custom Separators

```bash
# Namespace/Pod,IP format
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{","}{.status.podIP}{"\n"}{end}'

# Arrow separator
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.status.phase}{" -> "}{.spec.nodeName}{"\n"}{end}'

# Namespace:Pod format
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.namespace}{":"}{.metadata.name}{" -> "}{.status.podIP}{"\n"}{end}'
```

### Space-Separated to Newlines (tr trick)

```bash
kubectl get pods -n <namespace> -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n'
```

### CSV Export

```bash
echo "Name,Status,IP,Image" > pods.csv
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{","}{.status.phase}{","}{.status.podIP}{","}{.spec.containers[*].image}{"\n"}{end}' >> pods.csv
```

### Column-Aligned Tables

```bash
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.spec.containers[*].image}{"\t"}{.status.podIP}{"\n"}{end}' | column -t
```

## Pods

```bash
# Pod names
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# Pod names and namespace
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{end}'

# Pod IPs
kubectl get pods -n <namespace> -o jsonpath='{.items[*].status.podIP}'

# Pod phase/status
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Pods and their nodes
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'

# Pod creation timestamps
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}'

# Pod restart counts
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].restartCount}{"\n"}{end}'

# Container images
kubectl get pods -n <namespace> -o jsonpath='{.items[*].spec.containers[*].image}'

# Pod phase summary (pipe to uniq -c)
kubectl get pods -n <namespace> -o jsonpath='{.items[*].status.phase}' | tr ' ' '\n' | sort | uniq -c
```

### Filter Pods

```bash
# Running pods
kubectl get pods -n <namespace> -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'

# Failed pods
kubectl get pods -n <namespace> -o jsonpath='{.items[?(@.status.phase=="Failed")].metadata.name}'

# Not running
kubectl get pods -n <namespace> -o jsonpath='{.items[?(@.status.phase!="Running")].metadata.name}'

# Pods with specific label
kubectl get pods -n <namespace> -o jsonpath='{.items[?(@.metadata.labels.app=="myapp")].metadata.name}'

# Pods with restarts > 0
kubectl get pods -n <namespace> -o jsonpath='{.items[?(@.status.containerStatuses[0].restartCount>0)].metadata.name}'

# Pods with high restarts (> 5)
kubectl get pods -n <namespace> -o jsonpath='{.items[?(@.status.containerStatuses[0].restartCount>5)].metadata.name}'
```

### Container Details (Nested Range)

```bash
# Container names and images per pod
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}:{.image}{" "}{end}{"\n"}{end}'

# Container ports per pod
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*].ports[*]}{.containerPort}{" "}{end}{"\n"}{end}'

# Container resource requests per pod
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.resources.requests.cpu}{" "}{.resources.requests.memory}{" "}{end}{"\n"}{end}'

# Container resource limits per pod
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.resources.limits.cpu}{" "}{.resources.limits.memory}{" "}{end}{"\n"}{end}'

# All pod conditions (type:status pairs)
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.conditions[*]}{.type}{":"}{.status}{" "}{end}{"\n"}{end}'
```

### Readiness and Health

```bash
# Pod readiness status
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Container ready state
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[*].ready}{"\n"}{end}'

# Non-running pods with waiting reason
kubectl get pods -n <namespace> -o jsonpath='{range .items[?(@.status.phase!="Running")]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.status.containerStatuses[*].state.waiting.reason}{"\n"}{end}'

# Failed pods with exit codes
kubectl get pods -n <namespace> -o jsonpath='{range .items[?(@.status.phase=="Failed")]}{.metadata.name}{"\t"}{.status.containerStatuses[*].state.terminated.exitCode}{"\n"}{end}'
```

## Nodes

```bash
# Node names
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'

# Node names and Ready status
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Node internal IPs
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Node external IPs
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# All node IPs with types
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.addresses[*]}{.type}{":"}{.address}{" "}{end}{"\n"}{end}'

# Node capacity (CPU and memory)
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.cpu}{"\t"}{.status.capacity.memory}{"\n"}{end}'

# Node allocatable resources
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.cpu}{"\t"}{.status.allocatable.memory}{"\n"}{end}'

# Node OS, kernel, and container runtime
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.osImage}{"\t"}{.status.nodeInfo.kernelVersion}{"\n"}{end}'

# Node kubelet version
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'

# Node architecture
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.architecture}{"\n"}{end}'

# Node roles
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.kubernetes\.io/role}{"\n"}{end}'

# Node taints (key:effect)
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.taints[*]}{.key}{":"}{.effect}{" "}{end}{"\n"}{end}'

# Specific label (escaped dots)
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.kubernetes\.io/os}{"\n"}{end}'

# Ready nodes only
kubectl get nodes -o jsonpath='{.items[?(@.status.conditions[?(@.type=="Ready")].status=="True")].metadata.name}'
```

### Node Issues

```bash
# Not ready nodes with message
kubectl get nodes -o jsonpath='{range .items[?(@.status.conditions[?(@.type=="Ready")].status!="True")]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].message}{"\n"}{end}'

# Nodes with memory pressure
kubectl get nodes -o jsonpath='{.items[?(@.status.conditions[?(@.type=="MemoryPressure")].status=="True")].metadata.name}'

# Nodes with disk pressure
kubectl get nodes -o jsonpath='{.items[?(@.status.conditions[?(@.type=="DiskPressure")].status=="True")].metadata.name}'
```

## Deployments

```bash
# Deployment names and replica counts
kubectl get deployments -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.replicas}{"\t"}{.status.readyReplicas}{"\n"}{end}'

# Deployment images
kubectl get deployments -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[*].image}{"\n"}{end}'

# Deployment strategies
kubectl get deployments -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.strategy.type}{"\n"}{end}'

# Deployment conditions (Available, Progressing)
kubectl get deployments -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Available")].status}{"\n"}{end}'

# All deployment conditions
kubectl get deployments -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.conditions[*]}{.type}{":"}{.status}{" "}{end}{"\n"}{end}'

# Deployment resource requests
kubectl get deployments -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[*].resources.requests.cpu}{"\n"}{end}'

# Deployment environment variables
kubectl get deployments -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[*].env[*].name}{"\n"}{end}'

# Deployments with 0 replicas
kubectl get deployments -n <namespace> -o jsonpath='{.items[?(@.spec.replicas==0)].metadata.name}'
```

## StatefulSets and DaemonSets

```bash
# StatefulSet replicas
kubectl get statefulsets -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.replicas}{"\t"}{.status.readyReplicas}{"\n"}{end}'

# StatefulSet service names
kubectl get statefulsets -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.serviceName}{"\n"}{end}'

# DaemonSet desired vs ready
kubectl get daemonsets -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.desiredNumberScheduled}{"\t"}{.status.numberReady}{"\n"}{end}'

# DaemonSet update strategy
kubectl get daemonsets -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.updateStrategy.type}{"\n"}{end}'

# DaemonSet node selectors
kubectl get daemonsets -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.nodeSelector}{"\n"}{end}'
```

## Services

```bash
# Service names and types
kubectl get svc -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.type}{"\n"}{end}'

# Service cluster IPs
kubectl get svc -n <namespace> -o jsonpath='{.items[*].spec.clusterIP}'

# Service ports
kubectl get svc -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.ports[*].port}{"\n"}{end}'

# Service ports with target ports (nested range)
kubectl get services -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.ports[*]}{.port}{":"}{.targetPort}{" "}{end}{"\n"}{end}'

# LoadBalancer services with external IPs
kubectl get svc -n <namespace> -o jsonpath='{range .items[?(@.spec.type=="LoadBalancer")]}{.metadata.name}{"\t"}{.status.loadBalancer.ingress[*].ip}{"\n"}{end}'

# NodePort services with node ports
kubectl get services -n <namespace> -o jsonpath='{range .items[?(@.spec.type=="NodePort")]}{.metadata.name}{"\t"}{range .spec.ports[*]}{.nodePort}{" "}{end}{"\n"}{end}'

# Service selectors
kubectl get svc -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.selector}{"\n"}{end}'

# Service endpoints (clusterIP:port)
kubectl get svc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.clusterIP}{":"}{.spec.ports[0].port}{"\n"}{end}'
```

## Ingress

```bash
# Ingress hosts
kubectl get ingress -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.rules[*].host}{"\n"}{end}'

# Ingress TLS secret names
kubectl get ingress -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.tls[*].secretName}{"\n"}{end}'

# Ingress backend services
kubectl get ingress -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.rules[*].http.paths[*].backend.service.name}{"\n"}{end}'

# Ingress classes
kubectl get ingress -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.ingressClassName}{"\n"}{end}'
```

## Storage

### Persistent Volumes

```bash
# PV names and capacities
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.capacity.storage}{"\n"}{end}'

# PV access modes
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.accessModes[*]}{"\n"}{end}'

# PV reclaim policies
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.persistentVolumeReclaimPolicy}{"\n"}{end}'

# PV storage classes
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.storageClassName}{"\n"}{end}'

# PV status and claim binding
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.spec.claimRef.name}{"\n"}{end}'

# Available PVs
kubectl get pv -o jsonpath='{.items[?(@.status.phase=="Available")].metadata.name}'
```

### Persistent Volume Claims

```bash
# PVC names and requested storage
kubectl get pvc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.resources.requests.storage}{"\n"}{end}'

# PVC status
kubectl get pvc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Bound PVCs with volumes
kubectl get pvc -o jsonpath='{range .items[?(@.status.phase=="Bound")]}{.metadata.name}{"\t"}{.spec.volumeName}{"\n"}{end}'

# PVC storage classes
kubectl get pvc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.storageClassName}{"\n"}{end}'
```

## Secrets and ConfigMaps

```bash
# Secret names and types
kubectl get secrets -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.type}{"\n"}{end}'

# TLS secrets only
kubectl get secrets -n <namespace> -o jsonpath='{.items[?(@.type=="kubernetes.io/tls")].metadata.name}'

# Docker registry secrets
kubectl get secrets -n <namespace> -o jsonpath='{.items[?(@.type=="kubernetes.io/dockerconfigjson")].metadata.name}'

# Decode secret value (pipe to base64 -d)
kubectl get secret mysecret -o jsonpath='{.data.password}' | base64 -d

# ConfigMap data keys
kubectl get configmap myconfig -o jsonpath='{.data}'

# Specific configmap key (escaped dot)
kubectl get configmap myconfig -o jsonpath='{.data.app\.properties}'

# ConfigMaps with a specific data key
kubectl get configmaps -n <namespace> -o jsonpath='{.items[?(@.data.database_url)].metadata.name}'
```

## Namespaces

```bash
# Namespace names
kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'

# Namespace status
kubectl get namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Namespace creation times
kubectl get namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}'

# Namespaces with specific labels
kubectl get namespaces -o jsonpath='{range .items[?(@.metadata.labels.name)]}{.metadata.name}{"\n"}{end}'
```

## Events

```bash
# Recent events sorted by time
kubectl get events -n <namespace> -o jsonpath='{range .items[*]}{.lastTimestamp}{"\t"}{.type}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}' | sort

# Warning events only
kubectl get events -n <namespace> -o jsonpath='{.items[?(@.type=="Warning")].message}'

# Events for a specific object
kubectl get events -n <namespace> -o jsonpath='{.items[?(@.involvedObject.name=="mypod")].message}'

# Events by reason (count)
kubectl get events -n <namespace> -o jsonpath='{range .items[*]}{.reason}{"\n"}{end}' | sort | uniq -c | sort -rn
```

## NetworkPolicies

```bash
# NetworkPolicy pod selectors
kubectl get networkpolicies -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podSelector.matchLabels}{"\n"}{end}'

# NetworkPolicy types
kubectl get networkpolicies -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.policyTypes[*]}{"\n"}{end}'
```

## Resource Quotas and Limit Ranges

```bash
# Quota hard limits
kubectl get resourcequota -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.hard}{"\n"}{end}'

# Quota usage
kubectl get resourcequota -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.used}{"\n"}{end}'

# Specific quota values (escaped dots)
kubectl get resourcequota -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.hard.requests\.cpu}{"\t"}{.spec.hard.requests\.memory}{"\n"}{end}'

# LimitRange defaults
kubectl get limitrange -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.limits[*].default}{"\n"}{end}'
```

## Recursive Descent (..)

Search all levels of nesting:

```bash
# All images across all resources
kubectl get deployments,daemonsets,statefulsets -o jsonpath='{..image}' | tr ' ' '\n' | sort | uniq

# All resource limits
kubectl get pods -n <namespace> -o jsonpath='{..resources.limits}' | tr ' ' '\n' | sort | uniq

# All secret references
kubectl get pods -n <namespace> -o jsonpath='{..secretKeyRef.name}' | tr ' ' '\n' | sort | uniq

# All configmap references
kubectl get pods -n <namespace> -o jsonpath='{..configMapKeyRef.name}' | tr ' ' '\n' | sort | uniq

# All container names at any depth
kubectl get pods -n <namespace> -o jsonpath='{..containers[*].name}'
```

## Sorting with --sort-by

`--sort-by` takes a JSONPath expression:

```bash
# Sort by name
kubectl get pods -n <namespace> --sort-by=.metadata.name

# Sort by creation time
kubectl get pods -n <namespace> --sort-by=.metadata.creationTimestamp

# Sort by restart count
kubectl get pods -n <namespace> --sort-by='.status.containerStatuses[0].restartCount'

# Sort nodes by CPU
kubectl get nodes --sort-by=.status.capacity.cpu

# Sort PVs by capacity
kubectl get pv --sort-by=.spec.capacity.storage
```

## Custom Columns (Alternative to JSONPath)

No `items` needed — custom-columns iterates automatically:

```bash
# Pod info
kubectl get pods -n <namespace> -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP

# Node info
kubectl get nodes -o custom-columns=NODE:.metadata.name,CPU:.status.capacity.cpu,MEMORY:.status.capacity.memory

# Node taints
kubectl get nodes -o custom-columns=NODE:.metadata.name,TAINT:.spec.taints[*].effect,KEY:.spec.taints[*].key

# PV sorted
kubectl get pv -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage --sort-by=.spec.capacity.storage
```

## Combining with Shell Tools

```bash
# Sort output numerically
kubectl get nodes -o jsonpath='{range .items[*]}{.status.capacity.cpu}{"\t"}{.metadata.name}{"\n"}{end}' | sort -n

# Sort by timestamp
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.creationTimestamp}{"\t"}{.metadata.name}{"\n"}{end}' | sort

# Filter with awk
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}' | awk '$2=="Running" {print $1}'

# Filter with grep
kubectl get services -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.type}{"\n"}{end}' | grep LoadBalancer

# Count
kubectl get pods -n <namespace> -o jsonpath='{.items[*].metadata.name}' | wc -w
```

## Escaping Special Characters

Labels and annotations with dots or slashes need escaping:

```bash
# Dots in label keys
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.labels.app\.kubernetes\.io/name}{"\n"}{end}'

# Node OS label
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.kubernetes\.io/os}{"\n"}{end}'
```

## Bash Functions

```bash
pod-ips() {
    kubectl get pods -l "$1" -o jsonpath='{.items[*].status.podIP}'
}

svc-endpoints() {
    kubectl get svc "$1" -o jsonpath='{.spec.clusterIP}:{.spec.ports[*].port}'
}

deploy-images() {
    kubectl get deployment "$1" -o jsonpath='{.spec.template.spec.containers[*].image}'
}

count-by-status() {
    kubectl get "$1" -o jsonpath='{.items[*].status.phase}' | tr ' ' '\n' | sort | uniq -c
}

failing-pods() {
    kubectl get pods -n <namespace> -o jsonpath='{.items[?(@.status.phase!="Running")].metadata.name}'
}
```

## Aliases

```bash
alias kgpn='kubectl get pods -o jsonpath="{.items[*].metadata.name}"'
alias kgpi='kubectl get pods -o jsonpath="{.items[*].status.podIP}"'
alias kgsip='kubectl get svc -o jsonpath="{.items[*].spec.clusterIP}"'
alias kgni='kubectl get nodes -o jsonpath="{.items[*].status.addresses[?(@.type==\"InternalIP\")].address}"'
alias kstatus='kubectl get pods -o jsonpath="{.items[*].status.phase}" | tr " " "\n" | sort | uniq -c'
```

## Limitations

JSONPath in kubectl cannot:
- Complex boolean operations (AND across different levels)
- Aggregation (counting, summing, grouping)
- String manipulation or regex
- Sorting within the expression (use `--sort-by` instead)
- Null/empty value checking
- Create new structures or rename fields

When you hit these limits, use [jq with kubectl](articles/kubectl-jq-guide.md).

## Troubleshooting

```bash
# Test with a single resource first
kubectl get pod mypod -o jsonpath='{.metadata.name}'

# Explore structure with jq
kubectl get pods -o json | jq '.items[0] | keys'

# Use kubectl explain to understand fields
kubectl explain pod.spec.containers
kubectl explain node.status.capacity

# Count results
kubectl get pods -n <namespace> -o jsonpath='{.items[*].metadata.name}' | wc -w

# Common mistake: no line breaks without range
# Wrong — all on one line:
kubectl get pods -n <namespace> -o jsonpath='{.items[*].metadata.name}'
# Right — one per line:
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

## Notes

- JSONPath expressions are case-sensitive
- Array indices start at 0
- Missing fields return empty values (no error)
- Use single quotes around the full JSONPath expression
- The `$` root is implicit — `{.items}` and `{$.items}` are equivalent
- Complex filters can impact performance on large clusters
- Use `--field-selector` and `-l` for server-side filtering before JSONPath

## Additional Resource Types

### StorageClasses

```bash
# StorageClass names and provisioners
kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.provisioner}{"\n"}{end}'

# Default storage class
kubectl get storageclass -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'
```

### Pod Volumes and Environment

```bash
# Volume names from pods
kubectl get pods -n <namespace> -o jsonpath='{.items[*].spec.volumes[*].name}'

# Volume mount paths per container
kubectl get pods -n <namespace> -o jsonpath='{.items[*].spec.containers[*].volumeMounts[*].mountPath}'

# Environment variable names from pods
kubectl get pods -n <namespace> -o jsonpath='{.items[*].spec.containers[*].env[*].name}'

# All ports from pod containers
kubectl get pods -n <namespace> -o jsonpath='{.items[*].spec.containers[*].ports[*].containerPort}'
```

### RBAC (ServiceAccounts, Roles, Bindings)

```bash
# ServiceAccount names
kubectl get serviceaccounts -n <namespace> -o jsonpath='{.items[*].metadata.name}'

# RoleBinding subjects
kubectl get rolebindings -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{": "}{range .subjects[*]}{.name}{" "}{end}{"\n"}{end}'

# ClusterRoleBinding names and role references
kubectl get clusterrolebindings -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.roleRef.name}{"\n"}{end}'

# ClusterRoles
kubectl get clusterroles -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

### Endpoints

```bash
# Service endpoints (IP addresses)
kubectl get endpoints -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.subsets[*].addresses[*].ip}{"\n"}{end}'

# Endpoints with ports
kubectl get endpoints -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{": "}{range .subsets[*]}{range .addresses[*]}{.ip}{" "}{end}{end}{"\n"}{end}'
```

## Array Slicing

```bash
# First element
kubectl get pods -n <namespace> -o jsonpath='{.items[0].metadata.name}'

# Last element
kubectl get pods -n <namespace> -o jsonpath='{.items[-1:].metadata.name}'

# First two elements
kubectl get pods -n <namespace> -o jsonpath='{.items[0:2].metadata.name}'

# Specific indices
kubectl get pods -n <namespace> -o jsonpath='{.items[0,2,4].metadata.name}'
```

## Variables in Range

kubectl JSONPath supports variable binding within range expressions:

```bash
# Bind name and phase to variables
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{$name := .metadata.name}{$phase := .status.phase}{$name}{" is "}{$phase}{"\n"}{end}'

# ConfigMap key iteration with variables
kubectl get configmaps -o jsonpath='{range .items[*]}{.metadata.name}{": "}{range $k,$v := .data}{$k}{" "}{end}{"\n"}{end}'
```

> **Note:** Variable support in kubectl JSONPath is limited compared to full JSONPath spec. If variables don't work as expected, use jq instead.

## All Resources Summary

```bash
# Quick overview of all resources in namespace
kubectl get all -o jsonpath='{range .items[*]}{.kind}{"/"}{.metadata.name}{"\n"}{end}'
```

## Field Selectors Combined with JSONPath

```bash
# Running pods on specific node
kubectl get pods --field-selector=status.phase=Running,spec.nodeName=worker-1 \
    -o jsonpath='{.items[*].metadata.name}'

# Pods in Pending state
kubectl get pods --field-selector=status.phase=Pending \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[*].message}{"\n"}{end}'
```

## Deployment Extended Status

```bash
# Deployment replicas/ready/updated
kubectl get deployments -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.replicas}{"/"}{.status.readyReplicas}{"/"}{.status.updatedReplicas}{"\n"}{end}'

# Custom columns with updated replicas
kubectl get deployments -n <namespace> -o custom-columns=NAME:.metadata.name,READY:.status.readyReplicas,UP-TO-DATE:.status.updatedReplicas,AVAILABLE:.status.availableReplicas
```

## Ingress Nested Backends

```bash
# All backend service names from all ingress rules and paths
kubectl get ingress -n <namespace> -o jsonpath='{range .items[*]}{range .spec.rules[*]}{range .http.paths[*]}{.backend.service.name}{"\n"}{end}{end}{end}'
```

## JSONPath Quick Reference Table

| Purpose | Expression |
|---------|-----------|
| Pod names | `{.items[*].metadata.name}` |
| Pod status | `{.items[*].status.phase}` |
| Container images | `{.items[*].spec.containers[*].image}` |
| Pod IPs | `{.items[*].status.podIP}` |
| Service IPs | `{.items[*].spec.clusterIP}` |
| Service ports | `{.items[*].spec.ports[*].port}` |
| Node names | `{.items[*].metadata.name}` |
| Running pods | `{.items[?(@.status.phase=="Running")].metadata.name}` |
| Failed pods | `{.items[?(@.status.phase=="Failed")].metadata.name}` |
| By label | `{.items[?(@.metadata.labels.app=="nginx")].metadata.name}` |
| First container | `{.items[*].spec.containers[0].image}` |
| All env vars | `{.items[*].spec.containers[*].env[*].name}` |
| Volume names | `{.items[*].spec.volumes[*].name}` |
| Node conditions | `{.items[*].status.conditions[*].type}` |
| Secret types | `{.items[*].type}` |
| Node IPs (internal) | `{.items[*].status.addresses[?(@.type=="InternalIP")].address}` |
| Ready nodes | `{.items[?(@.status.conditions[?(@.type=="Ready")].status=="True")].metadata.name}` |
| LoadBalancer svcs | `{.items[?(@.spec.type=="LoadBalancer")].metadata.name}` |
| All images (recursive) | `{..image}` |

