# top Cheatsheet

`top` provides a real-time, dynamic view of system processes. It shows CPU usage, memory usage, load average, and per-process resource consumption.

## Launch Options

```bash
# Default (interactive)
top

# Show specific user's processes
top -u username

# Show specific PID(s)
top -p 1234
top -p 1234,5678,9012

# Batch mode (non-interactive, for scripts/logging)
top -bn1

# Batch mode with N iterations
top -bn5

# Run N iterations then exit
top -n 10

# Batch mode, specific user, sorted by memory
top -bn1 -u apache -o %MEM

# Set refresh interval (seconds)
top -d 2

# Start with threads shown
top -H

# Start with command line (not just name)
top -c

# Hide idle processes
top -i

# Secure mode (disable interactive commands)
top -s
```

## Header Explained

```
top - 14:30:01 up 5 days, 3:22,  2 users,  load average: 0.15, 0.20, 0.18
Tasks: 256 total,   1 running, 255 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.3 us,  1.0 sy,  0.0 ni, 96.5 id,  0.1 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem :  15923.4 total,   1234.5 free,   8765.4 used,   5923.5 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   6543.2 avail Mem
```

### Line 1: Uptime and Load

| Field | Meaning |
|-------|---------|
| `up 5 days, 3:22` | System uptime |
| `2 users` | Logged-in users |
| `load average: 0.15, 0.20, 0.18` | 1-min, 5-min, 15-min load average |

### Line 2: Tasks

| Field | Meaning |
|-------|---------|
| `total` | Total number of processes |
| `running` | Currently executing on CPU |
| `sleeping` | Waiting for event |
| `stopped` | Suspended (e.g., Ctrl+Z) |
| `zombie` | Completed but not reaped by parent |

### Line 3: CPU

| Field | Meaning |
|-------|---------|
| `us` | User space (non-nice) |
| `sy` | Kernel/system |
| `ni` | User space (niced/low priority) |
| `id` | Idle |
| `wa` | Waiting for I/O |
| `hi` | Hardware interrupts |
| `si` | Software interrupts |
| `st` | Stolen (virtualization — hypervisor) |

### Line 4-5: Memory

| Field | Meaning |
|-------|---------|
| `total` | Total physical RAM |
| `free` | Unused RAM |
| `used` | Used RAM |
| `buff/cache` | Buffer/page cache (reclaimable) |
| `avail Mem` | Available for new processes (free + reclaimable cache) |

## Process Columns

| Column | Meaning |
|--------|---------|
| `PID` | Process ID |
| `USER` | Process owner |
| `PR` | Priority (kernel scheduling) |
| `NI` | Nice value (-20 to 19, lower = higher priority) |
| `VIRT` | Virtual memory (total address space) |
| `RES` | Resident memory (actual RAM used) |
| `SHR` | Shared memory |
| `S` | State (R=running, S=sleeping, D=disk wait, Z=zombie, T=stopped) |
| `%CPU` | CPU usage percentage |
| `%MEM` | Memory usage percentage |
| `TIME+` | Total CPU time consumed |
| `COMMAND` | Process name or command line |

## Interactive Commands (While top is Running)

### Display and Layout

| Key | Action |
|-----|--------|
| `h` or `?` | Help screen |
| `q` | Quit |
| `d` or `s` | Change refresh interval (seconds) |
| `Space` | Force immediate refresh |
| `l` | Toggle load average line |
| `t` | Toggle task/CPU line (off → single → graph → double-graph) |
| `m` | Toggle memory line (off → numbers → graph) |
| `1` | Toggle per-CPU breakdown (show each CPU individually) |
| `I` | Toggle Irix/Solaris mode (CPU% divided by number of CPUs) |
| `H` | Toggle threads (show individual threads) |
| `c` | Toggle full command line vs. process name |
| `V` | Forest/tree view (parent-child hierarchy) |
| `J` | Justify numeric columns (left/right) |
| `x` | Highlight sort column |
| `b` | Toggle bold/highlight for running processes |
| `z` | Toggle color/mono |
| `W` | Save current settings to `~/.toprc` |

### Sorting

| Key | Sort By |
|-----|---------|
| `P` | %CPU (default) |
| `M` | %MEM |
| `N` | PID |
| `T` | TIME+ |
| `R` | Reverse sort order |
| `<` / `>` | Move sort column left/right |
| `f` | Field management (add/remove/reorder columns) |
| `o` or `O` | Filter by field value |

### Process Control

| Key | Action |
|-----|--------|
| `k` | Kill a process (prompts for PID and signal) |
| `r` | Renice a process (change priority) |
| `u` or `U` | Filter by user |
| `i` | Toggle idle processes (show only active) |
| `n` | Set number of processes to display |
| `L` | Search/locate string in COMMAND |
| `&` | Find next match |

### Common Signals (for `k` command)

| Signal | Number | Description |
|--------|--------|-------------|
| `SIGHUP` | 1 | Hangup (reload configuration) |
| `SIGINT` | 2 | Interrupt (Ctrl+C equivalent) |
| `SIGKILL` | 9 | Force kill (cannot be caught or ignored) |
| `SIGTERM` | 15 | Terminate gracefully (default) |
| `SIGCONT` | 18 | Continue if stopped |
| `SIGSTOP` | 19 | Stop process (cannot be caught) |

### Memory Display

| Key | Action |
|-----|--------|
| `e` | Change process memory scale (KiB → MiB → GiB → TiB) |
| `E` | Change header memory scale (KiB → MiB → GiB → TiB) |

## Sorting from Command Line

```bash
# Sort by memory usage
top -o %MEM

# Sort by CPU usage
top -o %CPU

# Sort by PID
top -o PID

# Sort by resident memory
top -o RES

# Sort by time
top -o TIME+

# Sort by virtual memory
top -o VIRT
```

## Filtering Processes

### Interactive Filtering

Press `o` (or `O` for case-insensitive) while top is running:

```
# Filter by user
FILTER: USER=root

# Filter by command name
FILTER: COMMAND=java

# Filter by CPU > 10%
FILTER: %CPU>10.0

# Filter by memory > 5%
FILTER: %MEM>5.0

# Filter by RES > 100 MiB (value in KiB)
FILTER: RES>100000
```

Press `=` to clear all filters.

### From Command Line

```bash
# Show only root processes
top -u root

# Show only specific PIDs
top -p $(pgrep -d, java)

# Show only processes matching a name
top -p $(pgrep -d, nginx)
```

## Batch Mode (For Scripts and Logging)

```bash
# Single snapshot (like ps but with top format)
top -bn1

# Single snapshot, sort by memory
top -bn1 -o %MEM

# Top 10 CPU consumers
top -bn1 -o %CPU | head -17

# Log every 5 seconds, 12 iterations (1 minute of data)
top -bd5 -n12 > top_output.log

# Specific user, batch mode, 3 iterations
top -bn3 -u www-data > web_processes.log

# Extract just the process list (skip header)
top -bn1 | tail -n +8

# Get top 5 memory consumers
top -bn1 -o %MEM | head -12 | tail -5

# One-liner: top CPU consumers with timestamp
echo "=== $(date) ===" && top -bn1 -o %CPU | head -15
```

## Multiple Windows / Field Groups

Press `A` to enter alternate display mode (4 field groups):

| Group | Default View |
|-------|-------------|
| 1 | Default (sorted by %CPU) |
| 2 | Job view |
| 3 | Memory view |
| 4 | User view |

Navigate between groups with `a` and `w`. Press `g` to choose a specific group.

## Field Management (f key)

Press `f` to enter field management. Use:
- Arrow keys to navigate
- `d` or `Space` to toggle field visibility
- `s` to set as sort field
- `Right arrow` to move field position
- `q` or `Esc` to exit

### Useful Additional Fields

| Field | Meaning |
|-------|---------|
| `nTH` | Number of threads |
| `P` | Last used CPU |
| `SWAP` | Swapped size |
| `nDRT` | Dirty pages |
| `WCHAN` | Wait channel (what kernel function process is sleeping in) |
| `Flags` | Process flags |
| `CGROUPS` | Control groups |
| `ENVIRON` | Environment variables |
| `nMaj` | Major page faults |
| `nMin` | Minor page faults |

## htop (Enhanced Alternative)

`htop` is a more user-friendly alternative with color, mouse support, and easier navigation:

```bash
# Install
sudo apt install htop     # Debian/Ubuntu
sudo yum install htop     # RHEL/CentOS
brew install htop         # macOS

# Launch
htop

# Filter by user
htop -u username

# Sort by memory
htop --sort-key=PERCENT_MEM
```

Key differences from top:
- Scroll horizontally and vertically
- Mouse support (click to sort, select)
- Tree view by default (F5)
- Process search (F3) and filter (F4)
- Setup menu (F2) for configuration
- Kill signal menu (F9)
- No batch mode

## Configuration File

top saves settings to `~/.toprc` (press `W` to save):

```bash
# Reset to defaults (delete config)
rm ~/.toprc

# Copy config between users
cp ~/.toprc /home/otheruser/.toprc
```

## Useful Patterns

```bash
# Monitor a specific application
top -p $(pgrep -d, -f "my-app")

# Watch for zombie processes
top -bn1 | grep -i zombie

# Check if system is CPU-bound or I/O-bound
# High wa% = I/O bound
# High us%+sy% with low id% = CPU bound
top -bn1 | grep "Cpu(s)"

# Find memory leaks (watch RES grow over time)
top -d5 -p <pid>

# Check steal time (virtualized environments)
top -bn1 | grep "Cpu(s)" | awk '{print "Steal: "$16"%"}'

# Export process list as CSV-like format
top -bn1 -o %CPU | tail -n +8 | awk '{print $1","$2","$9","$10","$12}'

# Alert on high CPU usage
top -bn1 -o %CPU | awk 'NR>7 && $9>80 {print $12, $9"%"}'
```

### Long-Running Captures

```bash
# Run top 1 hour in background (1-second interval, 3600 iterations)
nohup top -b -d 1 -n 3600 > top.out &

# Capture every 5 minutes for 48 hours (300s interval, 576 iterations)
nohup top -b -d 300 -n 576 &

# Capture every 1 minute for 48 hours (good for memory utilization history)
nohup top -b -d 60 -n 2880 &

# Capture with full command lines
top -c -b -n 1 > top_snapshot.txt

# Capture thread-level data for memory leak investigation
top -b -n 1 -H >> top.out

# Or continuous thread capture every 60 seconds
nohup top -b -d 60 -n 2880 -H > top_threads.out &
```

### Memory Utilization History

```bash
# Sort by RSS (resident memory), save config, then run in background
# Interactive: Shift+F → select RES as sort field → press 's' → press 'q'
# Then press 'W' to save to ~/.toprc

# Now batch captures will use the saved sort order
nohup top -b -d 60 -n 2880 > memory_history.out &
```

### Sort by I/O Wait State

```bash
# RHEL 6: Sort by process state showing I/O wait
# Press: O, w, Enter, R

# RHEL 7+: Sort by process state (S column)
# Press: f → navigate to "S" → press 's' to set sort → press 'q' → then 'R' to reverse
```

## Comparison: top vs htop vs atop

| Feature | top | htop | atop |
|---------|-----|------|------|
| Pre-installed | Yes | No | No |
| Color | Limited | Full | Full |
| Mouse support | No | Yes | No |
| Tree view | Yes (V) | Yes (F5) | No |
| Horizontal scroll | No | Yes | No |
| Historical data | No | No | Yes (logging) |
| Disk I/O per process | No | No | Yes |
| Network per process | No | No | Yes |
| Batch mode | Yes | No | Yes |
| Configuration | ~/.toprc | ~/.config/htop | /etc/atop |
