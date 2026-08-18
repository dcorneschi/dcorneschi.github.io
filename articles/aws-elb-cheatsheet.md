# AWS Load Balancer Cheatsheet

Elastic Load Balancing (ELB) distributes traffic across targets in multiple Availability Zones. AWS offers three load balancer types: Application Load Balancer (ALB), Network Load Balancer (NLB), and Gateway Load Balancer (GWLB).

## Load Balancer Types

AWS offers four types of Elastic Load Balancer:

- **Classic Load Balancer (CLB)** — oldest, basic load balancing at layer 4 and layer 7 (legacy, avoid for new workloads)
- **Application Load Balancer (ALB)** — layer 7, routes based on HTTP content (host, path, headers)
- **Network Load Balancer (NLB)** — layer 4, routes based on IP protocol data (TCP/UDP)
- **Gateway Load Balancer (GWLB)** — layer 3 gateway + layer 4 load balancing, used in front of virtual appliances (firewalls, IDS/IPS)

### Feature Comparison

| Feature | ALB | NLB | CLB | GWLB |
|---------|-----|-----|-----|------|
| OSI Layer | Layer 7 | Layer 4 | Layer 4/7 | Layer 3 Gateway + Layer 4 LB |
| Target Type | IP, Instance, Lambda | IP, Instance, ALB | N/A | IP, Instance |
| Protocols | HTTP, HTTPS | TCP, UDP, TLS | TCP, SSL, HTTP, HTTPS | IP |
| WebSockets | Yes | Yes | No | Yes |
| IP addresses as target | Yes | Yes | No | No |
| HTTP header-based routing | Yes | No | No | No |
| HTTP/2 / gRPC | Yes | No | No | No |
| Configurable idle timeout | Yes | No | Yes | No |
| Cross-zone load balancing | Yes (always on) | Yes (off by default) | Yes | Yes |
| SSL Offloading | Yes | Yes | Yes | No |
| Server Name Indication (SNI) | Yes | Yes | Yes | No |
| Sticky sessions | Yes | Yes | Yes | Yes |
| Static / Elastic IP | No | Yes (one per AZ) | No | No |
| Custom security policies | No | No | Yes | No |
| Preserve source IP | No (X-Forwarded-For) | Yes | No | Yes |
| Latency | ~milliseconds | ~microseconds | Higher | Low |
| Best for | Web apps, microservices, APIs | High perf, TCP/UDP, static IPs | Legacy | Firewalls, IDS/IPS appliances |

## Application Load Balancer (ALB)

### Create ALB

```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name my-alb \
  --type application \
  --subnets subnet-aaa subnet-bbb subnet-ccc \
  --security-groups sg-123

# Create internet-facing ALB
aws elbv2 create-load-balancer \
  --name my-alb \
  --type application \
  --scheme internet-facing \
  --subnets subnet-aaa subnet-bbb \
  --security-groups sg-123

# Create internal ALB
aws elbv2 create-load-balancer \
  --name internal-alb \
  --type application \
  --scheme internal \
  --subnets subnet-aaa subnet-bbb \
  --security-groups sg-456
```

### Target Groups

```bash
# Create target group (IP type — for Fargate/awsvpc)
aws elbv2 create-target-group \
  --name my-app-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-123 \
  --target-type ip \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# Create target group (instance type — for EC2)
aws elbv2 create-target-group \
  --name my-ec2-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-123 \
  --target-type instance

# Create target group (Lambda)
aws elbv2 create-target-group \
  --name my-lambda-tg \
  --target-type lambda

# Register targets
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-app-tg/xxx \
  --targets Id=10.0.1.10,Port=8080 Id=10.0.2.20,Port=8080

# Register EC2 instances
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-ec2-tg/xxx \
  --targets Id=i-1234567890abcdef0 Id=i-0987654321fedcba0

# Deregister targets
aws elbv2 deregister-targets \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-app-tg/xxx \
  --targets Id=10.0.1.10,Port=8080

# Check target health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-app-tg/xxx
```

### Listeners

```bash
# Create HTTP listener (redirect to HTTPS)
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/my-alb/xxx \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'

# Create HTTPS listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/my-alb/xxx \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:eu-west-1:123456789012:certificate/xxx \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/my-app-tg/xxx \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06

# List listeners
aws elbv2 describe-listeners --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/my-alb/xxx
```

### Listener Rules (Routing)

```bash
# Path-based routing
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/app/my-alb/xxx/yyy \
  --priority 10 \
  --conditions Field=path-pattern,Values='/api/*' \
  --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/api-tg/xxx

# Host-based routing
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/app/my-alb/xxx/yyy \
  --priority 20 \
  --conditions Field=host-header,Values='api.example.com' \
  --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/api-tg/xxx

# Header-based routing
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/app/my-alb/xxx/yyy \
  --priority 30 \
  --conditions Field=http-header,HttpHeaderConfig='{HttpHeaderName=X-Custom-Header,Values=["special"]}' \
  --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/special-tg/xxx

# Fixed response (maintenance page)
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/app/my-alb/xxx/yyy \
  --priority 99 \
  --conditions Field=path-pattern,Values='/maintenance' \
  --actions Type=fixed-response,FixedResponseConfig='{StatusCode=503,ContentType=text/html,MessageBody="<h1>Under Maintenance</h1>"}'

# List rules
aws elbv2 describe-rules --listener-arn arn:aws:elasticloadbalancing:...:listener/app/my-alb/xxx/yyy
```

### ALB Routing Capabilities

| Condition | Example |
|-----------|---------|
| Host header | `api.example.com`, `*.example.com` |
| Path pattern | `/api/*`, `/images/*` |
| HTTP header | `X-Custom-Header: value` |
| HTTP method | `GET`, `POST` |
| Query string | `?version=v2` |
| Source IP | `10.0.0.0/8` |

### Stickiness (Session Affinity)

```bash
# Enable stickiness on target group
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-tg/xxx \
  --attributes Key=stickiness.enabled,Value=true Key=stickiness.type,Value=lb_cookie Key=stickiness.lb_cookie.duration_seconds,Value=3600
```

## Network Load Balancer (NLB)

### Create NLB

```bash
# Create NLB (internet-facing)
aws elbv2 create-load-balancer \
  --name my-nlb \
  --type network \
  --subnets subnet-aaa subnet-bbb

# Create NLB with Elastic IPs (static IPs)
aws elbv2 create-load-balancer \
  --name my-nlb \
  --type network \
  --subnet-mappings SubnetId=subnet-aaa,AllocationId=eipalloc-111 SubnetId=subnet-bbb,AllocationId=eipalloc-222

# Create internal NLB
aws elbv2 create-load-balancer \
  --name internal-nlb \
  --type network \
  --scheme internal \
  --subnets subnet-aaa subnet-bbb
```

### NLB Target Groups

```bash
# TCP target group
aws elbv2 create-target-group \
  --name my-tcp-tg \
  --protocol TCP \
  --port 443 \
  --vpc-id vpc-123 \
  --target-type ip \
  --health-check-protocol TCP

# UDP target group
aws elbv2 create-target-group \
  --name my-udp-tg \
  --protocol UDP \
  --port 5060 \
  --vpc-id vpc-123 \
  --target-type ip

# TLS target group (NLB terminates TLS)
aws elbv2 create-target-group \
  --name my-tls-tg \
  --protocol TLS \
  --port 443 \
  --vpc-id vpc-123 \
  --target-type ip

# ALB as target (NLB → ALB pattern)
aws elbv2 create-target-group \
  --name alb-tg \
  --protocol TCP \
  --port 80 \
  --vpc-id vpc-123 \
  --target-type alb
```

### NLB Listeners

```bash
# TCP listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/net/my-nlb/xxx \
  --protocol TCP \
  --port 443 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/my-tcp-tg/xxx

# TLS listener (terminates TLS at NLB)
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/net/my-nlb/xxx \
  --protocol TLS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:eu-west-1:123456789012:certificate/xxx \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/my-tls-tg/xxx \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06

# UDP listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/net/my-nlb/xxx \
  --protocol UDP \
  --port 5060 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/my-udp-tg/xxx
```

## Health Checks

### Configuration

```bash
# Modify health check settings
aws elbv2 modify-target-group \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-tg/xxx \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-interval-seconds 15 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --matcher HttpCode=200-299
```

### Health Check Parameters

| Parameter | ALB | NLB |
|-----------|-----|-----|
| Protocol | HTTP, HTTPS | TCP, HTTP, HTTPS |
| Interval | 5-300s (default: 30s) | 10-30s (default: 30s) |
| Timeout | 2-120s (default: 5s) | N/A (10s for TCP) |
| Healthy threshold | 2-10 (default: 5) | 2-10 (default: 5) |
| Unhealthy threshold | 2-10 (default: 2) | 2-10 (default: 2) |
| Matcher | HTTP codes (200-499) | HTTP codes (200-499) |

## SSL/TLS Certificates

```bash
# List certificates in ACM
aws acm list-certificates --query 'CertificateSummaryList[].{Domain:DomainName,ARN:CertificateArn}' --output table

# Add additional certificate to listener (SNI)
aws elbv2 add-listener-certificates \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/app/my-alb/xxx/yyy \
  --certificates CertificateArn=arn:aws:acm:eu-west-1:123456789012:certificate/zzz

# List listener certificates
aws elbv2 describe-listener-certificates \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/app/my-alb/xxx/yyy
```

### SSL Policies

| Policy | TLS Versions | Use Case |
|--------|-------------|----------|
| `ELBSecurityPolicy-TLS13-1-2-2021-06` | TLS 1.3 + 1.2 | Recommended |
| `ELBSecurityPolicy-TLS-1-2-2017-01` | TLS 1.2 only | Strict compliance |
| `ELBSecurityPolicy-2016-08` | TLS 1.0+ | Legacy clients |
| `ELBSecurityPolicy-FS-1-2-2019-08` | TLS 1.2 + forward secrecy | High security |

## Access Logs

```bash
# Enable ALB access logs
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/my-alb/xxx \
  --attributes Key=access_logs.s3.enabled,Value=true Key=access_logs.s3.bucket,Value=my-lb-logs Key=access_logs.s3.prefix,Value=alb

# Enable NLB access logs
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/net/my-nlb/xxx \
  --attributes Key=access_logs.s3.enabled,Value=true Key=access_logs.s3.bucket,Value=my-lb-logs Key=access_logs.s3.prefix,Value=nlb
```

## Connection Draining (Deregistration Delay)

```bash
# Set deregistration delay (default: 300s)
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-tg/xxx \
  --attributes Key=deregistration_delay.timeout_seconds,Value=60
```

## Cross-Zone Load Balancing

```bash
# Enable cross-zone (ALB: always on; NLB: off by default)
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/net/my-nlb/xxx \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

## WAF Integration (ALB Only)

```bash
# Associate WAF web ACL with ALB
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:eu-west-1:123456789012:regional/webacl/my-waf/xxx \
  --resource-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/my-alb/xxx
```

## Common Patterns

### NLB → ALB (Static IPs + Layer 7 Routing)

Use NLB in front of ALB when you need static IPs combined with host/path-based routing:

```bash
# 1. Create NLB with EIPs
# 2. Create target group with target-type=alb
# 3. Register the ALB as target
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/alb-tg/xxx \
  --targets Id=arn:aws:elasticloadbalancing:...:loadbalancer/app/my-alb/yyy
```

### Weighted Target Groups (Blue/Green, Canary)

```bash
# Forward with weights (90% to blue, 10% to green)
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/app/my-alb/xxx/yyy \
  --default-actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        {"TargetGroupArn": "arn:...blue-tg...", "Weight": 90},
        {"TargetGroupArn": "arn:...green-tg...", "Weight": 10}
      ]
    }
  }]'
```

## List and Describe

```bash
# List all load balancers
aws elbv2 describe-load-balancers
aws elbv2 describe-load-balancers --query 'LoadBalancers[].{Name:LoadBalancerName,Type:Type,DNS:DNSName,Scheme:Scheme}' --output table

# List all target groups
aws elbv2 describe-target-groups
aws elbv2 describe-target-groups --query 'TargetGroups[].{Name:TargetGroupName,Protocol:Protocol,Port:Port,Type:TargetType}' --output table

# Get load balancer attributes
aws elbv2 describe-load-balancer-attributes --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/my-alb/xxx

# Get target group attributes
aws elbv2 describe-target-group-attributes --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-tg/xxx
```

## Delete Resources

```bash
# Delete listener
aws elbv2 delete-listener --listener-arn arn:aws:elasticloadbalancing:...:listener/app/my-alb/xxx/yyy

# Delete target group (must have no targets and not be referenced by a listener)
aws elbv2 delete-target-group --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-tg/xxx

# Delete load balancer
aws elbv2 delete-load-balancer --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/my-alb/xxx
```

## Troubleshooting

| Issue | Investigation |
|-------|---------------|
| 502 Bad Gateway | Target health check failing — check app is listening on the right port |
| 503 Service Unavailable | No healthy targets — check security groups, health check path |
| 504 Gateway Timeout | Target not responding within timeout — check app performance |
| Uneven distribution | Check cross-zone load balancing, target weights, stickiness |
| Connection refused | Security group on targets must allow traffic from LB SG |
| TLS errors | Check certificate matches domain, verify SSL policy supports client |
| Targets stuck draining | Reduce deregistration delay or wait for in-flight requests to complete |

```bash
# Check target health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-tg/xxx

# Check load balancer state
aws elbv2 describe-load-balancers --names my-alb --query 'LoadBalancers[].State'
```

## CloudWatch Metrics

### Common Metrics (All ELB Types)

| Metric | Description |
|--------|-------------|
| `HealthyHostCount` | Number of targets passing health checks |
| `UnHealthyHostCount` | Number of targets failing health checks |
| `RequestCount` | Total requests processed during a period |
| `ActiveConnectionCount` | Concurrent active connections |
| `NewConnectionCount` | New connections established per period |

### ALB Metrics

| Metric | Description |
|--------|-------------|
| `HTTPCode_Target_2XX_Count` | Successful responses from targets |
| `HTTPCode_Target_4XX_Count` | Client errors from targets |
| `HTTPCode_Target_5XX_Count` | Server errors from targets |
| `HTTPCode_ELB_5XX_Count` | Errors generated by the ALB itself (no healthy targets, capacity) |
| `HTTPCode_ELB_4XX_Count` | Client errors at the ALB level (malformed requests) |
| `TargetResponseTime` | Latency — time from request sent to response received |
| `RequestCountPerTarget` | Average requests per target (identifies uneven distribution) |
| `RejectedConnectionCount` | Connections rejected (max connections reached) |
| `RuleEvaluations` | Number of rules processed (complex routing setups) |
| `ConsumedLCUs` | Load Balancer Capacity Units consumed (billing) |

### NLB Metrics

| Metric | Description |
|--------|-------------|
| `ProcessedBytes` | Total bytes processed (TCP/UDP) |
| `TCP_Client_Reset_Count` | RST packets from client to target |
| `TCP_Target_Reset_Count` | RST packets from target to client |
| `TCP_ELB_Reset_Count` | RST packets generated by the NLB |
| `PeakPacketsPerSecond` | Highest PPS rate during a period |
| `ConsumedLCUs` | Capacity units for billing |

### CLB Metrics

| Metric | Description |
|--------|-------------|
| `Latency` | Time from request to response |
| `SurgeQueueLength` | Pending requests queued (max 1024) — key saturation indicator |
| `SpilloverCount` | Requests rejected because surge queue is full — dropping traffic |
| `BackendConnectionErrors` | Failed connections to instances |

### Key Monitoring Patterns

| Signal | Meaning |
|--------|---------|
| High `HTTPCode_ELB_5XX` | No healthy targets, or ALB can't handle the load |
| Rising `TargetResponseTime` | Targets becoming slow — consider scaling |
| `UnHealthyHostCount` > 0 | Investigate failing instances immediately |
| `SurgeQueueLength` climbing (CLB) | CLB saturated — migrate to ALB/NLB or scale backends |
| `SpilloverCount` > 0 (CLB) | Actively dropping requests — critical |
| `ConsumedLCUs` trending up | Watch for billing surprises; proxy for load growth |

All metrics are available in CloudWatch for alarms, dashboards, and auto-scaling policies.

### Spike Impact Analysis

What each metric spike means and its impact on your service:

#### Health and Capacity

| Metric | Impact of Spikes |
|--------|------------------|
| `HealthyHostCount` drop | Fewer pods serving traffic → remaining pods overloaded, latency increases, potential cascading failures |
| `UnHealthyHostCount` spike | Multiple pods/nodes failing simultaneously — deployment issue, resource exhaustion, or connectivity problem |
| `RequestCount` spike | Overwhelms backends, exhausts connection pools, triggers autoscaling delays or OOM kills |
| `TargetResponseTime` spike | Backend saturation — slow page loads, timeouts, potential ALB 504 errors |

#### Connections and Throughput

| Metric | Impact of Spikes |
|--------|------------------|
| `ActiveConnectionCount` spike | Can exhaust ALB connection limits or backend connection pools, causing 503 errors |
| `NewConnectionCount` spike | May indicate DDoS, bot surge, or sudden user influx — high CPU on ALB and targets |
| `TargetResponseTime` p99/p95 spike | Significant portion of users experiencing severe delays — resource contention, GC, or DB locks |
| `ProcessedBytes` spike | Large payload transfers — possible data export abuse, large uploads, or network saturation |

#### HTTP Response Codes

| Metric | Impact of Spikes |
|--------|------------------|
| `httpcode_target_2xx` spike | Generally healthy, but alongside high latency may indicate retry storms |
| `httpcode_target_3xx` spike | Redirect loops or misconfigured routing — extra latency, possible infinite loops |
| `httpcode_target_4xx` spike | Broken clients, expired tokens, or API contract changes — user-facing errors |
| `httpcode_target_5xx` spike | **Critical** — application crashing or timing out, direct user failures, SLA breach |
| `httpcode_elb_3xx` spike | Usually benign (HTTP→HTTPS), but could mean misconfigured listener rules |
| `httpcode_elb_4xx` spike | Requests rejected before reaching app — possible attack, WAF blocks, client misconfiguration |
| `httpcode_elb_5xx` spike | **Critical** — ALB cannot forward traffic (no healthy targets, target timeout) — complete service outage |

**Incident priority:** Focus on `httpcode_elb_5xx` + `UnHealthyHostCount` first (infrastructure-level failures), then `httpcode_target_5xx` (application-level).

## Best Practices

| Area | Recommendation |
|------|----------------|
| High availability | Use at least 2 AZs for the LB |
| Security | Always redirect HTTP → HTTPS |
| SSL | Use TLS 1.3 policy, ACM for free certificates |
| Health checks | Use a dedicated `/health` endpoint, not `/` |
| Draining | Set deregistration delay to match app graceful shutdown time |
| Access logs | Enable and ship to S3 for debugging |
| Cross-zone | Enable for NLB if targets are unevenly distributed across AZs |
| WAF | Associate WAF with ALB for web application protection |
| Monitoring | Set CloudWatch alarms on `UnHealthyHostCount`, `5XXCount`, `TargetResponseTime` |
| Cost | Delete idle load balancers — they charge hourly even with no traffic |
