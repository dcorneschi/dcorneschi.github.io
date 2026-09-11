# Tagging Built Images in Docker Compose

When Docker Compose builds an image, the `image` field controls what that image is named and tagged. Getting tagging right makes builds reproducible, rollbacks simple, and registry pushes predictable. This guide covers declarative tagging in the Compose file, dynamic and environment-based tags, applying multiple tags, and pushing.

For the full build block (`context`, `dockerfile`, `args`, `target`), see [Docker Compose Build Configuration and Build Context](articles/docker-compose-build-configuration.md). For why fixed tags matter, see [Pin Docker Image Versions Instead of latest](articles/docker-image-version-pinning.md).

## The image Field with build

When a service has both `build` and `image`, Compose builds from the Dockerfile and tags the result with the `image` value:

```yaml
services:
  myapp:
    build: .
    image: myapp:1.0.0    # the built image is tagged myapp:1.0.0
```

Without `build`, `image` just tells Compose which image to **pull**. With `build`, it becomes the **tag applied to the build output**. This is the declarative, version-controlled way to name builds, and it is the recommended approach.

> Avoid tagging builds `:latest`. A moving `latest` makes it impossible to tell which build is running or to roll back cleanly. Prefer an explicit version.

## Dynamic Tags with Interpolation

Use variable interpolation so the tag can change per build without editing the file. Compose resolves `${...}` from the shell or the project `.env` before building:

```yaml
services:
  myapp:
    build:
      context: .
      dockerfile: Dockerfile
    image: myregistry.example.com/myapp:${VERSION:-0.0.0-dev}
```

```bash
# Tag the build explicitly
VERSION=1.2.3 docker compose build

# Falls back to the default (0.0.0-dev) when VERSION is unset
docker compose build
```

Prefer a meaningful default over `latest`. A default like `0.0.0-dev` makes an untagged local build obvious rather than masquerading as a release.

See [Defining Variables in Docker Compose](articles/docker-compose-variables-env.md) for how interpolation and `.env` work.

### Common dynamic-tag sources

```bash
# Semantic version from a variable or CI tag
VERSION=1.4.2 docker compose build

# Short git commit for traceable, unique builds
VERSION=$(git rev-parse --short HEAD) docker compose build

# CI-provided tag (example variable names)
VERSION="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}" docker compose build
```

A git-SHA tag is useful in CI because every build is uniquely identifiable and traceable back to a commit.

## Building and Tagging

```bash
# Build all services, applying each service's image tag
docker compose build

# Build a single service
docker compose build myapp

# Build without cache
docker compose build --no-cache

# Confirm the resulting tag
docker compose config | grep image
docker images myregistry.example.com/myapp
```

`docker compose build` applies the `image` tag; `docker compose up --build` builds and starts in one step.

## Applying Multiple Tags

Compose assigns one `image` value per service, so additional tags are added afterward with `docker tag`. This is normal: tag once at build, then add aliases such as a floating channel or a registry path.

```bash
# Build with the canonical version tag
VERSION=1.0.0 docker compose build myapp

# Add extra tags that point at the same image
docker tag myapp:1.0.0 myapp:1.0
docker tag myapp:1.0.0 myregistry.example.com/myapp:1.0.0
```

`docker tag` only creates a new reference to the existing image; it does not rebuild. If you maintain a `latest`-style channel, point it at a specific version deliberately as part of a release step rather than building straight to `latest`.

## Environment-Specific Tags

You can encode an environment in the tag, though a version or SHA is usually a better identifier. If you do, still avoid an ambiguous default:

```yaml
services:
  myapp:
    build: .
    image: myapp:${ENVIRONMENT:-dev}
```

```bash
ENVIRONMENT=staging docker compose build
```

Prefer combining environment with a version (`myapp:1.4.2` promoted through dev → staging → prod) over environment-only tags, so the exact artifact is identifiable in every environment. Promoting the same image digest across environments is stronger still; see [Pin Docker Image Versions Instead of latest](articles/docker-image-version-pinning.md).

## Pushing Tagged Images

Compose can push the images it built and tagged:

```bash
# Push all service images defined with an image field
docker compose push

# Push a single service
docker compose push myapp
```

The `image` value must include the registry host for a non-Docker-Hub target (for example `myregistry.example.com/myapp:1.4.2`). Build, tag, then push as a pipeline:

```bash
VERSION=1.4.2 docker compose build && docker compose push
```

## Best Practices

- Set the `image` field in the Compose file so tagging is declarative and version-controlled.
- Use interpolation (`${VERSION}`) for dynamic tags, with a clear non-`latest` default.
- Prefer semantic versions (`1.4.2`) or git SHAs over `latest` for build output.
- Add extra tags with `docker tag` after the build; move a `latest`/channel tag deliberately at release time.
- Include the registry host in the tag when pushing to a private registry.
- For the strongest reproducibility, deploy by digest and promote the same image across environments.

## Quick Reference

```yaml
services:
  myapp:
    build:
      context: .
      dockerfile: Dockerfile
    image: myregistry.example.com/myapp:${VERSION:-0.0.0-dev}
```

```bash
# Build with an explicit version
VERSION=1.4.2 docker compose build myapp

# Build with a git SHA
VERSION=$(git rev-parse --short HEAD) docker compose build myapp

# Add extra tags to the built image
docker tag myregistry.example.com/myapp:1.4.2 myregistry.example.com/myapp:1.4

# Build and push
VERSION=1.4.2 docker compose build && docker compose push myapp

# Verify the resolved tag
docker compose config | grep image
```

For related material, see [Docker Compose Build Configuration and Build Context](articles/docker-compose-build-configuration.md), [Pin Docker Image Versions Instead of latest](articles/docker-image-version-pinning.md), and the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md).
