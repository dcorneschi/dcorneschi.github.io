# Linux NFS Troubleshooting

A comprehensive guide to diagnosing and resolving NFS issues — connection errors, mount failures, performance problems, authentication issues, and emergency recovery procedures.

## Common Error Messages and Solutions

### Connection Errors

#### "Connection refused" or "No route to host"

```bash
# Diagnosis
ping server_ip
telnet server_ip 2049
nmap -p 111,2049 server_ip

# Solution 1: Check firewall on server
sudo ufw status
sudo firewall-cmd --list-all
sudo iptables -L | grep -E "(111|2049)"

# Solution 2: Verify NFS service is running
systemctl status nfs-server        # RHEL/CentOS
systemctl status nfs-kernel-server # Ubuntu/Debian
systemctl status rpcbind

# Solution 3: Check network configuration
ip route show
ss -tuln | grep -E "(111|2049)"
```

#### "RPC: Program not registered"

```bash
# Diagnosis
rpcinfo -p server_ip

# Solution 1: Start RPC services
sudo systemctl start rpcbind
sudo systemctl start nfs-server

# Solution 2: Re-export and verify
sudo exportfs -ra
sudo exportfs -v

# Solution 3: Verify port registration
rpcinfo -p localhost | grep -E "(nfs|mount)"
```

#### "mount.nfs: access denied"

```bash
# Diagnosis
showmount -e server_ip
cat /etc/exports

# Solution 1: Verify client IP is in exports
sudo exportfs -v | grep client_ip

# Solution 2: Update exports
sudo nano /etc/exports
# Ensure client IP/network is allowed
sudo exportfs -ra

# Solution 3: Monitor for denied connections (on server)
sudo tcpdump -i any port 2049 and host client_ip
```

### Mount Errors

#### "Stale file handle"

```bash
# Cause: Server reboot or export change while mounted

# Solution 1: Unmount and remount
sudo umount /mnt/nfs-share
sudo mount /mnt/nfs-share

# Solution 2: Force unmount if needed
sudo umount -f /mnt/nfs-share
sudo umount -l /mnt/nfs-share  # lazy unmount

# Solution 3: Clear client cache
echo 3 | sudo tee /proc/sys/vm/drop_caches

# Solution 4: Restart NFS client services
sudo systemctl restart nfs-client.target
```

#### "Device or resource busy"

```bash
# Diagnosis
lsof +f -- /mnt/nfs-share
fuser -v /mnt/nfs-share
ps aux | grep /mnt/nfs-share

# Solution 1: Find and stop processes
sudo fuser -ck /mnt/nfs-share

# Solution 2: Change directory away from mount
cd /

# Solution 3: Wait and try again
sleep 5
sudo umount /mnt/nfs-share

# Solution 4: Use lazy unmount as last resort
sudo umount -l /mnt/nfs-share
```

#### "mount.nfs: Operation not permitted"

```bash
# Solution 1: Mount as root
sudo mount -t nfs server:/share /mnt/share

# Solution 2: Check SELinux (if enabled)
sudo setsebool -P nfs_export_all_rw on
sudo setsebool -P use_nfs_home_dirs on

# Solution 3: Verify export permissions on server
sudo exportfs -v | grep share

# Solution 4: Check mount options
mount | grep nfs-share
```

### Performance Issues

#### Slow NFS Performance

```bash
# Diagnosis
# 1. Check network latency
ping -c 10 server_ip

# 2. Test bandwidth
iperf3 -c server_ip

# 3. Check mount options
mount | grep nfs-share
nfsstat -m | grep -A10 /mnt/nfs-share

# 4. Monitor I/O
iostat -x 1
iotop

# Solution 1: Optimize mount options
sudo umount /mnt/nfs-share
sudo mount -t nfs -o vers=4,rsize=1048576,wsize=1048576,hard,timeo=600 \
  server:/share /mnt/nfs-share

# Solution 2: Increase server threads
# On server:
echo 'RPCNFSDCOUNT=16' | sudo tee -a /etc/default/nfs-kernel-server
sudo systemctl restart nfs-kernel-server

# Solution 3: Tune kernel parameters
echo 'net.core.rmem_max = 16777216' | sudo tee -a /etc/sysctl.conf
echo 'net.core.wmem_max = 16777216' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

#### High Network Utilization

```bash
# Diagnosis
iftop
nload
ss -i | grep nfs
nfsstat -c | grep -E "read|write"
nfsstat -s | grep -E "read|write"

# Solution 1: Use larger block sizes
# Remount with rsize=1048576,wsize=1048576

# Solution 2: Use async for better throughput (less safety)
sudo mount -o async server:/share /mnt/share
```

### Authentication and Security Issues

#### Kerberos Authentication Failures

```bash
# Diagnosis
klist
klist -k

# Solution 1: Renew Kerberos ticket
kinit username
klist

# Solution 2: Check NFS Kerberos services
systemctl status rpc-gssd
systemctl status rpc-svcgssd

# Solution 3: Mount with proper security
sudo mount -t nfs -o sec=krb5 server:/share /mnt/share

# Solution 4: Check krb5.conf configuration
sudo nano /etc/krb5.conf
```

#### Permission Denied on Files

```bash
# Diagnosis
ls -la /mnt/nfs-share/
id
getent passwd username

# On both client and server:
id username
ls -n /path/to/file

# Solution 1: Configure idmapping (NFSv4)
sudo nano /etc/idmapd.conf
sudo systemctl restart nfs-idmapd

# Solution 2: Use all_squash with specific user
# In /etc/exports:
/share client(rw,all_squash,anonuid=1000,anongid=1000)
sudo exportfs -ra
```

## Diagnostic Procedures

### Server-Side Diagnosis

```bash
# 1. Check service status
systemctl status nfs-server rpcbind nfs-mountd

# 2. Verify exports
exportfs -v
cat /etc/exports

# 3. Check ports and services
rpcinfo -p localhost
ss -tuln | grep -E "(111|2049)"

# 4. Test local mount
sudo mount -t nfs localhost:/share /mnt/test

# 5. Check logs
journalctl -u nfs-server -f
tail -f /var/log/messages | grep -i nfs

# 6. Monitor connections
sudo tcpdump -i any port 2049
```

### Client-Side Diagnosis

```bash
# 1. Test network connectivity
ping server_ip
telnet server_ip 2049

# 2. Check available exports
showmount -e server_ip

# 3. Test RPC connectivity
rpcinfo -p server_ip

# 4. Try manual mount with verbose output
sudo mount -t nfs -v server:/share /mnt/test

# 5. Check mount statistics
nfsstat -m

# 6. Review system logs
dmesg | grep -i nfs
journalctl -u nfs-client.target
```

### Performance Troubleshooting

#### Identify Bottlenecks

```bash
# 1. CPU Usage
top -p $(pgrep nfsd)

# 2. Memory Usage
free -h
cat /proc/meminfo | grep -i nfs

# 3. Disk I/O
iostat -x 1 5
iotop -o

# 4. Network I/O
iftop
nethogs
ss -i | grep :2049

# 5. NFS-specific metrics
nfsstat -s          # Server stats
nfsstat -c          # Client stats
cat /proc/net/rpc/nfsd
```

#### Performance Testing

```bash
# Sequential read test
dd if=/mnt/nfs-share/testfile of=/dev/null bs=1M count=100

# Sequential write test
dd if=/dev/zero of=/mnt/nfs-share/testfile bs=1M count=100 conv=fsync

# Random I/O test (if fio available)
fio --name=random-rw --ioengine=libaio --rw=randrw --bs=4k --numjobs=4 \
    --size=100m --runtime=60 --filename=/mnt/nfs-share/testfile

# Small file operations
time (for i in {1..100}; do touch /mnt/nfs-share/file$i; done)
time (ls /mnt/nfs-share/ > /dev/null)
time (rm /mnt/nfs-share/file*)
```

## Error Code Reference

| Error Code | Name | Common Causes |
|------------|------|---------------|
| EACCES (13) | Permission Denied | Export doesn't allow client IP, wrong user permissions, SELinux blocking |
| ESTALE (116) | Stale NFS File Handle | Server reboot while mounted, export changed, file deleted on server |
| ENOENT (2) | No Such File or Directory | Export path doesn't exist, mount point missing, DNS resolution failure |
| ETIMEDOUT (110) | Connection Timed Out | Network issues, server overloaded, firewall blocking |

## Log Analysis

### Key Log Locations

```bash
# System logs
/var/log/messages          # RHEL/CentOS
/var/log/syslog            # Ubuntu/Debian

# Service-specific logs
journalctl -u nfs-server
journalctl -u rpcbind
journalctl -u nfs-client.target

# Kernel logs
dmesg | grep -i nfs
```

### Important Log Patterns

```bash
# Connection issues
grep -i "connection refused" /var/log/messages
grep -i "no route to host" /var/log/messages

# Authentication problems
grep -i "authentication" /var/log/messages
grep -i "access denied" /var/log/messages

# Performance warnings
grep -i "server not responding" /var/log/messages
grep -i "timeout" /var/log/messages

# Service problems
grep -i "rpcbind" /var/log/messages
grep -i "nfsd" /var/log/messages
```

## Prevention and Monitoring

### NFS Health Check Script

```bash
cat << 'EOF' > /usr/local/bin/nfs-monitor.sh
#!/bin/bash
# NFS Health Check Script

# Check services
for service in nfs-server rpcbind; do
    if ! systemctl is-active --quiet $service; then
        echo "ALERT: $service is not running"
    fi
done

# Check exports
if [ -z "$(exportfs)" ]; then
    echo "WARNING: No active exports"
fi

# Check client mounts
mount -t nfs,nfs4 | while read line; do
    mount_point=$(echo $line | awk '{print $3}')
    if ! timeout 5 test -r "$mount_point" 2>/dev/null; then
        echo "ALERT: $mount_point not accessible"
    fi
done

# Check statistics for errors
if nfsstat -c 2>/dev/null | grep -q "retrans"; then
    retrans=$(nfsstat -c | grep "retrans" | awk '{print $2}')
    if [ "$retrans" -gt 100 ]; then
        echo "WARNING: High retransmission count: $retrans"
    fi
fi
EOF

chmod +x /usr/local/bin/nfs-monitor.sh
```

```bash
# Add to crontab (run every 5 minutes)
echo "*/5 * * * * /usr/local/bin/nfs-monitor.sh" | crontab -
```

### Configuration Backup

```bash
# Export configuration backup
sudo cp /etc/exports /etc/exports.backup.$(date +%Y%m%d)

# Service configuration backup
sudo cp /etc/default/nfs-kernel-server /etc/default/nfs-kernel-server.backup

# Create restore script
cat << 'EOF' > /root/nfs-restore.sh
#!/bin/bash
# NFS Configuration Restore Script
systemctl stop nfs-server
cp /etc/exports.backup /etc/exports
systemctl start nfs-server
exportfs -ra
EOF

chmod +x /root/nfs-restore.sh
```

### Security Hardening

```bash
# Disable NFSv2 and NFSv3 (use only NFSv4)
echo 'RPCNFSDOPTS="-N 2 -N 3"' | sudo tee -a /etc/default/nfs-kernel-server

# Use specific ports (easier for firewalling)
echo 'STATDOPTS="--port 32765 --outgoing-port 32766"' | sudo tee -a /etc/default/nfs-common
echo 'LOCKD_TCPPORT=32767' | sudo tee -a /etc/default/nfs-common
echo 'LOCKD_UDPPORT=32768' | sudo tee -a /etc/default/nfs-common

# Restrict exports to specific networks
# In /etc/exports:
# /share 192.168.1.0/24(rw,sync,no_subtree_check,root_squash)

# Enable logging for security auditing
echo 'rpcdebug="nfs rpc"' | sudo tee -a /etc/default/nfs-common
```

## Emergency Procedures

### Server Recovery

```bash
# If NFS server is unresponsive:
# 1. Check system resources
df -h
free -h
uptime

# 2. Restart services in order
sudo systemctl stop nfs-server
sudo systemctl stop rpcbind
sleep 5
sudo systemctl start rpcbind
sudo systemctl start nfs-server

# 3. Re-export shares
sudo exportfs -ra
sudo exportfs -v

# 4. Check client recovery
showmount -a
```

### Client Recovery

```bash
# If client mounts are hung:
# 1. Force unmount all NFS mounts
sudo umount -a -t nfs,nfs4 -f

# 2. Clear any remaining processes
sudo pkill -f nfs

# 3. Restart client services
sudo systemctl restart nfs-client.target

# 4. Remount shares
sudo mount -a

# 5. Verify mounts
mount | grep nfs
```

### Data Recovery

```bash
# If files are corrupted or missing:
# 1. Check server-side filesystem integrity
# On server:
fsck /dev/device_with_nfs_data

# 2. Compare checksums
find /server/path -type f -exec sha256sum {} \; > server_checksums
find /client/mount -type f -exec sha256sum {} \; > client_checksums
diff server_checksums client_checksums

# 3. Rsync verification (dry run)
rsync -avnc /server/path/ /client/mount/
```

## rpcdebug Flags Reference

The `rpcdebug` command enables kernel-level NFS debug logging. Enable only what you need — enabling all flags generates massive log output.

### Server-Side Modules (nfsd)

```bash
# Enable all server debug (very verbose)
rpcdebug -m nfsd all

# Enable specific categories
rpcdebug -m nfsd proc        # NFS procedure calls
rpcdebug -m nfsd fileop      # File operations
rpcdebug -m nfsd auth        # Authentication
rpcdebug -m nfsd export      # Export operations
rpcdebug -m nfsd fh          # File handle operations
rpcdebug -m nfsd svc         # Service dispatch
rpcdebug -m nfsd sock        # Socket operations
rpcdebug -m nfsd repcache    # Reply cache
rpcdebug -m nfsd xdr         # XDR encoding/decoding
rpcdebug -m nfsd lockd       # Lock daemon

# Disable all
rpcdebug -m nfsd -c all

# Common combinations:
# Permission issues:
rpcdebug -m nfsd auth export

# File access issues:
rpcdebug -m nfsd proc fileop fh

# Export issues:
rpcdebug -m nfsd export svc
```

### Client-Side Modules (nfs)

```bash
# Enable all client debug
rpcdebug -m nfs all

# Enable specific categories
rpcdebug -m nfs vfs          # VFS layer operations
rpcdebug -m nfs dircache     # Directory cache
rpcdebug -m nfs lookupcache  # Name lookup cache
rpcdebug -m nfs pagecache    # Page cache
rpcdebug -m nfs proc         # NFS procedure calls
rpcdebug -m nfs xdr          # XDR encoding/decoding
rpcdebug -m nfs file         # File operations
rpcdebug -m nfs root         # Root directory operations
rpcdebug -m nfs callback     # NFSv4 callbacks
rpcdebug -m nfs client       # Client state
rpcdebug -m nfs mount        # Mount operations

# Disable all
rpcdebug -m nfs -c all

# Common combinations:
# Mount issues:
rpcdebug -m nfs vfs mount

# Stale handle debugging:
rpcdebug -m nfs vfs proc file

# Performance investigation:
rpcdebug -m nfs pagecache dircache
```

### RPC Layer Debugging

```bash
# Enable RPC debug
rpcdebug -m rpc all
rpcdebug -m rpc auth         # RPC authentication
rpcdebug -m rpc call         # RPC calls
rpcdebug -m rpc xprt         # RPC transport
rpcdebug -m rpc bind         # Port binding
rpcdebug -m rpc sched        # Scheduler
rpcdebug -m rpc cache        # Cache operations

# View debug output
dmesg -w | grep -i "nfs\|rpc"
journalctl -k -f | grep -i "nfs\|rpc"
```

## Hung Mount Detection

### Detect Hung NFS Mounts

```bash
#!/bin/bash
# hung-mount-check.sh — Detect unresponsive NFS mounts

TIMEOUT=5

mount -t nfs,nfs4 | awk '{print $3}' | while read mount_point; do
    if ! timeout $TIMEOUT stat "$mount_point" >/dev/null 2>&1; then
        echo "HUNG: $mount_point is not responding (timeout: ${TIMEOUT}s)"
    else
        echo "OK: $mount_point"
    fi
done
```

### Monitor Retransmissions

High retransmit counts indicate network or server issues:

```bash
#!/bin/bash
# nfs-retrans-monitor.sh — Alert on high retransmission counts

THRESHOLD=50

grep -A 20 "^device" /proc/self/mountstats | while read line; do
    if echo "$line" | grep -q "^device"; then
        current_mount=$(echo "$line" | awk '{print $2}')
    fi
    if echo "$line" | grep -q "retrans:"; then
        retrans=$(echo "$line" | grep -oP 'retrans: \K\d+')
        if [ "$retrans" -gt "$THRESHOLD" ]; then
            echo "WARNING: $current_mount has $retrans retransmissions (threshold: $THRESHOLD)"
        fi
    fi
done
```

### NFSv4 ACL Permission Issues

```bash
# Symptom: Permission denied despite POSIX permissions looking correct

# Check NFSv4 ACLs (may override POSIX)
nfs4_getfacl /mnt/nfs-share/problem-file

# Common fix: reset ACLs to match POSIX
nfs4_setfacl -e /mnt/nfs-share/problem-file

# Remove all deny ACEs
nfs4_setfacl -x D::EVERYONE@:RWX /mnt/nfs-share/problem-file

# If idmapd isn't translating users correctly
# Check that Domain matches in /etc/idmapd.conf on both sides
grep Domain /etc/idmapd.conf
sudo nfsidmap -c    # Clear cache
sudo systemctl restart nfs-idmapd
```

## Troubleshooting Flowchart

| Symptom | First Check | Second Check | Resolution |
|---------|-------------|--------------|------------|
| Mount hangs | `ping server_ip` | `telnet server_ip 2049` | Fix firewall or start NFS service |
| Access denied | `showmount -e server` | `exportfs -v` on server | Update `/etc/exports`, run `exportfs -ra` |
| Stale handle | `umount -f /mount` | Remount the share | If persistent, restart `nfs-client.target` |
| Slow I/O | `nfsstat -m` | Check `rsize`/`wsize` | Increase buffer sizes, add `nconnect` |
| Users mapped to nobody | Check `/etc/idmapd.conf` | Verify Domain matches | Restart `nfs-idmapd` on both sides |
| Boot hangs | Check fstab options | Add `_netdev,bg` | Ensure network-dependent mount options |
| RPC timeout | `rpcinfo -p server` | Check port 111 firewall | Allow rpcbind through firewall |
