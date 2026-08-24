# EC2 Instance Network Burst Bandwidth — r7i.2xlarge

How "up to 12.5 Gbps" network actually works on burstable instances like `r7i.2xlarge`.

## r7i.2xlarge Network Specs

| Spec | Value |
|------|-------|
| **Advertised bandwidth** | "Up to 12.5 Gbps" |
| **Baseline bandwidth** | ~3.125 Gbps |
| **Burst bandwidth** | 12.5 Gbps |
| **Burst duration** | 5–60 minutes (depends on credit balance) |
| **vCPUs** | 8 |
| **ENA support** | Yes |
| **Internet gateway limit** | 5 Gbps (< 32 vCPUs) |

### What "Up to 12.5 Gbps" Actually Means

The instance does NOT get 12.5 Gbps sustained. It gets:

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  12.5 Gbps ─────────────── ████████  ← burst (uses credits)    │
│                            ████████                            │
│                            ████████                            │
│   3.125 Gbps ══════════════════════════════ ← baseline         │
│              (sustained indefinitely)                          │
│                                                                │
│  Time →                                                        │
└────────────────────────────────────────────────────────────────┘
```

- **Baseline (3.125 Gbps)**: What you can sustain forever without running out of credits
- **Burst (12.5 Gbps)**: Temporary — burns through network I/O credits
- Once credits are exhausted → drops back to 3.125 Gbps until credits recover

## How Network I/O Credits Work

Similar concept to T-instance CPU credits, but for network bandwidth:

1. **Credits accumulate** when you use less than baseline (3.125 Gbps)
2. **Credits are spent** when you exceed baseline
3. **Max credits** are given at launch — full bucket
4. **Separate buckets** for inbound and outbound traffic
5. **Best effort** — even with credits available, burst is not guaranteed (shared physical network)

### Credit Math (Simplified)

```
Credits earned:  (baseline - actual_usage) × time
Credits spent:   (actual_usage - baseline) × time

Example:
- Idle for 10 min at 0 Gbps → earns credits worth 10 min × 3.125 Gbps
- Burst at 12.5 Gbps → spends (12.5 - 3.125) = 9.375 Gbps worth of credits
- 10 min idle ≈ 3.3 min of burst at full 12.5 Gbps
```

### Burst Duration Estimates (r7i.2xlarge)

| Sustained burst rate | Approximate duration |
|---------------------|---------------------|
| 12.5 Gbps (max) | ~5-10 minutes from full credits |
| 10 Gbps | ~10-20 minutes |
| 6 Gbps | ~30-60 minutes |
| 3.125 Gbps (baseline) | Indefinite |

## Internet Gateway Limitation

Even during burst, traffic through an Internet Gateway is capped:

| Destination | Max bandwidth (r7i.2xlarge) |
|-------------|:---------------------------:|
| Same region (VPC-to-VPC, AZ-to-AZ) | Up to 12.5 Gbps (burst) |
| Through Internet Gateway | **5 Gbps hard cap** |
| Through NAT Gateway | 100 Gbps per NAT GW (not instance-limited) |
| Same placement group | Up to 10 Gbps single-flow |

Rule: instances with less than 32 vCPUs are limited to 5 Gbps through IGW regardless of instance bandwidth.

## Detecting Credit Exhaustion

### ENA Driver Metrics (Best Method)

```bash
# On the instance — check ENA counters
ethtool -S eth0 | grep -i allowance_exceeded
```

### Bandwidth vs PPS — Different Limits

The ENA metrics count **packets dropped**, but the *trigger* differs:

| ENA Metric | Trigger | Typical cause |
|-----------|---------|---------------|
| `bw_in_allowance_exceeded` | Inbound **bandwidth (Gbps)** exceeds limit | Large transfers, bulk downloads |
| `bw_out_allowance_exceeded` | Outbound **bandwidth (Gbps)** exceeds limit | Uploading to S3, cross-AZ replication |
| `pps_allowance_exceeded` | **Packets per second** exceeds PPS limit | Lots of tiny packets (DNS, health checks) |
| `conntrack_allowance_exceeded` | Connection tracking table full | Too many concurrent connections |
| `linklocal_allowance_exceeded` | PPS to local services exceeds limit | Excessive IMDS or DNS calls |

**Key distinction:**
- **`bw_*_allowance_exceeded`** = throughput problem (too many bytes/sec)
- **`pps_allowance_exceeded`** = packet rate problem (too many packets/sec regardless of size)

You can hit one without the other:
- Node doing bulk S3 downloads → hits `bw_in` limit (high bytes, normal PPS)
- Node with 500 pods doing rapid HTTP health checks → hits `pps` limit (low bytes, high PPS)

### r7i.2xlarge — Both Limits

| Limit type | Baseline (sustained) | Burst (temporary) |
|-----------|:--------------------:|:-----------------:|
| **Bandwidth** | 3.125 Gbps | 12.5 Gbps |
| **PPS** | ~300,000 pps | ~1,500,000 pps |
| **Connections tracked** | ~250,000 | — |

### Reading the Counters

These are **cumulative counters** since instance boot — an increasing value means you're hitting the limit:

```bash
# Check all allowance counters
ethtool -S eth0 | grep allowance

# Watch for increases over 10 seconds
BEFORE=$(ethtool -S eth0 | grep bw_out_allowance_exceeded | awk '{print $2}')
sleep 10
AFTER=$(ethtool -S eth0 | grep bw_out_allowance_exceeded | awk '{print $2}')
echo "Packets dropped in 10s: $((AFTER - BEFORE))"
```

### Interpreting Counter Values

The counters are **cumulative since instance boot**. Context matters:

| Counter value | Instance uptime | Interpretation |
|:------------:|:---------------:|----------------|
| < 1,000 | Days/weeks | Negligible — brief spikes during deploys |
| 3,000–5,000 | Days/weeks | Normal — occasional bursts, not a concern |
| 3,000–5,000 | Hours | Active throttling — investigate |
| > 10,000 | Hours | Sustained problem — size up the instance |
| Increasing in real-time | — | Currently hitting bandwidth limits |

```bash
# Check uptime for context
cat /proc/uptime | awk '{printf "%.1f hours (%.1f days)\n", $1/3600, $1/86400}'

# Quick math:
# 3713 drops over 7 days = ~22 drops/hour = not a problem
# 3713 drops over 1 hour = sustained throttling = consider sizing up
```

### Which Metric to Alert On

| Scenario | Alert on | Why |
|----------|----------|-----|
| Bulk data transfers (ETL, backups, S3) | `bw_in/out_allowance_exceeded` | Throughput-bound |
| Microservices with many small requests | `pps_allowance_exceeded` | Packet-rate-bound |
| Service mesh (Istio/Envoy sidecars) | Both | Doubles connections + adds overhead |
| High pod density (50+ pods/node) | `pps` + `conntrack` | Many connections, many health checks |
| Kafka/Elasticsearch data nodes | `bw_out_allowance_exceeded` | Replication traffic is high throughput |

## Check ENA Counters Across All EKS Nodes

### SSM Run Command (No SSH Needed)

```bash
# Target all nodes in the cluster by tag
aws ssm send-command \
  --targets "Key=tag:kubernetes.io/cluster/my-cluster,Values=owned" \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=[
    "echo === $(hostname) - $(curl -s http://169.254.169.254/latest/meta-data/instance-id) ===",
    "ethtool -S eth0 | grep allowance"
  ]' \
  --comment "Check ENA allowance counters"

# Get results
COMMAND_ID=<from-output>
aws ssm list-command-invocations --command-id $COMMAND_ID --details \
  | jq '.CommandInvocations[] | {instance: .InstanceId, status: .Status, output: .CommandPlugins[0].Output}'
```

### DaemonSet Approach (kubectl Only)

```bash
for pod in $(kubectl get pods -n kube-system -l app=network-debug -o name); do
  NODE=$(kubectl get ${pod} -n kube-system -o jsonpath='{.spec.nodeName}')
  echo "=== $NODE ==="
  kubectl exec -n kube-system ${pod} -- ethtool -S eth0 | grep allowance
done
```

### Check for Active Drops (Delta Over 10 Seconds)

```bash
for pod in $(kubectl get pods -n kube-system -l app=network-debug -o name); do
  NODE=$(kubectl get ${pod} -n kube-system -o jsonpath='{.spec.nodeName}')
  kubectl exec -n kube-system ${pod} -- bash -c "
    BEFORE=\$(ethtool -S eth0 | grep allowance_exceeded | awk '{sum+=\$2} END{print sum}')
    sleep 10
    AFTER=\$(ethtool -S eth0 | grep allowance_exceeded | awk '{sum+=\$2} END{print sum}')
    DIFF=\$((AFTER - BEFORE))
    if [ \$DIFF -gt 0 ]; then
      echo '$NODE: WARNING \$DIFF packets dropped in 10s'
      ethtool -S eth0 | grep allowance
    else
      echo '$NODE: OK no drops'
    fi
  " &
done
wait
```

### sshpt — Parallel SSH (pip install sshpt)

```bash
# Get all EKS node IPs
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}' \
  | tr ' ' '\n' > /tmp/eks-nodes.txt

# Collect ENA counters from all nodes
sshpt -f /tmp/eks-nodes.txt -u ec2-user -k ~/.ssh/my-key.pem \
  -c "echo === \$(hostname) ===; ethtool -S eth0 | grep allowance"

# Collect delta (active drops over 10 seconds)
sshpt -f /tmp/eks-nodes.txt -u ec2-user -k ~/.ssh/my-key.pem \
  -c "B=\$(ethtool -S eth0 | grep -E 'bw_(in|out)_allowance' | awk '{sum+=\$2}END{print sum}'); sleep 10; A=\$(ethtool -S eth0 | grep -E 'bw_(in|out)_allowance' | awk '{sum+=\$2}END{print sum}'); echo drops_in_10s=\$((A-B))"
```

Common sshpt options:

| Option | Description |
|--------|-------------|
| `-f <file>` | File with list of hosts (one per line), use `-` for stdin |
| `-u <user>` | SSH username (`ec2-user` for AL2, `ubuntu` for Ubuntu) |
| `-k <key>` | Path to SSH private key |
| `-s` | Use sudo |
| `-c "<command>"` | Command to run |
| `-o <file>` | Output to CSV file |
| `-t <threads>` | Number of concurrent connections (default: 10) |

### Datadog (Continuous, No SSH)

```
# Any node hitting bandwidth limits (rate of change = drops/sec)
diff(avg:aws.ec2.bw_in_allowance_exceeded{kube_cluster_name:my-cluster} by {host})
diff(avg:aws.ec2.bw_out_allowance_exceeded{kube_cluster_name:my-cluster} by {host})

# PPS limits
diff(avg:aws.ec2.pps_allowance_exceeded{kube_cluster_name:my-cluster} by {host})

# Alert: any node dropping packets due to bandwidth
avg(last_5m):diff(avg:aws.ec2.bw_out_allowance_exceeded{kube_cluster_name:my-cluster} by {host}) > 0
```

### CloudWatch Metrics

```
Namespace: AWS/EC2
MetricName: NetworkIn, NetworkOut (bytes per period)
```

But CloudWatch only shows 1-min or 5-min averages — it won't catch microbursts that last seconds.

### On the Node — Measure Actual Throughput

```bash
# Quick 10-second measurement
TX1=$(cat /sys/class/net/eth0/statistics/tx_bytes)
sleep 10
TX2=$(cat /sys/class/net/eth0/statistics/tx_bytes)
echo "TX: $(( (TX2 - TX1) * 8 / 10 / 1000000000 )) Gbps"

# Using iperf3 for actual bandwidth test
iperf3 -c <target-ip> -t 30 -P 4
```

## Network Debugging Tools: iptraf-ng and sar

### iptraf-ng in Batch Mode

`iptraf-ng` is interactive by default but has a background mode (`-B`) for scripting:

| Flag | Description |
|------|-------------|
| `-B` | Run in background (non-interactive), log to file |
| `-i eth0` | Monitor specific interface |
| `-t <min>` | Run for N minutes then stop (minimum: 1) |
| `-L <logfile>` | Write output to this log file |
| `-s eth0` | TCP/UDP traffic stats by port |
| `-g` | General interface statistics |
| `-d eth0` | Detailed stats (protocol breakdown) |

```bash
# General interface stats (1-minute capture) via SSM
aws ssm send-command \
  --targets "Key=tag:kubernetes.io/cluster/my-cluster,Values=owned" \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=[
    "yum install -y iptraf-ng -q 2>/dev/null",
    "iptraf-ng -g -B -t 1 -L /tmp/iptraf-general.log",
    "sleep 65",
    "echo === $(hostname) ===",
    "cat /tmp/iptraf-general.log",
    "rm -f /tmp/iptraf-general.log"
  ]' \
  --comment "iptraf general stats"
```

Faster alternatives (instant, no wait):

```bash
# Bytes per interface — instant
cat /proc/net/dev

# Connections by port — instant
ss -tn | awk '{print $4}' | cut -d: -f2 | sort | uniq -c | sort -rn | head -20

# Protocol breakdown — instant
cat /proc/net/snmp | grep -E "^(Tcp|Udp|Ip)"
```

### sar (System Activity Reporter)

`sar` from the `sysstat` package — plain text output, supports historical data.

```bash
# Install
yum install -y sysstat

# Network throughput per interface — 10 samples, 1 second apart
sar -n DEV 1 10

# Network errors (drops)
sar -n EDEV 1 10

# TCP retransmits and connection rates
sar -n TCP 1 10
```

Key `sar -n DEV` columns:

| Column | Meaning |
|--------|---------|
| `rxpck/s` | Received packets/sec |
| `txpck/s` | Transmitted packets/sec |
| `rxkB/s` | Received KB/sec |
| `txkB/s` | Transmitted KB/sec |

Key `sar -n TCP` columns:

| Column | Meaning |
|--------|---------|
| `active/s` | New outbound TCP connections/sec |
| `passive/s` | New inbound TCP connections/sec |
| `retrans/s` | TCP retransmits/sec (packet loss indicator) |

Across all nodes with sshpt:

```bash
sshpt -f /tmp/eks-nodes.txt -u ec2-user -k ~/.ssh/my-key.pem -s \
  -c "echo === \$(hostname) ===; sar -n DEV 1 5 | grep -E '(IFACE|eth0|Average)'"
```

### Historical sar Data

If sysstat service is running, it collects data every 10 minutes:

```bash
# Today's network stats
sar -n DEV -f /var/log/sa/sa$(date +%d)

# Specific time range
sar -n DEV -s 14:00:00 -e 15:00:00 -f /var/log/sa/sa$(date +%d)

# Peak per day across all sa files
for f in /var/log/sa/sa[0-9]*; do
  DAY=$(basename $f | sed 's/sa//')
  PEAK=$(sar -n DEV -f $f 2>/dev/null | grep eth0 | awk '{if($6>max)max=$6} END{printf "%.0f", max}')
  echo "Day $DAY: peak TX = ${PEAK} kB/s ($(echo "$PEAK * 8 / 1000000" | bc -l | xargs printf '%.2f') Gbps)"
done
```

## r7i Family — Full Network Specs

| Instance | vCPUs | Baseline | Burst | Notes |
|----------|:-----:|:--------:|:-----:|-------|
| r7i.large | 2 | ~0.781 Gbps | 12.5 Gbps | Very low baseline |
| r7i.xlarge | 4 | ~1.562 Gbps | 12.5 Gbps | |
| r7i.2xlarge | 8 | ~3.125 Gbps | 12.5 Gbps | |
| r7i.4xlarge | 16 | ~6.25 Gbps | 12.5 Gbps | |
| r7i.8xlarge | 32 | 12.5 Gbps | — | No burst, full sustained |
| r7i.12xlarge | 48 | 18.75 Gbps | — | No burst |
| r7i.16xlarge | 64 | 25.0 Gbps | — | No burst |
| r7i.24xlarge | 96 | 37.5 Gbps | — | No burst |
| r7i.48xlarge | 192 | 50.0 Gbps | — | No burst |

Pattern: instances <= 4xlarge (<= 16 vCPUs) have burstable network. 8xlarge and above get sustained full bandwidth.

## Implications for EKS

### Why This Matters for Kubernetes Workloads

1. **Pod-to-pod traffic across nodes** uses the node's network bandwidth
2. **Multiple pods compete** for the same node bandwidth budget
3. **A single pod doing bulk transfers** can exhaust credits for all pods on that node
4. **Service mesh sidecars** (Envoy/Istio) add overhead — effectively doubling network usage per request
5. **Log/metric shipping** (Datadog, Fluentd) is constant background load eating into baseline

### Sizing Guidance

| Workload pattern | Recommendation |
|-----------------|----------------|
| Steady ~2 Gbps per node | r7i.2xlarge is fine (within baseline) |
| Frequent bursts > 5 Gbps | Consider r7i.4xlarge or r7i.8xlarge |
| Sustained high throughput | Use 8xlarge+ (no burst model, full bandwidth) |
| Data-intensive pods (Kafka, Elasticsearch) | 8xlarge+ or dedicated node groups |
| Mixed workloads with occasional spikes | r7i.2xlarge works if spikes are short (< 5 min) |

### Monitoring Recommendations

```yaml
# Datadog monitor: alert when credits are being consumed
query: "avg(last_5m):diff(avg:aws.ec2.bw_out_allowance_exceeded{kube_cluster_name:my-cluster} by {host}) > 0"
name: "Network bandwidth exceeded on EKS node"
message: "Node {{host.name}} is hitting network bandwidth limits. Credits may be depleting."
```

## Check Your Instance Specs (CLI)

```bash
aws ec2 describe-instance-types \
  --instance-types r7i.2xlarge \
  --query "InstanceTypes[].[InstanceType, NetworkInfo.NetworkPerformance, NetworkInfo.NetworkCards[0].BaselineBandwidthInGbps, NetworkInfo.NetworkCards[0].PeakBandwidthInGbps]" \
  --output table
```

## Key Takeaways

- `r7i.2xlarge` baseline is **3.125 Gbps**, not 12.5 Gbps
- Burst to 12.5 Gbps is temporary (5-60 min depending on credits)
- Traffic through Internet Gateway is capped at **5 Gbps** regardless of burst
- Monitor `bw_in_allowance_exceeded` / `bw_out_allowance_exceeded` ENA metrics
- If you're consistently hitting burst, size up to r7i.4xlarge or r7i.8xlarge
- 8xlarge and above have no burst model — they get full sustained bandwidth

## Links

- [AWS EC2 Instance Network Bandwidth](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-network-bandwidth.html)
- [Monitor Network Performance (ENA)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-network-performance-ena.html)
- [EKS ENI Allowance Counters](articles/eks-ena-allowance-counters.md)
