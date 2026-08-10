<img src="/articles/images/proxmox-logo.svg" alt="Proxmox" width="350">

# Proxmox Cheatsheet

Comprehensive Proxmox VE reference guide covering VM and container management, storage configuration, networking, cluster operations, backup and restore, firewall rules, and CLI/API usage for virtualization infrastructure.

---

## Overview

Proxmox Virtual Environment (PVE) is an open-source server virtualization platform built on Debian. It combines KVM hypervisor and LXC containers with a web-based management interface, software-defined storage, and networking functionality.

### Key Components

| Component | Description |
|-----------|-------------|
| KVM | Full virtualization for VMs (QEMU/KVM) |
| LXC | Lightweight OS-level containers |
| Ceph | Distributed storage (optional) |
| ZFS | Advanced filesystem with snapshots |
| Corosync | Cluster communication |
| pve-cluster | Cluster filesystem (pmxcfs) |
| pve-firewall | Distributed firewall |
| PBS | Proxmox Backup Server (separate product) |

---

## System Information

### Hardware Information

```bash
lscpu
lsmem
lspci
lsblk
```

### System Status

```bash
pvesh get /nodes/localhost/status
```

### Service Status

```bash
systemctl status pve-cluster
systemctl status pvedaemon
systemctl status pveproxy
systemctl status pvestatd
systemctl status pvenetcommit
```

---

## Installation & Updates

### Post-Install: Disable Enterprise Repo (No Subscription)

```bash
# Disable enterprise repository
sed -i 's/^deb/# deb/' /etc/apt/sources.list.d/pve-enterprise.list

# Add no-subscription repository
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

# Update package index
apt update && apt full-upgrade -y
```

### Check Version

```bash
pveversion -v
```

### Update System

```bash
apt update && apt upgrade
```

### Reboot Node

```bash
shutdown -r now
```

---

## VM Management (qm)

### Creating VMs

| Command | Description |
|---------|-------------|
| `qm create <vmid>` | Create a new VM with given ID |
| `qm create <vmid> --name <name> --memory 2048 --cores 2` | Create VM with basic specs |
| `qm create <vmid> --cdrom local:iso/<file>.iso` | Create VM with ISO attached |
| `qm importdisk <vmid> <image> <storage>` | Import disk image into VM |
| `qm set <vmid> --scsi0 <storage>:vm-<vmid>-disk-0` | Attach imported disk |
| `qm set <vmid> --boot order=scsi0` | Set boot order |
| `qm set <vmid> --boot order=ide2;scsi0` | Boot from CD first (for install) |
| `qm set <vmid> --ide2 local:iso/<file>.iso,media=cdrom` | Attach ISO to CD drive |
| `qm set <vmid> --ide2 none,media=cdrom` | Eject ISO from CD drive |
| `qm set <vmid> --serial0 socket --vga serial0` | Add serial console |

### VM Lifecycle

| Command | Description |
|---------|-------------|
| `qm start <vmid>` | Start a VM |
| `qm shutdown <vmid>` | Graceful shutdown (ACPI) |
| `qm shutdown <vmid> --timeout 30 --forceStop` | Graceful shutdown, force after 30s |
| `qm stop <vmid>` | Force stop (immediate) |
| `qm reboot <vmid>` | Reboot a VM |
| `qm reset <vmid>` | Hard reset a VM |
| `qm suspend <vmid>` | Suspend VM to RAM |
| `qm resume <vmid>` | Resume suspended VM |
| `qm destroy <vmid>` | Delete VM and all its data |
| `qm destroy <vmid> --purge` | Delete VM including from backup jobs and HA |
| `qm sendkey <vmid> <key_event>` | Send key event to virtual machine |
| `qm showcmd <vmid>` | Show command line used to start the VM (debug info) |
| `qm set <vmid>` | Set virtual machine options (synchronous API) |

### VM Configuration

| Command | Description |
|---------|-------------|
| `qm config <vmid>` | Show VM configuration |
| `qm set <vmid> --memory 4096` | Change RAM |
| `qm set <vmid> --cores 4` | Change CPU cores |
| `qm set <vmid> --sockets 2` | Change CPU sockets |
| `qm set <vmid> --cpu cputype=host` | Set CPU type to host passthrough |
| `qm set <vmid> --net0 virtio,bridge=vmbr0` | Add/modify network interface |
| `qm set <vmid> --agent enabled=1` | Enable QEMU Guest Agent |
| `qm set <vmid> --onboot 1` | Start VM on host boot |
| `qm set <vmid> --startup order=1,up=30` | Set startup order and delay |
| `qm set <vmid> --protection 1` | Enable deletion protection |
| `qm pending <vmid>` | Show pending config changes |

### VM Disk Operations

| Command | Description |
|---------|-------------|
| `qm importdisk <vmid> <source> <storage>` | Import an external disk image as an unused disk in a VM |
| `qm move_disk <vmid> <disk> <storage>` | Move volume to different storage or to a different VM |
| `qm rescan` | Rescan all storages and update disk sizes and unused disk images |
| `qm resize <vmid> <disk> <size>` | Extend volume size |
| `qm unlink <vmid> --idlist <string>` | Unlink/delete disk images |
| `qm move_disk <vmid> scsi0 <target-storage>` | Move disk to another storage |
| `qm move_disk <vmid> scsi0 <target-storage> --delete` | Move disk and delete source |
| `qm set <vmid> --scsi1 <storage>:32` | Add a 32 GB disk |
| `qm set <vmid> --delete scsi1` | Remove a disk from config |
| `qemu-img convert <qcow2> <raw>` | Convert qcow2 to raw |
| `qemu-img convert -p -O qcow2 <raw> <qcow2>` | Convert back to qcow2 |

### VM Snapshots

| Command | Description |
|---------|-------------|
| `qm snapshot <vmid> <snapname>` | Create snapshot |
| `qm snapshot <vmid> <snapname> --description "text"` | Create snapshot with description |
| `qm listsnapshot <vmid>` | List all snapshots |
| `qm rollback <vmid> <snapname>` | Rollback to snapshot |
| `qm delsnapshot <vmid> <snapname>` | Delete a snapshot |
| `qm terminal <vmid>` | Open a terminal using a serial device (Ctrl+O to exit) |
| `qm vncproxy <vmid>` | Proxy VM VNC traffic to stdin/stdout |

### VM Templates

| Command | Description |
|---------|-------------|
| `qm template <vmid>` | Convert VM to template (irreversible) |
| `qm clone <vmid> <newid> --name <name> --full` | Full clone from template |
| `qm clone <vmid> <newid> --name <name>` | Linked clone (depends on source, less space) |

### VM Monitoring

| Command | Description |
|---------|-------------|
| `qm status <vmid>` | Show VM status |
| `qm status <vmid> --verbose` | Show VM resource usage (verbose) |
| `qm list` | List all VMs |
| `qm monitor <vmid>` | Enter QEMU monitor (interactive) |
| `qm guest cmd <vmid> ping` | Ping guest agent |
| `qm guest cmd <vmid> <command>` | Execute Qemu Guest Agent commands |
| `qm guest exec <vmid> -- <cmd>` | Execute command in guest via agent |
| `qm guest exec-status <vmid> <pid>` | Gets the status of the given pid started by the guest-agent |
| `qm guest passwd <vmid> <user>` | Sets the password for the given user to the given password |

---

## Container Management (pct)

### Creating Containers

| Command | Description |
|---------|-------------|
| `pveam update` | Update Container Template Database |
| `pveam available` | List all available templates |
| `pveam available --section system` | List available system templates |
| `pveam list <storage>` | List downloaded templates on storage |
| `pveam download <storage> <template>` | Download appliance templates |
| `pveam remove <template>` | Remove a template |
| `pct create <ctid> local:vztmpl/<template> --hostname <name> --memory 512 --rootfs local-lvm:8` | Create container |
| `pct create <ctid> <template> --unprivileged 1` | Create unprivileged container |

### Container Lifecycle

| Command | Description |
|---------|-------------|
| `pct start <ctid>` | Start container |
| `pct shutdown <ctid>` | Graceful shutdown |
| `pct stop <ctid>` | Force stop |
| `pct reboot <ctid>` | Reboot container |
| `pct suspend <ctid>` | Suspend the container (experimental) |
| `pct resume <ctid>` | Resume the container |
| `pct destroy <ctid>` | Delete container |
| `pct destroy <ctid> --purge` | Delete including from backup jobs and HA |
| `pct enter <ctid>` | Attach to container console |
| `pct console <ctid>` | Launch a console for the specified container |
| `pct exec <ctid> -- <cmd>` | Execute command inside container |
| `pct push <ctid> <src> <dst>` | Copy file from host into container |
| `pct pull <ctid> <src> <dst>` | Copy file from container to host |
| `pct migrate <ctid> <target>` | Migrate the container to another node |
| `pct restore <ctid> <template>` | Create or restore a container |
| `pct template <ctid>` | Create a Template |
| `pct unlock <ctid>` | Unlock the container |

### Container Configuration

| Command | Description |
|---------|-------------|
| `pct config <ctid>` | Show container configuration |
| `pct pending <ctid>` | Get container configuration, including pending changes |
| `pct set <ctid>` | Set container options |
| `pct set <ctid> --memory 1024` | Change RAM |
| `pct set <ctid> --cores 2` | Change CPU cores |
| `pct set <ctid> --swap 512` | Change swap |
| `pct set <ctid> --net0 name=eth0,bridge=vmbr0,ip=dhcp` | Configure network (DHCP) |
| `pct set <ctid> --net0 name=eth0,bridge=vmbr0,ip=10.0.0.5/24,gw=10.0.0.1` | Configure static IP |
| `pct set <ctid> --onboot 1` | Start on host boot |
| `pct set <ctid> --startup order=2,up=15` | Set startup order |
| `pct set <ctid> --features nesting=1` | Enable nesting (Docker in LXC) |
| `pct set <ctid> --unprivileged 1` | Set as unprivileged |
| `pct cpusets` | Print the list of assigned CPU sets |
| `pct resize <ctid> rootfs +5G` | Extend rootfs by 5 GB |
| `pct set <ctid> --mp0 <storage>:4,mp=/mnt/data` | Add mount point |
| `pct set <ctid> --mp0 /host/path,mp=/mnt/data` | Add bind mount (host dir → container) |

### Docker in LXC

To run Docker inside an LXC container:

```bash
# Create a privileged container with nesting enabled
pct create <ctid> <template> --unprivileged 0 --features nesting=1

# Or enable on existing container (must be stopped)
pct set <ctid> --unprivileged 0
pct set <ctid> --features nesting=1

# Optionally add bind mount for Docker data persistence
pct set <ctid> --mp0 /mnt/docker,mp=/var/lib/docker
```

> **Note:** Docker requires a privileged container or specific AppArmor/seccomp configuration. For unprivileged containers, use Podman rootless instead.

### Privileged vs Unprivileged Containers

| Type | Use Case | Security |
|------|----------|----------|
| Unprivileged (default) | Web servers, databases, apps | More secure — uses user namespaces |
| Privileged | Docker, NFS servers, FUSE mounts, special hardware | Less secure — full host access |

> **Best practice:** Always use unprivileged containers unless specifically required. For UID/GID mapping in unprivileged containers, edit `/etc/pve/lxc/<ctid>.conf`:
> ```
> lxc.idmap: u 0 100000 65536
> lxc.idmap: g 0 100000 65536
> ```

### Container Snapshots & Cloning

| Command | Description |
|---------|-------------|
| `pct snapshot <ctid> <snapname>` | Create snapshot |
| `pct listsnapshot <ctid>` | List snapshots |
| `pct rollback <ctid> <snapname>` | Rollback to snapshot |
| `pct delsnapshot <ctid> <snapname>` | Delete snapshot |
| `pct clone <ctid> <newid> --hostname <name> --full` | Full clone |
| `pct clone <ctid> <newid> --hostname <name>` | Linked clone |

### Container Disks

| Command | Description |
|---------|-------------|
| `pct df <ctid>` | Get the container's current disk usage |
| `pct fsck <ctid>` | Run a filesystem check (fsck) on a container volume |
| `pct fstrim <ctid>` | Run fstrim on a chosen CT and its mountpoints |
| `pct mount <ctid>` | Mount the container's filesystem on the host |
| `pct unmount <ctid>` | Unmount the container's filesystem |
| `pct move_volume <ctid> <volume>` | Move a rootfs-/mp-volume to a different storage or to a different container |
| `pct resize <ctid> <disk> <size>` | Resize a container mount point |
| `pct rescan` | Rescan all storages and update disk sizes and unused disk images |

### Container Listing

| Command | Description |
|---------|-------------|
| `pct list` | List all containers |
| `pct status <ctid>` | Show container status |
| `pct status <ctid> --verbose` | Show container resource usage (verbose) |

---

## PV, VG, LV Management

| Command | Description |
|---------|-------------|
| `pvcreate <disk>` | Create a PV |
| `pvremove <disk>` | Remove a PV |
| `pvs` | List all PVs |
| `vgcreate <vg> <disk>` | Create a VG |
| `vgremove <vg>` | Remove a VG |
| `vgs` | List all VGs |
| `lvcreate -L LV-SIZE -n <lv> <vg>` | Create a LV |
| `lvremove <vg>/<lv>` | Remove a LV |
| `lvs` | List all LVs |

---

## Storage Management

### Storage Commands (pvesm)

| Command | Description |
|---------|-------------|
| `pvesm add TYPE <storage>` | Create a new storage |
| `pvesm status` | Show all storage pools and usage |
| `pvesm list <storage>` | List content of a storage |
| `pvesm alloc <storage> <vmid> <name> <size>` | Allocate new disk volume |
| `pvesm free <storage>:<volume>` | Delete a volume |
| `pvesm remove <storage>` | Delete storage configuration |
| `pvesm scan nfs <server>` | Scan NFS exports on server |
| `pvesm scan iscsi <portal>` | Scan iSCSI targets |
| `pvesm scan lvm` | Scan LVM volume groups |
| `pvesm scan zfs` | Scan ZFS pools |
| `pvesm lvmscan` | An alias for pvesm scan lvm |
| `pvesm lvmthinscan` | An alias for pvesm scan lvmthin |
| `pvesm scan lvmthin <vg>` | List local LVM Thin Pools |
| `pvesm nfsscan <server>` | Scan NFS shares (alternative) |
| `pvesh get /storage` | List storage configuration via API |
| `pvesh get /nodes/<node>/storage` | Storage status on a node via API |
| `pvesh get /nodes/<node>/storage/local/content --content backup` | List backups on a storage via API |

### Adding Storage via CLI

```bash
# Add NFS storage
pvesm add nfs <storage-id> --server <ip> --export /path --content images,vztmpl,iso,backup

# Add LVM storage
pvesm add lvm <storage-id> --vgname <vg-name> --content images,rootdir

# Add LVM-Thin storage
pvesm add lvmthin <storage-id> --vgname <vg-name> --thinpool <pool-name> --content images,rootdir

# Add ZFS storage
pvesm add zfspool <storage-id> --pool <pool-name> --content images,rootdir

# Add Ceph RBD storage
pvesm add rbd <storage-id> --monhost <mon-ips> --pool <pool-name> --content images

# Add directory storage
pvesm add dir <storage-id> --path /mnt/data --content images,iso,vztmpl,backup

# Add iSCSI storage
pvesm add iscsi <storage-id> --portal <ip> --target iqn.2023-01.example.com:storage

# Remove storage (does not delete data)
pvesm remove <storage-id>
```

### Storage Content Types

| Type | Description |
|------|-------------|
| `images` | VM/CT disk images |
| `rootdir` | Container root directories |
| `vztmpl` | Container templates |
| `iso` | ISO images |
| `backup` | Backup files (vzdump) |
| `snippets` | Snippet files (cloud-init, hookscripts) |

### ZFS Commands

| Command | Description |
|---------|-------------|
| `zpool status` | Show pool health |
| `zpool list` | List pools with usage |
| `zpool scrub <pool>` | Start integrity check |
| `zfs list` | List all datasets |
| `zfs list -t snapshot` | List all snapshots |
| `zfs create <pool>/<dataset>` | Create dataset |
| `zfs snapshot <pool>/<dataset>@<name>` | Create snapshot |
| `zfs rollback <pool>/<dataset>@<name>` | Rollback to snapshot |
| `zfs destroy <pool>/<dataset>@<name>` | Delete snapshot |
| `zfs send <pool>/<dataset>@<snap> \| zfs recv <dest>` | Send/receive snapshot |

---

## Networking

### Network Configuration

```bash
# Show network interfaces
ip addr show
ip -br addr show

# List Proxmox network config
cat /etc/network/interfaces
```

### Bridge Management

```bash
# Standard bridge configuration in /etc/network/interfaces
auto vmbr0
iface vmbr0 inet static
    address 10.0.0.1/24
    gateway 10.0.0.254
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0

# VLAN-aware bridge
auto vmbr0
iface vmbr0 inet static
    address 10.0.0.1/24
    gateway 10.0.0.254
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

### Apply Network Changes

```bash
# Apply without reboot
ifreload -a

# Or restart networking service
systemctl restart networking
```

### SDN (Software-Defined Networking)

| Command | Description |
|---------|-------------|
| `pvesh get /cluster/sdn/vnets` | List VNets |
| `pvesh get /cluster/sdn/zones` | List zones |
| `pvesh get /cluster/sdn/subnets` | List subnets |
| `pvesh create /cluster/sdn/zones --zone <name> --type simple` | Create simple zone |
| `pvesh create /cluster/sdn/vnets --vnet <name> --zone <zone>` | Create VNet |

### Network Debugging

```bash
# Test connectivity
ping <gateway-ip>

# Check bridge status
brctl show

# Show routing table
ip route show
```

---

## Cluster Management

### Cluster Creation & Joining

| Command | Description |
|---------|-------------|
| `pvecm create <cluster-name>` | Create a new cluster |
| `pvecm add <ip-of-existing-node>` | Join an existing cluster |
| `pvecm status` | Show cluster status |
| `pvecm nodes` | List cluster nodes |
| `pvecm expected 1` | Set expected votes (split-brain recovery) |
| `pvecm delnode <node-name>` | Remove node from cluster |

### Cluster Filesystem

| Command | Description |
|---------|-------------|
| `pmxcfs` | Proxmox Cluster Filesystem daemon |
| `ls /etc/pve/` | Cluster configuration directory |
| `ls /etc/pve/nodes/<node>/` | Per-node configuration |
| `cat /etc/pve/corosync.conf` | View Corosync config |
| `cat /etc/pve/storage.cfg` | View storage config |
| `cat /etc/pve/.clusterlog` | View cluster log |

### High Availability (HA)

| Command | Description |
|---------|-------------|
| `ha-manager status` | Show HA status |
| `ha-manager add vm:<vmid>` | Add VM to HA |
| `ha-manager add ct:<ctid>` | Add container to HA |
| `ha-manager remove vm:<vmid>` | Remove from HA |
| `ha-manager set vm:<vmid> --state started` | Set desired state |
| `ha-manager set vm:<vmid> --group <group>` | Assign to HA group |
| `ha-manager set vm:<vmid> --max_relocate 3` | Set max relocations |
| `ha-manager set vm:<vmid> --max_restart 3` | Set max restarts |
| `ha-manager groupadd <group> --nodes node1,node2` | Create HA group |
| `ha-manager groupset <group> --restricted 1` | Restrict to group nodes only |

### Migration

| Command | Description |
|---------|-------------|
| `qm migrate <vmid> <target-node>` | Offline migration (VM must be stopped) |
| `qm migrate <vmid> <target-node> --online` | Live migration |
| `qm migrate <vmid> <target-node> --with-local-disks` | Migrate with local storage |
| `pct migrate <ctid> <target-node>` | Migrate container offline |
| `pct migrate <ctid> <target-node> --online` | Live migrate container |

---

## Backup & Restore (vzdump)

### Creating Backups

| Command | Description |
|---------|-------------|
| `vzdump <vmid>` | Backup VM/CT (default: snapshot mode) |
| `vzdump <vmid> --mode snapshot` | Snapshot-based backup (no downtime) |
| `vzdump <vmid> --mode suspend` | Suspend during backup |
| `vzdump <vmid> --mode stop` | Stop during backup (consistent) |
| `vzdump <vmid> --storage <storage>` | Backup to specific storage |
| `vzdump <vmid> --compress zstd` | Backup with zstd compression |
| `vzdump <vmid> --compress gzip` | Backup with gzip compression |
| `vzdump <vmid> --compress lzo` | Backup with lzo compression |
| `vzdump <vmid> --prune-backups keep-last=3` | Keep only the last 3 backups |
| `vzdump <vmid> --notes-template "{{guestname}}"` | Add notes to backup |
| `vzdump --all --exclude <vmid1>,<vmid2>` | Backup all except listed |
| `vzdump --all --storage <storage> --mode snapshot` | Backup all VMs and containers |

### Restoring Backups

| Command | Description |
|---------|-------------|
| `qmrestore <backup-file> <vmid>` | Restore VM from backup |
| `qmrestore <backup-file> <vmid> --storage <storage>` | Restore to specific storage |
| `qmrestore <backup-file> <vmid> --unique` | Restore with unique MAC/UUID |
| `pct restore <ctid> <backup-file>` | Restore container |
| `pct restore <ctid> <backup-file> --storage <storage>` | Restore CT to specific storage |
| `pct restore <ctid> <backup-file> --unprivileged 1` | Restore as unprivileged |

### Backup Schedules (vzdump.conf)

```bash
# View backup configuration
cat /etc/vzdump.conf

# /etc/vzdump.conf — global defaults
tmpdir: /tmp
dumpdir: /var/lib/vz/dump
storage: local
mode: snapshot
compress: zstd
prune-backups: keep-last=3
```

### Proxmox Backup Server Integration

| Command | Description |
|---------|-------------|
| `pvesm add pbs <id> --server <ip> --datastore <name> --username <user>@pbs --password <pass>` | Add PBS storage |
| `proxmox-backup-client login --repository <user>@<server>:<datastore>` | PBS client login |
| `proxmox-backup-client backup <name>.pxar:/path --repository <repo>` | Backup directory to PBS |
| `proxmox-backup-client list --repository <repo>` | List backups on PBS |
| `proxmox-backup-client restore <snapshot> <name>.pxar /target --repository <repo>` | Restore from PBS |

### Bulk Backups

```bash
# Backup multiple specific VMs
vzdump 100 101 102 --mode snapshot --compress zstd --storage local

# Backup all running VMs
qm list | awk '$3=="running" {print $1}' | xargs -I {} vzdump {} --mode snapshot --compress zstd

# Backup all running containers
pct list | awk '$2=="running" {print $1}' | xargs -I {} vzdump {} --mode snapshot --compress zstd
```

### Verify & Test Backups

```bash
# Verify backup file compression integrity
zstdcat /var/lib/vz/dump/vzdump-qemu-<vmid>-*.vma.zst > /dev/null

# Test restore to a temporary VM (full verification)
qmrestore <backup-file> 999 --unique
qm start 999
# Verify services, then cleanup:
qm stop 999 && qm destroy 999
```

### Cleanup Old Backups

```bash
# List backups older than 30 days
find /var/lib/vz/dump/ -name "vzdump-*" -mtime +30 -ls

# Delete backups older than 30 days (review with -ls first!)
find /var/lib/vz/dump/ -name "vzdump-*" -mtime +30 -delete

# Prune backups per storage retention policy
pvesm prune-backups <storage> --keep-last 3 --keep-daily 7 --keep-weekly 4

# Prune dry-run (shows what would be deleted)
pvesm prune-backups <storage> --keep-last 3 --dry-run
```

---

## Firewall

### Firewall Management

| Command | Description |
|---------|-------------|
| `pve-firewall status` | Show firewall status |
| `pve-firewall start` | Start firewall |
| `pve-firewall stop` | Stop firewall |
| `pve-firewall restart` | Restart firewall |
| `pve-firewall compile` | Compile and show iptables rules |
| `pve-firewall localnet` | Show local network info |
| `pve-firewall simulate --from outside --to host --source <ip> --dest <ip> --dport <port> --protocol tcp` | Simulate rule matching |

### Firewall Configuration Files

| File | Scope |
|------|-------|
| `/etc/pve/firewall/cluster.fw` | Cluster-wide rules |
| `/etc/pve/firewall/<vmid>.fw` | Per-VM/CT rules |
| `/etc/pve/nodes/<node>/host.fw` | Per-host rules |

### Firewall Rule Syntax (in .fw files)

```ini
[RULES]
# Direction Action [Options]
# IN/OUT    ACCEPT/DROP/REJECT  -source <cidr> -dest <cidr> -p <proto> -dport <port>

[RULES]
IN ACCEPT -source 10.0.0.0/24 -p tcp -dport 22 -log nolog
IN ACCEPT -source 10.0.0.0/24 -p tcp -dport 8006
IN DROP
OUT ACCEPT

[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT
```

---

## Cloud-Init

### Cloud-Init for VMs

| Command | Description |
|---------|-------------|
| `qm set <vmid> --ide2 <storage>:cloudinit` | Add cloud-init drive |
| `qm set <vmid> --ciuser <username>` | Set default user |
| `qm set <vmid> --cipassword <password>` | Set password |
| `qm set <vmid> --sshkeys <keyfile>` | Set SSH public keys (URL-encoded file) |
| `qm set <vmid> --ipconfig0 ip=10.0.0.5/24,gw=10.0.0.1` | Set static IP |
| `qm set <vmid> --ipconfig0 ip=dhcp` | Set DHCP |
| `qm set <vmid> --nameserver 8.8.8.8` | Set DNS nameserver |
| `qm set <vmid> --searchdomain example.com` | Set DNS search domain |
| `qm set <vmid> --cicustom "user=<storage>:snippets/user.yml"` | Custom user-data |
| `qm set <vmid> --cicustom "network=<storage>:snippets/network.yml"` | Custom network config |
| `qm cloudinit dump <vmid> <type>` | Get automatically generated cloudinit config |
| `qm cloudinit pending <vmid>` | Get the cloudinit configuration with both current and pending values |
| `qm cloudinit update <vmid>` | Regenerate and change cloudinit config drive |

---

## API Access (pvesh)

### Basic API Usage

| Command | Description |
|---------|-------------|
| `pvesh get /nodes` | List all nodes |
| `pvesh get /nodes/<node>/qemu` | List VMs on a node |
| `pvesh get /nodes/<node>/lxc` | List containers on a node |
| `pvesh get /nodes/<node>/status` | Node resource usage |
| `pvesh get /nodes/<node>/qemu/<vmid>/status/current` | VM status |
| `pvesh get /cluster/resources --type vm` | All VMs across cluster |
| `pvesh get /cluster/resources --type storage` | All storage across cluster |
| `pvesh get /cluster/tasks` | List recent tasks |
| `pvesh create /nodes/<node>/qemu/<vmid>/status/start` | Start VM via API |
| `pvesh create /nodes/<node>/qemu/<vmid>/status/stop` | Stop VM via API |
| `pvesh set /nodes/<node>/qemu/<vmid>/config --memory 4096` | Update VM config via API |

### API Tokens

```bash
# Create API token via CLI
pveum user token add <user>@<realm> <token-name> --privsep 0

# Use token with curl
curl -H "Authorization: PVEAPIToken=<user>@<realm>!<token>=<uuid>" \
  https://<host>:8006/api2/json/nodes
```

### REST API with curl

```bash
# Get authentication ticket
curl -k -d "username=root@pam&password=<pass>" \
  https://<host>:8006/api2/json/access/ticket

# Use ticket for subsequent requests
curl -k -b "PVEAuthCookie=<ticket>" -H "CSRFPreventionToken: <token>" \
  https://<host>:8006/api2/json/nodes

# Get VM list
curl -k -b "PVEAuthCookie=<ticket>" \
  https://<host>:8006/api2/json/nodes/<node>/qemu
```

---

## User & Permission Management (pveum)

| Command | Description |
|---------|-------------|
| `pveum user list` | List all users |
| `pvesh get /access/users` | List users via API |
| `pveum user add <user>@pve --password <pass>` | Add PVE user |
| `pveum user add <user>@pam` | Add PAM user |
| `pveum user modify <user>@<realm> --comment "description"` | Modify user properties |
| `pveum user modify <user>@<realm> --groups <group>` | Add user to group |
| `pveum user delete <user>@<realm>` | Delete user |
| `pveum passwd <user>@<realm>` | Change user password |
| `pveum group add <group>` | Create group |
| `pveum group set <group> --members <user>@<realm>` | Add user to group |
| `pvesh get /access/groups` | List groups via API |
| `pveum role list` | List roles |
| `pvesh get /access/roles` | List roles via API |
| `pveum role add <role> --privs "VM.Audit,VM.Console"` | Create custom role |
| `pveum acl modify / --users <user>@<realm> --roles <role>` | Assign role at path |
| `pveum acl modify /vms/<vmid> --users <user>@<realm> --roles PVEVMUser` | Grant VM access |
| `pveum acl list` | List all ACLs |

### Built-in Roles

| Role | Description |
|------|-------------|
| `Administrator` | Full admin privileges |
| `PVEAdmin` | Admin without system modifications |
| `PVEVMAdmin` | Full VM/CT management |
| `PVEVMUser` | VM console, start, stop |
| `PVEDatastoreAdmin` | Manage storage and backups |
| `PVEDatastoreUser` | Allocate disk space, view storage |
| `PVEPoolAdmin` | Manage resource pools |
| `PVEPoolUser` | View pool resources |
| `PVEAuditor` | Read-only access |
| `NoAccess` | Deny all access |

---

## Certificate Management

| Command | Description |
|---------|-------------|
| `pvenode cert info` | Show current certificate info |
| `pvenode cert set <cert.pem> <key.pem>` | Set custom certificate |
| `pvenode cert delete` | Remove custom cert (revert to self-signed) |
| `pvenode acme account register --contact <email>` | Register ACME (Let's Encrypt) |
| `pvenode acme cert order` | Order/renew ACME certificate |
| `pvenode acme cert revoke` | Revoke ACME certificate |
| `pvecm updatecerts` | Update cluster certificates |

---

## Disk & Hardware Passthrough

### PCI/GPU Passthrough

```bash
# Enable IOMMU in GRUB
# For Intel: add intel_iommu=on iommu=pt to GRUB_CMDLINE_LINUX_DEFAULT
# For AMD: add amd_iommu=on iommu=pt to GRUB_CMDLINE_LINUX_DEFAULT
nano /etc/default/grub
update-grub

# Add vfio modules
echo -e "vfio\nvfio_iommu_type1\nvfio_pci" >> /etc/modules
# Note: vfio_virqfd is not needed on kernel 6.2+ (PVE 8+)
update-initramfs -u -k all

# Blacklist GPU drivers (optional, for GPU passthrough)
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf
update-initramfs -u -k all

# Verify IOMMU groups
find /sys/kernel/iommu_groups/ -type l
```

### Attach PCI Device to VM

| Command | Description |
|---------|-------------|
| `lspci -nn` | List PCI devices with IDs |
| `qm set <vmid> --hostpci0 <bus:dev.fn>` | Pass through PCI device |
| `qm set <vmid> --hostpci0 <bus:dev.fn>,pcie=1` | Pass through as PCIe |
| `qm set <vmid> --hostpci0 <bus:dev.fn>,x-vga=1` | GPU passthrough (primary) |
| `qm set <vmid> --machine q35` | Required for PCIe passthrough |
| `qm set <vmid> --bios ovmf` | UEFI (recommended for passthrough) |

### USB Passthrough

| Command | Description |
|---------|-------------|
| `qm set <vmid> --usb0 host=<vendorid>:<productid>` | Pass through USB device by ID |
| `qm set <vmid> --usb0 host=<bus>-<port>` | Pass through USB by bus/port |
| `lsusb` | List USB devices |

---

## Task & Service Management

### Task Monitoring

| Command | Description |
|---------|-------------|
| `pvesh get /cluster/tasks` | List recent tasks |
| `pvesh get /nodes/<node>/tasks` | List node tasks |
| `pvesh get /nodes/<node>/tasks/<upid>/status` | Get task status |
| `pvesh get /nodes/<node>/tasks/<upid>/log` | Get task log |

### Key Services

| Service | Description |
|---------|-------------|
| `pvedaemon` | PVE API daemon |
| `pveproxy` | PVE Web UI proxy (port 8006) |
| `pvestatd` | PVE status daemon |
| `pve-cluster` | Cluster filesystem |
| `corosync` | Cluster communication |
| `pve-firewall` | Firewall daemon |
| `pve-ha-lrm` | HA Local Resource Manager |
| `pve-ha-crm` | HA Cluster Resource Manager |

```bash
# Restart web interface
systemctl restart pveproxy

# Restart API daemon
systemctl restart pvedaemon

# Restart cluster filesystem
systemctl restart pve-cluster

# Check all PVE services
systemctl list-units 'pve*' --type=service
```

---

## Web GUI

| Command | Description |
|---------|-------------|
| `service pveproxy restart` | Restart the Proxmox web GUI |

- Default URL: `https://<proxmox-ip>:8006`
- Default user: `root@pam`

---

## Resource Monitoring

```bash
# CPU usage
top
htop

# Memory usage
free -h

# Disk usage
df -h

# Network traffic
iftop
nethogs

# I/O statistics
iostat -x 1

# Process monitoring
ps aux | grep qemu
ps aux | grep lxc
```

---

## Troubleshooting

### Log Locations

| Log | Path |
|-----|------|
| System log | `/var/log/syslog` |
| PVE daemon | `journalctl -u pvedaemon` |
| PVE proxy | `journalctl -u pveproxy` |
| Firewall | `journalctl -u pve-firewall` |
| Cluster | `journalctl -u pve-cluster` |
| Corosync | `journalctl -u corosync` |
| Task logs | `/var/log/pve/tasks/` |
| QEMU logs | `/var/log/pve/qemu-server/<vmid>.log` |
| QEMU logs (alt) | `/var/log/qemu-server/<vmid>.log` |
| Container logs | `journalctl -u pve-container@<ctid>` |
| Firewall log | `/var/log/pve-firewall.log` |
| HA manager | `journalctl -u pve-ha-lrm` |

### Common Diagnostics

```bash
# Check node resource usage
pvesh get /nodes/$(hostname)/status

# Verify cluster health
pvecm status

# Check storage health
pvesm status

# Check cluster communication (debug)
pmxcfs -l

# View running QEMU processes
ps aux | grep qemu

# Check corosync communication
corosync-cmapctl | grep members

# Verify quorum
pvecm expected

# Force-start cluster with only this node (DANGEROUS — split-brain risk)
pvecm expected 1

# Repair cluster config after node removal
pvecm delnode <dead-node>
```

### Lock Handling

| Command | Description |
|---------|-------------|
| `qm unlock <vmid>` | Remove lock from a VM |
| `pct unlock <ctid>` | Remove lock from a container |
| `qm set <vmid> --lock <type>` | Manually set a lock (backup, snapshot, migrate, etc.) |

> **Warning:** Only unlock a VM/CT if you are certain the locking operation has failed or is no longer running.

---

## Useful One-Liners

```bash
# List all VMs and CTs with status
qm list && pct list

# Start all VMs configured with onboot
qm list | awk 'NR>1 {print $1}' | xargs -I {} qm config {} | grep -l "onboot: 1"

# Get IP addresses of all running VMs (requires guest agent)
for vmid in $(qm list | awk '/running/ {print $1}'); do
  echo "VM $vmid:"; qm guest cmd $vmid network-get-interfaces 2>/dev/null | grep -oP '"ip-address"\s*:\s*"\K[^"]+' || echo "  No agent"
done

# Bulk snapshot all running VMs
for vmid in $(qm list | awk '/running/ {print $1}'); do
  qm snapshot $vmid "auto-$(date +%Y%m%d)" --description "Automated snapshot"
done

# Find VMs using the most disk space
pvesh get /cluster/resources --type vm --output-format json | \
  python3 -c "import sys,json; [print(f'{v[\"vmid\"]:>6} {v.get(\"maxdisk\",0)/1073741824:>8.1f}G {v.get(\"name\",\"\")}') for v in sorted(json.load(sys.stdin)['data'], key=lambda x: x.get('maxdisk',0), reverse=True)]"

# Export VM config
qm config <vmid> > vm-<vmid>-config-backup.txt

# Check which node a VM is running on
pvesh get /cluster/resources --type vm --output-format json | \
  python3 -c "import sys,json; [print(f'{v[\"vmid\"]:>6} {v[\"node\"]:>12} {v.get(\"name\",\"\")}') for v in json.load(sys.stdin)['data']]"

# Start all VMs
qm list | awk 'NR>1 {print $1}' | xargs -I {} qm start {}

# Gracefully shutdown all VMs
qm list | awk 'NR>1 {print $1}' | xargs -I {} qm shutdown {}

# Start all containers
pct list | awk 'NR>1 {print $1}' | xargs -I {} pct start {}

# Gracefully shutdown all containers
pct list | awk 'NR>1 {print $1}' | xargs -I {} pct shutdown {}

# Count running VMs
qm list | awk '$3=="running"' | wc -l

# Count running containers
pct list | awk '$2=="running"' | wc -l

# List only running VMs (ID and name)
qm list | awk '$3=="running" {print $1, $2}'

# Total memory allocated to all VMs (in MB)
qm list | awk 'NR>1 {sum+=$4} END {print sum " MB"}'

# Total memory allocated across all guests (VMs + CTs)
grep -R "^memory:" /etc/pve/local/ 2>/dev/null | awk -F: '{sum+=$NF} END {print sum " MB"}'

# Find VMs using a specific storage
for vm in $(qm list | awk 'NR>1 {print $1}'); do
  qm config $vm 2>/dev/null | grep -q "local-lvm" && echo "VM $vm uses local-lvm"
done

# Clone a VM multiple times
for i in {101..105}; do
  qm clone 100 $i --name "node-$i" --full
done

# Remove all snapshots from a VM
for snap in $(qm listsnapshot <vmid> | awk 'NR>1 && !/current/ {print $2}'); do
  qm delsnapshot <vmid> $snap
done

# List sorted VMIDs from cluster
cat /etc/pve/.vmlist | python3 -c "import sys,json; [print(k) for k in sorted(json.load(sys.stdin)['ids'].keys(), key=int)]"

# Find unused disks across all guests
grep -sR "^unused" /etc/pve/nodes/

# Backup all running VMs with naming filter
qm list | grep "web" | awk '{print $1}' | xargs -I {} vzdump {} --mode snapshot --compress zstd
```

---

## Tips & Tricks

### Get Next Free VMID

```bash
pvesh get /cluster/nextid
```

### Discard / TRIM (Reclaim Thin-Provisioned Space)

VMs with thin-provisioned storage (LVM-thin, ZFS, Ceph) accumulate unused blocks. Use TRIM/discard to return space to the storage.

```bash
# VM: enable discard on disk (set in Hardware > Disk > Discard checkbox)
# Then inside the guest:
fstrim -av

# Or trigger via guest agent from the host:
qm guest exec <vmid> -- fstrim -av

# Container: trigger trim from host
pct fstrim <ctid>

# Trim all running containers (useful as a cron job)
pct list | awk '/running/ {print $1}' | while read ct; do pct fstrim $ct; done
```

> **Tip:** For VMs, most Linux distros enable `fstrim.timer` by default (weekly). Check with `systemctl status fstrim.timer` inside the guest.

### Snapshot Performance Warning

Long snapshot chains degrade VM performance significantly. Monitor and clean them regularly.

```bash
# Check snapshot count for a VM
qm listsnapshot <vmid>

# Remove all snapshots from a VM
for snap in $(qm listsnapshot <vmid> | awk 'NR>1 && !/current/ {print $2}'); do
  qm delsnapshot <vmid> $snap
done
```

### Preventing Full Storage

When thin-provisioned storage reaches 100%, VMs can freeze or corrupt. Monitor proactively.

```bash
# Alert if any storage exceeds 80% usage
pvesm status | tr -d '%' | awk 'NR>1 && $7>=80 {print "WARNING:", $1, $7"%"}'

# Cron example (add via crontab -e):
# */15 * * * * pvesm status 2>&1 | tr -d '\%' | awk '$7 >=80 {print $1,$7}' | mail -s "Storage alert" admin@example.com
```

### Installer Keyboard Shortcuts

| Shortcut | Description |
|----------|-------------|
| `Ctrl+Alt+F1` | Installer GUI |
| `Ctrl+Alt+F2` | Installer logs |
| `Ctrl+Alt+F3` | Shell/Terminal (useful for pre-install tasks) |

### IOMMU Groups (for PCI Passthrough)

```bash
# List all IOMMU groups and their devices
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
  echo "IOMMU Group ${g##*/}:"
  for d in $g/devices/*; do
    echo -e "\t$(lspci -nns ${d##*/})"
  done
done

# Quick one-liner view
lspci -vv | grep -P "\d:\d.*|IOMMU"

# Check which VMs use PCI passthrough
grep -sR "hostpci" /etc/pve/
```

### KSM (Kernel Samepage Merging)

KSM deduplicates identical memory pages across VMs. It starts at 80% host memory usage by default.

```bash
# Check KSM status
cat /sys/kernel/mm/ksm/run
cat /sys/kernel/mm/ksm/pages_shared

# Make KSM start sooner (at 70% usage) — edit /etc/ksmtuned.conf:
# KSM_THRES_COEF=30

# Make KSM more aggressive:
# KSM_NPAGES_MAX=5000
```

### Resource Planning Guidelines

| Resource | Recommendation |
|----------|---------------|
| Host RAM reserve | Leave 25% for the host and PVE overhead |
| Storage free space | Keep 10–20% free on thin-provisioned pools |
| CPU overcommit | 2:1 ratio is generally safe; 4:1 for low-usage VMs |
| Backup testing | Restore-test critical backups monthly |

### Clean System Logs

```bash
# Vacuum journal logs older than 7 days
journalctl --vacuum-time=7d

# Vacuum journal to max 500MB
journalctl --vacuum-size=500M

# Remove unused ISO images
find /var/lib/vz/template/iso/ -name "*.iso" -mtime +90 -ls
```

### Discard / TRIM Cron for VMs (via Guest Agent)

```bash
# Trim all running VMs (requires guest agent enabled)
qm list | grep "running" | awk '{print $1}' | while read vm; do
  echo "Trimming VM $vm"
  qm guest exec $vm -- fstrim -av
done

# Cron: weekly trim all running CTs (add via crontab -e)
# 30 0 * * 0 pct list | awk '/^[0-9]/ {print $1}' | while read ct; do pct fstrim $ct; done
```

> **Tip:** For VMs, also enable the `SSD Emulation` flag on the disk and set the `Discard` checkbox in Hardware > Disk settings.

### Serial Console (xterm.js) for VMs

Enable copy/paste-friendly xterm.js console for VMs:

```bash
# 1. Add serial port in VM Hardware (via GUI or CLI)
qm set <vmid> --serial0 socket

# 2. Inside the VM, add console to GRUB:
#    Edit /etc/default/grub:
#    GRUB_CMDLINE_LINUX_DEFAULT="console=ttyS0 console=tty0"
#    Then run: update-grub

# 3. Power off and start the VM (not just reboot) to apply hardware change

# Use xterm.js from the Console dropdown in the GUI
```

### Changing Node IP Address

Files to update when changing a node's IP:

```bash
# 1. Network interfaces (or use GUI: Node > System > Network)
nano /etc/network/interfaces

# 2. Hosts file (or GUI: Node > System > Hosts)
nano /etc/hosts

# 3. DNS resolver (or GUI: Node > System > DNS)
nano /etc/resolv.conf

# 4. Corosync config (if in cluster — increment config_version!)
nano /etc/pve/corosync.conf

# 5. Apply network changes
ifreload -a

# 6. Verify nothing was missed
grep -sR "old.ip.here" /etc/
```

### Prevent NIC Name Changes

NIC names can change when adding/removing PCI devices or on kernel upgrades, breaking `/etc/network/interfaces`.

```bash
# Check current NIC names and their PCI paths
ls -l /sys/class/net/*/device

# Check which driver each NIC uses
ls -l /sys/class/net/*/device/driver

# Use systemd .link files to pin NIC names persistently
# Create /etc/systemd/network/10-eno1.link:
# [Match]
# Path=pci-0000:01:00.0
# [Link]
# Name=eno1
```

### Temporary DHCP (for Network Troubleshooting)

```bash
# Temporarily use DHCP to test connectivity
ifdown vmbr0; dhclient -v

# When done testing, revert
dhclient -r; ifup vmbr0
```

### Kernel Boot Arguments

Press `E` at the boot menu to temporarily edit kernel arguments:

| Argument | Description |
|----------|-------------|
| `nomodeset` | Helps with boot hangs (especially Nvidia) |
| `debug` | Enable debugging messages |
| `fsck.mode=force` | Force filesystem check |
| `systemd.mask=pve-guests.service` | Prevent guests from starting on boot |
| `intel_iommu=on iommu=pt` | Enable IOMMU for passthrough |

### Find Unused Disks/Volumes

```bash
# Rescan to detect orphaned volumes
qm rescan
pct rescan

# Find unused disks in configs
grep -sR "^unused[0-9]" /etc/pve/

# Show paths of unused volumes
grep -sR "^unused[0-9]" /etc/pve/ | awk -F': ' '{print $2}' | xargs -I{} pvesm path {}

# Remove an unused disk from a VM
qm set <vmid> --delete unused0
```

### Shrink a Container Disk (ZFS)

```bash
# Check current usage
zfs list -o space,logicalused,refquota | grep <ctid>

# Set new smaller quota (must be larger than USED)
zfs set refquota=29G <pool>/subvol-<ctid>-disk-0

# Rescan to update config
pct rescan
```

### Monitor Disk SMART Information

```bash
# Watch temperatures across all disks
watch -cd -n5 'for i in /dev/{nvme[0-9]n1,sd[a-z]}; do
  [ -e "$i" ] && echo -e "\n[$i]" && smartctl -a $i | grep -Ei "Model|Serial|temperature"
done'

# Check for errors
for i in /dev/{nvme[0-9]n1,sd[a-z]}; do
  [ -e "$i" ] && echo "=== $i ===" && smartctl -a $i | grep -Ei "error|reallocat"
done
```

### Monitor Swap Usage by Process

```bash
# Install smem
apt install smem --no-install-suggests --no-install-recommends

# Show swap usage sorted by process
smem -atkr -s swap
```

### Check PCI Device Mapping (DRM / Disks)

```bash
# Which GPU corresponds to which renderD/card device
ls -l /sys/class/drm/*/device
# Cross-reference with: lspci | grep -i "VGA"

# Which disk is on which controller
ls -l /dev/disk/by-path/
# Cross-reference with: lspci | grep -i "SATA\|NVMe"
```

### No-Subscription Repository (PVE 9 / Debian 13)

```bash
# PVE 9 uses DEB822 format (.sources files)
cat > /etc/apt/sources.list.d/proxmox.sources << 'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

# Disable enterprise repo
sed -i '/^[^#]/s/^/#/' /etc/apt/sources.list.d/pve-enterprise.sources

apt update
```

### Enable Package Update Notifications

```bash
# Get notified about available updates via PVE notification system
pvesh set /cluster/options --notify package-updates=always
```

### Find Old Network Configs (ifupdown2)

```bash
# ifupdown2 keeps history of interfaces files
find /var/log/ifupdown2/ -name "interfaces" -ls
```

---

## Useful File Locations

| Path | Description |
|------|-------------|
| `/etc/pve/` | Cluster configuration directory |
| `/etc/pve/nodes/<node>/qemu-server/` | VM configuration files |
| `/etc/pve/nodes/<node>/lxc/` | Container configuration files |
| `/etc/pve/storage.cfg` | Storage configuration |
| `/etc/network/interfaces` | Network configuration |
| `/var/lib/vz/dump/` | Default backup location |
| `/var/lib/vz/template/` | Container templates |
| `/var/lib/vz/template/iso/` | ISO images |
| `/var/log/` | System logs |

---

## Emergency Access

If the web interface is not accessible:

```bash
# Reset root password
passwd

# Restart web proxy
systemctl restart pveproxy

# Check if services are running
systemctl status pve*
```

---

## Resources

- [Proxmox VE Official Site](https://www.proxmox.com/en)
- [Proxmox VE Wiki](https://pve.proxmox.com/wiki/Main_Page)
- [Proxmox VE Community Scripts](https://community-scripts.github.io/ProxmoxVE)
- [Terraform Provider for Proxmox](https://github.com/bpg/terraform-provider-proxmox)
- [Custom themes for Proxmox](https://github.com/IT-BAER/proxmorph)
