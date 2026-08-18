# Ephemeral Ports vs Conntrack Max — Two Different Limits

These two limits are constantly confused. They are **completely separate ceilings** with different causes, symptoms, and fixes. Exhausting one does not affect the other.

## Quick Comparison

| | Ephemeral Ports | Conntrack Max |
|---|---|---|
| Typical value | ~28,231 (`32768–60999`) | 65,536 – 262,144+ |
| What it counts | Outbound connections **per destination** | ALL flows (in + out, all destinations) |
| Limited by | 16-bit port number (max 65,535) | Kernel memory (hash table) |
| Scope | Per four-tuple (per destination) | Global per network namespace |
| Sysctl | `net.ipv4.ip_local_port_range` | `net.netfilter.nf_conntrack_max` |
| Failure symptom | `EADDRNOTAVAIL` ("cannot assign requested address") | `nf_conntrack: table full, dropping packet` |
| Present | Always | Only if firewall/NAT (conntrack) loaded |
| Who hits it | Clients/proxies hammering one backend | Busy nodes with many total connections (esp. K8s) |

## Ephemeral Ports (~28,231)

```bash
sysctl net.ipv4.ip_local_port_range    # e.g. "32768 60999" → 28,231 ports
```

- **Limits**: how many **outbound** connections you can make **to the same destination** (same dst IP + dst port).
- **Why ~28K**: it's a 16-bit port field (65,535 max) minus the reserved low range.
- **Scope is per four-tuple**: you can have 28,231 connections to `DB-A:5432` **and** another 28,231 to `DB-B:5432` at the same time. The limit is per destination, not global.
- **Exhaustion**: `connect()` fails with `EADDRNOTAVAIL`.
- **Who hits it**: a service making many short-lived outbound connections to a single backend (no connection pooling).

### Check ephemeral port pressure

```bash
# Range size
sysctl -n net.ipv4.ip_local_port_range | awk '{print $2 - $1, "ports available"}'

# Unique local ports tied up in TIME_WAIT (the REAL exhaustion number)
ss -Htn state time-wait | awk '{n=split($3,a,":"); print a[n]}' | sort -un | wc -l

# Outbound connections grouped by destination (who are we hammering?)
ss -Htn | awk '{print $5}' | sort | uniq -c | sort -rn | head
```

### Fix ephemeral port exhaustion

```bash
sysctl -w net.ipv4.tcp_tw_reuse=1                          # Reuse TIME_WAIT (outbound, safe)
sysctl -w net.ipv4.ip_local_port_range="10000 65535"       # Widen the range
# Real fix: connection pooling / keep-alive in the app
```

## Conntrack Max (e.g. 262,144)

```bash
sysctl net.netfilter.nf_conntrack_max    # e.g. 262144
cat /proc/sys/net/netfilter/nf_conntrack_count   # current usage
```

- **Limits**: total **firewall-tracked flows** on the whole machine — TCP + UDP + ICMP, inbound + outbound, all destinations combined.
- **Why**: each tracked flow consumes kernel memory in the conntrack hash table.
- **Scope is global** (per network namespace): every tracked connection counts toward one shared total.
- **Exhaustion**: `nf_conntrack: table full, dropping packet` in dmesg; new connections silently dropped.
- **Who hits it**: busy nodes (especially Kubernetes) with high total connection counts across all services.

### Check conntrack pressure

```bash
echo "$(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max)"
conntrack -S | grep -E "drop|insert_failed"    # non-zero = overflowing
```

### Fix conntrack exhaustion

```bash
sysctl -w net.netfilter.nf_conntrack_max=524288                       # More entries
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30         # Expire faster
# Or NOTRACK on high-traffic ports to bypass conntrack entirely
```

## The Key Insight

A **single outbound connection** uses:
- **1 ephemeral port** AND **1 conntrack entry**

A **single inbound connection** (to your server) uses:
- **0 ephemeral ports** (it reuses the listening port like :443) AND **1 conntrack entry**

So:

- You can **exhaust ephemeral ports** (28K to one destination) while **conntrack is nearly empty** — different problems.
- You can **fill conntrack** with lots of *inbound* connections while **ephemeral ports stay untouched**.
- Inbound TIME_WAIT does **not** consume ephemeral ports — thousands of them share one listening port.

### Important caveat for monitoring

When a script reports "TIME_WAIT as % of ephemeral ports", that denominator is only correct for **outbound** TIME_WAIT. If most TIME_WAIT sockets are **inbound** (server side), the percentage is misleading — those don't consume ephemeral ports.

To get the real outbound exhaustion number, count **unique local ports**:

```bash
ss -Htn state time-wait | awk '{n=split($3,a,":"); print a[n]}' | sort -un | wc -l
```

If that number is one or a handful (e.g., just `:443`), it's inbound — not an exhaustion risk. If it's approaching the full ephemeral range, it's real outbound exhaustion.

## Decision Guide

| Symptom | Which limit | Fix |
|---------|-------------|-----|
| `EADDRNOTAVAIL` on connect() | Ephemeral ports | `tcp_tw_reuse`, wider range, pooling |
| `nf_conntrack: table full` in dmesg | Conntrack max | Raise `nf_conntrack_max`, lower timeouts, NOTRACK |
| High TIME_WAIT but one local port | Neither (inbound, harmless) | Nothing needed |
| High TIME_WAIT across many local ports | Ephemeral ports | Pooling, `tcp_tw_reuse` |
| Random drops on a busy K8s node | Likely conntrack | Check `conntrack -S`, raise max |
