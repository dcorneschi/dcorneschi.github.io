# Linux Load Average Explained

Load average is one of the first metrics you check when a system feels slow, yet it's often misunderstood. On Linux specifically, it behaves differently than on other Unix systems.

## What Load Average Actually Measures

Load average represents the average number of processes in a **runnable** or **uninterruptible** state over 1, 5, and 15 minutes.

On Linux, two process states contribute:

| State | Meaning |
|-------|---------|
| `R` | Running or runnable (on the run queue, waiting for CPU) |
| `D` | Uninterruptible sleep (usually waiting on disk I/O) |

This is a critical distinction: **Linux includes I/O-blocked processes in load average**. A system with heavy disk activity can show high load even when CPUs are mostly idle.

## Checking Load Average

```bash
# Three standard ways
uptime
cat /proc/loadavg
w

# Example output of /proc/loadavg:
# 2.41 1.87 1.63 3/542 18743
#  1m   5m   15m  running/total  last_pid
```

The three numbers represent averages over 1, 5, and 15 minutes. The trend tells you whether things are getting worse or recovering.

## Interpreting the Numbers

The key comparison is **load average vs. number of logical CPUs**:

```bash
# Count logical CPUs
nproc
# or
cat /proc/cpuinfo | grep processor | wc -l
# or
lscpu | grep "^CPU(s):"
```

| Load vs CPUs | Meaning |
|---|---|
| Load < CPU count | System has capacity |
| Load = CPU count | Fully utilized, no headroom |
| Load > CPU count | Overloaded, processes are waiting |

Example: load of 4.0 on a 4-core system = full utilization. The same load on a 16-core system = 25% utilization.

## Distinguishing CPU Load from I/O Load

High load average doesn't always mean CPU saturation. Use `vmstat` to break it down:

```bash
vmstat 1 5
```

Key columns:

| Column | Meaning |
|--------|---------|
| `r` | Processes running or waiting for CPU |
| `b` | Processes in uninterruptible sleep (blocked on I/O) |

- High `r`, low `b` → CPU-bound load
- Low `r`, high `b` → I/O-bound load
- High both → mixed pressure

### Finding the culprits

```bash
# Find processes in R (running/runnable) or D (I/O blocked) state
ps -eLo state,pid,ppid,cmd | grep -e "^R" -e "^D"

# More detailed view with CPU and memory
ps aux | awk '$8 ~ /R|D/'

# Real-time view filtered to R and D states
top -b -n 1 | awk 'NR<=7 || /^ *[0-9]+ .*(R|D)/'
```

## Using sar for Historical Data

```bash
# Run queue and load average history
sar -q

# Output columns:
# runq-sz  — processes waiting for run time
# plist-sz — number of processes in the process list
# ldavg-1  — load average last 1 minute
# ldavg-5  — load average last 5 minutes
# ldavg-15 — load average last 15 minutes
```

If `sar` data isn't being collected, enable it:

```bash
# Enable sysstat collection (Debian/Ubuntu)
sudo systemctl enable sysstat
sudo systemctl start sysstat
```

## The D State Problem

The `D` (uninterruptible sleep) state is unique to Linux's load calculation. Processes enter this state when they're waiting on I/O that cannot be interrupted — typically disk operations or NFS mounts.

Common causes of high D-state process counts:

- Slow disk (failing drive, saturated SAN)
- NFS server not responding
- Heavy swap activity
- Kernel driver waiting on hardware

Diagnosing:

```bash
# Show processes in D state with their wait channel
ps -eo state,pid,wchan:32,cmd | grep "^D"

# Check I/O wait percentage (all devices, header once)
iostat -x 1 3

# Check a specific disk (header once)
iostat -x sda 1 3

# Per-process I/O stats
sudo iotop -boP -qqq -n 1 | sort -k6 -rn | head
# or
pidstat -d 1 3
```

If `%iowait` in `iostat` is high and load is high but `%user` + `%system` CPU is low, your load is I/O driven, not CPU driven.

## Load Average and Containers

In containerized environments:

- `/proc/loadavg` shows the **host** load average, not the container's
- A container with CPU limits might be throttled even when host load looks fine
- Check container-specific CPU pressure via cgroup:

```bash
# cgroups v2 — CPU pressure
cat /sys/fs/cgroup/cpu.pressure

# Shows: some avg10=X.XX avg60=X.XX avg300=X.XX total=XXXX
# "some" means at least one task was delayed

# cgroups v1
cat /sys/fs/cgroup/cpu,cpuacct/cpu.stat
```

For Kubernetes pods, the load average visible inside the pod is the node's load average, not the pod's.

## Pressure Stall Information (PSI)

Linux 4.20+ provides PSI, a more granular alternative to load average:

```bash
# CPU pressure
cat /proc/pressure/cpu
# some avg10=1.23 avg60=0.87 avg300=0.45 total=123456

# I/O pressure
cat /proc/pressure/io
# some avg10=5.67 avg60=3.21 avg300=1.98 total=789012
# full avg10=4.12 avg60=2.45 avg300=1.23 total=567890

# Memory pressure
cat /proc/pressure/memory
```

- `some` = percentage of time at least one task was stalled
- `full` = percentage of time all tasks were stalled (lost throughput)

PSI separates CPU, I/O, and memory pressure — something load average alone cannot do.

## Practical Troubleshooting Flow

```
High load average detected
        |
        v
Run: vmstat 1 5
        |
        +--> High 'r' column → CPU-bound
        |       → Check: top, perf top, pidstat -u
        |
        +--> High 'b' column → I/O-bound
        |       → Check: iostat -x, iotop, pidstat -d
        |
        +--> Both high → Mixed
                → Address I/O first (usually the bigger bottleneck)
```

## Key Takeaways

1. **Load average != CPU usage** — on Linux it includes I/O-blocked processes
2. **Always compare to CPU count** — absolute numbers mean nothing without context
3. **Use vmstat to differentiate** — `r` column for CPU pressure, `b` column for I/O pressure
4. **Look at trends** — compare 1m vs 15m to see direction
5. **Consider PSI** — on modern kernels it gives you cleaner signal than load average
6. **In containers** — `/proc/loadavg` is host-level; use cgroup pressure files instead

## Quick Commands Reference

```bash
# Load average
uptime
cat /proc/loadavg

# CPU count
nproc

# Real-time breakdown
vmstat 1

# Find R/D state processes
ps -eLo state,pid,cmd | grep -e "^R" -e "^D"

# I/O stats
iostat -x 1
iotop -b -n 1

# Historical data
sar -q

# PSI (Linux 4.20+)
cat /proc/pressure/cpu
cat /proc/pressure/io
cat /proc/pressure/memory

# Container CPU pressure (cgroups v2)
cat /sys/fs/cgroup/cpu.pressure
```
