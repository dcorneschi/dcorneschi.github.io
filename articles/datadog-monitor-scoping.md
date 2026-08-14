# How Datadog Monitors Apply to All Hosts Automatically

## Overview

When you create a Datadog monitor, you don't need to list every host it should apply to. Datadog monitors are dynamic — they automatically evaluate against any host (or resource) that reports the relevant metric. This article explains how query scoping works and how to use it effectively.

## The Query Structure

A typical Datadog monitor query looks like this:

```
avg(last_5m):avg:system.cpu.idle{*} by {host} > 90
```

There are two critical parts that control scope:

### `{*}` — The Filter (Scope)

The curly braces define **which data sources** the monitor evaluates. The wildcard `*` means "all hosts reporting this metric" with no filter applied.

- `{*}` — all hosts, no restrictions
- `{env:production}` — only hosts tagged with `env:production`
- `{host:srv-prod-01}` — one specific host
- `{team:backend,env:staging}` — hosts matching both tags

### `by {host}` — The Grouping

The `by` clause defines **how the monitor evaluates thresholds**. With `by {host}`, each host is evaluated independently and gets its own alert state.

Without `by {host}`, all hosts would be averaged together into a single value — rarely what you want.

## How Auto-Discovery Works

When you combine `{*}` with `by {host}`:

1. You deploy a new server and install the Datadog agent
2. The agent starts sending metrics (`system.cpu.idle`, `system.mem.pct_usable`, etc.)
3. Datadog automatically evaluates your existing monitors against the new host's data
4. If that host breaches a threshold, you get an alert specifically for that host

**No extra configuration needed.** You don't need to update Terraform, add the host to a list, or modify the monitor in any way. The monitor dynamically covers your entire fleet.

## Practical Examples

### General-purpose (applies to all hosts)

```hcl
query = "avg(last_5m):100 - avg:system.cpu.idle{*} by {host} > 90"
```

Every host running a Datadog agent is covered. New hosts are picked up automatically.

### Scoped to an environment

```hcl
query = "avg(last_5m):avg:system.mem.pct_usable{env:production} by {host} < 0.1"
```

Only hosts tagged `env:production` are evaluated. Staging/dev hosts are excluded.

### Scoped to a single host

```hcl
query = "avg(last_5m):avg:system.net.conntrack.count{host:srv-prod-01} > 90"
```

Only one specific host is monitored. This is rarely ideal — prefer tag-based scoping.

### Multi-dimensional grouping

```hcl
query = "avg(last_5m):avg:system.disk.in_use{*} by {host,device} > 0.85"
```

Evaluates per host AND per disk device. You get separate alerts for `/dev/vda1` and `/dev/vdb1` on the same host.

## No Data Behavior with `{*}`

When using wildcard scoping, hosts can silently disappear. If a host stops reporting metrics (crash, decommission, network issue), the monitor enters a **No Data** state for that group.

Key settings:

| Setting | Default | Recommendation |
|---------|---------|----------------|
| `notify_no_data` | `false` | Set to `true` for infrastructure monitors |
| `no_data_timeframe` | 2x evaluation window | 10-15 minutes for most host metrics |

```hcl
notify_no_data    = true
no_data_timeframe = 10
```

Without `notify_no_data: true`, a host can go offline and you'll never know — the monitor simply stops evaluating it. This is especially dangerous with `{*}` scoping because you have no static list to compare against.

---

## Tag Inheritance and Sources

The tags you use in monitor filters (`{env:production}`) can originate from multiple sources:

| Source | Example Tags | Applied How |
|--------|-------------|-------------|
| Agent config (`datadog.yaml`) | `env:production`, `team:platform` | `tags:` key in agent config |
| AWS integration | `instance-type:m5.xlarge`, `availability-zone:eu-west-1a` | Auto-collected from EC2 metadata |
| Azure integration | `resource_group:myRG`, `vm_size:Standard_B2s` | Auto-collected from Azure metadata |
| Kubernetes | `kube_namespace:default`, `pod_name:myapp-xyz` | Auto-collected by DaemonSet agent |
| Host labels (API) | Any custom tag | Applied via API or Terraform |

Common issues:
- Tag appears in the host list but **not in the metric** — the integration tag may not propagate to all metrics
- Tag delay — new hosts may take 1-2 minutes before tags are fully indexed
- Case sensitivity — tags are lowercase; `Env:Production` won't match `env:production`

---

## New Group Delay

When a new host starts reporting, metrics are often unstable during boot (high CPU from init scripts, memory settling, disk I/O from provisioning). Without protection, you'll get spurious alerts.

The `new_group_delay` parameter suppresses evaluation for new groups:

```hcl
new_group_delay = 300  # seconds — wait 5 minutes before evaluating a new host
```

This applies when:
- A new host appears (new group in `by {host}`)
- A new device appears (new group in `by {host,device}`)
- A new container/pod appears

**Don't confuse with:**
- `evaluation_delay` — delays evaluation for all groups (useful for cloud metrics that arrive late)
- `renotify_interval` — controls re-notification frequency for already-alerting groups

---

## Excluding Hosts and Tags

Use negation in the filter to exclude specific hosts or tags from a wildcard monitor:

```
# Exclude dev environment
avg(last_5m):100 - avg:system.cpu.idle{!env:dev} by {host} > 90

# Exclude specific hosts
avg(last_5m):avg:system.disk.in_use{!host:bastion-01} by {host,device} > 0.85

# Combine inclusion and exclusion
avg(last_5m):avg:system.mem.pct_usable{env:production,!team:sandbox} by {host} < 0.1
```

Negation rules:
- `!tag:value` — exclude hosts with this tag
- Multiple negations are AND-ed: `{!env:dev,!env:staging}` excludes both
- You can mix inclusion and exclusion: `{env:production,!role:batch}`

---

## Terraform Example

A complete `datadog_monitor` resource showing the full pattern:

```hcl
resource "datadog_monitor" "high_cpu" {
  name    = "High CPU Usage on {{host.name}}"
  type    = "metric alert"
  message = <<-EOT
    CPU usage above 90% for 5 minutes on {{host.name}}.

    Host: {{host.name}}
    Environment: {{env.name}}
    IP: {{host.ip}}

    @slack-alerts-infra
    @pagerduty-infra
  EOT

  query = "avg(last_5m):100 - avg:system.cpu.idle{*} by {host} > 90"

  monitor_thresholds {
    critical          = 90
    critical_recovery = 75
    warning           = 80
    warning_recovery  = 70
  }

  notify_no_data    = true
  no_data_timeframe = 10
  new_group_delay   = 300
  renotify_interval = 60
  timeout_h         = 1

  include_tags        = true
  require_full_window = false

  tags = [
    "team:platform",
    "managed-by:terraform",
    "category:infrastructure",
  ]
}
```

Key Terraform patterns:
- Use `{*} by {host}` — monitor auto-discovers all hosts
- Set `notify_no_data = true` — catch hosts going offline
- Set `new_group_delay` — avoid boot-time false positives
- Use template variables (`{{host.name}}`) in the message for context
- Tag your monitors for ownership and filtering in the Datadog UI

---

## Best Practices

| Practice | Reason |
|----------|--------|
| Use `{*} by {host}` for infrastructure monitors | Automatically covers new hosts |
| Use tag filters (`{env:production}`) over host filters | Scales with your fleet |
| Always include `by {host}` for host-level alerts | Avoids averaging across hosts |
| Use `by {host,device}` for disk monitors | Different devices have different capacities |
| Avoid hardcoding hostnames in queries | Breaks when hosts are replaced |

## When to Use Specific Scoping

General `{*}` monitors work well for:
- CPU, memory, disk, load — universal infrastructure metrics
- Agent health checks (host reporting, NTP sync)
- Network error rates

Use tag-based scoping for:
- Environment-specific thresholds (prod tighter than staging)
- Team-owned infrastructure
- Service-specific containers or processes

## Summary

Datadog monitors with `{*} by {host}` are "deploy once, monitor everything" — they automatically apply to any host reporting the metric. This is the recommended approach for infrastructure monitors managed via Terraform, as it eliminates the need to update monitor definitions when scaling your fleet.
