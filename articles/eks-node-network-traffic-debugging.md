# Seeing Network Traffic on EKS Nodes

How to view and debug network traffic for EKS cluster nodes — from AWS-level visibility down to per-process tracing.

## 1. VPC Flow Logs (AWS-level, no agent needed)

Enable VPC Flow Logs on the subnets or ENIs your nodes use:

```bash
# Enable flow logs on the VPC
aws ec2 create-flow-log \
  --resource-type VPC \
  --resource-ids vpc-0abc123 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs/eks-cluster

# Or per-ENI for a specific node
aws ec2 create-flow-log \
  --resource-type NetworkInterface \
  --resource-ids eni-0abc123 \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-flowlogs-bucket
```

Shows: source IP, dest IP, port, protocol, bytes, packets, accept/reject. No process-level info.

### Query Flow Logs with CloudWatch Insights

```
fields @timestamp, srcAddr, dstAddr, srcPort, dstPort, protocol, bytes, action
| filter dstAddr = "10.0.1.50"
| sort @timestamp desc
| limit 100
```

### Query Flow Logs with Athena (if sent to S3)

```sql
SELECT srcaddr, dstaddr, dstport, protocol, SUM(bytes) as total_bytes
FROM vpc_flow_logs
WHERE date = '2025/01/15'
  AND srcaddr LIKE '10.0.%'
GROUP BY srcaddr, dstaddr, dstport, protocol
ORDER BY total_bytes DESC
LIMIT 20;
```

## 2. Node-Level Tools (SSH into the node)

### Quick Ranking for EKS Troubleshooting

| Tool | What you see | Install |
|------|-------------|---------|
| `nethogs eth0` | Bandwidth per process, real-time | `yum install -y nethogs` |
| `iftop -i eth0 -nNP` | Bandwidth per connection pair | `yum install -y iftop` |
| `conntrack -L` | All NAT/connection tracking (kube-proxy flows) | `yum install -y conntrack-tools` |
| `tcpdump -i eth0 -nn` | Raw packet capture | Pre-installed |
| `ss -tnp` | Active TCP connections with process | Pre-installed |
| `atop` then press `n` | Network per process | `yum install -y atop` |
| `nload eth0` | Simple bandwidth graph per interface | `yum install -y nload` |
| `iptraf-ng` | Interactive stats, TCP/UDP breakdowns | `yum install -y iptraf-ng` |

### ss — Quick Socket Overview

```bash
# Show all listening sockets with the owning process
ss -tulnp

# Active connections with process
ss -tnp

# Count connections by state
ss -s
```

### nethogs — Bandwidth per Process

```bash
yum install -y nethogs
nethogs eth0
```

Best first step — immediately shows which process is eating bandwidth.

### iftop — Real-Time Bandwidth per Connection

```bash
yum install -y iftop
iftop -i eth0 -nNP
```

Shows source/destination pairs with live throughput. `-P` adds port numbers.

### conntrack — Connection Tracking Table

```bash
yum install -y conntrack-tools

conntrack -L              # list all tracked connections
conntrack -E              # watch events in real time
conntrack -L -p tcp --dport 443 | wc -l   # count HTTPS connections
conntrack -C              # count total entries
```

Critical on EKS since kube-proxy/iptables relies on conntrack. Shows NAT translations which helps map pod traffic to node traffic.

### tcpdump — Packet Capture

```bash
# All traffic on eth0
tcpdump -i eth0 -nn -c 100

# Filter by destination port
tcpdump -i eth0 -nn dst port 443

# Filter by host
tcpdump -i eth0 -nn host 10.0.1.50

# Capture to file for Wireshark analysis
tcpdump -i eth0 -nn -w /tmp/capture.pcap -c 10000

# DNS traffic
tcpdump -i eth0 -nn port 53
```

## 3. eBPF Tools (Production-Safe, Low Overhead)

### BCC Tools

```bash
yum install -y bcc-tools
export PATH=$PATH:/usr/share/bcc/tools

# Trace all new TCP connections with PID and process name
tcpconnect

# Top-like view of TCP traffic per process
tcptop

# Trace TCP retransmits
tcpretrans

# Trace DNS latency
/usr/share/bcc/tools/gethostlatency
```

### bpftrace One-Liners

```bash
yum install -y bpftrace

# Trace every new TCP connection with PID and comm
bpftrace -e 'kprobe:tcp_connect { printf("%d %s -> connecting\n", pid, comm); }'

# Count packets sent per process
bpftrace -e 'kprobe:tcp_sendmsg { @[comm] = count(); }'

# Histogram of packet sizes per process
bpftrace -e 'kprobe:tcp_sendmsg { @bytes[comm] = hist(arg2); }'
```

## 4. Per-Pod Traffic

```bash
# Find the pod's container ID
crictl ps | grep <pod-name>

# Get the PID
crictl inspect <container-id> | grep pid

# Enter its network namespace and trace
nsenter -t <pid> -n ss -tnp
nsenter -t <pid> -n tcpdump -i eth0 -nn -c 50
nsenter -t <pid> -n conntrack -L
```

## 5. Datadog NPM (if agent is deployed)

Enable Network Performance Monitoring in your Helm values:

```yaml
# datadog-values.yaml
datadog:
  networkMonitoring:
    enabled: true
agents:
  useHostNetwork: true    # critical — agent must see host interfaces
```

### Metrics

| Metric | What it shows |
|--------|--------------|
| `system.net.bytes_sent` | Outbound bytes per interface |
| `system.net.bytes_rcvd` | Inbound bytes per interface |
| `system.net.packets_in.count` | Packets received |
| `system.net.packets_out.count` | Packets sent |
| `system.net.tcp.established` | Active TCP connections |
| `system.net.conntrack.count` | Conntrack entries |

### Queries

```
# Bytes sent per node
avg:system.net.bytes_sent{kube_cluster_name:my-cluster} by {host}

# Use sum and short rollup to catch bursts
sum:system.net.bytes_sent{kube_cluster_name:my-cluster} by {host}.rollup(max, 10)

# Top 10 talkers
top(avg:system.net.bytes_sent{kube_cluster_name:my-cluster} by {host}, 10, 'mean', 'desc')

# Conntrack usage (if maxed out, packets drop)
avg:system.net.conntrack.count{kube_cluster_name:my-cluster} by {host}

# NPM flow-level: bytes per source/destination pod
sum:network.bytes_sent{kube_cluster_name:my-cluster} by {pod_name,destination}

# DNS request volume
sum:dns.requests{kube_cluster_name:my-cluster} by {pod_name}
```

### Troubleshooting: Datadog Shows Only a Few KB/s

| Reason | Fix |
|--------|-----|
| It's already a rate (bytes/sec) | Use `sum` or `max` instead of `avg` |
| `avg` flattens spikes | Use `.rollup(max, 10)` for short windows |
| Wrong interface | Check `by {device}` — ensure `eth0` is reported |
| hostNetwork not enabled | Set `agents.useHostNetwork: true` |
| Time window too wide | Narrow to last 15m or 1h |

Sanity check from the node:

```bash
TX1=$(cat /sys/class/net/eth0/statistics/tx_bytes)
sleep 10
TX2=$(cat /sys/class/net/eth0/statistics/tx_bytes)
echo "TX rate: $(( (TX2 - TX1) / 10 / 1024 )) KB/s"
```

If the node shows MB/s but Datadog shows KB/s → `hostNetwork: true` is missing or the agent is on the wrong interface.

## 6. Kubernetes-Native / CNI Options

| Option | What it gives you |
|--------|-------------------|
| **Cilium Hubble** | Full flow visibility with process context, L7 observability, network policy audit |
| **Calico flow logs** | Per-pod flow logging with policy context |
| **AWS VPC CNI flow logs** | ENI-level flow data, integrates with VPC Flow Logs |
| **kubectl top pod** | CPU/memory only (helps narrow suspects, no network) |

### Hubble (if using Cilium CNI)

```bash
hubble observe --namespace default
hubble observe --from-pod default/my-pod
hubble observe --to-ip 10.0.1.50
hubble observe --verdict DROPPED
```

## 7. Quick Troubleshooting Workflow

1. **Start broad** — VPC Flow Logs or Datadog NPM to identify which nodes/pods generate the most traffic
2. **Narrow down** — SSH to the node, run `nethogs` or `iftop` to see which processes
3. **Correlate** — `conntrack -L` to map pod IPs to NAT'd connections
4. **Deep dive** — `tcpdump` or `bpftrace` for specific connections or packet inspection
5. **For pods** — `nsenter` into the pod's network namespace for isolated capture

### Common Things to Look For

| Symptom | Likely cause | Tool to use |
|---------|-------------|-------------|
| High bandwidth on a node | A pod doing bulk transfers | `nethogs`, then `nsenter` into pod |
| Many connections to same IP | DNS hammering or connection leak | `conntrack -L`, `ss -tn` |
| Packet drops | Conntrack table full or SG rules | `conntrack -C`, `dmesg` |
| Slow responses | Retransmits, high latency | `tcpretrans` (bcc), Datadog NPM |
| Unknown external traffic | Pod calling external APIs | `iftop`, VPC Flow Logs |

## 8. Real-Time Monitoring Across All Nodes

### DaemonSet with netshoot (Quick and Dirty)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: network-debug
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: network-debug
  template:
    metadata:
      labels:
        app: network-debug
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: debug
        image: nicolaka/netshoot
        command: ["sleep", "infinity"]
        securityContext:
          privileged: true
      tolerations:
      - operator: Exists
```

```bash
# Real-time rate across all nodes — no install needed
for pod in $(kubectl get pods -n kube-system -l app=network-debug -o name); do
  NODE=$(kubectl get ${pod} -n kube-system -o jsonpath='{.spec.nodeName}')
  kubectl exec -n kube-system ${pod} -- bash -c "
    RX1=\$(cat /sys/class/net/eth0/statistics/rx_bytes)
    TX1=\$(cat /sys/class/net/eth0/statistics/tx_bytes)
    sleep 5
    RX2=\$(cat /sys/class/net/eth0/statistics/rx_bytes)
    TX2=\$(cat /sys/class/net/eth0/statistics/tx_bytes)
    echo '$NODE: RX=\$(( (RX2-RX1)/1024 )) KB/5s  TX=\$(( (TX2-TX1)/1024 )) KB/5s'
  " &
done
wait
```

Cleanup:

```bash
kubectl delete daemonset network-debug -n kube-system
```

### AWS SSM Run Command (No SSH, No DaemonSet)

```bash
aws ssm send-command \
  --targets "Key=tag:kubernetes.io/cluster/my-cluster,Values=owned" \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=[
    "echo === $(hostname) - $(curl -s http://169.254.169.254/latest/meta-data/instance-id) ===",
    "ss -s",
    "echo ---",
    "cat /proc/net/dev | grep eth0"
  ]' \
  --comment "Quick network snapshot"

COMMAND_ID=<from-output>
aws ssm list-command-invocations --command-id $COMMAND_ID --details \
  | jq '.CommandInvocations[] | {instance: .InstanceId, output: .CommandPlugins[0].Output}'
```

### Parallel SSH (pssh)

```bash
NODES=$(kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}')

for NODE in $NODES; do
  ssh -o StrictHostKeyChecking=no ec2-user@$NODE "echo === $NODE ===; ss -s" &
done
wait
```

### Comparison

| Approach | Best for | Requires | Cleanup needed |
|----------|----------|----------|----------------|
| **DaemonSet + netshoot** | Quick ad-hoc debugging | kubectl access | Yes — delete DaemonSet after |
| **SSM Run Command** | Production-safe, audited, no SSH | SSM agent (default on EKS AMIs) | No |
| **pssh** | If you already have SSH bastion | SSH keys + bastion | No |
| **Datadog NPM** | Continuous monitoring, historical | Datadog agent deployed | No |
