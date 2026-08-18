# /proc/net/sockstat Explained

The fastest way to get a system-wide socket summary without any tool — just read the file:

```bash
cat /proc/net/sockstat
```

Example output:

```
sockets: used 356
TCP: inuse 86 orphan 0 tw 12 alloc 98 mem 45
UDP: inuse 5 mem 2
UDPLITE: inuse 0
RAW: inuse 1
FRAG: inuse 0 memory 0
```

## Field-by-Field Breakdown

| Line | Field | Meaning |
|------|-------|---------|
| `sockets` | `used` | Total sockets allocated across all protocols |
| `TCP` | `inuse` | TCP sockets currently in use (ESTABLISHED + CLOSE_WAIT + FIN_WAIT, etc.) |
| `TCP` | `orphan` | TCP sockets with no owning process (app crashed or exited without closing) |
| `TCP` | `tw` | Sockets in TIME_WAIT state |
| `TCP` | `alloc` | Total TCP sockets allocated (including LISTEN + TIME_WAIT + all states) |
| `TCP` | `mem` | Pages of memory used by TCP sockets (1 page = 4KB typically) |
| `UDP` | `inuse` | UDP sockets currently in use |
| `UDP` | `mem` | Pages of memory used by UDP sockets |
| `UDPLITE` | `inuse` | UDP-Lite sockets (rarely used) |
| `RAW` | `inuse` | Raw sockets (ping, traceroute, custom protocols) |
| `FRAG` | `inuse` | IP fragment reassembly queues |
| `FRAG` | `memory` | Memory used for fragment reassembly |

## What to Watch For

```bash
# Quick health check one-liner
cat /proc/net/sockstat | awk '/TCP/ {print "InUse:",$2, "Orphan:",$4, "TW:",$6, "Alloc:",$8, "Mem:",$10"pages ("$10*4"KB)"}'
```

| Condition | What It Means |
|-----------|---------------|
| `orphan` growing | Processes dying without closing connections — potential crash or kill -9 |
| `orphan` near `tcp_max_orphans` | Kernel will start dropping orphaned connections |
| `tw` high (>20,000) | Many short-lived connections — check port exhaustion |
| `mem` high | TCP buffers consuming significant RAM — tune `tcp_mem` |
| `alloc` >> `inuse` | Many sockets in LISTEN or TIME_WAIT state |
| `inuse` near `ulimit -n` | Approaching file descriptor limit — service will refuse connections |

## Related Kernel Limits

```bash
# Max orphaned TCP sockets (default ~16K-65K depending on RAM)
sysctl net.ipv4.tcp_max_orphans

# TCP memory pressure thresholds (in pages): [low pressure high]
sysctl net.ipv4.tcp_mem
# When TCP mem usage hits "pressure", kernel starts reclaiming aggressively
# When it hits "high", new allocations are refused

# Max TIME_WAIT sockets
sysctl net.ipv4.tcp_max_tw_buckets

# Check current memory usage in bytes
cat /proc/net/sockstat | awk '/TCP/ {print "TCP memory:", $10 * 4, "KB"}'
```

## /proc/net/sockstat6 (IPv6)

```bash
cat /proc/net/sockstat6
```

```
TCP6: inuse 14
UDP6: inuse 3
UDPLITE6: inuse 0
RAW6: inuse 1
FRAG6: inuse 0 memory 0
```

Same concept, but for IPv6 sockets. Add both together for total system view.

## Using in Scripts and Monitoring

```bash
# Parse for Prometheus textfile collector or custom monitoring
while IFS= read -r line; do
    case "$line" in
        TCP:*) echo "$line" | awk '{print "tcp_inuse "$2, "\ntcp_orphan "$4, "\ntcp_tw "$6, "\ntcp_alloc "$8, "\ntcp_mem_pages "$10}' ;;
        UDP:*) echo "$line" | awk '{print "udp_inuse "$2, "\nudp_mem_pages "$4}' ;;
    esac
done < /proc/net/sockstat

# Simple alerting
ORPHANS=$(awk '/TCP/ {print $4}' /proc/net/sockstat)
MAX_ORPHANS=$(sysctl -n net.ipv4.tcp_max_orphans)
if [ "$ORPHANS" -gt $((MAX_ORPHANS * 80 / 100)) ]; then
    echo "TCP orphan sockets at ${ORPHANS}/${MAX_ORPHANS} (80%+ threshold)"
fi
```

## /proc/net/snmp — TCP Protocol Statistics

Another useful proc file for TCP health:

```bash
cat /proc/net/snmp | grep Tcp
```

Output (two lines — headers then values):

```
Tcp: RtoAlgorithm RtoMin RtoMax MaxConn ActiveOpens PassiveOpens AttemptFails EstabResets CurrEstab InSegs OutSegs RetransSegs InErrs OutRsts InCsumErrors
Tcp: 1 200 120000 -1 45231 12305 234 89 86 9823456 8765432 1234 0 456 0
```

### Key Fields

| Field | Meaning |
|-------|---------|
| `ActiveOpens` | Total outbound connections initiated (connect() calls) |
| `PassiveOpens` | Total inbound connections accepted (accept() calls) |
| `AttemptFails` | Connection attempts that failed |
| `EstabResets` | Established connections reset (RST received/sent) |
| `CurrEstab` | Currently established connections |
| `RetransSegs` | Total segments retransmitted (packet loss indicator) |
| `InErrs` | Segments received with errors |
| `OutRsts` | RST segments sent |

### Calculate Retransmit Rate

```bash
cat /proc/net/snmp | grep Tcp | awk 'NR==2 {printf "Retransmits: %d\nSegments Out: %d\nRetransmit Rate: %.4f%%\n", $12, $11, $12/$11*100}'
```

## /proc/net/netstat — Extended TCP Stats

```bash
cat /proc/net/netstat | grep TcpExt
```

Key fields to watch:

| Field | Meaning |
|-------|---------|
| `ListenOverflows` | Connections dropped because listen queue was full |
| `ListenDrops` | Same as above (older kernel naming) |
| `TCPAbortOnMemory` | Connections aborted due to memory pressure |
| `TCPAbortOnTimeout` | Connections aborted due to timeout |
| `TCPSynRetrans` | SYN retransmits (initial connection failing) |
| `TCPTimeouts` | Total TCP timeouts |

```bash
# One-liner to check listen queue drops
cat /proc/net/netstat | grep -oP 'ListenOverflows \K[0-9]+'

# Watch for growth over time
watch -n 5 "cat /proc/net/netstat | grep TcpExt | tr ' ' '\n' | grep -A1 -E 'ListenOverflows|ListenDrops|TCPAbortOnMemory'"
```

## Quick Reference: All /proc/net Files for TCP

| File | What It Shows |
|------|--------------|
| `/proc/net/sockstat` | Socket summary (counts, memory) |
| `/proc/net/sockstat6` | Same for IPv6 |
| `/proc/net/snmp` | Protocol-level counters (retransmits, resets) |
| `/proc/net/netstat` | Extended TCP stats (drops, aborts, timeouts) |
| `/proc/net/tcp` | Full list of every TCP socket (what `ss` reads) |
| `/proc/net/tcp6` | Same for IPv6 |
| `/proc/sys/net/ipv4/` | Kernel tuning parameters |

## "Out of socket memory" Error Explained

```
kernel: Out of socket memory
```

or on newer kernels (4.0+):

```
kernel: too many orphaned sockets
kernel: out of memory -- consider tuning tcp_mem
```

This is a **different** issue from conntrack table full. It's triggered by TCP's internal memory management.

### Two Causes

The error is triggered by `tcp_check_oom()` in the kernel when either:

1. **Too many orphan sockets** (most common case)
2. **TCP memory limit exceeded** (rare)

### What's an Orphan Socket?

An orphan socket is a TCP socket no longer associated with any file descriptor. After your app calls `close()`, the socket still exists in the kernel until TCP finishes the teardown (FIN_WAIT, TIME_WAIT, etc.). The app can't interact with it anymore — it's "orphaned."

Frontend servers (nginx, Varnish, HAProxy) typically have many orphans — this is normal.

### Diagnosing Which Case You're In

```bash
# Check TCP memory: current vs limits (values in PAGES, not bytes)
cat /proc/sys/net/ipv4/tcp_mem
# Output: 3093984 4125312 6187968
# Meaning: [low_threshold] [pressure_mode] [max_limit]
# Convert to bytes: pages × 4096

# Check current TCP memory usage (last field on TCP line)
grep TCP /proc/net/sockstat
# TCP: inuse 35938 orphan 21564 tw 70529 alloc 35942 mem 1894
#                                                      ^^^^
# "mem 1894" = 1894 pages = ~7.4MB of TCP memory used

# If mem << tcp_mem[2]: you're in Case 1 (orphan issue)
# If mem ≈ tcp_mem[2]: you're in Case 2 (memory issue)
```

### Case 1: Too Many Orphans (Most Common)

```bash
# Check orphan limit
cat /proc/sys/net/ipv4/tcp_max_orphans
# Default: 65536

# Check current orphan count
grep orphan /proc/net/sockstat
# TCP: inuse 35938 orphan 21564 tw 70529 alloc 35942 mem 1894
```

**The misleading part**: You might see the error even when orphan count is 2x-4x BELOW the limit.

The kernel uses a `shift` variable (0-2) that multiplies the orphan count to penalize "bad" sockets:
- Socket idle too long — shift +1 (effectively doubles the count)
- Socket received dubious ICMP — shift +1 (effectively quadruples the count)

The check is: `if (orphans << shift > tcp_max_orphans)` — meaning with shift=2, your effective limit is `tcp_max_orphans / 4`.

**Fix:**

```bash
# Observe peak orphan count during traffic, multiply by 4, round up
# Example: peak orphans = 25000 → set to 120000

sysctl -w net.ipv4.tcp_max_orphans=120000
echo "net.ipv4.tcp_max_orphans = 120000" >> /etc/sysctl.conf
```

### Case 2: TCP Memory Limit Exceeded (Rare)

```bash
# Check tcp_mem (in pages)
cat /proc/sys/net/ipv4/tcp_mem
# 3093984 4125312 6187968
# Convert: 6187968 pages × 4096 = ~23.6GB max

# Check current usage
awk '/TCP/ {print "TCP memory used:", $NF, "pages =", $NF*4/1024, "MB"}' /proc/net/sockstat

# If close to tcp_mem[2], increase it:
sysctl -w net.ipv4.tcp_mem="6187968 8250624 12375936"
```

This is rare but happens on systems where someone manually set `tcp_mem` too low.

### Summary

```bash
# Full diagnostic one-liner
echo "=== TCP Memory ===" && \
awk '/TCP/ {printf "Used: %d pages (%.1f MB)\n", $NF, $NF*4/1024}' /proc/net/sockstat && \
echo "Limits (pages): $(cat /proc/sys/net/ipv4/tcp_mem)" && \
echo "" && \
echo "=== Orphans ===" && \
awk '/TCP/ {printf "Current: %s\n", $4}' /proc/net/sockstat && \
echo "Max: $(cat /proc/sys/net/ipv4/tcp_max_orphans)" && \
echo "Effective max (with shift): $(($(cat /proc/sys/net/ipv4/tcp_max_orphans) / 4))"
```

### Key Takeaway

The "Out of socket memory" message is often a **false alarm** caused by the shift penalty on a few bad sockets. The fix is usually just increasing `tcp_max_orphans` to 4x your peak orphan count. Don't blindly tune every TCP knob — diagnose which case you're in first.
