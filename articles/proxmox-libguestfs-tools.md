# Why Proxmox Needs libguestfs-tools

When building VM templates on Proxmox — especially from downloaded cloud images — many guides start with:

```bash
apt update -y && apt install libguestfs-tools -y
```

This installs `virt-customize`, `virt-sysprep`, `virt-edit`, and related utilities that let you modify a disk image **offline**, before the VM ever boots. On Proxmox this matters because cloud images ship bare, and the clean way to prepare them is to inject packages, users, and settings directly into the image rather than booting it and configuring it by hand. This article explains what the package is for, why it is not installed by default, and the common Proxmox workflows that depend on it.

> These commands run on the **Proxmox VE host** (Debian-based), as `root`. For the full command reference across libvirt/KVM, see [KVM libguestfs Tools](articles/kvm-libguestfs-tools.md).

## What the Package Provides

`libguestfs-tools` is a collection of utilities that open and edit guest disk images (qcow2, raw, VMDK, and more) without booting the guest or mounting the image on the host kernel.

| Tool | Purpose |
|------|---------|
| `virt-customize` | Install packages, run commands, set passwords, and inject files into an existing image |
| `virt-sysprep` | "Generalize" an image for templating — remove SSH host keys, machine-id, logs, and other per-machine state |
| `virt-edit` | Edit a single file inside an image |
| `virt-cat` | Read a file out of an image |
| `virt-df` | Show filesystem usage inside an image |
| `virt-filesystems` | List partitions, filesystems, and LVM inside an image |
| `guestfish` | Interactive shell for deep inspection and scripting |

## Why It Is Not Installed by Default

Proxmox VE ships a minimal Debian base focused on running the hypervisor (`qemu`, `corosync`, `pve-*` tooling). Offline image editing is a template-preparation task, not something the hypervisor needs to run guests — so `libguestfs-tools` (and its sizable dependency chain, including an appliance used to safely open images) is left out. You install it on demand when you start building templates.

That is why the `apt update && apt install libguestfs-tools` step appears at the top of so many "build a Proxmox cloud-init template" guides: without it, the `virt-customize` / `virt-sysprep` commands that follow simply are not present.

```bash
apt update -y && apt install libguestfs-tools -y

# Verify
virt-customize --version
```

`apt update` refreshes the package index first so `apt install` can resolve the current version; the `-y` flags make both non-interactive, which is convenient in setup scripts.

## The Core Proxmox Workflow: Customizing a Cloud Image

Distributions publish minimal "cloud images" (Ubuntu, Debian, Rocky, etc.) designed to be booted with cloud-init. Straight out of the box they often lack the `qemu-guest-agent`, have no usable login, and carry no SSH keys. `virt-customize` fixes all of that offline, before you convert the image into a Proxmox template.

```bash
# Download a cloud image
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Install the guest agent so Proxmox can report IPs, do clean shutdowns, and fsfreeze backups
virt-customize -a jammy-server-cloudimg-amd64.img --install qemu-guest-agent

# Set a root password (or better, inject an SSH key)
virt-customize -a jammy-server-cloudimg-amd64.img --root-password password:changeme

# Inject an SSH public key for the default user
virt-customize -a jammy-server-cloudimg-amd64.img \
  --ssh-inject root:file:/root/.ssh/id_ed25519.pub

# Run arbitrary commands or drop in files
virt-customize -a jammy-server-cloudimg-amd64.img \
  --run-command 'systemctl enable qemu-guest-agent' \
  --timezone Europe/Bucharest
```

Then turn the prepared image into a Proxmox VM template:

```bash
qm create 9000 --name ubuntu-2204-template --memory 2048 --net0 virtio,bridge=vmbr0
qm importdisk 9000 jammy-server-cloudimg-amd64.img local-lvm
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --ide2 local-lvm:cloudinit --boot c --bootdisk scsi0 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1
qm template 9000
```

New VMs cloned from this template boot with the guest agent already installed and your key already present — no manual first-boot configuration.

## Why Install qemu-guest-agent This Way

Installing the guest agent into the image offline (rather than after first boot) means every clone has it from the very first boot. That gives Proxmox:

- Accurate guest IP reporting in the UI
- Clean ACPI-independent shutdown/reboot
- Filesystem freeze/thaw for consistent backups and snapshots

See [QEMU Guest Agent on Proxmox](articles/proxmox-qemu-guest-agent.md) for what the agent enables in detail.

## Generalizing an Image with virt-sysprep

Before templating a VM you have already booted, run `virt-sysprep` to strip per-machine identity so every clone is unique:

```bash
virt-sysprep -a myvm-disk.qcow2
```

By default this removes SSH host keys, `/etc/machine-id`, logs, shell history, cloud-init state, and more. Without it, cloned VMs can share SSH host keys or machine-ids, which breaks DHCP leases, SSH host verification, and monitoring identity.

## Inspecting and Repairing Images

Beyond templating, the same tools help when a guest will not boot:

```bash
# What filesystems does this image contain?
virt-filesystems -a broken.qcow2 --all --long -h

# Read a file without booting (e.g. check what cloud-init did)
virt-cat -a broken.qcow2 /var/log/cloud-init-output.log

# Fix a bad fstab that is blocking boot
virt-edit -a broken.qcow2 /etc/fstab

# Reset a forgotten root password on an existing disk
virt-customize -a broken.qcow2 --root-password password:newpass
```

## Common Pitfalls

- **Image must not be in use.** Editing a disk that a running VM has open risks corruption. Shut the VM down (or work on a copy) first.
- **Format matters for speed.** `virt-customize` works on qcow2 and raw directly. Editing images already imported into LVM-thin or ZFS means pointing the tools at the block device, which is more error-prone — customize the file *before* `qm importdisk`.
- **libguestfs appliance errors.** If tools fail with an appliance/supermin error, run `libguestfs-test-tool` to diagnose; on some kernels you may need to set `export LIBGUESTFS_BACKEND=direct`.
- **Disk space.** The package and its appliance pull in a fair amount of data — expected on a hypervisor host, just be aware on tight root filesystems.

## Summary

`libguestfs-tools` is not part of the minimal Proxmox base, so you install it explicitly with `apt update -y && apt install libguestfs-tools -y` whenever you build templates. It gives you `virt-customize` to bake the guest agent, users, keys, and packages into cloud images offline, and `virt-sysprep` to generalize images for cloning — the standard, repeatable way to produce Proxmox cloud-init templates without hand-configuring each VM on first boot.
