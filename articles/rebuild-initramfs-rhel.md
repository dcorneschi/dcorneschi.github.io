# How to Rebuild the initramfs / initrd in RHEL

<cite index="1-8">When adding new hardware to a system, or after changing configuration files that may be used very early in the boot process, or when changing the options on a kernel module, it may be necessary to rebuild the initial ramdisk (also known as initrd or initramfs) to include the proper kernel modules, files, and configuration directives.</cite>

Common reasons to rebuild:
- Added or changed a hardware driver (storage controller, NIC)
- Modified `/etc/lvm/lvm.conf` with root on LVM
- Modified `/etc/multipath.conf` with root on multipath
- Changed module options in `/etc/modprobe.d/`
- Changed `/etc/crypttab` (LUKS encryption)
- Updated firmware files needed at boot
- Kernel upgrade failed or produced a broken initramfs

## Rebuilding the initramfs (RHEL 6, 7, 8, 9, 10)

<cite index="1-20">If you are in a kernel version different to the initramfs you are building (REQUIRED if you are in Rescue Mode) you must specify the full kernel version, including architecture.</cite> <cite index="1-21">It is also recommended you make a backup copy of the initramfs in case the new version has an unexpected problem.</cite>

### Full Method (Explicit Kernel Version — Safest)

```bash
# 1. Make a backup
cp /boot/initramfs-<kernelVersion>.img /boot/initramfs-<kernelVersion>.img.bak

# 2. Rebuild initramfs for the specified kernel
dracut -f /boot/initramfs-<kernelVersion>.img <kernelVersion>
```

Replace `<kernelVersion>` with the full kernel version string including architecture.

**Example** (for kernel `4.18.0-305.19.1.el8_4.x86_64`):

```bash
cp /boot/initramfs-4.18.0-305.19.1.el8_4.x86_64.img \
   /boot/initramfs-4.18.0-305.19.1.el8_4.x86_64.bak.$(date +%m-%d-%H%M%S).img

dracut -f /boot/initramfs-4.18.0-305.19.1.el8_4.x86_64.img 4.18.0-305.19.1.el8_4.x86_64
```

### Shortcut Method (Current Running Kernel)

<cite index="1-12">In these examples you will see the usage of `$(uname -r)`, which is a way to pass the current kernel version into a command without actually typing it out.</cite>

> <cite index="1-22,1-23,1-24">**Caution!** YOU MUST BE 100% CERTAIN THAT YOU ARE BOOTED TO THE CORRECT VERSION OR YOU MAY CAUSE ADDITIONAL DAMAGE TO THE SYSTEM. IF YOU'RE NOT CERTAIN, THEN USE THE ABOVE METHOD.</cite>

```bash
# Make backup
cp /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).bak.$(date +%m-%d-%H%M%S).img

# Rebuild initramfs
dracut -f -v
```

The `-f` flag forces an overwrite of the existing image. The `-v` flag enables verbose output showing what gets included.

## Verify the Rebuild

### RHEL 8, 9, 10 (BLS — Boot Loader Specification)

<cite index="1-26">In RHEL8 and 9, be certain that the BLS configuration files in `/boot/loader/entries` includes the menu to the newly installed or created custom initramfs.</cite>

```bash
grep initrd /boot/loader/entries/*
```

Example output:

```
/boot/loader/entries/f81f518a...-4.18.0-240.el8.x86_64.conf:initrd /initramfs-4.18.0-240.el8.x86_64.img $tuned_initrd
/boot/loader/entries/f81f518a...-0-rescue.conf:initrd /initramfs-0-rescue-f81f518a....img
```

### RHEL 7

<cite index="1-27">In RHEL 7, be certain that the `/etc/grub2.cfg` and `/boot/grub2/grub.cfg` includes the menu to the newly installed or created custom initramfs.</cite>

```bash
# Check initrd entries in GRUB config
grep initrd /etc/grub2.cfg

# Check kernel menu entries
grep "menuentry " /boot/grub2/grub.cfg
```

<cite index="1-27,1-28">If the customized kernel menu entry does not appear in the grub configuration file(s), rebuild the grub menu. This rebuild is nominally performed by dracut, but can not be successfully completed in some corner cases.</cite>

```bash
# On BIOS-based machines
grub2-mkconfig -o /boot/grub2/grub.cfg

# On UEFI-based machines
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
```

## Rebuilding the initrd (RHEL 3, 4, 5 — Legacy)

These older versions use `mkinitrd` instead of `dracut`:

```bash
# Make a backup
cp /boot/initrd-$(uname -r).img /boot/initrd-$(uname -r).bak.$(date +%m-%d-%H%M%S).img

# Build the initrd
mkinitrd -f -v /boot/initrd-$(uname -r).img $(uname -r)
```

<cite index="1-30">The `-v` verbose flag causes mkinitrd to display the names of all the modules it is including in the initial ramdisk.</cite>

<cite index="1-31">The `-f` option will force an overwrite of any existing initial ramdisk image at the path you have specified.</cite>

<cite index="1-31">If you are in a kernel version different to the initrd you are building (including if you are in Rescue Mode) you must specify the full kernel version, without architecture:</cite>

```bash
mkinitrd -f -v /boot/initrd-2.6.18-348.2.1.el5.img 2.6.18-348.2.1.el5
```

## Working with Backups

<cite index="1-32,1-33">As mentioned previously, it is recommended that you take a backup of the previous initrd in case something goes wrong with the new one. If desired, it is possible to create a separate entry in `/boot/grub/grub.conf` for the backup initial ramdisk image, to conveniently choose the old version at boot time without needing to restore the backup.</cite>

### Dual GRUB Entry (RHEL 5/6 Legacy)

```
title Red Hat Enterprise Linux 5 (2.6.18-274.el5)
 root (hd0,0)
 kernel /vmlinuz-2.6.18-274.el5 ro root=LABEL=/
 initrd /initrd-2.6.18-274.el5.img

title Red Hat Enterprise Linux 5 w/ old initrd (2.6.18-274.el5)
 root (hd0,0)
 kernel /vmlinuz-2.6.18-274.el5 ro root=LABEL=/
 initrd /initrd-2.6.18-274.el5.bak.MM-DD-HHMMSS.img
```

### Temporary Boot with Old initrd (GRUB Edit Mode)

<cite index="1-35">If grub is secured with a password, press `p` and enter the password. Use the arrow keys to highlight the entry for the kernel you wish to boot. Press `e` for edit. Highlight the initrd line and press `e` again. Change the path for the initrd to the backup copy you made (such as `/initrd-2.6.18-274.el5.bak.01-01-123456.img`). Press Enter to temporarily save the changes you have made. Press `b` for boot.</cite>

> <cite index="1-36">Note: This procedure does not actually make the change persistent. The next boot will continue to use the original grub.conf configuration unless it is updated.</cite>

## Rebuild from Rescue Mode

When the system won't boot and you need to rebuild the initramfs from rescue media:

```bash
# 1. Boot from RHEL installation/rescue media
# 2. Select "Troubleshooting" → "Rescue a Red Hat Enterprise Linux system"
# 3. The rescue system will try to mount your root at /mnt/sysimage

# 4. Chroot into the installed system
chroot /mnt/sysimage

# 5. Verify kernel version (can't use uname -r in chroot — it shows rescue kernel)
ls /boot/vmlinuz-*
# Pick the version you need to rebuild

# 6. Rebuild (must specify full version — you're NOT running that kernel)
dracut -f /boot/initramfs-4.18.0-305.19.1.el8_4.x86_64.img 4.18.0-305.19.1.el8_4.x86_64

# 7. Exit chroot and reboot
exit
reboot
```

> **Important:** In rescue mode, `$(uname -r)` returns the rescue kernel version, NOT your installed kernel. Always specify the kernel version explicitly.

## Quick Reference by RHEL Version

| RHEL Version | Tool | Backup Command | Rebuild Command |
|-------------|------|----------------|-----------------|
| 10, 9, 8 | dracut | `cp /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak` | `dracut -f -v` |
| 7 | dracut | `cp /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak` | `dracut -f -v` |
| 6 | dracut | `cp /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak` | `dracut -f -v` |
| 5 | mkinitrd | `cp /boot/initrd-$(uname -r).img /boot/initrd-$(uname -r).img.bak` | `mkinitrd -f -v /boot/initrd-$(uname -r).img $(uname -r)` |
| 4 | mkinitrd | `cp /boot/initrd-$(uname -r).img /boot/initrd-$(uname -r).img.bak` | `mkinitrd -f -v /boot/initrd-$(uname -r).img $(uname -r)` |
| 3 | mkinitrd | `cp /boot/initrd-$(uname -r).img /boot/initrd-$(uname -r).img.bak` | `mkinitrd -f -v /boot/initrd-$(uname -r).img $(uname -r)` |

## Common dracut Options

| Option | Description |
|--------|-------------|
| `-f` / `--force` | Overwrite existing initramfs image |
| `-v` / `--verbose` | Show detailed output during build |
| `--add <module>` | Include a dracut module (e.g., `multipath`, `iscsi`) |
| `--add-drivers <driver>` | Include specific kernel modules (e.g., `megaraid_sas`) |
| `--omit <module>` | Exclude a dracut module (e.g., `plymouth`) |
| `--hostonly` | Build for this hardware only (smaller image) |
| `--no-hostonly` | Build generic image (all drivers — portable) |
| `--include <src> <dst>` | Include a file/directory in the initramfs |
| `--regenerate-all` | Rebuild initramfs for ALL installed kernels |
| `--kver <version>` | Specify kernel version (alternative to positional arg) |

## Examples

```bash
# Rebuild current kernel's initramfs (most common)
dracut -f -v

# Rebuild for a specific kernel version
dracut -f /boot/initramfs-5.14.0-362.el9.x86_64.img 5.14.0-362.el9.x86_64

# Rebuild all initramfs images
dracut --regenerate-all --force

# Include multipath support
dracut -f --add multipath

# Include multipath with full config directory
dracut -v -f -a multipath --include /etc/multipath /etc/multipath

# Include specific drivers
dracut -f --add-drivers "mpt3sas megaraid_sas"

# Include a custom config file
dracut -f --include /etc/multipath.conf /etc/multipath.conf

# Build a minimal (hostonly) image
dracut -f --hostonly

# Build a portable (generic) image for rescue
dracut -f --no-hostonly /boot/initramfs-$(uname -r)-generic.img

# Verify contents after rebuild
lsinitrd /boot/initramfs-$(uname -r).img | head -30
lsinitrd /boot/initramfs-$(uname -r).img | grep multipath
```

## Persistent dracut Configuration

Instead of passing options on every rebuild, make them permanent via `/etc/dracut.conf` or drop-in files in `/etc/dracut.conf.d/`:

### Add Multipath Permanently

Edit `/etc/dracut.conf` and modify:

```bash
# Find this line:
#add_dracutmodules+=""

# Change to:
add_dracutmodules+=" multipath "
```

Or create a drop-in (preferred):

```bash
echo 'add_dracutmodules+=" multipath "' | sudo tee /etc/dracut.conf.d/99-multipath.conf
```

Then rebuild:

```bash
dracut -v -f /boot/initramfs-$(uname -r).img $(uname -r)
```

### Add iSCSI Permanently

```bash
echo 'add_dracutmodules+=" iscsi "' | sudo tee /etc/dracut.conf.d/99-iscsi.conf
dracut -v -f
```

### Add Specific Drivers Permanently

```bash
echo 'add_drivers+=" megaraid_sas mpt3sas "' | sudo tee /etc/dracut.conf.d/99-drivers.conf
dracut -v -f
```

### Include Configuration Files Permanently

```bash
echo 'install_items+=" /etc/multipath.conf /etc/multipath/bindings "' | sudo tee /etc/dracut.conf.d/99-include.conf
dracut -v -f
```

> **Note:** After modifying `/etc/dracut.conf` or files in `/etc/dracut.conf.d/`, all future kernel installs will automatically include these settings when the initramfs is built.

## When to Rebuild

| Scenario | Action Required |
|----------|----------------|
| Changed `/etc/lvm/lvm.conf` (root on LVM) | Rebuild initramfs |
| Changed `/etc/multipath.conf` (root on multipath) | Rebuild initramfs |
| Changed module options in `/etc/modprobe.d/` | Rebuild initramfs |
| Changed `/etc/crypttab` (encrypted root) | Rebuild initramfs |
| Added new storage driver | Rebuild initramfs |
| Added firmware for boot device | Rebuild initramfs |
| Changed `/etc/dracut.conf.d/` settings | Rebuild initramfs |
| Normal kernel update via `dnf update` | **Automatic** (dracut runs in kernel RPM scriptlet) |
| Changed `/etc/fstab` (non-root entries) | Usually NOT needed |
| Changed network config (no network root) | Usually NOT needed |

## Troubleshooting

### Rebuild Fails in Rescue Mode

```bash
# If dracut fails, check:
# 1. Are /proc, /sys, /dev mounted in chroot?
mount -t proc proc /proc
mount -t sysfs sys /sys
mount --bind /dev /dev

# 2. Try chroot again
chroot /mnt/sysimage
dracut -f /boot/initramfs-<version>.img <version>
```

### System Doesn't Boot After Rebuild

```bash
# Boot into rescue or older kernel from GRUB menu
# Restore the backup:
cp /boot/initramfs-<version>.img.bak /boot/initramfs-<version>.img
reboot

# Or use GRUB edit mode (press 'e' at GRUB menu):
# Change the initrd line to point to the .bak file
# Press Ctrl+X to boot
```

### Verify initramfs Integrity

```bash
# Check if the file is valid
file /boot/initramfs-$(uname -r).img

# Check size (extremely small = likely corrupt)
ls -lh /boot/initramfs-$(uname -r).img

# List contents (should show many files)
lsinitrd /boot/initramfs-$(uname -r).img | wc -l

# Check for specific critical modules
lsinitrd /boot/initramfs-$(uname -r).img | grep -E "xfs|ext4|lvm|dm-mod"
```
