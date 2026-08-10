# ss Cheatsheet

Comprehensive ss (socket statistics) reference guide covering common flags, practical socket inspection examples, and real-world troubleshooting for connection states, port exhaustion, socket memory, and TCP retransmits. ss queries the kernel directly via netlink, making it significantly faster than netstat (which reads /proc).

## Common Flags

| Flag | Description |
|------|-------------|
| `-t` | TCP sockets |
| `-u` | UDP sockets |
| `-l` | Listening sockets only |
| `-a` | All sockets (listening + established) |
| `-n` | Don't resolve service names (show port numbers) |
| `-r` | Resolve IP addresses to hostnames |
| `-p` | Show process using the socket |
| `-e` | Show extended socket info |
| `-i` | Show TCP internal info (RTT, congestion) |
| `-m` | Show socket memory usage |
| `-o` | Show timer information |
| `-s` | Show socket statistics summary |
| `-w` | Show raw sockets |
| `-x` | Show Unix domain sockets |
| `-4` | IPv4 only |
| `-6` | IPv6 only |

## Practical Examples

### List all listening TCP ports with process names

```bash
ss -tlnp
```

### List all listening UDP ports

```bash
ss -ulnp
```

### Show all established connections

```bash
ss -tn
```

### Show all connections in every state

```bash
ss -tn state all
```

### Connections in each TCP state (full breakdown)

```bash
ss -Htn state all | awk '{print $1}' | sort | uniq -c | sort -rn
```

### Find what process is using a specific port

```bash
ss -tlnp | grep :8080
ss -tlnp 'sport = :8080'
```

### Filter by port number

```bash
ss -tn sport = :443                          # Source port 443
ss -tn dport = :443                          # Destination port 443
ss -tn sport = :80 or sport = :443           # Multiple ports
```

### Filter by port range

```bash
ss -tn sport gt :1024                        # Source port > 1024
ss -tn dport lt :1024                        # Destination port < 1024
ss -tn sport ge :8000 and sport le :9000     # Range 8000-9000
```

### Advanced filter expressions

```bash
# Connections to port 80 or 443 from specific subnet
ss -tn '( dport = :80 or dport = :443 ) and src 192.168.1.0/24'

# Not from localhost
ss -tn dst != 127.0.0.1

# Multiple conditions
ss -tn sport ge :1024 and sport le :65535 state established
```

### Filter operators

| Operator | Alias | Description |
|----------|-------|-------------|
| `=` | `eq` | Equal |
| `!=` | `ne` | Not equal |
| `>` | `gt` | Greater than |
| `<` | `lt` | Less than |
| `>=` | `ge` | Greater than or equal |
| `<=` | `le` | Less than or equal |
| `and` | | Logical AND |
| `or` | | Logical OR |
| `not` | | Logical NOT |
| `()` | | Grouping |

### Filter by state

```bash
ss -t state established         # Only established
ss -t state listening           # Only listening
ss -t state time-wait           # Only TIME_WAIT
ss -t state close-wait          # Only CLOSE_WAIT
ss -t state fin-wait-1          # Only FIN_WAIT_1
ss -t state syn-recv            # Only SYN_RECV (half-open)
```

### Filter by IP address

```bash
ss -tn dst 10.0.0.5            # Connections to specific IP
ss -tn src 192.168.1.10        # Connections from specific IP
ss -tn dst 10.0.0.0/24         # Connections to a subnet
```

### Count connections per state

```bash
ss -t | awk '{print $1}' | sort | uniq -c | sort -rn
```

### Count connections per remote IP

```bash
ss -tn | awk 'NR>1 {print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -10
```

### Show TCP connections with timer info

```bash
ss -tn -o
```

### Show detailed TCP info (RTT, congestion window)

```bash
ss -ti
```

### Show memory usage per socket

```bash
ss -tm
```

### Find CLOSE_WAIT connections (potential leak)

```bash
ss -tn state close-wait
ss -tnp state close-wait        # With process info to find the culprit
```

### Find TIME_WAIT connections (high volume = port exhaustion risk)

```bash
ss -tn state time-wait | wc -l
ss -t state time-wait                    # Show all TIME_WAIT connections
ss -t state time-wait sport = :80        # TIME_WAIT on your local port 80 (server side)
ss -t state time-wait dst :80            # TIME_WAIT to remote port 80 (client/proxy side)
ss -t state time-wait dst 10.0.0.50      # TIME_WAIT to a specific remote IP

# TIME_WAIT sockets grouped by destination (top offenders)
ss -Htn state time-wait | awk '{print $4}' | sort | uniq -c | sort -rn | head -10

# All connections to a specific destination
ss -Htn dst 10.0.0.50 | wc -l
ss -Htn state time-wait dst 10.0.0.50 | wc -l
```

### Show all UNIX domain sockets

```bash
ss -x
ss -xlp                         # Listening UNIX sockets with process
```

### Show socket summary (quick health check)

```bash
ss -s
```

### Check if a service is listening

```bash
ss -tlnp | grep nginx
ss -tlnp | grep :3306           # MySQL
ss -tlnp | grep :6379           # Redis
ss -tlnp | grep :5432           # PostgreSQL
```

### Detect SYN flood (many SYN_RECV)

```bash
ss -tn state syn-recv | wc -l
```

### Monitor connections in real-time (watch)

```bash
watch -n 2 'ss -tan | wc -l'                                     # Total connection count
watch -n 1 'ss -tan sport = :80 | grep ESTAB | wc -l'            # Monitor specific port
watch -n 1 'ss -tn state established | wc -l'                    # Established count
watch -n 1 'ss -s'                                               # Full summary
```

### Monitor connections over time

```bash
while true; do
    echo "$(date): $(ss -tan | grep ESTAB | wc -l) connections"
    sleep 60
done > connection_count.log
```

### Find connections to a specific service from all clients

```bash
ss -tn sport = :443 | awk 'NR>1 {print $5}' | cut -d: -f1 | sort -u
ss -tn sport = :22 | awk 'NR>1 {print $5}' | cut -d: -f1 | sort -u    # Who's SSHed in
```

### Check send/receive queue (buffered data)

```bash
ss -tn | awk '$2 > 0 || $3 > 0'   # Non-zero queues = potential issue
```

High `Recv-Q` on LISTEN = connections waiting to be accepted (app too slow).
High `Send-Q` on ESTAB = data waiting to be sent (network congestion or slow peer).


### Connection count per port (what services are busy)

```bash
ss -tn | awk 'NR>1 {print $4}' | rev | cut -d: -f1 | rev | sort | uniq -c | sort -rn
```

### Connections per state per port

```bash
ss -tan | awk 'NR>1 {split($4,a,":"); print $1, a[length(a)]}' | sort | uniq -c | sort -rn
```

### Top talkers — IPs with most connections

```bash
ss -tn | awk 'NR>1 {print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -20
```

### Connections per IP per port (detect abuse per service)

```bash
ss -tn | awk 'NR>1 {split($4,l,":"); split($5,r,":"); print r[1], l[length(l)]}' | sort | uniq -c | sort -rn | head -20
```

### Detect port exhaustion (ephemeral port usage)

```bash
cat /proc/sys/net/ipv4/ip_local_port_range    # Show ephemeral range (e.g. 32768-60999)
ss -tn | awk 'NR>1 {split($4,a,":"); p=a[length(a)]; if(p>=32768 && p<=60999) c++} END {print c, "/ 28231 ephemeral ports used"}'
```

> Ephemeral ports are only used for the client side of outbound connections.
Each time the box initiates a connection out (DNS, apt, curl, a DB client, a monitoring agent phoning home), the kernel grabs one port from the range `32768–60999`. Inbound connections to your listening services (sshd on 22, web server on 80/443) use the fixed service port locally, not an ephemeral one, so they don't add to this count. A server that mostly *listens* rather than *connects out* will idle at a handful.

> They're counted only while the connection is active.
Once a connection closes and clears `TIME_WAIT`, the port is freed and drops off the count. A quiet server idles at a handful; pressure only shows up under heavy outbound churn.

> To see what those active ephemeral ports actually are (local port, remote endpoint, and owning process):

```bash
ss -tnp state established '( sport >= 32768 )'
```

### Socket backlog — are connections being dropped?

```bash
ss -tln | awk 'NR>1 && $2 > 0 {print "WARNING: backlog on port", $4, "Recv-Q:", $2}'
```

On a LISTEN socket:
- `Recv-Q` = number of connections waiting to be accepted
- `Send-Q` = max backlog size (set by application)

If `Recv-Q` approaches `Send-Q`, connections are being dropped.

```bash
ss -tln | awk 'NR>1 {split($4,a,":"); printf "Port %-6s Backlog: %s/%s", a[length(a)], $2, $3; if($2/$3 > 0.8) printf " ⚠️  NEAR FULL"; print ""}'
```

### Orphaned sockets (no process attached)

```bash
ss -tn state close-wait | wc -l              # Orphaned connections (app forgot to close)
cat /proc/sys/net/ipv4/tcp_max_orphans       # System limit
```

### Socket memory usage per connection

```bash
ss -tnm | grep -A1 ESTAB | grep skmem
```

Output:
```
skmem:(r0,rb131072,t0,tb16384,f0,w0,o0,bl0,d0)
```

Fields:
- `r` = read buffer used
- `rb` = read buffer max
- `t` = write buffer used
- `tb` = write buffer max
- `f` = forward allocated
- `w` = queued for transmit
- `bl` = backlog
- `d` = drops

### Total memory used by all TCP sockets

```bash
cat /proc/net/sockstat
```

`mem` is in pages (4KB each). `mem 45` = ~180KB used by TCP.

### TCP retransmits (packet loss indicator)

```bash
ss -ti | grep -c retrans
ss -ti | grep retrans              # Show which connections have retransmits
```

Or from system-wide stats:

```bash
cat /proc/net/snmp | grep Tcp | awk 'NR==2 {print "Retransmits:", $12, "Segments sent:", $11, "Ratio:", $12/$11*100 "%"}'
```

### Connection duration — find long-lived connections

```bash
ss -tnoe | awk 'NR>1 && /timer/' | sort -t, -k2 -rn | head -10
```

### Check congestion control algorithm

```bash
ss -ti | grep -E 'cubic|bbr|reno'
```

### Find maximum segment size

```bash
ss -ti | grep mss
```

## Comparison with netstat

### ss advantages
- Much faster (uses netlink instead of /proc)
- More efficient on systems with many connections
- Better filtering capabilities
- Shows more detailed socket information

### Command equivalents

| netstat | ss |
|---------|-----|
| `netstat -tan` | `ss -tan` |
| `netstat -tulpn` | `ss -tulpn` |
| `netstat -s` | `ss -s` |
| `netstat -r` | `ip route` |
| `netstat -i` | `ip -s link` |

## TCP States Explained

### TCP Connection Overview

TCP is a **full-duplex**, connection-oriented protocol operating at the transport layer. A TCP connection goes through three phases:

> 1. **Connection establishment** — 3-way handshake (SYN → SYN-ACK → ACK)
> 2. **Data transfer** — bidirectional data flow (ESTABLISHED state)
> 3. **Connection termination** — 4-way handshake (FIN → ACK → FIN → ACK)

The connection passes through **11 states** during its lifetime: CLOSED, LISTEN, SYN_SENT, SYN_RECV, ESTABLISHED, FIN_WAIT_1, FIN_WAIT_2, CLOSE_WAIT, CLOSING, LAST_ACK, TIME_WAIT.

The OS manages each connection as a resource (socket + file descriptor). The `ss` command lets you inspect these states in real time.

### TCP State Transition Diagram

![TCP state machine](./images/tcp_state_diagram.svg)

### TCP States Reference (RFC 793)

| State | Who Has It | Definition | Meaning in Practice | Concern |
|-------|-----------|-----------|---------------------|---------|
| **CLOSED** | Neither | No connection exists | Connection fully terminated | Normal |
| **LISTEN** | Server | Waiting for a connection request from a remote endpoint (passive open) | Socket waiting for incoming connections (your service is ready) | If missing, service is down |
| **SYN_SENT** | Client | Sent a SYN, waiting for SYN-ACK (active open initiated) | Sent SYN, waiting for SYN-ACK from remote | Stuck = remote not reachable |
| **SYN_RECV** | Server | Received a SYN, sent SYN-ACK, waiting for final ACK (half-open) | Got SYN, sent SYN-ACK, waiting for final ACK | Many = SYN flood attack |
| **ESTABLISHED** | Both | Three-way handshake complete, connection is open and data flows | Connection is active, data flowing both ways | Normal healthy state |
| **FIN_WAIT_1** | Active closer | Sent a FIN (active close initiated), waiting for ACK or FIN | Your side sent FIN, waiting for remote ACK | Stuck = remote unresponsive or firewall issue |
| **FIN_WAIT_2** | Active closer | Received ACK for our FIN, waiting for remote's FIN | Remote ACKed the FIN, waiting for remote's FIN | Stuck = remote app not closing |
| **CLOSE_WAIT** | Passive closer | Received FIN from remote, sent ACK, waiting for local app to close | Remote closed, YOUR app hasn't called `close()` yet | App bug if growing — memory/fd leak |
| **CLOSING** | Both | Both sides sent FIN simultaneously, waiting for ACK | Both sides sent FIN simultaneously | Rare, transient |
| **LAST_ACK** | Passive closer | Sent FIN after receiving remote's FIN, waiting for final ACK | Sent FIN after CLOSE_WAIT, waiting for final ACK | Stuck = firewall dropping packets |
| **TIME_WAIT** | Active closer | Waiting 2×MSL (typically 60s) before fully closing, ensures late packets are handled | Connection closed properly, socket stays 60s for late packets | Normal, but too many = port exhaustion |

## Kernel Tuning Parameters Related to ss Output

```bash
# View current TCP tuning
sysctl net.ipv4.tcp_max_syn_backlog       # Max SYN_RECV queue
sysctl net.core.somaxconn                 # Max listen backlog
sysctl net.ipv4.tcp_max_tw_buckets        # Max TIME_WAIT sockets
sysctl net.ipv4.tcp_fin_timeout           # How long FIN_WAIT_2 lasts
sysctl net.ipv4.tcp_tw_reuse              # Reuse TIME_WAIT for outbound
sysctl net.ipv4.ip_local_port_range       # Ephemeral port range
sysctl net.ipv4.tcp_max_orphans           # Max orphaned (CLOSE_WAIT) sockets
```

## Metrics That Impact Server Performance

| Metric | How to check | Impact |
|--------|-------------|--------|
| **High ESTAB count** | `ss -tn state established \| wc -l` | Memory/CPU pressure, file descriptor exhaustion |
| **Growing CLOSE_WAIT** | `ss -tn state close-wait \| wc -l` | Memory leak, fd leak — app not closing sockets |
| **High TIME_WAIT** | `ss -tn state time-wait \| wc -l` | Ephemeral port exhaustion, can't open new connections |
| **SYN_RECV spike** | `ss -tn state syn-recv \| wc -l` | SYN flood attack or backlog overflow |
| **Recv-Q > 0 on LISTEN** | `ss -tln \| awk '$2 > 0'` | App too slow to accept connections, clients timeout |
| **Send-Q > 0 on ESTAB** | `ss -tn \| awk '$3 > 0'` | Network congestion or slow peer, memory buildup |
| **Retransmits** | `ss -ti \| grep retrans` | Packet loss, latency spikes, throughput drops |
| **High RTT** | `ss -ti \| grep rtt` | Slow responses, timeouts |
| **Orphaned sockets** | `cat /proc/net/sockstat` | Consuming memory with no process to clean up |
| **Ephemeral port usage** | `ss -tn \| awk 'NR>1' \| wc -l` vs range | Port exhaustion = "cannot assign address" errors |
| **Single IP with many connections** | `ss -tn \| awk 'NR>1 {print $5}' \| cut -d: -f1 \| sort \| uniq -c \| sort -rn \| head -5` | DDoS/abuse, resource hogging |
| **Socket memory (skmem)** | `ss -tnm` | Kernel memory pressure, OOM risk |
