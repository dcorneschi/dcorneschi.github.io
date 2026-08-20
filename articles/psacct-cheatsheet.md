# psacct / acct Cheatsheet

Process accounting (`psacct` on RHEL, `acct` on Ubuntu/Debian) tracks all commands executed by users, connection times, and resource usage. Useful for auditing, security investigations, and usage reporting.

## Installation

```bash
# RHEL/CentOS/Fedora
sudo dnf install -y psacct
sudo systemctl enable --now psacct

# Ubuntu/Debian
sudo apt install -y acct
sudo systemctl enable --now acct

# Verify service is running
systemctl status psacct    # RHEL
systemctl status acct      # Ubuntu
```

## Core Commands

| Command | Purpose |
|---------|---------|
| `ac` | Display connect time (login duration) statistics |
| `lastcomm` | Show last commands executed |
| `sa` | Summarize accounting information |
| `accton` | Turn process accounting on/off |
| `dump-acct` | Print raw accounting data |
| `dump-utmp` | Print utmp data in human-readable format |

## lastcomm — Last Commands Executed

Shows previously executed commands with user, terminal, and timestamp.

```bash
# Show all commands (most recent first)
lastcomm

# Show commands by a specific user
lastcomm --user root
lastcomm --user john

# Show a specific command
lastcomm --command vi
lastcomm --command sudo
lastcomm --command ssh

# Show commands on a specific terminal
lastcomm --tty pts/0

# Combine filters
lastcomm --user root --command rm

# Read from a specific accounting file
lastcomm --file /var/account/pacct

# Show commands with flags
# F = forked but didn't exec
# S = ran as superuser
# X = terminated with signal
# D = dumped core
lastcomm | head -20
```

### lastcomm Output Format

```
command   flags  user     tty      time
vi              john     pts/0    0.02 secs Thu Aug 15 10:23
rm         S    root     pts/1    0.01 secs Thu Aug 15 10:21
sudo       S    john     pts/0    0.00 secs Thu Aug 15 10:20
```

| Flag | Meaning |
|------|---------|
| `S` | Ran with superuser (root) privileges |
| `F` | Forked but did not exec |
| `X` | Terminated by a signal |
| `D` | Produced a core dump |
| `C` | Ran in a cgroup |

## ac — Connect Time Statistics

Shows how long users have been logged in.

```bash
# Total connect time (hours) for all users
ac

# Per-user connect time
ac -p

# Daily totals
ac -d

# Connect time for a specific user
ac -p john
ac john

# Daily totals for a specific user
ac -d john

# Read from a specific wtmp file
ac -f /var/log/wtmp
```

### Example Output

```bash
$ ac -dp
        john                              5.23
        root                              1.45
        deploy                            0.52
Aug 15  total        7.20
```

## sa — Summarize Process Accounting

Provides summary statistics of commands: CPU time, I/O, and call counts.

```bash
# Summary by command (default)
sa

# Sort by number of calls
sa -c

# Sort by CPU time (system + user)
sa -k

# Sort by average CPU time per call
sa -K

# Sort by real time
sa -n

# Show per-user summaries
sa -u

# Show per-user with command details
sa -m

# Show I/O statistics
sa -a

# Print percentages
sa -c

# Don't merge commands into "other"
sa -a -c

# Read from specific accounting file
sa /var/account/pacct

# Merge into summary file (savacct/usracct)
sa -s
```

### Example Output

```bash
$ sa -c
     124   0.52re    0.01cp         0avio     8k   bash
      89   0.31re    0.00cp         0avio     4k   ls
      45   0.18re    0.02cp         0avio    12k   grep
      23   1.45re    0.15cp         0avio    64k   python3
      12   0.05re    0.00cp         0avio     4k   cat
```

| Column | Meaning |
|--------|---------|
| Count | Number of times command ran |
| `re` | Real (elapsed) time in minutes |
| `cp` | CPU time (user + system) in minutes |
| `avio` | Average I/O operations |
| `k` | Average memory usage in KB |

## accton — Control Process Accounting

```bash
# Turn on accounting (default file: /var/account/pacct)
sudo accton /var/account/pacct

# Turn off accounting
sudo accton off

# Check if accounting is active
sudo accton

# Use a different accounting file
sudo accton /var/log/account/custom.acct
```

## dump-acct — Raw Accounting Data

```bash
# Dump raw accounting records (human-readable)
dump-acct /var/account/pacct

# Dump with specific format
dump-acct --raw /var/account/pacct

# Dump recent records
dump-acct /var/account/pacct | tail -50
```

## Accounting Files

| File | Purpose | Distribution |
|------|---------|-------------|
| `/var/account/pacct` | Process accounting data | RHEL |
| `/var/log/account/pacct` | Process accounting data | Ubuntu/Debian |
| `/var/account/savacct` | Summary of sa data | RHEL |
| `/var/account/usracct` | Per-user summary | RHEL |
| `/var/log/wtmp` | Login/logout records (used by `ac`) | All |

## Practical Examples

### Find Who Ran a Specific Command

```bash
# Who ran 'rm' recently?
lastcomm --command rm

# Who used 'sudo' today?
lastcomm --command sudo

# Who ran 'shutdown'?
lastcomm --command shutdown
lastcomm --command reboot
```

### Find What a User Executed

```bash
# All commands by user 'john'
lastcomm --user john

# Count commands per user
lastcomm | awk '{print $3}' | sort | uniq -c | sort -rn

# Commands by user with timestamps
lastcomm --user john | head -50
```

### Track Privilege Escalation

```bash
# Find all commands run as superuser
lastcomm | grep ' S '

# Find all sudo usage
lastcomm --command sudo

# Find su usage
lastcomm --command su
```

### Resource Usage Analysis

```bash
# Top CPU-consuming commands
sa -k | head -20

# Top commands by frequency
sa -c | head -20

# Per-user resource consumption
sa -m

# Commands using most I/O
sa -a | sort -k4 -rn | head -20
```

### Security Investigation

```bash
# What happened in the last hour (combine with timestamps)
lastcomm | head -100

# Find file deletion commands
lastcomm --command rm
lastcomm --command shred
lastcomm --command unlink

# Find network-related commands
lastcomm --command curl
lastcomm --command wget
lastcomm --command ssh
lastcomm --command scp

# Find compilation (potential exploit building)
lastcomm --command gcc
lastcomm --command make
lastcomm --command cc
```

### User Login Duration Report

```bash
# Total hours per user this month
ac -p

# Daily breakdown
ac -d

# Specific user's login pattern
ac -d john

# Compare user activity
ac -p | sort -k1 -rn
```

## Log Rotation

Process accounting files can grow large. Rotate them periodically:

```bash
# Rotate the accounting file
sudo mv /var/account/pacct /var/account/pacct.1
sudo accton /var/account/pacct

# Or use logrotate
cat << 'EOF' | sudo tee /etc/logrotate.d/psacct
/var/account/pacct {
    daily
    rotate 7
    compress
    notifempty
    missingok
    postrotate
        /usr/sbin/accton /var/account/pacct
    endscript
}
EOF

# Check current file size
ls -lh /var/account/pacct
du -sh /var/account/
```

## Comparison with Other Auditing Tools

| Feature | psacct/acct | auditd | lastlog/wtmp |
|---------|:-----------:|:------:|:------------:|
| Track all commands | Yes | With rules | No |
| User login time | Yes (`ac`) | Yes | Yes |
| Resource usage | Yes (`sa`) | No | No |
| Real-time monitoring | No | Yes | No |
| File access tracking | No | Yes | No |
| Lightweight | Yes | Heavier | Yes |
| Compliance (PCI/HIPAA) | Partial | Full | No |
| Per-user command history | Yes | With rules | No |

## Troubleshooting

### Accounting Not Working

```bash
# Check if service is running
systemctl status psacct    # RHEL
systemctl status acct      # Ubuntu

# Check if accounting file exists
ls -la /var/account/pacct        # RHEL
ls -la /var/log/account/pacct    # Ubuntu

# Manually enable accounting
sudo accton /var/account/pacct

# Check kernel support
cat /boot/config-$(uname -r) | grep CONFIG_BSD_PROCESS_ACCT
# Should show: CONFIG_BSD_PROCESS_ACCT=y
```

### lastcomm Shows Nothing

```bash
# Verify accounting is on
sudo accton

# Check file has data
ls -la /var/account/pacct
# If size is 0, accounting was just enabled — wait for commands to be logged

# Run a test command, then check
ls /tmp
lastcomm --command ls
```

### Large Accounting File

```bash
# Check size
du -sh /var/account/

# Rotate manually
sudo mv /var/account/pacct /var/account/pacct.old
sudo accton /var/account/pacct

# Compress old file
gzip /var/account/pacct.old

# Set up logrotate (see Log Rotation section above)
```

## Quick Reference

| Action | Command |
|--------|---------|
| Install (RHEL) | `sudo dnf install psacct && sudo systemctl enable --now psacct` |
| Install (Ubuntu) | `sudo apt install acct && sudo systemctl enable --now acct` |
| Last commands | `lastcomm` |
| Commands by user | `lastcomm --user username` |
| Specific command | `lastcomm --command sudo` |
| Login times | `ac -p` |
| Daily login totals | `ac -d` |
| Command summary | `sa` |
| Top commands by CPU | `sa -k` |
| Per-user summary | `sa -m` |
| Enable accounting | `sudo accton /var/account/pacct` |
| Disable accounting | `sudo accton off` |
| Dump raw data | `dump-acct /var/account/pacct` |
