# Test Network Speed Between Two Hosts

Network performance is not one number. Use controlled tests to measure TCP or UDP throughput, application-transfer tests to reproduce user traffic, and latency tools to measure round-trip time, jitter, packet loss, and route behavior.

> **Operational warning:** Throughput tests intentionally generate heavy traffic and can saturate links, increase latency, and affect production workloads. Agree on the test window, rate-limit UDP tests, restrict temporary listeners with a firewall, and stop them when testing is complete.

## Choose the Right Tool

| Tool | Measures best | Dedicated test server? | Important limitation |
|------|---------------|------------------------|----------------------|
| `iperf3` | Controlled TCP/UDP throughput, retransmits, jitter, and loss | Yes | Synthetic traffic; does not include disk or application overhead |
| `netperf` | Bulk throughput and request/response transaction rate | Yes | `TCP_RR` reports transactions per second, not pure network latency |
| `curl` / `wget` | HTTP transfer speed and request timing | No, but an HTTP endpoint is required | Includes server, TLS, proxy, cache, and storage effects |
| `scp` | Real SSH file-transfer performance | No, but SSH access is required | Includes encryption, CPU, and disk overhead |
| `nc` / `dd` | Minimal raw TCP stream throughput | A temporary listener | Few diagnostics; implementation-specific options |
| `ping` | ICMP round-trip time and loss | No dedicated server | ICMP may be filtered or deprioritized |
| `mtr` | Route changes, per-hop RTT, and apparent loss | No dedicated server | Intermediate-hop loss may only be control-plane rate limiting |
| Speedtest CLI | Performance to a public Internet test server | Automatically selected public server | Does not isolate the private path between two chosen hosts |

For a detailed command reference, see the [iperf3 Cheatsheet](articles/iperf3-cheatsheet.md) and [Diagnosing Packet Loss with mtr](articles/mtr-packet-loss-guide.md).

## Before Testing

Record the endpoints, direction, route, link state, and expected capacity. A result cannot exceed the slowest link in the complete path.

```bash
# Route and source address selected for the destination
ip route get 192.0.2.20

# Interface counters before the test
ip -s link show dev eth0

# Negotiated Ethernet speed, duplex, and link state
sudo ethtool eth0

# CPU usage can become the bottleneck on fast links
mpstat 1
```

Replace `192.0.2.20` and `eth0` with the remote host and local interface. For Wi-Fi, also record signal strength, channel use, and whether either endpoint is roaming.

Use consistent conditions:

1. Test host A to host B and then host B to host A.
2. Run each test for at least 20–30 seconds; very short tests overemphasize startup behavior.
3. Repeat tests and compare the median rather than selecting the fastest result.
4. Start with one TCP stream, then use parallel streams to determine whether a single flow is limited.
5. Watch CPU, interface errors, retransmissions, and packet loss during the test.
6. Document VPNs, tunnels, firewalls, proxies, QoS, Wi-Fi, and concurrent traffic.

## iperf3: Controlled Throughput Testing

`iperf3` is the preferred starting point for a controlled LAN, WAN, VPN, or cloud-network test. It generates traffic in memory, minimizing disk and application effects. It requires `iperf3` at both ends and is not compatible with `iperf2`.

### Install iperf3

```bash
# Debian and Ubuntu
sudo apt install iperf3

# RHEL, Rocky Linux, AlmaLinux, and Fedora
sudo dnf install iperf3

# macOS
brew install iperf3
```

### Basic TCP test

On the remote host:

```bash
# Listen on TCP port 5201
iperf3 -s
```

On the local host:

```bash
# Local client sends to the remote server for 30 seconds
iperf3 -c 192.0.2.20 -t 30
```

The default direction is client to server. The receiver's final throughput is normally the most useful end-to-end result.

### Test both directions

```bash
# Remote server sends back to the local client
iperf3 -c 192.0.2.20 -R -t 30

# Both directions simultaneously; requires a version supporting --bidir
iperf3 -c 192.0.2.20 --bidir -t 30
```

Run normal and reverse tests sequentially first. A simultaneous bidirectional test makes both directions compete for endpoint and network resources, which is useful but answers a different question.

### Use multiple TCP streams

```bash
iperf3 -c 192.0.2.20 -P 4 -t 30
```

Parallel streams can fill a high-bandwidth or high-latency path that one TCP flow cannot saturate. Report both single-stream and parallel results; `-P 4` can hide a single-flow window, loss, or CPU limitation.

### Test UDP loss and jitter

```bash
# Offer 100 Mbit/s first
iperf3 -c 192.0.2.20 -u -b 100M -t 30

# Offer 1 Gbit/s only when the path is expected to handle it
iperf3 -c 192.0.2.20 -u -b 1G -t 30

# Test the reverse UDP direction
iperf3 -c 192.0.2.20 -u -b 100M -R -t 30
```

`-b` is the **offered rate**, not proof that the network delivered that rate. Increase it gradually and examine receiver throughput, jitter, and lost datagrams. Sending `-b 1G` across a slower path intentionally overloads it and can disrupt other traffic.

### Save machine-readable results

```bash
iperf3 -c 192.0.2.20 -t 30 --json > iperf3-result.json
```

The official [ESnet iperf3 documentation](https://software.es.net/iperf/invoking.html) describes version-specific options and output fields.

### Firewall and listener safety

An `iperf3` server listens on TCP port `5201` by default. UDP tests also use UDP `5201` while retaining the TCP control connection. If a firewall change is needed, permit only the test client's source address and remove the rule afterward.

```bash
# Bind the server to a specific local address
iperf3 -s -B 192.0.2.20

# Use the same non-default port at both ends
iperf3 -s -p 5202
iperf3 -c 192.0.2.20 -p 5202
```

`iperf3` test traffic is not a substitute for an encrypted file transfer. Do not expose an unauthenticated test listener broadly or leave it running unnecessarily.

## netperf: Throughput and Request/Response Tests

`netperf` is useful when both bulk transfer and small request/response transaction behavior matter. Package availability varies by distribution.

On the remote host:

```bash
netserver
```

On the local host:

```bash
# Client-to-server TCP bulk throughput for 30 seconds
netperf -H 192.0.2.20 -t TCP_STREAM -l 30

# Server-to-client TCP bulk throughput
netperf -H 192.0.2.20 -t TCP_MAERTS -l 30

# Sequential TCP request/response transaction rate
netperf -H 192.0.2.20 -t TCP_RR -l 30
```

`TCP_STREAM` measures bulk throughput. `TCP_RR` reports completed transactions per second for sequential request/response exchanges; it reflects network RTT plus host and socket-processing time. Use `ping` or application timings when the requirement is stated directly in milliseconds.

The default `netserver` control port is TCP `12865`, and test connections may require additional firewall handling. Restrict access to trusted source addresses. Consult the [netperf project and manual](https://github.com/HewlettPackard/netperf) for test-specific options.

## HTTP Download Tests with curl and wget

HTTP tests need no dedicated benchmark daemon, but the destination must already serve a sufficiently large file. They measure the whole HTTP path: name resolution, connection setup, TLS, proxies, caches, server performance, and transfer throughput.

### Measure with curl

```bash
curl --fail --location --output /dev/null \
  --write-out 'HTTP %{http_code}\nDNS %{time_namelookup}s\nConnect %{time_connect}s\nTLS %{time_appconnect}s\nFirst byte %{time_starttransfer}s\nTotal %{time_total}s\nDownload %{speed_download} bytes/s\n' \
  http://192.0.2.20/largefile.bin
```

For HTTPS, `time_appconnect` includes the TLS handshake. `speed_download` is bytes per second, while tools such as `iperf3` commonly report bits per second.

### Measure with wget

```bash
wget --output-document=/dev/null http://192.0.2.20/largefile.bin
```

`wget` prints transfer rate and elapsed time in its progress output.

### Avoid misleading HTTP results

- Use a file large enough for the test to reach steady state.
- Confirm whether the file came from an application, reverse proxy, CDN, or cache.
- A cache-busting query changes the cache key but does not guarantee an origin fetch.
- `/dev/null` removes local destination-disk writes but not remote storage or server overhead.
- Do not use `--insecure` or `-k`; certificate verification is part of a realistic HTTPS test.
- Test from the same client network and through the same proxy or load balancer as the real workload.

See the [curl Cheatsheet](articles/curl-cheatsheet.md) for detailed timing, proxy, and cache diagnostics.

## SCP: Real SSH Transfer Performance

An SCP test includes SSH encryption, endpoint CPU, source reads, destination writes, and network transfer. This makes it representative of an actual SSH copy but unsuitable for isolating the network alone.

Create incompressible test data before timing the transfer:

```bash
# Create a 1 GiB file; use a smaller size when storage is limited
dd if=/dev/urandom of=/tmp/network-test-1g.bin bs=1M count=1024 status=progress

# Disable SSH compression so compressibility does not distort the result
time scp -o Compression=no /tmp/network-test-1g.bin user@192.0.2.20:/tmp/
```

After confirming the transfer is no longer needed, remove only the known test files:

```bash
rm /tmp/network-test-1g.bin
ssh user@192.0.2.20 'rm /tmp/network-test-1g.bin'
```

If SCP is much slower than `iperf3`, watch CPU usage, compare SSH ciphers, and test source and destination storage. Do not disable encryption merely to make the result look faster. See the [SSH Cheatsheet](articles/ssh-cheatsheet.md) for connection diagnostics.

## netcat and dd: Minimal Raw TCP Test

`nc` can create a simple unencrypted TCP stream. It may need to be installed, and its options differ among OpenBSD, traditional, GNU, and BusyBox implementations. Check `nc -h` on both hosts first.

On the receiver:

```bash
# OpenBSD netcat syntax
nc -l 9000 > /dev/null

# Some traditional implementations require -p
nc -l -p 9000 > /dev/null
```

On the sender, `pv` provides a live byte rate:

```bash
dd if=/dev/zero bs=1M count=1024 status=none | pv -rab | nc -N 192.0.2.20 9000
```

If `pv` is unavailable, time the pipeline:

```bash
time sh -c 'dd if=/dev/zero bs=1M count=1024 status=none | nc -N 192.0.2.20 9000'
```

`-N` tells OpenBSD netcat to shut down the socket after standard input reaches EOF. Other versions may use `-q 0` instead or close automatically. If the command hangs after sending all bytes, check the local netcat help rather than mixing incompatible options.

This test avoids destination-disk writes, but memory copying, `dd`, and endpoint CPU remain in the result. It provides less protocol detail than `iperf3` and should not replace it for repeatable benchmarking.

> **Security warning:** `nc` provides no authentication or encryption. Bind or firewall the listener to the intended peer, send no sensitive data, and close it immediately after the test.

## Latency, Loss, and Route Tests

Throughput can be low even when latency looks normal, and latency can become poor only while a link is saturated. Measure both idle and loaded conditions.

### ping: Round-trip time and endpoint loss

```bash
ping -c 20 192.0.2.20
```

Review minimum, average, maximum, and variability in RTT plus endpoint packet loss. ICMP handling can differ from application traffic, so a successful ping does not prove that TCP or UDP application ports work.

### mtr: Continuous route diagnostics

```bash
# 100-probe report with hostnames
mtr --report --wide --report-cycles 100 192.0.2.20

# Avoid DNS lookup delays
mtr --report --wide --no-dns --report-cycles 100 192.0.2.20
```

Loss shown at an intermediate hop is meaningful only when it continues through later hops or reaches the destination. Routers often rate-limit replies addressed to themselves while forwarding transit traffic normally.

### traceroute: Path and per-hop response timing

```bash
traceroute 192.0.2.20
traceroute -n 192.0.2.20
```

`traceroute` helps identify path changes and the approximate point where delay appears. Its hop timings are not a bandwidth measurement, and missing hops may simply block or rate-limit probe responses.

## Speedtest CLI: Public Internet Performance

Speedtest measures the path to a public test server, not directly between two arbitrary private hosts. Server selection, peering, congestion, and ISP shaping can all affect the result.

Two different tools are commonly confused:

- Ookla's official native client uses the `speedtest` command. Installation instructions are available on the [official Speedtest CLI page](https://www.speedtest.net/apps/cli).
- The community Python package commonly provided by Linux distributions uses `speedtest-cli`.

### Community speedtest-cli

```bash
# Debian and Ubuntu package
sudo apt install speedtest-cli

# Run with automatic server selection
speedtest-cli

# List nearby servers
speedtest-cli --list

# Use the numeric ID returned by --list
speedtest-cli --server <server-id>

# Concise output
speedtest-cli --simple
```

### Official Ookla client

```bash
# Run after installing from the official repository
speedtest

# List available servers
speedtest --servers

# Select a server
speedtest --server-id <server-id>
```

Do not compare results from different client implementations as if they were identical. Use the same client, server ID, endpoint, and time window for trend comparisons. Public tests also transfer significant data and disclose the test client's public IP to the service.

## Reading and Comparing Results

### Units

| Unit | Meaning |
|------|---------|
| `Mbit/s`, `Gbit/s` | Megabits or gigabits per second; common network-rate units |
| `MB/s`, `GB/s` | Megabytes or gigabytes per second; common file-transfer units |
| `ms` | Milliseconds of delay |
| `% loss` | Share of test datagrams or probes not received |
| `retransmits` | TCP segments resent after inferred loss or reordering |
| `jitter` | Variation in packet delay, normally reported for UDP tests |

Eight bits equal one byte, so divide bit/s by eight for the theoretical byte/s rate. Protocol headers, acknowledgements, encryption, and other overhead make application throughput lower than that theoretical value.

### Common patterns

| Result | Likely explanation | Next check |
|--------|--------------------|------------|
| Low single-stream TCP, good parallel TCP | TCP window, latency, loss, or per-flow shaping | RTT, retransmits, socket buffers, and one-flow requirements |
| Good `iperf3`, slow SCP | SSH CPU, cipher, or disk bottleneck | CPU and disk throughput at both endpoints |
| Good LAN `iperf3`, slow Speedtest | ISP, peering, WAN congestion, or public server | Repeat with a fixed nearby server and another time window |
| UDP loss rises above one offered rate | Path capacity or policer threshold exceeded | Lower `-b` and find the highest clean rate |
| High retransmits with low TCP throughput | Congestion, physical errors, Wi-Fi loss, or MTU issue | Interface counters, `mtr`, packet capture, and MTU |
| High latency only during throughput tests | Queueing or bufferbloat | QoS/SQM, queue depth, and tests below saturation |
| Different forward and reverse rates | Asymmetric path, shaping, Wi-Fi, or endpoint differences | Sequential `iperf3` normal and `-R` tests |

## Repeatable Test Sequence

Use this order to separate link, transport, and application effects:

```bash
# 1. Baseline latency and endpoint loss
ping -c 20 192.0.2.20

# 2. Controlled single-stream TCP throughput
iperf3 -c 192.0.2.20 -t 30

# 3. Reverse direction
iperf3 -c 192.0.2.20 -R -t 30

# 4. Parallel-stream ceiling
iperf3 -c 192.0.2.20 -P 4 -t 30

# 5. Rate-limited UDP jitter and loss
iperf3 -c 192.0.2.20 -u -b 100M -t 30

# 6. Route and loss localization
mtr --report --wide --no-dns --report-cycles 100 192.0.2.20
```

Then run the application-specific test—HTTP, SCP, database, backup, or another real workload—and compare it with the controlled baseline.

## Quick Reference

```bash
# iperf3 server and client
iperf3 -s
iperf3 -c 192.0.2.20 -t 30

# Reverse, simultaneous bidirectional, and parallel TCP
iperf3 -c 192.0.2.20 -R -t 30
iperf3 -c 192.0.2.20 --bidir -t 30
iperf3 -c 192.0.2.20 -P 4 -t 30

# UDP offered-load test
iperf3 -c 192.0.2.20 -u -b 1G -t 30

# netperf bulk throughput and request/response rate
netserver
netperf -H 192.0.2.20 -t TCP_STREAM -l 30
netperf -H 192.0.2.20 -t TCP_RR -l 30

# HTTP download to /dev/null
curl --fail --location --output /dev/null http://192.0.2.20/largefile.bin
wget --output-document=/dev/null http://192.0.2.20/largefile.bin

# Basic latency, route, and public Internet tests
ping -c 20 192.0.2.20
mtr --report --wide --report-cycles 100 192.0.2.20
traceroute -n 192.0.2.20
speedtest-cli
```

Content derived from the linked external documentation was rephrased for compliance with licensing restrictions.
