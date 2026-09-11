# Updating Docker Compose Containers

Updating a Compose stack means pulling newer images (or rebuilding), then recreating the affected containers. This guide covers the core commands, updating single or multiple projects, automated updates, a safe update workflow with backups and verification, and rollback.

The examples use Compose v2 (`docker compose`, no hyphen). If you still use the legacy v1 binary, substitute `docker-compose`. For the full command reference, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md).

> **Downtime note:** `docker compose up -d` recreates changed containers, which causes a brief interruption for those services. It is not zero-downtime on its own. True rolling updates need a scale-up/scale-down dance (below) or an orchestrator such as Swarm or Kubernetes.

## Core Update Commands

```bash
# Pull newer images, then recreate changed containers
docker compose pull && docker compose up -d

# Rebuild locally built images, then recreate
docker compose up -d --build

# Force recreate every container, even if unchanged
docker compose up -d --force-recreate

# Remove containers for services deleted from the file
docker compose up -d --remove-orphans
```

Compose only recreates containers whose image or configuration changed, so `pull && up -d` leaves unchanged services running.

## Update Specific Services

```bash
# Pull and update selected services
docker compose pull web api
docker compose up -d web api

# Update one service without touching its dependencies
docker compose pull web
docker compose up -d --no-deps web
```

`--no-deps` updates a single service without restarting its dependencies. This still recreates that service's container (a brief gap), so it reduces blast radius but is not itself zero-downtime.

## Update Multiple Projects

Each Compose project lives in its own directory. Iterate over them with a small script rather than `cd`-ing by hand. Use the `-f`/`--project-directory` flags or `cd` in a subshell so a failure in one does not strand you in the wrong directory.

```bash
#!/usr/bin/env bash
set -uo pipefail

projects=(
  /srv/stacks/webapp
  /srv/stacks/monitoring
  /srv/stacks/api
)

for dir in "${projects[@]}"; do
  echo "== Updating $dir =="
  if [ ! -f "$dir/docker-compose.yml" ] && [ ! -f "$dir/compose.yaml" ]; then
    echo "  no compose file, skipping"
    continue
  fi
  ( cd "$dir" && docker compose pull && docker compose up -d ) \
    && echo "  ok" || echo "  FAILED"
done
```

Running the pull/up inside `( … )` keeps each project's working directory isolated, and `set -uo pipefail` surfaces errors without aborting the whole loop on the first failure.

## Profiles

If your stack uses profiles, enable them for the update so profiled services are included:

```bash
docker compose --profile production pull
docker compose --profile production up -d

# Multiple profiles
docker compose --profile web --profile database up -d
```

There is no wildcard that reliably enables "all profiles"; enable the specific profiles you use. You can also set `COMPOSE_PROFILES=web,database` in the environment instead of repeating `--profile`.

## Automated Updates with Watchtower

Watchtower can watch running containers and update them when their image tag changes. It is convenient for homelabs but has real trade-offs worth understanding.

```yaml
services:
  watchtower:
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --schedule "0 0 4 * * *" --cleanup    # daily at 04:00
    restart: unless-stopped
```

Cautions:

- **It fights version pinning.** Watchtower re-pulls the tag a container uses. If you pin `myapp:1.4.2`, that tag should not move, so Watchtower has nothing to update; it is most "effective" against moving tags like `latest`, which undermines reproducibility and safe rollback. See [Pin Docker Image Versions Instead of latest](articles/docker-image-version-pinning.md).
- **Unattended updates can break things.** A new image can introduce a regression at 4 a.m. with no one watching. Use notifications and consider updating only non-critical services.
- **Mounting the Docker socket is a privilege.** Anything with `/var/run/docker.sock` is effectively root on the host.

Limit Watchtower to opt-in containers with labels:

```yaml
services:
  app:
    image: myapp:1.4.2
    labels:
      - "com.centurylinklabs.watchtower.enable=true"

  database:
    image: postgres:16.4
    labels:
      - "com.centurylinklabs.watchtower.enable=false"

  watchtower:
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --label-enable --schedule "0 0 4 * * *" --cleanup
    restart: unless-stopped
```

For notify-only workflows (no automatic changes), a tool like Diun watches for new images and alerts without pulling.

## A Safe Update Workflow

For anything with state, wrap the update in back up → update → verify, with rollback ready.

### 1. Back up state first

Databases must be dumped, not file-copied while running:

```bash
# PostgreSQL
docker compose exec -T db pg_dump -U user myapp > "backup-$(date +%F).sql"

# MySQL / MariaDB
docker compose exec -T db mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" --all-databases > "backup-$(date +%F).sql"
```

For named volumes, archive them while the service is stopped for consistency:

```bash
docker compose stop app
docker run --rm -v myproject_app_data:/data -v "$PWD:/backup" alpine \
  tar czf "/backup/app_data-$(date +%F).tar.gz" -C /data .
docker compose start app
```

### 2. Pin the new version and pull

Update the tag in the Compose file (or `.env`) to the new version, then:

```bash
docker compose config --quiet    # validate before applying
docker compose pull
```

### 3. Apply and verify

```bash
docker compose up -d
docker compose ps
docker compose logs --tail 50 app
```

Check health and endpoints:

```bash
# Wait for services with healthchecks to report healthy
docker compose up -d --wait

# Application endpoint
curl -fsS http://localhost:8080/health

# Dependencies
docker compose exec -T db pg_isready -U user
docker compose exec -T cache redis-cli ping
```

Use `docker compose up --wait` when your services define health checks; it blocks until they are healthy or a dependency fails. See [Docker Compose depends_on: Startup Order and Readiness](articles/docker-compose-depends-on.md).

## Reduced-Downtime Update for a Stateless Service

If a service is stateless and behind a load balancer or reverse proxy, you can overlap old and new containers by scaling up before scaling down:

```bash
docker compose pull web
docker compose up -d --no-deps --scale web=2 --no-recreate web
# wait for the new container to pass health checks
docker compose up -d --no-deps --scale web=1 web
```

This is a pragmatic single-host approximation of a rolling update; a real orchestrator does it more robustly.

## Rollback

Because you pinned versions, rollback is changing the tag back and reapplying:

```bash
# Edit the Compose file / .env back to the previous version, then:
docker compose up -d app

# Confirm the running image
docker inspect --format '{{ .Config.Image }}' "$(docker compose ps -q app)"
```

Restore data only if the new version migrated it in an incompatible way, using the backup from step 1. Avoid `docker compose down` for a single-service rollback; it stops the whole stack.

## Cleanup After Updating

Old images accumulate after updates. Prune deliberately:

```bash
# Remove dangling images
docker image prune -f

# Remove all unused images (more aggressive)
docker image prune -a -f
```

Do not casually add `--volumes` to prune commands; that deletes unused named volumes and their data. Review first with `docker volume ls -f dangling=true`.

## Scheduling

A systemd timer is cleaner than cron for logging and dependencies:

```ini
# /etc/systemd/system/compose-update@.service
[Unit]
Description=Update Docker Compose project %i
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
WorkingDirectory=/srv/stacks/%i
ExecStart=/usr/bin/docker compose pull
ExecStart=/usr/bin/docker compose up -d
```

```ini
# /etc/systemd/system/compose-update@.timer
[Unit]
Description=Weekly update for Docker Compose project %i

[Timer]
OnCalendar=Sun *-*-* 04:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now compose-update@webapp.timer
```

For unattended scheduling, prefer moving explicit version pins forward in git and letting the timer apply them, over auto-pulling moving tags.

## Method Comparison

| Method | Best for | Downtime | Complexity |
|--------|----------|----------|------------|
| `pull && up -d` | Single stack | Brief per changed service | Low |
| Per-service `--no-deps` | Limiting blast radius | Brief for that service | Low |
| Batch script | Many projects | Brief per project | Medium |
| Scale up/down | Stateless service on one host | Minimal | Medium |
| Watchtower | Homelab, opt-in services | Brief | Medium |
| Swarm / Kubernetes | Clusters | Rolling, near-zero | High |

## Quick Reference

```bash
# Standard update
docker compose pull && docker compose up -d

# Single service, limited blast radius
docker compose pull web && docker compose up -d --no-deps web

# Validate, apply, wait for health
docker compose config --quiet && docker compose up -d --wait

# Back up a database before updating
docker compose exec -T db pg_dump -U user myapp > "backup-$(date +%F).sql"

# Roll back (after editing the tag) and confirm
docker compose up -d app
docker inspect --format '{{ .Config.Image }}' "$(docker compose ps -q app)"

# Clean up old images
docker image prune -f
```

For related material, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md), [Pin Docker Image Versions Instead of latest](articles/docker-image-version-pinning.md), [Docker Compose depends_on: Startup Order and Readiness](articles/docker-compose-depends-on.md), and [Docker Restart Policies](articles/docker-restart-policies.md).
