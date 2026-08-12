<img src="/articles/images/datadog-logo.svg" alt="Datadog" width="150"/>

# Datadog API Reference

A practical reference for interacting with the Datadog API — authentication, client libraries, common endpoints, rate limits, and curl examples for monitors, dashboards, metrics, events, and more.

## Authentication

Every API request requires an **API key**. Some endpoints also require an **Application key**.

| Header | Purpose | Required For |
|--------|---------|-------------|
| `DD-API-KEY` | Identifies your organization | All endpoints |
| `DD-APPLICATION-KEY` | Identifies the user making the request (scoped permissions) | Endpoints that read or modify resources (monitors, dashboards, users) |

```bash
# Validate your API key
curl -s "https://api.datadoghq.com/api/v1/validate" \
  -H "DD-API-KEY: ${DD_API_KEY}" | jq '.'
```

### API Key vs Application Key

| Aspect | API Key | Application Key |
|--------|---------|-----------------|
| Scope | Organization-level | User-level (inherits user's permissions) |
| Used for | Submitting data (metrics, logs, events) | Reading/modifying resources (dashboards, monitors) |
| Can submit metrics | Yes | No (not by itself) |
| Can read dashboards | No (not by itself) | Yes (with API key) |
| OTR mode | N/A | One-Time Read — secret shown only at creation |

> **Note:** Application keys created after August 2025 have OTR (One-Time Read) mode enabled by default — the secret is shown only once at creation and cannot be retrieved later.

### Region-Specific Endpoints

| Region | API Endpoint |
|--------|-------------|
| US1 (default) | `https://api.datadoghq.com` |
| US3 | `https://api.us3.datadoghq.com` |
| US5 | `https://api.us5.datadoghq.com` |
| EU1 | `https://api.datadoghq.eu` |
| AP1 | `https://api.ap1.datadoghq.com` |
| US1-FED (GovCloud) | `https://api.ddog-gov.com` |

## Client Libraries

Official SDKs are available for most languages. These handle authentication, serialization, and rate limit retries.

### Python

```bash
# Legacy library (simple, for custom metrics and events)
pip install datadog

# Full API client (all endpoints, typed models)
pip3 install datadog-api-client
```

```python
# Legacy library usage
from datadog import initialize, api

initialize(api_key='<API_KEY>', app_key='<APP_KEY>')

# Query a metric
results = api.Metric.query(start=int(time.time()) - 3600, end=int(time.time()), query='avg:system.cpu.user{*}')

# Submit a custom metric
api.Metric.send(metric='my.custom.metric', points=42, tags=['env:production'])
```

```python
# Full API client usage
from datadog_api_client import Configuration, ApiClient
from datadog_api_client.v1.api.monitors_api import MonitorsApi

configuration = Configuration()
with ApiClient(configuration) as api_client:
    api_instance = MonitorsApi(api_client)
    monitors = api_instance.list_monitors()
    for m in monitors:
        print(f"{m.id}: {m.name} [{m.overall_state}]")
```

### Go

```bash
go mod init main && go get github.com/DataDog/datadog-api-client-go/v2/api/datadog
```

```go
import (
    "github.com/DataDog/datadog-api-client-go/v2/api/datadog"
    "github.com/DataDog/datadog-api-client-go/v2/api/datadogV1"
)
```

### JavaScript / TypeScript

```bash
npm install @datadog/datadog-api-client
# or
yarn add @datadog/datadog-api-client
```

```typescript
import { v1 } from '@datadog/datadog-api-client';

const configuration = v1.createConfiguration();
const apiInstance = new v1.MonitorsApi(configuration);

const monitors = await apiInstance.listMonitors();
```

### Ruby

```bash
gem install datadog_api_client -v 2.58.0
```

```ruby
require 'datadog_api_client'

DatadogAPIClient.configure do |config|
  config.server_variables[:site] = 'datadoghq.com'
end

api_instance = DatadogAPIClient::V1::MonitorsAPI.new
monitors = api_instance.list_monitors
```

### Java (Maven)

```xml
<dependency>
  <groupId>com.datadoghq</groupId>
  <artifactId>datadog-api-client</artifactId>
  <version>2.59.0</version>
  <scope>compile</scope>
</dependency>
```

### Rust

```bash
cargo add datadog-api-client
```

```rust
use datadog_api_client::datadog::Configuration;
use datadog_api_client::datadogV1::api_authentication::AuthenticationAPI;

#[tokio::main]
async fn main() {
    let configuration = Configuration::new();
    let api = AuthenticationAPI::with_config(configuration);
    let resp = api.validate().await;
    println!("{:#?}", resp);
}
```

## API Versions

| Version | Purpose |
|---------|---------|
| `/api/v1/` | Core functionality — monitors, dashboards, metrics, events, downtimes |
| `/api/v2/` | Newer endpoints — logs, security, incidents, usage, roles, API keys management |

Both versions are active. New features tend to land in v2 first, but v1 endpoints remain supported and are not deprecated.

## Common Endpoints

### Monitors

```bash
# List all monitors
curl -s "https://api.datadoghq.com/api/v1/monitors" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.[].name'

# Get a specific monitor
curl -s "https://api.datadoghq.com/api/v1/monitors/${MONITOR_ID}" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.'

# Create a monitor
curl -X POST "https://api.datadoghq.com/api/v1/monitors" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "High CPU on web hosts",
    "type": "metric alert",
    "query": "avg(last_5m):avg:system.cpu.user{service:web} by {host} > 90",
    "message": "CPU is above 90% on {{host.name}}. @slack-ops-alerts",
    "tags": ["team:platform", "env:production"],
    "options": {
      "thresholds": {
        "critical": 90,
        "warning": 80,
        "critical_recovery": 75
      },
      "notify_no_data": true,
      "no_data_timeframe": 10
    }
  }'

# Mute a monitor
curl -X POST "https://api.datadoghq.com/api/v1/monitor/${MONITOR_ID}/mute" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"end": 1700000000}'

# Delete a monitor
curl -X DELETE "https://api.datadoghq.com/api/v1/monitors/${MONITOR_ID}" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}"
```

### Dashboards

```bash
# List all dashboards
curl -s "https://api.datadoghq.com/api/v1/dashboard" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.dashboards[] | {id, title}'

# Get a dashboard definition
curl -s "https://api.datadoghq.com/api/v1/dashboard/${DASHBOARD_ID}" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.'

# Create a dashboard
curl -X POST "https://api.datadoghq.com/api/v1/dashboard" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Service Health",
    "layout_type": "ordered",
    "widgets": [
      {
        "definition": {
          "type": "timeseries",
          "requests": [{"q": "avg:system.cpu.user{*} by {host}", "display_type": "line"}],
          "title": "CPU Usage"
        }
      }
    ]
  }'

# Delete a dashboard
curl -X DELETE "https://api.datadoghq.com/api/v1/dashboard/${DASHBOARD_ID}" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}"
```

### Metrics

```bash
# Query a metric (last hour)
curl -s "https://api.datadoghq.com/api/v1/query" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  --data-urlencode "from=$(date -d '1 hour ago' +%s)" \
  --data-urlencode "to=$(date +%s)" \
  --data-urlencode "query=avg:system.cpu.user{env:production} by {host}"

# Submit a custom metric
curl -X POST "https://api.datadoghq.com/api/v2/series" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "series": [{
      "metric": "my.custom.metric",
      "type": 3,
      "points": [{"timestamp": '"$(date +%s)"', "value": 42.0}],
      "tags": ["env:production", "service:api"]
    }]
  }'

# List available metrics (search)
curl -s "https://api.datadoghq.com/api/v1/search?q=metrics:system.cpu" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.results.metrics'
```

> **Note:** The metrics query endpoint uses `--data-urlencode` for GET parameters. On macOS, replace `date -d '1 hour ago' +%s` with `date -v-1H +%s`.

### Events

```bash
# Post an event
curl -X POST "https://api.datadoghq.com/api/v1/events" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Deployment completed",
    "text": "Deployed v2.3.1 of web-service to production",
    "alert_type": "info",
    "tags": ["service:web", "env:production", "version:2.3.1"]
  }'

# Query events (last 24h)
curl -s "https://api.datadoghq.com/api/v1/events" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  --data-urlencode "start=$(date -d '24 hours ago' +%s)" \
  --data-urlencode "end=$(date +%s)" | jq '.events[] | {title, date_happened}'
```

### Downtimes

```bash
# Schedule a downtime (suppress alerts for 1 hour)
curl -X POST "https://api.datadoghq.com/api/v1/downtime" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "scope": ["env:production", "service:web"],
    "start": '"$(date +%s)"',
    "end": '"$(date -d '+1 hour' +%s)"',
    "message": "Maintenance window for web service deployment"
  }'

# List active downtimes
curl -s "https://api.datadoghq.com/api/v1/downtime" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.[] | {id, scope, message}'

# Cancel a downtime
curl -X DELETE "https://api.datadoghq.com/api/v1/downtime/${DOWNTIME_ID}" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}"
```

### Tags

```bash
# Get tags for a host
curl -s "https://api.datadoghq.com/api/v1/tags/hosts/${HOSTNAME}" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.tags'

# Add tags to a host
curl -X POST "https://api.datadoghq.com/api/v1/tags/hosts/${HOSTNAME}" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"tags": ["team:platform", "role:webserver"]}'
```

## Rate Limits

When you exceed the allowed request rate, the API returns HTTP 429 with headers indicating when to retry.

### Rate Limit Headers

| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Requests allowed in the current time period |
| `X-RateLimit-Period` | Time period length in seconds (calendar aligned) |
| `X-RateLimit-Remaining` | Requests remaining in the current period |
| `X-RateLimit-Reset` | Seconds until the limit resets |
| `X-RateLimit-Name` | Name of the rate limit bucket (use when requesting increases) |

### Default Limits

| Endpoint | Rate Limit |
|----------|-----------|
| Event submission | 250,000 events/minute per org |
| Metrics submission | Not rate limited (billed by volume) |
| Log submission | Not rate limited (billed by volume) |
| Monitor CRUD | Varies per endpoint |
| Dashboard CRUD | Varies per endpoint |
| Query endpoints | Varies per endpoint |

> **Note:** Rate limits can be increased by contacting Datadog support. Provide the `X-RateLimit-Name` value, your use case, and the desired target limit.

### Rate Limit Scopes

Different endpoints can be rate limited at different levels:

| Scope | Description |
|-------|-------------|
| Per organization | Total requests across all users and keys in the org |
| Per user (UUID) | Requests from a specific user account |
| Per API key | Requests using a specific application key |

Multiple endpoints can share the same rate limit bucket (identified by `X-RateLimit-Name`).

### API Usage Metrics

Datadog exposes rate limit consumption as metrics so you can monitor and alert on your own API usage:

| Metric | Scope | Description |
|--------|-------|-------------|
| `datadog.apis.usage.per_org` | Organization | Number of API requests to a specific endpoint |
| `datadog.apis.usage.per_org_ratio` | Organization | Ratio of requests to allowed limit |
| `datadog.apis.usage.per_user` | User UUID | Requests per unique user |
| `datadog.apis.usage.per_user_ratio` | User UUID | Ratio of user requests to limit |
| `datadog.apis.usage.per_api_key` | API key | Requests per unique application key |
| `datadog.apis.usage.per_api_key_ratio` | API key | Ratio of key requests to limit |

Available tags on these metrics:

| Tag | Description |
|-----|-------------|
| `app_key_id` | Application key ID used by the API client |
| `limit_name` | Name of the rate limit bucket |
| `limit_count` | Total requests allowed per period |
| `limit_period` | Reset period in seconds |
| `rate_limit_status` | `passed` (allowed) or `blocked` (429'd) |
| `user_uuid` | UUID of the user making the request |
| `child_org` | Child org name (parent org view only) |

### Monitoring Your API Usage

```
# Dashboard query: Total API requests by endpoint (roll up per minute)
sum:datadog.apis.usage.per_org{*} by {limit_name}.rollup(sum, 60)

# Monitor query: Alert when blocked requests occur
sum:datadog.apis.usage.per_org{rate_limit_status:blocked} by {limit_name}

# Who is getting rate limited?
sum:datadog.apis.usage.per_user{rate_limit_status:blocked} by {user_uuid,limit_name}

# Which app key is consuming the most?
sum:datadog.apis.usage.per_api_key{*} by {app_key_id,limit_name}
```

### Requesting a Rate Limit Increase

Open a support ticket with the following details:

```
Title:
    Request to increase rate limit on endpoint: <endpoint>

Details:
    We would like to request a rate limit increase for API endpoint: <endpoint>
    Example use cases/queries:
        <example API call as cURL or URL with payload>

    Motivation for increasing rate limit:
        <why you need more requests — e.g., deployment automation runs Y times/day>

    Desired target rate limit:
        <specific number or percentage increase>
```

Provide the `X-RateLimit-Name` from the response headers to help support identify the correct bucket.

### Handling Rate Limits in Scripts

```bash
#!/bin/bash
# Retry with exponential backoff on 429
call_api() {
  local url="$1"
  local retries=3
  local wait=5

  for i in $(seq 1 $retries); do
    response=$(curl -s -w "\n%{http_code}" "$url" \
      -H "DD-API-KEY: ${DD_API_KEY}" \
      -H "DD-APPLICATION-KEY: ${DD_APP_KEY}")

    http_code=$(echo "$response" | tail -1)
    body=$(echo "$response" | sed '$d')

    if [ "$http_code" -eq 429 ]; then
      echo "Rate limited. Waiting ${wait}s before retry $i/$retries..." >&2
      sleep $wait
      wait=$((wait * 2))
    else
      echo "$body"
      return 0
    fi
  done
  echo "Failed after $retries retries" >&2
  return 1
}
```

### Audit Trail for API Activity

For additional visibility beyond rate limit metrics, Datadog's Audit Trail provides:

- **IP address and geolocation** — identify where API requests originated
- **Actor type** — distinguish between service accounts and user accounts
- **Authentication method** — API key vs application key vs browser session
- **Correlated events** — configuration changes or security events at the same time

Query audit events via API:

```bash
curl -s "https://api.datadoghq.com/api/v2/audit/events" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  --data-urlencode "filter[query]=@type:audit" \
  --data-urlencode "filter[from]=now-1h" \
  --data-urlencode "page[limit]=25" | jq '.data[].attributes | {timestamp, type: .evt.name}'
```

## Environment Variables

Client libraries auto-detect credentials from environment variables:

```bash
export DD_API_KEY="your-api-key"
export DD_APP_KEY="your-application-key"
export DD_SITE="datadoghq.com"    # or datadoghq.eu, us3.datadoghq.com, etc.
```

When these are set, the Python, Go, Ruby, and JS clients work without explicit configuration:

```python
# No need to pass keys — picks them up from DD_API_KEY / DD_APP_KEY
from datadog_api_client import Configuration, ApiClient
configuration = Configuration()
```

## Useful Patterns

### Export All Monitors to JSON

```bash
#!/bin/bash
# Export all monitors as individual JSON files
curl -s "https://api.datadoghq.com/api/v1/monitors" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | \
  jq -c '.[]' | while read -r monitor; do
    id=$(echo "$monitor" | jq -r '.id')
    name=$(echo "$monitor" | jq -r '.name' | tr ' /' '_-')
    echo "$monitor" | jq '.' > "monitor_${id}_${name}.json"
    echo "Exported: $name ($id)"
  done
```

### Bulk Mute Monitors by Tag

```bash
#!/bin/bash
# Mute all monitors tagged with team:platform
END_TIME=$(date -d '+2 hours' +%s)  # macOS: date -v+2H +%s

curl -s "https://api.datadoghq.com/api/v1/monitors" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  --data-urlencode "monitor_tags=team:platform" | \
  jq -r '.[].id' | while read -r id; do
    curl -X POST "https://api.datadoghq.com/api/v1/monitor/${id}/mute" \
      -H "DD-API-KEY: ${DD_API_KEY}" \
      -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
      -H "Content-Type: application/json" \
      -d "{\"end\": ${END_TIME}}"
    echo "Muted monitor $id"
  done
```

### Check Monitor Status Programmatically

```bash
# Get all monitors in ALERT state
curl -s "https://api.datadoghq.com/api/v1/monitors" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | \
  jq '[.[] | select(.overall_state == "Alert")] | .[] | {id, name, overall_state}'
```

### Submit a Deployment Event from CI/CD

```bash
# Use in GitHub Actions, GitLab CI, Jenkins, etc.
curl -X POST "https://api.datadoghq.com/api/v1/events" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Deploy: '"${SERVICE_NAME}"' '"${VERSION}"'",
    "text": "Deployed by '"${CI_USER}"' from '"${CI_PIPELINE_URL}"'",
    "alert_type": "info",
    "source_type_name": "deployment",
    "tags": [
      "service:'"${SERVICE_NAME}"'",
      "version:'"${VERSION}"'",
      "env:'"${ENVIRONMENT}"'"
    ]
  }'
```

## API Use Cases by Category

The Datadog API is organized into three broad use cases: sending data in, visualizing data, and managing your account.

### Send Data to Datadog

| Category | Endpoints | Purpose |
|----------|-----------|---------|
| **Metrics** | `/api/v1/series`, `/api/v2/series` | Submit custom metrics, query time series |
| **Events** | `/api/v1/events` | Post deployment events, configuration changes, incidents |
| **Logs** | `/api/v2/logs` | Submit log entries directly (bypass Agent) |
| **Traces** | Tracing Agent API (port 8126) | Submit APM traces via the local Agent |
| **Synthetics** | `/api/v1/synthetics/tests` | Create/manage synthetic browser and API tests |
| **Service Checks** | `/api/v1/check_run` | Post custom check statuses for monitors |

### Integration Endpoints

| Integration | Endpoint |
|-------------|----------|
| AWS | `/api/v1/integration/aws` |
| Azure | `/api/v1/integration/azure` |
| GCP | `/api/v1/integration/gcp` |
| Slack | `/api/v1/integration/slack` |
| PagerDuty | `/api/v1/integration/pagerduty` |
| Webhooks | `/api/v1/integration/webhooks` |
| Cloudflare | `/api/v2/integration/cloudflare` |
| Fastly | `/api/v2/integration/fastly` |
| Jira | `/api/v2/integration/jira` |
| Microsoft Teams | `/api/v2/integration/ms-teams` |
| Okta | `/api/v2/integration/okta` |
| Opsgenie | `/api/v2/integration/opsgenie` |

### Visualize Data

| Category | Endpoints | Purpose |
|----------|-----------|---------|
| **Dashboards** | `/api/v1/dashboard` | Create, update, delete, list dashboards |
| **Dashboard Lists** | `/api/v2/dashboard/lists/manual` | Organize dashboards into lists |
| **Embeddable Graphs** | `/api/v1/graph/embed` | Generate embeddable graph iframes |
| **Graph Snapshots** | `/api/v1/graph/snapshot` | Capture a point-in-time PNG of a graph |
| **Host Tags** | `/api/v1/tags/hosts` | Manage tags that organize infrastructure views |
| **Service Dependencies** | `/api/v1/service_dependencies` | Map APM service relationships |

### Manage Your Account

| Category | Endpoints | Purpose |
|----------|-----------|---------|
| **Users** | `/api/v2/users` | Create, list, disable user accounts |
| **Roles** | `/api/v2/roles` | Manage RBAC roles and permissions |
| **Organizations** | `/api/v1/org` | Manage org settings, child orgs |
| **API/App Keys** | `/api/v2/api_keys`, `/api/v2/application_keys` | Rotate and manage credentials |
| **Authentication** | `/api/v1/validate` | Verify key validity |
| **IP Ranges** | `/api/v1/ip_ranges` | List Datadog IP prefixes (for firewall allowlisting) |
| **Usage Metering** | `/api/v2/usage` | Hourly, daily, monthly usage across products |
| **Logs Restriction Queries** | `/api/v2/logs/config/restriction_queries` | Grant granular logs access |
| **Audit Trail** | `/api/v2/audit/events` | Query API activity, config changes, security events |

### Monitors and Alerting

| Category | Endpoints | Purpose |
|----------|-----------|---------|
| **Monitors** | `/api/v1/monitor` | Create, update, mute, delete, search monitors |
| **Downtimes** | `/api/v1/downtime` | Schedule maintenance windows |
| **SLOs** | `/api/v1/slo` | Create and manage Service Level Objectives |
| **Service Checks** | `/api/v1/check_run` | Post check results for monitor evaluation |
| **Security Signals** | `/api/v2/security_monitoring/signals` | Query security monitoring detections |

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Authentication | API key (`DD-API-KEY`) for all requests. Add Application key (`DD-APPLICATION-KEY`) for read/write operations. |
| Regions | Use the correct base URL for your Datadog site (US1, US3, EU1, etc.) |
| Versions | v1 for core CRUD (monitors, dashboards, metrics). v2 for newer features (logs, security, roles). |
| Rate limits | 429 response = back off. Check `X-RateLimit-Reset` header. Metrics/logs are not rate limited. |
| Client SDKs | Available for Python, Go, JS/TS, Ruby, Java, Rust. Auto-read `DD_API_KEY` / `DD_APP_KEY` from env. |
| Best practice | Store keys in env vars or secrets managers. Never commit them to version control. |
