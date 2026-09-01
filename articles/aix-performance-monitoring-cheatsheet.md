# AIX Performance Monitoring Cheatsheet

Command reference for performance analysis on IBM AIX — understanding process memory metrics (SIZE/RSS/TRS, virtual vs physical), the `topas`/`nmon` interactive monitors, disk I/O tools (`iostat`, `sar -d`, `lvmstat`), CPU and process tools (`ps`, `mpstat`, `svmon`, `truss`, `fuser`, `procfiles`), and the `topasrec`/`topasout` recording features.

> Most monitoring commands run as an ordinary user, but enabling stats collection (`chdev -l sys0 -a iostat=true`), tuning (`schedo`), and tracing other users' processes (`truss -p`, `svmon -P`) require `root`. Tuning changes such as `schedo` affect the whole system — test outside production first.

## Memory Concepts

A process's memory divides into two kinds of data:

- **User data** — variables, dynamically allocated memory, function parameters and return values. Unique to each process.
- **Program data** (also called *Text*) — the program itself and shared libraries. Loaded once and linked as a shared library into many processes.

**Virtual Memory = Physical Memory + memory in paging space.**

| Metric | Meaning |
|--------|---------|
| `SIZE` | Virtual memory used by the process's **user data** |
| `RSS` | Physical memory used by **both** user data and program data |
| `TRS` | Physical memory used by **program data** (the shared part) |
| `RSS - TRS` | Physical memory used by **user data** |
| `SIZE - RSS` | Size of paging space used by user data |

```sh
# Physically installed memory in 4K pages (see the "size" field)
svmon -G

# Active virtual memory (avm) in 4K pages
vmstat
```

## topas — Interactive Monitor

`topas` is the interactive system monitor. Runtime keys toggle the sections shown:

| Key | Action |
|-----|--------|
| `a` | Return to the default topas screen |
| `c` | Toggle CPU section: graph → off → top CPU list |
| `C` | Cross-LPAR display (same as starting with `-C`) |
| `d` | Toggle Disk section: top disks → off → summary |
| `D` | Disk statistics display (same as `-D`) |
| `f` | Toggle file statistics: summary → top-3 filesystems → off |
| `h` | Help screen (lists additional runtime keys) |
| `p` | Toggle the top-process section on/off |
| `P` | Full process view (same as `-P`) |
| `n` | Toggle network: top interfaces → summary → off |
| `L` | LPAR view (same as `-L`) |
| `q` | Quit topas |

```sh
# Disk I/O statistics view
topas -D

# Make topas look like top (process view)
topas -P

# Monitor network throughput for all interfaces
topas -E

# CEC mode — statistics from other partitions
topas -C

# LPAR view (entitlement, physc, %entc, shared-pool usage)
topas -L
```

### Monitoring Multiple Shared Processor Pools

Multiple Shared Processor Pools are supported on POWER6 and POWER7. To monitor them:

1. Run `topas`, then press **`p`** to toggle on pool metrics. You'll see the default pool (pool id `0`) plus any user-defined pools.
2. The default display shows the LPARs in pool `0`.
3. Use the up/down arrow keys to place the cursor on another pool id in the left-hand **pool** column, then press **`f`** to toggle focus — the lower portion updates to show the LPAR associated with that pool.

## Disk Monitoring

### nmon

In `nmon`, pressing **`D`** cycles through four disk screens:

1. Each disk's throughput
2. Disk size, number of paths, and the connected adapter
3. Detailed per-disk statistics
4. Disk throughput statistics with a graphical throughput indicator

### iostat

```sh
# Watch the queue on a specific hdisk
iostat -D hdiskx 1 | awk '/queue/{print; getline;print $0}'

# Verbose disk stats for hdisk0 every 2 seconds
iostat -D hdisk0 2

# I/O stats by file system
iostat -F 2
```

Key `iostat -D` queue fields:

| Field | Meaning |
|-------|---------|
| `avgwqsz` | Average wait queue size |
| `avgsqsz` | Average service queue size |
| `avgtime` | Average time spent in the wait queue (ms) |
| `sqfull` | Rate of I/Os submitted to a full queue per second — a high value indicates `queue_depth` should be increased |

### sar -d

```sh
# Per-device disk activity
sar -d
```

| Field | Meaning |
|-------|---------|
| `avwait` | Average time spent in the wait queue (ms). `sar` computes it as total response time minus total service time, averaged over completed requests |
| `avserv` | Average I/O service time (ms) — total time the disk was busy processing at least one request, divided by completed requests |
| `avque` | Average number of I/Os in the wait queue (stochastic average of queue length sampled just before each I/O completes, including queued and in-service requests) |

Example interpretation: with 120 ms total response time, 80 ms service time, and 6 completed I/Os → average wait = (120−80)/6 ≈ 6.67 ms, average service = 80/6 ≈ 13.33 ms.

### lvmstat

```sh
# Report statistics for all LVs in a VG
lvmstat -v rootvg

# Enable advanced statistics gathering on a VG (-d to disable)
lvmstat -v rootvg -e

# Snapshot LVM info every second for 10 intervals
lvmstat -v rootvg 1 10

# Clear the counters for a VG
lvmstat -v rootvg -C

# Report statistics for a specific LV (e.g. hd6 paging)
lvmstat -l hd6
```

| Field | Meaning |
|-------|---------|
| `iocnt` | Number of read and write requests |
| `Kb_read` | Total data read during the interval (KB) |
| `Kb_wrtn` | Total data written during the interval (KB) |
| `Kbps` | Data transferred per second (KB/s) |

## CPU, Memory, and Process Tools

### ps

```sh
# Processes as a pstree-style output
ps -T1

# All processes, verbose columns, sorted by process group
ps avg

# Processes not connected to a terminal (e.g. services)
ps -t -

# All processes for a user
ps -fu patagt

# Process priorities
ps -el

# Process priorities including kernel processes
ps -ekl

# Processes with their threads' priorities
ps -kmo THREAD
```

### Process trees and working directories

```sh
# Process tree of a specific PID
proctree <pid>

# Print the current working directory of a process
procwdx 21318

# Summarize current system activity (logged-in users)
who -l
```

### CPU statistics and tuning

```sh
# Running CPU stats every 5 seconds
mpstat 5

# Per-CPU utilization (one section per logical CPU)
sar -P ALL 2 5

# Workload Manager class resource usage (when WLM is active)
wlmstat 5

# Change the time slice of one clock tick to 15 ms
schedo -o 15

# Network statistics for interfaces every 2 seconds
netstat 2
```

### vmstat / paging

```sh
# Recommended vmstat form (I/O, wide, timestamped, 2s interval)
vmstat -Iwt 2

# Paging space statistics
pstat -s

# Paging system variables
pstat -T
```

### svmon — memory segments and paging

```sh
# All memory segments for a running PID
svmon -P 274676

# Display paging in use, refresh every 2s
svmon -i 2

# Track a PID for a memory leak (5s interval, 5 samples)
svmon -P 1716352 -i 5 5 > ./mem.out

# Paging-space summaries
svmon -P -O summary=basic,unit=MB
svmon -P -O sortseg=pgsp | more
svmon -P -O sortseg=pgsp,unit=MB | sort -nrk 5 | head -20
```

### truss — system call tracing

```sh
# Trace open() calls for a command
truss -t open ls

# Trace open() calls for a running PID
truss -t open -p 274676

# Trace all system calls to a file
truss -e -o truss.out who

# Count system calls for a running process
truss -c -p <pid>
```

### fuser / procfiles — files and open handles

```sh
# Identify deleted files still held open by a process
fuser -V -d /tmp
ps -fp 512238

# Show which processes use a file system
fuser -cux /var

# Kill those processes
fuser -cuxk /var

# All open files for a PID
procfiles -n 274676

# All processes and the files they have open
procfiles -n `ls /proc`
```

### I/O stats collection

```sh
# Show current sys0 iostat setting
lsattr -E -l sys0 -a iostat

# Enable disk I/O history (immediate and persistent)
chdev -l sys0 -a iostat=true
```

## Handy One-Liners

```sh
# Generate real-time disk I/O
find / > /dev/null 2>&1 &

# Generate CPU load
dd if=/dev/random of=/dev/null

# Sort processes by resident memory (RSS)
ps aux | head -1 ; ps aux | sort -rn +3 -k 6,6 | head

# Find the 10 largest files under the current directory
find . -type f | xargs ls -ls | sort -rn | head
find . -type f | while read X; do du -sm "$X"; done | sort -rn | head

# Count Oracle client connections
ps -ef | grep -c LOCAL=NO
```

## Topas Recording

`topasrec` records performance data in binary format and can capture CEC-wide and cluster-wide metrics. By default output goes to `/etc/perf/daily`; when set up via `smitty topas`, an entry is added to `/etc/inittab`. `topasout` post-processes those recordings.

```sh
# Generate a report for the nmon analyzer from a recording
topasout -a prdfsl51_121017.topas
topasout -a /etc/perf/daily/silver8_090304.topas
```

| Item | Notes |
|------|-------|
| `topasrec` | Command that records performance data (binary), CEC/cluster-wide |
| `topasout` | Post-processes recordings into reports (e.g. for nmon analyzer) |
| `/usr/lpp/perfagent/daily.cf` | Configuration file |
| `xmdaily` | The identifier of the `topasrec` entry in `/etc/inittab` — not a command |
| `xmquery` | The defined name for the `xmtopas` daemon/service within `inetd` — not a command |

### xmtopas / remote monitoring

```sh
# Inspect inetd (xmquery is registered here)
lssrc -ls inetd

# Query the xmtopas service
xmquery /usr/bin/xmtopas xmtopas -p3 active

# Cross-partition (CEC) view
topas -C
```

`/etc/perf/Rsi.hosts` lists the hosts that a topas CEC/recording setup collects from.

## Quick Reference

| Task | Command |
|------|---------|
| Installed memory (4K pages) | `svmon -G` |
| Active virtual memory | `vmstat` |
| Disk I/O view | `topas -D` |
| Process view | `topas -P` |
| Network throughput | `topas -E` |
| Cross-LPAR (CEC) view | `topas -C` |
| Watch hdisk queue | `iostat -D hdiskx 1` |
| Disk activity fields | `sar -d` |
| LV statistics | `lvmstat -v rootvg` |
| CPU stats | `mpstat 5` |
| Recommended vmstat | `vmstat -Iwt 2` |
| Memory segments for PID | `svmon -P <pid>` |
| Trace open() for PID | `truss -t open -p <pid>` |
| Files open on a filesystem | `fuser -cux /var` |
| Open files for a PID | `procfiles -n <pid>` |
| Enable iostat history | `chdev -l sys0 -a iostat=true` |
| Record performance data | `topasrec` |
| Report from a recording | `topasout -a <file>.topas` |

## Related

- [AIX Paging Space Cheatsheet](articles/aix-paging-space-cheatsheet.md) — managing swap when `svmon`/`vmstat` show paging pressure.
- [AIX MPIO and Fibre Channel Cheatsheet](articles/aix-mpio-fibre-channel-cheatsheet.md) — `queue_depth` and path tuning behind the disk I/O metrics here.
- [AIX Error Logging and System Logs Cheatsheet](articles/aix-error-logging-cheatsheet.md) — correlating performance symptoms with logged hardware/software errors.
