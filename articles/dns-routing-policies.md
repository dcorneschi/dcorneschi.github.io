# Cloud DNS Routing Policies Explained

A DNS name like `app.example.com` doesn't have to resolve to the same IP for everyone. A
**routing policy** is the rule the authoritative DNS service applies when it decides
*which* answer to hand back for a query — based on weight, health, the resolver's
location, measured latency, or a simple static mapping. This is how a single hostname can
send European users to Frankfurt, North American users to Virginia, fail over to a standby
site when the primary dies, or shift 10% of traffic to a new version for a canary test.

This article walks through the six common routing policies — **Simple, Weighted,
Failover, Latency-based, Geolocation, and Geoproximity** — what each does, when to use it,
and the gotchas. Examples use AWS Route 53 terminology, but the same concepts appear in
Google Cloud DNS, Azure Traffic Manager / Azure DNS, and NS1.

> **Key mental model:** DNS routing decides *which record to return*, not how packets are
> forwarded. The client still connects directly to whatever IP it receives, and the answer
> is cached for the record's **TTL**. That TTL is the single biggest source of surprises —
> a failover or weight change won't take effect for clients until their cached answer
> expires.

---

## 1. Simple

The default. One record, one answer (or a fixed set of answers returned together), the
same for every client.

```text
app.example.com  →  203.0.113.10   (100% of queries)
```

- **Use it when** there's a single endpoint and no need for health checks, geography, or
  traffic splitting.
- If you put **multiple values** in one simple record, the resolver receives all of them
  and picks one (often round-robin). This is *not* load balancing — there are no health
  checks, so a dead endpoint is still handed out.

**Use case:** a personal site, an internal tool, or anything fronted by a load balancer
that already handles distribution and health.

## 2. Weighted

Split traffic across multiple endpoints by proportion. Each record gets a weight, and the
share of traffic is `weight / sum-of-weights`.

```text
app.example.com  →  App A   weight 70   (~70%)
app.example.com  →  App B   weight 30   (~30%)
```

- **Canary / blue-green deployments:** send 5% to a new version, watch metrics, then ramp
  up by adjusting weights.
- **A/B testing** and gradual migration between two backends or regions.
- Setting a weight to **0** takes an endpoint out of rotation without deleting the record.

**Gotcha:** the split is statistical and applied at resolution time, then cached for the
TTL. With long TTLs and few resolvers, real traffic can skew well off the configured
percentages. Keep TTLs short for canaries.

## 3. Failover

Active-passive. A **primary** record is returned as long as its health check passes; if
the primary is marked unhealthy, DNS returns the **secondary** (standby) instead.

```text
app.example.com  →  Primary   203.0.113.10   (healthy → returned)
                 ↳  Failover  198.51.100.20  (returned only when primary is unhealthy)
```

- Requires a **health check** on the primary — without one there's nothing to fail over
  *from*.
- **Use it for** disaster recovery: a hot standby in another region, or a static
  "maintenance" page served from object storage when the app is down.

**Gotcha:** failover speed is bounded by health-check interval + failure threshold **plus
the record TTL**. Clients holding a cached primary answer keep using it until the TTL
expires, so a 300s TTL means up to 5 extra minutes of pain. Short TTLs on failover records
are standard.

## 4. Latency-based

Return the endpoint that gives the **lowest network latency** to the requesting resolver,
based on the DNS provider's latency measurements between regions — not raw geographic
distance.

```text
User in India      →  Mumbai region
User in US-East    →  Virginia region
(app served from the least-latency location)
```

- **Use it for** multi-region active-active deployments where you want the best
  performance for each user and you run the same stack in several regions.
- Latency ≠ distance. The nearest region geographically isn't always the fastest; routing
  follows measured network performance.

**Gotcha:** the decision is made against the **resolver's** location, not the end user's.
A user on a distant public DNS resolver (or a corporate resolver in another region) can be
routed to the "wrong" region. Combine with health checks so latency routing never points
at a dead region.

## 5. Geolocation

Route based on the **geographic origin** of the query — continent, country, or (in some
providers) subdivision/state. The decision is about *where the user is*, regardless of
latency.

```text
Queries from Asia    →  Mumbai
Queries from Europe  →  Frankfurt
Everything else      →  Virginia (Default)
```

- **Use it for** compliance and data-residency ("EU users must hit EU infrastructure"),
  content localization (language, currency), and licensing/geo-restriction (blocking or
  redirecting specific regions).
- **Always configure a `Default` record.** If a query's location matches no rule and there
  is no default, the resolver gets **no answer** and resolution fails.

**Gotcha:** same resolver caveat as latency — geolocation keys off the resolver's location
when the client's isn't available (EDNS Client Subnet helps but isn't universal). VPNs and
public resolvers can therefore land users in the wrong bucket.

## 6. Geoproximity

Route based on the **distance between the user and the resource**, with an adjustable
**bias** that lets you expand or shrink a resource's geographic reach. Without bias it's
"nearest resource wins"; bias lets you deliberately pull more (or less) traffic toward a
location.

```text
bias 0  on both:   users go to whichever region is closer (clean midpoint split)
bias 50 on Mumbai: Mumbai's "reach" expands, capturing users who'd otherwise hit Hyderabad
```

- **Use it for** shifting load between regions (e.g., steer more traffic to a larger
  data center), or nudging the boundary between two sites without hard geographic rules.
- Distinct from **geolocation**: geolocation uses fixed political boundaries (this
  country → this endpoint); geoproximity uses **distance + a tunable bias** and
  interpolates the boundary between resources.

**Gotcha:** in Route 53, geoproximity requires **traffic flow** (traffic policies), not a
plain record set. Bias math is not linear — small bias changes can move the boundary more
than you expect, so adjust and observe.

## Choosing a Policy

| You want to… | Policy |
|--------------|--------|
| Point a name at one endpoint | Simple |
| Split traffic by percentage (canary, A/B) | Weighted |
| Fail over to a standby when primary is down | Failover |
| Give every user the fastest region | Latency-based |
| Route by country/continent (compliance, localization) | Geolocation |
| Route by distance with a tunable pull between sites | Geoproximity |

These are also **composable** in most providers. Common real-world stacks:

- **Latency + Failover:** route to the fastest healthy region, fail out any region whose
  health check fails.
- **Geolocation + Weighted:** send EU users to EU infrastructure (compliance), then
  canary-split within the EU by weight.
- **Failover + Simple:** primary app with a static maintenance page as the standby.

## Cross-Provider Terminology

| Concept | AWS Route 53 | Google Cloud DNS | Azure |
|---------|--------------|------------------|-------|
| Percentage split | Weighted routing | Weighted round robin | Traffic Manager: Weighted |
| Health-based standby | Failover routing | (via health checks) | Traffic Manager: Priority |
| Fastest region | Latency-based routing | — | Traffic Manager: Performance |
| By location | Geolocation routing | Geolocation policy | Traffic Manager: Geographic |
| Distance + bias | Geoproximity (traffic flow) | — | — |

> Note: Azure Traffic Manager operates at the DNS layer much like Route 53 routing
> policies; Azure DNS itself is authoritative hosting. The mapping above is by *behavior*,
> not an exact feature-for-feature equivalence.

## AWS Route 53 Specifics

Route 53 is the reference implementation for most of this article, and it has a few
details worth calling out. It actually offers **eight** routing policies — the six above
plus **multivalue answer** and **IP-based** routing.

### The two extra Route 53 policies

- **Multivalue answer routing.** Returns up to **eight healthy records** chosen at random
  from a larger set, and — unlike a plain multi-value *simple* record — it **honors health
  checks**, so unhealthy endpoints are omitted from the answer. It gives you rudimentary
  health-aware distribution without a load balancer, but it is *not* a load balancer: the
  client still picks one of the returned IPs.
- **IP-based routing.** Route based on the **CIDR block** the query's source IP falls into,
  using a **CIDR collection** of named locations you define. Useful when you know your
  users' IP ranges (ISPs, corporate networks) and want to steer them deterministically —
  e.g., send a specific ISP's ranges to a nearby PoP.

### Health checks

Route 53 **health checks** are the engine behind failover and the health-aware behavior of
weighted, latency, geolocation, geoproximity, and multivalue routing. Key points:

- Types: **endpoint** checks (HTTP/HTTPS/TCP against an IP or domain), **calculated**
  checks (combine other checks with AND/OR/threshold logic), and **CloudWatch
  alarm** checks (mark healthy/unhealthy from a metric).
- A record is only eligible to be returned when its associated health check passes. Attach
  one to every non-primary target you don't want handed out while it's down.
- Failover timing = health-check interval × failure threshold **+ the record TTL**. Keep
  failover-record TTLs short (e.g., 60s).

### Alias records (an AWS-only shortcut)

For AWS targets — **ALB/NLB, CloudFront, S3 website endpoints, API Gateway, another Route
53 record** — use an **alias** record instead of a CNAME:

- Alias records can sit at the **zone apex** (`example.com`), where CNAMEs are illegal.
- They're **free** to query (Route 53 doesn't charge for alias queries to AWS resources).
- With `Evaluate Target Health`, Route 53 inherits the target's health automatically —
  handy in failover and latency setups.

### Simple records vs Traffic Flow

- **Record-set level:** Simple, weighted, failover, latency, geolocation, multivalue, and
  IP-based can all be created directly as record sets (console, CLI, or API).
- **Traffic Flow (traffic policies):** a visual policy builder for **layering** policies
  into a tree (e.g., geolocation → then weighted → then failover) and reusing it across
  records. **Geoproximity** in particular is a traffic-flow feature (though Route 53 has
  since added a Create-record wizard path for it too). Traffic policies carry a separate
  per-policy monthly charge.

### Geolocation / geoproximity location estimate

Route 53 estimates the user's location from the resolver IP, improved by **EDNS0 Client
Subnet** when the recursive resolver supports it. Public resolvers that don't forward
client subnet can misplace users — the same resolver caveat noted above, stated here
because it directly affects geolocation and geoproximity accuracy on Route 53.

### Minimal CLI example (weighted)

```bash
# Two weighted A records for the same name, 70/30 split.
# Each weighted record needs a unique SetIdentifier.
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456ABCDEFG \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "blue",
        "Weight": 70,
        "TTL": 60,
        "ResourceRecords": [{"Value": "203.0.113.10"}]
      }
    }, {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "green",
        "Weight": 30,
        "TTL": 60,
        "ResourceRecords": [{"Value": "198.51.100.20"}]
      }
    }]
  }'
```

- **`SetIdentifier`** is required and must be unique per record in a policy group; it's how
  Route 53 distinguishes records that share a name and type.
- Add **`"HealthCheckId": "<id>"`** to a record to make it health-aware.
- Set a weight to **`0`** to drain an endpoint without deleting the record.

## Practical Tips and Common Mistakes

- **TTL dominates responsiveness.** Every policy that reacts to change (failover, weight
  changes, health) is gated by the record TTL plus resolver caching. Use short TTLs
  (30–60s) on records you expect to change; use longer TTLs on stable records to cut query
  volume and cost.
- **Health checks make routing safe.** Latency, weighted, and geolocation policies will
  happily return a dead endpoint unless health checks are attached. Pair health checks with
  any policy that can point at more than one target.
- **Always define a default/fallback** for geolocation so unmatched queries still resolve.
- **The resolver, not the user, is what's usually located.** Latency, geolocation, and
  geoproximity decisions key off the DNS resolver's IP unless EDNS Client Subnet is in
  play. Public resolvers and VPNs can misplace users.
- **DNS is not a load balancer.** It distributes *resolution*, not connections, and it
  can't react within a single connection. For fine-grained, real-time balancing, use an
  actual load balancer (ALB/NLB, GCLB, etc.) behind the DNS name.
- **Weighted splits are approximate.** They converge over many queries; short-term traffic
  can be lumpy, especially with caching and few resolvers.

---

### Sources

- [Amazon Route 53 routing policies](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)
- [Google Cloud DNS routing policies](https://cloud.google.com/dns/docs/routing-policies-overview)
- [Azure Traffic Manager routing methods](https://learn.microsoft.com/en-us/azure/traffic-manager/traffic-manager-routing-methods)
- [EDNS Client Subnet (RFC 7871)](https://datatracker.ietf.org/doc/html/rfc7871)
