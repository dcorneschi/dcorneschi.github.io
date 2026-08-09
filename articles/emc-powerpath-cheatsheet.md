# EMC PowerPath Cheatsheet

EMC PowerPath is a proprietary multipathing solution for Dell EMC (formerly EMC²) storage arrays. It provides automatic path failover, load balancing, and path management for VMAX, VNX, Unity, PowerStore, XtremIO, and other Dell EMC arrays.

## Install

```bash
# RHEL / CentOS / Rocky (from EMC RPM)
sudo rpm -ivh EMCpower.LINUX-*.rpm

# Check if installed
rpm -qa | grep EMCpower

# Verify version
powermt version

# Start PowerPath
sudo systemctl start PowerPath     # RHEL 7+
sudo /etc/init.d/PowerPath start   # RHEL 6

# Enable at boot
sudo systemctl enable PowerPath    # RHEL 7+
sudo chkconfig PowerPath on        # RHEL 6
```

## Essential Commands

### powermt — Primary Management Tool

```bash
# Display all PowerPath devices and paths
powermt display dev=all

# Display specific device
powermt display dev=emcpowera

# Display with HBA information
powermt display hba_mode

# Display device with ports
powermt display dev=all class=all

# Display summary (short format)
powermt display

# Check PowerPath configuration
powermt check

# Save configuration
powermt save

# Restore configuration from saved state
powermt restore
```

### Display Output Explained

```
Pseudo name=emcpowera
CLARiiON ID=CK200012345678 [prod-array]
Logical device ID=600601608E661A00E4710BF88370E211 [LUN 5]
state=alive; policy=CLAROpt; queued-IOs=0
Owner: default=SP A, current=SP A
==============================================================================
--- Host ---     - Loss of Data -  ---- I/O Path ---  -- Stats ---
### HW Path       I/O Paths  Inact State  Mode   Q-IOs Errors
==============================================================================
3   qla2xxx       sdb        0     alive  active 0     0
4   qla2xxx       sdc        0     alive  active 0     0
5   qla2xxx       sdd        0     alive  standby 0    0
6   qla2xxx       sde        0     alive  standby 0    0
```

### Path States

| State | Meaning |
|-------|---------|
| `alive` | Path is healthy |
| `dead` | Path has failed |
| `degraded` | Path is partially functional |

### Path Modes

| Mode | Meaning |
|------|---------|
| `active` | Path is handling I/O |
| `standby` | Path is available but not active (failover candidate) |
| `unlic` | Path is unlicensed |

## Path Management

### Fail and Restore Paths

```bash
# Manually fail a path (testing)
powermt remove hba=3

# Restore a failed path
powermt restore hba=3

# Restore all failed paths
powermt restore

# Check and recover paths automatically
powermt check
powermt restore
```

### Set Path Policy

```bash
# Show current policy
powermt display dev=emcpowera

# Change I/O policy for a device
powermt set policy=CLAROpt dev=emcpowera
powermt set policy=SymmOpt dev=emcpowera
powermt set policy=BasicFailover dev=emcpowera
powermt set policy=RoundRobin dev=emcpowera
powermt set policy=LeastBlocks dev=emcpowera
powermt set policy=LeastIOs dev=emcpowera

# Set policy for all devices
powermt set policy=CLAROpt dev=all
```

### I/O Policies

| Policy | Description | Use Case |
|--------|-------------|----------|
| `CLAROpt` | CLARiiON/VNX optimized (ALUA-aware) | VNX, CLARiiON |
| `SymmOpt` | Symmetrix/VMAX optimized | VMAX, PowerMax |
| `BasicFailover` | Active/passive (one path active) | Simple failover |
| `RoundRobin` | Distribute I/O across all paths | All arrays |
| `LeastBlocks` | Send I/O to path with least blocks queued | Performance |
| `LeastIOs` | Send I/O to path with least I/Os queued | Performance |
| `Adaptive` | Automatically adjusts based on load | General |
| `RoundRobinWithSubset` | Round-robin within preferred paths | ALUA arrays |

## Device Management

### Add and Remove Devices

```bash
# Scan for new LUNs
powermt check

# Configure new devices after SCSI rescan
powermt config

# Remove a device (after unmount and LVM cleanup)
powermt remove dev=emcpowera

# Remove all dead paths
powermt remove dev=all

# Display only dead/missing devices
powermt display dev=all | grep -A5 "dead"
```

### Manage HBAs

```bash
# Display HBA information
powermt display hba_mode

# Remove paths on a specific HBA
powermt remove hba=3

# Restore paths on a specific HBA
powermt restore hba=3

# Set HBA mode
powermt set mode=active hba=3
powermt set mode=standby hba=3
```

## Licensing

```bash
# Display license information
emcpreg -list

# Register a license key
emcpreg -add <key>

# Check license status
powermt check_registration

# Display registration details
emcpreg -list all
```

## Array-Specific Commands

### CLARiiON / VNX / Unity

```bash
# Display with SP (Storage Processor) ownership
powermt display dev=all

# Set trespass policy (automatic SP failover)
powermt set policy=CLAROpt dev=all

# Owner shows: default=SP A, current=SP A
# If current != default, a trespass has occurred
```

### Symmetrix / VMAX / PowerMax

```bash
# Set Symmetrix optimized policy
powermt set policy=SymmOpt dev=all

# Display Symmetrix device ID
powermt display dev=emcpowera
# Shows: Symmetrix ID=000195900XXX
```

### XtremIO

```bash
# Round-robin works best for all-flash
powermt set policy=RoundRobin dev=all
```

## Monitoring and Statistics

```bash
# Display I/O statistics
powermt display dev=all

# Check for errors
powermt display dev=all | grep -i "error\|dead\|degraded"

# Count paths per state
powermt display dev=all | grep -c "alive"
powermt display dev=all | grep -c "dead"

# Monitor path changes
watch -n 5 'powermt display dev=all | grep -E "state|Mode"'

# Log PowerPath events
cat /var/log/emcpower.log
tail -f /var/log/emcpower.log
```

## Configuration Files

| File | Purpose |
|------|---------|
| `/etc/emc/mpaa.conf` | PowerPath main configuration |
| `/etc/emc/mpaa.lams` | License and array management |
| `/etc/emcp_devicesDB.dat` | Device database |
| `/etc/emcp_registration` | Registration/license data |
| `/var/log/emcpower.log` | PowerPath log file |
| `/etc/powermt.custom` | Custom configuration (saved by `powermt save`) |

## Common Procedures

### Add a New LUN

```bash
# 1. Present the LUN from the storage array (array-side)

# 2. Rescan SCSI bus on the host
for host in /sys/class/scsi_host/host*/scan; do
    echo "- - -" > $host
done

# 3. Configure PowerPath to detect new device
powermt config

# 4. Verify new device appears
powermt display dev=all

# 5. Save configuration
powermt save
```

### Remove a LUN

```bash
# 1. Unmount the filesystem
umount /mountpoint

# 2. Remove from LVM (if used)
lvremove /dev/vg/lv
vgremove vg
pvremove /dev/emcpowera

# 3. Remove from PowerPath
powermt remove dev=emcpowera

# 4. Delete SCSI paths
echo 1 > /sys/block/sdb/device/delete
echo 1 > /sys/block/sdc/device/delete

# 5. Save configuration
powermt save

# 6. Remove LUN from storage array (array-side)
```

### Resize a LUN

Tested on RHEL 6+ without partitions on the LUN (raw filesystem or LVM directly on the device).

```bash
# 1. Expand LUN on storage array (array-side)
# The OS may auto-detect the change:
dmesg | grep -i "capacity.*change"
# sd 1:0:1:4: Capacity data has changed

# 2. Find native paths for the device
powermt display dev=emcpowera

# 3. Rescan each native path
for sd in $(powermt display dev=emcpowera | grep sd | awk '{print $3}'); do
    echo 1 > /sys/block/$sd/device/rescan
done

# Verify in dmesg:
# sdu: detected capacity change from 53687091200 to 107374182400

# 4. Save PowerPath configuration
powermt config
powermt save

# 5. Verify new size (underlying devices AND emcpower device)
fdisk -l /dev/emcpowera | grep "Disk /dev"
grep emcpowera /proc/partitions

# 6. Resize PV and verify
pvresize /dev/emcpowera
pvdisplay /dev/emcpowera

# 7. Extend LV (use all available space)
lvextend -l +100%FREE /dev/vg/lv
lvdisplay /dev/vg/lv

# 8. Resize the filesystem
resize2fs /dev/vg/lv           # ext2/ext3/ext4
xfs_growfs /mountpoint         # XFS (uses mount point, not device)
```

### Replace a Failed HBA

```bash
# 1. Remove dead paths on the failed HBA
powermt remove hba=3

# 2. Replace the physical HBA (requires downtime)

# 3. Rescan SCSI bus
echo "- - -" > /sys/class/scsi_host/host3/scan

# 4. Restore paths
powermt restore

# 5. Verify all paths are alive
powermt display dev=all

# 6. Save configuration
powermt save
```

## Troubleshooting

### Paths Showing "dead"

```bash
# Check which paths are dead
powermt display dev=all | grep dead

# Try to restore
powermt restore

# If restore fails, check physical connectivity
cat /sys/class/fc_host/host*/port_state

# Check for SCSI errors
dmesg | grep -i "scsi\|error\|fault"

# Force rescan
echo "- - -" > /sys/class/scsi_host/host*/scan
powermt check
powermt restore
```

### Device Shows "dead" (All Paths Down)

```bash
# Check array connectivity
powermt display dev=emcpowera

# Check if array SP is accessible
# (use array-specific CLI — naviseccli for VNX, symcli for VMAX)

# Check FC port state
cat /sys/class/fc_host/host*/port_state
# Should show: Online

# Check HBA link
cat /sys/class/fc_host/host*/speed
```

### "emcpowerX: open failed" Error

```bash
# Device may be in use
lsof /dev/emcpowera
fuser -mv /dev/emcpowera

# Or unmounted but still has LVM
pvs | grep emcpower
vgs
```

### PowerPath Not Starting

```bash
# Check service status
systemctl status PowerPath

# Check logs
cat /var/log/emcpower.log | tail -50
journalctl -u PowerPath

# Check license
emcpreg -list
powermt check_registration

# Verify kernel module is loaded
lsmod | grep emcp
```

### After OS Upgrade

```bash
# PowerPath may need reinstalling after kernel upgrade
rpm -qa | grep EMCpower

# Reinstall if needed
rpm -ivh --force EMCpower.LINUX-*.rpm

# Rebuild for new kernel
/etc/init.d/PowerPath config

# Restart
systemctl restart PowerPath
powermt config
powermt display dev=all
```

## PowerPath vs DM-Multipath

| Feature | PowerPath | DM-Multipath |
|---------|-----------|--------------|
| Vendor | Dell EMC (proprietary) | Open source (Linux kernel) |
| Cost | Licensed (per-server) | Free |
| Array support | EMC/Dell arrays (optimized) | All arrays (generic) |
| I/O policies | Array-specific optimization | Standard policies |
| Management | `powermt` | `multipath`, `multipathd` |
| Device names | `/dev/emcpowerX` | `/dev/mapper/mpathX` |
| Configuration | `/etc/emc/mpaa.conf` | `/etc/multipath.conf` |
| Kernel module | `emcp` | `dm-multipath` |

> **Note:** Dell EMC recommends DM-Multipath for newer deployments. PowerPath is primarily maintained for legacy environments and specific use cases requiring EMC-optimized failover.

## Quick Reference

```bash
powermt display dev=all              # Show all devices and paths
powermt display dev=emcpowera        # Show specific device
powermt check                        # Check and auto-recover paths
powermt restore                      # Restore failed paths
powermt remove hba=N                 # Fail an HBA (testing)
powermt config                       # Detect new devices
powermt save                         # Save configuration
powermt set policy=X dev=all         # Change I/O policy
powermt version                      # Show version
emcpreg -list                        # Show license
```

## Service Management (Extended)

```bash
# Start/stop PowerPath
powermt startup
powermt shutdown

# Load/unload kernel driver
powermt load
powermt unload
```

## Display Commands (Extended)

```bash
# Display by path class
powermt display class=dead
powermt display class=alive
powermt display class=standby

# Display I/O policy
powermt display policy

# Display options/settings
powermt display options

# Display paths only
powermt display paths

# Display I/O statistics
powermt display iocount

# Display with interval (real-time monitoring)
powermt display iocount interval=5

# Display detailed I/O stats
powermt display iocount detailed

# Display performance stats
powermt display stats

# Display event log
powermt display events

# Display errors per path
powermt display errors

# Display path load
powermt display load dev=all

# Display queue depths
powermt display queuedepth

# Display latency
powermt display latency

# Display license
powermt display license

# Display HBA info
powermt display hba_info

# Display kernel modules
powermt display modules

# Display internal state (debugging)
powermt display internal

# Display registration status
powermt display registration

# Reset statistics
powermt reset stats
```

## Path Priority and Autorestore

```bash
# Set path priority
powermt set priority=1 dev=emcpowera hba=3

# Enable periodic autorestore (recover dead paths automatically)
powermt set periodic_autorestore=on

# Or with interval (seconds)
powermt config periodic_autorestore=on interval=300

# Enable autofailback
powermt config autofailback=on

# Enable path monitoring
powermt config monitor=on
```

## Path Testing

```bash
# Test a specific path
powermt test hba=3 dev=emcpowera

# Test all paths for a device
powermt test dev=emcpowera

# Validate HBA connectivity
powermt check hba=3
```

## Advanced Policies

| Policy | Description |
|--------|-------------|
| `StreamIO` | Optimized for sequential I/O workloads |
| `AdaptiveOpt` | Dynamically adapts based on workload patterns |

```bash
# Set sequential I/O optimized policy
powermt set policy=StreamIO dev=emcpowera

# Set adaptive policy
powermt set policy=AdaptiveOpt dev=emcpowerb
```

## Timeouts and Tuning

```bash
# Set fail timeout (seconds before marking path dead)
powermt config fail_time=30 dev=all

# Set restore timeout (seconds before restoring a failed path)
powermt config restore_time=300 dev=all

# Set HBA optimization
powermt config hba_optimization=on

# Enable optimization for all devices
powermt config optimization=on dev=all
```

## Diagnostics

```bash
# Generate diagnostic package (for support)
powermt collect

# Enable verbose logging
powermt config verbose=on

# Check log files
tail -f /var/log/emcpower.log
grep -i powerpath /var/log/messages

# Additional log locations
# /var/log/powerpath.log
# /var/adm/powerpath.log
```

## Integration with Veritas Volume Manager (VxVM)

```bash
# Scan for new PowerPath devices in VxVM
vxdisk scandisks

# List PowerPath devices in VxVM
vxdisk list | grep -i emcpower
```

## Integration with Oracle ASM

```bash
# Discover PowerPath devices for ASM
/etc/init.d/oracleasm scandisks

# Create ASM disk on PowerPath device
/etc/init.d/oracleasm createdisk DATA1 /dev/emcpowera

# List ASM disks
/etc/init.d/oracleasm listdisks
```

## Emergency Procedures

```bash
# Emergency path failover (take specific HBA offline)
powermt set dev=emcpowera hba=3 mode=standby

# Force immediate path restoration
powermt restore force dev=emcpowera

# Emergency device removal
powermt remove dev=emcpowera force

# Panic recovery (last resort — clean slate)
powermt config force_clean=on
powermt remove dev=all
powermt check_registration dev=all
powermt config
powermt save
```

## Uninstall PowerPath

```bash
# 1. Remove all LUNs from PowerPath first (see procedures above)
powermt remove dev=all
powermt release

# 2. Stop the service
systemctl stop PowerPath.service    # RHEL 7+
/etc/init.d/PowerPath stop          # RHEL 5-6

# 3. Check installed package
rpm -qa | grep -i EMCpower
# or
rpm -qa | grep -i DellPower

# 4. Remove the package
rpm -e EMCpower.LINUX-<version>
# or
rpm -e DellPower.LINUX-<version>

# 5. Verify removal
rpm -qa | grep -i EMCpower
powermt version    # should fail: command not found
```

> **Warning:** Ensure all filesystems on PowerPath devices are unmounted and VGs are deactivated before uninstalling. Failure to do so may cause kernel panics or data loss.

## Version-Specific Notes

| Version | Key Features |
|---------|-------------|
| PowerPath 5.x | Advanced policies (AdaptiveOpt, StreamIO) |
| PowerPath 6.x | Enhanced VMware integration, VNX2 support |
| PowerPath 7.x | Improved analytics, PowerStore/Unity support |

## Additional Configuration Files

| File | Purpose |
|------|---------|
| `/etc/powerpath/powermt.custom` | Main PowerPath config |
| `/etc/powerpath/license.xml` | License file |
| `/etc/powerpath/powerpath.conf` | Additional configuration |
| `/var/log/powerpath.log` | Log file (alternative location) |
| `/var/adm/powerpath.log` | Log file (Solaris/older) |

## Install and Register

```bash
# Install from RPM
rpm -ivh /tmp/EMCPower.LINUX-6.2.0.00.00-051.RHEL6.x86_64.rpm

# Register license
emcpreg -install

# Verify registration
powermt check_registration

# Start service
/etc/init.d/PowerPath start          # RHEL 5-6
systemctl start PowerPath.service    # RHEL 7+
```

## Upgrade PowerPath (e.g., v7 → v8)

```bash
# 1. Copy new RPM to server
scp DellPower.LINUX-8.1.0.00.00-070.RHEL7.x86_64.rpm server:/tmp

# 2. Backup registration and config
cp /etc/emcp_registration /etc/emcp_registration.bkp
cp /etc/powermt_custom.xml /etc/powermt_custom.xml.bkp

# 3. Document existing device mappings
powermt display dev=all > ~/powerpath_before_upgrade.out

# 4. Unmount filesystems on PowerPath devices
umount <filesystem>

# 5. Deactivate volume groups
vgchange -a n <vg>

# 6. Upgrade RPM
rpm -Uhv /tmp/DellPower.LINUX-8.1.0.00.00-070.RHEL7.x86_64.rpm

# 7. Start PowerPath
systemctl start PowerPath.service

# 8. Activate volume groups
vgchange -a y <vg>

# 9. Mount filesystems
mount <filesystem>    # or: mount -a

# 10. Reboot server

# 11. Verify
less /var/log/powerpath_install.log
less /var/log/messages
powermt version
powermt display dev=all
```

## Additional Display Commands

```bash
# Display I/O stats at specified interval
powermt display every=5

# Display port status
powermt display port_mode

# Display pseudo device and LUN ID on same line
powermt display dev=all | grep -E 'Pseudo|Logical' | paste -d "  " - -

# Or with egrep
powermt display dev=all | egrep 'Pseudo|Logical' | paste -d " " - -
```

## Additional Path Commands

```bash
# Remove dead paths forcefully (no confirmation)
powermt check force

# Force all paths to active mode
powermt set mode=active dev=all force

# Release a device (removes pseudo from /dev and /sys)
powermt release dev=emcpowera

# Release all devices
powermt release
```

> **Note:** After `powermt remove`, always run `powermt release` — failing to do so leaves the pseudo device visible in `/dev` and `/sys`.

## Save and Load Configuration

```bash
# Save to default location (/etc/powermt_custom.xml)
powermt save

# Save to a specific file (for backup/versioning)
powermt save file=/etc/powermt.20240115

# Load a specific saved configuration
powermt load file=/etc/powermt.20240115
```

## Pseudo Device Management (emcpadm)

```bash
# List all pseudo devices
emcpadm getusedpseudos

# Rename a pseudo device
emcpadm renamepseudo -s emcpowerd -t emcpowera
```

## Remove a LUN (Full Procedure)

### Generic Steps (Any Multipath Software)

1. Close all users of the device and backup data as needed
2. `umount` any filesystems mounted on the device
3. Remove the device from any LVM/md using it; update `/etc/fstab`
4. Flush/remove the device from multipathing (PowerPath)
5. `blockdev --flushbufs` to flush outstanding I/O to all paths
6. Remove each path from the SCSI subsystem

### PowerPath-Specific

```bash
# 1. Unmount
umount /mountpoint

# 2. Remove from LVM
lvremove /dev/vg/lv
vgremove vg
pvremove /dev/emcpowera

# 3. Update /etc/fstab (remove or comment the entry)

# 4. Display device and paths
powermt display dev=emcpowera

# 5. Flush buffers
blockdev --flushbufs /dev/emcpowera

# 6. Remove from PowerPath
powermt remove dev=emcpowera

# 7. Release pseudo device
powermt release dev=emcpowera

# 8. Delete SCSI paths
for i in c d e f; do
    echo 1 > /sys/block/sd$i/device/delete
done

# 9. Save configuration
powermt save
```

## Remove All LUNs (Bulk)

```bash
# 1. Capture SCSI paths
powermt display dev=all | egrep "lpfc|qla" | awk '{print $3}' > stale-devices.lst

# 2. Remove all PowerPath devices
powermt remove dev=all

# 3. Delete underlying SCSI devices
for i in $(cat stale-devices.lst); do
    echo 1 > /sys/block/$i/device/delete
done

# 4. Release and save
powermt release
powermt save

# 5. Verify cleanup
emcpadm getusedpseudos
lsscsi | grep DGC
```

## Performance Monitoring (perfmon)

Enable detailed performance counters (throughput, response time, I/O size, queue depth, latency distribution):

```bash
# Enable performance monitoring
powermt set perfmon=on

# Display performance stats for all devices (verbose)
powermt display perf dev=all verbose

# Display performance stats per bus/HBA
powermt display perf bus verbose

# Disable when done (reduces overhead)
powermt set perfmon=off
```

Output includes:
- Read, Write, and Total Throughput
- Read, Write, and All Average Response Time
- Read, Write per-I/O size
- Queued-I/Os
- Latency Distribution
- Retry delta
- Error delta
