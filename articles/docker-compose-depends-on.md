# Docker Compose depends_on: Startup Order and Readiness

`depends_on` defines dependencies between Compose services and controls the order in which they start and stop. Its most important characteristic is also its most common trap: by default it waits only for the dependency's **container to start**, not for the application inside to be **ready**. This guide covers the syntax, condition types, the readiness gap, and how to handle it correctly.

For the broader command and file reference, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md).

## Basic Syntax

The short form lists the services this one depends on:

```yaml
services:
  web:
    image: nginx:1.27-alpine
    depends_on:
      - database
      - redis

  database:
    image: postgres:16.4

  redis:
    image: redis:7-alpine
```

Compose starts `database` and `redis` before `web`, and stops them in reverse order. Dependencies are matched by **service name** within the same Compose file.

> The top-level `version:` key is obsolete in current Compose and only emits a warning; omit it.

## The Readiness Gap

This is the key point: short-form `depends_on` waits for the container to **start**, not for the service to be **usable**. A Postgres container can be "started" a second before it actually accepts connections, so a dependent app can still fail to connect at boot.

There are two ways to close this gap: condition-based `depends_on` with health checks, or readiness handling in the application (retries or a wait script).

## Condition Types (Long Form)

The long form attaches a condition to each dependency:

| Condition | Waits until the dependency… | Requires |
|-----------|------------------------------|----------|
| `service_started` (default) | has started | — |
| `service_healthy` | reports healthy | a `healthcheck` on the dependency |
| `service_completed_successfully` | has exited with status 0 | a finite/one-shot service |

```yaml
services:
  web:
    build: .
    depends_on:
      database:
        condition: service_healthy
      redis:
        condition: service_started
      migrate:
        condition: service_completed_successfully

  database:
    image: postgres:16.4
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  redis:
    image: redis:7-alpine

  migrate:
    build: .
    command: ["./manage.py", "migrate"]
    depends_on:
      database:
        condition: service_healthy

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

`service_healthy` is the cleanest way to wait for real readiness. `service_completed_successfully` is ideal for one-shot init or migration jobs that must finish before the app starts.

## Handling Readiness in the Application

Even with health checks, well-behaved services should tolerate a dependency being briefly unavailable, since networks and databases can drop and recover at any time. Combine `depends_on` with retry logic and a restart policy.

### Retry in application code

```python
import time
import psycopg2
from psycopg2 import OperationalError

def wait_for_db(retries=30, delay=1):
    for _ in range(retries):
        try:
            psycopg2.connect(
                host="db", dbname="myapp", user="user", password="pass"
            ).close()
            return
        except OperationalError:
            print("Database not ready, retrying...")
            time.sleep(delay)
    raise SystemExit("Database not reachable after retries")

if __name__ == "__main__":
    wait_for_db()
    # start the application
```

Pair it with a restart policy so a hard failure still recovers:

```yaml
services:
  web:
    build: .
    depends_on:
      - db
    restart: on-failure
```

See [Docker Restart Policies](articles/docker-restart-policies.md) for choosing the policy.

### Wait scripts

A wait script blocks until a TCP port is open before launching the app:

```yaml
services:
  web:
    build: .
    depends_on:
      - db
    command: ["./wait-for-it.sh", "db:5432", "--", "python", "app.py"]

  db:
    image: postgres:16.4
```

Tools like [`wait-for-it`](https://github.com/vishnubob/wait-for-it) and [`dockerize`](https://github.com/jwilder/dockerize) check port reachability. Note that an open port still does not guarantee the service is fully initialized; a health check on the dependency is more accurate.

## Build Context Location Does Not Affect depends_on

A frequent misconception is that a service's build directory changes how `depends_on` behaves. It does not. `depends_on` operates on **service names** at the orchestration level, entirely independent of where each service builds from.

```yaml
services:
  frontend:
    build: ./client-app          # local subdirectory
    depends_on:
      - backend                  # works
      - database

  backend:
    build: ../api-server         # parent directory
    depends_on:
      - database                 # works

  database:
    image: postgres:16.4         # prebuilt image, no build context
```

You can freely mix local builds, builds from parent or absolute paths, remote Git-context builds, and prebuilt images. Two things stay true regardless of build location:

- **Service-to-service networking uses the service name.** A service reaches the database at hostname `db` no matter where `db` builds from.
- **Paths in the Compose file are resolved relative to the Compose file**, not to any service's build context. That applies to bind-mount sources and relative build contexts alike. See [Docker Compose Build Configuration and Build Context](articles/docker-compose-build-configuration.md).

### Common misconceptions

| Belief | Reality |
|--------|---------|
| "Building from `../other` breaks depends_on" | Build location is irrelevant to `depends_on` |
| "Services in different directories can't talk" | Any services in the same file resolve each other by name |
| "I must adjust depends_on for build paths" | `depends_on` only uses service names |

## Important Limitations

- **Readiness, not health, by default.** Plain `depends_on: [db]` waits only for the container to start. Use `condition: service_healthy` for real readiness.
- **`--no-deps` skips dependencies.** `docker compose up --no-deps web` starts `web` without its `depends_on` targets, which is useful for isolated restarts but bypasses ordering.
- **Swarm ignores it.** `depends_on` is honored by `docker compose` locally but not by `docker stack deploy` / Swarm mode, which disregards it.
- **It does not add a network dependency by itself.** Services still need to share a network to communicate; `depends_on` only orders startup.
- **Circular dependencies are rejected.** Compose refuses a dependency cycle.

## Verifying Startup Order

```bash
# Start and wait for all services with healthchecks to be healthy
docker compose up -d --wait

# Watch container/health status
docker compose ps

# See the resolved dependency graph and conditions
docker compose config

# Follow a dependent service to confirm it connected on boot
docker compose logs -f web
```

`docker compose up --wait` blocks until services with health checks report healthy (or a dependency fails), which is handy in CI.

## Key Takeaways

- `depends_on` controls **startup/shutdown order**, not application readiness.
- Use `condition: service_healthy` with a dependency `healthcheck` to wait for real readiness.
- Use `condition: service_completed_successfully` for one-shot init/migration jobs.
- Still add retries or a wait script in the app, plus a restart policy, for resilience.
- Build context location is irrelevant; only service names matter for dependencies and networking.
- Compose paths are relative to the Compose file, and `depends_on` is not honored by Swarm.

## Quick Reference

```yaml
services:
  web:
    build: .
    restart: on-failure
    depends_on:
      db:
        condition: service_healthy
      migrate:
        condition: service_completed_successfully
      cache:
        condition: service_started

  db:
    image: postgres:16.4
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  migrate:
    build: .
    command: ["./manage.py", "migrate"]
    depends_on:
      db:
        condition: service_healthy

  cache:
    image: redis:7-alpine
```

```bash
docker compose up -d --wait
docker compose ps
docker compose config
```

For related material, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md), [Docker Restart Policies](articles/docker-restart-policies.md), and [Docker Compose Build Configuration and Build Context](articles/docker-compose-build-configuration.md).
