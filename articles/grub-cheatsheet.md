# GRUB2 Cheatsheet

GRUB2 (GRand Unified Bootloader) manages the boot process on RHEL 7+ and Ubuntu systems. Configuration changes are made through `/etc/default/grub` and rebuilt with `grub2-mkconfig` — **never edit `/boot/grub2/grub.cfg` by hand**.

## Key Rule

> **Do NOT edit `/boot/grub2/grub.cfg` directly.** It's auto-generated. Edit `/etc/default/grub` or use `grubby` to make persistent changes.

## Configuration Files

| File | Purpose |
|------|---------|
| `/etc/default/grub` | Main settings (timeout, kernel args, defaults) |
| `/etc/grub.d/` | Script fragments that generate `grub.cfg` |
| `/boot/grub2/grub.cfg` | Auto-generated config (BIOS — RHEL) |
| `/boot/efi/EFI/redhat/grub.cfg` | Auto-generated config (UEFI — RHEL) |
| `/boot/grub/grub.cfg` | Auto-generated config (Ubuntu) |
| `/etc/grub2.cfg` | Symlink to `/boot/grub2/grub.cfg` (RHEL) |
| `/boot/grub2/grubenv` | Saved environment (default entry, boot_success) |
| `/etc/sysconfig/kernel` | Default kernel settings (RHEL) |
| `/boot/loader/entries/*.conf` | BLS entries (RHEL 8+) |

## List Kernel Entries

```bash
# grubby (preferred — works on all RHEL versions)
grubby --info=ALL

# Show just titles with index numbers
grep ^menuentry /boot/grub2/grub.cfg | cut -f 2 -d \' | nl -v 0

# Alternative (awk)
grep ^menuentry /etc/grub2.cfg | awk -F'--' '{print $1}'

# RHEL 8+ (BLS — Boot Loader Specification)
ls /boot/loader/entries/
cat /boot/loader/entries/*.conf

# Ubuntu
grep menuentry /boot/grub/grub.cfg | grep -v submenu
```

## Default Boot Entry

```bash
# Show current default index
grubby --default-index

# Show current default kernel path
grubby --default-kernel

# Show default title
grubby --default-title

# Show saved entry (from grubenv)
grub2-editenv list

# Set default by index
grubby --set-default-index 0
# or
grub2-set-default 0

# Set default by kernel path
grubby --set-default /boot/vmlinuz-5.14.0-362.el9.x86_64

# Set default by title (Ubuntu)
sudo grub-set-default "Ubuntu, with Linux 5.15.0-91-generic"

# Boot a specific entry ONCE (next boot only, then revert)
grub2-reboot 2
# Ubuntu:
sudo grub-reboot 2
```

## Kernel Arguments (grubby)

### Add Arguments

```bash
# Add argument to a specific kernel
grubby --args="rhgb" --update-kernel=/boot/vmlinuz-5.14.0-362.el9.x86_64

# Add multiple arguments
grubby --args="rhgb quiet" --update-kernel=/boot/vmlinuz-5.14.0-362.el9.x86_64

# Add argument to ALL kernels
grubby --args="rdblacklist=lpfc" --update-kernel=ALL

# Add argument to the default kernel
grubby --args="crashkernel=auto" --update-kernel=DEFAULT
```

### Remove Arguments

```bash
# Remove argument from specific kernel
grubby --remove-args="rhgb quiet" --update-kernel=/boot/vmlinuz-5.14.0-362.el9.x86_64

# Remove from ALL kernels
grubby --remove-args="rdblacklist=lpfc" --update-kernel=ALL

# Remove multiple arguments at once
grubby --remove-args="rhgb quiet LANG=en_US.UTF-8" --update-kernel=/boot/vmlinuz-5.14.0-362.el9.x86_64
```

### View Kernel Info

```bash
# Show info for specific kernel
grubby --info=/boot/vmlinuz-5.14.0-362.el9.x86_64

# Show info for all kernels
grubby --info=ALL

# Show current kernel's boot arguments
cat /proc/cmdline
```

## Rebuild grub.cfg

After editing `/etc/default/grub`, regenerate the config:

```bash
# ALWAYS backup first
sudo cp /boot/grub2/grub.cfg /boot/grub2/grub.cfg.bak

# RHEL (BIOS)
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# RHEL (UEFI)
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg

# Ubuntu (either BIOS or UEFI)
sudo update-grub
# (shortcut for: grub-mkconfig -o /boot/grub/grub.cfg)
```

## /etc/default/grub

```bash
# Common settings
GRUB_TIMEOUT=5                    # Seconds before auto-boot (0 = no menu)
GRUB_TIMEOUT_STYLE=menu           # menu, hidden, countdown
GRUB_DEFAULT=saved                # Boot last-saved entry (with grub2-set-default)
GRUB_SAVEDEFAULT=true             # Remember last-booted entry
GRUB_CMDLINE_LINUX="crashkernel=auto rhgb quiet"    # Args for ALL kernels
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"           # Args for default kernel only (Ubuntu)
GRUB_DISABLE_RECOVERY=false       # Show recovery entries
GRUB_DISABLE_SUBMENU=true         # Flatten menu (no submenus)
GRUB_TERMINAL_OUTPUT=console      # Output device (console, serial, gfxterm)
```

### Common Changes

```bash
# Disable splash/quiet (see boot messages)
sudo sed -i 's/rhgb quiet//' /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Set timeout to 10 seconds
sudo sed -i 's/GRUB_TIMEOUT=.*/GRUB_TIMEOUT=10/' /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Add kernel argument permanently (alternative to grubby)
# Edit GRUB_CMDLINE_LINUX in /etc/default/grub, then rebuild
sudo vi /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Enable serial console
echo 'GRUB_TERMINAL="serial console"' >> /etc/default/grub
echo 'GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"' >> /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

## Install / Reinstall GRUB

```bash
# RHEL (BIOS)
sudo grub2-install /dev/sda

# RHEL (UEFI)
sudo dnf reinstall grub2-efi-x64 shim-x64
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg

# Ubuntu (BIOS)
sudo grub-install /dev/sda
sudo update-grub

# Ubuntu (UEFI)
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi
sudo update-grub

# Reinstall from chroot (rescue mode)
mount /dev/sda2 /mnt
mount /dev/sda1 /mnt/boot
for fs in dev dev/pts proc sys run; do mount --bind /$fs /mnt/$fs; done
chroot /mnt
grub2-install /dev/sda
grub2-mkconfig -o /boot/grub2/grub.cfg
exit
umount -R /mnt
```

## GRUB Password Protection

```bash
# Generate password hash
grub2-setpassword
# Or manually:
grub2-mkpasswd-pbkdf2

# Creates /boot/grub2/user.cfg with:
# GRUB2_PASSWORD=grub.pbkdf2.sha512.10000.<hash>

# This protects editing boot entries (press 'e')
# To also protect booting, edit /etc/grub.d/10_linux
```

## Boot Troubleshooting

### Edit at Boot Time (Emergency)

1. At GRUB menu, press `e` to edit
2. Find the `linux` or `linux16` or `linuxefi` line
3. Add parameters at the end:
   - `rd.break` — drop to initramfs shell (before root mount)
   - `init=/bin/bash` — root shell (bypass systemd)
   - `systemd.unit=rescue.target` — single-user mode
   - `systemd.unit=emergency.target` — minimal emergency mode
   - `1` or `single` — legacy single-user mode
4. Press `Ctrl+X` or `F10` to boot

### Reset Root Password (RHEL 7+)

```bash
# 1. At GRUB menu, press 'e'
# 2. Add rd.break to end of linux line
# 3. Press Ctrl+X to boot
# 4. At initramfs shell:
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel    # Fix SELinux labels
exit
exit
# System reboots with new password
```

### Reset Root Password (Ubuntu)

```bash
# 1. At GRUB menu, select "Advanced options" → recovery mode
# 2. Select "root - Drop to root shell prompt"
# 3. Remount root read-write:
mount -o remount,rw /
passwd root
reboot
```

### Fix Broken GRUB

```bash
# Boot from rescue/live USB, then:
sudo mount /dev/sda2 /mnt
sudo mount /dev/sda1 /mnt/boot
# (UEFI: mount /dev/sda1 /mnt/boot/efi)
for fs in dev dev/pts proc sys run; do sudo mount --bind /$fs /mnt/$fs; done
sudo chroot /mnt

# Reinstall
grub2-install /dev/sda              # BIOS
grub2-mkconfig -o /boot/grub2/grub.cfg

# Exit and reboot
exit
sudo umount -R /mnt
reboot
```

## RHEL 8+ Boot Loader Specification (BLS)

RHEL 8+ uses BLS entries instead of parsing `grub.cfg` directly:

```bash
# List BLS entries
ls /boot/loader/entries/

# View an entry
cat /boot/loader/entries/*$(uname -r).conf

# Example entry:
# title Red Hat Enterprise Linux (5.14.0-362.el9.x86_64)
# version 5.14.0-362.el9.x86_64
# linux /vmlinuz-5.14.0-362.el9.x86_64
# initrd /initramfs-5.14.0-362.el9.x86_64.img
# options root=/dev/mapper/rhel-root ro crashkernel=auto rhgb quiet

# Edit kernel options for a specific entry
sudo grubby --args="new_param" --update-kernel=/boot/vmlinuz-5.14.0-362.el9.x86_64

# Verify
grep initrd /boot/loader/entries/*
```

## /etc/grub.d/ Scripts

| Script | Purpose |
|--------|---------|
| `00_header` | General settings (timeout, default) |
| `01_users` | User authentication (password) |
| `10_linux` | Linux kernel entries |
| `20_linux_xen` | Xen hypervisor entries |
| `30_os-prober` | Other operating systems (dual-boot) |
| `40_custom` | Custom menu entries (add yours here) |
| `41_custom` | Additional custom entries |

### Add Custom Boot Entry

```bash
# Add to /etc/grub.d/40_custom
sudo vi /etc/grub.d/40_custom

# Example: add a memtest entry
menuentry "MemTest86+" {
    linux16 /boot/memtest86+
}

# Rebuild
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

## One-Liners

```bash
# Show current boot arguments
cat /proc/cmdline

# List all kernels with index numbers
grubby --info=ALL | grep -E "^index|^kernel|^title"

# Quick: show default kernel
grubby --default-kernel

# Quick: show default index
grubby --default-index

# Set newest kernel as default
grubby --set-default $(ls -t /boot/vmlinuz-* | head -1)

# Set oldest kernel as default (rollback)
grubby --set-default $(ls -t /boot/vmlinuz-* | tail -1)

# Add console output for debugging (all kernels)
grubby --args="console=ttyS0,115200n8 console=tty0" --update-kernel=ALL

# Remove splash/quiet for all kernels
grubby --remove-args="rhgb quiet" --update-kernel=ALL

# Show which kernel will boot next
grub2-editenv list

# Check GRUB is installed on correct device
sudo grub2-install --recheck /dev/sda

# Verify grub.cfg syntax (catches errors)
grub2-script-check /boot/grub2/grub.cfg

# Compare running kernel args vs grub config
echo "Running: $(cat /proc/cmdline)"
echo "Config:  $(grubby --info=$(grubby --default-kernel) | grep args)"

# Force next boot to specific entry (one-time)
sudo grub2-reboot 2

# Ubuntu: list kernels
dpkg --list | grep linux-image

# Ubuntu: auto-remove old kernels
sudo apt autoremove --purge
```

## Tips & Tricks

### Keep Old Kernels for Rollback

```bash
# RHEL — keep last 3 kernels (default is 3)
# Check current setting
grep installonly_limit /etc/dnf/dnf.conf
# installonly_limit=3

# Change to keep 5
echo "installonly_limit=5" >> /etc/dnf/dnf.conf

# Ubuntu — manage manually
dpkg --list | grep linux-image
sudo apt remove linux-image-5.15.0-OLD-generic
```

### Boot Specific Kernel Once (Testing)

```bash
# Set for next boot only (reverts to default after)
sudo grub2-reboot 2

# Useful for testing a new kernel — if it fails, reboot goes back to default
# Combine with: GRUB_DEFAULT=saved in /etc/default/grub
```

### Disable OS Prober (Single-OS Systems)

```bash
# Prevents scanning for other OSes (faster grub2-mkconfig)
echo 'GRUB_DISABLE_OS_PROBER=true' >> /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

### Hidden Menu (No Visual Delay)

```bash
# Hide menu, show on Shift/Esc press only
GRUB_TIMEOUT=0
GRUB_TIMEOUT_STYLE=hidden

# Or show countdown without full menu
GRUB_TIMEOUT=3
GRUB_TIMEOUT_STYLE=countdown
```

### Serial Console (Headless Servers)

```bash
# Add to /etc/default/grub
GRUB_TERMINAL="serial console"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
GRUB_CMDLINE_LINUX="console=ttyS0,115200n8 console=tty0"

sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

### Check if BIOS or UEFI

```bash
# If this directory exists, you're UEFI
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"

# Determine correct grub.cfg path
[ -d /sys/firmware/efi ] && echo "/boot/efi/EFI/redhat/grub.cfg" || echo "/boot/grub2/grub.cfg"
```

## RHEL vs Ubuntu Differences

| Feature | RHEL 7+ | Ubuntu |
|---------|---------|--------|
| Package | `grub2-*` | `grub-*` |
| Commands | `grub2-mkconfig`, `grub2-install` | `grub-mkconfig`, `grub-install` |
| Shortcut | — | `update-grub` |
| Config (BIOS) | `/boot/grub2/grub.cfg` | `/boot/grub/grub.cfg` |
| Config (UEFI) | `/boot/efi/EFI/redhat/grub.cfg` | `/boot/efi/EFI/ubuntu/grub.cfg` |
| Kernel args tool | `grubby` | Edit `/etc/default/grub` + `update-grub` |
| BLS entries | Yes (RHEL 8+) | No |
| Symlink | `/etc/grub2.cfg` | — |
| Password | `grub2-setpassword` | `grub-mkpasswd-pbkdf2` + manual config |
| Set default | `grubby --set-default-index N` | `grub-set-default N` |

## Quick Reference

```bash
# View
cat /proc/cmdline                  # Current boot args
grubby --info=ALL                  # All kernel entries
grubby --default-kernel            # Default kernel path
grubby --default-index             # Default entry number
grub2-editenv list                 # Saved environment

# Modify kernel args
grubby --args="param" --update-kernel=ALL           # Add to all
grubby --remove-args="param" --update-kernel=ALL    # Remove from all
grubby --args="param" --update-kernel=DEFAULT       # Add to default

# Set default
grubby --set-default-index N
grub2-set-default N

# Rebuild config (after editing /etc/default/grub)
grub2-mkconfig -o /boot/grub2/grub.cfg              # RHEL BIOS
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg    # RHEL UEFI
update-grub                                          # Ubuntu

# Install/reinstall
grub2-install /dev/sda                              # RHEL BIOS
grub-install /dev/sda                               # Ubuntu BIOS

# Emergency
# Press 'e' at GRUB menu → add rd.break or init=/bin/bash → Ctrl+X
```
