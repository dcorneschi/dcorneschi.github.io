# Traefik Static vs Dynamic Configuration

Traefik splits its configuration into two kinds that behave very differently: **static** configuration is read once at startup, and **dynamic** configuration is watched and reloaded at runtime. Understanding which is which explains why some changes need a restart and others take effect instantly. This guide covers both, how they map to a Docker Compose setup, and the common patterns.

## The Two Configuration Types

### Static configuration

Read **once at startup**; changing it requires restarting Traefik. It defines the things that shape the process itself:

- Entry points (the ports Traefik listens on, e.g. `:80`, `:443`).
- Providers (Docker, file, Kubernetes, etc.) and how to reach them.
- Certificate resolvers (ACME/Let's Encrypt accounts and challenge type).
- API/dashboard, logging, and global options.

Static config can be supplied in exactly one of three ways (do not mix them for the same settings):

1. A config file: `traefik.yml`, `traefik.yaml`, or `traefik.toml`.
2. Command-line flags (common in Docker Compose).
3. Environment variables.

### Dynamic configuration

Watched and **reloaded at runtime** without a restart. It defines routing and traffic handling:

- Routers (rules that match requests to services).
- Services (backends and load balancing).
- Middlewares (auth, headers, redirects, rate limiting).
- TLS options and certificates.

Dynamic config comes from **providers**: Docker labels, a watched file (the file provider), Kubernetes CRDs/annotations, and others. Adding a labelled container or editing a watched file updates routing live.

| Aspect | Static | Dynamic |
|--------|--------|---------|
| When read | Startup only | Continuously, at runtime |
| Change requires restart | Yes | No |
| Defines | Entry points, providers, resolvers, API, logging | Routers, services, middlewares, TLS certs |
| Typical source | CLI flags / `traefik.yml` / env vars | Docker labels, file provider, K8s |

## A Compose Setup Uses Both

A typical Traefik service in `docker-compose.yml` combines static config (as CLI flags) with dynamic config (as labels).

### Static config via command-line flags

```yaml
services:
  traefik:
    image: traefik:v3.3
    command:
      - --api.dashboard=true
      - --log.level=INFO
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https
      - --certificatesresolvers.letsencrypt.acme.email=${ACME_EMAIL}
      - --certificatesresolvers.letsencrypt.acme.storage=/acme.json
      - --certificatesresolvers.letsencrypt.acme.tlschallenge=true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./acme.json:/acme.json
    restart: unless-stopped
```

Notes:

- Pin the image to a version (`traefik:v3.3`) rather than `latest`, so an upgrade with breaking config changes is deliberate.
- Mount the Docker socket **read-only** (`:ro`). It is still effectively root on the host, so treat exposing Traefik carefully.
- `acme.json` must exist and be `chmod 600`, or Traefik refuses to use it. Persist it on a volume so certificates survive restarts and you don't hit Let's Encrypt rate limits.
- Avoid `--api.insecure=true` and `--api.debug=true` in anything reachable; expose the dashboard through a secured router instead (below).

### Dynamic config via Docker labels

Labels on the Traefik container itself route the dashboard through a secured HTTPS router:

```yaml
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`${TRAEFIK_DOMAIN:-traefik.localhost}`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls=true"
      - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=dashboard-auth"
      - "traefik.http.middlewares.dashboard-auth.basicauth.users=${DASHBOARD_AUTH}"
```

`api@internal` is the built-in service that serves the dashboard/API. Putting a `basicauth` (or forward-auth) middleware in front of it is important — the dashboard exposes your whole routing topology and should never be open.

## Why This Split Works Well in Compose

- **No restart to add services.** New containers with `traefik.*` labels are picked up live by the Docker provider.
- **Clear separation.** Infrastructure (ports, resolvers, providers) is static; per-app routing is dynamic and lives next to each service.
- **Scales cleanly.** Onboarding a service is just adding labels to it.
- **Environment-driven.** `${ACME_EMAIL}`, `${TRAEFIK_DOMAIN}` etc. let one file serve multiple environments. See [Defining Variables in Docker Compose](articles/docker-compose-variables-env.md).
- **Automatic TLS.** The static ACME resolver plus per-router `tls.certresolver` gives per-service Let's Encrypt certificates.

## Routing a Service with Labels

### Basic HTTP routing

```yaml
services:
  myapp:
    image: myapp:1.4.2
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.example.com`)"
      - "traefik.http.services.myapp.loadbalancer.server.port=8080"
```

`loadbalancer.server.port` is the **container** port Traefik forwards to; it does not need to be published on the host, since Traefik reaches it over the shared Docker network. The service and its router should share a network with Traefik.

### HTTPS with a middleware

```yaml
services:
  myapp:
    image: myapp:1.4.2
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.example.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls=true"
      - "traefik.http.routers.myapp.tls.certresolver=letsencrypt"
      - "traefik.http.routers.myapp.middlewares=myapp-auth"
      - "traefik.http.middlewares.myapp-auth.basicauth.users=${MYAPP_AUTH}"
      - "traefik.http.services.myapp.loadbalancer.server.port=8080"
```

With `exposedbydefault=false` (set in the static config), every service must opt in with `traefik.enable=true` — a good default that avoids accidentally exposing containers.

## Referencing @file Middlewares

A middleware referenced as `something@file` must be defined by the **file provider**, which is dynamic config from a watched file — not automatically present. Enable it in the static config:

```yaml
    command:
      - --providers.file.directory=/etc/traefik/dynamic
      - --providers.file.watch=true
```

Then define shared middlewares in a watched dynamic file:

```yaml
# /etc/traefik/dynamic/middlewares.yml
http:
  middlewares:
    secure-headers:
      headers:
        stsSeconds: 31536000
        contentTypeNosniff: true
        frameDeny: true
    auth:
      basicAuth:
        usersFile: /etc/traefik/users.htpasswd
```

Now a label like `traefik.http.routers.myapp.middlewares=auth@file` resolves. The `@file` / `@docker` suffix names the provider a resource comes from; a label-defined middleware is `@docker` and needs no suffix when referenced from another label in the same provider.

## Provider Suffixes at a Glance

| Suffix | Source |
|--------|--------|
| `@docker` | Defined via Docker labels |
| `@file` | Defined in a file-provider dynamic file |
| `@internal` | Built in (e.g. `api@internal`, `dashboard@internal`) |
| `@kubernetescrd` / `@kubernetesingress` | Kubernetes providers |

## Common Pitfalls

- **Editing a CLI flag and expecting a live change.** Entry points, providers, and resolvers are static — restart Traefik after changing them.
- **Referencing `@file` middlewares without the file provider.** They won't resolve; enable `--providers.file` and point it at a watched directory.
- **Exposing the dashboard unprotected.** Always put auth in front of `api@internal`, and avoid `--api.insecure`.
- **`acme.json` permissions.** Must be `600`; Traefik refuses a world-readable ACME store.
- **Hitting Let's Encrypt rate limits.** Test with the ACME staging CA (`--certificatesresolvers.letsencrypt.acme.caserver=...staging...`) before switching to production.
- **Wrong `loadbalancer.server.port`.** Use the port the app listens on inside the container, not the published host port.

## Restarting Traefik After a Static Change

Because static configuration is read only at startup, any change to it requires restarting Traefik. Dynamic changes (routers, services, middlewares) from Docker labels or a watched file with `watch: true` reload automatically and need no restart.

Typical static changes that require a restart:

- Adding or modifying an entry point (for example a new `metrics` entrypoint on `:8080`).
- Changing providers, ACME resolvers, logging, or API/dashboard settings.
- Enabling or reconfiguring the metrics provider.

### How to restart

```bash
# Docker Compose (v2)
docker compose restart traefik

# Docker Compose (legacy v1)
docker-compose restart traefik

# Plain docker
docker restart <traefik-container-name>

# Docker Swarm
docker service update --force <traefik-service-name>
```

> If you changed the `command`/flags or a mounted `traefik.yml`, `docker compose up -d` re-reads the service definition and recreates the container, which also applies the new static config. Use `restart` when the config file changed but the Compose service definition did not; use `up -d` when you edited the flags or volumes in the Compose file itself.

## Exposing Prometheus Metrics

Traefik can expose Prometheus metrics. Enabling them touches **static** configuration in two places — the metrics provider and the entry point it binds to — so it needs a restart.

```yaml
    command:
      # ... existing static flags ...
      - --entrypoints.metrics.address=:8080
      - --metrics.prometheus=true
      - --metrics.prometheus.entrypoint=metrics
    ports:
      - "8080:8080"    # only if you need to reach /metrics from off-host
```

After restarting, metrics are served at `/metrics` on that entry point, e.g. `http://<host>:8080/metrics` when the port is published.

Two important cautions:

- **Publish `:8080` only if you must.** The `ports` mapping is what makes the metrics endpoint reachable from other hosts. If Prometheus scrapes over the internal Docker network, leave the port unpublished and let the scraper reach Traefik by service name — the metrics endpoint is unauthenticated.
- **Do not expose `/metrics` publicly.** It leaks routing and traffic detail. Keep it on an internal network, restrict it with a firewall, or front it with an authenticated router rather than binding a raw public port.

## Inspecting the API

Traefik's REST API exposes what it has discovered — routers, services, middlewares — which is invaluable for confirming that a container's labels were picked up. The API is enabled by `--api.dashboard=true` (static config) and is served by `api@internal`.

Query it through the same secured router that fronts the dashboard. If you protected the dashboard with `basicauth`, pass credentials:

```bash
# List discovered services (backends Traefik routes to)
curl -u "$TRAEFIK_USER:$TRAEFIK_PASS" https://traefik.example.com/api/http/services | jq .

# List routers (rule → service mappings)
curl -u "$TRAEFIK_USER:$TRAEFIK_PASS" https://traefik.example.com/api/http/routers | jq .

# List middlewares
curl -u "$TRAEFIK_USER:$TRAEFIK_PASS" https://traefik.example.com/api/http/middlewares | jq .

# Overview of providers, entrypoints, and feature status
curl -u "$TRAEFIK_USER:$TRAEFIK_PASS" https://traefik.example.com/api/overview | jq .

# Raw merged configuration Traefik is currently using
curl -u "$TRAEFIK_USER:$TRAEFIK_PASS" https://traefik.example.com/api/rawdata | jq .
```

Pass the credentials from environment variables or a `.netrc` file rather than hardcoding them in commands or scripts, and never commit real credentials. Filter with `jq` to find a specific service, for example:

```bash
curl -sf -u "$TRAEFIK_USER:$TRAEFIK_PASS" https://traefik.example.com/api/http/routers \
  | jq '.[] | select(.service=="myapp@docker")'
```

If a container you expected is missing from `/api/http/services`, check that it has `traefik.enable=true`, shares a network with Traefik, and that the labels are well-formed.

## Health Check with ping

Traefik provides a lightweight `/ping` endpoint that returns `200 OK` when the process is healthy — useful for load balancers, uptime monitors, and container health checks. Enable it in static config and bind it to an entry point (commonly the same internal one as metrics):

```yaml
    command:
      # ... existing static flags ...
      - --ping=true
      - --ping.entrypoint=metrics
      - --entrypoints.metrics.address=:8080
```

Then probe it:

```bash
# Returns HTTP 200 and the body "OK" when healthy
curl -fsS http://<host>:8080/ping
```

Use it as the container's own healthcheck so orchestration knows when Traefik is ready:

```yaml
services:
  traefik:
    image: traefik:v3.3
    healthcheck:
      test: ["CMD", "traefik", "healthcheck", "--ping"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 10s
```

The built-in `traefik healthcheck --ping` command calls `/ping` internally, so it works without needing `curl` or `wget` in the image. See [Docker Healthcheck Examples](articles/docker-healthcheck-examples.md).

Like metrics, keep `/ping` on an internal entry point rather than exposing it publicly.

## Quick Reference

```yaml
# Static (CLI flags on the traefik service)
command:
  - --providers.docker=true
  - --providers.docker.exposedbydefault=false
  - --providers.file.directory=/etc/traefik/dynamic
  - --providers.file.watch=true
  - --entrypoints.web.address=:80
  - --entrypoints.websecure.address=:443
  - --certificatesresolvers.letsencrypt.acme.email=${ACME_EMAIL}
  - --certificatesresolvers.letsencrypt.acme.storage=/acme.json
  - --certificatesresolvers.letsencrypt.acme.tlschallenge=true
```

```yaml
# Dynamic (labels on an app service)
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.myapp.rule=Host(`myapp.example.com`)"
  - "traefik.http.routers.myapp.entrypoints=websecure"
  - "traefik.http.routers.myapp.tls.certresolver=letsencrypt"
  - "traefik.http.services.myapp.loadbalancer.server.port=8080"
```

| Change | Restart needed? |
|--------|-----------------|
| Add/edit an entry point, provider, or ACME resolver | Yes (static) |
| Add a service with labels | No (dynamic) |
| Edit a watched file-provider file | No (dynamic) |
| Change log level or dashboard flags | Yes (static) |

For related material, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md), [Defining Variables in Docker Compose](articles/docker-compose-variables-env.md), and [SNI and TLS Certificates Guide](articles/sni-certificates-guide.md).
