# Linux User Quotas

Disk quotas limit how much storage space and how many files (inodes) users or groups can consume on a filesystem. They prevent a single user from filling up a shared disk and are essential on multi-user servers, mail servers, web hosting, and shared development environments.

## How Quotas Work

Quotas are enforced per-filesystem (not per-directory). Each user/group gets:
- **Block quota** — limits disk space (measured in KB or 1K blocks)
- **Inode quota** — limits number of files/directories

Each limit has two thresholds:
- **Soft limit** — can be exceeded temporarily (during grace period)
- **Hard limit** — absolute maximum, cannot be exceeded

```
Usage:    [-------|=========|XXXXXXX]
                  ^soft     ^hard
                  (warning)  (denied)
```

When a user exceeds the soft limit, a grace period countdown starts. If they don't reduce usage before the grace period expires, the soft limit becomes a hard limit.

## Install Quota Tools

```bash
# Check kernel support for quotas
grep CONFIG_QUOTA /boot/config-$(uname -r)
# Should show: CONFIG_QUOTA=y, CONFIG_QUOTACTL=y

# RHEL / CentOS / Rocky
sudo dnf install quota
sudo yum install quota    # RHEL 7

# Ubuntu / Debian
sudo apt install quota
```

## Enable Quotas on a Filesystem

### Edit /etc/fstab

Add `usrquota` and/or `grpquota` mount options:

```bash
# /etc/fstab — add quota options to a partition
/dev/sda2  /home  ext4  defaults,usrquota,grpquota  0  2

# LVM root filesystem example
/dev/mapper/VolGroup-lv_root  /  ext4  defaults,usrquota,grpquota  1  1
```

For XFS (options are different):

```bash
# XFS uses uquota/gquota (or usrquota/grpquota also works)
/dev/sda2  /home  xfs  defaults,uquota,gquota  0  0
```

### Remount or Reboot

```bash
# Remount without reboot
sudo mount -o remount /home

# Verify quota options are active
mount | grep /home
# Should show: usrquota,grpquota in the options

# Verify specifically
mount | grep usrquota
# Example: /dev/mapper/VolGroup-lv_root on / type ext4 (rw,usrquota,grpquota)
```

### Initialize Quota Database (ext4 only)

```bash
# Create quota database files (aquota.user, aquota.group)
sudo quotacheck -cum /home    # User quotas on /home
sudo quotacheck -cgm /home   # Group quotas on /home
sudo quotacheck -cugm /home  # Both user and group on /home

# Scan ALL filesystems with quota options enabled
sudo quotacheck -avugm

# Flags:
# -a = all filesystems in /etc/fstab with quota options
# -c = create new quota files
# -u = user quotas
# -g = group quotas
# -m = don't remount read-only (safe for mounted filesystems)
# -v = verbose
```

> **XFS note:** XFS does NOT use `quotacheck`. Quotas are built into the filesystem journal and activate automatically with mount options.

### Turn Quotas On

```bash
# Enable quotas (ext4)
sudo quotaon /home
sudo quotaon -aug    # All filesystems, user + group
sudo quotaon -avug   # Verbose

# Check status
sudo quotaon -p /home
sudo quotaon -pa     # All filesystems

# Disable quotas
sudo quotaoff /home
sudo quotaoff -aug
```

> **Auto-start:** On RHEL 6 and older, quotas are started automatically via `/etc/rc.d/rc.sysinit`. On RHEL 7+ with systemd, quotas are enabled automatically when the filesystem is mounted with quota options in `/etc/fstab`.

## Set User Quotas

### Using setquota (Non-Interactive)

```bash
# Syntax: setquota -u USER BLOCK_SOFT BLOCK_HARD INODE_SOFT INODE_HARD FILESYSTEM

# Set 1GB soft, 1.2GB hard limit (in 1K blocks), no inode limits
sudo setquota -u alice 1000000 1200000 0 0 /home

# Set 5GB soft, 6GB hard, 10000 files soft, 12000 files hard
sudo setquota -u bob 5000000 6000000 10000 12000 /home

# Set 500MB soft, 750MB hard (no inode limit)
sudo setquota -u charlie 500000 750000 0 0 /home

# Remove quotas for a user (set all to 0)
sudo setquota -u alice 0 0 0 0 /home
```

### Using edquota (Interactive)

```bash
# Edit user quota in $EDITOR
sudo edquota -u alice

# Example editor view:
# Disk quotas for user alice (uid 1001):
#   Filesystem  blocks   soft   hard   inodes  soft  hard
#   /dev/sda2   52400  1000000 1200000    125     0     0

# Copy quotas from one user to another (template)
sudo edquota -p alice bob charlie dave
# Applies alice's quota limits to bob, charlie, and dave
```

### Set Grace Period

The grace period is how long users can exceed their soft limit before it becomes hard:

```bash
# Edit grace period (applies to all users on the filesystem)
sudo edquota -t

# Example editor view:
# Grace period before enforcing soft limits for users:
# Time units may be: days, hours, minutes, or seconds
#   Filesystem     Block grace period     Inode grace period
#   /dev/sda2      7days                  7days

# Or set non-interactively (ext4)
sudo setquota -t 604800 604800 /home    # 7 days in seconds for both block and inode
```

## Set Group Quotas

```bash
# Set group quota (same syntax, -g instead of -u)
sudo setquota -g developers 10000000 12000000 0 0 /home

# Edit group quota interactively
sudo edquota -g developers

# Copy group quota as template
sudo edquota -g -p developers marketing
```

## View Quota Usage

### Per-User

```bash
# Check your own quota
quota

# Check specific user's quota
sudo quota -u alice
sudo quota -us alice    # Human-readable (short format)

# Verbose output
sudo quota -v -u alice
```

### Filesystem Report

```bash
# Show all users' quota on a filesystem
sudo repquota /home

# Human-readable with grace info
sudo repquota -as /home

# User quotas only
sudo repquota -u /home

# Group quotas only
sudo repquota -g /home

# Show users over quota
sudo repquota -s /home | grep '+' 
```

### Example repquota Output

```
*** Report for user quotas on device /dev/sda2
Block grace time: 7days; Inode grace time: 7days
                        Block limits                File limits
User            used    soft    hard  grace    used  soft  hard  grace
----------------------------------------------------------------------
root      -- 1048576       0       0              5     0     0       
alice     -+ 1100000 1000000 1200000  6days    150     0     0       
bob       --  450000 5000000 6000000            89 10000 12000       
charlie   --   25000  500000  750000            12     0     0       
```

Status flags:
- `--` = under both limits
- `-+` = over block soft limit (grace countdown active)
- `+-` = over inode soft limit
- `++` = over both soft limits

## XFS Quotas

XFS handles quotas differently — they're integrated into the filesystem journal.

### Enable

```bash
# In /etc/fstab:
/dev/sda2  /data  xfs  defaults,uquota,gquota,pquota  0  0
# pquota = project quotas (directory-level)

# Remount
sudo mount -o remount /data
```

### Set and View

```bash
# Set user quota on XFS (uses xfs_quota)
sudo xfs_quota -x -c "limit bsoft=1g bhard=1200m alice" /data

# Set with inode limits
sudo xfs_quota -x -c "limit bsoft=5g bhard=6g isoft=10000 ihard=12000 bob" /data

# View user quota
sudo xfs_quota -x -c "report -u -h" /data

# View group quota
sudo xfs_quota -x -c "report -g -h" /data

# View free space
sudo xfs_quota -x -c "free -h" /data

# Set grace period (7 days)
sudo xfs_quota -x -c "timer -u 7d" /data

# Remove quota for a user
sudo xfs_quota -x -c "limit bsoft=0 bhard=0 alice" /data
```

### XFS Project Quotas (Directory-Level)

XFS supports limiting entire directory trees (projects):

```bash
# 1. Define project in /etc/projects
echo "1:/data/project_a" >> /etc/projects

# 2. Map project name in /etc/projid
echo "project_a:1" >> /etc/projid

# 3. Initialize the project
sudo xfs_quota -x -c "project -s project_a" /data

# 4. Set project quota
sudo xfs_quota -x -c "limit -p bsoft=10g bhard=12g project_a" /data

# 5. Report project quotas
sudo xfs_quota -x -c "report -p -h" /data
```

## Quota Notifications

### Mail Warnings with warnquota

```bash
# Install (usually included with quota package)
# Configure /etc/warnquota.conf
sudo vi /etc/warnquota.conf

# Key settings:
MAIL_CMD = "/usr/sbin/sendmail -t"
FROM = "quota-admin@example.com"
SUBJECT = "Disk quota warning"
CC_TO = "admin@example.com"
MESSAGE = "Your disk usage on %d has exceeded the quota limit.|Please reduce your usage."
SIGNATURE = "System Administrator"

# Run manually
sudo warnquota

# Add to cron (check daily)
echo "0 6 * * * root /usr/sbin/warnquota" | sudo tee /etc/cron.d/warnquota
```

## Common Scenarios

### Web Hosting (Per-User)

```bash
# Each user gets 2GB space, 3000 files
for user in $(awk -F: '$3 >= 1000 && $7 !~ /nologin/ {print $1}' /etc/passwd); do
    sudo setquota -u "$user" 2000000 2500000 3000 4000 /home
done
```

### Development Team (Group-Based)

```bash
# Create team group
sudo groupadd -g 2000 devteam

# Set group quota (50GB shared)
sudo setquota -g devteam 50000000 55000000 0 0 /data

# Add users to group
sudo usermod -aG devteam alice
sudo usermod -aG devteam bob
```

### Mail Server

```bash
# Conservative quotas for mailboxes
sudo setquota -u postfix 0 0 0 0 /var/mail    # No quota for service
sudo setquota -u alice 500000 600000 0 0 /var/mail   # 500MB soft / 600MB hard
```

## Troubleshooting

### "quotaon: Cannot find filesystem to check"

```bash
# Quota options not in /etc/fstab
sudo vi /etc/fstab    # Add usrquota,grpquota
sudo mount -o remount /home
```

### "quotaon: using /home/aquota.user: No such file or directory"

```bash
# Quota database not created
sudo quotacheck -cugm /home
sudo quotaon /home
```

### User Can Still Write Beyond Hard Limit

```bash
# Check if quotas are actually on
sudo quotaon -p /home

# Check if correct filesystem
df /home    # Verify the device matches your fstab entry

# Quotas on wrong filesystem — user's files may be on / not /home
```

### Quota Not Working After Reboot

```bash
# Ensure quotaon runs at boot
# On systemd systems, quota is usually auto-enabled via mount options
# If not, enable the quota service:
sudo systemctl enable quota_nld    # RHEL
sudo systemctl enable quotaon      # Some systems

# Or add to rc.local
echo "quotaon -aug" >> /etc/rc.local
```

### XFS: "XFS Quotas not turned on"

```bash
# XFS quotas MUST be specified at mount time
# Cannot be added after mount — must unmount and remount (or reboot)
sudo umount /data
sudo mount -o uquota,gquota /dev/sda2 /data
# Or add to fstab and reboot
```

## One-Liners

```bash
# Set same quota for all regular users
for u in $(awk -F: '$3 >= 1000 && $7 !~ /nologin/ {print $1}' /etc/passwd); do
    sudo setquota -u "$u" 1000000 1200000 0 0 /home
done

# Show users over their soft limit
sudo repquota -s /home | grep '+'

# Show top disk users vs their quota
sudo repquota -s /home | sort -k3 -rn | head -10

# Quick quota report (human-readable)
sudo repquota -as /home

# Check if quotas are enabled on all filesystems
sudo quotaon -pa

# Display only users over quota
quota -q

# Query kernel quota statistics
quotastats

# Check filesystem block size (useful for calculating quota values)
sudo dumpe2fs /dev/sda1 2>/dev/null | grep -i 'Block size'

# Test quota enforcement with dd (creates ~400MB file)
dd if=/dev/zero of=~/quota_test bs=4k count=100000
rm ~/quota_test

# Copy one user's quota to all others in a group
GROUP_MEMBERS=$(getent group devteam | cut -d: -f4 | tr ',' ' ')
for u in $GROUP_MEMBERS; do
    sudo edquota -p template_user "$u"
done
```

## ext4 vs XFS Quota Differences

| Feature | ext4 | XFS |
|---------|------|-----|
| Enable method | fstab + quotacheck + quotaon | fstab mount options only |
| Database files | `aquota.user`, `aquota.group` | Built into journal (no files) |
| `quotacheck` needed | Yes | No |
| Project (directory) quotas | No | Yes (`pquota`) |
| Tool | `setquota`, `edquota`, `repquota` | `xfs_quota` |
| Online resize | Quotas preserved | Quotas preserved |
| Enable after mount | Yes (`quotaon`) | No (must remount) |

## Important Files

| File | Purpose |
|------|---------|
| `/etc/fstab` | Mount options (usrquota, grpquota) |
| `/home/aquota.user` | ext4 user quota database |
| `/home/aquota.group` | ext4 group quota database |
| `/etc/warnquota.conf` | Quota warning email configuration |
| `/etc/quotatab` | Filesystem descriptions for reports |
| `/etc/projects` | XFS project definitions |
| `/etc/projid` | XFS project ID mappings |

## Quick Reference

```bash
# Setup (ext4)
# 1. Add usrquota,grpquota to /etc/fstab
# 2. mount -o remount /home
# 3. quotacheck -cugm /home
# 4. quotaon /home

# Setup (XFS)
# 1. Add uquota,gquota to /etc/fstab
# 2. Reboot or remount

# Set quota
setquota -u USER BSOFT BHARD ISOFT IHARD /filesystem
edquota -u USER                    # Interactive
edquota -t                         # Set grace period
edquota -p TEMPLATE OTHER_USER     # Copy quota

# View quota
quota -u USER                      # Single user
repquota -as /home                 # All users, human-readable
xfs_quota -x -c "report -u -h" /  # XFS

# Manage
quotaon /home                      # Enable
quotaoff /home                     # Disable
quotacheck -cugm /home             # Rebuild database (ext4)
warnquota                          # Send warning emails
```
