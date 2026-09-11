# Docker Compose: Bind Mounts vs Named Volumes

In a Docker Compose `volumes:` entry, the same field can mean two different things. Whether you get a **bind mount** or a **named volume** depends entirely on how the source is written. Getting this wrong leads to surprises like empty directories, permission errors, or data that vanishes on recreate.

This guide explains the short-syntax rule, when to use each type, the long syntax, and the common pitfalls.

## The Quick Rule

A short-syntax service volume looks like `SOURCE:TARGET[:OPTIONS]`. The `SOURCE` decides the type:

> **If `SOURCE` starts with `/`, `./`, or `~/`, it is a bind mount. Otherwise it is a named volume.**

```yaml
services:
  app:
    volumes:
      - /host/path:/container/path        # bind mount (absolute)
      - ./relative/path:/container/path   # bind mount (relative to the compose file)
      - ~/home/path:/container/path       # bind mount (user home)
      - mydata:/container/path            # named volume
      - db_data:/var/lib/mysql            # named volume
```

Anything that looks like a filesystem path is a bind mount. A bare name that matches `[a-zA-Z0-9][a-zA-Z0-9_.-]*` is treated as a named volume.

## Bind Mounts

A bind mount maps a specific host directory or file into the container. The container sees exactly what is on the host at that path, and writes go straight back to the host.

```yaml
services:
  web:
    image: nginx:1.27.1
    volumes:
      - ./site:/usr/share/nginx/html:ro
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

Characteristics:

- The host path is managed by you, not by Docker.
- Relative paths resolve against the directory containing the Compose file, not the container's working directory.
- If the host path does not exist, Docker typically creates an empty directory for it, which is a common cause of "my config disappeared" confusion.
- Bind-mounting over a container directory hides whatever the image shipped at that path for the life of the mount.
- Host ownership and permissions apply inside the container, which can cause permission errors when UIDs differ.

Bind mounts are best for local development (live-editing source), injecting configuration files, and sharing host data you manage directly.

## Named Volumes

A named volume is storage managed by Docker. Its data lives in the Docker area (by default under the data root) and persists independently of any container.

```yaml
services:
  db:
    image: postgres:16.4
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

For named volumes, you must also declare them under the top-level `volumes:` key. A minimal declaration with no options is valid:

```yaml
services:
  app:
    volumes:
      - mydata:/data

volumes:
  mydata:
```

Characteristics:

- Docker creates and manages the storage; you refer to it by name.
- Data survives `docker compose down` and container recreation. It is removed only when you delete the volume (for example, `docker compose down -v` or `docker volume rm`).
- When a named volume is empty and first mounted, the content already present at the target path in the image is copied into the volume. Bind mounts do not do this.
- Compose namespaces volume names with the project name, so the actual volume may appear as `projectname_mydata`.

Named volumes are best for databases and other persistent application state, where Docker-managed storage and portability matter more than direct host access.

## Compose Namespacing and External Volumes

Because Compose prefixes volume names with the project, reference an existing volume created elsewhere with `external: true` to avoid the prefix and prevent accidental creation:

```yaml
services:
  app:
    volumes:
      - shared_data:/data

volumes:
  shared_data:
    external: true
```

With `external: true`, Compose expects the volume to already exist and will not create or remove it.

## Long Syntax

The long syntax is explicit about the mount type and supports options the short form cannot express. It removes any ambiguity about bind vs volume.

```yaml
services:
  app:
    volumes:
      - type: bind
        source: ./config
        target: /etc/app
        read_only: true

      - type: volume
        source: app_data
        target: /var/lib/app

      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 67108864   # 64 MB, in bytes

volumes:
  app_data:
```

Notes:

- `type: bind` requires the source path to exist by default; set `bind.create_host_path: true` to allow creation.
- `type: volume` uses a named volume; `source` is omitted for an anonymous volume.
- `type: tmpfs` stores data in memory only and is gone when the container stops.

## Anonymous Volumes

If you give only a target with no source, Docker creates an anonymous volume with a random ID:

```yaml
services:
  app:
    volumes:
      - /var/lib/app   # anonymous volume
```

Anonymous volumes are easy to leak because they accumulate unnamed and are hard to identify later. Prefer a named volume so you can manage its lifecycle deliberately.

## Read-Only and SELinux Options

Append options after the target in short syntax:

```yaml
services:
  app:
    volumes:
      - ./config:/etc/app:ro     # read-only
      - ./data:/data:rw          # read-write (default)
      - ./data:/data:z           # shared SELinux relabel
      - ./secret:/secret:Z       # private SELinux relabel
```

On SELinux-enforcing hosts (RHEL, Rocky Linux, AlmaLinux, Fedora), a bind mount without `:z` or `:Z` often causes permission-denied errors. Use `:z` for content shared between containers and `:Z` for content private to one container. Apply these relabel options with care, since `:Z` on a broad host path relabels everything under it.

## Choosing Between Them

| Need | Use |
|------|-----|
| Live-edit source during development | Bind mount |
| Inject a specific host config file | Bind mount (often `:ro`) |
| Persist database or app state | Named volume |
| Portable storage managed by Docker | Named volume |
| Fast, ephemeral scratch space | `tmpfs` |
| Share pre-existing storage across projects | External named volume |

A useful default: bind mounts for configuration and development, named volumes for persistent data.

## Common Pitfalls

- A misspelled bare source name silently becomes a new empty named volume instead of an error.
- Bind-mounting a nonexistent host path creates an empty directory and can mask image content.
- Forgetting the top-level `volumes:` declaration for a named volume causes a validation error.
- Expecting a bind mount to seed from image content; only empty named volumes copy existing target content.
- Assuming `docker compose down` deletes data; named volumes persist unless you add `-v`.
- Relative bind paths resolve from the Compose file location, which matters when running Compose from another directory.

## Inspecting Volumes and Mounts

```bash
# List Docker-managed volumes
docker volume ls

# Inspect a volume's mountpoint and driver
docker volume inspect projectname_db_data

# See what a running container actually mounted
docker inspect --format '{{ json .Mounts }}' <container> | jq

# List volumes for the current compose project
docker compose config --volumes

# Remove volumes for the project (destroys data)
docker compose down -v
```

For broader command coverage, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md) and the [Docker Cheatsheet](articles/docker-cheatsheet.md). For host-side bind mounts in `/etc/fstab`, see the [/etc/fstab Guide](articles/linux-fstab-guide.md).

## Quick Reference

```yaml
services:
  app:
    volumes:
      # Bind mounts — start with / ./ or ~/
      - /abs/host/path:/in/container
      - ./relative/path:/in/container
      - ~/home/path:/in/container
      - ./config:/etc/app:ro

      # Named volumes — bare names
      - app_data:/var/lib/app

      # Long syntax
      - type: volume
        source: app_data
        target: /var/lib/app

# Named volumes must be declared here
volumes:
  app_data:
```
