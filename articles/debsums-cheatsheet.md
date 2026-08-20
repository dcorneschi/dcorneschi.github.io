# debsums Cheatsheet

debsums verifies the integrity of installed package files by comparing their MD5 checksums against the values recorded in `/var/lib/dpkg/info/*.md5sums`. Useful for detecting corrupted, modified, or tampered files.

## Installation

```bash
sudo apt install -y debsums
```

## Basic Usage

### Verify All Installed Packages

```bash
# Check all files from all packages
sudo debsums

# Quiet mode — only show errors
sudo debsums -s

# Show only files that FAILED verification
sudo debsums -c

# Show only changed configuration files
sudo debsums -ce

# Show files that match (OK)
sudo debsums -a
```

### Verify a Specific Package

```bash
# Check a single package
sudo debsums nginx

# Check multiple packages
sudo debsums nginx openssh-server coreutils

# Check with full output (OK and FAILED)
sudo debsums -a nginx
```

## Output Interpretation

```bash
# Example output:
/usr/sbin/nginx                                    OK
/usr/share/doc/nginx/changelog.Debian.gz           OK
/etc/nginx/nginx.conf                              CHANGED
/usr/lib/nginx/modules/ngx_http_gzip_module.so     FAILED
```

| Status | Meaning |
|--------|---------|
| OK | File matches the expected checksum |
| CHANGED | Configuration file was modified (expected) |
| FAILED | Non-config file doesn't match (possible corruption or tampering) |

## Configuration File Checks

```bash
# Show only changed config files
sudo debsums -ce

# Show config files that match
sudo debsums -ae | grep "OK$"

# Ignore config file changes (check only non-config files)
sudo debsums --no-config

# List config files for a package
dpkg-query --showformat='${Conffiles}\n' --show nginx
```

## Generate Missing md5sums

Some packages don't ship with md5sums files. Generate them from the installed .deb:

```bash
# Generate md5sums for packages that are missing them
sudo debsums --generate=missing

# Generate md5sums for ALL packages (overwrite existing)
sudo debsums --generate=all

# Generate for a specific package using apt
sudo apt-get install --reinstall -d nginx
sudo debsums --generate=missing nginx

# Check which packages are missing md5sums
sudo debsums -l
```

## List Packages Without md5sums

```bash
# Show packages that have no md5sums file
sudo debsums -l

# Count packages without md5sums
sudo debsums -l | wc -l
```

## Practical Workflows

### Security Audit — Find Tampered Binaries

```bash
# Check all non-config files for unexpected changes
sudo debsums -c 2>/dev/null

# Check only binaries in /usr/bin and /usr/sbin
sudo debsums -c 2>/dev/null | grep -E '^/(usr/)?(s)?bin/'

# Check system libraries
sudo debsums -c 2>/dev/null | grep '/lib/'
```

### Find Modified Config Files

```bash
# List all configs that have been changed from defaults
sudo debsums -ce 2>/dev/null

# Show which package owns each changed config
sudo debsums -ce 2>/dev/null | while read -r file; do
    pkg=$(dpkg -S "$file" 2>/dev/null | cut -d: -f1)
    echo "$pkg: $file"
done
```

### Verify After System Compromise

```bash
# Full integrity check (log to file)
sudo debsums -c > /tmp/debsums-failed.txt 2>&1

# Check critical packages specifically
sudo debsums openssh-server openssh-client sudo coreutils login passwd

# Verify PAM modules
sudo debsums libpam-modules libpam-runtime

# Check cron and at
sudo debsums cron at
```

### Reinstall Corrupted Packages

```bash
# Find failed files and their packages
sudo debsums -c 2>/dev/null | while read -r file; do
    dpkg -S "$file" 2>/dev/null
done | cut -d: -f1 | sort -u

# Reinstall a package with corrupted files
sudo apt install --reinstall nginx

# Reinstall all packages with failed checksums
sudo debsums -c 2>/dev/null | while read -r file; do
    dpkg -S "$file" 2>/dev/null
done | cut -d: -f1 | sort -u | xargs sudo apt install --reinstall
```

### Compare Before and After

```bash
# Take a baseline snapshot
sudo debsums -a > /root/debsums-baseline.txt

# Later, compare
sudo debsums -a > /tmp/debsums-current.txt
diff /root/debsums-baseline.txt /tmp/debsums-current.txt
```

### Check Specific Directories

```bash
# Verify files in /usr/bin only
sudo debsums -c 2>/dev/null | grep '^/usr/bin/'

# Verify files in /etc only
sudo debsums -ce 2>/dev/null

# Verify shared libraries
sudo debsums -c 2>/dev/null | grep '\.so'
```

## Cron Job for Regular Verification

```bash
# Create a weekly integrity check
cat << 'EOF' | sudo tee /etc/cron.weekly/debsums-check
#!/bin/bash
LOGFILE="/var/log/debsums-check.log"
echo "=== debsums check $(date) ===" >> "$LOGFILE"
debsums -c >> "$LOGFILE" 2>&1
if [ $? -ne 0 ]; then
    echo "WARNING: debsums found integrity issues" | mail -s "debsums alert" root
fi
EOF
sudo chmod +x /etc/cron.weekly/debsums-check
```

## Integration with Other Tools

### With AIDE (Advanced Intrusion Detection)

```bash
# debsums checks against package md5sums (what was shipped)
# AIDE checks against a custom database (what you approved)
# Use both for defense in depth

sudo apt install -y aide
sudo aideinit
```

### With dpkg --verify

```bash
# dpkg also has a verify command (similar to rpm -V)
sudo dpkg --verify
sudo dpkg --verify nginx

# Output format:
# ??5?????? c /etc/nginx/nginx.conf
# Columns: SM5DLUGTP
# S = Size, M = Mode, 5 = MD5, D = Device, L = Link, U = User, G = Group, T = Time, P = Capabilities
```

### With apt-listchanges

```bash
# Track what changed during upgrades
sudo apt install -y apt-listchanges

# View changelogs before upgrading
sudo apt-get upgrade
```

## Options Reference

| Option | Description |
|--------|-------------|
| `-a`, `--all` | Show all results (OK + FAILED) |
| `-c`, `--changed` | Show only files that failed verification |
| `-e`, `--config` | Include config files in check |
| `-s`, `--silent` | Only report errors |
| `-l`, `--list-missing` | List packages without md5sums |
| `-g`, `--generate=` | Generate md5sums (`missing` or `all`) |
| `--no-config` | Skip config file checks |
| `-p`, `--packages` | Read package list from stdin |
| `--admindir=DIR` | Use alternative dpkg admin directory |
| `--root=DIR` | Use alternative root directory |

## Limitations

- Only checks files tracked by dpkg (not manually added files)
- Uses MD5 (weaker than SHA-256) — sufficient for corruption detection, not for security against sophisticated attacks
- Cannot detect files added to the system that don't belong to any package
- Config files marked as CHANGED is normal after customization
- Some packages don't ship md5sums (use `--generate=missing`)
- Does not verify file permissions, ownership, or timestamps (use `dpkg --verify` for that)

## Quick Reference

| Action | Command |
|--------|---------|
| Install | `sudo apt install debsums` |
| Check all packages | `sudo debsums` |
| Show failures only | `sudo debsums -c` |
| Check one package | `sudo debsums nginx` |
| Changed configs | `sudo debsums -ce` |
| Skip configs | `sudo debsums --no-config` |
| Missing md5sums | `sudo debsums -l` |
| Generate missing | `sudo debsums --generate=missing` |
| Quiet mode | `sudo debsums -s` |
| Full integrity audit | `sudo debsums -c 2>/dev/null` |
