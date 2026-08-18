# sysstat / sar Cheatsheet

System Activity Reporter — collect, report, and save system activity information. Part of the sysstat package, `sar` provides historical and real-time performance data for CPU, memory, disk, network, and more.

## Installation

```bash
# Debian/Ubuntu
sudo apt install sysstat
sudo systemctl enable sysstat
sudo systemctl start sysstat

# RHEL/CentOS/Fedora
sudo dnf install sysstat
sudo systemctl enable sysstat
sudo systemctl start sysstat
```

On Debian/Ubuntu, also enable collection in the config file:

```bash
# /etc/default/sysstat
ENABLED="true"
```

## Basic Syntax

```bash
sar [options] [interval] [count]
sar [options] -f /var/log/sa/saDD    # Read from historical file
```

## CPU Usage

```bash
sar                     # Default: CPU from today's history
sar 1 5                 # Real-time, 1 sec interval, 5 samples
sar -u 2 10             # CPU usage, 2 sec interval, 10 samples
sar -u ALL 1            # All CPU statistics
sar -P ALL 1            # Per-CPU statistics
sar -P 0,1 1            # Specific CPUs (0 and 1)
```

### CPU Output Fields

| Field | Description |
|-------|-------------|
| `%user` | User level (applications) |
| `%nice` | Nice priority user level |
| `%system` | System level (kernel) |
| `%iowait` | Waiting for I/O completion |
| `%steal` | Stolen by hypervisor (VMs) |
| `%idle` | Idle time |

## Memory Usage

```bash
sar -r 1 5              # Memory utilization
sar -r ALL 1            # All memory statistics
sar -R 1 5              # Memory statistics (frmpg/s bufpg/s)
sar -S 1 5              # Swap space utilization
sar -B 1 5              # Paging statistics
sar -W 1 5              # Swapping statistics
sar -H 1 5              # Huge pages utilization
```

### Memory Output Fields

| Field | Description |
|-------|-------------|
| `kbmemfree` | Free memory (KB) |
| `kbmemused` | Used memory (KB) |
| `%memused` | Percentage used |
| `kbbuffers` | Buffers (KB) |
| `kbcached` | Cached (KB) |
| `kbcommit` | Committed memory (KB) |
| `%commit` | Percentage of memory committed |

## Disk I/O

```bash
sar -b 1 5              # I/O and transfer rate statistics
sar -d 1 5              # Block device statistics
sar -d -p 1 5           # Block devices with pretty names
```

### Disk Output Fields

| Field | Description |
|-------|-------------|
| `tps` | Transactions per second |
| `rkB/s` | KB read per second |
| `wkB/s` | KB written per second |
| `areq-sz` | Average request size (KB) |
| `aqu-sz` | Average queue size |
| `await` | Average wait time (ms) |
| `%util` | Bandwidth utilization |

## Network

```bash
sar -n DEV 1 5          # Network device statistics
sar -n EDEV 1 5         # Network errors
sar -n NFS 1 5          # NFS client statistics
sar -n NFSD 1 5         # NFS server statistics
sar -n SOCK 1 5         # Socket statistics
sar -n IP 1 5           # IP traffic statistics
sar -n TCP 1 5          # TCP statistics
sar -n ETCP 1 5         # TCP errors
sar -n UDP 1 5          # UDP statistics
sar -n ALL 1 5          # All network statistics
```

### Network Output Fields

| Field | Description |
|-------|-------------|
| `IFACE` | Network interface |
| `rxpck/s` | Packets received per second |
| `txpck/s` | Packets transmitted per second |
| `rxkB/s` | KB received per second |
| `txkB/s` | KB transmitted per second |
| `rxcmp/s` | Compressed packets received/s |
| `txcmp/s` | Compressed packets transmitted/s |
| `rxmcst/s` | Multicast packets received/s |

## System Load and Processes

```bash
sar -q 1 5              # Queue length and load averages
sar -w 1 5              # Task creation and context switching
sar -v 1 5              # Kernel tables (inode, file, etc.)
```

## Historical Data Analysis

### View Stored Data

```bash
# Today's data
sar -u -f /var/log/sa/sa$(date +%d)

# Specific day (e.g., day 12)
sar -u -f /var/log/sa/sa12

# List available data files
ls /var/log/sa/sa*
```

### Time Range Filtering

```bash
# From specific start time
sar -s 10:00:00

# Time range
sar -s 10:00:00 -e 12:00:00

# Specific day + time range
sar -u -f /var/log/sa/sa12 -s 10:00:00 -e 11:00:00
```

## Output Formats (sadf)

```bash
# Machine-readable CSV
sadf -d /var/log/sa/sa12 -- -u

# JSON format
sadf -j /var/log/sa/sa12 -- -u

# XML format
sadf -x /var/log/sa/sa12 -- -u

# SVG graph
sadf -g /var/log/sa/sa12 -- -u > cpu_usage.svg

# Database-friendly (semicolons to commas)
sadf -d /var/log/sa/sa12 -- -u | sed 's/;/,/g'
```

## Paging Statistics

```bash
sar -B 1 5              # Paging statistics
```

### Page Fault Types

| Type | Description |
|------|-------------|
| Major fault | Causes disk I/O — page not in memory, must be read from disk. Repeated major faults indicate memory pressure. |
| Minor fault | No disk I/O — page is in memory but not yet mapped to the process address space. |

`pgpgin/s` and `pgpgout/s` show pages paged in/out from disk. High `pgpgin/s` combined with high major faults suggests the system is actively swapping.

## Custom Interval Collection

Capture statistics at custom intervals for ad-hoc analysis (writes to binary file):

```bash
# Save stats every 5 seconds for 10 minutes (120 samples)
nohup sar -o sar-$(hostname)-$(date +%m%d%Y).bin 5 120 >/dev/null 2>&1 &

# Save all stats every 5 seconds for 24 hours (17280 samples)
nohup sar -o sar-$(hostname)-$(date +%m%d%Y).bin 5 17280 >/dev/null 2>&1 &

# Read back the captured file
sar -u -f sar-$(hostname)-$(date +%m%d%Y).bin
```

## Filtering Specific Devices

### Specific network interface

```bash
sar -n DEV 1 --iface=eth0
```

### Specific disk device

```bash
sar -dp | egrep 'DEV|emcpowera'
sar -dp | egrep 'DEV|sda'
```

## Threshold Scanning Across All Files

Search all stored `sa` files for intervals exceeding thresholds:

### CPU iowait > 10%

```bash
for i in /var/log/sa/sa{01..30}; do
  [ -f "$i" ] && echo "=== $i ===" && sar -f "$i" | awk '$7 > 10 {print}'
done
```

### Disk await > 300ms

```bash
for i in /var/log/sa/sa{01..30}; do
  [ -f "$i" ] && echo "=== $i ===" && sar -dp -f "$i" | awk '$9 > 300 {print}'
done
```

### Disk tps > 1000

```bash
for i in /var/log/sa/sa{01..30}; do
  [ -f "$i" ] && echo "=== $i ===" && sar -dp -f "$i" | awk '$4 > 1000 {print}'
done
```

### Load average (ldavg-1) > 10

```bash
for i in /var/log/sa/sa{01..30}; do
  [ -f "$i" ] && echo "=== $i ===" && sar -q -f "$i" | awk '$5 > 10 {print}'
done
```

## Archiving SAR Data

```bash
# Collect all sar files for shipping to another team
tar -czvf /tmp/sar-$(hostname).tgz /var/log/sa/*

# Generate a full text report for a specific period
sar -A -p -f /var/log/sa/sa10 -s 19:00:00 -e 23:00:00 > $(hostname)-$(date +%m%d%Y).out
```

## Practical Examples

### Complete system overview

```bash
# CPU, memory, disk, network — 1 sec interval, 10 samples
sar -u -r -d -n DEV 1 10
```

### Performance investigation

```bash
# Yesterday's peak hours (2 PM to 4 PM)
sar -u -s 14:00:00 -e 16:00:00 -f /var/log/sa/sa$(date -d yesterday +%d)

# Memory usage during business hours
sar -r -s 09:00:00 -e 17:00:00
```

### Network troubleshooting

```bash
# Network errors
sar -n EDEV 1 10

# Interface stats with errors combined
sar -n DEV -n EDEV 1 5
```

### Disk bottleneck detection

```bash
# Disk I/O with device names
sar -d -p 2 5

# Extract iowait over time
sar -u | grep -v Average | awk '{print $1, $6}'
```

### Post-incident analysis

```bash
# All metrics during incident window (yesterday 10-11 AM)
sar -A -f /var/log/sa/sa$(date -d yesterday +%d) -s 10:00:00 -e 11:00:00
```

### Capacity planning

```bash
# Average CPU usage over last 7 days
for i in $(seq -w 1 7); do
    echo "Day $i:"
    sar -u -f /var/log/sa/sa$i | grep Average
done
```

### Generate reports

```bash
# Full text report
sar -A -f /var/log/sa/sa$(date +%d) > /tmp/daily_report.txt

# SVG graph report
sadf -g /var/log/sa/sa$(date +%d) -- -A > system_report.svg
```

## Alerting Example

```bash
#!/bin/bash
# Alert on high CPU usage
CPU_THRESHOLD=80
CPU_USAGE=$(sar 1 1 | grep Average | awk '{print 100-$NF}')
if (( $(echo "$CPU_USAGE > $CPU_THRESHOLD" | bc -l) )); then
    echo "High CPU usage: ${CPU_USAGE}%"
fi
```

## Generating Graphs

`sadf` produces SVG graphs directly from binary data files — useful for quick visual analysis or embedding in reports.

### SVG Output

```bash
# CPU usage graph for today
sadf -g -- -u > cpu_graph.svg

# CPU usage graph for a specific date
sadf -g /var/log/sa/sa05 -- -u > cpu_graph.svg

# Multiple metrics in one graph
sadf -g -- -u -r > cpu_and_memory.svg

# Network traffic graph
sadf -g -- -n DEV > network_graph.svg

# Disk I/O graph
sadf -g -- -d > disk_io_graph.svg

# All metrics, compact layout (one type per row)
sadf -g -O packed -- -A > all_metrics.svg

# 24-hour view starting from midnight
sadf -g -O oneday -T -- -u > cpu_24h.svg

# Specific time window
sadf -g -s 14:00:00 -e 18:00:00 -- -n DEV > network_afternoon.svg
```

### Graph Options

| Option | Description |
|--------|-------------|
| `-g` | Generate SVG output |
| `-O packed` | One metric type per row (compact layout) |
| `-O oneday` | Show full 24-hour period starting at midnight |
| `-T` | Use local time instead of UTC on X-axis |
| `-s HH:MM:SS` | Start time |
| `-e HH:MM:SS` | End time |

### Converting SVG to PNG

`sadf` only outputs SVG. Convert with `rsvg-convert` (recommended) or ImageMagick:

```bash
# Install conversion tools
sudo apt install librsvg2-bin imagemagick -y

# Using rsvg-convert (better quality)
sadf -g -- -u | rsvg-convert -o cpu_graph.png
sadf -g -- -u | rsvg-convert -w 1200 -o cpu_graph.png

# Using ImageMagick
sadf -g -- -u > /tmp/cpu.svg && convert /tmp/cpu.svg cpu_graph.png

# One-liner: full daily PNG report for yesterday
YESTERDAY=$(date -d "yesterday" +%d)
sadf -g /var/log/sa/sa${YESTERDAY} -O packed -- -A | rsvg-convert -w 1600 -o /tmp/perf-$(date -d yesterday +%F).png
```

### Network Graph Examples

```bash
# All interfaces — throughput
sadf -g -- -n DEV | rsvg-convert -w 1200 -o network_traffic.png

# Network errors (dropped, collisions, overruns)
sadf -g -- -n EDEV | rsvg-convert -w 1200 -o network_errors.png

# TCP statistics
sadf -g -- -n TCP | rsvg-convert -w 1200 -o tcp_stats.png

# All network metrics combined
sadf -g -O packed -- -n DEV -n EDEV -n TCP -n SOCK | rsvg-convert -w 1600 -o network_full.png
```

### Memory Graph Examples

```bash
# Memory utilization
sadf -g -- -r | rsvg-convert -w 1200 -o memory.png

# Memory + swap combined
sadf -g -- -r -S | rsvg-convert -w 1200 -o memory_and_swap.png

# Working hours only
sadf -g -s 09:00:00 -e 17:00:00 -- -r -S | rsvg-convert -w 1200 -o memory_working_hours.png
```

## Finding Network Traffic Spikes

One-liners to identify the highest throughput intervals across all stored `sar` data files.

### Top 10 by Total Throughput (rxkB/s + txkB/s)

```bash
{ sar -n DEV -f /var/log/sysstat/sa$(date +%d) 2>/dev/null | awk '/IFACE/{print "TOTAL", $0; exit}'; \
for f in /var/log/sysstat/sa[0-9]*; do
  sar -n DEV -f "$f" 2>/dev/null
done | awk '!/^$/ && !/Average/ && !/LINUX/ && /^[0-9]/ && $3 != "lo" {
  total = $5 + $6
  print total, $0
}' | sort -rn | head -10; }
```

### Top 10 by txkB/s Only

```bash
{ sar -n DEV -f /var/log/sysstat/sa$(date +%d) 2>/dev/null | awk '/IFACE/{print "txkB/s", $0; exit}'; \
for f in /var/log/sysstat/sa[0-9]*; do
  sar -n DEV -f "$f" 2>/dev/null
done | awk '!/^$/ && !/Average/ && !/LINUX/ && /^[0-9]/ && $3 != "lo" {
  print $6, $0
}' | sort -rn | head -10; }
```

### Top 10 for Specific Interface (with source file)

Replace `ens5` with your interface name:

```bash
{ sar -n DEV -f /var/log/sysstat/sa$(date +%d) 2>/dev/null | awk '/IFACE/{print "txkB/s", "FILE", $0; exit}'; \
for f in /var/log/sysstat/sa[0-9]*; do
  sar -n DEV -f "$f" 2>/dev/null | \
  awk -v file="$f" '!/^$/ && !/Average/ && !/LINUX/ && /^[0-9]/ && $3 == "ens5" {
    print $6, file, $0
  }'
done | sort -rn | head -10; }
```

### Single Interface with Labeled Output

```bash
for f in /var/log/sysstat/sa[0-9]*; do
  sar -n DEV -f "$f" 2>/dev/null
done | awk '/ens5/ {total=$5+$6; print $1, $2, $3, "rxkB/s:"$5, "txkB/s:"$6, "total:"total}' \
     | sort -t: -k4 -nr \
     | head -10
```

These scripts loop through all `sa*` files, filter out loopback and headers, calculate throughput, and sort to find peak intervals — useful for identifying traffic bursts or DDoS events.

## Real-Time Tools (sysstat package)

### mpstat — Per-CPU Core

```bash
# All CPUs, every 1 second
mpstat -P ALL 1

# Specific cores
mpstat -P 0,1,2,3 1 5
```

Useful for detecting unbalanced workloads across cores.

### pidstat — Per-Process Statistics

```bash
# CPU usage per process, every 2 seconds
pidstat 2

# I/O per process
pidstat -d 2

# Memory per process
pidstat -r 2

# All stats for a specific process
pidstat -p $(pgrep nginx) 1
```

### iostat — Disk Performance

```bash
# Extended stats, every 2 seconds, 5 iterations
iostat -xz 2 5
```

Key columns: `%util` (>80% = busy), `await` (I/O latency in ms), `r_await`/`w_await`.

## Ubuntu 22.04 vs 24.04 Differences

| Feature | Ubuntu 22.04 | Ubuntu 24.04 |
|---------|-------------|-------------|
| sysstat version | 12.5.2 | 12.6.1+ |
| Collection mechanism | systemd timer + cron | systemd timers only |
| PSI metrics support | No | Yes |
| Default interval | 10 minutes | 10 minutes |
| Data directory | `/var/log/sysstat` | `/var/log/sysstat` |

### PSI Metrics (Ubuntu 24.04+)

Pressure Stall Information shows how much time processes are stalled waiting for resources:

```bash
sar -q PSI-CPU     # CPU pressure
sar -q PSI-IO      # I/O pressure
sar -q PSI-MEM     # Memory pressure
```

### Check the Timer (Ubuntu 24.04)

```bash
systemctl status sysstat-collect.timer
systemctl list-timers | grep sysstat
```

## Daily Performance Report Script

```bash
#!/bin/bash
# /usr/local/bin/daily-perf-report.sh

REPORT_DATE=$(date -d "yesterday" +%d)
REPORT_FILE="/tmp/perf-report-$(date -d yesterday +%F).txt"

{
  echo "=== Daily Performance Report ==="
  echo "Date: $(date -d yesterday +%F)"
  echo ""
  echo "--- CPU Summary ---"
  sar -u -f /var/log/sa/sa${REPORT_DATE} | tail -1
  echo ""
  echo "--- Memory Summary ---"
  sar -r -f /var/log/sa/sa${REPORT_DATE} | tail -1
  echo ""
  echo "--- Disk I/O Summary ---"
  sar -d -p -f /var/log/sa/sa${REPORT_DATE} | tail -5
  echo ""
  echo "--- Network Summary ---"
  sar -n DEV -f /var/log/sa/sa${REPORT_DATE} | tail -5
  echo ""
  echo "--- Load Average Peak ---"
  sar -q -f /var/log/sa/sa${REPORT_DATE} | sort -k5 -rn | head -5
} > "$REPORT_FILE"

echo "Report saved to: $REPORT_FILE"
```

Schedule with cron:

```bash
chmod +x /usr/local/bin/daily-perf-report.sh
echo "0 6 * * * root /usr/local/bin/daily-perf-report.sh" | sudo tee /etc/cron.d/perf-report
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "Cannot open /var/log/sa/saXX" | Check `ENABLED="true"` in `/etc/default/sysstat` |
| No data collected | Verify service: `systemctl status sysstat` |
| Timer not running (24.04) | Check: `systemctl status sysstat-collect.timer` |
| Files exist but empty (< 1KB) | Check logs: `journalctl -u sysstat-collect --since today` |
| Permission issues | Fix: `sudo chown -R root:root /var/log/sysstat && sudo chmod 755 /var/log/sysstat` |

Manually trigger a collection to test:

```bash
sudo /usr/lib/sysstat/debian-sa1 1 1
```

## Tips

- Install sysstat on every server from day one — historical data is invaluable at 3 AM
- Enable the service to automatically collect data every 10 minutes
- Use `sadf` for machine-readable output (CSV, JSON, XML, SVG graphs)
- Combine `sar` with `iostat`, `mpstat`, and `pidstat` for deeper real-time analysis
- Use `-s` and `-e` to focus on specific time windows during investigations
- Archive old `sa` files for long-term capacity planning
- Feed JSON/CSV exports into Grafana or Prometheus pushgateway for dashboards
- On Ubuntu 24.04, use PSI metrics for clearer resource contention visibility

## See Also

- [Configuring sysstat on Ubuntu](configuring-sysstat-ubuntu.md) — Installation, systemd timers, collection intervals, and configuration files.
