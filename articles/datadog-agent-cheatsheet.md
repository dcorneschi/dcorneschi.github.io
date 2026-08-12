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

## Pup CLI

Pup is Datadog's official CLI — a single Rust binary with 325+ commands across 57 product domains. It replaces the legacy Dogshell tool.

### Installation

```bash
# macOS/Linux (Homebrew)
brew tap datadog-labs/pack
brew install datadog-labs/pack/pup

# Build from source
git clone https://github.com/DataDog/pup.git && cd pup
cargo build --release
cp target/release/pup /usr/local/bin/pup
```

### Authentication

```bash
# OAuth2 login (preferred — scoped access, no long-lived keys)
pup auth login

# Check auth status
pup auth status

# Test connection
pup auth test

# Logout
pup auth logout

# Refresh token
pup auth refresh

# Login to a specific site
pup auth login --site datadoghq.eu

# Login with a named org profile (for multi-account)
pup auth login --org staging-child

# List all stored sessions
pup auth list

# Fallback: use API keys (set environment variables)
export DD_API_KEY="your-api-key"
export DD_APP_KEY="your-app-key"
export DD_SITE="datadoghq.com"
```

### Command Structure

```bash
pup <domain> <action> [options]             # Simple commands
pup <domain> <subgroup> <action> [options]  # Nested commands
```

### Global Flags

| Flag | Description |
|------|-------------|
| `-o, --output` | Output format: `json`, `yaml`, `table` (default: json) |
| `--jq` | Filter/transform output with a jq expression |
| `-y, --yes` | Skip confirmation prompts |
| `--read-only` | Block all write operations |
| `--agent` | Enable agent mode (structured JSON output) |
| `--no-agent` | Disable agent mode |
| `--verbose` | Enable verbose logging |
| `--org <org>` | Use a named org profile for multi-account workflows |
| `--site` | Datadog site (only for `auth login` and `auth status`) |
| `--config` | Config file path (default: `~/.config/pup/config.yaml`) |

### Monitors

```bash
# List all monitors
pup monitors list

# List monitors filtered by tag
pup monitors list --tags="team:backend"
pup monitors list --tags="env:production"

# Get a specific monitor
pup monitors get 12345678

# Search monitors with query
pup monitors search --query="tag:(service:web-store AND team:backend)"

# Create a monitor from file
pup monitors create --file=monitor.json

# Update a monitor
pup monitors update 12345678 --file=monitor.json

# Diff monitor (compare local definition vs remote)
pup monitors diff 12345678

# Delete a monitor
pup monitors delete 12345678 --yes
```

### Logs

```bash
# Search logs for errors in the last hour
pup logs search --query="status:error" --from="1h"

# Search logs for a specific service over 7 days
pup logs search --query="service:api" --from="7d" --storage="flex"

# Search with a specific index
pup logs query --query="service:api" --index="main,security" --from="1h"

# Aggregate logs
pup logs aggregate --query="status:error" --from="1h"

# Manage saved views
pup logs saved-views list
pup logs saved-views create --file=saved-view.json
```

### Metrics

```bash
# Query CPU metrics for the last hour (v2 API — time-series data)
pup metrics query --query="avg:system.cpu.user{*}" --from="1h"

# Search metrics using classic query syntax (v1 API)
pup metrics search --query="avg:system.cpu.user{*}" --from="1h"

# List available metrics (with filter)
pup metrics list --filter="system.*"

# List tags for a metric
pup metrics tags list system.cpu.user --window-seconds=3600

# Submit time-series data
pup metrics timeseries --file=request.json
```

### Dashboards

```bash
# List all dashboards
pup dashboards list

# Get a specific dashboard
pup dashboards get abc-def-123

# Get a live dashboard URL with time range
pup dashboards url abc-def-123 --from=now-1w --to=now --live=true

# Delete a dashboard
pup dashboards delete abc-def-123 --yes
```

### Events

```bash
# Post a custom event (positional: title, then text)
pup events post --tags="version:1,application:web" --no_host --alert_type=info \
  "Deployment completed" "App v2.1.0 deployed to production"

# Search events
pup events search --query="@user.id:12345"

# List recent events
pup events list

# Get a specific event
pup events get <event-id>
```

### SLOs

```bash
# List all SLOs
pup slos list

# Get SLO details
pup slos get abc-123-def

# Check SLO status
pup slos status abc-123-def

# Delete an SLO
pup slos delete abc-123-def --yes
```

### Infrastructure & Hosts

```bash
# List hosts
pup infrastructure hosts list

# Get host details
pup infrastructure hosts get my-hostname

# Manage host tags
pup tags list my-hostname
pup tags get my-hostname
pup tags add my-hostname --tags="env:prod,team:backend"
pup tags update my-hostname --tags="env:staging"
pup tags delete my-hostname

# List containers
pup containers list

# List container images
pup containers images list

# List processes
pup processes list
```

### Downtime

```bash
# List active downtimes
pup downtime list

# Get downtime details
pup downtime get 12345

# Cancel a downtime
pup downtime cancel 12345
```

### Incidents

```bash
# List incidents
pup incidents list

# Get incident details
pup incidents get abc-123
```

### Synthetics

```bash
# List synthetic tests
pup synthetics tests list

# List available locations
pup synthetics locations list

# List test suites
pup synthetics suites search
```

### Security

```bash
# List security rules
pup security rules list

# List security signals
pup security signals list

# List security findings
pup security findings list

# List content packs
pup security content-packs list

# Audit logs
pup audit-logs list
pup audit-logs search --query="@action:monitor.modified"
```

### APM & Traces

```bash
# List APM services
pup apm services list

# Get service stats
pup apm services stats my-service

# List service operations
pup apm services operations my-service

# List service resources
pup apm services resources my-service

# List service dependencies
pup apm dependencies list

# Trace metrics (span-based metric definitions)
pup traces metrics list
pup traces metrics get <metric-id>
```

### On-Call

```bash
# List on-call teams
pup on-call teams list

# Get team details
pup on-call teams get <team-id>

# List pages (newest first)
pup on-call pages list

# Get a specific page
pup on-call pages get <page-id>

# Create a page
pup on-call pages create --file=page.json
```

### CI/CD

```bash
# List CI pipelines
pup cicd pipelines list

# List CI events
pup cicd events list

# List test results
pup cicd tests list

# List flaky tests
pup cicd flaky-tests list

# DORA metrics
pup cicd dora list
```

### Database Monitoring

```bash
# Search DBM query samples
pup dbm samples search --query="dbm_type:activity service:orders env:prod" --from="1h" --limit=10
```

### Runbooks

```bash
# List available runbooks
pup runbooks list

# Inspect a runbook's steps
pup runbooks describe incident-triage

# Run a runbook with variables
pup runbooks run deploy-service --arg SERVICE=payments --arg VERSION=1.2.3

# Dry-run (show steps without executing)
pup runbooks run deploy-service --dry-run

# Import a runbook from file
pup runbooks import ./my-runbook.yaml

# Validate a runbook file
pup runbooks validate ./my-runbook.yaml
```

### Workflows

```bash
# Get a workflow
pup workflows get <workflow-id>

# Create a workflow from file
pup workflows create --file=workflow.json

# Run a workflow
pup workflows run <workflow-id>

# List workflow instances
pup workflows instances list <workflow-id>

# Diff workflow (compare local vs remote)
pup workflows diff <workflow-id>
```

### Key & User Management

```bash
# List API keys
pup api-keys list

# Create an API key
pup api-keys create --name="my-new-key"

# List application keys
pup app-keys list

# List users
pup users list

# List organizations
pup organizations list
```

### Skills (AI Agent Integration)

```bash
# List available skills and agents
pup skills list

# Install skills for a specific platform
pup skills install claude
pup skills install cursor
pup skills install codex

# Install for all platforms
pup skills install all

# Install project-local (instead of user-global)
pup skills install claude --project

# Install a specific skill by name
pup skills install claude --name dd-monitors
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `DD_ACCESS_TOKEN` | Bearer token for stateless auth (highest priority) |
| `DD_API_KEY` | Datadog API key (optional if using OAuth2) |
| `DD_APP_KEY` | Datadog application key (optional if using OAuth2) |
| `DD_SITE` | Datadog site (default: `datadoghq.com`) |
| `DD_AUTO_APPROVE` | Auto-approve destructive operations (`true`/`false`) |
| `DD_TOKEN_STORAGE` | Token storage backend (`keychain` or `file`) |
| `FORCE_AGENT_MODE` | Force agent mode for AI agent workflows (`1`/`true`) |

### Using --jq for Filtering

```bash
# Extract only monitor names
pup monitors list --jq '.[].name'

# Monitor names and IDs as a table
pup monitors list --jq '.[] | {name, id, overall_state}' -o table

# Filter monitors by name pattern
pup monitors list --jq '.[] | select(.name | endswith("prod"))' -o table

# Count error logs
pup logs search --query="status:error" --jq '.data | length'

# Get just host names
pup infrastructure hosts list --jq '.[].host_name'

# Combine jq with table output
pup slos list --jq '.[] | select(.name | contains("api"))' -o table

# Get monitor IDs only (useful for scripting)
pup monitors list --tags="team:backend" --jq '.[].id'

# Extract names and states for alerting monitors
pup monitors list --jq '.[] | select(.overall_state == "Alert") | {name, id}'
```

### Service Catalog

```bash
# List services in the catalog
pup service-catalog list

# Get service details
pup service-catalog get my-service

# IDP agent commands (Service Catalog agent access)
pup idp assist my-service     # Full context: owner, on-call, health, dependencies
pup idp find "payment"        # Search entities by name (defaults to kind:service)
pup idp owner my-service      # Ownership + on-call responders
pup idp deps my-service       # Upstream/downstream dependencies
pup idp register ./service.datadog.yaml  # Register a service definition
```

## Dogshell (Legacy — Deprecated)

Dogshell is the legacy Python CLI bundled with `datadogpy`. It has been replaced by Pup CLI but still works for basic operations.

### Installation

```bash
pip install datadog
```

### Configuration

Create `~/.dogrc`:

```ini
[Connection]
apikey = MY_API_KEY
appkey = MY_APP_KEY
api_host = https://api.datadoghq.com
```

Or run any `dog` command — it prompts for credentials on first use.

### Metrics

```bash
# Post a metric
dog metric post test_metric 1.0 --tags "env:test,service:web"

# Post multiple tags
dog metric post cpu.usage 85.5 --tags "host:web-01,env:production,team:backend"
```

### Events

```bash
# Post an event
dog event post "Deployment started" "Deploying version 2.1.0 to production" \
  --tags "env:production,service:web" --type "deploy"
```

### Monitors

```bash
# List all monitors
dog monitor show_all

# Show a specific monitor
dog monitor show <MONITOR_ID>

# Mute a monitor
dog monitor mute <MONITOR_ID>

# Unmute a monitor
dog monitor unmute <MONITOR_ID>

# Mute all monitors
dog monitor mute_all
```

### Service Checks

```bash
# Submit a service check
dog service_check check my_app.health 0 --tags "env:production"
# Status: 0=OK, 1=WARNING, 2=CRITICAL, 3=UNKNOWN
```

### Downtimes

```bash
# Schedule a downtime
dog downtime post 'env:production' --start $(date +%s) --end $(($(date +%s) + 3600))
```

### Available Commands

```bash
dog -h                # Full list of commands
dog metric -h         # Metric subcommands
dog event -h          # Event subcommands
dog monitor -h        # Monitor subcommands
dog downtime -h       # Downtime subcommands
dog service_check -h  # Service check subcommands
dog tag -h            # Tag subcommands
dog host -h           # Host subcommands
dog search -h         # Search subcommands
dog comment -h        # Comment subcommands
```

## Dogwrap

Dogwrap wraps shell commands and generates Datadog events based on their exit code. Useful for monitoring cron jobs, batch scripts, and one-off commands.

### Installation

```bash
pip install datadog
```

### Basic Usage

```bash
dogwrap -n "<EVENT_TITLE>" -k <DATADOG_API_KEY> "<COMMAND>"
```

### Send Events on Errors Only

```bash
# Post an event only if the command exits non-zero
dogwrap -n "DB Vacuum" -k $DD_API_KEY --submit_mode errors \
  "psql -c 'vacuum verbose my_table' 2>&1"
```

### Send Events on Every Run

```bash
# Post an event for every execution (success or failure)
dogwrap -n "Nightly Backup" -k $DD_API_KEY --submit_mode all \
  "/usr/local/bin/backup.sh"
```

### Use with Cron

```bash
# crontab entry — wrap a cron job to get Datadog events
0 0 * * * dogwrap -n "Vacuum mytable" -k $DD_API_KEY --submit_mode errors "psql -c 'vacuum verbose my_table' 2>&1 >> /var/log/postgres_vacuums.log"
```

### Target a Specific Site

```bash
# Send to EU site
dogwrap -n "Backup" -k $DD_API_KEY -s eu --submit_mode all "/usr/local/bin/backup.sh"

# Available sites: us3, us5, eu, ap1
```

### Submit Modes

| Mode | Sends Event When |
|------|------------------|
| `errors` | Command exits with non-zero code |
| `all` | Every run, regardless of exit code |

## DogStatsD Shell Usage

Send metrics, events, and service checks directly from the shell via UDP to the DogStatsD daemon (port 8125).

### Datagram Format

```
<METRIC_NAME>:<VALUE>|<TYPE>|@<SAMPLE_RATE>|#<TAG_KEY_1>:<TAG_VALUE_1>,<TAG_2>
```

### Metric Types

| Type | Code | Description | Example |
|------|------|-------------|---------|
| Count | `c` | Increment/decrement a counter | `page.views:1\|c` |
| Gauge | `g` | Record a value at a point in time | `fuel.level:0.5\|g` |
| Histogram | `h` | Track distribution of values | `request.time:320\|h` |
| Distribution | `d` | Server-side histogram | `page.views:15\|d` |
| Timer | `ms` | Execution time (alias for histogram) | `db.query:12\|ms` |
| Set | `s` | Count unique elements | `users.uniques:1234\|s` |

### Send Metrics via Shell

```bash
# Counter — increment page views
echo "page.views:1|c|#env:production,service:web" | nc -u -w1 127.0.0.1 8125

# Gauge — record current queue depth
echo "queue.depth:42|g|#env:production" | nc -u -w1 127.0.0.1 8125

# Histogram — record request duration
echo "request.duration:320|h|#service:api,env:prod" | nc -u -w1 127.0.0.1 8125

# Distribution — record page load time
echo "page.load:1.5|d|#env:production" | nc -u -w1 127.0.0.1 8125

# Set — track unique visitors
echo "users.uniques:user123|s|#env:production" | nc -u -w1 127.0.0.1 8125

# Counter with sample rate (50%)
echo "requests:1|c|@0.5|#env:prod,country:us" | nc -u -w1 127.0.0.1 8125
```

### Value Packing (v1.1 — Agent 6.25+/7.25+)

Send multiple values in a single datagram for histograms and distributions:

```bash
# Multiple values separated by colons
echo "page.views:1:2:32|d|#env:prod" | nc -u -w1 127.0.0.1 8125
echo "request.time:120:340:250|h|@0.5|#service:api" | nc -u -w1 127.0.0.1 8125
```

### Send Events via DogStatsD

Format: `_e{<TITLE_LENGTH>,<TEXT_LENGTH>}:<TITLE>|<TEXT>|d:<TIMESTAMP>|h:<HOSTNAME>|p:<PRIORITY>|t:<ALERT_TYPE>|#<TAGS>`

```bash
# Send an error event
echo '_e{21,36}:An exception occurred|Cannot parse CSV file from 10.0.0.17|t:warning|#err_type:bad_file' | nc -u -w1 127.0.0.1 8125

# Event with newline in text
echo '_e{21,42}:An exception occurred|Cannot parse JSON request:\\n{"foo: "bar"}|p:low|#err_type:bad_request' | nc -u -w1 127.0.0.1 8125

# Deployment event
echo '_e{19,28}:Deployment complete|App v2.1.0 deployed to prod|t:info|#env:prod,service:web' | nc -u -w1 127.0.0.1 8125
```

### Send Service Checks via DogStatsD

Format: `_sc|<NAME>|<STATUS>|d:<TIMESTAMP>|h:<HOSTNAME>|#<TAGS>|m:<MESSAGE>`

Status codes: `0` = OK, `1` = WARNING, `2` = CRITICAL, `3` = UNKNOWN

```bash
# Service check — OK
echo '_sc|Redis connection|0|#env:prod|m:Connection successful' | nc -u -w1 127.0.0.1 8125

# Service check — CRITICAL
echo '_sc|Redis connection|2|#env:dev|m:Redis connection timed out after 10s' | nc -u -w1 127.0.0.1 8125

# Service check — WARNING with timestamp
echo "_sc|Disk space|1|d:$(date +%s)|#host:web-01|m:Disk at 85% capacity" | nc -u -w1 127.0.0.1 8125
```

### DogStatsD over Unix Socket

If configured to use UDS instead of UDP:

```bash
# Send via Unix socket
echo "page.views:1|c|#env:prod" | socat - UNIX-CONNECT:/var/run/datadog/dsd.socket
```

Configure in `datadog.yaml`:

```yaml
dogstatsd_socket: /var/run/datadog/dsd.socket
```

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
| Pup: login | `pup auth login` |
| Pup: list monitors | `pup monitors list --tags="env:production"` |
| Pup: search logs | `pup logs search --query="status:error" --from="1h"` |
| Pup: query metrics | `pup metrics query --query="avg:system.cpu.user{*}" --from="1h"` |
| Pup: list hosts | `pup infrastructure hosts list` |
| Pup: run runbook | `pup runbooks run <name> --arg KEY=VALUE` |
| Dogshell: post metric | `dog metric post my_metric 1.0 --tags "env:test"` |
| Dogwrap: wrap command | `dogwrap -n "Title" -k $DD_API_KEY --submit_mode errors "cmd"` |
| DogStatsD: send counter | `echo "metric:1\|c\|#tag:val" \| nc -u -w1 127.0.0.1 8125` |
