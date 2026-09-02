# HP-UX History, Versions, and Support Lifecycle

A reference to HP-UX from its 1980s origins to its end of standard support on **December 31, 2025** — the full version history, the dual version-numbering scheme, the hardware architectures each release ran on, and the end-of-life (EOL / end-of-support-life) dates for every 11i version. HP-UX is HP/HPE's proprietary UNIX, based on UNIX System V (initially System III), first released in 1984 and certified to The Open Group's UNIX 03 standard in its final form.

## The Two Numbering Schemes

HP-UX has been versioned two different ways over its life, and both appear in documentation, so it helps to hold both in mind:

- **Decimal scheme** (through 11.00) — a classic `major.minor` number: `9.05`, `10.20`, `11.00`.
- **11i marketing scheme** (from 11.11 onward) — HP rebranded releases as **11i vN** while keeping an underlying `B.11.xx` release string. The lowercase **i** was a marketing tag for "Internet-enabled." The result is a *dual* identity: every modern release has both a marketing name (`11i v3`) and a version string (`B.11.31`).

The version string is what `uname -r` prints (see [HP-UX System Information](articles/hpux-system-information.md)); the marketing name is what appears on media and in support documents.

| Marketing name | Version string | `uname -r` |
|----------------|----------------|------------|
| 11i v1 | B.11.11 | `B.11.11` |
| 11i v1.6 | B.11.22 | `B.11.22` |
| 11i v2 | B.11.23 | `B.11.23` |
| 11i v3 | B.11.31 | `B.11.31` |

## Early History (Decimal Releases)

HP-UX grew up alongside HP's hardware, moving across three processor families — Motorola 68000, HP's proprietary FOCUS, and finally PA-RISC — before the Itanium era.

| Release | Year | Highlights |
|---------|------|-----------|
| **1.0** | 1982 | First release, for HP 9000 Series 500 (layered on the Series-500 "SUN" OS, unrelated to Sun Microsystems) |
| **1.0** | 1984 | AT&T System III based; HP Integral PC (kernel ran from ROM) |
| **2.0** | 1984 | First release for HP's Motorola 68000-based workstations |
| **5.0** | 1985 | AT&T System V based; introduced the *Starbase* graphics API and HP Windows/9000 |
| **3.x** | 1988 | HP 9000 Series 600/800 only (developed in parallel with the 5.x/6.x workstation line) |
| **6.x** | 1988 | Series 300; added BSD sockets and, in 6.2, X11; introduced Context Dependent Files (CDF) |
| **7.x** | 1990 | Series 300/400/600/700/800; provided OSF/Motif; last with HP Windows/9000 |
| **8.x** | 1991 | Shared libraries introduced |
| **9.x** | 1992 | Introduced **SAM** (System Administration Manager) and **LVM** (in 9.00 for Series 800); adopted the Visual User Environment |
| **10.0** | 1995 | Converged workstation/server lines; SVR4-style file layout (`/opt`, `/etc/rc.config.d`, `/home`); introduced **Software Distributor (SD-UX)** |
| **10.10** | 1996 | Introduced the **Common Desktop Environment (CDE)**; UNIX95 compliance |
| **10.20** | 1996 | 64-bit **PA-RISC 2.0** support; PAM; VxFS allowed for root (boot kernel still on HFS); 32-bit UIDs/GIDs |
| **10.30** | 1997 | Developer release; first kernel-thread support (1:1 model) |
| **11.00** | 1997 | First **64-bit-addressing** release; ran 32-bit apps on 64-bit systems; SMP support; added Integrity groundwork |

Several ideas that define modern HP-UX arrived in this era: **LVM** and **SAM** in 9.x, the **SD-UX** packaging model and the SVR4 directory layout in 10.0, **CDE** in 10.10, and 64-bit computing in 10.20 (PA-RISC 2.0) and 11.00 (64-bit addressing). Note that CDFs, introduced in the 3.x/6.x era, were removed in 10.0 because of their security risks.

## The 11i Era (Modern Releases)

From 11.11 onward, HP-UX settled into the **11i vN** line and the PA-RISC → Itanium transition. `11i v1.5` (B.11.20) was the first Itanium-capable release; `11i v2` was the first to ship for **both** PA-RISC (HP 9000) and Itanium (HP Integrity) from one release.

| Marketing name | Version | Released | Architecture(s) | Notable |
|----------------|---------|----------|-----------------|---------|
| **11i v1** | B.11.11 | Dec 2000 | HP 9000 (PA-RISC) | The long-lived, widely deployed PA-RISC release |
| **11i v1.5** | B.11.20 | 2001 | Integrity (Itanium) | First Itanium-capable HP-UX |
| **11i v1.6** | B.11.22 | Jun 2002 | Integrity | Itanium interim release |
| **11i v2** | B.11.23 | Oct 2003 | HP 9000 **and** Integrity | Kernel intrusion detection, stack-overflow protection, RBAC |
| **11i v3** | B.11.31 | Feb 2007 | HP 9000 and Integrity | Agile device view, native multipathing, largest scale limits |

**11i v3** is the final and most capable release. On its largest supported hardware (a Superdome 2 with Itanium 9560 processors) it scales to 256 cores, 8 TB of memory, 32 TB filesystems, and 16 TB files. It introduced the **agile addressing** device model and native multipathing described in [HP-UX Device Management (ioscan)](articles/hpux-device-management-ioscan.md), and continued to receive twice-yearly Operating Environment Update Releases (OEURs) until 2025.

## Operating Environments (OEs)

Within a given 11i version, software is packaged as an **Operating Environment** — a tested, licensed bundle of the base OS plus a set of applications. Only one OE is installed at a time. The OE names differ between the v1/v2 generation and the v3 generation:

**11i v1 / v2 OEs:** Foundation OE (`HPUX11i-OE`), Enterprise OE (`HPUX11i-OE-Ent`), Mission Critical OE (`HPUX11i-OE-MC`), and Technical Computing OE (`HPUX11i-TCOE`).

**11i v3 OEs (from the March 2008 kit):** Base OE (`HPUX11i-BOE`), Virtual Server OE (`HPUX11i-VSE-OE`), High Availability OE (`HPUX11i-HA-OE`), and Data Center OE (`HPUX11i-DC-OE`).

For how OEs are selected at install time and updated with `update-ux`, see [HP-UX Installation and Ignite-UX](articles/hpux-installation-ignite.md). To check the installed OE on a running system:

```bash
swlist -l bundle "HPUX11i-*OE*"     # which OE bundle is installed
uname -r                            # base OS version string (e.g. B.11.31)
```

## Support Lifecycle (End of Life)

HP-UX support is tracked per version and, for the releases that ran on two architectures, **per architecture** — the PA-RISC (HP 9000) and Integrity (Itanium) variants of the same version reached end of support on different dates. The following are the end-of-standard-support (EOL / end-of-support-life) dates:

| Version | Released | End of support | Hardware |
|---------|----------|----------------|----------|
| 11i v1 (B.11.11) | 2000-12-01 | 2015-12-31 | HP 9000 (PA-RISC) |
| 11i v1.6 (B.11.22) | 2002-06-02 | (Itanium interim; superseded) | Integrity |
| 11i v2 (B.11.23) | 2003-10-01 | 2015-12-31 | HP 9000 and Integrity |
| 11i v3 (B.11.31) | 2007-02-01 | **2021-03-31** | HP 9000 (PA-RISC) |
| 11i v3 (B.11.31) | 2007-02-01 | **2025-12-31** | Integrity (Itanium) |

Earlier decimal releases (10.20 and prior) were already obsolete long before this, with HP support for 10.20 ending on 2003-06-30.

Key dates to remember:

- **2015-12-31** — HP-UX 11i **v1 and v2** left standard support.
- **2021-03-31** — the **PA-RISC** (HP 9000) variant of 11i v3 left support.
- **2025-12-31** — the **Integrity** (Itanium) variant of 11i v3, the last supported HP-UX anywhere, left standard support. This coincides with the end of the Itanium era.

The final HP-UX release was **11i v3 release 2505** (B.11.31, the May 2025 OEUR) — the last twice-yearly Operating Environment Update Release before support ended. After end of standard support, HPE offers limited "mature"/custom support on Integrity for a further period, but no new development or Sustaining Engineering.

## Why HP-UX Ended

HP-UX's fate was tied to **Itanium**. After the PA-RISC → Itanium transition (11i v1.5 through v3), HP-UX ran only on Intel Itanium-based HPE Integrity servers. Intel wound down Itanium — shipping its final Itanium 9700-series CPUs in 2021 — and with no future processor to run on, HP-UX's roadmap ended with the twice-yearly 11i v3 updates and the 2025-12-31 support cutoff. There is no HP-UX 11i v4; the OS was never ported off Itanium.

## Related Articles

- [HP-UX System Information and Initial Configuration](articles/hpux-system-information.md) — identifying the running version and OE
- [HP-UX Installation and Ignite-UX](articles/hpux-installation-ignite.md) — OEs, media kits, and `update-ux`
- [HP-UX SD-UX Software Structure, IPD, and swlist](articles/hpux-swlist-software-structure.md) — the software packaging model introduced in 10.0
- [HP-UX Boot Process (PA-RISC and Integrity)](articles/hpux-boot-process.md) — the two firmware architectures across the PA-RISC/Itanium split
- [HP-UX Device Management (ioscan)](articles/hpux-device-management-ioscan.md) — the agile addressing model introduced in 11i v3
- [HP-UX Patch Management](articles/hpux-patch-management.md) — Quality Pack bundles and OEURs
