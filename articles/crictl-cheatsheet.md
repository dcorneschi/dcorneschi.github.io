<p align="center">
  <img src="/articles/images/kubernetes-logo.svg" alt="Kubernetes Logo" width="200"/>
</p>

<h1 align="center">critctl Cheatsheet</h1>

Comprehensive crictl reference guide featuring configuration examples, pod and container management commands, and practical tips for debugging Kubernetes nodes at the container runtime level.

## Configuration

### Set Runtime Endpoint

```bash
# Set containerd endpoint
crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock

# Set CRI-O endpoint
crictl config runtime-endpoint unix:///var/run/crio/crio.sock

# Set Docker endpoint (if using dockershim)
crictl config runtime-endpoint unix:///var/run/dockershim.sock
```

### Configuration File

```bash
# View current config
crictl config --list

# Edit config file (usually at /etc/crictl.yaml)
cat /etc/crictl.yaml
```

## Container Management

### List Containers

```bash
# List all containers
crictl ps

# List all containers (including stopped)
crictl ps -a

# List containers with additional info
crictl ps -v

# Filter by state
crictl ps --state running
crictl ps --state exited
crictl ps --state created

# Filter by pod
crictl ps --pod <pod-id>
```

### Container Information

```bash
# Get container details
crictl inspect <container-id>

# Get container stats
crictl stats
crictl stats <container-id>

# Get container logs
crictl logs <container-id>
crictl logs -f <container-id>  # Follow logs
crictl logs --tail 100 <container-id>  # Last 100 lines
crictl logs --since 1h <container-id>  # Logs from last hour
```

### Container Operations

```bash
# Start a container
crictl start <container-id>

# Stop a container
crictl stop <container-id>

# Remove a container
crictl rm <container-id>

# Remove all stopped containers
crictl rm $(crictl ps -aq --state exited)

# Execute command in container
crictl exec -it <container-id> /bin/bash
crictl exec -it <container-id> sh
crictl exec <container-id> ls -la

# Attach to container
crictl attach <container-id>
```

## Pod Management

### List Pods

```bash
# List all pods
crictl pods

# List pods with verbose output
crictl pods -v

# Filter by state
crictl pods --state ready
crictl pods --state notready

# Filter by namespace
crictl pods --namespace kube-system

# Filter by label
crictl pods --label app=nginx
```

### Pod Information

```bash
# Get pod details
crictl inspectp <pod-id>

# Get pod stats
crictl statsp <pod-id>
```

### Pod Operations

```bash
# Stop a pod
crictl stopp <pod-id>

# Remove a pod
crictl rmp <pod-id>

# Run a pod from config file
crictl runp pod-config.yaml
```

## Image Management

### List Images

```bash
# List all images
crictl images

# List images with verbose output
crictl images -v
```

### Image Operations

```bash
# Pull an image
crictl pull nginx:latest
crictl pull docker.io/library/nginx:latest

# Remove an image
crictl rmi <image-id>
crictl rmi nginx:latest

# Get image details
crictl inspecti <image-id>
```

## Runtime Information

### System Information

```bash
# Get runtime version
crictl version

# Get runtime info
crictl info
```

## Debugging and Troubleshooting

### Port Forwarding

```bash
# Port forward from pod
crictl port-forward <pod-id> 8080:80
```

## Advanced Usage

### JSON Output

```bash
# Get output in JSON format
crictl ps -o json
crictl pods -o json
crictl images -o json
```

### Batch Operations

```bash
# Stop all running containers
crictl ps -q | xargs crictl stop

# Remove all exited containers
crictl ps -aq --state exited | xargs crictl rm

# Remove unused images
crictl images -q | xargs crictl rmi
```

## Common Filters and Options

### Output Options

```bash
# Quiet mode (IDs only)
crictl ps -q
crictl pods -q
crictl images -q
```

## Useful Aliases

```bash
# Add to ~/.bashrc or ~/.zshrc
alias cps='crictl ps'
alias cpsa='crictl ps -a'
alias cpods='crictl pods'
alias cimages='crictl images'
alias cexec='crictl exec -it'
alias clogs='crictl logs'
```

## Common Use Cases

### Clean Up Resources

```bash
# Clean up stopped containers
crictl ps -a | grep Exited | awk '{print $1}' | xargs crictl rm

# Clean up unused images
crictl images | grep '<none>' | awk '{print $3}' | xargs crictl rmi
```

### Monitor Resources

```bash
# Real-time container stats
watch crictl stats

# Check image sizes
crictl images --digests
```

## Notes
- crictl is primarily for debugging and should not be used for production container management
- Always use Kubernetes APIs (kubectl) for production operations
- Some commands may require elevated privileges depending on the runtime socket permissions
- Container IDs and Pod IDs are typically shortened in output but full IDs work for all operations

## Links 

* [https://github.com/kubernetes-sigs/cri-tools](https://github.com/kubernetes-sigs/cri-tools)
