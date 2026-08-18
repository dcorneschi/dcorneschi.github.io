# Diagnosing Packet Loss with mtr (and friends)

Packet loss is one of the most impactful and hardest-to-spot network problems. This guide covers detecting it (passive) and locating *where* it happens (active, hop-by-hop with `mtr`).

## Why Packet Loss Matters

- **TCP**: lost packets trigger retransmissions — latency spikes (hundreds of ms) and big throughput drops. Even 1-2% loss can cripple TCP because congestion control backs off hard.
- **UDP**: no retransmission — packets are gone. Causes choppy VoIP, dropped DNS, missing metrics/logs.
- **Cascading effects**: timeouts pile up, connection pools exhaust, health checks flap, app looks "slow" with no obvious cause.

## Passive vs Active Detection

| Approach | What it shows | Cost |
|----------|---------------|------|
| **Passive** | Loss the kernel already recorded (drops, retransmits) | Free, always on |
| **Active** | Loss to a target *right now*, hop by hop | Generates probe traffic |

### Passive (already recorded by the kernel)

```bash
# Interface RX/TX drops
awk 'NR>2 {gsub(":"," "); if($4>0||$12>0) print $1, "RX_drop:"$4, "TX_drop:"$12}' /proc/net/dev

# TCP retransmit rate (retransmits = packets lost in transit)
awk '/^Tcp:/{c++; if(c==2) printf "Retransmit rate: %.4f%%\n", $12/$11*100}' /proc/net/snmp

# UDP receive buffer errors (dropped because app too slow)
awk '/^Udp:/{c++; if(c==2){for(i=1;i<=NF;i++) if(h[i]=="RcvbufErrors") print "UDP RcvbufErrors:", $i}} /^Udp:/{if(c==1) for(i=1;i<=NF;i++) h[i]=$i}' /proc/net/snmp
```

## mtr — Hop-by-Hop Loss (the key tool)

`mtr` combines `ping` and `traceroute`. It continuously probes every hop on the path and shows loss % per hop — so you can tell *where* the loss happens (your box, LAN, gateway, or upstream).

### Install

```bash
# RHEL / CentOS / Rocky
yum install -y mtr

# Ubuntu / Debian
apt install -y mtr-tiny      # or 'mtr' for the GUI version
```

### Basic Usage

```bash
mtr 8.8.8.8                  # Interactive, live-updating
mtr -r -c 100 8.8.8.8        # Report mode: 100 packets then summary
mtr -rw -c 100 8.8.8.8       # Wide report (full hostnames)
mtr -n 8.8.8.8               # Don't resolve DNS (faster)
mtr -b 8.8.8.8               # Show both IP and hostname
```

### Useful Options

| Option | Description |
|--------|-------------|
| `-r` | Report mode (run N cycles, print summary, exit) |
| `-c <n>` | Number of pings per hop |
| `-w` | Wide report (don't truncate hostnames) |
| `-n` | No DNS resolution |
| `-b` | Show both hostname and IP |
| `-i <sec>` | Interval between packets |
| `-s <bytes>` | Packet size |
| `-u` | Use UDP instead of ICMP |
| `-T` | Use TCP instead of ICMP |
| `-P <port>` | Port for TCP/UDP probes |
| `-o "LSDR NBAW JMXI"` | Custom field order |

### Reading the Output

```bash
mtr -rwn -c 100 8.8.8.8
```

```
HOST: myserver              Loss%  Snt  Last  Avg  Best  Wrst StDev
  1. 192.168.1.1             0.0%  100   0.3   0.4   0.3   1.2   0.1
  2. 10.50.0.1               0.0%  100   1.2   1.4   1.1   3.5   0.3
  3. 72.14.x.x               2.0%  100  12.3  13.1  11.8  45.2   4.1
  4. 8.8.8.8                 0.0%  100  12.5  12.9  12.1  18.3   1.0
```

| Column | Meaning |
|--------|---------|
| `Loss%` | Packet loss at that hop |
| `Snt` | Packets sent |
| `Last` | Most recent RTT (ms) |
| `Avg` | Average RTT |
| `Best/Wrst` | Min/max RTT |
| `StDev` | RTT jitter (high = unstable) |

### Interpreting Loss — The Critical Skill

**Loss on a middle hop that DOESN'T continue to the destination = ignore it.**
Routers often deprioritize or rate-limit ICMP to their *own* address (the TTL-expired replies mtr relies on), so they report fake loss while still forwarding traffic fine. This is the #1 mtr misreading.

```
  3. router-A      30.0%   ← looks alarming
  4. router-B       0.0%   ← but loss didn't propagate = hop 3 is fine
  5. dest           0.0%
```

**Loss that STARTS at a hop and CONTINUES to the destination = real problem.**

```
  3. router-A       0.0%
  4. router-B       5.0%   ← loss starts here
  5. dest           5.0%   ← and continues = real loss between hop 3 and 4
```

**Loss at the final hop only** = the destination itself is dropping (overloaded, rate-limiting, or firewall).

### TCP/UDP Probes (when ICMP is blocked or you want realism)

Many firewalls drop ICMP but allow TCP/UDP. Probe the actual service port:

```bash
mtr -T -P 443 example.com    # TCP probes to port 443 (like real HTTPS)
mtr -u -P 53 8.8.8.8         # UDP probes to port 53 (like real DNS)
```

This also tests loss for the *actual* traffic type your app uses, not just ICMP.

## Quick Diagnostic Workflow

```bash
# 1. Is there loss at all? (passive, instant)
awk '/^Tcp:/{c++; if(c==2) printf "Retransmit rate: %.4f%%\n", $12/$11*100}' /proc/net/snmp

# 2. Is it local (NIC/driver)? Check interface drops
awk 'NR>2 {gsub(":"," "); if($4>0||$12>0) print $1, "drops RX:"$4" TX:"$12}' /proc/net/dev

# 3. Where on the path? (active, hop-by-hop)
mtr -rwn -c 100 <destination>

# 4. To the gateway specifically (isolate LAN vs upstream)
mtr -rwn -c 100 $(ip route | awk '/default/{print $3; exit}')
```

## Where the Loss Is — What to Check

| Loss location | Likely cause | What to check |
|---------------|--------------|---------------|
| Local interface drops | NIC ring buffer, driver, CPU | `ethtool -S`, `ethtool -g`, IRQ balance |
| First hop (gateway) | LAN: cable, switch port, duplex | Cable, switch logs, `ethtool eth0` |
| Middle hops (continuing) | ISP / transit congestion | Contact ISP with mtr report |
| Final hop only | Destination overloaded/rate-limiting | Server load, firewall rules |
| All hops equally | Local NIC or first link saturated | Bandwidth usage, `bmon` |

## Other Active Tools

```bash
# ping — simple loss to one target
ping -c 100 -i 0.2 8.8.8.8 | grep "packet loss"

# fping — ping many hosts at once
fping -c 100 -q host1 host2 host3

# hping3 — custom TCP/UDP probes (loss to a specific port)
hping3 -c 100 -S -p 443 example.com

# iperf3 — measures loss for UDP throughput tests
iperf3 -c <server> -u -b 100M    # reports loss % at the end
```
