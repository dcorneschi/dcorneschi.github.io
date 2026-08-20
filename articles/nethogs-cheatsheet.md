# NetHogs Cheatsheet

NetHogs is a per-process network bandwidth monitoring tool. Unlike `iftop` or `nload` which show traffic per interface, NetHogs groups bandwidth by process — making it easy to identify which application is consuming network resources.

## Installation

```bash
# RHEL/CentOS/Fedora
sudo dnf install -y nethogs

# Ubuntu/Debian
sudo apt install -y nethogs

# From EPEL (if not in base repos)
sudo dnf install -y epel-release && sudo dnf install -y nethogs

# macOS (limited/experimental support)
brew install nethogs

# From source
git clone https://github.com/raboof/nethogs
cd nethogs && make && sudo make install
```

## Basic Usage

```bash
# Monitor all interfaces (requires root)
sudo nethogs

# Monitor a specific interface
sudo nethogs eth0

# Monitor multiple interfaces
sudo nethogs eth0 wlan0

# Monitor with a specific refresh interval (seconds)
sudo nethogs -d 2

# Monitor in KB/s instead of default KB/s
sudo nethogs -v 0    # KB/s
sudo nethogs -v 1    # Total KB
sudo nethogs -v 2    # Total B
sudo nethogs -v 3    # Total MB
```

## Command Line Options

| Option | Description |
|--------|-------------|
| `-d <seconds>` | Refresh interval (default: 1) |
| `-v <mode>` | View mode: 0=KB/s, 1=total KB, 2=total B, 3=total MB |
| `-c <count>` | Number of refreshes, then exit (for scripting) |
| `-t` | Tracemode (output suitable for parsing) |
| `-p` | Promiscuous mode (sniff packets on all hosts in the subnet) |
| `-s` | Sort output by sent traffic |
| `-a` | Monitor all devices (even loopback) |
| `device` | Interface(s) to monitor (default: all non-loopback) |

## Interactive Keys

While NetHogs is running:

| Key | Action |
|-----|--------|
| `q` | Quit |
| `s` | Sort by SENT traffic |
| `r` | Sort by RECEIVED traffic |
| `m` | Cycle display mode (KB/s → KB → B → MB → KB/s) |

## Output Columns

```
PID    USER    PROGRAM                           DEV     SENT      RECEIVED
1234   root    /usr/bin/apt                      eth0    45.2      1024.5   KB/sec
5678   nginx   /usr/sbin/nginx                   eth0    512.0     128.3    KB/sec
9012   user    sshd: user@pts/0                  eth0    1.2       0.8      KB/sec
```

| Column | Description |
|--------|-------------|
| PID | Process ID |
| USER | Process owner |
| PROGRAM | Full path to the executable |
| DEV | Network interface |
| SENT | Outgoing traffic |
| RECEIVED | Incoming traffic |

## Practical Examples

### Find Top Bandwidth Consumers

```bash
# Run and sort by received (downloads)
sudo nethogs -d 1

# Press 'r' to sort by received
# Press 's' to sort by sent
```

### Monitor Specific Interface

```bash
# Monitor only the primary ethernet
sudo nethogs ens192

# Monitor only the Docker bridge
sudo nethogs docker0

# Monitor VPN interface
sudo nethogs tun0
```

### Non-Interactive Mode (Scripting)

```bash
# Run for 10 refresh cycles and exit
sudo nethogs -c 10 -d 5 eth0

# Tracemode output (parseable)
sudo nethogs -t -c 5 -d 2 eth0

# Capture to file
sudo nethogs -t -c 60 -d 1 eth0 > /tmp/nethogs-output.txt 2>&1
```

### Parse Tracemode Output

```bash
# Capture and extract top talkers
sudo nethogs -t -c 10 -d 1 eth0 2>/dev/null | \
    grep -v "^$\|Refreshing\|unknown" | \
    sort -t$'\t' -k2 -n -r | head -10
```

### Monitor During a Specific Time Window

```bash
# Run for 5 minutes (300 refreshes at 1-second intervals)
sudo timeout 300 nethogs -t -d 1 eth0 > /tmp/nethogs-5min.log 2>&1
```

### Identify Bandwidth Hogs in Real-Time

```bash
# Quick check — who's using bandwidth right now?
sudo nethogs -d 1 -c 5

# Show totals instead of rate
sudo nethogs -v 1 -d 1 -c 10
```

## Combining with Other Tools

### With watch

```bash
# Refresh every 2 seconds showing summary
watch -n 2 'sudo nethogs -c 1 -t 2>/dev/null | grep -v "^$" | head -20'
```

### With grep (Filter Specific Process)

```bash
# Monitor only nginx traffic
sudo nethogs -t -d 1 eth0 2>/dev/null | grep nginx
```

### Compare with iftop

```bash
# nethogs — per-process bandwidth
sudo nethogs eth0

# iftop — per-connection bandwidth (host pairs)
sudo iftop -i eth0

# nload — per-interface total bandwidth
nload eth0
```

## Troubleshooting

### "Permission denied"

```bash
# nethogs requires root or CAP_NET_RAW capability
sudo nethogs

# Or set capability (persistent)
sudo setcap cap_net_raw,cap_net_admin=eip /usr/sbin/nethogs
# Then run without sudo:
nethogs eth0
```

### "No such device"

```bash
# List available interfaces
ip link show

# Use the correct interface name
sudo nethogs ens192    # not eth0 on newer systems
```

### No Output / Empty Display

```bash
# Ensure there is active network traffic
# nethogs only shows processes with active connections

# Try monitoring all interfaces
sudo nethogs -a

# Check if the interface has traffic
ip -s link show eth0
```

### "Unknown connection" Entries

```bash
# These are connections that nethogs can't map to a PID
# Common causes:
# - Kernel traffic (iptables, conntrack)
# - Short-lived connections that closed before mapping
# - Traffic from containers or network namespaces

# Use -p for promiscuous mode to capture more
sudo nethogs -p eth0
```

## NetHogs vs Other Tools

| Tool | Shows | Granularity | Requires Root |
|------|-------|-------------|:-------------:|
| nethogs | Per-process bandwidth | Process | Yes |
| iftop | Per-connection bandwidth | Host pair | Yes |
| nload | Per-interface bandwidth | Interface | No |
| bmon | Per-interface with graphs | Interface | No |
| vnstat | Historical interface stats | Interface | No |
| ss/netstat | Connection states | Socket | No |
| tcpdump | Raw packets | Packet | Yes |

## Log Bandwidth with Timestamps

```bash
# Append timestamps to tracemode output
sudo nethogs -t -d 5 eth0 | while read line; do
    echo "$(date '+%Y-%m-%d %H:%M:%S') $line"
done >> /var/log/nethogs.log
```

## Per-IP Packet Counts (Alternatives)

NetHogs doesn't track per-IP packet counts. Use these tools instead:

### conntrack (Best for Packet Counts per IP)

```bash
sudo conntrack -L -o extended | \
    awk '{for(i=1;i<=NF;i++) if($i ~ /src=|packets=/) printf "%s ", $i; print ""}' | \
    sort | uniq -c | sort -rn
```

### tcpdump + awk (Quick Snapshot)

```bash
# Capture and summarize (Ctrl+C to stop, or use -c to limit)
sudo tcpdump -nn -q -t -i eth0 2>/dev/null | \
    awk '{print $2}' | cut -d. -f1-4 | sort | uniq -c | sort -rn | head -20
```

### iptables Accounting

```bash
# Add counters
sudo iptables -I INPUT -j ACCEPT
sudo iptables -I OUTPUT -j ACCEPT

# View packet counts per source IP
sudo iptables -L INPUT -v -n | sort -k1 -rn
```

### ss (Connected IPs with Connection Count)

```bash
ss -tn | awk 'NR>1 {print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn
```

### iptraf-ng (Real-Time per-IP Packets)

```bash
sudo iptraf-ng
# Navigate to: IP Traffic Monitor → select interface
```

### Per-IP Tool Selection

| Goal | Tool |
|------|------|
| Real-time per-IP packets | `iptraf-ng` |
| Historical/logged counts | `conntrack -L` |
| Quick snapshot | `tcpdump` + `awk` |
| Long-term accounting | `iptables` or `vnstat` |

## Notes

- Requires root/sudo (uses libpcap)
- Works on Linux; macOS support is experimental
- Doesn't capture UDP well on older versions — update to latest if needed
- On containers, run on the host or use `--net=host`
- The `-p` flag enables promiscuous mode (sniff packets on all hosts in the subnet)

## Quick Reference

| Action | Command |
|--------|---------|
| Basic monitoring | `sudo nethogs` |
| Specific interface | `sudo nethogs eth0` |
| Set refresh rate | `sudo nethogs -d 2` |
| Show totals (KB) | `sudo nethogs -v 1` |
| Tracemode (parseable) | `sudo nethogs -t` |
| Fixed number of cycles | `sudo nethogs -c 10` |
| Sort by sent | Press `s` |
| Sort by received | Press `r` |
| Cycle display mode | Press `m` |
| Quit | Press `q` |
