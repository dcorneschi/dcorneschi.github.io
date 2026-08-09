# SAN Storage Commands

SCSI scanning, HBA management, Fibre Channel diagnostics, and storage troubleshooting on Linux.

## Scan for New LUNs

### Classic SCSI Bus Rescan

```bash
# Scan all HBA hosts
for i in $(ls /sys/class/scsi_host/); do
    echo "- - -" > /sys/class/scsi_host/$i/scan
done

# With progress output
for i in $(ls /sys/class/scsi_host/); do
    echo "Scanning $i...Completed"
    echo "- - -" > /sys/class/scsi_host/$i/scan
done

# Scan specific host
echo "- - -" > /sys/class/scsi_host/host0/scan
```

### Using rescan-scsi-bus.sh

```bash
# Install sg3_utils
yum install sg3_utils    # RHEL
apt install sg3-utils    # Ubuntu

# Rescan all SCSI buses
rescan-scsi-bus.sh

# Rescan with verbose output
rescan-scsi-bus.sh -v

# Rescan and add new devices only (no remove)
rescan-scsi-bus.sh --add
```

### Rescan Specific Device (After Resize)

```bash
# Rescan a single disk (after VMware/SAN expansion)
echo 1 > /sys/block/sda/device/rescan
echo 1 > /sys/block/sdb/device/rescan
```

### Legacy Scan (RHEL 3 — /proc Method)

```bash
cat /proc/scsi/scsi
echo "scsi add-single-device 0 0 3 0" > /proc/scsi/scsi
cat /proc/scsi/scsi
```

## HBA Information

### List HBAs

```bash
# Find Fibre Channel HBAs
lspci | grep -i fibre

# Find QLogic HBAs
lspci | grep -i qlogic

# Find Emulex HBAs
lspci | grep -i emulex

# Detailed HBA info (replace with your PCI address)
lspci -v -s 03:00.1
```

### HBA Details from sysfs

```bash
# Model description
cat /sys/class/scsi_host/host0/model_desc

# Firmware version
cat /sys/class/scsi_host/host0/fwrev

# Option ROM version
cat /sys/class/scsi_host/host0/option_rom_version

# PCI slot and scsi_host mapping
ls -l /sys/class/scsi_host
```

### Queue Depth

```bash
# Per-device queue depth
cat /sys/block/sdb/device/queue_depth

# QLogic default max queue depth
cat /sys/module/qla2xxx/parameters/ql2xmaxqdepth
# 32

# Emulex default LUN queue depth
cat /sys/module/lpfc/parameters/lpfc_lun_queue_depth
# 30

# Number of requests in queue
cat /sys/block/sdb/queue/nr_requests
```

## Fibre Channel

### Port Status

```bash
# Port state (Online / Linkdown)
for i in /sys/class/fc_host/host*/port_state; do
    echo "$i: $(cat $i)"
done

# Port WWN (World Wide Name)
for i in /sys/class/fc_host/host*/port_name; do
    echo "$i: $(cat $i)"
done

# Port speed
for i in /sys/class/fc_host/host*/speed; do
    echo "$i: $(cat $i)"
done
```

### FC Statistics and Error Checking

```bash
# Check for CRC errors
cat /sys/class/fc_host/host2/statistics/invalid_crc_count

# TX word errors
cat /sys/class/fc_host/host2/statistics/invalid_tx_word_count

# Frame errors
cat /sys/class/fc_host/host2/statistics/error_frames

# Loss of signal
cat /sys/class/fc_host/host2/statistics/loss_of_signal_count

# Check all FC hosts for errors
for host in /sys/class/fc_host/host*/statistics; do
    echo "=== $(basename $(dirname $host)) ==="
    for stat in invalid_crc_count invalid_tx_word_count error_frames loss_of_signal_count; do
        echo "  $stat: $(cat $host/$stat 2>/dev/null)"
    done
done
```

### Identify HBA Errors in Logs

```bash
# List HBAs by PCI address
lspci | grep -i qlogic

# Count Abort errors per HBA
grep -w '04:00.0' /var/log/messages* | grep Abort | wc -l
grep -w '04:00.1' /var/log/messages* | grep Abort | wc -l

# Identify host device for FC host
ls -l /sys/class/fc_host
```

## Disk Mapping and Path Information

```bash
# Disk-to-path mapping (by-id)
ls -l /dev/disk/by-id/

# Disk-to-path mapping (by-path — shows HBA/target/LUN)
ls -l /dev/disk/by-path/

# Find the physical port a LUN is connected to
udevadm info --query=path --name /dev/sdf

# Find which HBA presents each LUN
ls -l /sys/class/scsi_disk/

# Find HBA port for a specific device
udevadm info --query=path --name /dev/sdd
```

## Block Device Information

```bash
# Get disk size in sectors (512 bytes)
blockdev --getsize /dev/mapper/mpathd

# Get disk size in bytes
blockdev --getsize64 /dev/mapper/mpathd

# Flush disk buffers
blockdev --flushbufs /dev/mapper/mpathd

# Read-only filesystems
grep "[[:space:]]ro[[:space:],]" /proc/mounts
```

## I/O Scheduler

```bash
# Display current I/O scheduler for a disk
cat /sys/block/sda/queue/scheduler

# Change I/O scheduler (online, not persistent)
echo deadline > /sys/block/sda/queue/scheduler
echo noop > /sys/block/sdb/queue/scheduler
echo cfq > /sys/block/sdc/queue/scheduler

# Change I/O scheduler persistently (all kernels)
grubby --update-kernel=ALL --args="elevator=cfq"

# For RHEL 8+ (mq schedulers)
echo mq-deadline > /sys/block/sda/queue/scheduler
echo none > /sys/block/nvme0n1/queue/scheduler
```

## Change Storage Device Load Order

Control the order in which storage drivers load (useful when SAN boots before local disk):

```bash
vi /etc/default/grub
```

```
GRUB_CMDLINE_LINUX="rd.lvm.lv=rootvg/rootlv rd.lvm.lv=rootvg/swaplv rd.driver.pre=megaraid_sas,lpfc"
```

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

This ensures `megaraid_sas` (local RAID controller) loads before `lpfc` (Emulex FC HBA), so the local disk is always `/dev/sda`.

## SAR Storage Monitoring

```bash
# View load average from sar
sudo sar -q -f /var/log/sa/sa07 | less
# If load average > runq-sz → processes in "D" state (uninterruptible I/O wait)

# View disk I/O stats from sar
sudo sar -d -f /var/log/sa/sa07 | less
# Low avgqu-sz with high await → storage-side latency issue

# Check TPS only for physical device
sudo sar -d -f /var/log/sa/sa07 | grep sdb

# Collect sar data for support
tar -czvf /tmp/sar-$(hostname).tgz /var/log/sa/*
```

## EMC Tools

### emcgrab (Diagnostic Collection)

```bash
cd /opt
wget ftp://ftp.emc.com/pub/emcgrab/Unix/emcgrab_Linux_v4.7.10.tar
tar -xvf emcgrab_Linux_v4.7.10.tar
cd emcgrab
./emcgrab.sh
ls -l outputs/
```

### inq (EMC Inquiry Tool)

Download from `ftp.emc.com/pub/symm3000/inquiry`

```bash
# Display EMC LUN information
./inq -no_dots

# Display with WWN
./inq -wwn
```

### SANsurfer FC HBA CLI (scli)

```bash
yum install scli-1.7.2-7.i386.rpm glibc.i686
scli
```

## Troubleshooting

### Identify "Abort" Issues

Look for PowerPath or FC errors:

```bash
# Check FC statistics for each host
for host in /sys/class/fc_host/host*/statistics; do
    crc=$(cat $host/invalid_crc_count 2>/dev/null)
    tx=$(cat $host/invalid_tx_word_count 2>/dev/null)
    err=$(cat $host/error_frames 2>/dev/null)
    los=$(cat $host/loss_of_signal_count 2>/dev/null)
    if [ "$crc" != "0x0" ] || [ "$tx" != "0x0" ] || [ "$err" != "0x0" ] || [ "$los" != "0x0" ]; then
        echo "ERRORS on $(basename $(dirname $host)): crc=$crc tx=$tx err=$err los=$los"
    fi
done

# Check messages for Abort commands
grep -i "abort" /var/log/messages | tail -20

# Check for SCSI errors
dmesg | grep -i "scsi\|error\|fault\|abort"
```

### Processes in D State (Uninterruptible I/O)

```bash
# Find processes stuck in D state
ps aux | awk '$8 ~ /D/ {print}'

# Check which device is causing the wait
cat /proc/<pid>/wchan

# Check for blocked I/O
cat /proc/<pid>/stack
```

## List SCSI Devices

```bash
# lsscsi — clean view of all SCSI devices
lsscsi
# [0:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sda
# [2:0:0:1]    disk    DGC      RAID 5           0533  /dev/sdb
# [2:0:1:1]    disk    DGC      RAID 5           0533  /dev/sdc

# With size
lsscsi -s

# With SCSI generic device
lsscsi -g

# Verbose (shows sysfs paths)
lsscsi -v

# Install if not present
yum install lsscsi     # RHEL
apt install lsscsi     # Ubuntu
```

## SCSI Inquiry (sg_inq)

```bash
# Detailed SCSI inquiry (vendor, product, serial, WWID)
sg_inq /dev/sdb

# Show only serial number
sg_inq --page=0x80 /dev/sdb

# Show device identification (WWN/NAA)
sg_inq --page=0x83 /dev/sdb

# Install
yum install sg3_utils    # RHEL
apt install sg3-utils    # Ubuntu
```

## Get Device WWID (scsi_id)

```bash
# Get WWID for a device
/lib/udev/scsi_id -g -u /dev/sdb

# All disks with WWIDs
for dev in /dev/sd?; do
    echo "$dev: $(/lib/udev/scsi_id -g -u $dev 2>/dev/null)"
done

# Alternative path (older RHEL)
/sbin/scsi_id -g -u -s /block/sdb
```

## Remove a SCSI Device

```bash
# Set path offline first
echo offline > /sys/block/sdb/device/state

# Delete the SCSI device
echo 1 > /sys/block/sdb/device/delete

# Verify it's gone
lsscsi
lsblk
```

> **Warning:** Always unmount filesystems, deactivate LVM, and remove from multipathing before deleting a SCSI device.

## FC Host Extended Attributes

```bash
# Node name (WWNN)
cat /sys/class/fc_host/host0/node_name

# Port name (WWPN)
cat /sys/class/fc_host/host0/port_name

# Fabric name
cat /sys/class/fc_host/host0/fabric_name

# Supported speeds
cat /sys/class/fc_host/host0/supported_speeds

# Port type
cat /sys/class/fc_host/host0/port_type

# Port ID (FC address)
cat /sys/class/fc_host/host0/port_id

# All attributes for all FC hosts
systool -c fc_host -v
# Requires: yum install sysfsutils
```

## FC Timeout Tuning

Control how long the system waits before marking a path as failed:

```bash
# View current timeouts
cat /sys/class/fc_remote_ports/rport-*/dev_loss_tmo
cat /sys/class/fc_remote_ports/rport-*/fast_io_fail_tmo

# Set dev_loss_tmo (seconds before path is permanently removed)
# Default: 30 (some arrays recommend 600 or higher)
echo 600 > /sys/class/fc_remote_ports/rport-3:0-0/dev_loss_tmo

# Set fast_io_fail_tmo (seconds before I/O fails back to multipath layer)
# Default: off (wait full dev_loss_tmo). Set lower for faster failover.
echo 5 > /sys/class/fc_remote_ports/rport-3:0-0/fast_io_fail_tmo

# Set for all remote ports
for rport in /sys/class/fc_remote_ports/rport-*/dev_loss_tmo; do
    echo 600 > $rport
done
for rport in /sys/class/fc_remote_ports/rport-*/fast_io_fail_tmo; do
    echo 5 > $rport
done
```

### Make Persistent (udev Rule)

```bash
# /etc/udev/rules.d/99-fc-timeouts.rules
ACTION=="add", SUBSYSTEM=="fc_remote_ports", ATTR{dev_loss_tmo}="600"
ACTION=="add", SUBSYSTEM=="fc_remote_ports", ATTR{fast_io_fail_tmo}="5"
```

```bash
# Reload udev rules
udevadm control --reload-rules
```

### Timeout Recommendations

| Timeout | Default | Recommendation | Purpose |
|---------|---------|---------------|---------|
| `dev_loss_tmo` | 30s | 600s (SAN) | Time before SCSI device is removed |
| `fast_io_fail_tmo` | off | 5s | Time before I/O is failed to multipath layer |

- Short `fast_io_fail_tmo` = faster failover to alternate path
- Long `dev_loss_tmo` = path not permanently removed during brief outages (fabric switch reboot, cable reseat)

## Persistent Device Naming (udev Rules)

Create consistent device names that survive reboots:

```bash
# /etc/udev/rules.d/99-san-disks.rules

# By WWID
KERNEL=="sd*", SUBSYSTEM=="block", ENV{ID_SERIAL}=="36006016031a0220004862e259803df11", SYMLINK+="san/data_lun01"

# By vendor + serial
KERNEL=="sd*", ATTR{vendor}=="DGC", ATTR{serial}=="FCNPR063600652", SYMLINK+="san/emc_lun5"
```

```bash
# Reload rules
udevadm control --reload-rules
udevadm trigger

# Verify
ls -la /dev/san/
```

## Identifying Devices: When to Use What

| Tool | Best For |
|------|----------|
| `lsscsi` | Quick list of all SCSI devices with vendor/model |
| `lsblk` | Device tree with mount points and sizes |
| `multipath -ll` | Multipath device → path mapping |
| `ls /dev/disk/by-id/` | Persistent names (WWID-based) |
| `ls /dev/disk/by-path/` | HBA:target:LUN path mapping |
| `sg_inq /dev/sdX` | Detailed SCSI inquiry (serial, NAA, pages) |
| `/lib/udev/scsi_id` | Get WWID for a device |
| `udevadm info --query=all --name=/dev/sdX` | All udev properties for a device |

## iSCSI (iscsiadm)

### Install

```bash
yum install iscsi-initiator-utils    # RHEL
apt install open-iscsi               # Ubuntu
```

### Configure Initiator Name

```bash
# View current initiator name
cat /etc/iscsi/initiatorname.iscsi

# Set initiator name (unique per host)
echo "InitiatorName=iqn.2024-01.com.example:server01" > /etc/iscsi/initiatorname.iscsi

# Restart service
systemctl restart iscsid
```

### Discovery and Login

```bash
# Discover targets on a portal
iscsiadm -m discovery -t sendtargets -p 192.168.1.100

# List discovered targets
iscsiadm -m node

# Login to a specific target
iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 --login

# Login to all discovered targets
iscsiadm -m node --login

# Logout from a target
iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 --logout

# Logout from all targets
iscsiadm -m node --logout
```

### Session Management

```bash
# Show active sessions
iscsiadm -m session

# Show session details
iscsiadm -m session -P 3

# Rescan iSCSI sessions (after LUN add/resize)
iscsiadm -m session --rescan

# Show connection info
iscsiadm -m session -P 1
```

### Automatic Login at Boot

```bash
# Set node to auto-start
iscsiadm -m node -T iqn.2024-01.com.storage:lun01 -p 192.168.1.100 -o update -n node.startup -v automatic

# Enable iSCSI service at boot
systemctl enable iscsid
systemctl enable iscsi
```

### Configure CHAP Authentication

```bash
# Set CHAP credentials
iscsiadm -m node -T <target> -p <portal> -o update -n node.session.auth.authmethod -v CHAP
iscsiadm -m node -T <target> -p <portal> -o update -n node.session.auth.username -v myuser
iscsiadm -m node -T <target> -p <portal> -o update -n node.session.auth.password -v mypassword
```

### Remove a Target

```bash
# Logout
iscsiadm -m node -T <target> -p <portal> --logout

# Delete node record
iscsiadm -m node -T <target> -p <portal> -o delete

# Remove discovery record
iscsiadm -m discoverydb -t sendtargets -p <portal> -o delete
```

### iSCSI Configuration Files

| File | Purpose |
|------|---------|
| `/etc/iscsi/initiatorname.iscsi` | Initiator IQN name |
| `/etc/iscsi/iscsid.conf` | iSCSI daemon configuration |
| `/var/lib/iscsi/nodes/` | Discovered target configs |
| `/var/lib/iscsi/send_targets/` | Discovery portal configs |
