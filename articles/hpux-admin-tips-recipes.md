# HP-UX Administration Tips and Recipes

A grab-bag of practical HP-UX recipes: tracking down an unlinked open file that's silently filling a filesystem, setting up shell history and prompts, recovering the root password from single-user mode, renaming and re-importing volume groups, bulk-renaming logical volumes, adding LUNs/external disks, extending filesystems online, and changing VG limits with `vgmodify`. See also [HP-UX LVM](articles/hpux-lvm.md) and [HP-UX Filesystem Management](articles/hpux-filesystem-management.md).

## Finding an Unlinked Open File

A common "disk is full but I can't find the file" scenario: a program (often a logger) has a file open, then someone `rm`s the file. Deleting only removes the directory entry (unlinks the inode); the process keeps its open file handle and keeps writing, so the space is consumed but no visible file accounts for it. The space isn't reclaimed until the process closes the handle (or exits).

### Why `bdf` and `du` Disagree

This is the tell-tale symptom: `bdf` reports a filesystem is 95% full, but `du -sk` on that mount point adds up to far less. `du` walks the directory tree and can only count files that still have a name (a directory entry). An unlinked-but-open file has no directory entry, so `du` cannot see it, yet its blocks are still allocated on disk and counted by `bdf` (which reads the filesystem's free-block accounting directly). Whenever `bdf` and `du` disagree by a large margin, suspect an unlinked open file (or a file hidden underneath a mount point — a file written to a directory *before* something was mounted over it).

Find open files whose link count has dropped to zero:

```bash
lsof +L /home        # list all open files; look for a 0 in the NLINK column
lsof -a +L1 /home    # only files with a link count below 1 (i.e. unlinked)
```

`lsof` is not part of the base OS; it ships as a contributed/porting-centre package and typically installs under `/usr/local/bin` or `/opt/`. If `lsof` isn't available, `fuser` can identify the processes holding a filesystem busy, and you can inspect a process's open file descriptors by hand:

```bash
fuser -cu /home                        # PIDs (and owners) with files open on /home
ls -l /proc/<pid>/fd 2>/dev/null       # not populated on all 11i releases; lsof is more reliable
```

### Reclaiming the Space

You do **not** have to kill the process to get the space back — that risks losing data or disrupting a service. The blocks are freed the moment the last open handle to the inode is closed, so the safest options in order of preference are:

1. Ask the application to reopen its log (many daemons re-open on `SIGHUP`, or via a `logrotate`-style `copytruncate`).
2. Truncate the file *through its still-open descriptor* rather than deleting it. If you catch it before deletion, `> /path/logfile` (redirecting nothing into it) truncates in place and instantly frees the space while keeping the inode valid for the writer.
3. As a last resort, stop and restart the process that holds the handle.

The lesson for the future: truncate log files (`: > file` or `cp /dev/null file`), never `rm` a file that a running process is actively writing.

## Enabling Command History

The default HP-UX login shell is POSIX `sh` (`/usr/bin/sh`), which supports `vi`-style command-line editing and a persistent history file — but only if `HISTFILE` is set. Without it, history lives only in memory for the current session and is lost at logout, and the up-arrow/`ESC k` recall many admins expect simply doesn't work across logins.

```bash
vi .profile
```

Add:

```bash
HISTFILE=/.sh_history
export HISTFILE
```

Then create the file:

```bash
touch .sh_history
```

A more complete history setup also sizes the history and turns on `vi` editing so you can recall and edit past commands (press `ESC` to enter command mode, then `k`/`j` to walk back/forward, `/text` to search):

```bash
HISTFILE=$HOME/.sh_history;  export HISTFILE
HISTSIZE=1000;               export HISTSIZE
EDITOR=vi;                   export EDITOR
set -o vi
```

A few gotchas worth knowing:

- History is written when the shell exits *cleanly*. A session killed with `kill -9`, a dropped network connection, or a panic may not flush recent commands.
- Each interactive shell shares the one `HISTFILE`, so parallel sessions interleave their entries — the file is not per-terminal.
- Setting `HISTFILE` to a path on a full or read-only filesystem silently disables history; if history "stops working," check that the file is writable and the filesystem has space.

## Customizing the Prompt (PS1)

```bash
# user@host:/current/dir>
PS1=${LOGNAME}@$(hostname):'$PWD\> '
```

A fuller root `.profile` snippet:

```bash
HISTFILE=~/.sh_history;                 export HISTFILE
EDITOR=vi;                              export EDITOR
PS1="`whoami`@`hostname`"'[${PWD}] > '; export PS1
```

## Recovering the root Password

If the root password is lost, the fix is to interrupt the boot, load the kernel in single-user mode (which comes up as root on the console *without* prompting for a password), and reset it. This works because single-user mode is intended for exactly this kind of recovery — which is also why physical/console access to an HP-UX server must be treated as equivalent to root access, and why consoles live behind the [Management Processor](articles/hpux-management-processor.md) and locked racks.

The interaction differs by hardware generation. On PA-RISC systems you interrupt the boot to reach **ISL** (the Initial System Loader) and hand `hpux` the `-iS` flag; on Integrity (Itanium) systems you interrupt at the **EFI Boot Manager** / BCH and pass the same option to `hpux.efi` through the boot option or the `HPUX>` loader prompt. (See [HP-UX Boot Process](articles/hpux-boot-process.md) for the full boot-interaction details on each platform.)

PA-RISC via ISL:

```
>boot
Interact with IPL ? Y
ISL> hpux -iS
# once at the single-user shell:
passwd root
```

The `-iS` (equivalently `-is`) tells the kernel to enter run level **S** (single-user). At the resulting shell:

```bash
passwd root        # set a new password interactively
sync               # flush the change to disk before rebooting
shutdown -r now    # or: reboot
```

### Gotchas

- The root filesystem is usually mounted **read-only** in early single-user mode. If `passwd` fails to write, remount it read-write first: `mount -o remount / ` (or on older releases, `mount -o rw /`), then retry.
- On a **trusted** (converted) system the encrypted password lives in `/tcb/files/auth/`, not `/etc/passwd`. `passwd root` still does the right thing, but be aware of where the change lands.
- Always `sync` (or shut down cleanly) after changing the password so the write is committed; pulling power from a single-user shell can lose the change.
- This procedure only recovers the password when you have console access. It is not a way around it remotely — which is the point.

## Renaming a Volume Group

There is no `vgrename` in classic LVM. A volume group's identity is the directory under `/dev` (e.g. `/dev/vg01`) plus its `group` device file, and the kernel tracks membership by the disks' embedded VGID and the `/etc/lvmtab` registry — not by the human-readable name. So "renaming" is really: export the VG's layout to a **map file**, remove it from `/etc/lvmtab`, build a fresh `/dev/<newname>` directory and `group` node, then import the same physical volumes under the new name using the map.

The map file (`-m`) is what preserves the *logical volume names*. Export/import with `-s` (share/scan by VGID) but without a map file will still find the data and the LVs, but they come back as the default `lvol1`, `lvol2`, … — anything referencing them by name (fstab entries, application configs) then breaks. Always capture the map file.

```bash
bdf | grep vg01
umount /fs                                              # unmount the filesystem
vgcfgbackup /dev/vg01                                   # back up the VG config
vgchange -a n /dev/vg01                                 # deactivate the VG
vgexport -v -s -m /etc/lvmconf/vg02.map /dev/vg01       # export config to a map file (-p previews without touching lvmtab)

mkdir /dev/vg02                                         # create the new VG directory
ls -l /dev/*/group                                      # find an unused minor number
mknod /dev/vg02/group c 64 0x1a0000                     # create the new group device

vgimport -v -s -m /etc/lvmconf/vg01.map /dev/vg02       # import under the new name (-N for persistent DSF naming)

vgchange -a y /dev/vg02                                 # activate
mount /fs                                               # remount
```

A few important details:

- **The minor number in `mknod` must be unique.** The `group` device is a character special with major **64**; the minor encodes the VG number as `0xNN0000`. The `ls -l /dev/*/group` step exists precisely so you pick an `NN` not already in use — reusing one corrupts LVM's view of your VGs. In the example above `0x1a0000` means VG number `0x1a` (26 decimal).
- **On 11i v3, prefer `-N` (agile/persistent DSF naming)** so the imported VG references the persistent `/dev/disk/diskN` device files rather than legacy `/dev/dsk/cXtYdZ` hardware paths, which is important on multipathed SAN storage.
- After importing, confirm the result before trusting it: `vgdisplay -v /dev/vg02` should list the same LVs and sizes, and `strings /etc/lvmtab` should show the new VG. Update `/etc/fstab` to reference the new VG name before rebooting.
- Keep the map file. If anything goes wrong you can re-import the original disks under the original name from the same map.

## Bulk-Renaming Logical Volumes

Rename LVs containing `old` to the same name with `new`, across volume groups `vg11`–`vg17`:

```bash
for vg in 1 2 3 4 5 6 7
do
   cd /dev/vg1${vg}
   for lv in *old*
   do
      p=`echo $lv | awk -Fold '{print $1}'`
      s=`echo $lv | awk -Fold '{print $2}'`
      echo "Renaming /dev/vg1${vg}/$lv -> /dev/vg1${vg}/${p}new${s}"
      mv $lv ${p}new${s}
   done
done
```

## Adding a LUN / External Disk

When storage is presented to the host (a new internal disk, or a LUN zoned/mapped from a SAN array), the OS doesn't see it until the I/O subsystem is rescanned and device files are created. The workflow is: rescan hardware (`ioscan`), create device special files (`insf`), verify the disk responds (`diskinfo`), then bring it into LVM (`pvcreate` + `vgextend`) and finally grow the LV and filesystem on top. See [HP-UX Device Management (ioscan)](articles/hpux-device-management-ioscan.md) and [HP-UX Fibre Channel and SAN](articles/hpux-fibre-channel-san.md) for the discovery details on multipathed storage.

Snapshot the I/O tree before and after so you can see exactly what appeared:

```bash
ioscan -fnCdisk > /tmp/ioscan1.txt
insf -eCdisk                                            # create device files for new hardware
ioscan -fnCdisk > /tmp/ioscan2.txt
diff /tmp/ioscan1.txt /tmp/ioscan2.txt
ioscan -fnC disk

diskinfo /dev/rdsk/c33t1d3                              # verify the new disk
vgdisplay -v vg01

pvcreate -f /dev/rdsk/c0t4d0                            # initialize as a physical volume
vgextend vg01 /dev/dsk/c0t4d0                           # add it to the VG

lvextend -L 11776 /dev/vg01/lvol3                       # grow the LV (MB)
fsadm -F vxfs -b 11776M /oradata                        # grow the VxFS filesystem
```

## Adding a New LUN as a Fresh VG

```bash
pvcreate /dev/rdsk/cxtxdx                               # create the PV

mkdir /dev/vg_ggs                                       # create the VG
mknod /dev/vg_ggs/group c 64 0x050000
vgcreate /dev/vg_ggs /dev/dsk/cxtxdx                    # (add -s <MB> to set a non-default extent size)

lvcreate -L 128 -n lvggsdatadirchk /dev/vg_ggs          # create the LV
newfs -F vxfs /dev/vg_ggs/rlvggsdatadirchk              # create the filesystem
mkdir -p /ggsdata/dirchk                                # mount point

vi /etc/fstab                                           # add the fstab entry:
# /dev/vg_ggs/lvggsdatadirchk /ggsdata/dirchk vxfs rw,suid,delaylog 0 2
```

## Extending a Filesystem Online

Growing a mounted VxFS filesystem is a two-layer operation and **order matters**: you must make the *logical volume* bigger first (`lvextend`), then tell the *filesystem* to use the new space (`fsadm -b`). If you grow the filesystem first it has nowhere to expand into; if you only grow the LV, `bdf` still shows the old size because the filesystem hasn't been told about the extra extents.

Online grow (no unmount) requires the **HP OnlineJFS** (VxFS Advanced/Full) license. With base JFS you can still extend, but `fsadm` must be run against an unmounted filesystem. If `fsadm -b` fails on a mounted filesystem with a licensing error, that's the missing OnlineJFS entitlement, not a syntax problem.

```bash
pvcreate /dev/disk/rdiskxxx
vgextend /dev/vgorafs1 /dev/disk/diskxxx
lvextend -L 61440M /dev/vgorafs1/lvopt_oraprod
fsadm -F vxfs -b 61440M /opt/app/oracle/product
bdf /opt/app/oracle/product
```

Notes on the sizes and flags:

- `lvextend -L` takes a size in **MB** by default (append nothing) — here `61440M` = 60 GB. You can instead use `-l` to specify a number of extents. The LV size after extend must be **at least** the filesystem size you pass to `fsadm`.
- `fsadm -b` sets the *new total size* of the filesystem, not an increment. Passing a value smaller than the current size attempts a shrink (VxFS can shrink online too, but it's slower and riskier — take a backup first).
- To extend a LV onto a *specific* disk (for predictable placement), name the PV: `lvextend -L 61440M /dev/vgorafs1/lvopt_oraprod /dev/disk/diskxxx`.
- Always confirm with `bdf` (or `df -k`) afterward; if the number didn't change, the `fsadm` step didn't take.

## Changing VG Limits with vgmodify

Classic LVM Version 1.0 volume groups are created with fixed ceilings baked in at `vgcreate` time: **Max PV** (how many physical volumes the VG can hold), **Max LV** (how many logical volumes), and **Max PE per PV** (which, together with the extent size, caps how large each disk in the VG can be). These limits reserve space in the VG's on-disk metadata, so they can't just be edited live. Historically the only way to raise them was to back up, `vgexport`, recreate the VG with bigger limits, `vgimport`, and restore — an outage. `vgmodify` (11i v2/v3) changes several of these limits in place, without recreating the VG, provided the VG is deactivated first.

`vgmodify` is also the tool for the common "my new SAN LUN is bigger than Max PE per PV allows, so the extra space is invisible" problem: raising Max PE per PV lets LVM address the full size of the larger disk. (LVM 2.x volume groups largely remove these static limits, but many production systems still run 1.0 VGs.)

Both changes require the VG to be deactivated first. It's good practice to `vgcfgbackup` before applying, so you can restore the previous metadata if the change doesn't fit:

```bash
vgchange -a n vg01
vgcfgbackup vg01               # safety: back up current metadata first
vgmodify -p 255 -n vg01        # apply the new Max PV limit
vgchange -a y vg01
```

Change **Max PV** (maximum physical volumes per VG):

```bash
vgchange -a n vg01
vgmodify -p 255 -n vg01
vgchange -a y vg01
```

Change **Max PE per PV** (maximum physical extents per physical volume):

```bash
vgchange -a n vg01
vgmodify -e 8128 -n vg01
vgchange -a y vg01
```

## Command Reference

| Task | Command |
|------|---------|
| Find unlinked open files | `lsof -a +L1 /home` |
| Enable shell history | `HISTFILE=/.sh_history; export HISTFILE` |
| Recover root password | `ISL> hpux -iS` then `passwd root` |
| Rename a VG | `vgexport -s -m <map> <vg>` + `vgimport -s -m <map> <newvg>` |
| Add disk to VG | `pvcreate -f <rdsk>` + `vgextend <vg> <dsk>` |
| Extend LV | `lvextend -L <size> <lv>` |
| Grow VxFS online | `fsadm -F vxfs -b <size> <mountpoint>` |
| Change Max PV | `vgmodify -p <n> -n <vg>` (VG deactivated) |
| Change Max PE per PV | `vgmodify -e <n> -n <vg>` (VG deactivated) |

## Troubleshooting Quick Reference

| Symptom | Likely cause | First thing to check |
|---------|--------------|----------------------|
| `bdf` shows full, `du` shows plenty free | Unlinked open file, or file hidden under a mount | `lsof -a +L1 <fs>`; check for mounts over populated dirs |
| Command history not saved between logins | `HISTFILE` unset or file not writable | `echo $HISTFILE`; verify the file exists and the FS isn't full/read-only |
| Locked out — root password lost | — | Boot single-user (`ISL> hpux -iS` / EFI equivalent), `passwd root`, `sync` |
| `passwd` fails in single-user mode | Root filesystem mounted read-only | `mount -o remount /` then retry |
| LVs came back as `lvol1`, `lvol2`… after VG rename | Imported without the `-m` map file | Re-import with the saved map file to restore names |
| New SAN LUN not visible | I/O tree not rescanned / no device files | `ioscan -fnC disk` then `insf -e` |
| `fsadm -b` fails on a mounted FS | Missing HP OnlineJFS license | Unmount and grow offline, or license OnlineJFS |
| Filesystem still old size after extend | Only the LV was grown, not the FS | Re-run `fsadm -F vxfs -b <size> <mountpoint>` |
| New large disk's extra space unusable | Max PE per PV too low for the disk size | `vgmodify -e <n> -n <vg>` (VG deactivated) |

## Related Articles

- [HP-UX LVM](articles/hpux-lvm.md)
- [HP-UX Filesystem Management](articles/hpux-filesystem-management.md)
- [HP-UX Swap Management](articles/hpux-swap-management.md)
- [HP-UX Boot Process](articles/hpux-boot-process.md)
- [HP-UX Device Management (ioscan)](articles/hpux-device-management-ioscan.md)
- [HP-UX Fibre Channel and SAN](articles/hpux-fibre-channel-san.md)
- [HP-UX Management Processor (MP / GSP / iLO)](articles/hpux-management-processor.md)
- [HP-UX User and Password Administration](articles/hpux-user-password-administration.md)
- [HP-UX Performance Monitoring and Event Management](articles/hpux-performance-monitoring.md)


