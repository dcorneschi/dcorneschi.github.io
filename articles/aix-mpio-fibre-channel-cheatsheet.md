# AIX MPIO and Fibre Channel Cheatsheet

Command reference for storage multipathing on IBM AIX — inspecting Fibre Channel adapters (`lsattr`, `fcstat`, `lsdev`), listing and managing MPIO paths (`lspath`, `chpath`, `rmpath`), tuning disk attributes (queue depth, health check, reserve policy, load-balancing algorithm), VSCSI path priority on VIO clients, and bulk cleanup of missing/failed paths.

> Many of these commands change how disks are reached and require `root`. Disabling or removing the wrong path — or setting `reserve_policy`/`algorithm` on a disk in use — can cause I/O errors or data access loss. Confirm the path layout with `lspath` first, work one adapter at a time, and note that `-P` changes only take effect after a reboot.

## Fibre Channel Adapters

```sh
# How the adapter is connected
#   attach = none|switch ; al = no cable (FC) or direct connect
lsattr -El fscsi0

# Is the link up or down?
fcstat -D fcs0 | grep Attention
fcstat fcs0 | grep running

# List FC adapters
lsdev -Cc adapter | grep fcs

# Detailed disk info (type, serial numbers) via EMC inq
/usr/lpp/EMC/Symmetrix/bin/inq.aix64_51

# Detailed HBA info
/usr/lpp/EMC/Symmetrix/bin/inq.aix64_51 -hba
```

## Listing MPIO Paths (lspath)

```sh
# List paths in the Missing state
lspath -s Missing

# Status of the paths and the parent device/adapter for a disk
lspath -Hl hdisk0

# Map MPIO paths to their VSCSI parent adapters
lspath -F "name path_id parent connection status"

# Detailed path status with a chosen field order
lspath -H -l hdisk127 -F "status name parent path_id connection"
```

## Managing Paths (chpath / rmpath)

```sh
# Enable / disable a path
chpath -l hdisk0 -p vscsi0 -s enable
chpath -l hdisk0 -p vscsi0 -s disable

# Change the path priority of an MPIO device on a VIO client
chpath -l hdisk0 -p vscsi1 -a priority=25

# Remove a specific path
rmpath -l hdisk2 -p vscsi1 -d

# Remove ALL paths from all disks attached to an adapter
rmpath -p fscsi2 -d

# Remove a path identified by its connection (WWPN,LUN)
rmpath -dl hdisk2 -p fscsi0 -w 500009720831ed5c,7a000000000000
```

## VSCSI Adapter Tuning

```sh
# Set VSCSI error recovery to fast_fail (persist with -P)
chdev -l vscsi0 -a vscsi_err_recov=fast_fail -P

# Set the VSCSI path timeout (seconds)
chdev -l vscsi0 -a vscsi_path_to=30 -P
```

## Disk Attribute Tuning

```sh
# Queue depth (persist with -P)
chdev -l hdisk0 -a queue_depth=32 -P

# Health-check interval in seconds
chdev -l hdisk0 -a hcheck_interval=60 -P

# No SCSI reserve + round-robin load balancing (takes effect immediately)
chdev -l hdisk0 -a reserve_policy=no_reserve -a algorithm=round_robin

# Same, deferred until the next reboot
chdev -l hdisk0 -a reserve_policy=no_reserve -a algorithm=round_robin -P
```

| Attribute | Purpose |
|-----------|---------|
| `queue_depth` | Number of outstanding I/O requests the disk will queue |
| `hcheck_interval` | How often (seconds) MPIO health-checks the paths |
| `reserve_policy` | SCSI reservation: `no_reserve` needed for shared/round-robin access |
| `algorithm` | Path selection: `fail_over` (default) or `round_robin` |
| `vscsi_err_recov` | `fast_fail` fails I/O quickly when a VSCSI path is lost |
| `vscsi_path_to` | Timeout before a VSCSI path is considered failed |

## VSCSI Path Priority (VIO Client)

By default a Virtual I/O Client (VIOC) uses the **first** VIOS for all VSCSI traffic. Confirm which VSCSI adapter is active by running `nmon` on the client and pressing **`a`** — it shows the VSCSI adapter currently in use.

```sh
# Show a disk's attributes on a given path
lspath -AE -l hdisk2 -p vscsi2

# Set a path priority, then verify
chpath -l hdisk0 -p vscsi0 -a priority=2
lspath -AHE -l hdisk0 -p vscsi0
```

Lower priority values are preferred; set different priorities per VSCSI adapter to balance load across two VIOSes.

## Cleaning Up Missing / Failed Paths

### Summary and quick fixes

```sh
# Count paths grouped by disk and status
lspath | awk '{print $1,$NF}' | sort | uniq -c

# Re-enable every Failed path
lspath | grep Failed | awk '{print "chpath -l "$2" -s enable -p "$3}' | ksh

# Remove every Defined/Missing path (built from the connection field)
lspath -F "name:connection:parent:path_status:status" \
  | egrep "Defined|Missing" \
  | awk -F : '{print "rmpath -l "$1" -p "$3" -w "$2" -d"}' | ksh
```

### Remove a single missing path

```sh
# Find the status of all paths
lspath -F name,connection,path_status,parent

# Remove the missing path by its connection
rmpath -d -l hdisk1 -p vscsi0 -w 820000000000
```

### Remove failed MPIO paths across matching disks

```sh
# Example: all IBM 2107 (DS8000) disks
for disk in `lsdev -Cc disk | grep 2107 | awk '{ print $1 }'`
do
    for path in `lspath -l $disk -F "status connection" | grep Failed | awk '{ print $2 }'`
    do
        echo $disk
        rmpath -l $disk -w $path -d
    done
done
```

### More drastic: reconfigure an adapter

When paths are stuck missing, removing and re-detecting the adapter can clear them. Do this **one adapter at a time**:

```sh
rmdev -dl fsc3 -R
cfgmgr
```

## Quick Reference

| Task | Command |
|------|---------|
| Adapter connection type | `lsattr -El fscsi0` |
| Link up/down | `fcstat fcs0 \| grep running` |
| List FC adapters | `lsdev -Cc adapter \| grep fcs` |
| List missing paths | `lspath -s Missing` |
| Paths for a disk | `lspath -Hl hdisk0` |
| Enable a path | `chpath -l hdisk0 -p vscsi0 -s enable` |
| Disable a path | `chpath -l hdisk0 -p vscsi0 -s disable` |
| Set path priority | `chpath -l hdisk0 -p vscsi1 -a priority=25` |
| Remove a path | `rmpath -l hdisk2 -p vscsi1 -d` |
| Remove all paths on an adapter | `rmpath -p fscsi2 -d` |
| Queue depth | `chdev -l hdisk0 -a queue_depth=32 -P` |
| Health-check interval | `chdev -l hdisk0 -a hcheck_interval=60 -P` |
| Round-robin, no reserve | `chdev -l hdisk0 -a reserve_policy=no_reserve -a algorithm=round_robin` |
| VSCSI fast_fail | `chdev -l vscsi0 -a vscsi_err_recov=fast_fail -P` |
| Reconfigure adapter | `rmdev -dl <fscsi> -R; cfgmgr` |

## Related

- [AIX Storage Provisioning Tasks](articles/aix-storage-provisioning-tasks.md) — adding LUNs and building VGs/filesystems on top of these paths.
- [AIX Performance Monitoring Cheatsheet](articles/aix-performance-monitoring-cheatsheet.md) — `iostat`/`sar -d` disk metrics that reflect `queue_depth` and path tuning.
- [AIX LVM Cheatsheet](articles/aix-lvm-cheatsheet.md) — the VG/LV/PV layer that sits above MPIO disks.
