# blktrace Guide

`blktrace` is a block layer I/O tracing tool for Linux. It captures detailed information about I/O requests as they travel through the block layer, helping diagnose performance issues, identify bottlenecks, and understand application I/O patterns.

## Overview

blktrace traces I/O events at the block device level — below the filesystem but above the device driver. It captures every request from submission to completion, showing exactly what the kernel does with each I/O operation.

| Component | Purpose |
|-----------|---------|
| `blktrace` | Capture block I/O events from the kernel |
| `blkparse` | Parse and display blktrace output |
| `btrace` | Shorthand for `blktrace -d /dev/sdX -o - \| blkparse -i -` |
| `btt` | Analyze blktrace data for latency breakdown |
| `blkiomon` | Monitor block I/O latency in real time |
| `iowatcher` | Generate graphs from blktrace data |

## Installation

### RHEL / CentOS / Rocky / AlmaLinux

```bash
# RHEL 7
yum install blktrace

# RHEL 8+
dnf install blktrace
```

### Debian / Ubuntu

```bash
apt install blktrace
```

### Verify Installation

```bash
which blktrace
blktrace -V
```

## Prerequisites

blktrace uses the kernel's `debugfs` interface. It must be mounted:

```bash
# Check if debugfs is mounted
mount | grep debugfs

# Mount if not present
mount -t debugfs debugfs /sys/kernel/debug

# Or add to /etc/fstab for persistence
# debugfs  /sys/kernel/debug  debugfs  defaults  0 0
```

blktrace requires root privileges.

## Basic Usage

### Quick Trace with btrace

`btrace` is the simplest way to get a live trace:

```bash
# Live trace on /dev/sda
btrace /dev/sda

# Trace for 10 seconds
timeout 10 btrace /dev/sda

# Trace a specific partition
btrace /dev/sda1
```

### Trace Without Options

Running `blktrace` with just the device generates trace files in the current directory:

```bash
# Generates sda.blktrace.0, sda.blktrace.1, ... (one per CPU)
blktrace /dev/sda
# Press Ctrl+C to stop

# Then parse the output
blkparse sda.blktrace.0
```

### Capture and Analyze Separately

```bash
# Step 1: Capture trace data (Ctrl+C to stop)
blktrace -d /dev/sda -o trace

# Step 2: Parse the captured data (also generates binary for btt)
blkparse -i trace -d trace.bin -o trace.txt

# Step 3: Analyze with btt
btt -i trace.bin > analysis.txt
```

### Capture for a Fixed Duration

```bash
# Trace for 30 seconds
blktrace -d /dev/sda -w 30 -o trace

# Trace for 60 seconds with output directory
blktrace -d /dev/sda -w 60 -D /tmp/traces -o mytrace
```

## Understanding blkparse Output

### Output Format

Each line of `blkparse` output looks like:

```
8,0    3        1     0.000000000   697  A   W 223490 + 8 <- (8,1) 223488
8,0    3        2     0.000001000   697  Q   W 223490 + 8 [dd]
8,0    3        3     0.000007000   697  G   W 223490 + 8 [dd]
8,0    3        4     0.000009000   697  I   W 223490 + 8 [dd]
8,0    3        5     0.000012000   697  D   W 223490 + 8 [dd]
8,0    3        6     0.000350000   697  C   W 223490 + 8 [0]
```

### Field Breakdown

| Field | Description |
|-------|-------------|
| `8,0` | Device major,minor number |
| `3` | CPU number |
| `1` | Sequence number |
| `0.000000000` | Timestamp (seconds.nanoseconds) |
| `697` | PID of the process |
| `A` | Action code (event type) |
| `W` | RWBS description: R=Read, W=Write, D=Discard, B=Barrier, S=Sync, F=Flush |
| `223490` | Sector number |
| `+ 8` | Number of sectors (8 sectors = 4 KB) |
| `[dd]` | Process name |

### Action Codes (I/O Lifecycle)

| Code | Name | Description |
|------|------|-------------|
| `A` | Remap | I/O was remapped to a different device (e.g., DM/LVM/RAID) |
| `B` | Bounce | I/O was bounced (buffer was in high memory) |
| `C` | Complete | I/O completed (returned from hardware) |
| `D` | Dispatch | Request was sent to the device driver |
| `F` | Front merge | I/O was front merged with request on queue |
| `G` | Get request | A request structure was allocated |
| `I` | Insert | Request was inserted into the I/O scheduler queue |
| `M` | Back merge | I/O was back merged with request on queue |
| `P` | Plug | The queue was plugged (batching requests) |
| `Q` | Queue | I/O handled by request queue code |
| `S` | Sleep | No available request structures (process sleeps) |
| `T` | Unplug (timeout) | The queue was unplugged due to timeout |
| `U` | Unplug (request) | The queue was unplugged (flushing to driver) |
| `X` | Split | A BIO was split |

### I/O Request Lifecycle

```
Q → G → I → D → C
│         │
│         └── M (merged with existing request)
│
└── A (remapped from another device)
```

1. **Q** (Queue) — application submits I/O
2. **G** (Get) — request structure allocated
3. **I** (Insert) — placed in scheduler queue
4. **M** (Merge) — combined with adjacent request (or)
5. **D** (Dispatch) — sent to device driver
6. **C** (Complete) — hardware signals completion

## Common Options

### blktrace Options

| Option | Description |
|--------|-------------|
| `-d DEVICE` | Device to trace |
| `-o FILE` | Output file prefix |
| `-D DIR` | Output directory |
| `-w SECONDS` | Trace duration |
| `-a ACTION` | Filter by action mask |
| `-A MASK` | Set action mask (hex) |
| `-b SIZE` | Buffer size (default 512 KB) |
| `-n BUFFERS` | Number of buffers (default 4) |
| `-r RELAY` | Debugfs mount point path |

### blkparse Options

| Option | Description |
|--------|-------------|
| `-i FILE` | Input file prefix |
| `-o FILE` | Output file |
| `-f FORMAT` | Custom output format |
| `-d FILE` | Output binary file for btt |
| `-s` | Show per-program stats |
| `-q` | Quiet (no per-event output) |
| `-M` | Output in microseconds |

### Action Filter Masks

```bash
# Trace only writes
blktrace -d /dev/sda -a write -o trace

# Trace only reads
blktrace -d /dev/sda -a read -o trace

# Trace only completions
blktrace -d /dev/sda -a complete -o trace

# Trace issues (dispatches) and completions
blktrace -d /dev/sda -a issue -a complete -o trace

# Available masks:
# read, write, flush, sync, queue, requeue, issue,
# complete, fs, pc, notify, ahead, meta, discard, drv_data
```

## Latency Analysis with btt

`btt` (blktrace timeline) breaks down I/O latency into stages:

```bash
# Generate binary data for btt
blkparse -i trace -d trace.bin

# Run btt analysis
btt -i trace.bin

# Output to file
btt -i trace.bin -o analysis
```

### btt Output Explained

```
==================== All Coverage ====================
 DEV |       Q2Q       |       Q2C       |       D2C       |
 --- | --------------- | --------------- | --------------- |
(8,0) |   0.000500000   |   0.000350000   |   0.000200000   |
```

| Metric | Meaning |
|--------|---------|
| Q2Q | Time between consecutive I/O submissions (inter-arrival time) |
| Q2G | Queue to Get request (time to allocate request struct) |
| G2I | Get to Insert (time in scheduler queue) |
| I2D | Insert to Dispatch (scheduler latency) |
| D2C | Dispatch to Complete (device service time) |
| Q2C | Queue to Complete (total I/O latency) |

```
Total I/O latency (Q2C) = Q2G + G2I + I2D + D2C
```

- **High Q2G** — request allocation bottleneck
- **High G2I or I2D** — I/O scheduler congestion
- **High D2C** — slow device (disk, controller, or path)

## Practical Examples

### Trace a Specific Workload

```bash
# Start tracing in the background
blktrace -d /dev/sda -o workload_trace &
BLKTRACE_PID=$!

# Run your workload
dd if=/dev/zero of=/tmp/testfile bs=1M count=100 oflag=direct

# Stop tracing
kill $BLKTRACE_PID
wait $BLKTRACE_PID

# Analyze
blkparse -i workload_trace -d workload_trace.bin -o workload_trace.txt
btt -i workload_trace.bin
```

### Identify I/O Patterns

```bash
# Capture 10 seconds of I/O
blktrace -d /dev/sda -w 10 -o pattern

# Show per-process statistics
blkparse -i pattern -s

# Show I/O size distribution
blkparse -i pattern | awk '/D/{print $10}' | sort -n | uniq -c | sort -rn
```

### Detect Sequential vs Random I/O

```bash
# Capture trace
blktrace -d /dev/sda -w 10 -o seqrand

# Extract sector numbers for writes
blkparse -i seqrand | awk '/D.*W/{print $8}' > sectors.txt

# Check if sectors are sequential (diff should be constant for sequential)
awk 'NR>1{print $1-prev} {prev=$1}' sectors.txt | sort -n | uniq -c | sort -rn | head
```

### Monitor I/O Latency Distribution

```bash
# Capture trace
blktrace -d /dev/sda -w 30 -o latency

# Parse and extract D2C (device latency) values
blkparse -i latency | awk '/C/{print $4}' > completion_times.txt
```

### Trace Multiple Devices

```bash
# Trace both sda and sdb simultaneously
blktrace -d /dev/sda -d /dev/sdb -o multi

# Parse all trace files (blktrace creates per-device, per-CPU files)
blkparse -i multi -o multi_trace.txt
```

### Trace LVM/Device Mapper

```bash
# Find the underlying device
dmsetup ls
lvs -o +devices

# Trace the physical device
blktrace -d /dev/sda -w 10 -o lvm_trace

# The 'A' (remap) events show DM→physical mapping
blkparse -i lvm_trace | grep ' A '
```

## Custom Output Formats

`blkparse -f` accepts format specifiers:

| Specifier | Description |
|-----------|-------------|
| `%a` | Action (single character) |
| `%c` | CPU number |
| `%C` | Command (process name) |
| `%d` | Direction (R/W) |
| `%D` | Device major,minor |
| `%n` | Number of sectors |
| `%N` | Number of bytes |
| `%p` | PID |
| `%S` | Sector number |
| `%t` | Timestamp (seconds) |
| `%T` | Timestamp (nanoseconds portion) |
| `%5e` | Event sequence number |

```bash
# Custom format: timestamp, PID, process, action, direction, sector, size
blkparse -i trace -f "%t.%T %p %C %a %d %S + %n\n"

# Show only dispatched I/Os with size in bytes
blkparse -i trace -f "%t.%T %C %a %d sector=%S size=%N\n" | grep ' D '
```

## Real-World Scenarios

### Diagnosing Slow Database Queries

```bash
# Trace the database disk during slow query
blktrace -d /dev/sda -w 60 -o db_trace

# Identify the database process I/O
blkparse -i db_trace | grep "mysqld\|mariadbd\|postgres"

# Check if I/O is random (many different sectors) or sequential
blkparse -i db_trace | awk '/D.*mariadbd/{print $8}' | head -50
```

### Finding I/O Storms

```bash
# Quick 10-second trace
btrace /dev/sda | awk '/D/{count++} END{print count, "I/Os dispatched"}'

# Track I/O rate per second
btrace /dev/sda | awk '/D/{sec=int($4); ios[sec]++} END{for(s in ios) print s, ios[s], "IOPS"}'
```

### Comparing I/O Schedulers

```bash
# Check current scheduler
cat /sys/block/sda/queue/scheduler

# Trace with current scheduler
blktrace -d /dev/sda -w 10 -o sched_mq
blkparse -i sched_mq -d sched_mq.bin
btt -i sched_mq.bin -o sched_mq_analysis

# Change scheduler and re-test
echo "none" > /sys/block/sda/queue/scheduler
blktrace -d /dev/sda -w 10 -o sched_none
blkparse -i sched_none -d sched_none.bin
btt -i sched_none.bin -o sched_none_analysis

# Compare D2C (device latency) between both
diff sched_mq_analysis.avg sched_none_analysis.avg
```

### Verifying Direct I/O Behavior

```bash
# Start trace
blktrace -d /dev/sda -w 10 -o dio_test &

# Write with direct I/O (bypasses page cache)
dd if=/dev/zero of=/mnt/testfile bs=4k count=1000 oflag=direct

# Write with buffered I/O
dd if=/dev/zero of=/mnt/testfile2 bs=4k count=1000

# Stop and compare — direct I/O shows immediate D events,
# buffered I/O may batch writes through writeback
kill %1
blkparse -i dio_test | grep ' D ' | wc -l
```

## blkiomon — Real-Time Monitoring

`blkiomon` provides periodic I/O latency statistics:

```bash
# Monitor /dev/sda with 5-second intervals
blktrace -d /dev/sda -o - | blkiomon -I 5 -h -

# Output includes:
# - Request size histograms
# - Latency histograms (D2C)
# - IOPS and throughput
```

## iowatcher — Visualization

`iowatcher` generates SVG graphs from blktrace data:

```bash
# Install
yum install iowatcher    # RHEL 7
dnf install iowatcher    # RHEL 8+

# Capture and generate graph
blktrace -d /dev/sda -w 30 -o io_graph
iowatcher -t io_graph -o io_graph.svg

# Compare two traces
iowatcher -t trace1 -t trace2 -o comparison.svg
```

## Performance Impact

blktrace has low but non-zero overhead:

- Uses per-CPU relay channels (efficient)
- Writes trace data to disk (put output on a different device if possible)
- Each traced event is ~32 bytes in the kernel buffer

```bash
# Minimize overhead: write to tmpfs or another disk
blktrace -d /dev/sda -D /tmp/traces -o trace

# Increase buffer size for high-IOPS devices
blktrace -d /dev/nvme0n1 -b 4096 -n 8 -o trace
```

## Troubleshooting blktrace

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `BLKTRACESETUP: No such file or directory` | debugfs not mounted | `mount -t debugfs debugfs /sys/kernel/debug` |
| `BLKTRACESETUP: Invalid argument` | Device doesn't support tracing | Check device exists, try physical device instead of partition |
| Dropped events | Buffers too small for I/O rate | Increase `-b` (buffer size) and `-n` (buffer count) |
| No output | No I/O on the device | Generate I/O or check you're tracing the correct device |
| Permission denied | Not running as root | Use `sudo` |

### Check Available Devices

```bash
# List block devices
lsblk

# Check device major:minor
ls -la /dev/sda

# Check trace status via debugfs
cat /sys/kernel/debug/block/sda/msg 2>/dev/null
```

## Quick Reference

```bash
# Live trace (simplest)
btrace /dev/sda

# Timed capture + analysis
blktrace -d /dev/sda -w 10 -o trace
blkparse -i trace -d trace.bin -o trace.txt
btt -i trace.bin

# Filter only writes
blktrace -d /dev/sda -a write -w 10 -o writes

# Per-process stats
blkparse -i trace -s

# Count IOPS during trace
blkparse -i trace | grep -c ' D '

# Average I/O size
blkparse -i trace | awk '/D/{sum+=$10; count++} END{print sum/count, "sectors avg"}'

# Show I/O by process
blkparse -i trace | awk '/D/{procs[$NF]++} END{for(p in procs) print procs[p], p}' | sort -rn
```
