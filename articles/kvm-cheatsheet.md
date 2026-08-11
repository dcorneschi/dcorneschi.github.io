# KVM / virsh Cheatsheet

## VM States

| State | Description |
|-------|-------------|
| `running` | The domain is currently running on a CPU |
| `idle` | The domain is idle — waiting on I/O or has nothing to do |
| `paused` | Paused via `virsh suspend` — still consumes memory but not scheduled |
| `in shutdown` | Shutting down — guest OS notified and stopping gracefully |
| `shut off` | Not running — fully shut down or not yet started |
| `crashed` | Ended violently — only if configured not to restart on crash |
| `pmsuspended` | Suspended by guest power management (e.g., S3 sleep state) |

## VM Lifecycle

```bash
# List running VMs
virsh list

# List all VMs (including stopped)
virsh list --all

# List VMs set to autostart
virsh list --autostart

# Start a VM
virsh start <vm-name>

# Shutdown (graceful — sends ACPI signal)
virsh shutdown <vm-name>

# Force stop (like pulling the power)
virsh destroy <vm-name>

# Force stop without resorting to SIGKILL (returns error if guest doesn't stop)
virsh destroy <vm-name> --graceful

# Reboot
virsh reboot <vm-name>

# Suspend (pause in memory)
virsh suspend <vm-name>

# Resume from suspend
virsh resume <vm-name>

# Save VM state to file (hibernate)
virsh save <vm-name> /path/to/save-file

# Restore VM from saved state
virsh restore /path/to/save-file

# Reset (hard reset, no graceful shutdown)
virsh reset <vm-name>

# Send ACPI shutdown signal
virsh shutdown <vm-name> --mode acpi

# Rename a VM (must be off)
virsh domrename <old-name> <new-name>
```

## VM Creation

```bash
# Create VM from ISO
virt-install \
    --name myvm \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/myvm.qcow2,size=20 \
    --os-variant ubuntu22.04 \
    --network bridge=br0 \
    --graphics vnc,listen=0.0.0.0 \
    --cdrom /path/to/installer.iso

# Create VM with default NAT network
virt-install \
    --name myvm \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/myvm.qcow2,size=20 \
    --os-variant rocky9.0 \
    --network network=default \
    --graphics spice \
    --cdrom /path/to/installer.iso

# Headless VM (serial console, no graphics)
virt-install \
    --name headless \
    --ram 1024 \
    --vcpus 1 \
    --disk path=/var/lib/libvirt/images/headless.qcow2,size=10 \
    --os-variant generic \
    --network network=default \
    --graphics none \
    --extra-args='console=ttyS0,115200n8 serial' \
    --location /path/to/installer.iso

# Import existing disk image (no install)
virt-install \
    --name imported \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/existing.qcow2 \
    --os-variant generic \
    --network network=default \
    --import \
    --graphics vnc

# Create VM with multiple disks
virt-install \
    --name multi-disk \
    --ram 4096 \
    --vcpus 4 \
    --disk path=/var/lib/libvirt/images/root.qcow2,size=30 \
    --disk path=/var/lib/libvirt/images/data.qcow2,size=100 \
    --os-variant rocky9.0 \
    --network network=default \
    --cdrom /path/to/installer.iso

# Create VM with cloud-init (NoCloud datasource)
virt-install \
    --name cloud-vm \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/cloud.qcow2 \
    --os-variant ubuntu22.04 \
    --network network=default \
    --cloud-init user-data=/path/to/user-data.yaml \
    --import

# List available OS variants
osinfo-query os
osinfo-query os | grep -i ubuntu
osinfo-query os | grep -i rhel
osinfo-query os | grep -i rocky
```

## VM Deletion

```bash
# Undefine (remove VM definition, keep disk)
virsh undefine <vm-name>

# Undefine and remove all storage
virsh undefine <vm-name> --remove-all-storage

# Undefine with NVRAM (UEFI VMs)
virsh undefine <vm-name> --nvram

# Undefine with snapshots
virsh undefine <vm-name> --snapshots-metadata
```

## VM Information

```bash
# Detailed VM info
virsh dominfo <vm-name>

# VM XML configuration
virsh dumpxml <vm-name>

# VM disk usage
virsh domblklist <vm-name>

# VM network interfaces
virsh domiflist <vm-name>

# VM IP address (requires guest agent)
virsh domifaddr <vm-name>

# VM memory stats
virsh dommemstat <vm-name>

# VM CPU stats
virsh cpu-stats <vm-name>

# VM block device stats
virsh domblkstat <vm-name> <device>

# VM interface stats
virsh domifstat <vm-name> <interface>

# VM state
virsh domstate <vm-name>

# VM ID
virsh domid <vm-name>

# VM UUID
virsh domuuid <vm-name>

# VNC display port
virsh vncdisplay <vm-name>

# SPICE display URI
virsh domdisplay <vm-name>
```

## VM Configuration

```bash
# Edit VM XML (opens in $EDITOR)
virsh edit <vm-name>

# Edit with a specific editor
EDITOR=nano virsh edit <vm-name>

# Define a VM from XML file
virsh define /etc/libvirt/qemu/vm01.xml

# Set autostart (start on host boot)
virsh autostart <vm-name>

# Disable autostart
virsh autostart --disable <vm-name>

# Set max memory
virsh setmaxmem <vm-name> 4G --config

# Set current memory (hot-plug if supported)
virsh setmem <vm-name> 2G

# Set vCPUs (hot-plug if supported)
virsh setvcpus <vm-name> 4 --config --maximum
virsh setvcpus <vm-name> 4 --config

# Change VM title/description
virsh desc <vm-name> --title "My Web Server"
virsh desc <vm-name> "Production web server running nginx"
```

## Console and Display

```bash
# Serial console (text-based access)
virsh console <vm-name>
# Exit with: Ctrl+] or Ctrl+5

# VNC display port
virsh vncdisplay <vm-name>
# :0 means port 5900, :1 means 5901, etc.

# Find VNC port from XML
virsh dumpxml <vm-name> | grep vnc

# Print hypervisor capabilities
virsh capabilities

# Open graphical console (requires virt-viewer)
virt-viewer <vm-name>

# Open graphical console with remote host
virt-viewer -c qemu+ssh://user@host/system <vm-name>
```

## Snapshots

```bash
# Create snapshot (auto-generated name with timestamp)
virsh snapshot-create <vm-name>

# Create snapshot with specific name
virsh snapshot-create-as <vm-name> --name "before-upgrade" --description "Pre-upgrade state"

# Create snapshot of running VM (includes memory)
virsh snapshot-create-as <vm-name> --name "live-snap" --live

# List snapshots
virsh snapshot-list <vm-name>

# Show snapshot info
virsh snapshot-info <vm-name> --snapshotname "before-upgrade"

# Show snapshot XML
virsh snapshot-dumpxml <vm-name> --snapshotname "before-upgrade"

# Show current snapshot
virsh snapshot-current <vm-name>

# Revert to snapshot
virsh snapshot-revert <vm-name> --snapshotname "before-upgrade"

# Revert and start (if VM was running when snapped)
virsh snapshot-revert <vm-name> --snapshotname "before-upgrade" --running

# Delete snapshot
virsh snapshot-delete <vm-name> --snapshotname "before-upgrade"

# Delete all snapshots
virsh snapshot-list <vm-name> --name | xargs -I{} virsh snapshot-delete <vm-name> --snapshotname {}
```

## Cloning

```bash
# Clone a VM (must be shut down)
virt-clone --original <source-vm> --name <new-vm> --auto-clone

# Clone with specific disk path
virt-clone --original <source-vm> --name <new-vm> \
    --file /var/lib/libvirt/images/new-vm.qcow2

# Clone with multiple disks
virt-clone --original <source-vm> --name <new-vm> \
    --file /var/lib/libvirt/images/new-root.qcow2 \
    --file /var/lib/libvirt/images/new-data.qcow2
```

## Disk Management

```bash
# List VM disks
virsh domblklist <vm-name>

# Create a new disk image
qemu-img create -f qcow2 /var/lib/libvirt/images/new-disk.qcow2 50G

# Create disk with preallocation
qemu-img create -f qcow2 -o preallocation=metadata /var/lib/libvirt/images/disk.qcow2 100G

# Disk image info
qemu-img info /var/lib/libvirt/images/myvm.qcow2

# Resize disk image (increase only)
qemu-img resize /var/lib/libvirt/images/myvm.qcow2 +20G

# Online block resize (VM must be running)
virsh blockresize <vm-name> /var/lib/libvirt/images/myvm.qcow2 20G

# Convert format (raw to qcow2)
qemu-img convert -f raw -O qcow2 input.img output.qcow2

# Convert format (qcow2 to raw)
qemu-img convert -f qcow2 -O raw input.qcow2 output.img

# Compress qcow2 image
qemu-img convert -O qcow2 -c input.qcow2 compressed.qcow2

# Attach disk to running VM (hot-plug)
virsh attach-disk <vm-name> /var/lib/libvirt/images/extra.qcow2 vdb \
    --driver qemu --subdriver qcow2 --persistent

# Detach disk
virsh detach-disk <vm-name> vdb --persistent

# Detach disk (live + persistent)
virsh detach-disk --domain <vm-name> --persistent --live --target vdb

# Attach disk via XML
virsh attach-device <vm-name> disk.xml --persistent

# Check disk for errors
qemu-img check /var/lib/libvirt/images/myvm.qcow2
```

## Network Management

### Default NAT Network

```bash
# List networks
virsh net-list --all

# Start network
virsh net-start default

# Set autostart
virsh net-autostart default

# Stop network
virsh net-destroy default

# Network info
virsh net-info default

# Network XML config
virsh net-dumpxml default

# Edit network
virsh net-edit default

# DHCP leases
virsh net-dhcp-leases default
```

### Create Custom Network

```bash
# Define network from XML
cat > /tmp/my-network.xml << EOF
<network>
  <name>isolated</name>
  <bridge name="virbr1"/>
  <ip address="10.10.10.1" netmask="255.255.255.0">
    <dhcp>
      <range start="10.10.10.100" end="10.10.10.200"/>
    </dhcp>
  </ip>
</network>
EOF

virsh net-define /tmp/my-network.xml
virsh net-start isolated
virsh net-autostart isolated
```

### VM Network Interfaces

```bash
# List VM interfaces
virsh domiflist <vm-name>

# Get VM IP (requires qemu-guest-agent)
virsh domifaddr <vm-name>

# Attach new NIC
virsh attach-interface <vm-name> --type network --source default --model virtio --persistent

# Attach NIC to bridge
virsh attach-interface <vm-name> --type bridge --source br0 --model virtio --persistent

# Detach NIC
virsh detach-interface <vm-name> --type network --mac 52:54:00:xx:xx:xx --persistent

# Change NIC network
virsh domiflist <vm-name>
virsh detach-interface <vm-name> --type network --mac <mac-addr> --persistent
virsh attach-interface <vm-name> --type network --source new-network --model virtio --persistent
```

## Storage Pools

```bash
# List pools
virsh pool-list --all

# Pool info
virsh pool-info default

# Pool XML
virsh pool-dumpxml default

# Start pool
virsh pool-start <pool-name>

# Autostart pool
virsh pool-autostart <pool-name>

# Stop pool
virsh pool-destroy <pool-name>

# Create directory-based pool
virsh pool-define-as mypool dir - - - - /data/vms
virsh pool-build mypool
virsh pool-start mypool
virsh pool-autostart mypool

# Delete and recreate default pool
virsh pool-destroy default
virsh pool-undefine default
virsh pool-define-as --name default --type dir --target /new-path
virsh pool-autostart default
virsh pool-start default

# Refresh pool (scan for new volumes)
virsh pool-refresh <pool-name>

# Delete pool
virsh pool-destroy <pool-name>
virsh pool-undefine <pool-name>
```

### Storage Volumes

```bash
# List volumes in a pool
virsh vol-list <pool-name>

# Volume info
virsh vol-info <vol-name> --pool <pool-name>

# Create volume
virsh vol-create-as <pool-name> new-disk.qcow2 20G --format qcow2

# Delete volume
virsh vol-delete <vol-name> --pool <pool-name>

# Resize volume
virsh vol-resize <vol-name> 50G --pool <pool-name>

# Upload file to volume
virsh vol-upload <vol-name> /path/to/local/file --pool <pool-name>

# Download volume to file
virsh vol-download <vol-name> /path/to/local/file --pool <pool-name>

# Clone volume
virsh vol-clone <source-vol> <new-vol> --pool <pool-name>
```

## CPU and Memory

```bash
# View CPU info
virsh vcpuinfo <vm-name>

# Set vCPU count (config only, apply on next boot)
virsh setvcpus <vm-name> 4 --config --maximum
virsh setvcpus <vm-name> 4 --config

# Hot-add vCPUs (if supported)
virsh setvcpus <vm-name> 4 --live

# Pin vCPU to physical CPU
virsh vcpupin <vm-name> 0 2-3

# View memory
virsh dommemstat <vm-name>

# Set memory (config, apply on next boot)
virsh setmaxmem <vm-name> 8G --config
virsh setmem <vm-name> 4G --config

# Hot-add memory (if supported by guest OS)
virsh setmem <vm-name> 4G --live

# View CPU model
virsh cpu-models x86_64
```

## Migration

```bash
# Live migrate (shared storage required)
virsh migrate --live <vm-name> qemu+ssh://dest-host/system

# Live migrate with specific bandwidth (MiB/s)
virsh migrate --live --bandwidth 100 <vm-name> qemu+ssh://dest-host/system

# Offline migrate (copy disk)
virsh migrate --offline --persistent <vm-name> qemu+ssh://dest-host/system

# Migrate with disk copy (no shared storage)
virsh migrate --live --copy-storage-all <vm-name> qemu+ssh://dest-host/system

# Check migration progress
virsh domjobinfo <vm-name>

# Cancel migration
virsh domjobabort <vm-name>
```

## Guest Agent

```bash
# Check if guest agent is running
virsh qemu-agent-command <vm-name> '{"execute":"guest-info"}' 2>/dev/null && echo "Agent running" || echo "Agent not running"

# Get IP via guest agent
virsh domifaddr <vm-name> --source agent

# Freeze filesystems (for consistent backup)
virsh domfsfreeze <vm-name>

# Thaw filesystems
virsh domfsthaw <vm-name>

# Get filesystem info
virsh domfsinfo <vm-name>

# Trim/discard unused blocks
virsh domfstrim <vm-name>

# Execute command in guest (requires agent)
virsh qemu-agent-command <vm-name> '{"execute":"guest-exec","arguments":{"path":"/bin/uname","arg":["-a"]}}'
```

## Backup and Restore

```bash
# Method 1: Snapshot-based backup
virsh snapshot-create-as <vm-name> --name backup --disk-only --quiesce
cp /var/lib/libvirt/images/myvm.qcow2 /backup/myvm-backup.qcow2
virsh blockcommit <vm-name> vda --active --pivot

# Method 2: Dump XML + copy disk (VM must be off)
virsh dumpxml <vm-name> > /backup/myvm.xml
cp /var/lib/libvirt/images/myvm.qcow2 /backup/

# Restore from backup
cp /backup/myvm.qcow2 /var/lib/libvirt/images/
virsh define /backup/myvm.xml
virsh start <vm-name>

# Method 3: Save/restore (includes RAM state)
virsh save <vm-name> /backup/myvm-state
# Restore later:
virsh restore /backup/myvm-state
```

## Monitoring

```bash
# Live CPU/memory stats for all VMs
virt-top

# VM block stats
virsh domblkstat <vm-name> vda

# VM network stats
virsh domifstat <vm-name> vnet0

# All VM memory stats
for vm in $(virsh list --name); do echo "=== $vm ==="; virsh dommemstat $vm; done

# VM list with memory and CPU
virsh list --all --title

# Node (host) info
virsh nodeinfo

# Node memory stats
virsh nodememstats

# Node CPU stats
virsh nodecpustats
```

## Bulk Operations

```bash
# Start all VMs
virsh list --all --name | xargs -I{} virsh start {}

# Shutdown all running VMs
virsh list --name | xargs -I{} virsh shutdown {}

# Force stop all running VMs
virsh list --name | xargs -I{} virsh destroy {}

# Set autostart on all VMs
virsh list --all --name | xargs -I{} virsh autostart {}

# List VMs sorted by memory usage
virsh list --all | tail -n +3 | awk '{print $2}' | while read vm; do
    [ -n "$vm" ] && echo "$vm: $(virsh dommemstat $vm 2>/dev/null | grep actual | awk '{print $2/1024 " MB"}')"
done

# Snapshot all running VMs
for vm in $(virsh list --name); do
    virsh snapshot-create-as "$vm" --name "bulk-$(date +%Y%m%d)" --description "Scheduled backup"
done
```

## Remote Management

```bash
# Connect to remote host
virsh -c qemu+ssh://user@remote-host/system list --all

# Set default connection URI
export LIBVIRT_DEFAULT_URI="qemu+ssh://user@remote-host/system"

# Remote console
virt-viewer -c qemu+ssh://user@remote-host/system <vm-name>

# Copy VM to remote host (offline)
virsh dumpxml <vm-name> > vm.xml
scp vm.xml user@remote:/tmp/
scp /var/lib/libvirt/images/myvm.qcow2 user@remote:/var/lib/libvirt/images/
ssh user@remote "virsh define /tmp/vm.xml"
```

## Useful Commands

```bash
# List all VM IPs (requires guest agent)
for vm in $(virsh list --name); do
    ip=$(virsh domifaddr "$vm" --source agent 2>/dev/null | awk '/ipv4/{print $4}' | cut -d/ -f1)
    echo "$vm: ${ip:-N/A}"
done

# Find which VM uses a disk image
virsh list --all --name | while read vm; do
    [ -n "$vm" ] && virsh domblklist "$vm" 2>/dev/null | grep -q "/path/to/disk" && echo "$vm"
done

# Get total resources used by all VMs
echo "Total vCPUs: $(virsh list --name | xargs -I{} virsh vcpucount {} --current 2>/dev/null | paste -sd+ | bc)"
echo "VMs running: $(virsh list --name | wc -l)"

# Export VM as OVA (requires guestfish/libguestfs)
virsh shutdown <vm-name>
virt-v2v -i libvirt -o local -os /tmp/export -of ova <vm-name>

# Check QEMU version
qemu-system-x86_64 --version

# Check libvirt version
virsh version

# View host capabilities
virsh capabilities | head -50

# List supported OS variants
osinfo-query os | head -20
```

## libguestfs Tools (VM Filesystem Access)

Access VM disk contents without booting the VM:

```bash
# List files in a VM
virt-ls -l -d <vm-name> /etc

# Display a file from a VM
virt-cat -d <vm-name> /etc/fstab

# Edit a file in a VM (VM must be shut down)
virt-edit -d <vm-name> /etc/fstab

# Display disk usage
virt-df -h -d <vm-name>

# List filesystems
virt-filesystems -l -h -d <vm-name>

# List partitions
virt-filesystems -l -h --partitions -d <vm-name>

# Display log messages
virt-log -d <vm-name> | less
```

## Important Files and Directories

| Path | Description |
|------|-------------|
| `/var/lib/libvirt/images/` | Default VM disk images location |
| `/etc/libvirt/qemu/` | VM XML configuration files |
| `/var/log/libvirt/qemu/` | Per-VM log files (`<domain>.log`) |
| `$HOME/.virtinst/virt-install.log` | virt-install tool log |
| `$HOME/.virt-manager/virt-manager.log` | virt-manager GUI log |
| `/etc/libvirt/libvirtd.conf` | libvirt daemon configuration |
| `/etc/libvirt/qemu.conf` | QEMU driver configuration |
| `/var/run/libvirt/` | Runtime files (sockets, PID files) |

## Recipes

### Delete a VM Completely (Definition + Disk)

```bash
virsh shutdown vm01
virsh undefine vm01
virsh vol-delete --pool default vm01.qcow2
```

### Increase Memory (e.g., 1 GB → 2 GB)

```bash
virsh dominfo vm01
virsh shutdown vm01
virsh setmaxmem vm01 2048 --config
virsh setmem vm01 2048 --config
virsh dominfo vm01
virsh start vm01
```

### Increase CPUs (e.g., 1 → 2)

```bash
virsh dominfo vm01
virsh shutdown vm01
virsh setvcpus --domain vm01 --maximum 2 --config
virsh setvcpus --domain vm01 --count 2 --config
virsh dominfo vm01
virsh start vm01
```

### Clone a VM

```bash
virsh shutdown vm01
virt-clone --original vm01 --name vm01-clone --file vm01-clone.qcow2
```

### Shutdown All Running VMs

```bash
for i in $(virsh list | grep running | awk '{print $2}'); do
    virsh shutdown $i
done
```

### Add a New Disk to a Running VM

```bash
qemu-img create -f qcow2 /var/lib/libvirt/images/vm01-data.qcow2 50G
virsh attach-disk vm01 /var/lib/libvirt/images/vm01-data.qcow2 vdb \
    --driver qemu --subdriver qcow2 --persistent
```

### Migrate VM XML to Another Host

```bash
virsh dumpxml vm01 > vm01.xml
scp vm01.xml user@new-host:/tmp/
scp /var/lib/libvirt/images/vm01.qcow2 user@new-host:/var/lib/libvirt/images/
ssh user@new-host "virsh define /tmp/vm01.xml && virsh start vm01"
```

## Notes

- `virsh destroy` does an **ungraceful** immediate power-off — can corrupt guest filesystems. Use `virsh shutdown` for graceful stops. The `--graceful` flag avoids SIGKILL if the guest doesn't stop in a reasonable timeout (returns an error instead of forcing).
- `virsh undefine` removes the XML configuration. If the VM is running, it becomes a *transient* domain and the config is removed when it stops. Disk images are NOT deleted unless `--remove-all-storage` is used.
- VM XML files live in `/etc/libvirt/qemu/` — never edit them directly. Use `virsh edit` instead.
- The guest agent (`qemu-guest-agent`) must be installed inside the VM for `domifaddr --source agent`, `domfsfreeze`, and other guest-aware commands to work.

## Advanced CPU, NUMA, and Hugepages

### CPU Pinning (Emulator and IOThreads)

```bash
# Pin emulator threads to specific CPUs
virsh emulatorpin <vm-name> 0-1

# Pin IOThread to specific CPU
virsh iothreadpin <vm-name> 1 4

# Enable/disable individual vCPUs
virsh setvcpu <vm-name> 3 --enable --config
virsh setvcpu <vm-name> 3 --disable --config
```

### Hugepages (Host)

```bash
# Allocate 2MB hugepages
echo 4096 | sudo tee /proc/sys/vm/nr_hugepages

# Verify
cat /proc/meminfo | grep Huge

# Make persistent (add to /etc/sysctl.conf)
echo "vm.nr_hugepages = 4096" | sudo tee -a /etc/sysctl.conf
```

### Hugepages XML

```xml
<memoryBacking>
  <hugepages>
    <page size="2048" unit="KiB"/>
  </hugepages>
  <locked/>
</memoryBacking>
```

### CPU Passthrough XML

```xml
<cpu mode="host-passthrough">
  <feature policy="require" name="topoext"/>
</cpu>
<cputune>
  <vcpupin vcpu="0" cpuset="2-3"/>
  <emulatorpin cpuset="0-1"/>
  <iothreadpin iothread="1" cpuset="4"/>
</cputune>
```

## Block Jobs and Backups

### Live Block Commit/Copy

```bash
# Commit overlay into base (flatten chain)
virsh blockcommit <vm-name> vda --active --pivot --verbose

# Live block copy (mirror disk to new location)
virsh blockcopy <vm-name> vda /path/target.qcow2 --wait --verbose --pivot

# Check block job status
virsh blockjob <vm-name> vda --info
```

### Backing Images (Overlays)

```bash
# Create an overlay (delta on top of base image)
qemu-img create -f qcow2 -b base.qcow2 -F qcow2 overlay.qcow2

# Merge overlay back into base
qemu-img commit overlay.qcow2

# Show full backing chain
qemu-img info --backing-chain overlay.qcow2
```

### Dirty Bitmaps (Incremental Backup)

```bash
# Start incremental backup
virsh backup-begin <vm-name> --checkpoint cp1 --disks vda --incremental

# End backup
virsh backup-end <vm-name>
```

### Export Disk via NBD

```bash
# Connect disk image as block device
sudo qemu-nbd --connect=/dev/nbd0 disk.qcow2

# Access with guestfish
guestfish --ro -a /dev/nbd0 -i

# Disconnect
sudo qemu-nbd --disconnect /dev/nbd0
```

## PCI Passthrough (VFIO)

### Enable IOMMU (Host Kernel Cmdline)

Add to `/etc/default/grub` (GRUB_CMDLINE_LINUX):

```bash
# Intel
intel_iommu=on iommu=pt

# AMD
amd_iommu=on iommu=pt
```

Then: `grub2-mkconfig -o /boot/grub2/grub.cfg` and reboot.

### Bind Device to vfio-pci

```bash
# Find the device
lspci -nn | grep -i nvidia  # or your device

# Unbind from current driver
echo "0000:3b:00.0" | sudo tee /sys/bus/pci/devices/0000:3b:00.0/driver/unbind

# Bind to vfio-pci
echo "8086 10ed" | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
```

### Attach PCI Device to VM

```bash
# List PCI devices
virsh nodedev-list | grep pci

# Detach from host
virsh nodedev-detach pci_0000_3b_00_0

# Attach to VM (via XML)
virsh attach-device <vm-name> pci-device.xml --live
```

### VFIO Device XML

```xml
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x3b" slot="0x00" function="0x0"/>
  </source>
</hostdev>
```

## SR-IOV

```bash
# Create Virtual Functions
echo 4 | sudo tee /sys/class/net/enp3s0f0/device/sriov_numvfs

# Verify VFs created
lspci | grep "Virtual Function"
ip link show enp3s0f0
```

## Advanced Networking

### macvtap (Direct Physical Interface Access)

```bash
# Attach macvtap interface (guest gets own MAC on physical network)
virsh attach-interface <vm-name> --type direct --source eth0 --model virtio --live
```

> **Note:** With macvtap, the guest can communicate with the external network but NOT with the host.

### Open vSwitch

```bash
# Create OVS bridge
ovs-vsctl add-br ovsbr0
ovs-vsctl add-port ovsbr0 enp3s0

# Define libvirt network using OVS (ovs-network.xml)
virsh net-define ovs-network.xml
virsh net-start ovs
virsh net-autostart ovs
```

### Interface Tuning

```bash
# Set link state
virsh domif-setlink <vm-name> vnet0 up

# Set bandwidth limits (KiB/s)
virsh domiftune <vm-name> vnet0 --inbound 1000 --outbound 1000

# Check link state
virsh domif-getlink <vm-name> vnet0
```

## Performance Tuning

### I/O Tuning

```bash
# Set block I/O weight (100-1000)
virsh blkiotune <vm-name> --weight 800

# Block device stats
virsh domblkstat <vm-name> vda

# Check domain block errors
virsh domblkerror <vm-name>
```

### Recommended Settings

- Use `virtio` bus for disks and network (not IDE/e1000)
- Use `cache=none` with `aio=native` for best I/O
- Enable multi-queue virtio-net for high network throughput
- Use `virtio-scsi` with iothreads for multiple disks
- Enable vhost-net for network acceleration

## UEFI Boot (OVMF)

```bash
virt-install \
    --name vm-uefi \
    --ram 4096 \
    --vcpus 4 \
    --cpu host-passthrough \
    --machine q35 \
    --boot uefi \
    --disk size=40,format=qcow2,bus=virtio \
    --cdrom /path/to/installer.iso \
    --network network=default,model=virtio
```

## Cloud-Init Seed ISO

```bash
# Create seed ISO for cloud images
cloud-localds seed.iso user-data meta-data

# Attach to VM
virsh attach-disk <vm-name> seed.iso sdb --type cdrom --live
```

## virtiofs (Shared Filesystem)

### XML Configuration

```xml
<filesystem type="mount" accessmode="passthrough">
  <source dir="/srv/share"/>
  <target dir="shared"/>
  <driver type="virtiofs"/>
</filesystem>
```

### Mount Inside Guest

```bash
mount -t virtiofs shared /mnt
```

## Live Migration (Advanced)

```bash
# Set migration speed limit
virsh migrate-setspeed <vm-name> 1G

# Set maximum downtime (milliseconds)
virsh migrate-setmaxdowntime <vm-name> 200

# Live migrate with unsafe mode (skip some checks)
virsh migrate --live --persistent --unsafe <vm-name> qemu+ssh://dest/system

# Storage migration (no shared storage needed)
virsh migrate --live --copy-storage-all <vm-name> qemu+ssh://dest/system
```

## Capabilities and Conversion

```bash
# Show domain capabilities (supported features, firmwares, CPU models)
virsh domcapabilities

# Compare CPU features
virsh cpu-compare cpu-baseline.xml

# Convert libvirt XML to qemu command line
virsh domxml-to-native qemu-argv vm.xml

# Get volume path from pool
virsh vol-path --pool default disk1.qcow2
```

## Troubleshooting (Extended)

```bash
# Check libvirtd logs
journalctl -u libvirtd --no-pager -e

# Domain job info (migration/backup progress)
virsh domjobinfo <vm-name>

# Block errors
virsh domblkerror <vm-name>

# Interface link state
virsh domif-getlink <vm-name> vnet0

# Reset a stuck VM (hard reset without destroy)
virsh reset <vm-name>
```

## qemu-system Direct Launch (No libvirt)

For edge cases where libvirt doesn't support a feature:

```bash
sudo qemu-system-x86_64 \
    -enable-kvm \
    -cpu host \
    -smp 8,sockets=1,cores=8,threads=1 \
    -m 16G \
    -object memory-backend-file,id=mem0,size=16G,mem-path=/dev/hugepages,share=on \
    -numa node,memdev=mem0,cpus=0-7 \
    -drive if=virtio,file=disk.qcow2,cache=none,aio=native,format=qcow2 \
    -netdev tap,id=n0,ifname=tap0,script=no,downscript=no,vhost=on \
    -device virtio-net-pci,netdev=n0,mq=on,rx_queue_size=1024,tx_queue_size=1024 \
    -machine q35,accel=kvm \
    -display none \
    -serial mon:stdio
```
