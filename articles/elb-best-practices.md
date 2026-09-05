# Elastic Load Balancing Best Practices

Elastic Load Balancing (ELB) is the front door for most workloads on AWS. Getting it right
means picking the correct load balancer type, wiring it into the AWS services that improve
availability, understanding exactly what happens when a target fails, and running it with
proper visibility. This article walks the full picture — portfolio, availability patterns,
failure behavior, serverless targets, and production operations — with the practical
guidance that matters most.

> This is a companion to the [ALB vs NLB comparison](articles/eks-load-balancer-alb-vs-nlb.md)
> and the [DNS routing policies](articles/dns-routing-policies.md) articles; here the focus
> is on operating ELB well, not just choosing a type.

---

## The ELB portfolio

AWS offers four load balancer types. Pick by the layer you need to operate at and the
protocols you carry.

| Type | Layer | Protocols | Best for |
|------|-------|-----------|----------|
| **Application Load Balancer (ALB)** | 7 (request) | HTTP, HTTPS, gRPC | Web apps, microservices, content/host/path routing |
| **Network Load Balancer (NLB)** | 4 (connection) | TCP, TLS, UDP | Extreme scale, low latency, static IPs, PrivateLink |
| **Gateway Load Balancer (GWLB)** | 3 (packet) | All IP traffic | Inserting virtual appliances (firewalls, IDS/IPS) transparently |
| **Classic Load Balancer (CLB)** | 4–7 | HTTP/S, TCP | Legacy EC2-Classic only — avoid for new work |

- **ALB** routes on request content — host, path, headers, query string, HTTP method,
  source IP — and supports advanced rules, redirects, fixed responses, and native
  authentication.
- **NLB** operates at the connection level, preserves the client source IP, gives you a
  **static IP per AZ** (and an Elastic IP option), and scales to millions of requests per
  second with ultra-low latency.
- **GWLB** transparently steers all IP traffic through a fleet of third-party appliances,
  giving them a single scalable entry/exit point.
- **CLB** is the previous generation; use ALB or NLB for anything new.

Across all types, ELB gives you three core benefits: **scalability** (it scales itself),
**security** (integrated TLS, security groups, WAF, auth), and **availability** (health
checks and multi-AZ distribution).

## Logical model

Both ALB and NLB share the same conceptual pieces:

- **Listener** — accepts incoming connections on a protocol/port (e.g. HTTPS:443).
- **Rules** (ALB) — evaluate request attributes and choose an action/target group.
- **Target group** — a set of registered targets (EC2 instances, IPs, Lambda, or even an
  ALB behind an NLB), each health-checked independently.

A target group can point at EC2 instances, raw IPs (including on-prem via Direct
Connect/VPN), containers (ECS/EKS), or Lambda.

## Availability: ELB is a distributed system, not a box

ELB isn't a single appliance — it's a managed, horizontally-scaled service spread across
Availability Zones. Two implications drive most best practices:

- **The load balancer scales by changing its own DNS.** As traffic grows, ELB adds
  capacity and publishes new IPs behind its DNS name. **Always connect via the DNS name,
  never a resolved IP**, or you'll pin to capacity that can disappear.
- **It's multi-AZ.** Enable all the AZs your targets live in so ELB can route around a
  failed zone.

### Layering AWS services in front of ELB

These improve global availability and performance ahead of the load balancer:

- **Amazon Route 53** — DNS-level routing across multiple ALBs/NLBs. Use records pointing
  at several load balancers, enable **Evaluate Target Health**, and set a **short TTL (~60s)**
  so failover is quick. (See the DNS routing policies article for the policy choices.)
- **Amazon CloudFront** — caches static assets at the edge and fronts the ALB via CNAME,
  cutting origin load and latency.
- **AWS Global Accelerator** — anycast static IPs that route users over the AWS backbone
  to the nearest healthy regional endpoint; good for latency and fast regional failover.

### Services behind ELB

- **Auto Scaling groups** replace unhealthy instances (using the ELB health check as the
  ASG health check) and scale on demand.
- **ECS / EKS** register/deregister container targets automatically.
- **Lambda / Fargate** provide serverless targets (ALB → Lambda; Fargate as IP targets).

A common pattern: Route 53 (or Global Accelerator) → ALB → Auto Scaling group / ECS-EKS,
duplicated across a primary and backup region, with CloudFront and S3 for static content.

## How targets fail — and why the LB type changes the client experience

Understanding the failure lifecycle is essential for zero-downtime operations.

### ALB target failure

1. A client is connected to the ALB; each request is routed to a healthy target.
2. A target starts failing its **ALB health check**.
3. **In-flight requests are allowed to complete**; the target is marked unhealthy.
4. **Future requests route only to healthy targets.**

Because ALB works per-request, failure is graceful — clients rarely notice, and
connection draining (deregistration delay) lets existing requests finish.

### NLB target failure

1. A client has an active **flow** (connection) pinned to a target.
2. The target fails its **NLB health check**.
3. **New flows go to healthy targets**, but the existing flow to the failed target is
   broken — the client may see a **TCP reset** and must reconnect.

Because NLB works per-connection, a target failure can break live connections. This is why
**client reconnect logic matters more with NLB**.

### Client connectivity best practices

These make clients resilient to the behavior above:

1. **Use and respect DNS** — connect by name, honor TTLs, don't cache IPs forever.
2. **Refresh DNS on reconnect** — re-resolve when a connection drops so you pick up new
   ELB capacity or a healthy AZ.
3. **Use exponential backoff with jitter** — on retry, so a fleet of clients doesn't
   stampede a recovering endpoint in lockstep.

## Health checks are the availability engine

Everything above depends on health checks:

- **ELB health checks** decide which targets receive traffic and (via ASG) which instances
  get replaced.
- **Route 53 health checks** decide which ELB IPs/records are returned at the DNS layer,
  removing an unhealthy AZ or load balancer from rotation.

Configure health-check paths that actually exercise the app (a `/health` endpoint that
checks dependencies), tune thresholds to fail fast but avoid flapping, and make sure a
health check failing means "don't send traffic here" — not a false alarm that pulls
healthy capacity.

## Serverless and advanced ALB patterns

An ALB can front an entire app without any servers you manage:

- **Lambda targets** for API/logic (`/objects` → Lambda).
- **Fargate** containers as IP targets (`/application` → Fargate).
- **Fixed responses** for simple static replies and health endpoints.
- **HTTP→HTTPS redirect** actions at the listener.
- **Amazon Cognito / OIDC authentication** at the ALB (`/login`), offloading auth before
  the request reaches your app.
- **AWS WAF** attached for L7 protection.

This lets one ALB route by path to a mix of Lambda, Fargate, Cognito auth, redirects, and
fixed responses — a fully serverless front end.

### NLB advanced patterns

- **ALB-as-a-target of NLB** — put an NLB (static IP / PrivateLink) in front of an ALB to
  get both a fixed IP entry point and L7 routing.
- **PrivateLink** — expose a service privately to other VPCs/accounts through an NLB
  endpoint service.
- **Zonal DNS** — NLB publishes a per-AZ IP; you can target a single zone's stack when you
  need zonal isolation, or the regional name for cross-AZ distribution.

## Operations and planning for production

Availability isn't only architecture — it's how you run the thing.

### Operational visibility

- **Metrics and dashboards.** Track `RequestCount`, `ProcessedBytes`,
  `ActiveConnectionCount`, `NewConnectionCount`, target/ELB `HTTPCode_*` counts, and
  latency percentiles (**P50 and P99**, not just averages). Build a dashboard you can read
  during an incident.
- **Alarms.** Alarm on the metrics that signal user impact — 5xx rates
  (`HTTPCode_ELB_5XX_Count`, `HTTPCode_Target_5XX_Count`), latency, and unhealthy host
  count. Use **multiple alarms at different thresholds** (e.g. a 10% error "warning" and a
  25% error "critical") and require **multiple periods** to cut noise.
- **Operational hygiene.** Review these regularly, not just after an outage.

### Operational priorities during an incident

- **Mitigating impact is always priority #1** — shift traffic, fail over, scale out, roll
  back. Restore the user experience first.
- **Finding root cause is secondary.** Diagnose after you've stopped the bleeding; don't
  hold up mitigation to chase the "why."

## Best-practices checklist

- **Connect via the ELB DNS name**, never a cached IP.
- **Enable all target AZs** and cross-zone load balancing where appropriate.
- **Use ALB for HTTP/gRPC**, NLB for TCP/UDP/TLS + static IP/PrivateLink, GWLB for
  appliance insertion.
- **Set Route 53 Evaluate Target Health + short TTL (~60s)** for DNS-level failover.
- **Build clients to re-resolve DNS on reconnect and back off with jitter** (critical for
  NLB).
- **Tune health checks** to fail fast without flapping; point them at a real dependency
  check.
- **Use deregistration delay / connection draining** for graceful deploys.
- **Offload TLS, auth (Cognito/OIDC), and WAF** to the ALB where it simplifies the app.
- **Dashboard P50/P99 latency and error codes; alarm at multiple thresholds.**
- **Rehearse failover;** during incidents, mitigate first, root-cause later.

---

### Sources

- [Elastic Load Balancing features](https://aws.amazon.com/elasticloadbalancing/features/)
- [Application Load Balancer User Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
- [Network Load Balancer User Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html)
- [Gateway Load Balancer User Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html)
- [Routing traffic to an ELB load balancer (Route 53)](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html)
- [CloudWatch metrics for your Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html)
- [Best practices for exponential backoff and jitter (AWS Architecture Blog)](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
