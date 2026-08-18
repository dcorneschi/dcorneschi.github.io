# netstat Cheatsheet

Network statistics tool for viewing network connections, routing tables, interface statistics, and more. `ss` is the modern replacement and is faster, but `netstat` is still widely used and available on legacy systems.

## Install

```bash
# Debian / Ubuntu
sudo apt install net-tools

# RHEL / CentOS / Fedora
sudo dnf install net-tools    # RHEL 8+
sudo yum install net-tools    # RHEL 7
```

## Basic Usage

```bash
netstat                 # Show active connections
netstat -a              # Show all connections and listening ports
netstat -t              # Show TCP connections
netstat -u              # Show UDP connections
netstat -l              # Show listening ports
netstat -n              # Show numerical addresses (no DNS resolution)
netstat -p              # Show program/PID (requires root)
```

## Common Combinations

```bash
netstat -tuln           # TCP & UDP listening ports (numeric)
netstat -tulpn          # TCP & UDP listening ports with programs
netstat -tan            # All TCP connections (numeric)
netstat -uan            # All UDP connections (numeric)
netstat -anp            # All connections with programs (requires root)
netstat -tupn           # TCP & UDP with programs (numeric)
```

## TCP Connections

```bash
netstat -t              # TCP connections
netstat -ta             # All TCP (listening + established)
netstat -tan            # All TCP (numeric)
netstat -tp             # TCP with process info
netstat -tanp           # All TCP, numeric, with process
```

## UDP Connections

```bash
netstat -u              # UDP connections
netstat -ua             # All UDP
netstat -uan            # All UDP (numeric)
netstat -up             # UDP with process info
netstat -uanp           # All UDP, numeric, with process
```

## Listening Ports

```bash
netstat -l              # All listening ports
netstat -lt             # Listening TCP ports
netstat -lu             # Listening UDP ports
netstat -lx             # Listening Unix sockets
netstat -lnp            # Listening ports with programs
netstat -tulnp          # TCP & UDP listening with programs
```

## Numeric Display

```bash
netstat -n              # Don't resolve hostnames
netstat --numeric-hosts # Don't resolve hostnames
netstat --numeric-ports # Don't resolve port names
netstat --numeric-users # Don't resolve usernames
```

## Extended Information

```bash
netstat -e              # Extended information
netstat -o              # Display timers
netstat -c              # Continuous output
netstat -c 2            # Update every 2 seconds
```

## Protocol Specific

```bash
netstat -t              # TCP only
netstat -u              # UDP only
netstat -x              # Unix domain sockets
netstat -w              # Raw sockets
netstat --tcp           # TCP (long form)
netstat --udp           # UDP (long form)
```

## Routing Table

```bash
netstat -r              # Routing table
netstat -rn             # Routing table (numeric)
netstat -rne            # Routing table with extended info
```

## Interface Statistics

```bash
netstat -i              # Interface statistics
netstat -ie             # Interface statistics (extended)
netstat -s              # Protocol statistics
netstat -st             # TCP statistics
netstat -su             # UDP statistics
```

## Multicast Information

```bash
netstat -g              # Multicast group membership
netstat -gn             # Multicast (numeric)
```

## Connection States

| State | Description |
|-------|-------------|
| `LISTEN` | Listening for connections |
| `ESTABLISHED` | Connection established |
| `SYN_SENT` | Sent SYN, waiting for response |
| `SYN_RECEIVED` | Received SYN, sent SYN+ACK |
| `FIN_WAIT1` | Sent FIN, waiting for ACK |
| `FIN_WAIT2` | Got FIN ACK, waiting for FIN |
| `TIME_WAIT` | Waiting after close (2x MSL) |
| `CLOSE` | Connection closed |
| `CLOSE_WAIT` | Remote closed, waiting for local close |
| `LAST_ACK` | Waiting for final ACK |
| `CLOSING` | Both sides closing simultaneously |

## Output Format

```
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1234/sshd
tcp        0      0 192.168.1.10:22         192.168.1.20:54321      ESTABLISHED 5678/sshd
```

| Column | Meaning |
|--------|---------|
| `Proto` | Protocol (tcp, udp, unix, etc.) |
| `Recv-Q` | Receive queue — bytes not read by application |
| `Send-Q` | Send queue — bytes not acknowledged by remote |
| `Local Address` | Local IP:Port |
| `Foreign Address` | Remote IP:Port |
| `State` | Connection state |
| `PID/Program` | Process ID and name (with `-p`) |

## Practical Examples

```bash
# Find which process is using a port
sudo netstat -tulpn | grep :80
sudo netstat -tulpn | grep ':3306'

# Show all listening services
sudo netstat -tulpn

# Show established connections
netstat -tan | grep ESTABLISHED
netstat -tanp | grep ESTABLISHED

# Find connections to specific IP
netstat -tan | grep 192.168.1.100
netstat -tanp | grep 192.168.1.100

# Show TIME_WAIT connections
netstat -tan | grep TIME_WAIT
netstat -tan | grep TIME_WAIT | wc -l

# Monitor connections continuously
netstat -tc                    # Continuous TCP
watch -n 1 'netstat -tan'     # Update every second

# Show summary statistics
netstat -s
netstat -st                   # TCP only
netstat -su                   # UDP only

# Show network interface statistics
netstat -i
netstat -ie                   # Extended info

# Show Unix sockets
netstat -x
netstat -xa                   # All Unix sockets
netstat -xp                   # With process info

# Find half-open connections (potential SYN flood)
netstat -tan | grep SYN_RECV
```

## One-Liners

```bash
# Count connections per state
netstat -tan | awk '{print $6}' | sort | uniq -c | sort -rn

# Top 10 IPs by connection count
netstat -tan | grep ESTABLISHED | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -10

# Count connections by IP (all states)
netstat -tan | grep ESTABLISHED | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn

# Connections to specific port
netstat -tan | grep ':80 '
netstat -tan | awk '$4 ~ /:80$/'

# Local ports in use
netstat -tuln | awk '{print $4}' | grep -o '[0-9]*$' | sort -n | uniq

# Count by protocol
netstat -tan | awk '{print $1}' | sort | uniq -c

# Check for SYN flood
netstat -tan | grep SYN_RECV | wc -l

# Monitor connection rate
watch -n 1 'netstat -tan | grep ESTABLISHED | wc -l'

# Detect port scans (many SYN_RECV from same IP)
netstat -tan | grep SYN_RECV | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn
```

## Automation Scripts

### Monitor connections

```bash
#!/bin/bash
while true; do
    echo "=== $(date) ==="
    echo "Established: $(netstat -tan | grep ESTABLISHED | wc -l)"
    echo "TIME_WAIT: $(netstat -tan | grep TIME_WAIT | wc -l)"
    echo "Listening: $(netstat -tln | wc -l)"
    echo ""
    sleep 5
done
```

### Alert on high connection count

```bash
#!/bin/bash
THRESHOLD=1000
COUNT=$(netstat -tan | grep ESTABLISHED | wc -l)
if [ $COUNT -gt $THRESHOLD ]; then
    echo "ALERT: $COUNT established connections (threshold: $THRESHOLD)"
fi
```

### Generate connection report

```bash
#!/bin/bash
echo "=== Network Connection Report ==="
echo "Date: $(date)"
echo ""
echo "Listening Ports:"
netstat -tuln
echo ""
echo "Established Connections: $(netstat -tan | grep ESTABLISHED | wc -l)"
echo ""
echo "Top 10 Connected IPs:"
netstat -tan | grep ESTABLISHED | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -10
```

## netstat to ss Migration

| netstat | ss equivalent |
|---------|---------------|
| `netstat -tuln` | `ss -tuln` |
| `netstat -tanp` | `ss -tanp` |
| `netstat -s` | `ss -s` |
| `netstat -i` | `ip -s link` |
| `netstat -r` | `ip route` |
| `netstat -g` | `ip maddr` |

## Tips

- Use `-n` to avoid slow DNS lookups
- Add `-p` (as root) to see which processes own connections
- `grep` is your friend for filtering output
- For scripting, avoid `-p` if you don't need process info (faster)
- Use `ss` instead for better performance on busy systems
- Combine with `watch` for continuous monitoring
- High `Recv-Q` or `Send-Q` indicates application bottleneck
- Many `TIME_WAIT` connections are normal after high traffic
- `-c` provides continuous output but can be resource-intensive
- `ss` uses netlink and is significantly faster when thousands of sockets exist
