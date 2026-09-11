# Where to Store Docker Compose Bind-Mount Data on the Host

When a Docker Compose stack uses bind mounts, you choose where that data lives on the host. A consistent, predictable layout under a single parent directory makes backups, permissions, and migration far easier. `/opt/docker` is a popular and reasonable choice, and this guide covers how to structure it, along with the trade-offs versus other locations and versus named volumes.

This applies to **bind mounts** (host paths). For Docker-managed **named volumes**, the storage location is controlled separately by the daemon's data root; see [Move the Docker Data Directory (data-root)](articles/docker-move-data-root.md). For the difference between the two, see [Docker Compose: Bind Mounts vs Named Volumes](articles/docker-compose-volumes-vs-bind-mounts.md).

## Is /opt a Good Choice?

Yes. A dedicated tree such as `/opt/docker` works well for self-hosted stacks:

- It is a single, predictable location for all containerized application data.
- It keeps application data separate from OS files.
- It is straightforward to back up as one tree or per service.
- It gives each service a clear, consistent structure.
- Ownership and permissions are easy to reason about in one place.

### A note on the Filesystem Hierarchy Standard

Under the [Filesystem Hierarchy Standard](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html):

- `/opt` is intended for add-on application software packages.
- `/srv` is intended for data served by the system (web content, application data).
- `/var/lib` is for variable state maintained by applications.

By a strict reading, `/srv` is the most technically correct home for service data you host. In practice, `/opt/docker` and `/srv/docker` are both widely used and perfectly workable. Pick one convention and apply it consistently; consistency matters more than the exact top-level directory.

## Recommended Directory Structure

Group by service, and give each service predictable subdirectories:

```text
/opt/docker/
├── plex/
│   ├── config/
│   ├── transcode/
│   └── media/          # or a symlink/mount to bulk media storage
├── nextcloud/
│   ├── data/
│   ├── config/
│   └── database/
├── nginx-proxy/
│   ├── conf.d/
│   ├── certs/
│   └── logs/
└── monitoring/
    ├── prometheus/
    ├── grafana/
    └── alertmanager/
```

Keeping `config`, `data`, and `logs` separate per service makes it obvious what to back up, what to exclude, and what to reset.

## Example Compose File

```yaml
services:
  app:
    image: myapp:1.4.2
    volumes:
      - /opt/docker/myapp/data:/app/data
      - /opt/docker/myapp/config:/app/config:ro
      - /opt/docker/myapp/logs:/app/logs
```

Mounting configuration read-only (`:ro`) is a small, worthwhile safeguard for files the container should not modify. Pin the image to a version rather than `latest`; see [Pin Docker Image Versions Instead of latest](articles/docker-image-version-pinning.md).

> The top-level `version:` key is obsolete in current Docker Compose and only produces a warning. Omit it.

## Alternative Locations

| Location | Rationale | Notes |
|----------|-----------|-------|
| `/opt/docker` | Add-on software convention; widely used | Fine for self-hosted stacks |
| `/srv/docker` | FHS location for served data | Arguably the most correct choice |
| `/var/lib/<stack>` | Variable application state | Avoid nesting inside `/var/lib/docker`, which the daemon manages |
| `/home/docker` | Dedicated non-login user home | Workable, but `/home` is sometimes mounted with restrictions such as `nosuid` |

Do not place bind-mount data **inside** `/var/lib/docker`; that path is owned by the Docker daemon and mixing hand-managed files there can cause conflicts.

## Setting Up the Layout

### Create the structure

```bash
sudo mkdir -p /opt/docker/{plex,nextcloud,nginx-proxy}/{config,data,logs}
```

### Set ownership to match the container user

Ownership must match the UID/GID the container runs as, not just your login user. Many images run as a specific non-root UID.

```bash
# Example: a service that runs as UID/GID 1000
sudo chown -R 1000:1000 /opt/docker/myapp
```

For mapping the container user to host file ownership, see [Set the Container User (UID/GID) in Docker Compose](articles/docker-compose-set-container-user.md).

### Set sensible permissions

Blanket `755`/`644` is fine for non-sensitive config and data, but it is **wrong for secrets**: `644` and `755` are world-readable. Tighten anything sensitive.

```bash
# General app data and config (not secret)
sudo find /opt/docker/myapp -type d -exec chmod 755 {} \;
sudo find /opt/docker/myapp -type f -exec chmod 644 {} \;

# Secrets: restrict to owner only
sudo chmod 700 /opt/docker/myapp/secrets
sudo chmod 600 /opt/docker/myapp/secrets/*
```

## Backups

A single parent directory makes backups simple, but back up **consistent** data.

```bash
# Snapshot the whole tree
sudo rsync -aAX --delete /opt/docker/ /backup/opt-docker/

# Or per service
sudo rsync -aAX /opt/docker/nextcloud/ /backup/nextcloud/
```

Important caveats:

- Databases should be dumped or quiesced, not copied live. Copying a running database's files can yield a corrupt backup. Use the database's own dump tool, or stop the service with `docker compose stop` before copying.
- `-aAX` preserves permissions, ACLs, and extended attributes, which matters for correct ownership on restore.
- Exclude large regenerable data (caches, transcode scratch) to keep backups lean.

## When to Prefer Named Volumes Instead

Host bind mounts are excellent for config you edit and data you want to see directly. Named volumes are often better for:

- Databases, where Docker-managed storage avoids host UID/permission friction.
- Portable stacks that should not depend on a specific host path.
- Data you never need to edit directly from the host.

Many real stacks mix both: bind mounts for configuration, named volumes for database state. See the comparison in [Docker Compose: Bind Mounts vs Named Volumes](articles/docker-compose-volumes-vs-bind-mounts.md).

## Putting the Tree on Its Own Storage

For growth and isolation, back the directory with a dedicated disk or logical volume so container data cannot fill the root filesystem. Mount it at the parent path and persist it in `/etc/fstab`.

```bash
sudo mount /dev/vg_data/lv_docker /opt/docker
echo "/dev/vg_data/lv_docker /opt/docker xfs defaults 0 2" | sudo tee -a /etc/fstab
```

See the [LVM Cheatsheet](articles/lvm-cheatsheet.md) and the [/etc/fstab Guide](articles/linux-fstab-guide.md).

## SELinux Note

On RHEL, Rocky Linux, AlmaLinux, and Fedora, bind mounts under a custom path may need a relabel or the `:z`/`:Z` mount option so containers can access them.

```yaml
services:
  app:
    image: myapp:1.4.2
    volumes:
      - /opt/docker/myapp/data:/app/data:Z
```

Use `:z` for data shared between containers and `:Z` for data private to one container. Apply these carefully, since `:Z` relabels everything under the path.

## Running as a Specific Host User

When you run a stack as a login user (for example `dcorneschi`) and mount data under `/opt`, permission errors usually mean the host directory ownership does not match the UID the container writes as. There are three ways to line them up.

### Match directory ownership to the user (recommended)

```bash
sudo chown -R dcorneschi:dcorneschi /opt/docker/myapp
```

### Run the container as that user's UID/GID

```yaml
services:
  your-service:
    user: "${DOCKER_UID:-1000}:${DOCKER_GID:-1000}"
    volumes:
      - /opt/docker/myapp/data:/app/data
```

```bash
export DOCKER_UID=$(id -u dcorneschi) DOCKER_GID=$(id -g dcorneschi)
docker compose up -d
```

Use your own variable names rather than `UID`/`GID`: `UID` is read-only in bash and `GID` is often unset, so `export UID=...` fails and `${UID}` may not resolve as intended. See [Set the Container User (UID/GID) in Docker Compose](articles/docker-compose-set-container-user.md).

### Share access through a group

If changing ownership outright is not feasible, grant a group instead and make the tree group-writable:

```bash
sudo chgrp -R dcorneschi /opt/docker/myapp
sudo chmod -R 2775 /opt/docker/myapp     # 2xxx sets setgid so new files inherit the group
```

Reserve group-writable (`775`) for data shared between a user and a service; keep secrets at `600`.

### The docker group

Adding a user to the `docker` group lets it run Docker without `sudo`:

```bash
sudo usermod -aG docker dcorneschi
# Log out and back in for the new group to take effect
```

> **Security warning:** membership in the `docker` group is effectively root on the host. Any member can start a container that mounts the whole filesystem and gain full control. Grant it deliberately, and prefer rootless Docker or Podman where that risk is unacceptable.

## macOS and Docker Desktop

On macOS, Docker Engine runs inside a Linux VM, so host bind mounts are shared into that VM rather than used directly. Two consequences matter for a `/opt` layout:

- The host path must be within Docker Desktop's shared paths. Add it under **Settings → Resources → File Sharing** if it is not already covered.
- Linux `chown`/`chmod` on the macOS side do not map cleanly into the VM. File ownership and permission behavior differ from a native Linux host, and `/opt` is not a conventional data location on macOS.

For portable stacks on macOS, named volumes avoid host-path sharing and UID-mapping quirks entirely. A common pattern is to develop on macOS with named volumes and use the `/opt` bind-mount layout on Linux servers.

## Best Practices Summary

- Choose one parent directory (`/opt/docker` or `/srv/docker`) and use it everywhere.
- Group by service, with consistent `config`/`data`/`logs` subdirectories.
- Set ownership to the container's UID/GID, not just your user.
- Use `755`/`644` for ordinary files but `700`/`600` for secrets.
- Dump databases rather than copying live files.
- Mount configuration read-only where possible.
- Consider named volumes for databases and portable stacks.
- Document the layout in a README or Compose comments.

## Quick Reference

```bash
# Create a per-service structure
sudo mkdir -p /opt/docker/myapp/{config,data,logs}

# Ownership to the container UID/GID (example 1000:1000)
sudo chown -R 1000:1000 /opt/docker/myapp

# Permissions: general vs secret
sudo chmod 755 /opt/docker/myapp/config
sudo chmod 600 /opt/docker/myapp/secrets/*

# Back up the tree (dump databases separately)
sudo rsync -aAX --delete /opt/docker/ /backup/opt-docker/
```

```yaml
services:
  app:
    image: myapp:1.4.2
    volumes:
      - /opt/docker/myapp/data:/app/data
      - /opt/docker/myapp/config:/app/config:ro
      - /opt/docker/myapp/logs:/app/logs
```

For related material, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md) and [Docker Compose: Bind Mounts vs Named Volumes](articles/docker-compose-volumes-vs-bind-mounts.md).
