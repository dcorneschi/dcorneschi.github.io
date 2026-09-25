# Preventing Services from Starting Automatically in Docker Compose

By default, `docker compose up -d` starts every service defined in the Compose file. Sometimes you want a service to stay defined but *not* start automatically — debug tools, one-off maintenance jobs, environment-specific extras, or a temporarily disabled service. This guide covers the ways to do that, from the clean modern approach (`profiles`) to older or situational workarounds, with the tradeoffs of each.

For the broader command and file reference, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md).

## 1. Profiles (Recommended)

Profiles are the native, purpose-built mechanism, available since Compose v1.28 (and standard in Compose v2). A service assigned to a profile stays inactive unless that profile is explicitly enabled.

```yaml
services:
  my-service:
    image: nginx:1.27-alpine
    profiles:
      - manual   # inactive unless the "manual" profile is enabled
```

```bash
docker compose up -d                       # my-service does NOT start
docker compose --profile manual up -d      # my-service starts
COMPOSE_PROFILES=manual docker compose up -d   # same, via env var
```

A service with **no** `profiles:` key always starts. A service *with* one or more profiles starts only when at least one of its profiles is enabled. Enabling a profile also implicitly starts services that a profiled service depends on, so its `depends_on` targets come up too.

> The top-level `version:` key is obsolete in current Compose and only emits a warning; omit it.

## 2. Scale to Zero

You can start a service with zero replicas so nothing actually runs. This works on older Compose versions that predate profiles.

```bash
docker compose up -d --scale my-service=0
```

Or declare it in the file (Swarm-oriented `deploy` block, honored by Compose for replica count):

```yaml
services:
  my-service:
    image: nginx:1.27-alpine
    deploy:
      replicas: 0
```

This is less expressive than profiles — there's no named grouping, and the intent is not obvious to someone reading the file — but it's a quick way to keep a service defined while suppressing it.

## 3. restart: "no" Plus a Manual Stop

`restart: "no"` only controls whether a **crashed or exited** container is restarted; it does **not** prevent the initial start. The service still comes up on `up -d`, and you have to stop it yourself.

```yaml
services:
  my-service:
    image: nginx:1.27-alpine
    restart: "no"
```

```bash
docker compose up -d
docker compose stop my-service
```

This is not really "prevention" — the container runs at least once. Prefer profiles unless you specifically want this start-once-then-stop behavior. See [Docker Restart Policies](articles/docker-restart-policies.md) for what `restart:` actually governs.

## 4. Comment the Service Out

The bluntest option: comment the service block so Compose never sees it.

```yaml
services:
  # my-service:
  #   image: nginx:1.27-alpine
  #   ports:
  #     - "80:80"
```

Fine for a quick, temporary local disable, but it requires editing the file each time, doesn't version cleanly, and is easy to forget. Profiles express "optional" without mutating the file.

## 5. Separate Compose File

Keep optional services in their own file and include it only when wanted. Compose merges multiple files left to right.

`docker-compose.yml` holds the always-on services; `docker-compose.optional.yml` holds the extras:

```yaml
# docker-compose.optional.yml
services:
  optional-service:
    image: nginx:1.27-alpine
```

```bash
docker compose up -d                                                    # base only
docker compose -f docker-compose.yml -f docker-compose.optional.yml up -d   # base + optional
```

This gives complete separation and suits per-environment splits (a common pattern is a base file plus a `docker-compose.override.yml` that Compose loads automatically). The cost is more files to manage; for optional services inside one environment, profiles are usually cleaner.

## Putting Profiles to Work

A realistic layout: production services always run, and tooling is grouped behind named profiles.

```yaml
services:
  # Always runs (no profile)
  app:
    image: myapp:latest
    ports:
      - "3000:3000"

  database:
    image: postgres:16.4
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

  # Development tooling
  adminer:
    image: adminer
    profiles: [dev]
    ports:
      - "8080:8080"

  # Testing
  test-db:
    image: postgres:16.4
    profiles: [test]
    environment:
      POSTGRES_DB: test_db

  # Maintenance — attached to two profiles
  backup:
    image: backup-tool
    profiles: [maintenance, backup]

  # Monitoring
  monitoring:
    image: prom/prometheus
    profiles: [monitoring]

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

```bash
docker compose up -d                                  # app + database only
docker compose --profile dev up -d                    # + adminer
docker compose --profile test up -d                   # + test-db
docker compose --profile dev --profile monitoring up -d   # + adminer + monitoring
docker compose --profile maintenance up -d            # + backup
```

A service listed under several profiles (like `backup`) activates if **any** of them is enabled. You can enable multiple profiles at once by repeating `--profile` or setting `COMPOSE_PROFILES=dev,monitoring`.

## Comparison

| Method | Pros | Cons | Best for |
|--------|------|------|----------|
| **Profiles** | Native, named groups, everything in one file, self-documenting | Needs Compose v1.28+ | Optional/grouped services |
| **Scale 0** | Works on older versions, simple | No naming, intent unclear | Quick suppression |
| **Separate files** | Full separation | More files to track | Per-environment splits |
| **Comment out** | Trivial | Manual edits, easy to forget | Throwaway local disable |
| **restart: "no"** | Simple | Container still starts once — not true prevention | Start-once-then-stop |

## Verifying What Will Start

```bash
docker compose config --services            # all services Compose knows about
docker compose --profile dev config --services   # services active with a profile enabled
docker compose ps                           # what is actually running now
docker compose up -d --dry-run              # preview actions without applying (Compose v2.20+)
```

`docker compose config` resolves the effective configuration, so it's the fastest way to confirm which services a given profile set will bring up before you run `up`.

## Key Takeaways

- `docker compose up -d` starts every service that has **no** profile assigned.
- **Profiles** are the recommended way to keep a service defined but inactive; enable them with `--profile` or `COMPOSE_PROFILES`.
- **Scale 0** and **separate files** are valid alternatives for older setups or per-environment separation.
- **`restart: "no"` does not prevent startup** — the container runs once; you must stop it.
- Use `docker compose config --services` to confirm exactly what a profile selection will start.

## Quick Reference

```yaml
services:
  web:
    image: nginx:1.27-alpine    # always starts

  debug-tool:
    image: busybox
    profiles: [debug]           # only with --profile debug
    command: sleep infinity
```

```bash
docker compose up -d                     # web only
docker compose --profile debug up -d     # web + debug-tool
docker compose --scale web=0 up -d       # start nothing for web
docker compose config --services         # list all defined services
```

For related material, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md), [Docker Compose depends_on: Startup Order and Readiness](articles/docker-compose-depends-on.md), and [Docker Restart Policies](articles/docker-restart-policies.md).
