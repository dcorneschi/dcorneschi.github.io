# Docker Restart Policies

A restart policy tells the Docker daemon whether and when to automatically restart a container after it exits or after the daemon itself restarts. Choosing the right policy is key to service availability, clean debugging, and realistic failure testing. This guide covers all four policies, their differences, and how to apply, inspect, and test them.

## Policy Overview

| Policy | Restarts on crash / non-zero exit | Restarts on clean exit (0) | Survives daemon/host reboot | Typical use |
|--------|-----------------------------------|----------------------------|-----------------------------|-------------|
| `no` (default) | No | No | No | One-off tasks, debugging |
| `on-failure[:N]` | Yes, up to `N` times if set | No | Only if it was still retrying | Batch jobs, transient failures |
| `always` | Yes | Yes | Yes | Long-running services |
| `unless-stopped` | Yes | Yes | Yes, unless it was manually stopped | Recommended for most services |

The practical difference between `always` and `unless-stopped` shows up around a reboot: if you manually `docker stop` a container, `always` will still start it again when the daemon restarts, whereas `unless-stopped` leaves it stopped because you stopped it deliberately.

## unless-stopped: The Common Recommendation

`unless-stopped` restarts a container on crash and after a daemon or host reboot, but respects a manual stop. That makes it a good default for services you want to stay up without overriding your own deliberate shutdowns.

```bash
# Run a service with unless-stopped
docker run -d --restart unless-stopped --name my-service nginx:1.27-alpine

# Apply the policy to an existing container (takes effect immediately)
docker update --restart unless-stopped my-service
```

`docker update --restart` changes the policy live; you do not need to recreate or restart the container for the new policy to apply.

## The Policies in Detail

### no

The default. The container is never restarted automatically. Use it for one-off commands and when debugging, so a crashing container stays down for inspection.

```bash
docker run -d --restart no --name debug-run myapp:1.0.0
```

### on-failure[:max-retries]

Restarts only when the container exits with a non-zero status. An optional retry limit caps the attempts; once exhausted, Docker stops trying.

```bash
# Retry up to 5 times on non-zero exit, then give up
docker run -d --restart on-failure:5 --name batch-job myapp:1.0.0
```

A clean exit (status 0) does not trigger a restart, which suits batch jobs that should run to completion and only retry on error.

### always

Restarts the container regardless of exit status, and starts it again after a daemon restart even if it was manually stopped (the daemon restarts it when it next starts). Use with care, since it overrides a manual stop across daemon restarts.

```bash
docker run -d --restart always --name critical-service myapp:1.0.0
```

### unless-stopped

Like `always`, but a container you stopped manually stays stopped across daemon restarts. This is usually the more intuitive choice for services.

```bash
docker run -d --restart unless-stopped --name web myapp:1.0.0
```

## Restart Policies in Docker Compose

Use the `restart` key. The values match the CLI: `no`, `on-failure`, `always`, `unless-stopped`.

```yaml
services:
  web:
    image: nginx:1.27-alpine
    restart: unless-stopped
    ports:
      - "80:80"
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:80/"]
      interval: 30s
      timeout: 10s
      retries: 3

  database:
    image: mysql:8.4
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root
    volumes:
      - db-data:/var/lib/mysql
    secrets:
      - db_root
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 30s
      timeout: 10s
      retries: 5

  cache:
    image: redis:7-alpine
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 15s
      timeout: 5s
      retries: 3

volumes:
  db-data:

secrets:
  db_root:
    file: ./db_root.txt
```

> Compose v2 ignores the obsolete top-level `version:` key; omit it. Prefer a Docker secret or an `_FILE` environment variable over putting a database password directly in the Compose file. See [Defining Variables in Docker Compose](articles/docker-compose-variables-env.md).

### Restart policy vs healthcheck

A restart policy reacts to the container **exiting**. A healthcheck marks a running container `healthy` or `unhealthy` but does **not**, on its own, restart an unhealthy container under plain Docker or Compose. To act on health, use an external supervisor, an orchestrator, or a tool such as `autoheal`. Do not assume a failing healthcheck alone will trigger `restart:`.

## Inspecting the Restart Policy

There is no `docker ps` format placeholder for the restart policy, so use `docker inspect`.

```bash
# Restart policy of a container
docker inspect --format '{{ .HostConfig.RestartPolicy.Name }}' my-service

# Policy name plus max retries
docker inspect --format \
  'policy={{ .HostConfig.RestartPolicy.Name }} max={{ .HostConfig.RestartPolicy.MaximumRetryCount }}' \
  my-service

# How many times Docker has restarted it
docker inspect --format '{{ .RestartCount }}' my-service

# Current state and last exit code
docker inspect --format '{{ .State.Status }} exit={{ .State.ExitCode }}' my-service
```

With `jq`:

```bash
docker inspect my-service | jq '.[0].HostConfig.RestartPolicy, .[0].RestartCount, .[0].State.ExitCode'
```

## Updating and Removing a Policy

```bash
# Set or change the policy on running containers (applies immediately)
docker update --restart unless-stopped container1 container2

# Disable automatic restarts
docker update --restart no my-service
```

`docker stop` prevents further automatic restarts until you start the container again; `docker start` re-arms the policy.

```bash
docker stop my-service    # will not auto-restart while stopped
docker start my-service   # policy active again
```

## Testing Restart Behavior

`docker kill` simulates a crash, so a container with a restart policy should come back on its own.

```bash
docker run -d --restart unless-stopped --name web-server -p 8080:80 nginx:1.27-alpine

# Simulate a crash
docker kill web-server
sleep 5

# Confirm it came back and count restarts
docker ps --filter "name=web-server" --format 'table {{.Names}}\t{{.Status}}'
docker inspect --format '{{ .RestartCount }}' web-server
```

A simple pass/fail test script:

```bash
#!/usr/bin/env bash
set -euo pipefail

name="${1:-restart-test}"

docker run -d --restart unless-stopped --name "$name" -p 8080:80 nginx:1.27-alpine
sleep 5

docker kill "$name"
sleep 8

if docker ps --filter "name=$name" --filter "status=running" --format '{{.Names}}' | grep -qx "$name"; then
  echo "PASS: $name restarted automatically (RestartCount=$(docker inspect --format '{{ .RestartCount }}' "$name"))"
else
  echo "FAIL: $name did not restart"
fi

docker rm -f "$name"
```

### Chaos-style loop for a group

Label the containers you want to target, then kill them at random and confirm recovery:

```bash
#!/usr/bin/env bash
set -euo pipefail

mapfile -t containers < <(docker ps --filter "label=tier=app" --format '{{.Names}}')
if [ "${#containers[@]}" -eq 0 ]; then
  echo "No containers with label tier=app"; exit 1
fi

echo "Targeting: ${containers[*]} (Ctrl+C to stop)"
while true; do
  target="${containers[RANDOM % ${#containers[@]}]}"
  echo "$(date +%T) killing $target"
  docker kill "$target" >/dev/null
  sleep 15
  if docker ps --filter "name=$target" --filter "status=running" --format '{{.Names}}' | grep -qx "$target"; then
    echo "$(date +%T) $target recovered"
  else
    echo "$(date +%T) $target did NOT recover"
  fi
  sleep $(( 30 + RANDOM % 60 ))
done
```

Start the target services with a matching label and `unless-stopped`:

```bash
docker run -d --name app1 --label tier=app --restart unless-stopped nginx:1.27-alpine
docker run -d --name app2 --label tier=app --restart unless-stopped nginx:1.27-alpine
docker run -d --name app3 --label tier=app --restart unless-stopped nginx:1.27-alpine
```

## Monitoring Restarts

```bash
# Watch status changes
watch -n 2 'docker ps --format "table {{.Names}}\t{{.Status}}"'

# Stream restart and die/start events for a container
docker events --filter container=my-service --filter event=restart --filter event=die

# Follow logs across restarts
docker logs -f my-service
```

A climbing `RestartCount` combined with a crash loop usually points to a failing command or a config error inside the container; check `docker logs` and the last `State.ExitCode`.

## When a Container Will Not Restart

- It was stopped manually with `docker stop` (expected with `unless-stopped`).
- It exited cleanly (status 0) under `no` or `on-failure`.
- `on-failure:N` exhausted its retry budget.
- The Docker daemon is not running; containers restart only once the daemon is back.
- Docker applies an increasing delay between rapid restarts, so recovery from a tight crash loop is not instantaneous.

Diagnose with:

```bash
docker inspect --format '{{ .State.Status }} exit={{ .State.ExitCode }} restarts={{ .RestartCount }}' my-service
docker logs --tail 50 my-service
```

## Choosing a Policy

| Scenario | Recommended policy |
|----------|--------------------|
| Production or long-running service | `unless-stopped` |
| Service that must return even after a deliberate stop across reboots | `always` |
| Batch or one-shot job that should retry only on error | `on-failure:N` |
| Debugging a crash | `no` |
| Failure/chaos testing of recovery | `unless-stopped` |

## Quick Reference

```bash
# Run with a policy
docker run -d --restart unless-stopped --name web nginx:1.27-alpine
docker run -d --restart on-failure:5 --name job myapp:1.0.0

# Change or remove a policy (immediate)
docker update --restart unless-stopped web
docker update --restart no web

# Inspect the policy and restart count
docker inspect --format '{{ .HostConfig.RestartPolicy.Name }}' web
docker inspect --format '{{ .RestartCount }}' web

# Test recovery
docker kill web && sleep 5 && docker ps --filter name=web

# Stop restarts / resume
docker stop web
docker start web
```

For related material, see the [Docker Cheatsheet](articles/docker-cheatsheet.md), [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md), and [Defining Variables in Docker Compose](articles/docker-compose-variables-env.md).
