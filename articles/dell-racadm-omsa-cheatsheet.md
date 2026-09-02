# Dell racadm and OMSA Cheatsheet

Command reference for managing Dell PowerEdge servers from Linux — `racadm` for the iDRAC/DRAC out-of-band controller, server power state control, and OMSA (OpenManage Server Administrator) for in-band hardware inspection of storage, chassis, and system health.

`racadm` talks to the iDRAC (the baseboard management controller) and works even when the OS is down. OMSA (`omreport`/`omconfig`) runs inside the OS and reports on the physical hardware — controllers, disks, temps, power, fans.

## racadm — iDRAC / DRAC Management

`racadm` is provided by the `srvadmin-racadm` package (historically `srvadmin-racadm5`). It can run locally on the host or remotely against an iDRAC.

### Reset the Controller

```bash
# Reset the RAC (default is a soft reset after ~3s)
racadm racreset

# Explicit reset types
racadm racreset soft       # microprocessor reset; PCI config preserved
racadm racreset hard       # full RAC reset, closest to power-on; PCI config lost
racadm racreset graceful   # same as soft

# Add a delay of 1-60 seconds before reset (default 3)
racadm racreset soft 10
```

- **soft / graceful** — resets the processor core to restart RAC firmware. The RAC log, database, and selected daemons shut down gracefully first. PCI configuration is preserved.
- **hard** — resets the entire RAC, as close to a power-on reset as software allows. Use as a last resort; PCI configuration is lost.
- **`<delay>`** — seconds to wait before the reset sequence starts (valid 1–60, default 3).

### System Information and Logs

```bash
# General server info
racadm getsysinfo

# Service Tag (Dell asset identifier)
racadm getsvctag

# NIC configuration of the iDRAC
racadm getniccfg

# System Event Log — check why the health/amber LED is blinking
racadm getsel

# Clear the System Event Log
racadm clrsel

# RAC access/audit log (user-level login activity)
racadm getraclog

# Enclosure/sensor health
racadm getsensorinfo

# Installed firmware inventory
racadm swinventory
```

> Note: on modern iDRAC the "clear SEL" subcommand is `clrsel` (older/legacy syntax used `clear`). Check `racadm help` on your generation if one is rejected.

### Server Power Control

```bash
racadm serveraction powerstatus   # show current power status (ON / OFF)
racadm serveraction powerup        # power on
racadm serveraction powerdown      # power off
racadm serveraction powercycle     # power off then on (like the front-panel button)
racadm serveraction hardreset      # reset/reboot the managed system
```

| Action | Effect |
|--------|--------|
| `powerstatus` | Reports current power state (ON/OFF) |
| `powerup` | Powers the system on |
| `powerdown` | Powers the system off |
| `powercycle` | Full off→on cycle, like pressing the power button |
| `hardreset` | Reboot/reset the managed system |

### Firmware, Password, and Factory Reset

```bash
# Update firmware
racadm fwupdate

# Change the password for a user slot (index -i 2 = second user, often admin)
racadm config -g cfgUserAdmin -o cfgUserAdminPassword -i 2 'new_password'

# Reset the iDRAC/RAC configuration to factory defaults
racadm racresetcfg
```

`racresetcfg` wipes iDRAC settings (including network config), so make sure you have another access path before running it remotely.

## OMSA — OpenManage Server Administrator

OMSA runs in the OS and reports the actual hardware state. `omreport` reads; `omconfig` changes.

### Managing the OMSA Services

```bash
/opt/dell/srvadmin/sbin/srvadmin-services.sh status
/opt/dell/srvadmin/sbin/srvadmin-services.sh start
/opt/dell/srvadmin/sbin/srvadmin-services.sh stop
/opt/dell/srvadmin/sbin/srvadmin-services.sh restart
/opt/dell/srvadmin/sbin/srvadmin-services.sh enable    # start on boot
/opt/dell/srvadmin/sbin/srvadmin-services.sh disable
```

### System and Chassis Overview

```bash
# Find the iDRAC IP from within the OS
omreport chassis remoteaccess

# Chassis overview and full system summary (all components)
omreport chassis
omreport system summary

# OS and component versions
omreport system operatingsystem
omreport system version

# Discover available options on your system
omreport system -?
omreport chassis -?
```

### Health: Thermals, CPU, Memory, Power

```bash
omreport chassis temps            # temperature probes
omreport chassis processors       # CPU info
omreport chassis memory           # memory slots
omreport chassis fans             # fans
omreport chassis volts            # voltage probes
omreport chassis batteries        # batteries
omreport chassis pwrsupplies      # power supplies
omreport chassis slots            # PCI slots
omreport chassis nics             # network interfaces
omreport chassis pwrmonitoring    # power consumption
omreport chassis pwrmanagement    # power inventory and budget
```

### Logs and Events

```bash
omreport system alertlog          # alert log
omreport system events            # events
omreport system alertaction       # configured alert actions
omreport system esmlog            # Embedded System Management (hardware) log
```

### Storage: Controllers, Virtual and Physical Disks

```bash
# Controllers and virtual disks
omreport storage controller
omreport storage vdisk

# Physical disks on a given controller
omreport storage pdisk controller=0
```

Identify a disk by blinking its LED:

```bash
omconfig storage pdisk action=blink   controller=0 pdisk=0:0:1
omconfig storage pdisk action=unblink controller=0 pdisk=0:0:1
```

Import/clear a foreign config and assign a hot spare (common after replacing a disk):

```bash
# Change a disk's state from Foreign to Ready
omconfig storage controller action=clearforeignconfig controller=0

# Add the disk as a Global Hot Spare (Dell's recommended approach)
omconfig storage pdisk action=assignglobalhotspare controller=0 pdisk=0:0:1 assign=yes
```

Export the RAID controller log (written to `/var/log/lsi_<date>.log`):

```bash
omconfig storage controller action=exportlog controller=0
```

### Deleting Virtual Disks

Deleting a virtual disk destroys everything on it — filesystems, volumes, data — and removes it from the controller configuration.

```bash
omconfig storage vdisk action=deletevdisk controller=0 vdisk=0
```

Behaviors and cautions to know before deleting:

- **Data loss is total.** All filesystems and volumes on the VD are destroyed.
- **Hot spare side effects.** Deleting the *last* VD associated with a controller may auto-unassign its global hot spares. Deleting the *last* VD of a disk group turns that group's dedicated hot spares into global hot spares. If you delete all VDs that a global hot spare backs, that global hot spare is automatically removed.
- **Boot VD warning.** When allowed, you can delete a boot virtual drive — this happens out-of-band (sideband), independent of the OS, so a warning is shown first.
- **Stale data on recreate.** If you delete a VD and immediately recreate one with identical characteristics, the controller may recognize the old data as if nothing was deleted. Re-initialize the new VD if you don't want the old data:

  ```bash
  # Clear old data after recreating a VD with the same characteristics
  omconfig storage vdisk action=initialize controller=0 vdisk=0
  ```

- **Privileges.** You need the **Login** and **Server Control** privileges to delete virtual disks.

Foreign configurations (disks moved from another controller) can also be imported from the System Setup / BIOS menu — see the Dell KB linked in [References](#references).

## Licensing and Service Tag

```bash
# View the installed iDRAC license
racadm license view

# Import a perpetual license file onto the embedded iDRAC
racadm license import -f /mountpath/yourlicensefilename.xml -c idrac.embedded.1

# Change the system Service Tag (e.g. after a board swap)
smbios-sys-info --service-tag --set=YOURTAG
```

`smbios-sys-info` comes from Dell's `srvadmin`/`libsmbios` tooling. Changing the Service Tag is normally only needed after a system board replacement.

## CMC — Chassis Management Controller

The CMC manages a blade chassis (e.g. PowerEdge M1000e) — a separate controller from the per-blade iDRAC. SSH to the CMC hostname and use `racadm` there.

### Chassis Inventory and Health

```bash
racadm getmodinfo       # health, presence, service tags of chassis, blades, switches
racadm getmacaddress    # MAC addresses of every blade/chassis Ethernet interface
racadm getsensorinfo    # fan speeds, ambient temperature, power supply status
racadm getpbinfo        # power budget/status
racadm getpminfo        # power management info
racadm getversion       # blade iDRAC versions and blade types
racadm getversion -c    # blade CPLD versions
racadm getversion -b    # blade BIOS versions
```

### Collecting CMC Support Logs

Run these to gather diagnostics; redirect over SSH to capture everything to one file:

```bash
# From your workstation — dump all CMC logs to a local file
ssh mym1000ehostname dumplogs > /tmp/cmc-logs.txt
```

Individual log/info commands available in the CMC shell:

```sh
getsysinfo
getioinfo
getdcinfo
getpbinfo
getsel
getraclog
racdump
dumplogs
```

## Firmware Package Handling

Dell Linux firmware bundles (`.BIN`) can be extracted without applying them:

```bash
sh Network_Firmware_82J79_LN_08.07.26_A00-00.BIN --extract <dir>
```

## racadm vs OMSA — When to Use Which

| Need | Use | Works when OS is down? |
|------|-----|------------------------|
| Power control, remote console reset | `racadm` (iDRAC) | Yes |
| Read/clear System Event Log (LED cause) | `racadm getsel` / `clrsel` | Yes |
| iDRAC network/password/firmware | `racadm` | Yes |
| Physical/virtual disk state, RAID actions | OMSA (`omreport`/`omconfig storage`) | No (needs OS) |
| Temps, fans, PSUs, memory health | OMSA (`omreport chassis ...`) | No (needs OS) |
| Find the iDRAC IP from the OS | `omreport chassis remoteaccess` | No (needs OS) |

## Tips

- The amber/health LED usually maps to an entry in the SEL — start with `racadm getsel` (or `omreport system esmlog`) to find the cause, then `clrsel` once resolved.
- `pdisk` targets use `connector:enclosure:slot` form (e.g. `0:0:1`); confirm the exact ID with `omreport storage pdisk controller=0`.
- Prefer assigning a replaced drive as a Global Hot Spare over forcing it online, so the controller rebuilds cleanly.
- OMSA has been superseded on the newest PowerEdge generations by iDRAC-centric tooling; on those, drive/health operations may live under `racadm storage` and the iDRAC UI instead.

## References

- [Dell OpenManage Server Administrator documentation](https://www.dell.com/support/kbdoc/en-us/000177052) — official Dell docs
- [iDRAC RACADM CLI Guide](https://www.dell.com/support/manuals/en-us/idrac9-lifecycle-controller-v4.x-series/idrac_4.00.00.00_racadm_pub) — official Dell docs
- [Dell DRAC (Wikipedia)](https://en.wikipedia.org/wiki/Dell_DRAC) — background
