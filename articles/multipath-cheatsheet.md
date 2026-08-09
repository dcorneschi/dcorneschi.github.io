# Multipath Cheatsheet

Device Mapper Multipath (DM-Multipath) provides redundant paths between a server and storage. If one path fails, I/O is rerouted through the remaining paths — providing failover and optionally load balancing.

## Install

```bash
# RHEL / CentOS / Rocky
sudo yum install device-mapper-multipath    # RHEL 7
sudo dnf install device-mapper-multipath    # RHEL 8+

# Ubuntu / Debian
sudo apt install multipath-tools

# Enable and start
sudo systemctl enable multipathd
sudo systemctl start multipathd
sudo systemctl status multipathd
```

## Generate Default Configuration

```bash
# Generate /etc/multipath.conf with default settings
sudo mpathconf --enable

# Enable with user-friendly names
sudo mpathconf --enable --user_friendly_names y

# Enable with all options
sudo mpathconf --enable --with_multipathd y --with_module y --with_chkconfig y

# View available mpathconf options
mpathconf --help
```

## Setup Procedure (Step by Step)

1. Install: `yum install device-mapper-multipath` (or `dnf`)
2. Generate config: `mpathconf --enable --user_friendly_names y`
3. Edit `/etc/multipath.conf` — configure blacklist, set defaults
4. Dry-run: `multipath -d` (verify without applying)
5. Start daemon: `systemctl start multipathd`
6. Detect devices: `multipath -v2`
7. Verify: `multipath -ll`
8. Enable at boot: `systemctl enable multipathd`

## Identify Storage Arrays

```bash
# View all attached SCSI devices
cat /proc/scsi/scsi

# Determine vendor and model for a specific disk
cat /sys/block/sda/device/vendor
cat /sys/block/sda/device/model

# Check all disks
for dev in /sys/block/sd*; do
    echo "$(basename $dev): $(cat $dev/device/vendor) $(cat $dev/device/model)"
done

# Then check defaults for your array
cat /usr/share/doc/device-mapper-multipath-*/multipath.conf.defaults | grep -A5 "vendor.*$(cat /sys/block/sdb/device/vendor | tr -d ' ')"
```

Use this information to determine if custom device configuration is needed or if the built-in defaults are sufficient.

## Configuration File

`/etc/multipath.conf`

### Minimal Configuration

```
defaults {
    user_friendly_names yes
    find_multipaths yes
}
```

### Full Example

```
defaults {
    user_friendly_names     yes
    find_multipaths         yes
    polling_interval        5
    path_selector           "round-robin 0"
    path_grouping_policy    multibus
    failback                immediate
    no_path_retry           fail
}

blacklist {
    devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
    devnode "^sd[a-z]$"    # blacklist local disks
}

devices {
    device {
        vendor                  "NETAPP"
        product                 "LUN"
        path_grouping_policy    group_by_prio
        path_selector           "round-robin 0"
        path_checker            tur
        features                "3 queue_if_no_path pg_init_retries 50"
        prio                    ontap
        failback                immediate
        no_path_retry           queue
    }
}

multipaths {
    multipath {
        wwid    360000000000000001
        alias   data_lun01
    }
    multipath {
        wwid    360000000000000002
        alias   data_lun02
    }
}
```

### Key Configuration Options

| Option | Description | Values |
|--------|-------------|--------|
| `user_friendly_names` | Use `mpathN` instead of WWID | `yes` / `no` |
| `find_multipaths` | Only create multipath for devices with multiple paths | `yes` / `no` / `smart` / `greedy` |

> **`find_multipaths yes`** is the recommended setting for most environments. It excludes any device from multipath until at least two paths to that device are detected simultaneously — eliminating the need to manually blacklist single-path devices.
| `polling_interval` | Seconds between path checks | Integer (default: 5) |
| `path_selector` | Load balancing algorithm | `round-robin 0`, `queue-length 0`, `service-time 0` |
| `path_grouping_policy` | How paths are grouped | `failover`, `multibus`, `group_by_prio`, `group_by_node_name` |
| `failback` | When to revert to preferred path | `immediate`, `manual`, `followover`, integer (seconds) |
| `no_path_retry` | Behavior when all paths fail | `fail`, `queue`, integer (retry count) |
| `path_checker` | Method to check path health | `tur`, `readsector0`, `directio` |

### Path Checkers

| Checker | Description |
|---------|-------------|
| `tur` | Test Unit Ready — default, sends SCSI TUR command |
| `readsector0` | Read first sector of device |
| `directio` | Direct I/O read test |
| `hp_sw` | HP StorageWorks specific |
| `emc_clariion` | EMC CLARiiON specific |
| `rdac` | RDAC/LSI controller specific |

### Path Grouping Policies

| Policy | Description |
|--------|-------------|
| `failover` | One path per group (active/standby) |
| `multibus` | All paths in one group (active/active) |
| `group_by_prio` | Group by priority value |
| `group_by_node_name` | Group by target node |
| `group_by_serial` | Group by serial number |

## Essential Commands

### View Multipath Devices

```bash
# Show all multipath devices (brief)
multipath -ll

# Show all multipath devices (verbose)
multipath -ll -v3

# Show topology
multipath -ll

# Show specific device
multipath -ll mpathb

# Show WWIDs
multipath -v2

# Dry-run — show what would be configured without applying
multipath -d -v3

# Test configuration file syntax
multipath -t

# List all multipath devices via device-mapper
dmsetup ls --target multipath

# Example output:
# mpathb (360000000000000001) dm-3 NETAPP,LUN
# size=100G features='3 queue_if_no_path pg_init_retries 50' hwhandler='0' wp=rw
# |-+- policy='round-robin 0' prio=50 status=active
# | `- 3:0:0:1 sdb 8:16 active ready running
# `-+- policy='round-robin 0' prio=10 status=enabled
#   `- 4:0:0:1 sdd 8:48 active ready running

# List multipath devices only (names)
multipath -l

# Show in flat format
multipathd show maps
```

### Path Status

```bash
# Show all paths
multipathd show paths

# Show paths in formatted columns
multipathd show paths format "%d %s %t %i %o %T %z"

# Show path details for a specific device
multipathd show paths raw format "%d %D %s"

# Check path state
multipath -ll | grep -E "status|running|faulty"
```

### Manage Multipath Devices

```bash
# Reconfigure (reload config without restart)
sudo multipath -r

# Flush (remove) a specific multipath device
sudo multipath -f mpathb

# Flush all unused multipath devices
sudo multipath -F

# Force reload all multipath maps
sudo multipath

# Force recreation of multipath device bindings
sudo multipath -B

# Add a new path
sudo multipath -a /dev/sdb

# Remove a path from a device
sudo multipathd remove path sdb
```

### Rescan for New Devices

```bash
# Rescan SCSI bus (find new LUNs)
for host in /sys/class/scsi_host/host*/scan; do
    echo "- - -" | sudo tee $host
done

# Then reconfigure multipath
sudo multipath -r

# Or rescan specific host
echo "- - -" | sudo tee /sys/class/scsi_host/host0/scan

# Verify new paths
multipath -ll
```

### Resize a Multipath Device

```bash
# After storage-side LUN expansion:

# 1. Rescan SCSI devices
for sd in /sys/block/sd*/device/rescan; do
    echo 1 | sudo tee $sd
done

# 2. Reload multipath map
sudo multipathd resize map mpathb

# 3. Verify new size
multipath -ll mpathb

# 4. Grow the filesystem (if needed)
sudo xfs_growfs /mountpoint           # XFS
sudo resize2fs /dev/mapper/mpathb     # ext4
```

## multipathd Interactive Console

```bash
# Enter interactive mode
sudo multipathd -k

# Or send individual commands
sudo multipathd show maps
sudo multipathd show paths
sudo multipathd show config
sudo multipathd show topology
sudo multipathd reconfigure
```

### Common multipathd Commands

```bash
multipathd show maps            # List all multipath devices
multipathd show paths           # List all paths
multipathd show config          # Show running configuration
multipathd show topology        # Show full topology
multipathd show maps topology   # Maps with path details
multipathd show paths format "%d %s %t %T"  # Custom format

multipathd reconfigure          # Reload configuration
multipathd resize map <name>    # Resize after LUN expansion
multipathd add path <dev>       # Add a path
multipathd remove path <dev>    # Remove a path
multipathd fail path <dev>      # Mark a path as failed
multipathd reinstate path <dev> # Restore a failed path
multipathd suspend map <name>   # Suspend I/O on a map
multipathd resume map <name>    # Resume I/O on a map
```

### Vendor-Specific Device Example (EMC)

```
devices {
    device {
        vendor                  "EMC"
        product                 "SYMMETRIX"
        path_grouping_policy    group_by_prio
        getuid_callout          "/sbin/scsi_id -g -u -s /block/%n"
        path_selector           "round-robin 0"
        path_checker            tur
        hardware_handler        "1 emc"
        prio                    emc
        failback                immediate
    }
}
```

## Blacklisting

Prevent local disks or specific devices from being managed by multipath:

```
# /etc/multipath.conf
blacklist {
    # By device node pattern
    devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
    devnode "^sd[a]$"          # local boot disk

    # By WWID
    wwid "26001405d27e00d0000900000490000"

    # By vendor/product
    device {
        vendor "ATA"
        product "*"
    }

    # By device property
    property "(ID_SCSI_VPD|ID_WWN)"
}

blacklist_exceptions {
    # Allow specific devices even if they match blacklist
    wwid "3600508b4000c4a5b0000300000490000"
}
```

```bash
# Check if a device is blacklisted
sudo multipath -c /dev/sda
# sda is not a valid multipath device path

# List blacklisted devices
sudo multipath -v3 2>&1 | grep "blacklist"
```

## Working with Device Names

```bash
# Find WWID of a device
sudo /lib/udev/scsi_id -g -u /dev/sdb

# Find all WWIDs
for dev in /dev/sd?; do
    echo "$dev: $(sudo /lib/udev/scsi_id -g -u $dev)"
done

# Find multipath device for a path
multipath -ll | grep sdb

# Find which dm device corresponds to a multipath name
ls -la /dev/mapper/mpathb
# lrwxrwxrwx 1 root root 7 ... /dev/mapper/mpathb -> ../dm-3

# Get WWID from multipath device
multipathd show maps format "%n %w"

# List all device mapper devices
dmsetup ls
dmsetup table
dmsetup info
```

## Using Multipath Devices

### Mount and fstab

```bash
# Always use /dev/mapper/mpathN or by-id paths (not /dev/sdX)

# Mount
sudo mount /dev/mapper/mpathb /data

# fstab entry (use mapper path or UUID)
/dev/mapper/mpathb    /data    xfs    defaults,_netdev    0 0

# Or by UUID (preferred)
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx    /data    xfs    defaults,_netdev    0 0

# Get UUID
sudo blkid /dev/mapper/mpathb
```

> **Important:** Use `_netdev` mount option for SAN-attached storage to ensure network is up before mount.

### LVM on Multipath

```bash
# Configure LVM to use multipath devices
# /etc/lvm/lvm.conf:
# filter = [ "a|/dev/mapper/mpath.*|", "r|/dev/sd.*|" ]
# OR
# filter = [ "a|/dev/dm-.*|", "r|/dev/sd.*|" ]

# Create PV on multipath device
sudo pvcreate /dev/mapper/mpathb

# Create VG
sudo vgcreate vg_data /dev/mapper/mpathb

# Create LV
sudo lvcreate -n lv_data -l 100%FREE vg_data

# Format and mount
sudo mkfs.xfs /dev/vg_data/lv_data
sudo mount /dev/vg_data/lv_data /data
```

### Partitions on Multipath

Always create partitions on the multipath device (`/dev/mapper/mpathX`), not on individual paths (`/dev/sdX`). Commands sent to the multipath device are passed through to the underlying physical disk.

```bash
# Create partition with parted (preferred)
sudo parted /dev/mapper/mpathb mklabel gpt
sudo parted /dev/mapper/mpathb mkpart primary xfs 0% 100%

# Or with fdisk
sudo fdisk /dev/mapper/mpathb

# Trigger partition detection with kpartx
# RHEL 7/8/9:
sudo kpartx -a -v /dev/mapper/mpathb

# RHEL 6 (requires -p p for partition separator):
sudo kpartx -a -p p -v /dev/mapper/mpathb

# Force multipathd to refresh and show new partition in /dev/mapper/
sudo multipath -r

# Partitions appear as:
# /dev/mapper/mpathb1, /dev/mapper/mpathb2, etc.
# (RHEL 6 with -p p: /dev/mapper/mpathbp1, /dev/mapper/mpathbp2)

# Remove partition mappings
sudo kpartx -d /dev/mapper/mpathb
```

> **Note:** If a partition doesn't show in `/dev/mapper/` after creation, run `kpartx -a` followed by `multipath -r`. A reboot is not required.

## Path States

| State | Meaning |
|-------|---------|
| `active` | Path is handling I/O |
| `enabled` | Path is available but not currently active |
| `faulty` | Path has failed |
| `shaky` | Path is intermittently failing |
| `ghost` | Path is accessible but not usable (some arrays) |

| Checker State | Meaning |
|---------------|---------|
| `running` | Path checker is active, path is healthy |
| `ready` | Path is ready for I/O |
| `faulty` | Path failed health check |
| `timeout` | Path check timed out |

## Failover Testing

```bash
# Manually fail a path (simulate failure)
sudo multipathd fail path sdb

# Check status — should show one path faulty
multipath -ll

# Restore the path
sudo multipathd reinstate path sdb

# Verify both paths active
multipath -ll

# Alternative: disable path via sysfs
echo offline | sudo tee /sys/block/sdb/device/state

# Restore
echo running | sudo tee /sys/block/sdb/device/state

# Alternative: fail path using "fail" state
echo "fail" | sudo tee /sys/block/sdb/device/state

# Remove a path entirely (simulate cable pull)
echo 1 | sudo tee /sys/block/sdb/device/delete
# To recover: rescan SCSI bus
```

## Performance Monitoring

```bash
# Monitor I/O statistics per device
iostat -x 1

# Check multipath device I/O stats
cat /proc/diskstats | grep dm-

# Check I/O scheduler on multipath device
cat /sys/block/dm-0/queue/scheduler

# Check queue depth per path
cat /sys/block/sdb/queue/nr_requests

# Check FC port state
cat /sys/class/fc_host/host*/port_state
cat /sys/class/fc_transport/target*/port_state
```

## Troubleshooting

### Multipath Device Not Created

```bash
# Check if multipath sees the paths
sudo multipath -v3

# Check if device is blacklisted
sudo multipath -c /dev/sdb

# Verify SCSI devices exist
lsscsi

# Check if multipathd is running
sudo systemctl status multipathd

# Manually trigger device creation
sudo multipath -a /dev/sdb
sudo multipath -r
```

### Path Shows "faulty"

```bash
# Check path details
multipathd show paths

# Check SCSI layer
cat /sys/block/sdb/device/state

# Check if FC link is up
cat /sys/class/fc_host/host*/port_state

# Rescan to recover
echo "- - -" | sudo tee /sys/class/scsi_host/host0/scan
sudo multipathd reinstate path sdb
```

### "device-mapper: remove ioctl failed: Device or resource busy"

```bash
# Something is using the device — find what
sudo fuser -mv /dev/mapper/mpathb
sudo lsof /dev/mapper/mpathb
sudo dmsetup info mpathb

# Unmount, close LVM, then flush
sudo umount /data
sudo vgchange -an vg_data
sudo multipath -f mpathb
```

### Lost Paths After Reboot

```bash
# Ensure multipathd starts before mounting
sudo systemctl enable multipathd

# Check that multipath module loads early
sudo dracut --force    # RHEL — rebuild initramfs with multipath
sudo update-initramfs -u    # Ubuntu

# Verify multipath is in initramfs
lsinitrd | grep multipath    # RHEL
```

### Check Multipath Logs

```bash
# Systemd journal
journalctl -u multipathd -f

# Kernel messages
dmesg | grep -i "mpath\|multipath\|scsi"

# Traditional log files
tail -f /var/log/messages       # RHEL
tail -f /var/log/syslog         # Ubuntu

# Increase verbosity (temporarily)
sudo multipathd -v3

# Enable debug logging in /etc/multipath.conf
# defaults {
#     verbosity 3
# }

# Temporary debug mode (foreground, verbose)
sudo multipathd -v3 -d
```

## Best Practices

1. Always backup `/etc/multipath.conf` before changes
2. Test configuration with `multipath -t` before applying
3. Use meaningful aliases for multipath devices
4. Regularly monitor path status with `multipath -ll`
5. Configure appropriate timeouts for your storage array
6. Use `find_multipaths yes` to avoid claiming single-path devices
7. Blacklist boot and OS devices appropriately
8. Set up alerting for path failures
9. Use `_netdev` in fstab for SAN-mounted filesystems
10. Rebuild initramfs after config changes (`dracut --force` or `update-initramfs -u`)

## Important Files

| File | Purpose |
|------|---------|
| `/etc/multipath.conf` | Main configuration |
| `/etc/multipath/bindings` | WWID to friendly name mappings |
| `/etc/multipath/wwids` | Known multipath WWIDs |
| `/var/lib/multipath/bindings` | Alternative bindings location (some distros) |
| `/sys/block/dm-*/slaves/` | Show underlying paths for a dm device |

## Quick Reference

```bash
multipath -ll                  # Show all multipath devices
multipath -r                   # Reconfigure (reload)
multipath -f <name>            # Flush (remove) a device
multipath -F                   # Flush all unused devices
multipath -a /dev/sdX          # Add a path
multipath -c /dev/sdX          # Check if device is valid multipath
multipathd show paths          # Show path status
multipathd show maps           # Show maps
multipathd fail path sdX       # Fail a path (testing)
multipathd reinstate path sdX  # Restore a path
multipathd resize map <name>   # Resize after LUN expand
multipathd reconfigure         # Reload config
```

## Device Name Paths

Multipath devices appear in three locations under `/dev`:

| Path | Usage |
|------|-------|
| `/dev/dm-N` | Internal to device-mapper — **never use directly** |
| `/dev/mpath/mpathN` | All multipath devices in one place — may not be available early at boot |
| `/dev/mapper/mpathN` | **Recommended** — persistent, created early in boot |

Always use `/dev/mapper/mpathN` for creating PVs, filesystems, and fstab entries.

## User-Friendly Names

By default, device names are the WWID (e.g., `360000000000000001`). Enable friendly names for human-readable paths like `/dev/mapper/mpath0`:

```
# /etc/multipath.conf
defaults {
    user_friendly_names yes
}
```

> If multipath output does not show all LUNs, run `lsscsi` — it shows all LUNs including those missing from multipath output.

## Aliases

Aliases are configured only in `/etc/multipath.conf` (not in `/etc/multipath/bindings`). All WWIDs including aliases are tracked in `/etc/multipath/wwids`.

```bash
# Add aliases procedure
multipath -F
systemctl stop multipathd          # or: /etc/init.d/multipathd stop (RHEL 6)
vi /etc/multipath.conf             # add aliases in multipaths section
systemctl start multipathd         # or: /etc/init.d/multipathd start (RHEL 6)
```

## Install on RHEL 6 (Legacy)

```bash
yum install device-mapper-multipath
chkconfig multipathd on
/etc/init.d/multipathd start

# Enable config and daemon
mpathconf --enable --with_multipathd y
```

## Additional Commands

```bash
# Print more information (verbosity level 4)
multipath -v4 -ll

# List all native SCSI devices under multipath
multipath -ll | grep sd | cut -d: -f4 | awk '{print $2}'

# Get running config from multipathd
multipathd -k'show config'

# Rescan a specific disk (after storage-side expansion)
echo 1 > /sys/block/sdX/device/rescan

# Flush block device buffers (important for ASM devices)
blockdev --flushbufs /dev/sdX
```

## Resize a Multipath LUN (Full Procedure)

```bash
# 1. Identify paths
multipath -ll mpathd

# 2. Rescan each underlying path
echo 1 > /sys/block/sdh/device/rescan
echo 1 > /sys/block/sdq/device/rescan
echo 1 > /sys/block/sdd/device/rescan
echo 1 > /sys/block/sdm/device/rescan

# 3. Resize the multipath map
multipathd -k"resize map mpathd"

# 4. Resize LVM (if applicable)
pvresize /dev/mapper/mpathd
lvextend -l +100%FREE /dev/mapper/vg-lv
xfs_growfs /mountpoint       # or: resize2fs /dev/mapper/vg-lv
```

## Remove a Path to Storage

```bash
# 1. Set path offline
echo offline > /sys/block/sdX/device/state

# 2. Delete the path
echo 1 > /sys/block/sdX/device/delete
```

## Remove a Storage Device Completely

```bash
# 1. Unmount
umount /path/to/volume

# 2. Remove from LVM
vgreduce <vg> /dev/mapper/mpathX
pvremove /dev/mapper/mpathX

# 3. Verify no processes using the paths
multipath -ll mpathX
lsof /dev/sdh
lsof /dev/sdm

# 4. Flush and remove the multipath device
multipath -f mpathX    # or: multipath -f <WWID>

# 5. Flush buffers on each path (especially for ASM)
blockdev --flushbufs /dev/sdh
blockdev --flushbufs /dev/sdm

# 6. Delete SCSI paths
echo 1 > /sys/block/sdh/device/delete
echo 1 > /sys/block/sdm/device/delete
```

## Fix "mpathX in use" Error

This happens when logical volumes were not removed before removing the paths:

```bash
# Remove the LV device-mapper entry first
dmsetup remove volumegroup-lvname

# Then remove the multipath map
dmsetup remove /dev/mapper/mpathX

# Now flush works
multipath -f mpathX
```

## Remove Stale Devices

```bash
# Remove missing PVs from VG
vgreduce --removemissing <VG>

# Remove stale device-mapper node
dmsetup remove /dev/<VG>/<LV>

# Delete stale SCSI device
echo 1 > /sys/block/<disk>/device/delete
```

## Consistent Device Names in a Cluster

### Method 1: Synchronize user_friendly_names

1. Set up all multipath devices on **one machine**
2. On all other machines, stop and flush:
   ```bash
   systemctl stop multipathd
   multipath -F
   ```
3. Copy `/etc/multipath/bindings` from the first machine to all others
4. Start multipathd on all machines:
   ```bash
   systemctl start multipathd
   ```

### Method 2: Synchronize Aliases

1. Configure aliases in `/etc/multipath.conf` on **one machine**
2. On all other machines, stop and flush:
   ```bash
   systemctl stop multipathd
   multipath -F
   ```
3. Copy `/etc/multipath.conf` from the first machine to all others
4. Start multipathd on all machines:
   ```bash
   systemctl start multipathd
   ```

> Without synchronization, each node may assign different friendly names to the same WWID.

## Additional Important Files

| File | Purpose |
|------|---------|
| `/etc/multipath/bindings` | WWID ↔ friendly name mappings (RHEL 6+) |
| `/var/lib/multipath/bindings` | Bindings location (RHEL 4-5) |
| `/usr/share/doc/device-mapper-multipath-*/multipath.conf.defaults` | Complete list of default configuration values |
| `/usr/share/doc/device-mapper-multipath-*/multipath.conf.annotated` | Configuration options with descriptions |

## Understanding multipath -ll Output

```
multipath device:  alias (wwid) dm-N VENDOR,PRODUCT
                   [size][features][hwhandler][rw]
path group:        \_ scheduling_policy [priority][status]
path:              \_ host:channel:id:lun devnode major:minor [path_status][dm_status]
```

Example:

```
mpath0 (20017380006c00034) dm-5 IBM,2810XIV
[size=32G][features=0][hwhandler=0][rw]
\_ round-robin 0 [prio=1][active]
 \_ 0:0:0:1  sda  8:0   [active][ready]
\_ round-robin 0 [prio=1][enabled]
 \_ 1:0:0:1  sdb  8:16  [active][ready]
```

## Multipath Command Verbosity Levels

| Flag | Output |
|------|--------|
| `multipath -v0` | No output |
| `multipath -v1` | Created or updated multipath names only |
| `multipath -v2` | All detected paths, multipaths, and maps |
| `multipath -v3` | All detected paths plus detailed info |
| `multipath -v4` | Maximum verbosity |

## Force Path Group Policy

```bash
# Force maps to a specific policy
multipath -p failover
multipath -p multibus
multipath -p group_by_serial
multipath -p group_by_prio
multipath -p group_by_node_name
```

## Change Queuing Policy at Runtime

If `queue_if_no_path` is set and all paths are down, I/O hangs indefinitely. Change the policy without editing the config:

```bash
# Switch from queue_if_no_path to fail_if_no_path (at runtime)
dmsetup message mpath0 0 "fail_if_no_path"

# Restore queuing
dmsetup message mpath0 0 "queue_if_no_path"
```

> **Tip:** To avoid I/O hangs, use `no_path_retry N` (retry N times then fail) instead of `queue_if_no_path` (queue forever).

## multipathd Interactive Console (Extended)

```bash
multipathd -k

# Full command list:
list|show paths
list|show maps|multipaths
list|show maps|multipaths status
list|show maps|multipaths stats
list|show maps|multipaths topology
list|show topology
list|show map|multipath $map topology
list|show config
list|show blacklist
list|show devices
add path $path
remove|del path $path
add map|multipath $map
remove|del map|multipath $map
switch|switchgroup map|multipath $map group $group
reconfigure
suspend map|multipath $map
resume map|multipath $map
reinstate path $path
fail path $path
disablequeueing map|multipath $map
restorequeueing map|multipath $map
disablequeueing maps|multipaths
restorequeueing maps|multipaths
resize map|multipath $map
quit
```

## Reload Configuration (After Editing multipath.conf)

```bash
# Method 1: Reload service
systemctl reload multipathd        # or: service multipathd reload

# Method 2: Flush and recreate
multipath -F
multipath -v2

# Method 3: Dry-run first, then apply
multipath -d     # dry-run — verify
multipath -v2    # apply
```

## HP Array Support (Non-Default)

```
devices {
    device {
        vendor                  "HP"
        product                 "OPEN-V."
        getuid_callout          "/sbin/scsi_id -g -u -p0x80 -s /block/%n"
    }
}
```

## udev Rules and Large LUN Counts

With many LUNs, udev triggers `multipath` on every block device add, slowing discovery. To fix, comment out this line in `/etc/udev/rules.d/40-multipath.rules`:

```
# Comment out or remove this line:
# KERNEL!="dm-[0-9]*", ACTION=="add", PROGRAM=="/bin/bash -c '/sbin/lsmod | /bin/grep ^dm_multipath'", RUN+="/sbin/multipath -v0 %M:%m"
```

The `multipathd` daemon will still create devices automatically — this only disables redundant udev-triggered calls.

## Multipath and DR (Disaster Recovery)

When failing over to a DR SAN with different WWIDs, three files must be updated:

1. `/etc/multipath.conf` — update WWIDs to DR values
2. `/var/lib/multipath/bindings` — update WWID-to-alias mappings
3. `/boot/initrd` (initramfs) — contains embedded copies of both files

### Update initrd for DR

```bash
# Backup
cp /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak

# Method 1: Rebuild initramfs (RHEL 7+)
dracut --force

# Method 2: Manual (RHEL 5/6 — older initrd format)
mkdir /tmp/initrd && cd /tmp/initrd
cp /boot/initrd-$(uname -r).img initrd.img.gz
gunzip initrd.img.gz
cpio -id < initrd.img
rm initrd.img

# Edit etc/multipath.conf and var/lib/multipath/bindings with DR WWIDs

# Repack
find . | cpio --quiet -H newc -o | gzip -9 -n > /boot/initrd-$(uname -r).img

# Reboot
reboot
```

### Cutback to LIVE

Restore the backed-up files and rebuild initramfs again with the original LIVE WWIDs.

## dmsetup Commands for Multipath

```bash
# List all device-mapper devices (major:minor)
dmsetup ls

# Show device-mapper table (targets)
dmsetup table

# Show device info
dmsetup info

# List only multipath targets
dmsetup ls --target multipath

# Show status of a multipath device
dmsetup status /dev/mapper/mpath0

# Remove a device-mapper device
dmsetup remove /dev/mapper/mpath0

# Change queuing at runtime
dmsetup message mpath0 0 "fail_if_no_path"
```
