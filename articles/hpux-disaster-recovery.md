# HP-UX Disaster Recovery (DRD and Ignite-UX)

Ensuring an HP-UX system can boot again after a failure or disaster. Three complementary layers: **mirroring** the boot disk (protects against hardware failure), **Dynamic Root Disk (DRD)** clones (protect against admin error, bad patches, and corruption), and **Ignite-UX recovery archives** (rebuild the OS onto new hardware). Covers HP-UX 11i v1–v3.

## Why Three Layers?

No single mechanism protects against every failure mode, which is why a robust HP-UX recovery strategy stacks three independent layers. The key insight is that each layer defends against a *different class* of failure, and the classes do not overlap cleanly:

- A **mirror** protects against a physical disk dying, but it faithfully replicates whatever you write — including a bad patch, a `rm -rf`, or filesystem corruption. Both mirror halves go bad together in those cases.
- A **DRD clone** is a frozen point-in-time copy that later changes to the live disk do *not* touch, so it survives exactly the admin-error and bad-patch scenarios that defeat a mirror. But it usually lives on a disk inside the same chassis, so it does not help if the whole machine or storage array is lost.
- An **Ignite-UX archive** lives off the box (on tape or an Ignite server), so it survives loss of the entire system and can rebuild onto replacement hardware or a DR site.

Think of it as defence in depth: mirror for hardware faults (zero downtime), DRD for logical faults (fast local recovery), Ignite for catastrophe (rebuild from scratch). Skipping a layer leaves a specific gap — for example, mirroring alone gives a false sense of safety against the most common real-world cause of an unbootable system, which is human error rather than disk failure.

## The Three Layers of Boot Protection

| Approach | Protects against | Notes |
|----------|------------------|-------|
| **Boot-disk mirror** (LVM Mirrordisk/UX, VxVM, SmartArray, SAN) | Primary boot disk hardware failure | OS keeps running on the mirror; but mirrors also replicate admin errors/corruption |
| **DRD clone** | Admin errors, bad patches, corruption | Point-in-time clone of vg00; stays untouched by later changes to the active disk |
| **Ignite-UX recovery archive** | Total loss of boot disk *and* clone; disaster recovery | Bootable tape or network archive; restore to onsite or DR-site hardware |

### Mirroring Solutions

1. LVM Mirrordisk/UX
2. VxVM mirroring
3. HP SmartArray controller (hardware RAID)
4. SAN-based mirroring

A mirror keeps the system accessible if the primary boot disk fails. But admin errors, security breaches, and critical software defects can render **both** mirror halves unbootable — which is where DRD and Ignite archives come in.

## Dynamic Root Disk (DRD)

DRD creates a point-in-time **clone** of the `vg00` volume group:

- The original `vg00` stays active.
- The cloned `vg00` stays **inactive** until needed.
- Unlike a mirror, the clone is **unaffected** by later changes to `vg00` — so admin errors that corrupt both mirror halves leave the DRD clone intact.

DRD is an optional, free product on the 11i v2 and v3 application media. Installing it does **not** require a reboot.

```bash
swinstall -s /tmp/DynRootDisk*.depot DynRootDisk
swlist DynRootDisk               # confirm DRD is installed and check its version
```

DRD logs to `/var/opt/drd/drd.log`. Get in the habit of tailing this log during long clone/sync operations — it records progress, warnings, and the exact reason for any failure, which is far more useful than the terse messages printed to the terminal.

The mechanism that makes DRD work is worth understanding. `vg00` is the root volume group, and DRD copies its logical volumes block-for-block onto an idle target disk, then rewrites the clone's LVM metadata so the clone is a self-consistent, separately named volume group (`drd00`) rather than a duplicate of `vg00`. That renaming is critical: two active volume groups with the same name and IDs would confuse LVM. Because the clone is a distinct VG that is left deactivated, nothing you subsequently do to the running `vg00` — patching, editing config, deleting files — reaches the clone. It is a true point-in-time snapshot on independent media, not a live mirror.

### Requirements and Limitations

- The target must be a **whole disk** at least as large as the space in use on `vg00`; DRD will not clone onto a partition or an in-use LVM disk unless you pass `overwrite=true`.
- DRD clones **`vg00` only** — it does not clone data volume groups. Protect those separately (mirroring, backups, or their own recovery archives).
- The clone captures the OS as it was at clone time; use `drd sync` to refresh it, or re-clone.
- On 11i v3 the clone works with agile device paths (`/dev/disk/diskN`); on v2 use the legacy paths. Mixing them up is a common source of "device not found" errors.

### Why DRD Reduces Downtime

**Unplanned downtime** — without DRD, an OS misconfiguration may force a restore from tape. With DRD, just activate and boot the clone.

**Planned downtime** — installing patches, tuning the kernel, or updating the OS may need downtime. With DRD you apply those changes to the *inactive* clone without touching the running system, then reboot onto the clone during a maintenance window. If it misbehaves, reboot back onto the original disk.

### Creating a DRD Clone

```bash
ioscan -kfnNC disk               # list all disks
lvmadm -l                        # which disks are LVM disks? (11i v3)
strings /etc/lvmtab              # which disks are LVM disks? (11i v1/v2)
vxdisk list                      # which disks are VxVM disks?
diskinfo /dev/rdisk/disk3        # verify target disk size

# Clone the current active boot disk to disk3
drd clone -t /dev/disk/disk3 -x overwrite=true
drd clone -t /dev/disk/disk3 -x overwrite=false -x mirror_disk=/dev/disk/disk4
```

The `-x overwrite=true` option allows the target to be overwritten even if it already contains LVM/filesystem headers. `mirror_disk` clones onto a mirrored pair so the clone itself is protected by mirroring. A first clone of a typical root disk takes on the order of tens of minutes depending on how much data `vg00` holds and the speed of the disks; plan for it to run while the system is up (DRD does not quiesce the running OS).

A subtle but important point: because `drd clone` copies a *running* filesystem, files that are mid-write at copy time can be captured in an inconsistent state. In practice this is rarely a problem for the OS itself, but if a critical application writes into `vg00` you should quiesce it (or accept that its files reflect an arbitrary instant). This is another reason DRD targets `vg00` only — application data belongs in its own volume group with its own consistency strategy.

### Synchronizing a DRD Clone

The boot disk changes over time; `drd sync` brings the clone up to date. Note that sync does **not** copy every file — review the list first.

```bash
drd sync -p                                              # preview which files need syncing
view /var/opt/drd/sync/files_to_be_copied_by_drd_sync    # review the list
drd sync                                                 # perform the sync

# Alternative: rebuild the clone entirely
drd clone -t /dev/disk/disk3 -x overwrite=true

drd status                                               # verify clone status
drd runcmd swlist                                        # software on the inactive image
drd runcmd view /var/adm/sw/swagent.log                  # inactive image's SD-UX log
```

### DRD-Safe Commands (drd runcmd)

Files in the inactive image aren't accessible to normal HP-UX commands by default. `drd runcmd` runs a **DRD-safe** command against the inactive image: it temporarily imports and mounts the inactive VG and filesystems under `/var/opt/drd/mnts/sysimage_001/`, runs the command using the inactive image's own executables, then exports and unmounts — leaving the active image untouched.

DRD-safe commands: `swinstall`, `swremove`, `swlist`, `swmodify`, `swverify`, `swjob`, `kctune`, `update-ux`, `view`.

#### Managing software on the inactive image

```bash
drd runcmd swlist                                        # list installed software
drd runcmd swinstall -s server:/mydepot PHSS_NNNNN       # install a patch
drd runcmd swremove PHSS_NNNNN                           # remove a patch
drd runcmd view /var/adm/sw/swagent.log                  # view the SD-UX log
drd runcmd swinstall -s server:/mydepot Update-UX        # update the Update-UX tool
drd runcmd update-ux -s server:/mydepot                  # update the OS
drd runcmd view /var/adm/sw/update-ux.log
```

#### Managing kernel tunables on the inactive image

Tunable changes that would normally need a reboot can be staged on the clone; they take effect when you boot it.

```bash
drd runcmd kctune nswapdev                               # view a tunable
drd runcmd kctune nswapdev=64                            # modify a tunable
drd runcmd view /var/adm/kc.log                          # kernel change log
```

### Accessing the Inactive Image Directly

```bash
drd mount                                                # import as drd00, mount under sysimage_001
diff /etc/passwd /var/opt/drd/mnts/sysimage_001/etc/passwd   # compare — do NOT modify the active image
drd umount                                               # unmount the inactive image
```

When booted from a clone, `drd mount` mounts the *original* system's filesystems under `sysimage_000` instead of `sysimage_001`.

### Activating / Deactivating a DRD Image

`drd activate` makes the inactive image the primary boot disk (it rewrites the boot path so firmware boots the clone next, and can reboot immediately). The elegance of the DRD workflow is that activation is fully reversible up until you commit to it: you can `drd activate -p` to preview, activate without rebooting, and `drd deactivate` to change your mind. Even after booting the clone, the original disk is untouched and still bootable, so falling back is just a matter of pointing the boot path back at it and rebooting.

The typical maintenance-window pattern is: clone → `drd runcmd` the risky change onto the clone → `drd activate` → reboot onto the clone → validate. If validation fails, reboot back onto the original `vg00` (the boot menu still lists it) and you have lost only the reboot time, not the system.

```bash
drd activate -p                     # preview only
drd activate -x reboot=false        # make it primary, don't reboot yet
shutdown -ry 0                      # reboot manually if reboot=true wasn't used

drd deactivate                      # changed your mind (before reboot)
drd activate                        # after reverting to the original, re-activate + reboot
drd status                          # which disk is the active boot disk?

# Wipe a clone by zeroing the front of the disk
dd if=/dev/zero of=/dev/rdisk/disk3 bs=1048576 count=1024
```

## Ignite-UX Recovery Archives

If both the boot disk and the DRD clone fail, an Ignite recovery archive rebuilds the OS. Both commands create a **bootable** image of the boot disk:

- `make_tape_recovery` — writes the bootable archive to tape.
- `make_net_recovery` — writes the archive to a file on a local/remote Ignite-UX server.

The distinguishing feature of an Ignite recovery archive versus a DRD clone is *location*. The archive lives off the machine, so it survives events that take out the whole box — a destroyed chassis, a corrupted storage array, or a site failure. The trade-off is speed and effort: recovering from an archive means booting the Ignite install kernel, rebuilding LVM structures and filesystems from scratch, and streaming the archive back, which is far slower than simply booting a DRD clone. That is exactly why these are complementary rather than competing tools — DRD for fast local recovery, archives for true disaster recovery.

Decide up front how much of the system each archive should hold. The default (essential OS files) is small and quick and is enough to make the machine boot and be reachable, after which you restore application data from your normal backups. A full-`vg00` archive is larger but restores a more complete system in one pass. Match the choice to your recovery-time objective and to how your data backups are organized.

By default an archive includes a boot image, the root VG's LVM configuration, and the **critical** files/directories from `vg00` needed to boot. The default include list is `/opt/ignite/recovery/mnr_essentials`:

```
/sbin      /dev        /stand      /stand/vmunix
/usr/bin   /usr/ccs    /usr/conf   /usr/lbin
/usr/lib   /usr/newconfig  /usr/sbin  /usr/sam
/usr/share /usr/obam   /usr/include
/bin       /lib        /etc
```

```bash
view /opt/ignite/recovery/mnr_essentials                          # default include list (client)
view /var/opt/ignite/clients/<client>/recovery/default           # net_recovery defaults (server)
```

### Customizing Include / Exclude

Store persistent options in an `archive_content` file so every backup is consistent:

- `make_tape_recovery` reads `/var/opt/ignite/recovery/archive_content` on the client.
- `make_net_recovery` reads `/var/opt/ignite/clients/<client>/recovery/archive_content` on the server.

```bash
# Essential core OS files only
/opt/ignite/bin/make_net_recovery -s <ignite_server>

# Complete root VG
/opt/ignite/bin/make_net_recovery -Av -s <ignite_server>

# Complete root VG to a different archive location
/opt/ignite/bin/make_net_recovery -Av -a host:/archive_dir -s <ignite_server>

# Multiple whole VGs
/opt/ignite/bin/make_net_recovery -x inc_entire=vg00 -x inc_entire=vg01 -s <ignite_server>

# Interactive content selection
/opt/ignite/bin/make_net_recovery -i -s <ignite_server>
```

### Creating a Tape Recovery Archive

```bash
swlist IGNITE                                                    # confirm Ignite-UX is installed
make_tape_recovery -a /dev/rtape/tape0_BESTn -x inc_entire=vg00 -d "my client archive"
more /var/opt/ignite/recovery/latest/recovery.log               # review the log
```

### Creating a Network Recovery Archive

```bash
swlist IGNITE                                                    # client Ignite must match server version
bdf /var/opt/ignite/recovery/archives/                          # check server disk space
make_net_recovery -s servername -x inc_entire=vg00 -n 3 -d "my client archive"
cd /var/opt/ignite/clients/<client>/recovery/                   # server-side logs
```

You can turn a `make_net_recovery` image into a bootable recovery DVD with `make_media_install`:

```bash
view /opt/ignite/data/scripts/examples/make_media_install
```

Because drivers/configuration are model-specific, a `make_net_recovery` archive should generally be **restored only on the system that created it**. To clone a configuration onto *different* servers, use [`make_sys_image`](articles/hpux-installation-ignite.md) instead.

### Restoring From a Recovery Archive

To recover a non-bootable system:

1. Power on and interrupt the boot process.
2. Boot from the tape or Ignite-UX server.
3. Let Ignite rebuild the boot disk, LVM structures, and filesystems.
4. Let Ignite restore the archived files.
5. Verify the recovery.
6. Restore the most recent data backup with your backup software.

Two restore modes:

- **Non-interactive** — restores the archive's configuration and files verbatim. Normal mode for recovering a corrupted boot disk.
- **Interactive** — interrupt to tweak configuration: convert HFS→VxFS, resize the root filesystem, resize primary swap, or change network configuration.

## Configuring make_net_recovery End to End

### On the Ignite-UX server

```bash
mkdir -p /var/opt/ignite/recovery/archives/client_host
chown bin:bin /var/opt/ignite/recovery/archives/client_host
```

Export the archive directory. On 11i v1/v2 use `/etc/exports`:

```
/var/opt/ignite/clients -anon=2
/var/opt/ignite/recovery/archives/client_host -anon=2,access=client_host
```

```bash
exportfs -av
```

On 11i v3 use `/etc/dfs/dfstab`:

```
share -F nfs -o sec=sys,anon=2,rw=client /var/opt/ignite/recovery/archives/client
```

```bash
shareall -F nfs
```

Add the client to `/etc/hosts`:

```
10.10.10.2 client_host
```

Define the recovery defaults in `/var/opt/ignite/clients/<client>/recovery/default`:

```
RECOVERY_TYPE=net
RECOVERY_LOCATION=<ignite_server>:/archives/<client>
TAPE_DESTINATION=none
RECOVERY_DESCRIPTION=Recovery Archive
SAVE_NUM_ARCHIVES=1
ARCHIVE_TYPE=tar
```

### On the client

```bash
# Register a brand-new client and take the first archive
dd_new_client -s <ignite_server>
make_net_recovery -x inc_entire=vg00 -x exclude=/var/DAILY_SAVED -s <ignite_server>

# Or point at an explicit archive location
make_net_recovery -a <ignite_server>:/archives/<client>
```

### Restoring Individual Files From an Archive

```bash
gzcat <image_name> | pax -r -f - etc/hosts      # restores etc/hosts to the current directory
gzcat <image_name> | tar -xvf - etc/hosts       # tar equivalent
```

### Recovery Log Locations

On the server under `/var/opt/ignite/recovery/clients/0x{LLA}/recovery/<date,time>/`:

- `recovery.log` — progress and error log
- `flist` — tar archive contents

## Troubleshooting

- **`drd clone` fails with "device in use" or "not a whole disk".** The target is already part of an active volume group or is a partition. Verify it is truly idle (`lvmadm -l` on v3, `strings /etc/lvmtab` on v1/v2, `vxdisk list` for VxVM) and pass `-x overwrite=true` only when you are certain the disk is disposable.
- **`drd runcmd` cannot find the inactive image.** DRD imports the clone as `drd00` and mounts it under `/var/opt/drd/mnts/`. If a previous run was interrupted, the mount may be stale; run `drd umount` to clean up, then retry. Check `/var/opt/drd/drd.log` for the specific step that failed.
- **Booted the clone but it comes up as the original.** The boot path was not actually changed, or firmware fell back to the primary path. Confirm with `drd status` which image is active and re-check the boot path with `setboot`.
- **`make_net_recovery` fails to write to the server.** Almost always an NFS export or permissions problem on the archive directory. Re-check the export (`exportfs -av` on v1/v2, `shareall -F nfs` on v3), directory ownership, and that the client's version of Ignite matches the server's.
- **Recovery archive restored but the system will not boot.** The archive may have been restored onto hardware that differs from the machine that created it — `make_net_recovery`/`make_tape_recovery` are machine-specific. For cross-hardware rebuilds use `make_sys_image` (see [HP-UX Installation and Ignite-UX](articles/hpux-installation-ignite.md)). Also verify the boot LVs with `lvlnboot -v`.
- **Restore succeeds but application data is missing.** Expected if the archive held only `vg00` or the essential OS files — application data comes back from your regular data backups, not the Ignite archive.

```bash
drd status                          # which image is active, clone health
setboot                             # show/adjust primary and alternate boot paths
lvlnboot -v                         # verify boot/root/swap/dump LVs after a restore
```

## Command Reference

| Task | Command |
|------|---------|
| Install DRD | `swinstall -s DynRootDisk*.depot DynRootDisk` |
| Clone boot disk | `drd clone -t /dev/disk/disk3 -x overwrite=true` |
| Preview / run sync | `drd sync -p` / `drd sync` |
| Run cmd on inactive image | `drd runcmd swlist` |
| Patch inactive image | `drd runcmd swinstall -s <depot> PHSS_NNNNN` |
| Tune inactive kernel | `drd runcmd kctune <tunable>=<val>` |
| Mount / unmount clone | `drd mount` / `drd umount` |
| Activate / deactivate | `drd activate -x reboot=false` / `drd deactivate` |
| Clone status | `drd status` |
| Tape recovery | `make_tape_recovery -a <tape> -x inc_entire=vg00` |
| Net recovery | `make_net_recovery -s <server> -x inc_entire=vg00` |
| Register net client | `dd_new_client -s <server>` |
| Restore single file | `gzcat <image> \| pax -r -f - etc/hosts` |

## Related Articles

- [HP-UX Installation and Ignite-UX](articles/hpux-installation-ignite.md)
- [HP-UX LVM](articles/hpux-lvm.md)
- [HP-UX Boot Process](articles/hpux-boot-process.md)
- [HP-UX Patch Management](articles/hpux-patch-management.md)
- [HP-UX Kernel Configuration and Tuning](articles/hpux-kernel-configuration.md)
- [HP-UX Filesystem Management](articles/hpux-filesystem-management.md)
- [HP-UX Crash Dump Analysis](articles/hpux-crash-dump-analysis.md)
