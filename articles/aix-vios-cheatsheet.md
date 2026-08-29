# AIX VIOS Cheatsheet

Command reference for the IBM PowerVM Virtual I/O Server (VIOS) — the special AIX-based appliance that owns physical adapters and virtualizes storage and networking for client LPARs. You administer it through the restricted `padmin` shell using `ioscli` commands (typically typed without the `ioscli` prefix). This covers concepts, system commands, devices, virtual SCSI/NPIV storage, the virtual media repository, Shared Ethernet Adapters and failover, monitoring, updates, and cloning.

> Log in as **`padmin`**. The shell is restricted — run `oem_setup_env` to drop to a full root AIX shell when you need standard commands (use sparingly; IBM support prefers the `ioscli` interface). For LVM/backup context on the underlying AIX, see the [AIX LVM](articles/aix-lvm-cheatsheet.md) and [Backup and Recovery](articles/aix-backup-recovery-cheatsheet.md) cheatsheets.

## What VIOS Provides

The Virtual I/O Server provides virtual storage and shared Ethernet to client LPARs, letting physical adapters (with attached disks and optical devices) be shared by one or more clients. VIOS partitions aren't meant to run applications or general user logins — VIOS runs in its own partition. It enables:

- Sharing physical resources between partitions
- Creating partitions without additional physical I/O resources
- Creating more partitions than there are I/O slots or physical devices (dedicated I/O, virtual I/O, or both)
- Maximizing physical resource utilization

**Disk resources it can present:** logical volumes as disks, virtual SCSI disks, and Virtual Fibre Channel (NPIV).
**Network resources:** Shared Ethernet Adapters (SEA) and Integrated Virtual Ethernet (IVE).

### Product details

- Component of the PowerVM Editions feature, supplied as an AIX `mksysb` image
- Installed from HMC, managed system, NIM server, or via a deployed system plan
- A customized AIX-based appliance; runs only in special VIOS partitions
- Must have physical I/O slots for storage and networking
- Sizing guidance: start with ~512 MB memory (monitor and grow), minimum 16 GB disk, dedicated processors or a shared pool. vSCSI-only servers use little CPU; SEA support makes CPU sizing important for performance.

## Virtual I/O Basics

Each partition is configured by default with **10 virtual I/O slots** (more can be added); each slot can hold a virtual adapter instance so partitions can share devices and get virtual Ethernet between partitions on the same system. Virtual adapters behave like real adapter cards to the OS and appear in inventory — and, like physical adapters, must be deconfigured from the OS before a DLPAR remove.

Every partition needs at least **two adapter slots**: one for a boot device (SCSI or Fibre Channel) and one for an Ethernet adapter (also required for the HMC to do DLPAR and serviceable-event functions).

### Creating a Virtual SCSI host adapter

1. On the **HMC**, in the VIO server's profile, create a new Virtual SCSI Host Adapter and assign it to your client LPAR.
2. Assign the **same slot ID** to a new client adapter in the client LPAR's profile.

## The ioscli Interface

Commands are the `ioscli` subcommands; in the `padmin` shell you type them without the prefix. From a root shell you call them by full path.

```sh
/usr/ios/cli/ioscli lsmap -all

# Handy alias from a root/OEM shell
alias i=/usr/ios/cli/ioscli
i lsmap -all

export CLI_DEBUG=33      # show the AIX commands running "under the covers"
oem_setup_env            # drop to the root AIX shell (then 'exit' to return)
```

## System and Version

```sh
ioslevel                 # VIOS software version (like AIX oslevel)
lssw                     # installed software
license                  # view the VIOS license
license -accept          # accept the license
motd "login message"     # view/set /etc/motd
uptime
lsfware                  # firmware level
oem_platform_level       # underlying AIX level
lsgcl                    # timestamped VIOS command history
```

## Users and Shell

VIOS supports HMC-like user types: **prime administrator** (only one — `padmin`), **service representative**, and **development engineer**.

```sh
lsuser                   # list users
lsuser padmin            # attributes of a user
mkuser <name>            # create a user
rmuser <name>            # remove a user
passwd                   # change your password
who
```

Use `cfgassist` for a SMIT-style menu (set date/time/timezone, change passwords, system security, TCP/IP config, install/update software). You can also set the clock directly with `chdate`.

## Devices

VIOS uses the standard AIX device model (`hdisk`, `ent`, `fcs`, `vhost`, `vfchost`, `vtscsi`, `vtopt`).

```sh
lsdev                              # all devices and states
lsdev -type disk                   # by type
lsdev -type adapter
lsdev -type optical                # optical devices
lsdev -virtual                     # virtual devices only
lsdev -virtual -type adapter
lsdev -virtual -type disk
lsdev -dev fcs0 -attr              # attributes of a device
lsdev -dev fcs0 -vpd               # Vital Product Data (like lscfg -vpl)
lsdev -dev vhost0 -vpd             # same as lscfg -vpl vhost0
lsdev -dev hdisk0 -range queue_depth   # valid range for an attribute
lsdev -dev hdisk10 -parent         # parent device
lsdev -dev fcs0 -child             # children of a device
lsdev -slots                       # physical and virtual adapter slots

cfgdev                             # configure newly attached devices (like cfgmgr)
cfgdev -dev fcs0                   # configure from a parent device

chdev -dev hdisk0 -attr <name>=<value>       # change an attribute
rmdev -dev name_hdisk1             # remove and delete a device
rmdev -dev name_hdisk1 -ucfg       # remove but leave Defined
rmdev -dev vhostX -recursive       # remove an adapter and all child devices

lspath                             # MPIO paths
lspath -l hdisk55 -F name,parent,connection,status
lslot="lsslot -c slot | grep C69"  # find the vhost for slot number 69

# Disk discovery helpers
lspv -free                         # unmapped disks
lspv -size                         # disks with sizes
lslv -pv lvname                    # LVs on a PV
lsvg -lv datavg                    # LVs in a VG (like lsvg -l)
```

## Storage: Virtual SCSI Mapping

VIOS maps backing devices (whole disks, logical volumes, or files) to client LPARs through `vhost` (virtual SCSI server) adapters as `vtscsi`/`vtopt` virtual targets.

```sh
# List mappings (which backing device serves which client)
lsmap -all
lsmap -all | more
lsmap -vadapter vhost0
lsmap -virtual                     # all virtual devices on the VIOS

# Map a whole disk to a client via vhost0 (auto or named VTD)
mkvdev -vdev hdisk3 -vadapter vhost0
mkvdev -vdev hdisk1 -vadapter vhost1 -dev name_hdisk1

# Map a logical volume as a virtual disk
mkvdev -vdev clientlv -vadapter vhost0 -dev vtscsi1

# Remove a virtual target device
rmvdev -vtd vtscsi0
rmvdev -vdev hdisk5                 # or by backing device
rmdev -dev name_hdisk1             # (also removes the VTD)
```

### Prepare disks for use as VTDs

```sh
# Disable SCSI reservations on each disk used as a VTD
chdev -dev hdisk1 -attr reserve_policy=no_reserve -perm
lsdev -dev hdisk7 -attr            # check flags
```

### Storage pools and backing devices

```sh
lssp                                   # list storage pools
lssp -detail -sp rootvg
mksp datapool hdisk3                   # create a storage pool
mkbdsp -sp datapool 20G -bd clientbd -vadapter vhost0   # carve + map a backing device
rmbdsp -sp datapool -bd clientbd       # remove a backing device
```

### FC adapter tuning (for MPIO/dual-VIOS)

```sh
# Fast-fail error recovery + dynamic tracking (survives cabling/SAN changes)
chdev -dev fscsi0 -attr fc_err_recov=fast_fail -perm
chdev -dev fscsi0 -attr dyntrk=yes -perm
# Combined (AIX -l/-a form, deferred with -P)
chdev -l fscsi0 -a dyntrk=yes -a fc_err_recov=fast_fail -P

# Force point-to-point link type
chdev -dev fcs0 -attr init_link=pt2pt -perm
```

### Queue depth (with a quiesced filesystem)

```sh
lsattr -El hdisk6 -a queue_depth                 # ODM value
echo scsidisk hdisk6 | kdb | grep queue_depth    # real (running) value, in hex

umount /test
varyoffvg testvg
chdev -l hdisk6 -a queue_depth=256
varyonvg testvg
mount /test

lsattr -El hdisk6 -a queue_depth
echo "ibase=16 ; 100" | bc                       # convert a hex value to decimal
```

## Storage: NPIV (Virtual Fibre Channel)

With NPIV, client LPARs get their own virtual WWPNs through a `vfchost` adapter mapped to a physical FC port.

```sh
# Mappings and status
lsmap -all -npiv
lsmap -npiv -vadapter vfchost0

# Inventory
lsnports                           # physical FC ports and NPIV readiness (adapter + SAN switch)
lsdev -dev vfchost*                # virtual FC server adapters
lsdev -dev fcs*                    # physical FC adapters
lsdev -dev fcs* -vpd               # physical FC adapter properties
lscfg -vl fcsX                     # (on the AIX client) virtual FC properties

# Map / unmap a vfchost to a physical FC port
vfcmap -vadapter vfchost0 -fcp fcs0
vfcmap -vadapter vfchost0 -fcp ''  # unmap
echo "vfcs" | kdb                  # which vfchost is used on the VIOS

# Reporting one-liners
lsmap -all -npiv -field "FC name" Status Name ClntName -fmt :             # every VFC + attached host
lsmap -all -npiv -field "FC name" Status Name ClntName -fmt : | grep -v fcs   # lpars not mapped to a physical adapter
lsmap -all -npiv -field "FC name" Status Name ClntName -fmt : | grep NOT | grep fcs  # not connected (maybe shut down)
/usr/ios/cli/ioscli lsmap -all -npiv -field "FC name" -fmt : | sort -n | uniq -c     # FCS distribution

viostat -adapter                   # adapter I/O stats
```

### Remove a vfchost

```sh
lsdev -dev vfchost*                # verify
lsmap -npiv -vadapter vfchost10
vfcmap -vadapter vfchost10 -fcp    # unmap
rmdev -dev vfchost10               # remove; then adjust DLPAR & profile
```

### Log a client virtual FC into the SAN (from the HMC)

```sh
chnportlogin -m sys0090 -o login -p prddbl4b -n default
lsnportlogin -m sys0090 --filter "\"lpar_ids=4\""
```

WWPN status codes: `0` = not activated, `1` = activated, `2` = unknown.

## Optical: Virtual Media Repository

Serve ISO images to clients as virtual optical drives.

```sh
mkrep -sp rootvg -size 10G                 # create the repository (virtual library)
mkvopt -name aix71dvd1 -file AIX71_TL0SP1_1.iso    # add an ISO
mkvopt -name dvd.AIX_6.1.iso -dev cd0 -ro  # create an ISO from a physical CD/DVD

mkvdev -fbo -vadapter vhost4               # create a file-backed virtual optical target
loadopt -vtd vtopt0 -disk dvd.AIX_6.1.iso  # load an ISO into the target
unloadopt -vtd vtopt0                      # eject

lsrep                                      # show repository contents
chrep                                      # change repository characteristics
rmrep                                      # remove the repository
lsvopt                                     # file-backed optical device info
chvopt / rmvopt                            # change / remove a virtual optical disk
```

### Map a physical optical drive to a client

```sh
lsdev -type optical                # does VIOS own an optical device?
lsmap -all | grep cd0              # is cd0 already mapped?
mkvdev -vdev cd0 -vadapter vhost0  # map cd0 to a client
rmdev -dev vtopt0 -recursive       # remove the VTD
```

## Networking

### Interfaces and link aggregation

```sh
mkvdev -lnagg ent0 ent1                       # EtherChannel / 802.3ad link aggregation
mkvdev -vlan ent9 -tagid 200                  # VLAN-tagged interface over ent9 (can be a SEA)

# Configure IP on an interface
chdev -dev en3 -attr netaddr=10.0.0.5 netmask=255.0.0.0 state=up
chdev -dev en7 -perm -attr netaddr=20.20.20.20 -attr netmask=255.255.255.0 -attr state=up
mktcpip -hostname vios1 -inetaddr 10.0.0.10 -interface en5 \
  -netmask 255.255.255.0 -gateway 10.0.0.1

# Inspect
lstcpip                            # TCP/IP settings
lstcpip -adapters                  # virtual and physical network adapters
netstat -state                     # interface state
netstat -state -num                # numeric only
netstat -routtable                 # routing table
netstat -routtable -num
netstat -v | grep Prio             # which VIOS SEA is active
entstat -drt <sea> | grep -i vlan  # display a VLAN
```

### Shared Ethernet Adapter (SEA)

A SEA bridges a virtual Ethernet (from the hypervisor) to a physical adapter so clients reach the external network.

```sh
lsmap -all -net                        # virtual/SEA network mappings
lsmap -all -net -field sea             # list SEAs
lsmap -vadapter ent11 -net             # a specific virtual ethernet device

# Simple SEA: physical <PHYS>, virtual <VIRT>, internal VLAN <VLAN>
mkvdev -sea <PHYS> -vadapter <VIRT> -default <VIRT> -defaultid <VLAN>

# SEA failover: add a control channel and rely on trunk priority
mkvdev -sea <PHYS> -vadapter <VIRT> -default <VIRT> -defaultid <VLAN> \
  -attr ha_mode=auto ctl_chan=<CONT>
mkvdev -sea ent3 -vadapter ent4 -default ent4 -defaultid 4010 -attr ha_mode=auto ctl_chan=ent5

# SEA failover with load sharing (two trunk adapters)
mkvdev -sea ent1 -vadapter ent4,ent5 -default ent4 -defaultid 10 \
  -attr ha_mode=sharing ctl_chan=ent6

rmdev -dev <sea>                       # remove a SEA
```

Field meanings: `<PHYS>` physical adapter (backend); `<VIRT>` virtual adapter; `<VLAN>` internal VLAN ID; `default` the virtual adapter for non-VLAN-tagged packets; `<CONT>` a second virtual adapter for the control channel.

### SEA failover testing

Trunk priority is set at virtual-adapter creation (e.g. VIOS1 = 1, VIOS2 = 2). SEAs (say `ent14`) are created on both VIOS with `ha_mode=auto`.

```sh
# Check state
lsattr -El ent14 | grep ha_mode                 # ha_mode=auto
netstat -v ent14 | grep Active                  # Priority + Active: True/False
entstat -all ent14 | grep -i active

# Manually push the primary to standby (fail over to the other VIOS)
chdev -l ent14 -a ha_mode=standby               # or: chdev -dev <sea> -attr ha_mode=standby
netstat -v ent14 | grep Active                  # Active: False
errpt | head                                    # "BECOME BACKUP"

# Switch back to primary
chdev -l ent14 -a ha_mode=auto
netstat -v ent14 | grep Active                  # Active: True
errpt | head                                    # "BECOME PRIMARY"
```

On the partner VIOS, `netstat -v ent14 | grep Active` and `errpt | head` show the mirror-image `BECOME PRIMARY`/`BECOME BACKUP` transitions.

## Virtual Terminal / Console

Open a console to a client LPAR from the VIOS/HMC.

```sh
lssyscfg -r lpar -F name,lpar_id   # list LPARs and IDs
mkvt -id 5                         # open a virtual terminal to LPAR 5
rmvt -id 5                         # force-close the session
# Type  ~.  (tilde dot) to end the session
```

## Monitoring and Performance

```sh
sysstat                    # who + uptime summary
topas                      # real-time performance monitor
topas -cecdisp             # cross-LPAR (CEC) view
viostat                    # I/O statistics (like iostat)
viostat -extdisk           # extended per-disk stats
viostat -adapter           # per-adapter stats
vmstat
entstat -all ent0          # ethernet adapter statistics
fcstat fcs0                # fibre channel adapter statistics

errlog                     # brief error log (like errpt)
errlog -ls                 # full error log entries
errlog -rm 30              # remove entries older than 30 days
```

## Updates and Maintenance

```sh
ioslevel                                   # current level
updateios -commit                          # commit previously applied updates
updateios -install -accept -dev /home/padmin/update   # install a fixpack/update
updateios -cleanup                         # clean up incomplete installs
updateios -remove <fileset>                # remove an update
```

### Patching a SEA-failover VIOS (drain first)

```sh
chdev -dev <sea> -attr ha_mode=standby     # push traffic to the partner VIOS
mount nimmaster:/export/viofp80 /mnt
updateios -install -accept -dev /home/padmin/update
updateios -commit
shutdown -restart
chdev -dev <sea> -attr ha_mode=auto        # return to primary after reboot
```

## Backup and Cloning

```sh
# Full VIOS system backup (like mksysb)
backupios -file /home/padmin/viobackup/vios1.mksysb -mksysb

# Virtual/device configuration backup (mappings only)
viosbr -backup -file /home/padmin/viosbr.bak
viosbr -backup -file backup -frequency daily -numfiles 5   # daily, keep 5, in /home/padmin/cfgbackups
viosbr -view -file <file>                                  # inspect a backup
viosbr -restore -file /home/padmin/cfgbackups/backup.03.tar.gz

# Volume group structure (metadata) backup/restore
savevgstruct <vg>                          # back up an online VG's structure
restorevgstruct                            # restore VG structures to empty disks
```

### Clone the VIOS rootvg with alt_root_vg

```sh
unmirrorios hdisk1
reducevg rootvg hdisk1
su - padmin
alt_root_vg -target hdisk1
bootlist -mode normal hdisk0 hdisk1
bootlist -ls
# ... update VIOS on the clone ...
shutdown -restart          # after reboot, active rootvg is hdisk1; original is now old_rootvg
exportvg <vg>              # drop the disk from the alternate VG
mirrorios                  # re-mirror
```

## Installation and Shutdown

```sh
installios                 # install the VIOS (run from the HMC)
shutdown -restart          # reboot
shutdown -restart -force   # force reboot
shutdown                   # power off
```

> On a dual-VIOS setup, always update/reboot **one VIOS at a time** and confirm client paths and SEA failover are healthy on the surviving VIOS before touching the second.

## Quick Reference

| Task | Command |
|------|---------|
| VIOS version | `ioslevel` |
| Full root shell | `oem_setup_env` |
| Show underlying AIX commands | `export CLI_DEBUG=33` |
| Discover new devices | `cfgdev` |
| List vSCSI mappings | `lsmap -all` |
| Map a disk to a client | `mkvdev -vdev hdisk3 -vadapter vhost0` |
| Disable SCSI reserve on a VTD disk | `chdev -dev hdisk1 -attr reserve_policy=no_reserve -perm` |
| List NPIV mappings | `lsmap -all -npiv` |
| Map virtual FC | `vfcmap -vadapter vfchost0 -fcp fcs0` |
| List NPIV-capable ports | `lsnports` |
| Virtual media repository | `mkrep -sp rootvg -size 10G` / `lsrep` |
| List SEA/net mappings | `lsmap -all -net` |
| Create a SEA | `mkvdev -sea <PHYS> -vadapter <VIRT> -default <VIRT> -defaultid <VLAN>` |
| SEA failover state | `netstat -v ent14 \| grep Active` |
| Virtual console to LPAR | `mkvt -id <id>` (`~.` to exit) |
| I/O / adapter stats | `viostat -extdisk` / `viostat -adapter` |
| Full system backup | `backupios -file <file> -mksysb` |
| Config backup | `viosbr -backup -file <file>` |
| Clone rootvg | `alt_root_vg -target hdisk1` |
| Reboot | `shutdown -restart` |

## Related

- [AIX LVM Cheatsheet](articles/aix-lvm-cheatsheet.md) — volume groups and logical volumes backing vSCSI devices
- [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md) — mksysb concepts (backupios is the VIOS equivalent)
- [AIX NIM Cheatsheet](articles/aix-nim-cheatsheet.md) — installing/restoring VIOS backups over the network
- [AIX SMIT Cheatsheet](articles/aix-smit-cheatsheet.md) — the menu tooling on the underlying AIX
