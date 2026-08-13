# chroot Guide

`chroot` (change root) changes the apparent root directory for a process and its children. The process sees the specified directory as `/` and cannot access anything outside it. It's used for system recovery, building packages in isolation, testing, and lightweight containment.

## How chroot Works

```
Normal filesystem view:         After chroot /mnt/sysimage:
/                               /mnt/sysimage becomes /
├── bin/                        ├── bin/        (was /mnt/sysimage/bin)
├── etc/                        ├── etc/        (was /mnt/sysimage/etc)
├── usr/                        ├── usr/        (was /mnt/sysimage/usr)
├── mnt/                        └── var/        (was /mnt/sysimage/var)
│   └── sysimage/
│       ├── bin/
│       ├── etc/
│       └── ...
```

The chrooted process:
- Sees `<chroot-dir>` as its root (`/`)
- Cannot access files above the chroot directory
- Uses libraries, binaries, and configs from inside the chroot
- Inherits the kernel (and its running modules) from the host
- Does NOT provide full isolation (root can escape — see "Security" section)

## chroot in Kickstart / Anaconda

When the kickstart process runs, it creates a filesystem in memory — a ramdisk where it does its own operations. This is the environment available to the `%post` section with the `--nochroot` option.

Once kickstart/anaconda finishes the main installation and moves to the `%post` section, it chroots to the filesystem of the new installation (mounted at `/mnt/sysimage`). Your `%post` scripts then feel like they're on the filesystem of the newly installed system.

```bash
# Default %post — runs chrooted into the new system
%post
echo "This runs as if / is the new installation"
yum install -y httpd
systemctl enable httpd
%end

# %post --nochroot — runs in the anaconda ramdisk environment
%post --nochroot
echo "This runs in the installer's ramdisk"
echo "/mnt/sysimage is the new system from here"
cp /some/installer/file /mnt/sysimage/etc/myconfig.conf
%end
```

| Mode | Root Directory | Access to New System |
|------|---------------|---------------------|
| `%post` (default) | New installation (`/mnt/sysimage` chrooted as `/`) | Direct — `/etc` is the new system's `/etc` |
| `%post --nochroot` | Installer ramdisk | Via `/mnt/sysimage/` prefix |

## Basic Usage

```bash
# Change root to a directory
sudo chroot /path/to/new/root

# Change root and run a specific command
sudo chroot /path/to/new/root /bin/bash
sudo chroot /path/to/new/root /usr/bin/passwd root

# Change root as a specific user
sudo chroot --userspec=user:group /path/to/new/root /bin/bash

# Set working directory inside chroot
sudo chroot /path/to/new/root --workdir=/home/user /bin/bash
```

## System Recovery (Most Common Use Case)

The primary use of chroot is recovering a broken Linux system from rescue media or a live USB.

### Full Recovery Procedure

```bash
# 1. Boot from rescue/live media

# 2. Identify your root partition
lsblk
blkid
# or
fdisk -l

# 3. Mount the root filesystem
mount /dev/sda2 /mnt              # Simple partition
# or
mount /dev/mapper/vg-root /mnt    # LVM root

# 4. Mount /boot (if separate)
mount /dev/sda1 /mnt/boot

# 5. Mount /boot/efi (if UEFI)
mount /dev/sda1 /mnt/boot/efi

# 6. Mount virtual filesystems (REQUIRED for most operations)
mount --bind /dev /mnt/dev
mount --bind /dev/pts /mnt/dev/pts
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
mount --bind /run /mnt/run

# 7. (Optional) DNS resolution inside chroot
cp /etc/resolv.conf /mnt/etc/resolv.conf

# 8. Enter the chroot
chroot /mnt /bin/bash

# 9. You're now "inside" the broken system — fix things:
#    - Reset password:      passwd root
#    - Fix GRUB:            grub2-install /dev/sda && grub2-mkconfig -o /boot/grub2/grub.cfg
#    - Rebuild initramfs:   dracut -f
#    - Fix fstab:           vi /etc/fstab
#    - Reinstall package:   dnf reinstall kernel
#    - Fix SELinux:         touch /.autorelabel

# 10. Exit and unmount
exit
umount -R /mnt
# or unmount individually:
umount /mnt/dev/pts
umount /mnt/dev
umount /mnt/proc
umount /mnt/sys
umount /mnt/run
umount /mnt/boot
umount /mnt

# 11. Reboot
reboot
```

### RHEL Rescue Mode (from Installation Media)

```bash
# 1. Boot from RHEL ISO → "Troubleshooting" → "Rescue a Red Hat Enterprise Linux system"
# 2. Rescue mode auto-detects and mounts at /mnt/sysimage
# 3. Chroot in:
chroot /mnt/sysimage

# 4. Fix the system, then:
exit
reboot
```

### Ubuntu Live USB Recovery

```bash
# 1. Boot Ubuntu live USB → "Try Ubuntu"
# 2. Open terminal

# Find root partition
sudo fdisk -l
sudo blkid

# Mount and chroot
sudo mount /dev/sda2 /mnt
sudo mount /dev/sda1 /mnt/boot    # if separate
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo mount --bind /run /mnt/run
sudo chroot /mnt

# Fix system (update-grub, dpkg --configure -a, passwd, etc.)
exit
sudo umount -R /mnt
sudo reboot
```

## Why Virtual Filesystems Are Required

| Mount | Why It's Needed |
|-------|-----------------|
| `/dev` | Access to device nodes (disks, terminals, random) |
| `/dev/pts` | Pseudo-terminal devices (needed for su, sudo, ssh) |
| `/proc` | Process info, kernel parameters (`/proc/cmdline`, `/proc/mounts`) |
| `/sys` | Kernel device/driver info (needed by udev, grub, dracut) |
| `/run` | Runtime data (systemd, dbus sockets, udev) |

Without these mounts, many commands will fail inside the chroot:
- `grub2-install` → "cannot find a device for /boot"
- `dracut` → "failed to find /proc/cmdline"
- `systemctl` → "Failed to connect to bus"
- `dnf`/`apt` → may fail to resolve DNS or access repos

## One-Liner: Mount All + Chroot

```bash
# RHEL/Ubuntu — full chroot setup in one shot
ROOT=/mnt
mount /dev/mapper/vg-root $ROOT && \
mount /dev/sda1 $ROOT/boot && \
for fs in dev dev/pts proc sys run; do mount --bind /$fs $ROOT/$fs; done && \
chroot $ROOT /bin/bash
```

### Cleanup One-Liner

```bash
# Unmount everything
ROOT=/mnt
for fs in dev/pts dev proc sys run boot; do umount $ROOT/$fs 2>/dev/null; done
umount $ROOT
```

## Common Recovery Tasks Inside chroot

### Reset Root Password

```bash
chroot /mnt /bin/bash
passwd root
# If SELinux is enabled (RHEL):
touch /.autorelabel
exit
```

### Fix GRUB

```bash
chroot /mnt /bin/bash

# RHEL (BIOS)
grub2-install /dev/sda
grub2-mkconfig -o /boot/grub2/grub.cfg

# RHEL (UEFI)
dnf reinstall grub2-efi-x64 shim-x64
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg

# Ubuntu (BIOS)
grub-install /dev/sda
update-grub

# Ubuntu (UEFI)
grub-install --target=x86_64-efi --efi-directory=/boot/efi
update-grub
```

### Rebuild initramfs

```bash
chroot /mnt /bin/bash

# RHEL
dracut -f -v

# Ubuntu
update-initramfs -u
```

### Fix Broken Packages

```bash
chroot /mnt /bin/bash

# RHEL
dnf check
dnf reinstall kernel
dnf reinstall grub2-*

# Ubuntu
dpkg --configure -a
apt --fix-broken install
apt install --reinstall linux-image-$(uname -r)
```

### Fix fstab

```bash
chroot /mnt /bin/bash
vi /etc/fstab
# Fix UUID, mount point, or filesystem type
# Verify:
findmnt --verify
```

### Reinstall SELinux Labels (RHEL)

```bash
chroot /mnt /bin/bash
touch /.autorelabel
exit
# On next boot, SELinux will relabel the entire filesystem
```

### Reinstall Kernel

```bash
chroot /mnt /bin/bash

# RHEL — list available kernels and reinstall
dnf list installed kernel*
dnf reinstall kernel-core

# Ubuntu
apt install --reinstall linux-image-generic
update-initramfs -u
update-grub
```

## chroot for Package Building

Create an isolated build environment:

```bash
# Install minimal system into a directory (RHEL)
sudo dnf --installroot=/opt/build-root --releasever=9 install @minimal-environment

# Enter the build environment
sudo chroot /opt/build-root /bin/bash

# Install build tools inside
dnf install gcc make rpm-build

# Build your package
rpmbuild -ba /path/to/spec
exit
```

### Ubuntu / Debian (debootstrap)

```bash
# Install debootstrap
sudo apt install debootstrap

# Create a minimal Ubuntu chroot
sudo debootstrap noble /opt/ubuntu-chroot http://archive.ubuntu.com/ubuntu

# Mount virtual filesystems
sudo mount --bind /dev /opt/ubuntu-chroot/dev
sudo mount --bind /proc /opt/ubuntu-chroot/proc
sudo mount --bind /sys /opt/ubuntu-chroot/sys

# Enter
sudo chroot /opt/ubuntu-chroot /bin/bash

# Inside the chroot:
apt update
apt install build-essential
# ... build packages ...
exit

# Cleanup
sudo umount /opt/ubuntu-chroot/{dev,proc,sys}
```

## chroot for Testing

### Test in a Different RHEL Version

```bash
# Create a RHEL 8 chroot on a RHEL 9 host
sudo dnf --installroot=/opt/rhel8-test --releasever=8 install bash coreutils dnf

sudo mount --bind /dev /opt/rhel8-test/dev
sudo mount --bind /proc /opt/rhel8-test/proc
sudo mount --bind /sys /opt/rhel8-test/sys

sudo chroot /opt/rhel8-test /bin/bash
cat /etc/redhat-release    # Shows RHEL 8
```

### Test Library Compatibility

```bash
# Create chroot with specific library versions
sudo chroot /opt/test-env /bin/bash
ldd /usr/bin/myapp    # Check library dependencies inside the chroot
```

## arch-chroot (Arch Linux Helper)

If `arch-chroot` is available (Arch, or installable on others), it handles virtual filesystem mounts automatically:

```bash
# Mounts /dev, /proc, /sys, etc. automatically
arch-chroot /mnt

# Equivalent manual steps:
mount --bind /dev /mnt/dev
mount --bind /dev/pts /mnt/dev/pts  
mount -t proc proc /mnt/proc
mount -t sysfs sys /mnt/sys
mount -t tmpfs tmpfs /mnt/run
chroot /mnt /bin/bash
```

## Script: Portable chroot Helper

```bash
#!/bin/bash
# chroot-helper.sh — mount, chroot, unmount cleanly
# Usage: sudo ./chroot-helper.sh /mnt [command]

set -euo pipefail

ROOT="${1:?Usage: $0 <root-dir> [command]}"
COMMAND="${2:-/bin/bash}"

echo "Mounting virtual filesystems on $ROOT..."
mount --bind /dev "$ROOT/dev"
mount --bind /dev/pts "$ROOT/dev/pts"
mount --bind /proc "$ROOT/proc"
mount --bind /sys "$ROOT/sys"
mount --bind /run "$ROOT/run" 2>/dev/null || true

# Copy DNS config
cp -L /etc/resolv.conf "$ROOT/etc/resolv.conf" 2>/dev/null || true

echo "Entering chroot at $ROOT..."
chroot "$ROOT" $COMMAND
RETVAL=$?

echo "Cleaning up mounts..."
umount "$ROOT/dev/pts" 2>/dev/null || true
umount "$ROOT/dev" 2>/dev/null || true
umount "$ROOT/proc" 2>/dev/null || true
umount "$ROOT/sys" 2>/dev/null || true
umount "$ROOT/run" 2>/dev/null || true

exit $RETVAL
```

```bash
# Usage
sudo ./chroot-helper.sh /mnt
sudo ./chroot-helper.sh /mnt "passwd root"
sudo ./chroot-helper.sh /mnt "dracut -f -v"
```

## LVM Root Recovery (Full Example)

When root is on LVM:

```bash
# 1. Boot rescue media
# 2. Activate LVM
vgscan
vgchange -ay

# 3. Find logical volumes
lvs

# 4. Mount
mount /dev/mapper/rhel-root /mnt
mount /dev/sda1 /mnt/boot
mount /dev/sda2 /mnt/boot/efi    # if UEFI

# 5. Mount virtual filesystems
for fs in dev dev/pts proc sys run; do mount --bind /$fs /mnt/$fs; done

# 6. Chroot and fix
chroot /mnt /bin/bash
# ... fix ...
exit

# 7. Clean up
for fs in dev/pts dev proc sys run boot/efi boot; do umount /mnt/$fs 2>/dev/null; done
umount /mnt
vgchange -an
reboot
```

## Encrypted Root (LUKS) Recovery

```bash
# 1. Boot rescue media
# 2. Open the LUKS volume
cryptsetup luksOpen /dev/sda3 cryptroot
# Enter passphrase

# 3. Activate LVM on top (if LVM on LUKS)
vgscan
vgchange -ay

# 4. Mount
mount /dev/mapper/vg-root /mnt
mount /dev/sda2 /mnt/boot
mount /dev/sda1 /mnt/boot/efi

# 5. Virtual filesystems + chroot
for fs in dev dev/pts proc sys run; do mount --bind /$fs /mnt/$fs; done
chroot /mnt /bin/bash

# ... fix ...
exit

# 6. Clean up
for fs in dev/pts dev proc sys run boot/efi boot; do umount /mnt/$fs 2>/dev/null; done
umount /mnt
vgchange -an
cryptsetup luksClose cryptroot
reboot
```

## chroot vs Containers vs Namespaces

| Feature | chroot | systemd-nspawn | Docker/Podman |
|---------|--------|----------------|---------------|
| Filesystem isolation | Yes | Yes | Yes |
| Process isolation | No | Yes (PID namespace) | Yes |
| Network isolation | No | Optional | Yes |
| User isolation | No | Optional | Yes |
| cgroup limits | No | Yes | Yes |
| Root escape possible | Yes (trivially for root) | Harder | Harder |
| Use case | Recovery, builds | Testing, builds | Production containers |
| Setup complexity | Minimal | Low | Medium |

## Security Limitations

**chroot is NOT a security boundary.** A process running as root inside a chroot can escape:

```bash
# Classic chroot escape (as root):
mkdir /tmp/escape && chroot /tmp/escape
# Then from inside:
mkdir breakout
mount --bind / breakout
chroot breakout
# You're now in the real root
```

For actual isolation, use:
- `systemd-nspawn` — lightweight container with proper namespaces
- `unshare` — create isolated namespaces manually
- Podman/Docker — full container isolation
- SELinux/AppArmor — MAC-based confinement

## Troubleshooting

### "chroot: failed to run command '/bin/bash': No such file or directory"

```bash
# The binary or its libraries don't exist in the chroot

# Check if bash exists
ls /mnt/bin/bash

# Check library dependencies
ldd /mnt/bin/bash
# If libraries are missing, the chroot won't work

# Check architecture mismatch
file /mnt/bin/bash
# Must match your running kernel's architecture
```

### "cannot access '/dev/null': No such file or directory"

```bash
# Virtual filesystems not mounted
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
```

### Commands fail with "connection refused" or "bus error"

```bash
# Missing /run mount (needed for dbus/systemd communication)
mount --bind /run /mnt/run
```

### DNS Not Working Inside chroot

```bash
# Copy resolv.conf from the host
cp -L /etc/resolv.conf /mnt/etc/resolv.conf
# -L follows symlinks (resolv.conf is often a symlink on systemd systems)
```

### "unable to resolve host" After chroot

```bash
# Set hostname inside chroot
echo "myhost" > /mnt/etc/hostname

# Or export it
chroot /mnt /bin/bash -c "hostname myhost && exec bash"
```

### Cannot Run systemctl Inside chroot

```bash
# systemctl requires running systemd — not available in chroot
# Use alternatives:
chroot /mnt /bin/bash

# Instead of systemctl enable:
ln -s /usr/lib/systemd/system/service.service /etc/systemd/system/multi-user.target.wants/

# Instead of systemctl disable:
rm /etc/systemd/system/multi-user.target.wants/service.service
```

## One-Liners

```bash
# Quick chroot with all mounts (most common recovery pattern)
mount /dev/sda2 /mnt && for fs in dev dev/pts proc sys run; do mount --bind /$fs /mnt/$fs; done && chroot /mnt

# Check if you're in a chroot
ls -di /    # Inode != 2 means you're in a chroot
# or
cat /proc/1/root    # Shows chroot path if applicable
# or
stat -c %d:%i /     # Compare with host

# Run a single command in chroot without interactive shell
chroot /mnt /bin/bash -c "dracut -f && grub2-mkconfig -o /boot/grub2/grub.cfg"

# List all processes in a chroot
ls -la /proc/*/root 2>/dev/null | grep /mnt

# Detect chroot from inside
if [ "$(stat -c %d:%i /)" != "$(stat -c %d:%i /proc/1/root/.)" ]; then
    echo "I'm in a chroot"
fi
```

## systemd-nspawn (Modern chroot Replacement)

`systemd-nspawn` is a lightweight container tool that wraps chroot with proper namespaces. It handles virtual filesystem mounts automatically and provides better isolation than plain chroot.

```bash
# Basic usage — like chroot but with automatic /dev, /proc, /sys
sudo systemd-nspawn -D /mnt

# Boot the system inside the container (runs systemd as PID 1)
sudo systemd-nspawn -D /mnt -b

# Run a specific command
sudo systemd-nspawn -D /mnt /usr/bin/passwd root

# With network access (virtual ethernet pair)
sudo systemd-nspawn -D /mnt --network-veth

# Private (isolated) network
sudo systemd-nspawn -D /mnt --private-network

# Default: shares host network (no flag needed)

# Bind mount a host directory into the container
sudo systemd-nspawn -D /mnt --bind=/home/user/packages:/mnt/packages

# Read-only root (for testing)
sudo systemd-nspawn -D /mnt --read-only

# As a specific user
sudo systemd-nspawn -D /mnt --user=testuser

# With resource limits
sudo systemd-nspawn -D /mnt --property=MemoryMax=512M
```

### systemd-nspawn vs chroot

| Feature | `chroot` | `systemd-nspawn` |
|---------|----------|-----------------|
| Mounts `/dev`, `/proc`, `/sys` | Manual | Automatic |
| PID namespace isolation | No | Yes (PID 1 inside is not host PID 1) |
| Network isolation | No | Optional (`--private-network`) |
| Hostname isolation | No | Yes (uses directory name) |
| Root escape | Trivial | Prevented (unless `--privileged`) |
| Requires systemd on host | No | Yes |
| Use for recovery | Yes | Yes |
| Use for builds | Yes | Yes (better) |

### Recovery with systemd-nspawn

```bash
# Mount root and enter — no manual virtual fs mounts needed
mount /dev/mapper/vg-root /mnt
mount /dev/sda1 /mnt/boot

# Enter (handles /dev, /proc, /sys, /run automatically)
sudo systemd-nspawn -D /mnt

# Fix things, then exit
dracut -f
exit

umount /mnt/boot
umount /mnt
```

## schroot (Debian/Ubuntu Managed chroot Environments)

`schroot` manages persistent chroot environments with configuration files. Useful for maintaining multiple build environments or running services in isolation.

```bash
# Install
sudo apt install schroot debootstrap

# Create a chroot environment
sudo debootstrap jammy /srv/chroot/jammy http://archive.ubuntu.com/ubuntu
```

### Configure schroot

```bash
# Create config file
cat <<'EOF' | sudo tee /etc/schroot/chroot.d/jammy.conf
[jammy]
description=Ubuntu 22.04 Jammy build environment
type=directory
directory=/srv/chroot/jammy
users=builduser
groups=sbuild
root-groups=root
profile=default
personality=linux
preserve-environment=true
EOF
```

### Use schroot

```bash
# List available chroots
schroot -l

# Enter the chroot (as current user)
schroot -c jammy

# Enter as root
schroot -c jammy -u root

# Run a command
schroot -c jammy -- apt update
schroot -c jammy -- dpkg-buildpackage -us -uc

# Begin a session (persistent, survives logout)
schroot -b -c jammy -n my-session
schroot -r -c my-session    # Resume
schroot -e -c my-session    # End session
```

### schroot vs Plain chroot

| Feature | Plain `chroot` | `schroot` |
|---------|---------------|-----------|
| Config files | None | `/etc/schroot/chroot.d/` |
| Non-root users | No (requires sudo) | Yes (configurable per-user access) |
| Mount management | Manual | Automatic (via profiles) |
| Session support | No | Yes (persistent sessions) |
| Profiles | No | Yes (`/etc/schroot/default/`) |
| Use case | Recovery, one-off | Repeated builds, multi-user |

## NFS Root Recovery

When the root filesystem is on NFS:

```bash
# 1. Boot rescue media with networking support

# 2. Configure network
ip addr add 192.168.1.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.1.1

# 3. Mount NFS root
mount -t nfs 192.168.1.100:/export/rootfs /mnt
# or with options:
mount -t nfs -o vers=4,tcp 192.168.1.100:/export/rootfs /mnt

# 4. Mount virtual filesystems
for fs in dev dev/pts proc sys; do mount --bind /$fs /mnt/$fs; done

# 5. DNS (critical for NFS — the chroot needs to resolve the NFS server)
cp -L /etc/resolv.conf /mnt/etc/resolv.conf

# 6. Chroot
chroot /mnt /bin/bash

# Fix issues (network config, fstab, etc.)
vi /etc/fstab
vi /etc/sysconfig/network-scripts/ifcfg-eth0    # RHEL
vi /etc/netplan/*.yaml                           # Ubuntu

exit

# 7. Unmount
for fs in dev/pts dev proc sys; do umount /mnt/$fs; done
umount /mnt
```

> **Note:** For NFS root recovery, the rescue environment must have network connectivity to the NFS server. Ensure the rescue kernel has NFS client support (most do).

## Bind Mount Pitfalls

### Forgetting to Unmount Before Reboot

If you reboot without unmounting bind mounts inside the chroot:
- The system may hang during shutdown waiting for busy filesystems
- `/dev`, `/proc`, `/sys` unmount failures can leave the system in an unclean state
- On next boot, fsck may detect inconsistencies

**Prevention:**

```bash
# Always unmount recursively before reboot
umount -R /mnt

# Or use a trap in scripts
trap 'umount -R /mnt 2>/dev/null' EXIT

# Check for remaining mounts before reboot
mount | grep /mnt
# If anything is still mounted, unmount it
```

### "Target is busy" Error

```bash
# Find what's using the mount
fuser -vm /mnt/dev
lsof +D /mnt

# Common causes:
# - A shell still running inside the chroot (exit all shells first)
# - A background process started inside the chroot
# - Current directory is inside the mount

# Fix: exit all chroot shells, then:
cd /    # Make sure YOU aren't inside the mount
umount -l /mnt/dev    # Lazy unmount (detaches immediately, cleans up when unused)
```

### Stale Mounts After Crash

```bash
# If the system crashed while mounts were active:
# On next boot, check for stale bind mounts
mount | grep "on /mnt"

# Force unmount if needed
umount -f /mnt/proc
umount -f /mnt/sys
umount -f /mnt/dev

# Lazy unmount as last resort
umount -l /mnt/dev
```

### Double-Mounting /dev

```bash
# Problem: running the mount commands twice creates nested bind mounts
# Check before mounting:
mountpoint -q /mnt/dev && echo "Already mounted" || mount --bind /dev /mnt/dev
```

### Safer Mount Pattern (Idempotent)

```bash
#!/bin/bash
# Safe mount that won't double-mount
ROOT=/mnt

safe_bind() {
    local src=$1 dst=$2
    mountpoint -q "$dst" || mount --bind "$src" "$dst"
}

safe_bind /dev "$ROOT/dev"
safe_bind /dev/pts "$ROOT/dev/pts"
safe_bind /proc "$ROOT/proc"
safe_bind /sys "$ROOT/sys"
safe_bind /run "$ROOT/run"
```

## Installation Boot Options (Rescue & Install)

These boot options are entered at the GRUB/installer boot prompt:

| Boot Command | Purpose |
|-------------|---------|
| `linux text` | Force text mode install |
| `linux rescue` | Run rescue environment |
| `linux rescue nomount` | Rescue mode without auto-mounting installed partitions |
| `linux lowres` | Force GUI installer to run at 640x480 |
| `linux noprobe` | Don't auto-detect hardware, prompt user instead |
| `linux asknetwork` | Prompt for network config in first install stage |
| `linux ks=http://server/ks.cfg` | Load kickstart file from HTTP/HTTPS server |

## Virtual Consoles During Installation

Switch between consoles during RHEL/Fedora installation using keyboard shortcuts:

| Console | Keystroke | Contents |
|---------|-----------|----------|
| 1 | `Ctrl+Alt+F1` | Installation dialog |
| 2 | `Ctrl+Alt+F2` | Shell prompt |
| 3 | `Ctrl+Alt+F3` | Install log (messages from installation program) |
| 4 | `Ctrl+Alt+F4` | System-related messages |
| 5 | `Ctrl+Alt+F5` | Other messages |
| 7 | `Ctrl+Alt+F7` | X graphical display |

### tmux Windows (RHEL 8+ / Anaconda)

In RHEL 8+ the installer uses tmux instead of virtual consoles. Switch with `Ctrl+b` then the window number:

| Shortcut | Contents |
|----------|----------|
| `Ctrl+b 1` | Main installation program window (text prompts, VNC Direct Mode, debugging) |
| `Ctrl+b 2` | Interactive shell prompt with root privileges |
| `Ctrl+b 3` | Installation log — `/tmp/anaconda.log` |
| `Ctrl+b 4` | Storage log — `/tmp/storage.log` |
| `Ctrl+b 5` | Program log — `/tmp/program.log` |

> These are invaluable during rescue mode — `Ctrl+b 2` gives you a root shell for running `chroot`, `lvm`, `blkid`, etc. while the main rescue interface is on window 1.

## Important Notes

1. **Kernel stays the same** — chroot does NOT change the running kernel. If you need a different kernel version for `depmod` or module operations, it must match what's installed in the chroot.
2. **`$(uname -r)` in chroot** returns the host/rescue kernel version, not what's installed in the chroot. Always specify versions explicitly.
3. **SELinux contexts** may be wrong after chroot modifications. Use `touch /.autorelabel` (RHEL) before exiting.
4. **Always unmount** before rebooting, especially when the chroot directory is on a separate device.
5. **`umount -R /mnt`** recursively unmounts everything under `/mnt` — the easiest cleanup method.
