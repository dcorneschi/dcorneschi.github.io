# Importing OVA and qcow2 Images into Proxmox

This guide covers importing virtual machines into Proxmox VE from OVA files (VMware exports) and qcow2/VMDK disk images.

## Importing an OVA File

An OVA file is a tar archive containing a VM definition (`.ovf`) and one or more disk images (`.vmdk`).

### Step 1: Upload the OVA to the Proxmox Host

```bash
scp guestvm1.ova root@proxmox-host:/tmp/
```

### Step 2: Extract the OVA

```bash
cd /tmp
tar xvf guestvm1.ova
```

This produces:

- `guestvm1.ovf` — VM configuration (CPU, RAM, network)
- `guestvm1-disk1.vmdk` — disk image
- `guestvm1.mf` — manifest (checksums)

### Step 3: Create a VM in Proxmox

Create an empty VM shell to receive the disk:

```bash
# Create a VM with ID 200 (choose any unused ID)
qm create 200 \
  --name guestvm1 \
  --memory 2048 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-single \
  --ostype l26 \
  --bios seabios
```

OS type options:

| Value | OS |
|-------|-----|
| `l26` | Linux 2.6+ / 3.x / 4.x / 5.x / 6.x |
| `win10` | Windows 10 / 2016 / 2019 |
| `win11` | Windows 11 / 2022 |
| `other` | Other |

### Step 4: Import the Disk

```bash
# Import VMDK to local-lvm storage (converts to raw or qcow2 internally)
qm importdisk 200 /tmp/guestvm1-disk1.vmdk local-lvm

# Or import to a directory-based storage (keeps qcow2)
qm importdisk 200 /tmp/guestvm1-disk1.vmdk local --format qcow2
```

### Step 5: Attach the Imported Disk to the VM

After `importdisk`, the disk appears as an "Unused Disk" in the VM config. Attach it:

```bash
# Attach as SCSI disk (recommended for performance)
qm set 200 --scsi0 local-lvm:vm-200-disk-0

# Or attach as VirtIO disk
qm set 200 --virtio0 local-lvm:vm-200-disk-0

# Or attach as IDE (for older guests)
qm set 200 --ide0 local-lvm:vm-200-disk-0
```

### Step 6: Set Boot Order

```bash
# Set boot from the imported disk
qm set 200 --boot order=scsi0
```

### Step 7: Start the VM

```bash
qm start 200
```

## Importing a qcow2 Image

### Step 1: Upload the qcow2 to the Proxmox Host

```bash
scp CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2 root@proxmox-host:/tmp/
```

### Step 2: Create a VM Shell

```bash
qm create 201 \
  --name centos9-cloud \
  --memory 2048 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-single \
  --ostype l26 \
  --bios seabios \
  --agent enabled=1
```

### Step 3: Import the qcow2 Disk

```bash
# Import to local-lvm
qm importdisk 201 /tmp/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2 local-lvm

# Or keep as qcow2 on directory storage
qm importdisk 201 /tmp/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2 local --format qcow2
```

### Step 4: Attach the Disk

```bash
qm set 201 --scsi0 local-lvm:vm-201-disk-0
```

### Step 5: Set Boot Order and Additional Options

```bash
# Set boot disk
qm set 201 --boot order=scsi0

# Add serial console (for cloud images)
qm set 201 --serial0 socket --vga serial0

# Add cloud-init drive (for cloud images)
qm set 201 --ide2 local-lvm:cloudinit
```

### Step 6: Configure cloud-init (For Cloud Images)

```bash
# Set cloud-init parameters
qm set 201 --ciuser admin
qm set 201 --cipassword "MyPassword123"
qm set 201 --sshkeys ~/.ssh/id_ed25519.pub
qm set 201 --ipconfig0 ip=192.168.50.100/24,gw=192.168.50.1
qm set 201 --nameserver 192.168.50.10
qm set 201 --searchdomain homelab.local
```

### Step 7: Resize the Disk (Cloud Images Are Small)

```bash
# Resize to 30 GB
qm resize 201 scsi0 30G

# Or add space
qm resize 201 scsi0 +20G
```

### Step 8: Start the VM

```bash
qm start 201
```

## Importing a VMDK Image

Same process as qcow2 — `qm importdisk` handles VMDK natively:

```bash
qm create 202 --name vmware-guest --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single --ostype l26
qm importdisk 202 /tmp/guestvm1-disk1.vmdk local-lvm
qm set 202 --scsi0 local-lvm:vm-202-disk-0
qm set 202 --boot order=scsi0
qm start 202
```

## Importing a Raw Image

```bash
qm importdisk 203 /tmp/disk.raw local-lvm --format raw
qm set 203 --scsi0 local-lvm:vm-203-disk-0
```

## Storage Backends

The import target storage depends on your Proxmox setup:

| Storage | Type | Notes |
|---------|------|-------|
| `local-lvm` | LVM-thin | Default thin-provisioned storage, stores as raw |
| `local` | Directory | Stores as qcow2 (supports snapshots) |
| `local-zfs` | ZFS | Stores as zvol, best performance |
| `ceph-pool` | Ceph/RBD | Distributed storage |
| `nfs-storage` | NFS | Network storage |

```bash
# List available storage
pvesm status

# Check storage content types
pvesm status | grep -E "images|rootdir"
```

## Creating a Template from an Imported Image

After importing and configuring a cloud image, convert it to a template for fast cloning:

```bash
# Configure the VM (cloud-init, resize, etc.) first, then:
qm template 201

# Clone from template
qm clone 201 300 --name new-vm --full
qm set 300 --ipconfig0 ip=192.168.50.101/24,gw=192.168.50.1
qm start 300
```

## UEFI Boot (For UEFI-Based VMs)

If the source VM used UEFI boot:

```bash
# Create VM with OVMF (UEFI) BIOS
qm create 204 \
  --name uefi-vm \
  --memory 2048 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-single \
  --ostype l26 \
  --bios ovmf \
  --efidisk0 local-lvm:1,efitype=4m,pre-enrolled-keys=0

# Then import disk and attach as usual
qm importdisk 204 /tmp/disk.qcow2 local-lvm
qm set 204 --scsi0 local-lvm:vm-204-disk-0
qm set 204 --boot order=scsi0
```

## Via the Web UI

You can also import via the Proxmox web interface:

1. Upload the disk image to a storage (Datacenter → Storage → Content → Upload)
2. Create a new VM without a disk (uncheck "Add Disk" during creation)
3. Go to VM → Hardware → Add → Hard Disk → select the uploaded image
4. Or use the CLI method above (more flexible)

> **Note:** The web UI does not directly support OVA import. Use the CLI method for OVA files.

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| `storage does not support format` | LVM-thin doesn't store qcow2 | Use `local` (directory) storage or omit `--format` |
| VM won't boot | Wrong boot order or BIOS type | Check `qm set --boot`, try `--bios ovmf` for UEFI VMs |
| No network after import | Wrong NIC driver | Change to `virtio` NIC: `qm set <id> --net0 virtio,bridge=vmbr0` |
| Disk shows as "Unused" | Not attached after import | Run `qm set <id> --scsi0 <storage>:vm-<id>-disk-0` |
| Cloud image — no login | No cloud-init configured | Add `--ide2 <storage>:cloudinit` and set `--ciuser`/`--cipassword` |
| Slow disk performance | Using IDE bus | Switch to SCSI with virtio-scsi-single |
| UEFI VM won't boot | Missing EFI disk | Add `--efidisk0` and set `--bios ovmf` |

## Quick Reference

```bash
# Import OVA
tar xvf vm.ova
qm create 200 --name vm --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single --ostype l26
qm importdisk 200 disk.vmdk local-lvm
qm set 200 --scsi0 local-lvm:vm-200-disk-0 --boot order=scsi0
qm start 200

# Import qcow2
qm create 201 --name vm --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single --ostype l26
qm importdisk 201 image.qcow2 local-lvm
qm set 201 --scsi0 local-lvm:vm-201-disk-0 --boot order=scsi0
qm start 201

# Import qcow2 cloud image with cloud-init
qm create 201 --name vm --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single --ostype l26 --agent enabled=1
qm importdisk 201 cloud-image.qcow2 local-lvm
qm set 201 --scsi0 local-lvm:vm-201-disk-0 --boot order=scsi0
qm set 201 --ide2 local-lvm:cloudinit --serial0 socket --vga serial0
qm set 201 --ciuser admin --cipassword pass --ipconfig0 ip=192.168.50.100/24,gw=192.168.50.1
qm resize 201 scsi0 30G
qm start 201

# Create template for cloning
qm template 201
qm clone 201 300 --name new-vm --full
```
