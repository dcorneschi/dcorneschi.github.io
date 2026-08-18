# Linux CPU Steal Time

CPU steal (`%steal` or `st`) is the percentage of time a virtual CPU waits for a real CPU while the hypervisor is servicing another VM. High steal time means your VM isn't getting the CPU cycles it needs — the host is overcommitted or noisy neighbors are consuming resources.

## What %steal Means

| %steal | Interpretation |
|--------|----------------|
| 0-2% | Normal for virtualized environments |
| 2-5% | Slightly overcommitted host, usually acceptable |
| 5-10% | Performance impact likely, investigate |
| 10%+ | Severe contention, action required |

Steal time applies to:
- Cloud VMs (EC2, Azure VMs, GCE) — especially burstable instances (T-series)
- KVM/QEMU guests
- Xen guests
- VMware guests (shown differently — `%ready` in esxtop)

## Checking Steal Time

### Quick Check

```bash
# Current steal time (st column)
top -bn1 | head -3

# One-liner: just the steal value
top -bn1 | grep '%Cpu' | awk '{print "Steal: " $16 "%"}'

# Or with mpstat
mpstat 1 1 | awk '/Average/ {print "Steal: " $9 "%"}'
```

### Real-Time Monitoring

```bash
# mpstat every second
mpstat 1

# Per-CPU steal time
mpstat -P ALL 1

# vmstat (st column)
vmstat 1

# sar real-time
sar -u 1 5

# top — look at the 'st' value in the CPU line
top
```

### Historical Analysis

```bash
# Today's steal time from sar
sar -u | awk 'NR>2 && !/Average/ {print $1, "steal:", $(NF-1)"%"}'

# Peak steal time today
sar -u | awk 'NR>2 && !/Average/ {print $1, "steal:", $(NF-1)"%"}' | sort -t: -k2 -rn | head -10

# Steal time from specific date
sar -u -f /var/log/sa/sa05 | awk 'NR>2 && !/Average/ {print $1, "steal:", $(NF-1)"%"}'

# Average steal for the day
sar -u | awk '/Average/ {print "Average steal:", $(NF-1)"%"}'
```

## One-Liners

### Detection and Alerting

```bash
# Alert if steal > 5%
steal=$(mpstat 1 1 | awk '/Average/ {print $9}'); echo "$steal" | awk '{if ($1 > 5.0) print "HIGH STEAL: "$1"%"; else print "OK: "$1"%"}'

# Continuous monitoring with threshold
while true; do mpstat 1 1 | awk '/Average/ {if ($9 > 5.0) printf "\033[31mSTEAL: %s%%\033[0m\n", $9; else printf "steal: %s%%\n", $9}'; done

# Log steal time every minute
while true; do echo "$(date '+%Y-%m-%d %H:%M:%S') steal: $(mpstat 1 1 | awk '/Average/ {print $9}')%"; sleep 60; done >> /var/log/steal.log

# One-shot: steal in last 10 seconds
mpstat 10 1 | awk '/Average/ {print $9}'
```

### Correlation with Other Metrics

```bash
# Steal + iowait + system together
mpstat 1 5 | awk '/Average/ && /all/ {print "user:"$3"% system:"$5"% iowait:"$6"% steal:"$9"% idle:"$12"%"}'

# Per-CPU steal (find which vCPU is starved)
mpstat -P ALL 1 1 | awk '$3 != "all" && $9 > 0 {print "CPU"$3": steal="$9"%"}'

# Steal vs load average
echo "steal: $(mpstat 1 1 | awk '/Average/ {print $9}')% | load: $(cat /proc/loadavg | awk '{print $1, $2, $3}')"

# Top processes during high steal
ps -eo pid,pcpu,pmem,comm --sort=-pcpu | head -10
```

### Scan All sar Files for Steal > 5%

```bash
for f in /var/log/sa/sa[0-9]*; do
  result=$(sar -u -f "$f" 2>/dev/null | awk '!/Average/ && !/Linux/ && /^[0-9]/ && $(NF-1) > 5.0 {print}')
  [ -n "$result" ] && echo "=== $f ===" && echo "$result"
done
```

## RHEL-Specific

### Check Steal with sar (RHEL 7/8/9)

```bash
# Install sysstat if not present
dnf install sysstat -y
systemctl enable --now sysstat

# View steal column
sar -u

# %steal is second-to-last column in sar -u output
sar -u | awk 'NR>2 && !/Average/ && $(NF-1) > 0 {print}'
```

### Identifying the Hypervisor

```bash
# What virtualization platform
systemd-detect-virt

# Detailed VM info
dmidecode -t system | grep -i 'manufacturer\|product'

# Check for KVM/Xen/VMware
cat /sys/hypervisor/type 2>/dev/null || echo "Not Xen"
lscpu | grep -i hypervisor
virt-what
```

### RHEL on AWS — Burstable Instances

```bash
# Check instance type (is it T-series/burstable?)
curl -s http://169.254.169.254/latest/meta-data/instance-type

# Check CPU credit balance (from inside the VM)
# No direct way — use CloudWatch or:
cat /proc/cpuinfo | grep "model name"
```

### Tuning (KVM Host)

```bash
# Pin vCPUs to physical CPUs (on KVM host)
virsh vcpupin <vm-name> 0 2
virsh vcpupin <vm-name> 1 3

# Set CPU shares (cgroups)
virsh schedinfo <vm-name> --set cpu_shares=2048

# Check current pinning
virsh vcpuinfo <vm-name>
```

## Ubuntu-Specific

### Check Steal (Ubuntu 22.04/24.04)

```bash
# Install sysstat
apt install sysstat -y
sed -i 's/ENABLED="false"/ENABLED="true"/' /etc/default/sysstat
systemctl restart sysstat

# Real-time steal
mpstat 1

# Historical
sar -u
```

### Ubuntu on Cloud

```bash
# Detect virtualization
systemd-detect-virt

# Check instance metadata (AWS)
curl -s http://169.254.169.254/latest/meta-data/instance-type

# Check instance metadata (Azure)
curl -s -H "Metadata:true" "http://169.254.169.254/metadata/instance/compute?api-version=2021-02-01" | jq '.vmSize'

# Check instance metadata (GCP)
curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/machine-type
```

## Why Steal Happens

| Cause | Description |
|-------|-------------|
| Overcommitted host | More vCPUs allocated than physical cores |
| Noisy neighbors | Another VM on the same host consuming excessive CPU |
| Burstable instances | T-series (AWS), B-series (Azure) — CPU throttled when credits run out |
| CPU ready time | VMware — VM waiting in queue for physical CPU |
| Host maintenance | Hypervisor background tasks (live migration, snapshots) |
| Incorrect NUMA placement | vCPUs scheduled across NUMA nodes |

## Remediation

### Cloud Providers

| Action | When |
|--------|------|
| Upgrade instance type | Burstable → dedicated (M/C series) |
| Use dedicated hosts | Eliminate noisy neighbors |
| Enable unlimited mode | AWS T-series — prevents throttling (costs more) |
| Move to different AZ | May land on less loaded host |
| Stop/start the VM | Re-places on a (potentially) different host |

### On-Premises KVM/VMware

| Action | When |
|--------|------|
| Reduce host overcommit ratio | Target < 3:1 vCPU:pCPU |
| CPU pin critical VMs | Guarantee physical core access |
| Live migrate VMs | Balance load across hosts |
| Increase CPU shares | Prioritize important VMs |
| Add physical hosts | If all hosts are overcommitted |
| Check NUMA topology | Ensure VMs fit within a single NUMA node |

### Application Level

```bash
# Identify CPU-heavy processes during high steal
pidstat 1 5 | sort -k8 -rn | head -10

# Check if processes are blocked waiting for CPU
ps -eo stat,pid,comm | grep '^R'

# Reduce CPU pressure: nice/renice
renice +10 -p <pid>

# Use cgroups to limit specific processes
systemd-run --scope -p CPUQuota=50% /path/to/program
```

## Steal vs Other CPU Metrics

| Metric | Meaning |
|--------|---------|
| `%user` | CPU spent running user applications |
| `%system` | CPU spent in kernel |
| `%iowait` | CPU idle waiting for I/O |
| `%steal` | CPU stolen by hypervisor for other VMs |
| `%idle` | CPU truly idle |

Key insight: If `%idle` is 0 and `%steal` is high, your VM needs CPU but can't get it. If `%idle` is high and `%steal` is high, something unusual is happening (investigate hypervisor).

## Monitoring and Alerting

### Datadog/Prometheus Alert Thresholds

| Level | Threshold | Action |
|-------|-----------|--------|
| Warning | %steal > 5% for 5 min | Investigate |
| Critical | %steal > 10% for 5 min | Page on-call, consider migration |
| Emergency | %steal > 20% for 2 min | Immediate action required |

### Simple Cron Alert

```bash
#!/bin/bash
# /usr/local/bin/steal-alert.sh
THRESHOLD=5
STEAL=$(mpstat 10 1 | awk '/Average/ {print $9}')

if (( $(echo "$STEAL > $THRESHOLD" | bc -l) )); then
    echo "HIGH CPU STEAL: ${STEAL}% on $(hostname) at $(date)" | \
      mail -s "CPU Steal Alert: $(hostname)" admin@example.com
fi
```

```bash
# Run every 5 minutes
echo "*/5 * * * * root /usr/local/bin/steal-alert.sh" > /etc/cron.d/steal-alert
```

## Tips and Tricks

- **Burstable instances** (AWS T2/T3, Azure B-series) are the #1 cause of steal in cloud — check CPU credits first
- **Stop/start** (not reboot) re-places a VM on a potentially different physical host
- **Steal + high load average + low idle** = classic CPU starvation pattern
- **sar** is the best historical tool — if steal happens at 3 AM, you'll have the data
- **Per-CPU steal** (`mpstat -P ALL`) reveals whether one vCPU is worse than others (NUMA issue)
- **VMware doesn't show steal** — use esxtop `%RDY` (ready time) instead
- **Xen** shows steal accurately; **KVM** shows steal accurately
- **Container hosts**: steal seen inside containers reflects the underlying VM's steal
- **Don't confuse with iowait** — iowait means CPU is idle waiting for disk; steal means CPU wants to work but can't
