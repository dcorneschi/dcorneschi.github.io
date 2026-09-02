# HP-UX Boot Process (PA-RISC and Integrity)

Booting and shutting down HP-UX spans two hardware architectures with very different firmware: **PA-RISC** systems use the BootROM (POST/PDC/IODC) with the **BCH** (Boot Console Handler) and **ISL/IPL**, while **Integrity** (Itanium) systems use **EFI/UEFI** firmware with the EFI Boot Manager and shell. This guide covers state changes (shutdown/reboot), the Management Processor, both boot processes end to end, interacting with the firmware, and the boot-disk structures.

### Why Two Boot Architectures

HP-UX ran on Hewlett-Packard's own **PA-RISC** processors for most of its life, then transitioned to Intel **Itanium** (the "Integrity" server line) starting with 11i v2. Because the two processor families use completely different firmware standards, the boot chain looks different on each — but the *goal* is identical: bring the CPU out of reset, test hardware, locate a boot device, load a small OS-loader utility from that device, and hand control to the HP-UX kernel (`/stand/vmunix`). Understanding the two firmware environments matters because the same recovery task (boot single-user, boot an alternate kernel, boot without LVM quorum) is spelled differently on each. Knowing which prompt you are at — `BCH`, `ISL>`, `HPUX>`, `Shell>`, or `MP>` — tells you which command set applies.

### The Management Processor (MP / GSP / iLO)

Nearly every modern HP-UX server has a dedicated service processor, variously called the **MP** (Management Processor), **GSP** (Guardian Service Processor) on older systems, or **iLO** (Integrated Lights-Out) on the newest Integrity hardware. It runs independently of the OS and stays powered as long as the cabinet has standby power, so you can reach it even when HP-UX is hung or halted. It provides the **remote console**, power control, reset, event/error logs, and (on partitionable servers) partition management. From the `MP>` main menu the two most important sub-menus are **CM** (Command Menu, for `pc`/`pe`/`rs`/`tc`/`df`) and **CO** (Console, which attaches you to the live system console). Because the MP is your lifeline for out-of-band recovery, learning its menus is as important as learning the OS boot commands.

## Changing System State

Several commands move an HP-UX system between run states.

| From state | To do this | Use |
|------------|-----------|-----|
| Multi-user | Reboot / halt / go single-user | `shutdown` |
| Single-user | Reboot or halt | `reboot` |
| Halt | Reset and boot | MP `rs` command (workstations: reset button) |
| Power-off | Power on | MP `pc` command (or the power button) |
| Hung | Reset and boot | MP `rs` or `tc` (use with care — may corrupt) |

Choosing the right transition matters: `shutdown` is the graceful path from multi-user mode because it notifies logged-in users, lets the `rc` framework stop services in order, flushes filesystem buffers, and unmounts cleanly. Skipping the graceful path (a hard reset or `tc`) risks JFS/VxFS replay on the next boot, stale LVM mirrors that must resync, and — in the worst case — data loss for applications that had dirty buffers. Reserve the MP reset commands for systems that are genuinely hung and no longer responding on the console.

### shutdown (from multi-user)

`shutdown` broadcasts a warning, waits a grace period, runs the `/sbin/rc*.d/K*` kill scripts, and unmounts non-critical filesystems. It must be run from the root (`/`) directory or it will complain and refuse; it also requires you to be root or listed in `/etc/shutdown.allow`. Internally `shutdown` calls `reboot`/`init` to make the final state change, so the two commands are layered rather than unrelated.

```bash
shutdown              # go to single-user mode
shutdown -h           # halt
shutdown -r           # reboot
shutdown -y 600       # single-user in 600s, no confirmation (y)
shutdown -hy 600      # halt in 600s
shutdown -ry 600      # reboot in 600s
shutdown -ry 0        # reboot immediately
```

### reboot (from single-user)

```bash
reboot -h             # halt the system
reboot                # reboot
```

### From the halt state

After halting with `reboot -h` or `shutdown -h`, initiate boot with the **`rs`** (reset) command on the MP Command Menu. `rs` can run on a live system too, but doing so may cause corruption. Once halted, power off safely by pressing the cabinet power switch or, on servers, the MP **`pc`** (power control) command — midrange and Superdome servers use **`pe`** (power entity) instead.

### On a hung system

If an application or OS bug hangs the system, the MP **`tc`** or **`rs`** commands reset and boot it. Use with care — they can cause corruption.

The difference between the two matters for support cases. **`rs`** is a plain reset — it restarts the partition without preserving diagnostic state. **`tc`** issues a **transfer of control**, which forces the processor to save its state and dump memory before resetting, producing a crash dump that HPE support (or `crashinfo`) can analyze to find *why* the system hung. If you need root-cause analysis of a hang, prefer `tc` so the dump is captured; if you only need the system back and don't care why it hung, `rs` is faster. Both are destructive to in-flight I/O, so treat them as a last resort after the console is confirmed unresponsive.

### MP Command Menu Reference

These commands live under the MP **CM** (Command Menu). They act out-of-band, so they work even when HP-UX itself is unresponsive.

```
MP:CM> pc      # power control: turn a partition/cabinet on or off (entry-class servers)
MP:CM> pe      # power entity: power control on midrange and Superdome complexes
MP:CM> rs      # reset a partition and boot it
MP:CM> tc      # transfer-of-control reset (captures a crash dump)
MP:CM> df      # display FRU / product and serial information
MP:CM> vfp     # virtual front panel: live boot/OS state indicators
MP:CM> sl      # show logs (event and error logs)
MP:CM> bo      # boot a partition (nPartition-capable servers)
```

To reach the live OS console from the MP main menu, use **`CO`** (Console); to return to the MP menu from an attached console, type the escape sequence **`Ctrl-B`**.

## PA-RISC Boot Process

### Major Players

- **Firmware** — POST, PDC, IODC, and PDC/BCH (in the BootROM chip on every PA-RISC SPU).
- **NVRAM** — holds primary/alternate boot paths and console path; modified via the BCH `path` command or the HP-UX `setboot` command.
- **Boot Area** — a 2 MB region in **LIF** (Logical Interchange Format) containing ISL/IPL, the AUTO file, and the HPUX utility.
- **LVM & filesystems** — the kernel in `/stand` (`/stand/vmunix`), filesystems, and swap.

### The Sequence

```
POST/PDC/IODC  →  BCH Boot Manager  →  ISL/IPL + HPUX  →  vmunix
(self-test)       (consults NVRAM     (AUTO file picks    (mounts root,
                   for boot device)    and boots kernel)   starts init)
```

1. **POST/PDC/IODC** — hardware self-tests and system initialization. **POST** (Power-On Self Test) checks CPU, memory, and core I/O. **PDC** (Processor Dependent Code) is the low-level firmware that abstracts the specific processor to the rest of the boot chain, and **IODC** (I/O Dependent Code) does the same for each I/O module so the firmware can talk to disks and cards it wasn't hard-coded to know about.
2. **BCH** — determines the boot disk and console paths (recorded in NVRAM), then loads ISL/IPL from the boot disk's LIF area. If **autoboot** is enabled the BCH proceeds automatically after a short countdown; pressing a key during that window drops you to the interactive BCH menu instead.
3. **ISL/IPL + HPUX** — the **ISL** (Initial System Loader) is a small OS-independent loader; it runs **IPL** (Initial Program Loader) which invokes the **HPUX** secondary loader utility. HPUX reads the **AUTO file** for the default boot string, then loads the kernel from `/stand`.
4. **vmunix** — the kernel scans hardware, mounts root, and starts `init`, which brings the system to multi-user mode.

The reason for this many stages is layering: each stage knows just enough to find and start the next, so firmware never has to understand LVM or a filesystem. The BCH only needs to read a raw **LIF** volume; ISL only needs to load the HPUX utility; and only the HPUX utility understands the LVM-encapsulated `/stand`. This is also why a boot can fail cleanly at a well-defined stage — a bad AUTO file stops you at ISL, a corrupt kernel stops you at HPUX, and a missing quorum stops you inside `vmunix`.

### Boot Disk Structure

- The kernel filesystem (`/stand`, holding `/stand/vmunix`) must be **HFS** — the loaders cannot read JFS/VxFS for the kernel.
- The `/` (root) filesystem holds `/etc`, `/sbin`, `/dev` (needed early) and may be HFS or JFS.
- The disk also contains a primary **swap** area enabled early in boot.
- **LVM vs whole-disk:** a non-LVM boot disk has the 2 MB boot LIF area, an HFS `/` (including `/stand`, `/sbin`, `/etc`, `/dev`), and a primary swap section.

### Interacting with the BCH

Press **Esc** during autoboot to drop to the BCH. Useful commands:

```
> help                       # view help menu
> in all                     # view system information
> search                     # list all possible boot devices
> search ipl                 # list devices containing an IPL
> path                       # display stable-storage paths
> path pri  x/x.x.x          # set the primary boot path
> path haa  x/x.x.x          # set the HA alternate path
> path alt  x/x.x.x          # set the alternate path
> boot pri                   # boot from the primary path
> boot haa                   # boot from the HA alternate path
> boot alt                   # boot from the alternate path
> boot x/x.x.x               # boot from an arbitrary device
> boot lan.x.x.x.x install   # boot from an Ignite server at that IP
> reset                      # reset the system
> auto                       # show autoboot/autosearch flags
> autoboot on|off            # set the autoboot flag
> autosearch on|off          # set the autosearch flag
> secure                     # display/set secure boot mode
```

### Viewing/Modifying NVRAM from Multi-User Mode (setboot)

```bash
setboot                    # display BCH boot paths
setboot -p x/x.x.x         # set the primary boot device
setboot -a x/x.x.x         # set the alternate boot device
setboot -h x/x.x.x         # set the high-availability (HA) boot device
```

### Interacting with the ISL/IPL

To boot a non-default kernel or mode, boot a device with the `isl` argument, then answer `yes` to interact:

```
Main Menu: Enter command or menu > boot pri isl
Main Menu: Enter command or menu > boot alt isl
Interact with IPL (Y, N, or Cancel)? > y
```

ISL/IPL / HPUX-utility commands:

```
ISL> hpux show autofile              # display the AUTO file
ISL> hpux ls                         # list contents of /stand
ISL> hpux                            # boot the default kernel
ISL> hpux -is                        # boot to single-user mode
ISL> hpux -lq                        # boot without LVM quorum
ISL> hpux -lm                        # boot to LVM maintenance mode
ISL> hpux /stand/backup/vmunix       # boot an alternate kernel
ISL> hpux /stand/vmunix nproc=6000   # boot with a modified kernel parameter
```

Backup-kernel path differs by release:

```
ISL> hpux /stand/vmunix.prev         # 11i v1
ISL> hpux /stand/backup/vmunix       # 11i v2 and v3
```

> Booting to single-user mode (`-is`) logs you in as root at the console **without a password** — this is how you recover a lost root password. **LVM maintenance mode** (`-lm`) activates no volume groups (so you can work on vg00); **single-user** has vg00 active with only `/` and `/stand` mounted.

The `-lq` (no-quorum) option deserves special mention. LVM refuses to activate a volume group unless more than half of its physical volumes (and their VGDA/VGSA copies) are present — this **quorum** rule prevents a "split-brain" where two halves of a mirrored VG diverge. But if you lose a disk in a two-disk mirrored vg00, exactly half is gone and quorum fails, so the system cannot boot normally. Booting with `-lq` tells LVM to override the quorum check and activate vg00 from the surviving disk so you can get to a running system and repair the mirror. Use it only when you know which disk is good; overriding quorum with genuinely inconsistent copies can corrupt data.

### Viewing the AUTO File / LIF Area from Multi-User Mode

The LIF area isn't readable with `ls`/`cat`. Use `lifls`/`lifcp` (a `-` destination sends a LIF file to stdout):

```bash
lifls /dev/rdisk/diska            # list a disk's boot (LIF) area
lifcp /dev/rdisk/diska:AUTO -     # print the AUTO file to stdout
```

## Integrity (Itanium) Boot Process

Integrity servers use an OS-neutral, modular firmware boot interface:

- Older Integrity servers: Intel **EFI** (Extensible Firmware Interface).
- Newer Integrity servers: **UEFI** (Unified EFI).

### Major Players

- **Firmware** — POST, PAL, SAL, the EFI Boot Manager, and the EFI shell.
- **NVRAM** — modified via the EFI shell `bcfg` command or the HP-UX `setboot` command.
- **Boot disk** — subdivided into EFI/UEFI disk partitions.

### Boot Disk Structure

HP-UX Integrity boot disks have three EFI/UEFI partitions plus supporting structures:

| Structure | Purpose | Device file (11i v1/v2 → v3) |
|-----------|---------|------------------------------|
| MBR | Legacy Intel record, ignored by EFI (aids dual-readability) | — |
| GPT | GUID Partition Table; replicated top and bottom of disk | — |
| System partition | OS loader `\efi\hpux\hpux.efi`, the `\efi\hpux\auto` boot-string file, and utilities (FAT32) | `cxtxdxs1` → `diska_p1` |
| OS partition | Encapsulated LVM boot PV (PVRA/VGRA/BDRA/BBRA + vg00 extents) | `cxtxdxs2` → `diska_p2` |
| HP Service Partition (HPSP) | Optional ~400 MB FAT32 with offline diagnostics (e-Diag) | `cxtxdxs3` → `diska_p3` |

Partitions are configured with **Ignite-UX** or the **`idisk`** utility.

```bash
idisk /dev/rdisk/diska                    # view the EFI partition table
idisk -wf /tmp/idf /dev/rdsk/c0t1d0       # create partitions from a description file
```

### Managing the EFI System Partition

EFI (FAT32) filesystems aren't accessible with normal UNIX commands; HP-UX ships `efi_ls`, `efi_cp`, `efi_rm`:

```bash
# List an EFI filesystem / a directory within it
efi_ls -d /dev/rdisk/diska_p1 /efi/hpux

# View a file (copy it out to stdout/tty with -u)
efi_cp -d /dev/rdisk/diska_p1 -u /efi/hpux/auto /dev/tty

# Edit the AUTO file: copy out, edit, copy back
efi_cp -d /dev/rdisk/diska_p1 -u /efi/hpux/auto /tmp/auto
vi /tmp/auto
efi_cp -d /dev/rdisk/diska_p1 /tmp/auto /efi/hpux/auto

# Or set the boot string directly with mkboot
mkboot -a "boot vmunix -lq" /dev/rdisk/diska_p1
```

Inspect the OS partition (an encapsulated LVM PV) with normal LVM tools:

```bash
pvdisplay /dev/disk/diska_p2
```

### Correlating HP-UX and EFI Hardware Paths

The PA-RISC boot manager uses HP-UX hardware addresses; the Integrity boot manager uses EFI/UEFI addresses. To boot an alternate device you must correlate the two:

```bash
ioscan -eN                # show HP-UX and EFI hardware paths (Integrity only)
```

### Interacting with the EFI Shell

Interrupt EFI autoboot and choose the EFI shell from the Boot Manager menu. It lists boot devices, runs `startup.nsh` from the default disk, and gives a `Shell>` prompt.

```
Shell> help -b              # general EFI shell info (paged)
Shell> help -a -b           # list all commands
Shell> help command         # help on one command
Shell> reset                # reset the system
Shell> bcfg boot dump       # show boot manager entries (bcfg edits NVRAM)
```

Common HP EFI binaries (`.efi`) on the system partition:

```
\efi\hpux\hpux.efi                     # the HP-UX kernel loader
\install.efi                           # install menu utility (DVD)
\efi\boot\launchmenu.efi               # e-Diagnostics menu utility
\efi\hp\tools\network\ifconfig.efi     # EFI ifconfig
\efi\hp\tools\network\ping.efi         # EFI ping
\efi\hp\tools\network\route.efi        # EFI route
\efi\hp\tools\network\ftp.efi          # EFI ftp
```

### Interacting with the hpux.efi Loader

From the EFI shell, launching `hpux.efi` gives the `HPUX>` loader prompt:

```
HPUX> help -d                     # help
HPUX> ll                          # list /stand
HPUX> showauto                    # show the \efi\hpux\auto boot string
HPUX> setauto "boot vmunix"       # change the auto boot string
HPUX> boot vmunix                 # boot to multi-user
HPUX> boot vmunix -is             # boot to single-user
HPUX> boot vmunix -lq             # boot without quorum check
HPUX> boot vmunix -lm             # boot to LVM maintenance mode
HPUX> boot vmunix -tm             # boot to tunable maintenance mode
HPUX> boot vmunix nproc=4000      # boot with a kernel parameter override
HPUX> exit                        # exit the loader (or: reset)
```

## When to Override Autoboot

On either architecture you may need a manual boot to:

- Re-install the OS from CD/DVD.
- Boot from a recovery tape if the primary disk is corrupt.
- Boot from an alternate disk if the primary is corrupt.
- Boot an alternate kernel if `/stand/vmunix` is corrupt.
- Boot to single-user mode for maintenance.
- Boot to single-user mode to reset a lost root password.

## Boot-Related Commands

```bash
# Mirror every logical volume in vg00 (HP-UX 11.31) onto another disk
find /dev/vg00 -type b | while read L; do
  lvextend -m 1 "$L" /dev/disk/<mirror-disk>
done

# Remove the boot area entirely from vg00
lvrmboot -r /dev/vg00

# Per-service startup status recorded at boot
cat /etc/rc.log
```

### Network / Ignite Boot (Integrity direct boot profiles)

```bash
lanaddress                        # list NIC MAC addresses
dbprofile cp test profile         # copy a direct boot profile
dbprofile rm test                 # remove a direct boot profile
lanboot -dn ignite                # boot using the named dbprofile
lanboot select -dn ignite         # pick a specific NIC to boot from
reconnect -r                      # re-connect EFI drivers (NPARs: NICs not showing)
```

## Command Reference

| Task | PA-RISC | Integrity |
|------|---------|-----------|
| Firmware interface | BCH (`Esc` to interrupt) | EFI/UEFI shell + Boot Manager |
| List boot devices | `search` | Boot Manager menu / `bcfg boot dump` |
| Set boot paths (firmware) | `path pri/alt/haa` | `bcfg` |
| Set boot device (running) | `setboot -p/-a/-h` | `setboot` |
| Single-user boot | `hpux -is` (ISL) | `boot vmunix -is` (HPUX>) |
| LVM maintenance boot | `hpux -lm` | `boot vmunix -lm` |
| Boot without quorum | `hpux -lq` | `boot vmunix -lq` |
| Alternate kernel | `hpux /stand/backup/vmunix` | `boot /stand/backup/vmunix` |
| View AUTO file | `lifcp ...:AUTO -` | `efi_cp -u .../auto /dev/tty` / `showauto` |
| Partition the boot disk | (LIF/whole-disk or LVM) | `idisk`, Ignite-UX |
| EFI file management | — | `efi_ls` / `efi_cp` / `efi_rm` |
| HP-UX ↔ EFI paths | — | `ioscan -eN` |

## Troubleshooting the Boot

| Symptom | Likely cause | Where to fix it |
|---------|--------------|-----------------|
| Stops at BCH / EFI Boot Manager, no OS | Autoboot disabled, or primary path wrong/empty | `search`/`bcfg boot dump` to find the disk, then `path`/`bcfg` or `setboot` |
| `ISL>` reports it can't find the AUTO file or kernel | Corrupt LIF/AUTO, or `/stand/vmunix` missing | Boot alternate kernel (`hpux /stand/vmunix.prev`), then rebuild boot area with `mkboot` |
| Kernel panics early with a quorum/VG error | Lost a disk in mirrored vg00, quorum not met | Boot `hpux -lq` / `boot vmunix -lq`, then repair the mirror |
| Boots to single-user but `/` is read-only | Normal at that stage | `mount -o remount` or `fsck` then continue |
| Hang with no console output at all | Firmware/hardware fault, or hung before console handoff | MP `vfp` to see boot state, `sl` to read logs, `tc` to force a dump |
| Integrity NICs missing from EFI for LAN boot | EFI drivers not connected (common on nPars) | `reconnect -r` in the EFI shell |

The general recovery strategy is to move *down* the boot chain: if the firmware can't find a device, fix the path there; if ISL/HPUX can't load a kernel, boot an alternate kernel and repair `/stand`; if the kernel starts but can't mount root, boot single-user or `-lm` and repair LVM/filesystems. Booting from an **Ignite-UX** server or a **make_net_recovery**/**make_tape_recovery** image is the fallback when the boot disk itself is unusable.

## Related Articles

- [HP-UX LVM (Logical Volume Manager)](articles/hpux-lvm.md) — boot-disk mirroring, quorum, and vg00 recovery
- [HP-UX Virtual Partitions (vPars)](articles/hpux-vpars.md) — how `vpmon` inserts itself ahead of the kernel at boot
