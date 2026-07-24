# RHEL Boot Modes and Troubleshooting

<img src="/articles/images/rhel-logo.svg" alt="RHEL Logo" width="200">

A guide to RHEL boot modes, rescue environments, and troubleshooting techniques for recovering from system failures.

---

## Runlevel 1

* Process `rc.sysinit` and `rc1.d` scripts
* `/` file system is mounted read-write
* Network is not activated

---

## Single-User Mode / Rescue Target

* Process only `rc.sysinit` (RHEL 5-6)
* Append `1` or `single` at the end of the kernel boot line
  * RHEL 7-10: `systemd.unit=rescue.target`, `1`, `s`, or `single`
* `/` file system is mounted read-write
* Local file systems are mounted and network is disabled
* Do not use single-user mode if your file system cannot be mounted successfully
* You cannot use single-user mode if the runlevel 1 configuration on your system is corrupted
* Switch to Rescue mode (RHEL 7-10):

```sh
systemctl rescue
```

> **Note:** On RHEL 8-10, the concept of runlevels is fully replaced by systemd targets. The `rescue.target` is the equivalent of single-user mode and provides a minimal environment with the root filesystem mounted read-write and essential services running.

---

## Emergency Mode

* Runs `sulogin` only
* `/etc/rc.d/rc.sysinit` and `/etc/rc1.d` are **not** run (RHEL 5-6)
* The root file system is mounted **read-only**
* Append `emergency` at the end of the kernel boot line
  * RHEL 7-10: `systemd.unit=emergency.target`, `emergency`, or `-b`
* Use `mount -o remount,rw /` to edit configuration files
* Switch to Emergency mode (RHEL 7-10):

```sh
systemctl emergency
```

> **Note:** On RHEL 8-10, `emergency.target` starts only the root filesystem (read-only) and a shell. No services, no network, no other filesystems. Useful when `/etc/fstab` is corrupted or a filesystem fails to mount.

---

## Skip init / rd.break

Use this when `/sbin/init` is corrupted or to reset the root password.

**RHEL 5-6:**

* Append `init=/bin/bash` to the kernel command line at boot
* The root filesystem is read-only — remount with:

```sh
mount -o remount,rw /
```

**RHEL 7-10:**

* Append `rd.break` to the end of the `linux` line in the GRUB2 editor
* The system stops just before handing control from initramfs to the real root filesystem
* The real filesystem is mounted read-only at `/sysroot` — remount with:

```sh
mount -o remount,rw /sysroot
```

* Alternatively, append `rw init=/sysroot/bin/sh` to bypass init entirely

**Common use cases:**

* Reset root password
* Check root filesystem integrity
* Fix corrupted `/etc/fstab`

---

## Reset Root Password (RHEL 7-10)

The procedure is identical on RHEL 7, 8, 9, and 10:

1. Reboot and interrupt the GRUB2 countdown by pressing any key
2. Press `e` to edit the default kernel entry
3. Find the `linux` line (called `linux16` on RHEL 7 BIOS systems) and append `rd.break` at the end
4. Optionally remove `rhgb quiet` for verbose boot output
5. Press `Ctrl+X` to boot

At the `switch_root:/#` prompt:

```sh
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel       # required for SELinux relabelling
exit
exit
```

> **Important:** The `touch /.autorelabel` step is mandatory on SELinux-enforcing systems. Without it, the new password hash in `/etc/shadow` will have the wrong SELinux context and login will be denied. Alternatively, use `restorecon -v /etc/shadow` for a faster targeted relabel.

---

## Booting to the Debug Shell (RHEL 7-10)

1. Add the following parameter at the end of the `linux` line (or `linux16` on RHEL 7 BIOS): `systemd.debug-shell` or `debug`
2. During the boot process, `systemd-debug-generator` will configure the debug shell on **TTY9**
3. Press `Ctrl+Alt+F9` to connect to the debug shell
   * If working with a virtual machine, sending this key combination requires support from the virtualization application (e.g., in Virtual Machine Manager: **Send Key → Ctrl+Alt+F9**)
4. To verify you are in the debug shell:

```sh
systemctl status $$
```

**Enabling persistently:**

The debug shell can be set to start on every boot:

```sh
systemctl enable debug-shell
```

> **Warning:** Permanently enabling the debug shell is a security risk because no authentication is required to use it. Disable it when the debugging session has ended.

---

## Dracut Emergency Shell (RHEL 8-10)

When the initramfs (dracut) cannot find or mount the root filesystem, it automatically drops into an emergency shell. This happens before systemd starts.

**Common causes:**

* Missing or corrupted LVM/disk
* Incorrect `root=` kernel parameter
* Broken LUKS encryption setup
* Missing storage drivers

**At the dracut shell:**

```sh
# List available block devices
lsblk

# Check LVM
lvm lvscan
lvm vgchange -ay

# Attempt to mount root manually
mount /dev/mapper/rhel-root /sysroot

# Exit to retry boot
exit
```

> **Note:** The dracut shell is distinct from the systemd `emergency.target`. Dracut runs earlier in the boot process, within the initramfs, before the real root filesystem is accessible.

---

## Install Kernel from Rescue Mode

When booted into rescue mode, install a new kernel and rebuild the GRUB configuration.

**RHEL 7:**

```sh
cd /mnt/install/repo/Packages
rpm -ivh --root=/mnt/sysimage kernel-3.10.0-514.el7.x86_64
chroot /mnt/sysimage
grub2-mkconfig -o /boot/grub2/grub.cfg
touch /.autorelabel    # required if using SELinux
```

**RHEL 8-10:**

```sh
chroot /mnt/sysimage
dnf install kernel
grub2-mkconfig -o /boot/grub2/grub.cfg
touch /.autorelabel    # required if using SELinux
exit
```

> **Note:** On RHEL 8+, `dnf` replaces `rpm -ivh` for kernel installation as it handles dependencies automatically. Ensure network is available in rescue mode or use the installation media as a local repository.

---

## Change the Verbosity of Debug Logs During Booting

### Temporary (single boot)

1. Remove the arguments `rhgb quiet` from the kernel boot line
2. Add `loglevel=7` and `systemd.log_level=debug` instead
3. Press `Ctrl+X` to boot
4. Run `dmesg` after the system is up to review boot messages

### Persistent

Edit the GRUB defaults file:

```sh
vi /etc/default/grub
```

Update the `GRUB_CMDLINE_LINUX` line:

```sh
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap loglevel=7 systemd.log_level=debug"
```

Rebuild the GRUB configuration:

```sh
grub2-mkconfig -o /boot/grub2/grub.cfg
```

> **RHEL 9.3+ note:** Starting with RHEL 9.3, GRUB uses the Boot Loader Specification (BLS) exclusively. Kernel parameters are stored in individual BLS snippet files under `/boot/loader/entries/` rather than in the monolithic `grub.cfg`. Use `grubby` to manage kernel parameters persistently:

```sh
# Add a parameter to all kernels
grubby --update-kernel=ALL --args="loglevel=7 systemd.log_level=debug"

# Remove a parameter from all kernels
grubby --update-kernel=ALL --remove-args="rhgb quiet"

# Show current parameters for the default kernel
grubby --info=DEFAULT
```

---

## GRUB2 Differences Across RHEL Versions

| Feature | RHEL 7 | RHEL 8 | RHEL 9 | RHEL 10 |
|---------|--------|--------|--------|---------|
| GRUB version | 2.02 | 2.02 | 2.06 | 2.06+ |
| Kernel line name (BIOS) | `linux16` | `linux` | `linux` | `linux` |
| Kernel line name (UEFI) | `linuxefi` | `linux` | `linux` | `linux` |
| Boot entry format | `grub.cfg` | BLS + `grubenv` | BLS only (9.3+) | BLS only |
| Persistent params tool | `grub2-mkconfig` | `grubby` / `grub2-mkconfig` | `grubby` | `grubby` |
| Params file | `/etc/default/grub` | `/etc/default/grub` + `grubenv` | BLS entries in `/boot/loader/entries/` | BLS entries in `/boot/loader/entries/` |

> **Key change in RHEL 8+:** The `linux16` and `initrd16` directives used on RHEL 7 BIOS systems are replaced by `linux` and `initrd`. When editing the GRUB boot entry, always look for the line starting with `linux` regardless of firmware type.

---

## Summary: Boot Interrupt Methods by RHEL Version

| Method | RHEL 5-6 | RHEL 7 | RHEL 8-10 |
|--------|----------|--------|-----------|
| Single-user / Rescue | `1` or `single` | `systemd.unit=rescue.target` | `systemd.unit=rescue.target` |
| Emergency mode | `emergency` | `systemd.unit=emergency.target` | `systemd.unit=emergency.target` |
| Skip init / Break | `init=/bin/bash` | `rd.break` | `rd.break` |
| Debug shell | N/A | `systemd.debug-shell` | `systemd.debug-shell` |
| Root password reset | `init=/bin/bash` + `passwd` | `rd.break` + `chroot /sysroot` | `rd.break` + `chroot /sysroot` |
| GRUB line to edit | `kernel` | `linux16` (BIOS) / `linuxefi` (UEFI) | `linux` |
