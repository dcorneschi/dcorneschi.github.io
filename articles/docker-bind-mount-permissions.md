# Fixing Docker Bind-Mount Permission Errors

Most "permission denied" errors with Docker bind mounts come down to one mismatch: the UID or GID the container process runs as does not match the ownership of the host files it is mounting. Docker does not remap ownership for bind mounts — the host's numeric UID/GID is what the container sees.

The core principle is simple:

> **Match the container's UID/GID to the ownership of the host files it must read and write.**

This guide explains the permission model, the reliable ways to align it, platform differences, and how to diagnose problems. For choosing bind mounts versus named volumes, see [Docker Compose: Bind Mounts vs Named Volumes](articles/docker-compose-volumes-vs-bind-mounts.md); for the mechanics of the `user:` directive, see [Set the Container User (UID/GID) in Docker Compose](articles/docker-compose-set-container-user.md).

## Why This Happens

Containers share the host kernel, and file permissions are enforced by numeric UID and GID, not usernames. A file owned by UID 1000 on the host appears owned by UID 1000 inside the container, regardless of whether a user with that ID exists in the image.

- If the container runs as root (the default) and writes to a bind mount, the new files are **root-owned on the host**, and you then need `sudo` to touch them.
- If the container runs as a non-root UID that does not own the mounted files, writes fail with permission denied.

Named volumes avoid much of this because Docker initializes their ownership from the image, which is why they are the smoother choice for data you never edit from the host.

## Aligning UID/GID

### Match host ownership to the container user

Set the service user, then own the host directory to the same IDs:

```yaml
services:
  app:
    image: myapp:1.4.2
    user: "1000:1000"   # UID:GID
    volumes:
      - ./data:/app/data
```

```bash
# Match host ownership to the container user
sudo chown -R 1000:1000 ./data

# Or use your own account's IDs
sudo chown -R "$(id -u):$(id -g)" ./data
```

### Make the user configurable

Parameterize the IDs so the same file works across machines:

```yaml
services:
  app:
    image: myapp:1.4.2
    user: "${DOCKER_UID:-1000}:${DOCKER_GID:-1000}"
    volumes:
      - ./data:/app/data
```

```bash
export DOCKER_UID=$(id -u) DOCKER_GID=$(id -g)
docker compose up -d
```

> **bash caveat:** avoid the common `UID=$(id -u) GID=$(id -g) docker compose up` form. `UID` is read-only in bash and `GID` is usually unset, so this often does not behave as intended. Use distinct names such as `DOCKER_UID`/`DOCKER_GID`.

### Persist the IDs in .env

Compose reads a project `.env` for interpolation, so you can record the IDs once:

```bash
cat > .env <<EOF
DOCKER_UID=$(id -u)
DOCKER_GID=$(id -g)
EOF
```

```yaml
services:
  app:
    image: myapp:1.4.2
    user: "${DOCKER_UID}:${DOCKER_GID}"
    volumes:
      - ./data:/app/data
```

See [Defining Variables in Docker Compose](articles/docker-compose-variables-env.md) for how interpolation and `.env` differ from container environment variables.

## Pre-create Directories

If a bind-mount source path does not exist, Docker creates it **as root**, which reintroduces the very problem you are trying to avoid. Create and own directories before the first `up`:

```bash
mkdir -p ./data ./logs ./config
sudo chown -R "$(id -u):$(id -g)" ./data ./logs ./config

chmod 755 ./data ./logs
chmod 700 ./config          # tighter for sensitive config
```

Use `755`/`644` for ordinary data, but keep secrets at `700`/`600`; `755` and `644` are world-readable. For a refresher on the numeric model, see [Linux File Permissions Guide](articles/linux-file-permissions.md).

## Read-Only and tmpfs Mounts

Mount anything the container should not modify as read-only, which sidesteps write-permission questions entirely:

```yaml
services:
  app:
    image: myapp:1.4.2
    user: "1000:1000"
    volumes:
      - ./config:/app/config:ro    # read-only
      - ./data:/app/data:rw        # read-write, needs matching ownership
      - type: tmpfs
        target: /app/cache         # in-memory scratch, no host ownership
```

Read-only config can even stay root-owned on the host, since the container only needs to read it.

## When Named Volumes Are Better

For data that does not need direct host access, a named volume avoids ownership friction because Docker manages it:

```yaml
services:
  db:
    image: postgres:16.4
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

Databases are the classic case: they run as a specific in-image UID (for example `postgres`), and a named volume spares you from hand-owning the host directory to match.

## Platform Differences

### Linux

UID/GID map directly between host and container. Matching ownership as described above is both necessary and sufficient. This is the environment where getting the numbers right matters most.

### macOS (Docker Desktop)

Docker Engine runs in a Linux VM, and the file-sharing layer translates access for bind mounts, so mounts under your home directory often "just work" without manual `chown`. Two caveats:

- The host path must be within Docker Desktop's shared paths (**Settings → Resources → File Sharing**).
- Ownership semantics differ from native Linux, so do not rely on macOS-side `chown`/`chmod` mapping cleanly into the container. Still set `user:` for predictability and security.

### Windows (WSL2, Docker Desktop)

Similar to macOS: permissions are largely abstracted for bind-mount access. For best results, keep project files inside the WSL2 filesystem rather than on a `/mnt/c` Windows path, and still set `user:` for consistency.

## Diagnosing Problems

Compare ownership on both sides:

```bash
# Host
ls -la ./data

# Inside the container
docker compose exec app ls -la /app/data
```

Confirm the user the container actually runs as:

```bash
docker compose exec app id
```

Test a write and check who owns the result:

```bash
docker compose exec app sh -c 'touch /app/data/.permtest'
ls -la ./data/.permtest       # should be your UID/GID, not root
rm ./data/.permtest
```

If the file comes back root-owned, the container is running as root; set `user:`. If the write fails outright, the container UID does not match the directory owner; fix ownership or the `user:` value.

## Fixing a Mismatch

```bash
# Option 1: change host ownership to match the container user
sudo chown -R "$(id -u):$(id -g)" ./data

# Option 2: share access via a common group and make the tree group-writable
sudo chgrp -R mygroup ./data
sudo chmod -R g+rw ./data

# Option 3: run the container as the UID that already owns the files
#   set user: "<owner-uid>:<owner-gid>" in the compose file
```

Prefer changing ownership or the `user:` value over loosening permissions. Reserve group-writable modes for genuinely shared data, and never make secrets world- or group-readable.

## Anti-Patterns

- **Root container writing to a bind mount.** Omitting `user:` leaves new host files root-owned, forcing `sudo` to manage them.
- **Mismatched UID/GID.** Setting `user: "999:999"` while the host directory is owned by `1000` yields permission denied.
- **Overriding an image's expected user blindly.** Some images (databases, `nginx`) expect to access files as a specific in-image user; forcing an unrelated `user:` can break them. Check with `docker run --rm <image> id` first.
- **Letting Docker create missing bind-mount paths.** Nonexistent sources are created as root. Pre-create and own them.
- **Loosening permissions instead of fixing ownership.** `chmod -R 777` "works" but is a security problem; align IDs instead.

## Production Consistency

Use a stable, dedicated UID/GID across all hosts so ownership is portable:

```yaml
services:
  app:
    image: myapp:1.4.2
    user: "1001:1001"
    volumes:
      - /opt/docker/myapp/data:/app/data
```

```bash
# Create a matching service account (no login) on each server
sudo useradd -u 1001 -M -s /usr/sbin/nologin appuser
sudo chown -R 1001:1001 /opt/docker/myapp/data
```

Document the requirement so deployments are reproducible: which directories, which UID/GID, and which modes. For the host directory layout itself, see [Where to Store Docker Compose Bind-Mount Data on the Host](articles/docker-compose-host-data-layout.md).

## Setup Script

A small script makes the setup repeatable per project:

```bash
#!/usr/bin/env bash
set -euo pipefail

uid="$(id -u)"
gid="$(id -g)"

mkdir -p data logs config
chown -R "$uid:$gid" data logs config
chmod 755 data logs
chmod 700 config

printf 'DOCKER_UID=%s\nDOCKER_GID=%s\n' "$uid" "$gid" > .env

echo "Prepared directories and wrote .env (DOCKER_UID=$uid, DOCKER_GID=$gid)."
echo "Start with: docker compose up -d"
```

With `DOCKER_UID`/`DOCKER_GID` in `.env`, `docker compose up -d` picks them up automatically for a `user: "${DOCKER_UID}:${DOCKER_GID}"` service.

## Summary

| Scenario | Recommendation |
|----------|----------------|
| Development | Match your host UID/GID: `user: "${DOCKER_UID}:${DOCKER_GID}"` |
| Production | Fixed, documented UID/GID across hosts: `user: "1001:1001"` |
| Read-only data | Mount `:ro`; ownership is less critical |
| Database data | Prefer a named volume, or match the image's user |
| Logs | Match the container UID/GID and ensure the dir is writable |
| Shared config | Root-owned plus `:ro` is fine |
| Multi-user access | Common group with group-writable perms |

## Quick Reference

```bash
# Your IDs
id -u; id -g

# Match host ownership
sudo chown -R "$(id -u):$(id -g)" ./data

# What user does the container run as?
docker compose exec app id

# Write test
docker compose exec app sh -c 'touch /app/data/.permtest' && ls -la ./data/.permtest && rm ./data/.permtest

# Persist IDs for compose
printf 'DOCKER_UID=%s\nDOCKER_GID=%s\n' "$(id -u)" "$(id -g)" > .env
```

For related material, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md), [Set the Container User (UID/GID) in Docker Compose](articles/docker-compose-set-container-user.md), and [Docker Compose: Bind Mounts vs Named Volumes](articles/docker-compose-volumes-vs-bind-mounts.md).
