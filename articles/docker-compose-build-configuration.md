# Docker Compose Build Configuration and Build Context

The `build` key in a Docker Compose service tells Compose to build an image from a Dockerfile instead of pulling one. Its most important and most misunderstood field is `context`, which defines what files the build can see. This guide covers the build options, how context works, and the patterns that keep builds fast and correct.

For Dockerfile authoring and multi-stage builds, see [Building Docker Images with Dockerfile](articles/docker-build-image-guide.md). For multi-architecture builds, see [Docker Buildx and Multi-Platform Builds](articles/docker-buildx-multi-platform-builds.md).

## Basic Build Options

### Build from the current directory

The short form sets the build context; Compose looks for `Dockerfile` inside it.

```yaml
services:
  app:
    build: .
```

### Set an explicit context path

```yaml
services:
  app:
    build:
      context: ./app
```

### Use a custom Dockerfile name

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.prod
```

The `dockerfile` path is resolved **relative to the context**, not to the Compose file. Keep the Dockerfile inside the context directory.

## Build Arguments, Targets, and Labels

### Build arguments

`args` supplies values to `ARG` instructions at build time. Both list and map forms are valid; pick one style per block for readability.

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
        API_URL: "https://api.example.com"
        BUILD_VERSION: "1.2.3"
```

Each name must have a matching `ARG` in the Dockerfile to take effect. Do not pass secrets (tokens, passwords, keys) as build args — they are visible in the image history. Use BuildKit `--secret` mounts instead; see the secrets section of [Docker Buildx and Multi-Platform Builds](articles/docker-buildx-multi-platform-builds.md).

### Target a specific stage

For a multi-stage Dockerfile, `target` builds up to the named stage.

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
```

### Cache sources

```yaml
services:
  app:
    build:
      context: .
      cache_from:
        - myapp:latest
        - node:20-alpine
```

### Labels

```yaml
services:
  app:
    build:
      context: .
      labels:
        com.example.version: "1.0"
        com.example.environment: "production"
```

### Build network and platform

```yaml
services:
  app:
    build:
      context: .
      network: host       # or none, or container:<name>
      platform: linux/amd64
```

Note that `platform` here requests a single target platform. Building one image for several architectures at once needs buildx; see the multi-platform guide.

## build and image Together

When a service specifies both `build` and `image`, Compose builds the image and tags it with the `image` value. On later runs it can reuse that tag.

```yaml
services:
  app:
    build: .
    image: myapp:1.0.0
```

Prefer a real version tag over `latest` so the built artifact is identifiable and reproducible; see [Pin Docker Image Versions Instead of latest](articles/docker-image-version-pinning.md).

## Why the Build Context Matters

The **build context** is the set of files Compose packages and sends to the builder. The Dockerfile can only `COPY`/`ADD` files that live inside this context, and the context size directly affects build speed.

### The Dockerfile can only see the context

```yaml
services:
  app:
    build:
      context: ./my-app
```

```dockerfile
# Paths in COPY are relative to the context (./my-app)
COPY package.json ./
COPY src/ ./src/
```

A `COPY ../something` that points outside the context fails, because those files were never sent to the builder.

### Smaller context, faster and safer builds

Everything in the context is transferred to the build backend, even files the Dockerfile never copies. A bloated context slows builds and risks pulling in files you did not intend.

```yaml
# Sends the entire project
services:
  app:
    build:
      context: .

# Sends only what the app build needs
services:
  app:
    build:
      context: ./app
```

The primary tool for trimming the context is a `.dockerignore` file in the context directory. It excludes files even when the context is broad:

```dockerignore
.git
.gitignore
node_modules
*.log
.env
**/.DS_Store
Dockerfile*
docker-compose*.yml
```

Excluding `.git`, local `node_modules`, and secret files such as `.env` speeds builds and avoids leaking sensitive data into an image layer.

### Context versus Dockerfile location

The Dockerfile can sit in a subdirectory of the context. The `dockerfile` path is relative to `context`:

```yaml
services:
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile   # ./docker/Dockerfile, still inside the context
```

Because Compose requires the Dockerfile to be reachable within (or given relative to) the context, keep it inside the context tree rather than trying to point at a path above it.

## Common Project Layouts

### Per-service context (microservices)

Each service builds from its own directory, keeping contexts small and independent.

```yaml
services:
  user-service:
    build:
      context: ./services/user

  order-service:
    build:
      context: ./services/order
```

### Monorepo with shared code

When services share libraries at the repository root, use a root context and point `dockerfile` at each service's Dockerfile so the build can reach the shared code.

```yaml
services:
  api:
    build:
      context: .
      dockerfile: api/Dockerfile

  web:
    build:
      context: .
      dockerfile: web/Dockerfile
```

Here a root-level `.dockerignore` matters even more, because the context is the whole repository.

## Complete Example

```yaml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.prod
      target: production
      args:
        NODE_ENV: production
        API_URL: ${API_URL:-http://localhost:3000}
      cache_from:
        - frontend:latest
    image: frontend:1.0.0
    ports:
      - "80:80"

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      target: runtime
      args:
        DATABASE_URL: ${DATABASE_URL}
    image: backend:1.0.0
    depends_on:
      - database
    environment:
      NODE_ENV: production
      JWT_SECRET_FILE: /run/secrets/jwt
    secrets:
      - jwt

  database:
    image: postgres:16.4
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:

secrets:
  jwt:
    file: ./secrets/jwt.txt
  db_password:
    file: ./secrets/db_password.txt
```

Compared with passing `JWT_SECRET` and the database password as plain build args or environment values, this keeps secrets out of image history and out of the committed Compose file by using Docker secrets and `_FILE` conventions. See [Defining Variables in Docker Compose](articles/docker-compose-variables-env.md).

## Building and Verifying

```bash
# Build (or rebuild) images for the stack
docker compose build

# Build without cache
docker compose build --no-cache

# Build a single service
docker compose build backend

# Build and start
docker compose up --build

# Show the fully resolved build config (interpolated)
docker compose config
```

The top-level `version:` key is obsolete in current Compose and only emits a warning; omit it.

## Common Pitfalls

- Referencing files outside the context with `COPY ../...`; they were never sent to the builder.
- A missing or incomplete `.dockerignore`, so `.git` or `node_modules` bloats the context.
- Expecting `dockerfile:` to be relative to the Compose file; it is relative to `context`.
- Passing secrets through `args`; they persist in image history. Use BuildKit `--secret`.
- Forgetting that `platform:` requests one platform; multi-arch needs buildx.
- Assuming `build` always rebuilds; Compose reuses an existing image unless you run `docker compose build` or `up --build`.

## Quick Reference

```yaml
services:
  app:
    build:
      context: ./app                 # what the builder can see
      dockerfile: Dockerfile.prod    # relative to context
      target: production             # multi-stage target
      args:
        NODE_ENV: production
      cache_from:
        - myapp:latest
      labels:
        com.example.version: "1.0"
      network: host
      platform: linux/amd64
    image: myapp:1.0.0               # tag applied to the built image
```

```bash
docker compose build --no-cache app
docker compose up --build
docker compose config
```

For related material, see [Building Docker Images with Dockerfile](articles/docker-build-image-guide.md), [Docker Buildx and Multi-Platform Builds](articles/docker-buildx-multi-platform-builds.md), and the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md).
