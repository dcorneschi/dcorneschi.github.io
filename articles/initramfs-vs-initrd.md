# initramfs vs initrd

Both `initramfs` and `initrd` provide an early root filesystem that the kernel loads into memory at boot — before the real root filesystem is available. They contain drivers, scripts, and tools needed to mount the actual root device (LVM, RAID, encrypted disks, network filesystems, etc.). Modern Linux uses initramfs exclusively, but understanding the differences explains why legacy documentation still references initrd.

## The Problem They Solve

The kernel needs drivers to access the root filesystem, but those drivers might live on the root filesystem itself. This chicken-and-egg problem is solved by packing essential drivers and mount logic into an archive loaded alongside the kernel at boot.

Without an initramfs/initrd:
- Kernel can't mount root on LVM, RAID, iSCSI, NFS, or encrypted volumes
- Kernel can't load filesystem modules (XFS, Btrfs) that aren't compiled in
- Kernel can't handle complex storage stacks (multipath, LUKS on LVM on RAID)

## Key Differences

| Feature | initrd (legacy) | initramfs (modern) |
|---------|----------------|-------------------|
| Format | Compressed filesystem image (ext2/ext3) | Compressed cpio archive |
| Kernel mechanism | Block device (ramdisk) | tmpfs (ramfs) |
| Memory management | Fixed size, wastes unused space | Grows/shrinks dynamically |
| Cache behavior | Double-cached (block layer + page cache) | Single-cached (page cache only) |
| Pivot mechanism | `pivot_root` then unmount old root | `switch_root` — deletes initramfs contents |
| Root device | `/dev/ram0` block device | Mounted as rootfs directly |
| When used | Kernel 2.4 and earlier (mostly) | Kernel 2.6+ (2004 onwards) |
| RHEL version | RHEL 4 and earlier | RHEL 5+ |
| File naming | `/boot/initrd-<version>.img` | `/boot/initramfs-<version>.img` |
| Build tool | `mkinitrd` | `dracut` (RHEL), `update-initramfs` (Ubuntu) |

## How initrd Works (Legacy)

1. Bootloader loads the initrd image into memory
2. Kernel decompresses it and writes it to a ramdisk block device (`/dev/ram0`)
3. Kernel mounts `/dev/ram0` as the temporary root filesystem
4. `/linuxrc` script runs — loads modules, discovers real root device
5. `pivot_root` switches to the real root filesystem
6. Old initrd is unmounted and freed

**Problems:**
- Ramdisk has a fixed size (set at creation) — wastes memory if too large, fails if too small
- Data passes through the block layer AND the page cache (double caching)
- Requires a filesystem driver (ext2) compiled into the kernel just to read the initrd
- `pivot_root` is complex and error-prone

## How initramfs Works (Modern)

1. Bootloader loads the initramfs cpio archive into memory
2. Kernel unpacks it directly into a tmpfs instance (rootfs)
3. Kernel executes `/init` (not `/linuxrc`)
4. Init script loads modules, assembles storage (LVM, RAID, LUKS), mounts real root
5. `switch_root` deletes all initramfs files, then moves the real root to `/`
6. PID 1 (`systemd` or `init`) starts from the real root

**Advantages:**
- No fixed size — uses only as much memory as the contents require
- No double caching — files live directly in the page cache
- No block device needed — simpler kernel code path
- `switch_root` is cleaner than `pivot_root`
- Can contain any file type (devices nodes, symlinks, FIFOs)

## File Naming

Despite the differences, many systems still use "initrd" in filenames for historical reasons:

| Distribution | Filename | Actual Format |
|-------------|----------|---------------|
| RHEL 7/8/9 | `/boot/initramfs-$(uname -r).img` | initramfs (cpio+gzip/xz) |
| Ubuntu 20.04+ | `/boot/initrd.img-$(uname -r)` | initramfs (cpio+gzip/lz4) |
| Debian | `/boot/initrd.img-$(uname -r)` | initramfs (cpio+gzip) |
| SUSE | `/boot/initrd-$(uname -r)` | initramfs (cpio+xz) |

> **Note:** Ubuntu/Debian name the file `initrd.img` but it IS an initramfs (cpio archive). The name is kept for compatibility with bootloader configs.

## Inspect initramfs Contents

### RHEL (dracut-based)

```bash
# List contents
lsinitrd /boot/initramfs-$(uname -r).img
lsinitrd    # Current running kernel's initramfs

# List only files (no metadata)
lsinitrd -f /boot/initramfs-$(uname -r).img

# Show specific file content
lsinitrd -f /etc/cmdline.d/00-default.conf

# List included modules
lsinitrd --list-modules
lsinitrd -m

# Show dracut modules included
lsinitrd | grep -E "^drwx|^-rwx" | head -20

# Extract to directory for inspection
mkdir /tmp/initramfs
cd /tmp/initramfs
/usr/lib/dracut/skipcpio /boot/initramfs-$(uname -r).img | zcat | cpio -idmv
# Or for xz-compressed:
/usr/lib/dracut/skipcpio /boot/initramfs-$(uname -r).img | xzcat | cpio -idmv
```

### Ubuntu (initramfs-tools-based)

```bash
# List contents
lsinitramfs /boot/initrd.img-$(uname -r)

# Extract to directory
mkdir /tmp/initramfs && cd /tmp/initramfs
unmkinitramfs /boot/initrd.img-$(uname -r) .

# Manual extraction (works on both RHEL and Ubuntu)
mkdir /tmp/initramfs && cd /tmp/initramfs
zcat /boot/initrd.img-$(uname -r) | cpio -idmv 2>/dev/null

# For lz4-compressed (Ubuntu 22.04+)
lz4cat /boot/initrd.img-$(uname -r) | cpio -idmv 2>/dev/null

# Check compression type
file /boot/initrd.img-$(uname -r)
```

### Generic Extraction (Any Distro)

```bash
# Determine compression and extract
IMG=/boot/initramfs-$(uname -r).img
mkdir /tmp/initramfs && cd /tmp/initramfs

case $(file -b "$IMG") in
  *gzip*)   zcat "$IMG" | cpio -idmv ;;
  *XZ*)     xzcat "$IMG" | cpio -idmv ;;
  *LZ4*)    lz4cat "$IMG" | cpio -idmv ;;
  *Zstandard*) zstdcat "$IMG" | cpio -idmv ;;
  *cpio*)   cpio -idmv < "$IMG" ;;
esac
```

> **Note:** Some initramfs images have a microcode prepended (early cpio for CPU microcode). Use `skipcpio` (RHEL) or skip the first cpio archive manually.

## Build / Rebuild initramfs

### RHEL — dracut

```bash
# Rebuild initramfs for the current kernel
sudo dracut --force

# Rebuild for a specific kernel version
sudo dracut --force /boot/initramfs-5.14.0-362.el9.x86_64.img 5.14.0-362.el9.x86_64

# Rebuild with verbose output
sudo dracut --force --verbose

# Add a specific module
sudo dracut --force --add "iscsi multipath"

# Add a specific kernel module (driver)
sudo dracut --force --add-drivers "megaraid_sas mpt3sas"

# Include a specific file
sudo dracut --force --include /etc/multipath.conf /etc/multipath.conf

# Omit a module
sudo dracut --force --omit "plymouth"

# Build a minimal initramfs (host-only — only drivers for this hardware)
sudo dracut --force --hostonly

# Build a generic initramfs (includes all drivers — rescue/portable)
sudo dracut --force --no-hostonly

# Rebuild all installed kernels
sudo dracut --regenerate-all --force

# Show what would be included (dry-run)
dracut --print-cmdline
```

### Ubuntu — update-initramfs

```bash
# Update initramfs for the current kernel
sudo update-initramfs -u

# Update for a specific kernel version
sudo update-initramfs -u -k 5.15.0-91-generic

# Create a new initramfs
sudo update-initramfs -c -k $(uname -r)

# Delete an initramfs
sudo update-initramfs -d -k 5.15.0-91-generic

# Update all kernels
sudo update-initramfs -u -k all

# Verbose output
sudo update-initramfs -u -v
```

### Manual cpio Method (Emergency)

```bash
# Create a minimal initramfs by hand (emergency recovery)
mkdir -p /tmp/myinitramfs/{bin,sbin,etc,proc,sys,dev,lib,lib64,mnt/root}

# Copy busybox (static) for basic utilities
cp /bin/busybox /tmp/myinitramfs/bin/
cd /tmp/myinitramfs/bin
for cmd in sh mount umount ls cat echo mkdir; do
    ln -s busybox $cmd
done

# Create init script
cat > /tmp/myinitramfs/init << 'EOF'
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
echo "Emergency shell — type 'exit' to continue boot"
/bin/sh
mount /dev/sda1 /mnt/root
exec switch_root /mnt/root /sbin/init
EOF
chmod +x /tmp/myinitramfs/init

# Pack it
cd /tmp/myinitramfs
find . | cpio -o -H newc | gzip > /boot/initramfs-emergency.img
```

## Configuration

### RHEL — dracut.conf

```bash
# Main config
/etc/dracut.conf

# Drop-in configs (preferred)
/etc/dracut.conf.d/*.conf

# Example: always include multipath and iscsi
cat <<'EOF' | sudo tee /etc/dracut.conf.d/99-storage.conf
add_dracutmodules+=" multipath iscsi "
add_drivers+=" dm-multipath mpt3sas "
EOF

# Example: force hostonly mode (smaller, hardware-specific)
cat <<'EOF' | sudo tee /etc/dracut.conf.d/99-hostonly.conf
hostonly="yes"
hostonly_cmdline="yes"
EOF

# Example: include firmware
cat <<'EOF' | sudo tee /etc/dracut.conf.d/99-firmware.conf
install_items+=" /lib/firmware/bnx2x/* "
EOF

# Rebuild after config changes
sudo dracut --force
```

### Ubuntu — initramfs.conf

```bash
# Main config
/etc/initramfs-tools/initramfs.conf

# Key settings
MODULES=most          # most (default), dep (dependencies only), list (manual)
COMPRESS=lz4          # gzip, lz4, xz, zstd
UMASK=0077            # Permissions for the initramfs file

# Add modules
echo "dm-multipath" >> /etc/initramfs-tools/modules
echo "iscsi_tcp" >> /etc/initramfs-tools/modules

# Add custom scripts (hooks)
# /etc/initramfs-tools/scripts/init-premount/
# /etc/initramfs-tools/scripts/init-bottom/
# /etc/initramfs-tools/hooks/

# Rebuild after changes
sudo update-initramfs -u
```

## What's Inside

Typical contents of an initramfs:

```
/init                          # Main init script (PID 1 in initramfs)
/bin/                          # Essential binaries (sh, mount, udevadm, etc.)
/sbin/                         # System binaries (modprobe, blkid, lvm, mdadm)
/lib/modules/<version>/        # Kernel modules (.ko files)
/lib/firmware/                 # Device firmware
/etc/                          # Config files (fstab, lvm, multipath, modprobe.d)
/usr/lib/systemd/              # systemd units (if using systemd in initramfs)
/usr/bin/                      # Additional tools
/proc/ /sys/ /dev/             # Virtual filesystem mount points
```

### Common Modules Included

| Module | Purpose |
|--------|---------|
| `ext4`, `xfs`, `btrfs` | Root filesystem drivers |
| `sd_mod`, `ahci`, `nvme` | Storage controller drivers |
| `dm-mod`, `dm-crypt` | Device mapper (LVM, LUKS) |
| `raid0`, `raid1`, `raid456` | Software RAID |
| `iscsi_tcp`, `libiscsi` | iSCSI initiator |
| `dm-multipath` | Multipath I/O |
| `virtio_blk`, `virtio_scsi` | Virtual machine storage |
| `overlay` | OverlayFS (live systems) |

## Boot Parameters Affecting initramfs

Pass these via GRUB (`/etc/default/grub` → `GRUB_CMDLINE_LINUX`):

### Debugging

```bash
# Drop to shell at default breakpoint (before pivot to real root)
rd.break

# Drop to shell at a specific breakpoint
rd.break=cmdline        # After kernel cmdline parsing
rd.break=pre-udev       # Before udev starts
rd.break=pre-trigger    # Before udev trigger
rd.break=initqueue      # During device discovery
rd.break=pre-mount      # Before mounting root
rd.break=mount          # At mount stage
rd.break=pre-pivot      # Before switch_root
rd.break=cleanup        # During cleanup

# Ubuntu equivalents
break=premount          # Before mounting root
break=mount             # At mount stage

# Shell on errors (automatically drops to shell if something fails)
rd.shell

# Enable verbose debug output from dracut scripts
rd.debug

# Enable informational logging (less verbose than rd.debug)
rd.info

# Debug logging (Ubuntu)
debug
```

### Root Device

```bash
# Specify root device
root=/dev/mapper/vg-root
root=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
root=LABEL=rootfs

# Root filesystem type (usually auto-detected)
rootfstype=xfs
rootfstype=ext4

# Root mount options
rootflags=subvol=root,compress=zstd        # btrfs example
rootflags=noatime,errors=remount-ro        # ext4 example

# Mount root read-only initially (default — allows fsck before writes)
ro

# Mount root read-write (less common)
rw

# Wait for slow root device (USB, iSCSI, some RAID controllers)
rootdelay=10
rootwait
```

### Driver and Module Control

```bash
# Blacklist a module in initramfs
rd.driver.blacklist=nouveau
rd.driver.blacklist=nouveau rd.driver.blacklist=radeon

# Blacklist system-wide (persists after boot)
modprobe.blacklist=nouveau

# Force early loading of a module
rd.driver.pre=ahci

# Delay loading until after other drivers
rd.driver.post=uas
```

### LVM

```bash
# Enable/disable LVM support
rd.lvm=1                            # Enable (default)
rd.lvm=0                            # Disable (faster boot if no LVM)

# Activate specific volume group only
rd.lvm.vg=vg_root
rd.lvm.vg=vg_root rd.lvm.vg=vg_data

# Activate specific logical volume only
rd.lvm.lv=vg_root/lv_root
```

### LUKS (Encrypted Root)

```bash
# Enable/disable LUKS support
rd.luks=1                           # Enable (default)
rd.luks=0                           # Disable (faster if no encryption)

# Unlock specific LUKS device by UUID
rd.luks.uuid=12345678-1234-1234-1234-123456789abc

# Open with a custom device-mapper name
rd.luks.name=12345678-1234-1234-1234-123456789abc=cryptroot
# Results in /dev/mapper/cryptroot

# Use a key file instead of passphrase
rd.luks.key=/root/keyfile:UUID=<usb_uuid>

# Pass options to cryptsetup
rd.luks.options=timeout=60
```

### RAID (mdadm)

```bash
# Enable/disable software RAID
rd.md=1                             # Enable (default)
rd.md=0                             # Disable

# Activate specific RAID array by UUID
rd.md.uuid=12345678:12345678:12345678:12345678

# Disable auto-assembly of arrays
rd.auto=0
```

### Multipath

```bash
# Enable/disable multipath support
rd.multipath=1
rd.multipath=0
```

### Network Boot

```bash
# DHCP
ip=dhcp

# Static IP
ip=192.168.1.10::192.168.1.1:255.255.255.0:hostname:eth0:none
# Format: ip=<client-ip>:<server-ip>:<gw-ip>:<netmask>:<hostname>:<iface>:<autoconf>

# Force network initialization
rd.neednet=1

# Specify boot network interface
bootdev=eth0

# iSCSI root
netroot=iscsi:192.168.1.100::3260::iqn.2024-01.com.storage:root
rd.iscsi.initiator=iqn.2024-01.com.example:server01

# NFS root
root=nfs:192.168.1.100:/export/rootfs
```

### Init Override

```bash
# Override default init program
init=/bin/bash                      # Emergency root shell (bypasses all init)
init=/bin/sh                        # Minimal shell
init=/usr/lib/systemd/systemd      # Explicit systemd

# Boot to specific systemd target
systemd.unit=rescue.target          # Single-user (rescue) mode
systemd.unit=emergency.target       # Minimal emergency mode
systemd.unit=multi-user.target      # Multi-user without GUI
systemd.unit=graphical.target       # Full desktop

# Runlevel shortcuts (SysV compatibility)
single                              # Single-user mode
S                                   # Single-user mode
1                                   # Single-user mode
3                                   # Multi-user
5                                   # Graphical
emergency                           # Emergency target
```

> **Security note:** `init=/bin/bash` gives immediate root access without authentication. Physical access = root access unless Secure Boot + disk encryption are configured.

### Timeouts

```bash
# Global timeout for initramfs operations (0 = wait forever)
rd.timeout=0

# How long to wait for devices to appear (default: 30)
rd.retry=60
```

### Miscellaneous

```bash
# Skip filesystem check on root
rd.skipfsck

# Ignore /etc/fstab in initramfs
rd.fstab=0

# Live system parameters
rd.live.image
rd.writable.fsimg
rd.live.overlay=/dev/sda1

# Plymouth (splash screen)
rd.plymouth=0
plymouth.enable=0
```

### How to Add Kernel Parameters

**Temporarily (one boot):**
1. At GRUB menu, press `e` to edit
2. Find the line starting with `linux` or `linux16` or `linuxefi`
3. Add parameters at the end of that line
4. Press `Ctrl+X` or `F10` to boot

**Permanently:**
```bash
# 1. Edit GRUB defaults
sudo vi /etc/default/grub

# 2. Add to GRUB_CMDLINE_LINUX
GRUB_CMDLINE_LINUX="rd.lvm.lv=vg0/root rd.luks.uuid=12345..."

# 3. Regenerate GRUB config
sudo grub2-mkconfig -o /boot/grub2/grub.cfg          # RHEL
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg # RHEL (EFI)
sudo update-grub                                       # Ubuntu

# 4. View current boot parameters
cat /proc/cmdline
```

## The switch_root Mechanism

After the initramfs init script mounts the real root filesystem:

1. Real root is mounted at `/sysroot` (RHEL) or `/root` (Ubuntu)
2. Virtual filesystems (`/dev`, `/proc`, `/sys`) are moved to the new root
3. `switch_root` atomically changes the root directory
4. All initramfs files are deleted (memory freed)
5. `exec` replaces the initramfs init with the real `/sbin/init` (systemd)
6. PID 1 continues as systemd from the real root

```bash
# What switch_root does internally:
mount --move /dev /sysroot/dev
mount --move /proc /sysroot/proc
mount --move /sys /sysroot/sys
cd /sysroot
# Delete everything in old root
find / -xdev -delete 2>/dev/null
# Switch root and exec real init
exec switch_root /sysroot /sbin/init
```

> **initrd used `pivot_root`** (more complex: old root remains mounted, must be manually unmounted). **initramfs uses `switch_root`** (cleaner: old root is deleted entirely, memory reclaimed automatically).

## Common Use Cases

### Encrypted Root (LUKS)

```
Boot → initramfs loads → prompts for passphrase → cryptsetup opens LUKS →
mounts decrypted device → switch_root to real filesystem
```

### LVM Root

```
Boot → initramfs loads → vgscan/vgchange activates VGs →
mounts logical volume → switch_root
```

### Network Boot (NFS/iSCSI)

```
Boot → initramfs loads → network drivers loaded → DHCP/static IP configured →
NFS mount or iSCSI login → mounts network root → switch_root
```

### Software RAID Root

```
Boot → initramfs loads → mdadm assembles arrays →
mounts RAID device → switch_root
```

### Recovery / Rescue

```bash
# Password reset flow:
# 1. Add rd.break to kernel cmdline
# 2. System drops to initramfs shell with root on /sysroot
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel    # SELinux relabel on RHEL
exit
exit
# System continues boot
```

## Memory Considerations

| Aspect | Details |
|--------|---------|
| Compressed size | Typically 20–100 MB (depends on included drivers) |
| Uncompressed in RAM | 50–300 MB (tmpfs, dynamically sized) |
| After switch_root | Memory fully reclaimed — initramfs contents deleted |
| Low-memory systems | Large initramfs can be problematic (< 512 MB RAM) |
| hostonly builds | Smaller (only hardware-specific drivers: 15–30 MB) |
| generic builds | Larger (all drivers for portability: 50–100 MB) |

```bash
# Check initramfs compressed size
ls -lh /boot/initramfs-$(uname -r).img

# Estimate uncompressed size
lsinitrd /boot/initramfs-$(uname -r).img | tail -1
```

## Security Aspects

| Concern | Details |
|---------|---------|
| Runs as root | initramfs init executes with full privileges |
| Can contain secrets | Encryption keys, network credentials, TPM-sealed keys |
| Rootkit vector | Modified initramfs can compromise the system silently |
| Secure Boot | Validates kernel + initramfs signature (if configured) |
| Physical access | Anyone with physical access can add `init=/bin/bash` |

**Hardening:**

```bash
# Restrict initramfs file permissions
chmod 600 /boot/initramfs-*.img
chmod 600 /boot/initrd.img-*

# Verify initramfs hasn't been tampered with (RHEL)
rpm -V dracut

# Check file hashes
sha256sum /boot/initramfs-$(uname -r).img > /root/initramfs.sha256
# Later verify:
sha256sum -c /root/initramfs.sha256
```

> **Secure Boot** ensures only signed kernels and initramfs images can boot. Without it, an attacker with physical access can modify the initramfs to install a rootkit or bypass authentication.

## Troubleshooting

### System Doesn't Boot After Kernel Update

```bash
# Boot into rescue/older kernel from GRUB menu
# Then rebuild initramfs:

# RHEL
sudo dracut --force --verbose

# Ubuntu
sudo update-initramfs -u

# Verify it was created
ls -la /boot/initramfs-$(uname -r).img    # RHEL
ls -la /boot/initrd.img-$(uname -r)       # Ubuntu
```

### Drop to initramfs Emergency Shell

```bash
# If you see "dracut:/#" or "(initramfs)" prompt:

# Check what went wrong
journalctl    # If systemd is in initramfs
cat /run/initramfs/rdsosreport.txt    # RHEL dracut report
dmesg | tail -50

# List available block devices
blkid
ls /dev/mapper/
ls /dev/sd* /dev/nvme*
lvm lvscan

# Try mounting root manually
mkdir -p /sysroot
mount /dev/mapper/vg-root /sysroot
# Then exit to continue boot
exit
```

### initramfs Too Large

```bash
# Check size
ls -lh /boot/initramfs-*.img    # RHEL
ls -lh /boot/initrd.img-*       # Ubuntu

# Use hostonly mode (RHEL) — only includes drivers for current hardware
echo 'hostonly="yes"' | sudo tee /etc/dracut.conf.d/99-hostonly.conf
sudo dracut --force

# Exclude unnecessary modules
echo 'omit_dracutmodules+=" plymouth anaconda "' | sudo tee /etc/dracut.conf.d/99-omit.conf
sudo dracut --force

# Change compression (xz is smaller but slower, lz4 is fast but larger)
echo 'compress="xz"' | sudo tee /etc/dracut.conf.d/99-compress.conf
sudo dracut --force

# Ubuntu: change MODULES from "most" to "dep"
sudo sed -i 's/MODULES=most/MODULES=dep/' /etc/initramfs-tools/initramfs.conf
sudo update-initramfs -u
```

### Missing Driver in initramfs

```bash
# Check if a module is included
lsinitrd /boot/initramfs-$(uname -r).img | grep megaraid    # RHEL
lsinitramfs /boot/initrd.img-$(uname -r) | grep megaraid    # Ubuntu

# Add a driver permanently
# RHEL
echo 'add_drivers+=" megaraid_sas "' | sudo tee /etc/dracut.conf.d/99-drivers.conf
sudo dracut --force

# Ubuntu
echo "megaraid_sas" >> /etc/initramfs-tools/modules
sudo update-initramfs -u

# Add temporarily (one-time rebuild)
sudo dracut --force --add-drivers "megaraid_sas"    # RHEL

# Regenerate module dependencies before rebuilding (important after manual module install)
sudo depmod -a
sudo dracut --force
```

### Kernel Panic: VFS Unable to Mount Root

Common causes:
- Root device driver not in initramfs
- Wrong `root=` parameter in GRUB
- LVM/RAID/LUKS tools missing from initramfs
- Root UUID changed (after disk clone or resize)

```bash
# Fix from rescue mode:
# 1. Boot rescue kernel or live USB
# 2. Mount root filesystem
mount /dev/mapper/vg-root /mnt
mount /dev/sda1 /mnt/boot    # If separate /boot

# 3. Chroot
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
chroot /mnt

# 4. Rebuild initramfs
dracut --force                        # RHEL
update-initramfs -u                   # Ubuntu

# 5. Update GRUB if root= is wrong
grub2-mkconfig -o /boot/grub2/grub.cfg          # RHEL
update-grub                                       # Ubuntu

# 6. Exit and reboot
exit
umount -R /mnt
reboot
```

## One-Liners

```bash
# Show initramfs size for all kernels
ls -lhS /boot/initramfs-*.img 2>/dev/null; ls -lhS /boot/initrd.img-* 2>/dev/null

# Check what type/compression an initramfs uses
file /boot/initramfs-$(uname -r).img

# List kernel modules in the initramfs
lsinitrd /boot/initramfs-$(uname -r).img | grep '\.ko'       # RHEL
lsinitramfs /boot/initrd.img-$(uname -r) | grep '\.ko'       # Ubuntu

# Check if a specific module is present
lsinitrd -m 2>/dev/null | grep -q multipath && echo "multipath: YES" || echo "multipath: NO"

# Compare initramfs size before/after rebuild
BEFORE=$(stat -c%s /boot/initramfs-$(uname -r).img)
sudo dracut --force
AFTER=$(stat -c%s /boot/initramfs-$(uname -r).img)
echo "Before: $((BEFORE/1024/1024))MB  After: $((AFTER/1024/1024))MB"

# Show dracut modules available on the system
dracut --list-modules 2>/dev/null

# Show what dracut would include for current system
dracut --print-cmdline

# Quick rescue: regenerate ALL initramfs images
sudo dracut --regenerate-all --force    # RHEL
sudo update-initramfs -u -k all         # Ubuntu

# Backup before modifying
sudo cp /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak
```

## dracut vs update-initramfs vs mkinitcpio vs mkinitrd

| Feature | dracut (RHEL 7+) | update-initramfs (Ubuntu) | mkinitcpio (Arch) | mkinitrd (Legacy) |
|---------|-----------------|--------------------------|-------------------|-------------------|
| Distribution | RHEL, Fedora, SUSE | Ubuntu, Debian | Arch Linux | RHEL 5-6 (legacy) |
| Config | `/etc/dracut.conf.d/` | `/etc/initramfs-tools/` | `/etc/mkinitcpio.conf` | `/etc/sysconfig/mkinitrd/` |
| Module system | dracut modules | initramfs-tools hooks | mkinitcpio hooks | Monolithic script |
| hostonly mode | `--hostonly` | `MODULES=dep` | `autodetect` hook | Default behavior |
| Inspect tool | `lsinitrd` | `lsinitramfs`, `unmkinitramfs` | `lsinitcpio` | `gzip -dc \| cpio -t` |
| Force rebuild | `dracut --force` | `update-initramfs -u` | `mkinitcpio -P` | `mkinitrd -f` |
| All kernels | `dracut --regenerate-all` | `update-initramfs -u -k all` | `mkinitcpio -P` | Loop per kernel |

> **Other tools:** Alpine Linux uses `mkinitfs`, Gentoo uses `genkernel` or `dracut`.

## Important Files

| File | Purpose |
|------|---------|
| `/boot/initramfs-<ver>.img` | initramfs image (RHEL) |
| `/boot/initrd.img-<ver>` | initramfs image (Ubuntu/Debian — name is historical) |
| `/etc/dracut.conf` | dracut main config (RHEL) |
| `/etc/dracut.conf.d/*.conf` | dracut drop-in configs (RHEL) |
| `/etc/initramfs-tools/initramfs.conf` | initramfs-tools main config (Ubuntu) |
| `/etc/initramfs-tools/modules` | Extra modules to include (Ubuntu) |
| `/etc/initramfs-tools/hooks/` | Custom hook scripts (Ubuntu) |
| `/etc/initramfs-tools/scripts/` | Custom boot scripts (Ubuntu) |
| `/usr/lib/dracut/modules.d/` | Available dracut modules (RHEL) |
| `/usr/share/initramfs-tools/` | initramfs-tools framework (Ubuntu) |
| `/run/initramfs/rdsosreport.txt` | dracut debug report (RHEL, after failed boot) |

## Timeline

| Year | Milestone |
|------|-----------|
| 1995 | initrd introduced in Linux 1.3.73 |
| 2001 | initrd widely used (kernel 2.4, ext2 ramdisk) |
| 2004 | initramfs introduced in kernel 2.6.0 (cpio-based tmpfs) |
| 2005 | RHEL 4 — last to use classic initrd |
| 2007 | RHEL 5 — switches to initramfs (dracut not yet default) |
| 2011 | RHEL 6 — dracut becomes the default initramfs builder |
| 2014 | RHEL 7 — dracut + systemd in initramfs |
| 2020+ | All major distros use initramfs exclusively |
