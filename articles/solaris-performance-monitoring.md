# Solaris Performance and Resource Monitoring

Inspecting CPU, memory, and swap usage on Oracle Solaris using the native tools — `prstat` (the Solaris equivalent of `top`), `swap` for swap/virtual memory, `psrinfo` for CPU inventory, and `mdb` for kernel memory stats. This guide focuses on the commands that differ from Linux and the Solaris-specific concepts (zones, projects) behind them.

> On Solaris, prefer `prstat` over `top` — it's the supported, always-present tool and is zone/project aware. `top` may not be installed by default.

## prstat — Active Process Statistics

`prstat` continuously reports the most resource-hungry processes, like `top`, but with Solaris-native grouping options.

```bash
# Report users consuming the most system resources (aggregated per user)
prstat -a

# Append new reports instead of overprinting (good for logging to a file)
prstat -c 2 > prstat.txt

# Monitor a single process, refreshing every 5 seconds
prstat -p 28983 5

# Aggregate usage per zone
prstat -Z

# Aggregate usage per project
prstat -J

# Sort processes by resident set size (physical memory)
prstat -s rss

# Show per-user totals including memory/swap
prstat -t
```

| Option | Purpose |
|--------|---------|
| `-a` | Report by process *and* by user (top resource-consuming users) |
| `-c` | Print new reports below old ones (non-overprinting; good for files) |
| `-p <pid>` | Restrict to a specific process |
| `-Z` | Summarize per **zone** |
| `-J` | Summarize per **project** |
| `-s rss` | Sort by resident set size (RSS = physical RAM used) |
| `-t` | Report totals per user |
| `<interval>` | Trailing number = refresh interval in seconds |

Zones and projects are Solaris resource-management constructs — `-Z` and `-J` let you attribute load to a container or a workload grouping, which has no direct `top` equivalent on Linux.

Sample `prstat` output:

```
   PID USERNAME  SIZE   RSS STATE  PRI NICE      TIME  CPU PROCESS/NLWP
  1387 oracle    2312M 2101M sleep   59    0   0:14:02 3.1% oracle/11
   928 root       168M  120M sleep   59    0   0:02:41 0.4% java/24
   ...
Total: 84 processes, 512 lwps, load averages: 0.42, 0.38, 0.35
```

Columns: `SIZE` = virtual size, `RSS` = resident (physical) memory, `STATE` = run state, `TIME` = CPU time, `NLWP` = number of lightweight processes (threads).

## System-Wide Statistics (vmstat, mpstat, iostat, sar)

Beyond per-process, these sample system-wide activity (all take an interval and count):

```bash
# Virtual memory / CPU / paging summary, every 2s
vmstat 2 5

# Per-CPU utilization, every 2s
mpstat 2 5

# Per-disk I/O with device names, every 2s
iostat -xnz 2 5

# System Activity Reporter — collected history or live
sar -u 2 5        # CPU
sar -r 2 5        # memory/swap
sar -d 2 5        # disk
```

- `vmstat` — the `r`/`b` columns (run/block queue), `pi`/`po` (page in/out), `sr` (scan rate — sustained high `sr` means memory pressure).
- `mpstat` — spot a single hot CPU vs balanced load; watch `smtx` (mutex contention) and `csw` (context switches).
- `iostat -xnz` — `%b` (busy), `asvc_t` (average service time in ms); high service time signals a slow/saturated disk.
- `sar` — reads archived data from `/var/adm/sa/` when given a date, useful for after-the-fact analysis.

## Swap and Virtual Memory

Solaris distinguishes on-disk swap areas from total virtual memory (which includes RAM).

```bash
# List configured swap areas (on-disk swap devices/files)
swap -l

# Summary of virtual memory usage (includes RAM-backed anonymous memory)
swap -s

# Percentage of free swap (from swap -l: free/total * 100)
swap -l | grep -v swapfile | awk '{ print $5/$4 * 100.0; }'

# Sort processes by swap usage
prstat -t
top -d1 | grep swap    # if top is installed
```

- `swap -l` — lists physical swap areas with total (`blocks`) and free columns.
- `swap -s` — reports total virtual memory (allocated, reserved, used, available), counting RAM plus disk swap. The two commands report different things; use `-l` for disk swap devices and `-s` for the overall VM picture.

## CPU Inventory

```bash
# Physical CPU/socket info — sockets, cores, threads, speed, model
psrinfo -pv

# (related) per-virtual-processor status
psrinfo -v
```

- `psrinfo -pv` — **physical** view: number of physical processors and the cores/threads each provides.
- `psrinfo -v` — lists each virtual processor (hardware thread) and whether it's online.

## Memory Statistics (mdb)

```bash
# Kernel view of total/available memory by category
echo ::memstat | mdb -k
```

`::memstat` is a kernel debugger macro (`mdb -k` attaches to the live kernel) that breaks memory down into categories — kernel, ZFS ARC, anonymous, page cache, free — giving a truer picture of "available" memory than a single free counter. It can take a moment to run on large systems.

Sample `::memstat` output:

```
Page Summary                Pages                MB  %Tot
------------     ----------------  ----------------  ----
Kernel                     245678              1919   12%
ZFS File Data              812345              6346   40%
Anon                       456789              3568   22%
Exec and libs               34567               270    2%
Page cache                  89012               695    4%
Free (cachelist)            67890               530    3%
Free (freelist)            345678              2700   17%
Total                     2039359             15932  100%
```

On ZFS systems the **ZFS File Data** (ARC) line is often large — that memory is reclaimable under pressure, so it's not "used up" the way `Anon` (application) memory is.

### DTrace for Deep Analysis

For questions the counters can't answer (which syscalls, which files, which functions), Solaris ships **DTrace**:

```bash
# What processes are doing the most disk I/O (bytes)?
dtrace -n 'io:::start { @[execname] = sum(args[0]->b_bcount); }'

# Count system calls by process
dtrace -n 'syscall:::entry { @[execname] = count(); }'
```

DTrace is safe to run on production and adds near-zero overhead when idle.

## Troubleshooting Performance

| Symptom | What to check | Command |
|---------|---------------|---------|
| High load, sluggish system | Which process/user/zone | `prstat -a`, `prstat -Z` |
| Suspected memory pressure | Sustained page scan rate (`sr`) | `vmstat 2` (watch `sr`, `po`) |
| Slow disk / high latency | Per-disk busy % and service time | `iostat -xnz 2` (`%b`, `asvc_t`) |
| One CPU pegged | Per-CPU balance, mutex contention | `mpstat 2` (`smtx`, `usr`/`sys`) |
| "Out of memory" but free looks low | Reclaimable ZFS ARC vs real usage | `echo ::memstat \| mdb -k` |
| Swapping | Active swap-out | `swap -l`, `vmstat` (`po` column) |

```bash
# Quick triage: top consumers + system rates side by side
prstat -a -n 10 1 1 ; vmstat 1 3
```

## Command Reference

| Task | Command |
|------|---------|
| Top processes (like `top`) | `prstat` |
| Top users | `prstat -a` / `prstat -t` |
| Monitor one PID | `prstat -p <pid> <interval>` |
| Per-zone usage | `prstat -Z` |
| Per-project usage | `prstat -J` |
| Sort by memory (RSS) | `prstat -s rss` |
| Log to file | `prstat -c <interval> > file` |
| List disk swap | `swap -l` |
| Virtual memory summary | `swap -s` |
| CPU inventory | `psrinfo -pv` |
| Kernel memory breakdown | `echo ::memstat \| mdb -k` |

## Notes for Linux Admins

| You'd use on Linux | On Solaris |
|--------------------|------------|
| `top` / `htop` | `prstat` |
| `free -h` | `swap -s`, `echo ::memstat \| mdb -k` |
| `swapon -s` / `free` (swap line) | `swap -l` |
| `lscpu` / `/proc/cpuinfo` | `psrinfo -pv` |
| cgroup accounting | zones (`prstat -Z`) / projects (`prstat -J`) |

## References

- [prstat(1M) man page](https://docs.oracle.com/cd/E23824_01/html/821-1462/prstat-1m.html) — official Oracle docs
- [Monitoring System Performance in Oracle Solaris](https://docs.oracle.com/cd/E37838_01/html/E56831/index.html) — official Oracle docs
- [swap(1M) man page](https://docs.oracle.com/cd/E23824_01/html/821-1462/swap-1m.html) — official Oracle docs
