# Rebuilding Docker Compose Images and Containers

Compose reuses an existing image unless you explicitly tell it to rebuild, so a code or Dockerfile change may not show up until you force a rebuild. This guide walks through the options from lightest to most thorough, explains when each is needed, and clarifies what `--no-cache` and `--force-recreate` actually do.

This is about **rebuilding locally built images** (services with a `build:` section). To update services that pull prebuilt images from a registry, see [Updating Docker Compose Containers](articles/docker-compose-updating-containers.md). For the build block itself, see [Docker Compose Build Configuration and Build Context](articles/docker-compose-build-configuration.md).

## First, Understand What Compose Reuses

- `docker compose up` builds an image only if one does not already exist for the service; otherwise it reuses the existing image and only recreates containers whose image or config changed.
- `docker compose build` rebuilds the image but uses the layer cache, so unchanged Dockerfile steps are reused.
- `--no-cache` tells the builder to ignore the layer cache and run every Dockerfile step fresh.
- `--force-recreate` recreates containers even when nothing changed, but does **not** by itself rebuild the image.

Most "my change didn't take effect" problems are one of two things: you didn't rebuild the image, or a cached layer (for example a cached `git clone` or dependency install) served stale content.

## Option 1: Rebuild a Specific Service

The lightest option — rebuild one service without touching the rest.

```bash
# Rebuild using the cache (fast)
docker compose build web

# Rebuild ignoring the layer cache (fresh)
docker compose build --no-cache web

# Rebuild, then recreate just that container
docker compose up -d --build web
```

Use `--no-cache` when a step that Docker thinks is unchanged actually produced stale output — a common example is a `RUN git clone` or `RUN curl` that fetches "latest" but whose cache layer never invalidates.

## Option 2: Rebuild All Services and Recreate

A quick, one-command rebuild of everything:

```bash
docker compose up -d --build --force-recreate
```

- `--build` rebuilds images before starting (using the cache).
- `--force-recreate` recreates containers even if their config is unchanged, ensuring they run from the freshly built image.

Add `-d` to run detached; without it, `up` stays in the foreground streaming logs. The reference command `docker compose up --build --force-recreate` (no `-d`) runs in the foreground, which is fine interactively but blocks your shell.

## Option 3: Clean Rebuild (Down, Build No-Cache, Up)

The most thorough routine rebuild — remove containers, rebuild images from scratch, start again:

```bash
docker compose down
docker compose build --no-cache
docker compose up -d
```

Plain `docker compose down` removes containers and the default network but **keeps named volumes**, so persistent data survives. Use this when you want certainty that images are rebuilt fresh without discarding data.

## Option 4: Remove Images First, Then Rebuild

When you want to clear out the old images too:

```bash
# Remove containers AND the images this project built/uses
docker compose down --rmi local     # only locally built (untagged-by-registry) images
# or
docker compose down --rmi all       # every image referenced by the compose file

docker compose up -d --build
```

`--rmi local` removes only images that don't have a custom tag (typically the ones Compose built), while `--rmi all` removes every image the file references, including pulled base images — which then have to be re-pulled. Prefer `--rmi local` unless you specifically want to re-pull everything.

> Neither `--rmi` flag removes volumes. To also discard data, add `-v` (`docker compose down -v`) — destructive, so be sure. See the volume notes in the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md).

## Which Option to Use

| Situation | Option |
|-----------|--------|
| Changed one service's code/Dockerfile | Option 1: `build [service]` (add `--no-cache` if a step is stale) |
| Quick rebuild of the whole stack | Option 2: `up -d --build --force-recreate` |
| Want certainty images are fresh, keep data | Option 3: `down` → `build --no-cache` → `up -d` |
| Also clear out old images | Option 4: `down --rmi local` → `up -d --build` |

Escalate only as far as you need: a cached rebuild is fastest, `--no-cache` is the fix for stale layers, and image removal is rarely necessary for routine changes.

## A Note on BuildKit Cache

With BuildKit (the default builder), `--no-cache` invalidates the normal layer cache, but explicit cache mounts (`RUN --mount=type=cache,...`) persist separately by design. If a dependency cache still looks stale after `--no-cache`, that cache mount is why; clear it in the Dockerfile logic or with `docker builder prune`. See [Docker Buildx and Multi-Platform Builds](articles/docker-buildx-multi-platform-builds.md).

## Verifying and Cleaning Up

```bash
# See the images Compose is using for each service
docker compose images

# Confirm containers are running the rebuilt image
docker compose ps

# Check the image ID a running container uses
docker inspect --format '{{ .Image }}' "$(docker compose ps -q web)"
```

Rebuilds leave dangling (untagged) images behind. Prune them periodically:

```bash
# Remove dangling images only (safe)
docker image prune

# Remove all unused images, including tagged ones not in use (aggressive)
docker image prune -a
```

`docker image prune -a` removes any image not currently used by a container, which can delete base images you'll need again and force a re-pull — use it deliberately.

## Common Pitfalls

- **Expecting `up` alone to rebuild.** It reuses an existing image; use `up --build` or `build` first.
- **`--force-recreate` without `--build`.** Recreates the container from the **same** image; it does not pick up code changes that require a rebuild.
- **Stale cached layers.** A step Docker thinks is unchanged can serve old content; `--no-cache` (or a cache-busting `ARG`) fixes it.
- **`--rmi all` re-pulling everything.** It removes base images too, adding download time; `--rmi local` is usually what you want.
- **Assuming `down` deletes data.** Named volumes persist unless you add `-v`.

## Quick Reference

```bash
# One service, cached / fresh
docker compose build web
docker compose build --no-cache web
docker compose up -d --build web

# Whole stack, quick
docker compose up -d --build --force-recreate

# Clean rebuild, keep data
docker compose down && docker compose build --no-cache && docker compose up -d

# Rebuild and drop old locally built images
docker compose down --rmi local && docker compose up -d --build

# Inspect and clean up
docker compose images
docker compose ps
docker image prune
```

For related material, see [Docker Compose Build Configuration and Build Context](articles/docker-compose-build-configuration.md), [Updating Docker Compose Containers](articles/docker-compose-updating-containers.md), [Tagging Built Images in Docker Compose](articles/docker-compose-image-tagging.md), and the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md).
