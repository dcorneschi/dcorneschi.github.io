# Docker Healthcheck Examples

A container healthcheck is a command Docker runs periodically inside the container to decide whether the application is actually working, not just whether the process started. This guide gives practical `healthcheck` examples for HTTP, TCP, and service-specific checks, plus the common pitfalls that make checks silently fail.

Examples use Compose v2 syntax (no top-level `version:` key). For how a healthcheck relates to startup ordering and restarts, see [Docker Compose depends_on: Startup Order and Readiness](articles/docker-compose-depends-on.md) and [Docker Restart Policies](articles/docker-restart-policies.md).

## Healthcheck Fields

| Field | Meaning | Default |
|-------|---------|---------|
| `test` | The command to run; exit `0` = healthy, `1` = unhealthy | — |
| `interval` | Time between checks | `30s` |
| `timeout` | Max time for one check before it counts as a failure | `30s` |
| `retries` | Consecutive failures before the container is `unhealthy` | `3` |
| `start_period` | Grace window where failures don't count toward `retries` | `0s` |
| `start_interval` | Check frequency during `start_period` (Engine 25+) | — |

The health state appears in `docker ps` and `docker inspect`. A healthcheck marks state only; it does **not** restart the container by itself.

## CMD vs CMD-SHELL, and the Tooling Trap

`test` accepts two forms:

- `["CMD", "curl", "-f", "http://localhost/"]` — exec form, runs the binary directly (no shell).
- `["CMD-SHELL", "curl -f http://localhost/ || exit 1"]` — runs through `/bin/sh -c`, so you can use pipes, `&&`, `||`, and variables.

> **The most common mistake:** assuming `curl` exists in the image. It usually does **not**. `nginx:alpine`, `node:alpine`, and most Alpine-based images ship BusyBox `wget` but **no `curl`**. A `CMD ["curl", ...]` healthcheck on those images fails every time with "executable not found", marking the container unhealthy regardless of the app. Either use `wget`, or install `curl` in your image, or use a check the image already supports.

## HTTP Checks

### With wget (works on Alpine/BusyBox images)

`--spider` makes a request without saving the body — ideal for a liveness ping.

```yaml
services:
  web:
    image: nginx:1.27-alpine
    ports:
      - "8080:80"
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:80/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
```

### With curl (only where curl is installed)

Use `-f` so an HTTP error status (>= 400) makes curl exit non-zero.

```yaml
services:
  api:
    image: myapi:1.0.0          # an image that includes curl
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 60s
```

If you are unsure whether curl exists, prefer `wget`, or gate on it:

```yaml
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost:3000/health || wget -q --spider http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### HTTPS with a self-signed certificate

```yaml
    # curl: -k skips cert verification
    test: ["CMD", "curl", "-fk", "https://localhost:443/"]
```

```yaml
    # wget equivalent
    test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "--no-check-certificate", "https://localhost:443/"]
```

Skipping verification is acceptable for a **local loopback** healthcheck to the container's own port; do not use `-k`/`--no-check-certificate` for real external calls.

### Custom headers, methods, and auth

Because these use multiple arguments, keep them in exec form or move to `CMD-SHELL`:

```yaml
    # Header
    test: ["CMD", "curl", "-f", "-H", "Accept: application/json", "http://localhost:3000/api/health"]
```

```yaml
    # Bearer token from an environment variable (needs a shell for expansion)
    test: ["CMD-SHELL", "curl -fsS -H \"Authorization: Bearer $TOKEN\" http://localhost:3000/health"]
```

Prefer a dedicated unauthenticated `/health` endpoint over embedding credentials in the check. A `POST` healthcheck is possible but unusual; a lightweight `GET` liveness endpoint is the norm.

## TCP and Port Checks

A port check confirms something is listening, but not that the application is truly ready — a database can accept a TCP connection before it accepts queries. Prefer a real readiness command (like `pg_isready`) when the image provides one.

```yaml
services:
  cache:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]        # better than a bare port check
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 10s
```

If you only have `nc`, note its options vary by build and `nc` is not always present:

```yaml
    # TCP port open?  -z = scan without sending data
    test: ["CMD", "nc", "-z", "localhost", "5432"]
```

```yaml
    # Multiple ports (needs a shell)
    test: ["CMD-SHELL", "nc -z localhost 3000 && nc -z localhost 3001"]
```

```yaml
    # With a connect timeout
    test: ["CMD", "nc", "-w", "3", "-z", "localhost", "8080"]
```

A UDP "check" with `nc -uz` is unreliable: UDP is connectionless, so `nc` often reports success even when nothing is listening. Avoid depending on it.

Checking a **remote** service from within a healthcheck (for example `nc -z db 5432`) is discouraged. Each container should report its own health; use `depends_on` with `condition: service_healthy` to express cross-service readiness instead.

## Service-Specific Checks

Use the tool each image already provides — it is the most accurate readiness signal and avoids the curl/nc availability problem.

```yaml
services:
  postgres:
    image: postgres:16.4
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  mysql:
    image: mysql:8.4
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongodb:
    image: mongo:7.0
    # mongosh replaced the legacy "mongo" shell in MongoDB 5+
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping')"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  rabbitmq:
    image: rabbitmq:3-management-alpine
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  elasticsearch:
    image: elasticsearch:8.14.3
    environment:
      - discovery.type=single-node
    healthcheck:
      # ES 8 enables security/HTTPS by default; adjust scheme/auth to your config
      test: ["CMD-SHELL", "curl -fsS http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s
```

Two corrections worth noting: use `mongosh` (not the removed `mongo` shell) on modern MongoDB, and remember Elasticsearch 8 turns on security and HTTPS by default, so a plain `http://localhost:9200` check may need the correct scheme and credentials.

## Composite and Custom Checks

Combine conditions with a shell, but keep the command fast and cheap since it runs on every interval.

```yaml
    # Port up AND app healthy AND a readiness file exists
    test: ["CMD-SHELL", "nc -z localhost 3000 && wget -q --spider http://localhost:3000/health && test -f /app/ready"]
```

For anything non-trivial, ship a script in the image and call it:

```dockerfile
# Dockerfile
COPY healthcheck.sh /usr/local/bin/healthcheck.sh
RUN chmod +x /usr/local/bin/healthcheck.sh
HEALTHCHECK --interval=30s --timeout=10s --retries=3 --start-period=45s \
  CMD /usr/local/bin/healthcheck.sh
```

Baking the check into the image with `HEALTHCHECK` keeps it version-controlled and available even outside Compose. A Compose `healthcheck:` overrides the image's `HEALTHCHECK`.

Prefer baking the script into the image over bind-mounting it (`volumes: - ./healthcheck.sh:/healthcheck.sh`), since a bind mount is not present when the image runs elsewhere and can carry the wrong permissions or line endings.

## Disabling a Healthcheck

To turn off a healthcheck inherited from the image:

```yaml
    healthcheck:
      test: ["NONE"]
```

## Inspecting Health

```bash
# Health state in the status column
docker compose ps
docker ps

# Current health status
docker inspect --format '{{ .State.Health.Status }}' "$(docker compose ps -q web)"

# Recent check results (exit codes and output) — invaluable for debugging
docker inspect --format '{{ json .State.Health }}' "$(docker compose ps -q web)"

# Wait for all healthchecked services to become healthy
docker compose up -d --wait
```

If a check is failing, `docker inspect` health `Log` shows the command's exit code and output — usually revealing "executable not found" (missing tool) or a real application error.

## Best Practices

- Use a tool the image actually has: `wget` on Alpine/BusyBox, `curl` only where installed, or the service's own client (`pg_isready`, `redis-cli`, `mysqladmin`, `rabbitmq-diagnostics`, `mongosh`).
- Point at a real `/health` endpoint that exercises the app, not just `/`.
- Use `start_period` for slow-starting services so early failures don't flip it to unhealthy.
- Keep the check lightweight; it runs on every `interval`.
- Have each container report only its **own** health; express cross-service readiness with `depends_on`.
- Remember a healthcheck does not restart anything — pair it with a restart policy or orchestrator.

## Why Healthchecks Matter

A container being "up" only means its main process started, not that the application is usable. Healthchecks close that gap, and several parts of the ecosystem depend on them:

- **Dependency ordering.** `depends_on: condition: service_healthy` waits for a dependency to be genuinely ready, not merely started — so an app doesn't race a database that is still initializing.
- **Startup gating in CI and deploys.** `docker compose up -d --wait` blocks until healthchecked services report healthy (or one fails), giving a deploy or pipeline a clear pass/fail signal.
- **Load balancer and proxy routing.** Reverse proxies and orchestrators can keep traffic off a container until it is healthy and pull it out when it goes unhealthy, avoiding requests to a broken instance.
- **Automated recovery.** Swarm replaces unhealthy tasks; tools like `autoheal` restart unhealthy containers. None of that triggers without a healthcheck.
- **Observability.** `docker ps` and `docker inspect` expose health state and the check log, turning "is it actually working?" into something you can see and alert on rather than guess.

Without a healthcheck, all of the above fall back to "the process is running," which is a much weaker guarantee and a common source of flaky startups and silent failures.

## One-Liners, Tips, and Tricks

```bash
# Show health status for every running container at a glance
docker ps --format 'table {{.Names}}\t{{.Status}}'

# Just the health word for one container (healthy | unhealthy | starting | none)
docker inspect -f '{{ .State.Health.Status }}' web

# Fail fast in scripts: exit non-zero unless the container is healthy
docker inspect -f '{{ .State.Health.Status }}' web | grep -q healthy || exit 1

# See exactly why a check is failing — the last probe's exit code and output
docker inspect -f '{{ (index .State.Health.Log 0).ExitCode }} {{ (index .State.Health.Log 0).Output }}' web

# List only unhealthy containers
docker ps --filter health=unhealthy --format '{{.Names}}'

# List containers that are still in the start_period (starting)
docker ps --filter health=starting --format '{{.Names}}'

# Wait for one container to become healthy, then continue (up to ~60s)
until [ "$(docker inspect -f '{{.State.Health.Status}}' web)" = healthy ]; do sleep 2; done

# Bring up a stack and block until everything healthy (great for CI)
docker compose up -d --wait --wait-timeout 120

# Run a service's own healthcheck command manually to debug it
docker exec web sh -c 'wget -q --spider http://localhost/ && echo OK || echo FAIL'

# Confirm whether a tool the check relies on even exists in the image
docker run --rm nginx:1.27-alpine sh -c 'command -v curl || echo "curl NOT present"'

# Count containers by health state
docker ps --format '{{.Status}}' | grep -oE 'healthy|unhealthy|starting' | sort | uniq -c
```

Tips:

- **`--filter health=unhealthy` is the fastest triage.** Use it in monitoring or a cron check to find broken containers without parsing `inspect` output.
- **Read the health `Log` before changing the app.** A failing check is often the check itself — a missing `curl`, wrong port, or HTTPS-vs-HTTP mismatch — not the service. `docker inspect` shows the exit code and captured output.
- **Test the check command with `docker exec` first.** If it passes interactively but fails as a healthcheck, the difference is usually the tool's availability in exec form or a shell feature that needs `CMD-SHELL`.
- **Match `start_period` to real startup time.** Databases, JVM apps, and Elasticsearch often need 30–60s; too short a grace period flaps a healthy service to `unhealthy` on boot.
- **Keep the check cheap.** It runs every `interval` forever; a heavy query or full-page fetch adds constant load. A tiny `/health` endpoint or a native ping is best.
- **`--wait` turns health into a gate.** In CI, `docker compose up -d --wait` fails the job if a dependency never becomes healthy, catching bad deploys early.
- **`autoheal` bridges the restart gap.** Since Docker won't restart an unhealthy container on its own, an `autoheal` sidecar (which watches for `unhealthy` and restarts) is a common single-host complement to a restart policy.

## Quick Reference

```yaml
# HTTP (Alpine/BusyBox — wget)
healthcheck:
  test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 30s

# HTTP (image with curl)
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/health"]

# Postgres / Redis / MySQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
# test: ["CMD", "redis-cli", "ping"]
# test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]

# Disable an inherited check
healthcheck:
  test: ["NONE"]
```

```bash
docker inspect --format '{{ json .State.Health }}' "$(docker compose ps -q web)"
docker compose up -d --wait
```

For related material, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md), [Docker Compose depends_on: Startup Order and Readiness](articles/docker-compose-depends-on.md), and [Docker Restart Policies](articles/docker-restart-policies.md).
