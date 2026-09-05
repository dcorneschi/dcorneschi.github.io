# iperf3 Cheatsheet

iperf3 is a tool for active measurement of the maximum achievable bandwidth on IP
networks. It runs a client and a server, generates TCP/UDP/SCTP traffic between them, and
reports throughput, jitter, packet loss, and retransmits. Use it to benchmark links,
validate NIC and switch speeds, and troubleshoot slow transfers.

> iperf3 is **not** wire-compatible with iperf2. The client and server must both be
> iperf3, and ideally the **same version** — mismatched 3.x versions occasionally
> disagree on newer options.

## Installation

```bash
# RHEL/CentOS/Fedora
sudo dnf install -y iperf3

# Ubuntu/Debian
sudo apt install -y iperf3

# macOS
brew install iperf3

# Alpine
apk add iperf3

# Verify
iperf3 --version
```

## The Core Model: Server and Client

iperf3 always needs a **server** (listening) on one end and a **client** (connecting) on
the other. By default the client sends and the server receives.

```bash
# On host A — start the server (listens on TCP 5201 by default)
iperf3 -s

# On host B — connect to the server and run a 10s test
iperf3 -c host-a
```

- `-s` / `--server` — run as server.
- `-c <host>` / `--client <host>` — run as client, connecting to `<host>`.
- Default port is **5201** (TCP and UDP). Change with `-p`.
- The client drives all test parameters; the server just accepts them.

## Basic Tests

```bash
# Default test (TCP, 10 seconds, single stream, client → server)
iperf3 -c 10.0.0.1

# Set duration to 30 seconds
iperf3 -c 10.0.0.1 -t 30

# Reverse direction — server sends, client receives (test download)
iperf3 -c 10.0.0.1 -R

# Bidirectional — measure both directions at once (iperf3 3.7+)
iperf3 -c 10.0.0.1 --bidir

# Multiple parallel streams (fills a link a single stream can't saturate)
iperf3 -c 10.0.0.1 -P 4

# Report every 1 second (default) vs every 2 seconds
iperf3 -c 10.0.0.1 -i 2

# Print in a specific format: k/m/g/t bits, K/M/G/T bytes
iperf3 -c 10.0.0.1 -f m     # megabits/s
```

## Common Flags

| Flag | Long form | Meaning |
|------|-----------|---------|
| `-s` | `--server` | Run as server |
| `-c host` | `--client host` | Run as client to `host` |
| `-p N` | `--port N` | Port (default 5201) |
| `-t N` | `--time N` | Test duration in seconds (default 10) |
| `-n SIZE` | `--bytes SIZE` | Send a fixed number of bytes (e.g. `-n 1G`) instead of by time |
| `-k N` | `--blockcount N` | Send N blocks instead of by time |
| `-i N` | `--interval N` | Interval between reports (seconds) |
| `-P N` | `--parallel N` | Number of parallel streams |
| `-R` | `--reverse` | Reverse: server sends to client |
| `--bidir` | | Test both directions simultaneously |
| `-u` | `--udp` | Use UDP instead of TCP |
| `-b RATE` | `--bandwidth RATE` | Target bitrate (e.g. `-b 100M`); `0` = unlimited (TCP default) |
| `-w SIZE` | `--window SIZE` | Socket buffer / TCP window size (e.g. `-w 256K`) |
| `-M N` | `--set-mss N` | Set TCP MSS |
| `-l SIZE` | `--length SIZE` | Buffer/read-write length per packet |
| `-f X` | `--format X` | Output format: `k m g t` (bits), `K M G T` (bytes) |
| `-J` | `--json` | JSON output |
| `-4` / `-6` | | Force IPv4 / IPv6 |
| `-B addr` | `--bind addr` | Bind to a specific local interface/address |
| `-V` | `--verbose` | Verbose output (CPU, retransmits, system info) |
| `-D` | `--daemon` | Run server as a daemon |
| `-1` | `--one-off` | Server handles one client then exits |

## UDP Tests (jitter and packet loss)

TCP measures throughput; UDP is how you measure **jitter and loss**, because UDP won't
back off. You must set a target rate with `-b`, otherwise iperf3 defaults UDP to a low
1 Mbit/s.

```bash
# UDP at 100 Mbit/s for 20s — reports jitter and lost/total datagrams
iperf3 -c 10.0.0.1 -u -b 100M -t 20

# Push UDP as hard as possible (0 = no limit) to find the loss threshold
iperf3 -c 10.0.0.1 -u -b 0

# UDP reverse (server → client) to test the download path for loss
iperf3 -c 10.0.0.1 -u -b 500M -R

# Tune datagram size to probe MTU/fragmentation behavior
iperf3 -c 10.0.0.1 -u -b 200M -l 1400
```

Read the **server-side** (or receiver-side) summary for UDP — that's where jitter and
loss are reported, since the receiver is what actually observes them.

## Running the Server

```bash
# Foreground on the default port
iperf3 -s

# Listen on a custom port
iperf3 -s -p 5202

# Bind to one interface only
iperf3 -s -B 10.0.0.1

# Daemonize and log to a file
iperf3 -s -D --logfile /var/log/iperf3.log

# Handle exactly one test then exit (handy in scripts/CI)
iperf3 -s -1
```

### systemd service (persistent server)

```ini
# /etc/systemd/system/iperf3.service
[Unit]
Description=iperf3 server
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/iperf3 -s
Restart=on-failure
User=nobody

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now iperf3
```

> Modern iperf3 packages often ship `iperf3.service` and a socket-activated
> `iperf3@.service` already. Check with `systemctl cat iperf3` before writing your own.

## One-Liners

```bash
# Quick sanity check between two hosts (10s TCP)
iperf3 -c 10.0.0.1

# Saturate a 10GbE link with parallel streams and 30s duration
iperf3 -c 10.0.0.1 -P 8 -t 30

# Measure download (server → you) instead of upload
iperf3 -c 10.0.0.1 -R

# Both directions at once, per-second output in megabits
iperf3 -c 10.0.0.1 --bidir -i 1 -f m

# UDP loss/jitter test at 1Gbit/s
iperf3 -c 10.0.0.1 -u -b 1G -t 20

# Transfer a fixed 5 GB and report the average rate (good for storage/backup planning)
iperf3 -c 10.0.0.1 -n 5G

# Force IPv6
iperf3 -6 -c fd00::1

# JSON output piped to jq for the end-of-test throughput (bits/sec)
iperf3 -c 10.0.0.1 -J | jq '.end.sum_received.bits_per_second'

# JSON: retransmits (a proxy for congestion/loss on TCP)
iperf3 -c 10.0.0.1 -J | jq '.end.sum_sent.retransmits'

# Loop a short test every minute and append Gbit/s to a CSV
while true; do
  ts=$(date +%FT%T)
  bps=$(iperf3 -c 10.0.0.1 -t 5 -J | jq '.end.sum_received.bits_per_second')
  echo "$ts,$(echo "scale=2; $bps/1000000000" | bc)" >> throughput.csv
  sleep 60
done

# Test to a public iperf3 server (many are listed at iperf3serverlist.net)
iperf3 -c ping.online.net -p 5200-5209

# One-shot server for a single scripted test, then it exits
iperf3 -s -1 &
iperf3 -c 127.0.0.1 -t 5
```

## Tips and Tricks

- **A single TCP stream rarely fills a fast link.** On 10GbE+ or high-latency (high
  bandwidth-delay product) paths, one stream is limited by the TCP window. Use `-P` to
  run parallel streams, and/or raise the window with `-w`. If `-P 8` is much faster than
  `-P 1`, the bottleneck is per-stream windowing, not the link.

- **Bandwidth-delay product sets the window you need.** Required window ≈ bandwidth ×
  round-trip time. For 1 Gbit/s at 40 ms RTT that's ~5 MB — far above default socket
  buffers. Bump kernel limits (`net.core.rmem_max`, `net.core.wmem_max`,
  `net.ipv4.tcp_rmem`, `net.ipv4.tcp_wmem`) or use `-w`.

- **Test both directions.** `-R` (reverse) and `--bidir` catch asymmetric problems —
  duplex mismatches, one-way policing, or an uplink that's slower than the downlink.

- **UDP for loss/jitter, TCP for throughput.** TCP hides loss by retransmitting and
  backing off; its `Retr` column hints at trouble. To *measure* loss and jitter, use
  `-u -b <rate>` and read the receiver summary.

- **Watch the retransmits (`Retr`) column on TCP.** Nonzero and growing retransmits mean
  packet loss or congestion somewhere in the path — the link may be clean but a switch,
  policer, or buffer is dropping.

- **CPU can be the bottleneck, not the network.** At 10/25/40GbE a single core may max
  out before the link does. Run with `-V` to see CPU utilization; if the sender/receiver
  CPU is pegged, add streams (`-P`) to spread across cores or check offload settings.

- **Check NIC offloads and MTU.** `ethtool -k <iface>` for GRO/GSO/TSO, and confirm jumbo
  frames end-to-end (`ip link ... mtu 9000`) if you expect them. An MTU mismatch shows up
  as loss or capped throughput, often only in one direction.

- **Pin CPUs for repeatable results** with `-A` (affinity), e.g. `-A 2,3` to pin client
  and server threads. Reduces run-to-run variance on multi-core boxes.

- **`-t` (time) vs `-n` (bytes).** Use `-t` for steady-state throughput; use `-n`/`-k`
  when you care about how long a fixed transfer takes (backup windows, image copies).

- **Firewalls: open control *and* data.** iperf3 uses the same port (5201) for the
  control connection and TCP data, but for the classic multi-port setups make sure the
  chosen `-p` port is open on the **server**. For UDP tests, allow UDP on that port too.

- **Use `-1` on the server in automation** so it cleans itself up after one client
  instead of lingering.

- **JSON (`-J`) is your friend for scripting.** Parse
  `.end.sum_received.bits_per_second`, `.end.sum_sent.retransmits`, and (UDP)
  `.end.sum.jitter_ms` / `.end.sum.lost_percent` rather than scraping human output.

- **Version-match client and server.** Behavior of `--bidir`, `--reverse`, and JSON
  fields has shifted across 3.x releases. If results look wrong, check both
  `iperf3 --version`.

## Reading the Output

```
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   1.10 GBytes  9.45 Gbits/sec    0   3.14 MBytes
...
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  11.0 GBytes  9.42 Gbits/sec    0    sender
[  5]   0.00-10.00  sec  11.0 GBytes  9.42 Gbits/sec         receiver
```

- **Bitrate** — the headline throughput. Trust the final `receiver` line for what
  actually arrived.
- **Retr** — TCP retransmits during the interval. `0` is ideal; growth signals loss.
- **Cwnd** — the TCP congestion window; a small Cwnd on a fast/high-latency link is the
  classic "single stream can't fill the pipe" symptom.
- For UDP, the receiver line adds **Jitter** and **Lost/Total Datagrams** with a loss
  percentage — that's the data you ran the UDP test to get.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `unable to connect to server: Connection refused` | Server not running or wrong port | Start `iperf3 -s`, match `-p` on both ends |
| `Connection refused` only sometimes | Server exited after a `-1` one-off test | Restart server, or drop `-1` |
| Throughput far below link speed, `-P 1` | TCP window too small for the BDP | Add `-P`, raise `-w`, tune `tcp_rmem`/`tcp_wmem` |
| Throughput fine one way, poor the other | Asymmetric policing / duplex mismatch | Test with `-R` and `--bidir`, check switch port config |
| High CPU, capped throughput | Single core saturated at high speed | Add streams `-P`, check NIC offloads with `ethtool -k` |
| UDP shows huge loss at high `-b` | You exceeded the path's capacity (expected) | Lower `-b` to find the clean threshold |
| Results vary wildly run to run | CPU scheduling / background traffic | Pin with `-A`, run longer `-t`, quiesce the host |
| `error - the server is busy running a test` | Another client is connected (iperf3 is single-test) | Wait, or run multiple servers on different ports |
| Blocked by firewall | Port closed on server | Open TCP (and UDP for `-u`) on the `-p` port |

---

### Sources

- [iperf3 documentation (ESnet / iperf.fr)](https://iperf.fr/iperf-doc.php)
- [iperf3 manual page (software.es.net)](https://software.es.net/iperf/invoking.html)
- [iperf3 FAQ — tuning and known issues](https://software.es.net/iperf/faq.html)
