# Monitor Interface Traffic — RX/TX Packets and Bandwidth

Commands to monitor real-time network traffic, packets per second, and throughput on a specific interface.

## sar (sysstat)

```bash
# Real-time stats every 1 second, filter for eth0
sar -n DEV 1 | grep eth0

# Show header + eth0 lines (easier to read columns)
sar -n DEV 1 | grep -E "IFACE|eth0"

# Only packets columns (rxpck/s, txpck/s)
sar -n DEV 1 | grep eth0 | awk '{print $1, $2, "rxpck:", $3, "txpck:", $4}'

# Historical stats from today
sar -n DEV -f /var/log/sysstat/sa$(date +%d) | grep eth0

# Last 10 minutes
sar -n DEV -s $(date -d '10 min ago' +%H:%M:%S) | grep eth0
```

Output columns: `IFACE rxpck/s txpck/s rxkB/s txkB/s rxcmp/s txcmp/s rxmcst/s`

## ip (iproute2)

```bash
# Snapshot of RX/TX packets and bytes
ip -s link show eth0

# Watch changes every 1 second
watch -n 1 'ip -s link show eth0'

# JSON output (scriptable)
ip -s -j link show eth0 | jq '.[0].stats64'
```

## /proc/net/dev

```bash
# Raw counters
cat /proc/net/dev | grep eth0

# Watch live
watch -n 1 'cat /proc/net/dev | grep eth0'
```

## ifstat

```bash
# All interfaces, every 1 second
ifstat 1

# Specific interface
ifstat -i eth0 1

# With totals
ifstat -t -i eth0 1
```

## nload

```bash
# Visual real-time bandwidth graph
nload eth0

# Multiple interfaces
nload eth0 eth1
```

## iftop

```bash
# Live per-connection bandwidth (requires root)
sudo iftop -i eth0

# Show port numbers
sudo iftop -i eth0 -P

# Don't resolve hostnames (faster)
sudo iftop -i eth0 -n
```

## vnstat

```bash
# Live rate monitoring
vnstat -l -i eth0

# 5-second interval stats
vnstat -tr 5 -i eth0

# Daily/monthly summaries
vnstat -d -i eth0
vnstat -m -i eth0
```

## bmon

```bash
# Curses-based real-time monitor
bmon -p eth0
```

## ethtool

```bash
# NIC-level stats (rx/tx packets, errors, drops)
ethtool -S eth0

# Filter for packets
ethtool -S eth0 | grep -i "packets"
```

## netstat / ss

```bash
# Interface stats summary
netstat -i

# ss socket stats
ss -s
```

## dstat

```bash
# Combined CPU + network view
dstat -n -N eth0

# Packets + bytes
dstat --net-packets -N eth0
```

## nstat / rtacct

```bash
# Kernel network counters (per second)
nstat -a -z | grep -i eth0

# Reset and show delta
nstat -r
```

## One-Liner: Calculate PPS from /proc

```bash
# Packets per second (RX + TX)
while true; do
  r1=$(cat /sys/class/net/eth0/statistics/rx_packets)
  t1=$(cat /sys/class/net/eth0/statistics/tx_packets)
  sleep 1
  r2=$(cat /sys/class/net/eth0/statistics/rx_packets)
  t2=$(cat /sys/class/net/eth0/statistics/tx_packets)
  echo "RX: $((r2-r1)) pps | TX: $((t2-t1)) pps"
done
```

## One-Liner: Calculate Bandwidth from /proc

```bash
# Bytes/KB per second (RX + TX)
while true; do
  r1=$(cat /sys/class/net/eth0/statistics/rx_bytes)
  t1=$(cat /sys/class/net/eth0/statistics/tx_bytes)
  sleep 1
  r2=$(cat /sys/class/net/eth0/statistics/rx_bytes)
  t2=$(cat /sys/class/net/eth0/statistics/tx_bytes)
  echo "RX: $(( (r2-r1) / 1024 )) KB/s | TX: $(( (t2-t1) / 1024 )) KB/s"
done
```

## Interface Errors and Drops (sar -n EDEV)

```bash
# Error stats per interface (complements -n DEV)
sar -n EDEV 1 | grep -E "IFACE|eth0"
```

Output columns: `IFACE rxerr/s txerr/s coll/s rxdrop/s txdrop/s txcarr/s rxfram/s rxfifo/s txfifo/s`

## Link Speed and Utilization

```bash
# Check link speed (Mbps)
cat /sys/class/net/eth0/speed

# Check link state
cat /sys/class/net/eth0/operstate

# Calculate % utilization (requires link speed and current throughput)
SPEED=$(cat /sys/class/net/eth0/speed)  # e.g. 1000 (Mbps)
RX_BYTES1=$(cat /sys/class/net/eth0/statistics/rx_bytes)
sleep 1
RX_BYTES2=$(cat /sys/class/net/eth0/statistics/rx_bytes)
RX_MBPS=$(( (RX_BYTES2 - RX_BYTES1) * 8 / 1000000 ))
echo "RX: ${RX_MBPS} Mbps / ${SPEED} Mbps = $(( RX_MBPS * 100 / SPEED ))% utilization"
```

## Queue Discipline Drops (tc)

```bash
# Show queuing discipline stats (drops from traffic shaping)
tc -s qdisc show dev eth0

# Watch for growing "dropped" counter
watch -n 1 'tc -s qdisc show dev eth0'
```

## Summary

| Tool | Shows | Install |
|------|-------|---------|
| `sar` | PPS, bandwidth, errors | `sysstat` |
| `ip -s` | Cumulative counters | built-in |
| `ifstat` | RX/TX bandwidth | `ifstat` |
| `nload` | Bandwidth graph | `nload` |
| `iftop` | Per-connection bandwidth | `iftop` |
| `vnstat` | Live + historical | `vnstat` |
| `bmon` | Curses monitor | `bmon` |
| `ethtool -S` | NIC-level counters | `ethtool` |
| `dstat` | Combined system stats | `dstat` |
