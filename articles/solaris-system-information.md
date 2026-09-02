# Solaris System Information and Inventory

Quick commands to identify an Oracle Solaris system — OS release, installed software/patch revisions, memory, hardware and diagnostics, service descriptions, and the software cluster it was installed from. Useful for auditing an unfamiliar box or gathering details for a support case.

## OS Release and Version

```bash
# Solaris release string (version, build, date)
cat /etc/release

# Machine, OS revision, and patch revision summary
showrev

# Just the OS name/version
uname -a
uname -r        # release (e.g. 5.11 = Solaris 11)
```

- `/etc/release` gives the friendly release name and build (e.g. "Oracle Solaris 11.4").
- `showrev` reports hostname, hostid, release, kernel architecture, application architecture, and (with `-p`) installed patches.
- `uname -r` shows the SunOS release number: `5.10` = Solaris 10, `5.11` = Solaris 11.

Sample outputs:

```
$ cat /etc/release
                             Oracle Solaris 11.4 SPARC
  Copyright (c) 1983, 2018, Oracle and/or its affiliates.  All rights reserved.
                            Assembled 16 August 2018

$ uname -a
SunOS server01 5.11 11.4.0.0.1.15.0 sun4v sparc sun4v

$ showrev
Hostname: server01
Hostid: 84a1b2c3
Release: 5.11
Kernel architecture: sun4v
Application architecture: sparc
Kernel version: SunOS 5.11 11.4
```

`uname` fields: nodename (`server01`), release (`5.11`), version (`11.4...`), machine hardware (`sun4v`), processor (`sparc`).

### Architecture (32/64-bit) and Host ID

```bash
# Show supported instruction sets and native (kernel) bits
isainfo -v
isainfo -b        # 32 or 64
isainfo -kv       # kernel ISA

# Host ID (used for licensing) and hostname
hostid
hostname
```

## Memory

```bash
# Total physical memory
prtconf | grep -i memory
```

`prtconf` prints the system configuration; the "Memory size" line reports total installed RAM. (See also `echo ::memstat | mdb -k` for a memory breakdown — covered in the performance article.)

## Hardware and Diagnostics

```bash
# System configuration + diagnostic info (hardware, FRUs, fault status)
prtdiag

# Verbose hardware view
prtdiag -v

# Full device/config tree
prtconf

# CPU inventory
psrinfo -pv
```

- `prtdiag` — hardware configuration and diagnostic status (memory, I/O, fans, temperature, failed FRUs). Especially useful on SPARC and Oracle x86 hardware.
- `prtconf` — the device tree and total memory.

## Software Revisions and Patches

```bash
# Machine, software revision, and patch revision info
showrev

# Installed patches only
showrev -p

# Package/patch details (Solaris 10)
pkginfo -l SUNWxxx
```

## Services

```bash
# List services with their human-readable descriptions
svcs -o FMRI,DESC

# All service states
svcs -a
```

`svcs -o FMRI,DESC` is handy for discovering what a service actually does when its FMRI name isn't obvious. See the [Solaris SMF](articles/solaris-smf-services.md) article for full service management.

## Install Cluster and Logs

```bash
# The software cluster (metacluster) the system was installed from
cat /var/sadm/system/admin/CLUSTER

# Installation log
cat /var/sadm/system/logs/install_log
```

- `/var/sadm/system/admin/CLUSTER` names the install metacluster (e.g. `SUNWCreq` core, `SUNWCuser` end-user, `SUNWCall` entire, `SUNWCXall` entire + OEM) — tells you how much of Solaris was installed.
- `/var/sadm/system/logs/install_log` records the original OS installation.

## Command Reference

| Task | Command |
|------|---------|
| OS release | `cat /etc/release` |
| OS/kernel version | `uname -a`, `uname -r` |
| Machine + revisions | `showrev` |
| Installed patches | `showrev -p` |
| Total memory | `prtconf \| grep -i memory` |
| Hardware diagnostics | `prtdiag` (`-v`) |
| Device tree | `prtconf` |
| CPU inventory | `psrinfo -pv` |
| Service descriptions | `svcs -o FMRI,DESC` |
| Install cluster | `cat /var/sadm/system/admin/CLUSTER` |
| Install log | `/var/sadm/system/logs/install_log` |

## Notes and Gotchas

| Situation | Note |
|-----------|------|
| `prtdiag` says "not supported" | `prtdiag` mainly targets Sun/Oracle hardware; on generic x86 or VMs it may return little — use `prtconf`/`smbios` instead |
| Solaris 10 vs 11 patch info | `showrev -p` is meaningful on Solaris 10; on Solaris 11 use `pkg info entire` for the SRU/version |
| `/etc/release` edited | It's a plain text file and can be altered; cross-check with `uname` and `pkg info` |
| Zone vs global | Inside a non-global zone, hardware commands (`prtdiag`, `prtconf`) reflect the global zone's view or are restricted |

```bash
# Solaris 11: the authoritative OS version/SRU
pkg info entire | egrep 'Version|Branch'

# x86 firmware/hardware inventory when prtdiag is thin
smbios | less
```

## Quick "What Is This Box?" Snapshot

```bash
echo "=== Release ==="   && cat /etc/release
echo "=== Version ==="   && uname -a
echo "=== Revision ===" && showrev | head
echo "=== Memory ==="    && prtconf | grep -i memory
echo "=== CPU ==="       && psrinfo -pv
echo "=== Hardware ===" && prtdiag | head -20
echo "=== Cluster ===" && cat /var/sadm/system/admin/CLUSTER
```

## References

- [showrev(1M) man page](https://docs.oracle.com/cd/E23824_01/html/821-1462/showrev-1m.html) — official Oracle docs
- [prtdiag(1M) man page](https://docs.oracle.com/cd/E23824_01/html/821-1462/prtdiag-1m.html) — official Oracle docs
- [prtconf(1M) man page](https://docs.oracle.com/cd/E23824_01/html/821-1462/prtconf-1m.html) — official Oracle docs
