<img src="/articles/images/datadog-logo.svg" alt="Datadog" width="150"/>

# Datadog Agent Cheatsheet

Quick reference for managing the Datadog Agent on Linux hosts. Covers installation, service management, configuration, checks, logs, and troubleshooting.

## Installation

### One-line install (latest version)

```bash
DD_API_KEY="your-api-key" DD_SITE="datadoghq.com" bash -c "$(curl -L https://install.datadoghq.com/scripts/install_script_agent7.sh)"
```

### Install a specific version

```bash
DD_API_KEY="your-api-key" DD_AGENT_MAJOR_VERSION=7 DD_AGENT_MINOR_VERSION=50.0-1 \
  bash -c "$(curl -L https://install.datadoghq.com/scripts/install_script_agent7.sh)"
```

### Region-specific DD_SITE values

| Region | DD_SITE |
|--------|---------|
| US1 (default) | `datadoghq.com` |
| US3 | `us3.datadoghq.com` |
| US5 | `us5.datadoghq.com` |
| EU | `datadoghq.eu` |
| AP1 | `ap1.datadoghq.com` |

## API Key vs Application Key

Datadog uses two types of keys for different purposes:

| | API Key | Application Key |
|--|---------|-----------------|
| **Purpose** | Submit data (metrics, logs, traces, events) | Read data and manage resources via API |
| **Used by** | Datadog Agent, integrations, libraries | Scripts, Terraform, API automation |
| **Access level** | Write-only — can push data, nothing else | Read/write — scoped to the user who created it |
| **Risk if leaked** | Attacker can send garbage data to your account | Attacker can read your data, modify monitors, dashboards, etc. |
| **Required for agent?** | Yes | No |
| **Where to find** | Organization Settings > API Keys | Organization Settings > Application Keys |

### When you need which

- **Agent installation** — API key only. The agent submits metrics/logs/traces but never reads back.
- **Terraform provider** — Both. API key to identify the account, application key to create/read/update/delete resources.
- **API queries** (list monitors, get dashboards, search logs) — Both. API key for auth, application key for authorization.
- **Custom metrics from code** (DogStatsD, client libraries) — API key only.

### Key format

```bash
# API key — 32-character hex string
api_key: a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4

# Application key — 40-character hex string
app_key: b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3
```

### Scoping application keys

Application keys inherit the permissions of the user who created them. If that user is an admin, the key has admin access. Best practice:

- Create a dedicated service account with minimal permissions
- Generate the application key from that account
- Use RBAC roles to limit what the key can do

### Environment variables

```bash
# For the agent (datadog.yaml picks these up)
export DD_API_KEY="your-api-key"

# For API/Terraform usage
export DD_API_KEY="your-api-key"
export DD_APP_KEY="your-application-key"
```

## Service Management

### systemd (RHEL 7+, Ubuntu 16.04+, Debian 8+)

```bash
# Start / Stop / Restart
sudo systemctl start datadog-agent
sudo systemctl stop datadog-agent
sudo systemctl restart datadog-agent

# Status
sudo systemctl status datadog-agent

# Enable / Disable on boot
sudo systemctl enable datadog-agent
sudo systemctl disable datadog-agent
```

### Using the datadog-agent command directly

```bash
sudo datadog-agent run           # Run in foreground (useful for debugging)
sudo datadog-agent version       # Show agent version
sudo datadog-agent hostname      # Show resolved hostname
```

## Key File Paths

| Path | Description |
|------|-------------|
| `/etc/datadog-agent/datadog.yaml` | Main agent configuration |
| `/etc/datadog-agent/conf.d/` | Integration check configurations |
| `/opt/datadog-agent/` | Agent binary and embedded Python |
| `/var/log/datadog/agent.log` | Main agent log |
| `/var/log/datadog/process-agent.log` | Process agent log |
| `/var/log/datadog/trace-agent.log` | APM trace agent log |
| `/var/log/datadog/jmxfetch.log` | JMX fetch log |
| `/opt/datadog-agent/bin/agent/agent` | Agent binary |
| `/etc/datadog-agent/auth_token` | IPC auth token (used by CLI) |

## Configuration (datadog.yaml)

### Essential settings

```yaml
api_key: <YOUR_API_KEY>
site: datadoghq.com

# Hostname override (default: auto-detected)
hostname: my-server-01

# Tags applied to all metrics/logs from this host
tags:
  - env:production
  - team:platform
  - service:web-api
```

### Reserved tag keys

Datadog treats certain tag keys as special — they enable cross-product correlation and scoping:

| Tag Key | Purpose |
|---------|---------|
| `host` | Correlation between metrics, traces, processes, and logs |
| `device` | Segregation of metrics, traces, processes, and logs by device or disk |
| `source` | Span filtering and automated pipeline creation for Log Management |
| `service` | Scoping of application-specific data across metrics, traces, and logs |
| `env` | Scoping of application-specific data across metrics, traces, and logs |
| `version` | Scoping of application-specific data across metrics, traces, and logs |
| `team` | Assigning ownership to any resources |

These three together (`service`, `env`, `version`) form [unified service tagging](https://docs.datadoghq.com/getting_started/tagging/unified_service_tagging/) — they link metrics, traces, and logs for a given service in a given environment at a given version. Always set them.

### Tag naming rules

Tags use the format `<key>:<value>` (recommended) or just `<value>`:

- Must **start with a letter**
- Can contain: letters (Unicode), numbers, underscores, minuses, colons, periods, forward slashes
- Max **200 characters** (key + `:` + value combined)
- Normalized to **lowercase** (avoid camelCase)
- All other characters (spaces, commas, emoji) are converted to underscores

```bash
# Good tags
env:production
service:payment-api
team:platform
region:eu-west-1

# Bad tags (will be normalized or rejected)
Env:Production         # camelCase → normalized to env:production
2024-deploy:v1         # starts with a digit — may not work consistently
user_id:12345          # unbounded source — causes cardinality explosion
```

Avoid tags from unbounded sources (timestamps, user IDs, request IDs) — they cause metric cardinality to grow without limit.

See: [Getting Started with Tags](https://docs.datadoghq.com/getting_started/tagging/)

### Metric units

When submitting custom metrics, specify a unit in the Datadog UI (Metric Summary > Edit > Metadata) so graphs display human-readable values. Without a unit, Datadog uses SI notation (k, M, G, T).

Common unit categories:

| Category | Units |
|----------|-------|
| Bytes | `bit`, `byte`, `kibibyte (KiB)`, `mebibyte (MiB)`, `gibibyte (GiB)`, `tebibyte (TiB)` |
| Time | `nanosecond (ns)`, `microsecond (μs)`, `millisecond (ms)`, `second (s)`, `minute (min)`, `hour (hr)`, `day`, `week` |
| Percentage | `percent (%)`, `apdex`, `fraction` |
| CPU | `nanocore (ncores)`, `millicore (mcores)`, `core`, `kilocore (Kcores)` |
| Network | `connection (conn)`, `request (req)`, `packet (pkt)`, `response (rsp)`, `message (msg)` |
| System | `process (proc)`, `thread`, `host`, `node`, `instance` |
| Disk | `file`, `inode`, `sector`, `block (blk)` |
| DB | `table`, `index (idx)`, `lock`, `transaction (tx)`, `query`, `row` |
| Cache | `hit`, `miss`, `eviction`, `get`, `set` |
| General | `error (err)`, `read (rd)`, `write (wr)`, `operation (op)`, `event`, `task` |

Auto-formatting examples:

| Unit | Raw Value | Displayed |
|------|-----------|-----------|
| byte | 1 | 1 B |
| kibibyte | 1234235 | 1.18 GiB |
| hertz | 6345223 | 6.35 MHz |
| cent | 1337 | 13.37 $ |
| nanosecond | 0 | 0s |
| second | 0.032123 | 32.12ms |
| second | 967 | 16m 7s |
| second | 86390 | 1d |

See: [Metrics Units](https://docs.datadoghq.com/metrics/units/)

### Autodiscovery template variables

When configuring integrations for containers (Docker, Kubernetes, ECS), use these template variables in `conf.d` annotations or labels to dynamically resolve container values:

| Template Variable | Description |
|-------------------|-------------|
| `%%host%%` | Container's network IP |
| `%%host_<NETWORK>%%` | IP on a specific network (falls back to `%%host%%` if not found) |
| `%%port%%` | Highest exposed port (sorted ascending) |
| `%%port_<N>%%` | Nth port sorted ascending (`%%port_0%%` = lowest) |
| `%%port_<NAME>%%` | Port associated with a named port |
| `%%pid%%` | Container process ID |
| `%%hostname%%` | Hostname from container config (use when `%%host%%` can't get a reliable IP, e.g., ECS awsvpc) |
| `%%env_<ENV_VAR>%%` | Value of an environment variable visible to the Agent |
| `%%kube_namespace%%` | Kubernetes namespace |
| `%%kube_pod_name%%` | Kubernetes pod name |
| `%%kube_pod_uid%%` | Kubernetes pod UID |

**Example — Kubernetes pod annotation for an nginx check:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    ad.datadoghq.com/nginx.checks: |
      {
        "nginx": {
          "instances": [{"nginx_status_url": "http://%%host%%:%%port%%/nginx_status"}]
        }
      }
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 8080
```

**Example — Docker label for a Redis check:**

```yaml
labels:
  com.datadoghq.ad.checks: '{"redisdb": {"instances": [{"host": "%%host%%", "port": "%%port%%"}]}}'
```

**Platform support:**

| Variable | Docker | ECS Fargate | Kubernetes |
|----------|--------|-------------|------------|
| `%%host%%` | Yes | Yes | Yes |
| `%%port%%` | Yes | No | Yes |
| `%%pid%%` | Yes | No | No |
| `%%env_*%%` | Yes | Yes | Yes |
| `%%hostname%%` | Yes | No | No |
| `%%kube_namespace%%` | No | No | Yes |
| `%%kube_pod_name%%` | No | No | Yes |
| `%%kube_pod_uid%%` | No | No | Yes |

See: [Autodiscovery Template Variables](https://docs.datadoghq.com/containers/guide/template_variables/)

### Network and ports

All Agent traffic is outbound over SSL (TCP). No sessions are initiated from Datadog back to the Agent.

**Outbound ports (firewall allowlist):**

| Port | Protocol | Purpose |
|------|----------|---------|
| 443 | TCP | Metrics, APM, containers, live processes, logs (HTTP), network monitoring |
| 123 | UDP | NTP time synchronization |

**Inbound ports (local host only):**

| Port | Protocol | Purpose |
|------|----------|---------|
| 5002 | TCP | Agent browser GUI |
| 8126 | TCP | APM trace receiver (and profiler) |
| 8125 | UDP | DogStatsD (custom metrics) |
| 5001 | TCP | IPC API (inter-process communication) |
| 5000 | TCP | go_expvar server |
| 6062 | TCP | Process Agent debug endpoints |
| 6162 | TCP | Process Agent runtime config |

**Domains to allowlist:**

| Service | Domain |
|---------|--------|
| Metrics / Agent data | `<VERSION>-app.agent.datadoghq.com` (e.g., `7-31-0-app.agent.datadoghq.com`) |
| APM traces | `trace.agent.datadoghq.com` |
| Live containers / processes | `process.datadoghq.com` |
| Logs (HTTP) | `http-intake.logs.datadoghq.com` |
| API | `api.datadoghq.com` |
| Flare | `<VERSION>-flare.agent.datadoghq.com` |
| Installation | `install.datadoghq.com`, `apt.datadoghq.com`, `yum.datadoghq.com` |

> Adjust domains for your region (e.g., `.datadoghq.eu` for EU).

**Static IP ranges:**

```bash
# Get all IP ranges for your site
curl -s https://ip-ranges.datadoghq.com | jq .

# Get IPs for a specific service
curl -s https://ip-ranges.datadoghq.com/logs.json | jq '.prefixes_ipv4'
curl -s https://ip-ranges.datadoghq.com/apm.json | jq '.prefixes_ipv4'
```

**Data buffering when network is unavailable:**

The Agent stores metrics in memory if the network goes down. Configure disk-based buffering for durability:

```yaml
# In datadog.yaml
forwarder_retry_queue_payloads_max_size: 15728640    # Max memory buffer (bytes, default ~15MB)
forwarder_storage_max_size_in_bytes: 52428800         # Enable disk buffer (e.g., 50MB)
forwarder_storage_path: /opt/datadog-agent/run/transactions_to_retry
```

See: [Agent Network Traffic](https://docs.datadoghq.com/agent/configuration/network/)

### Other essential settings

```yaml
# Enable log collection (disabled by default)
logs_enabled: true

# Enable APM (enabled by default on port 8126)
apm_config:
  enabled: true

# Enable process monitoring
process_config:
  process_collection:
    enabled: true

# Enable network performance monitoring
network_config:
  enabled: true

# Proxy settings
proxy:
  http: http://proxy.example.com:3128
  https: http://proxy.example.com:3128
  no_proxy:
    - 169.254.169.254
```

### Log level

```yaml
# Options: trace, debug, info, warn, error, critical, off
log_level: info
```

Change log level at runtime without restarting:

```bash
sudo datadog-agent config set log_level debug

# List all runtime-configurable parameters
sudo datadog-agent config list-runtime

# Get current value of a setting
sudo datadog-agent config get log_level
```

## Agent Status & Diagnostics

### Full status report

```bash
sudo datadog-agent status
```

This shows:
- Agent version and uptime
- Hostname resolution
- API key validation (last 4 chars)
- Running checks and their status
- Log agent status
- Forwarder status (payload queue)
- DogStatsD metrics count

### Specific status sections

The output of `status` includes all sections (Collector, Forwarder, DogStatsD, etc.). To filter:

```bash
sudo datadog-agent status | grep -A 30 "Running Checks"   # Check runner info
sudo datadog-agent status | grep -A 20 "Forwarder"        # Payload submission info
```

### Health check

```bash
sudo datadog-agent health
```

### Configuration dump (resolved values)

```bash
# Print all configurations loaded & resolved of a running agent
sudo datadog-agent configcheck

# Show the complete runtime configuration
sudo datadog-agent config
```

### Diagnose connectivity

```bash
sudo datadog-agent diagnose
```

Runs connectivity tests to Datadog intake endpoints, checks certificate validation, DNS resolution, and proxy configuration.

### Flare (support bundle)

```bash
sudo datadog-agent flare <case-id>
```

Collects logs, configuration (sanitized), status output, and system info into a tarball and uploads it to Datadog support.

## Integration Checks

### List all running checks

```bash
sudo datadog-agent status
```

The "Running Checks" section in the output lists all active checks, their run count, metric samples, and execution time.

### Run a single check (dry run)

```bash
# Run once and show output (doesn't submit to Datadog)
sudo datadog-agent check <check_name>

# Example
sudo datadog-agent check nginx
sudo datadog-agent check disk
sudo datadog-agent check cpu
```

### Run a check with debug-level rate info

```bash
sudo datadog-agent check <check_name> --check-rate
```

### Configuration for a check

Checks live in `/etc/datadog-agent/conf.d/<check_name>.d/conf.yaml`:

```bash
# List available check templates
ls /etc/datadog-agent/conf.d/

# Example: enable nginx check
sudo cp /etc/datadog-agent/conf.d/nginx.d/conf.yaml.example \
        /etc/datadog-agent/conf.d/nginx.d/conf.yaml
sudo vim /etc/datadog-agent/conf.d/nginx.d/conf.yaml
sudo systemctl restart datadog-agent
```

### Example check configuration (HTTP check)

```yaml
# /etc/datadog-agent/conf.d/http_check.d/conf.yaml
init_config:

instances:
  - name: My Website
    url: https://example.com
    timeout: 5
    http_response_status_code: 200
    tags:
      - env:production
      - service:web
```

## Log Collection

### Enable log collection

In `/etc/datadog-agent/datadog.yaml`:

```yaml
logs_enabled: true
```

### Configure a log source

Create `/etc/datadog-agent/conf.d/<integration>.d/conf.yaml`:

```yaml
# Example: collect custom application logs
# /etc/datadog-agent/conf.d/custom_app.d/conf.yaml
logs:
  - type: file
    path: /var/log/myapp/*.log
    service: myapp
    source: python
    tags:
      - env:production

  - type: file
    path: /var/log/myapp/error.log
    service: myapp
    source: python
    log_processing_rules:
      - type: multi_line
        name: java_stack_trace
        pattern: \d{4}-\d{2}-\d{2}
```

### Log collection types

| Type | Use Case |
|------|----------|
| `file` | Tail a log file |
| `journald` | Collect from systemd journal |
| `tcp` / `udp` | Listen on a network port |
| `docker` | Container logs (via socket) |

### journald collection

```yaml
logs:
  - type: journald
    container_mode: true
    include_units:
      - docker.service
      - kubelet.service
```

## DogStatsD

DogStatsD runs on UDP port `8125` by default. It accepts custom metrics from your applications.

### Test DogStatsD

```bash
# Send a counter metric
echo "custom.metric.count:1|c|#env:test" | nc -u -w1 127.0.0.1 8125

# Send a gauge
echo "custom.metric.gauge:42|g|#env:test,service:web" | nc -u -w1 127.0.0.1 8125

# Send a histogram
echo "custom.metric.request_time:320|h|#env:test" | nc -u -w1 127.0.0.1 8125
```

### DogStatsD metric types

| Type | Suffix | Description |
|------|--------|-------------|
| Counter | `c` | Tracks number of events |
| Gauge | `g` | Tracks a value at a point in time |
| Histogram | `h` | Tracks distribution of values |
| Distribution | `d` | Like histogram but computed server-side |
| Set | `s` | Counts unique elements |
| Timer | `ms` | Tracks execution time (alias for histogram) |

## Process Agent

### Enable full process collection

```yaml
# In datadog.yaml
process_config:
  process_collection:
    enabled: true
```

### Process agent status

```bash
sudo datadog-agent status
```

The process agent section appears in the standard status output when process collection is enabled.

## APM (Trace Agent)

The trace agent listens on port `8126` by default.

### Verify trace agent is running

```bash
curl http://localhost:8126/info
```

### Check trace agent config

```yaml
# In datadog.yaml
apm_config:
  enabled: true
  env: production
  # Limit resource names (reduce cardinality)
  max_traces_per_second: 10
  # Ignore certain resources
  ignore_resources:
    - "GET /health"
    - "GET /ready"
```

## Upgrades

### Upgrade to latest version

```bash
# Debian/Ubuntu
sudo apt-get update && sudo apt-get install --only-upgrade datadog-agent

# RHEL/CentOS/Amazon Linux
sudo yum clean expire-cache && sudo yum install datadog-agent
```

### Check current version

```bash
sudo datadog-agent version
```

## Uninstall

### Debian/Ubuntu

```bash
sudo apt-get remove datadog-agent -y
sudo apt-get purge datadog-agent -y    # also removes config files
```

### RHEL/CentOS

```bash
sudo yum remove datadog-agent
```

### Clean up residual files

```bash
sudo rm -rf /etc/datadog-agent
sudo rm -rf /var/log/datadog
sudo rm -rf /opt/datadog-agent
sudo userdel dd-agent
sudo groupdel dd-agent
```

## Troubleshooting

### Agent won't start

```bash
# Check for config syntax errors
sudo datadog-agent configcheck

# Run in foreground to see startup errors
sudo datadog-agent run

# Check journald logs
sudo journalctl -u datadog-agent -f
```

### Agent not submitting data

```bash
# Verify API key
sudo datadog-agent status | grep "API Key"

# Test connectivity to Datadog
sudo datadog-agent diagnose

# Check forwarder status (look for errors/retries in the Forwarder section)
sudo datadog-agent status | grep -A 20 "Forwarder"
```

### Check not running

```bash
# Validate check configuration
sudo datadog-agent configcheck

# Run the check manually with full debug output
sudo datadog-agent check <check_name> --log-level debug
```

### Permission issues

The agent runs as the `dd-agent` user. Common fixes:

```bash
# Grant log file access
sudo usermod -aG adm dd-agent           # Debian/Ubuntu (syslog group)
sudo chmod 644 /var/log/myapp/app.log    # Or fix file permissions

# Grant Docker socket access (for Docker check)
sudo usermod -aG docker dd-agent
sudo systemctl restart datadog-agent

# Grant journald access
sudo usermod -aG systemd-journal dd-agent
sudo systemctl restart datadog-agent
```

### High memory or CPU usage

```bash
# Check which integrations are consuming resources
sudo datadog-agent status

# Check number of custom metrics
sudo datadog-agent status | grep "Total Metrics"

# Reduce check frequency (default is 15s)
# In the check's conf.yaml:
# min_collection_interval: 60
```

### Debug log level

```bash
# Enable debug temporarily (no restart needed)
sudo datadog-agent config set log_level debug

# Reset back to info
sudo datadog-agent config set log_level info

# Or set permanently in datadog.yaml:
# log_level: debug
```

## Datadog API — Useful curl Commands

These commands require both an API key and an Application key. Set them as environment variables:

```bash
export DD_API_KEY="your-api-key"
export DD_APP_KEY="your-application-key"
export DD_SITE="https://api.datadoghq.com"   # adjust for your region
```

### List all dashboards

```bash
curl -s -X GET "${DD_SITE}/api/v1/dashboard" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.dashboards[] | {id, title}'
```

### List all monitors

```bash
curl -s -X GET "${DD_SITE}/api/v1/monitor" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.[] | {id, name, type, overall_state}'
```

### List monitors in alert state

```bash
curl -s -X GET "${DD_SITE}/api/v1/monitor?monitor_tags=&group_states=alert" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.[] | {id, name, overall_state}'
```

### Search hosts

```bash
# List all hosts
curl -s -X GET "${DD_SITE}/api/v1/hosts" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.host_list[] | {name: .host_name, apps, up: .is_up}'

# Filter by name
curl -s -X GET "${DD_SITE}/api/v1/hosts?filter=web-prod" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.host_list[].host_name'
```

### List active downtimes

```bash
curl -s -X GET "${DD_SITE}/api/v2/downtime" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.data[] | {id: .id, scope: .attributes.monitor_identifier, message: .attributes.message}'
```

### Mute a monitor

```bash
curl -s -X POST "${DD_SITE}/api/v1/monitor/<MONITOR_ID>/mute" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"end": 1700000000}'
```

### Send a custom event

```bash
curl -s -X POST "${DD_SITE}/api/v1/events" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Deployment completed",
    "text": "App v2.1.0 deployed to production",
    "priority": "normal",
    "tags": ["env:production", "service:web"],
    "alert_type": "info"
  }'
```

### Query a metric

```bash
# Get CPU usage for the last hour
NOW=$(date +%s)
FROM=$((NOW - 3600))

curl -s -X GET "${DD_SITE}/api/v1/query?from=${FROM}&to=${NOW}&query=avg:system.cpu.user{*}" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.series[0].pointlist[-5:]'
```

### Validate API key

```bash
curl -s -X GET "${DD_SITE}/api/v1/validate" \
  -H "DD-API-KEY: ${DD_API_KEY}" | jq .
```

### API endpoints by region

| Region | Base URL |
|--------|----------|
| US1 (default) | `https://api.datadoghq.com` |
| US3 | `https://api.us3.datadoghq.com` |
| US5 | `https://api.us5.datadoghq.com` |
| EU | `https://api.datadoghq.eu` |
| AP1 | `https://api.ap1.datadoghq.com` |

## Quick Reference Table

| Task | Command |
|------|---------|
| Start agent | `sudo systemctl start datadog-agent` |
| Stop agent | `sudo systemctl stop datadog-agent` |
| Restart agent | `sudo systemctl restart datadog-agent` |
| Agent status | `sudo datadog-agent status` |
| Agent version | `sudo datadog-agent version` |
| Run a check | `sudo datadog-agent check <name>` |
| List checks | `sudo datadog-agent status` (see Running Checks section) |
| Config validation | `sudo datadog-agent configcheck` |
| Connectivity test | `sudo datadog-agent diagnose` |
| Change log level | `sudo datadog-agent config set log_level debug` |
| View agent logs | `sudo tail -f /var/log/datadog/agent.log` |
| Send flare | `sudo datadog-agent flare <case-id>` |
| Show hostname | `sudo datadog-agent hostname` |
