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
sudo iotop -t -o -p <pid> -b -n 86400 -t -qqq > iotop.out &
```

### Filter processes by I/O threshold
```bash
# Show processes with > 50MB/s I/O
while true; do sudo iotop -bot -n 1 -P -qqq | awk '/M\/s/ && ($5 > 50 || $7 > 50) {print}'; done
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
- `c` - Toggle full command line
- `f` - Change UID and PID filters
- `s` - Freeze/resume data collection
- `1`-`9` - Toggle hiding specific columns
- `0` - Show all columns
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

## Limitations
- Requires kernel with I/O accounting support (CONFIG_TASK_IO_ACCOUNTING)
- Must run as root to see all processes
- Some virtual environments may not support it
- Thread-level monitoring can be overwhelming (use -P to show processes only)

## Tips
- Combine `-o` and `-a` for best troubleshooting: `sudo iotop -oa`
- High `IO>` percentage means process is I/O bound
- Compare "Total" vs "Actual" to see cache effectiveness
- If command shows `[kernel]`, it's a kernel thread (system I/O)
- Press `a` to toggle between current and accumulated mode while running
