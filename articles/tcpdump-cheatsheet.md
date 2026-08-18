# tcpdump Cheatsheet

`tcpdump` is a command-line packet analyzer that captures and displays network traffic. It uses the libpcap library and can write captures to pcap files for analysis in tools like Wireshark.

## Basic Syntax

```bash
tcpdump [options] [expression]
```

Older versions of tcpdump truncate packets to 68 or 96 bytes. Use `-s 0` to capture full packets.

## Output Format

```
time-stamp src > dst:  flags  data-seqno  ack  window  urgent  options
```

## Install

```bash
# RHEL / CentOS / Fedora
sudo dnf install tcpdump

# Ubuntu / Debian
sudo apt install tcpdump

# Verify
tcpdump --version
```

## Basic Capture

```bash
# Capture on default interface (first non-loopback)
sudo tcpdump

# Capture on a specific interface
sudo tcpdump -i eth0

# Capture on all interfaces
sudo tcpdump -i any

# Capture with verbose output
sudo tcpdump -v
sudo tcpdump -vv     # more verbose
sudo tcpdump -vvv    # maximum verbosity

# Capture N packets and stop
sudo tcpdump -c 100

# Capture without DNS resolution (faster output)
sudo tcpdump -n

# Capture without DNS and port name resolution
sudo tcpdump -nn

# Show available interfaces
sudo tcpdump -D
```

## Output Options

```bash
# Print packet content in hex and ASCII
sudo tcpdump -X

# Print packet content in hex only
sudo tcpdump -x

# Print packet content in ASCII only
sudo tcpdump -A

# Show absolute TCP sequence numbers (not relative)
sudo tcpdump -S

# Don't print timestamps
sudo tcpdump -t

# Print unformatted (Unix epoch) timestamps
sudo tcpdump -tt

# Print time differences between packets
sudo tcpdump -ttt

# Print timestamp in readable format
sudo tcpdump -tttt

# Print delta between packets (microsecond precision)
sudo tcpdump -ttttt

# Quiet output (less protocol info)
sudo tcpdump -q

# Show Ethernet header (MAC addresses)
sudo tcpdump -e

# Don't resolve domain names (show FQDN as short name)
sudo tcpdump -N

# Don't put interface in promiscuous mode
sudo tcpdump -p

# Increase snap length (capture full packets)
sudo tcpdump -s 0          # capture entire packet
sudo tcpdump -s 1500       # capture first 1500 bytes (default: 262144)
```

## Write and Read Capture Files

```bash
# Write capture to a pcap file
sudo tcpdump -i eth0 -w /tmp/capture.pcap

# Write with packet count limit
sudo tcpdump -i eth0 -c 1000 -w /tmp/capture.pcap

# Write with file rotation (100 MB per file, keep 10 files)
sudo tcpdump -i eth0 -w /tmp/capture-%Y%m%d-%H%M%S.pcap -C 100 -W 10

# Write with time-based rotation (new file every 3600 seconds)
sudo tcpdump -i eth0 -w /tmp/capture.pcap -G 3600

# Read a pcap file
tcpdump -r /tmp/capture.pcap

# Read with filters applied
tcpdump -r /tmp/capture.pcap -nn port 443

# Read and show packet contents
tcpdump -r /tmp/capture.pcap -X

# Read with verbose output
tcpdump -r /tmp/capture.pcap -vvv

# Pipe between tcpdump instances
sudo tcpdump -i eth0 -w - | tcpdump -r -
```

## Filter by Host

```bash
# Capture traffic to/from a specific host
sudo tcpdump -i eth0 host 192.168.1.100

# Capture traffic FROM a specific host
sudo tcpdump -i eth0 src host 192.168.1.100

# Capture traffic TO a specific host
sudo tcpdump -i eth0 dst host 192.168.1.100

# Capture traffic between two hosts
sudo tcpdump -i eth0 host 192.168.1.100 and host 192.168.1.200

# Filter by hostname (DNS lookup)
sudo tcpdump -i eth0 host server01.example.com

# Capture traffic to/from a subnet
sudo tcpdump -i eth0 net 192.168.1.0/24

# Capture traffic FROM a subnet
sudo tcpdump -i eth0 src net 10.0.0.0/8

# Exclude a specific host
sudo tcpdump -i eth0 not host 192.168.1.1
```

## Filter by Port

```bash
# Capture traffic on a specific port
sudo tcpdump -i eth0 port 80

# Capture traffic on source port
sudo tcpdump -i eth0 src port 443

# Capture traffic on destination port
sudo tcpdump -i eth0 dst port 22

# Capture a range of ports
sudo tcpdump -i eth0 portrange 8000-9000

# Capture multiple ports
sudo tcpdump -i eth0 port 80 or port 443

# Capture everything except SSH (useful when capturing over SSH)
sudo tcpdump -i eth0 not port 22

# Capture HTTP and HTTPS
sudo tcpdump -i eth0 port 80 or port 443
```

## Filter by Protocol

```bash
# Capture only TCP traffic
sudo tcpdump -i eth0 tcp

# Capture only UDP traffic
sudo tcpdump -i eth0 udp

# Capture only ICMP (ping)
sudo tcpdump -i eth0 icmp

# Capture only ARP
sudo tcpdump -i eth0 arp

# Capture only IPv6 traffic
sudo tcpdump -i eth0 ip6

# Capture only VLAN tagged traffic
sudo tcpdump -i eth0 vlan

# Capture only GRE encapsulated traffic
sudo tcpdump -i eth0 proto gre
```

## TCP Flags Filters

```bash
# Capture TCP SYN packets (connection initiation)
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'

# Capture only SYN packets (no SYN-ACK)
sudo tcpdump -i eth0 'tcp[tcpflags] == tcp-syn'

# Capture SYN-ACK packets
sudo tcpdump -i eth0 'tcp[tcpflags] == (tcp-syn|tcp-ack)'

# Capture TCP FIN packets (connection termination)
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-fin != 0'

# Capture TCP RST packets (connection reset)
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0'

# Capture TCP PUSH packets
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-push != 0'

# Capture TCP URG (urgent) packets
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-urg != 0'

# Alternative byte offset notation:
# Display all ACK packets (tcp[13] is the flags byte)
sudo tcpdump -i eth0 'tcp[13] & 16 != 0'

# Display all SYN packets via byte offset
sudo tcpdump -i eth0 'tcp[13] & 2 != 0'

# Display SYN-ACK packets (SYN=2 + ACK=16 = 18)
sudo tcpdump -i eth0 'tcp[13] = 18'

# SYN or ACK flags on a specific port and host
sudo tcpdump -i eth0 -nn -p -e 'host 10.0.0.5 and port 28000 and tcp[tcpflags] & (tcp-syn|tcp-ack) != 0'
```

## Combined Filters

```bash
# HTTP GET requests from a specific host
sudo tcpdump -i eth0 -A src host 192.168.1.100 and dst port 80

# DNS queries (UDP port 53)
sudo tcpdump -i eth0 -nn udp port 53

# DNS queries to a specific server
sudo tcpdump -i eth0 -nn dst host 8.8.8.8 and udp port 53

# HTTPS traffic from a subnet
sudo tcpdump -i eth0 -nn src net 10.0.0.0/8 and dst port 443

# All traffic except SSH and DNS
sudo tcpdump -i eth0 not port 22 and not port 53

# Filter out ARP and SSH
sudo tcpdump -i eth0 not arp and not port 22

# Traffic between two hosts on a specific port
sudo tcpdump -i eth0 host 192.168.1.10 and host 192.168.1.20 and port 3306

# Non-ICMP traffic from a host
sudo tcpdump -i eth0 src host 192.168.1.100 and not icmp

# Traffic from host AND destined for port 80 or 443 (escaped parentheses)
sudo tcpdump -i eth0 'src 192.168.1.30 and \(dst port 80 or dst port 443\)'

# TCP traffic from a host destined for a specific port
sudo tcpdump -i eth0 tcp and src 192.168.1.30 and dst port 80

# Capture only large packets (potential data transfer)
sudo tcpdump -i eth0 'greater 1000'

# Capture only small packets (potential control traffic)
sudo tcpdump -i eth0 'less 100'

# Capture everything with maximum verbosity
sudo tcpdump -i eth0 -nnvvXSs 1514
```

## HTTP and Application Layer

```bash
# Capture HTTP requests (show ASCII content)
sudo tcpdump -i eth0 -A -s 0 port 80

# Capture HTTP GET requests
sudo tcpdump -i eth0 -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

# Capture HTTP GET requests (byte offset match for "GET ")
sudo tcpdump -i eth0 -A -s 0 'tcp dst port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'

# Capture HTTP POST requests (byte offset match for "POST")
sudo tcpdump -i eth0 -A -s 0 'tcp dst port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354'

# Capture HTTP Host headers
sudo tcpdump -i eth0 -A -s 0 port 80 | grep -i "Host:"

# Capture HTTP User-Agent
sudo tcpdump -i eth0 -A -s 0 port 80 | grep -i "User-Agent:"

# Find HTTP User-Agent (verbose)
sudo tcpdump -i eth0 -vvAls0 | grep 'User-Agent:'

# Find cleartext passwords in HTTP, FTP, SMTP, IMAP, POP3, Telnet
sudo tcpdump -i eth0 port http or port ftp or port smtp or port imap or port pop3 or port telnet -lA | \
  egrep -i -B5 'pass=|pwd=|log=|login=|user=|username=|pw=|passw=|passwd=|password=|pass:|user:|username:|password:|login:|pass |user '

# Capture SMTP traffic
sudo tcpdump -i eth0 -nn port 25

# Capture POP3 traffic
sudo tcpdump -i eth0 -nn port 110

# Capture IMAP traffic
sudo tcpdump -i eth0 -nn port 143

# Capture secure email (IMAPS / POP3S)
sudo tcpdump -i eth0 -nn port 993 or port 995

# Capture NFS traffic
sudo tcpdump -i eth0 -nn port 2049

# Capture NTP traffic
sudo tcpdump -i eth0 -nn udp port 123

# Capture DHCP traffic
sudo tcpdump -i eth0 -nn udp port 67 or udp port 68

# Capture LDAP traffic
sudo tcpdump -i eth0 -nn port 389 or port 636

# Capture MySQL/MariaDB traffic
sudo tcpdump -i eth0 -nn port 3306

# Capture Redis traffic
sudo tcpdump -i eth0 -nn port 6379

# Capture syslog traffic
sudo tcpdump -i eth0 -nn udp port 514
```

## Advanced Byte Offset Filters

```bash
# IP packets with TTL less than 10 (detect routing loops)
sudo tcpdump -i eth0 'ip[8] < 10'

# IP packets with Don't Fragment flag set
sudo tcpdump -i eth0 'ip[6] & 0x40 != 0'

# IP packets with More Fragments flag (fragmented traffic)
sudo tcpdump -i eth0 'ip[6] & 0x20 != 0'

# All fragmented packets (fragment offset != 0 or MF flag set)
sudo tcpdump -i eth0 'ip[6:2] & 0x3fff != 0'

# TCP packets with specific window size of 0 (zero window)
sudo tcpdump -i eth0 'tcp[14:2] = 0'

# ICMP echo requests only (type 8)
sudo tcpdump -i eth0 'icmp[icmptype] == icmp-echo'

# ICMP echo replies only (type 0)
sudo tcpdump -i eth0 'icmp[icmptype] == icmp-echoreply'

# ICMP unreachable
sudo tcpdump -i eth0 'icmp[icmptype] == icmp-unreach'
```

## Troubleshooting Recipes

```bash
# Diagnose slow DNS — capture all DNS traffic with timestamps
sudo tcpdump -i eth0 -nn -tttt udp port 53

# Find who is ARP flooding
sudo tcpdump -i eth0 -nn arp | awk '{print $NF}' | sort | uniq -c | sort -rn | head

# Detect SYN flood (many SYN without ACK)
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] == tcp-syn' | awk '{print $5}' | sort | uniq -c | sort -rn | head

# Monitor for port scans (SYN packets without ACK — connection attempts)
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack = 0'

# Monitor TCP retransmissions (partial — look for duplicate SEQs)
sudo tcpdump -i eth0 -nn tcp | grep -i retransmit

# Monitor large data transfers (full MSS segments)
sudo tcpdump -i eth0 'tcp and greater 1460'

# Capture traffic for a specific duration (30 seconds)
timeout 30 sudo tcpdump -i eth0 -nn -w /tmp/capture-30s.pcap

# Watch for ICMP unreachable (network issues)
sudo tcpdump -i eth0 -nn 'icmp[icmptype] == 3'

# Monitor connection resets to a service
sudo tcpdump -i eth0 -nn dst port 80 and 'tcp[tcpflags] & tcp-rst != 0'

# Check for TCP connections not completing handshake
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] == tcp-syn' -c 100 -w /tmp/syns.pcap

# Capture only new connections (SYN only, not SYN-ACK)
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] == tcp-syn'

# Watch multicast/broadcast traffic
sudo tcpdump -i eth0 -nn broadcast or multicast

# Monitor for failed connections (RST packets)
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-rst != 0'

# Monitor ICMP traffic (network issues or reconnaissance)
sudo tcpdump -i eth0 -v icmp

# Capture large packets that might indicate problems (> MTU)
sudo tcpdump -i any greater 1500
```

## Performance and Large Captures

```bash
# Capture with kernel-level buffer size (reduce drops on busy interfaces)
sudo tcpdump -i eth0 -B 4096 -w /tmp/capture.pcap

# Check for dropped packets (shown at end of capture)
# "N packets dropped by kernel" in summary

# Capture in background
sudo tcpdump -i eth0 -w /tmp/capture.pcap &
TCPDUMP_PID=$!
# ... do your test ...
kill $TCPDUMP_PID

# Capture with ring buffer (5 files of 100 MB each)
sudo tcpdump -i eth0 -w /tmp/capture.pcap -C 100 -W 5

# Minimal overhead capture (no resolution, binary only)
sudo tcpdump -i eth0 -nn -s 96 -w /tmp/capture.pcap

# Count packets per second (rough estimate)
sudo tcpdump -i eth0 -nn -q 2>/dev/null | pv -l > /dev/null

# Capture with dynamic filename (hostname + timestamp)
sudo tcpdump -i eth0 -s 0 -w /tmp/$(hostname)-$(date +"%Y-%m-%d-%H-%M-%S").pcap

# Capture to specific host and port with dynamic filename
sudo tcpdump -i eth0 -s 0 host 10.0.0.5 and dst port 28000 \
  -w /tmp/$(hostname)-$(date +"%Y-%m-%d-%H-%M-%S").pcap

# Run tcpdump in background with nohup (survives logout)
nohup tcpdump -i bond0 -B 65536 -C 200 -W 10 -s 0 tcp port 1521 \
  -w /tmp/capture.pcap > /tmp/tcpdump.log 2>&1 &

# Stop a background tcpdump gracefully (SIGINT flushes buffers)
pkill -2 tcpdump

# Oracle DB traffic capture across multiple hosts
DATE=$(date +%Y%m%d.%H%M)
sudo tcpdump -i bond0 -w /tmp/tcpdump.$(uname -n).$DATE.pcap \
  'tcp port 1521 and (host 10.80.20.207 or host 10.80.20.206 or host 10.80.20.208)' &
```

## Filter Syntax Quick Reference

| Operator | Description | Example |
|----------|-------------|---------|
| `host` | Match source or destination | `host 10.0.0.1` |
| `src` / `dst` | Direction qualifier | `src host 10.0.0.1` |
| `net` | Match a subnet | `net 192.168.0.0/16` |
| `port` | Match source or destination port | `port 443` |
| `portrange` | Match a port range | `portrange 8000-9000` |
| `tcp` / `udp` / `icmp` | Protocol filter | `tcp` |
| `and` / `&&` | Logical AND | `host 10.0.0.1 and port 80` |
| `or` / `\|\|` | Logical OR | `port 80 or port 443` |
| `not` / `!` | Logical NOT | `not port 22` |
| `greater` | Packet size greater than | `greater 1000` |
| `less` | Packet size less than | `less 64` |
| `(expr)` | Grouping | `(port 80 or port 443) and host 10.0.0.1` |

## Useful One-Liners

```bash
# Top talkers by source IP
sudo tcpdump -i eth0 -nn -c 10000 -q 2>/dev/null | awk '{print $3}' | cut -d. -f1-4 | sort | uniq -c | sort -rn | head -20

# Top 10 hosts by packets (alternative)
sudo tcpdump -i eth0 -nn -t -c 1000 | cut -f 1,2,3,4 -d '.' | sort | uniq -c | sort -rn | head -n 10

# Top destination ports
sudo tcpdump -i eth0 -nn -c 10000 -q 2>/dev/null | awk '{print $5}' | cut -d. -f5 | sort | uniq -c | sort -rn | head -20

# Capture and immediately analyze with tshark
sudo tcpdump -i eth0 -w - | tshark -r -

# Quick bandwidth estimate (bytes per second)
sudo tcpdump -i eth0 -nn -q -w /tmp/bw.pcap &
sleep 10 && kill %1
ls -la /tmp/bw.pcap    # size / 10 = bytes/sec

# Extract unique IPs from a capture file
tcpdump -r /tmp/capture.pcap -nn | awk '{print $3; print $5}' | cut -d. -f1-4 | sort -u

# Watch for specific HTTP response codes (in plain HTTP)
sudo tcpdump -i eth0 -A -s 0 port 80 | grep "HTTP/1.1"

# Capture packets and show only the summary line count
sudo tcpdump -i eth0 -nn -c 1000 -q 2>&1 | tail -3

# Monitor VLAN traffic and show VLAN IDs
sudo tcpdump -i eth0 -nn -e vlan

# Quick packet hex dump of first 3 packets
sudo tcpdump -i eth0 -XX -c 3

# Comprehensive web traffic analysis
sudo tcpdump -i any -A -s 0 'tcp port 80 or tcp port 443' -w /tmp/web_traffic.pcap

# Quick network diagnostic
sudo tcpdump -i any -nn -c 100 'icmp or arp'

# Security audit capture (everything except SSH)
sudo tcpdump -i any -X -s 0 'not port 22' -w /tmp/security_audit.pcap

# Capture TCP between two specific hosts
sudo tcpdump -i eth0 -w /tmp/two-host-tcp.pcap tcp and \(host 169.144.0.1 or host 169.144.0.20\)

# Capture SSH traffic between two hosts (specific ports)
sudo tcpdump -i eth0 -w /tmp/ssh-two-hosts.pcap src 169.144.0.1 and port 22 and dst 169.144.0.20

# Capture UDP between two hosts with snap length limit
sudo tcpdump -i eth0 -w /tmp/two-host-udp.pcap -s 1000 udp and \(host 169.144.0.10 and host 169.144.0.20\)
```

## Tips and Best Practices

- Always quote complex filter expressions to avoid shell interpretation: `'port 80 or port 443'`
- Use `sudo` or run as root — most interfaces require elevated privileges
- Use `-n` and `-nn` to avoid DNS lookups that slow output and generate additional traffic
- Limit packet size with `-s` when you only need headers (reduces CPU and disk usage)
- Be aware of capture file sizes in long-running captures — use `-C` and `-W` for rotation
- When capturing over SSH, always exclude your SSH port: `not port 22`
- Use `-w` to write to file for large captures — real-time display adds overhead
- Prefer specific filters over capturing everything to reduce CPU and storage impact

## Related Tools

| Tool | Description |
|------|-------------|
| Wireshark | GUI packet analyzer — reads tcpdump pcap files directly |
| tshark | Command-line Wireshark — richer protocol dissection than tcpdump |
| ngrep | Network grep — search for regex patterns in packet payloads |
| tcpflow | Reconstructs TCP sessions from packet captures into separate files |
| tcpreplay | Replay pcap files back onto the network for testing |
| capinfos | Display summary information about pcap files |
| editcap | Edit and transform pcap files (split, filter, truncate) |
