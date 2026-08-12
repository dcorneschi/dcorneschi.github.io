<img src="/articles/images/datadog-logo.svg" alt="Datadog" width="150"/>

# Datadog Dashboards Guide

A practical guide to building effective Datadog dashboards — covering dashboard types, widget selection, query patterns, template variables, layout strategies, and best practices for presenting operational data to both technical and non-technical audiences.

## What Dashboards Solve

Dashboards centralize your data into a single visual surface. Instead of jumping between Metrics Explorer, Log Analytics, APM traces, and Infrastructure views, you build a single page that tells a story about the health and performance of a system.

They solve three problems:

1. **Communication** — Presenting technical information visually so colleagues, managers, and stakeholders can quickly grasp what's happening without parsing raw numbers
2. **Context** — Grouping related metrics, logs, and events together so patterns and correlations become obvious
3. **Speed** — Providing a single starting point for incident triage instead of navigating to five different Datadog sections

A dashboard without purpose is just a wall of graphs. Before building one, know who will read it and what decisions it should help them make.

## Dashboard Types

Datadog offers three dashboard layouts:

| Type | Layout | Use Case |
|------|--------|----------|
| **Screenboard** | Free-form, drag anywhere | Executive dashboards, NOC screens, presentations |
| **Timeboard** | Automatic grid, all graphs share time | Debugging, incident response, correlating metrics |
| **PowerPack** | Reusable widget group | Standardized component shared across dashboards |

> **Note:** In the Datadog API and Terraform, these map to `layout_type: "free"` (Screenboard) and `layout_type: "ordered"` (Timeboard). The UI no longer uses the Screenboard/Timeboard names — you simply choose the layout when creating a new dashboard.

### Screenboard vs Timeboard

| Aspect | Screenboard | Timeboard |
|--------|------------|-----------|
| Layout | Free-form (pixel-level placement) | Auto-arranged grid |
| Time sync | Each widget can have its own timeframe | All widgets share the same timeframe |
| Time cursor | No shared cursor | Hover on one graph, all graphs show the same point |
| Best for | Status boards, business metrics, wall displays | Debugging, investigating spikes, incident triage |
| Template variables | Yes | Yes |
| Sharing | Public URL available | Public URL available |

**Rule of thumb:** If you need to correlate graphs during an incident (hover over a spike on one graph and see what happened at the same time on all other graphs), use a Timeboard. If you're building a display for a TV on the wall, use a Screenboard.

## Widgets

Widgets are the building blocks of a dashboard. Without them, the dashboard is a blank page. Choosing the right widget for your data makes the difference between a dashboard people use and one they ignore.

### Core Widget Types

| Widget | Purpose | When to Use |
|--------|---------|-------------|
| **Timeseries** | Line/area/bar chart over time | Metrics trending, load patterns, latency evolution |
| **Query Value** | Single number, big and bold | Current error rate, active users, queue depth |
| **Top List** | Ranked list of groups | Top hosts by CPU, busiest endpoints, heaviest services |
| **Table** | Tabular data with sorting | Multi-dimensional breakdowns, host inventories |
| **Heatmap** | Distribution over time | Latency percentiles, request size distribution |
| **Distribution** | Histogram of current values | See the shape of your latency distribution |
| **Change** | Percentage change vs previous period | Week-over-week comparison of traffic, revenue, errors |
| **SLO** | SLO status and error budget | Track service reliability targets |
| **Monitor Summary** | Status of multiple monitors | Operations overview — what's alerting |
| **Event Stream** | Live event feed | Deployments, alerts, config changes |
| **Log Stream** | Live log tail | Recent errors, access logs, audit trail |
| **Notes & Links** | Markdown text, links | Section headers, runbook links, context for viewers |
| **Group** | Container for other widgets | Visual sections, collapsible areas |
| **Alert Graph** | Monitor's metric with status | Show exactly what a monitor is evaluating |

### Timeseries Widget

The workhorse of dashboards. Shows the evolution of a metric over time.

```
Metric:    avg:system.cpu.user{*} by {host}
Display:   Line chart
Timeframe: Past 4 hours
```

Tips:
- Use `avg`, `max`, or `sum` based on what matters — `avg` hides spikes, `max` shows worst case
- Group by meaningful tags (`host`, `service`, `env`, `availability-zone`)
- Add markers for thresholds (horizontal lines at alert/warning boundaries)
- Overlay events (deploys, alerts) to correlate changes with metric shifts

### Query Value Widget

Displays a single number prominently. Use it for the most important metrics that need to be glanceable.

```
Metric:    sum:trace.http.request.hits{service:web-store,env:production}.as_count()
Display:   Query Value
Timeframe: Past 5 minutes
```

Tips:
- Apply conditional formatting — green/yellow/red based on thresholds
- Keep the timeframe short (last 5 min or last 1 min) for "current state" values
- Pair with a Timeseries widget below it showing the same metric's history
- Use formulas for ratios: `a / b * 100` for percentages (e.g., error rate)

### Top List Widget

Shows a ranked list. Useful for identifying outliers.

```
Metric:    avg:system.cpu.user{*} by {host}
Display:   Top List (top 10)
Order:     Descending
```

Use cases:
- Top 10 hosts by CPU usage
- Busiest API endpoints by request count
- Services with the highest error rates
- Pods consuming the most memory

### Notes & Links Widget

Adds context to your dashboard. Without notes, viewers have to guess what they're looking at.

Use them for:
- **Section headers** — Break the dashboard into logical areas
- **Runbook links** — "If this graph is red, follow [this runbook](url)"
- **Data explanations** — "This metric is delayed by 15 minutes from CloudWatch"
- **Owner information** — "Dashboard maintained by Platform team. Slack: #platform-eng"

Supports full Markdown including links, bold, lists, and headers.

## Template Variables

Template variables add dropdown filters to the top of your dashboard. They let a single dashboard serve multiple teams, environments, or services.

### Defining Template Variables

| Variable | Tag | Default |
|----------|-----|---------|
| `$env` | `env` | `production` |
| `$service` | `service` | `*` |
| `$host` | `host` | `*` |
| `$region` | `region` | `*` |

### Using Template Variables in Queries

```
avg:system.cpu.user{env:$env,service:$service,host:$host} by {host}
```

When a user selects `env:staging` from the dropdown, all widgets on the dashboard filter to staging data.

### Tips

- Set sensible defaults (e.g., `production`) so the dashboard is useful immediately
- Use `*` as default for secondary filters — don't pre-filter too narrowly
- Order variables from broadest to narrowest: env → service → host
- Some widgets (notes, images) don't support template variables — that's fine

## Query Patterns

### Basic Metric Query

```
avg:system.cpu.user{env:production,service:api} by {host}
```

### Formulas — Error Rate Percentage

```
a = sum:trace.http.request.errors{service:web}.as_count()
b = sum:trace.http.request.hits{service:web}.as_count()

Formula: (a / b) * 100
```

### Formulas — Requests Per Second

```
a = sum:trace.http.request.hits{service:web}.as_count()

Formula: a / 60   (if using 1-minute rollup)
```

### Rollup — Control Aggregation Window

```
avg:system.cpu.user{*}.rollup(max, 300)
```

This takes the max value in each 5-minute window instead of the average. Useful for catching spikes that would be smoothed out by averaging.

### Forecast Functions

```
forecast(avg:system.disk.in_use{*}, 'linear', 1)
```

Projects future values based on historical trends. Useful for capacity planning dashboards.

### Anomaly Detection Overlay

```
anomalies(avg:system.cpu.user{*}, 'agile', 3)
```

Overlays anomaly detection bands on a timeseries. The gray band shows the expected range — points outside it are anomalous.

### Week-Over-Week Comparison

```
a = avg:trace.http.request.hits{service:web}.as_count()
b = calendar_shift(avg:trace.http.request.hits{service:web}.as_count(), "-7d")
```

Display both on the same timeseries to compare current traffic against last week.

> **Note:** The older `week_before()` function is deprecated. Use `calendar_shift()` with `-7d` instead. Similarly, use `-1d` instead of `day_before()` and `-1mo` instead of `month_before()`.

### Excluding Specific Values

```
avg:system.cpu.user{service:api,!host:test-host-01} by {host}
```

The `!` prefix excludes a tag value. Useful for filtering out canary hosts, test environments, or known noisy sources.

## Dashboard Layout Best Practices

### The Inverted Pyramid

Structure your dashboard like a news article — most important information first:

```
┌─────────────────────────────────────────────────────────────┐
│  Row 1: Health at a glance (query values, SLO status)       │
│  "Is everything OK right now?"                              │
├─────────────────────────────────────────────────────────────┤
│  Row 2: Key metrics over time (timeseries, 2-3 graphs)      │
│  "What's the trend?"                                        │
├─────────────────────────────────────────────────────────────┤
│  Row 3: Breakdown by component (top lists, tables)           │
│  "Where is the problem?"                                    │
├─────────────────────────────────────────────────────────────┤
│  Row 4: Supporting detail (logs, events, traces)             │
│  "What happened exactly?"                                   │
└─────────────────────────────────────────────────────────────┘
```

### Section With Groups

Use Group widgets to organize dashboards into collapsible sections:

- **Overview** — Query values for key numbers
- **Performance** — Latency, throughput, error rate graphs
- **Infrastructure** — CPU, memory, disk, network
- **Dependencies** — Database, cache, external API metrics
- **Deployments & Changes** — Event stream showing recent deploys

### Widget Sizing

- **Query values** — Small (2x1 or 3x1). They're meant to be scanned quickly.
- **Timeseries** — Medium to large (6x3 or 4x3). Trends need space to be readable.
- **Top lists** — Medium (4x3). Show enough items without scrolling.
- **Notes** — Full width (12x1) for section headers. Small for inline context.

## Dashboard Patterns

### Service Health Dashboard

A dashboard per service showing the RED metrics (Rate, Errors, Duration):

| Section | Widgets |
|---------|---------|
| Header | Service name, owner, runbook link (Notes widget) |
| Status | Query values: requests/sec, error rate %, p99 latency |
| Trends | Timeseries: request rate, error rate, latency percentiles (p50, p95, p99) |
| Breakdown | Top list: errors by endpoint, latency by endpoint |
| Dependencies | Timeseries: database query time, cache hit rate, external API latency |
| Infrastructure | Timeseries: CPU, memory, pod count |

### Infrastructure Overview Dashboard

For the ops team monitoring the fleet:

| Section | Widgets |
|---------|---------|
| Status | Monitor summary: how many monitors are OK/WARN/ALERT |
| Compute | Heatmap: CPU distribution across hosts. Top list: busiest hosts |
| Memory | Timeseries: memory usage by host group. Query value: total memory pressure |
| Disk | Timeseries: disk utilization. Forecast: days until full |
| Network | Timeseries: network bytes in/out, packet drops, retransmits |
| Events | Event stream: recent alerts, host shutdowns, scaling events |

### Business KPI Dashboard

For managers and stakeholders:

| Section | Widgets |
|---------|---------|
| Revenue | Query value: orders/minute. Change widget: vs last week |
| Availability | SLO widget: 99.9% target. Query value: current uptime |
| User Experience | Timeseries: page load time. Query value: Apdex score |
| Traffic | Timeseries: active users. Change: vs same day last week |
| Errors | Query value: user-facing error rate. Top list: most common errors |

## Sharing and Access

### Sharing Options

| Method | Visibility | Use Case |
|--------|-----------|----------|
| Dashboard link | Anyone in your Datadog org | Team collaboration |
| Public URL | Anyone with the link (read-only, no auth) | NOC screens, stakeholders without Datadog access |
| Scheduled report | Email, Slack (PNG snapshot) | Weekly status updates |
| Embed | iframe in wiki/internal tools | Internal portals |

### Public URLs

Generate a public URL to share a read-only view without requiring Datadog login:

1. Open the dashboard
2. Click the gear icon → "Generate Public URL"
3. Optionally set an expiration date

Public dashboards respect template variable defaults but viewers cannot change filters.

### Scheduled Reports (Snapshots)

Send automatic snapshots of your dashboard at fixed intervals:

- Daily/weekly/monthly
- To Slack channels or email
- Useful for "Monday morning status" or "end-of-sprint review"

## API Management

### Create a Dashboard via API

```bash
curl -X POST "https://api.datadoghq.com/api/v1/dashboard" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Web Service Health",
    "layout_type": "ordered",
    "widgets": [
      {
        "definition": {
          "type": "timeseries",
          "requests": [
            {
              "q": "avg:system.cpu.user{service:web} by {host}",
              "display_type": "line"
            }
          ],
          "title": "CPU by Host"
        }
      }
    ],
    "template_variables": [
      {
        "name": "env",
        "prefix": "env",
        "default": "production"
      }
    ]
  }'
```

### Export / Import Dashboards

```bash
# Export a dashboard to JSON
curl -s "https://api.datadoghq.com/api/v1/dashboard/${DASHBOARD_ID}" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" | jq '.' > dashboard.json

# Import it back (e.g., to another org or environment)
curl -X POST "https://api.datadoghq.com/api/v1/dashboard" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -H "Content-Type: application/json" \
  -d @dashboard.json
```

### Terraform

```hcl
resource "datadog_dashboard" "web_service" {
  title       = "Web Service Health"
  layout_type = "ordered"

  template_variable {
    name    = "env"
    prefix  = "env"
    default = "production"
  }

  widget {
    timeseries_definition {
      title = "Request Rate"
      request {
        q            = "sum:trace.http.request.hits{env:$env,service:web}.as_count()"
        display_type = "bars"
      }
    }
  }

  widget {
    query_value_definition {
      title = "Error Rate %"
      request {
        q          = "(sum:trace.http.request.errors{env:$env,service:web}.as_count() / sum:trace.http.request.hits{env:$env,service:web}.as_count()) * 100"
        aggregator = "avg"
      }
      precision = 2
    }
  }
}
```

## Tips and Best Practices

### Do

- **Name dashboards clearly** — Include the service or team name: "Platform — Kubernetes Cluster Health"
- **Add a Notes widget at the top** — Explain what the dashboard is for and who owns it
- **Use conditional formatting** on query values — Green/yellow/red makes status instantly visible
- **Keep dashboards focused** — One dashboard per concern. A dashboard that monitors everything monitors nothing.
- **Set meaningful defaults for template variables** — The dashboard should be useful without touching any filters
- **Use consistent time ranges** — If most graphs show 4 hours, don't have one showing 24 hours without reason
- **Add event overlays** — Show deploys, alerts, and config changes on timeseries graphs
- **Review and prune regularly** — Remove widgets that nobody looks at

### Don't

- **Don't put 50 graphs on one dashboard** — If you need to scroll for 30 seconds, split it into multiple dashboards
- **Don't use raw counters without context** — "5000 errors" means nothing without knowing the total request count. Show rates or percentages.
- **Don't rely on color alone** — Add thresholds, markers, and text for accessibility
- **Don't duplicate metrics across many dashboards** — Use PowerPacks for reusable widget groups
- **Don't leave graphs without titles** — Every widget should have a human-readable title describing what it shows
- **Don't use overly complex queries** — If a query needs 5 formulas and 3 functions, split it into multiple widgets

### Performance

- Dashboards with 40+ widgets load slowly — keep it under 30 when possible
- Use rollup to reduce data points on wide time ranges: `.rollup(avg, 300)` for 5-minute buckets
- Avoid `by {*}` groupings that explode cardinality — group by specific tags

## Integrations: Where Dashboard Data Comes From

Dashboards visualize data — but that data has to come from somewhere. Datadog integrations are the pipeline that feeds metrics, logs, and traces into your widgets.

### Integration Types

| Type | How It Works | Examples |
|------|-------------|----------|
| **Agent-based** | Installed alongside the Datadog Agent. A Python `check` class collects metrics on a schedule. | Redis, PostgreSQL, Nginx, Docker, system checks |
| **Authentication / Crawler** | Configured in the Datadog UI with API credentials. Datadog crawls the provider's API. | AWS, Azure, GCP, Slack, PagerDuty |
| **Library** | Embedded in your application code via the Datadog SDK/API. | APM tracing (Python, Node.js, Java, Go), custom metrics via DogStatsD |

### How Integrations Feed Dashboards

```
┌─────────────────────────┐     ┌───────────────┐     ┌─────────────────┐
│ Agent-based             │     │               │     │                 │
│  system.cpu.user        │────▶│               │     │   Dashboard     │
│  redis.info.clients     │     │               │     │                 │
├─────────────────────────┤     │   Datadog     │     │  ┌───────────┐  │
│ Crawler-based           │────▶│   Backend     │────▶│  │ Widget A  │  │
│  aws.ec2.cpuutilization │     │               │     │  │ Widget B  │  │
├─────────────────────────┤     │               │     │  │ Widget C  │  │
│ Library / APM           │────▶│               │     │  └───────────┘  │
│  trace.http.request.hits│     │               │     │                 │
│  custom.metric.name     │     └───────────────┘     └─────────────────┘
└─────────────────────────┘
```

### Out-of-the-Box Dashboards

Every integration you enable comes with a pre-built dashboard. These are useful starting points:

- **AWS EC2** → CPU, network, disk, status checks across your fleet
- **PostgreSQL** → Connections, query throughput, replication lag, locks
- **Nginx** → Requests/sec, active connections, response codes
- **Kubernetes** → Pod status, resource usage, node health
- **Docker** → Container CPU/memory, image counts, running vs stopped

Find them at: **Dashboards** → **Dashboard List** → search by integration name. They're tagged `Preset`.

> **Tip:** Don't edit preset dashboards directly — clone them first (gear icon → "Clone dashboard"), then customize the copy. Preset dashboards can be overwritten by Datadog during updates.

### Custom Agent Checks

When the 800+ built-in integrations don't cover your use case, you can build your own Agent check. Common reasons:

- Monitoring a proprietary internal service
- Pulling metrics from a custom API endpoint
- Collecting business-specific KPIs from a database query

A minimal custom check:

```python
# /etc/datadog-agent/checks.d/my_service.py
from datadog_checks.base import AgentCheck

class MyServiceCheck(AgentCheck):
    def check(self, instance):
        # Collect your metric
        value = self._get_metric_from_service()
        self.gauge('my_service.response_time', value, tags=['env:production'])

    def _get_metric_from_service(self):
        # Your collection logic here
        import requests
        resp = requests.get('http://localhost:8080/health')
        return resp.elapsed.total_seconds()
```

```yaml
# /etc/datadog-agent/conf.d/my_service.d/conf.yaml
instances:
  - min_collection_interval: 30
```

Once the check is running, the metric `my_service.response_time` becomes available in dashboard widgets just like any built-in metric.

### Metric Naming Conventions

Understanding the naming pattern helps when building dashboard queries:

| Pattern | Source | Example |
|---------|--------|---------|
| `system.*` | Agent system checks | `system.cpu.user`, `system.mem.used` |
| `aws.*` | AWS integration (crawler) | `aws.ec2.cpuutilization`, `aws.elb.request_count` |
| `trace.*` | APM library integration | `trace.http.request.hits`, `trace.flask.request.duration` |
| `docker.*` | Docker Agent check | `docker.cpu.usage`, `docker.mem.rss` |
| `kubernetes.*` | Kubernetes Agent check | `kubernetes.pods.running`, `kubernetes.cpu.requests` |
| `<integration>.*` | Named integration | `redis.info.clients`, `nginx.net.connections` |
| `custom.*` or any name | DogStatsD / custom checks | `custom.orders.count`, `my_service.response_time` |

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Dashboard type | Timeboard for debugging (synced time), Screenboard for presentation |
| Widget choice | Match the widget to the question: "how much now?" = Query Value, "what's the trend?" = Timeseries, "where?" = Top List |
| Template variables | Make one dashboard serve multiple environments/services with dropdown filters |
| Layout | Inverted pyramid: status → trends → breakdown → detail |
| Notes | Always add context — who owns this, what it monitors, links to runbooks |
| Sharing | Public URLs for non-Datadog users, scheduled reports for recurring updates |
| Maintenance | Review quarterly, remove unused widgets, update as services evolve |
