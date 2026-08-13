# iSCSI Cheatsheet

iSCSI (Internet Small Computer Systems Interface) enables block-level storage access over TCP/IP networks. It maps SCSI commands to TCP packets, allowing remote storage to appear as local disks — without dedicated SAN hardware like Fibre Channel.

**Terminology:**
- **Initiator** — the client that connects to remote storage (your server)
- **Target** — the storage server that exports LUNs
- **IQN** — iSCSI Qualified Name, a unique identifier for initiators and targets
- **Portal** — an IP:port combination where a target listens (default port: 3260)
- **LUN** — Logical Unit Number, a block device exported by the target
- **CHAP** — Challenge-Handshake Authentication Protocol, used for initiator/target authentication

## Install

```bash
# RHEL / CentOS / Rocky / Alma
sudo yum install iscsi-initiator-utils       # RHEL 7
sudo dnf install iscsi-initiator-utils       # RHEL 8+

# Ubuntu / Debian
sudo apt install open-iscsi

# Enable and start
sudo systemctl enable iscsid
sudo systemctl start iscsid
sudo systemctl status iscsid
```

## Important Files

| File | Purpose |
|------|---------|
| `/etc/iscsi/initiatorname.iscsi` | Initiator IQN (unique per host) |
| `/etc/iscsi/iscsid.conf` | Main iSCSI configuration (timeouts, CHAP, startup) |
| `/var/lib/iscsi/nodes/` | Discovered targets and their settings |
| `/var/lib/iscsi/send_targets/` | Discovery records |
| `/var/lib/iscsi/ifaces/` | Interface bindings |

## Set the Initiator Name

Each host must have a unique IQN. Set it before connecting to any targets:

```bash
# View current IQN
cat /etc/iscsi/initiatorname.iscsi

# Set a custom IQN
echo "InitiatorName=iqn.2024-01.com.example:server01" | sudo tee /etc/iscsi/initiatorname.iscsi

# Restart iscsid after changing
sudo systemctl restart iscsid
```

> **Important:** The IQN must be unique across all hosts in your environment. Storage arrays use the IQN for LUN masking.

## Discovery

```bash
# Discover targets on a portal
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.100

# Discover on a non-default port
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.100:3261

# List discovered targets
sudo iscsiadm -m node

# Show discovery records
sudo iscsiadm -m discovery

# Delete a discovery record
sudo iscsiadm -m discovery -p 192.168.1.100 -o delete

# Rediscover (refresh target list)
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.100 -o update
```

## Login (Connect to Target)

```bash
# Log in to a specific target
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 --login

# Log in to ALL discovered targets
sudo iscsiadm -m node --login

# Log in to all targets on a specific portal
sudo iscsiadm -m node -p 192.168.1.100 --login

# Verify session
sudo iscsiadm -m session

# Show session details
sudo iscsiadm -m session -P 3
```

## Logout (Disconnect)

```bash
# Log out of a specific target
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 --logout

# Log out of ALL targets
sudo iscsiadm -m node --logout

# Log out of all targets on a specific portal
sudo iscsiadm -m node -p 192.168.1.100 --logout
```

> **Warning:** Unmount filesystems and deactivate LVM before logging out, or you'll risk data loss.

## Session Management

```bash
# List active sessions
sudo iscsiadm -m session

# Show sessions with details (verbosity levels 0-3)
sudo iscsiadm -m session -P 0    # target names only
sudo iscsiadm -m session -P 1    # target + portal info
sudo iscsiadm -m session -P 2    # + negotiated parameters
sudo iscsiadm -m session -P 3    # + attached SCSI devices

# Show stats for a session
sudo iscsiadm -m session -s

# Rescan a session (detect new LUNs)
sudo iscsiadm -m session --rescan

# Rescan a specific session
sudo iscsiadm -m session -r <SID> --rescan
```

## Automatic Login at Boot

```bash
# Enable automatic login for a specific target
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update -n node.startup -v automatic

# Disable automatic login
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update -n node.startup -v manual

# Enable automatic login for ALL targets
sudo iscsiadm -m node -o update -n node.startup -v automatic

# Verify startup setting
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 | grep node.startup
```

## CHAP Authentication

### One-way CHAP (Target Authenticates Initiator)

```bash
# Set CHAP username and password for a target
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.auth.authmethod -v CHAP
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.auth.username -v myinitiator
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.auth.password -v mysecretpassword
```

### Mutual CHAP (Both Sides Authenticate)

```bash
# Set initiator credentials (target authenticates initiator)
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.auth.authmethod -v CHAP
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.auth.username -v myinitiator
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.auth.password -v initiatorpassword

# Set target credentials (initiator authenticates target)
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.auth.username_in -v mytarget
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.auth.password_in -v targetpassword
```

### Discovery CHAP

```bash
# Set CHAP for discovery (in /etc/iscsi/iscsid.conf)
# discovery.sendtargets.auth.authmethod = CHAP
# discovery.sendtargets.auth.username = myuser
# discovery.sendtargets.auth.password = mypassword

# Or set per-portal
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.100 \
    --op=update --name=discovery.sendtargets.auth.authmethod --value=CHAP
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.100 \
    --op=update --name=discovery.sendtargets.auth.username --value=myuser
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.100 \
    --op=update --name=discovery.sendtargets.auth.password --value=mypassword
```

## Configuration Tuning

### Timeout Settings

Edit `/etc/iscsi/iscsid.conf` or set per-node:

```bash
# Set replacement timeout (seconds to wait before failing I/O on path loss)
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.timeo.replacement_timeout -v 120

# Login timeout
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.conn[0].timeo.login_timeout -v 15

# Noop (keepalive) interval and timeout
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.conn[0].timeo.noop_out_interval -v 5
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.conn[0].timeo.noop_out_timeout -v 5
```

### Key iscsid.conf Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `node.startup` | `manual` | `automatic` to login at boot |
| `node.session.timeo.replacement_timeout` | `120` | Seconds before failing I/O on lost session |
| `node.conn[0].timeo.noop_out_interval` | `5` | Keepalive ping interval (seconds) |
| `node.conn[0].timeo.noop_out_timeout` | `5` | Seconds to wait for keepalive response |
| `node.session.nr_sessions` | `1` | Number of sessions per target (multipath) |
| `node.session.queue_depth` | `32` | Max commands queued per session |
| `node.conn[0].iscsi.MaxRecvDataSegmentLength` | `262144` | Max receive data segment (bytes) |
| `node.conn[0].iscsi.FirstBurstLength` | `262144` | Max unsolicited data (bytes) |
| `node.conn[0].iscsi.MaxBurstLength` | `16776192` | Max data per sequence (bytes) |

## Interface Binding

Bind iSCSI traffic to a specific network interface:

```bash
# Create an interface configuration
sudo iscsiadm -m iface -I iscsi0 --op=new

# Bind to a specific NIC
sudo iscsiadm -m iface -I iscsi0 --op=update -n iface.net_ifacename -v eth1

# Or bind by IP address
sudo iscsiadm -m iface -I iscsi0 --op=update -n iface.ipaddress -v 10.0.0.5

# List configured interfaces
sudo iscsiadm -m iface

# Show interface details
sudo iscsiadm -m iface -I iscsi0

# Discover using specific interface
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.100 -I iscsi0

# Log in using specific interface
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -I iscsi0 --login

# Delete an interface
sudo iscsiadm -m iface -I iscsi0 --op=delete
```

## iSCSI with Multipath

For redundancy and load balancing, use multiple paths to the same target:

```bash
# Step 1: Set multiple sessions per target (alternative to multiple portals)
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.nr_sessions -v 2

# Step 2: Or discover from multiple portals (preferred)
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.100
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.101

# Step 3: Login to all paths
sudo iscsiadm -m node --login

# Step 4: Configure multipath
# Install multipath tools
sudo dnf install device-mapper-multipath    # RHEL
sudo apt install multipath-tools            # Ubuntu

# Enable multipath
sudo mpathconf --enable --user_friendly_names y
sudo systemctl enable multipathd
sudo systemctl start multipathd

# Step 5: Verify multipath sees both paths
sudo multipath -ll
```

### Example multipath.conf for iSCSI

```
defaults {
    user_friendly_names yes
    find_multipaths yes
    path_grouping_policy failover
    no_path_retry 12
}

devices {
    device {
        vendor                "IET"
        product               "VIRTUAL-DISK"
        path_selector         "round-robin 0"
        path_grouping_policy  multibus
        path_checker          tur
        failback              immediate
    }
}
```

## iSCSI Target Setup (Linux as Storage Server)

### Using targetcli (RHEL 7+ / Ubuntu)

```bash
# Install
sudo dnf install targetcli             # RHEL
sudo apt install targetcli-fb          # Ubuntu

# Start targetcli interactive shell
sudo targetcli

# Create a backing store (block device or file)
/> /backstores/block create disk01 /dev/sdb
/> /backstores/fileio create disk02 /var/iscsi/disk02.img 10G

# Create an iSCSI target
/> /iscsi create iqn.2024-01.com.storage:target01

# Create a LUN
/> /iscsi/iqn.2024-01.com.storage:target01/tpg1/luns create /backstores/block/disk01

# Set the portal (listening address)
/> /iscsi/iqn.2024-01.com.storage:target01/tpg1/portals create 192.168.1.100

# Create an ACL (allow specific initiator)
/> /iscsi/iqn.2024-01.com.storage:target01/tpg1/acls create iqn.2024-01.com.example:server01

# Set CHAP on the ACL
/> /iscsi/iqn.2024-01.com.storage:target01/tpg1/acls/iqn.2024-01.com.example:server01 set auth userid=myuser
/> /iscsi/iqn.2024-01.com.storage:target01/tpg1/acls/iqn.2024-01.com.example:server01 set auth password=mypassword

# Disable ACL enforcement (allow any initiator — not recommended for production)
/> /iscsi/iqn.2024-01.com.storage:target01/tpg1 set attribute authentication=0 demo_mode_write_protect=0 generate_node_acls=1

# Save and exit
/> saveconfig
/> exit

# Enable target service at boot
sudo systemctl enable target       # RHEL
sudo systemctl enable rtslib-fb-targetctl   # Ubuntu (may vary)
```

### Using tgtd (Legacy — RHEL 6 / Older Ubuntu)

```bash
# Install
sudo yum install scsi-target-utils    # RHEL 6
sudo apt install tgt                  # Ubuntu

# Create a target in /etc/tgt/conf.d/iscsi.conf
cat <<'EOF' | sudo tee /etc/tgt/conf.d/iscsi.conf
<target iqn.2024-01.com.storage:target01>
    backing-store /dev/sdb
    initiator-address 192.168.1.0/24
    incominguser myuser mypassword
</target>
EOF

# Restart tgt
sudo systemctl restart tgtd
sudo systemctl enable tgtd

# Verify
sudo tgtadm --mode target --op show
```

## Firewall Rules

```bash
# RHEL (firewalld)
sudo firewall-cmd --permanent --add-port=3260/tcp
sudo firewall-cmd --reload

# Ubuntu (ufw)
sudo ufw allow 3260/tcp

# iptables (manual)
sudo iptables -A INPUT -p tcp --dport 3260 -j ACCEPT
```

## Recipes

### Recipe: Full Discovery-to-Mount Workflow

```bash
#!/bin/bash
# Discover, login, format, and mount an iSCSI LUN

TARGET_PORTAL="192.168.1.100"
TARGET_IQN="iqn.2024-01.com.storage:lun01"
MOUNT_POINT="/data"

# 1. Discover targets
sudo iscsiadm -m discovery -t sendtargets -p "$TARGET_PORTAL"

# 2. Login
sudo iscsiadm -m node -T "$TARGET_IQN" -p "$TARGET_PORTAL" --login

# 3. Set automatic login at boot
sudo iscsiadm -m node -T "$TARGET_IQN" -p "$TARGET_PORTAL" -o update \
    -n node.startup -v automatic

# 4. Wait for device to appear
sleep 2

# 5. Find the new disk (last SCSI disk added)
NEW_DISK=$(lsblk -d -n -o NAME,TRAN | grep iscsi | tail -1 | awk '{print $1}')
echo "New iSCSI disk: /dev/$NEW_DISK"

# 6. Create filesystem
sudo mkfs.xfs "/dev/$NEW_DISK"

# 7. Mount
sudo mkdir -p "$MOUNT_POINT"
sudo mount "/dev/$NEW_DISK" "$MOUNT_POINT"

# 8. Add to fstab (use UUID, _netdev for network storage)
UUID=$(sudo blkid -s UUID -o value "/dev/$NEW_DISK")
echo "UUID=$UUID $MOUNT_POINT xfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
```

### Recipe: Discover and Login to All Targets on a Portal

```bash
# One-liner: discover and immediately login to all targets
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.100 && sudo iscsiadm -m node -p 192.168.1.100 --login
```

### Recipe: Find Which iSCSI Target a Disk Belongs To

```bash
# Map /dev/sdX to its iSCSI target
lsblk -S | grep iscsi

# Or from session details
sudo iscsiadm -m session -P 3 | grep -A20 "Target:"

# Find the target for a specific device
for session_dir in /sys/class/iscsi_session/session*/; do
    target=$(cat "$session_dir/targetname" 2>/dev/null)
    for dev in "$session_dir"/device/target*/*/block/*; do
        [ -d "$dev" ] && echo "$(basename $dev) -> $target"
    done
done
```

### Recipe: Remove an iSCSI Disk Cleanly

```bash
#!/bin/bash
# Cleanly remove an iSCSI LUN

TARGET_IQN="iqn.2024-01.com.storage:lun01"
TARGET_PORTAL="192.168.1.100"
MOUNT_POINT="/data"

# 1. Unmount
sudo umount "$MOUNT_POINT"

# 2. Remove fstab entry
sudo sed -i "\|$MOUNT_POINT|d" /etc/fstab

# 3. Logout
sudo iscsiadm -m node -T "$TARGET_IQN" -p "$TARGET_PORTAL" --logout

# 4. Delete the node record
sudo iscsiadm -m node -T "$TARGET_IQN" -p "$TARGET_PORTAL" -o delete

# 5. Verify
sudo iscsiadm -m session
```

### Recipe: Resize an iSCSI LUN (After Storage-Side Expansion)

```bash
# 1. Rescan the session to pick up the new size
sudo iscsiadm -m session --rescan

# 2. Verify new size
lsblk

# 3. If using LVM:
sudo pvresize /dev/sdX
sudo lvextend -l +100%FREE /dev/mapper/vg-lv
sudo xfs_growfs /mountpoint          # XFS
# or
sudo resize2fs /dev/mapper/vg-lv    # ext4

# 4. If raw filesystem (no LVM):
sudo xfs_growfs /mountpoint          # XFS (mounted)
sudo resize2fs /dev/sdX              # ext4 (can be mounted)
```

### Recipe: iSCSI Boot (Netboot from iSCSI)

```bash
# RHEL: set iSCSI root in dracut
echo 'add_dracutmodules+=" iscsi "' | sudo tee /etc/dracut.conf.d/iscsi.conf

# Set kernel parameters in GRUB
# netroot=iscsi:192.168.1.100::3260::iqn.2024-01.com.storage:boot
# rd.iscsi.initiator=iqn.2024-01.com.example:server01

# Rebuild initramfs
sudo dracut --force --add iscsi

# Ubuntu: install and configure for iscsi root
sudo apt install open-iscsi
# Configure /etc/iscsi/iscsi.initramfs
echo "ISCSI_AUTO=true" | sudo tee /etc/iscsi/iscsi.initramfs
sudo update-initramfs -u
```

## One-Liners

```bash
# List all active iSCSI sessions
sudo iscsiadm -m session

# List iSCSI disks only
lsblk -S -o NAME,TRAN,SIZE,VENDOR,MODEL | grep iscsi

# Show all connected targets and their devices
sudo iscsiadm -m session -P 3 | grep -E "Target|disk|attached"

# Count active iSCSI sessions
sudo iscsiadm -m session | wc -l

# Get the IQN of this host
cat /etc/iscsi/initiatorname.iscsi | grep -v "^#"

# Check if iscsid is running
systemctl is-active iscsid

# Quick check: discovery + login in one go
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.100 --login

# Show iSCSI device UUIDs
for dev in $(lsblk -S -n -o NAME,TRAN | awk '/iscsi/{print $1}'); do
    echo "/dev/$dev: $(sudo blkid -s UUID -o value /dev/$dev)"
done

# Find all iSCSI devices and their sizes
sudo iscsiadm -m session -P 3 | grep -E "scsi disk|Attached|Target" | paste - - -

# Rescan all sessions at once
sudo iscsiadm -m session --rescan

# Show iSCSI connection speed/stats
sudo iscsiadm -m session -s

# Logout from everything (emergency — unmount first!)
sudo iscsiadm -m node --logoutall=all

# Delete ALL discovered nodes (clean slate)
sudo iscsiadm -m node -o delete

# Check replacement timeout for all targets
sudo iscsiadm -m session -P 2 | grep "Replacement Timeout"

# Show negotiated iSCSI parameters
sudo iscsiadm -m session -P 2 | grep -E "MaxRecv|MaxBurst|FirstBurst|MaxXmit"
```

## Troubleshooting

### Cannot Discover Targets

```bash
# Check network connectivity
ping 192.168.1.100
nc -zv 192.168.1.100 3260

# Check firewall on target
sudo firewall-cmd --list-ports    # RHEL
sudo ufw status                   # Ubuntu

# Run discovery with debug
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.100 -d8

# Check if iscsid is running
sudo systemctl status iscsid
```

### Login Fails

```bash
# Debug login
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 --login -d8

# Common causes:
# - CHAP credentials mismatch
# - Target ACL doesn't include your IQN
# - Wrong IQN in /etc/iscsi/initiatorname.iscsi
# - Firewall blocking port 3260

# Verify your IQN
cat /etc/iscsi/initiatorname.iscsi

# Check CHAP settings
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 | grep auth
```

### Session Drops / Timeouts

```bash
# Check session status
sudo iscsiadm -m session -P 2

# Review system logs
journalctl -u iscsid --since "10 minutes ago"
dmesg | grep -i iscsi

# Check network for packet loss
ping -c 100 192.168.1.100 | tail -3

# Increase replacement timeout (prevents premature failover)
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.timeo.replacement_timeout -v 300

# Adjust noop timers (keepalive)
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.conn[0].timeo.noop_out_interval -v 10
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.conn[0].timeo.noop_out_timeout -v 15
```

### I/O Errors on iSCSI Disk

```bash
# Check session state
sudo iscsiadm -m session -P 1

# Check kernel messages
dmesg | grep -i "iscsi\|scsi\|sd[a-z]"

# Check if target is still reachable
sudo iscsiadm -m session --rescan

# Force session re-login
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 --logout
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 --login
```

### Devices Not Appearing After Login

```bash
# Rescan the SCSI bus
for host in /sys/class/scsi_host/host*/scan; do
    echo "- - -" | sudo tee "$host"
done

# Or rescan the specific session
sudo iscsiadm -m session --rescan

# Check SCSI devices
lsscsi
cat /proc/scsi/scsi

# Verify session has LUNs attached
sudo iscsiadm -m session -P 3
```

### "Could not log into all portals" Error

```bash
# Usually means already logged in — check existing sessions
sudo iscsiadm -m session

# If stale, logout first then retry
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 --logout
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 --login

# Or delete and rediscover
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o delete
sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.100
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 --login
```

## Performance Tuning

```bash
# Increase queue depth (default 32)
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.queue_depth -v 128

# Increase max receive data segment (default 262144)
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.conn[0].iscsi.MaxRecvDataSegmentLength -v 262144

# Enable immediate data (default yes)
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.iscsi.ImmediateData -v Yes

# Enable initial R2T (default yes — set No for sequential writes)
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.iscsi.InitialR2T -v No

# Increase first burst length
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.session.iscsi.FirstBurstLength -v 262144

# Increase max burst length
sudo iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update \
    -n node.conn[0].iscsi.MaxBurstLength -v 16776192

# Use jumbo frames on the network (MTU 9000)
sudo ip link set eth1 mtu 9000
# Make persistent in /etc/sysconfig/network-scripts/ifcfg-eth1 (RHEL)
# or /etc/netplan/*.yaml (Ubuntu)
```

### Verify Negotiated Parameters

```bash
sudo iscsiadm -m session -P 2 | grep -E "MaxRecv|MaxBurst|FirstBurst|ImmediateData|InitialR2T|MaxXmit|HeaderDigest|DataDigest"
```

## iscsiadm Modes Reference

| Mode | Description |
|------|-------------|
| `discovery` | Discover targets on a portal |
| `node` | Manage target records (login, logout, update, delete) |
| `session` | View and manage active sessions |
| `iface` | Manage network interface bindings |
| `host` | Show iSCSI host information |
| `fw` | Show firmware iSCSI boot targets |

## iscsiadm Operations

| Operation | Flag | Description |
|-----------|------|-------------|
| `--login` / `-l` | | Log in to target(s) |
| `--logout` / `-u` | | Log out of target(s) |
| `--rescan` | | Rescan session for new LUNs |
| `-o new` | `--op=new` | Create new record |
| `-o delete` | `--op=delete` | Delete a record |
| `-o update` | `--op=update` | Update a record |
| `-o show` | `--op=show` | Show a record |
| `-n` | `--name` | Parameter name to update |
| `-v` | `--value` | Parameter value |
| `-P` | `--print` | Verbosity level (0-3) |
| `-d` | `--debug` | Debug level (0-8) |

## RHEL vs Ubuntu Differences

| Feature | RHEL / CentOS | Ubuntu / Debian |
|---------|--------------|-----------------|
| Package | `iscsi-initiator-utils` | `open-iscsi` |
| Service | `iscsid` | `iscsid` (+ `open-iscsi` for auto-login) |
| Target package | `targetcli` | `targetcli-fb` |
| Target service | `target` | `rtslib-fb-targetctl` |
| Config | `/etc/iscsi/iscsid.conf` | `/etc/iscsi/iscsid.conf` |
| IQN | `/etc/iscsi/initiatorname.iscsi` | `/etc/iscsi/initiatorname.iscsi` |
| Auto-login service | `iscsi.service` | `open-iscsi.service` |
| initramfs rebuild | `dracut --force` | `update-initramfs -u` |

### Ubuntu-Specific: Enable Auto-Login Service

```bash
# Ubuntu uses open-iscsi.service for automatic logins at boot
sudo systemctl enable open-iscsi
sudo systemctl start open-iscsi

# This triggers login for all nodes with node.startup = automatic
```

### RHEL-Specific: Enable Auto-Login Service

```bash
# RHEL uses iscsi.service for automatic logins at boot
sudo systemctl enable iscsi
sudo systemctl start iscsi
```

## Best Practices

1. **Unique IQN per host** — storage arrays use the IQN for LUN masking; duplicate IQNs cause data corruption
2. **Use _netdev in fstab** — ensures network is up before mounting iSCSI filesystems
3. **Dedicated VLAN/subnet** — isolate iSCSI traffic from general network traffic
4. **Jumbo frames** — use MTU 9000 on all switches and NICs in the iSCSI path
5. **Use multipath** — multiple paths protect against NIC/switch/cable failure
6. **CHAP authentication** — prevents unauthorized initiators from accessing LUNs
7. **Adjust replacement_timeout** — set higher in production (300s+) to ride out brief outages
8. **Monitor sessions** — alert on session drops (`iscsiadm -m session` returns empty)
9. **Unmount before logout** — always unmount and deactivate LVM before disconnecting
10. **Backup node records** — copy `/var/lib/iscsi/nodes/` before making changes

## Quick Reference

```bash
# Discovery
iscsiadm -m discovery -t sendtargets -p <IP>

# Login
iscsiadm -m node -T <IQN> -p <IP> --login

# Logout
iscsiadm -m node -T <IQN> -p <IP> --logout

# List sessions
iscsiadm -m session

# Session details + devices
iscsiadm -m session -P 3

# Set automatic login
iscsiadm -m node -T <IQN> -p <IP> -o update -n node.startup -v automatic

# Rescan for new LUNs
iscsiadm -m session --rescan

# Delete a node record
iscsiadm -m node -T <IQN> -p <IP> -o delete

# Logout all
iscsiadm -m node --logoutall=all
```
