# Building Docker Images with Dockerfile

## Dockerfile Instructions Reference

| Instruction | Description |
|-------------|-------------|
| `FROM` | Base image from a container registry (Docker Hub, ECR, GCR, Quay) |
| `RUN` | Execute commands during image build |
| `ENV` | Set environment variables (available at build time and runtime) |
| `ARG` | Set build-time only variables (not available in running container) |
| `COPY` | Copy local files and directories into the image |
| `ADD` | Like COPY but supports URLs and tar auto-extraction (prefer COPY) |
| `WORKDIR` | Set working directory for RUN, CMD, COPY, ENTRYPOINT |
| `EXPOSE` | Declare the port the container listens on |
| `VOLUME` | Create or mount a volume |
| `USER` | Set the user/UID for running the container |
| `LABEL` | Add metadata to the image |
| `SHELL` | Set default shell for RUN, CMD, ENTRYPOINT |
| `CMD` | Default command when container starts (overridable via CLI) |
| `ENTRYPOINT` | Main executable when container starts (defaults to `/bin/sh -c`) |

## Dockerfile Format

```dockerfile
# Everything on the left is an INSTRUCTION
# Everything on the right is an ARGUMENT
FROM ubuntu:22.04
LABEL maintainer="user@example.com"
RUN apt-get -y update && apt-get -y install nginx
COPY files/default /etc/nginx/sites-available/default
COPY files/index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```

The file must be named `Dockerfile` (no extension).

## Building an Image

```bash
# Build from current directory (Dockerfile in .)
docker build -t myapp:1.0 .

# Build from a different path
docker build -t myapp:1.0 /path/to/folder

# Build with a specific Dockerfile
docker build -t myapp:1.0 -f /path/to/Dockerfile .

# Build with build arguments
docker build --build-arg VERSION=3.9 -t myapp:1.0 .

# Build without cache
docker build --no-cache -t myapp:1.0 .
```

The `.` (dot) refers to the build context — the directory containing files available during build.

## Tagging Images

```bash
# Tag during build
docker build -t nginx:1.0 .

# Add another tag to existing image
docker tag nginx:1.0 myregistry/nginx:1.0

# Multiple tags for the same image
docker build -t nginx:latest -t nginx:1.0 .
```

Tagging strategies:
- **Stable tags** — same tag, content changes over time (e.g., `latest`, `stable`)
- **Unique tags** — different tag per build (e.g., commit SHA, build number, timestamp)

## Running the Built Image

```bash
# Run detached with port mapping
docker run -d -p 8080:80 --name webserver nginx:1.0

# Run interactive
docker run -it --name mycontainer myapp:1.0 /bin/bash

# Run with environment variables
docker run -d -e APP_ENV=production --name myapp myapp:1.0
```

## Pushing to a Registry

```bash
# Login to Docker Hub
docker login

# Tag for your registry
docker tag nginx:1.0 username/nginx:1.0

# Push
docker push username/nginx:1.0

# Login to a different registry
docker login ghcr.io
docker tag nginx:1.0 ghcr.io/username/nginx:1.0
docker push ghcr.io/username/nginx:1.0
```

## Heredoc Syntax in Dockerfile

Consolidate multiple commands or inline file content directly in the Dockerfile.

```dockerfile
# Multiple RUN commands in one layer
RUN <<EOF
apt-get update
apt-get upgrade -y
apt-get install -y nginx
EOF

# Inline file creation
COPY <<EOF /usr/share/nginx/html/index.html
<html>
  <body><h1>Hello from Docker</h1></body>
</html>
EOF

# Inline script execution
RUN python3 <<EOF
with open("/hello", "w") as f:
    print("Hello", file=f)
    print("World", file=f)
EOF
```

## Multi-Stage Builds

Reduce final image size by separating build and runtime stages.

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Runtime stage
FROM alpine:3.19
COPY --from=builder /app/myapp /usr/local/bin/myapp
ENTRYPOINT ["myapp"]
```

## .dockerignore

Exclude files from the build context to speed up builds and avoid leaking secrets.

```text
.git
.env
node_modules
*.log
__pycache__
.DS_Store
```

## CMD vs ENTRYPOINT

| Feature | CMD | ENTRYPOINT |
|---------|-----|------------|
| Purpose | Default arguments | Main executable |
| Override | `docker run <image> <cmd>` | `docker run --entrypoint <cmd>` |
| Multiple | Only last one applies | Only last one applies |
| Shell form | `CMD command arg1` | `ENTRYPOINT command arg1` |
| Exec form | `CMD ["cmd", "arg1"]` | `ENTRYPOINT ["cmd", "arg1"]` |

Combined usage:

```dockerfile
ENTRYPOINT ["python3"]
CMD ["app.py"]
# Runs: python3 app.py
# Override: docker run myapp script.py → python3 script.py
```

## Best Practices

- Use official or verified base images from trusted registries
- Use minimal base images (`alpine`, `distroless`) for production
- Use specific tags (not `latest`) for reproducible builds
- Combine RUN commands to reduce layers: `RUN apt-get update && apt-get install -y pkg`
- Run as non-root user with `USER` instruction
- Use `.dockerignore` to exclude unnecessary files
- Never put secrets or credentials in the Dockerfile
- Use `COPY` over `ADD` unless you need URL download or tar extraction
- Place frequently changing instructions (COPY app code) at the bottom to leverage cache
- Use multi-stage builds to separate build dependencies from runtime
- One process per container
- Use a linter like [hadolint](https://github.com/hadolint/hadolint) to catch issues

## Base Image Registries

| Registry | URL |
|----------|-----|
| Docker Hub | [hub.docker.com](https://hub.docker.com) |
| Google Distroless | [gcr.io/distroless](https://github.com/GoogleContainerTools/distroless) |
| AWS ECR Public | [gallery.ecr.aws](https://gallery.ecr.aws) |
| Red Hat Quay | [quay.io](https://quay.io) |

## Using Non-Docker Hub Base Images

```dockerfile
# Google distroless
FROM gcr.io/distroless/static-debian12

# AWS ECR public
FROM public.ecr.aws/nginx/nginx:latest

# Red Hat UBI
FROM registry.access.redhat.com/ubi9/ubi-minimal
```

## BuildKit Features

BuildKit is the modern build backend (default since Docker 23.0). Enable it on older versions with `DOCKER_BUILDKIT=1`.

```bash
# Enable BuildKit (older Docker versions)
DOCKER_BUILDKIT=1 docker build -t myapp:1.0 .

# Or set permanently in /etc/docker/daemon.json
# { "features": { "buildkit": true } }
```

### Cache Mounts

Persist package manager caches across builds to speed up repeated installs.

```dockerfile
# apt cache mount
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y python3

# pip cache mount
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Go module cache
RUN --mount=type=cache,target=/go/pkg/mod \
    go build -o /app .
```

### Secret Mounts

Pass secrets during build without baking them into image layers.

```dockerfile
# Mount a secret file (not stored in any layer)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install

# Mount SSH agent for private repo access
RUN --mount=type=ssh \
    git clone git@github.com:org/private-repo.git
```

```bash
# Pass secrets at build time
docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp:1.0 .

# Pass SSH socket
docker build --ssh default -t myapp:1.0 .
```

### Bind Mounts

Mount files from the build context or other build stages without copying.

```dockerfile
# Mount requirements.txt without COPY
RUN --mount=type=bind,source=requirements.txt,target=/tmp/requirements.txt \
    pip install -r /tmp/requirements.txt
```

## Multi-Platform Builds (buildx)

Build images for multiple architectures from a single machine.

```bash
# Create a multi-platform builder
docker buildx create --name multiarch --use

# Build for multiple platforms
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.0 --push .

# Build for a specific platform (useful for M1/M2 Mac targeting amd64)
docker buildx build --platform linux/amd64 -t myapp:1.0 .

# Inspect builder capabilities
docker buildx inspect --bootstrap

# List available builders
docker buildx ls
```

In the Dockerfile, use `TARGETARCH` and `TARGETOS` for platform-aware logic:

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.21 AS builder
ARG TARGETARCH
ARG TARGETOS
WORKDIR /app
COPY . .
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /app/myapp

FROM alpine:3.19
COPY --from=builder /app/myapp /usr/local/bin/myapp
ENTRYPOINT ["myapp"]
```

## HEALTHCHECK Instruction

Define a command Docker runs to verify the container is healthy.

```dockerfile
# HTTP health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# TCP check (without curl)
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/ || exit 1

# Custom script
HEALTHCHECK --interval=60s --timeout=10s \
    CMD /usr/local/bin/healthcheck.sh
```

| Option | Default | Description |
|--------|---------|-------------|
| `--interval` | 30s | Time between checks |
| `--timeout` | 30s | Max time for a check to complete |
| `--start-period` | 0s | Grace period before checks count as failures |
| `--retries` | 3 | Consecutive failures before marking unhealthy |

```bash
# Check container health status
docker inspect --format='{{.State.Health.Status}}' mycontainer

# View health check logs
docker inspect --format='{{json .State.Health}}' mycontainer | jq
```

## Inspecting Images

```bash
# Show image layers and sizes
docker history myapp:1.0

# Detailed image metadata (env, cmd, labels, layers)
docker inspect myapp:1.0

# Show only specific fields
docker inspect --format='{{.Config.Env}}' myapp:1.0
docker inspect --format='{{.Config.ExposedPorts}}' myapp:1.0

# List images with sizes
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Show all layers (including intermediate)
docker history --no-trunc myapp:1.0
```

### dive — Interactive Layer Explorer

[dive](https://github.com/wagoodman/dive) shows layer-by-layer file changes and wasted space.

```bash
# Install
brew install dive                    # macOS
apt install dive                     # Debian/Ubuntu

# Analyze an image
dive myapp:1.0

# CI mode — fail if image efficiency is below threshold
dive myapp:1.0 --ci --lowestEfficiency=0.9
```

## Security Scanning

Scan images for known vulnerabilities (CVEs) before deploying.

### Docker Scout (built-in)

```bash
# Quick scan
docker scout cves myapp:1.0

# Detailed recommendations
docker scout recommendations myapp:1.0

# Compare two image versions
docker scout compare myapp:1.0 myapp:2.0

# SBOM (Software Bill of Materials)
docker scout sbom myapp:1.0
```

### Trivy

```bash
# Install
brew install trivy                   # macOS
apt install trivy                    # Debian/Ubuntu

# Scan an image
trivy image myapp:1.0

# Only show HIGH and CRITICAL
trivy image --severity HIGH,CRITICAL myapp:1.0

# Scan and fail CI if vulnerabilities found
trivy image --exit-code 1 --severity CRITICAL myapp:1.0

# Scan a Dockerfile for misconfigurations
trivy config Dockerfile
```

### Grype

```bash
# Install
brew install grype                   # macOS
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s

# Scan an image
grype myapp:1.0

# Only critical vulnerabilities
grype myapp:1.0 --only-fixed --fail-on critical

# Output as JSON for CI pipelines
grype myapp:1.0 -o json > scan-results.json
```

### Comparison

| Tool | Maintained By | Strengths |
|------|---------------|-----------|
| Docker Scout | Docker | Built-in, recommendations, SBOM |
| Trivy | Aqua Security | Fast, Dockerfile scanning, broad coverage |
| Grype | Anchore | SBOM-based, pairs with Syft, CI-friendly |

## Common Build Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| `port is already allocated` | Another service using the port | Use a different `-p` mapping or stop the conflicting service |
| `Failed to download package` | No internet in container or DNS issue | Check network, use `--network=host` during build if needed |
| Build fails with syntax error | Invalid Dockerfile instruction | Lint with hadolint, check instruction spelling |
| Image too large | Too many layers, unnecessary packages | Use multi-stage builds, minimal base, and `.dockerignore` |
| Cache not working | Changing files early in Dockerfile | Reorder instructions — put stable ones first |

## Image vs Container

| | Image | Container |
|-|-------|-----------|
| State | Read-only layers | Writable layer on top of image |
| Lifecycle | Persists until deleted | Created/started/stopped/removed |
| Analogy | VM golden image / class | Running VM / instance |
| Relationship | Can exist without containers | Requires an image to run |
