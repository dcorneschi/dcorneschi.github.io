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
kubectl config view --raw                        # Merged config WITH raw cert data and secrets
kubectl config get-contexts -o name              # Just the context names
kubectl config set-cluster <name> --proxy-url=<url>            # Set a proxy for a cluster entry
kubectl config set-credentials <user> --username=<u> --password=<p>   # Add a basic-auth user
kubectl config unset users.<user>                # Delete a user entry
kubectl api-resources                            # Supported API resources
kubectl api-versions                             # Supported API versions
kubectl get --raw "/" | jq                       # List all API endpoints exposed by the server
kubectl explain <resource>                       # Docs for a resource (e.g. pods, svc, deploy)
kubectl explain pod.spec.containers              # Docs for a nested field
kubectl apply -R -f .                            # Recursively apply all manifests in a directory tree
kubectl get cs --kubeconfig <file>              # Use an alternate kubeconfig for one command
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

# Sort nodes by current usage
kubectl top nodes --sort-by=cpu                  # Highest CPU first
kubectl top nodes --sort-by=memory              # Highest memory first
kubectl top nodes --no-headers | sort -k2 -n     # Lowest CPU first
kubectl top nodes --no-headers | sort -k4 -n     # Lowest memory first

# Sort pods across all namespaces
kubectl top pods -A --sort-by=cpu                # All pods, highest CPU first
kubectl top pods -A --sort-by=memory             # All pods, highest memory first

# Sort nodes by allocatable capacity (no metrics-server needed)
kubectl get nodes --sort-by='.status.allocatable.cpu'
kubectl get nodes --sort-by='.status.allocatable.memory'
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

# Compare node IPs with a known IP list
comm -12 <(kubectl get nodes -o wide --no-headers | awk '{print $6}' | sort) <(sort ip_list.txt)   # IPs active in both
comm -23 <(sort ip_list.txt) <(kubectl get nodes -o wide --no-headers | awk '{print $6}' | sort)   # IPs in list but NOT active nodes
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

# Show exact creation timestamp instead of relative age
kubectl get pods -o custom-columns="NAME:.metadata.name,CREATED:.metadata.creationTimestamp"
kubectl get pods -o custom-columns="NAME:.metadata.name,STATUS:.status.phase,CREATED:.metadata.creationTimestamp"
kubectl get pods -A -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,CREATED:.metadata.creationTimestamp"

# Count by restart
kubectl get pods -A -o json | jq '.items[] | [.status.containerStatuses[0].restartCount, .metadata.name] | join("\t")' -r | sort -n

# Pods not ready
kubectl get pods -A -o custom-columns=NAMESPACE:metadata.namespace,POD:metadata.name,READY-true:status.containerStatuses[*].ready | grep false
kubectl get pods -A -o json | jq -r '.items[] | select(.status.containerStatuses[].ready == false) | "\(.metadata.namespace) \(.metadata.name)"' | sort -u | column -t

# Pods per node (count)
kubectl get pods -A -o json | jq '.items | group_by(.spec.nodeName) | map({"nodeName": .[0].spec.nodeName, "count": length}) | sort_by(.count)'

# Pods per namespace (count)
kubectl get pods -A --no-headers | awk '{print $1}' | sort | uniq -c | awk '{print $2, $1}'

# Pods per namespace (with column alignment)
kubectl get pods -A --no-headers | awk '{print $1}' | sort | uniq -c | awk 'BEGIN {printf "%-30s %s\n", "NAMESPACE", "PODS"} {printf "%-30s %s\n", $2, $1}'

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
kubectl rollout pause deploy <deployment-name> -n <namespace>    # Pause an in-progress rollout
kubectl rollout resume deploy <deployment-name> -n <namespace>   # Resume a paused rollout
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
kubectl get sts
kubectl scale sts/<statefulset-name> --replicas=5              # Scale a StatefulSet
kubectl delete sts/<statefulset-name> --cascade=false         # Delete the StatefulSet but leave its pods running
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

# Create ConfigMaps imperatively
kubectl create configmap app-config --from-literal=ENV=production
kubectl create configmap app-config --from-literal=ENV=prod --from-literal=TIER=web
kubectl create configmap app-config --from-file=config.properties
kubectl create configmap app-config --from-file=./config-dir/     # All files in a directory

# List configmaps and secrets together
kubectl get configmaps,secrets -n <namespace>
```

## Secrets

```sh
kubectl get secrets -A
kubectl describe secrets <secret-name> -n <namespace>

# Create a generic secret with multiple keys
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Create a secret from files
kubectl create secret generic db-creds --from-file=./username.txt --from-file=./password.txt

# Decode a single field
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
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
kubectl set env daemonset aws-node -n kube-system WARM_IP_TARGET=10

# Verify a value was set
kubectl get ds aws-node -n kube-system -o yaml | grep -A1 WARM_IP_TARGET
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

# Set-based label selectors (in / notin / exists)
kubectl get pods -A -l 'env in (production, development)'   # Value in a set
kubectl get pods -A -l 'env notin (staging)'                # Value not in a set
kubectl get pods -A -l 'env'                                # Label key exists
kubectl get pods -A -l '!env'                               # Label key does NOT exist

# Add an annotation
kubectl annotate pods my-pod description="my app"

# Remove an annotation
kubectl annotate pods my-pod description-

# Find pods that have a specific annotation KEY (existence check, any value)
kubectl get pods -A -o json | jq -r '.items[].metadata | select(.annotations | has("kubernetes.io/psp")) | [.namespace, .name] | @tsv'
```

## Patching Resources

```sh
# Strategic merge patch
kubectl patch node <node-name> -p '{"spec":{"unschedulable":true}}'

# Update container image
kubectl patch pod my-pod -p '{"spec":{"containers":[{"name":"app","image":"nginx:1.25"}]}}'

# JSON patch (positional array)
kubectl patch pod my-pod --type='json' -p='[{"op":"replace","path":"/spec/containers/0/image","value":"nginx:1.25"}]'

# JSON patch — append to an array with the "-" token (e.g. add an env var to a container)
kubectl patch daemonset aws-node -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"WARM_IP_TARGET","value":"5"}}]'

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

# Non-running pods with reasons
kubectl get pods -A --field-selector=status.phase!=Running -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,STATUS:.status.phase,REASON:.status.containerStatuses[*].state.waiting.reason

# Exclude Running, Completed, and Succeeded (focus on problematic)
kubectl get pods -A | grep -v -E "(Running|Completed|Succeeded|STATUS)"

# Pods in CrashLoopBackOff specifically
kubectl get pods -A -o jsonpath='{range .items[?(@.status.containerStatuses[0].state.waiting.reason=="CrashLoopBackOff")]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{end}'

# Pods with high restart counts (>5)
kubectl get pods -A -o jsonpath='{range .items[?(@.status.containerStatuses[0].restartCount>5)]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.status.containerStatuses[0].restartCount}{"\n"}{end}'

# Non-running pods with log command hint
kubectl get pods -A --field-selector=status.phase!=Running -o jsonpath='{range .items[*]}{"kubectl logs -n "}{.metadata.namespace}{" "}{.metadata.name}{"\n"}{end}'

# Find nodes where a DaemonSet pod is NOT scheduled
ns=kube-system
pod_template=aws-node
kubectl get node | grep -v "$(kubectl -n ${ns} get pod -o wide | grep ${pod_template} | awk '{print $7}' | xargs -n 1 echo -n "\|")"
```

### Non-Running Pods Report Function

```bash
# Add to .bashrc or .zshrc
non-running-pods() {
    echo "=== Non-Running Pods Report ==="
    kubectl get pods -A --field-selector=status.phase!=Running -o custom-columns=\
NAMESPACE:.metadata.namespace,\
NAME:.metadata.name,\
STATUS:.status.phase,\
REASON:.status.containerStatuses[*].state.waiting.reason,\
AGE:.metadata.creationTimestamp

    echo -e "\n=== Summary ==="
    kubectl get pods -A --field-selector=status.phase!=Running -o jsonpath='{.items[*].status.phase}' | tr ' ' '\n' | sort | uniq -c
}
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
## Advanced jq Queries

```sh
# Ports < 1024 for all containers across all pods (privileged ports)
kubectl get pod -A -o json | jq 'try(.items[] | select((.spec.containers[].ports != null) and (.spec.containers[].ports[].containerPort < 1024)) | { "name": .metadata.name, "namespace": .metadata.namespace, "ports": .spec.containers[].ports[].containerPort })'

# Pods with a specific nodeSelector key — print the pod name and the selector value
kubectl get pods -o json | jq '.items[] | select(.spec.nodeSelector | has("<selector-key>")) | [.metadata.name, .spec.nodeSelector["<selector-key>"]] | @tsv' -r

# All pods running on a specific node (jq form)
kubectl get po -A -o json | jq '.items[] | select(.spec.nodeName == "<node-name>") | .metadata.name'

# All images across all namespaces, sorted by how many times each is used
kubectl get pods --all-namespaces -o jsonpath="{..image}" | tr -s '[[:space:]]' '\n' | sort | uniq -c | sort

# ServiceAccount annotations (namespace, name, annotations)
kubectl get sa -o json | jq '.items[].metadata | select(.annotations) | {"namespace": .namespace, "name": .name, "annotations": .annotations}'

# Any resource type carrying a specific label
kubectl get all -A -o json | jq '.items[] | select(.metadata.labels["<label-key>"] == "<label-value>") | [.metadata.name, .kind] | @tsv' -r
```
## Context and Kubeconfig Inspection

```sh
kubectl config view -o jsonpath='{.users[].name}'    # First user in kubeconfig
kubectl config view -o jsonpath='{.users[*].name}'   # All users
```

## More Node & Pod Queries

```sh
# ExternalIPs of all nodes
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# All worker nodes (exclude control-plane via negated label selector)
kubectl get node --selector='!node-role.kubernetes.io/control-plane'

# Which nodes are Ready
JSONPATH='{range .items[*]}{@.metadata.name}:{range @.status.conditions[*]}{@.type}={@.status};{end}{end}' \
  && kubectl get nodes -o jsonpath="$JSONPATH" | grep "Ready=True"

# Version label of all pods matching a selector
kubectl get pods --selector=app=<app-label> -o jsonpath='{.items[*].metadata.labels.version}'

# Secrets currently in use by pods (via secretKeyRef)
kubectl get pods -o json | jq '.items[].spec.containers[].env[]?.valueFrom.secretKeyRef.name' | grep -v null | sort | uniq

# containerIDs of all initContainers across every pod (cleanup helper)
kubectl get pods --all-namespaces -o jsonpath='{range .items[*].status.initContainerStatuses[*]}{.containerID}{"\n"}{end}' | cut -d/ -f3

# Names of pods belonging to a particular ReplicationController
sel=${$(kubectl get rc <rc-name> --output=json | jq -j '.spec.selector | to_entries | .[] | "\(.key)=\(.value),"')%?}
echo $(kubectl get pods --selector=$sel --output=jsonpath={.items..metadata.name})

# Run a command (e.g. env) across all pods that have a default container
for pod in $(kubectl get po --output=jsonpath={.items..metadata.name}); do echo $pod && kubectl exec -it $pod -- env; done
```

## Events (newer `kubectl events` command)

```sh
kubectl events --types=Warning                       # Only warning events
```

## Subresources

```sh
kubectl get deployment <deployment-name> --subresource=status   # Status subresource
```

## Replacing Resources

```sh
# Replace a resource from stdin
cat pod.json | kubectl replace -f -

# Force replace (delete + recreate — causes a service outage)
kubectl replace --force -f ./pod.json

# Bump a single-container pod's image tag in place
kubectl get pod <pod-name> -o yaml | sed 's/\(image: <image>\):.*$/\1:v4/' | kubectl replace -f -

# Expose a ReplicationController as a Service (port 80 → container 8000)
kubectl expose rc <rc-name> --port=80 --target-port=8000
```

## Scaling Resources

```sh
kubectl scale --replicas=3 rs/<name>                                # Scale a ReplicaSet
kubectl scale --replicas=3 -f foo.yaml                              # Scale a resource defined in a file
kubectl scale --current-replicas=2 --replicas=3 deployment/<name>   # Conditional scale (only if current == 2)
kubectl scale --replicas=5 rc/foo rc/bar rc/baz                     # Scale multiple controllers at once
```

## More JSON Patch Operations

```sh
# Remove an element (e.g. disable a livenessProbe)
kubectl patch deployment <name> --type json -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"}]'

# Add an element at a specific positional array index
kubectl patch sa default --type='json' -p='[{"op": "add", "path": "/secrets/1", "value": {"name": "<secret-name>"}}]'
```

## More Ways to Run & Attach to Pods

```sh
kubectl run -i --tty busybox --image=busybox:1.28 -- sh            # Run a throwaway interactive pod
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml # Generate a pod spec to a file
kubectl attach <pod-name> -i                                       # Attach to a running container
kubectl exec --stdin --tty <pod-name> -- /bin/sh                   # Interactive shell (single-container)
kubectl exec <pod-name> -c <container-name> -- ls /                # Run command in a specific container
```
## Component Status

```sh
kubectl get componentstatus                          # scheduler, controller-manager, etcd health
```

## Aggregate Resource Usage (metrics-server)

```sh
# Total memory usage across all pods
kubectl top po -A | awk '{print $4}' | sed 1d | tr -d 'Mi' | \
  awk 'BEGIN {total=0}{total+=$1}END {print "Total Memory Usage of all Pods:", total, "Mi"}'

# Total CPU usage across all pods
kubectl top po -A | awk '{print $3}' | sed 1d | tr -d 'm' | \
  awk 'BEGIN {total=0}{total+=$1}END {print "Total CPU Usage of all Pods:", total, "m"}'

# Total memory usage of all running pods on a specific node
kubectl get po -A --field-selector spec.nodeName=<node-name>,status.phase==Running -o wide | sed 1d | awk '{print $1" "$2}' | \
  while read namespace pod; do kubectl top pods --no-headers --namespace $namespace $pod; done | \
  awk '{print $3}' | tr -d 'Mi' | awk 'BEGIN {total=0}{total+=$1}END {print "Total Memory on this Node:", total, "Mi"}'

# Total CPU usage of all running pods on a specific node
kubectl get po -A --field-selector spec.nodeName=<node-name>,status.phase==Running -o wide | sed 1d | awk '{print $1" "$2}' | \
  while read namespace pod; do kubectl top pods --no-headers --namespace $namespace $pod; done | \
  awk '{print $2}' | tr -d 'm' | awk 'BEGIN {total=0}{total+=$1}END {print "Total CPU on this Node:", total, "m"}'
```

## EKS Add-On Troubleshooting (CoreDNS / VPC CNI / kube-proxy)

```sh
# List CoreDNS pods and the nodes they run on
kubectl get pod -n kube-system -o wide -l eks.amazonaws.com/component=coredns
kubectl get po -n kube-system -l k8s-app=kube-dns -o wide

# Grab a CoreDNS pod name and query its Prometheus metrics via the API proxy
COREDNS_POD=$(kubectl get pod -n kube-system -l eks.amazonaws.com/component=coredns -o jsonpath='{.items[0].metadata.name}')
kubectl get --raw /api/v1/namespaces/kube-system/pods/$COREDNS_POD:9153/proxy/metrics | grep 'coredns_dns_request_count_total'

# CoreDNS configmap and deployment
kubectl get cm coredns -o yaml -n kube-system
kubectl -n kube-system get deploy coredns -o yaml

# Add-on versions (extract image tag)
kubectl describe deployment coredns  --namespace kube-system | grep Image | cut -d "/" -f 3    # CoreDNS
kubectl describe daemonset aws-node   --namespace kube-system | grep Image | cut -d "/" -f 2    # VPC CNI
kubectl describe daemonset kube-proxy --namespace kube-system | grep Image | cut -d "/" -f 3    # kube-proxy

# Dump logs from all CoreDNS pods
for p in $(kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o name); do kubectl logs --namespace=kube-system $p; done

# Extract CoreDNS logs from every pod into a file
for i in $(kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o name); do echo $i; kubectl logs -n kube-system $i -c coredns >> corednslogs.txt; done

# Dump logs from all running aws-node (VPC CNI) pods
for i in $(kubectl get pods -n kube-system -o wide -l k8s-app=aws-node | egrep "aws-node" | grep Running | awk '{print $1}'); do echo $i; kubectl logs $i -n kube-system; echo; done
```
## Resource Types & API Objects

```sh
kubectl get crd                                  # List all CustomResourceDefinitions
kubectl get storageclass                         # List storage classes
kubectl get resourcequota -A                     # List resource quotas
kubectl get limitrange -A                        # List limit ranges
kubectl get csr                                  # List certificate signing requests
kubectl get networkpolicy -A                     # List network policies
```

## Quotas, Limits & Resource Requests

```sh
# Set container resource limits on a deployment
kubectl set resources deployment nginx -c=nginx --limits=cpu=200m
kubectl set resources deployment nginx -c=nginx --limits=cpu=200m,memory=512Mi

# Set requests as well as limits
kubectl set resources deployment nginx -c=nginx --requests=cpu=100m,memory=256Mi --limits=cpu=500m,memory=512Mi
```

## Exposing Services

```sh
# Expose a deployment as a LoadBalancer service
kubectl expose deployment/my-app --type=LoadBalancer --name=my-service

# Expose an existing service as a new LoadBalancer service
kubectl expose service/my-svc --type=LoadBalancer --name=my-lb

# Patch an existing service to type LoadBalancer
kubectl patch svc <service-name> -p '{"spec":{"type":"LoadBalancer"}}'

# Get a service's ClusterIP / first port via go-template
kubectl get service <service-name> -o go-template='{{.spec.clusterIP}}'
kubectl get service <service-name> -o go-template='{{(index .spec.ports 0).port}}'
```

## Watching & Extra Port-Forward Targets

```sh
kubectl get pods -n <namespace> --watch            # Stream pod state changes
kubectl port-forward rs/<replicaset-name> 6379:6379   # Port-forward to a ReplicaSet
```

## Pod initContainer Status

```sh
# Print the initContainer statuses for a pod
kubectl get pod <pod-name> --template '{{.status.initContainerStatuses}}'
```
## Resource Shortnames

Common shortnames accepted anywhere a resource type is expected (e.g. `kubectl get po`):

| Shortname | Full resource |
|-----------|---------------|
| `po` | pods |
| `deploy` | deployments |
| `rs` | replicasets |
| `svc` | services |
| `ns` | namespaces |
| `cm` | configmaps |
| `pv` | persistentvolumes |
| `pvc` | persistentvolumeclaims |
| `ing` | ingresses |
| `no` | nodes |
| `sa` | serviceaccounts |
| `ds` | daemonsets |
| `sts` | statefulsets |
| `netpol` | networkpolicies |

```sh
kubectl api-resources    # Authoritative list of all types + their shortnames
```

## Ephemeral & Throwaway Debug Pods

```sh
# Attach an ephemeral debug container to a running pod, sharing a target container's namespace (K8s 1.25+)
kubectl debug -it <pod-name> --image=busybox --target=<container-name>

# One-off throwaway pod for DNS resolution checks (auto-deleted on exit)
kubectl run debug --rm -it --image=busybox -- nslookup kubernetes

# One-off throwaway pod for HTTP connectivity checks
kubectl run debug --rm -it --image=curlimages/curl -- curl -v http://<service>:80
```

## Quick Diagnostic Greps

```sh
# Just the Events block from a describe
kubectl describe pod <pod-name> | grep -A10 "Events:"

# Container state (handy for CrashLoopBackOff)
kubectl describe pod <pod-name> | grep -A5 "State:"

# Allocated resources per node
kubectl describe nodes | grep -A5 "Allocated resources"

# Recent events, newest last
kubectl get events --sort-by='.lastTimestamp' | tail -30
```

## Expose as ClusterIP

```sh
kubectl expose deploy nginx --port=80 --type=ClusterIP     # Default service type (internal only)
```
