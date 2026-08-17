# Configuring sysstat for Performance Monitoring

When you need to understand what your Linux server has been doing over the past few hours (or days), sysstat is the tool you reach for. It collects, reports, and archives system activity information including CPU, memory, disk I/O, and network statistics. Think of it as a flight recorder for your server.

This guide covers installation and configuration on Ubuntu 22.04/24.04 LTS and RHEL/CentOS 6–10.

## What sysstat Provides

The sysstat package includes several utilities:

- **sar** — System Activity Reporter, the main tool for viewing historical data
- **iostat** — CPU and I/O statistics (real-time only)
- **mpstat** — Per-processor statistics (real-time only)
- **pidstat** — Per-process statistics (real-time only, not collected historically)
- **tapestat** — Tape drive statistics
- **cifsiostat** — CIFS filesystem statistics
- **sadc** — System Activity Data Collector (the backend that collects data)
- **sadf** — Displays sar data in different formats (CSV, JSON, XML)

> **Note:** The data collected by `sadc` and viewed with `sar` is system-wide (CPU, memory, disk, network). Real-time tools like `iostat`, `mpstat`, and `pidstat` only report live statistics — they cannot show historical data. Use `sar` for historical queries (e.g., `sar -d` for disk, `sar -P ALL` for per-CPU, `sar -b` for I/O transfer rates).

## Installation

Install sysstat on both Ubuntu 22.04 and 24.04:

```bash
sudo apt update
sudo apt install sysstat -y
```

Check the installed version:

```bash
sar -V
```

Typical versions:

- Ubuntu 22.04: sysstat 12.5.2
- Ubuntu 24.04: sysstat 12.6.1+

## Enabling sysstat Data Collection

> **Note:** On Ubuntu 24.04, sar is enabled by default — the systemd timer collects metrics out of the box. On Ubuntu 22.04, you need to enable it explicitly.

Start and enable the sysstat service:

```bash
sudo systemctl start sysstat
sudo systemctl enable sysstat
```

Verify it's running:

```bash
sudo systemctl status sysstat
```

You should see `active (exited)` — this is normal because sysstat relies on timers rather than a long-running daemon.

## Understanding the Collection Mechanism

Both Ubuntu 22.04 and 24.04 ship with a systemd timer (`sysstat-collect.timer`) that triggers data collection every 10 minutes.

> **Note:** On Ubuntu 22.04, the systemd timer is shipped but **not** enabled by default. On Ubuntu 24.04, it is active out of the box.

> **Note:** A cron file (`/etc/cron.d/sysstat`) is still shipped with the package but is no longer used. The systemd timer is the proper mechanism for managing collection.

Check if the timer is running:

```bash
systemctl list-timers | grep sysstat
```

The relevant units are:

- `sysstat-collect.timer` — triggers data collection every 10 minutes
- `sysstat-summary.timer` — generates daily summaries

Check the timer configuration:

```bash
systemctl cat sysstat-collect.timer
```

## Configuring Collection Interval

The default 10-minute interval may be too coarse for troubleshooting short-lived performance issues. We'll configure it to collect every 1 minute.

### Step 1: Enable the systemd timer (Ubuntu 22.04 only)

On Ubuntu 22.04, the timer is not active by default. Enable it first:

```bash
sudo systemctl enable sysstat-collect.timer
sudo systemctl start sysstat-collect.timer
```

On Ubuntu 24.04, skip this step — the timer is already running.

### Step 2: Override the timer interval

```bash
sudo systemctl edit sysstat-collect.timer
```

This opens an override file. Add:

```ini
[Timer]
OnCalendar=
OnCalendar=*:00/1
```

The first empty `OnCalendar=` clears the default 10-minute schedule. The second sets it to every 1 minute.

### Step 3: Reload and restart

```bash
sudo systemctl daemon-reload
sudo systemctl restart sysstat-collect.timer
```

### Step 4: Verify the new interval

```bash
systemctl list-timers sysstat-collect.timer
```

The `NEXT` and `LEFT` columns should now show a 1-minute gap.

Wait a couple of minutes and confirm with:

```bash
sar
```

You should now see entries every 1 minute.

## Configuring Data Retention

By default, sysstat keeps data for 7 days. To change this:

```bash
sudo vi /etc/sysstat/sysstat
```

Find the `HISTORY` parameter and adjust it:

```
# How long to keep log files (in days)
HISTORY=28
```

This keeps 28 days of historical data. On Ubuntu 24.04, you may also see `SA_DIR` which controls where data files are stored (default: `/var/log/sysstat`).

### Disk Space Considerations

Each daily data file is typically 2-5 MB depending on the number of CPUs, disks, and network interfaces. With 28 days retention:

- Minimal server: ~56 MB
- Server with many interfaces/disks: ~140 MB

If you're using a 1-minute collection interval, files will be roughly 10x larger than the default 10-minute interval.

## RHEL / CentOS Configuration (6–10)

### Installation

| RHEL Version | Command |
|--------------|---------|
| RHEL 6 | `sudo yum install sysstat -y` |
| RHEL 7 | `sudo yum install sysstat -y` |
| RHEL 8 | `sudo dnf install sysstat -y` |
| RHEL 9 | `sudo dnf install sysstat -y` |
| RHEL 10 | `sudo dnf install sysstat -y` |

Enable and start sysstat:

```bash
sudo systemctl enable sysstat
sudo systemctl start sysstat
```

### RHEL 6: Cron-Based Collection

RHEL 6 uses cron instead of systemd timers. The collection interval is configured in `/etc/cron.d/sysstat`.

#### Change interval to 1 minute

```bash
sudo sed -i 's|^\*/10|*/1|' /etc/cron.d/sysstat
```

This changes `*/10 * * * *` to `*/1 * * * *`.

#### Set retention to 28 days

```bash
sudo sed -i 's/^HISTORY=.*/HISTORY=28/' /etc/sysstat/sysstat
```

#### Restart crond

```bash
sudo service crond restart
```

### RHEL 7–10: systemd Timer Collection

RHEL 7 and later use `sysstat-collect.timer` (same mechanism as Ubuntu). The default interval is 10 minutes.

#### Step 1: Create a timer override for 1-minute collection

```bash
sudo mkdir -p /etc/systemd/system/sysstat-collect.timer.d

echo -e "[Timer]\nOnCalendar=\nOnCalendar=*:00/1" | sudo tee /etc/systemd/system/sysstat-collect.timer.d/override.conf
```

The first `OnCalendar=` clears the default 10-minute schedule. The second sets it to every 1 minute.

#### Step 2: Set retention to 28 days

```bash
sudo sed -i 's/^HISTORY=.*/HISTORY=28/' /etc/sysstat/sysstat
```

#### Step 3: Reload and restart

```bash
sudo systemctl daemon-reload
sudo systemctl restart sysstat-collect.timer
```

#### Step 4: Verify

```bash
systemctl list-timers sysstat-collect.timer
```

Wait a couple of minutes, then confirm data is being collected every minute:

```bash
sar
```

### RHEL Configuration File Location

| RHEL Version | Config File |
|--------------|-------------|
| RHEL 6 | `/etc/sysconfig/sysstat` |
| RHEL 7 | `/etc/sysstat/sysstat` |
| RHEL 8 | `/etc/sysstat/sysstat` |
| RHEL 9 | `/etc/sysstat/sysstat` |
| RHEL 10 | `/etc/sysstat/sysstat` |

> **Note:** On RHEL 6, the config file is at `/etc/sysconfig/sysstat` and the retention variable is `HISTORY`. On RHEL 7+, it moved to `/etc/sysstat/sysstat` but the variable name stays the same.


