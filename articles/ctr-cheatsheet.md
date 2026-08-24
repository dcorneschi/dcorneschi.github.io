# ctr Cheatsheet (containerd CLI)

`ctr` is the low-level CLI for containerd — the container runtime used by Kubernetes (EKS, GKE, self-managed). Unlike `crictl` (which speaks the CRI API), `ctr` talks directly to containerd's gRPC API.

## Basic Structure

```sh
ctr [global options] command [command options] [arguments...]
```

## Global Options

| Flag | Description |
|------|-------------|
| `--address`, `-a` | Address for containerd's gRPC server (default: `/run/containerd/containerd.sock`) |
| `--timeout` | Total timeout for ctr commands |
| `--connect-timeout` | Timeout for connecting to containerd |
| `--namespace`, `-n` | Namespace to use (default: `default`) |
| `--debug` | Enable debug output |

> Kubernetes uses the `k8s.io` namespace. To see Kubernetes containers: `ctr -n k8s.io containers list`

## Image Management

### Pull Images

```sh
# Pull an image
ctr images pull docker.io/library/nginx:latest

# Pull with specific platform
ctr images pull --platform linux/amd64 docker.io/library/nginx:latest

# Pull all platforms
ctr images pull --all-platforms docker.io/library/nginx:latest
```

### List Images

```sh
# List all images
ctr images list

# Quiet output (image names only)
ctr images list -q

# List with digests
ctr images list --digests

# List Kubernetes images
ctr -n k8s.io images list
```

### Remove Images

```sh
# Remove an image
ctr images remove docker.io/library/nginx:latest

# Remove multiple images
ctr images remove nginx:latest ubuntu:20.04
```

### Tag Images

```sh
# Tag an image
ctr images tag docker.io/library/nginx:latest my-registry/nginx:v1.0
```

### Import/Export Images

```sh
# Export image to tar
ctr images export nginx.tar docker.io/library/nginx:latest

# Export with all platforms
ctr images export --all-platforms nginx.tar docker.io/library/nginx:latest

# Import image from tar
ctr images import nginx.tar
```

## Container Management

### Create Containers

```sh
# Create a container
ctr containers create docker.io/library/nginx:latest nginx-container

# Create with specific runtime
ctr containers create --runtime io.containerd.runc.v2 docker.io/library/nginx:latest my-container
```

### List Containers

```sh
# List all containers
ctr containers list

# Quiet output (names only)
ctr containers list -q

# List Kubernetes containers
ctr -n k8s.io containers list
```

### Delete Containers

```sh
# Delete a container
ctr containers delete nginx-container

# Force delete
ctr containers delete --force nginx-container
```

### Container Information

```sh
# Get container info (full spec)
ctr containers info nginx-container

# Get container labels
ctr containers label nginx-container
```

## Task Management (Running Containers)

A **task** is a running instance of a container. Containers are just metadata; tasks are the actual processes.

### Run (Create + Start in One Step)

```sh
# Run interactively and remove when done
ctr run --rm -t docker.io/library/ubuntu:22.04 temp-container bash

# Run detached
ctr run -d docker.io/library/nginx:latest web-server

# Run with environment variables
ctr run --env "VAR1=value1" --env "VAR2=value2" docker.io/library/ubuntu:22.04 my-container bash

# Run with host network
ctr run --net-host docker.io/library/nginx:latest web-host

# Run with no network
ctr run --net-none docker.io/library/ubuntu:22.04 isolated bash

# Run with memory limit
ctr run --memory-limit 536870912 docker.io/library/nginx:latest limited-web
```

### Start a Task (From Existing Container)

```sh
# Start task (detached)
ctr tasks start nginx-container

# Start interactively
ctr tasks start -t nginx-container
```

### List Tasks

```sh
# List running tasks
ctr tasks list

# List Kubernetes tasks
ctr -n k8s.io tasks list
```

### Task Operations

```sh
# Kill a task
ctr tasks kill nginx-container

# Kill with specific signal
ctr tasks kill --signal SIGTERM nginx-container
ctr tasks kill --signal SIGKILL nginx-container

# Pause a task
ctr tasks pause nginx-container

# Resume a task
ctr tasks resume nginx-container

# Delete a task (must be stopped first)
ctr tasks delete nginx-container

# Execute command in running task
ctr tasks exec --exec-id my-session nginx-container bash

# Attach to running task
ctr tasks attach nginx-container
```

### Task Metrics

```sh
# Get task metrics (CPU, memory)
ctr tasks metrics nginx-container

# Get task processes
ctr tasks ps nginx-container
```

## Namespace Management

Containerd uses namespaces to isolate resources. Kubernetes uses the `k8s.io` namespace.

```sh
# List namespaces
ctr namespaces list

# Create namespace
ctr namespaces create my-namespace

# Remove namespace
ctr namespaces remove my-namespace

# Use specific namespace for any command
ctr --namespace k8s.io images list
ctr -n k8s.io containers list
ctr -n k8s.io tasks list
```

## Content Management

```sh
# List content (image layers, manifests)
ctr content list

# Get content info
ctr content info sha256:<digest>

# Delete content
ctr content delete sha256:<digest>

# Fetch content (download without creating image)
ctr content fetch docker.io/library/nginx:latest
```

## Snapshot Management

```sh
# List snapshots
ctr snapshots list

# Create snapshot
ctr snapshots prepare my-snapshot parent-snapshot

# Remove snapshot
ctr snapshots remove my-snapshot

# Get snapshot disk usage
ctr snapshots usage my-snapshot

# List Kubernetes snapshots
ctr -n k8s.io snapshots list
```

## Plugin Management

```sh
# List plugins
ctr plugins list

# Filter by type
ctr plugins list | grep -i runtime
```

## Complete Container Lifecycle

```sh
# 1. Pull image
ctr images pull docker.io/library/nginx:latest

# 2. Create container
ctr containers create docker.io/library/nginx:latest web-server

# 3. Start task
ctr tasks start web-server

# 4. Check it's running
ctr tasks list

# 5. Exec into it
ctr tasks exec --exec-id debug web-server sh

# 6. Stop it
ctr tasks kill web-server

# 7. Delete task
ctr tasks delete web-server

# 8. Delete container
ctr containers delete web-server

# 9. Remove image (optional)
ctr images remove docker.io/library/nginx:latest
```

## ctr vs crictl vs docker

| Feature | `ctr` | `crictl` | `docker` |
|---------|:-----:|:--------:|:--------:|
| Talks to | containerd gRPC API | CRI API (any CRI runtime) | Docker daemon |
| Namespace support | Yes (`-n k8s.io`) | N/A (always CRI namespace) | N/A |
| Image management | Full | Limited | Full |
| Container lifecycle | Full | Read + exec only | Full |
| Used for | Low-level debugging, image ops | Kubernetes troubleshooting | Docker-specific workflows |
| On EKS nodes | Yes (containerd is the runtime) | Yes (CRI is exposed) | No (Docker removed) |

## Debugging Kubernetes Containers with ctr

```sh
# Check containerd service status
systemctl status containerd

# See what Kubernetes sees (use k8s.io namespace)
ctr -n k8s.io images list
ctr -n k8s.io containers list
ctr -n k8s.io tasks list

# Find a specific pod's containers
ctr -n k8s.io containers list | grep <pod-name>

# Get container info (full OCI spec)
ctr -n k8s.io containers info <container-id>

# Check image layers
ctr -n k8s.io content list | head -20

# Check snapshot usage (disk)
ctr -n k8s.io snapshots list
```

### Inspecting Container Mounts and Storage

```sh
# Find mount points for a specific container
cat /proc/mounts | grep <container-id>

# Check overlay filesystem disk usage
df -h | grep overlay

# Check all overlay mounts for a container
mount | grep <container-id>
```

## Quick Reference

```sh
# Images
ctr images pull <image>
ctr images list
ctr images remove <image>
ctr images export <file.tar> <image>
ctr images import <file.tar>
ctr images tag <source> <target>

# Containers
ctr containers create <image> <name>
ctr containers list
ctr containers info <name>
ctr containers delete <name>

# Tasks (running containers)
ctr tasks start <container>
ctr tasks list
ctr tasks exec --exec-id <id> <container> <cmd>
ctr tasks kill <container>
ctr tasks delete <container>

# Quick run
ctr run --rm -t <image> <name> <command>

# Kubernetes namespace
ctr -n k8s.io <any command>
```

## Important Notes

- Always clean up tasks before deleting containers (`kill` → `delete task` → `delete container`)
- Use `--rm` flag for temporary containers
- Kubernetes containers live in the `k8s.io` namespace — always use `-n k8s.io`
- Tasks are the running instances; containers are just the configuration
- `ctr` requires root access (or access to the containerd socket)
- Unlike `crictl`, `ctr` can create/modify containers — use carefully on production nodes
