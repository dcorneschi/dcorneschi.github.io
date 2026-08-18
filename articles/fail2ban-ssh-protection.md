# Protect SSH with fail2ban

fail2ban monitors log files for repeated authentication failures and temporarily bans offending IP addresses using firewall rules. It significantly reduces brute-force SSH attacks with minimal configuration.

## How It Works

1. fail2ban watches log files (e.g., `/var/log/auth.log`, `/var/log/secure`)
2. When an IP exceeds the failure threshold within a time window, it triggers a ban
3. A firewall rule (iptables, nftables, or firewalld) blocks the IP for a configurable duration
4. After the ban time expires, the IP is automatically unbanned

## Installation

### RHEL 8/9 (Rocky, AlmaLinux, CentOS Stream)

fail2ban is available from EPEL:

```bash
dnf install epel-release -y
dnf install fail2ban fail2ban-firewalld -y
```

> **Note:** The `fail2ban-firewalld` package integrates with firewalld instead of raw iptables.

### Ubuntu 22.04 / 24.04

```bash
apt update
apt install fail2ban -y
```

## Enable and Start the Service

```bash
systemctl enable fail2ban --now
systemctl status fail2ban
```

## Configuration Files

| File | Purpose |
|------|---------|
| `/etc/fail2ban/fail2ban.conf` | Main daemon settings (log level, socket, PID) |
| `/etc/fail2ban/jail.conf` | Default jail definitions — **do not edit directly** |
| `/etc/fail2ban/jail.local` | Local overrides — **create this file** |
| `/etc/fail2ban/jail.d/*.conf` | Drop-in jail configs (highest priority) |
| `/etc/fail2ban/filter.d/` | Regex filter definitions |
| `/etc/fail2ban/action.d/` | Ban/unban action scripts |

Always use `jail.local` or files in `jail.d/` for customization. Package updates overwrite `jail.conf`.

### Enable the Log File

By default fail2ban may log to syslog. To use a dedicated log file, edit `/etc/fail2ban/fail2ban.conf`:

```bash
vi /etc/fail2ban/fail2ban.conf
```

```ini
logtarget = /var/log/fail2ban.log
```

## Configure the SSH Jail

Create `/etc/fail2ban/jail.local`:

```bash
vi /etc/fail2ban/jail.local
```

### RHEL Configuration

```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5
banaction = firewallcmd-rich-rules[actiontype=<multiport>]
banaction_allports = firewallcmd-rich-rules[actiontype=<allports>]

[sshd]
enabled = true
port = ssh
logpath = /var/log/secure
backend = auto
```

### RHEL Drop-in Jail File (iptables + sendmail)

Alternatively, create a drop-in file at `/etc/fail2ban/jail.d/sshd.local` for systems using iptables:

```bash
vi /etc/fail2ban/jail.d/sshd.local
```

```ini
[ssh-iptables]
enabled  = true
filter   = sshd
action   = iptables[name=SSH, port=ssh, protocol=tcp]
           sendmail-whois[name=SSH, dest=root, sender=fail2ban@example.com]
logpath  = /var/log/secure
maxretry = 5
```

### Ubuntu Configuration

```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log
backend = systemd
```

### Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `bantime` | How long an IP stays banned | `10m` |
| `findtime` | Time window to count failures | `10m` |
| `maxretry` | Failures before ban triggers | `5` |
| `ignoreip` | IPs/CIDRs that are never banned | `127.0.0.1/8 ::1` |
| `port` | Port(s) to block (name or number) | `ssh` |
| `backend` | Log reading method (`auto`, `systemd`, `polling`) | `auto` |

## Apply Configuration

Restart fail2ban after changes:

```bash
systemctl restart fail2ban
```

## Whitelist Trusted IPs

Add trusted networks to `ignoreip` so you never lock yourself out:

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 192.168.1.0/24 10.0.0.0/8
```

## Using fail2ban-client

### Check Status

```bash
# Overall status
fail2ban-client status

# SSH jail status (shows banned IPs and current count)
fail2ban-client status sshd
```

### Manually Ban/Unban

```bash
# Ban an IP
fail2ban-client set sshd banip 203.0.113.50

# Unban an IP
fail2ban-client set sshd unbanip 203.0.113.50
```

### Reload Configuration

```bash
# Reload without restarting
fail2ban-client reload

# Reload a specific jail
fail2ban-client reload sshd
```

## Verify Bans Are Working

Check the firewall for fail2ban rules:

### RHEL (firewalld)

```bash
firewall-cmd --list-rich-rules
```

### Ubuntu (iptables/nftables)

```bash
iptables -L f2b-sshd -n --line-numbers
```

### Verify with Packet Counters

Use `-nvL` to see packet and byte counters for banned IPs:

```bash
iptables -nvL
```

## Aggressive Configuration Example

For servers under heavy attack:

```ini
[sshd]
enabled = true
port = ssh
maxretry = 3
findtime = 5m
bantime = 24h
```

## Incremental Ban Time

fail2ban 0.11+ supports increasing ban duration for repeat offenders:

```ini
[DEFAULT]
bantime.increment = true
bantime.factor = 2
bantime.maxtime = 4w
bantime.overalljails = true
```

This doubles the ban time on each repeat offense, up to 4 weeks.

## Monitor fail2ban Logs

```bash
# Live log monitoring
tail -f /var/log/fail2ban.log

# Recent bans
grep "Ban" /var/log/fail2ban.log | tail -20

# Count bans per IP
grep "Ban" /var/log/fail2ban.log | awk '{print $NF}' | sort | uniq -c | sort -rn | head -10
```

## Custom SSH Port

If SSH runs on a non-standard port (e.g., 2222):

```ini
[sshd]
enabled = true
port = 2222
logpath = /var/log/secure
maxretry = 5
```

## Protect Additional Services

fail2ban can protect more than SSH. Common jails:

```ini
[apache-auth]
enabled = true
port = http,https
logpath = /var/log/httpd/error_log
maxretry = 5

[nginx-http-auth]
enabled = true
port = http,https
logpath = /var/log/nginx/error.log
maxretry = 5

[postfix]
enabled = true
port = smtp,465,587
logpath = /var/log/maillog
maxretry = 5
```

## Email Notifications

Send alerts when IPs get banned:

```ini
[DEFAULT]
destemail = admin@example.com
sender = fail2ban@example.com
mta = sendmail
action = %(action_mwl)s
```

Action shortcuts:

| Action | Behavior |
|--------|----------|
| `%(action_)s` | Ban only (default) |
| `%(action_mw)s` | Ban + email with whois |
| `%(action_mwl)s` | Ban + email with whois + log lines |

## Troubleshooting

| Issue | Fix |
|-------|-----|
| fail2ban not banning | Check `logpath` matches actual log location |
| Wrong backend on RHEL 9 | Use `backend = systemd` if journal-only (no `/var/log/secure`) |
| Bans not appearing in firewall | Verify `banaction` matches your firewall (firewalld vs iptables) |
| Locked yourself out | Access via console, then `fail2ban-client set sshd unbanip <your-ip>` |
| Service fails to start | Check syntax: `fail2ban-client -t` |
| Regex not matching | Test filter: `fail2ban-regex /var/log/secure /etc/fail2ban/filter.d/sshd.conf` |

## Useful Commands Reference

```bash
# Test a filter regex against a log file
fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf

# Test and show all matched lines
fail2ban-regex /var/log/secure /etc/fail2ban/filter.d/sshd.conf --print-all-matched

# Show all jails
fail2ban-client status

# Show banned IPs for sshd
fail2ban-client get sshd banned

# Check current settings for a jail
fail2ban-client get sshd bantime
fail2ban-client get sshd maxretry
fail2ban-client get sshd findtime

# Flush all bans (all jails)
fail2ban-client unban --all
```

## Generating Reports

Count SSH failed attempts per day:

```bash
cat /var/log/secure* | grep 'Failed password' | grep sshd | awk '{print $1,$2}' | sort -k 1,1 | uniq -c
```

Top 10 most banned IPs:

```bash
grep "Ban" /var/log/fail2ban.log* | awk '{print $NF}' | sort | uniq -c | sort -rn | head
```

Failed SSH attempts by source IP:

```bash
grep 'Failed password' /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -20
```
