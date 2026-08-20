# Performance Co-Pilot (PCP) Cheatsheet

Performance Co-Pilot (PCP) is an open source framework for monitoring, analyzing, and responding to live and historical system performance. It uses a distributed, plug-in based architecture with collector hosts (running PMDAs) and monitor hosts.

## Installation

### Collector Host (Monitored System)

```bash
# RHEL/CentOS/Fedora
sudo dnf install pcp

# Ubuntu/Debian
sudo apt install pcp

# Enable and start services
sudo systemctl enable --now pmcd pmlogger
```

### Monitor Host (Analysis System)

```bash
# Additional monitoring tools
sudo dnf install pcp-doc pcp-gui pcp-system-tools    # RHEL/Fedora
sudo apt install pcp-doc pcp-gui pcp-system-tools    # Ubuntu/Debian
```

### Zero-Config (Quick Start)

```bash
# Installs and enables everything with sensible defaults
sudo dnf install pcp-zeroconf    # RHEL/Fedora
sudo apt install pcp-zeroconf    # Ubuntu/Debian
```

## Core Services

| Service | Purpose |
|---------|---------|
| `pmcd` | Performance Metrics Collector Daemon — serves metrics to clients |
| `pmlogger` | Archives metrics to disk for retrospective analysis |
| `pmie` | Performance Metrics Inference Engine — rules and alerts |
| `pmproxy` | REST API and time series proxy (for Grafana) |

```bash
# Check PCP status
pcp

# Service management
sudo systemctl status pmcd pmlogger pmie
sudo systemctl restart pmcd
```

## PMDAs (Performance Metrics Domain Agents)

PMDAs provide metrics from different subsystems:

```bash
# List installed PMDAs
ls /var/lib/pcp/pmdas/

# Install a PMDA (e.g., PostgreSQL)
cd /var/lib/pcp/pmdas/postgresql
sudo ./Install

# Remove a PMDA
cd /var/lib/pcp/pmdas/postgresql
sudo ./Remove

# Common PMDAs:
# linux     — CPU, memory, disk, network (enabled by default)
# proc      — per-process metrics
# xfs       — XFS filesystem metrics
# nfsclient — NFS client metrics
# postgresql — PostgreSQL database
# mysql     — MySQL/MariaDB
# docker    — Docker containers
# podman    — Podman containers
```

## Listing and Exploring Metrics

```bash
# List all available metrics
pminfo

# List metrics matching a pattern
pminfo disk
pminfo mem.util
pminfo network.interface

# Show metric description
pminfo -t disk.dev.read

# Show metric details (description, type, semantics, units, values)
pminfo -dfmtT disk.partitions.read

# Count available metrics
pminfo | wc -l
```

## Live Monitoring

### pmval — Monitor a Single Metric

```bash
# Monitor disk writes per partition every 2 seconds
pmval -t 2sec -f 3 disk.partitions.write

# Monitor specific instance
pmval -t 2sec -i sda 'disk.dev.read'

# Monitor memory free
pmval -t 1sec mem.freemem

# Monitor CPU user time
pmval -t 2sec kernel.all.cpu.user

# Monitor from remote host
pmval -t 2sec -h acme.com mem.freemem
```

### pmrep — Flexible Reporting

```bash
# CPU, memory, and disk with timestamps in CSV
pmrep -p -b GB -t 2sec -o csv kernel.all.sysfork mem.util.free mem.util.used

# Use predefined metricsets (from pmrep.conf)
pmrep -t 5sec :vmstat
pmrep -t 5sec :sar-u
pmrep -t 5sec :disk-total
pmrep -t 5sec :network-interface

# Monitor specific interface
pmrep -i eth0 -v network.interface.out

# Output to file
pmrep -t 5sec -o csv -F output.csv :vmstat
```

### pmstat — System Overview (vmstat-like)

```bash
# Local system
pmstat -t 2sec

# Monitor multiple hosts
pmstat -t 2sec -h host1.com -h host2.com

# From an archive
pmstat -t 10m -S @09:00 -T @10:00 -a /var/log/pcp/pmlogger/hostname/20240901
```

### pmiostat — Disk I/O (iostat-like)

```bash
# Live disk I/O every 2 seconds
pmiostat -t 2sec

# From archive
pmiostat -t 1h -a /var/log/pcp/pmlogger/hostname/20240901
```

### pcp atop / pcp htop — Interactive Monitoring

```bash
# top-like system monitoring
pcp atop

# htop-like system monitoring
pcp htop

# sar-like reporting
pcp atopsar

# free-like memory display
pcp free

# uptime-like load display
pcp uptime

# dstat-like combined metrics
pcp dstat
```

### pcp-dstat — Detailed Usage

```bash
pcp dstat                # Default output
pcp dstat 5              # Update every 5 seconds
pcp dstat -c             # CPU stats only
pcp dstat -d             # Disk stats only
pcp dstat -n             # Network stats only
pcp dstat -m             # Memory stats only
pcp dstat -cdn           # CPU, disk, network combined
pcp dstat --top-cpu      # Top CPU consuming process
pcp dstat --top-mem      # Top memory consuming process
```

### pcp-iostat — Extended Disk Stats

```bash
pcp iostat               # Default I/O stats
pcp iostat 5             # Update every 5 seconds
pcp iostat -x            # Extended stats (await, svctm, %util)
pcp iostat -x 2          # Extended with 2-second interval
```

### pmdumptext — Custom Columns

```bash
# CPU load, memory, disk writes with timestamps
pmdumptext -Xlimu -t 2sec 'kernel.all.load[1]' mem.util.used disk.partitions.write

# Monitor remote host
pmdumptext -Xlimu -t 2sec -h acme.com 'kernel.all.load[1]' mem.util.used

# Save to file with specific samples
pmdumptext -t 60 -s 60 'kernel.all.load[1]' > load.txt

# CSV delimiter for export
pmdumptext -d, -a archive 'kernel.all.load[1]' 'mem.util.used' > metrics.csv
```

## Process Monitoring

```bash
# List all process metrics
pminfo proc

# Monitor open file descriptors for PID 1234
pmval -t 2sec 'proc.fd.count[1234]'

# CPU time, RSS, and threads for PID 1234
pmdumptext -Xlimu -t 2sec 'proc.psinfo.utime[1234]' 'proc.memory.rss[1234]' 'proc.psinfo.threads[1234]'

# Monitor "hot" processes by name
sudo pmstore hotproc.control.config 'fname == "java"'
pminfo -f hotproc
```

## Archive Management

Archives are stored under `/var/log/pcp/pmlogger/<hostname>/` and are self-contained, machine-independent.

### Inspect Archives

```bash
# Check what an archive covers (host, time range, timezone)
pmdumplog -L /var/log/pcp/pmlogger/hostname/20240901

# Check PCP config at the time of archive creation
pcp -a /var/log/pcp/pmlogger/hostname/20240901

# List metrics available in an archive
pminfo -a /var/log/pcp/pmlogger/hostname/20240901
```

### Replay from Archives

```bash
# Replay metric values from archive
pmval -f 3 disk.partitions.write -a /var/log/pcp/pmlogger/hostname/20240901

# Replay between 9 AM and 10 AM with 2-second interval
pmval -d -t 2sec -f 3 disk.partitions.write \
    -S @09:00 -T @10:00 -a /var/log/pcp/pmlogger/hostname/20240901

# Replay with pmrep
pmrep -a /var/log/pcp/pmlogger/hostname/20240901 -A 5min -t 5min -Z UTC :vmstat

# Replay in atop
pcp atop -b 09:00 -r /var/log/pcp/pmlogger/hostname/20240901

# Replay in pmstat
pmstat -t 10m -S @09:00 -T @10:00 -a /var/log/pcp/pmlogger/hostname/20240901
```

### Archive Statistics

```bash
# Calculate averages, min/max between 9 AM and 10 AM
pmlogsummary -HlfiImM -S @09:00 -T @10:00 \
    /var/log/pcp/pmlogger/hostname/20240901 disk.partitions.write mem.freemem

# Compare two archives (different time periods)
pmdiff -S @02:00 -T @03:00 -B @09:00 -E @10:00 \
    /var/log/pcp/pmlogger/hostname/20240902 /var/log/pcp/pmlogger/hostname/20240901

# Merge archives
pmlogextract archive1 archive2 merged-archive
```

## Remote Monitoring

```bash
# Query remote host
pminfo -f -h remotehost mem.util.free

# Monitor remote host metrics
pmval -t 2sec -h remotehost kernel.all.load

# Add remote host for centralized logging
echo "remotehost n n PCP_LOG_DIR/pmlogger/remotehost -r -T24h10m -c config.remotehost" | \
    sudo tee -a /etc/pcp/pmlogger/control
sudo systemctl restart pmlogger

# Discover PCP hosts on the network
pmfind -s pmcd
```

## PMIE (Alerts and Rules)

```bash
# Enable PMIE
sudo systemctl enable --now pmie

# Example rule: alert if memory usage > 5 GB
cat << 'EOF' > /tmp/pmie-rules.conf
bloated = (mem.util.used > 5 Gbyte) -> print "%v memory used on %h!";
EOF

# Check syntax
pmie -C /tmp/pmie-rules.conf

# Run against an archive
pmie -t 1min -c /tmp/pmie-rules.conf -S @09:00 -T @10:00 \
    -a /var/log/pcp/pmlogger/hostname/20240901

# Run live
pmie -t 30sec -c /tmp/pmie-rules.conf
```

## Web Interface (Grafana)

```bash
# Install and enable PCP web services
sudo systemctl enable --now pmproxy valkey

# Install Grafana PCP plugin
sudo dnf install grafana-pcp    # RHEL/Fedora
sudo apt install grafana-pcp    # Ubuntu/Debian

# Access Grafana
# http://localhost:3000
```

## Importing sar/iostat Data

```bash
# Import iostat data to PCP archive
iostat -t -x 2 > iostat.out
iostat2pcp iostat.out iostat.pcp
pmchart -t 2sec -a iostat.pcp

# Import sar data to PCP archive
sar2pcp /var/log/sa/sa15 sar.pcp
pmchart -t 2sec -a sar.pcp

# Packages needed:
# pcp-import-iostat2pcp
# pcp-import-sar2pcp
```

## Common Metrics Reference

### CPU

| Metric | Description |
|--------|------------|
| `kernel.all.load` | Load averages (1, 5, 15 min) |
| `kernel.all.cpu.user` | User CPU time |
| `kernel.all.cpu.sys` | System CPU time |
| `kernel.all.cpu.idle` | Idle CPU time |
| `kernel.all.cpu.wait.total` | I/O wait time |
| `kernel.all.nprocs` | Number of processes |
| `kernel.all.runnable` | Runnable processes |

### Memory

| Metric | Description |
|--------|------------|
| `mem.util.free` | Free memory |
| `mem.util.used` | Used memory |
| `mem.util.cached` | Page cache |
| `mem.util.bufmem` | Buffer memory |
| `mem.util.swapFree` | Free swap |
| `mem.util.available` | Available memory |

### Disk

| Metric | Description |
|--------|------------|
| `disk.dev.read` | Reads per device |
| `disk.dev.write` | Writes per device |
| `disk.dev.total_bytes` | Bytes transferred per device |
| `disk.dev.avactive` | Average active time |
| `disk.all.read` | Total reads (all devices) |
| `disk.all.write` | Total writes (all devices) |

### Network

| Metric | Description |
|--------|------------|
| `network.interface.in.bytes` | Bytes received |
| `network.interface.out.bytes` | Bytes transmitted |
| `network.interface.in.packets` | Packets received |
| `network.interface.out.packets` | Packets transmitted |
| `network.interface.in.errors` | Receive errors |
| `network.interface.out.errors` | Transmit errors |

### Process

| Metric | Description |
|--------|------------|
| `proc.psinfo.utime` | User CPU time |
| `proc.psinfo.stime` | System CPU time |
| `proc.memory.rss` | Resident set size |
| `proc.memory.vmsize` | Virtual memory size |
| `proc.psinfo.threads` | Thread count |
| `proc.fd.count` | Open file descriptors |

## Manual Archive Logging

```bash
# Log every 60 seconds for 24 hours (1440 samples)
pmlogger -t 60s -s 1440 /tmp/mylog

# Log specific metrics only
pmlogger -t 10s -c myconfig.conf /tmp/custom-archive

# Configure logging retention per host
sudo vi /etc/pcp/pmlogger/control
# Format: HOSTNAME  y  n  DIRECTORY  -t INTERVAL
# Example: localhost y n /var/log/pcp/pmlogger/localhost -t 60s
```

### Archive Verification and Maintenance

```bash
# Verify archive integrity
pmlogcheck /var/log/pcp/pmlogger/hostname/20240901

# Compress old archives
find /var/log/pcp/pmlogger -name "*.0" -mtime +7 -exec gzip {} \;

# Extract specific time range from archive
pmlogextract -S @10:00 -T @18:00 input_archive output_archive
```

## PMIE Alert Rules

### Rule Syntax

```
metric_expression -> action;

# Actions:
# print "message"       — print to stdout
# alarm "message"       — trigger alarm
# shell "command"       — run a shell command
# syslog "message"      — write to syslog
```

### Practical Rule Examples

```bash
cat > /etc/pcp/pmie/custom-rules.pmie << 'EOF'
// High CPU usage alert
kernel.all.cpu.user > 80 ->
    shell "echo 'High CPU: %v%%' | mail -s 'PCP Alert: CPU' admin@example.com";

// Low free memory alert
mem.util.free < 500 Mbyte ->
    shell "echo 'Low memory: %v free on %h' | logger -t pcp-alert";

// High load average
kernel.all.load[1] > 10 ->
    print "High load on %h: %v";

// Disk I/O alert
disk.dev.total[sda] > 1000 ->
    shell "logger 'High disk I/O on sda: %v ops/s'";

// Low disk space
filesys.free < 10 %_sample filesys.capacity ->
    shell "echo 'Low disk space on %h' | mail -s 'Disk Alert' admin@example.com";
EOF

# Check syntax
pmie -C /etc/pcp/pmie/custom-rules.pmie

# Run against live system
pmie -t 30sec -c /etc/pcp/pmie/custom-rules.pmie

# Run against archive (test rules historically)
pmie -t 1min -c /etc/pcp/pmie/custom-rules.pmie \
    -S @09:00 -T @17:00 -a /var/log/pcp/pmlogger/hostname/20240901

# Enable as service
sudo systemctl enable --now pmie
```

## Custom PMDA (Python)

```bash
# Create PMDA directory
sudo mkdir -p /var/lib/pcp/pmdas/myapp
cd /var/lib/pcp/pmdas/myapp

# Create Python PMDA
cat > pmda.py << 'EOF'
from pcp.pmda import PMDA, pmdaMetric
import cpmapi as c_api

class MyAppPMDA(PMDA):
    def __init__(self, name, domain):
        PMDA.__init__(self, name, domain)

    def myapp_fetch(self, item, inst):
        return [42, 1]

if __name__ == '__main__':
    MyAppPMDA('myapp', 245).run()
EOF

# Install the PMDA
sudo ./Install

# Verify
pminfo myapp
pmval myapp.metric_name
```

## Data Export

```bash
# Export to CSV
pmdumptext -d, -a archive 'kernel.all.load[1]' 'mem.util.used' > metrics.csv

# Export to JSON (use pcp2json)
pcp2json -t 1 -s 1 kernel.all.load > metrics.json

# Export specific time range
pmdumptext -d, -S @09:00 -T @17:00 -a archive 'kernel.all.cpu.user' > cpu.csv
```

## Troubleshooting

### Service Issues

```bash
# Check service status
sudo systemctl status pmcd pmlogger pmie
sudo journalctl -u pmcd --no-pager -n 50
sudo journalctl -u pmlogger --no-pager -n 50

# View PCP logs
sudo tail -f /var/log/pcp/pmcd/pmcd.log
sudo tail -f /var/log/pcp/pmlogger/pmlogger.log

# Restart services
sudo systemctl restart pmcd pmlogger
```

### Metric Issues

```bash
# Verify metrics are available
pminfo -f kernel.all.load

# Check PMDA status
pminfo -f pmcd.agent.status

# Reinstall a PMDA
cd /var/lib/pcp/pmdas/linux
sudo ./Remove
sudo ./Install
sudo systemctl restart pmcd
```

### Archive Issues

```bash
# Verify archive integrity
pmlogcheck archive_file

# Check archive disk space
df -h /var/log/pcp/pmlogger/

# Rebuild corrupt archive (extract valid portion)
pmlogextract -S @start -T @end bad_archive recovered_archive
```

### Connection Issues

```bash
# Test remote PMCD connectivity (default port 44321)
pminfo -h remote_host kernel.all.load

# Check firewall
sudo firewall-cmd --list-ports | grep 44321
sudo firewall-cmd --permanent --add-port=44321/tcp
sudo firewall-cmd --reload
```

## Comparison with Other Tools

| Feature | PCP | sar/sysstat | Prometheus | Nagios |
|---------|-----|-------------|------------|--------|
| Architecture | Distributed | Local | Pull-based | Agent-based |
| Historical data | Built-in archives | sa files | TSDB | Plugins |
| Visualization | pmchart, Grafana | Text/sadf | Grafana | Web UI |
| Alerting | pmie | None (manual) | Alertmanager | Built-in |
| Custom metrics | PMDAs | None | Exporters | Plugins |
| Cloud-native | No | No | Yes | No |
| Overhead | Very low | Very low | Low-medium | Low |
| Process-level | Yes | Limited | Via node_exporter | Limited |

## Configuration Files

| File | Purpose |
|------|---------|
| `/etc/pcp/pmcd/pmcd.conf` | PMCD configuration (PMDAs) |
| `/etc/pcp/pmlogger/control` | Hosts to log |
| `/etc/pcp/pmie/control` | Hosts for PMIE rules |
| `/var/log/pcp/pmlogger/` | Archive storage location |
| `/var/lib/pcp/pmdas/` | PMDA installation directories |

## Quick Reference

| Action | Command |
|--------|---------|
| Check PCP status | `pcp` |
| List metrics | `pminfo` |
| Metric details | `pminfo -dfmtT <metric>` |
| Monitor metric | `pmval -t 2sec <metric>` |
| System overview | `pmstat -t 2sec` |
| Disk I/O | `pmiostat -t 2sec` |
| Interactive (atop) | `pcp atop` |
| Interactive (htop) | `pcp htop` |
| Interactive (dstat) | `pcp dstat` |
| Flexible reporting | `pmrep -t 5sec :vmstat` |
| Archive info | `pmdumplog -L <archive>` |
| Replay archive | `pmval -a <archive> <metric>` |
| Compare archives | `pmdiff <archive1> <archive2>` |
| Verify archive | `pmlogcheck <archive>` |
| Remote host | `pmval -h <host> <metric>` |
| Discover hosts | `pmfind -s pmcd` |
| Export CSV | `pmdumptext -d, -a archive metric > out.csv` |
| Export JSON | `pcp2json -t 1 -s 1 <metric>` |
