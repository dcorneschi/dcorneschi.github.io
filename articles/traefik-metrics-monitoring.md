# Traefik Metrics and Monitoring

Traefik can export metrics to Prometheus, OpenTelemetry, StatsD, InfluxDB v2, and Datadog. This guide shows how to enable each in a Docker Compose setup, the correct entry-point wiring, the metrics Traefik actually exposes, and how to secure the endpoint.

Enabling metrics is **static** configuration, so it requires a restart. For the static-vs-dynamic distinction and restart commands, see [Traefik Static vs Dynamic Configuration](articles/traefik-static-vs-dynamic-config.md).

## Bind Metrics to a Dedicated Entry Point

The most important decision: **do not expose metrics on your public web entry point.** By default the Prometheus metrics endpoint is served on the internal `traefik` entry point. The clean, safe pattern is a dedicated `metrics` entry point on a separate port that you only reach internally.

```yaml
services:
  traefik:
    image: traefik:v3.3
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      # Dedicated internal entry point for metrics
      - "--entrypoints.metrics.address=:8082"
      # Prometheus on that entry point
      - "--metrics.prometheus=true"
      - "--metrics.prometheus.entryPoint=metrics"
      - "--metrics.prometheus.addEntryPointsLabels=true"
      - "--metrics.prometheus.addServicesLabels=true"
      - "--metrics.prometheus.addRoutersLabels=true"
    ports:
      - "80:80"
      - "443:443"
      # Publish :8082 only if Prometheus scrapes from off-host
      - "8082:8082"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    restart: unless-stopped
```

Metrics are then at `http://<host>:8082/metrics`. If Prometheus scrapes over the shared Docker network, drop the `8082:8082` publish entirely and let it reach Traefik by service name — the endpoint is unauthenticated.

> The source pattern of `--metrics.prometheus.entryPoint=web` puts metrics on the public HTTP port, which exposes them to the internet. Use a separate `metrics` entry point instead.

## Prometheus Labels and Buckets

```yaml
    command:
      - "--metrics.prometheus=true"
      - "--metrics.prometheus.entryPoint=metrics"
      - "--metrics.prometheus.addEntryPointsLabels=true"   # default true
      - "--metrics.prometheus.addServicesLabels=true"      # default true
      - "--metrics.prometheus.addRoutersLabels=true"       # default false
      - "--metrics.prometheus.buckets=0.1,0.3,1.2,5.0"     # latency histogram buckets
```

`addRoutersLabels` defaults to **false** because per-router labels raise cardinality; enable it only if you need per-router breakdowns. High-cardinality labels are the main driver of Prometheus memory growth, so keep only the label sets you actually query.

## Prometheus + Grafana Stack

A minimal monitoring stack, with metrics kept on the internal `metrics` entry point and Traefik routing Prometheus/Grafana over HTTPS:

```yaml
services:
  traefik:
    image: traefik:v3.3
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.metrics.address=:8082"
      - "--metrics.prometheus=true"
      - "--metrics.prometheus.entryPoint=metrics"
      - "--metrics.prometheus.buckets=0.1,0.3,1.2,5.0"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    networks:
      - web
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:v2.54.1
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
    networks:
      - web
    restart: unless-stopped

  grafana:
    image: grafana/grafana:11.2.0
    environment:
      - GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/grafana_admin
    secrets:
      - grafana_admin
    volumes:
      - grafana-data:/var/lib/grafana
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.example.com`)"
      - "traefik.http.routers.grafana.entrypoints=websecure"
      - "traefik.http.routers.grafana.tls=true"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"
    networks:
      - web
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:

networks:
  web:

secrets:
  grafana_admin:
    file: ./secrets/grafana_admin.txt
```

Prometheus reaches Traefik over the shared `web` network on the metrics port, so no host port for metrics is published. Grafana's admin password is a Docker secret rather than a plaintext `GF_SECURITY_ADMIN_PASSWORD=admin`.

### prometheus.yml

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: traefik
    # Scrape the metrics entry point over the internal network
    static_configs:
      - targets: ["traefik:8082"]
    metrics_path: /metrics
    scrape_interval: 15s

  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
```

Scrape the **metrics entry point port** (`8082`), not the dashboard/API port. The source's `traefik:8080` target only works if metrics happen to share that port, which they should not.

### Grafana dashboards

Import a community Traefik dashboard by ID from grafana.com, then point it at your Prometheus data source. Traefik also publishes an official dashboard; check the current ID on grafana.com rather than relying on an old hardcoded number, since dashboard IDs and their Traefik-version compatibility change over time.

## Securing the Metrics Endpoint

If you must expose `/metrics` through a router (rather than an internal-only port), route the internal Prometheus service through an authenticated router using `manualRouting`:

```yaml
    command:
      - "--metrics.prometheus=true"
      - "--metrics.prometheus.manualRouting=true"    # disable the built-in metrics router
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.metrics.rule=Host(`traefik.example.com`) && Path(`/metrics`)"
      - "traefik.http.routers.metrics.entrypoints=websecure"
      - "traefik.http.routers.metrics.tls=true"
      - "traefik.http.routers.metrics.service=prometheus@internal"
      - "traefik.http.routers.metrics.middlewares=metrics-auth"
      - "traefik.http.middlewares.metrics-auth.basicauth.users=${METRICS_AUTH}"
```

`manualRouting: true` disables Traefik's default internal metrics router so your custom, authenticated router for `prometheus@internal` takes over. Generate the `basicauth` hash with `htpasswd` and supply it via `${METRICS_AUTH}`; in a raw Compose file (not env-substituted) you must double every `$` in the hash as `$$`.

Prefer keeping metrics on an internal network over exposing them at all.

## Other Metrics Backends

Traefik pushes to these backends; enabling any of them is static config.

### StatsD

```yaml
    command:
      - "--metrics.statsd=true"
      - "--metrics.statsd.address=statsd:8125"
      - "--metrics.statsd.pushInterval=10s"
      - "--metrics.statsd.addEntryPointsLabels=true"
      - "--metrics.statsd.addServicesLabels=true"
```

### InfluxDB v2

Current Traefik uses the InfluxDB v2 provider (`influxdb2`), which authenticates with a token and organization — not the old v1 `database` field.

```yaml
    command:
      - "--metrics.influxdb2=true"
      - "--metrics.influxdb2.address=http://influxdb:8086"
      - "--metrics.influxdb2.token=${INFLUX_TOKEN}"
      - "--metrics.influxdb2.org=myorg"
      - "--metrics.influxdb2.bucket=traefik"
      - "--metrics.influxdb2.pushInterval=10s"
```

Supply the token via an environment variable or secret, never inline.

### Datadog

Datadog metrics are sent to a local Datadog Agent (DogStatsD), typically on UDP `8125`:

```yaml
    command:
      - "--metrics.datadog=true"
      - "--metrics.datadog.address=datadog-agent:8125"
      - "--metrics.datadog.pushInterval=10s"
      - "--metrics.datadog.addEntryPointsLabels=true"
      - "--metrics.datadog.addServicesLabels=true"
```

Alternatively, expose Prometheus metrics and have the Datadog Agent's OpenMetrics/Prometheus check scrape `/metrics` — useful if you already run the Agent. See the [Datadog Agent Cheatsheet](articles/datadog-agent-cheatsheet.md).

### OpenTelemetry

Modern Traefik also supports OTLP, which is worth preferring for new setups that already run an OpenTelemetry collector:

```yaml
    command:
      - "--metrics.otlp=true"
      - "--metrics.otlp.http.endpoint=http://otel-collector:4318/v1/metrics"
```

## What Traefik Exposes

Metric names use the `traefik_` prefix and are grouped by scope. Note the naming: Traefik uses **entrypoint**, **router**, and **service** scopes — there is no `traefik_backend_*` family in v2/v3 (that was v1 terminology).

| Scope | Example metrics |
|-------|-----------------|
| Entry point | `traefik_entrypoint_requests_total`, `traefik_entrypoint_request_duration_seconds`, `traefik_entrypoint_open_connections` |
| Router | `traefik_router_requests_total`, `traefik_router_request_duration_seconds` (needs `addRoutersLabels`) |
| Service | `traefik_service_requests_total`, `traefik_service_request_duration_seconds`, `traefik_service_server_up`, `traefik_service_open_connections` |
| Config | `traefik_config_reloads_total`, `traefik_config_last_reload_success` |
| TLS | `traefik_tls_certs_not_after` (certificate expiry timestamp) |

`traefik_service_server_up` and `traefik_tls_certs_not_after` are especially useful for alerting: the first flags a dead backend, the second lets you alert before a certificate expires.

Exact metric names can vary slightly by Traefik version; confirm against your running instance:

```bash
# List available metric names (from a host that can reach the metrics port)
curl -s http://<host>:8082/metrics | grep -E '^# HELP traefik_' | awk '{print $3}'
```

## Troubleshooting

| Symptom | Check |
|---------|-------|
| No metrics at all | `--metrics.prometheus=true` set? Restart done (static config)? |
| Endpoint unreachable | Correct entry-point port (`:8082`), and published or on the scrape network? |
| Prometheus target down | Scrape `targets` point at the metrics port, `metrics_path: /metrics`? |
| Missing router metrics | `addRoutersLabels=true` is required and off by default |
| High Prometheus memory | Reduce label cardinality (drop `addRoutersLabels`), raise `scrape_interval`, set retention |
| Metrics reachable publicly | Move to an internal entry point or add auth via `manualRouting` |

```bash
# Confirm the endpoint responds
curl -fsS http://<host>:8082/metrics | head

# Confirm Traefik sees a target's config reloads succeeding
curl -s http://<host>:8082/metrics | grep traefik_config_last_reload_success
```

## Quick Reference

```yaml
# Static (dedicated internal metrics entry point + Prometheus)
command:
  - "--entrypoints.metrics.address=:8082"
  - "--metrics.prometheus=true"
  - "--metrics.prometheus.entryPoint=metrics"
  - "--metrics.prometheus.addServicesLabels=true"
  - "--metrics.prometheus.buckets=0.1,0.3,1.2,5.0"
```

```yaml
# prometheus.yml scrape job
scrape_configs:
  - job_name: traefik
    static_configs:
      - targets: ["traefik:8082"]
    metrics_path: /metrics
```

```bash
# Verify
curl -fsS http://<host>:8082/metrics | head
```

For related material, see [Traefik Static vs Dynamic Configuration](articles/traefik-static-vs-dynamic-config.md), the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md), [Defining Variables in Docker Compose](articles/docker-compose-variables-env.md), and the [Datadog Agent Cheatsheet](articles/datadog-agent-cheatsheet.md).
