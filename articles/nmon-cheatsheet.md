# nmon Cheatsheet

`nmon` (Nigel's Monitor) is an interactive performance monitor for Linux and AIX that shows CPU, memory, disk, network, filesystems, NFS, and top processes in one terminal view. It can also record metrics to a file for later analysis. This cheatsheet covers interactive keys, capture mode, output analysis, and automation.

For related tools, see the [iostat Cheatsheet](articles/iostat-cheatsheet.md), [sysstat / sar Cheatsheet](articles/sysstat-sar-cheatsheet.md), [top Cheatsheet](articles/top-cheatsheet.md), and [Performance Co-Pilot (PCP) Cheatsheet](articles/pcp-cheatsheet.md).

## Installation

```bash
# Debian and Ubuntu
sudo apt install nmon

# RHEL, Rocky Linux, AlmaLinux, and Fedora
sudo dnf install nmon      # older releases: sudo yum install nmon

# Arch Linux (AUR)
yay -S nmon
```

`nmon` is in the standard repos on most distributions; prefer the package over a source build. On RHEL-family systems it comes from EPEL.

## Interactive Mode

Start it with no arguments:

```bash
nmon
```

Press a key to toggle each panel on or off; press several to stack multiple panels. Press the same key again to hide that panel.

### Panel keys

| Key | Panel |
|-----|-------|
| `c` | CPU utilization (per core, bar chart) |
| `C` | CPU utilization, wider/alternate view |
| `m` | Memory and swap |
| `d` | Disk I/O |
| `D` | Disk I/O with more detail / cycle disk views |
| `n` | Network interfaces |
| `N` | NFS statistics |
| `j` | Filesystem usage |
| `k` | Kernel and load statistics |
| `t` | Top processes |
| `V` | Virtual memory / paging |
| `g` | User-defined disk groups (needs a group file) |
| `l` | Long-term CPU average (single line) |

Available panels vary by version and platform (some, like detailed resource or partition views, differ between Linux and AIX). Press `h` to see the exact keys your build supports.

### Control keys

| Key | Action |
|-----|--------|
| `h` | Help / key list |
| `q` | Quit |
| `+` | Faster refresh (halve the interval) |
| `-` | Slower refresh (double the interval) |
| `.` | Show only busy disks/processes (compact) |
| `b` | Toggle black-and-white mode |
| `space` | Refresh immediately |

By default nmon refreshes every 2 seconds interactively.

## Capture Mode (Recording)

Capture mode writes a timestamped data file for later analysis instead of drawing the screen. The two key options are `-s` (seconds between snapshots) and `-c` (number of snapshots).

```bash
# Record a snapshot every 30s, 120 times → 1 hour of data
nmon -f -s 30 -c 120

# Every 60s for 24 hours (1440 snapshots)
nmon -f -s 60 -c 1440
```

`-f` starts spreadsheet-format capture and returns the shell prompt while nmon records in the background. By default it writes `<hostname>_<YYYYMMDD>_<HHMM>.nmon` in the current directory.

### Capture options

| Option | Meaning |
|--------|---------|
| `-f` | Start capture in spreadsheet (CSV-like) format |
| `-s N` | Seconds between snapshots (default 2) |
| `-c N` | Number of snapshots to collect |
| `-t` | Include top processes |
| `-T` | Include top processes **and** their command-line arguments |
| `-F name` | Write to a specific filename |
| `-m dir` | Change to `dir` before writing the output file |

`-t` and `-T` both enable top-process capture; `-T` adds command arguments. They are not combined with each other — pick one.

```bash
# Custom filename with a date, top processes included
nmon -F "myserver_$(date +%Y%m%d).nmon" -t -s 60 -c 720

# Write into a dedicated directory
nmon -f -t -s 60 -c 60 -m /var/log/nmon
```

The total capture duration is `s × c` seconds. For example `-s 300 -c 288` is 300 × 288 = 86,400 s = 24 hours.

### Common capture profiles

| Purpose | Command |
|---------|---------|
| 1 hour, 30s resolution | `nmon -f -t -s 30 -c 120` |
| 24 hours, 5-min resolution | `nmon -f -t -s 300 -c 288` |
| 1 week, 15-min resolution | `nmon -f -t -s 900 -c 672` |
| Incident: 1 hour, 5s resolution | `nmon -f -t -s 5 -c 720` |

Finer intervals mean larger files and slightly more overhead; match resolution to how long you need to watch.

## Reading the Interactive Display

### CPU (`c`)

- `User%` — application (user-space) CPU.
- `Sys%` — kernel/system CPU.
- `Wait%` — time blocked on I/O (high values point to a disk bottleneck).
- `Idle%` — unused CPU.
- `Steal%` — CPU taken by the hypervisor (relevant on VMs).

### Memory (`m`)

Watch total vs free RAM and swap. High RAM "used" with most of it as cache is normal; **low free RAM together with rising swap use** indicates real memory pressure.

### Disk (`d`)

Read/write throughput and a busy percentage per device. A device pinned near 100% busy is saturated; uneven load across devices can indicate a RAID/LVM imbalance.

### Network (`n`)

Per-interface read/write throughput. Sustained rates near link capacity, or errors/drops, indicate a network bottleneck or fault.

### Top processes (`t`)

PID, CPU%, memory, and command for the busiest processes — the quickest way to attribute load to a specific process.

## Analyzing Captured Files

An `.nmon` file is line-oriented CSV: each line starts with a section tag (`CPU_ALL`, `MEM`, `DISKBUSY`, `NET`, etc.), and `ZZZZ` lines mark snapshot timestamps.

### The nmon analyser

The classic path is the **nmon analyser**, a spreadsheet (Excel/LibreOffice) macro from the nmon project that turns an `.nmon` file into graphs automatically. Download it from the nmon site and open your `.nmon` file with it.

### Quick command-line inspection

```bash
# Section tags present in the file
cut -d, -f1 myfile.nmon | sort -u

# Pull one metric family
grep '^CPU_ALL,' myfile.nmon
grep '^MEM,'      myfile.nmon
grep '^DISKBUSY,' myfile.nmon
grep '^NET,'      myfile.nmon

# Snapshot timestamps (ZZZZ,Tnnn,HH:MM:SS,DD-MON-YYYY)
grep '^ZZZZ,' myfile.nmon
```

Column positions vary by nmon version, so confirm the header line for a section before extracting a fixed field:

```bash
# Show the header row for the CPU_ALL section
grep -m1 '^CPU_ALL,' myfile.nmon
```

Because layouts differ across versions, prefer the nmon analyser (or a parser like `nmonchart`/`pmda-nmon`) over hardcoded `awk` column numbers for anything you rely on.

## Automation

### Cron

```bash
# Every day at 06:00, capture 14 hours at 5-min resolution into a log dir
0 6 * * * /usr/bin/nmon -f -t -s 300 -c 168 -m /var/log/nmon
```

Point captures at a dedicated directory with `-m`, and make sure it exists and is writable by the cron user.

### systemd service

Run one long capture that restarts daily:

```ini
# /etc/systemd/system/nmon-record.service
[Unit]
Description=nmon performance recording
After=network-online.target

[Service]
Type=forking
ExecStart=/usr/bin/nmon -f -t -s 300 -c 288 -m /var/log/nmon
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Use `Type=forking` because `nmon -f` daemonizes and returns. Enable it with `sudo systemctl enable --now nmon-record.service`.

### Retention cleanup

```bash
# Delete captures older than 30 days; compress ones older than 7
find /var/log/nmon -name '*.nmon' -mtime +30 -delete
find /var/log/nmon -name '*.nmon' -mtime +7 ! -name '*.gz' -exec gzip {} +
```

## User-Defined Disk Groups

Group related disks so the `g` panel and capture aggregate them:

```text
# /etc/nmon/diskgroups.conf
data sda sdb
logs sdc
```

```bash
nmon -g /etc/nmon/diskgroups.conf
```

Each line is a group name followed by its devices.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Garbled display | `reset`, then `export TERM=xterm-256color`; or run `nmon -b` for black-and-white |
| "already running" / duplicate captures | `pgrep -a nmon`; stop stray recorders with `pkill nmon` |
| No output file | Check the `-m` directory exists and is writable; check free space with `df -h` |
| Missing panels | Your build/platform may not support that key — press `h` for the supported list |
| Extraction gives wrong values | Column order differs by version; re-check the section header line |

```bash
# Is a capture already running?
pgrep -a nmon

# Stop capture recorders
pkill nmon
```

## nmon vs Other Tools

| Tool | Strength | When to prefer |
|------|----------|----------------|
| `nmon` | One-screen overview + easy file capture | Quick whole-system view, ad-hoc recording |
| `top`/`htop` | Live process focus | Interactive process triage |
| `sar` | Detailed, scriptable historical data | Long-term trending, scripting |
| PCP | Distributed, extensible metrics | Fleet-wide, integrated monitoring |

See the [top Cheatsheet](articles/top-cheatsheet.md), [sysstat / sar Cheatsheet](articles/sysstat-sar-cheatsheet.md), and [Performance Co-Pilot (PCP) Cheatsheet](articles/pcp-cheatsheet.md).

## Quick Reference

```text
Interactive keys:
  c CPU     m Memory   d Disk     n Network   N NFS
  j FS      k Kernel   t Top      V VM        l CPU avg
  h Help    q Quit     + Faster   - Slower    . Compact   b B/W

Capture:
  nmon -f -s 30 -c 120            # 1 hour, 30s snapshots
  nmon -f -t -s 300 -c 288        # 24 hours, top processes
  nmon -F name.nmon -s 60 -c 60   # custom filename
  nmon -f -s 60 -c 1440 -m /var/log/nmon   # into a directory

Duration = s × c seconds.
```

```bash
# Inspect a capture
cut -d, -f1 myfile.nmon | sort -u        # sections present
grep '^CPU_ALL,' myfile.nmon             # CPU rows
grep -m1 '^MEM,' myfile.nmon             # header for MEM columns
```

For related material, see the [iostat Cheatsheet](articles/iostat-cheatsheet.md), [Understanding vmstat Output](articles/understanding-vmstat-output.md), and the [AIX Performance Monitoring Cheatsheet](articles/aix-performance-monitoring-cheatsheet.md).
