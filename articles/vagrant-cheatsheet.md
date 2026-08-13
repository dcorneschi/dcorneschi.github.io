# Vagrant Cheatsheet

Vagrant creates and manages reproducible development environments using virtual machines. It abstracts providers (VirtualBox, libvirt/KVM, VMware, Hyper-V, Docker) behind a single workflow — `vagrant up` builds a VM from a declarative `Vagrantfile`, every time, on any machine.

## Install

```bash
# macOS (Homebrew)
brew install --cask vagrant

# Ubuntu / Debian
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant

# RHEL / Fedora
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo dnf install vagrant

# Verify
vagrant --version
```

## VM Lifecycle

```bash
# Initialize a new Vagrantfile with a base box
vagrant init ubuntu/jammy64
vagrant init generic/rhel9

# Create a minimal Vagrantfile (no comments or helpers)
vagrant init -m ubuntu/jammy64

# Force overwrite an existing Vagrantfile
vagrant init -f ubuntu/jammy64

# Start and provision the VM
vagrant up

# Start without running provisioners
vagrant up --no-provision

# Start with a specific provider
vagrant up --provider=libvirt
vagrant up --provider=virtualbox
vagrant up --provider=vmware_desktop

# SSH into the running VM
vagrant ssh

# SSH with a specific machine (multi-machine)
vagrant ssh web
vagrant ssh db

# Run a command via SSH without entering the shell
vagrant ssh -c "uname -a"
vagrant ssh -c "sudo systemctl status nginx"

# Check VM status
vagrant status

# Check all Vagrant VMs on this host
vagrant global-status
vagrant global-status --prune    # Remove stale entries

# Suspend (save state to disk)
vagrant suspend

# Resume from suspend
vagrant resume

# Halt (graceful shutdown)
vagrant halt

# Force halt (power off)
vagrant halt --force

# Restart (halt + up)
vagrant reload
vagrant reload --provision    # Also re-run provisioners

# Destroy (delete VM completely)
vagrant destroy
vagrant destroy -f            # No confirmation prompt
vagrant destroy --graceful    # Graceful shutdown before destroy
vagrant destroy 91a563f       # Destroy by machine ID (from global-status)

# Display guest→host port mappings
vagrant port
vagrant port web              # Specific machine

# Provision a running VM (re-run provisioners)
vagrant provision

# Show SSH config (useful for manual SSH or Ansible inventory)
vagrant ssh-config
vagrant ssh-config web        # Specific machine
```

## Box Management

Boxes are the base images that Vagrant uses to create VMs.

> **Browse available boxes:** https://app.vagrantup.com/boxes/search

```bash
# Add a box
vagrant box add ubuntu/jammy64
vagrant box add generic/rhel9
vagrant box add hashicorp/bionic64

# Add a specific version
vagrant box add centos/7 --box-version=1901.01

# Add from URL with a custom name
vagrant box add --name custom/rhel6 https://example.com/rhel6.box

# Add from URL
vagrant box add mybox https://example.com/box/mybox.box

# Add with specific provider
vagrant box add generic/rhel9 --provider=libvirt

# Force re-add (overwrite existing)
vagrant box add --force mybox mybox.box

# List installed boxes
vagrant box list

# Check for updates
vagrant box outdated
vagrant box outdated --global    # All boxes

# Update a box
vagrant box update
vagrant box update --box ubuntu/jammy64

# Remove a box
vagrant box remove ubuntu/jammy64
vagrant box remove centos/7 --box-version=1901.01     # Specific version
vagrant box remove centos/7 --provider=vmware_desktop  # Specific provider
vagrant box remove centos/7 --all                      # All versions

# Remove all old versions of installed boxes
vagrant box prune
vagrant box prune --dry-run

# Repackage an installed box (redistribute it)
vagrant box repackage centos/7 virtualbox 1901.01

# Package a running VM into a new box
vagrant package
vagrant package --output mybox.box
vagrant package --vagrantfile Vagrantfile.pkg    # Include Vagrantfile in box

# Export a non-Vagrant VM (e.g., from VirtualBox) as a base box
vagrant package --base "My VM Name" --output custom-base.box
```

### Popular Boxes

| Box | OS | Providers |
|-----|----|-----------| 
| `ubuntu/jammy64` | Ubuntu 22.04 | VirtualBox |
| `ubuntu/noble64` | Ubuntu 24.04 | VirtualBox |
| `generic/ubuntu2204` | Ubuntu 22.04 | VirtualBox, libvirt, VMware, Hyper-V |
| `generic/rhel9` | RHEL 9 | VirtualBox, libvirt, VMware |
| `generic/rocky9` | Rocky Linux 9 | VirtualBox, libvirt, VMware |
| `generic/debian12` | Debian 12 | VirtualBox, libvirt, VMware |
| `centos/stream9` | CentOS Stream 9 | VirtualBox, libvirt |
| `hashicorp/bionic64` | Ubuntu 18.04 | VirtualBox |

> **Tip:** Boxes from `generic/*` support multiple providers. Use them when targeting libvirt/KVM.

## Vagrantfile

### Minimal

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
end
```

### Full Example

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  config.vm.hostname = "devbox"

  # Network
  config.vm.network "private_network", ip: "192.168.56.10"
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "forwarded_port", guest: 443, host: 8443

  # Synced folder
  config.vm.synced_folder "./app", "/opt/app"
  config.vm.synced_folder ".", "/vagrant", disabled: true    # Disable default share

  # Provider settings (VirtualBox)
  config.vm.provider "virtualbox" do |vb|
    vb.name = "devbox"
    vb.memory = 2048
    vb.cpus = 2
    vb.gui = false
    vb.linked_clone = true
  end

  # Provider settings (libvirt/KVM)
  config.vm.provider "libvirt" do |lv|
    lv.memory = 2048
    lv.cpus = 2
    lv.storage_pool_name = "default"
    lv.driver = "kvm"
    lv.nested = true         # Nested virtualization
    lv.cpu_mode = "host-passthrough"
  end

  # Provisioning — Shell
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y nginx
    systemctl enable nginx
  SHELL

  # Provisioning — Ansible
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
    ansible.become = true
  end
end
```

## Networking

### Port Forwarding

```ruby
# Forward host:8080 → guest:80
config.vm.network "forwarded_port", guest: 80, host: 8080

# Forward with specific protocol
config.vm.network "forwarded_port", guest: 53, host: 5300, protocol: "udp"

# Forward with host IP binding (restrict to localhost)
config.vm.network "forwarded_port", guest: 3306, host: 3306, host_ip: "127.0.0.1"

# Auto-correct port collisions
config.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true
```

### Private Network (Host-Only)

```ruby
# Static IP
config.vm.network "private_network", ip: "192.168.56.10"

# DHCP
config.vm.network "private_network", type: "dhcp"
```

### Public Network (Bridged)

```ruby
# DHCP (bridged to host's network)
config.vm.network "public_network"

# Static IP on bridged network
config.vm.network "public_network", ip: "192.168.1.100"

# Specify bridge interface
config.vm.network "public_network", bridge: "en0: Wi-Fi (AirPort)"
```

## Synced Folders

```ruby
# Default: current directory → /vagrant in guest
# (auto-enabled unless disabled)

# Custom sync
config.vm.synced_folder "./src", "/opt/app"

# NFS (faster on macOS/Linux)
config.vm.synced_folder "./src", "/opt/app", type: "nfs"

# rsync (works everywhere, one-way)
config.vm.synced_folder "./src", "/opt/app", type: "rsync",
  rsync__exclude: [".git/", "node_modules/"]

# SMB (Windows hosts)
config.vm.synced_folder "./src", "/opt/app", type: "smb"

# VirtualBox shared folders (default for VirtualBox)
config.vm.synced_folder "./src", "/opt/app", type: "virtualbox"

# Disable default /vagrant share
config.vm.synced_folder ".", "/vagrant", disabled: true

# With mount options
config.vm.synced_folder "./src", "/opt/app",
  owner: "www-data", group: "www-data",
  mount_options: ["dmode=775", "fmode=664"]
```

## Provisioning

### Shell

```ruby
# Inline script
config.vm.provision "shell", inline: "echo Hello"

# External script
config.vm.provision "shell", path: "scripts/setup.sh"

# With arguments
config.vm.provision "shell", path: "scripts/setup.sh", args: ["arg1", "arg2"]

# Run as non-root
config.vm.provision "shell", inline: "whoami", privileged: false

# Run only once (default) vs every time
config.vm.provision "shell", inline: "echo once", run: "once"
config.vm.provision "shell", inline: "echo always", run: "always"

# Named provisioner (for selective running)
config.vm.provision "base", type: "shell", inline: "apt-get update"
config.vm.provision "app", type: "shell", path: "scripts/app.sh"
# vagrant provision --provision-with base
```

### Ansible

```ruby
# Ansible (runs from host)
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "playbook.yml"
  ansible.become = true
  ansible.inventory_path = "inventory"
  ansible.extra_vars = { env: "dev" }
  ansible.tags = "setup"
  ansible.verbose = "v"
end

# Ansible Local (runs inside guest — no Ansible needed on host)
config.vm.provision "ansible_local" do |ansible|
  ansible.playbook = "playbook.yml"
  ansible.become = true
  ansible.install_mode = "pip"    # or "default" for package manager
end
```

### Docker

```ruby
config.vm.provision "docker" do |d|
  d.pull_images "nginx"
  d.run "nginx", args: "-p 80:80 -v /opt/app:/usr/share/nginx/html:ro"
end
```

### File

```ruby
config.vm.provision "file", source: "./config/nginx.conf", destination: "/tmp/nginx.conf"
```

### Run Provisioners Selectively

```bash
# Run all provisioners
vagrant provision

# Run only specific provisioner
vagrant provision --provision-with shell
vagrant provision --provision-with ansible

# Run named provisioner
vagrant provision --provision-with base
```

## Multi-Machine

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"

  config.vm.define "web" do |web|
    web.vm.hostname = "web"
    web.vm.network "private_network", ip: "192.168.56.10"
    web.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
    end
    web.vm.provision "shell", inline: "apt-get update && apt-get install -y nginx"
  end

  config.vm.define "db" do |db|
    db.vm.hostname = "db"
    db.vm.network "private_network", ip: "192.168.56.11"
    db.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
    end
    db.vm.provision "shell", inline: "apt-get update && apt-get install -y postgresql"
  end

  config.vm.define "lb", primary: true do |lb|
    lb.vm.hostname = "lb"
    lb.vm.network "private_network", ip: "192.168.56.9"
    lb.vm.network "forwarded_port", guest: 80, host: 8080
  end
end
```

```bash
# Multi-machine commands
vagrant up              # Start all machines
vagrant up web          # Start only 'web'
vagrant ssh db          # SSH into 'db'
vagrant halt web db     # Halt specific machines
vagrant destroy -f      # Destroy all
vagrant status          # Show status of all
```

## Provider-Specific Settings

### VirtualBox

```ruby
config.vm.provider "virtualbox" do |vb|
  vb.name = "myvm"
  vb.memory = 4096
  vb.cpus = 4
  vb.gui = false
  vb.linked_clone = true
  vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  vb.customize ["modifyvm", :id, "--ioapic", "on"]
  vb.customize ["modifyvm", :id, "--audio", "none"]

  # Additional disk
  vb.customize ["createhd", "--filename", "data.vdi", "--size", 50 * 1024]
  vb.customize ["storageattach", :id, "--storagectl", "SATA Controller",
    "--port", 1, "--device", 0, "--type", "hdd", "--medium", "data.vdi"]
end
```

### libvirt (KVM)

```ruby
config.vm.provider "libvirt" do |lv|
  lv.memory = 4096
  lv.cpus = 4
  lv.driver = "kvm"
  lv.nested = true
  lv.cpu_mode = "host-passthrough"
  lv.storage_pool_name = "default"
  lv.machine_virtual_size = 50    # Disk size in GB

  # Additional disk
  lv.storage :file, size: "20G", type: "qcow2"

  # Network
  lv.management_network_name = "vagrant-mgmt"
  lv.management_network_address = "192.168.121.0/24"
end
```

### VMware (Desktop/Workstation)

```ruby
config.vm.provider "vmware_desktop" do |vmw|
  vmw.vmx["memsize"] = "4096"
  vmw.vmx["numvcpus"] = "4"
  vmw.gui = false
end
```

## Plugins

```bash
# List installed plugins
vagrant plugin list

# Install a plugin
vagrant plugin install vagrant-vbguest
vagrant plugin install vagrant-libvirt
vagrant plugin install vagrant-disksize
vagrant plugin install vagrant-hostmanager
vagrant plugin install vagrant-cachier
vagrant plugin install vagrant-vmware-desktop

# Install a plugin license (VMware — no longer required since plugin was open-sourced)
# vagrant plugin license vagrant-vmware-desktop license.lic

# Install multiple plugins at once
for p in vagrant-disksize vagrant-persistent-storage vagrant-registration; do
  vagrant plugin install $p
done

# Update plugins
vagrant plugin update
vagrant plugin update vagrant-vbguest

# Uninstall
vagrant plugin uninstall vagrant-vbguest

# Repair (fix broken plugins after Vagrant upgrade)
vagrant plugin repair
vagrant plugin expunge --reinstall    # Nuclear: remove all and reinstall
```

### Useful Plugins

| Plugin | Purpose |
|--------|---------|
| `vagrant-libvirt` | KVM/QEMU/libvirt provider |
| `vagrant-vmware-desktop` | VMware Workstation/Fusion provider |
| `vagrant-vbguest` | Auto-install VirtualBox Guest Additions |
| `vagrant-disksize` | Resize VirtualBox disks |
| `vagrant-persistent-storage` | Add persistent storage to VMs |
| `vagrant-hostmanager` | Manage `/etc/hosts` entries across machines |
| `vagrant-cachier` | Cache apt/yum packages between provisions |
| `vagrant-scp` | SCP files to/from guests |
| `vagrant-reload` | Reboot as a provisioner step |
| `vagrant-env` | Load `.env` files as variables |
| `vagrant-registration` | Auto-register RHEL systems with subscription-manager |

## Snapshots

```bash
# Save a snapshot
vagrant snapshot save mysnap
vagrant snapshot save web mysnap    # Multi-machine

# List snapshots
vagrant snapshot list

# Restore a snapshot
vagrant snapshot restore mysnap

# Delete a snapshot
vagrant snapshot delete mysnap

# Quick save/restore (push/pop stack)
vagrant snapshot push
vagrant snapshot pop
vagrant snapshot pop --no-provision
```

## One-Liners

```bash
# Create and start a VM in one shot
vagrant init ubuntu/jammy64 && vagrant up

# Quick throwaway VM
vagrant init generic/ubuntu2204 --minimal && vagrant up && vagrant ssh

# Show SSH connection details
vagrant ssh-config | grep -E "Host|Port|User|IdentityFile"

# SCP a file into the VM
vagrant ssh -c "cat > /tmp/file" < localfile.txt

# Run a script inside the VM
vagrant ssh -c "bash -s" < local-script.sh

# Get VM's IP address
vagrant ssh -c "hostname -I" | awk '{print $1}'

# Destroy all Vagrant VMs on this system
vagrant global-status | awk '/running|poweroff|saved/{print $1}' | xargs -I {} vagrant destroy {} -f

# Prune stale global-status entries
vagrant global-status --prune

# Export a running VM to a box file
vagrant package --output my-configured-box.box

# Use that box locally
vagrant box add mybox my-configured-box.box
vagrant init mybox && vagrant up
```

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `VAGRANT_HOME` | Vagrant home directory (default: `~/.vagrant.d`) |
| `VAGRANT_VAGRANTFILE` | Custom Vagrantfile name |
| `VAGRANT_DEFAULT_PROVIDER` | Default provider (e.g., `libvirt`, `virtualbox`) |
| `VAGRANT_NO_PARALLEL` | Disable parallel VM creation |
| `VAGRANT_LOG` | Log level (`debug`, `info`, `warn`, `error`) |
| `VAGRANT_DOTFILE_PATH` | Path for `.vagrant` directory |
| `VAGRANT_BOX_UPDATE_CHECK_DISABLE` | Disable box update checks |
| `VAGRANT_CHECKPOINT_DISABLE` | Disable update checks for Vagrant itself |

```bash
# Set default provider
export VAGRANT_DEFAULT_PROVIDER=libvirt

# Enable debug logging
VAGRANT_LOG=debug vagrant up

# Use a custom Vagrantfile
VAGRANT_VAGRANTFILE=Vagrantfile.dev vagrant up
```

## Debugging

```bash
# Debug SSH connection
vagrant ssh --debug
VAGRANT_LOG=debug vagrant ssh

# Debug vagrant up with full log
vagrant up --debug &> vagrant.log

# Debug with provisioning
vagrant up --provision --debug &> debug_log

# Provision with visible output + log file
vagrant up --provision | tee vagrant.log

# View debug log
tail -f vagrant.log | grep -i error
```

## Save and Restore a VM as a Box

### Export a Configured VM

```bash
# 1. Halt the VM
vagrant halt

# 2. Package it as a box file
vagrant package --output myapp.box

# 3. Add the box locally
vagrant box add --force myapp myapp.box
```

### Restore on Another Machine

```bash
# 1. Copy the .box file and Vagrantfile to the new machine
# 2. Add the box
vagrant box add myapp myapp.box

# 3. Create Vagrantfile (or restore from backup)
vagrant init myapp

# 4. Start it up
vagrant up
```

### Destroy All VMs (VMware Fusion Example)

```bash
# Destroy all running VMware VMs
for i in $(vagrant global-status | grep vmware_fusion | awk '{print $1}'); do
  vagrant destroy "$i" -f
done

# Remove all VMware boxes
for i in $(vagrant box list | grep vmware | cut -f1 -d " "); do
  vagrant box remove "$i" --provider vmware_desktop --all
done

# Destroy all VMs regardless of provider
vagrant global-status | awk '/running|poweroff|saved/{print $1}' | xargs -I {} vagrant destroy {} -f
```

## Guest VM Notes

```bash
# Vagrant creates a sudoers entry for the vagrant user
cat /etc/sudoers.d/vagrant
# Contents: vagrant ALL=(ALL) NOPASSWD: ALL

# Default insecure SSH key is replaced on first vagrant up
# Public key is injected into ~vagrant/.ssh/authorized_keys
```

## Tips & Tricks

### Speed Up Provisioning

```ruby
# Use vagrant-cachier to cache packages
if Vagrant.has_plugin?("vagrant-cachier")
  config.cache.scope = :box
  config.cache.auto_detect = true
end
```

### Conditionals in Vagrantfile

```ruby
# OS-specific settings
if Vagrant::Util::Platform.darwin?
  config.vm.synced_folder "./src", "/opt/app", type: "nfs"
else
  config.vm.synced_folder "./src", "/opt/app", type: "rsync"
end

# Only apply if plugin is installed
if Vagrant.has_plugin?("vagrant-disksize")
  config.disksize.size = "50GB"
end
```

### Variables in Vagrantfile

```ruby
# Use Ruby variables
NODES = 3
BOX = "generic/ubuntu2204"
SUBNET = "192.168.56"

Vagrant.configure("2") do |config|
  (1..NODES).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.box = BOX
      node.vm.hostname = "node#{i}"
      node.vm.network "private_network", ip: "#{SUBNET}.#{10 + i}"
      node.vm.provider "virtualbox" do |vb|
        vb.memory = 1024
      end
    end
  end
end
```

### Generate Ansible Inventory from Vagrant

```bash
# Auto-generate inventory from ssh-config
vagrant ssh-config | awk '
  /^Host / {host=$2}
  /HostName/ {ip=$2}
  /Port/ {port=$2}
  /User/ {user=$2}
  /IdentityFile/ {key=$2; printf "%s ansible_host=%s ansible_port=%s ansible_user=%s ansible_ssh_private_key_file=%s\n", host, ip, port, user, key}
'
```

### Faster Destroy + Up Cycle

```bash
# Destroy and rebuild in one line
vagrant destroy -f && vagrant up

# Rebuild only one machine
vagrant destroy web -f && vagrant up web
```

### DNS Resolution Between VMs

```ruby
# With vagrant-hostmanager plugin
config.hostmanager.enabled = true
config.hostmanager.manage_guest = true
config.hostmanager.manage_host = true
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `VBoxManage: error: VBOX_E_OBJECT_NOT_FOUND` | `vagrant global-status --prune` then `vagrant up` |
| Shared folder mount fails | Install guest additions: `vagrant plugin install vagrant-vbguest` |
| SSH connection timeout | Check provider is running, check network, try `vagrant halt && vagrant up` |
| Port collision | Add `auto_correct: true` to forwarded_port or change host port |
| Box download fails | Try manual: `vagrant box add --provider=virtualbox ubuntu/jammy64` |
| `vagrant up` hangs on network config | Check VirtualBox host-only network: `VBoxManage list hostonlyifs` |
| libvirt permission denied | Add user to `libvirt` group: `sudo usermod -aG libvirt $USER` |
| NFS export fails (macOS) | Check NFS is enabled: `sudo nfsd status`; check `/etc/exports` |
| Slow synced folders | Switch from VirtualBox shared to NFS or rsync |
| Plugin won't install after upgrade | `vagrant plugin expunge --reinstall` |

## Important Files

| Path | Purpose |
|------|---------|
| `Vagrantfile` | VM configuration (per-project) |
| `.vagrant/` | Machine state, IDs, keys (per-project, gitignore this) |
| `~/.vagrant.d/` | Vagrant home — boxes, plugins, global config |
| `~/.vagrant.d/boxes/` | Downloaded box images |
| `~/.vagrant.d/plugins.json` | Installed plugins |
| `~/.vagrant.d/Vagrantfile` | Global Vagrantfile (applies to all projects) |
| `~/.vagrant.d/insecure_private_key` | Default insecure SSH key (replaced on first boot) |

## Quick Reference

```bash
# Lifecycle
vagrant init <box>          # Create Vagrantfile
vagrant up                  # Start VM
vagrant ssh                 # Connect
vagrant halt                # Stop
vagrant destroy             # Delete
vagrant reload              # Restart

# Status
vagrant status              # This project
vagrant global-status       # All VMs

# Provisioning
vagrant provision           # Re-run provisioners
vagrant reload --provision  # Restart + provision

# Boxes
vagrant box list            # Installed boxes
vagrant box add <box>       # Download box
vagrant box update          # Update box
vagrant box prune           # Remove old versions

# Snapshots
vagrant snapshot save <name>
vagrant snapshot restore <name>
vagrant snapshot list
vagrant snapshot delete <name>

# SSH
vagrant ssh-config          # Show SSH details
vagrant ssh -c "command"    # Run command
```
