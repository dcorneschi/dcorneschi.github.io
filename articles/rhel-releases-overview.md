# RHEL Releases Overview

<p align="center">
  <img src="images/rhel-logo.svg" alt="RHEL Logo" width="120">
</p>

A comprehensive overview of Red Hat Enterprise Linux major releases, their key features, and important changes between versions.

## Releases

| Version | Codename | Release Date | Kernel | Based On |
|---------|----------|-------------|--------|----------|
| RHEL 2.1 | Pensacola/Panama | Mar 2002 | 2.4.9  | Red Hat Linux 7.2 |
| RHEL 3  | Taroon   | Oct 2003    | 2.4.21 | Red Hat Linux 9 |
| RHEL 4  | Nahant   | Feb 2005    | 2.6.9  | Fedora Core 3 |
| RHEL 5  | Tikanga  | Mar 2007    | 2.6.18 | Fedora Core 6 |
| RHEL 6  | Santiago | Nov 2010    | 2.6.32 | Fedora 12/13 |
| RHEL 7  | Maipo    | Jun 2014    | 3.10   | Fedora 19/20 |
| RHEL 8  | Ootpa    | May 2019    | 4.18   | Fedora 28 |
| RHEL 9  | Plow     | May 2022    | 5.14   | Fedora 34 / CentOS Stream 9 |
| RHEL 10 | Coughlan | May 2025    | 6.12   | CentOS Stream 10 |

---

## Lifecycle and End of Support

RHEL versions 8, 9, and 10 deliver a **10-year lifecycle** across three production phases: Full Support, Maintenance Support, and Extended Life Phase. With Extended Life Cycle (ELC) add-ons, customers can receive errata for up to 14 years and beyond.

| Version | Full Support Ends | Maintenance Ends | End of Life |
|---------|-------------------|-----------------|-------------|
| RHEL 3  | Oct 2006          | Oct 2010        | Jan 2014 (ELS) |
| RHEL 4  | Feb 2009          | Feb 2012        | Mar 2017 (ELS) |
| RHEL 5  | Mar 2013          | Mar 2017        | Nov 2020 (ELS) |
| RHEL 6  | May 2016          | Nov 2020        | Jun 2024 (ELS) |
| RHEL 7  | Aug 2019          | Jun 2024        | Jun 2028 (ELS) |
| RHEL 8  | May 2024          | May 2029        | May 2029+ (ELC) |
| RHEL 9  | May 2027          | May 2032        | May 2032+ (ELC) |
| RHEL 10 | May 2030          | May 2035        | May 2035+ (ELC) |

> **Note:** Starting with RHEL 9, Red Hat unified extended support under Extended Life Cycle (ELC), replacing the older EUS/E4S/ELS offerings with a consistent 6-year support window for eligible even-numbered minor releases.

**Reference:** [Red Hat Enterprise Linux Life Cycle](https://access.redhat.com/support/policy/updates/errata)

---

## In-Place Upgrade Paths (Leapp)

Red Hat supports in-place upgrades between consecutive major versions using the `leapp` utility. Skipping major versions is not supported — you must upgrade sequentially.

| Source | Target | Notes |
|--------|--------|-------|
| RHEL 7.9 | RHEL 8.10 | Last supported path from RHEL 7 |
| RHEL 8.8, 8.10 | RHEL 9.2, 9.4 | Multiple minor version combinations supported |
| RHEL 9.4, 9.6 | RHEL 10.0, 10.2 | Latest supported paths |

**Key points:**
* Direct upgrades across multiple major versions (e.g., RHEL 7 → RHEL 9) are **not supported**. You must perform sequential upgrades (7 → 8 → 9).
* CentOS Stream 9 can be directly converted and upgraded to RHEL 10 using `convert2rhel` + `leapp`.
* Always run `leapp preupgrade` first to identify potential issues before committing.

**Reference:** [Supported Upgrade Paths](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/upgrading_from_rhel_9_to_rhel_10/supported-upgrade-paths)

---

## New stuff in RHEL 6

* ext4 is the default file system
* UUID is used by default in the `/etc/fstab` file
* Upstart has replaced SysV init
* Mounting via NFS defaults to NFS 4
* In addition to `/etc/sysctl.conf`, there is also `/etc/sysctl.d/` directory. Instead of modifying `/etc/sysctl.conf` directly, files can be placed under `/etc/sysctl.d/`
* `/etc/modprobe.d/` directory instead of `/etc/modprobe.conf`

---

## New stuff in RHEL 7

* Based on Fedora 19/20, upstream kernel version 3.10
* XFS is the new default filesystem (scale up to 500 TB)
* ext4 scales to 50 TB
* CRC32 checksums for XFS and ext4 reduce file system repair times

In Red Hat Enterprise Linux 7, four directories in `/` now have identical contents as their counterparts located in `/usr`:

* `/bin` and `/usr/bin`
* `/sbin` and `/usr/sbin`
* `/lib` and `/usr/lib`
* `/lib64` and `/usr/lib64`

In older versions of Red Hat Enterprise Linux, these were distinct directories containing different sets of files. In RHEL 7, the directories in `/` are symbolic links to the matching directories in `/usr`.

### System and command changes between RHEL 6 and RHEL 7

Between RHEL 6 and RHEL 7 there are a number of changes to tools, commands, and workflows. Changes likely to affect common administrative tasks:

* Anaconda RHEL installer completely redesigned
* Legacy GRUB boot loader replaced by GRUB2
* Procedure for bypassing root password prompt at boot completely different
* SysV init system and all related tools replaced by **systemd**
* ext4 replaced by **XFS** as default filesystem type
* Directories `/bin`, `/sbin`, `/lib` and `/lib64` are now all under `/usr`
* Network interfaces have a new naming scheme based on physical device location (e.g., `eth0` might become `enp0s3`)
* `ntpd` replaced by **chronyd** as the default network time protocol daemon
* GNOME 2 replaced by **GNOME 3** as default desktop environment
* System registration and subscription now handled exclusively with Red Hat Subscription Management (RHSM)
* MySQL replaced by **MariaDB**
* `tgtd` replaced by **targetcli**
* High Availability Add-On: RGManager removed in favor of Pacemaker; all CMAN features merged into Corosync (`qdiskd` replaced by votequorum plugin); all tools unified into `pcs`
* `ifconfig` and `route` further deprecated in favor of `ip`
* `netstat` further deprecated in favor of `ss`
* System user UID range extended from 0–499 to 0–999
* `locate` no longer available by default (available as `mlocate` package)
* `nc` (netcat) replaced by `nmap-ncat`

### XFS

XFS is the RHEL 7 default file system. Key benefits:

* Ability to manage up to 500 TB file systems with files up to 50 TB in size
* Best performance for most workloads (especially with high speed storage and larger number of cores)
* Less CPU intensive than most other file systems (better optimizations around lock contention)
* The most robust at large scale (has been run at 100+ TB sizes for many years)
* The most common file system in multiple key upstream communities (Ceph, Gluster, OpenStack)
* Pioneered most of the techniques now in ext4 for performance (like delayed allocation)
* No file system check at boot time
* CRC checksums on all metadata blocks

---

## New stuff in RHEL 8

* Based on Fedora 28, upstream kernel version 4.18, systemd 239, and GNOME 3.28
* The Cockpit web console is available by default
* New version of YUM (v4) based on DNF technology. Compatible with YUM v3 (RHEL 7)
* RPM v4.14 — RPM now validates the whole package contents before starting the installation
* Content is distributed through two main repositories: **BaseOS** and **Application Stream (AppStream)**
* Supports up to 4 PB of physical memory
* **Wayland** is the default display server instead of Xorg
* XFS now supports shared copy-on-write data extents
* **nftables** replaces iptables as the default network filtering framework
* Python 3.6 is the default Python version
* PHP 7.2 included
* Nginx 1.14 available in core repository

---

## New stuff in RHEL 9

* Based on Fedora 34, upstream kernel version 5.14
* systemd 249 (vs 239 in RHEL 8)
* Built with GCC 11 and the latest versions of LLVM, Rust, and Go compilers
* Based on glibc 2.34 for long-term platform stability
* **Link Time Optimization (LTO)** enabled by default in userspace for the first time
* **Python 3.9** is the default Python version for the life of RHEL 9
* bash 5.1.8 (vs 4.4 in RHEL 8)
* **GNOME 40** with Wayland as the default display server
* **PipeWire** replaces PulseAudio as the default audio server
* All application packaging methods (modules, SCLs, Flatpaks, traditional RPMs) incorporated into Application Streams
* Container improvements: automatic container updates and rollbacks with Podman for edge deployments
* Image Builder as-a-Service for building standardized OS images via a hosted service
* **Integrity Measurement Architecture (IMA)** to dynamically verify OS integrity
* Kernel live patch management available via the web console
* Red Hat Insights enhancements: Resource Optimization and Malware Detection
* ext4 supports timestamps beyond the year 2038
* `ansible-core` 2.12 replaces `ansible-engine` (delivered as AppStream)
* Support for multiple architectures: x86_64, aarch64, IBM POWER9, Power10, IBM Z

### Key changes from RHEL 8 to RHEL 9

* Network configuration files moved from `/etc/sysconfig/network-scripts/ifcfg-*` to NetworkManager keyfiles (`/etc/NetworkManager/system-connections/`). The `network-scripts` package is fully removed.
* OpenSSL updated to 3.0
* SSH root login with password disabled by default
* OpenSSH SCP protocol deprecated — `scp` now uses SFTP by default (use `-O` flag to restore old behavior)
* **SHA-1** signed packages blocked by default in the DEFAULT crypto policy
* SELinux can no longer be disabled via `/etc/sysconfig/selinux` — requires kernel parameter `selinux=0`
* `tuned` no longer installed by default (must be added manually)
* `teamd` deprecated — bonding is the preferred method for NIC teaming
* `iptables` deprecated — `nftables` is the only firewall framework
* `mailx` replaced by `s-nail`
* `redhat-support-tool` removed
* `abrtd` (crash reporting daemon) removed
* GRUB menu hidden by default if previous boot was successful (disable with `grub2-editenv - unset menu_auto_hide`)

**References:**
* [RHEL 9 Networking: Say Goodbye to ifcfg Files](https://www.redhat.com/en/blog/rhel-9-networking-say-goodbye-ifcfg-files-and-hello-keyfiles)

---

## New stuff in RHEL 10

* Based on CentOS Stream 10, upstream kernel version 6.12
* Support commitments extend to 2035
* **RHEL Lightspeed** — AI-powered command line assistant for troubleshooting and guidance
* **Image mode for RHEL** (bootc) — deploy the OS as a bootable container image using Podman and Containerfiles
* **Post-quantum cryptography** — new quantum-resistant algorithms for key exchange
* **Encrypted DNS** support for secure domain name resolution
* Streamlined **FIPS validation** — separates CVE remediation from certificate validation
* **Podman 5** with quadlet pod support and enhanced system roles
* **RISC-V** developer preview (on SiFive HiFive P550 board)
* Cockpit web console enhancements: remote system management, built-in text editor, Stratis filesystem limits, HA Add-On management
* No X.Org Server — **Wayland only**
* Python 3.12 is the default Python version
* PHP 8.3, NGINX 1.26, Git 2.47, MySQL 8.4, Maven 3.9
* Windows Subsystem for Linux (WSL) support for running RHEL dev environments
* AIDE (Advanced Intrusion Detection Environment) system role
* Hardware security module support (technology preview)
* Red Hat Insights advisor available in Red Hat Satellite for disconnected environments

### Key changes from RHEL 9 to RHEL 10

* CNI network provider removed for containers — Netavark is the only option
* Image mode (bootc) is a first-class deployment method alongside traditional package mode
* RHEL extensions repository for trusted developer tools and libraries
* Kernel updated from 5.14 to 6.12
