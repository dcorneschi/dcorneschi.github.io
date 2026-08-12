<img src="/articles/images/datadog-logo.svg" alt="Datadog" width="150"/>

# Datadog Monitor Notification Variables

Guide to using variables in Datadog monitor notification messages. Covers conditional variables for state-based messaging, attribute and tag variables for dynamic content, and template variables for threshold and value information.

## Conditional Variables

Conditional variables use if-else logic to display different messages depending on the monitor's state. They work in both the subject and body of notifications.

### Available Conditional Variables

| Variable | Displays text if... |
|----------|---------------------|
| `{{#is_alert}}` | The monitor alerts |
| `{{^is_alert}}` | The monitor does not alert |
| `{{#is_warning}}` | The monitor warns |
| `{{^is_warning}}` | The monitor does not warn |
| `{{#is_no_data}}` | The monitor is triggered for missing data |
| `{{^is_no_data}}` | The monitor is not triggered for missing data |
| `{{#is_recovery}}` | The monitor recovers from ALERT, WARNING, UNKNOWN, or NO DATA |
| `{{^is_recovery}}` | The monitor does not recover |
| `{{#is_warning_recovery}}` | The monitor recovers from WARNING to OK |
| `{{^is_warning_recovery}}` | The monitor does not recover from WARNING to OK |
| `{{#is_alert_recovery}}` | The monitor recovers from ALERT to WARNING or OK |
| `{{^is_alert_recovery}}` | The monitor does not recover from ALERT to OK |
| `{{#is_alert_to_warning}}` | The monitor transitions from ALERT to WARNING |
| `{{^is_alert_to_warning}}` | The monitor does not transition from ALERT to WARNING |
| `{{#is_no_data_recovery}}` | The monitor recovers from NO DATA |
| `{{^is_no_data_recovery}}` | The monitor does not recover from NO DATA |
| `{{#is_unknown}}` | The monitor is in unknown state |
| `{{^is_unknown}}` | The monitor is not in unknown state |
| `{{#is_renotify}}` | The monitor is renotifying |
| `{{^is_renotify}}` | The monitor is not renotifying |
| `{{#is_priority 'value'}}` | The monitor has priority value (P1 to P5) |
| `{{#is_match}}` | The context matches the provided substring |
| `{{^is_match}}` | The context does not match the provided substring |
| `{{#is_exact_match}}` | The context exactly matches the provided string |
| `{{^is_exact_match}}` | The context does not exactly match the provided string |

### Syntax Rules

- Every conditional variable must have an opening and closing pair
- The `#` prefix opens a block, `/` closes it, `^` negates
- State-based variables (`is_alert`, `is_warning`, etc.) must have their own message block — a monitor can only be in one state at a time
- You can nest conditionals that match on attributes (like `is_match` inside `is_renotify`)

### Basic State Examples

Send a notification when the monitor alerts:

```
{{#is_alert}}
  The service is DOWN. Immediate action required. @oncall-team@company.com
{{/is_alert}}
```

Send a notification when the monitor warns:

```
{{#is_warning}}
  The service is degraded. Investigate soon. @dev-team@company.com
{{/is_warning}}
```

Send a notification when the monitor recovers:

```
{{#is_recovery}}
  The service has recovered. No action needed. @oncall-team@company.com
{{/is_recovery}}
```

### is_match — Substring Matching

Search for a substring in a tag variable:

```
{{#is_match "<TAG_VARIABLE>.name" "<COMPARISON_STRING>"}}
  This displays if <COMPARISON_STRING> is found in <TAG_VARIABLE>.
{{/is_match}}
```

Notify the DB team if the triggering host has a role containing "db":

```
{{#is_match "host.role.name" "db"}}
  This host has a database role. @db-team@company.com
{{/is_match}}
```

Match against multiple strings (OR logic):

```
{{#is_match "host.role.name" "db" "database"}}
  This host has a database role. @db-team@company.com
{{/is_match}}
```

Use the negation (`^`) for "does not match":

```
{{^is_match "host.role.name" "db"}}
  This host does NOT have a database role. @slack-general
{{/is_match}}
```

Or use `{{else}}` for if/else logic in a single block:

```
{{#is_match "host.role.name" "db"}}
  Database host detected. @db-team@company.com
{{else}}
  Non-database host. @slack-general
{{/is_match}}
```

### is_exact_match — Exact String Matching

Search for an exact string in a tag variable:

```
{{#is_exact_match "<TAG_VARIABLE>.name" "<COMPARISON_STRING>"}}
  This displays if <TAG_VARIABLE> is exactly <COMPARISON_STRING>.
{{/is_exact_match}}
```

Notify the dev team if the host is exactly named "production":

```
{{#is_exact_match "host.name" "production"}}
  Alert on the production host! @dev-team@company.com
{{/is_exact_match}}
```

Match multiple exact strings:

```
{{#is_exact_match "host.name" "production" "staging"}}
  Alert on production or staging. @dev-team@company.com
{{/is_exact_match}}
```

Use with `{{value}}` to match the breached threshold value:

```
{{#is_exact_match "value" "5"}}
  The threshold value is exactly 5. @dev-team@company.com
{{/is_exact_match}}
```

Check if a tag or attribute is empty or does not exist:

```
{{#is_exact_match "host.datacenter" ""}}
  Warning: datacenter tag is missing or empty for this host.
{{/is_exact_match}}
```

### is_renotify — Escalation Messages

Send escalation messages only on renotification:

```
{{#is_renotify}}
{{#is_match "env" "production"}}
  ESCALATION: This production alert has not been resolved. @escalation-team@company.com
{{/is_match}}
{{/is_renotify}}
```

Send a detailed first alert and a shorter escalation:

```
{{^is_renotify}}
This monitor is alerting. @dev-team@company.com

To resolve:
1. Check the service dashboard
2. Review recent deployments
3. Restart the service if needed
{{/is_renotify}}

This message is sent on both the first trigger and escalation.

{{#is_renotify}}
  ESCALATION: Still unresolved. @manager@company.com
{{/is_renotify}}
```

On renotification, the recipient sees:

```
This message is sent on both the first trigger and escalation.

ESCALATION: Still unresolved. @manager@company.com
```

## Attribute and Tag Variables

Use attribute and tag variables to render dynamic, context-specific alert messages.

### Multi-Alert Variables

When a monitor is grouped by a tag in `key:value` format, use this syntax to include the value in notifications:

```
{{ key.name }}
```

For example, if the monitor is grouped by `env`:

```
Alert triggered on environment: {{ env.name }}
```

If a group has multiple values for the same key, they display as a comma-separated string in lexicographic order.

### Tag Keys with Periods

Wrap tag keys containing periods in brackets:

```
{{ [dot.key.test].name }}
```

### Log, Trace, and RUM Facets

For monitors grouped by a facet, use:

```
{{ @facet_key.name }}
```

Example for a log monitor grouped by `@machine_id`:

```
This alert was triggered on {{ @machine_id.name }}
```

For facets with periods, use brackets:

```
{{ [@network.client.ip].name }}
```

### Host Metadata Variables

When grouped by host, these metadata variables are available:

```
Agent Version: {{host.metadata_agent_version}}
Machine: {{host.metadata_machine}}
Platform: {{host.metadata_platform}}
Processor: {{host.metadata_processor}}
```

### Kubernetes Metadata Variables

When grouped by `kube_namespace` and `kube_cluster_name`:

```
Cluster name: {{kube_namespace.cluster_name}}
Namespace name: {{kube_namespace.display_name}}
Namespace status: {{kube_namespace.status}}
Namespace labels: {{kube_namespace.labels}}
```

When grouped by `pod_name`, `kube_namespace`, and `kube_cluster_name`:

```
Cluster name: {{pod_name.cluster_name}}
Pod name: {{pod_name.name}}
Pod phase: {{pod_name.phase}}
```

### Service Catalog Variables

When grouped by service:

```
Service name: {{service.name}}
Team name: {{service.team}}
Docs: {{service.docs}}
Links: {{service.links}}
```

Access a specific link by name:

```
{{service.links[Runbook]}}
```

### Matching Attribute/Tag Variables by Monitor Type

| Monitor Type | Variable Syntax |
|-------------|-----------------|
| Audit Trail | `{{audit.attributes.key}}` or `{{audit.message}}` |
| CI Pipeline | `{{cipipeline.attributes.key}}` |
| CI Test | `{{citest.attributes.key}}` |
| Database Monitoring | `{{databasemonitoring.attributes.key}}` |
| Error Tracking | `{{issue.attributes.key}}` |
| Log | `{{log.attributes.key}}` or `{{log.tags.key}}` |
| RUM | `{{rum.attributes.key}}` or `{{rum.tags.key}}` |
| Synthetic Monitoring | `{{synthetics.attributes.key}}` |
| Trace Analytics | `{{span.attributes.key}}` or `{{span.tags.key}}` |

Example for a log monitor:

```
{{ log.attributes.[error.message] }}
{{ log.tags.env }}
```

### Reserved Attributes

| Monitor Type | Syntax | Available Attributes |
|-------------|--------|---------------------|
| Log | `{{log.key}}` | message, service, status, source, span_id, timestamp, trace_id, link, host |
| Trace Analytics | `{{span.key}}` | env, operation_name, resource_name, service, status, span_id, timestamp, trace_id, type, link |
| RUM | `{{rum.key}}` | service, status, timestamp, link |
| Event | `{{event.key}}` | attributes, host.name, id, link, title, text, tags |
| CI Pipeline | `{{cipipeline.key}}` | service, env, resource_name, ci_level, trace_id, span_id, pipeline_fingerprint, operation_name, status, timestamp, link |
| CI Test | `{{citest.key}}` | service, env, resource_name, trace_id, span_id, operation_name, status, timestamp, link |

### Explorer Links

Use these to link directly to the relevant explorer, scoped to matching events:

```
{{log.link}}
{{span.link}}
{{rum.link}}
{{issue.link}}
```

## Template Variables

Built-in variables for threshold and value information:

| Variable | Description |
|----------|-------------|
| `{{value}}` | The value that breached the alert threshold |
| `{{threshold}}` | The alert threshold value |
| `{{warn_threshold}}` | The warning threshold value |
| `{{alert_recovery_threshold}}` | The value that recovered from ALERT |
| `{{warn_recovery_threshold}}` | The value that recovered from WARN |
| `{{ok_threshold}}` | The value that recovered a Service Check monitor |
| `{{comparator}}` | The relational operator (>, <, >=, <=) |
| `{{first_triggered_at}}` | UTC timestamp when the monitor first triggered |
| `{{first_triggered_at_epoch}}` | Same as above, in epoch milliseconds |
| `{{last_triggered_at}}` | UTC timestamp when the monitor last triggered |
| `{{last_triggered_at_epoch}}` | Same as above, in epoch milliseconds |
| `{{triggered_duration_sec}}` | Seconds the monitor has been in a triggered state |

### Triggered Variable Behavior

- `{{first_triggered_at}}` is set when the monitor group goes from OK to a non-OK state, or when a new group appears in non-OK state
- `{{last_triggered_at}}` is set on any transition to a non-OK state, regardless of previous state (including WARN to ALERT, ALERT to WARN)
- Renotification events show the same values if the state has not changed — use `{{triggered_duration_sec}}` to show elapsed time

Example timeline:

| Transition | first_triggered_at | last_triggered_at | triggered_duration_sec |
|-----------|-------------------|-------------------|----------------------|
| OK → WARN | A | A | 0 |
| WARN → ALERT | A | B | B - A |
| ALERT → NO DATA | A | C | C - A |
| NO DATA → OK | A | C | D - A |

### Local Time Conversion

Convert timestamps to a specific timezone:

```
{{local_time 'last_triggered_at' 'Asia/Tokyo'}}
```

Output format: `yyyy-MM-dd HH:mm:ss±HH:mm` (e.g., `2021-05-31 23:43:27+09:00`)

Use any value from the [tz database](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

## Composite Monitor Variables

Access sub-monitor values and status:

```
{{ a.value }}
{{ a.status }}
```

Possible status values: `OK`, `Alert`, `Warn`, `No Data`.

For composite monitors with log sub-monitors:

```
{{ a.log.message }}
{{ a.log.my_facet }}
```

## Character Escaping

By default, variable content is HTML-encoded. Use triple braces for raw, unencoded output:

| Syntax | Output |
|--------|--------|
| `{{variable}}` | HTML-encoded (default) |
| `{{{variable}}}` | Raw, unencoded |

Example — double braces encode `&` as `&amp;`:

```
{{check_message}}
→ https://status.example.com/check?service=web&amp;region=us-east
```

Triple braces preserve the URL:

```
{{{check_message}}}
→ https://status.example.com/check?service=web&region=us-east
```

Use triple braces when `{{check_message}}` contains URLs with query parameters (common in HTTP Check monitors).

## Advanced Usage

### Dynamic Handles

Route notifications dynamically based on tags:

```
@slack-{{service.name}} There is an ongoing issue with {{service.name}}.
```

If `service:ad-server` triggers, the notification goes to `#ad-server` on Slack.

Add a fallback handle for attributes that might not exist:

```
{{#is_exact_match "kube_namespace.owner" ""}}
  @slack-fallback-channel
{{/is_exact_match}}
```

### Dynamic Links

Link to a system dashboard scoped to the alerting host:

```
https://app.datadoghq.com/dash/integration/system_overview?tpl_var_scope=host:{{host.name}}
```

Link to a dashboard with a relative time range around the alert:

```
https://app.datadoghq.com/dashboard/<DASHBOARD_ID>/<DASHBOARD_NAME>?from_ts={{eval "last_triggered_at_epoch-10*60*1000"}}&to_ts={{eval "last_triggered_at_epoch+10*60*1000"}}&live=false
```

Link to the host map filtered by service:

```
https://app.datadoghq.com/infrastructure/map?filter=service:{{service.name}}
```

Link to monitors for a specific host:

```
https://app.datadoghq.com/monitors/manage?q=scope:host:{{host.name}}
```

Link to logs around the time of the alert:

```
https://app.datadoghq.com/logs?from_ts={{eval "last_triggered_at_epoch-10*60*1000"}}&to_ts={{eval "last_triggered_at_epoch+10*60*1000"}}&live=false
```

### Comments

Add comments that won't appear in the rendered notification:

```
{{!-- this is a comment --}}
```

### Raw Output

To output literal double curly braces (e.g., for documentation or templating tools):

```
{{{{raw}}}}
{{ <TEXT_1> }} {{ <TEXT_2> }}
{{{{/raw}}}}
```

Output:

```
{{ <TEXT_1> }} {{ <TEXT_2> }}
```

Use raw formatting with conditional variables (note the four braces):

```
{{{{is_match "host.name" "web-01"}}}}
{{ .matched }} the host name
{{{{/is_match}}}}
```

### URL Encoding

Encode variable values for use in URLs:

```
https://app.datadoghq.com/services/{{urlencode "service.name"}}
```

## Important Notes

- Text or `@-notification` handles placed **outside** conditional blocks are sent on every state transition
- Text or handles **inside** conditional blocks are only sent when the condition matches
- If you use `@-notification` inside an alert/warning block, configure a matching recovery block so the recipient gets recovery notifications
- If an attribute or tag key doesn't exist on the selected event, the variable renders empty — avoid using these for routing with `{{#is_match}}` handles unless you add a fallback

## Quick Reference

| Want to... | Use |
|-----------|-----|
| Send different messages per state | `{{#is_alert}}`, `{{#is_warning}}`, `{{#is_recovery}}` |
| Route by tag substring | `{{#is_match "tag.name" "value"}}` |
| Route by exact tag value | `{{#is_exact_match "tag.name" "value"}}` |
| Show the breached value | `{{value}}` |
| Show the threshold | `{{threshold}}` |
| Show how long it's been alerting | `{{triggered_duration_sec}}` |
| Include a dynamic Slack channel | `@slack-{{service.name}}` |
| Link to logs at alert time | `from_ts={{eval "last_triggered_at_epoch-10*60*1000"}}` |
| Output raw curly braces | `{{{{raw}}}} ... {{{{/raw}}}}` |
| Avoid HTML encoding in URLs | `{{{variable}}}` (triple braces) |
| Check if tag is empty/missing | `{{#is_exact_match "tag.name" ""}}` |
| Send escalation-only messages | `{{#is_renotify}} ... {{/is_renotify}}` |
