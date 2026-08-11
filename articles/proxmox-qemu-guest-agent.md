# QEMU Guest Agent on Proxmox

## Why QEMU Guest Agent is Required

The QEMU Guest Agent (`qemu-guest-agent`) is a daemon that runs inside a VM and communicates with the Proxmox host via a VirtIO serial channel. Without it, Proxmox has limited visibility into the guest OS.

### What Works Without the Guest Agent

| Feature | Without Agent | With Agent |
|---------|--------------|------------|
| Start/stop VM | ✓ | ✓ |
| Hard reset | ✓ | ✓ |
| VNC/SPICE console | ✓ | ✓ |
| Disk resize (host side) | ✓ | ✓ |
| Basic monitoring (CPU/memory from hypervisor) | ✓ | ✓ |

### What Requires the Guest Agent

| Feature | Without Agent | With Agent |
|---------|--------------|------------|
| **Graceful shutdown** | ACPI signal (may be ignored) | Clean OS shutdown |
| **IP address display** in web UI | ✗ (shows nothing) | ✓ (shows guest IPs) |
| **Filesystem freeze** for consistent backups | ✗ (crash-consistent only) | ✓ (application-consistent) |
| **`qm guest exec`** (run commands remotely) | ✗ | ✓ |
| **`qm guest file pull/push`** (transfer files) | ✗ | ✓ |
| **Accurate OS info** in summary | ✗ | ✓ (shows OS name, kernel) |
| **Time sync** after snapshot restore | ✗ | ✓ |
| **Trim/discard** unused blocks | ✗ | ✓ (fstrim via host) |

### Impact on Backups

This is the most critical reason to install the guest agent:

- **Without agent:** Proxmox takes a snapshot while the VM is running. Disk state may be inconsistent (like pulling the power plug). The backup is "crash-consistent" — usually fine for Linux with journaling filesystems, but databases may need recovery.

- **With agent:** Proxmox tells the guest agent to freeze filesystems (`fsfreeze`) before taking the snapshot. All pending writes are flushed. The backup is "application-consistent" — safe for databases and stateful applications.

## Installing the Guest Agent

### Linux (RHEL/CentOS/Rocky/Alma)

```bash
dnf install -y qemu-guest-agent
systemctl enable --now qemu-guest-agent
```

### Linux (Ubuntu/Debian)

```bash
apt install -y qemu-guest-agent
systemctl enable --now qemu-guest-agent
```

### Linux (RHEL 7 / CentOS 7)

```bash
yum install -y qemu-guest-agent
systemctl enable qemu-guest-agent
systemctl start qemu-guest-agent
```

### Windows

1. Mount the VirtIO drivers ISO (available in Proxmox: local storage → ISO Images → `virtio-win`)
2. Run the installer: `D:\guest-agent\qemu-ga-x86_64.msi`
3. Or install via PowerShell:

```powershell
Start-Process msiexec.exe -ArgumentList '/i D:\guest-agent\qemu-ga-x86_64.msi /quiet' -Wait
```

The service starts automatically after installation.

### Via cloud-init (For Templates)

```yaml
#cloud-config
packages:
  - qemu-guest-agent

runcmd:
  - systemctl enable --now qemu-guest-agent
```

## Enabling in Proxmox

The guest agent must also be enabled in the VM configuration (host side):

### Via Web UI

1. Select VM → **Options**
2. Double-click **QEMU Guest Agent**
3. Check **Use QEMU Guest Agent**
4. Optionally check:
   - **Run guest-trim after a disk move or VM migration**
   - **Freeze/thaw guest filesystems on backup for consistency**

### Via CLI

```bash
# Enable guest agent
qm set <vmid> --agent enabled=1

# Enable with trim and freeze options
qm set <vmid> --agent enabled=1,fstrim_cloned_disks=1,type=virtio

# Verify
qm config <vmid> | grep agent
```

> **Important:** After enabling the agent in Proxmox, the VM must be fully stopped and restarted (not just rebooted) for the VirtIO serial channel to be created.

## Verifying the Agent is Working

### From the Proxmox Host

```bash
# Check if agent responds
qm agent <vmid> ping

# Get network interfaces and IPs
qm agent <vmid> network-get-interfaces

# Get OS info
qm agent <vmid> get-osinfo

# Get filesystem info
qm agent <vmid> get-fsinfo

# Get hostname
qm agent <vmid> get-host-name

# Get timezone
qm agent <vmid> get-timezone

# Get memory info
qm agent <vmid> get-memory-blocks
```

### From the Web UI

- VM → **Summary** — should show IP addresses and OS info
- VM → **Summary** → "IPs" column in the network section

### Inside the Guest

```bash
# Check service is running
systemctl status qemu-guest-agent

# Check the VirtIO serial channel exists
ls /dev/virtio-ports/
# Should show: org.qemu.guest_agent.0

# Check logs
journalctl -u qemu-guest-agent
```

## Using the Guest Agent

### Run Commands Inside the VM (No SSH Needed)

```bash
# Execute a command
qm guest exec <vmid> -- uptime
qm guest exec <vmid> -- df -h
qm guest exec <vmid> -- cat /etc/hostname

# Get the result (if PID is returned)
qm guest exec-status <vmid> <PID>

# Run and wait for result
qm guest exec <vmid> --timeout 30 -- yum update -y
```

### Transfer Files

```bash
# Pull a file from the guest to the host
qm guest file pull <vmid> /etc/hostname /tmp/guest-hostname

# Push a file from the host to the guest
qm guest file push <vmid> /tmp/config.txt /etc/myapp/config.txt
```

### Freeze/Thaw Filesystems

```bash
# Freeze (for consistent snapshot)
qm agent <vmid> fsfreeze-freeze

# Check freeze status
qm agent <vmid> fsfreeze-status

# Thaw
qm agent <vmid> fsfreeze-thaw
```

### Trim Unused Blocks (Reclaim Space)

```bash
# Trigger fstrim inside the guest
qm agent <vmid> fstrim
```

This reclaims unused space on thin-provisioned storage (LVM-thin, ZFS, Ceph).

### Sync Time

```bash
# Sync guest clock with host (useful after snapshot restore)
qm agent <vmid> set-time
```

## Troubleshooting

### Agent Not Responding

```bash
# Check from host
qm agent <vmid> ping
# Error: QEMU guest agent is not running

# Possible causes:
# 1. Agent not installed in the guest
# 2. Agent not enabled in VM options (host side)
# 3. VM needs full stop+start (not reboot) after enabling
# 4. VirtIO serial channel not present
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "QEMU guest agent is not running" | Agent not installed or not started | Install + `systemctl enable --now qemu-guest-agent` |
| Agent installed but not responding | Not enabled in Proxmox VM options | Enable in Options → QEMU Guest Agent, then stop+start VM |
| No `/dev/virtio-ports/` in guest | VirtIO serial channel not created | Stop+start (not reboot) the VM after enabling in Proxmox |
| IPs not showing in web UI | Agent running but not yet queried | Wait 30s, or click refresh |
| Windows agent not starting | Service not installed | Install from VirtIO ISO: `guest-agent/qemu-ga-x86_64.msi` |
| Backups not application-consistent | "Freeze" option not checked | Enable "Freeze/thaw guest filesystems on backup" in Options |
| `fsfreeze-freeze` fails | Guest has unsupported FS or agent too old | Update guest agent, check filesystem support |

### Debug Logging

Inside the guest:

```bash
# Check agent logs
journalctl -u qemu-guest-agent -f

# Restart agent
systemctl restart qemu-guest-agent

# Check channel
file /dev/virtio-ports/org.qemu.guest_agent.0
```

## Best Practices

- **Always install the guest agent** in production VMs — consistent backups alone justify it
- **Install via `packages` in cloud-init** (not `runcmd`) so it's available early
- **Enable `fstrim_cloned_disks`** to reclaim space after cloning or migration
- **Stop+start (not reboot)** after enabling the agent in Proxmox config
- **Include in templates** — install and enable before converting to template

## Quick Reference

```bash
# Install (Linux)
dnf install -y qemu-guest-agent && systemctl enable --now qemu-guest-agent

# Enable in Proxmox
qm set <vmid> --agent enabled=1,fstrim_cloned_disks=1

# Stop+start VM (required after enabling)
qm stop <vmid> && qm start <vmid>

# Verify
qm agent <vmid> ping
qm agent <vmid> network-get-interfaces

# Run command in guest
qm guest exec <vmid> -- hostname

# Pull file from guest
qm guest file pull <vmid> /var/log/syslog /tmp/guest-syslog

# Freeze for backup
qm agent <vmid> fsfreeze-freeze

# Trim unused space
qm agent <vmid> fstrim
```
