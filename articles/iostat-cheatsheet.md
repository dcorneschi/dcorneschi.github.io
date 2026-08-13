# iostat Cheatsheet

Quick reference for `iostat` — the tool that names the disk when `top` shows high `%iowait`.

## Installation

`iostat` ships in the **sysstat** package, which isn't on minimal installs:

```bash
# Debian / Ubuntu
apt install sysstat

# RHEL / Fedora
dnf install sysstat
```

Installing sysstat also provides `sar`, `pidstat`, `mpstat`, `nfsiostat`, and `tapestat`. Enable the `sadc` collector if you want historical archiving with `sar`.

## The Canonical Invocation

```bash
iostat -x 1 10
```

Extended stats (`-x`), sample every 1 second, ten samples. **Always ignore the first sample** — it's the average since boot and meaningless for anything live. Use `-y` to suppress it automatically, or just scroll past it.

## Common Invocations

| Command | What It Does |
|---------|--------------|
| `iostat` | Single report since boot for all CPU and devices |
| `iostat 1` | Continuous reports every 1 second |
| `iostat 1 10` | Basic stats, 1s intervals, 10 samples |
| `iostat -x 1 10` | Extended stats, 1s intervals, 10 samples |
| `iostat -xz 1` | Extended, skip zero-activity rows (`-z`), runs forever — best for watching a live system |
| `iostat -xt 1 10` | Add a timestamp before each report |
| `iostat -xm 1 10` | Throughput in MiB/s instead of KiB/s (handy on fast NVMe) |
| `iostat -xN 1 10` | Resolve device mapper names (`vg00-root` instead of `dm-0`) |
| `iostat -d -x 1` | Device stats only, skip the CPU summary |
| `iostat -c 1 10` | Only the CPU report (same numbers as `top` in a one-shot form) |
| `iostat -p sda 1 10` | Include partition-level stats for `sda` (`sda1`, `sda2`, ...) |
| `iostat -p ALL 1 10` | Every device and every partition |
| `iostat -y -x 1 10` | Skip the since-boot first report (`-y`) — equivalent to manually ignoring it |
| `iostat -o JSON -x 1 10` | Machine-readable JSON output for scripts and dashboards |
| `iostat --human -x 1 10` | Auto-scaled units in the throughput columns |
| `iostat -x sda nvme0n1 1 10` | Extended stats for specific devices only |
| `iostat -xg MyGroup sda sdb 1 10` | Group devices and show a combined summary line |

## How to Read the Output

### The Three-Step Method

1. **Ignore the first sample.** It's the average since boot. Scroll past it and read the second table.

2. **Scan `%util` and `aqu-sz` together.**
   - `%util` = percentage of wall time the device had at least one request in flight
   - `aqu-sz` = average queue depth (how many requests were stacked up)
   - Both near zero → device is idle, not your problem
   - `%util` high but `aqu-sz` well under 1 → device is doing real work but not backed up
   - `%util` high and `aqu-sz` > 1 → requests are queuing behind each other — **that's saturation**

3. **Check the latency columns — `r_await` and `w_await`.**
   - Average ms per read/write request, queue time included
   - Healthy NVMe: under 1ms
   - Healthy SSD: under 5ms
   - Healthy spinning HDD: under 20ms
   - Numbers 5-10x above those → the device is hurting

## Example Patterns

### Idle — nothing to worry about

```
Device     r/s    w/s   rkB/s   wkB/s  r_await  w_await  aqu-sz  %util
nvme0n1   0.50   3.00    8.0    24.0     0.15     0.80    0.00    0.30
```

All numbers near zero. Whatever is making the server slow, it's not disk I/O. Look at CPU, network, or application-level locks.

### Active but not saturated

```
Device     r/s     w/s    rkB/s    wkB/s  r_await  w_await  aqu-sz  %util
nvme0n1  450.00  1200.00  3600.0  9600.0    0.30     0.90    0.65   88.40
```

The drive is constantly busy (`%util` near 90%) but queue depth stays under 1 and latency is sub-millisecond. This is the typical "lying `%util`" pattern on NVMe — the device has plenty of headroom.

### Saturated — processes are stalling

```
Device     r/s     w/s    rkB/s    wkB/s  r_await  w_await  aqu-sz  %util
sda      120.00  380.00   960.0   3040.0   38.00    95.00   14.20   99.80
```

Queue depth of 14, write latency near 100ms. Requests are piling up. This correlates with high `%wa` in `top`. Next step: run `iotop -oP` to find which process is responsible.

### High iowait but all devices calm

```
avg-cpu:  %user  %nice  %system  %iowait  %steal  %idle
           5.20   0.00     1.80    25.40    0.00   67.60

Device     r/s    w/s   rkB/s   wkB/s  r_await  w_await  aqu-sz  %util
nvme0n1   2.00   8.00   16.0    64.0     0.20     0.50    0.01    0.80
dm-0      2.00   8.00   16.0    64.0     0.35     0.70    0.01    0.90
```

The CPU is reporting 25% iowait, yet no local device is busy. Likely cause: an NFS or network filesystem mount — the kernel counts those waits as iowait but they don't appear in per-device stats. Check with `nfsiostat`.

### Device mapper stacking

```
Device     r/s     w/s    rkB/s    wkB/s  r_await  w_await  aqu-sz  %util
dm-0      85.00  310.00   680.0   2480.0    0.50     3.80    1.10    4.20
nvme0n1   80.00  295.00   680.0   2480.0    0.25     1.50    0.45    3.60
```

Same throughput on both rows — `dm-0` (LVM or dm-crypt) sits on top of `nvme0n1`. The extra latency on `dm-0` is the overhead of the mapper layer (encryption, snapshot bookkeeping). Read the physical device row for hardware health; read the dm row for what your filesystem experiences.

### No merging happening

```
Device     r/s     w/s   rrqm/s  wrqm/s  rareq-sz  wareq-sz  r_await  w_await
sdb      4500.00  800.00   0.00    0.00      4.0       4.0      2.10     5.30
```

Zero merge activity and tiny 4KB request sizes — this is random I/O from a database using `O_DIRECT` or issuing `fsync` per write. The kernel can't coalesce adjacent requests. Confirm this is intentional for your workload.

### Flush latency dragging commits

```
Device     w/s    wkB/s  w_await  f/s   f_await  aqu-sz  %util
nvme0n1  650.00  5200.0    1.10  45.00   22.50    0.80   72.00
```

Write latency is fine (1.1ms) but flush latency is 22ms and climbing. Every database commit triggers a flush — this is why transaction throughput drops. The SSD's write cache is getting overwhelmed. Solutions: faster storage, a battery-backed write cache, or batching commits at the application level.

## Column Reference

### Device Identification

| Column | Meaning |
|--------|---------|
| `Device` | Block device name as it appears in `/dev` (`sda`, `nvme0n1`, `dm-0`, `dm-1`) |

### IOPS (I/O Operations Per Second)

| Column | Meaning |
|--------|---------|
| `r/s` | Read requests completed per second (after merging) |
| `w/s` | Write requests completed per second (after merging) |
| `d/s` | Discard (TRIM) requests completed per second |
| `f/s` | Flush requests completed per second |

### Throughput

| Column | Meaning |
|--------|---------|
| `rkB/s` | KiB per second read |
| `wkB/s` | KiB per second written |
| `dkB/s` | KiB per second discarded |

### Request Merging

| Column | Meaning |
|--------|---------|
| `rrqm/s` | Read requests merged per second |
| `wrqm/s` | Write requests merged per second |
| `drqm/s` | Discard requests merged per second |
| `%rrqm` | Percentage of read requests that got merged |
| `%wrqm` | Percentage of write requests that got merged |
| `%drqm` | Percentage of discard requests that got merged |

### Latency

| Column | Meaning |
|--------|---------|
| `r_await` | Average ms per read request (queue time included) |
| `w_await` | Average ms per write request (queue time included) |
| `d_await` | Average ms per discard request |
| `f_await` | Average ms per flush request |

### Request Size

| Column | Meaning |
|--------|---------|
| `rareq-sz` | Average read request size in KiB |
| `wareq-sz` | Average write request size in KiB |
| `dareq-sz` | Average discard request size in KiB |

### Saturation & Utilization

| Column | Meaning |
|--------|---------|
| `aqu-sz` | Average queue length — how many requests were waiting or in flight. Sustained > 1 means requests are queuing; > 10 means the device is buried. (Older versions: `avgqu-sz`) |
| `%util` | Percentage of wall time the device had at least one request in flight. **Lies on modern SSD/NVMe** — see warning below. |

## The %util Warning

`%util` was designed when a disk could service one request at a time (single spinning platter, one head, one seek). On that hardware, 100% meant truly saturated.

On **modern NVMe and SATA SSDs**, one device can serve dozens of requests in parallel (NVMe specs up to 64K queues x 64K commands). `%util` can show 100% while the drive is barely working — because as soon as one request lands a new one starts.

**For modern storage, trust:**
- `aqu-sz` (average queue depth)
- `r_await` / `w_await` (real latency)

`%util` is still useful on spinning disks and as an "is anything happening?" indicator, but treat it as a hint, not a verdict.

## The Diagnostic Chain

```
top (shows high %iowait)
  └─→ iostat (names the device, shows queue depth + latency)
       └─→ iotop (identifies which process is hammering the disk)

df + du → when the disk is full instead of slow
```

| Tool | Purpose |
|------|---------|
| `top` | Spots the symptom — CPU is bored, waiting on I/O (`%wa` column) |
| `iostat` | Names the device, measures the queue and latency |
| `iotop` | Per-process I/O — the `top` of the disk world |
| `df` | How full is the filesystem |
| `du` | Which directory ate the space |
| `lsblk` | See device stacking (LVM, dm-crypt, RAID layers) |

## Gotchas

| Gotcha | Details |
|--------|---------|
| First sample is since boot | Always ignore it, or pass `-y` to suppress it. Reading the first sample is the #1 mistake. |
| `%util` lies on modern storage | On NVMe and parallel SATA SSDs, 100% utilization does not mean saturated. Use `aqu-sz` and `*_await` instead. |
| May not be installed | Ships in `sysstat`, which isn't on minimal Debian/Ubuntu/Alpine installs. |
| Device mapper double-counts | I/O through LVM, dm-crypt, or RAID appears on the `dm-*`/`md*` rows AND on the physical device. Don't sum them. Use `lsblk` to see the stack. |
| `%iowait` is a CPU state, not a disk state | It only counts when the CPU was idle AND waiting on a disk request. A busy CPU with heavy I/O can show low `%iowait`. The device-level `aqu-sz` and `*_await` columns are the real indicator. |
| NFS doesn't appear | `iostat` reads `/proc/diskstats`, which only tracks local block devices. For NFS, use `nfsiostat` (also in sysstat). |
| `tps` is requests, not bytes | A `tps` of 1000 could be 1000 x 4KB random reads (4 MB/s) or 1000 x 1MB sequential reads (1 GB/s). Always cross-read with `kB/s`. |
| Column names changed between versions | The `await` → `r_await`/`w_await` split happened around sysstat v11; `avgqu-sz` was renamed to `aqu-sz`. Awk-based parsers break across distros. |

## Recipes

### Capture JSON samples for post-mortem analysis

```bash
iostat -xyo JSON 1 60 > /tmp/iostat-$(date +%Y%m%d-%H%M%S).json
```

One minute of 1-second samples in structured JSON. Parse later with `jq` or feed into a dashboard.

### Compare throughput before and after a change

```bash
iostat -xym 1 30 | awk '/^nvme0n1/ {print $4, $5}'
```

Prints just `rkB/s` and `wkB/s` for `nvme0n1` for 30 seconds — pipe to a file and diff against the baseline run.

### Monitor an LVM volume and its physical device together

```bash
iostat -x dm-0 nvme0n1 1 10
```

Watch the device mapper layer and the underlying disk side by side. If `w_await` on `dm-0` is significantly higher than on `nvme0n1`, the overhead is in the mapper layer (encryption, snapshot COW, etc.).

### Group all disks into a single system-wide IOPS summary

```bash
iostat -xg ALL_DISKS ALL 1 5
```

Reports individual devices plus a summary line labeled `ALL_DISKS` that totals IOPS and throughput across everything. Useful when you just want "is this server I/O heavy overall?"

### Correlate iowait spikes with the responsible device

```bash
iostat -xt 1 | awk '/avg-cpu/ {getline; iowait=$4} /^nvme|^sd|^dm/ {if ($NF+0 > 50) print ts, $1, "util="$NF, "aqu-sz="$(NF-1), "iowait="iowait} /^[0-9]/ {ts=$0}'
```

A one-liner that prints a line only when a device crosses 50% `%util`, showing the timestamp, device name, utilization, queue depth, and the current iowait. Handy for catching intermittent spikes in a scrolling terminal.

### Feed iostat into sar for historical collection

If you need retroactive analysis rather than real-time watching, enable `sysstat` collection instead:

```bash
# Enable the sysstat timer (systemd)
systemctl enable --now sysstat

# Review yesterday's disk stats
sar -dp -f /var/log/sysstat/sa$(date -d yesterday +%d)
```

`sar -d` reads from the same `/proc/diskstats` source as `iostat`, but sampled every 10 minutes by default and stored for 28 days.

## Scripting Notes

`iostat` is not ideal for scripting alerts — the output format has shifted between versions. For long-term archives use `sar` (same sysstat package, designed for historical storage). For JSON dashboards use `iostat -o JSON`. For per-process I/O (which `iostat` deliberately doesn't show), use `iotop` or `pidstat -d`.

## Performance Thresholds (General Guidelines)

| Metric | Good | Investigate | Critical |
|--------|------|-------------|----------|
| `%iowait` | < 5% | > 10% | > 30% |
| `%util` (HDD) | < 80% | > 90% | > 98% |
| `%util` (SSD/NVMe) | Unreliable — check `aqu-sz` instead | — | — |
| `aqu-sz` | < 2 | > 5 | > 10 |
| `r_await` (SSD) | < 5ms | > 10ms | > 50ms |
| `r_await` (HDD) | < 20ms | > 50ms | > 100ms |
| `w_await` (SSD) | < 5ms | > 10ms | > 50ms |
| `w_await` (HDD) | < 20ms | > 50ms | > 100ms |

> **Remember:** For NVMe/SSDs, `%util` approaching 100% does NOT mean saturated. Trust `aqu-sz` and `*_await` for modern storage.

## Automation Scripts

### Disk Health Check

```bash
#!/bin/bash
echo "=== Disk I/O Health Check ==="
echo "Date: $(date)"
echo ""

iostat -x 1 5 | awk '
BEGIN {print "Device", "Avg_Util%", "Avg_Await_ms", "Status"}
/^[s,h,v,n]/ {
    device=$1
    util+=$NF
    await+=$10
    count++
    if (count==5) {
        avg_util=util/5
        avg_await=await/5
        status="OK"
        if (avg_util>80 || avg_await>20) status="WARNING"
        if (avg_util>95 || avg_await>50) status="CRITICAL"
        printf "%-10s %.2f %.2f %s\n", device, avg_util, avg_await, status
        util=0; await=0; count=0
    }
}' | column -t
```

### Alert on High I/O Wait

```bash
#!/bin/bash
THRESHOLD=10
while true; do
    IOWAIT=$(iostat -c 1 2 | awk '/avg-cpu/ {getline; print $4}' | tail -1)
    if (( $(echo "$IOWAIT > $THRESHOLD" | bc -l) )); then
        echo "$(date): High I/O wait: ${IOWAIT}%"
    fi
    sleep 10
done
```

### Continuous Dashboard

```bash
#!/bin/bash
while true; do
    clear
    echo "=== I/O Statistics — $(date) ==="
    iostat -xmt 1 2 | tail -n +4
    sleep 5
done
```

### Log to File with Timestamps

```bash
# Record baseline for 10 minutes (60 samples at 10s interval)
iostat -xmt 10 60 > iostat_baseline_$(hostname)_$(date +%Y%m%d).txt &

# Peak hour monitoring
iostat -xmt 60 > iostat_peak_$(date +%Y%m%d_%H%M).log &
```

### Monitor Multiple Disks Side-by-Side

```bash
watch -n 2 'iostat -x sda sdb nvme0n1 | grep -E "^(Device|sd|nvme)"'
```

## Troubleshooting iostat Itself

| Issue | Fix |
|-------|-----|
| `command not found` | Install: `apt install sysstat` or `dnf install sysstat` |
| No stats / empty output | Enable sysstat: `systemctl enable --now sysstat` |
| Permission denied | Run with `sudo` (rarely needed — only for some kernel stats) |
| Only shows since-boot averages | Add an interval: `iostat -x 1` or use `-y` to skip first report |
| Device names are `dm-0`, `dm-1` | Use `-N` to show LVM names, or check `lsblk` for mapping |
| macOS: not available | Install via Homebrew: `brew install sysstat` (limited functionality) |

## Related Tools

| Tool | Focus | Install |
|------|-------|---------|
| `iotop` | Per-process I/O (requires root) | `apt install iotop` / `dnf install iotop` |
| `vmstat` | General system — memory, swap, basic I/O | Built-in (procps) |
| `sar -d` | Historical disk stats (same source as iostat) | sysstat package |
| `pidstat -d` | Per-process I/O (no root needed) | sysstat package |
| `dstat` | All-in-one (CPU, disk, net) in columns | `apt install dstat` / `dnf install dstat` |
| `blktrace` | Block-layer tracing (deep debugging) | `apt install blktrace` / `dnf install blktrace` |
| `nfsiostat` | NFS I/O stats (not shown by iostat) | sysstat package |

## Additional Options

| Flag | Description |
|------|-------------|
| `-j ID` | Display persistent device names (by-id) |
| `-j UUID` | Display persistent device names (by-uuid) |
| `-j PATH` | Display persistent device names (by-path) |
| `--human` | Auto-scaled human-readable units |
| `-o JSON` | Output in JSON format |
| `-g <name> <devs>` | Group devices and show combined summary |
| `-s` | Short (narrow) output format |
| `--dec={0,1,2}` | Number of decimal places to use |

## See Also

- [Understanding iostat -x Output](articles/understanding-iostat-x-output.md) — deep-dive into every field, formulas, stall patterns, and `blktrace` correlation
