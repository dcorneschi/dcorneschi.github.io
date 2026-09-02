# Increasing Disk Space of an Ubuntu Vagrant Box on Provisioning

The default disk in a Vagrant base box is often too small — you hit "no space left on device" while installing large software, need extra room for swap, or want a throwaway lab to experiment with filesystems. This guide shows how to grow the disk of an Ubuntu 22.04 / 24.04 Vagrant box automatically at `vagrant up`, using the `vagrant-disksize` plugin plus a provisioning script.

The approach: the plugin enlarges the underlying VirtualBox disk, then a shell script grows the partition, the LVM physical volume, the logical volume, and finally the filesystem to fill the new space.

## Requirements

- **Vagrant** 2.3+ (2.4.x recommended for VirtualBox 7 support)
- **VirtualBox** 6.1 or 7.0 (match versions Vagrant supports for your release)
- **`vagrant-disksize` plugin**
- An Ubuntu base box — `bento/ubuntu-22.04` or `bento/ubuntu-24.04`. These use LVM for the root filesystem, which the provisioning script assumes.

> Note: `bento` boxes for Jammy/Noble are LVM-backed with an `ext4` root, so the script below uses `resize2fs`. If you use a box with an XFS root, swap that step for `xfs_growfs /` (see [notes](#filesystem-notes)).

## Install the Plugin

The plugin from Simon Protheroe lets you resize the primary disk with one line in the `Vagrantfile`. Vagrant won't auto-install it, so add it first:

```bash
vagrant plugin install vagrant-disksize
```

Plugin limitations to keep in mind:

- Only resizes the **first, primary** disk of the VirtualBox VM.
- Can only **grow** a disk, never shrink it. Requesting a smaller size is ignored with a warning.
- Does **not** resize the partition or filesystem — that's what the provisioning script handles.

## Vagrantfile

```ruby
# Fail early if the plugin is missing
unless Vagrant.has_plugin?("vagrant-disksize")
  raise 'vagrant-disksize is not installed! Run: vagrant plugin install vagrant-disksize'
end

Vagrant.configure("2") do |config|
  config.vm.box      = "bento/ubuntu-24.04"   # or bento/ubuntu-22.04
  config.vm.hostname = "diskextend"

  config.vm.provider "virtualbox" do |vb|
    vb.name   = "DISKEXTEND"
    vb.memory = 2048
    vb.cpus   = 2
  end

  # Grow the default disk to 96 GB
  config.disksize.size = "96GB"

  # Resize the partition + LVM + filesystem after boot
  config.vm.provision "shell", path: "disk-extend.sh"
end
```

On `vagrant up`, the plugin resizes the virtual disk and prints something like:

```
==> DISKEXTEND: Resized disk: old 65536 MB, req 98304 MB, new 98304 MB
==> DISKEXTEND: You may need to resize the filesystem from within the guest.
```

The VirtualBox disk is now bigger, but the guest still sees the old partition/filesystem size. The provisioning script fixes that.

## Provisioning Script: disk-extend.sh

Place this next to your `Vagrantfile`. It uses `growpart` (from `cloud-guest-utils`) to extend the partition non-interactively, then walks the LVM stack up to the filesystem.

```bash
#!/bin/bash
set -euo pipefail

echo "> Installing filesystem tools"
export DEBIAN_FRONTEND=noninteractive
sudo apt-get update -y
sudo apt-get install -y cloud-guest-utils lvm2 e2fsprogs

# Discover the real device layout instead of hard-coding it
ROOT_SRC=$(findmnt -no SOURCE /)                 # e.g. /dev/mapper/ubuntu--vg-ubuntu--lv
PART=$(lsblk -npo PKNAME "$ROOT_SRC" | head -n1) # partition holding the PV, e.g. /dev/sda3
DISK=/dev/$(lsblk -no PKNAME "$PART" | head -n1) # underlying disk, e.g. /dev/sda
PART_NUM=$(echo "$PART" | grep -o '[0-9]*$')     # partition number, e.g. 3

echo "> Root FS:        $ROOT_SRC"
echo "> Partition:      $PART (num $PART_NUM)"
echo "> Disk:           $DISK"
echo "> Before: $(df -h / | awk 'NR==2 {print $2" total, "$4" free"}')"

echo "> Growing partition to fill the disk"
sudo growpart "$DISK" "$PART_NUM" || echo "growpart: nothing to do"

echo "> Resizing the LVM physical volume"
sudo pvresize "$PART"

echo "> Extending the logical volume to 100% of free space"
sudo lvextend -l +100%FREE "$ROOT_SRC" || echo "lvextend: nothing to do"

echo "> Growing the ext4 filesystem"
sudo resize2fs "$ROOT_SRC"

echo "> After:  $(df -h / | awk 'NR==2 {print $2" total, "$4" free"}')"
```

### Why `growpart` instead of `fdisk`?

The original approach fed keystrokes to interactive `fdisk` with a heredoc (delete partition, recreate it larger, write). That works but is fragile — it depends on partition numbers, sector offsets, and exact prompt ordering, and a small mismatch can corrupt the partition table.

`growpart` (part of `cloud-guest-utils`, standard on Ubuntu cloud/Vagrant images) is purpose-built for exactly this: it grows an existing partition in place to consume free space after it, non-interactively and safely. It's the same tool `cloud-init` uses to expand root disks on first boot. Note the space in the syntax — `growpart /dev/sda 3`, not `growpart /dev/sda3`.

### Why device discovery instead of hard-coded `/dev/sda1`?

Ubuntu 22.04/24.04 LVM layouts commonly put the PV on `/dev/sda3` (with a separate EFI/boot partition), and the LV path is `/dev/mapper/ubuntu--vg-ubuntu--lv`, not the older `vagrant-vg/root`. Discovering the layout with `findmnt` / `lsblk` at runtime makes the script portable across box versions instead of breaking when the naming changes.

## The LVM Resize Chain

Each layer must be grown in order — enlarging the disk alone changes nothing the OS sees:

```
VirtualBox disk (grown by the plugin)
        │  growpart
        ▼
Partition  /dev/sda3
        │  pvresize
        ▼
LVM physical volume (PV)
        │  (volume group gains free extents automatically)
        ▼
Logical volume  /dev/mapper/ubuntu--vg-ubuntu--lv
        │  lvextend -l +100%FREE
        ▼
Filesystem (ext4)
        │  resize2fs
        ▼
Usable space on /
```

## Verify the Result

After `vagrant up` (or `vagrant provision` on an existing box):

```bash
vagrant ssh -c "df -h / && lsblk && sudo vgs && sudo lvs"
```

You should see `/` reporting close to the requested size (a few GB is reserved for EFI/boot and metadata):

```
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   93G  2.1G   87G   3% /
```

## Filesystem Notes

- **ext4 root** (default on bento Ubuntu): use `resize2fs "$ROOT_SRC"`.
- **XFS root** (some images): replace the resize step with `sudo xfs_growfs /` — XFS grows by mount point, not device, and can only grow, never shrink.
- **No LVM** (rare for these boxes): skip `pvresize`/`lvextend` and run `growpart` + `resize2fs` directly against the root partition.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Disk in VirtualBox is bigger but `/` unchanged | Only the plugin ran; partition/FS not grown | Run the provisioning script (`vagrant provision`) |
| `growpart: NOCHANGE: partition is size ...` | Partition already fills the disk | Harmless — script's `|| echo` handles it |
| `lvextend` says "nothing to do" | LV already uses all free extents | Harmless; ensure `pvresize` ran first |
| Plugin prints a shrink warning | Requested size ≤ current size | Only increases are supported; request a larger size |
| `pvresize` can't find PV | Non-LVM box or wrong device | Check `lsblk`; use the no-LVM path above |
| `vagrant-disksize is not installed!` | Plugin missing | `vagrant plugin install vagrant-disksize` |

## When to Use This

- Building dev/test boxes that need more room than the base image provides.
- Creating swap-heavy lab VMs for data processing experiments.
- Practicing LVM and filesystem operations in a disposable environment you can `vagrant destroy` and rebuild.

It's meant for local development and testing. It resizes only the first disk and assumes a single-PV LVM root, which is fine for everyday Vagrant work but not a general-purpose storage provisioning tool.

## References

- [vagrant-disksize plugin](https://github.com/sprotheroe/vagrant-disksize) — plugin source
- [`growpart` / cloud-utils](https://github.com/canonical/cloud-utils) — partition growing tool
- [LVM administration (Ubuntu)](https://ubuntu.com/server/docs) — official docs
