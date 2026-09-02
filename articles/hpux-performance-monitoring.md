# HP-UX Performance Monitoring and Event Management

Watching an HP-UX system's health: a quick free-memory one-liner with `vmstat`, the interactive **Glance** performance tool and its shortcut keys, the `sar` activity reporter (including HBA and LUN-path variants), and event monitoring through **SFM** (System Fault Management) and **EMS** with `evweb`. Related: [HP-UX Kernel Configuration and Tuning](articles/hpux-kernel-configuration.md) (`kcusage`) and [HP-UX Fibre Channel and SAN](articles/hpux-fibre-channel-san.md).

## Quick Free-Memory Check

`vmstat` reports free memory in 4 KB pages (column 5). This one-liner converts it to megabytes:

```bash
vmstat 1 2 | tail -1 | awk '{printf "%d%s\n", ($5*4)/1024, "MB" }'
```

Taking two samples (`1 2`) and using the last line gives a settled reading rather than the boot-time average from a single sample.

### Why the First vmstat Line Is a Lie

This is the single most common `vmstat` mistake. The **first** line of output is not a snapshot of "now" — it's an average of every counter since the system booted. On a box that's been up for months, that first line tells you almost nothing about current conditions. Every subsequent line is a genuine delta over the sampling interval. That's exactly why the one-liner asks for two samples and discards the first with `tail -1`: the second line reflects the last one second, not months of history. The same "ignore the first sample" rule applies to `sar`, `iostat`, and most other interval-based tools on HP-UX.

### Reading the Key Columns

`vmstat` groups its columns; the ones that matter most for a quick health read are:

| Column | Meaning | What to watch for |
|--------|---------|-------------------|
| `r` (procs) | Threads runnable / waiting for CPU | Persistently `> number of CPUs` means CPU-bound |
| `free` | Free memory in **pages** (×4 KB) | Very low free alone isn't alarming on HP-UX |
| `pi` / `po` | Pages paged in / out | Sustained `po` (page-outs) signals real memory pressure |
| `us` / `sy` / `id` | CPU % user / system / idle | High `sy` can indicate driver or syscall overhead |

The crucial nuance on HP-UX: **low free memory is normal and healthy**. Unused RAM is wasted RAM, so the kernel and the filesystem buffer cache expand to use it. The reliable pressure signal is not free memory but sustained **page-out** activity (`po`) and swap activity — that means the system is being forced to evict pages, not merely caching aggressively. Judge memory health by paging behavior, not by the free counter.

## Glance

Glance (part of **GlancePlus**, a separately licensed HP product) is an interactive real-time performance monitor. It comes in two forms driven by the same data collector: the character/terminal interface (`glance`) and the Motif/X11 GUI (`gpm`). Unlike a one-shot tool, Glance continuously samples the kernel's performance instrumentation and presents live, drill-down views — global CPU/memory/disk/network on one screen, then per-process, per-disk, or per-filesystem detail on the next keystroke.

Where Glance earns its keep is **bottleneck attribution**. `vmstat` tells you the system is busy; Glance tells you *which process, which disk, or which filesystem* is responsible, and it colour-codes/flags resources that cross utilization thresholds so a saturated CPU or a hot disk jumps out immediately. It also exposes wait-state analysis for a single process (why is this process not running — waiting on CPU, disk, memory, a lock, the network?), which is hard to get from the classic tools.

A note on availability: because GlancePlus is a licensed add-on, it may not be installed on every host. On systems without it you fall back to `top` (bundled) for a live process view and `sar`/`vmstat`/`iostat` for subsystem sampling. GlancePlus and its OpenView/Performance Agent lineage are also end-of-life on modern HP-UX, so treat it as "use it if it's there."

Once running, single keystrokes switch between reports:

| Key | Report |
|-----|--------|
| `a` | All CPU detail |
| `c` | CPU report |
| `d` | Disk report (then `b`: buffer cache page) |
| `g` | Global process list (initial screen) |
| `i` | I/O by filesystem |
| `j` | Adjust data refresh interval |
| `l` | LAN — network by interface |
| `m` | Memory report |
| `M` | Single-process memory detail |
| `N` | Global NFS activity |
| `n` | NFS by system |
| `o` | Change process-listing thresholds |
| `s` | Single-process detail |
| `t` | System table utilization |
| `u` | I/O by disk |
| `v` | I/O by logical volume |
| `w` | Swap report |
| `W` | Single-process wait states |
| `y` | Renice a process |

Common launch options: `glance -j 5` sets a 5-second refresh interval; `glance -p` starts on the process screen; and inside the tool `?` or `h` shows the help/key legend. To capture what you see for later review, Glance can log to a file, but for scripted trend collection the companion Performance Agent (`scopeux`/`midaemon`) or `sar`'s own logging is the usual choice.

## sar (System Activity Reporter)

`sar` is the portable, scriptable counterpart to Glance: no license, no GUI, just periodic counters written to stdout (or to a binary log). It has two modes that are easy to confuse. **Interactive/live mode** takes an interval and count and samples in real time — `sar -u 1 5` means "CPU utilization, one-second interval, five samples." **Historical mode** reads a previously recorded binary data file with `-f` and reports on the past, which is how you answer "what was the system doing at 3 a.m. last night?"

Historical mode depends on the **sar data collector** being enabled. Two cron jobs (traditionally `sa1`, which samples into today's binary file, and `sa2`, which produces a daily text report) populate `/var/adm/sa/saDD` files, one per day of the month. If those aren't scheduled, there's no history to read back:

```bash
sar -u -f /var/adm/sa/sa15        # replay CPU stats recorded on the 15th
sar -f /var/adm/sa/sa15 -s 02:00 -e 04:00   # narrow to a time window
```

Add an interval and count (e.g. `sar -u 1 5`) for live sampling.

| Option | Reports |
|--------|---------|
| `-u` | CPU utilization |
| `-d` | Disk activity |
| `-b` | Buffer cache activity |
| `-w` | Swapping activity (sar is weak on memory detail) |
| `-q` | Run-queue lengths (vmstat is better here) |
| `-c` | System calls |
| `-v` | Kernel table utilization (nfile, etc.) |
| `-H` | HBA port-level activity |
| `-L` | Per-LUN-path activity |

```bash
sar -u 1 5          # CPU utilization, 5 one-second samples
sar -L 1 5          # per-LUN-path performance
sar -H 1 5          # per-HBA performance
```

The `-H` and `-L` options are useful for [SAN](articles/hpux-fibre-channel-san.md) troubleshooting, breaking I/O down by HBA port and by individual LUN path. On multipathed storage this is how you tell whether load is balanced across paths or piling onto one HBA — a single hot path often points at a failed-over or misconfigured multipath setup rather than a genuinely overloaded array.

Reading `sar -d` (disk activity), the columns that matter are `%busy` (how much of the interval the device was servicing requests), `avque` (average queue length — requests waiting), `avwait` (time a request sat in the queue), and `avserv` (time the device took to service it once started). A device pegged at high `%busy` with a growing `avque` and rising `avwait` is your bottleneck; if `avserv` itself is high, the underlying storage (or path) is slow rather than merely busy.

### iostat and top

Two always-present, unlicensed tools round out the quick-look toolkit:

```bash
iostat 2 5          # per-disk throughput (KB/s) and utilization, 5 samples at 2s
top                 # live per-process CPU, plus per-CPU load and memory summary
```

`iostat` is the fastest way to see raw device throughput and, like `vmstat`, its first sample is a since-boot average to be ignored. `top` gives an at-a-glance ranking of CPU-hungry processes and a per-processor breakdown, which is handy on multi-core/cell systems where one saturated CPU can hide in a low-looking global average. Neither offers Glance's wait-state attribution, but both are available on every host.

## SFM and EMS Event Management

Performance monitoring answers "is the system fast enough?"; **event management** answers "is the hardware healthy?" — and the two are related, because a failing component (a disk being retried, a fan slowing, memory correcting errors) often shows up as a performance anomaly before it becomes an outage.

**System Fault Management (SFM)** is the 11i v2/v3 framework that monitors hardware health (CPUs, memory, power, fans, disks) and reports faults through the WBEM/CIM infrastructure. It largely supersedes the older **Event Monitoring Service (EMS)** hardware monitors, though EMS is still present and still underlies some monitoring on older configurations. Both ultimately feed events to subscribers — email, an OpenView/management station via SNMP traps, or the local logs — so that a predicted or actual hardware fault reaches an administrator automatically. SFM logs through these files:

| File | Contents |
|------|----------|
| `/var/opt/sfm/log/sfm.log` | SFM provider module log |
| `/var/opt/sfm/log/event.log` | Events monitored by SFM |

`evweb` is the command-line (and web) interface for viewing events and managing subscriptions:

```bash
evweb eventviewer -L          # list events
evweb eventviewer -E -n 35    # show details of event number 35
evweb subscribe -L            # list subscriptions created via EVWEB
```

## A Bottleneck-Hunting Method

Tools are only useful with a method behind them. A reliable first pass on a "the system is slow" complaint is to check the four classic subsystems in order and let each result steer the next:

1. **CPU** — `vmstat 1 5` (watch `r` vs CPU count, and `us`/`sy`/`id`) or `sar -u 1 5`. A long run queue and low idle means CPU-bound; then use Glance or `top` to find the guilty process.
2. **Memory** — the same `vmstat` output: sustained `po` (page-outs) and swap activity, not low free memory, is the real signal. Confirm with `sar -w`. If memory-bound, find the process with Glance's single-process memory view (`M`).
3. **Disk / I/O** — `sar -d 1 5` or `iostat`. Look for a device at high `%busy` with a rising queue (`avque`/`avwait`); if it's SAN, drill to the path with `sar -L` / `sar -H`.
4. **Network** — Glance's LAN report (`l`) or `netstat -i` / `lanadmin` for interface errors and collisions.

The discipline that matters: change **one** thing at a time, measure again over a representative interval, and always ignore the first sample of any interval-based tool. A "slow" system is usually bottlenecked on exactly one of these four resources; identifying which one before touching anything saves you from chasing symptoms.

### Common Pitfalls

- **Trusting a single sample.** Boot-time averages (the first line) and one-second blips both mislead. Take several samples and look at the trend.
- **Reading low free memory as a problem.** On HP-UX the buffer cache deliberately consumes free RAM; page-outs, not free pages, indicate pressure.
- **Assuming Glance is installed.** It's a licensed add-on; script your monitoring around `sar`/`vmstat`/`iostat` so it works everywhere.
- **No historical data.** If the `sar` collector (`sa1`/`sa2`) was never enabled, you can't investigate last night's spike after the fact. Turn it on before you need it.
- **Ignoring hardware events.** A slowly failing disk or correcting-memory condition can masquerade as a performance problem; check SFM/EMS events (`evweb`) as part of the pass.

## Command Reference

| Task | Command |
|------|---------|
| Free memory in MB | `vmstat 1 2 \| tail -1 \| awk '{printf "%d%s\n", ($5*4)/1024, "MB"}'` |
| Live CPU/memory sampling | `vmstat 1 5` |
| Per-disk throughput | `iostat 2 5` |
| Top CPU consumers | `top` |
| Replay recorded sar data | `sar -u -f /var/adm/sa/sa<DD>` |
| Interactive monitor | `glance` |
| CPU / disk / memory sampling | `sar -u 1 5` / `sar -d 1 5` / `sar -w 1 5` |
| Per-HBA activity | `sar -H 1 5` |
| Per-LUN-path activity | `sar -L 1 5` |
| List events | `evweb eventviewer -L` |
| Event detail | `evweb eventviewer -E -n <num>` |
| List subscriptions | `evweb subscribe -L` |

## Related Articles

- [HP-UX Kernel Configuration and Tuning](articles/hpux-kernel-configuration.md)
- [HP-UX Fibre Channel and SAN](articles/hpux-fibre-channel-san.md)
- [HP-UX Swap Management](articles/hpux-swap-management.md)
- [HP-UX Crash Dump Analysis with Q4](articles/hpux-crash-dump-analysis.md)
- [HP-UX Management Processor (MP / GSP / iLO)](articles/hpux-management-processor.md)
- [HP-UX Administration Tips and Recipes](articles/hpux-admin-tips-recipes.md)
