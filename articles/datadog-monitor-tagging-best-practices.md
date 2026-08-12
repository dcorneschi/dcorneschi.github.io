<img src="/articles/images/datadog-logo.svg" alt="Datadog" width="150"/>

# Datadog Monitor Tagging Best Practices

Guide to tagging Datadog monitors effectively. Covers recommended tag strategies, filtering and searching monitors, scheduling downtime by tag, enriching dashboards, and organizing SLOs through inherited monitor tags.

## Why Tag Your Monitors

Monitor tags add dimensions that let you filter, aggregate, and visualize monitors the same way you would metrics, logs, or traces. Well-tagged monitors provide immediate context:

- **Who** owns this monitor (team)
- **What** service or resource it covers
- **Where** it runs (environment)
- **How critical** it is (severity/priority)
- **Why** it exists (SLI type, compliance, etc.)

Anyone in the organization can look at a monitor's tags and instantly understand its purpose without reading the query or message body.

## Recommended Tags

At a minimum, every monitor should carry these tags:

| Tag | Purpose | Example |
|-----|---------|---------|
| `team:<name>` | Ownership and routing | `team:backend` |
| `service:<name>` | Which service is monitored | `service:web-store` |
| `env:<name>` | Environment scope | `env:production` |

Beyond the minimum, consider adding:

| Tag | Purpose | Example |
|-----|---------|---------|
| `severity:<level>` | Priority classification | `severity:high`, `severity:p1` |
| `resource_name:<endpoint>` | Specific APM endpoint | `resource_name:checkout_controller` |
| `sli:<type>` | SLI category for SLO use | `sli:latency`, `sli:availability` |

### Key:Value vs Keyless Tags

Datadog allows keyless tags (just a value, no key). These can describe something unique about a specific monitor — for example, tagging a monitor with `test` during experimentation. However, prefer `key:value` format wherever possible because:

- Key:value tags are easier to search and filter
- They group naturally (all `team:*` values together)
- They scale better across organizations
- They're harder to create inconsistently

### Example: Fully Tagged APM Monitor

A well-tagged APM monitor might carry:

```
service:web-store
env:production
resource_name:shoppingcartcontroller_checkout
severity:high
team:backend
sli:throughput
```

## Filtering Monitors by Tag

### In the UI

On the Manage Monitors page, use tag facets in your search query:

```
tag:service:web-store tag:team:backend tag:severity:high
```

Boolean logic is supported:

```
tag:service:web-store AND tag:team:backend
```

For keyless tags:

```
tag:test
```

### With the API

Search monitors programmatically using the Monitors API:

```python
from datadog import initialize, api

options = {
    'api_key': '<DATADOG_API_KEY>',
    'app_key': '<DATADOG_APPLICATION_KEY>'
}
initialize(**options)

api.Monitor.search(
    query="tag:(service:web-store AND team:backend AND severity:high)"
)
```

### With Terraform

Use the `tags` argument in the `datadog_monitor` resource:

```hcl
resource "datadog_monitor" "checkout_throughput" {
  name    = "High error rate on checkout"
  type    = "query alert"
  query   = "avg(last_5m):avg:trace.servlet.request.errors{service:web-store} > 0.05"
  message = "Checkout error rate is elevated. @slack-backend-alerts"

  tags = [
    "service:web-store",
    "env:production",
    "team:backend",
    "severity:high",
    "sli:availability",
  ]
}
```

## Filtering Events by Monitor Tags

When monitors trigger or recover, Datadog creates events. Filter these in the event stream:

```
sources:alert tag:team:backend tag:service:web-store
```

Programmatic event search:

```python
from datadog import initialize, api
import time

options = {
    'api_key': '<DATADOG_API_KEY>',
    'app_key': '<DATADOG_APPLICATION_KEY>'
}
initialize(**options)

end_time = time.time()
start_time = end_time - 3600  # last hour

api.Event.query(
    start=start_time,
    end=end_time,
    sources=["alert"],
    tags=["team:backend", "service:web-store"],
    unaggregated=True
)
```

## Scheduling Downtime by Tag

During maintenance windows, you can suppress monitor notifications using tags instead of manually listing individual monitors.

### In the UI

Navigate to Monitors > Manage Downtime > Schedule Downtime, then enter tags in the monitor tags field:

```
team:backend
service:web-store
env:production
```

All monitors matching those tags will be muted for the specified window. The monitors still transition to ALERT/WARN states — only notifications are suppressed.

### With the API

```python
from datadog import initialize, api
import time

options = {
    'api_key': '<DATADOG_API_KEY>',
    'app_key': '<DATADOG_APPLICATION_KEY>'
}
initialize(**options)

start_ts = int(time.time())
end_ts = start_ts + (2 * 60 * 60)  # 2 hours
end_recurrence_ts = start_ts + (4 * 7 * 24 * 60 * 60)  # 4 weeks

recurrence = {
    'type': 'weeks',
    'period': 1,
    'week_days': ['Sat'],
    'until_date': end_recurrence_ts
}

api.Downtime.create(
    scope='env:production',
    monitor_tags='team:backend,service:web-store',
    start=start_ts,
    end=end_ts,
    recurrence=recurrence
)
```

### With Terraform

```hcl
resource "datadog_downtime" "weekly_maintenance" {
  scope       = "env:production"
  monitor_tags = ["team:backend", "service:web-store"]

  recurrence {
    type       = "weeks"
    period     = 1
    week_days  = ["Sat"]
  }

  start = 1700000000
  end   = 1700007200

  message = "Weekly maintenance window for backend services"
}
```

## Enhancing Dashboards with Monitor Tags

### Monitor Summary Widget

Add a Monitor Summary widget to any dashboard and filter using the same tag-based query syntax:

```
tag:service:web-store tag:team:backend
```

This shows the real-time state of all matching monitors in a single widget — useful for team-level or service-level status boards.

### Overlaying Monitor Events on Graphs

On timeseries widgets, overlay events filtered by monitor tags to correlate alert activity with metric trends:

```
sources:alert tag:service:web-store tag:team:backend
```

This makes it easy to spot whether a metric spike coincided with a triggered monitor.

## Organizing SLOs with Monitor Tags

### Tag SLI Monitors by Type

If a monitor serves as a Service Level Indicator (SLI), tag it with the type of SLI:

| SLI Type | Tag |
|----------|-----|
| Availability | `sli:availability` |
| Latency | `sli:latency` |
| Throughput | `sli:throughput` |
| Error rate | `sli:error_rate` |

### Automatic Tag Inheritance

When you create a monitor-based SLO, the SLO automatically inherits the tags of its constituent monitors. This means:

- You can filter the SLO list by `team`, `service`, `env`, or `sli` tags
- No need to manually re-tag each SLO
- Changes to monitor tags propagate to SLOs

### Searching SLOs

Use inherited tags in the Service Level Objectives view:

```
tag:team:backend tag:sli:latency tag:env:production
```

## Tagging Strategy Checklist

| Practice | Why |
|----------|-----|
| Use `key:value` format | Consistent grouping and filtering |
| Tag every monitor with `team`, `service`, `env` | Minimum viable context |
| Add `severity` or `priority` | Helps reduce alert fatigue, forces prioritization at creation time |
| Tag APM monitors with `resource_name` | Identifies the specific endpoint |
| Tag SLI monitors with `sli:<type>` | Enables SLO organization and filtering |
| Avoid unbounded tag values | Prevents cardinality explosion |
| Prefer lowercase, hyphenated values | Consistent naming across the organization |
| Review and standardize periodically | Prevents tag drift and inconsistency |

## Quick Reference

| Want to... | How |
|-----------|-----|
| Find all monitors for a team | `tag:team:backend` on Manage Monitors page |
| Mute monitors during maintenance | Schedule downtime with `monitor_tags` filter |
| Show monitor status on a dashboard | Monitor Summary widget with tag query |
| Correlate alerts with metrics | Overlay events filtered by monitor tags on timeseries |
| Organize SLOs by team or type | Tag underlying monitors — SLOs inherit automatically |
| Search monitors via API | `api.Monitor.search(query="tag:(key:value)")` |
| Search monitor events via API | `api.Event.query(sources=["alert"], tags=["key:value"])` |
