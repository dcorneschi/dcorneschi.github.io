# Timezone Configuration on RHEL and Ubuntu

Configuring the correct timezone ensures accurate timestamps in logs, cron jobs, databases, and application events. Modern Linux systems use `timedatectl` (part of systemd) as the primary tool, with `/etc/localtime` as the symlink that determines the active timezone.

## Check Current Timezone

```bash
# Show full time/date/timezone status
timedatectl

# Show just the timezone
timedatectl show --property=Timezone --value

# Alternative methods
cat /etc/timezone                         # Ubuntu/Debian only
ls -la /etc/localtime                     # Shows symlink target
date +%Z                                  # Short timezone abbreviation (EST, UTC, etc.)
date +%:z                                 # UTC offset (+05:30, -04:00)
date                                      # Full date with timezone
```

Example `timedatectl` output:

```
               Local time: Thu 2026-08-13 14:30:00 UTC
           Universal time: Thu 2026-08-13 14:30:00 UTC
                 RTC time: Thu 2026-08-13 14:30:00
                Time zone: UTC (UTC, +0000)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

## Set Timezone

### Using timedatectl (Preferred — RHEL & Ubuntu)

```bash
# Set timezone
sudo timedatectl set-timezone Europe/London
sudo timedatectl set-timezone America/New_York
sudo timedatectl set-timezone Asia/Tokyo
sudo timedatectl set-timezone UTC

# Verify
timedatectl
```

> No reboot or service restart required. Changes take effect immediately for new processes. Running processes (including shells) pick up the change on next timezone lookup.

### Using Symlink (Manual Method)

```bash
# Link /etc/localtime to the desired zone
sudo ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime

# On Ubuntu/Debian, also update /etc/timezone
echo "Europe/London" | sudo tee /etc/timezone

# Apply (Ubuntu/Debian)
sudo dpkg-reconfigure -f noninteractive tzdata
```

### Using tzdata Reconfigure (Ubuntu/Debian)

```bash
# Interactive — presents a menu
sudo dpkg-reconfigure tzdata

# Non-interactive — preseed and reconfigure
echo "tzdata tzdata/Areas select Europe" | sudo debconf-set-selections
echo "tzdata tzdata/Zones/Europe select London" | sudo debconf-set-selections
sudo DEBIAN_FRONTEND=noninteractive dpkg-reconfigure -f noninteractive tzdata
```

### Legacy RHEL (6 and Older)

```bash
# RHEL 5-6 / CentOS 5-6 (no systemd, no timedatectl)
# Remove existing localtime and create new symlink
sudo rm -f /etc/localtime
sudo ln -s /usr/share/zoneinfo/Europe/Bucharest /etc/localtime
date

# Edit /etc/sysconfig/clock
sudo vi /etc/sysconfig/clock
# ZONE="Europe/Bucharest"
# UTC=true

# Apply
source /etc/sysconfig/clock
```

## List Available Timezones

### Common Timezones

| Timezone | Region |
|----------|--------|
| `America/New_York` | US Eastern Time (EST/EDT) |
| `America/Chicago` | US Central Time (CST/CDT) |
| `America/Denver` | US Mountain Time (MST/MDT) |
| `America/Los_Angeles` | US Pacific Time (PST/PDT) |
| `Europe/London` | UK (GMT/BST) |
| `Europe/Paris` | Central Europe (CET/CEST) |
| `Europe/Berlin` | Germany (CET/CEST) |
| `Asia/Tokyo` | Japan (JST) |
| `Asia/Shanghai` | China (CST) |
| `Asia/Kolkata` | India (IST) |
| `Australia/Sydney` | Eastern Australia (AEST/AEDT) |
| `UTC` | Coordinated Universal Time |

```bash
# List all timezones
timedatectl list-timezones

# Filter by region
timedatectl list-timezones | grep America
timedatectl list-timezones | grep Europe
timedatectl list-timezones | grep Asia

# Search for a city
timedatectl list-timezones | grep -i london
timedatectl list-timezones | grep -i new_york
timedatectl list-timezones | grep -i tokyo

# Count available timezones
timedatectl list-timezones | wc -l

# Alternative: browse the zoneinfo directory
ls /usr/share/zoneinfo/
ls /usr/share/zoneinfo/America/
ls /usr/share/zoneinfo/Europe/
```

## NTP Synchronization

### Check NTP Status

```bash
# systemd-timesyncd (default on Ubuntu)
timedatectl
timedatectl timesync-status
systemctl status systemd-timesyncd

# chronyd (default on RHEL 8+)
chronyc tracking
chronyc sources -v
systemctl status chronyd
```

### Enable/Disable NTP

```bash
# Enable NTP synchronization
sudo timedatectl set-ntp true

# Disable NTP (for manual time setting)
sudo timedatectl set-ntp false
```

### Configure chronyd (RHEL)

```bash
# Install chrony (if not already present)
sudo dnf install chrony

# Start and enable
sudo systemctl start chronyd && sudo systemctl enable chronyd
sudo systemctl status chronyd

# Main config file
sudo vi /etc/chrony.conf

# Common settings
# server ntp1.example.com iburst
# pool 2.rhel.pool.ntp.org iburst

# Open NTP port in firewall
sudo firewall-cmd --permanent --add-service=ntp
sudo firewall-cmd --reload

# Restart after config changes
sudo systemctl restart chronyd

# Check synchronization
chronyc tracking
chronyc sources
chronyc sources -v
chronyc sourcestats
```

### Configure systemd-timesyncd (Ubuntu)

```bash
# Config file
sudo vi /etc/systemd/timesyncd.conf

# Example
# [Time]
# NTP=ntp1.example.com ntp2.example.com
# FallbackNTP=ntp.ubuntu.com

# Restart
sudo systemctl restart systemd-timesyncd

# Status
timedatectl timesync-status
```

### Legacy NTP Commands (ntpdate / ntpq / ntpstat)

These tools are from the older `ntp` package (replaced by chrony on RHEL 7+ and timesyncd on Ubuntu). Still useful on legacy systems or for quick one-off checks:

```bash
# Query NTP server without setting the clock
ntpdate -q pool.ntp.org

# Set the clock via NTP (ntpd/chronyd must be stopped first)
sudo systemctl stop chronyd    # or: systemctl stop ntpd
sudo ntpdate 0.centos.pool.ntp.org
sudo systemctl start chronyd

# Use unprivileged port (> 1023) — useful behind strict firewalls
ntpdate -u 0.centos.pool.ntp.org

# Debug mode (don't adjust clock, just show what would happen)
ntpdate -d 0.centos.pool.ntp.org

# Show NTP synchronization status
ntpstat

# List NTP peers and their state (numeric format)
ntpq -pn

# Install legacy ntp tools (if needed)
sudo dnf install ntp ntpstat       # RHEL
sudo apt install ntpdate ntpstat   # Ubuntu (older releases)
```

> **Note:** `ntpdate` is deprecated on modern systems. Use `chronyc makestep` (RHEL) or restart `systemd-timesyncd` (Ubuntu) for one-shot sync instead.

## Hardware Clock (RTC)

```bash
# Show hardware clock time
sudo hwclock --show
sudo hwclock -r

# Set hardware clock to a specific date/time
sudo hwclock --set --date="2026-08-13 14:30:00"

# Set hardware clock from system clock (system → hardware)
sudo hwclock --systohc
sudo hwclock -w

# Set system clock from hardware clock (hardware → system)
sudo hwclock --hctosys
sudo hwclock -s

# Set hardware clock to UTC (recommended for Linux)
sudo timedatectl set-local-rtc false

# Set hardware clock to local time (needed for Windows dual-boot)
sudo timedatectl set-local-rtc true

# Check if RTC is set to UTC or local
timedatectl | grep "RTC in local TZ"
```

> **Best practice:** Keep the hardware clock in UTC (`set-local-rtc false`). Only set to local time if dual-booting with Windows (which expects local time in the RTC).

## Set Time Manually

```bash
# Set date and time manually (NTP must be disabled first)
sudo timedatectl set-ntp false
sudo timedatectl set-time "2026-08-13 14:30:00"

# Set just date
sudo timedatectl set-time "2026-08-13"

# Set just time
sudo timedatectl set-time "14:30:00"

# Re-enable NTP after
sudo timedatectl set-ntp true

# Legacy: using date command
sudo date -s "2026-08-13 14:30:00"
sudo date --set="14:30:00"
```

## Per-User / Per-Process Timezone

The `TZ` environment variable overrides the system timezone for a single command or user session:

```bash
# Run a command in a different timezone
TZ=America/New_York date
TZ=Asia/Tokyo date
TZ=UTC date

# Compare multiple timezones
for tz in UTC America/New_York Europe/London Asia/Tokyo; do
    printf "%-20s %s\n" "$tz" "$(TZ=$tz date '+%Y-%m-%d %H:%M:%S %Z')"
done

# Set for current shell session
export TZ=Europe/Berlin
date   # Shows Berlin time

# Set for a specific user (add to ~/.bashrc or ~/.profile)
echo 'export TZ="America/Los_Angeles"' >> ~/.bashrc

# Use in cron jobs (cron uses system timezone by default)
TZ=UTC 0 5 * * * /path/to/script.sh

# Set timezone for a systemd service
# [Service]
# Environment="TZ=UTC"
```

## Cloud-Init Timezone

```yaml
#cloud-config
timezone: Europe/London
```

Or using runcmd:

```yaml
#cloud-config
runcmd:
  - timedatectl set-timezone Europe/London
```

## Ansible Timezone

```yaml
- name: Set timezone
  community.general.timezone:
    name: Europe/London

# Or using timedatectl directly
- name: Set timezone via command
  command: timedatectl set-timezone Europe/London
  changed_when: false
```

## Docker Timezone

```dockerfile
# Method 1: Set TZ environment variable
ENV TZ=Europe/London
RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Method 2: Bind mount host timezone (docker run)
# docker run -v /etc/localtime:/etc/localtime:ro -v /etc/timezone:/etc/timezone:ro image

# Method 3: Install tzdata non-interactively
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y tzdata && \
    ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime && \
    echo "Europe/London" > /etc/timezone && \
    dpkg-reconfigure -f noninteractive tzdata
```

Docker Compose:

```yaml
services:
  app:
    image: myapp
    environment:
      - TZ=Europe/London
    volumes:
      - /etc/localtime:/etc/localtime:ro
```

## Terraform / Packer

```hcl
# In a Packer provisioner or Terraform user_data
provisioner "shell" {
  inline = [
    "sudo timedatectl set-timezone UTC",
  ]
}
```

## One-Liners

```bash
# Quick set to UTC
sudo timedatectl set-timezone UTC

# Show current timezone name only
cat /etc/timezone 2>/dev/null || timedatectl show -p Timezone --value

# Show UTC offset
date +%:z

# Show epoch seconds (seconds since 1970-01-01 00:00:00 UTC)
date +%s

# Display calendar for a year
cal 2026

# Display calendar for current month
cal

# Convert a timestamp between timezones
TZ=America/New_York date -d "TZ=\"UTC\" 2026-08-13 14:00:00"

# Find timezone for a country code
timedatectl list-timezones | grep -i "US\|America"

# Check if NTP is synchronized
timedatectl | grep -i "synchronized"

# Show all timezone abbreviations in use
zdump /usr/share/zoneinfo/posix/* 2>/dev/null | grep "$(date +%Y)" | awk '{print $NF}' | sort -u

# Compare system time with NTP (RHEL)
chronyc tracking | grep "System time"

# Show timezone of a remote server
ssh user@server 'timedatectl show -p Timezone --value'

# Set timezone in a script (works on both RHEL and Ubuntu)
#!/bin/bash
TIMEZONE="Europe/London"
if command -v timedatectl &>/dev/null; then
    sudo timedatectl set-timezone "$TIMEZONE"
else
    sudo ln -sf "/usr/share/zoneinfo/$TIMEZONE" /etc/localtime
    echo "$TIMEZONE" | sudo tee /etc/timezone
fi
```

## Troubleshooting

### Timezone Doesn't Change After Setting

```bash
# Check if /etc/localtime is a proper symlink
ls -la /etc/localtime
# Should point to /usr/share/zoneinfo/...

# If it's a regular file (not a symlink), fix it
sudo rm /etc/localtime
sudo ln -s /usr/share/zoneinfo/Europe/London /etc/localtime

# Running processes won't see the change until restarted
# (or they re-read /etc/localtime)
sudo systemctl restart rsyslog    # Fix log timestamps
sudo systemctl restart cron       # Fix cron schedule
```

### Logs Show Wrong Timezone

```bash
# journalctl uses UTC internally, displays in local time
journalctl --since "1 hour ago"

# Force UTC display in journalctl
journalctl --utc

# rsyslog — restart after timezone change
sudo systemctl restart rsyslog

# Check if TZ is overriding system timezone
env | grep TZ
```

### Cron Jobs Run at Wrong Time

```bash
# Cron uses the system timezone at the time crond starts
# After a timezone change, restart cron:
sudo systemctl restart cron       # Ubuntu
sudo systemctl restart crond      # RHEL

# Or set timezone explicitly in crontab
CRON_TZ=UTC
0 5 * * * /path/to/script.sh
```

### NTP Not Syncing

```bash
# Check if NTP is enabled
timedatectl | grep "NTP service"

# Enable NTP
sudo timedatectl set-ntp true

# Check chrony (RHEL)
chronyc tracking
chronyc activity
sudo systemctl status chronyd

# Check timesyncd (Ubuntu)
systemctl status systemd-timesyncd
timedatectl timesync-status

# Check firewall (NTP uses UDP port 123)
sudo firewall-cmd --list-services | grep ntp    # RHEL
sudo ufw status | grep 123                      # Ubuntu

# Force NTP sync now
sudo chronyc makestep     # RHEL
sudo systemctl restart systemd-timesyncd   # Ubuntu
```

### "RTC in local TZ: yes" Warning

```bash
# Warning: The system is configured to read the RTC time in the local time zone.
# This can cause various problems including incorrect DST transitions.
# Fix: set RTC to UTC
sudo timedatectl set-local-rtc false
```

## RHEL vs Ubuntu Differences

| Feature | RHEL 8/9 | Ubuntu 22.04/24.04 |
|---------|----------|-------------------|
| Primary tool | `timedatectl` | `timedatectl` |
| NTP daemon | chronyd | systemd-timesyncd |
| NTP config | `/etc/chrony.conf` | `/etc/systemd/timesyncd.conf` |
| Timezone file | `/etc/localtime` (symlink) | `/etc/localtime` (symlink) + `/etc/timezone` |
| tzdata package | `tzdata` | `tzdata` |
| Reconfigure | — | `dpkg-reconfigure tzdata` |
| Legacy config | `/etc/sysconfig/clock` (RHEL 6) | — |
| NTP port | UDP 123 (chronyd) | UDP 123 or NTS (timesyncd) |
| Default timezone | UTC (server installs) | UTC (server installs) |

## Important Files

| File | Purpose |
|------|---------|
| `/etc/localtime` | Symlink to active timezone file — determines system timezone |
| `/etc/timezone` | Timezone name in text (Ubuntu/Debian only) |
| `/usr/share/zoneinfo/` | Timezone database (all zone files) |
| `/etc/chrony.conf` | Chrony NTP config (RHEL) |
| `/etc/ntp.conf` | Legacy NTP daemon config (ntpd — RHEL 6 and older) |
| `/etc/systemd/timesyncd.conf` | systemd-timesyncd config (Ubuntu) |
| `/etc/sysconfig/clock` | Legacy timezone config (RHEL 6) |
| `/etc/adjtime` | Hardware clock drift adjustment |

## Best Practices

1. **Use UTC on servers** — avoids DST confusion in logs, cron, and distributed systems
2. **Use `timedatectl`** — works consistently on both RHEL and Ubuntu (systemd-based)
3. **Keep RTC in UTC** — set `set-local-rtc false` unless dual-booting Windows
4. **Enable NTP** — clock drift causes certificate errors, auth failures, and log correlation issues
5. **Restart services after timezone changes** — especially cron, rsyslog, and databases
6. **Use TZ variable for per-app overrides** — don't change system timezone for one application
7. **Set timezone in automation** — include `timedatectl set-timezone` in cloud-init, Packer, or Ansible
8. **Update tzdata regularly** — timezone rules change (countries modify DST dates)
9. **Use canonical names** — `Europe/London` not `GMT` or `BST` (handles DST correctly)
10. **Log in UTC** — applications should log in UTC; display in local time at the presentation layer
