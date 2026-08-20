# Vagrantfile Examples

A collection of Vagrantfile examples demonstrating different multi-VM and single-VM configurations using VirtualBox.

## Prerequisites

- [Vagrant](https://www.vagrantup.com/downloads) installed
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads) installed
- Vagrant plugins (for Example 05):
  - `vagrant-persistent-storage`
  - `vagrant-registration`

## Example 01 — Puppet Server with Agents (CentOS 7)

Provisions a 3-node Puppet environment on CentOS 7.2:

| VM | IP Address | Memory | CPUs |
|----|-----------|--------|------|
| puppet | 192.168.10.21 | 4 GB | 2 |
| puppetagent-1 | 192.168.10.22 | default | default |
| puppetagent-2 | 192.168.10.23 | default | default |

The Puppet server is configured with autosigning enabled, firewalld rules for port 8140, and NTP. Both agents auto-register with the server on first boot.

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

config.vm.define "puppet" do |puppet|
    puppet.vm.box = "bento/centos-7.2"
    puppet.vm.network "private_network", ip: "192.168.10.21"
    puppet.vm.hostname = "puppet"
    puppet.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "4096"]
      vb.customize ["modifyvm", :id, "--cpus", "2"]
      end
    puppet.vm.provision "shell", inline: <<-SHELL
      sudo echo "192.168.10.22 puppetagent-1" | sudo tee -a /etc/hosts
      sudo echo "192.168.10.23 puppetagent-2" | sudo tee -a /etc/hosts
      sudo systemctl enable firewalld
      sudo systemctl start firewalld
      sudo firewall-cmd --permanent --zone=public --add-port=8140/tcp
      sudo yum -y install ntp
      sudo timedatectl set-timezone America/New_York
      sudo systemctl start ntpd
      sudo firewall-cmd --add-service=ntp --permanent
      sudo rpm -ivh https://yum.puppetlabs.com/puppetlabs-release-pc1-el-7.noarch.rpm
      sudo yum -y install puppetserver
      sudo touch /etc/puppetlabs/puppet/autosign.conf
      sudo echo "*" | sudo tee -a /etc/puppetlabs/puppet/autosign.conf
      sudo firewall-cmd --reload
      sudo systemctl enable puppetserver
      sudo systemctl start puppetserver
    SHELL
  end

config.vm.define "puppetagent-1" do |puppetagent1|
    puppetagent1.vm.box = "bento/centos-7.2"
    puppetagent1.vm.network "private_network", ip: "192.168.10.22"
    puppetagent1.vm.hostname = "puppetagent-1"
    puppetagent1.vm.provision "shell", inline: <<-SHELL
       sudo echo "192.168.10.21 puppet" | sudo tee -a /etc/hosts
       sudo timedatectl set-timezone America/New_York
       sudo rpm -ivh https://yum.puppetlabs.com/puppetlabs-release-pc1-el-7.noarch.rpm
       sudo yum -y install puppet-agent
       sudo /opt/puppetlabs/bin/puppet agent --test
    SHELL
    end

  config.vm.define "puppetagent-2" do |puppetagent2|
      puppetagent2.vm.box = "bento/centos-7.2"
      puppetagent2.vm.network "private_network", ip: "192.168.10.23"
      puppetagent2.vm.hostname = "puppetagent-2"
      puppetagent2.vm.provision "shell", inline: <<-SHELL
       sudo echo "192.168.10.21 puppet" | sudo tee -a /etc/hosts
       sudo timedatectl set-timezone America/New_York
       sudo rpm -ivh https://yum.puppetlabs.com/puppetlabs-release-pc1-el-7.noarch.rpm
       sudo yum -y install puppet-agent
       sudo /opt/puppetlabs/bin/puppet agent --test
    SHELL
  end
```

## Example 02 — Dynamic Multi-Node Cluster (Ubuntu 16.04)

Creates a master + N worker nodes using a loop. Node count is controlled by the `NODE_COUNT` variable.

| VM | IP Address |
|----|-----------|
| master | 10.0.0.10 |
| node1 | 10.0.0.11 |
| node2 | 10.0.0.12 |

All VMs get `avahi-daemon` installed for mDNS-based hostname resolution on the private network. Easily scalable by changing `NODE_COUNT`.

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

BOX_IMAGE = "bento/ubuntu-16.04"
NODE_COUNT = 2

Vagrant.configure("2") do |config|
  config.vm.define "master" do |subconfig|
    subconfig.vm.box = BOX_IMAGE
    subconfig.vm.hostname = "master"
    subconfig.vm.network :private_network, ip: "10.0.0.10"
  end

  (1..NODE_COUNT).each do |i|
    config.vm.define "node#{i}" do |subconfig|
      subconfig.vm.box = BOX_IMAGE
      subconfig.vm.hostname = "node#{i}"
      subconfig.vm.network :private_network, ip: "10.0.0.#{i + 10}"
    end
  end

  # Install avahi on all machines
  config.vm.provision "shell", inline: <<-SHELL
    apt-get install -y avahi-daemon libnss-mdns
  SHELL
end
```

## Example 03 — App / DB / Mail Server Stack (CentOS 8)

Spins up a 3-tier infrastructure on CentOS 8:

| VM | Hostname | IP Address |
|----|----------|-----------|
| app | appserver.srg.com | 10.0.0.101 |
| db | dbserver.srg.com | 10.0.0.102 |
| mail | mailserver.srg.com | 10.0.0.103 |

All VMs share a global VirtualBox provider config: 2 GB RAM, 2 CPUs, headless mode. Box update checks are disabled.

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box_check_update = false

  #App server
  config.vm.define "app" do |app|
    app.vm.hostname = "appserver.srg.com"
    app.vm.box = "centos/8"
    app.vm.network :private_network, ip: "10.0.0.101"
  end

  #DB server
  config.vm.define "db" do |db|
    db.vm.hostname = "dbserver.srg.com"
    db.vm.box = "centos/8"
    db.vm.network :private_network, ip: "10.0.0.102"
  end

  #Mail server
  config.vm.define "mail" do |mail|
    mail.vm.hostname = "mailserver.srg.com"
    mail.vm.box = "centos/8"
    mail.vm.network :private_network, ip: "10.0.0.103"
  end

  config.vm.provider "virtualbox" do |vb|
     vb.gui = false
     vb.memory = "2048"
     vb.cpus = 2
  end
end
```

## Example 04 — Single RHEL 8 Build Server

A single RHEL 8 VM with additional storage and Red Hat Subscription Manager integration:

- Base box: `generic/rhel8`
- 20 GB persistent data disk mounted at `/data` (LVM, ext4)
- Bridged networking via Wi-Fi
- Synced folder: host `~/Downloads` → guest `/vagrant_data`
- RHSM credentials read from environment variables `SUB_USERNAME` and `SUB_PASSWORD`
- 2 GB RAM, 2 CPUs, headless mode

### Required Plugins

```bash
vagrant plugin install vagrant-persistent-storage
vagrant plugin install vagrant-registration
```

### Usage

```bash
export SUB_USERNAME="your-rh-username"
export SUB_PASSWORD="your-rh-password"
vagrant up
```

### Vagrantfile

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  # THE VAGRANT SOURCE BASE BOX
  config.vm.box = "generic/rhel8"

  # NAME THE VM SO WE CAN IDENTIFY IT
  config.vm.define "rhelbuilds"
  config.vm.hostname = "rhelbuilds"

  # DISKS
  # use this via the vagrant-persistent-storage plugin to add other disks
  config.persistent_storage.enabled = true
  config.persistent_storage.diskdevice = '/dev/sdb'
  config.persistent_storage.location = "datadisk.vdi"
  config.persistent_storage.size = 20000
  config.persistent_storage.mountname = 'data'
  config.persistent_storage.filesystem = 'ext4'
  config.persistent_storage.mountpoint = '/data'
  config.persistent_storage.volgroupname = 'datavg'

  # NETWORKING - gets DHCP address from my local net
  config.vm.network "public_network", bridge: "en0: Wi-Fi (Wireless)"

  # SHARED HOST FOLDER
  config.vm.synced_folder "~/Downloads", "/vagrant_data"

  # RHSM BITS using the vagrant-registration plugin
  config.registration.username = ENV['SUB_USERNAME']
  config.registration.password = ENV['SUB_PASSWORD']
  #config.registration.pools = ['pool1', 'pool2']
  config.registration.unregister_on_halt = false

  # VM CUSTOMISATIONS
  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.name = "rhelbuilds"
    vb.memory = "2048"
    vb.cpus = "2"
  end

end
```

## Common Operations

```bash
# Start all VMs
vagrant up

# Start a specific VM
vagrant up puppet

# SSH into a VM
vagrant ssh puppet

# Check status
vagrant status

# Halt all VMs
vagrant halt

# Destroy all VMs
vagrant destroy -f

# Destroy a specific VM
vagrant destroy puppetagent-1 -f

# Reload (restart + re-provision)
vagrant reload --provision

# Show SSH config
vagrant ssh-config
```
