# AWS EC2: Why Device Names Change and How to Use Labels in fstab

## The Problem

On EC2 instances built on the AWS Nitro System (most modern instance types), EBS volumes are exposed as NVMe block devices. The NVMe device names (`/dev/nvme0n1`, `/dev/nvme1n1`, etc.) are **not stable** — they can change between reboots, stop/start cycles, or instance type changes.

If your `/etc/fstab` references device names like `/dev/nvme1n1` or `/dev/xvdf`, you risk:

- Volumes mounting to the wrong mount point (data on `/logs`, logs on `/data`)
- Boot failures when the expected device name doesn't exist
- Data corruption if the wrong volume gets written to

## Why Device Names Change

### NVMe Device Enumeration

On Nitro instances, the NVMe driver discovers devices in parallel. The order in which devices respond during boot determines their `/dev/nvmeXn1` numbering. This order is **not guaranteed** to be consistent.

```
Before reboot:                    After reboot:
  /dev/nvme0n1 → root              /dev/nvme0n1 → root
  /dev/nvme1n1 → data (100GB)      /dev/nvme1n1 → logs (50GB)   ← swapped!
  /dev/nvme2n1 → logs (50GB)       /dev/nvme2n1 → data (100GB)  ← swapped!
```

### Instance Type Changes

When moving between instance families, the device naming scheme itself can change:

| Instance Family | Device Name Format | Example |
|----------------|-------------------|---------|
| T2, M4, C4 (Xen) | `/dev/xvd[a-z]` | `/dev/xvdf` |
| T3, M5, C5+ (Nitro) | `/dev/nvme[0-26]n1` | `/dev/nvme1n1` |

An fstab entry for `/dev/xvdf` will fail completely after migrating from T2 to T3 because that device path no longer exists.

### Attach/Detach Cycles

Detaching and reattaching an EBS volume (or attaching additional volumes) can also change the NVMe device numbering for all volumes on the instance.

## The Solution: Use Labels or UUIDs

Instead of device names, reference volumes by their filesystem **LABEL** or **UUID** in `/etc/fstab`. These are properties of the filesystem itself — they stay the same regardless of which device path the kernel assigns.

### Labels vs UUIDs

| Aspect | LABEL | UUID |
|--------|-------|------|
| Human-readable | Yes (`data`, `logs`, `backup`) | No (`a1b2c3d4-5678-...`) |
| Uniqueness | Must be manually kept unique | Automatically unique (generated at mkfs) |
| Survives reformat | No (new mkfs = new label unless set again) | No (new mkfs = new UUID) |
| Easy to identify | Yes — `lsblk -f` shows name | Harder — long hex string |
| Best for | Servers with multiple EBS volumes that need clear identification | Single-volume servers, automated provisioning |
| Risk | Duplicate labels cause ambiguous mounts | None (collisions statistically impossible) |

> **Recommendation:** Use **LABEL** when you have multiple EBS volumes and want fstab entries that humans can read and verify at a glance. Use **UUID** when volumes are provisioned automatically (Terraform, CloudFormation) and you don't want to manage label assignment.

## Setting a Filesystem Label

### At Format Time

```bash
# ext4 — set label during mkfs
mkfs.ext4 -L data /dev/nvme1n1

# XFS — set label during mkfs
mkfs.xfs -L logs /dev/nvme2n1
```

### On an Existing Filesystem

```bash
# ext4 — change label (filesystem can be mounted)
e2label /dev/nvme1n1 data
# or
tune2fs -L data /dev/nvme1n1

# XFS — change label (filesystem must be unmounted)
xfs_admin -L logs /dev/nvme2n1
```

### Verify Labels

```bash
# Show all block devices with labels and UUIDs
lsblk -f

# Show label for a specific device
blkid /dev/nvme1n1
# /dev/nvme1n1: LABEL="data" UUID="a1b2c3d4-..." TYPE="ext4"

# Find device by label
blkid -L data
# /dev/nvme1n1
```

## Configuring /etc/fstab

### Bad — Using Device Names (Don't Do This)

```bash
# WRONG — device names can change between reboots on Nitro instances
/dev/nvme1n1  /data  ext4  defaults,noatime  0  2
/dev/nvme2n1  /logs  xfs   defaults,noatime  0  2
```

### Good — Using Labels

```bash
# CORRECT — labels are stable regardless of device enumeration order
LABEL=data  /data  ext4  defaults,noatime,nofail  0  2
LABEL=logs  /logs  xfs   defaults,noatime,nofail  0  2
```

### Good — Using UUIDs

```bash
# CORRECT — UUIDs are stable and automatically unique
UUID=a1b2c3d4-5678-90ab-cdef-1234567890ab  /data  ext4  defaults,noatime,nofail  0  2
UUID=12345678-abcd-efgh-ijkl-123456789012  /logs  xfs   defaults,noatime,nofail  0  2
```

### Always Add nofail

The `nofail` option prevents boot failure if a volume is missing. This is critical on EC2 where:

- An EBS volume might fail to attach during a stop/start
- You might have detached a volume for maintenance
- A snapshot restore might be in progress

Without `nofail`, a missing volume causes the instance to drop into emergency mode — and on EC2, you can't access the console to fix it without a rescue instance.

## Complete Workflow

### 1. Create and Format the Volume

```bash
# Format with a label
sudo mkfs.ext4 -L data /dev/nvme1n1
```

### 2. Create the Mount Point

```bash
sudo mkdir -p /data
```

### 3. Add the fstab Entry

```bash
echo 'LABEL=data  /data  ext4  defaults,noatime,nofail  0  2' | sudo tee -a /etc/fstab
```

### 4. Test the Mount

```bash
# Mount everything in fstab without rebooting
sudo mount -a

# Verify
df -h /data
mount | grep /data
```

### 5. Verify It Survives a Reboot

```bash
sudo reboot
# After reboot:
lsblk -f
mount | grep /data
```

## Cloud-Init and User Data

When provisioning EC2 instances with cloud-init, format volumes with labels and configure fstab in your user data:

```yaml
#cloud-config
# Format attached EBS volumes with labels
disk_setup:
  /dev/nvme1n1:
    table_type: gpt
    layout: true
    overwrite: false

fs_setup:
  - label: data
    filesystem: ext4
    device: /dev/nvme1n1
    overwrite: false

mounts:
  - ["LABEL=data", "/data", "ext4", "defaults,noatime,nofail", "0", "2"]
```

Or with a runcmd script for more control:

```yaml
#cloud-config
runcmd:
  - |
    # Wait for the volume to appear
    while [ ! -b /dev/nvme1n1 ]; do sleep 1; done

    # Only format if not already formatted
    if ! blkid /dev/nvme1n1; then
      mkfs.ext4 -L data /dev/nvme1n1
    fi

    mkdir -p /data

    # Add to fstab if not already present
    if ! grep -q 'LABEL=data' /etc/fstab; then
      echo 'LABEL=data  /data  ext4  defaults,noatime,nofail  0  2' >> /etc/fstab
    fi

    mount -a
```

## Terraform Example

```hcl
resource "aws_ebs_volume" "data" {
  availability_zone = aws_instance.web.availability_zone
  size              = 100
  type              = "gp3"

  tags = {
    Name = "web-data"
  }
}

resource "aws_volume_attachment" "data" {
  device_name = "/dev/sdf"
  volume_id   = aws_ebs_volume.data.id
  instance_id = aws_instance.web.id
}
```

Then in your user data or provisioner:

```bash
# The device_name "/dev/sdf" is just a hint to EC2
# On Nitro, it appears as /dev/nvme*n1 — don't rely on the path
# Format with a label and use LABEL= in fstab
mkfs.ext4 -L data /dev/$(lsblk -o NAME,SERIAL | grep vol${VOLUME_ID//-/} | awk '{print $1}')
```

> **Tip:** AWS stores the EBS volume ID in the NVMe device's serial number. You can use `nvme id-ctrl /dev/nvmeXn1` or `lsblk -o NAME,SERIAL` to map NVMe devices back to their EBS volume IDs.

## Identifying NVMe Devices

When you need to figure out which NVMe device corresponds to which EBS volume:

```bash
# Show NVMe device serial numbers (contains EBS volume ID)
lsblk -o NAME,SIZE,SERIAL,FSTYPE,LABEL,MOUNTPOINT

# Use the AWS NVMe tool (Amazon Linux, Ubuntu with aws-ec2-utils)
sudo ebsnvme-id /dev/nvme1n1
# Volume ID: vol-0abc123def456789

# Or use nvme-cli
sudo nvme id-ctrl /dev/nvme1n1 -o json | jq -r '.sn'
# vol0abc123def456789  (no hyphens)
```

### Mapping EBS Volume IDs to NVMe Devices

```bash
#!/bin/bash
# List all NVMe EBS volumes with their volume IDs and labels
for dev in /dev/nvme*n1; do
  vol_id=$(sudo nvme id-ctrl "$dev" -o json 2>/dev/null | jq -r '.sn' | sed 's/^vol/vol-/')
  label=$(lsblk -no LABEL "$dev" 2>/dev/null)
  size=$(lsblk -no SIZE "$dev" 2>/dev/null | head -1)
  echo "$dev  size=$size  vol_id=$vol_id  label=${label:-<none>}"
done
```

## Instance Store (Ephemeral) Volumes

Instance store volumes (NVMe SSDs on instance types like i3, c5d, m5d) have a different behavior:

- Data is **lost** on stop/start (but persists on reboot)
- Device names can also change between reboots
- They don't have persistent UUIDs (reformatted on stop/start)

For instance store volumes, use labels set during boot:

```yaml
#cloud-config
bootcmd:
  - mkfs.xfs -f -L ephemeral /dev/nvme1n1
  - mkdir -p /mnt/ephemeral
  - mount LABEL=ephemeral /mnt/ephemeral
```

Or in fstab with `nofail` and `noauto` (mount via a boot script instead):

```
LABEL=ephemeral  /mnt/ephemeral  xfs  defaults,noatime,nofail  0  0
```

> **Note:** For instance store, using `bootcmd` with `mkfs` ensures the volume is formatted on every boot (since data is lost on stop/start anyway). Don't do this for EBS volumes — you'd wipe your data.

## Common Scenarios

### Scenario 1: Volumes Mount to Wrong Directories

**Symptom:** After reboot, `/data` contains log files and `/logs` contains application data.

**Cause:** fstab uses `/dev/nvme1n1` and `/dev/nvme2n1`, and the NVMe enumeration order changed.

**Fix:**
```bash
# Unmount
sudo umount /data /logs

# Add labels
sudo e2label /dev/nvme1n1 data  # check with blkid first!
sudo xfs_admin -L logs /dev/nvme2n1

# Update fstab
# Replace /dev/nvme1n1 with LABEL=data
# Replace /dev/nvme2n1 with LABEL=logs

# Remount
sudo mount -a
```

### Scenario 2: Instance Fails to Boot After Type Change

**Symptom:** Instance becomes unreachable after changing from T2 to T3.

**Cause:** fstab has entries for `/dev/xvdf` which doesn't exist on Nitro (it's now `/dev/nvmeXn1`).

**Fix:** Detach the root volume, attach it to a rescue instance, fix fstab to use LABEL or UUID, reattach, and boot.

### Scenario 3: Terraform Destroy/Recreate Causes Mount Failure

**Symptom:** After recreating an EBS volume, fstab UUID entry no longer matches.

**Cause:** New volume = new filesystem = new UUID.

**Fix:** Use LABEL instead of UUID. In your provisioning script, always apply the same label when formatting — it stays consistent across volume replacements.

## Summary

| Rule | Why |
|------|-----|
| Never use `/dev/nvmeXn1` or `/dev/xvdX` in fstab | Device names change on reboot, stop/start, or instance type change |
| Use `LABEL=` for multi-volume servers | Human-readable, easy to verify, consistent across volume replacements |
| Use `UUID=` for automated provisioning | Guaranteed unique, no label management needed |
| Always add `nofail` | Prevents boot failure if a volume is missing or slow to attach |
| Set labels at format time (`mkfs -L`) | Easiest and cleanest approach |
| Use `lsblk -f` to verify | Shows device → label → UUID → mountpoint mapping |
| Test with `sudo mount -a` before rebooting | Catches fstab errors before they brick your instance |
