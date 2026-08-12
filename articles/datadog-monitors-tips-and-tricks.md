<img src="/articles/images/datadog-logo.svg" alt="Datadog" width="150"/>

# Datadog Monitors Tips & Tricks

A collection of lesser-known features, hidden settings, and advanced techniques for building better Datadog monitors. Covers anti-flapping strategies, noise reduction, notification control, query tricks, API-only options, and operational patterns that are hard to discover in the UI.

## Reduce Flapping with Recovery Thresholds

The single most effective way to prevent a monitor from flapping between ALERT and OK is to set a recovery threshold that is different from the alert threshold. This creates a hysteresis gap (dead zone) where the monitor stays in its current state.

```
Alert threshold:    > 90%
Recovery threshold: < 75%
```

The monitor triggers at 90% but won't recover until the metric drops below 75%. This prevents rapid toggling when a value hovers around a single boundary.

Recovery thresholds work on:
- Threshold alerts
- Change alerts
- Anomaly detection monitors

Set them in the UI under "Alert Conditions" > "Alert Recovery Threshold" / "Warning Recovery Threshold", or via the API:

```json
{
  "thresholds": {
    "critical": 90,
    "critical_recovery": 75,
    "warning": 80,
    "warning_recovery": 65
  }
}
```

## New Group Delay — Prevent Alerts on Container Startup

When a new container, pod, or autoscaling instance spins up, it often has high CPU or latency during initialization. The `new_group_delay` option delays evaluation for new groups:

```json
{
  "new_group_delay": 300
}
```

This gives new groups 5 minutes (300 seconds) to stabilize before the monitor starts evaluating them. The default is 60 seconds. It applies to any group-by value not seen in the last 24 hours.

In the UI: Advanced Options > "Delay evaluation for new groups by N seconds"

Use cases:
- Container orchestration (Kubernetes, ECS) where pods start frequently
- Autoscaling groups adding instances under load
- Blue-green deployments creating new service instances

## Evaluation Delay — Handle Backfilled Cloud Metrics

Cloud providers (AWS, GCP, Azure) report metrics with a delay. If your monitor evaluates before the data arrives, you get false "No Data" alerts.

```json
{
  "evaluation_delay": 900
}
```

This delays evaluation by 15 minutes (900 seconds). Datadog recommends 15 minutes for cloud metrics. The maximum is 86400 seconds (24 hours).

In the UI: Advanced Options > "Delay evaluation by N seconds"

| Provider | Recommended Delay |
|----------|-------------------|
| AWS CloudWatch | 900s (15 min) |
| GCP Stackdriver | 900s (15 min) |
| Azure Monitor | 900s (15 min) |
| Division formulas | 60s (1 min) |

The 60-second delay for division formulas ensures both the numerator and denominator queries have complete data before the ratio is calculated.

## Require Full Window of Data

The `require_full_window` option controls whether the monitor waits for a complete evaluation window before alerting.

```json
{
  "require_full_window": false
}
```

**Set to `false` (recommended) for:**
- Sparse metrics (reported infrequently)
- Metrics with gaps
- Custom metrics sent by cron jobs

**Set to `true` for:**
- High-frequency infrastructure metrics
- Metrics where partial windows would give misleading results

When `true` and the window isn't fully populated, the evaluation is skipped — which can look like "No Data".

## Notification Preset — Control What Gets Sent

The `notification_preset_name` option controls how much metadata appears in notification messages. Most people never change this, but it dramatically cleans up noisy Slack/email alerts:

| Preset | Shows |
|--------|-------|
| `show_all` (default) | Query, handles, snapshot, footer |
| `hide_query` | Hides the query from the notification |
| `hide_handles` | Hides @-mentions from the message body |
| `hide_query_and_handles` | Hides both query and handles |
| `hide_all` | Hides query, handles, and footer |
| `show_only_snapshot` | Only the graph snapshot |
| `hide_handles_and_footer` | Hides handles and the footer |

Set via the API:

```json
{
  "notification_preset_name": "hide_query"
}
```

In the UI: "Configure Notifications" section > toggle "Show Query" / "Show Handles" checkboxes.

This is particularly useful when you use notification rules (centralized routing) and don't want the individual `@slack-channel` handles cluttering the message.

## Include Tags in Notification Title

The `include_tags` option controls whether triggering tags appear in the notification title:

```json
{
  "include_tags": true
}
```

- `true` (default): `[Triggered on {host:web-01}] High CPU Usage`
- `false`: `[Triggered] High CPU Usage`

Set to `false` when:
- You have many tags and the title becomes unreadable
- You already include the relevant information in the message body
- You're using dynamic handles and the title is cluttered

## Auto-Resolve for Counters and Sparse Metrics

Some metrics only report when something happens (error counters, event counts). They never report 0, so the monitor never recovers naturally.

```json
{
  "timeout_h": 2
}
```

This auto-resolves the monitor after 2 hours without data. Range: 0-24 hours.

In the UI: Advanced Options > "Automatically resolve monitor after N hours"

Use cases:
- Error counters that only increment on failure
- Custom event metrics
- Batch job metrics that report once per run

**Important:** Auto-resolve only works when data stops being submitted. If data is still coming in above the threshold, the monitor stays in ALERT regardless of this setting.

## Renotification — Escalation Without Extra Monitors

Instead of building a separate "escalation monitor", use renotification settings:

```json
{
  "renotify_interval": 60,
  "renotify_occurrences": 3,
  "renotify_statuses": ["alert", "no data"],
  "escalation_message": "This alert has been open for over an hour. @manager@company.com"
}
```

This sends the escalation message every 60 minutes, up to 3 times, but only if the monitor is still in ALERT or NO DATA.

Combine with conditional variables in the message body for even more control:

```
{{#is_renotify}}
ESCALATION: This alert has not been resolved.
{{#is_match "env" "production"}}
@pagerduty-critical
{{/is_match}}
{{/is_renotify}}
```

## On Missing Data — The Modern Alternative to notify_no_data

The legacy `notify_no_data` boolean is being replaced by the more flexible `on_missing_data` option:

| Option | Behavior |
|--------|----------|
| `default` | Count queries treat empty as 0; other types show last known status |
| `show_no_data` | Shows NO DATA status without notifying |
| `show_and_notify_no_data` | Shows NO DATA and sends a notification |
| `resolve` | Resolves the monitor (sets status to OK) |

The `resolve` option is particularly useful for error-counting monitors — if there's no data, it means no errors, so the monitor should be OK:

```json
{
  "on_missing_data": "resolve"
}
```

## Group Retention — Drop Ephemeral Groups Faster

In dynamic environments (Kubernetes, serverless), groups appear and disappear constantly. By default, a group stays in the monitor's status for 24 hours after it stops reporting.

```json
{
  "group_retention_duration": "2h"
}
```

This drops non-reporting groups after 2 hours instead of 24. Range: 1 hour to 72 hours.

In the UI: Advanced Options > "Remove non-reporting groups after N"

Use cases:
- Short-lived containers that only exist for minutes
- Spot instances that terminate unpredictably
- CI/CD runners that scale to zero
- Serverless functions with cold starts

## Formulas & Functions — Monitor Ratios, Not Absolutes

Instead of alerting on raw counts, monitor the ratio. This prevents false alerts during traffic spikes:

```
Query A: sum:http.requests.errors{service:web-store}.as_count()
Query B: sum:http.requests.total{service:web-store}.as_count()
Formula: a / b * 100
Alert when: > 5 (5% error rate)
```

This fires when the error *rate* exceeds 5%, not when the absolute number of errors is high. A traffic spike that doubles both errors and total requests won't trigger the alert.

Other useful formulas:

| Use Case | Formula |
|----------|---------|
| Error rate (%) | `errors / total * 100` |
| Cache hit ratio | `hits / (hits + misses) * 100` |
| Availability | `(1 - errors / total) * 100` |
| Saturation | `used / capacity * 100` |
| Per-instance average | `total_metric / count_of_instances` |

## Composite Monitors — Reduce Noise with AND/OR Logic

A single metric alert can be noisy. Composite monitors combine multiple monitors with boolean logic:

```
Monitor A: High CPU on host
Monitor B: High memory on host
Monitor C: Elevated error rate for service

Composite: A && B && C
```

This only fires when ALL three conditions are true simultaneously. Use cases:

- **High CPU is only a problem if users are affected**: `high_cpu && elevated_latency`
- **Disk full matters only on active hosts**: `disk_90_percent && host_receiving_traffic`
- **Alert on canary deployment issues**: `canary_errors_high && canary_latency_high`

Operators: `&&` (AND), `||` (OR), `!` (NOT)

Access sub-monitor data in composite notifications:

```
CPU: {{ a.value }}% (status: {{ a.status }})
Memory: {{ b.value }}% (status: {{ b.status }})
Errors: {{ c.value }} req/s (status: {{ c.status }})
```

## Detection Methods Beyond Simple Thresholds

Most monitors use "Threshold Alert" — but there are four other detection methods:

### Change Alert

Alert when a metric changes by a specific amount or percentage compared to a previous period:

```
Alert if the avg of system.cpu.user changes by more than 50%
compared to 1 hour ago
```

Useful for catching sudden spikes that are relative to normal behavior.

### Anomaly Detection

Uses historical data to build a baseline, then alerts on deviations:

- **Agile**: Quick to adapt, good for metrics that shift frequently
- **Robust**: Slow to adapt, ignores recent anomalies when building the baseline
- **Basic**: Simple computation, no seasonality

Key settings:
- `trigger_window`: How long the metric must be anomalous before triggering
- `recovery_window`: How long it must be normal before recovering

```json
{
  "threshold_windows": {
    "trigger_window": "last_15m",
    "recovery_window": "last_15m"
  }
}
```

### Outlier Detection

Detects when one member of a group behaves differently from its peers. Requires at least 3 group members.

Use case: One host's CPU is at 95% while the rest of the group is at 30%.

Algorithms:
- **DBSCAN** (recommended): Density-based clustering
- **MAD**: Median Absolute Deviation

### Forecast Alert

Predicts when a metric will cross a threshold in the future:

```
Alert if system.disk.in_use is predicted to reach 90% within 1 week
```

Algorithms:
- **Linear**: Constant rate of change
- **Seasonal**: Accounts for weekly/daily patterns

## Notification Tricks

### Dynamic Handles — Route by Tag

Send alerts to different channels based on the triggering group:

```
@slack-{{service.name}}-alerts
```

If `service:payments` triggers → `#payments-alerts`
If `service:auth` triggers → `#auth-alerts`

### Fallback for Missing Tags

Always add a fallback when using dynamic handles:

```
{{#is_exact_match "team.name" ""}}
  @slack-unowned-alerts
{{/is_exact_match}}

{{^is_exact_match "team.name" ""}}
  @slack-{{team.name}}-alerts
{{/is_exact_match}}
```

### Link to Relevant Dashboard with Time Context

```
[View Dashboard](https://app.datadoghq.com/dashboard/abc-123/my-dashboard?from_ts={{eval "last_triggered_at_epoch-10*60*1000"}}&to_ts={{eval "last_triggered_at_epoch+10*60*1000"}}&live=false)
```

### Link to Logs at Alert Time

```
[View Logs](https://app.datadoghq.com/logs?from_ts={{eval "last_triggered_at_epoch-10*60*1000"}}&to_ts={{eval "last_triggered_at_epoch+10*60*1000"}}&live=false&query=service:{{service.name}})
```

### Hide Boilerplate with Comments

```
{{!-- Internal note: this monitor was created for JIRA-1234 --}}
{{!-- Last reviewed: 2025-01-15 by @oncall --}}
```

### Raw Output for Template Conflicts

If your message needs literal curly braces:

```
{{{{raw}}}}
Expected format: {{ .metadata.name }}
{{{{/raw}}}}
```

## API-Only Options Worth Knowing

These options are not exposed in the UI but can be set via the API or Terraform:

| Option | Description |
|--------|-------------|
| `notify_by` | Override alert grouping. Set to `["*"]` to force simple-alert behavior on a grouped query |
| `restricted_roles` | Lock monitor editing to specific role UUIDs |
| `group_retention_duration` | Custom time before ephemeral groups are dropped |
| `renotify_statuses` | Limit renotification to specific states (e.g., only `alert`, not `no data`) |

## Terraform Patterns

### Monitor with All Advanced Options

```hcl
resource "datadog_monitor" "error_rate" {
  name    = "High Error Rate - {{service.name}}"
  type    = "query alert"
  query   = "sum(last_5m):sum:http.errors{env:production} by {service}.as_count() / sum:http.requests{env:production} by {service}.as_count() * 100 > 5"
  message = <<-EOT
    Error rate for {{service.name}} is {{value}}%.

    {{#is_alert}}
    Immediate investigation required. @pagerduty-critical
    {{/is_alert}}

    {{#is_recovery}}
    Error rate has recovered. @slack-incidents
    {{/is_recovery}}

    {{#is_renotify}}
    Still unresolved after {{triggered_duration_sec}}s. @manager@company.com
    {{/is_renotify}}
  EOT

  tags = ["service:web-store", "team:backend", "env:production", "severity:high"]

  monitor_thresholds {
    critical          = 5
    critical_recovery = 2
    warning           = 3
    warning_recovery  = 1
  }

  notify_no_data    = false
  new_group_delay   = 300
  evaluation_delay  = 60
  include_tags      = true
  require_full_window = false
  timeout_h         = 0

  renotify_interval    = 60
  renotify_occurrences = 3
  escalation_message   = "This alert has been open for over an hour."

  notification_preset_name = "hide_query"
}
```

### Composite Monitor in Terraform

```hcl
resource "datadog_monitor" "composite_service_degraded" {
  name    = "Service Degraded - High Errors AND High Latency"
  type    = "composite"
  query   = "${datadog_monitor.error_rate.id} && ${datadog_monitor.high_latency.id}"
  message = <<-EOT
    Both error rate and latency are elevated for the service.

    Error rate: {{ a.value }}% (threshold: 5%)
    P99 latency: {{ b.value }}ms (threshold: 500ms)

    @pagerduty-critical @slack-incidents
  EOT

  tags = ["team:backend", "env:production", "severity:critical"]
}
```

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Alert on raw counts without normalization | Traffic spikes cause false alerts | Use ratios (errors/total) |
| No recovery threshold | Monitor flaps between ALERT and OK | Set recovery threshold with a gap |
| `require_full_window: true` on sparse metrics | Skipped evaluations, phantom No Data | Set to `false` |
| `new_group_delay: 0` in Kubernetes | Alerts on every pod startup | Set 60-300s delay |
| No evaluation delay for cloud metrics | False No Data alerts | Set 900s for AWS/GCP/Azure |
| Separate monitors for initial + escalation | Hard to maintain, duplicate logic | Use `renotify_interval` + `escalation_message` |
| Alerting on 1-minute window for noisy metrics | Constant false positives | Use 5-15 minute windows |
| Tag values from unbounded sources | Cardinality explosion, slow monitors | Use bounded dimensions only |
| No `on_missing_data: resolve` for error counts | Permanent NO DATA when no errors occur | Set to `resolve` |

## Quick Reference

| Want to... | Setting |
|-----------|---------|
| Prevent flapping | Set `critical_recovery` below `critical` |
| Skip alerts on new containers | `new_group_delay: 300` |
| Handle AWS metric delay | `evaluation_delay: 900` |
| Auto-resolve counter alerts | `timeout_h: 2` |
| Escalate after N minutes | `renotify_interval: 60` |
| Limit escalation count | `renotify_occurrences: 3` |
| Clean up notification content | `notification_preset_name: hide_query` |
| Drop ephemeral groups faster | `group_retention_duration: 2h` |
| Treat no errors as healthy | `on_missing_data: resolve` |
| Alert on rate, not count | Use formula: `a / b * 100` |
| Reduce noise with conditions | Composite monitors with `&&` |
| Route by service tag | `@slack-{{service.name}}` |
| Lock editing to a team | `restricted_roles: [<UUID>]` |
