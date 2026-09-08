# kubectl + sed Combinations

Powerful combinations of `kubectl` and `sed` for filtering resources, extracting
fields, editing manifests, and processing logs.

> **When to prefer native kubectl:** parsing tabular output with `sed` is handy
> for ad-hoc work, but it's positional and breaks when column layout changes.
> For anything scripted or long-lived, prefer `kubectl` output flags:
> `-o name` (bare names), `-o jsonpath=...`, `-o custom-columns=...`, and
> `--field-selector`. Use the `sed` patterns below for quick interactive tasks.

## Basic Patterns

### Get and filter resources

```bash
# Pod names only
kubectl get pods --no-headers | sed 's/ .*//'

# Pods with a specific status
kubectl get pods --no-headers | sed -n '/Running/p'

# Extract pod IPs
kubectl get pods -o wide --no-headers | sed 's/.* \([0-9]\+\.[0-9]\+\.[0-9]\+\.[0-9]\+\) .*/\1/'

# LoadBalancer external IPs
kubectl get svc --no-headers | sed -n 's/.*LoadBalancer.*\([0-9]\+\.[0-9]\+\.[0-9]\+\.[0-9]\+\).*/\1/p'
```

### Resource name manipulation

```bash
# Names without prefixes
kubectl get deployments --no-headers | sed 's/^[^ ]* *//' | sed 's/ .*//'

# Names matching a pattern
kubectl get pods --no-headers | sed -n '/web-/p' | sed 's/ .*//'

# Strip namespace column (from -A output)
kubectl get pods -A --no-headers | sed 's/^[^ ]* *//' | sed 's/ .*//'

# Turn deployment names into "<name>-svc"
kubectl get deployments --no-headers | sed 's/ .*//' | sed 's/$/\-svc/'
```

## Resource Information Extraction

### Pods

```bash
# Name and restart count
kubectl get pods --no-headers | sed 's/\([^ ]*\).* \([0-9]*\) .*/\1: \2 restarts/'

# Failed pods only
kubectl get pods --no-headers | sed -n '/Error\|Failed\|CrashLoopBackOff/p' | sed 's/ .*//'

# Ready vs total containers
kubectl get pods --no-headers | sed 's/\([^ ]*\) *\([0-9]*\/[0-9]*\).*/\1: \2/'
```

### Nodes

```bash
# Node names only
kubectl get nodes --no-headers | sed 's/ .*//'

# Node roles
kubectl get nodes --no-headers | sed 's/[^ ]* *[^ ]* *\([^ ]*\).*/\1/'

# Nodes on a specific version
kubectl get nodes --no-headers | sed -n '/v1.28/p' | sed 's/ .*//'

# Node external IPs
kubectl get nodes -o wide --no-headers | sed 's/.* \([0-9]\+\.[0-9]\+\.[0-9]\+\.[0-9]\+\) .*/\1/'
```

### Services and ingress

```bash
# Service ports
kubectl get svc --no-headers | sed 's/.*:\([0-9]*\).*/\1/'

# Ingress hosts
kubectl get ingress --no-headers | sed 's/[^ ]* *\([^ ]*\).*/\1/'

# Service cluster IPs
kubectl get svc --no-headers | sed 's/[^ ]* *[^ ]* *\([0-9\.]*\).*/\1/'
```

## YAML Manipulation

### Modify resource specs

```bash
# Change image tag
kubectl get deployment myapp -o yaml | sed 's/image: myapp:.*/image: myapp:v2.0/' | kubectl apply -f -

# Update replica count
kubectl get deployment myapp -o yaml | sed 's/replicas: [0-9]*/replicas: 5/' | kubectl apply -f -

# Change a service type
kubectl get svc myapp -o yaml | sed 's/type: ClusterIP/type: LoadBalancer/' | kubectl apply -f -
```

> For a single field, `kubectl patch` or `kubectl set image` is safer than a
> blanket `sed` substitution, which can match more lines than intended.

### Remove metadata for backup/recreation

```bash
# Strip status and server-populated metadata
kubectl get deployment myapp -o yaml \
  | sed '/^status:/,$d' \
  | sed '/resourceVersion:/d;/uid:/d;/creationTimestamp:/d'
```

### Template generation

```bash
# Turn an existing resource into a placeholder template
kubectl get deployment myapp -o yaml \
  | sed 's/myapp/{{APP_NAME}}/g' \
  | sed 's/namespace: .*/namespace: {{NAMESPACE}}/'

# Generate multiple resources from one template
for app in web api worker; do
  kubectl get deployment template -o yaml | sed "s/template/$app/g" | kubectl apply -f -
done
```

## Log Processing

```bash
# Errors only
kubectl logs myapp-pod | sed -n '/ERROR/p'

# Strip leading timestamps
kubectl logs myapp-pod | sed 's/^[0-9-]* [0-9:]* *//'

# Extract a JSON "message" field
kubectl logs myapp-pod | sed -n 's/.*"message":"\([^"]*\)".*/\1/p'

# Filter by level
kubectl logs myapp-pod | sed -n '/WARN\|ERROR\|FATAL/p'

# Request paths from access logs
kubectl logs nginx-pod | sed -n 's/.*"\(GET\|POST\|PUT\|DELETE\) \([^ ]*\) .*/\2/p'
```

### Across multiple pods

```bash
# Logs from all pods matching a pattern
kubectl get pods --no-headers | sed -n '/web-/p' | sed 's/ .*//' | xargs -I {} kubectl logs {}

# Errors from each pod of a label, with headers
kubectl get pods -l app=myapp --no-headers | sed 's/ .*//' \
  | xargs -I {} sh -c 'echo "=== {} ===" && kubectl logs {} | sed -n "/ERROR/p"'
```

## Events and Status

```bash
# Warning events only
kubectl get events --no-headers | sed -n '/Warning/p'

# Count pods by status
kubectl get pods --no-headers | sed 's/.* \([^ ]*\) .*/\1/' | sort | uniq -c

# Summarize node readiness
kubectl get nodes --no-headers | sed 's/[^ ]* *\([^ ]*\).*/\1/' | sort | uniq -c

# Probe-failure summary
kubectl get events --no-headers | sed -n '/Readiness\|Liveness/p' \
  | sed 's/.* \([^ ]*\) probe failed.*/\1/' | sort | uniq -c
```

## Batch Operations

```bash
# Scale every deployment to 3
kubectl get deployments --no-headers | sed 's/ .*//' \
  | xargs -I {} kubectl scale deployment {} --replicas=3

# Restart deployments matching a pattern
kubectl get deployments --no-headers | sed -n '/web-/p' | sed 's/ .*//' \
  | xargs -I {} kubectl rollout restart deployment {}

# Update image on matching deployments
kubectl get deployments --no-headers | sed -n '/api-/p' | sed 's/ .*//' \
  | xargs -I {} kubectl set image deployment {} container=myimage:v2.0
```

## Cleanup

```bash
# Delete failed/evicted pods
kubectl get pods --no-headers | sed -n '/Error\|Failed\|Evicted/p' | sed 's/ .*//' \
  | xargs kubectl delete pod

# Delete replica sets scaled to zero
kubectl get rs --no-headers | sed -n '/.*0 *0 *0/p' | sed 's/ .*//' \
  | xargs kubectl delete rs
```

> Deleting pods managed by a controller just recreates them — delete the owning
> Deployment/Job instead when you mean to remove the workload.

## Aliases and Functions

```bash
# ~/.bashrc / ~/.zshrc
alias kgpn='kubectl get pods --no-headers | sed "s/ .*//"'
alias kgdn='kubectl get deployments --no-headers | sed "s/ .*//"'
alias kgpf='kubectl get pods --no-headers | sed -n "/Error\|Failed\|CrashLoop/p" | sed "s/ .*//"'

# Logs from all pods matching a pattern
klogs() {
  kubectl get pods --no-headers | sed -n "/$1/p" | sed 's/ .*//' \
    | xargs -I {} sh -c 'echo "=== {} ===" && kubectl logs {} --tail=20'
}

# Scale all deployments matching a pattern: kscale <pattern> <replicas>
kscale() {
  kubectl get deployments --no-headers | sed -n "/$1/p" | sed 's/ .*//' \
    | xargs -I {} kubectl scale deployment {} --replicas="$2"
}

# Quick resource counts
kcount() {
  echo "Pods:        $(kubectl get pods --no-headers | wc -l)"
  echo "Deployments: $(kubectl get deployments --no-headers | wc -l)"
  echo "Services:    $(kubectl get svc --no-headers | wc -l)"
  echo "ConfigMaps:  $(kubectl get configmaps --no-headers | wc -l)"
}
```

## Quick Reference

```bash
# Pod names only
kubectl get pods --no-headers | sed 's/ .*//'

# Failed pods
kubectl get pods --no-headers | sed -n '/Error\|Failed/p' | sed 's/ .*//'

# Pod IPs
kubectl get pods -o wide --no-headers | sed 's/.* \([0-9\.]*\) .*/\1/'

# Clean metadata for backup
kubectl get deployment myapp -o yaml | sed '/^status:/,$d' | sed '/resourceVersion:\|uid:\|creationTimestamp:/d'

# Error/warning logs only
kubectl logs pod-name | sed -n '/ERROR\|WARN/p'
```

### sed Pattern Reference

```bash
's/old/new/'           # replace first occurrence
's/old/new/g'          # replace all occurrences
'/pattern/p'           # print lines matching pattern (with -n)
'/pattern/d'           # delete lines matching pattern
's/^[^ ]* *//'         # remove the first word
's/ .*//'              # keep only the first word
'/start/,/end/p'       # print from start to end pattern
```

## Related

- [kubectl Cheatsheet](articles/kubectl-cheatsheet.md)
- [kubectl + jq Guide](articles/kubectl-jq-guide.md)
- [kubectl JSONPath Guide](articles/kubectl-jsonpath-guide.md)
- [sed Cheatsheet](articles/sed-cheatsheet.md)

## Skills Practiced

- Extracting fields and filtering resources by piping `kubectl` output through `sed`
- Editing manifests inline (`get -o yaml | sed | apply -f -`) and templating
- Processing logs and events, and running batch operations with `xargs`
- Knowing when native `kubectl` output flags are safer than positional `sed`
