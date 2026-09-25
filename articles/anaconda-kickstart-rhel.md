# Automated RHEL Installs with Anaconda and Kickstart

Anaconda is Red Hat's installer; **kickstart** is the file that answers all of Anaconda's questions so an install runs unattended. Feed a kickstart file to the installer and it partitions disks, sets up networking, selects packages, and runs post-install scripts with no interaction — the foundation of repeatable, scriptable provisioning. This guide covers writing a kickstart file, hosting it and an install repo over HTTP, booting the installer against it, and the most useful directives.

For package management afterward, see the [dnf / yum Cheatsheet](articles/dnf-yum-cheatsheet.md); for version-specific differences, the [RHEL Releases Overview](articles/rhel-releases-overview.md); and for fleet-scale provisioning, [Registering Hosts in Foreman](articles/foreman-host-registration.md).

## Ways to Create a Kickstart File

- **By hand** — edit a `.cfg` in a text editor (most control; the approach used here).
- **From a finished install** — every install writes `/root/anaconda-ks.cfg` reflecting the choices made. Copy and adapt it.
- **Anaconda's interactive mode** — step through a manual install and reuse the generated `anaconda-ks.cfg`.
- **Red Hat Kickstart Generator** — a web form in the Customer Portal that produces a starting file.
- **`system-config-kickstart`** — an old GTK tool. It's legacy and **not available on RHEL 8+**; prefer editing `anaconda-ks.cfg` by hand.

> When loading a kickstart over the network from the kernel command line, the installer supports **NFS, HTTP(S), and FTP** only. This guide uses HTTP.

## Validate Before You Deploy

A broken kickstart wastes a full install cycle. Validate syntax with `ksvalidator` (from the `pykickstart` package) against the target version:

```sh
yum install pykickstart
ksvalidator rhel6-anaconda-ks.cfg
ksvalidator -v RHEL7 rhel7-ks.cfg     # check against a specific version's syntax
```

## Hosting the Install Tree and Kickstart over HTTP

The installer needs two things reachable over the network: the **install tree** (the DVD contents = the package repo) and the **kickstart file**. Serve both with Apache.

### Build a filesystem for the repo (optional but tidy)

```sh
pvcreate /dev/sdb
vgcreate vg_repo /dev/sdb
lvcreate -l 100%FREE -n lv_repo vg_repo
mkfs.xfs /dev/vg_repo/lv_repo          # or mkfs.ext4 on older RHEL

mkdir /repo
echo '/dev/mapper/vg_repo-lv_repo /repo xfs defaults 0 0' >> /etc/fstab
mount /repo
```

### Copy the DVD contents into the tree

```sh
mkdir -p /repo/centos/6.9/x86_64
mount -o loop /path/to/CentOS-6.9-x86_64-bin-DVD1.iso /mnt
shopt -s dotglob                       # include dotfiles like .treeinfo
cp -avr /mnt/* /repo/centos/6.9/x86_64/
umount /mnt
```

> Copy **everything**, including hidden files. The installer relies on `.treeinfo` and `.discinfo` at the top of the tree; missing them causes "not a valid install tree" errors.

### Install and enable Apache

```sh
yum install httpd
# RHEL 5-6
chkconfig httpd on && service httpd start
# RHEL 7+
systemctl enable --now httpd

mkdir /repo/kickstart
chown apache:apache /repo/kickstart
chmod 755 /repo/kickstart              # apache must be able to read/traverse
```

### Apache access config differs by version

**RHEL 6** (Apache 2.2) uses `Order/Allow`:

```apache
# /etc/httpd/conf/httpd.conf
Alias /repo "/repo"
<Directory "/repo">
    Options Indexes FollowSymLinks
    AllowOverride None
    Order allow,deny
    Allow from all
</Directory>
```

```sh
service httpd restart
```

**RHEL 7+** (Apache 2.4) uses `Require`:

```apache
# /etc/httpd/conf.d/repo.conf
Alias /repo "/repo"
<Directory "/repo">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
    # or restrict to a subnet:
    # Require ip 192.168.1.0/24
</Directory>
```

```sh
systemctl restart httpd
```

## Booting the Installer Against a Kickstart

At the boot menu, edit the boot line (press **Tab** on BIOS menus, **e** on GRUB2) and append the kickstart URL. **The parameter name changed between major versions:**

### RHEL/CentOS 6 — `ks=`

```sh
linux ks=http://server/kickstart/centos6.cfg
linux ks=http://server/kickstart/rhel6.cfg selinux=0
linux ks=http://server/kickstart/centos6.cfg ksdevice=eth0 \
      ip=192.168.1.23 netmask=255.255.255.0 gateway=192.168.1.1 dns=192.168.1.1
```

### RHEL/CentOS 7+ — `inst.ks=`

```sh
inst.ks=http://server/kickstart/centos7.cfg
# keep legacy ethX names instead of predictable names (enpXsY):
inst.ks=http://server/kickstart/centos7.cfg net.ifnames=0 biosdevname=0
```

> The `inst.` prefix (`inst.ks=`, `inst.repo=`) was introduced in RHEL 7; the bare `ks=` form is RHEL 6 and earlier. Mixing them up is the most common "my kickstart is ignored" cause.

### SELinux gotcha on RHEL 6

On RHEL 6, `selinux --disabled` in the kickstart is **ignored** during installation. To disable SELinux for the install itself, append `selinux=0` to the boot line (as shown above).

## Anaconda Post-Install Console

On RHEL 7+, `%post` script output appears on **virtual console 5** (in the installer's tmux: `Ctrl+b` then `5`). Handy for watching a post-script or debugging a hang.

## Kickstart Directives Reference

A kickstart file has three parts: **commands** (top), a `%packages` section, and optional `%pre`/`%post` scripts.

### Point at the install source

```sh
url --url=http://192.168.1.22/repo/rhel6      # network install tree
# extra repos available during install:
repo --name=updates --baseurl=http://repo.example.com/centos/6/updates/x86_64/
repo --name=epel    --baseurl=http://repo.example.com/epel/6/x86_64/
```

### Disk partitioning — LVM

```sh
zerombr                                        # clear the MBR
bootloader --location=mbr
part /boot --fstype=ext4 --size=512
part pv.01 --size=1 --grow --ondisk=sda
volgroup vg_root --pesize=4096 pv.01
logvol /     --fstype=ext4 --size=1024 --vgname=vg_root --name=lv_root
logvol /usr  --fstype=ext4 --size=3072 --vgname=vg_root --name=lv_usr
logvol /var  --fstype=ext4 --size=5120 --vgname=vg_root --name=lv_var
logvol /tmp  --fstype=ext4 --size=1024 --vgname=vg_root --name=lv_tmp
logvol /home --fstype=ext4 --size=512  --vgname=vg_root --name=lv_home
logvol swap  --fstype=swap --size=1024 --vgname=vg_root --name=lv_swap
```

> A swap logvol must be `--fstype=swap`, not `ext4`. To size swap by hardware, use `part swap --recommended` (non-LVM) or compute it in `%pre` (below).

### Disk partitioning — plain partitions

```sh
part /boot --fstype=ext4 --size=512
part swap  --fstype=swap --recommended         # size based on RAM
part /     --fstype=ext4 --size=1024 --grow
part /var  --fstype=ext4 --size=5120
part /usr  --fstype=ext4 --size=3072
part /home --fstype=ext4 --size=512
part /tmp  --fstype=ext4 --size=1024
```

### Bootloader with an encrypted password

Generate a hash and reference it (never store a plaintext password):

```sh
# older GRUB: grub-md5-crypt   |  GRUB2/newer: grub2-mkpasswd-pbkdf2
bootloader --location=mbr --md5pass=$1$uogZn/$zgx4e3ZQ/Qe8S/JcDGEg6/
```

### Networking

```sh
network --device=eth0 --bootproto=dhcp --hostname=web01.example.com
network --device=eth0 --bootproto=static --ip=192.168.1.23 \
        --netmask=255.255.255.0 --gateway=192.168.1.1 --nameserver=192.168.1.1
network --device=eth0 --bootproto=query        # prompt for network config
```

### Services and firewall

```sh
services --enabled=sshd,ntpd,chronyd
firewall --enabled --ssh --http
firewall --service=ssh --service=smtp --port=143:tcp,80:tcp,443:tcp
```

### Other common commands

```sh
skipx                     # don't configure the X Window System
logging --level=info      # install log verbosity
```

### Package selection

```sh
%packages
@core
@base
@ X Window System
@ GNOME Desktop Environment
httpd
vim-enhanced
-NetworkManager           # a leading - excludes a package
%end
```

Group names are prefixed with `@`; individual packages are listed plainly; `-name` excludes.

## %pre and %post Scripts

`%pre` runs before installation (useful for computing partitioning); `%post` runs after, in the installed system's context.

### Compute swap in %pre and include a generated file

```sh
%pre
mem=$(grep MemTotal /proc/meminfo | awk '{print $2}')   # KB
swap=$(( mem / 1000 * 2 ))
[ "$swap" -gt 3000 ] && swap=3000                        # cap at ~3 GB
cat > /tmp/storage.cfg <<CFG
logvol swap --fstype=swap --size=$swap --vgname=vg_root --name=lv_swap
CFG
%end
```

Then pull it into the command section with `%include`:

```sh
%include /tmp/storage.cfg
# %include also works over the network:
%include http://instsvr.example.com/scripts/post-config
```

### A typical %post

```sh
%post --log=/root/postinstall.log
chvt 3                                           # switch to a visible console
# fetch and run an external post script
wget -q -O /tmp/postinstall.sh http://192.168.1.100/scripts/postinstall.sh
bash /tmp/postinstall.sh

# write resolv.conf
cat > /etc/resolv.conf <<EOF
domain example.com
nameserver 192.168.1.1
nameserver 192.168.1.2
EOF
chvt 1
%end
```

> Use `%post --log=/root/postinstall.log` so the script's output is captured into the installed system automatically — cleaner than redirecting by hand.

## Key Takeaways

- Kickstart automates Anaconda: partitioning, networking, packages, and `%pre`/`%post` scripts, unattended.
- **Boot parameter differs by version:** `ks=URL` on RHEL 6, `inst.ks=URL` on RHEL 7+.
- Serve both the **install tree** (full DVD copy, including `.treeinfo`) and the kickstart over HTTP/NFS/FTP.
- Apache access syntax differs: `Order/Allow` (RHEL 6 / httpd 2.2) vs `Require` (RHEL 7+ / httpd 2.4).
- On RHEL 6, `selinux --disabled` is ignored during install — add `selinux=0` to the boot line.
- **Always** `ksvalidator` the file first, and watch `%post` on console 5 (tmux `Ctrl+b 5`) on RHEL 7+.

## Quick Reference

```sh
# Validate
ksvalidator ks.cfg

# Copy a DVD into an HTTP-served tree (include dotfiles!)
mount -o loop rhel.iso /mnt && shopt -s dotglob && cp -avr /mnt/* /repo/rhel7/

# Boot the installer against the kickstart
linux ks=http://server/kickstart/rhel6.cfg selinux=0        # RHEL 6
inst.ks=http://server/kickstart/rhel7.cfg net.ifnames=0     # RHEL 7+

# Kickstart essentials
url --url=http://192.168.1.22/repo/rhel7
network --bootproto=dhcp --device=eth0 --hostname=host01
%packages
@core
%end
```

## Links

- [CentOS Community Kickstarts](https://github.com/CentOS/Community-Kickstarts)

For related material, see the [dnf / yum Cheatsheet](articles/dnf-yum-cheatsheet.md), the [RHEL Releases Overview](articles/rhel-releases-overview.md), and [Registering Hosts in Foreman](articles/foreman-host-registration.md).
