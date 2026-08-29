# IBM AIX: An Overview

**AIX** (Advanced Interactive eXecutive) is IBM's proprietary line of Unix operating systems, first shipped in 1986 and still developed today. Modern AIX runs on Power ISA-based servers and workstations — IBM's Power Systems line — alongside IBM i and Linux. It is a System V-derived Unix with 4.3BSD-compatible extensions, certified to the Single UNIX Specification (UNIX 03 from AIX 5.3, and UNIX V7 from 7.2 TL5).

> This article is a conceptual overview to orient the [AIX cheatsheets](articles/aix-overview.md#related) collected on this site. Content is paraphrased from the [Wikipedia article on IBM AIX](https://en.wikipedia.org/wiki/IBM_AIX) and rephrased for compliance with licensing restrictions.

## At a Glance

| | |
|---|---|
| Developer | IBM |
| Family | Unix (System V) with BSD extensions |
| First release | February 1986 |
| Latest | 7.3 (TL4, 2025) |
| Platforms (current) | Power ISA |
| Kernel | Monolithic with dynamically loadable modules |
| Default shell | KornShell (ksh88) |
| Default desktop | Common Desktop Environment (CDE); GNOME/KDE optional |
| License | Proprietary |

## Origins and the Unix Lineage

IBM's involvement with Unix began around 1979, when it helped Bell Labs port Unix to the System/370 as a build host for telephone-switch software. Through the early-to-mid 1980s IBM shipped several mainframe Unix efforts — a TSS/370-based system, then **VM/IX** (System III, via Interactive Systems Corporation), and **IX/370** (System V) — before AIX itself appeared.

ISC also built the first AIX for the **IBM RT PC** workstation, introduced in January 1986. It was based on UNIX System V Releases 1 and 2 with source from 4.2/4.3 BSD; by its developers' account the original source was about a million lines of code, shipped on eight 1.2 MB floppies. The RT was built on the **IBM ROMP** — the first commercial RISC chip, derived from IBM Research's 801.

## History in Brief

- **RT PC era (1986–87):** The first AIX (sometimes "AIX/RT") ran on the RT PC. A novel aspect was the **Virtual Resource Manager (VRM)** microkernel: keyboard, mouse, display, disks, and network were controlled by the microkernel, and you could hot-key between operating systems with Alt-Tab. Much of AIX v2's kernel was written in the **PL.8** language. AIX Version 2 followed in 1987.
- **RS/6000 and AIX v3 (1990):** AIX Version 3 ("AIX/6000") launched with the first POWER1-based RS/6000 models. It was **incompatible** with the RT PC's AIX v2 and dropped the PL.8 microkernel for a "purer" design. This platform was later renamed pSeries, then System p, and finally **Power Systems**. v3 introduced the journaling filesystem (JFS) and shared libraries (see below).
- **AIX v4 (1994):** Added symmetric multiprocessing and matured through the decade to 4.3.3 (1999); 4.1.3 (1995) made **CDE** the default GUI, replacing the older AIXwindows Desktop. A modified v4.1 shipped as the OS for Apple's Network Server line (AIX for Apple Network Servers) — not to be confused with Apple's earlier A/UX.
- **Other ports:** Over the years IBM produced AIX for the **PS/2** (AIX/386, by Locus Computing, 1988–95), **System/370 and ESA/390 mainframes** (AIX/370 in 1990, then AIX/ESA on OSF/1 in 1991), and a beta for **IA-64 (Itanium)** under **Project Monterey** (with SCO), which was discontinued in 2002 for lack of interest.
- **AIX 6 (2007):** Introduced Workload Partitions, Live Partition Mobility, and role-based access control.
- **AIX 7.1 (2010) onward:** Added Cluster Aware AIX and large-scale scalability; **7.2** (2015) brought live kernel updates; **7.3** (2021) requires POWER8 or newer.

## Why AIX Is Notable

AIX pioneered several ideas now common across Unix-like systems:

- **First journaling file system (JFS)** — avoids full filesystem consistency checks on every boot.
- **Shared libraries** — smaller binaries, less RAM and disk use, no forced static linking.
- **Virtualization and dynamic resource allocation** — processor, disk, and network virtualization, plus fractional (micro-partition) processor units.
- **Mainframe-derived reliability engineering** — availability and serviceability concepts brought down from IBM's mainframe designs.

## Version Lifecycle (Power releases)

| Version | Released | Notable additions |
|---------|----------|-------------------|
| 5L 5.1 | 2001 | 64-bit kernel (optional), JFS2, POWER4, "L" = Linux affinity |
| 5L 5.2 | 2002 | MPIO, iSCSI initiator, Dynamic LPAR participation |
| 5L 5.3 | 2004 | NFSv4, Virtual SCSI/Ethernet, SMT, Micro-Partitioning (POWER5) |
| 6.1 | 2007 | WPARs, Live Partition Mobility, RBAC, encrypting JFS2, ProbeVue |
| 7.1 | 2010 | Cluster Aware AIX, RBAC with domains, 256 cores/1024 threads per LPAR |
| 7.2 | 2015 | Live kernel update, flash filesystem caching, secure boot (POWER9) |
| 7.3 | 2021 | Requires POWER8+ |

## Core System-Management Concepts

Three ideas define day-to-day AIX administration and set it apart from Linux:

- **SMIT (System Management Interface Tool)** — a menu-driven interface (`smit`/`smitty`) that builds and runs the underlying commands. Pressing **F6** reveals the exact command; actions are logged to `smit.script` and `smit.log`. See the [SMIT cheatsheet](articles/aix-smit-cheatsheet.md).
- **ODM (Object Data Manager)** — a binary database of system configuration (devices, network, LVM, installed software, SMIT menus), roughly analogous to the Windows registry, managed with `odmget`/`odmadd`/etc. See the [ODM cheatsheet](articles/aix-odm-cheatsheet.md).
- **LVM (Logical Volume Manager)** — AIX's storage abstraction of volume groups, logical volumes, and physical volumes, which influenced other Unix LVM implementations. See the [LVM cheatsheet](articles/aix-lvm-cheatsheet.md).

The default shell is **ksh88**; the default GUI is **CDE**, with GNOME and KDE available through the free AIX Toolbox for Linux Applications.

## Where AIX Fits Today

AIX targets mission-critical enterprise workloads on IBM Power hardware — databases, ERP, and other systems that prize reliability, availability, and serviceability. It's typically deployed as LPARs managed by an [HMC](articles/aix-hmc-cheatsheet.md), with I/O virtualized through a [VIOS](articles/aix-vios-cheatsheet.md), and provisioned/patched over the network with [NIM](articles/aix-nim-cheatsheet.md).

## Related

- [AIX SMIT Cheatsheet](articles/aix-smit-cheatsheet.md) — the menu-driven admin tool
- [AIX ODM Cheatsheet](articles/aix-odm-cheatsheet.md) — the configuration database
- [AIX LVM Cheatsheet](articles/aix-lvm-cheatsheet.md) — storage: VGs, LVs, PVs
- [AIX Package Management](articles/aix-package-management-cheatsheet.md) and [Software Updates and Fixes](articles/aix-software-updates-fixes-cheatsheet.md) — filesets, installp, and patching
- [AIX NIM](articles/aix-nim-cheatsheet.md), [VIOS](articles/aix-vios-cheatsheet.md), and [HMC](articles/aix-hmc-cheatsheet.md) — install, virtualization, and hardware management
- [AIX Backup and Recovery](articles/aix-backup-recovery-cheatsheet.md), [Boot and Init](articles/aix-boot-init-cheatsheet.md), [Filesystems](articles/aix-filesystems-cheatsheet.md), [Devices and Hardware](articles/aix-devices-hardware-cheatsheet.md), [Users and Groups](articles/aix-users-groups-cheatsheet.md), [NFS](articles/aix-nfs-cheatsheet.md), [LDAP](articles/aix-ldap-cheatsheet.md), [Cron](articles/aix-cron-cheatsheet.md), [CDE/X11](articles/aix-cde-x11-cheatsheet.md), and [PowerVM Concepts](articles/aix-powervm-virtualization-concepts.md)

---

*Source: [IBM AIX — Wikipedia](https://en.wikipedia.org/wiki/IBM_AIX). Content was rephrased for compliance with licensing restrictions.*
