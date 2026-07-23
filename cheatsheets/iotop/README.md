## Overview
Interactive I/O monitoring tool that shows disk I/O usage by processes. Like `top` but for disk I/O.
**Requires root privileges** to display all processes.

Requires Python ≥ 2.7 and a Linux kernel ≥ 2.6.20 with `CONFIG_TASK_DELAY_ACCT`, `CONFIG_TASKSTATS`, `TASK_IO_ACCOUNTING` and `CONFIG_VM_EVENT_COUNTERS` enabled.

### How it works

- Reads per-process I/O stats from `/proc/<pid>/io` and uses the kernel's **taskstats netlink interface** to get delay accounting data (SWAPIN, IO%)
- Requires root because the taskstats netlink socket needs `CAP_NET_ADMIN`
- Updates every 1 second by default (configurable with `-d`), polling all `/proc/*/io` on each tick
- **Total** vs **Actual** in the header: Total includes cached writes that may not have hit disk yet; Actual reflects physical disk I/O only

### Verify I/O accounting is enabled

If `/proc/self/io` exists, I/O accounting is built into the kernel:

```bash
cat /proc/self/io
```
```
rchar: 1900
wchar: 0
syscr: 7
syscw: 0
read_bytes: 0
write_bytes: 0
cancelled_write_bytes: 0
```

### /proc/pid/io Field Descriptions

| Field | Description |
|-------|-------------|
| `rchar` | Bytes this task has caused to be read from storage |
| `wchar` | Bytes this task has caused or shall cause to be written to disk |
| `syscr` | Number of read I/O operations (read, pread syscalls) |
| `syscw` | Number of write I/O operations (write, pwrite syscalls) |
| `read_bytes` | Bytes actually fetched from the storage layer |
| `write_bytes` | Bytes actually sent to the storage layer |
| `cancelled_write_bytes` | Bytes not written due to pagecache truncation |

> **RHEL note:** Red Hat backported I/O accounting in RHEL 5.4 (kernel 2.6.18-164.el5) but it works correctly starting with RHEL 5.6 (kernel 2.6.18-238.el5). Since 2012, iotop is part of RHEL.

## Installation
```bash
# Debian/Ubuntu
sudo apt install iotop

# RHEL/CentOS/Fedora
sudo yum install iotop
sudo dnf install iotop

# Arch Linux
sudo pacman -S iotop
```

## Commands
```bash
sudo iotop                                                # Start iotop (interactive mode)
sudo iotop -o                                             # Only show processes doing I/O
sudo iotop -P                                             # Only show processes (no threads)
sudo iotop -a                                             # Accumulated mode (total since start)
sudo iotop -d 2                                           # Update every 2 seconds (default: 1s)
sudo iotop -p PID                                         # Monitor specific PID
sudo iotop -u USER                                        # Show specific user's processes
sudo iotop -b                                             # Batch mode (for logging/parsing)
sudo iotop -b -n 10                                       # Batch mode, 10 iterations
sudo iotop -k                                             # Display in kilobytes
sudo iotop -t                                             # Include timestamp
sudo iotop -q                                             # Quiet mode (only summary)
sudo iotop -qq                                            # Even quieter (only per-process)
sudo iotop -qqq                                           # Quietest (only totals)
sudo iotop -oa                                            # Active + accumulated (best for troubleshooting)
sudo iotop -oPa                                           # Active + processes only + accumulated
sudo iotop -c                                             # Show full command line
```

## Recipes

### Log single process to file
```bash
sudo iotop -tbp <pid> -qqq > iostats.log
```

### Log to file with timestamp for one minute
```bash
sudo iotop -bot -n 60 -d 1 -qqq > iotop.log
```

### Continuous logging
```bash
sudo iotop -bot -qqq > iotop_$(date +%Y%m%d_%H%M%S).log
```

### Total I/O summary every second
```bash
sudo iotop -b -d 1 -qq | grep "^Total"
```

### Monitor PID for 24 hours in background
```bash
sudo iotop -t -o -p <pid> -b -n 86400 -t -qqq > iotop_$(date +%Y%m%d_%H%M%S).log &
```

### Monitor all PIDs of a process by name
```bash
sudo iotop $(pgrep nginx | sed 's/^/-p /' | tr '\n' ' ')
```

### Top 5 processes by write I/O
```bash
sudo iotop -boP -qqq -n 1 | sort -k6 -rn | head -5
```

### Top 5 processes by read I/O
```bash
sudo iotop -boP -qqq -n 1 | sort -k4 -rn | head -5
```

### Filter processes by I/O threshold
```bash
# Show processes with > 50MB/s I/O
while true; do sudo iotop -bot -n 1 -P -qqq | awk '/M\/s/ && ($5 > 50 || $7 > 50) {print}'; done
```

### Capture I/O activity
```bash
#!/bin/bash
# capture-iotop.sh

OUTPUT="/tmp/iotop_$(date +%Y%m%d_%H%M%S).log"
DURATION=60   # seconds
INTERVAL=1    # seconds between samples

echo "Capturing iotop data to $OUTPUT for ${DURATION}s"

sudo iotop -botP -qqq -d $INTERVAL -n $DURATION > "$OUTPUT"

echo "Done. Output saved to $OUTPUT"
```

## Interactive Commands

### Navigation
- `↑/↓` or `k/j` - Move up/down
- `←/→` or `h/l` - Change sort column
- `Home/End` - Jump to first/last

### Sorting
- `r` - Reverse sort order
- `o` - Toggle showing only processes doing I/O
- `p` - Toggle showing processes vs threads
- `a` - Toggle accumulated mode
- `i` - Change priority (ionice)

### Display
- `q` - Quit
- `Space` - Force refresh
- `?` or `h` - Help

## Understanding the Display

### Header
```
Total DISK READ:       5.23 M/s | Total DISK WRITE:      10.44 M/s
Actual DISK READ:      5.23 M/s | Actual DISK WRITE:      8.12 M/s
```
- **Total** - All I/O requests (including cached)
- **Actual** - Physical disk I/O

### Columns
```
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 1234  be/4  root        0.00 B    512.00 K  0.00 %  15.23 %  [kworker/u16:2]
 5678  be/4  mysql      10.23 M     5.12 M  0.00 %  89.45 %  mysqld
```

- `TID` - Thread ID (or PID with -P)
- `PRIO` - I/O priority (be=best-effort, rt=real-time, idle)
- `USER` - Username
- `DISK READ` - Disk read rate
- `DISK WRITE` - Disk write rate
- `SWAPIN` - Percentage of time swapping in
- `IO>` - Percentage of time spent in I/O wait
- `COMMAND` - Command name

## I/O Priority Classes
- `rt` - Real-time (highest priority)
- `be` - Best-effort (normal priority, 0-7)
- `idle` - Idle (lowest priority)

## Alternative: iotop-c

Re-implementation of iotop written in C — lighter on resources, preferred for systems under heavy load:

```bash
# Installation
sudo apt install iotop-c       # Debian/Ubuntu
sudo dnf install iotop-c       # RHEL/Fedora (requires epel-release)
sudo pacman -S iotop-c         # Arch

# Usage is identical
sudo iotop-c -o
sudo iotop-c -oPa
```

> **Note:** On Ubuntu 22.04/24.04 the Python version of iotop shows `?unavailable?` for SWAPIN/IO% columns. Fix with `sudo sysctl -w kernel.task_delayacct=1`.

Enable delay accounting (required on newer kernels where it's off by default):
```bash
# Temporary (until reboot)
sudo sysctl -w kernel.task_delayacct=1

# Permanent
echo 'kernel.task_delayacct=1' | sudo tee /etc/sysctl.d/99-iotop.conf
sudo sysctl -p /etc/sysctl.d/99-iotop.conf
```

### Hide specific columns (iotop-c only)
```bash
sudo iotop-c -1                # Hide PID/TID column
sudo iotop-c -2                # Hide PRIO column
sudo iotop-c -3                # Hide USER column
sudo iotop-c -4                # Hide DISK READ column
sudo iotop-c -5                # Hide DISK WRITE column
sudo iotop-c -6                # Hide SWAPIN column
sudo iotop-c -7                # Hide IO column
sudo iotop-c -8                # Hide GRAPH column
sudo iotop-c -9                # Hide COMMAND column
```

## Comparison with Other Tools

### iotop vs iostat
- **iotop** - Per-process I/O usage
- **iostat** - Per-device I/O statistics

### iotop vs atop
- **iotop** - Real-time, interactive
- **atop** - Historical data, process accounting

### iotop vs pidstat
- **iotop** - Interactive, quick identification
- **pidstat -d** - Better for scripting, logging, and trending

### When to use what
- **iotop** - Which process is doing I/O?
- **iostat** - Is my disk saturated?

## Generate Test I/O

Useful to verify iotop is working correctly:

```bash
# Sustained write (slower, visible in iotop for longer)
dd if=/dev/zero of=/tmp/testfile bs=4k count=500000 oflag=direct

# Sustained read
dd if=/tmp/testfile of=/dev/null bs=4k iflag=direct

# Cleanup
rm -f /tmp/testfile
```

> `oflag=direct` / `iflag=direct` bypasses the page cache, forcing actual disk I/O that iotop can see.

## Running in Docker/Containers

iotop needs kernel-level access. Inside containers, use `--privileged` or add capabilities:

```bash
# Privileged mode
docker run --privileged -it ubuntu bash -c "apt install -y iotop && iotop"

# Minimal capabilities
docker run --cap-add SYS_PTRACE --cap-add NET_ADMIN --pid=host -it ubuntu bash
```

> Without these, iotop will fail with "Netlink error: Operation not permitted".

## Changing I/O Priority with ionice

Change the I/O scheduling priority of running processes:

```bash
# Set a background job to idle I/O priority (only gets I/O when nothing else needs it)
sudo ionice -c 3 -p $(pgrep backupjob)

# Set best-effort with low priority (7=lowest)
sudo ionice -c 2 -n 7 -p $(pgrep rsync)

# Set real-time priority for a database (use with caution)
sudo ionice -c 1 -n 0 -p $(pgrep postgres | head -1)

# Start a command with idle I/O priority
sudo ionice -c 3 tar czf /backup/archive.tar.gz /data/
```

## Troubleshooting Disk Saturation

### Troubleshooting workflow

```bash
# 1. Is the disk busy? (single device, header once)
iostat -dx 1 sda | awk '/^Device/ {if(!h){h=1; print}; next} /^sda/ {print}'

# 2. Which process is responsible?
sudo iotop -oP

# 3. What files does that process have open?
sudo lsof -p <PID> -Fn | grep ^n | cut -c2-

# 4. What is it actually doing? (syscalls with file paths and timing)
sudo strace -p <PID> -e trace=read,write,openat -T -y 2>&1 | head -50
```

### Alternative tools at each step

| Step | Tool | Use case |
|------|------|----------|
| Disk saturation | `iostat -dx 1` | Per-device utilization, queue depth, await |
| Disk saturation | `dstat -d` | Quick read/write throughput overview |
| Per-process I/O | `iotop -oP` | Real-time, interactive |
| Per-process I/O | `pidstat -d 1` | Better for scripting and logging |
| Open files | `lsof -p <PID>` | All open file descriptors |
| Open files | `ls -la /proc/<PID>/fd` | No lsof needed, works in minimal containers |
| Syscall tracing | `strace -p <PID> -T -y` | Per-syscall timing with resolved paths |
| Syscall tracing | `perf trace -p <PID>` | Lower overhead than strace |

## Why iotop and iostat Show Different Numbers

They measure I/O at different layers of the storage stack:

| Tool | Data source | What it measures |
|------|-------------|-----------------|
| `iotop` | `/proc/<pid>/io` | I/O requests from userspace (including page cache writes) |
| `iostat` | `/proc/diskstats` | Completed block device operations (physical disk I/O) |

### The write path

1. Process calls `write()` → data goes to **page cache** (memory) → iotop counts it immediately
2. Kernel dirty page writeback (`pdflush`/`kworker`) flushes pages to disk asynchronously
3. Physical write hits the block device → iostat counts it

The delay between step 1 and step 3 depends on:
- `vm.dirty_writeback_centisecs` — how often the kernel checks for dirty pages (default: 500 = 5s)
- `vm.dirty_ratio` — percentage of memory that can be dirty before forcing synchronous writes
- `vm.dirty_background_ratio` — threshold to start background writeback

### The read path

Reads are more aligned between iotop and iostat because:
- If data is in page cache → no disk I/O, neither iotop nor iostat shows it
- If data is not cached → physical read happens → both tools see it
- Exception: iotop shows `read_bytes` only for actual storage reads, but `rchar` includes cache hits

### Common discrepancies

| Situation | iotop | iostat |
|-----------|-------|--------|
| Process writes to page cache, kernel hasn't flushed yet | High write | Low/zero |
| Kernel writeback flushing old dirty pages | Low (no user process active) | High write |
| Process reads from page cache | Low/zero | Zero |
| `O_DIRECT` / `oflag=direct` writes | High write | High write (bypasses cache) |

### Verify with dirty page stats
```bash
# See current dirty page state
grep -E "Dirty|Writeback" /proc/meminfo

# Watch writeback in real-time
watch -n 1 'grep -E "Dirty|Writeback" /proc/meminfo'
```
- Press `a` to toggle between current and accumulated mode while running
