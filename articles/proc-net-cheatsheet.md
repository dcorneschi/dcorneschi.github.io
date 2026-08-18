# /proc/net — Linux Network Statistics from the Kernel

`/proc/net/` is a virtual filesystem exposing real-time network statistics directly from the kernel. No tools needed — just `cat` the files. Every monitoring tool (ss, netstat, Prometheus node_exporter) ultimately reads from here.

## Quick Reference: Key Files

| File | What It Shows |
|------|--------------|
| `/proc/net/sockstat` | Socket counts and memory (TCP, UDP, RAW) |
| `/proc/net/sockstat6` | Same for IPv6 |
| `/proc/net/tcp` | Every IPv4 TCP socket with full details |
| `/proc/net/tcp6` | Every IPv6 TCP socket |
| `/proc/net/udp` | Every IPv4 UDP socket |
| `/proc/net/udp6` | Every IPv6 UDP socket |
| `/proc/net/snmp` | Protocol counters (TCP/UDP/IP/ICMP) |
| `/proc/net/netstat` | Extended TCP/IP stats (drops, aborts, ECN) |
| `/proc/net/dev` | Per-interface traffic counters |
| `/proc/net/arp` | ARP table (IP to MAC mappings) |
| `/proc/net/route` | Kernel routing table |
| `/proc/net/if_inet6` | IPv6 addresses per interface |
| `/proc/net/ipv6_route` | IPv6 routing table |
| `/proc/net/unix` | UNIX domain sockets |
| `/proc/net/raw` | Raw sockets |
| `/proc/net/nf_conntrack` | Netfilter connection tracking table |
| `/proc/net/stat/nf_conntrack` | Conntrack stats (drops, inserts, searches) |
| `/proc/net/fib_trie` | Full routing trie (all routes) |
| `/proc/net/bonding/*` | Bond interface status |

## /proc/net/sockstat — Socket Summary

```bash
cat /proc/net/sockstat
```

```
sockets: used 356
TCP: inuse 86 orphan 0 tw 12 alloc 98 mem 45
UDP: inuse 5 mem 2
UDPLITE: inuse 0
RAW: inuse 1
FRAG: inuse 0 memory 0
```

| Field | Meaning |
|-------|---------|
| `TCP inuse` | Active TCP sockets (ESTABLISHED, CLOSE_WAIT, FIN_WAIT, etc.) |
| `TCP orphan` | Sockets with no process (app died without closing) |
| `TCP tw` | TIME_WAIT sockets |
| `TCP alloc` | Total allocated (all states including LISTEN) |
| `TCP mem` | Memory in pages (x4KB) |
| `UDP inuse` | Active UDP sockets |
| `FRAG memory` | Memory used for IP fragment reassembly |

```bash
# Quick one-liner
awk '/TCP/ {printf "InUse:%s Orphan:%s TW:%s Alloc:%s Mem:%s pages (%dKB)\n",$2,$4,$6,$8,$10,$10*4}' /proc/net/sockstat
```

## /proc/net/tcp — All TCP Sockets (Raw)

This is what `ss` and `netstat` read. One line per socket.

```bash
cat /proc/net/tcp | head -5
```

```
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode
   0: 0100007F:0035 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 12345
   1: 00000000:0016 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 23456
   2: 0A00000A:0016 0A000005:D431 01 00000000:00000000 02:000000FF 00000000     0        0 34567
```

### Decoding the Fields

| Column | Meaning | Notes |
|--------|---------|-------|
| `sl` | Slot number | Sequential |
| `local_address` | Local IP:Port | Hex (little-endian IP, big-endian port) |
| `rem_address` | Remote IP:Port | Same format |
| `st` | State | Hex: 01=ESTABLISHED, 0A=LISTEN, 06=TIME_WAIT |
| `tx_queue:rx_queue` | Send/receive queue bytes | |
| `tr:tm->when` | Timer type and countdown | |
| `retrnsmt` | Retransmit counter | |
| `uid` | Owner UID | |
| `timeout` | Socket timeout | |
| `inode` | Socket inode number | Links to process via /proc/PID/fd |

### State Hex Codes

| Hex | State |
|-----|-------|
| `01` | ESTABLISHED |
| `02` | SYN_SENT |
| `03` | SYN_RECV |
| `04` | FIN_WAIT1 |
| `05` | FIN_WAIT2 |
| `06` | TIME_WAIT |
| `07` | CLOSE |
| `08` | CLOSE_WAIT |
| `09` | LAST_ACK |
| `0A` | LISTEN |
| `0B` | CLOSING |

### Decode IP Address from Hex

```bash
# Convert hex IP (little-endian) to human-readable
# Example: 0100007F = 127.0.0.1
printf '%d.%d.%d.%d\n' 0x7F 0x00 0x00 0x01

# One-liner to decode /proc/net/tcp
awk 'NR>1 {
    split($2,l,":"); split($3,r,":");
    lp=strtonum("0x"l[2]); rp=strtonum("0x"r[2]);
    printf "Local: %d.%d.%d.%d:%d  Remote: %d.%d.%d.%d:%d  State: %s\n",
        strtonum("0x"substr(l[1],7,2)), strtonum("0x"substr(l[1],5,2)),
        strtonum("0x"substr(l[1],3,2)), strtonum("0x"substr(l[1],1,2)), lp,
        strtonum("0x"substr(r[1],7,2)), strtonum("0x"substr(r[1],5,2)),
        strtonum("0x"substr(r[1],3,2)), strtonum("0x"substr(r[1],1,2)), rp, $4
}' /proc/net/tcp
```

### Count Sockets by State from /proc/net/tcp

```bash
awk 'NR>1 {state[$4]++} END {for(s in state) print s, state[s]}' /proc/net/tcp | sort -k2 -rn
```

Output:

```
01 86    # ESTABLISHED
0A 12    # LISTEN
06 45    # TIME_WAIT
08 5     # CLOSE_WAIT
```

## /proc/net/snmp — Protocol Counters

Cumulative counters since boot for IP, ICMP, TCP, UDP.

```bash
cat /proc/net/snmp | grep Tcp
```

```
Tcp: RtoAlgorithm RtoMin RtoMax MaxConn ActiveOpens PassiveOpens AttemptFails EstabResets CurrEstab InSegs OutSegs RetransSegs InErrs OutRsts InCsumErrors
Tcp: 1 200 120000 -1 45231 12305 234 89 86 9823456 8765432 1234 0 456 0
```

### Key TCP Fields

| Field | Meaning | What to Watch |
|-------|---------|---------------|
| `ActiveOpens` | Outbound connect() calls (cumulative) | Rate = new outbound connections/sec |
| `PassiveOpens` | Inbound accept() calls (cumulative) | Rate = new inbound connections/sec |
| `AttemptFails` | Failed connection attempts | Growing = network/service issues |
| `EstabResets` | Connections reset via RST | Growing = app crashes or rejects |
| `CurrEstab` | Currently established connections | Same as `ss -tn state established \| wc -l` |
| `RetransSegs` | Retransmitted segments (cumulative) | Rate / OutSegs = packet loss % |
| `InErrs` | Segments with errors | Should be near zero |
| `OutRsts` | RST segments sent | High = rejecting connections |

### Calculate Connection Rate and Retransmit Rate

```bash
# Snapshot counters, wait, snapshot again, calculate rate
cat /proc/net/snmp | grep Tcp | awk 'NR==2 {
    printf "Active Opens (outbound): %d\n", $5
    printf "Passive Opens (inbound): %d\n", $6
    printf "Current Established: %d\n", $9
    printf "Retransmits: %d / %d segments = %.4f%%\n", $12, $11, $12/$11*100
    printf "Failed Attempts: %d\n", $7
    printf "Resets: %d\n", $8
}'
```

### Monitor Rate Over Time

```bash
while true; do
    RETRANS=$(awk '/^Tcp:/ && NR>1 {print $12}' /proc/net/snmp)
    OUTSEGS=$(awk '/^Tcp:/ && NR>1 {print $11}' /proc/net/snmp)
    sleep 5
    RETRANS2=$(awk '/^Tcp:/ && NR>1 {print $12}' /proc/net/snmp)
    OUTSEGS2=$(awk '/^Tcp:/ && NR>1 {print $11}' /proc/net/snmp)
    RDIFF=$((RETRANS2 - RETRANS))
    ODIFF=$((OUTSEGS2 - OUTSEGS))
    if [ "$ODIFF" -gt 0 ]; then
        echo "$(date +%H:%M:%S) Retransmits: $RDIFF / $ODIFF segs ($(echo "scale=4; $RDIFF/$ODIFF*100" | bc)%)"
    fi
done
```

### IP Counters

```bash
cat /proc/net/snmp | grep -E "^Ip:"
```

Key fields: `InReceives`, `InDiscards`, `OutRequests`, `OutDiscards`, `ForwDatagrams`

### UDP Counters

```bash
cat /proc/net/snmp | grep -E "^Udp:"
```

Key fields: `InDatagrams`, `OutDatagrams`, `InErrors`, `RcvbufErrors` (buffer overflow), `SndbufErrors`

```bash
# Check for UDP buffer overflows (packets dropped because app too slow)
cat /proc/net/snmp | grep Udp | awk 'NR==2 {printf "UDP RcvbufErrors: %d\nUDP SndbufErrors: %d\n", $5, $6}'
```

## /proc/net/netstat — Extended TCP Stats

More detailed TCP counters not in /proc/net/snmp.

```bash
cat /proc/net/netstat | grep TcpExt
```

### Critical Fields

| Field | Meaning | Why It Matters |
|-------|---------|----------------|
| `ListenOverflows` | Connections dropped (listen queue full) | App can't accept() fast enough |
| `ListenDrops` | Same (older naming) | Service appears down to clients |
| `TCPAbortOnMemory` | Aborted due to memory pressure | OOM risk |
| `TCPAbortOnTimeout` | Aborted due to timeout | Network issues |
| `TCPAbortOnData` | Aborted with data in buffer | App crashed mid-transfer |
| `TCPAbortOnClose` | Aborted during close | Ungraceful shutdown |
| `TCPSynRetrans` | SYN retransmits | Initial handshake failing |
| `TCPTimeouts` | Total TCP timeouts | Widespread connectivity problems |
| `TCPBacklogDrop` | Dropped from backlog queue | Server overloaded |
| `TCPOFOQueue` | Out-of-order segments queued | Network reordering |
| `TCPSACKRecovery` | Recoveries using SACK | Packet loss recovery |
| `TCPLossProbes` | TLP probes sent | Tail loss detection |

### Parse Key Values

```bash
# Extract specific counters
paste <(cat /proc/net/netstat | grep TcpExt | head -1 | tr ' ' '\n') \
      <(cat /proc/net/netstat | grep TcpExt | tail -1 | tr ' ' '\n') | \
      grep -E "ListenOverflows|ListenDrops|TCPAbortOn|TCPSynRetrans|TCPTimeouts|TCPBacklogDrop"
```

### Monitor Listen Queue Drops

```bash
# Non-zero = connections being refused silently
watch -n 5 "grep -oP 'ListenOverflows \K[0-9]+' /proc/net/netstat"
```

## /proc/net/dev — Interface Traffic

Per-interface byte/packet/error counters.

```bash
cat /proc/net/dev
```

```
Inter-|   Receive                                                |  Transmit
 face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
    lo: 123456   1234    0    0    0     0          0         0  123456   1234    0    0    0     0       0          0
  eth0: 98765432 654321  0    2    0     0          0       100  45678901 432100  0    0    0     0       0          0
```

### Key Columns

| Column | Meaning |
|--------|---------|
| `bytes` | Total bytes transferred |
| `packets` | Total packets |
| `errs` | Hardware/driver errors |
| `drop` | Packets dropped (buffer full, no memory) |
| `fifo` | FIFO buffer errors |
| `frame` | Framing errors (RX) |
| `colls` | Collisions (TX, rare on modern networks) |
| `carrier` | Carrier errors (link flaps) |

### Monitor Interface Drops

```bash
# Non-zero drops = packets lost at kernel level
awk 'NR>2 {gsub(":"," "); if($4>0 || $12>0) printf "  %s RX_drop:%d TX_drop:%d\n", $1, $4, $12}' /proc/net/dev
```

### Calculate Bandwidth per Interface

```bash
# Bytes/sec over 1 second interval
IFACE="eth0"
RX1=$(awk -v iface="$IFACE:" '$1==iface {print $2}' /proc/net/dev)
TX1=$(awk -v iface="$IFACE:" '$1==iface {print $10}' /proc/net/dev)
sleep 1
RX2=$(awk -v iface="$IFACE:" '$1==iface {print $2}' /proc/net/dev)
TX2=$(awk -v iface="$IFACE:" '$1==iface {print $10}' /proc/net/dev)
echo "RX: $(( (RX2-RX1) / 1024 )) KB/s  TX: $(( (TX2-TX1) / 1024 )) KB/s"
```

## /proc/net/arp — ARP Table

```bash
cat /proc/net/arp
```

```
IP address       HW type     Flags       HW address            Mask     Device
192.168.1.1      0x1         0x2         aa:bb:cc:dd:ee:ff     *        eth0
10.0.0.50        0x1         0x2         11:22:33:44:55:66     *        eth0
```

| Flag | Meaning |
|------|---------|
| `0x2` | Complete (resolved) |
| `0x0` | Incomplete (pending resolution) |
| `0x6` | Permanent (manually set) |

```bash
# Count ARP entries per interface
awk 'NR>1 {count[$NF]++} END {for(d in count) print d, count[d]}' /proc/net/arp
```

## /proc/net/route — Routing Table

```bash
cat /proc/net/route
```

```
Iface   Destination Gateway     Flags RefCnt Use Metric Mask        MTU Window IRTT
eth0    00000000    0101A8C0    0003  0      0   100    00000000    0   0      0
eth0    0000A8C0    00000000    0001  0      0   100    00FFFFC0    0   0      0
```

IPs are in hex (little-endian). Use `ip route` for human-readable output, but this is what the kernel stores.

```bash
# Decode default gateway
awk '/00000000/ && $2=="00000000" {
    gw=$3;
    printf "Default GW: %d.%d.%d.%d via %s\n",
        strtonum("0x"substr(gw,7,2)), strtonum("0x"substr(gw,5,2)),
        strtonum("0x"substr(gw,3,2)), strtonum("0x"substr(gw,1,2)), $1
}' /proc/net/route
```

## /proc/net/nf_conntrack — Connection Tracking (Firewall)

If using iptables/nftables with stateful rules, this shows every tracked connection.

```bash
cat /proc/net/nf_conntrack | head -5
```

```
ipv4  2 tcp  6 117 TIME_WAIT src=192.168.1.10 dst=93.184.216.34 sport=54102 dport=443 src=93.184.216.34 dst=192.168.1.10 sport=443 dport=54102 [ASSURED] mark=0 use=2
ipv4  2 tcp  6 431999 ESTABLISHED src=192.168.1.10 dst=10.0.0.50 sport=54108 dport=5432 src=10.0.0.50 dst=192.168.1.10 sport=5432 dport=54108 [ASSURED] mark=0 use=2
```

### Conntrack Stats

```bash
# Total tracked connections
wc -l /proc/net/nf_conntrack

# Max connections (if full, new connections are dropped!)
sysctl net.netfilter.nf_conntrack_max

# Current vs max
echo "$(wc -l < /proc/net/nf_conntrack) / $(sysctl -n net.netfilter.nf_conntrack_max)"

# By state
awk '{print $4}' /proc/net/nf_conntrack | sort | uniq -c | sort -rn

# By protocol
awk '{print $3}' /proc/net/nf_conntrack | sort | uniq -c | sort -rn
```

### Conntrack Table Full = Dropped Connections

```bash
# Check for drops
cat /proc/net/stat/nf_conntrack | head -2
# Column 3 = "drop" (connections dropped because table full)

# Or via dmesg
dmesg | grep "nf_conntrack: table full"

# Fix: increase limit
sysctl -w net.netfilter.nf_conntrack_max=262144
```

## /proc/net/unix — UNIX Domain Sockets

```bash
cat /proc/net/unix | head -10
```

Shows all local (inter-process) sockets — used by systemd, docker, databases, etc.

```bash
# Count UNIX sockets by type
awk 'NR>1 {types[$5]++} END {for(t in types) print t, types[t]}' /proc/net/unix
# Type 1=STREAM, 2=DGRAM, 5=SEQPACKET
```

## /proc/sys/net — Kernel Tuning Parameters

Not in `/proc/net/` but directly related — these control the behavior of everything above.

```bash
# TCP tuning
sysctl net.ipv4.tcp_max_syn_backlog       # SYN queue size
sysctl net.core.somaxconn                  # Listen backlog max
sysctl net.ipv4.tcp_max_tw_buckets        # Max TIME_WAIT sockets
sysctl net.ipv4.tcp_fin_timeout           # Orphaned FIN_WAIT2 timeout
sysctl net.ipv4.tcp_tw_reuse              # Reuse TIME_WAIT sockets
sysctl net.ipv4.ip_local_port_range       # Ephemeral port range
sysctl net.ipv4.tcp_max_orphans           # Max orphaned sockets
sysctl net.ipv4.tcp_mem                   # TCP memory limits [low pressure high]
sysctl net.ipv4.tcp_rmem                  # Per-socket read buffer [min default max]
sysctl net.ipv4.tcp_wmem                  # Per-socket write buffer [min default max]
sysctl net.core.rmem_max                  # Max receive buffer
sysctl net.core.wmem_max                  # Max send buffer
sysctl net.core.netdev_max_backlog        # NIC input queue before kernel processes

# Conntrack
sysctl net.netfilter.nf_conntrack_max     # Max tracked connections
sysctl net.netfilter.nf_conntrack_tcp_timeout_established   # Timeout for ESTABLISHED
sysctl net.netfilter.nf_conntrack_tcp_timeout_time_wait     # Timeout for TIME_WAIT

# ARP
sysctl net.ipv4.neigh.default.gc_thresh1  # Min ARP entries before GC
sysctl net.ipv4.neigh.default.gc_thresh2  # Soft limit
sysctl net.ipv4.neigh.default.gc_thresh3  # Hard limit
```

## Complete Health Check Script

```bash
#!/bin/bash
# /proc/net health check — no tools required

echo "=== Socket Summary ==="
cat /proc/net/sockstat
echo ""

echo "=== TCP State Breakdown ==="
awk 'NR>1 {state[$4]++} END {
    split("01:ESTAB 02:SYN_SENT 03:SYN_RECV 04:FIN_WAIT1 05:FIN_WAIT2 06:TIME_WAIT 07:CLOSE 08:CLOSE_WAIT 09:LAST_ACK 0A:LISTEN 0B:CLOSING", names, " ")
    for(n in names) {split(names[n],kv,":"); map[kv[1]]=kv[2]}
    for(s in state) printf "  %-12s %d\n", map[s], state[s]
}' /proc/net/tcp | sort -k2 -rn
echo ""

echo "=== TCP Retransmit Rate ==="
cat /proc/net/snmp | grep Tcp | awk 'NR==2 {if($11>0) printf "  %.4f%% (%d retrans / %d segs)\n", $12/$11*100, $12, $11}'
echo ""

echo "=== Listen Queue Drops ==="
paste <(grep TcpExt /proc/net/netstat | head -1 | tr ' ' '\n') \
      <(grep TcpExt /proc/net/netstat | tail -1 | tr ' ' '\n') | grep ListenOverflows
echo ""

echo "=== Interface Drops ==="
awk 'NR>2 {gsub(":"," "); if($4>0||$12>0) printf "  %s RX_drop:%d TX_drop:%d\n",$1,$4,$12}' /proc/net/dev
echo ""

echo "=== Conntrack Usage ==="
if [ -f /proc/net/nf_conntrack ]; then
    CURR=$(wc -l < /proc/net/nf_conntrack)
    MAX=$(sysctl -n net.netfilter.nf_conntrack_max 2>/dev/null || echo "N/A")
    echo "  $CURR / $MAX"
else
    echo "  Conntrack not loaded"
fi
```
