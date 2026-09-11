# Docker Buildx and Multi-Platform Builds

`docker buildx` extends the standard `docker build` with the BuildKit engine, adding multi-platform builds, better caching, and advanced features such as build secrets and SSH forwarding. This guide compares the two, covers installation on macOS and Linux, and shows practical multi-platform build examples.

For Dockerfile fundamentals, including `WORKDIR`, `COPY`, and multi-stage builds, see [Building Docker Images with Dockerfile](articles/docker-build-image-guide.md).

## docker build vs docker buildx

| Aspect | `docker build` | `docker buildx` |
|--------|----------------|-----------------|
| Backend | Classic builder or BuildKit | BuildKit, always |
| Platforms per build | One (usually the host's) | One or many simultaneously |
| Cross-platform output | No | Yes, via emulation or remote nodes |
| Registry cache export | Limited | Yes (`--cache-to`/`--cache-from`) |
| Build secrets, SSH forwarding | Limited | Yes |
| Typical use | Simple local single-arch builds | Multi-arch images and CI pipelines |

On current Docker Engine and Docker Desktop, `docker build` already routes through BuildKit and is effectively a front end to buildx. Use plain `docker build` for quick single-platform work; reach for `docker buildx build` when you need multiple architectures, registry-backed cache, or the advanced BuildKit features.

> A single-platform `docker buildx build` loads the result into your local image store much like `docker build`. A **multi-platform** build cannot be loaded into the local store and must be pushed to a registry (or exported), because the local store holds one platform per tag.

## Installing Buildx

### macOS

Docker Desktop ships buildx and enables it by default. Confirm it is present:

```bash
docker buildx version
```

If that succeeds, you are ready. Without Docker Desktop, install the plugin with Homebrew:

```bash
brew install docker-buildx
```

Then create and select a builder:

```bash
docker buildx create --name mybuilder --use
docker buildx ls
```

To install manually, download the macOS binary from the [buildx releases page](https://github.com/docker/buildx/releases/latest) and place it at `~/.docker/cli-plugins/docker-buildx` with the executable bit set.

### Linux

Package manager (Debian and Ubuntu, with Docker's official repository configured):

```bash
sudo apt-get update
sudo apt-get install -y docker-buildx-plugin
```

Manual install of a specific release — check the [releases page](https://github.com/docker/buildx/releases/latest) for the current version and substitute it:

```bash
mkdir -p ~/.docker/cli-plugins
# Replace vX.Y.Z with the current release
curl -fsSL -o ~/.docker/cli-plugins/docker-buildx \
  https://github.com/docker/buildx/releases/download/vX.Y.Z/buildx-vX.Y.Z.linux-amd64
chmod +x ~/.docker/cli-plugins/docker-buildx
```

Verify and create a builder that supports multi-platform emulation:

```bash
docker buildx version
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap
```

> Prefer the package manager or a pinned binary from the official releases over piping an install script into a shell. If you do use an upstream script, review it first. See [Downloading and Running Remote Installation Scripts Safely](articles/download-run-install-scripts.md).

### Enabling emulation for other architectures

To build for an architecture different from your host (for example `arm64` on an `amd64` machine), register QEMU emulators. Docker Desktop includes this; on plain Linux, install it once:

```bash
docker run --privileged --rm tonistiigi/binfmt --install all
```

Emulated builds are correct but slower than native. For heavy multi-arch pipelines, consider native builder nodes per architecture instead of emulation.

## Multi-Platform Build Examples

The examples assume a builder created with `docker buildx create --use`. The default `docker` driver cannot produce multi-platform output; the `docker-container` driver created by `buildx create` can.

### Build for multiple architectures and push

Multi-platform builds must be pushed to a registry (or exported), not loaded locally:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myregistry/myimage:1.0.0 \
  --push .
```

This produces a single tag backed by a manifest list, so clients on either architecture pull the right image automatically.

### Build a single platform and load locally

```bash
docker buildx build \
  --platform linux/amd64 \
  -t myimage:1.0.0 \
  --load .
```

`--load` works only for a single platform. Combining `--load` with several `--platform` values fails.

### Build without pushing or loading

Omit both `--push` and `--load` to build and discard the output — useful for verifying a build succeeds in CI:

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t myimage:1.0.0 .
```

Add `--output type=oci,dest=image.tar` to export an OCI archive instead.

### Pass build arguments

```dockerfile
FROM ubuntu:24.04
ARG NODE_MAJOR=20
RUN apt-get update && apt-get install -y ca-certificates curl gnupg
```

```bash
docker buildx build --build-arg NODE_MAJOR=20 -t myimage:1.0.0 --load .
```

### Use registry-backed cache

Export and reuse the build cache through a registry so CI runs and teammates share layers:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-to type=registry,ref=myregistry/myimage:buildcache,mode=max \
  --cache-from type=registry,ref=myregistry/myimage:buildcache \
  -t myregistry/myimage:1.0.0 \
  --push .
```

`mode=max` exports cache for all stages, including intermediate ones, at the cost of a larger cache image. The default `mode=min` caches only the final stage.

### Platform-aware Dockerfile

BuildKit sets build/target platform arguments automatically. Use them to build efficiently, especially with cross-compilers:

```dockerfile
# Build on the native platform, then target another
FROM --platform=$BUILDPLATFORM golang:1.23 AS builder
ARG TARGETOS
ARG TARGETARCH
WORKDIR /src
COPY . .
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /out/app

FROM alpine:3.20
COPY --from=builder /out/app /usr/local/bin/app
ENTRYPOINT ["app"]
```

Building the compile stage on `$BUILDPLATFORM` (the host) and cross-compiling to `$TARGETARCH` avoids emulating the whole toolchain and is much faster than emulating the build.

## Managing Builders

```bash
# List builders and their nodes
docker buildx ls

# Create and switch to a new builder
docker buildx create --name multiarch --use

# Show a builder's platforms and status; start it if stopped
docker buildx inspect --bootstrap

# Switch the active builder
docker buildx use multiarch

# Remove a builder
docker buildx rm multiarch
```

Inspect the resulting manifest list after a multi-arch push:

```bash
docker buildx imagetools inspect myregistry/myimage:1.0.0
```

## Advanced BuildKit Features

Buildx exposes BuildKit capabilities that keep secrets out of image layers:

```bash
# Mount a secret at build time (not stored in the image)
docker buildx build --secret id=npmrc,src=$HOME/.npmrc -t myimage:1.0.0 --load .

# Forward the host SSH agent for private dependency fetches
docker buildx build --ssh default -t myimage:1.0.0 --load .
```

In the Dockerfile, consume them without persisting to a layer:

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci --omit=dev
COPY . .
CMD ["node", "index.js"]
```

Do not pass secrets through `--build-arg`; build arguments are visible in image history. Use `--secret` instead.

## Common Pitfalls

- Trying to `--load` a multi-platform build. Push to a registry, or build one platform at a time for local loading.
- Using the default `docker` driver for multi-platform builds. Create a `docker-container` builder with `docker buildx create --use`.
- Forgetting emulation on plain Linux. Register QEMU with `binfmt` before building non-native architectures.
- Passing secrets via `--build-arg`. They leak into image history; use `--secret`.
- Pinning a stale buildx binary in docs or scripts. Reference the releases "latest" page and update deliberately.

## Quick Reference

```bash
# Version and builders
docker buildx version
docker buildx ls
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap

# Enable emulation on plain Linux (once)
docker run --privileged --rm tonistiigi/binfmt --install all

# Multi-arch build + push (must push, cannot --load)
docker buildx build --platform linux/amd64,linux/arm64 -t myregistry/myimage:1.0.0 --push .

# Single-arch build loaded locally
docker buildx build --platform linux/amd64 -t myimage:1.0.0 --load .

# Registry cache
docker buildx build --cache-to type=registry,ref=myregistry/myimage:buildcache,mode=max \
  --cache-from type=registry,ref=myregistry/myimage:buildcache \
  -t myregistry/myimage:1.0.0 --push .

# Inspect the published manifest list
docker buildx imagetools inspect myregistry/myimage:1.0.0
```

Pin image versions rather than using `latest` (see [Pin Docker Image Versions Instead of latest](articles/docker-image-version-pinning.md)), and for the wider command set see the [Docker Cheatsheet](articles/docker-cheatsheet.md) and [Building Docker Images with Dockerfile](articles/docker-build-image-guide.md).
