<p align="center">
  <img src="/articles/images/docker-logo.svg" alt="Docker logo" width="200"/>
</p>

<h1 align="center">Docker Cheatsheet</h1>

## Container Lifecycle

```bash
# Create a container (without starting)
docker create --name myapp nginx:latest

# Run a container (create + start)
docker run -d --name myapp nginx:latest

# Run with port mapping
docker run -d -p 8080:80 --name web nginx

# Run with volume mount
docker run -d -v /host/path:/container/path nginx

# Run with environment variables
docker run -d -e "DB_HOST=localhost" -e "DB_PORT=5432" myapp

# Run interactively
docker run -it ubuntu:22.04 bash

# Run and remove on exit
docker run --rm -it alpine sh

# Start a stopped container
docker start myapp

# Stop a container (SIGTERM, then SIGKILL after timeout)
docker stop myapp

# Stop with custom timeout (seconds)
docker stop -t 30 myapp

# Kill a container (SIGKILL immediately)
docker kill myapp

# Restart a container
docker restart myapp

# Pause / unpause
docker pause myapp
docker unpause myapp

# Remove a container
docker rm myapp

# Remove a running container (force)
docker rm -f myapp

# Forcefully remove all containers (including running ones)
docker container rm --force $(docker container ls --all --quiet)

# Remove all stopped containers
docker container prune

# Rename a container
docker rename old_name new_name
```

## Container Inspection

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# List only container IDs
docker ps -q

# List without truncation (full IDs, commands)
docker ps --no-trunc

# List last N created containers
docker ps -a -n 2

# Filter containers by status
docker ps -a -f "status=exited"
docker ps -a -f "status=running"

# List with custom format
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Inspect container details (JSON)
docker inspect myapp

# Get specific field from inspect
docker inspect -f '{{.State.Status}}' myapp
docker inspect -f '{{.NetworkSettings.IPAddress}}' myapp
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' myapp

# View container logs
docker logs myapp

# Follow logs (tail)
docker logs -f myapp

# Logs with timestamps
docker logs -t myapp

# Last N lines
docker logs --tail 100 myapp

# Logs since a time
docker logs --since 2h myapp
docker logs --since "2024-01-01T00:00:00" myapp

# View resource usage (live)
docker stats

# Stats for specific container
docker stats myapp

# One-shot stats (no stream)
docker stats --no-stream

# View running processes inside container
docker top myapp

# View port mappings
docker port myapp

# View filesystem changes
docker diff myapp

# Wait for container to stop and get exit code
docker wait myapp
```

## Executing Commands in Containers

```bash
# Execute command in running container
docker exec myapp ls /app

# Interactive shell
docker exec -it myapp bash
docker exec -it myapp sh

# Execute as specific user
docker exec -u root -it myapp bash
docker exec -u 1000:1000 -it myapp bash

# Execute with environment variable
docker exec -e "DEBUG=true" myapp ./script.sh

# Execute with working directory
docker exec -w /app myapp ls
```

## Images

```bash
# List local images
docker images
docker image ls

# List with specific format
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# List dangling images
docker images -f dangling=true

# Pull an image
docker pull nginx
docker pull nginx:1.25
docker pull nginx@sha256:abc123...

# Pull with full registry path
docker pull docker.io/library/redis:alpine

# Build an image
docker build -t myapp:1.0 .

# Build with specific Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Build with build arguments
docker build --build-arg VERSION=1.0 -t myapp .

# Build without cache
docker build --no-cache -t myapp .

# Build for specific platform
docker build --platform linux/amd64 -t myapp .

# Tag an image
docker tag myapp:1.0 registry.example.com/myapp:1.0

# Tag methods
docker build -t user/repo:tag .            # at build time
docker tag existing-image user/repo:tag    # existing image
docker commit container-id user/repo:tag   # from running container

# Push an image
docker push registry.example.com/myapp:1.0

# Remove an image
docker rmi nginx:latest

# Remove unused images
docker image prune

# Remove all unused images (not just dangling)
docker image prune -a

# Save image to tar
docker save -o myapp.tar myapp:1.0

# Load image from tar
docker load -i myapp.tar

# Export container filesystem to tar
docker export myapp > myapp-fs.tar

# Import filesystem as image
docker import myapp-fs.tar myapp:imported

# View image history (layers)
docker history myapp:1.0

# Inspect image details
docker image inspect nginx:latest
```

## Volumes

```bash
# Create a volume
docker volume create mydata

# List volumes
docker volume ls

# Inspect a volume
docker volume inspect mydata

# Remove a volume
docker volume rm mydata

# Remove unused volumes
docker volume prune

# Run with named volume
docker run -d -v mydata:/app/data myapp

# Run with bind mount
docker run -d -v /host/path:/container/path myapp

# Run with read-only mount
docker run -d -v /host/path:/container/path:ro myapp

# Run with tmpfs mount
docker run -d --tmpfs /tmp:rw,size=100m myapp

# Copy files to/from container
docker cp file.txt myapp:/app/
docker cp myapp:/app/file.txt ./
docker cp myapp:/app/logs/ ./local-logs/
```

## Networks

```bash
# List networks
docker network ls

# Create a network
docker network create mynet

# Create with specific driver
docker network create --driver bridge mynet

# Create with subnet
docker network create --subnet=172.20.0.0/16 mynet

# Inspect a network
docker network inspect mynet

# Connect container to network
docker network connect mynet myapp

# Disconnect container from network
docker network disconnect mynet myapp

# Run container on specific network
docker run -d --network mynet --name myapp nginx

# Run with static IP
docker run -d --network mynet --ip 172.20.0.10 --name myapp nginx

# Run with hostname
docker run -d --network mynet --hostname myapp.local --name myapp nginx

# Remove a network
docker network rm mynet

# Remove unused networks
docker network prune

# DNS resolution (containers on same network resolve by name)
docker run -d --network mynet --name db postgres
docker run -d --network mynet --name app myapp
# app can reach db at hostname "db"
```

## Docker Compose

### Install Docker Compose (Standalone)

```bash
# Download latest version
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Verify
docker-compose --version
```

### Docker Compose Without sudo

```bash
groupadd docker
usermod -aG docker $USER
chown root:docker /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Test
docker run hello-world
docker-compose --version
```

### Commands

```bash
# Start services (detached)
docker compose up -d

# Start with build
docker compose up -d --build

# Start specific service
docker compose up -d web

# Force recreate a single service (without restarting dependencies)
docker compose up -d --force-recreate --no-deps gitea

# Force recreate a service (restart it and dependencies)
docker compose up -d --force-recreate nagios

# Start with specific env file
docker compose --env-file .env.local up -d traefik

# Stop services
docker compose down

# Stop and remove volumes
docker compose down -v

# Stop and remove images
docker compose down --rmi all

# View running services
docker compose ps

# View logs
docker compose logs
docker compose logs -f web

# Logs without color and limited lines
docker compose logs --no-color --tail=200 traefik

# View resolved config with env-file (check variable substitution)
docker compose --env-file .env config | grep -E 'VAR1|VAR2'

# Execute command in service
docker compose exec web bash

# Run one-off command in service
docker compose run --rm web bash

# Scale a service
docker compose up -d --scale worker=3

# Restart a service
docker compose restart web

# Pull latest images
docker compose pull

# Build images
docker compose build
docker compose build --no-cache

# View compose config (merged)
docker compose config

# List images used by services
docker compose images
```

## Docker Compose File (docker-compose.yml)

```yaml
version: "3.8"

services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    environment:
      - NGINX_HOST=example.com
    depends_on:
      - app
    restart: unless-stopped
    networks:
      - frontend

  app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - frontend
      - backend

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=mydb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - backend

volumes:
  pgdata:

networks:
  frontend:
  backend:
```

## Dockerfile Reference

```dockerfile
# Base image
FROM ubuntu:22.04

# Labels
LABEL maintainer="user@example.com"
LABEL version="1.0"

# Arguments (build-time only)
ARG DEBIAN_FRONTEND=noninteractive
ARG APP_VERSION=1.0

# Environment variables (persist at runtime)
ENV APP_HOME=/app
ENV PATH="${APP_HOME}/bin:${PATH}"

# Working directory
WORKDIR /app

# Install packages
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Copy files
COPY package.json package-lock.json ./
COPY src/ ./src/

# Add files (supports URLs and auto-extracts tarballs)
ADD https://example.com/app.tar.gz /tmp/

# Run commands
RUN npm ci --production

# Expose ports (documentation only)
EXPOSE 3000

# Create non-root user
RUN useradd -r -s /bin/false appuser
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# Default command
CMD ["node", "src/index.js"]

# Alternative: entrypoint + cmd
ENTRYPOINT ["node"]
CMD ["src/index.js"]
```

### Multi-stage Build

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
USER node
CMD ["node", "dist/index.js"]
```

## Registry and Login

```bash
# Login to Docker Hub
docker login

# Login to private registry
docker login registry.example.com

# Login with credentials (CI/CD)
echo "$TOKEN" | docker login registry.example.com -u user --password-stdin

# Logout
docker logout
docker logout registry.example.com

# Search Docker Hub
docker search nginx

# Pull from private registry
docker pull registry.example.com/myapp:1.0

# Push to private registry
docker tag myapp:1.0 registry.example.com/myapp:1.0
docker push registry.example.com/myapp:1.0
```

## System and Cleanup

```bash
# View disk usage
docker system df

# Detailed disk usage
docker system df -v

# Remove all unused data (containers, images, networks, volumes)
docker system prune

# Including volumes
docker system prune --volumes

# Including all unused images (not just dangling)
docker system prune -a

# Remove stopped containers
docker container prune

# Remove unused images
docker image prune -a

# Remove unused volumes
docker volume prune

# Remove unused networks
docker network prune

# View Docker daemon info
docker info

# View Docker version
docker version
```

## Run Options Reference

| Flag | Description |
|------|-------------|
| `-d` | Detached mode (background) |
| `-it` | Interactive + TTY (shell access) |
| `--rm` | Remove container on exit |
| `--name name` | Assign a name |
| `-p host:container` | Port mapping |
| `-P` | Publish all exposed ports |
| `-v host:container` | Bind mount or volume |
| `--mount type=...` | More explicit mount syntax |
| `-e KEY=VALUE` | Set environment variable |
| `--env-file file` | Load env vars from file |
| `-w /path` | Working directory |
| `-u user:group` | Run as user/group |
| `--network name` | Connect to network |
| `--hostname name` | Set container hostname |
| `--dns 8.8.8.8` | Set DNS server |
| `--restart policy` | Restart policy (no/always/unless-stopped/on-failure) |
| `--memory 512m` | Memory limit |
| `--cpus 1.5` | CPU limit |
| `--gpus all` | GPU access |
| `--privileged` | Full host access (dangerous) |
| `--cap-add SYS_ADMIN` | Add Linux capability |
| `--cap-drop ALL` | Drop all capabilities |
| `--read-only` | Read-only filesystem |
| `--pid host` | Share host PID namespace |
| `--network host` | Share host network |
| `--log-driver json-file` | Logging driver |
| `--log-opt max-size=10m` | Log options |
| `--entrypoint /bin/sh` | Override entrypoint |
| `--init` | Use tini as PID 1 |
| `--platform linux/amd64` | Target platform |

## Resource Limits

```bash
# Memory limit
docker run -d --memory=512m --memory-swap=1g myapp

# CPU limit
docker run -d --cpus=1.5 myapp

# CPU shares (relative weight)
docker run -d --cpu-shares=512 myapp

# Pin to specific CPUs
docker run -d --cpuset-cpus="0,1" myapp

# I/O limit
docker run -d --device-read-bps /dev/sda:1mb myapp
docker run -d --device-write-bps /dev/sda:1mb myapp

# PID limit
docker run -d --pids-limit 100 myapp
```

## Logging

```bash
# View container logs
docker logs myapp

# Follow + tail
docker logs -f --tail 50 myapp

# With timestamps
docker logs -t myapp

# Since a relative time
docker logs --since 30m myapp

# Between two times
docker logs --since "2024-01-01" --until "2024-01-02" myapp

# Configure logging driver at run time
docker run -d --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 myapp

# View logging driver config
docker inspect -f '{{.HostConfig.LogConfig}}' myapp
```

## Health Checks

```bash
# In Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# At runtime
docker run -d \
    --health-cmd="curl -f http://localhost:8080/health || exit 1" \
    --health-interval=30s \
    --health-timeout=3s \
    --health-retries=3 \
    --health-start-period=5s \
    myapp

# Disable health check
docker run -d --no-healthcheck myapp

# Check health status
docker inspect -f '{{.State.Health.Status}}' myapp
docker inspect -f '{{json .State.Health}}' myapp | jq

# Check healthcheck config for a container or image
docker inspect --format='{{ .Config.Healthcheck }}' myapp
docker inspect myapp | grep -A 10 '"Healthcheck"'
```

## Multi-Platform Builds (Buildx)

```bash
# Create a builder
docker buildx create --name mybuilder --use

# Build for multiple platforms
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.0 --push .

# Build and load locally (single platform)
docker buildx build --platform linux/amd64 -t myapp:1.0 --load .

# List builders
docker buildx ls

# Inspect builder
docker buildx inspect mybuilder
```

## Security

```bash
# Run as non-root user
docker run -d -u 1000:1000 myapp

# Drop all capabilities and add only needed ones
docker run -d --cap-drop ALL --cap-add NET_BIND_SERVICE myapp

# Read-only filesystem
docker run -d --read-only --tmpfs /tmp myapp

# No new privileges
docker run -d --security-opt no-new-privileges myapp

# Use seccomp profile
docker run -d --security-opt seccomp=security-profile.json myapp

# Scan image for vulnerabilities
docker scout cves myapp:1.0
docker scout quickview myapp:1.0

# View image provenance
docker scout provenance myapp:1.0

# Use specific image tags (avoid :latest in production)
# Good:  FROM node:20.11.0-alpine
# Bad:   FROM node:latest
```

## Troubleshooting

```bash
# View container exit code
docker inspect -f '{{.State.ExitCode}}' myapp

# View last container state
docker inspect -f '{{.State.Status}}' myapp

# View OOM kill status
docker inspect -f '{{.State.OOMKilled}}' myapp

# Check why a container exited
docker logs myapp 2>&1 | tail -20

# View events (real-time Docker daemon events)
docker events
docker events --filter "container=myapp"
docker events --filter "event=die"

# Inspect container filesystem
docker exec myapp ls -la /app

# Export filesystem for analysis
docker export myapp | tar -tf - | head -50

# Debug a failed build (use intermediate layer)
docker run -it <intermediate-image-id> sh

# Check container resource usage
docker stats myapp --no-stream

# View container mounts
docker inspect -f '{{json .Mounts}}' myapp | jq
```

## Useful One-Liners

```bash
# Stop all running containers
docker stop $(docker ps -q)

# Remove all stopped containers
docker rm $(docker ps -aq)

# Remove all images
docker rmi $(docker images -q)

# Remove dangling images
docker rmi $(docker images -f "dangling=true" -q)

# Kill all running containers
docker kill $(docker ps -q)

# Remove all unused resources
docker system prune -af --volumes

# Get IP address of a container
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' myapp

# Get container ID by name
docker inspect -f '{{.Id}}' myapp

# List containers by size
docker ps -s

# Follow logs from multiple containers
docker compose logs -f web app

# Copy all files from container to host
docker cp myapp:/app/. ./backup/

# Run a temporary database for testing
docker run --rm -d -p 5432:5432 -e POSTGRES_PASSWORD=test postgres:16

# Quick one-off command (no leftover container)
docker run --rm alpine cat /etc/os-release

# Monitor container events
docker events --format '{{.Time}} {{.Actor.Attributes.name}} {{.Action}}'

# Commit container changes to an image
docker commit myapp myapp:snapshot

# List files inside a container (without exec)
docker export myapp | tar -t

# Import a tar as an image
cat myapp.tar | docker import - myapp:imported

# Find Docker root directory
docker info --format '{{ .DockerRootDir }}'

# Find logging driver type
docker info | grep Logging

# View Docker daemon logs (systemd)
journalctl -fu docker.service

# Add user to docker group (avoids sudo)
usermod -aG docker $USER

# Find container by volume ID
docker ps --filter volume=<volume-name-or-id>

# Kill container's PID 1 from host (force stop without docker stop)
kill -9 $(docker inspect --format '{{.State.Pid}}' myapp)

# Update images and recreate containers
docker compose pull && docker compose up -d

# List all environment variables of a container
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' myapp

# Check a specific env var in a running container
docker compose exec myapp /bin/sh -lc 'printf "%s\n" "$MY_VAR"'

# Check healthcheck configuration for a container or image
docker inspect --format='{{ .Config.Healthcheck }}' myapp

# List volumes for a specific container
docker inspect myapp | jq '.[].Mounts'
docker inspect -f '{{range .Mounts}}{{.Type}}:{{.Source}}:{{.Destination}}{{println}}{{end}}' myapp
docker inspect myapp | grep -A3 -e "Volumes\":"

# List dangling volumes
docker volume ls -qf dangling=true

# Remove dangling volumes
docker volume rm $(docker volume ls -qf dangling=true)

# List all container volumes (detailed)
docker ps -a --format '{{ .ID }}' | xargs -I {} docker inspect -f '{{ .Name }}{{ printf "\n" }}{{ range .Mounts }}{{ printf "\n\t" }}{{ .Type }} {{ if eq .Type "bind" }}{{ .Source }}{{ end }}{{ .Name }} => {{ .Destination }}{{ end }}{{ printf "\n" }}' {}

# Find containers listening on specific ports
docker ps | grep -E ":80|:443"

# Find reverse proxy containers
docker ps | grep -E "(traefik|nginx|caddy|envoy|haproxy)"

# Check all containers for specific labels (e.g., Traefik)
for container in $(docker ps -q); do
    name=$(docker inspect --format '{{.Name}}' $container | sed 's/\///')
    echo "=== $name ==="
    docker inspect $container | grep -i traefik | head
done

# Find images by layer hash
for I in $(docker image ls --format '{{.ID}}'); do
    F=$(docker image inspect $I | grep "<hash>")
    if [ -n "$F" ]; then echo $I; fi
done

# Find volumes by ID/hash
for I in $(docker volume ls | grep -v DRIVER | awk '{print $2}' | sort | uniq); do
    F=$(docker volume inspect $I | grep "${ID}")
    if [ -n "$F" ]; then echo $I; fi
done

# Find containers by ID/hash
for I in $(docker container ls --no-trunc | grep -v CONTAINER | awk '{print $1}' | sort | uniq); do
    F=$(docker container inspect $I | grep "${ID}")
    if [ -n "$F" ]; then echo $I; fi
done

# Export container filesystem to file
docker export myapp > myapp-fs.tar

# Check container image name
docker inspect --format '{{.Config.Image}}' myapp

# Export container/image lists to files
docker ps -a --format 'table {{.Names}}\t{{.ID}}\t{{.Image}}\t{{.Status}}' > containers.txt
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.ID}}\t{{.Size}}' > images.txt
docker volume ls --format 'table {{.Name}}\t{{.Driver}}' > volumes.txt
```

## Restart Policies

```bash
--restart no              # Never restart (default)
--restart on-failure      # Restart on non-zero exit code
--restart on-failure:3    # Restart max 3 times on failure
--restart always          # Always restart (including on daemon start)
--restart unless-stopped  # Restart unless manually stopped
```

## Development Patterns

```bash
# Hot reload with volume mounting (source code changes reflect immediately)
docker run -v $(pwd):/app -p 3000:3000 myapp:dev

# Override entrypoint for debugging
docker run -it --entrypoint /bin/sh myapp

# Run container with shell access (ignore CMD)
docker run -it --entrypoint /bin/bash myapp

# Quick development database
docker run --rm -d -p 5432:5432 -e POSTGRES_PASSWORD=dev postgres:16
docker run --rm -d -p 6379:6379 redis:alpine
docker run --rm -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=dev mysql:8

# Run with host networking (Linux only, bypasses port mapping)
docker run -d --network host myapp
```

## Production Patterns

```bash
# Run with restart policy and health check
docker run -d \
    --restart unless-stopped \
    --health-cmd="curl -f http://localhost:8080/health || exit 1" \
    --health-interval=30s \
    -p 80:8080 \
    myapp:prod

# Resource-limited production container
docker run -d \
    --restart unless-stopped \
    --memory=512m \
    --cpus=1.0 \
    --read-only \
    --tmpfs /tmp \
    --cap-drop ALL \
    --security-opt no-new-privileges \
    -p 8080:8080 \
    myapp:prod
```

## Aliases

Add to `~/.bashrc` or `~/.zshrc`:

```bash
alias dps='docker ps'
alias dpsa='docker ps -a'
alias dimg='docker images'
alias dlog='docker logs -f'
alias dstart='docker start $(docker ps -aq)'
alias dstop='docker stop $(docker ps -aq)'
alias drm='docker rm $(docker ps -aq)'
alias dri='docker rmi $(docker images -q)'
alias dprune='docker system prune -af --volumes'
```

### dbash Function

```bash
dbash() {
    docker exec -it $(docker ps -aqf "name=$1") bash
}
# Usage: dbash mycontainer
```

## Important Files and Directories

| Path | Description |
|------|-------------|
| `/var/lib/docker/` | Docker root directory (images, containers, volumes) |
| `/var/lib/docker/containers/[ID]/[ID]-json.log` | Container log file (json-file driver) |
| `/var/lib/docker/volumes/` | Named volumes |
| `/var/lib/docker/overlay2/` | Image layers (overlay2 storage driver) |
| `/etc/docker/daemon.json` | Docker daemon configuration |
| `/lib/systemd/system/docker.service` | Docker systemd service file |
| `~/.docker/config.json` | User Docker config (auth, plugins) |
| `/etc/sysconfig/docker` | Docker daemon startup options (RHEL/CentOS) |
| `/etc/sysconfig/docker-storage` | Storage config (RHEL/CentOS, auto-managed) |
| `/etc/sysconfig/docker-storage-setup` | Storage config override (RHEL/CentOS) |
| `/usr/share/container-storage-setup/container-storage-setup` | Default storage setup config (RHEL/CentOS) |

```bash
# Find Docker root directory
docker info --format '{{ .DockerRootDir }}'

# View daemon configuration
cat /etc/docker/daemon.json

# Example daemon.json
# {
#   "log-driver": "json-file",
#   "log-opts": { "max-size": "10m", "max-file": "3" },
#   "storage-driver": "overlay2",
#   "default-address-pools": [{ "base": "172.17.0.0/12", "size": 24 }]
# }
```

## Tools

- [dry](https://github.com/moncho/dry) — terminal UI for Docker containers and images
- [dockcheck](https://github.com/mag37/dockcheck) — check for Docker image updates
- [watchtower](https://github.com/containrrr/watchtower) — automatically update running containers when images change
- [lazydocker](https://github.com/jesseduffield/lazydocker) — terminal UI for Docker and Docker Compose
- [ctop](https://github.com/bcicen/ctop) — top-like interface for container metrics
- [dive](https://github.com/wagoodman/dive) — explore Docker image layers and optimize size

## Advanced Networking

### Network Drivers

```bash
# Create overlay network (Docker Swarm)
docker network create --driver overlay my_overlay

# Create macvlan network (direct physical interface access)
docker network create -d macvlan \
    --subnet=192.168.1.0/24 \
    --gateway=192.168.1.1 \
    -o parent=eth0 my_macvlan

# Create network with custom MTU
docker network create --opt com.docker.network.driver.mtu=1500 my_network
```

### DNS and Service Discovery

```bash
# Custom DNS for a container
docker run -d --dns 8.8.8.8 --dns-search example.com nginx

# Check DNS resolution inside a container
docker exec myapp nslookup db

# Set hostname
docker run -d -h myhost --name web nginx
```

### Port Mapping Advanced

```bash
# Random host port mapping (Docker assigns a port)
docker run -d -P nginx

# Map to specific interface
docker run -d -p 192.168.1.10:8080:80 nginx

# Map to all interfaces explicitly
docker run -d -p 0.0.0.0:8080:80 nginx

# Expose multiple ports
docker run -d -p 80:80 -p 443:443 -p 8000:8000 myapp

# Link containers (legacy — use networks instead)
docker run -d --link web:web nginx
```

### Network Debugging

```bash
# Check network connectivity
docker exec myapp ping -c 3 8.8.8.8

# Check open ports
docker exec myapp netstat -tlnp

# Get network configuration
docker exec myapp ip addr

# DNS resolution test
docker exec myapp nslookup google.com

# Traceroute
docker exec myapp traceroute google.com

# Check iptables rules
docker exec myapp iptables -L
```

## Volume Backup and Restore

```bash
# Backup a named volume to a tar.gz
docker run --rm \
    -v my_volume:/data \
    -v $(pwd):/backup \
    alpine tar czf /backup/volume_backup.tar.gz /data

# Restore a volume from backup
docker run --rm \
    -v my_volume:/data \
    -v $(pwd):/backup \
    alpine tar xzf /backup/volume_backup.tar.gz -C /

# Remove volume when removing container
docker rm -v myapp

# Anonymous volume (Docker-managed, no name)
docker run -d -v /data nginx
```

## Advanced Security

### Content Trust (Image Signing)

```bash
# Enable content trust (forces signed images)
export DOCKER_CONTENT_TRUST=1

# Sign image when pushing
docker push myrepo/myimage:tag

# Verify image signature
docker trust inspect myrepo/myimage:tag

# Disable content trust temporarily
DOCKER_CONTENT_TRUST=0 docker pull untrusted/image:tag
```

### Capabilities Inspection

```bash
# List capabilities inside a container
docker exec myapp capsh --print

# Run with all capabilities dropped except specific ones
docker run -d \
    --cap-drop ALL \
    --cap-add NET_ADMIN \
    --cap-add SYS_TIME \
    myapp
```

### AppArmor and Seccomp

```bash
# Use default AppArmor profile
docker run -d --security-opt apparmor=docker-default nginx

# Disable AppArmor
docker run -d --security-opt apparmor=unconfined nginx

# Disable seccomp (for debugging only)
docker run -d --security-opt seccomp=unconfined nginx

# Use custom seccomp profile
docker run -d --security-opt seccomp=custom-profile.json nginx
```

### Secrets Management (Docker Swarm)

```bash
# Create a secret
echo "my_secret_value" | docker secret create my_secret -

# Create from file
docker secret create my_cert ./server.crt

# List secrets
docker secret ls

# Use secret in a service
docker service create --secret my_secret nginx

# Remove secret
docker secret rm my_secret

# In Compose: secrets mounted at /run/secrets/<name>
```

### Registry Credentials

```bash
# View stored credentials
cat ~/.docker/config.json

# Login with token (CI/CD)
echo "$TOKEN" | docker login registry.example.com -u user --password-stdin
```

## Image Optimization

### Build with Multiple Tags

```bash
docker build -t myapp:latest -t myapp:v1.0 -t myapp:$(git rev-parse --short HEAD) .
```

### Layer Analysis

```bash
# View image layers
docker inspect --format='{{json .RootFS.Layers}}' myimage:tag | jq

# Compressed image save
docker save myimage:tag | gzip > myimage.tar.gz

# Load compressed image
docker load < myimage.tar.gz

# Prune images older than specific time
docker image prune --filter "until=72h"
```

### .dockerignore

```
# .dockerignore — exclude from build context
.git
.gitignore
node_modules
npm-debug.log
.env
.DS_Store
*.md
docker-compose*.yml
Dockerfile*
```

## Advanced Container Management

### Namespace Options

```bash
# Share host PID namespace (see host processes)
docker run -d --pid=host nginx

# Share host IPC namespace (shared memory)
docker run -d --ipc=host nginx

# Share host UTS namespace (hostname/domainname)
docker run -d --uts=host nginx
```

### Attach and Detach

```bash
# Attach to a running container's stdout/stderr
docker attach myapp

# Detach without stopping (from attach): Ctrl+P, Ctrl+Q

# Attach with signal forwarding disabled
docker attach --no-stdin myapp
```

### Block I/O Limits

```bash
# Block I/O weight (relative, 10-1000)
docker run -d --blkio-weight=200 myapp

# Per-device weight
docker run -d --blkio-weight-device=/dev/sda:100 myapp

# Read IOPS limit
docker run -d --device-read-iops /dev/sda:1000 myapp

# Write IOPS limit
docker run -d --device-write-iops /dev/sda:1000 myapp
```

### Memory Advanced

```bash
# Memory soft limit (triggers reclaim under pressure)
docker run -d --memory-reservation=256m myapp

# Kernel memory limit
docker run -d --kernel-memory=50m myapp

# Disable OOM killer
docker run -d --oom-kill-disable --memory=512m myapp
```

## Advanced Debugging

### strace Inside Container

```bash
# Debug with strace (requires SYS_PTRACE)
docker run -it --cap-add SYS_PTRACE --security-opt seccomp=unconfined \
    myapp strace -c ls

# Trace syscalls of a running process
docker exec --cap-add SYS_PTRACE myapp strace -p 1
```

### Container Stats Formatting

```bash
# Custom stats format
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Memory only
docker stats --format "{{.Container}}\t{{.MemUsage}}"

# JSON output
docker stats --no-stream --format "{{json .}}"
```

### Containers by Exit Code

```bash
# List containers that exited with error
docker ps -a --filter "exited=1"

# List containers that exited successfully
docker ps -a --filter "exited=0"
```

### Logs Advanced

```bash
# Logs between two times
docker logs --since "2024-01-01T00:00:00" --until "2024-12-31T23:59:59" myapp

# Logs with details (extra attributes from logging driver)
docker logs --details myapp
```

## Performance Tuning

### Storage Driver

```bash
# View current storage driver
docker info | grep "Storage Driver"

# Available drivers: overlay2, aufs, btrfs, devicemapper, vfs, zfs

# Check data/metadata space
docker info | grep -E "Data Space|Metadata Space"
```

### Network Performance

```bash
# Host network (no NAT overhead, Linux only)
docker run -d --network host nginx

# Disable inter-container communication on a network
docker network create --opt com.docker.network.bridge.enable_icc=false isolated_net
```

### Startup Optimization

```bash
# Don't allocate TTY if not needed (saves resources)
docker run -d nginx    # not: docker run -dit nginx

# Use exec form in Dockerfile (no shell overhead)
# Good:  ENTRYPOINT ["nginx", "-g", "daemon off;"]
# Bad:   ENTRYPOINT nginx -g "daemon off;"

# Use --init for proper signal handling (no zombie processes)
docker run -d --init myapp
```

## Production Checklist

- Use specific base image tags (not `latest`)
- Run as non-root user
- Drop unnecessary capabilities (`--cap-drop ALL`)
- Set memory limits (`--memory`)
- Set CPU limits (`--cpus`)
- Configure health checks
- Use restart policies (`--restart unless-stopped`)
- Mount volumes for persistent data
- Configure log rotation (`--log-opt max-size=10m --log-opt max-file=3`)
- Use secrets for sensitive data
- Enable read-only filesystem where possible (`--read-only`)
- Use `--init` for proper PID 1 behavior
- Scan images for vulnerabilities before deploying
- Regularly prune unused resources

## Testing Pattern

```bash
# Run tests in a disposable container
docker run --rm \
    -v $(pwd):/app \
    -w /app \
    node:20 \
    npm test

# Run with specific env
docker run --rm \
    -v $(pwd):/app \
    -w /app \
    -e CI=true \
    -e NODE_ENV=test \
    node:20 \
    npm run test:coverage
```
