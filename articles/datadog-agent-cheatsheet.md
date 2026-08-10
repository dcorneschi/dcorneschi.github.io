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
