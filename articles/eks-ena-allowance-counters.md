# EKS ENI Allowance Counters (ENA Driver)

These are the ENA (Elastic Network Adapter) driver-level counters that track when EC2 enforces network limits at the hypervisor. They're exposed per-ENI via `ethtool -S <interface>`.

## The 5 Allowance Counters

| Counter | What it means |
|---------|---------------|
| `bw_in_allowance_exceeded` | Inbound bandwidth throttled — packets queued/dropped because you hit the instance's ingress bandwidth cap |
| `bw_out_allowance_exceeded` | Outbound bandwidth throttled — same for egress |
| `pps_allowance_exceeded` | Packets per second limit hit — applies bidirectionally, drops are silent |
| `conntrack_allowance_exceeded` | Connection tracking table full at the hypervisor level — new flows get dropped before reaching the kernel conntrack |
| `linklocal_allowance_exceeded` | Rate limit on link-local traffic (169.254.169.254) — affects IMDS metadata calls and VPC DNS resolver (169.254.169.253) |

## How to Read Them

```bash
# Per-ENI stats
ethtool -S eth0 | grep allowance

# Output example:
#   bw_in_allowance_exceeded: 0
#   bw_out_allowance_exceeded: 0
#   pps_allowance_exceeded: 142
#   conntrack_allowance_exceeded: 0
#   linklocal_allowance_exceeded: 0
```

These are **cumulative counters** (monotonically increasing since boot). You want to monitor the **rate of change** — any non-zero delta means you're being throttled.

## Getting Them Into Datadog

### Option 1: Datadog Agent NPM (Network Performance Monitoring)

If you enable NPM, newer agents (7.33+) expose some of these automatically.

### Option 2: Custom Check

```yaml
# /etc/datadog-agent/conf.d/ena_counters.d/conf.yaml
instances:
  - interface: eth0
```

```python
# /etc/datadog-agent/checks.d/ena_counters.py
import subprocess
from datadog_checks.base import AgentCheck

class ENACountersCheck(AgentCheck):
    def check(self, instance):
        iface = instance.get('interface', 'eth0')
        result = subprocess.run(
            ['ethtool', '-S', iface],
            capture_output=True, text=True
        )
        counters = [
            'bw_in_allowance_exceeded',
            'bw_out_allowance_exceeded',
            'pps_allowance_exceeded',
            'conntrack_allowance_exceeded',
            'linklocal_allowance_exceeded',
        ]
        for line in result.stdout.splitlines():
            for counter in counters:
                if counter in line:
                    value = int(line.split(':')[1].strip())
                    self.monotonic_count(
                        f'ena.{counter}',
                        value,
                        tags=[f'interface:{iface}']
                    )
```

### Option 3: CloudWatch (Instance-Level, Not Per-ENI)

The same data is available in CloudWatch as aggregate per instance:

- `NetworkBandwidthInAllowanceExceeded`
- `NetworkBandwidthOutAllowanceExceeded`
- `NetworkPacketsPerSecondAllowanceExceeded`
- `ConntrackAllowanceExceeded`
- `LinkLocalPacketRateExceeded`

Pull via the Datadog AWS integration — no agent work needed, but 5-min granularity and no per-ENI breakdown.

## Important Details

- **These drops are invisible to the kernel** — `iptables`, `tcpdump`, and `system.net.packets_in.drop` won't show them
- Counters are per-ENI, so on nodes with multiple ENIs (VPC CNI warm pool), check all interfaces
- The limits are per-instance-type and documented by AWS (e.g., m5.large baseline: 10 Gbps / ~750k PPS)
- Burst is allowed briefly via network credits (similar to CPU credits on t-series), but once exhausted you get hard-throttled

## Alert Strategy

```
Alert if rate(ena.pps_allowance_exceeded) > 0 over 2 min
Alert if rate(ena.bw_in_allowance_exceeded) > 0 over 2 min
Alert if rate(ena.conntrack_allowance_exceeded) > 0 over 1 min  # more urgent
Alert if rate(ena.linklocal_allowance_exceeded) > 0 over 5 min
```

The `conntrack_allowance_exceeded` one is the most critical — it causes connection failures that look like application bugs (timeouts, refused connections) and are extremely hard to diagnose without this counter.

## Deep Dive: `pps_allowance_exceeded`

### What It Is

Every EC2 instance type has a hard limit on how many **packets per second** (PPS) it can process, enforced at the hypervisor (Nitro card) level. When you exceed that limit, the Nitro card silently drops packets before they ever reach the instance's kernel or network stack.

### Why It's Dangerous

- **Completely silent** — no kernel log, no tcpdump trace, no iptables counter, no application error. The packet just vanishes.
- **Bidirectional** — the limit applies to the **sum of ingress + egress** packets. So if your limit is 750k PPS, sending 400k out + receiving 400k in = 800k total = throttled.
- **Per-instance, not per-ENI** — even if you have multiple ENIs, the PPS budget is shared across all of them.
- **Affects all protocols equally** — TCP, UDP, ICMP, ARP. A DNS-heavy node burning PPS on small UDP packets can starve TCP traffic.

### What Counts as a "Packet"

Every single frame counts as one packet regardless of size:

- A 64-byte TCP ACK = 1 packet
- A 1500-byte full MTU frame = 1 packet
- A 9001-byte jumbo frame = 1 packet

This means **small packet workloads are the worst case**. A service doing millions of tiny DNS lookups, Redis GET/SET, or health checks burns through PPS limits far faster than one streaming large payloads.

### Burst vs Baseline

Like bandwidth, PPS has a **baseline** and a **burst** allowance:

| Concept | Description |
|---------|-------------|
| **Baseline** | Sustained PPS you can maintain indefinitely |
| **Burst** | Higher PPS available temporarily using network credits |
| **Credits** | Accumulate when below baseline, consumed when bursting |
| **Credit exhaustion** | Hard-capped to baseline, counter starts incrementing |

### Common Scenarios That Hit PPS Limits

| Scenario | Why |
|----------|-----|
| High-frequency microservices (gRPC, REST) | Many small request/response packets |
| DNS-heavy pods (short TTLs, no caching) | Tons of tiny UDP packets to CoreDNS → VPC resolver |
| Health checks at scale | Kubelet, ALB, readiness probes — all small packets |
| Redis/Memcached clients | Tiny GET/SET packets at high rates |
| Logging/metrics sidecars | Many small UDP/TCP packets to collectors |
| Service mesh (Envoy/Istio) | Doubles the packet count — pod→envoy→network→envoy→pod |
| Conntrack-heavy flows with short-lived connections | SYN/FIN overhead per connection |

### How to Diagnose

```bash
# Check current counter value
ethtool -S eth0 | grep pps_allowance_exceeded

# Watch the rate of change (run twice, 10s apart)
BEFORE=$(ethtool -S eth0 | grep pps_allowance_exceeded | awk '{print $2}')
sleep 10
AFTER=$(ethtool -S eth0 | grep pps_allowance_exceeded | awk '{print $2}')
echo "Drops in 10s: $(($AFTER - $BEFORE))"

# Check current PPS rate
sar -n DEV 1 5
# or
cat /proc/net/dev
```

### How to Fix

| Solution | Effect |
|----------|--------|
| **Upgrade instance type** | Bigger instance = higher PPS limit |
| **Enable jumbo frames (MTU 9001)** | Same data in fewer packets — dramatic PPS reduction for bulk transfers |
| **Reduce DNS lookups** | Use NodeLocal DNSCache, increase TTLs |
| **Connection pooling** | Fewer new connections = fewer SYN/FIN packets |
| **Batch small writes** | Combine metrics/logs into fewer, larger packets |
| **Spread pods across more nodes** | Distribute PPS budget across fleet |
| **Disable unnecessary health checks** | Reduce probe frequency |

### Jumbo Frames Math Example

Transferring 1 GB/s:

- MTU 1500: ~700,000 PPS needed
- MTU 9001: ~120,000 PPS needed

Same throughput, 83% fewer packets. Only works within the same VPC/placement group.

### Monitoring in Datadog

```
# Rate of PPS drops (should be zero)
rate(ena.pps_allowance_exceeded) > 0

# Current PPS load (to forecast before hitting limits)
rate(system.net.packets_in.count) + rate(system.net.packets_out.count)
```

Alert when the combined PPS approaches 80% of your instance type's known limit, not just when drops start.

## Deep Dive: `bw_in_allowance_exceeded` / `bw_out_allowance_exceeded`

### What They Are

Every EC2 instance type has a hard bandwidth cap enforced at the Nitro card (hypervisor). When traffic exceeds the allowed throughput, excess packets are queued and eventually dropped.

- `bw_in_allowance_exceeded` — increments when **inbound** traffic is throttled
- `bw_out_allowance_exceeded` — increments when **outbound** traffic is throttled

These are **independent limits** — unlike PPS which is bidirectional, bandwidth is enforced separately for ingress and egress.

### Why They're Dangerous

- **Invisible to the OS** — the kernel sees no error, `iftop`/`nload` still shows traffic flowing, but some packets are being silently dropped or delayed at the Nitro level
- **Manifests as latency first** — before full drops, traffic gets queued at the hypervisor, adding jitter/latency that looks like application slowness
- **Affects all traffic equally** — pod-to-pod, pod-to-internet, EBS traffic (on some instance types), everything shares the same pipe
- **Correlates with retransmits** — TCP will recover via retransmission, but UDP (DNS, metrics, logs) just loses data

### Single-Flow vs Multi-Flow

AWS enforces **two different limits**:

| Limit | Description |
|-------|-------------|
| **Single flow** | Capped at ~5 Gbps (even on 100 Gbps instances). A "flow" = unique 5-tuple |
| **Multi-flow aggregate** | The instance type's advertised limit (e.g., 10 Gbps, 25 Gbps) |

So even on a `m5.8xlarge` (10 Gbps), a single TCP connection between two pods maxes out at ~5 Gbps.

### EBS vs Network Shared Bandwidth

On smaller instances (m5.large, m5.xlarge), **EBS traffic shares the network bandwidth cap**. Heavy disk I/O (pulling container images, writing logs) directly competes with pod network traffic.

Larger instances (m5.4xlarge+) have **dedicated EBS bandwidth** that doesn't count against network limits.

### Common Scenarios That Hit Bandwidth Limits

| Scenario | Why |
|----------|-----|
| Container image pulls (large images) | Burst of ingress during deployments/scaling |
| Log shipping (Fluentd/Fluent Bit → S3/ES) | Sustained egress from every node |
| Data-intensive pods (ETL, ML training) | Bulk transfers between pods or to S3 |
| Cross-AZ traffic | Same bandwidth cap, but you're paying for it twice |
| EBS-heavy workloads on small instances | Shared pipe between EBS and network |
| Backup/snapshot jobs | Sudden spike in egress |
| Multiple pods streaming simultaneously | Aggregate easily exceeds node limit |

### How to Diagnose

```bash
# Check current counter values
ethtool -S eth0 | grep bw_.*allowance

# Watch the rate of change
watch -n 5 "ethtool -S eth0 | grep bw_.*allowance"

# Check current bandwidth usage
sar -n DEV 1 5

# Quick throughput check in Gbps
RX_BYTES_1=$(cat /sys/class/net/eth0/statistics/rx_bytes)
sleep 1
RX_BYTES_2=$(cat /sys/class/net/eth0/statistics/rx_bytes)
echo "Ingress: $(( ($RX_BYTES_2 - $RX_BYTES_1) * 8 / 1000000000 )) Gbps"
```

### How to Fix

| Solution | Effect |
|----------|--------|
| **Upgrade instance type** | More bandwidth (m5.large=10Gbps burst → m5.4xlarge=10Gbps sustained) |
| **Use placement groups** | 10 Gbps full bisection bandwidth between instances in the same group |
| **Enable jumbo frames (MTU 9001)** | Less per-packet overhead = slightly more effective throughput |
| **Compress before sending** | Reduce bytes on the wire (gzip logs, protobuf vs JSON) |
| **Limit container image sizes** | Smaller images = less ingress burst during scaling events |
| **Use instance types with dedicated EBS bandwidth** | Free up network bandwidth for pod traffic |
| **Stagger deployments/scaling** | Avoid all nodes pulling images simultaneously |
| **Multi-flow parallelism** | Split large transfers into multiple connections (bypass 5 Gbps single-flow cap) |

### Key Gotcha: Burst Gives a False Sense of Security

On `m5.large`, you might see 5+ Gbps for minutes and think you're fine. But the baseline is only 0.75 Gbps. Once network credits burn out (which can happen quickly under sustained load), you crash down to baseline and suddenly everything gets throttled. The `bw_*_allowance_exceeded` counters won't increment during burst — only after credits are depleted.

### Monitoring in Datadog

```
# Alert on any throttling
rate(ena.bw_in_allowance_exceeded) > 0 over 2 min
rate(ena.bw_out_allowance_exceeded) > 0 over 2 min

# Proactive: alert when approaching limits (% of known baseline)
(rate(system.net.bytes_rcvd) * 8) / <instance_baseline_bps> > 0.8
(rate(system.net.bytes_sent) * 8) / <instance_baseline_bps> > 0.8
```

## Instance Bandwidth and PPS Reference

| Instance Type | Baseline Bandwidth | Burst Bandwidth | EBS Dedicated? | PPS (approx) |
|---------------|-------------------|-----------------|----------------|--------------|
| m5.large | 0.75 Gbps | Up to 10 Gbps | No (shared) | ~750,000 |
| m5.xlarge | 1.25 Gbps | Up to 10 Gbps | No (shared) | ~1,000,000 |
| m5.2xlarge | 2.5 Gbps | Up to 10 Gbps | No (shared) | ~1,500,000 |
| m5.4xlarge | 5 Gbps | 10 Gbps | Yes | ~2,000,000 |
| m5.8xlarge | 10 Gbps | 10 Gbps | Yes | ~4,000,000 |
| m5.12xlarge | 12 Gbps | 12 Gbps | Yes | ~6,000,000 |
| m5.16xlarge | 20 Gbps | 20 Gbps | Yes | ~8,000,000 |
| m5.24xlarge | 25 Gbps | 25 Gbps | Yes | ~12,000,000 |

Check the exact limits for your instance types in the [AWS EC2 Network Bandwidth docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-network-bandwidth.html).
