# kubectl Cheatsheet

## Aliases

```sh
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ka='kubectl apply -f'
alias kdel='kubectl delete'
alias kex='kubectl exec -it'
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes'
alias kgs='kubectl get svc'
alias kcontexts='kubectl config get-contexts'
alias kctx='kubectl config use-context'
```

## Cluster Management

```sh
kubectl version                                  # Client and server versions
kubectl cluster-info                             # Control plane and cluster services address
kubectl cluster-info dump                        # Dump info for debugging
kubectl config view                              # Show merged kubeconfig
kubectl config current-context                   # Display the current-context
kubectl config get-contexts                      # Display all contexts
kubectl config use-context <context_name>        # Switch context
kubectl config delete-context <context_name>     # Delete a context
kubectl api-resources                            # Supported API resources
kubectl api-versions                             # Supported API versions
kubectl auth can-i --list                        # List all your permissions
```

```sh
kubectl get all -A
kubectl get all -n <namespace>
kubectl get nodes,all -A
kubectl get all -A -o json | jq -r '.items[].metadata.labels' | sed 's/,//g' | sort | uniq   # All labels
```

## Performance

```sh
kubectl top node                                 # Metrics for all nodes
kubectl top node <node_name>                     # Metrics for a given node
kubectl top pod -A                               # Metrics for all pods
kubectl top pods -A | sort -rn -k3 | head        # Top 10 CPU pods
kubectl top pods -A | sort -rn -k4 | head        # Top 10 Memory pods
kubectl top pod -A --containers                  # Metrics for containers
kubectl top pod -n <namespace>                   # Pods in a namespace
kubectl top pod -n <namespace> --sort-by=cpu     # Sort by CPU
kubectl top pod -n <namespace> --sort-by=memory  # Sort by Memory
kubectl top pod <pod_name> --containers          # Pod and its containers
kubectl top pod -l name=<myLabel>                # Pods by label
```

## Nodes

```sh
kubectl get nodes
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl describe nodes | grep OS
kubectl get nodes -o wide --label-columns topology.kubernetes.io/zone
kubectl get nodes -o wide --sort-by=.metadata.creationTimestamp         # Sort by age
kubectl get nodes --no-headers -o custom-columns=NAME:.metadata.name   # Just node names
kubectl wait --for=condition=Ready nodes --all --timeout=300s

# Cordon/Uncordon/Drain
kubectl cordon <node-name>
kubectl uncordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>

# View taints
kubectl describe node <node-name> | grep Taints
kubectl get nodes -o='custom-columns=NodeName:.metadata.name,TaintKey:.spec.taints[*].key,TaintValue:.spec.taints[*].value,TaintEffect:.spec.taints[*].effect'

# CPU/MEM allocated for pods on each node
kubectl describe nodes | sed -n '/Non-terminated Pods/,/Allocated resources/p'
kubectl describe nodes | awk '/Non-terminated Pods/,/Allocated resources/'
```

## Pods

```sh
kubectl get pods -A
kubectl get pods -o wide -A
kubectl get pods -n kube-system
kubectl get pods -A --show-labels
kubectl get pods -o wide -A | grep <node-name>

kubectl exec -it <pod> -n <ns> -- /bin/bash
kubectl exec -it <pod> -n <ns> -- /bin/bash -c "ls -l /"

# Filter by status
kubectl get pods --field-selector status.phase!=Running                # Non-running pods
kubectl get pods -A --field-selector=status.phase=Running              # Running pods
kubectl get pods -A --field-selector=status.phase=Pending              # Pending pods

# Sort
kubectl get pods -A --sort-by=.metadata.creationTimestamp                                   # By age
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'                   # By restart count
kubectl get pods -A -o wide --sort-by="{.spec.nodeName}"                                    # By node

# Custom columns
kubectl get pods -A -o custom-columns="POD:metadata.name,PHASE:status.phase"
kubectl get pods -A -o custom-columns="POD:metadata.name,STATE:status.containerStatuses[*].ready"
kubectl get pods -A -o custom-columns="POD:metadata.name,STATE:status.containerStatuses[*].state.waiting.reason" | grep -v none
kubectl get pods -A -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,CONTAINERS_READY:.status.containerStatuses[*].ready'
kubectl get pods -A -o='custom-columns=NameSpace:.metadata.namespace,NAME:.metadata.name,CONTAINERS:.spec.containers[*].name'
kubectl get pods -A --output=custom-columns="NAME:.metadata.name,IMAGE:.spec.containers[*].image"

# Count by restart
kubectl get pods -A -o json | jq '.items[] | [.status.containerStatuses[0].restartCount, .metadata.name] | join("\t")' -r | sort -n

# Pods not ready
kubectl get pods -A -o custom-columns=NAMESPACE:metadata.namespace,POD:metadata.name,READY-true:status.containerStatuses[*].ready | grep false
kubectl get pods -A -o json | jq -r '.items[] | select(.status.containerStatuses[].ready == false) | "\(.metadata.namespace) \(.metadata.name)"' | sort -u | column -t

# Pods per node (count)
kubectl get pods -A -o json | jq '.items | group_by(.spec.nodeName) | map({"nodeName": .[0].spec.nodeName, "count": length}) | sort_by(.count)'

# Delete pods by name pattern
kubectl get pods --no-headers | grep "node-debugger" | awk '{print $1}' | xargs kubectl delete pod
```

### List Pods Per Node

```sh
# All pods for every node
for i in $(kubectl get nodes --no-headers | cut -d " " -f1); do
  echo "=== $i ==="
  kubectl get pods -A --no-headers -o wide --field-selector spec.nodeName=$i
done

# Pods for a specific node group
for i in $(kubectl get nodes -l eks.amazonaws.com/nodegroup=my-node-group --no-headers | cut -d " " -f1); do
  echo "=== $i ==="
  kubectl get pods -A --no-headers -o wide --field-selector spec.nodeName=$i
done
```

### Pods with Containers Not Ready

```sh
kubectl get pods -A | grep -Pv '\s+([1-9]+[\d]*)\/\1\s+'

kubectl get pods -A -o go-template='{{ range $item := .items }}{{ range .status.conditions }}{{ if (or (and (eq .type "PodScheduled") (eq .status "False")) (and (eq .type "Ready") (eq .status "False"))) }}{{ $item.metadata.name}} {{"\n"}}{{ end }}{{ end }}{{ end }}'
```

## Deployments

```sh
kubectl get deployment -A
kubectl get deployment coredns -n kube-system
kubectl describe deployment coredns -n kube-system
kubectl edit deployment coredns -n kube-system
kubectl delete deployment nginx-deployment -n nginx

kubectl rollout status deployment nginx-deployment
kubectl rollout history deploy coredns -n kube-system
kubectl rollout undo deploy <deployment-name> -n <namespace>
kubectl rollout undo deploy <deployment-name> --to-revision=50 -n <namespace>
```

## Namespaces

```sh
kubectl get ns
kubectl create namespace <name>
kubectl get namespace <name>
kubectl describe namespace kube-system
kubectl delete namespace <name>
kubectl edit namespace <name>
```

## Creating Resources Quickly

```sh
kubectl run nginx --image=nginx --port=80
kubectl run nginx --image=nginx --port=80 --dry-run=client -o yaml
kubectl expose pod nginx --port=80 --type=NodePort

kubectl create deploy --image=nginx:1.23 nginx --replicas=4
kubectl expose deploy nginx --type=NodePort --port=8080 --target-port=80
```

## DaemonSets

```sh
kubectl get ds -A
kubectl edit ds kube-proxy -n kube-system
kubectl delete ds <daemonset_name>
kubectl describe ds aws-node -n kube-system
```

## ReplicaSets

```sh
kubectl get rs -A
kubectl describe replicasets <rs-name> -n <namespace>
kubectl scale deployment <deployment-name> -n kube-system --replicas=2
```

## StatefulSets

```sh
kubectl get statefulset -A
```

## Services

```sh
kubectl get services -A
kubectl describe services kube-dns -n kube-system
kubectl edit services kube-dns -n kube-system
kubectl get services -A --sort-by=.metadata.name
kubectl get svc -v=9                             # Show full API request
```

## ConfigMaps

```sh
kubectl get configmap -A
kubectl get configmap -n kube-system
kubectl describe configmap aws-auth -n kube-system
kubectl describe configmap coredns -n kube-system
kubectl get configmap coredns -n kube-system -o yaml
```

## Secrets

```sh
kubectl get secrets -A
kubectl describe secrets <secret-name> -n <namespace>
```

## Service Accounts

```sh
kubectl get serviceaccounts -A
kubectl describe serviceaccounts <sa-name> -n <namespace>
```

## Events

```sh
kubectl get events -A
kubectl get events -n <namespace>
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl get events -A --field-selector involvedObject.kind!=Pod
kubectl get events -A --field-selector involvedObject.kind=Pod
kubectl get events -A --field-selector type=Warning
kubectl get events -A --field-selector type!=Normal
kubectl get events --field-selector involvedObject.kind=Node
kubectl get events --field-selector involvedObject.kind=Node -w
kubectl get events --field-selector involvedObject.kind=Node,involvedObject.name=<node-name>
```

## Logs

```sh
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --since=12h
kubectl logs <pod-name> -n <namespace> --tail=50
kubectl logs <pod-name> -n <namespace> -f                    # Follow
kubectl logs <pod-name> -n <namespace> > pod.log             # Save to file
kubectl logs <pod-name> -n <namespace> -c <container-name>   # Specific container
kubectl logs <pod-name> -n <namespace> --previous            # Previous instance
kubectl logs -l app=nginx -n <namespace>                     # By label
```

## HPA

```sh
kubectl get hpa -A
```

## PDB

```sh
kubectl get pdb -A
```

## Persistent Volumes

```sh
kubectl get pv
kubectl get pv --sort-by=.spec.capacity.storage
kubectl get pvc -A
```

## RBAC

```sh
kubectl get clusterroles
kubectl get clusterroles | grep eks
kubectl describe clusterrole <role-name>
kubectl get clusterroles <role-name> -o yaml

kubectl get clusterrolebindings
kubectl get clusterrolebindings | grep eks

kubectl get roles -A
kubectl describe role <role-name> -n <namespace>

kubectl auth can-i --list
kubectl auth can-i create pods
kubectl auth can-i get pods --as=developer
kubectl auth can-i --list --as=system:serviceaccount:default:my-sa
```

## VPC CNI Environment Variables

```sh
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
kubectl set env daemonset aws-node -n kube-system ENI_CONFIG_LABEL_DEF=topology.kubernetes.io/zone
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
```

## kubectl Verbosity

| Level | Description |
|:-----:|-------------|
| `--v=0` | Generally useful for cluster operators |
| `--v=1` | Reasonable default log level |
| `--v=2` | Useful steady state info and significant changes |
| `--v=3` | Extended information about changes |
| `--v=4` | Debug level verbosity |
| `--v=5` | Trace level verbosity |
| `--v=6` | Display requested resources |
| `--v=7` | Display HTTP request headers |
| `--v=8` | Display HTTP request contents |
| `--v=9` | Display HTTP request contents without truncation |




## Autocompletion

```sh
# Bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Bash with alias
alias k=kubectl
complete -o default -F __start_kubectl k

# Zsh
source <(kubectl completion zsh)
echo '[[ $commands[kubectl] ]] && source <(kubectl completion zsh)' >> ~/.zshrc

# Fish
echo 'kubectl completion fish | source' > ~/.config/fish/completions/kubectl.fish
```

## Multiple Kubeconfig Files

```sh
# Merge multiple kubeconfig files
export KUBECONFIG=~/.kube/config:~/.kube/cluster2-config

# View merged config
kubectl config view

# Set default namespace for current context
kubectl config set-context --current --namespace=my-namespace
```

## Output Formats

| Format | Flag | Description |
|--------|------|-------------|
| Wide | `-o wide` | Additional info (node name, IPs) |
| YAML | `-o yaml` | Full YAML output |
| JSON | `-o json` | Full JSON output |
| Name | `-o name` | Only resource name |
| JSONPath | `-o jsonpath='{...}'` | Extract specific fields |
| Custom columns | `-o custom-columns=...` | Table with specific columns |
| Go template | `-o go-template='{{...}}'` | Go template formatting |

```sh
# All images in the cluster
kubectl get pods -A -o=custom-columns='DATA:spec.containers[*].image'

# Images grouped by pod
kubectl get pods -o=custom-columns="NAME:.metadata.name,IMAGE:.spec.containers[*].image"

# Decode secrets without external tools
kubectl get secret my-secret -o go-template='{{range $k,$v := .data}}{{"### "}}{{$k}}{{"\n"}}{{$v|base64decode}}{{"\n\n"}}{{end}}'
```

## Labels and Annotations

```sh
# Add a label
kubectl label pods my-pod env=prod

# Overwrite a label
kubectl label pods my-pod env=staging --overwrite

# Remove a label
kubectl label pods my-pod env-

# Add an annotation
kubectl annotate pods my-pod description="my app"

# Remove an annotation
kubectl annotate pods my-pod description-
```

## Patching Resources

```sh
# Strategic merge patch
kubectl patch node <node-name> -p '{"spec":{"unschedulable":true}}'

# Update container image
kubectl patch pod my-pod -p '{"spec":{"containers":[{"name":"app","image":"nginx:1.25"}]}}'

# JSON patch (positional array)
kubectl patch pod my-pod --type='json' -p='[{"op":"replace","path":"/spec/containers/0/image","value":"nginx:1.25"}]'

# Patch a deployment's replicas via scale subresource
kubectl patch deployment nginx --subresource='scale' --type='merge' -p '{"spec":{"replicas":3}}'
```

## Tainting Nodes

```sh
# Add a taint
kubectl taint nodes <node-name> dedicated=gpu:NoSchedule

# Remove a taint
kubectl taint nodes <node-name> dedicated=gpu:NoSchedule-
```

## Copy Files To/From Containers

```sh
# Local to pod
kubectl cp /tmp/file.txt my-pod:/tmp/file.txt

# Pod to local
kubectl cp my-pod:/tmp/file.txt /tmp/file.txt

# Specific container
kubectl cp /tmp/file.txt my-pod:/tmp/file.txt -c my-container

# Cross-namespace
kubectl cp my-namespace/my-pod:/tmp/foo /tmp/bar
```

> Requires `tar` in the container image. For advanced use cases, use `kubectl exec` with tar directly.

## Debugging

```sh
# Interactive debug session in an existing pod
kubectl debug my-pod -it --image=busybox:1.36

# Debug a node (creates a privileged pod on the node)
kubectl debug node/<node-name> -it --image=busybox:1.36

# Attach to a running container
kubectl attach my-pod -i

# Port forward (pod)
kubectl port-forward my-pod 8080:80

# Port forward (service)
kubectl port-forward svc/my-service 8080:80

# Port forward (deployment)
kubectl port-forward deploy/my-deployment 8080:80
```

## Diff (Compare Manifest vs Cluster)

```sh
# Show what would change if you applied this manifest
kubectl diff -f deployment.yaml
```

## Auto-Scaling

```sh
# Create an HPA
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80

# Check HPA
kubectl get hpa
```

## Deleting Resources

```sh
# Delete by file
kubectl delete -f ./pod.yaml

# Delete with no grace period
kubectl delete pod my-pod --now

# Delete multiple
kubectl delete pod,service baz foo

# Delete by label
kubectl delete pods,services -l name=myLabel

# Delete with multiple label conditions
kubectl delete pods -l app=myapp,version=v1

# Delete all in namespace
kubectl -n my-ns delete pod,svc --all

# Delete by pattern (awk)
kubectl get pods --no-headers | awk '/pattern1|pattern2/{print $1}' | xargs kubectl delete pod

# Delete by regex pattern
kubectl get pods -o name | grep -E "pod/test-[0-9]+" | xargs kubectl delete
```

### Delete by Status

```sh
# Delete all failed pods
kubectl delete pods --field-selector=status.phase=Failed --all-namespaces

# Delete succeeded pods (completed jobs)
kubectl delete pods --field-selector=status.phase=Succeeded --all-namespaces

# Delete evicted pods
kubectl get pods -A --field-selector=status.phase=Failed | grep Evicted | awk '{print $2 " -n " $1}' | xargs -I {} kubectl delete pod {}

# Delete pods on a specific node
kubectl delete pods --field-selector=spec.nodeName=worker-1
```

### Delete by Advanced Filtering

```sh
# Delete pods with high restart count (>5)
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.containerStatuses[0].restartCount}{"\n"}{end}' | \
  awk '$2 > 5 {print $1}' | xargs kubectl delete pod

# Delete pods with specific container image
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.containers[0].image}{"\n"}{end}' | \
  grep "nginx:1.14" | awk '{print $1}' | xargs kubectl delete pod

# Delete pods created before a specific date
kubectl get pods -o json | jq -r '.items[] | select(.metadata.creationTimestamp < "2024-01-01T00:00:00Z") | .metadata.name' | \
  xargs kubectl delete pod

# Delete pods with specific annotation
kubectl get pods -o json | jq -r '.items[] | select(.metadata.annotations["delete-me"] == "true") | .metadata.name' | \
  xargs kubectl delete pod
```

### Graceful vs Force Deletion

```sh
# Standard graceful (30s default)
kubectl delete pods -l app=myapp

# Custom grace period
kubectl delete pods -l app=myapp --grace-period=60

# Immediate force delete (use with caution)
kubectl delete pods -l app=myapp --force --grace-period=0

# Force delete stuck Terminating pods
kubectl get pods | grep Terminating | awk '{print $1}' | xargs kubectl delete pod --force --grace-period=0
```

### Safe Deletion Practices

```sh
# Always dry-run first
kubectl delete pods -l app=myapp --dry-run=client

# Check what you're about to delete
kubectl get pods -l app=myapp -o wide

# Check if pods are managed by a controller (they'll be recreated)
kubectl get pods -l app=myapp -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.metadata.ownerReferences[0].kind}{"\n"}{end}'
# If owned by ReplicaSet/Deployment → delete the deployment instead
```

### Cleanup Script

```sh
#!/bin/bash
# Quick pod cleanup script
echo "Deleting failed pods..."
kubectl delete pods --field-selector=status.phase=Failed -A
echo "Deleting succeeded pods..."
kubectl delete pods --field-selector=status.phase=Succeeded -A
echo "Deleting evicted pods..."
kubectl get pods -A --field-selector=status.phase=Failed | grep Evicted | awk '{print $2 " -n " $1}' | xargs -I {} kubectl delete pod {}
echo "Done."
```

## Interacting with Deployments and Services

```sh
# Logs from a deployment
kubectl logs deploy/my-deployment
kubectl logs deploy/my-deployment -c my-container

# Exec into a deployment's pod
kubectl exec deploy/my-deployment -- ls /

# Rolling update image
kubectl set image deployment/frontend www=nginx:v2

# Rolling restart
kubectl rollout restart deployment/frontend

# Watch rollout
kubectl rollout status -w deployment/frontend
```


## Advanced Node Queries

```sh
# Nodes sorted by memory capacity
kubectl get no -o json | jq -r '.items | sort_by(.status.capacity.memory)[] | [.metadata.name, .status.capacity.memory] | @tsv'

# Get internal IPs of all nodes
kubectl get nodes -o json | jq -r '.items[].status.addresses[]? | select(.type == "InternalIP") | .address'

# Get pod CIDR for each node (useful for CNI troubleshooting)
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}' | tr " " "\n"
```

## Advanced Pod Queries

```sh
# Print resource requests and limits for all pods
kubectl get pods -n <namespace> -o=custom-columns='NAME:spec.containers[*].name,MEMREQ:spec.containers[*].resources.requests.memory,MEMLIM:spec.containers[*].resources.limits.memory,CPUREQ:spec.containers[*].resources.requests.cpu,CPULIM:spec.containers[*].resources.limits.cpu'

# Find non-running pods (excluding Completed jobs)
kubectl get pods -A --field-selector=status.phase!=Running | grep -v Complete

# Find nodes where a DaemonSet pod is NOT scheduled
ns=kube-system
pod_template=aws-node
kubectl get node | grep -v "$(kubectl -n ${ns} get pod -o wide | grep ${pod_template} | awk '{print $7}' | xargs -n 1 echo -n "\|")"
```

## Service Discovery

```sh
# Get services with their selectors (find which pods a service targets)
kubectl get svc -n <namespace> -o wide

# Print all services and their nodePorts
kubectl get --all-namespaces svc -o json | jq -r '.items[] | [.metadata.name, ([.spec.ports[].nodePort | tostring] | join("|"))] | @tsv'
```

## Logs (Additional)

```sh
# Logs with timestamps
kubectl logs -n <namespace> <pod> --timestamps

# All containers in a pod
kubectl logs -n <namespace> <pod> --all-containers

# All pods with a specific label
kubectl logs -n <namespace> -l app=nginx -f

# Combine tail + follow + timestamps
kubectl logs -n <namespace> <pod> --tail=100 --timestamps -f
```

## Copy Secrets Between Namespaces

```sh
# Copy all secrets from one namespace to another
kubectl get secrets -o json --namespace old-ns | \
  jq '.items[].metadata.namespace = "new-ns"' | \
  kubectl create -f -

# Copy a single secret
kubectl get secret my-secret -n old-ns -o yaml | \
  sed 's/namespace: old-ns/namespace: new-ns/' | \
  kubectl apply -f -
```

## Create Self-Signed TLS Secret

```sh
# Generate cert and key
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=app.example.com/O=MyOrg"

# Create the secret
kubectl -n <namespace> create secret tls my-tls-secret --key tls.key --cert tls.crt

# Cleanup
rm tls.key tls.crt
```


## Extended Aliases

```sh
# Get all namespaces shortcuts
alias kgpa='kubectl get pods -A'
alias kgda='kubectl get deployments -A'
alias kgsa='kubectl get services -A'
alias kgaa='kubectl get all -A'

# Describe shortcuts
alias kdp='kubectl describe pod'
alias kdd='kubectl describe deployment'

# Logs
alias klf='kubectl logs -f'

# Namespace and context
alias kgns='kubectl get namespaces'
alias kn='kubectl config set-context --current --namespace'
```

## Functions for Partial Name Matching

```sh
# Get pod by partial name (usage: kgpn <pattern> [namespace])
kgpn() {
  local ns_flag=""
  [[ -n "$2" ]] && ns_flag="-n $2"
  kubectl get pods $ns_flag | grep "$1"
}

# Get logs from pod by partial name (usage: klogn <pattern> [namespace])
klogn() {
  local ns_flag=""
  [[ -n "$2" ]] && ns_flag="-n $2"
  local pod=$(kubectl get pods --no-headers $ns_flag | grep "$1" | head -1 | awk '{print $1}')
  [[ -z "$pod" ]] && echo "No pod found matching: $1" && return 1
  kubectl logs -f $ns_flag "$pod"
}

# Exec into pod by partial name (usage: kexn <pattern> [namespace])
kexn() {
  local ns_flag=""
  [[ -n "$2" ]] && ns_flag="-n $2"
  local pod=$(kubectl get pods --no-headers $ns_flag | grep "$1" | head -1 | awk '{print $1}')
  [[ -z "$pod" ]] && echo "No pod found matching: $1" && return 1
  kubectl exec -it $ns_flag "$pod" -- /bin/sh
}

# Port forward by pod pattern (usage: kpf <pattern> <local:remote> [namespace])
kpf() {
  local ns_flag=""
  [[ -n "$3" ]] && ns_flag="-n $3"
  local pod=$(kubectl get pods --no-headers $ns_flag | grep "$1" | head -1 | awk '{print $1}')
  [[ -z "$pod" ]] && echo "No pod found matching: $1" && return 1
  kubectl port-forward $ns_flag "$pod" "$2"
}

# Watch pods (usage: kwp [namespace])
kwp() {
  local ns_flag=""
  [[ -n "$1" ]] && ns_flag="-n $1"
  watch kubectl get pods $ns_flag
}

# Restart deployment (usage: krestart <deployment> [namespace])
krestart() {
  local ns_flag=""
  [[ -n "$2" ]] && ns_flag="-n $2"
  kubectl rollout restart deployment "$1" $ns_flag
}

# Events sorted by time (usage: kevents [namespace])
kevents() {
  local ns_flag=""
  [[ -n "$1" ]] && ns_flag="-n $1"
  kubectl get events --sort-by='.lastTimestamp' $ns_flag
}
```

Usage:

```sh
kgpn nginx                    # Find pods matching "nginx" in current namespace
klogn api staging             # Tail logs from first pod matching "api" in staging
kexn postgres production      # Exec into postgres pod in production
kpf redis 6379:6379 cache     # Port-forward to redis pod in cache namespace
kwp kube-system               # Watch pods in kube-system
krestart web-app production   # Rolling restart of web-app in production
kevents monitoring            # Recent events in monitoring namespace
```

## View Current Namespace

```sh
# Show current namespace
kubectl config view --minify | grep namespace:

# Or with jsonpath
kubectl config view --minify -o jsonpath='{..namespace}'
```

## Service Endpoints

```sh
# Get endpoints (shows pod IPs backing a service)
kubectl get endpoints
kubectl get endpoints <service-name>
kubectl get endpoints -A
```
