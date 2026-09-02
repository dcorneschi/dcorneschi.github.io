# HP-UX Installation and Ignite-UX

Installing HP-UX — cold installs from Core media or an Ignite-UX network server, golden-image cloning, choosing an Operating Environment (OE) and install-time security bundle, initiating the install on PA-RISC (BCH) and Integrity (EFI), navigating the Ignite-UX menus, verifying the result, and post-install configuration. Covers HP-UX 11i v1–v3.

## Concepts: How Ignite-UX Fits Together

Ignite-UX (often shortened to **Ignite** or **IUX**) is HP's framework for installing, cloning, and recovering HP-UX systems. It is worth understanding the moving parts before touching a command, because the same tool set is used for three quite different jobs — a first-time install, mass deployment of a standard build, and disaster recovery of an existing system.

- **The Ignite server** is an HP-UX host that stores install sources (OE depots), configuration files, and per-client recovery archives, and that serves them over the network. Clients boot from it using the `boot lan` (PA-RISC) or an EFI network boot (Integrity) path.
- **A client** is any system being installed or recovered. During an Ignite session the client boots a small in-memory HP-UX kernel (the *install kernel*), lays down LVM/VxVM structures and filesystems, then pulls software from the source.
- **Configuration files** (`config`, `INDEX`, per-client `config` fragments under `/var/opt/ignite/`) drive what gets installed and how. The `instl_adm` command edits and validates the defaults; `manage_index` maintains the menu of selectable configurations.

The distinction that trips people up most: an Ignite **recovery archive** (`make_net_recovery` / `make_tape_recovery`) is model- and configuration-specific and is meant to rebuild *the same* machine, whereas a **golden image** (`make_sys_image`) is a portable prototype meant to be pushed onto *many different* machines. Restoring a recovery archive onto dissimilar hardware often produces an unbootable system because the archived kernel and I/O configuration do not match the new box.

## Cold Installs and Golden Images

A **cold install** puts HP-UX on a bare machine, pulled from the HP-UX **Core** DVDs or from an **Ignite-UX** network install server. A cold install always erases the target root disk — it is not an upgrade path. To move from one HP-UX release to another while preserving data and software, use `update-ux` (covered below) rather than a cold install.

To install many identical systems (same LVM layout, filesystems, software, patches), build a prototype and propagate a **Golden Image** — stored on an Ignite install tape or an Ignite network server. The workflow is: build and fully configure one reference (prototype) system, capture it with `make_sys_image`, register the resulting archive on the Ignite server, then install other clients from it. Because the image is captured *after* the prototype was patched and tuned, every clone starts life already patched and tuned — a large time saver over installing each machine from Core media and patching it individually.

```bash
# Build a system image (golden image) from a prototype host
cp /opt/ignite/data/scripts/make_sys_image /tmp
make_sys_image -v -u -s local -d /image
/tmp/make_sys_image -v -s local -d /mnt -f /tmp/user_exclude_files   # with an exclude list
```

Common gotchas when building a golden image:

- **Strip machine-specific identity from the prototype** before capture, or every clone will come up with the same hostname, IP address, and SSH host keys. Ignite prompts for network identity at install time, but files like `/etc/ssh/ssh_host_*` and application licence files tied to a MAC address or UUID are copied verbatim.
- **Exclude transient and large data areas** (application data, `/var/tmp`, core files, logs) with an exclude file so the image stays small and generic.
- **Keep the prototype's OS release, OE, and patch level identical** to what you want on the clones — a golden image cannot change the OS version during deployment.

## Operating Environments (OEs)

An **OE** is a collection of software products tested, licensed, installed, and managed as a unit. All OEs share a common **Base OS** (kernel, I/O, memory, LVM, core components); each adds a different set of applications. **Only one OE** can be installed at a time (you can supplement or upgrade later).

### 11i v1 / v2 OEs

- 11i Technical Computing OE (`HPUX11i-TCOE`)
- 11i Foundation OE (`HPUX11i-OE`)
- 11i Enterprise OE (`HPUX11i-OE-Ent`)
- 11i Mission Critical OE (`HPUX11i-OE-MC`)

### 11i v3 OEs (March 2008 kit)

Each 11i v3 OE has **Required** (must install), **Recommended** (default, deselectable), and **Optional** (manual) components.

- Base OE (`HPUX11i-BOE`)
- Virtual Server OE (`HPUX11i-VSE-OE`)
- High Availability OE (`HPUX11i-HA-OE`)
- Data Center OE

The reason OEs exist is to guarantee a *tested* combination. Rather than leaving each admin to assemble their own set of products and hope the versions interoperate, HP ships an OE as a single bundle whose members were qualified together. This is also why only one OE can be installed at once — mixing OEs would defeat the tested-combination guarantee. You can still add individual products on top of an OE afterward; you just cannot layer a second OE.

```bash
# Which OE bundle is installed?
swlist -l bundle "HPUX11i-*OE*"

# What software came with the OE?
swlist -l product "HPUX11i-*OE*"

# Which OEs are on the core depot?
swlist -l bundle -s /dvd "HPUX11i-*OE*"
```

### Upgrading the OE (update-ux)

`update-ux` is the supported in-place path between OEs and between minor HP-UX releases. Unlike a cold install it preserves user data, most configuration, and installed applications. The one non-negotiable rule is that you **must** update the `Update-UX` tool itself first — the version of `update-ux` on the running system may be too old to understand the layout of the newer OE depot, and running it stale is a common cause of failed or partial updates.

```bash
swinstall -s /dvd Update-UX          # first update the Update-UX tool itself
update-ux -s /dvd HPUX11i-OE-MC      # then update to the target OE
update-ux -p -s /dvd HPUX11i-BOE     # -p = preview: report what would change, install nothing
update-ux -s /dvd HPUX11i-BOE OnlineDiag   # update the OE and add extra bundles in one pass
```

Before any `update-ux` run, take a fresh recovery archive or DRD clone (see [HP-UX Disaster Recovery](articles/hpux-disaster-recovery.md)) so you can fall back if the update misbehaves. `update-ux` writes a detailed log to `/var/adm/sw/update-ux.log`; check it after every run. Common pitfalls: insufficient free space in `/`, `/var`, `/usr`, or `/stand`; a depot that does not contain the exact OE bundle name you passed; and not rebooting when the update stages kernel changes.

## Install-Time Security Bundles

During an **11i v2/v3** install/update you can pick an install-time security bundle (not available in 11i v1). Ignite installs **Bastille** and optionally runs `bastille -b` with a preconfigured profile.

| Bundle | Bastille config |
|--------|-----------------|
| `Sec00Tools` | Default-installed (tools only) |
| `Sec10Host` | `HOST.config` |
| `Sec20MngDMZ` | `MANDMZ.config` |
| `Sec30DMZ` | `DMZ.config` |

The security bundles differ by how much they lock the system down, and choosing one at install time is far easier than retrofitting hardening onto a production box. `Sec00Tools` merely installs the tooling (Bastille, IPFilter, Secure Shell) without changing behaviour, so it is a safe default when you are unsure. `Sec10Host` applies a host-based hardening profile suitable for a general server. `Sec20MngDMZ` and `Sec30DMZ` progressively tighten the configuration for managed-DMZ and full-DMZ (internet-facing) roles — they disable more services and apply stricter IPFilter rules. The trade-off is that stronger profiles can disable services your application actually needs, so validate connectivity after applying anything above `Sec10Host`.

```bash
# Apply a Bastille config after install
bastille -b -f /etc/opt/sec_mgmt/bastille/configs/defaults/HOST.config

# Which config is currently applied?
bastille -l

# Revert Bastille's changes if hardening broke something
bastille -r
```

A frequent gotcha: install-time security bundles are only offered on **11i v2/v3**. On 11i v1 you install and run Bastille manually after the OS is up. Also remember that Bastille's changes are reversible with `bastille -r`, so if a hardening step blocks a needed service you can roll back rather than reinstall.

## Source Media

The 11i media kit is four DVDs — **only the first** is required for the OS install/update:

| DVD | Contents |
|-----|----------|
| Core / Operating Environment | Base OS (required) |
| Instant Information | Full HP-UX documentation set |
| Applications | Additional software |
| Internet Express | Pre-compiled open-source products |

- 11i **v1** kits include a **Support+** CD of recommended patches.
- 11i **v2/v3** kits have **no** Support+ CD — the **Quality Pack** patch bundles are on the OE DVD and install automatically.

## Initiating the Install

The two hardware families reach the Ignite menus by completely different firmware paths. PA-RISC systems use **BCH** (Boot Console Handler) and hand off to **ISL/IPL**; Integrity (Itanium) systems use **EFI** (Extensible Firmware Interface). Knowing which you are on determines every keystroke below. In both cases the key discipline is the same: interrupt the automatic boot, point firmware at the install source, and then **let go** — the Ignite install kernel takes over and presents its own menus. Fiddling with ISL or IPL by hand after selecting the install source is a classic way to derail a network install.

### PA-RISC (BCH)

1. Access the console interface (via the [MP](articles/hpux-management-processor.md)).
2. Start the boot: `shutdown -ry` or `MP:CM> rs`.
3. Press **Escape** to interrupt autoboot and reach the BCH main menu.
4. Boot the install source; **don't** interact with ISL/IPL; wait for the Ignite-UX menus.

```
search ipl                       # find bootable devices; lists them as P0, P1, ...
boot P0                          # boot the searched device at index 0 (e.g. the DVD)
boot pri                         # boot the primary boot path
boot lan 10.1.1.1 install        # boot/install from an Ignite-UX server
boot lan install                 # broadcast for any Ignite server on the subnet
```

The `install` keyword is what tells firmware to boot the Ignite install kernel rather than a normal OS. `boot lan <ip> install` targets one specific Ignite server, which is more reliable than a broadcast when several servers or routed subnets are involved. If `search ipl` does not list your DVD, confirm the drive is seated and try `search` (without `ipl`) to see all paths.

### Integrity (EFI)

1. Access the console interface.
2. Start the boot: `shutdown -ry` or `MP:CM> rs`.
3. Press **Escape** to interrupt EFI autoboot.
4. Select the DVD/tape/Ignite source; wait for the Ignite-UX menus.

On Integrity, network install is driven from the EFI shell rather than a `boot lan` command. A typical sequence is to enter the EFI Shell, list the mappable devices, and launch the DBProfile/network boot entry:

```
Shell> map -r                    # list boot/filesystem devices and network handles
Shell> dbprofile -dn <profile>   # (optional) inspect a stored network boot profile
Shell> lanboot select            # pick a NIC and PXE/Ignite-boot from it
```

For a DVD install, `map -r` will show a filesystem handle such as `fs0:` for the optical drive; select it and run the installer from there. If the NIC does not appear, verify the network cable and that the correct core I/O LAN port is used for boot.

## Navigating the Ignite-UX Menus

Choose a user-interface mode:

| Mode | What you can do |
|------|-----------------|
| **Guided Installation** | Set basic parameters via a menu on the target; **no** LVM/filesystem tuning |
| **Advanced Installation** | Full control — tune LVM/filesystems, networking, and select software/patches |
| **Remote graphical (on the Ignite server)** | Do the rest of the config on the Ignite server, not the target |

Guided mode is the quickest path for a straightforward single-disk install where the defaults are acceptable. Choose Advanced mode when you need a specific vg00 layout (multiple logical volumes, non-default filesystem sizes, HFS vs VxFS choices), when you want to pre-select software beyond the OE, or when you are laying down mirroring at install time. The remote graphical option is convenient in a data centre: you drive the whole configuration from the Ignite server's GUI (`/opt/ignite/bin/ignite`) while the target just runs the install kernel and waits.

Regardless of mode, the tabbed configuration screen groups the same decisions: **Basic** (root disk, file system type, root swap, languages), **Software** (which bundles/patches), **System** (hostname, time zone, network, root password), and **File System** (per-volume sizes). Review the **File System** tab carefully — undersizing `/var` or `/stand` is a frequent regret that is painful to fix after the system is in production.

## Verifying the Install

Check for SD-UX errors — especially ERRORs and WARNINGs:

```bash
more /etc/rc.log                 # boot / service startup errors
swverify \*                       # verify all installed products
more /var/adm/sw/swinstall.log
more /var/adm/sw/swagent.log
```

## Post-Install Configuration

Common cleanup steps: install patches, move root's home to `/root`, update config files, install applications, import volume groups, populate the HPSP, and back up the system.

### Move root's Home to /root

```bash
mkdir /root
chown root:sys /root
chmod 700 /root
mv /.profile /.shrc /.sh_history /root
usermod -d /root root
vi /root/.profile                # update HISTFILE if needed
```

### Import Volume Groups

On the source system, export a map file; copy it over; recreate the group device file; import and activate. (See [HP-UX LVM](articles/hpux-lvm.md) for the group-device details.)

```bash
# On the source system
vgchange -a n vg01
vgexport -s -v -m /etc/lvmconf/vg01.map vg01

# On the new system
scp otherhost:/etc/lvmconf/vg01.map /etc/lvmconf/vg01.map
mkdir /dev/vg01
mknod /dev/vg01/group c 64 0x010000
chmod 640 /dev/vg01/group
chown root:sys /dev/vg01/group
vgimport -s -v -m /etc/lvmconf/vg01.map vg01
vgchange -a y vg01
vgcfgbackup vg01
# then add the filesystems to /etc/fstab
```

### Populate the HPSP (Integrity)

Ignite creates the HP Service Partition but you must copy the e-Diag utilities to it from the Diagnostics & Utilities CD:

```
# 1. Insert the diagnostics/utility CD, reboot to the EFI shell
SHELL> map -b -r -fs                                  # find the DVD/CD device
SHELL> fsn:\efi\boot\launchmenu\launchmenu.efi        # run the diag menu; follow prompts to copy
```

### Back Up the New System

```bash
make_tape_recovery -x inc_entire=vg00     # bootable recovery tape of vg00
drd clone -t /dev/rdisk/disk3             # Dynamic Root Disk clone to another disk
```

## Ignite Server Management

```bash
/opt/ignite/bin/manage_index -l           # list configured installation images
/opt/ignite/bin/instl_adm -T              # check config syntax
ls -l /var/opt/ignite/recovery/archives   # client recovery archives
/opt/ignite/bin/instl_adm -d              # dump the current default config values
/opt/ignite/bin/ignite                    # launch the Ignite server GUI/TUI
```

The Ignite server keeps two important trees under `/var/opt/ignite/`. The `clients/` directory holds one subdirectory per known client (named by link-level address), containing that client's install and recovery configuration. The `recovery/archives/` directory holds the actual `make_net_recovery` archive files. When disk space on the server runs low, this archive tree is usually the culprit — trim old per-client archives (respecting your `SAVE_NUM_ARCHIVES` policy) rather than deleting client config directories.

## Troubleshooting

Even a well-prepared install can stall. A few recurring situations and where to look:

- **Client boots but never reaches the Ignite menu (network install).** Almost always a boot-services problem on the server: `instl_bootd`, `tftp`, and `bootpd`/DHCP must be enabled in `/etc/inetd.conf` and the client's address reachable. Confirm with `swlist IGNITE` that Ignite is installed and check `/var/opt/ignite/logs/` on the server.
- **`boot lan` finds no server.** Use the explicit `boot lan <ip> install` form; broadcasts do not cross routers. Verify the client and server share a subnet or that a boot helper/relay is configured.
- **Install aborts with SD-UX errors partway through software load.** Read `/var/adm/sw/swinstall.log` and `/var/adm/sw/swagent.log` on the client (accessible from the Ignite session's shell). Corrupt or incomplete depots and missing dependencies are the usual causes.
- **System installs but will not boot afterward.** Check that the boot path and LVM/BDRA (Boot Data Reserved Area) were written correctly with `lvlnboot -v`; on a mirrored root confirm both members are bootable. See [HP-UX Boot Process](articles/hpux-boot-process.md).
- **Recovery archive restored to different hardware is unbootable.** Expected — `make_net_recovery` is machine-specific. Use `make_sys_image` for cross-hardware cloning.

```bash
lvlnboot -v                              # verify the boot, root, swap, and dump LVs on vg00
setboot                                  # show/adjust primary and alternate boot paths
```

## Command Reference

| Task | Command |
|------|---------|
| Which OE installed | `swlist -l bundle "HPUX11i-*OE*"` |
| OE contents | `swlist -l product "HPUX11i-*OE*"` |
| Update the OE | `swinstall -s /dvd Update-UX` + `update-ux -s /dvd <OE>` |
| Apply Bastille config | `bastille -b -f <config>` / `bastille -l` |
| PA-RISC install (BCH) | `search ipl`, `boot P0` / `boot pri`, `boot lan <ip> install` |
| Verify install | `swverify \*`, `more /etc/rc.log` |
| Move root home | `usermod -d /root root` |
| Import a VG | `vgimport -s -v -m <map> <vg>` |
| Build golden image | `make_sys_image -v -s local -d <dir>` |
| Recovery backup | `make_tape_recovery -x inc_entire=vg00`, `drd clone` |
| Ignite images / syntax | `manage_index -l`, `instl_adm -T` |
| Verify boot LVs | `lvlnboot -v` |
| Show/set boot paths | `setboot` |

## Related Articles

- [HP-UX Disaster Recovery (DRD and Ignite-UX)](articles/hpux-disaster-recovery.md)
- [HP-UX Software Depots and swinstall](articles/hpux-software-depots-swinstall.md)
- [HP-UX Patch Management](articles/hpux-patch-management.md)
- [HP-UX Boot Process](articles/hpux-boot-process.md)
- [HP-UX LVM](articles/hpux-lvm.md)
- [HP-UX Management Processor](articles/hpux-management-processor.md)
- [HP-UX Kernel Configuration and Tuning](articles/hpux-kernel-configuration.md)
