# HAProxy Ingress Dashboard Metrics

A reference guide for understanding HAProxy ingress controller metrics on a Datadog dashboard — covering connection, latency, throughput, error, and infrastructure metrics across the ALB → HAProxy → Pod request flow.

## Request Flow: Client → ALB → HAProxy → Pods

```
┌──────────┐       ┌─────────────────────┐       ┌──────────────────────────────┐       ┌──────────────────┐
│          │ HTTPS │                     │ HTTP  │                              │ HTTP  │                  │
│  Client  │─────▶ │  AWS ALB            │─────▶ │  HAProxy Ingress Controller  │──────▶│  Backend Pods    │
│          │       │  (Load Balancer)    │       │  (Kubernetes DaemonSet/      │       │  (app services)  │
└──────────┘       └─────────────────────┘       │   Deployment)                │       └──────────────────┘
                                                 └──────────────────────────────┘
```

| Layer | Role | Key Dashboard Metrics |
|-------|------|----------------------|
| **ALB** | Terminates TLS, distributes traffic across HAProxy pods, health-checks targets | Active connections, 5xx (502/503/504), target health, connection errors, response time |
| **HAProxy** | Kubernetes ingress controller — routes requests to the correct backend service based on rules | Sessions (current vs. limit), frontend request rate, queue time/count, CPU/MEM, pod count, restarts |
| **Backend Pods** | Your application services processing the actual requests | Backend connect time, response time, 2xx/4xx/5xx responses, bytes in/out |

Failure cascade direction:

```
Backend slow/down → HAProxy queues fill → Sessions saturate → ALB gets 5xx/timeouts → Client errors
```

## HAProxy: Frontend vs Backend

```
                         HAProxy Ingress Controller
                   ┌─────────────────────────────────────────────────┐
                   │                                                 │
                   │   FRONTEND              BACKEND                 │
                   │   (_front_https)        (per service)           │
                   │                                                 │
 Client ──▶ ALB ──▶│   Accepts traffic    ──▶│  Routes to pods       │──▶ Pods
                   │   Sessions & limits     │   Connect & respond   │
                   │   Request rate          │   Queue management    │
                   │   Total responses       │   Per-service errors  │
                   │                                                 │
                   └─────────────────────────────────────────────────┘
```

| | **Frontend** | **Backend** |
|---|---|---|
| **What it is** | Single shared HTTPS listener (`_front_https`) | One logical backend per Kubernetes service |
| **Grouped by** | Not grouped (aggregate view) | `by {service}` — shows each app separately |
| **Metrics** | `haproxy.frontend.session.current/limit`, `frontend.requests.rate`, `frontend.response.*` | `haproxy.backend.connect.time`, `backend.response.time`, `backend.queue.*`, `backend.response.4xx/5xx`, `backend.bytes.*` |
| **Spike means** | Total inbound traffic surge hitting HAProxy | A specific backend service is slow, queuing, or erroring |
| **Action** | Scale HAProxy pods or check ALB | Investigate the specific service pod health |

Reading the dashboard with this in mind:

- **Frontend widgets** (Sessions, Session Limits, Request Rate, Total Responses) = "How much traffic is coming in?"
- **Backend widgets** (Response Time, Connect Time, Queue, 4xx/5xx, Bytes) = "How are individual services handling it?"

If frontend is spiking but backend is fine → traffic surge, HAProxy coping well.
If backend is spiking but frontend is normal → a specific service has a problem (check which `service` tag is elevated).

## Connection and Session Metrics

| Widget | Metric | What It Measures |
|--------|--------|------------------|
| **Backend connection time** | `haproxy.backend.connect.time` | Time (ms) for HAProxy to establish a TCP connection to the backend server |
| **Backend connection session** | `haproxy.backend.session.current` | Number of active sessions on each backend |
| **HAProxy Sessions** | `haproxy.frontend.session.current` + `haproxy.backend.session.current` | Total active sessions at the frontend (HTTPS) and backend layers |
| **HAProxy Session Limits** | Frontend/backend sessions vs. `haproxy.frontend.session.limit` | How close current sessions are to the configured maximum |
| **ALB connections** | `aws.applicationelb.active_connection_count` | Active TCP connections on the AWS ALB upstream of HAProxy |

Impact of spikes:

- **Connection time spike** → Backend pods are overloaded, experiencing network issues, or starting up slowly. Users see increased latency.
- **Session spike** → Traffic surge or slow backends holding sessions open longer. If sessions approach the **session limit**, new connections will be **rejected** (503 errors).
- **ALB connection spike** → High inbound traffic volume hitting your cluster.

## Latency Metrics

| Widget | Metric | What It Measures |
|--------|--------|------------------|
| **Backend response time** | `haproxy.backend.response.time` | End-to-end time from request sent to backend until full response received |
| **Queue time** | `haproxy.backend.queue.time` | Time requests spend waiting in the HAProxy queue before being dispatched |
| **Queue count** | `haproxy.backend.queue.current` | Number of requests currently queued |
| **LB response times** | ALB p99 + avg response time, HAProxy avg + max response time | Full-stack latency from ALB → HAProxy → backend |

Impact of spikes:

- **Response time spike** → Backend services are slow; directly causes user-facing latency.
- **Queue time/count spike** → All backend server slots are full; requests are **waiting in line**. This is a leading indicator of saturation — if sustained, it leads to timeouts and 5xx errors.
- **LB response time spike** → End-to-end degradation visible to clients; could be ALB-level, HAProxy-level, or backend-level.

## Throughput and Traffic Metrics

| Widget | Metric | What It Measures |
|--------|--------|------------------|
| **Bytes sent/received** | `haproxy.backend.bytes.out_rate` / `in_rate` | Data transfer rate to/from backends |
| **Frontend Request Rate** | `haproxy.frontend.requests.rate` | Requests per second hitting the HAProxy frontend |
| **2xx Responses** | `haproxy.backend.response.2xx` | Successful responses |
| **Total Responses** | Frontend 1xx/2xx/3xx/4xx/5xx/other | Full breakdown of response codes at the frontend |

Impact of spikes:

- **Bytes spike** → Large payloads or high request volume; can saturate network bandwidth or trigger pod resource limits.
- **Request rate spike** → Traffic surge (legitimate or attack). May cascade into session/queue/latency spikes if capacity isn't sufficient.
- **2xx spike** → Generally healthy — just more successful traffic.

## Error Metrics

| Widget | Metric | What It Measures |
|--------|--------|------------------|
| **4xx Responses** | `haproxy.backend.response.4xx` | Client errors (bad requests, unauthorized, not found) |
| **5xx Responses** | `haproxy.backend.response.5xx` | Server errors from HAProxy backends |
| **5xx Responses (ALB)** | `aws.applicationelb.httpcode_target_5xx`, `httpcode_elb_5xx`, 500/502/503/504 | Server errors at the ALB layer |
| **ALB Errors** | `aws.applicationelb.target_connection_error_count` | ALB failed to connect to a target (HAProxy pod) |

Impact of spikes:

- **4xx spike** → Misconfigured clients, bad routes, or authentication issues. Usually not a backend problem but impacts user experience.
- **5xx spike (HAProxy)** → Backend pods are crashing, timing out, or returning errors. **Direct customer impact** — users see failures.
- **5xx spike (ALB)** → More severe: 502 = bad gateway (HAProxy down), 503 = no healthy targets, 504 = timeout. Indicates HAProxy itself is unhealthy or unreachable.
- **ALB connection errors spike** → ALB cannot reach HAProxy pods at all — possibly pods are crashing, not passing health checks, or being terminated.

## Infrastructure and Resource Metrics

| Widget | Metric | What It Measures |
|--------|--------|------------------|
| **HAProxy CPU** | `kubernetes.cpu.usage.total` (haproxy pods) | CPU consumed by each HAProxy ingress controller pod |
| **HAProxy MEM** | `kubernetes.memory.usage` (haproxy pods) | Memory consumed by each HAProxy pod |
| **HAProxy pod count** | `kubernetes.pods.running` (haproxy-ingress) | Number of running HAProxy pods |
| **HAProxy Pod Restarts** | `kubernetes.containers.restarts` (haproxy-ingress) | Cumulative container restart count |
| **ALB target health** | `aws.applicationelb.healthy_host_count` / `un_healthy_host_count` | How many HAProxy targets the ALB considers healthy vs. unhealthy |

Impact of spikes:

- **CPU spike** → HAProxy is compute-bound processing traffic; if it hits limits, pods get throttled → latency increases or pods get OOMKilled.
- **Memory spike** → Risk of OOMKill if the pod exceeds its memory limit, causing an abrupt restart and dropped connections.
- **Pod restarts spike** → HAProxy pods are crashing. **Every restart drops all active connections** on that pod, causing a wave of 502/503 errors.
- **Unhealthy hosts spike** → ALB is removing HAProxy pods from rotation, concentrating traffic on fewer pods (cascading overload risk).
- **Pod count drop** → Fewer pods sharing the load; correlates with capacity issues above.

## Error Code Origin Guide

```
  Client ──────▶ AWS ALB ──────▶ HAProxy FRONTEND ──────▶ HAProxy BACKEND ──────▶ Pods
                   │               (_front_https)              (per service)          │
                   │                     │                          │                 │
                   ▼                     ▼                          ▼                 ▼
              ┌─────────┐         ┌────────────┐            ┌────────────┐     ┌──────────┐
              │ 502: Bad│         │ 503: Sess- │            │ 502: Pod   │     │ 500: App │
              │  gateway│         │  ion limit │            │ unreachable│     │ exception│
              │  (no HAP│         │  reached   │            │            │     │          │
              │  roxy)  │         │            │            │ 503: No    │     │ 4xx: App │
              │         │         │ 5xx: Proxy │            │  available │     │  rejects │
              │ 503: No │         │  overload  │            │  servers   │     │  request │
              │  healthy│         │            │            │            │     │          │
              │  targets│         │ 4xx: Bad   │            │ 504: Pod   │     │ OOMKill/ │
              │         │         │  route/auth│            │  timeout   │     │  crash   │
              │ 504: HAP│         │            │            │            │     │  (502)   │
              │  roxy   │         │            │            │ 4xx: Pass- │     │          │
              │  timeout│         │            │            │  through   │     │          │
              └─────────┘         └────────────┘            └────────────┘     └──────────┘
```

| Code | ALB Generated | Frontend Generated | Backend Generated | Pod Generated |
|------|--------------|-------------------|-------------------|---------------|
| **400** | - | Bad request format | - | App validation failure |
| **401/403** | - | Auth rule rejection | - | App auth/permission |
| **404** | - | No matching route | - | App endpoint not found |
| **500** | - | HAProxy internal error | - | Application exception |
| **502** | HAProxy pod down/crashing | - | Cannot connect to pod | Pod crash mid-request |
| **503** | No healthy HAProxy targets | Session limit hit | No available backend servers | App overloaded |
| **504** | HAProxy response timeout | - | Pod response timeout | - |

## Quick Troubleshooting

- **5xx on ALB widget but not HAProxy** → Problem is between ALB and HAProxy (pods restarting, health check failing)
- **5xx on Frontend but not Backend** → HAProxy itself is saturated (check session limits)
- **5xx on Backend only** → Specific service pods are failing (check which `service` tag)
- **4xx on Backend** → Application-level rejections (not an infra problem)

## Typical Spike Cascade

A common failure pattern on this dashboard:

```
Traffic spike → Sessions approach limit → Queue builds up → Response time increases → CPU/MEM spike → Pod restarts → Unhealthy hosts → 5xx errors
```

The dashboard is designed to let you trace this chain from left to right.

## Datadog Integration Setup

To enable HAProxy monitoring in Datadog:

1. **Enable the HAProxy stats endpoint** (already exposed on port 9153 or 8404 by most Helm charts)
2. **Install the Datadog Agent** on the cluster with HAProxy integration enabled
3. **Configure the integration** in `haproxy.d/conf.yaml`:

```yaml
instances:
  - url: http://%%host%%:8404/stats
    # Or use annotations for auto-discovery:
    # ad.datadoghq.com/haproxy.check_names: '["haproxy"]'
    # ad.datadoghq.com/haproxy.init_configs: '[{}]'
    # ad.datadoghq.com/haproxy.instances: '[{"url":"http://%%host%%:8404/stats"}]'
```

Or annotate the HAProxy pods for Datadog auto-discovery:

```bash
kubectl annotate pod -l app=haproxy -n haproxy-ingress \
  ad.datadoghq.com/haproxy.check_names='["haproxy"]' \
  ad.datadoghq.com/haproxy.init_configs='[{}]' \
  ad.datadoghq.com/haproxy.instances='[{"url":"http://%%host%%:8404/stats"}]'
```

## Key Performance Indicators (KPIs)

| KPI | Formula | Healthy |
|-----|---------|---------|
| Availability | `servers_up / (servers_up + servers_down)` | > 0.99 |
| Error Rate | `(4xx + 5xx) / total_requests` | < 0.01 |
| Queue Saturation | `queue_current / maxconn` | < 0.5 |
| Session Utilization | `session_current / session_limit` | < 0.8 |

## Recommended Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| Backend servers down | `haproxy.backend.servers_down > 0` | Critical |
| High 5xx rate | `rate(backend.response.5xx) > 10/min` | Critical |
| Session limit approaching | `frontend.session.current > 80% of limit` | Warning |
| Response time degradation | `backend.response.time > 2s (p99)` | Warning |
| Queue depth high | `backend.queue.current > 50` for 5 min | Warning |
| Pod restarts | `kubernetes.containers.restarts > 0` (haproxy) | Critical |
| Unhealthy targets | `alb.un_healthy_host_count > 0` | Critical |
