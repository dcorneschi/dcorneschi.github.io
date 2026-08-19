# Linux NFS Cheatsheet

Network File System (NFS) allows sharing directories over a network. The server exports directories, and clients mount them as if they were local filesystems. NFS is the standard for shared storage in Linux environments — from homelabs to enterprise clusters.

## NFS Versions

| Version | Features |
|---------|----------|
| NFSv3 | Stateless, UDP/TCP, separate mount protocol, no built-in security |
| NFSv4 | Stateful, TCP only, single port (2049), Kerberos support, ACLs |
| NFSv4.1 | Sessions, pNFS (parallel NFS), trunking, directory delegation |
| NFSv4.2 | Server-side copy, sparse files, space reservations, labeled NFS (SELinux) |

## Installation

### Server (RHEL / Rocky / AlmaLinux)

```bash
# Install NFS server packages
sudo dnf install -y nfs-utils

# Enable and start services
sudo systemctl enable --now nfs-server rpcbind

# Verify services
sudo systemctl status nfs-server
rpcinfo -p | grep nfs
```

### Server (Ubuntu / Debian)

```bash
# Install NFS server packages
sudo apt install -y nfs-kernel-server

# Enable and start services
sudo systemctl enable --now nfs-kernel-server

# Verify services
sudo systemctl status nfs-kernel-server
```

### Client (RHEL / Rocky / AlmaLinux)

```bash
sudo dnf install -y nfs-utils
```

### Client (Ubuntu / Debian)

```bash
sudo apt install -y nfs-common
```

### Arch Linux

```bash
sudo pacman -S nfs-utils
```

## Service Management

### Start/Enable NFS Services

```bash
# Ubuntu/Debian
sudo systemctl start nfs-kernel-server
sudo systemctl enable nfs-kernel-server
sudo systemctl start rpcbind
sudo systemctl enable rpcbind

# RHEL/CentOS/Rocky
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
sudo systemctl start rpcbind
sudo systemctl enable rpcbind
```

### Check Service Status

```bash
sudo systemctl status nfs-kernel-server  # Ubuntu/Debian
sudo systemctl status nfs-server         # RHEL/CentOS/Rocky
sudo systemctl status rpcbind
```

### Restart NFS Server

```bash
sudo systemctl restart nfs-kernel-server  # Ubuntu/Debian
sudo systemctl restart nfs-server         # RHEL/CentOS/Rocky
```

## Server Configuration

### /etc/exports Syntax

```
/path/to/share    client(options)    client2(options)
```

### Common Examples

```bash
# Share to a single host
/data    192.168.1.100(rw,sync,no_subtree_check)

# Share to an entire subnet
/data    192.168.1.0/24(rw,sync,no_subtree_check)

# Share to multiple networks
/data    192.168.1.0/24(rw,sync,no_subtree_check) 10.0.0.0/8(ro,sync)

# Share to everyone (use cautiously)
/shared    *(ro,sync,no_subtree_check)

# Share with hostname/wildcard
/data    *.example.com(rw,sync,no_subtree_check)

# Share with root squash disabled (NFS clients root = server root)
/data    192.168.1.0/24(rw,sync,no_root_squash,no_subtree_check)

# Share mapping all users to anonymous
/public    192.168.1.0/24(rw,sync,all_squash,anonuid=1000,anongid=1000)
```

### Export Options Reference

| Option | Description |
|--------|-------------|
| `rw` | Read-write access |
| `ro` | Read-only access (default) |
| `sync` | Write data to disk before replying (safe, recommended) |
| `async` | Reply before data is written to disk (faster, risk of corruption) |
| `no_subtree_check` | Disable subtree checking (recommended, better performance) |
| `subtree_check` | Verify file is in exported subtree (default, slower) |
| `root_squash` | Map root (UID 0) to anonymous user (default, secure) |
| `no_root_squash` | Allow remote root to have root privileges on server |
| `all_squash` | Map all UIDs/GIDs to anonymous user |
| `anonuid=N` | Set anonymous user UID (used with squash options) |
| `anongid=N` | Set anonymous group GID (used with squash options) |
| `no_all_squash` | Don't squash non-root users (default) |
| `wdelay` | Delay writing to disk if another write request is imminent (default) |
| `no_acl` | Disable ACL support (ACLs are supported by default) |
| `sec=sys` | AUTH_SYS (UID/GID based, default) |
| `sec=krb5` | Kerberos authentication |
| `sec=krb5i` | Kerberos authentication + integrity checking |
| `sec=krb5p` | Kerberos authentication + integrity + encryption |
| `fsid=0` | Mark as NFSv4 pseudo-root |
| `crossmnt` | Allow clients to traverse mount points within the export |
| `insecure` | Allow connections from ports above 1024 |

### Apply Export Changes

```bash
# Export all shares defined in /etc/exports
sudo exportfs -a

# Re-export all shares (refresh)
sudo exportfs -r

# Re-export all with verbose output
sudo exportfs -rv

# Export a specific share temporarily (not saved in /etc/exports)
sudo exportfs -o rw,sync,no_subtree_check 192.168.1.100:/data

# Export ignoring /etc/exports (inline only)
sudo exportfs -i -o rw,sync 192.168.1.0/24:/shared

# Unexport a share
sudo exportfs -u 192.168.1.100:/data

# Unexport all
sudo exportfs -ua

# Flush export cache
sudo exportfs -f

# Show current exports with options
sudo exportfs -v

# Show current exports (brief)
sudo exportfs -s
```

### Advanced Export Examples

```bash
# High-performance setup
/fastdata 192.168.1.0/24(rw,async,no_subtree_check,no_root_squash)

# Secure share with user mapping
/secure 192.168.1.0/24(rw,sync,no_subtree_check,all_squash,anonuid=1000,anongid=1000)

# Read-only public share
/public *(ro,sync,no_subtree_check,all_squash)

# Development share
/dev-share 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash,no_all_squash)
```

### Directory Permissions

```bash
# Make directory world-readable for NFS
sudo chmod 755 /path/to/nfs/share

# Set ownership (Ubuntu/Debian)
sudo chown nobody:nogroup /path/to/nfs/share

# Set ownership (RHEL/CentOS)
sudo chown nfsnobody:nfsnobody /path/to/nfs/share
```

## Firewall Configuration

### firewalld (RHEL / Rocky / AlmaLinux)

```bash
# Allow NFS service
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-services
```

### ufw (Ubuntu / Debian)

```bash
# Allow NFS from specific subnet
sudo ufw allow from 192.168.1.0/24 to any port nfs

# Or allow specific ports
sudo ufw allow 2049/tcp
sudo ufw allow 111/tcp
sudo ufw allow 111/udp
```

### NFS Ports

| Port | Service | Protocol | Notes |
|------|---------|----------|-------|
| 2049 | nfsd | TCP/UDP | NFS server (only port needed for NFSv4) |
| 111 | rpcbind | TCP/UDP | Port mapper (NFSv3) |
| 20048 | mountd | TCP/UDP | Mount daemon (NFSv3) |
| Dynamic | statd | TCP/UDP | Lock recovery (NFSv3) |
| Dynamic | lockd | TCP/UDP | File locking (NFSv3) |

### Fix Static Ports for NFSv3 (firewall-friendly)

```bash
# /etc/nfs.conf (RHEL 8+, Ubuntu 20.04+)
[lockd]
port = 32803
udp-port = 32803

[mountd]
port = 20048

[statd]
port = 662
outgoing-port = 2020
```

```bash
# /etc/sysconfig/nfs (RHEL 5–7)
RQUOTAD_PORT=875
LOCKD_TCPPORT=32803
LOCKD_UDPPORT=32769
MOUNTD_PORT=892
STATD_PORT=662
```

```bash
# Restart NFS services after changing ports
sudo systemctl restart nfs-server
```

### iptables Rules (Legacy)

```bash
# /etc/sysconfig/iptables
-A INPUT -p tcp -m tcp --dport 111 -j ACCEPT
-A INPUT -p udp -m udp --dport 111 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 662 -j ACCEPT
-A INPUT -p udp -m udp --dport 662 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 875 -j ACCEPT
-A INPUT -p udp -m udp --dport 875 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 892 -j ACCEPT
-A INPUT -p udp -m udp --dport 892 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 2049 -j ACCEPT
-A INPUT -p udp -m udp --dport 2049 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 32803 -j ACCEPT
-A INPUT -p udp -m udp --dport 32769 -j ACCEPT
```

### iptables with Source Restriction

```bash
# Allow NFS traffic from specific subnet only
sudo iptables -A INPUT -p tcp --dport 111 -s 192.168.1.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 2049 -s 192.168.1.0/24 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 111 -s 192.168.1.0/24 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 2049 -s 192.168.1.0/24 -j ACCEPT
```

## Mounting NFS Shares

### Manual Mount

```bash
# Create mount point
sudo mkdir -p /mnt/nfs-share

# Mount NFSv4 share
sudo mount -t nfs4 192.168.1.10:/data /mnt/nfs-share

# Mount specifying NFS version
sudo mount -t nfs -o vers=4.2 192.168.1.10:/data /mnt/nfs-share

# Mount NFSv3 share
sudo mount -t nfs -o vers=3 192.168.1.10:/data /mnt/nfs-share

# Mount read-only
sudo mount -t nfs -o ro 192.168.1.10:/data /mnt/nfs-share

# Mount with specific options
sudo mount -t nfs -o rw,hard,timeo=600,retrans=2,rsize=1048576,wsize=1048576 \
  192.168.1.10:/data /mnt/nfs-share
```

### Persistent Mount (/etc/fstab)

```bash
# NFSv4 mount
192.168.1.10:/data  /mnt/nfs-share  nfs4  defaults,_netdev  0  0

# NFSv4 with specific options
192.168.1.10:/data  /mnt/nfs-share  nfs4  rw,hard,timeo=600,retrans=2,_netdev  0  0

# NFSv3 mount
192.168.1.10:/data  /mnt/nfs-share  nfs  vers=3,defaults,_netdev  0  0

# Soft mount with timeout (returns error instead of hanging)
192.168.1.10:/data  /mnt/nfs-share  nfs4  soft,timeo=300,_netdev  0  0

# Background mount (boot continues if NFS server is down)
192.168.1.10:/data  /mnt/nfs-share  nfs4  defaults,bg,_netdev  0  0
```

### Production fstab Examples

```bash
# High-availability mount
192.168.1.100:/critical /mnt/critical nfs vers=4,rw,hard,bg,timeo=600,retrans=2,_netdev 0 0

# Development mount (softer settings)
192.168.1.100:/dev /mnt/dev nfs vers=4,rw,soft,bg,timeo=100,retrans=1,_netdev 0 0

# Backup mount (large transfers)
192.168.1.100:/backup /mnt/backup nfs vers=4,rw,hard,bg,rsize=1048576,wsize=1048576,_netdev 0 0

# Home directories
192.168.1.100:/home /home nfs vers=4,rw,hard,bg,timeo=600,_netdev 0 0

# High-performance mount
server:/share /mnt/share nfs4 rw,hard,bg,rsize=1048576,wsize=1048576,_netdev 0 0

# Read-only mount
server:/share /mnt/share nfs4 ro,hard,bg,_netdev 0 0
```

```bash
# Test fstab entry without rebooting
sudo mount -a

# Verify mount
df -hT /mnt/nfs-share
mount | grep nfs
```

### Mount Options Reference

| Option | Description |
|--------|-------------|
| `hard` | Retry indefinitely on timeout (default, recommended for data integrity) |
| `soft` | Return error after retrans retries (risk of data corruption) |
| `timeo=N` | Timeout in deciseconds (default 600 = 60 seconds) |
| `retrans=N` | Number of retries before giving up (soft) or reporting (hard) |
| `rsize=N` | Read buffer size in bytes (default 1048576 = 1MB) |
| `wsize=N` | Write buffer size in bytes (default 1048576 = 1MB) |
| `_netdev` | Wait for network before mounting (essential for NFS in fstab) |
| `bg` | Retry mount in background if first attempt fails |
| `fg` | Mount in foreground (default) |
| `intr` | Allow signals to interrupt NFS operations (deprecated in NFSv4) |
| `nolock` | Disable NFS locking (for NFSv3 without statd) |
| `nfsvers=N` | Specify NFS protocol version |
| `sec=sys` | AUTH_SYS authentication (default) |
| `sec=krb5` | Kerberos authentication |
| `proto=tcp` | Use TCP (default for NFSv4) |
| `proto=udp` | Use UDP (NFSv3 only) |
| `noatime` | Don't update access times (performance improvement) |
| `nodiratime` | Don't update directory access times |
| `actimeo=N` | Set all attribute cache timeouts to N seconds |
| `noacl` | Turns off all ACL processing |
| `noexec` | Prevents execution of binaries on mounted file systems |
| `nosuid` | Disables set-user-identifier or set-group-identifier bits |
| `nointr` | Don't allow keyboard interrupts on hard mounts |
| `ac` | Enable attribute caching (default) |
| `noac` | Disable attribute caching (strict consistency, slower) |
| `lock` | Use NLM locking protocol (default) |
| `nconnect=N` | Number of TCP connections per mount (kernel 5.3+, max 16) |
| `lookupcache=none` | Disable name lookup cache (multi-client consistency) |

### sec= Mode Details

| Mode | Description |
|------|-------------|
| `sec=sys` | Uses local UNIX UIDs and GIDs by using AUTH_SYS to authenticate NFS operations |
| `sec=krb5` | Uses Kerberos V5 instead of local UNIX UIDs and GIDs to authenticate users |
| `sec=krb5i` | Uses Kerberos V5 for user authentication and performs integrity checking of NFS operations using secure checksums to prevent data tampering |
| `sec=krb5p` | Uses Kerberos V5 for user authentication, integrity checking, and encrypts NFS traffic to prevent traffic sniffing. Most secure setting but highest performance overhead |

### RHEL Version Differences

```bash
# RHEL 5 — NFSv3 by default, mount NFSv4 with -t nfs4
mount -t nfs host:/remote/export /local/directory        # NFSv3
mount -t nfs4 host:/remote/export /local/directory       # NFSv4

# RHEL 6 & 7 — NFSv4 by default, use -o vers= to specify
mount -t nfs host:/remote/export /local/directory        # NFSv4 (default)
mount -t nfs -o vers=4 host:/remote/export /local/directory   # NFSv4 explicit
mount -t nfs -o vers=3 host:/remote/export /local/directory   # NFSv3

# NFSv3 with UDP (legacy)
mount -t nfs -o vers=3,proto=udp host:/remote/export /local/directory

# NFSv4.1 explicit
mount -t nfs -o vers=4.1 host:/remote/export /local/directory

# NFSv4.2 explicit
mount -t nfs -o vers=4.2 host:/remote/export /local/directory

# Enable verbose logging while mounting
mount -vvv nfsserver:/abc/xyz /mnt
```

## AutoFS (Automounting)

AutoFS mounts NFS shares on-demand when accessed and unmounts them after a period of inactivity.

### Installation

```bash
# RHEL / Rocky
sudo dnf install -y autofs

# Ubuntu / Debian
sudo apt install -y autofs

# Enable and start
sudo systemctl enable --now autofs
```

### Direct Map (Fixed Mount Points)

```bash
# /etc/auto.master (or /etc/auto.master.d/nfs.autofs)
/-    /etc/auto.direct

# /etc/auto.direct
/mnt/nfs-share    -rw,hard,timeo=600    192.168.1.10:/data
/mnt/backups      -ro                    192.168.1.10:/backups
```

### Indirect Map (Dynamic Mount Points)

```bash
# /etc/auto.master
/nfs    /etc/auto.nfs    --timeout=300

# /etc/auto.nfs (mount points created under /nfs/)
data      -rw,hard,timeo=600    192.168.1.10:/data
backups   -ro                    192.168.1.10:/backups
media     -rw                    192.168.1.10:/media
```

Accessing `/nfs/data` automatically mounts `192.168.1.10:/data`.

### Wildcard Map (Mount All Exports)

```bash
# /etc/auto.master
/nfs    /etc/auto.nfs    --timeout=300

# /etc/auto.nfs
*    -rw,hard    192.168.1.10:/&
```

Any access to `/nfs/<name>` mounts `192.168.1.10:/<name>`.

### Apply Changes

```bash
sudo systemctl restart autofs

# Debug automount
sudo automount -f -v
```

## NFSv4 Specific Configuration

### NFSv4 Pseudo-Root

NFSv4 uses a single pseudo-root filesystem. All exports appear under one namespace:

```bash
# /etc/exports — NFSv4 pseudo-root
/srv/nfs        192.168.1.0/24(rw,sync,fsid=0,crossmnt,no_subtree_check)
/srv/nfs/data   192.168.1.0/24(rw,sync,no_subtree_check)
/srv/nfs/home   192.168.1.0/24(rw,sync,no_subtree_check)
```

```bash
# Client mounts relative to pseudo-root
sudo mount -t nfs4 192.168.1.10:/data /mnt/data
sudo mount -t nfs4 192.168.1.10:/home /mnt/home
```

### NFSv4 ID Mapping

NFSv4 maps users by name instead of UID/GID. Both client and server must agree on the domain:

```bash
# /etc/idmapd.conf (both server and client)
[General]
Domain = example.com
```

```bash
# Restart idmapd
sudo systemctl restart nfs-idmapd

# Clear ID mapping cache
sudo nfsidmap -c
```

### NFSv4 Only (Disable NFSv3)

```bash
# /etc/nfs.conf (RHEL 8+)
[nfsd]
vers3 = n
vers4 = y
vers4.1 = y
vers4.2 = y
```

```bash
# /etc/default/nfs-kernel-server (Ubuntu/Debian)
RPCNFSDOPTS="-N 2 -N 3"
```

### NFSv4 Bind Mounts

NFSv4 uses a pseudo-root. To export directories outside the pseudo-root, use bind mounts:

```bash
# Create the NFSv4 root structure
sudo mkdir -p /srv/nfs4/share

# Bind mount the actual directory into the pseudo-root
sudo mount --bind /actual/share /srv/nfs4/share

# Make persistent in /etc/fstab
/actual/share    /srv/nfs4/share    none    bind    0    0
```

```bash
# /etc/exports
/srv/nfs4        192.168.1.0/24(rw,sync,fsid=0,crossmnt,no_subtree_check)
/srv/nfs4/share  192.168.1.0/24(rw,sync,no_subtree_check)
```

### Disable NFSv4 (Legacy /etc/sysconfig/nfs, RHEL 5–7)

```bash
# /etc/sysconfig/nfs
RPCNFSDARGS="-N 4"
```

```bash
sudo systemctl restart nfs-server
```

## Security

### SELinux (RHEL / Rocky)

```bash
# Check NFS-related booleans
getsebool -a | grep nfs

# Allow NFS home directories
sudo setsebool -P use_nfs_home_dirs on

# Allow NFS exports to be read/write
sudo setsebool -P nfs_export_all_rw on

# Set SELinux context for exported directory
sudo semanage fcontext -a -t nfs_t "/data(/.*)?"
sudo restorecon -Rv /data

# Check context
ls -Zd /data
```

### TCP Wrappers (Legacy, NFSv2/v3 Only)

TCP wrappers only apply to NFSv2 and NFSv3. For RHEL 5, replace `rpcbind` with `portmap`.

Note: CIDR notation `192.168.1.0/24` is not supported (IPv6 rules only). Use `192.168.1.` or `192.168.1.0/255.255.255.0`.

```bash
# /etc/hosts.deny
ALL: ALL
```

```bash
# /etc/hosts.allow
rpcbind: 192.168.1.
lockd: 192.168.1.
mountd: 192.168.1.
rquotad: 192.168.1.
statd: 192.168.1.
```

### Kerberos Authentication (NFSv4)

```bash
# Server: /etc/exports
/data    192.168.1.0/24(rw,sync,sec=krb5p,no_subtree_check)

# Client mount with Kerberos
sudo mount -t nfs4 -o sec=krb5p server.example.com:/data /mnt/data

# Verify Kerberos ticket
klist
```

## Performance Tuning

### Server Tuning

```bash
# Increase NFS server threads (default is 8)
# /etc/nfs.conf (RHEL 8+)
[nfsd]
threads = 16

# /etc/sysconfig/nfs (RHEL 5–7)
RPCNFSDCOUNT=16

# /etc/default/nfs-kernel-server (Ubuntu/Debian)
RPCNFSDCOUNT=16

# Or set at runtime
sudo rpc.nfsd 16

# Check current thread count
cat /proc/fs/nfsd/threads
```

### Client Tuning

```bash
# Mount with optimal buffer sizes and multiple connections
sudo mount -t nfs4 -o rsize=1048576,wsize=1048576,nconnect=8,noatime \
  192.168.1.10:/data /mnt/data
```

### Workload-Specific Mount Profiles

```bash
# Large file transfers / high-throughput
sudo mount -t nfs -o vers=4,rsize=1048576,wsize=1048576,hard,bg,timeo=600 \
  server:/share /mnt/share

# Small file operations / low-latency
sudo mount -t nfs -o vers=4,rsize=32768,wsize=32768,hard,timeo=50 \
  server:/share /mnt/share

# Database workloads (sync for durability)
sudo mount -t nfs -o vers=4,rsize=8192,wsize=8192,hard,sync,timeo=600 \
  server:/share /mnt/share

# Mount with specific server ports (if non-standard)
sudo mount -t nfs -o port=2049,mountport=20048 server:/share /mnt/share
```

### Kernel Parameters

```bash
# Increase NFS read-ahead (number of pages)
echo 256 > /sys/class/bdi/0:*/read_ahead_kb

# Sunrpc slot table (increase for parallel operations)
echo 128 > /proc/sys/sunrpc/tcp_slot_table_entries

# Make persistent via sysctl
echo "sunrpc.tcp_slot_table_entries = 128" >> /etc/sysctl.d/nfs.conf
sudo sysctl -p /etc/sysctl.d/nfs.conf

# Network buffer tuning (improves NFS throughput)
echo 'net.core.rmem_default = 262144' >> /etc/sysctl.conf
echo 'net.core.rmem_max = 16777216' >> /etc/sysctl.conf
echo 'net.core.wmem_default = 262144' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 16777216' >> /etc/sysctl.conf
sudo sysctl -p

# Client-side write cache tuning
echo 'vm.dirty_background_ratio = 5' >> /etc/sysctl.conf
echo 'vm.dirty_ratio = 10' >> /etc/sysctl.conf
sudo sysctl -p
```

### Performance Testing

```bash
# Write test
dd if=/dev/zero of=/mnt/nfs-share/testfile bs=1M count=1024 oflag=direct

# Read test
dd if=/mnt/nfs-share/testfile of=/dev/null bs=1M iflag=direct

# Cleanup
rm /mnt/nfs-share/testfile
```

## Monitoring and Diagnostics

### Show Exports

```bash
# Show server exports (from client)
showmount -e 192.168.1.10

# Show all exports
showmount -a 192.168.1.10

# Show connected clients (from server)
showmount

# Show directories being accessed
showmount -d

# Show exports without header line
showmount --no-headers -e 192.168.1.10

# Display export list with detailed info
showmount -v

# Display RPC programs on a host
rpcinfo -p 192.168.1.10

# NFS Server Configuration GUI Tool (legacy, RHEL 6)
# yum install system-config-nfs
# system-config-nfs
```

### NFS Statistics

```bash
# Client statistics
nfsstat -c

# Server statistics
nfsstat -s

# Client RPC statistics
nfsstat -rc

# Server RPC statistics
nfsstat -rs

# Client stats with numbers
nfsstat -cn

# Server stats with numbers
nfsstat -sn

# All statistics (verbose)
nfsstat -v

# Zero/reset all statistics
sudo nfsstat -z

# Raw NFS server statistics from kernel
cat /proc/net/rpc/nfsd

# Show per-mount statistics
nfsstat -m

# Per-mount statistics for specific mount
nfsstat -m /mnt/nfs-share

# Mountstats (detailed per-mount)
cat /proc/self/mountstats

# mountstats command (if available)
mountstats /mnt/nfs-share
mountstats --nfs /mnt/nfs-share    # NFS-specific output
mountstats --rpc /mnt/nfs-share    # RPC-specific output

# Show mounts via /proc
cat /proc/mounts | grep -i nfs

# NFS operations per second
nfsiostat 1

# NFS I/O statistics for specific mount
nfsiostat /mnt/nfs-share

# Focus on specific operations
nfsstat -c | grep -E "read|write"
```

### RPC Information

```bash
# Show all RPC services on local host
rpcinfo -p
rpcinfo -p localhost

# Show RPC services on remote host
rpcinfo -p 192.168.1.10

# Test specific RPC services over TCP
rpcinfo -t 192.168.1.10 nfs
rpcinfo -t 192.168.1.10 mountd

# Test specific RPC services over UDP
rpcinfo -u 192.168.1.10 nfs

# Show services summary
rpcinfo -s 192.168.1.10

# Broadcast for NFS version 4
rpcinfo -b nfs 4

# Filter for NFS and mount services
rpcinfo -p 192.168.1.10 | grep -E "(nfs|mount)"
```

### Packet Capture

```bash
# Capture traffic on NFS client
tcpdump -s0 -i eth0 host <nfsserverip> -w /tmp/dump.client.pcap

# Capture traffic on NFS server
tcpdump -s0 -i eth0 host <nfsclientip> -w /tmp/dump.server.pcap
```

### Active Connections

```bash
# Show NFS mount info
mount -t nfs4

# Show all NFS mounts with disk usage
df -h -t nfs
df -h -t nfs4

# Check if specific path is mounted
mountpoint /mnt/nfs-share

# Show NFS file handles
cat /proc/fs/nfsd/filehandle

# Show RPC information
rpcinfo -p localhost
rpcinfo -p 192.168.1.10

# Check NFS version negotiated
nfsstat -m
```

### Stale File Handles

```bash
# Check for stale mounts
df -h 2>&1 | grep "Stale"

# Force unmount a stale NFS mount
sudo umount -f /mnt/nfs-share

# Lazy unmount (detach immediately, cleanup later)
sudo umount -l /mnt/nfs-share
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Mount hangs | Check firewall (port 2049), verify server is running, use `bg` option |
| Permission denied | Check `/etc/exports` options, re-export with `exportfs -r`, check UID/GID mapping |
| Stale file handle | Unmount and remount (`umount -l` then `mount`) |
| Connection refused | Verify `rpcbind` and `nfs-server` are running on server |
| No route to host | Check firewall, network connectivity, and NFS ports |
| Access denied by server | Run `exportfs -v` on server, verify client IP is in allowed range |
| Slow performance | Increase `rsize`/`wsize`, use `nconnect`, increase server threads |
| D state processes (unkillable) | NFS server unreachable with `hard` mount — fix server or use `soft` mount |
| Boot hangs waiting for NFS | Add `_netdev` and `bg` options in fstab |
| NFSv4 maps users to nobody | Check `/etc/idmapd.conf` Domain on both client and server |
| showmount: RPC timed out | Firewall blocking port 111 (rpcbind) |
| Write errors / read-only | Verify export has `rw` option and directory permissions on server |

### Debug NFS Operations

```bash
# Mount with debugging enabled
sudo mount -t nfs -o vers=4,rw,hard,bg,debug server:/share /mnt/share

# Enable NFS debug logging (server)
rpcdebug -m nfsd all

# Enable NFS debug logging (client)
rpcdebug -m nfs all

# Disable debug
rpcdebug -m nfsd -c all
rpcdebug -m nfs -c all

# Check kernel NFS debug messages
dmesg | grep -i nfs
journalctl -u nfs-server --no-pager -n 50
journalctl -u nfs-client.target --no-pager -n 50

# Check system logs
sudo tail -f /var/log/messages      # RHEL/CentOS
sudo tail -f /var/log/syslog        # Ubuntu/Debian
```

### Connectivity Tests

```bash
# Test NFS port
nc -zv 192.168.1.10 2049

# Test rpcbind port
nc -zv 192.168.1.10 111

# RPC check
rpcinfo -t 192.168.1.10 nfs
rpcinfo -t 192.168.1.10 mountd

# Test with timeout
timeout 5 mount -t nfs4 192.168.1.10:/data /mnt/test
```

## Unmounting

```bash
# Standard unmount
sudo umount /mnt/nfs-share

# Force unmount (if server is unresponsive)
sudo umount -f /mnt/nfs-share

# Lazy unmount (detach from filesystem, cleanup when idle)
sudo umount -l /mnt/nfs-share

# Find processes using the mount point
fuser -mv /mnt/nfs-share
lsof +D /mnt/nfs-share
lsof +f -- /mnt/nfs-share

# Kill processes using the mount, then unmount
fuser -km /mnt/nfs-share && sudo umount /mnt/nfs-share

# Kill processes (alternate, sends SIGKILL)
sudo fuser -ck /mnt/nfs-share
```

## Common Patterns

### Shared Home Directories

```bash
# Server /etc/exports
/home    192.168.1.0/24(rw,sync,no_subtree_check)

# Client /etc/fstab
192.168.1.10:/home  /home  nfs4  rw,hard,_netdev  0  0
```

### Shared Web Content

```bash
# Server /etc/exports
/var/www    192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)

# Client /etc/fstab
192.168.1.10:/var/www  /var/www  nfs4  rw,hard,noatime,_netdev  0  0
```

### Read-Only Media Share

```bash
# Server /etc/exports
/media    192.168.1.0/24(ro,sync,no_subtree_check,all_squash)

# Client /etc/fstab
192.168.1.10:/media  /mnt/media  nfs4  ro,hard,_netdev  0  0
```

### Kubernetes NFS Provisioner

```bash
# Server /etc/exports
/srv/nfs/k8s    192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
```

## NFS Setup by RHEL Version

### Server Setup

#### RHEL 5

```bash
yum install nfs-utils portmap
service portmap start; chkconfig portmap on
service nfslock start; chkconfig nfslock on
service nfs start; chkconfig nfs on
```

#### RHEL 6

```bash
yum install nfs-utils rpcbind
service rpcbind start; chkconfig rpcbind on
service nfslock start; chkconfig nfslock on
service nfs start; chkconfig nfs on
```

#### RHEL 7

```bash
yum install nfs-utils rpcbind
systemctl start rpcbind nfs-lock nfs-server
systemctl enable rpcbind nfs-lock nfs-server
```

### Client Setup

#### RHEL 5

```bash
yum install nfs-utils portmap
service portmap start; chkconfig portmap on
service nfslock start; chkconfig nfslock on
chkconfig netfs on
```

#### RHEL 6

```bash
yum install nfs-utils rpcbind
service rpcbind start; chkconfig rpcbind on
service nfslock start; chkconfig nfslock on
chkconfig netfs on
```

#### RHEL 7

```bash
yum install nfs-utils rpcbind
systemctl start rpcbind nfs-lock
systemctl enable rpcbind nfs-lock
# netfs service replaced by systemd-fstab-generator
# (for every entry in /etc/fstab a systemd unit file is generated in /run/systemd/generator)
```

#### RHEL 8+

```bash
yum install nfs-utils
# rpcbind started automatically as a dependency
```

## Important File Paths

| File | Purpose |
|------|---------|
| `/etc/exports` | NFS server export definitions |
| `/etc/exports.d/*.exports` | Additional export files (included automatically) |
| `/etc/fstab` | Persistent mount configuration |
| `/etc/nfs.conf` | NFS daemon configuration (RHEL 8+, Ubuntu 20.04+) |
| `/etc/sysconfig/nfs` | NFS daemon configuration (RHEL 5–7, controls ports and services) |
| `/etc/default/nfs-kernel-server` | NFS server configuration (Ubuntu/Debian, threads and options) |
| `/etc/nfsmount.conf` | NFS client mount defaults |
| `/etc/idmapd.conf` | NFSv4 ID mapping configuration |
| `/etc/auto.master` | AutoFS master map |
| `/var/lib/nfs/etab` | Current export table (runtime) |
| `/proc/fs/nfsd/` | NFS server kernel parameters |
| `/proc/self/mountstats` | Per-mount NFS statistics |

## Advanced Diagnostics

### Network Connections

```bash
# Show NFS-related connections
ss -tuln | grep :2049          # NFS daemon listening
ss -tuln | grep :111           # RPC portmapper listening
ss -tup | grep :2049           # Active NFS connections

# Using netstat (if available)
netstat -tuln | grep :2049
netstat -an | grep :111

# Find NFS network connections with lsof
lsof -i :2049                  # NFS port connections
lsof -i :111                   # RPC portmapper connections
lsof -N                        # All NFS open files
```

### tshark / Wireshark

```bash
# Capture NFS traffic
tshark -i any -f "port 2049" -w nfs.pcap
tshark -i any -f "port 2049" -V        # Verbose real-time

# Comprehensive NFS + RPC capture
tcpdump -i any "(port 111 or port 2049)" -w nfs_traffic.pcap

# Filter NFS packets from capture file
tshark -r nfs.pcap -Y "nfs"
tshark -r nfs.pcap -Y "rpc"
```

### Service Status

```bash
# NFS Server services
systemctl status nfs-server           # RHEL/CentOS/Rocky
systemctl status nfs-kernel-server    # Ubuntu/Debian
systemctl status rpcbind
systemctl status nfs-mountd
systemctl status nfs-idmapd

# Client services
systemctl status nfs-client.target
systemctl status rpc-gssd             # For Kerberos

# Reload without restart
systemctl reload nfs-server
```

### journalctl Queries

```bash
# NFS-related logs
journalctl -u nfs-server              # Server logs
journalctl -u nfs-kernel-server       # Ubuntu server logs
journalctl -u rpcbind                 # RPC bind logs
journalctl -u nfs-client.target       # Client logs

# Follow logs in real-time
journalctl -f -u nfs-server
journalctl -f | grep -i nfs

# Time-range queries
journalctl --since "1 hour ago" | grep -i nfs
journalctl --since today | grep -i nfs
journalctl --since "2024-01-01" --until "2024-01-02" -u nfs-server
```

### Filesystem Inspection

```bash
# Disk usage on NFS mount
du -sh /mnt/nfs-share                         # Summary
du -h --max-depth=1 /mnt/nfs-share            # Top-level directories

# Check permissions (numeric UIDs/GIDs)
ls -la /mnt/nfs-share
ls -n /mnt/nfs-share                          # Show numeric UIDs/GIDs

# Check mount point itself
ls -ld /mnt/nfs-share
```

### strace — System Call Tracing

```bash
# Trace mount operation
strace -o mount.trace mount -t nfs server:/share /mnt/share

# Trace file operations on NFS mount
strace -e trace=file ls /mnt/nfs-share

# Attach to running process using NFS
strace -p $(pidof some_nfs_process)
```

### dmesg — Kernel Messages

```bash
# NFS-related kernel messages
dmesg | grep -i nfs
dmesg | grep -i rpc

# Watch for new messages in real-time
dmesg -w

# Clear kernel message buffer then monitor
sudo dmesg -c
dmesg -w
```

### I/O Performance Testing (iozone)

```bash
# Install iozone (if not available)
# RHEL: yum install iozone / Debian: apt install iozone3

# Basic auto test
iozone -a -g 1G -f /mnt/nfs-share/testfile

# Specific read/write test
iozone -i 0 -i 1 -f /mnt/nfs-share/testfile -s 100M

# Cleanup
rm -f /mnt/nfs-share/testfile
```

## Automation and Scripting

### Test Multiple NFS Servers

```bash
for server in server1 server2 server3; do
    echo "Testing $server:"
    showmount -e $server 2>/dev/null || echo "  Failed to connect"
done
```

### Mount All Exports from a Server

```bash
showmount -e server_ip --no-headers | awk '{print $1}' | while read share; do
    mkdir -p /mnt/nfs$(basename $share)
    mount -t nfs server_ip:$share /mnt/nfs$(basename $share)
done
```

### NFS Mount Health Check

```bash
#!/bin/bash
mount -t nfs,nfs4 | while read line; do
    mount_point=$(echo $line | awk '{print $3}')
    if timeout 5 ls $mount_point >/dev/null 2>&1; then
        echo "$mount_point: OK"
    else
        echo "$mount_point: FAILED"
    fi
done
```

### Full NFS Health Check

```bash
echo "=== NFS Services ==="
systemctl is-active nfs-server rpcbind
echo -e "\n=== Exports ==="
exportfs -v
echo -e "\n=== Active Mounts ==="
mount | grep nfs
echo -e "\n=== Statistics ==="
nfsstat -s | head -10
echo -e "\n=== NFS Port ==="
ss -tuln | grep 2049
```

## Quick Reference

```bash
# Server: export and verify
sudo exportfs -ra && sudo exportfs -v

# Client: mount and verify
sudo mount -t nfs4 192.168.1.10:/data /mnt/data && df -hT /mnt/data

# Check NFS version in use
nfsstat -m | grep vers

# List all NFS mounts
mount -t nfs,nfs4

# Check server availability from client
showmount -e 192.168.1.10

# Reload NFS server config without restarting
sudo exportfs -r
```

## NFSv4 ACLs

NFSv4 has its own ACL system, separate from POSIX ACLs. Requires `nfs4-acl-tools` package.

```bash
# Install NFSv4 ACL tools
sudo dnf install -y nfs4-acl-tools    # RHEL/Rocky
sudo apt install -y nfs4-acl-tools    # Ubuntu/Debian

# View ACLs on a file
nfs4_getfacl /mnt/nfs-share/file.txt

# Set ACL (grant read/write to specific user)
nfs4_setfacl -a A::user@example.com:RW /mnt/nfs-share/file.txt

# Set ACL (grant read to group)
nfs4_setfacl -a A:g:devs@example.com:R /mnt/nfs-share/project/

# Remove a specific ACE
nfs4_setfacl -x A::user@example.com:RW /mnt/nfs-share/file.txt

# Set ACL recursively
nfs4_setfacl -R -a A::user@example.com:RWX /mnt/nfs-share/dir/

# Replace all ACLs from a file
nfs4_setfacl -S acl-file.txt /mnt/nfs-share/file.txt

# Edit ACLs interactively
nfs4_editfacl /mnt/nfs-share/file.txt
```

### NFSv4 ACL Permissions

#### Aliases (Shortcuts)

| Alias | Expands To | Description |
|-------|-----------|-------------|
| `R` | `rntcy` | Generic read |
| `W` | `watTNcCy` (+`D` for dirs) | Generic write |
| `X` | `xtcy` | Generic execute |

#### Individual Permissions

| Letter | Description |
|--------|-------------|
| `r` | Read file data / list directory |
| `w` | Write file data / create files in directory |
| `a` | Append data to file / create subdirectories |
| `x` | Execute file / traverse directory |
| `d` | Delete the file |
| `D` | Delete child (files/dirs within a directory) |
| `t` | Read attributes |
| `T` | Write attributes |
| `n` | Read named attributes |
| `N` | Write named attributes |
| `c` | Read ACL |
| `C` | Write ACL |
| `o` | Write owner (change ownership) |
| `y` | Synchronize |

## nfsconf — NFS Configuration Management (RHEL 8+)

```bash
# Get a configuration value
nfsconf --get nfsd threads
nfsconf --get nfsd vers4.2

# Set a configuration value
sudo nfsconf --set nfsd threads 16
sudo nfsconf --set nfsd vers3 n

# Unset a configuration value
sudo nfsconf --unset nfsd vers3

# Dump full configuration
nfsconf --dump

# Check specific section
nfsconf --get lockd port
nfsconf --get statd port
nfsconf --get mountd port
```

## NFS over RDMA

For high-performance networks (InfiniBand, RoCE) where low latency and high throughput are required.

```bash
# Load RDMA kernel modules
sudo modprobe xprtrdma    # Client
sudo modprobe svcrdma     # Server

# Mount using RDMA transport
sudo mount -t nfs -o proto=rdma,port=20049 server:/data /mnt/data

# Persistent mount in /etc/fstab
server:/data  /mnt/data  nfs  proto=rdma,port=20049,rsize=1048576,wsize=1048576,hard,_netdev  0  0

# Enable RDMA on NFS server
echo "rdma 20049" > /proc/fs/nfsd/portlist

# Or configure in /etc/nfs.conf
# [nfsd]
# rdma=y
# rdma-port=20049

# Verify RDMA transport is active
cat /proc/fs/nfsd/portlist
nfsstat -m | grep proto
```

## Best Practices

1. Use NFSv4.2 where possible — better security, performance, and features
2. Always use `sync` on exports — `async` risks data loss on crash
3. Keep `root_squash` enabled unless you have a specific need for `no_root_squash`
4. Use `_netdev` in fstab to prevent boot issues if the NFS server is unreachable
5. Use `hard` mounts for data integrity — `soft` mounts risk silent corruption
6. Match UID/GID between client and server for consistent permissions
7. Use Kerberos (`sec=krb5p`) in environments where security matters
8. Increase NFS server threads for busy file servers (16–64 depending on load)
9. Use `nconnect` for high-throughput workloads (kernel 5.3+)
10. Monitor with `nfsstat` and `nfsiostat` to identify bottlenecks
