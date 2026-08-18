<img src="/articles/images/kubernetes-logo.svg" alt="Kubernetes" width="150">

# Using jq with kubectl

kubectl outputs JSON with `-o json`. Pipe it to jq for powerful filtering, transformation, and formatting that goes far beyond what JSONPath or custom-columns can do.

## The Basic Pattern

```bash
kubectl get pods -n <namespace> -o json | jq '<expression>'
```

jq expressions are **composable** — every filter takes input and produces output. Chain them with `|`:

```bash
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.status.phase == "Running") | .metadata.name'
```

This reads as: take all items → keep only running pods → extract the name.

## Composing Commands Step by Step

The key to jq is building up expressions incrementally. Start simple, add complexity:

```bash
# Step 1: See the raw structure
kubectl get pods -n <namespace> -o json | jq '.items[0]' | head -50

# Step 2: Identify the fields you need
kubectl get pods -n <namespace> -o json | jq '.items[0] | keys'

# Step 3: Extract a single field
kubectl get pods -n <namespace> -o json | jq '.items[].metadata.name'

# Step 4: Add more fields
kubectl get pods -n <namespace> -o json | jq '.items[] | {name: .metadata.name, phase: .status.phase}'

# Step 5: Add filtering
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.status.phase == "Running") | {name: .metadata.name}'

# Step 6: Format the output
kubectl get pods -n <namespace> -o json | jq -r '.items[] | select(.status.phase == "Running") | .metadata.name'
```

## Pods

### Basic Pod Info

```bash
# All pod names
kubectl get pods -n <namespace> -o json | jq -r '.items[].metadata.name'

# Pod names and status
kubectl get pods -n <namespace> -o json | jq -r '.items[] | "\(.metadata.name)\t\(.status.phase)"'

# Pod names, namespace, and status
kubectl get pods -n <namespace> -o json | jq -r '.items[] | "\(.metadata.namespace)\t\(.metadata.name)\t\(.status.phase)"'

# Pods with IP addresses
kubectl get pods -n <namespace> -o json | jq -r '.items[] | "\(.metadata.name)\t\(.status.podIP)"'

# Pod names and the node they're running on
kubectl get pods -n <namespace> -o json | jq -r '.items[] | "\(.metadata.name)\t\(.spec.nodeName)"'
```

### Filter Pods

```bash
# Only running pods
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.status.phase == "Running") | .metadata.name'

# Pods NOT running
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.status.phase != "Running") | {name: .metadata.name, phase: .status.phase}'

# Pods in CrashLoopBackOff
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.status.containerStatuses[]?.state.waiting?.reason == "CrashLoopBackOff") | .metadata.name'

# Pods with restarts > 0
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.status.containerStatuses[]?.restartCount > 0) | {name: .metadata.name, restarts: .status.containerStatuses[0].restartCount}'

# Pods older than 24 hours (using now)
kubectl get pods -n <namespace> -o json | jq --argjson cutoff "$(date -d '24 hours ago' +%s)" '.items[] | select((.metadata.creationTimestamp | fromdateiso8601) < $cutoff) | .metadata.name'

# Pods using more than 1Gi memory request
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.spec.containers[].resources.requests.memory // "" | test("Gi")) | .metadata.name'
```

### Container Details

```bash
# All container images in use
kubectl get pods -n <namespace> -o json | jq -r '.items[].spec.containers[].image' | sort -u

# All init container images
kubectl get pods -n <namespace> -o json | jq -r '.items[].spec.initContainers[]?.image' | sort -u

# Pods and their container images
kubectl get pods -n <namespace> -o json | jq -r '.items[] | "\(.metadata.name)\t\(.spec.containers[].image)"'

# Containers with resource limits
kubectl get pods -n <namespace> -o json | jq '.items[].spec.containers[] | select(.resources.limits != null) | {name: .name, limits: .resources.limits}'

# Containers WITHOUT resource limits
kubectl get pods -n <namespace> -o json | jq '.items[] | {pod: .metadata.name, containers: [.spec.containers[] | select(.resources.limits == null) | .name]} | select(.containers | length > 0)'

# Environment variables for a specific container
kubectl get pod mypod -o json | jq '.spec.containers[0].env[]'

# All containers with their ports
kubectl get pods -n <namespace> -o json | jq '.items[].spec.containers[] | {name: .name, ports: [.ports[]?.containerPort]}'
```

### Pod Events and Conditions

```bash
# Pod conditions (Ready, Initialized, etc.)
kubectl get pod mypod -o json | jq '.status.conditions[] | {type: .type, status: .status}'

# Pods that are not Ready
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.status.conditions[]? | select(.type == "Ready" and .status == "False")) | .metadata.name'

# Last termination reason
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.status.containerStatuses[]?.lastState.terminated != null) | {name: .metadata.name, reason: .status.containerStatuses[0].lastState.terminated.reason}'
```

## Deployments

```bash
# Deployment names and replica counts
kubectl get deployments -n <namespace> -o json | jq -r '.items[] | "\(.metadata.name)\t\(.spec.replicas)\t\(.status.readyReplicas // 0)"'

# Deployments not fully available
kubectl get deployments -n <namespace> -o json | jq '.items[] | select(.status.readyReplicas != .spec.replicas) | {name: .metadata.name, desired: .spec.replicas, ready: (.status.readyReplicas // 0)}'

# Deployment images
kubectl get deployments -n <namespace> -o json | jq -r '.items[] | "\(.metadata.name)\t\(.spec.template.spec.containers[0].image)"'

# Deployments with specific label
kubectl get deployments -n <namespace> -o json | jq '.items[] | select(.metadata.labels.app == "web") | .metadata.name'

# Deployment strategy
kubectl get deployments -n <namespace> -o json | jq '.items[] | {name: .metadata.name, strategy: .spec.strategy.type}'
```

## Services

```bash
# Service names and types
kubectl get services -n <namespace> -o json | jq -r '.items[] | "\(.metadata.name)\t\(.spec.type)\t\(.spec.clusterIP)"'

# LoadBalancer services with external IPs
kubectl get services -n <namespace> -o json | jq '.items[] | select(.spec.type == "LoadBalancer") | {name: .metadata.name, external: .status.loadBalancer.ingress[]?.ip}'

# Service ports
kubectl get services -n <namespace> -o json | jq '.items[] | {name: .metadata.name, ports: [.spec.ports[] | "\(.port):\(.targetPort)/\(.protocol)"]}'

# Services with selectors
kubectl get services -n <namespace> -o json | jq '.items[] | {name: .metadata.name, selector: .spec.selector}'
```

## Nodes

```bash
# Node names and status
kubectl get nodes -o json | jq -r '.items[] | "\(.metadata.name)\t\(.status.conditions[] | select(.type == "Ready") | .status)"'

# Node capacity (CPU, memory)
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, cpu: .status.capacity.cpu, memory: .status.capacity.memory}'

# Node allocatable vs capacity
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, cpu_capacity: .status.capacity.cpu, cpu_allocatable: .status.allocatable.cpu, mem_capacity: .status.capacity.memory, mem_allocatable: .status.allocatable.memory}'

# Node taints
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, taints: [.spec.taints[]? | "\(.key)=\(.value // ""):\(.effect)"]}'

# Node labels
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, labels: .metadata.labels}'

# Nodes with specific label
kubectl get nodes -o json | jq '.items[] | select(.metadata.labels["node.kubernetes.io/instance-type"] != null) | {name: .metadata.name, type: .metadata.labels["node.kubernetes.io/instance-type"]}'

# Node OS and kernel
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, os: .status.nodeInfo.osImage, kernel: .status.nodeInfo.kernelVersion, container_runtime: .status.nodeInfo.containerRuntimeVersion}'

# Node internal IPs
kubectl get nodes -o json | jq -r '.items[] | "\(.metadata.name)\t\(.status.addresses[] | select(.type == "InternalIP") | .address)"'
```

## Secrets and ConfigMaps

```bash
# List secret names and types
kubectl get secrets -n <namespace> -o json | jq -r '.items[] | "\(.metadata.name)\t\(.type)"'

# Decode a specific secret value
kubectl get secret mysecret -o json | jq -r '.data["password"] | @base64d'

# Decode ALL values in a secret
kubectl get secret mysecret -o json | jq '.data | map_values(@base64d)'

# ConfigMap data keys
kubectl get configmap myconfig -o json | jq '.data | keys'

# ConfigMap content
kubectl get configmap myconfig -o json | jq -r '.data | to_entries[] | "--- \(.key) ---\n\(.value)"'
```

## PersistentVolumes and Claims

```bash
# PV names, capacity, and status
kubectl get pv -o json | jq -r '.items[] | "\(.metadata.name)\t\(.spec.capacity.storage)\t\(.status.phase)"'

# PVCs and their bound volumes
kubectl get pvc -n <namespace> -o json | jq -r '.items[] | "\(.metadata.name)\t\(.spec.volumeName)\t\(.status.phase)"'

# Unbound PVs
kubectl get pv -o json | jq '.items[] | select(.status.phase == "Available") | {name: .metadata.name, capacity: .spec.capacity.storage}'

# PVs sorted by capacity
kubectl get pv -o json | jq '.items | sort_by(.spec.capacity.storage) | .[] | {name: .metadata.name, capacity: .spec.capacity.storage}'
```

## Labels and Annotations

```bash
# Get all labels for pods
kubectl get pods -n <namespace> -o json | jq '.items[] | {name: .metadata.name, labels: .metadata.labels}'

# Find pods with a specific label
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.metadata.labels.app == "frontend") | .metadata.name'

# Find pods missing a required label
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.metadata.labels.team == null) | .metadata.name'

# Get all unique label keys across all pods
kubectl get pods -n <namespace> -o json | jq '[.items[].metadata.labels | keys[]] | unique'

# Get all annotation keys
kubectl get pods -n <namespace> -o json | jq '[.items[].metadata.annotations // {} | keys[]] | unique'

# Find pods with a specific annotation
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.metadata.annotations["prometheus.io/scrape"] == "true") | .metadata.name'
```

## Aggregation and Counting

```bash
# Count pods per namespace
kubectl get pods -n <namespace> -o json | jq '.items | group_by(.metadata.namespace) | map({namespace: .[0].metadata.namespace, count: length}) | sort_by(.count) | reverse'

# Count pods per node
kubectl get pods -n <namespace> -o json | jq '.items | group_by(.spec.nodeName) | map({node: .[0].spec.nodeName, count: length})'

# Count pods per status
kubectl get pods -n <namespace> -o json | jq '.items | group_by(.status.phase) | map({status: .[0].status.phase, count: length})'

# Count container images in use
kubectl get pods -n <namespace> -o json | jq '[.items[].spec.containers[].image] | group_by(.) | map({image: .[0], count: length}) | sort_by(.count) | reverse'

# Total requested CPU across all pods
kubectl get pods -n <namespace> -o json | jq '[.items[].spec.containers[].resources.requests.cpu // "0" | rtrimstr("m") | tonumber] | add'

# Pods per deployment (via owner reference)
kubectl get pods -n <namespace> -o json | jq '.items | group_by(.metadata.ownerReferences[0].name) | map({owner: .[0].metadata.ownerReferences[0].name, count: length})'
```

## Formatting Output

### TSV for Shell Processing

```bash
# Tab-separated (pipe to awk, cut, sort, etc.)
kubectl get pods -n <namespace> -o json | jq -r '.items[] | [.metadata.name, .status.phase, .status.podIP] | @tsv'

# Sort by name
kubectl get pods -n <namespace> -o json | jq -r '.items | sort_by(.metadata.name) | .[] | [.metadata.name, .status.phase] | @tsv'
```

### CSV for Spreadsheets

```bash
# With header
echo "name,namespace,status,ip" && \
kubectl get pods -n <namespace> -o json | jq -r '.items[] | [.metadata.name, .metadata.namespace, .status.phase, .status.podIP] | @csv'
```

### Custom Formatted Strings

```bash
# Formatted report
kubectl get pods -n <namespace> -o json | jq -r '.items[] | "Pod: \(.metadata.name) | Status: \(.status.phase) | IP: \(.status.podIP // "N/A")"'

# Markdown table
echo "| Pod | Status | Node |" && echo "|-----|--------|------|" && \
kubectl get pods -n <namespace> -o json | jq -r '.items[] | "| \(.metadata.name) | \(.status.phase) | \(.spec.nodeName) |"'
```

### Multiple Fields on One Line

```bash
# Print multiple fields (one value per line, interleaved)
aws ssm get-parameters --names /path/to/param | jq -r '.Parameters[] | .Value, .LastModifiedDate'

# As an array (shows brackets)
aws ssm get-parameters --names /path/to/param | jq -r '.Parameters[] | [.Value, .LastModifiedDate]'

# Join fields with a separator
aws ssm get-parameters --names /path/to/param | jq -r '.Parameters[] | [.Value, .LastModifiedDate] | join(" ")'

# Tab-separated using paste (pairs alternating lines)
aws ssm get-parameters --names /path/to/param | jq -r '.Parameters[] | .Value, .LastModifiedDate' | paste - -
```

### JSON Output (Reshaped)

```bash
# Clean JSON array of objects
kubectl get pods -n <namespace> -o json | jq '[.items[] | {name: .metadata.name, status: .status.phase, ip: .status.podIP}]'

# Save to file
kubectl get pods -n <namespace> -o json | jq '[.items[] | {name: .metadata.name, image: .spec.containers[0].image}]' > pods.json
```

## Combining with Shell Commands

```bash
# Delete all failed pods
kubectl get pods -n <namespace> -o json | jq -r '.items[] | select(.status.phase == "Failed") | .metadata.name' | xargs kubectl delete pod

# Restart all deployments in a namespace
kubectl get deployments -n <namespace> -o json | jq -r '.items[].metadata.name' | xargs -I{} kubectl rollout restart deployment/{}

# Get logs from all pods with a label
kubectl get pods -l app=web -o json | jq -r '.items[].metadata.name' | xargs -I{} sh -c 'echo "=== {} ===" && kubectl logs {} --tail=5'

# Scale down all deployments
kubectl get deployments -n <namespace> -o json | jq -r '.items[].metadata.name' | xargs -I{} kubectl scale deployment/{} --replicas=0

# Copy a secret to another namespace
kubectl get secret mysecret -o json | jq 'del(.metadata.namespace, .metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp)' | kubectl apply -n other-namespace -f -

# Find which pods are using a specific configmap
CONFIGMAP="myconfig"
kubectl get pods -n <namespace> -o json | jq --arg cm "$CONFIGMAP" '.items[] | select(.spec.volumes[]?.configMap.name == $cm) | .metadata.name'
```

## Discovering JSON Structure

When you don't know the structure, use these techniques. Kubernetes resources have deeply nested JSON — pods alone have hundreds of fields across metadata, spec, and status. Listing all paths lets you find the exact field you need without reading documentation or guessing at nesting levels.

```bash
# List ALL paths in compact format (the starting point for exploration)
kubectl get pods -A -o json | jq -c paths

# See top-level keys
kubectl get pods -n <namespace> -o json | jq '.items[0] | keys'

# See all paths (dot notation)
kubectl get pods -n <namespace> -o json | jq -r '.items[0] | paths(scalars) | join(".")'

# See paths containing a keyword
kubectl get pods -n <namespace> -o json | jq -c '.items[0] | paths | select(. | join(".") | contains("restart"))'

# Search paths with grep (alternative)
kubectl get pods -A -o json | jq -c 'paths|join(".")' | grep startTime

# Alternative: grep raw paths then convert commas to dots
kubectl get pods -A -o json | jq -c paths | grep startTime | sed 's/,/./g'

# Get paths AND values in a single dot-notation line
kubectl get pods -A -o json | jq -r 'paths(scalars) as $p | $p + [getpath($p)] | join(".")'

# See structure without values (keys only, recursive)
kubectl get pods -n <namespace> -o json | jq '.items[0] | .. | objects | keys' | sort -u

# Get a single pod in full to study
kubectl get pod <name> -o json | jq '.' > pod-structure.json
```

## Handling Missing Fields

```bash
# Use // for defaults (null coalescing)
kubectl get pods -n <namespace> -o json | jq '.items[] | {name: .metadata.name, ip: (.status.podIP // "pending")}'

# Use ? to suppress errors
kubectl get pods -n <namespace> -o json | jq '.items[] | {name: .metadata.name, restart: (.status.containerStatuses[0]?.restartCount // 0)}'

# Skip items where a field doesn't exist
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.status.podIP != null) | {name: .metadata.name, ip: .status.podIP}'

# Handle optional arrays
kubectl get pods -n <namespace> -o json | jq '.items[] | {name: .metadata.name, init_containers: [.spec.initContainers[]?.name] | length}'
```

## Multi-Resource Queries

```bash
# Compare desired vs actual replicas across all deployments
kubectl get deployments -n <namespace> -o json | jq -r '
    .items[] |
    select(.status.readyReplicas != .spec.replicas) |
    "\(.metadata.namespace)/\(.metadata.name)\tdesired:\(.spec.replicas)\tready:\(.status.readyReplicas // 0)"'

# Find pods not owned by a ReplicaSet (orphans)
kubectl get pods -n <namespace> -o json | jq '.items[] | select(.metadata.ownerReferences == null) | .metadata.name'

# Match services to their pod endpoints
kubectl get endpoints -n <namespace> -o json | jq '.items[] | {service: .metadata.name, endpoints: [.subsets[]?.addresses[]?.ip]}'
```

## Performance Tips

```bash
# Use --field-selector to reduce data before jq
kubectl get pods --field-selector=status.phase=Running -o json | jq '...'

# Use -l (label selector) to filter server-side
kubectl get pods -l app=web -o json | jq '...'

# Limit to specific namespace
kubectl get pods -n production -o json | jq '...'

# Use --chunk-size for large clusters
kubectl get pods -A --chunk-size=100 -o json | jq '...'

# Pipe through jq -c for compact output (less memory)
kubectl get pods -n <namespace> -o json | jq -c '.items[] | {name: .metadata.name}'
```

## Reusable Scripts

### Pod Health Report

```bash
#!/bin/bash
# pod-health.sh — Show pods with issues

kubectl get pods -n <namespace> -o json | jq -r '
    .items[] |
    select(
        .status.phase != "Running" and .status.phase != "Succeeded"
        or (.status.containerStatuses[]?.restartCount // 0) > 5
    ) |
    [.metadata.namespace, .metadata.name, .status.phase,
     (.status.containerStatuses[0]?.restartCount // 0 | tostring)] |
    @tsv' | column -t
```

### Image Inventory

```bash
#!/bin/bash
# image-inventory.sh — List all images with their pods

kubectl get pods -n <namespace> -o json | jq -r '
    .items[] |
    .metadata as $meta |
    .spec.containers[] |
    [$meta.namespace, $meta.name, .name, .image] |
    @tsv' | sort | column -t
```

### Resource Usage Summary

```bash
#!/bin/bash
# resource-summary.sh — Requested resources per namespace

kubectl get pods -n <namespace> -o json | jq '
    .items |
    group_by(.metadata.namespace) |
    map({
        namespace: .[0].metadata.namespace,
        pods: length,
        containers: [.[].spec.containers | length] | add,
        images: [.[].spec.containers[].image] | unique | length
    }) | sort_by(.pods) | reverse'
```

### Stale Resources Finder

```bash
#!/bin/bash
# stale-resources.sh — Find old resources

echo "=== Pods older than 7 days ==="
kubectl get pods -n <namespace> -o json | jq -r --argjson cutoff "$(date -d '7 days ago' +%s 2>/dev/null || date -v-7d +%s)" '
    .items[] |
    select((.metadata.creationTimestamp | fromdateiso8601) < $cutoff) |
    [.metadata.namespace, .metadata.name, .metadata.creationTimestamp] |
    @tsv' | column -t
```

## jq vs JSONPath vs custom-columns

| Need | Use |
|------|-----|
| Quick field extraction | `kubectl -o jsonpath='{.items[*].metadata.name}'` |
| Nice table output | `kubectl -o custom-columns=NAME:.metadata.name` |
| Filtering by value | jq with `select()` |
| Counting / aggregation | jq with `group_by`, `length`, `add` |
| Complex transformation | jq (reshaping, merging, math) |
| Pipe to other commands | jq with `-r` + `xargs` |
| Decode secrets | jq with `@base64d` |
| Generate reports | jq with `@csv`, `@tsv`, string interpolation |

## JSONPath Equivalents

When jq is overkill, kubectl's built-in JSONPath works for simple extraction.

### Converting jq to JSONPath

```bash
# jq — complex filtering (nodes without taints, that are Ready)
kubectl get nodes -o json | jq -r '.items[] | select(.spec.taints|not) | select(.status.conditions[].reason=="KubeletReady" and .status.conditions[].status=="True") | .metadata.name'

# JSONPath — simplified (filter by Ready condition)
kubectl get nodes -o jsonpath='{range .items[?(@.status.conditions[*].type=="Ready")]}{.metadata.name}{"\n"}{end}'

# Practical alternatives (server-side filtering)
kubectl get nodes --field-selector=spec.unschedulable!=true -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# Or with grep
kubectl get nodes --no-headers | grep Ready | grep -v SchedulingDisabled | awk '{print $1}'
```

### JSONPath Formatting

```bash
# Custom headers with $'' syntax
kubectl get nodes -o jsonpath=$'NAME\tSTATUS\tVERSION\n{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'

# Space-separated to newline-separated (tr trick)
kubectl get pods -n <namespace> -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n'

# Multiple fields with custom separator
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.status.phase}{" -> "}{.spec.nodeName}{"\n"}{end}'

# CSV format with headers
kubectl get pods -n <namespace> -o jsonpath=$'NAMESPACE/POD,IP\n{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{","}{.status.podIP}{"\n"}{end}'

# Namespace:Pod format
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.namespace}{":"}{.metadata.name}{" -> "}{.status.podIP}{"\n"}{end}'
```

### JSONPath Common Patterns

```bash
# Node capacity
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.cpu}{"\t"}{.status.capacity.memory}{"\n"}{end}'

# Node OS and architecture
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.osImage}{"\t"}{.status.nodeInfo.architecture}{"\n"}{end}'

# Node kubelet version
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'

# Pod names, IPs, and nodes
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\t"}{.spec.nodeName}{"\n"}{end}'

# Pod restart counts
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].restartCount}{"\n"}{end}'

# Resource requests and limits
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].resources.requests.cpu}{"\t"}{.spec.containers[0].resources.limits.memory}{"\n"}{end}'

# Service endpoints (name, ClusterIP, port)
kubectl get services -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.clusterIP}{"\t"}{.spec.ports[0].port}{"\n"}{end}'

# Ingress hosts
kubectl get ingress -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.rules[*].host}{"\n"}{end}'
```

### JSONPath Filtering

```bash
# Filter by label
kubectl get pods -n <namespace> -o jsonpath='{range .items[?(@.metadata.labels.app=="nginx")]}{.metadata.name}{"\n"}{end}'

# Filter by phase
kubectl get pods -n <namespace> -o jsonpath='{range .items[?(@.status.phase=="Running")]}{.metadata.name}{"\n"}{end}'

# Filter nodes by Ready condition
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'
```

## JSONPath Limitations

JSONPath cannot handle:
- Complex boolean operations (`and`, `not`, nested `or`)
- Multiple criteria filtering in a single expression
- Null/empty value checking (`select(.field|not)`)
- Aggregation (counting, summing, grouping)
- String manipulation or regex
- Sorting within the expression

When you hit these limits, switch to jq:

```bash
# JSONPath can't do this — use jq
# Nodes without taints AND in Ready state
kubectl get nodes -o json | jq -r '
    .items[] |
    select(.spec.taints | not) |
    select(.status.conditions[] | select(.type == "Ready" and .status == "True")) |
    .metadata.name'

# JSONPath can't count — use jq
kubectl get pods -n <namespace> -o json | jq '.items | length'

# JSONPath can't sort — use jq or --sort-by
kubectl get pods -n <namespace> -o json | jq -r '.items | sort_by(.metadata.creationTimestamp) | .[] | .metadata.name'
```

## Troubleshooting JSONPath

```bash
# Test with a single item first
kubectl get pods -n <namespace> -o jsonpath='{.items[0].metadata.name}'

# Verify the path exists (use jq to explore)
kubectl get pods -n <namespace> -o json | jq '.items[0].metadata.name'

# Count results
kubectl get pods -n <namespace> -o jsonpath='{.items[*].metadata.name}' | wc -w

# Common mistake: missing range for line-by-line output
# Wrong — all names on one line:
kubectl get pods -n <namespace> -o jsonpath='{.items[*].metadata.name}'

# Right — one name per line:
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# Common mistake: using jsonpath when jq is needed
# Can't filter by multiple conditions in jsonpath — use jq instead
```

## Best Practices

| Scenario | Approach |
|----------|----------|
| Simple field extraction | JSONPath `'{.items[*].metadata.name}'` |
| Formatted table | `custom-columns=NAME:.metadata.name` |
| Line-by-line output | JSONPath `range` or `jq -r` |
| Complex filtering | jq with `select()` |
| Counting / aggregation | jq with `group_by`, `length` |
| Server-side filtering | `--field-selector`, `-l` labels |
| Quick grep-style filtering | `--no-headers` + `grep` + `awk` |
| Scripts and automation | jq (more robust error handling) |

## JSONPath Operators Quick Guide

```bash
$                           # Root
@                           # Current object
.field                      # Object field
['field'] or ["field"]      # Bracket notation
[*]                         # All array elements
[0]                         # First array element
[0,1]                       # Multiple elements
[?(<expression>)]           # Filter expression
..field                     # Recursive descent
```

Common filter expressions:

```bash
[?(@.field)]                # Field exists
[?(@.field=="value")]       # Field equals value
[?(@.field!="value")]       # Field not equals value
[?(@.field>5)]              # Numeric comparison
[?(!@.field)]               # Field doesn't exist
```

## StatefulSets and DaemonSets

### StatefulSets

```bash
# StatefulSet replicas
kubectl get statefulsets -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.replicas}{"\t"}{.status.readyReplicas}{"\n"}{end}'

# StatefulSet service names
kubectl get statefulsets -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.serviceName}{"\n"}{end}'

# StatefulSet images
kubectl get statefulsets -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[*].image}{"\n"}{end}'
```

### DaemonSets

```bash
# DaemonSet node selectors
kubectl get daemonsets -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.nodeSelector}{"\n"}{end}'

# DaemonSet update strategy
kubectl get daemonsets -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.updateStrategy.type}{"\n"}{end}'

# DaemonSet desired vs ready
kubectl get daemonsets -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.desiredNumberScheduled}{"\t"}{.status.numberReady}{"\n"}{end}'
```

## Storage Details

### PV Access Modes and Reclaim Policy

```bash
# PV access modes
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.accessModes[*]}{"\n"}{end}'

# PV reclaim policies
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.persistentVolumeReclaimPolicy}{"\n"}{end}'

# PV storage classes
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.storageClassName}{"\n"}{end}'

# PV status and claim binding
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.spec.claimRef.name}{"\n"}{end}'
```

### PVC Details

```bash
# PVC requested storage
kubectl get pvc -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.resources.requests.storage}{"\n"}{end}'

# PVC storage classes
kubectl get pvc -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.storageClassName}{"\n"}{end}'

# Bound PVCs with their volumes
kubectl get pvc -n <namespace> -o jsonpath='{range .items[?(@.status.phase=="Bound")]}{.metadata.name}{"\t"}{.spec.volumeName}{"\n"}{end}'
```

## Secrets by Type

```bash
# TLS secrets
kubectl get secrets -o jsonpath='{.items[?(@.type=="kubernetes.io/tls")].metadata.name}'

# Docker registry secrets
kubectl get secrets -o jsonpath='{.items[?(@.type=="kubernetes.io/dockerconfigjson")].metadata.name}'

# Opaque secrets
kubectl get secrets -o jsonpath='{.items[?(@.type=="Opaque")].metadata.name}'
```

## ConfigMap Specific Keys

```bash
# Get a specific key from a configmap (dot in key name escaped)
kubectl get configmap myconfig -o jsonpath='{.data.app\.properties}'

# ConfigMaps that contain a specific data key
kubectl get configmaps -o jsonpath='{.items[?(@.data.database_url)].metadata.name}'

# ConfigMap creation times
kubectl get configmaps -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}'
```

## Ingress Details

```bash
# Ingress TLS secret names
kubectl get ingress -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.tls[*].secretName}{"\n"}{end}'

# Ingress backend services
kubectl get ingress -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.rules[*].http.paths[*].backend.service.name}{"\n"}{end}'

# Ingress classes
kubectl get ingress -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.ingressClassName}{"\n"}{end}'
```

## NetworkPolicies

```bash
# NetworkPolicy pod selectors
kubectl get networkpolicies -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podSelector.matchLabels}{"\n"}{end}'

# NetworkPolicy policy types
kubectl get networkpolicies -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.policyTypes[*]}{"\n"}{end}'
```

## Events

```bash
# Recent events sorted by time
kubectl get events -o jsonpath='{range .items[*]}{.lastTimestamp}{"\t"}{.type}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}' | sort

# Warning events only
kubectl get events -o jsonpath='{.items[?(@.type=="Warning")].message}'

# Events for a specific object
kubectl get events -o jsonpath='{.items[?(@.involvedObject.name=="mypod")].message}'

# Events by reason (count with uniq)
kubectl get events -o jsonpath='{range .items[*]}{.reason}{"\t"}{.message}{"\n"}{end}' | sort | uniq -c
```

## Resource Quotas and Limit Ranges

```bash
# Resource quota hard limits
kubectl get resourcequota -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.hard}{"\n"}{end}'

# Resource quota usage
kubectl get resourcequota -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.used}{"\n"}{end}'

# Specific quota values (escaped dots)
kubectl get resourcequota -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.hard.requests\.cpu}{"\t"}{.spec.hard.requests\.memory}{"\n"}{end}'

# LimitRange defaults
kubectl get limitrange -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.limits[*].default}{"\n"}{end}'

# LimitRange max values
kubectl get limitrange -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.limits[*].max}{"\n"}{end}'
```

## Namespace Queries

```bash
# All namespace names
kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'

# Namespace status
kubectl get namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Namespace creation times
kubectl get namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}'

# Namespaces with specific labels
kubectl get namespaces -o jsonpath='{range .items[?(@.metadata.labels.name)]}{.metadata.name}{"\n"}{end}'
```

## Node Extended Info

```bash
# External IPs
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# Node roles
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.kubernetes\.io/role}{"\n"}{end}'

# Node allocatable resources
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.cpu}{"\t"}{.status.allocatable.memory}{"\n"}{end}'

# All node IP addresses with types
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.addresses[*]}{.type}{":"}{.address}{" "}{end}{"\n"}{end}'

# Node taints (key:effect format)
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.taints[*]}{.key}{":"}{.effect}{" "}{end}{"\n"}{end}'

# Specific node label (escaped dots)
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.kubernetes\.io/os}{"\n"}{end}'
```

## Deployment Extended

```bash
# Deployment conditions
kubectl get deployments -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.conditions[*]}{.type}{":"}{.status}{" "}{end}{"\n"}{end}'

# Deployment environment variables
kubectl get deployments -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[*].env[*].name}{"\n"}{end}'

# Deployment resource requests
kubectl get deployments -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[*].resources.requests.cpu}{"\n"}{end}'

# Deployments with 0 replicas
kubectl get deployments -o jsonpath='{.items[?(@.spec.replicas==0)].metadata.name}'
```

## Nested Range

For pods with multiple containers, use nested range:

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

## Recursive Searches Across Resources

```bash
# All images across deployments, daemonsets, statefulsets
kubectl get deployments,daemonsets,statefulsets -o jsonpath='{..image}' | tr ' ' '\n' | sort | uniq

# All resource limits across all pods
kubectl get pods -n <namespace> -o jsonpath='{..resources.limits}' | tr ' ' '\n' | sort | uniq

# All secret references in pods
kubectl get pods -n <namespace> -o jsonpath='{..secretKeyRef.name}' | tr ' ' '\n' | sort | uniq

# All configmap references
kubectl get pods -n <namespace> -o jsonpath='{..configMapKeyRef.name}' | tr ' ' '\n' | sort | uniq
```

## Service Ports and NodePorts

```bash
# Service ports with target ports
kubectl get services -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.ports[*]}{.port}{":"}{.targetPort}{" "}{end}{"\n"}{end}'

# NodePort services with node ports
kubectl get services -n <namespace> -o jsonpath='{range .items[?(@.spec.type=="NodePort")]}{.metadata.name}{"\t"}{range .spec.ports[*]}{.nodePort}{" "}{end}{"\n"}{end}'
```

## Troubleshooting JSONPath (Extended)

### Node Issues

```bash
# Nodes not ready with message
kubectl get nodes -o jsonpath='{range .items[?(@.status.conditions[?(@.type=="Ready")].status!="True")]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].message}{"\n"}{end}'

# Nodes with memory pressure
kubectl get nodes -o jsonpath='{.items[?(@.status.conditions[?(@.type=="MemoryPressure")].status=="True")].metadata.name}'

# Nodes with disk pressure
kubectl get nodes -o jsonpath='{.items[?(@.status.conditions[?(@.type=="DiskPressure")].status=="True")].metadata.name}'
```

### Pod Issues

```bash
# Pods with high restart count (> 5)
kubectl get pods -n <namespace> -o jsonpath='{.items[?(@.status.containerStatuses[0].restartCount>5)].metadata.name}'

# Non-running pods with waiting reason
kubectl get pods -n <namespace> -o jsonpath='{range .items[?(@.status.phase!="Running")]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.status.containerStatuses[*].state.waiting.reason}{"\n"}{end}'

# Failed pods with exit codes
kubectl get pods -n <namespace> -o jsonpath='{range .items[?(@.status.phase=="Failed")]}{.metadata.name}{"\t"}{.status.containerStatuses[*].state.terminated.exitCode}{"\n"}{end}'
```

### Escaping Special Characters

```bash
# Labels with dots (escape with backslash)
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.labels.app\.kubernetes\.io/name}{"\n"}{end}'

# Annotations with slashes
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}{"\n"}{end}'
```

### Validate Structure with kubectl explain

```bash
# Understand resource structure
kubectl explain pod.spec.containers
kubectl explain node.status.capacity
kubectl explain deployment.spec.strategy
kubectl explain service.spec.ports
```

## Bash Functions and Aliases

### Functions

```bash
# Get pod IPs by label
pod-ips() {
    kubectl get pods -l "$1" -o jsonpath='{.items[*].status.podIP}'
}

# Get service endpoint
svc-endpoints() {
    kubectl get svc "$1" -o jsonpath='{.spec.clusterIP}:{.spec.ports[*].port}'
}

# Get deployment images
deploy-images() {
    kubectl get deployment "$1" -o jsonpath='{.spec.template.spec.containers[*].image}'
}

# Count resources by status
count-by-status() {
    kubectl get "$1" -o jsonpath='{.items[*].status.phase}' | tr ' ' '\n' | sort | uniq -c
}

# Get failing pods
failing-pods() {
    kubectl get pods -n <namespace> -o jsonpath='{.items[?(@.status.phase!="Running")].metadata.name}'
}
```

### Aliases

```bash
# Add to ~/.bashrc or ~/.zshrc
alias kgpn='kubectl get pods -n <namespace> -o jsonpath="{.items[*].metadata.name}"'
alias kgpi='kubectl get pods -n <namespace> -o jsonpath="{.items[*].status.podIP}"'
alias kgsip='kubectl get svc -o jsonpath="{.items[*].spec.clusterIP}"'
alias kgni='kubectl get nodes -o jsonpath="{.items[*].status.addresses[?(@.type==\"InternalIP\")].address}"'
alias kstatus='kubectl get pods -n <namespace> -o jsonpath="{.items[*].status.phase}" | tr " " "\n" | sort | uniq -c'
```

## Combining JSONPath with Shell Tools

```bash
# Sort pods by creation time
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.creationTimestamp}{"\t"}{.metadata.name}{"\n"}{end}' | sort

# Sort nodes by CPU capacity (numeric)
kubectl get nodes -o jsonpath='{range .items[*]}{.status.capacity.cpu}{"\t"}{.metadata.name}{"\n"}{end}' | sort -n

# Filter with awk
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}' | awk '$2=="Running" {print $1}'

# Filter with grep
kubectl get services -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.type}{"\n"}{end}' | grep LoadBalancer

# Formatted table with column
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.spec.containers[*].image}{"\t"}{.status.podIP}{"\n"}{end}' | column -t

# Pod phase summary
kubectl get pods -n <namespace> -o jsonpath='{.items[*].status.phase}' | tr ' ' '\n' | sort | uniq -c

# CSV export
echo "Name,Status,IP,Image" > pods.csv
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{","}{.status.phase}{","}{.status.podIP}{","}{.spec.containers[*].image}{"\n"}{end}' >> pods.csv
```

## Notes

- JSONPath expressions are case-sensitive
- Use quotes around JSONPath expressions to avoid shell interpretation
- Array indices start at 0
- Missing fields return empty values without errors
- Complex filters can impact performance on large clusters
- Consider `jq` for aggregation, counting, and complex transformations
- Use `--field-selector` and `-l` for server-side filtering before JSONPath
