<img src="/articles/images/docker-logo.svg" alt="Docker" width="150">

# Docker Compose: Running Containers Without Root

## Overview

Running containers as root is the default behavior in Docker, but it's a security risk. If a container is compromised, an attacker running as UID 0 inside the container has a much easier path to escaping to the host — especially if volumes are mounted or capabilities are not dropped.

Running as a non-root user introduces friction: file permission mismatches, bind mount ownership issues, and images that assume root. This article covers what to check and how to configure things correctly when deploying a Docker Compose service as a non-root user.

## The Checklist

Before deploying a non-root container, verify these items:

| # | Check | Why |
|---|-------|-----|
| 1 | Does the image support non-root? | Some images require root for their entrypoint |
| 2 | What UID/GID does the container process need? | Must match file ownership on bind mounts |
| 3 | Are bind-mounted files/directories writable? | Host UID must match container UID |
| 4 | Does the app write to container-internal paths? | Paths like `/tmp`, `/var/log`, app data dirs must be writable |
| 5 | Does the app bind to a privileged port (<1024)? | Non-root can't bind to ports below 1024 without capabilities |
| 6 | Are volume permissions set correctly? | Named volumes inherit permissions from the image; bind mounts don't |

## Checking if an Image Supports Non-Root

### Inspect the Dockerfile / Image

```bash
# Check the default USER in the image
docker inspect <image> --format '{{.Config.User}}'

# If empty, the image runs as root by default
# If it shows something like "1000" or "appuser", it already runs non-root
```

```bash
# Check what user the entrypoint expects
docker inspect <image> --format '{{json .Config.Entrypoint}}'
docker inspect <image> --format '{{json .Config.Cmd}}'
```

```bash
# Run the image and check the running user
docker run --rm <image> id
# uid=0(root) gid=0(root) groups=0(root)   ← runs as root
# uid=1000(app) gid=1000(app) groups=1000(app)  ← already non-root
```

### Signs That an Image Requires Root

| Indicator | Explanation |
|-----------|-------------|
| Entrypoint runs `apt-get`, `apk add`, or package installs | Package managers require root |
| Entrypoint modifies `/etc/` files | System config files are owned by root |
| Entrypoint starts services with `service` or `systemctl` | Service management requires root |
| App binds to port 80 or 443 | Privileged ports require root (or `CAP_NET_BIND_SERVICE`) |
| Entrypoint uses `gosu` or `su-exec` to drop privileges | Image expects to start as root, then drop to a lower user |
| Entrypoint runs `chown` on startup | Trying to fix permissions at runtime — needs root to do so |

### Signs That an Image Supports Non-Root

| Indicator | Explanation |
|-----------|-------------|
| `USER` directive in Dockerfile (not `USER root`) | Image was designed to run non-root |
| App listens on a high port (8080, 9090, etc.) | No privileged port binding |
| Data directories created with specific UID/GID in Dockerfile | Paths are pre-owned by the non-root user |
| Documentation mentions `--user` or `PUID`/`PGID` | Explicitly supports user remapping |

### Common Images and Their Non-Root Support

| Image | Default User | Non-Root Support | Notes |
|-------|-------------|-----------------|-------|
| `nginx` | root | `nginxinc/nginx-unprivileged` variant available | Official unprivileged image listens on 8080 |
| `postgres` | root (drops to `postgres`) | Uses `gosu` — needs to start as root | Entrypoint runs `chown` then drops to UID 999 |
| `redis` | root (drops to `redis`) | Uses `gosu` — needs to start as root | Drops to UID 999 |
| `node` | root | Supports `--user` / `USER` | No privileged port by default |
| `python` | root | Supports `--user` / `USER` | No privileged port by default |
| `traefik` | root | Supports non-root since v2.x | Needs access to Docker socket if used as provider |
| `bitnami/*` | UID 1001 | All Bitnami images run as non-root | Designed for non-root from the start |
| `linuxserver/*` | root (configurable) | Uses `PUID`/`PGID` env vars | Entrypoint fixes permissions then drops to specified UID |
| `grafana/grafana` | UID 472 | Runs non-root by default | Data dir must be owned by 472:472 |
| `prom/prometheus` | `nobody` (65534) | Runs non-root by default | Data dir must be owned by 65534:65534 |

## Understanding UID/GID Mapping

Docker containers share the host kernel. UIDs inside the container map directly to UIDs on the host — UID 1000 inside the container is the same UID 1000 on the host.

```
┌─────────────────────────────────────────────────────────┐
│ Host                                                    │
│                                                         │
│   /srv/app/data (owned by UID 1000:GID 1000)            │
│       │                                                 │
│       ▼ bind mount                                      │
│   ┌─────────────────────────────────────────────────┐   │
│   │ Container                                       │   │
│   │   /data (seen as UID 1000:GID 1000)             │   │
│   │   Process running as UID 1000 → CAN write       │   │
│   │   Process running as UID 0    → CAN write       │   │
│   │   Process running as UID 33   → CANNOT write    │   │
│   └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Key Rules

- **UID 0 (root) inside the container** can read/write anything (unless read-only mount or `--cap-drop ALL`)
- **Non-root UID** must match the file ownership or be in the correct group
- **User names don't matter** — only the numeric UID/GID matters for permission checks
- **Named volumes** inherit ownership from the image's directory at first creation
- **Bind mounts** always reflect the host filesystem ownership — Docker does not remap

## Configuring UID/GID in Docker Compose

### Method 1: `user` Directive

```yaml
services:
  app:
    image: myapp:latest
    user: "1000:1000"    # UID:GID
    volumes:
      - ./data:/app/data
```

This overrides whatever `USER` is set in the image. The process runs as UID 1000, GID 1000.

### Method 2: Environment Variables (linuxserver.io Pattern)

```yaml
services:
  app:
    image: linuxserver/sonarr
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - ./config:/config
      - /media:/media
```

The entrypoint script creates a user with the specified UID/GID, fixes permissions on mounted paths, then runs the app as that user. This pattern requires the container to start as root.

### Method 3: Build with a Specific UID

```dockerfile
FROM node:20-slim

ARG UID=1000
ARG GID=1000

RUN groupadd -g ${GID} appgroup && \
    useradd -u ${UID} -g appgroup -m appuser && \
    mkdir -p /app/data && \
    chown -R ${UID}:${GID} /app

USER appuser
WORKDIR /app
COPY --chown=appuser:appgroup . .
RUN npm ci --omit=dev

EXPOSE 8080
CMD ["node", "server.js"]
```

```yaml
services:
  app:
    build:
      context: .
      args:
        UID: 1000
        GID: 1000
    volumes:
      - ./data:/app/data
```

### Method 4: Match Host User Dynamically

```yaml
services:
  app:
    image: myapp:latest
    user: "${DOCKER_UID:-1000}:${DOCKER_GID:-1000}"
    volumes:
      - ./data:/app/data
```

```bash
# Export before running (note: UID is read-only in bash, use a different var or .env)
export DOCKER_UID=$(id -u)
export DOCKER_GID=$(id -g)
docker compose up -d
```

Or in `.env`:

```env
DOCKER_UID=1000
DOCKER_GID=1000
```

## File Permissions for Bind Mounts

### The Problem

```bash
# Host: files owned by your user (UID 1000)
$ ls -la ./data/
drwxr-xr-x  2 dan dan 4096 Jan 10 12:00 .
-rw-r--r--  1 dan dan  512 Jan 10 12:00 config.json

# Container runs as UID 33 (www-data in nginx)
# → Permission denied when writing to /app/data/
```

### The Fix: Match UIDs

**Option A — Change host file ownership to match the container UID:**

```bash
# If container runs as UID 33 (www-data)
sudo chown -R 33:33 ./data/
```

**Option B — Run the container as your host UID:**

```yaml
services:
  app:
    image: myapp:latest
    user: "1000:1000"
    volumes:
      - ./data:/app/data
```

**Option C — Use group permissions:**

```bash
# Add your host user and the container user to a shared group
# Make files group-writable
chmod -R g+w ./data/

# In the container, ensure the process's GID matches
```

**Option D — Use an init script to fix permissions:**

```yaml
services:
  app:
    image: myapp:latest
    user: "0:0"    # start as root
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        chown -R 1000:1000 /app/data
        exec su-exec 1000:1000 /app/start.sh
    volumes:
      - ./data:/app/data
```

### Permission Matrix

| Scenario | Container UID | Host File Owner | Result |
|----------|--------------|----------------|--------|
| Default (no `user:`) | 0 (root) | Any | Works (root can access everything) |
| `user: "1000:1000"` | 1000 | 1000:1000 | Works |
| `user: "1000:1000"` | 1000 | 0:0 (root) | Read-only (if `o+r`) or denied |
| `user: "1000:1000"` | 1000 | 33:33 | Denied (unless `o+rw` or shared group) |
| `user: "33:33"` | 33 | 1000:1000 | Denied |

## Named Volumes vs Bind Mounts

| Aspect | Named Volume | Bind Mount |
|--------|-------------|-----------|
| Ownership | Copies from image on first create | Reflects host filesystem |
| Permission issues | Rare (auto-populated) | Common |
| Can `chown` from host | No (stored in Docker's area) | Yes |
| Use case | Database data, caches | Config files, source code, shared data |

```yaml
services:
  db:
    image: postgres:16
    volumes:
      # Named volume — Postgres manages ownership internally
      - pgdata:/var/lib/postgresql/data
      # Bind mount — YOU must ensure correct ownership
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro

volumes:
  pgdata:
```

> **Tip:** Named volumes copy the directory contents and permissions from the image on first creation. If the image has `/var/lib/postgresql/data` owned by `postgres` (UID 999), the named volume will inherit that ownership. This does NOT happen with bind mounts.

## Handling Privileged Ports

Non-root processes cannot bind to ports below 1024. You have several options:

### Option 1: Use a High Port Inside, Map to Low Port Outside

```yaml
services:
  web:
    image: nginxinc/nginx-unprivileged   # listens on 8080 internally
    user: "101:101"             # nginx user
    ports:
      - "80:8080"              # host 80 → container 8080
      - "443:8443"
```

### Option 2: Add NET_BIND_SERVICE Capability

```yaml
services:
  web:
    image: myapp:latest
    user: "1000:1000"
    cap_add:
      - NET_BIND_SERVICE
    cap_drop:
      - ALL
    ports:
      - "80:80"
```

> **Note:** `CAP_NET_BIND_SERVICE` allows binding to any port below 1024. This is more secure than running as root, but it's still a capability grant.

### Option 3: Set sysctl to Allow Unprivileged Port Binding

```yaml
services:
  web:
    image: myapp:latest
    user: "1000:1000"
    sysctls:
      - net.ipv4.ip_unprivileged_port_start=0
    ports:
      - "80:80"
```

This sets the lowest unprivileged port to 0, allowing any user to bind any port. This only affects the container's network namespace.

## Common Issues and Fixes

### Issue 1: Container Exits with Permission Denied

```
Error: EACCES: permission denied, open '/app/data/config.json'
```

**Diagnosis:**

```bash
# Check what UID the container runs as
docker compose exec app id

# Check file ownership on the mount
docker compose exec app ls -la /app/data/

# Compare UIDs
```

**Fix:** Match the host file ownership to the container UID.

### Issue 2: Cannot Write to /tmp or Internal Paths

Some apps write to `/tmp`, `/var/log`, or other internal paths that are owned by root in the image.

```yaml
services:
  app:
    image: myapp:latest
    user: "1000:1000"
    tmpfs:
      - /tmp
      - /var/log/app
    volumes:
      - ./data:/app/data
```

`tmpfs` mounts are writable by any user and don't persist — perfect for temp files and logs.

### Issue 3: npm/pip/yarn Cache Permission Errors

Package managers cache to user home directories that may not exist or be writable.

```yaml
services:
  app:
    build:
      context: .
    user: "1000:1000"
    environment:
      - HOME=/app
      - npm_config_cache=/app/.npm
    volumes:
      - ./data:/app/data
    tmpfs:
      - /app/.npm
```

### Issue 4: Container Creates Files Owned by Root on Host

When a container runs as root and writes to a bind mount, the files appear as root on the host:

```bash
$ ls -la ./data/
-rw-r--r-- 1 root root 1024 Jan 10 12:00 output.json   # created by container as root
```

**Fix:** Run the container as your UID:

```yaml
services:
  app:
    image: myapp:latest
    user: "1000:1000"
    volumes:
      - ./data:/app/data
```

### Issue 5: Entrypoint Requires Root (gosu/su-exec Pattern)

Many official images (Postgres, Redis, MongoDB) need to start as root, then drop privileges:

```
# Postgres entrypoint does:
# 1. Start as root
# 2. chown data directories
# 3. exec gosu postgres postgres ...
```

For these images, **do not set `user:` in docker-compose.yml**. The image handles privilege dropping internally. Instead, verify it's working:

```bash
# After container starts, check the running process UID
docker compose exec db ps aux
# Should show postgres process running as "postgres" (UID 999), not root
```

### Issue 6: Read-Only Filesystem Errors

If you set `read_only: true` for security hardening, ensure writable paths are provided:

```yaml
services:
  app:
    image: myapp:latest
    user: "1000:1000"
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
    volumes:
      - app-data:/app/data
```

## Security Hardening Checklist

A complete non-root deployment should also consider:

```yaml
services:
  app:
    image: myapp:latest
    user: "1000:1000"
    read_only: true
    security_opt:
      - no-new-privileges:true    # prevent setuid escalation
    cap_drop:
      - ALL                       # drop all capabilities
    cap_add:
      - NET_BIND_SERVICE          # only if needed for low ports
    tmpfs:
      - /tmp
    volumes:
      - app-data:/app/data
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
```

| Directive | Purpose |
|-----------|---------|
| `user: "1000:1000"` | Run as non-root |
| `read_only: true` | Prevent writes to container filesystem |
| `no-new-privileges:true` | Prevent privilege escalation via setuid binaries |
| `cap_drop: ALL` | Remove all Linux capabilities |
| `cap_add: [...]` | Add back only what's needed |
| `tmpfs: [/tmp]` | Provide writable temp space without persisting |

## Debugging Permissions

### Quick Diagnostic Commands

```bash
# What UID is the process running as inside the container?
docker compose exec <service> id

# What's the ownership of the mounted directory?
docker compose exec <service> ls -lan /app/data

# What UID owns the process?
docker compose exec <service> cat /proc/1/status | grep -E "^(Uid|Gid)"

# Can the process write to the directory?
docker compose exec <service> sh -c 'touch /app/data/test && echo "OK" || echo "DENIED"'

# What's the host-side ownership?
ls -lan ./data/

# What UID does the image default to?
docker inspect <image> --format '{{.Config.User}}'
```

### The Numeric UID Trap

```bash
# Inside container:
$ ls -la /app/
drwxr-xr-x 2 1000 1000 4096 Jan 10 12:00 data

# This might show as a name if /etc/passwd has an entry:
$ ls -la /app/
drwxr-xr-x 2 appuser appgroup 4096 Jan 10 12:00 data

# Always use -n flag for numeric UIDs (no name resolution)
$ ls -lan /app/
drwxr-xr-x 2 1000 1000 4096 Jan 10 12:00 data
```

Names are cosmetic. Permission checks are always numeric.

## Complete Working Example

A full example deploying a Node.js app with Postgres, all running non-root with proper permissions:

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      args:
        UID: 1000
        GID: 1000
    user: "1000:1000"
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    tmpfs:
      - /tmp
    ports:
      - "3000:3000"
    volumes:
      - ./uploads:/app/uploads   # host dir must be owned by 1000:1000
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/myapp
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    # Do NOT set user: — postgres entrypoint needs root to chown then drops to UID 999
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - DAC_OVERRIDE    # needed for chown in entrypoint
      - FOWNER          # needed for chown in entrypoint
      - SETUID          # needed for gosu
      - SETGID          # needed for gosu
    volumes:
      - pgdata:/var/lib/postgresql/data    # named volume — permissions handled by image
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d myapp"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

```bash
# Prepare host directories
mkdir -p ./uploads
chown 1000:1000 ./uploads
```

## Summary

| Rule | Explanation |
|------|-------------|
| Check image default USER | Know what UID the image expects to run as |
| Match bind mount ownership to container UID | Docker doesn't remap UIDs on bind mounts |
| Use `ls -lan` (numeric) for diagnosis | User names are irrelevant — only UID/GID numbers matter |
| Don't set `user:` on images that use gosu/su-exec | They need root to start, then drop privileges themselves |
| Use named volumes for database data | They inherit permissions from the image automatically |
| Use `tmpfs` for ephemeral writable paths | /tmp, caches, pid files |
| Map high ports inside to low ports outside | Avoids needing `CAP_NET_BIND_SERVICE` |
| Always add `no-new-privileges` and `cap_drop: ALL` | Defense in depth even when running non-root |
