# Installing Podman on RHEL 7–10

Podman is a daemonless container engine for developing, managing, and running OCI containers. This guide covers installation across all supported RHEL versions.

## Version Matrix

| RHEL Version | Default Podman Version | Container Tools Stream |
|--------------|----------------------|------------------------|
| RHEL 7 | 1.6.x (via Extras) | N/A |
| RHEL 8 | 4.x (module stream) | container-tools:rhel8 |
| RHEL 9 | 4.x–5.x | container-tools (AppStream) |
| RHEL 10 | 5.x+ | container-tools (AppStream) |

## RHEL 7

Podman on RHEL 7 is available through the Extras repository. Note that RHEL 7 reached end of maintenance in June 2024.

### Install

```bash
# Enable the Extras repo (if not already enabled)
sudo subscription-manager repos --enable=rhel-7-server-extras-rpms

# Install podman
sudo yum install -y podman

# Verify
podman --version
podman info
```

### Notes for RHEL 7

- Podman on RHEL 7 is version 1.6.x — significantly older, missing many features
- No support for `podman compose` or pods
- No rootless containers (requires RHEL 8+)
- cgroups v1 only
- Consider upgrading to RHEL 8+ for full Podman functionality

## RHEL 8

RHEL 8 uses Application Streams (modules) for container tools.

### Install (Default Stream)

```bash
# Install the container-tools module (includes podman, buildah, skopeo)
sudo dnf module install -y container-tools

# Or install podman individually
sudo dnf install -y podman

# Verify
podman --version
podman info
```

### Install Specific Module Stream

```bash
# List available streams
sudo dnf module list container-tools

# Reset module (if switching streams)
sudo dnf module reset container-tools

# Install a specific stream version
sudo dnf module install -y container-tools:rhel8

# Or enable and install
sudo dnf module enable -y container-tools:rhel8
sudo dnf install -y podman
```

### Install Latest Available Version

```bash
# The rhel8 stream typically has the latest stable version
sudo dnf module reset container-tools
sudo dnf module install -y container-tools:rhel8

podman --version
```

### Enable Rootless Containers

```bash
# Rootless is enabled by default on RHEL 8
# Verify user namespaces are enabled
sysctl user.max_user_namespaces
# Should be > 0 (default: 28633)

# If not set, enable it
echo "user.max_user_namespaces = 28633" | sudo tee /etc/sysctl.d/userns.conf
sudo sysctl --system

# Verify subuid/subgid entries exist for your user
grep $USER /etc/subuid
grep $USER /etc/subgid

# If missing, add them
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER

# Apply changes
podman system migrate
```

## RHEL 9

Container tools are available directly from AppStream without module streams.

### Install

```bash
# Install podman
sudo dnf install -y podman

# Install full container tools suite (podman, buildah, skopeo, crun)
sudo dnf install -y container-tools

# Verify
podman --version
podman info
```

### Install Additional Tools

```bash
# Buildah (build images without Dockerfile)
sudo dnf install -y buildah

# Skopeo (inspect and copy container images)
sudo dnf install -y skopeo

# Podman Compose (docker-compose compatibility)
sudo dnf install -y podman-compose

# Container networking plugins
sudo dnf install -y containernetworking-plugins

# Podman Docker compatibility (provides /usr/bin/docker alias)
sudo dnf install -y podman-docker
```

### Rootless Setup (Default on RHEL 9)

```bash
# Rootless works out of the box on RHEL 9
# Verify
podman unshare cat /proc/self/uid_map

# Check user namespace support
sysctl user.max_user_namespaces

# Check subuid/subgid
cat /etc/subuid
cat /etc/subgid
```

## RHEL 10

RHEL 10 ships with Podman 5.x and uses cgroups v2 by default.

### Install

```bash
# Install podman
sudo dnf install -y podman

# Install full container tools suite
sudo dnf install -y container-tools

# Verify
podman --version
podman info
```

### New Features in Podman 5.x (RHEL 10)

```bash
# Podman machine (for running Linux containers on non-Linux hosts)
podman machine init
podman machine start

# Improved compose support
podman compose up -d

# Quadlet (systemd integration for containers)
# Place .container files in ~/.config/containers/systemd/ (rootless)
# or /etc/containers/systemd/ (rootful)

# Hypervisor-based containers (confidential containers)
podman run --runtime=kata ...
```

## Post-Installation Steps (All Versions)

### Verify Installation

```bash
# Check version
podman --version

# System info
podman info

# Run test container
podman run --rm hello-world

# Run interactive container
podman run -it --rm registry.access.redhat.com/ubi9/ubi bash
```

### Configure Registries

```bash
# Edit registries configuration
sudo vi /etc/containers/registries.conf

# Common registries to add under [registries.search]
# registries = ['registry.access.redhat.com', 'registry.redhat.io', 'docker.io', 'quay.io']
```

Example `/etc/containers/registries.conf`:

```toml
[registries.search]
registries = ['registry.access.redhat.com', 'registry.redhat.io', 'docker.io', 'quay.io']

[registries.insecure]
registries = []

[registries.block]
registries = []
```

### Configure Storage

```bash
# View storage configuration
cat /etc/containers/storage.conf

# Default storage location:
# Root:     /var/lib/containers/storage
# Rootless: ~/.local/share/containers/storage

# Change storage driver (if needed)
# Edit /etc/containers/storage.conf:
# [storage]
# driver = "overlay"
```

### Authenticate to Registries

```bash
# Login to Red Hat registry
podman login registry.redhat.io

# Login to Docker Hub
podman login docker.io

# Login to custom registry
podman login myregistry.example.com

# Credentials stored in:
# ${XDG_RUNTIME_DIR}/containers/auth.json (rootless)
# /run/containers/0/auth.json (root)
```

### Enable Podman Socket (Docker API Compatibility)

```bash
# Rootless (user service)
systemctl --user enable --now podman.socket
systemctl --user status podman.socket

# Set DOCKER_HOST for tools expecting Docker socket
export DOCKER_HOST=unix:///run/user/$(id -u)/podman/podman.sock

# Root (system service)
sudo systemctl enable --now podman.socket
sudo systemctl status podman.socket
```

### Auto-Start Containers with systemd

```bash
# Generate systemd unit file from running container
podman generate systemd --new --name mycontainer > ~/.config/systemd/user/mycontainer.service

# Reload and enable
systemctl --user daemon-reload
systemctl --user enable --now mycontainer.service

# For root containers
sudo podman generate systemd --new --name mycontainer > /etc/systemd/system/mycontainer.service
sudo systemctl daemon-reload
sudo systemctl enable --now mycontainer.service
```

### Quadlet (RHEL 9.2+ / Podman 4.4+)

```bash
# Create a .container file
cat > ~/.config/containers/systemd/webapp.container << 'EOF'
[Container]
Image=docker.io/library/nginx:latest
PublishPort=8080:80
Volume=./html:/usr/share/nginx/html:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
EOF

# Reload systemd to pick up quadlet files
systemctl --user daemon-reload

# Start the service
systemctl --user start webapp.service
systemctl --user status webapp.service
```

## Docker Migration

### Install Docker Compatibility Layer

```bash
# Provides /usr/bin/docker symlink pointing to podman
sudo dnf install -y podman-docker

# docker commands now use podman
docker run hello-world
docker ps
docker images
```

### Key Differences from Docker

| Feature | Docker | Podman |
|---------|--------|--------|
| Daemon | dockerd (required) | Daemonless |
| Root required | Yes (default) | No (rootless by default) |
| Socket | /var/run/docker.sock | /run/user/UID/podman/podman.sock |
| Compose | docker-compose / docker compose | podman-compose / podman compose |
| Systemd integration | Limited | Native (Quadlet) |
| Pod support | No (Swarm/K8s) | Yes (podman pod) |
| Build tool | docker build | podman build / buildah |
| cgroups | v1 or v2 | v2 (default on RHEL 9+) |

### Migrate Docker Compose Files

```bash
# Install podman-compose
sudo dnf install -y podman-compose

# Run existing docker-compose.yml
podman-compose up -d

# Or use podman compose (built-in, requires compose plugin)
podman compose up -d
```

## Troubleshooting

### "Permission denied" on Rootless

```bash
# Check user namespaces
sysctl user.max_user_namespaces
# If 0, enable it:
sudo sysctl -w user.max_user_namespaces=28633
echo "user.max_user_namespaces = 28633" | sudo tee /etc/sysctl.d/userns.conf

# Check subuid/subgid
grep $USER /etc/subuid /etc/subgid
# If missing:
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER
podman system migrate
```

### "ERRO[0000] cannot find UID/GID"

```bash
# Reset podman storage
podman system reset

# Re-run migration
podman system migrate
```

### Slow Image Pulls

```bash
# Check DNS resolution
podman run --rm busybox nslookup registry.access.redhat.com

# Use specific DNS
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf

# Check proxy settings
env | grep -i proxy
```

### Container Networking Issues

```bash
# Check CNI plugins (RHEL 8)
rpm -qa | grep containernetworking

# Install if missing
sudo dnf install -y containernetworking-plugins

# Reset network configuration
podman network rm podman
podman network create podman

# Check firewall (if containers can't reach external networks)
sudo firewall-cmd --zone=public --add-masquerade --permanent
sudo firewall-cmd --reload
```

### Storage Issues

```bash
# Check available space
df -h /var/lib/containers
df -h ~/.local/share/containers

# Clean up unused images and containers
podman system prune -a

# Reset all storage (DESTRUCTIVE)
podman system reset
```

### SELinux Denials

```bash
# Check for SELinux denials
sudo ausearch -m avc -ts recent | grep container

# Use :Z or :z for bind mounts
podman run -v /host/path:/container/path:Z myimage

# :Z = private relabel (single container)
# :z = shared relabel (multiple containers)
```

## Quick Reference

| Action | Command |
|--------|---------|
| Install (RHEL 7) | `sudo yum install -y podman` |
| Install (RHEL 8) | `sudo dnf module install -y container-tools` |
| Install (RHEL 9/10) | `sudo dnf install -y podman` |
| Check version | `podman --version` |
| System info | `podman info` |
| Run test container | `podman run --rm hello-world` |
| Enable socket | `systemctl --user enable --now podman.socket` |
| Docker compat | `sudo dnf install -y podman-docker` |
| Reset storage | `podman system reset` |
| Clean up | `podman system prune -a` |
