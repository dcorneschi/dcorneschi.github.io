# VirtualBox CLI Cheatsheet

VBoxManage is the command-line interface for Oracle VirtualBox. It provides full control over VM lifecycle, storage, networking, snapshots, and configuration.

## Add VBoxManage to PATH

```bash
# Windows CMD
SET PATH=%PATH%;C:\Program Files\Oracle\VirtualBox

# Windows PowerShell
$env:PATH = $env:PATH + ";C:\Program Files\Oracle\VirtualBox"

# Git Bash / MSYS2 on Windows
export PATH=$PATH:/C/Program\ Files/Oracle/VirtualBox

# macOS (Homebrew installs to /usr/local/bin automatically)
# Manual install path:
export PATH=$PATH:/Applications/VirtualBox.app/Contents/MacOS

# Linux (usually in PATH after package install)
which VBoxManage
```

## VM Lifecycle

### List VMs

```bash
# List all registered VMs
VBoxManage list vms

# List running VMs
VBoxManage list runningvms

# Detailed info about a VM
VBoxManage showvminfo "MyVM"

# Machine-readable output
VBoxManage showvminfo "MyVM" --machinereadable
```

### Create VM

```bash
# Create and register a VM
VBoxManage createvm --name "MyVM" --ostype Ubuntu_64 --register

# List available OS types
VBoxManage list ostypes

# Create with specific folder
VBoxManage createvm --name "MyVM" --ostype RedHat_64 --basefolder /path/to/vms --register
```

### Modify VM Settings

```bash
# Set memory (MB)
VBoxManage modifyvm "MyVM" --memory 4096

# Set CPUs
VBoxManage modifyvm "MyVM" --cpus 2

# Set VRAM (MB)
VBoxManage modifyvm "MyVM" --vram 128

# Set boot order
VBoxManage modifyvm "MyVM" --boot1 dvd --boot2 disk --boot3 none --boot4 none

# Enable I/O APIC (required for 64-bit guests)
VBoxManage modifyvm "MyVM" --ioapic on

# Enable PAE/NX
VBoxManage modifyvm "MyVM" --pae on

# Set clipboard mode
VBoxManage modifyvm "MyVM" --clipboard-mode bidirectional

# Set drag and drop
VBoxManage modifyvm "MyVM" --draganddrop bidirectional

# Set description
VBoxManage modifyvm "MyVM" --description "Web server development VM"
```

### Start / Stop / Pause

```bash
# Start VM (GUI mode)
VBoxManage startvm "MyVM"

# Start headless (no GUI window)
VBoxManage startvm "MyVM" --type headless

# Start in detachable mode (can detach GUI later)
VBoxManage startvm "MyVM" --type separate

# Pause VM
VBoxManage controlvm "MyVM" pause

# Resume VM
VBoxManage controlvm "MyVM" resume

# Save state (hibernate)
VBoxManage controlvm "MyVM" savestate

# Power off (hard shutdown)
VBoxManage controlvm "MyVM" poweroff

# Send ACPI shutdown signal (graceful)
VBoxManage controlvm "MyVM" acpipowerbutton

# Reset (hard reboot)
VBoxManage controlvm "MyVM" reset
```

### Delete VM

```bash
# Unregister and delete all files
VBoxManage unregistervm "MyVM" --delete

# Unregister only (keep files)
VBoxManage unregistervm "MyVM"
```

## Storage

### Create Virtual Disk

```bash
# Create VDI disk (dynamically allocated)
VBoxManage createmedium disk --filename /path/to/disk.vdi --size 50000 --format VDI

# Create fixed-size disk
VBoxManage createmedium disk --filename /path/to/disk.vdi --size 50000 --format VDI --variant Fixed

# Create VMDK disk
VBoxManage createmedium disk --filename /path/to/disk.vmdk --size 50000 --format VMDK

# Resize a disk (MB)
VBoxManage modifymedium disk /path/to/disk.vdi --resize 100000

# Compact a disk (reclaim space)
VBoxManage modifymedium disk /path/to/disk.vdi --compact

# List registered media
VBoxManage list hdds
VBoxManage list dvds
VBoxManage list floppies
```

### Storage Controllers

```bash
# Add SATA controller
VBoxManage storagectl "MyVM" --name "SATA" --add sata --controller IntelAhci

# Add IDE controller
VBoxManage storagectl "MyVM" --name "IDE" --add ide

# Add SCSI controller
VBoxManage storagectl "MyVM" --name "SCSI" --add scsi

# Add NVMe controller
VBoxManage storagectl "MyVM" --name "NVMe" --add pcie --controller NVMe

# Set port count
VBoxManage storagectl "MyVM" --name "SATA" --portcount 4

# Remove controller
VBoxManage storagectl "MyVM" --name "SATA" --remove
```

### Attach Media

```bash
# Attach disk to SATA port 0
VBoxManage storageattach "MyVM" --storagectl "SATA" --port 0 --device 0 \
    --type hdd --medium /path/to/disk.vdi

# Attach ISO to IDE
VBoxManage storageattach "MyVM" --storagectl "IDE" --port 0 --device 0 \
    --type dvddrive --medium /path/to/installer.iso

# Detach media (eject)
VBoxManage storageattach "MyVM" --storagectl "IDE" --port 0 --device 0 \
    --type dvddrive --medium emptydrive

# Attach Guest Additions ISO
VBoxManage storageattach "MyVM" --storagectl "IDE" --port 0 --device 0 \
    --type dvddrive --medium additions
```

## Networking

### Network Adapters

```bash
# NAT (default — VM can access internet, host can't reach VM)
VBoxManage modifyvm "MyVM" --nic1 nat

# Bridged (VM gets IP on same network as host)
VBoxManage modifyvm "MyVM" --nic1 bridged --bridgeadapter1 "en0: Wi-Fi"

# Host-only (VM-to-host only, no internet)
VBoxManage modifyvm "MyVM" --nic1 hostonly --hostonlyadapter1 "vboxnet0"

# Internal network (VM-to-VM only)
VBoxManage modifyvm "MyVM" --nic1 intnet --intnet1 "mynetwork"

# NAT Network (multiple VMs share NAT with inter-VM communication)
VBoxManage modifyvm "MyVM" --nic1 natnetwork --nat-network1 "NatNetwork"

# Disable adapter
VBoxManage modifyvm "MyVM" --nic1 none

# Set adapter type
VBoxManage modifyvm "MyVM" --nictype1 virtio
VBoxManage modifyvm "MyVM" --nictype1 82545EM

# Set MAC address
VBoxManage modifyvm "MyVM" --macaddress1 080027A1B2C3

# Enable promiscuous mode
VBoxManage modifyvm "MyVM" --nicpromisc1 allow-all
```

### Port Forwarding (NAT)

```bash
# Add port forward (host:2222 → guest:22)
VBoxManage modifyvm "MyVM" --natpf1 "ssh,tcp,,2222,,22"

# Add HTTP forward
VBoxManage modifyvm "MyVM" --natpf1 "http,tcp,,8080,,80"

# Add HTTPS forward
VBoxManage modifyvm "MyVM" --natpf1 "https,tcp,,8443,,443"

# Delete port forward
VBoxManage modifyvm "MyVM" --natpf1 delete "ssh"

# List port forwards
VBoxManage showvminfo "MyVM" | grep "NIC.*Rule"
```

### NAT Networks

```bash
# Create NAT network
VBoxManage natnetwork add --netname "NatNetwork" --network "10.0.2.0/24" --enable --dhcp on

# List NAT networks
VBoxManage list natnets

# Add port forward to NAT network
VBoxManage natnetwork modify --netname "NatNetwork" --port-forward-4 "ssh:tcp:[]:2222:[10.0.2.5]:22"

# Delete NAT network
VBoxManage natnetwork remove --netname "NatNetwork"
```

### Host-Only Networks

```bash
# Create host-only interface
VBoxManage hostonlyif create

# Configure host-only interface
VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0

# List host-only interfaces
VBoxManage list hostonlyifs

# Remove host-only interface
VBoxManage hostonlyif remove vboxnet0

# Enable DHCP on host-only network
VBoxManage dhcpserver add --ifname vboxnet0 --ip 192.168.56.100 \
    --netmask 255.255.255.0 --lowerip 192.168.56.101 --upperip 192.168.56.254 --enable
```

## Snapshots

```bash
# Take a snapshot
VBoxManage snapshot "MyVM" take "clean-install" --description "Fresh OS install"

# List snapshots
VBoxManage snapshot "MyVM" list

# Restore a snapshot
VBoxManage snapshot "MyVM" restore "clean-install"

# Restore to current (latest) snapshot
VBoxManage snapshot "MyVM" restorecurrent

# Delete a snapshot
VBoxManage snapshot "MyVM" delete "clean-install"

# Show snapshot info
VBoxManage snapshot "MyVM" showvminfo "clean-install"

# Edit snapshot description
VBoxManage snapshot "MyVM" edit "clean-install" --description "Updated description"
```

## Cloning

```bash
# Full clone
VBoxManage clonevm "MyVM" --name "MyVM-Clone" --register

# Linked clone (shares base disk, saves space)
VBoxManage clonevm "MyVM" --name "MyVM-Linked" --options link --register

# Clone specific snapshot
VBoxManage clonevm "MyVM" --name "MyVM-Clone" --snapshot "clean-install" --options link --register

# Clone to specific folder
VBoxManage clonevm "MyVM" --name "MyVM-Clone" --basefolder /path/to/vms --register
```

## Import / Export

### OVA/OVF

```bash
# Import OVA/OVF
VBoxManage import appliance.ova

# Import with settings override
VBoxManage import appliance.ova --vsys 0 --vmname "NewName" --memory 4096 --cpus 2

# Dry run (show what would be imported)
VBoxManage import appliance.ova --dry-run

# Export to OVA
VBoxManage export "MyVM" --output myvm.ova

# Export multiple VMs
VBoxManage export "VM1" "VM2" --output multi.ova

# Export with manifest
VBoxManage export "MyVM" --output myvm.ova --manifest
```

### Disk Conversion

```bash
# Convert VMDK to VDI
VBoxManage clonemedium disk source.vmdk target.vdi --format VDI

# Convert VDI to VMDK
VBoxManage clonemedium disk source.vdi target.vmdk --format VMDK

# Convert raw to VDI
VBoxManage convertfromraw disk.img disk.vdi --format VDI

# Convert with variant
VBoxManage clonemedium disk source.vdi target.vdi --variant Fixed
```

## Guest Control (Guest Additions Required)

```bash
# Run a command in the guest
VBoxManage guestcontrol "MyVM" run --exe "/bin/ls" --username user --password pass -- -la /tmp

# Copy file from host to guest
VBoxManage guestcontrol "MyVM" copyto --target-directory /tmp/ /path/on/host/file.txt \
    --username user --password pass

# Copy file from guest to host
VBoxManage guestcontrol "MyVM" copyfrom --target-directory /tmp/ /path/in/guest/file.txt \
    --username user --password pass

# Create directory in guest
VBoxManage guestcontrol "MyVM" mkdir /tmp/newdir --username user --password pass

# List running processes in guest
VBoxManage guestcontrol "MyVM" process list --username user --password pass

# Check Guest Additions status
VBoxManage guestproperty enumerate "MyVM" | grep "GuestAdd"
```

## Shared Folders

```bash
# Add shared folder
VBoxManage sharedfolder add "MyVM" --name "shared" --hostpath /path/on/host --automount

# Add read-only shared folder
VBoxManage sharedfolder add "MyVM" --name "shared" --hostpath /path/on/host --readonly --automount

# Add with specific mount point
VBoxManage sharedfolder add "MyVM" --name "shared" --hostpath /path/on/host \
    --automount --auto-mount-point /mnt/shared

# Remove shared folder
VBoxManage sharedfolder remove "MyVM" --name "shared"

# List shared folders
VBoxManage showvminfo "MyVM" | grep "Shared folders"
```

## USB

```bash
# Enable USB 2.0 (EHCI)
VBoxManage modifyvm "MyVM" --usbehci on

# Enable USB 3.0 (xHCI)
VBoxManage modifyvm "MyVM" --usbxhci on

# List USB devices on host
VBoxManage list usbhost

# Add USB filter (auto-attach device to VM)
VBoxManage usbfilter add 0 --target "MyVM" --name "MyUSB" \
    --vendorid 1234 --productid 5678

# Remove USB filter
VBoxManage usbfilter remove 0 --target "MyVM"
```

## Display and Remote

```bash
# Set VRAM
VBoxManage modifyvm "MyVM" --vram 256

# Enable 3D acceleration
VBoxManage modifyvm "MyVM" --accelerate3d on

# Set video capture
VBoxManage modifyvm "MyVM" --recording on --recordingfile /path/to/video.webm

# Enable VRDE (VirtualBox Remote Desktop Extension)
VBoxManage modifyvm "MyVM" --vrde on
VBoxManage modifyvm "MyVM" --vrdeport 3389
VBoxManage modifyvm "MyVM" --vrdeauthtype null

# Set screen resolution
VBoxManage controlvm "MyVM" setvideomodehint 1920 1080 32

# Take screenshot
VBoxManage controlvm "MyVM" screenshotpng /tmp/screenshot.png
```

## Miscellaneous

### Properties and Metrics

```bash
# List system properties
VBoxManage list systemproperties

# List guest properties
VBoxManage guestproperty enumerate "MyVM"

# Get specific guest property
VBoxManage guestproperty get "MyVM" "/VirtualBox/GuestInfo/Net/0/V4/IP"

# Get guest IP address
VBoxManage guestproperty get "MyVM" "/VirtualBox/GuestInfo/Net/0/V4/IP" | awk '{print $2}'

# Metrics (CPU, memory, disk, network)
VBoxManage metrics setup --period 1 --samples 5 "MyVM"
VBoxManage metrics query "MyVM"
```

### Logging and Debugging

```bash
# VM log files location
VBoxManage showvminfo "MyVM" | grep "Log folder"

# Set extra data
VBoxManage setextradata "MyVM" "GUI/ScaleFactor" "2.0"

# Get extra data
VBoxManage getextradata "MyVM" "GUI/ScaleFactor"

# Enable serial port (for console debugging)
VBoxManage modifyvm "MyVM" --uart1 0x3F8 4 --uartmode1 file /tmp/vm-console.log
```

### DHCP Servers

```bash
# List DHCP servers
VBoxManage list dhcpservers

# Add DHCP server
VBoxManage dhcpserver add --netname "intnet" --ip 10.0.0.1 \
    --netmask 255.255.255.0 --lowerip 10.0.0.100 --upperip 10.0.0.200 --enable

# Modify DHCP server
VBoxManage dhcpserver modify --netname "intnet" --lowerip 10.0.0.50 --upperip 10.0.0.250

# Remove DHCP server
VBoxManage dhcpserver remove --netname "intnet"
```

## Batch Operations

```bash
# Start all VMs headless
for vm in $(VBoxManage list vms | awk -F'"' '{print $2}'); do
    VBoxManage startvm "$vm" --type headless
done

# Power off all running VMs
for vm in $(VBoxManage list runningvms | awk -F'"' '{print $2}'); do
    VBoxManage controlvm "$vm" acpipowerbutton
done

# Take snapshot of all running VMs
for vm in $(VBoxManage list runningvms | awk -F'"' '{print $2}'); do
    VBoxManage snapshot "$vm" take "backup-$(date +%Y%m%d)" --description "Daily backup"
done

# List all VMs with their state
VBoxManage list vms --long | grep -E "^Name:|^State:"
```

## Quick Reference

| Action | Command |
|--------|---------|
| List VMs | `VBoxManage list vms` |
| List running | `VBoxManage list runningvms` |
| VM info | `VBoxManage showvminfo "VM"` |
| Create VM | `VBoxManage createvm --name "VM" --ostype Ubuntu_64 --register` |
| Start (headless) | `VBoxManage startvm "VM" --type headless` |
| Graceful shutdown | `VBoxManage controlvm "VM" acpipowerbutton` |
| Power off | `VBoxManage controlvm "VM" poweroff` |
| Save state | `VBoxManage controlvm "VM" savestate` |
| Take snapshot | `VBoxManage snapshot "VM" take "name"` |
| Restore snapshot | `VBoxManage snapshot "VM" restore "name"` |
| Clone VM | `VBoxManage clonevm "VM" --name "Clone" --register` |
| Port forward | `VBoxManage modifyvm "VM" --natpf1 "ssh,tcp,,2222,,22"` |
| Import OVA | `VBoxManage import file.ova` |
| Export OVA | `VBoxManage export "VM" --output file.ova` |
| Create disk | `VBoxManage createmedium disk --filename d.vdi --size 50000` |
| Resize disk | `VBoxManage modifymedium disk d.vdi --resize 100000` |
| Get guest IP | `VBoxManage guestproperty get "VM" "/VirtualBox/GuestInfo/Net/0/V4/IP"` |
| Screenshot | `VBoxManage controlvm "VM" screenshotpng /tmp/shot.png` |
