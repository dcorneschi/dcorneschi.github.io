# NFS Performance Testing and Monitoring

Methods to test and monitor NFS filesystem read performance — useful for Docker Swarm deployments using NFS-mounted configuration files, homelabs, and production environments.

## Simple Read Tests with time

### Basic File Read Test

```bash
# Test reading a file from NFS mount
time cat /mnt/nfs/traefik-config/traefik.yml > /dev/null

# Test reading directory listing
time ls -la /mnt/nfs/traefik-config/

# Test copying from NFS to local storage
time cp /mnt/nfs/traefik-config/traefik.yml /tmp/test-file
```

### Multiple Run Test

```bash
# Run multiple times to get average
for i in {1..5}; do
    echo "Run $i:"
    time cat /mnt/nfs/traefik-config/traefik.yml > /dev/null
done
```

## Throughput Testing with dd

### Create Test File (if you have write access)

```bash
# Create a 100MB test file on NFS
dd if=/dev/zero of=/mnt/nfs/test-file bs=1M count=100

# Test read speed
time dd if=/mnt/nfs/test-file of=/dev/null bs=1M

# Test with different block sizes
time dd if=/mnt/nfs/test-file of=/dev/null bs=4K
time dd if=/mnt/nfs/test-file of=/dev/null bs=64K
time dd if=/mnt/nfs/test-file of=/dev/null bs=1M

# Clean up
rm /mnt/nfs/test-file
```

## Real-time I/O Monitoring

### Using iostat

```bash
# Install if not available
sudo apt install sysstat

# Monitor NFS I/O in real-time (1-second intervals)
iostat -x 1

# Monitor specific device/mount
iostat -x 1 | grep nfs

# Extended statistics
iostat -x -d 1
```

### Using iotop

```bash
# Install if needed
sudo apt install iotop

# Monitor I/O by process (shows which processes are using I/O)
sudo iotop -o

# Monitor only processes doing I/O
sudo iotop -a
```

## NFS-Specific Statistics

### NFS Mount Statistics

```bash
# Show NFS mount statistics
nfsstat -m

# Show detailed NFS client statistics
nfsstat -c

# Show server statistics (if running NFS server)
nfsstat -s

# Monitor NFS operations in real-time
watch -n 1 'nfsstat -c | grep -A 10 "Client rpc stats"'
```

### NFS Mount Information

```bash
# Show all NFS mounts
mount | grep nfs

# Show mount options for specific mount
mount | grep /mnt/nfs/traefik-config

# Show NFS version and options
cat /proc/mounts | grep nfs
```

## Network Performance Testing

### Ping Test to NFS Server

```bash
# Get NFS server IP
NFS_SERVER=$(mount | grep /mnt/nfs | awk '{print $1}' | cut -d':' -f1 | head -1)

# Test network latency
ping -c 10 $NFS_SERVER

# Test with different packet sizes
ping -c 5 -s 1024 $NFS_SERVER
ping -c 5 -s 8192 $NFS_SERVER
```

### Network Throughput Test

```bash
# Install iperf3 on both client and server
sudo apt install iperf3

# On NFS server:
iperf3 -s

# On client:
iperf3 -c $NFS_SERVER
```

## File Access Monitoring

### Using inotify to Monitor File Access

```bash
# Install inotify tools
sudo apt install inotify-tools

# Monitor file access in real-time
sudo inotifywait -m -r /mnt/nfs/traefik-config/

# Monitor specific events
sudo inotifywait -m -e access,open,close /mnt/nfs/traefik-config/traefik.yml
```

### Using strace to Monitor Docker Container Access

```bash
# Get container PID
CONTAINER_ID=$(docker ps --format "{{.ID}}" --filter "name=traefik")
PID=$(docker inspect $CONTAINER_ID --format "{{.State.Pid}}")

# Monitor file system calls
sudo strace -e trace=openat,read,close -p $PID 2>&1 | grep nfs
```

## Automated Benchmark Script

```bash
#!/bin/bash
# nfs-benchmark.sh — NFS read performance benchmark

NFS_MOUNT="/mnt/nfs"
TEST_FILE="$NFS_MOUNT/test-file"
RESULTS_FILE="/tmp/nfs-benchmark-$(date +%Y%m%d_%H%M%S).log"

echo "=== NFS Performance Benchmark ===" | tee "$RESULTS_FILE"
echo "Date: $(date)" | tee -a "$RESULTS_FILE"
echo "Mount: $NFS_MOUNT" | tee -a "$RESULTS_FILE"
echo "" | tee -a "$RESULTS_FILE"

# Test 1: Directory listing
echo "--- Directory Listing ---" | tee -a "$RESULTS_FILE"
time ls -la "$NFS_MOUNT/" 2>&1 | tee -a "$RESULTS_FILE"
echo "" | tee -a "$RESULTS_FILE"

# Test 2: Sequential read (create test file first)
echo "--- Sequential Read (100MB) ---" | tee -a "$RESULTS_FILE"
dd if=/dev/zero of="$TEST_FILE" bs=1M count=100 2>/dev/null
time dd if="$TEST_FILE" of=/dev/null bs=1M 2>&1 | tee -a "$RESULTS_FILE"
echo "" | tee -a "$RESULTS_FILE"

# Test 3: Small block read
echo "--- Small Block Read (4K) ---" | tee -a "$RESULTS_FILE"
time dd if="$TEST_FILE" of=/dev/null bs=4K 2>&1 | tee -a "$RESULTS_FILE"
echo "" | tee -a "$RESULTS_FILE"

# Test 4: Multiple small file reads
echo "--- Multiple Small File Reads ---" | tee -a "$RESULTS_FILE"
for i in {1..10}; do
    dd if=/dev/urandom of="$NFS_MOUNT/small-$i" bs=1K count=10 2>/dev/null
done
time (for i in {1..10}; do cat "$NFS_MOUNT/small-$i" > /dev/null; done) 2>&1 | tee -a "$RESULTS_FILE"
echo "" | tee -a "$RESULTS_FILE"

# Test 5: NFS statistics
echo "--- NFS Mount Statistics ---" | tee -a "$RESULTS_FILE"
nfsstat -m 2>&1 | tee -a "$RESULTS_FILE"

# Cleanup
rm -f "$TEST_FILE" "$NFS_MOUNT"/small-*

echo "" | tee -a "$RESULTS_FILE"
echo "Results saved to: $RESULTS_FILE"
```

```bash
# Run the benchmark
chmod +x nfs-benchmark.sh
./nfs-benchmark.sh
```

## Performance Optimization Tips

### Mount Options for Better Performance

```bash
# Optimized NFS mount in /etc/fstab
192.168.50.9:/volume1/docker-swarm/traefik/config /mnt/nfs/traefik-config nfs rw,nfsvers=3,proto=tcp,port=2049,insecure,nolock,soft,timeo=30,retrans=2,rsize=32768,wsize=32768 0 0
```

### Key Performance Mount Options

| Option | Value | Purpose |
|--------|-------|---------|
| `rsize` | 32768 | Read buffer size (32KB) |
| `wsize` | 32768 | Write buffer size (32KB) |
| `timeo` | 30 | Timeout in deciseconds (3 seconds) |
| `retrans` | 2 | Number of retransmissions |
| `proto` | tcp | Use TCP protocol |
| `nolock` | - | Disable file locking for better performance |
| `noatime` | - | Don't update access times |
| `nconnect` | 4-16 | Multiple TCP connections (kernel 5.3+) |

## Troubleshooting Performance Issues

### Check for Common Issues

```bash
# Check NFS server availability
showmount -e 192.168.50.9

# Check network connectivity
traceroute 192.168.50.9

# Check for packet loss
ping -c 100 192.168.50.9 | grep "packet loss"
```

### Monitor System Resources

```bash
# Check CPU usage
top -p $(pgrep nfsd)

# Check memory usage
free -h

# Check network interface statistics
cat /proc/net/dev

# Check for NFS-related errors in logs
dmesg | grep -i nfs
journalctl -u nfs-client.target
```

## Expected Performance Baselines

### Typical NFS Performance (Gigabit Network)

| Workload | Expected Performance |
|----------|---------------------|
| Small files (< 1KB) | 1–10ms read time |
| Medium files (1–10MB) | 10–100MB/s throughput |
| Large files (> 10MB) | 50–100MB/s throughput |
| Directory listings | < 5ms for small directories |

### Alerting Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| Read latency | > 50ms | > 100ms |
| Error rate | > 0.5% | > 1% |
| Throughput | < 20MB/s | < 10MB/s |

## Monitoring in Production

### Continuous Monitoring Setup

```bash
# Add to crontab for regular performance checks
# Run every hour and log results
0 * * * * /path/to/nfs-benchmark.sh >> /var/log/nfs-performance.log 2>&1

# Monitor NFS stats every 5 minutes
*/5 * * * * nfsstat -c | grep -A 5 "Client rpc stats" >> /var/log/nfs-stats.log
```

## Integration with Docker Swarm

### Test NFS Performance Impact on Container Startup

```bash
# Time container deployment
time docker stack deploy -c docker-compose.yml traefik

# Monitor container startup with NFS dependencies
docker service logs -f traefik_traefik | grep -i "config\|error\|start"
```

### Validate Configuration Loading

```bash
# Check if Traefik successfully loads config from NFS
docker service logs traefik_traefik | grep -i "configuration\|provider\|swarm"

# Test configuration reload
docker service update --force traefik_traefik
```

### Docker Swarm Considerations

- Configuration files are typically small (< 1MB) — latency matters more than throughput
- Read performance is more important than write for config mounts
- Network latency affects small file operations disproportionately
- Consider using `noatime` and `nolock` for read-heavy config mounts
- Use `soft` mounts for non-critical config to avoid hanging containers

## fio — Flexible I/O Tester

`fio` provides repeatable, scriptable NFS benchmarks that simulate real workloads.

### Installation

```bash
sudo apt install fio        # Ubuntu/Debian
sudo dnf install fio        # RHEL/Rocky
```

### Sequential Read/Write

```bash
# Sequential read (simulates large file reads)
fio --name=seq-read --directory=/mnt/nfs-share --ioengine=libaio \
    --rw=read --bs=1M --size=512M --numjobs=1 --runtime=60 \
    --time_based --group_reporting

# Sequential write
fio --name=seq-write --directory=/mnt/nfs-share --ioengine=libaio \
    --rw=write --bs=1M --size=512M --numjobs=1 --runtime=60 \
    --time_based --group_reporting
```

### Random I/O (Database Workloads)

```bash
# Random read (4K blocks, simulates database reads)
fio --name=rand-read --directory=/mnt/nfs-share --ioengine=libaio \
    --rw=randread --bs=4k --size=256M --numjobs=4 --runtime=60 \
    --time_based --group_reporting --iodepth=16

# Random read/write mix (70% read, 30% write)
fio --name=mixed-rw --directory=/mnt/nfs-share --ioengine=libaio \
    --rw=randrw --rwmixread=70 --bs=4k --size=256M --numjobs=4 \
    --runtime=60 --time_based --group_reporting --iodepth=16
```

### Small File Operations (Config Files)

```bash
# Simulate many small file reads (Docker config use case)
fio --name=small-files --directory=/mnt/nfs-share --ioengine=libaio \
    --rw=randread --bs=4k --size=1M --numjobs=8 --nrfiles=100 \
    --runtime=30 --time_based --group_reporting

# Metadata-heavy workload (create/stat/delete)
fio --name=metadata --directory=/mnt/nfs-share --ioengine=filecreate \
    --filesize=4k --nrfiles=1000 --numjobs=1
```

### fio Job File (Reusable)

```bash
# Save as nfs-benchmark.fio
cat << 'EOF' > nfs-benchmark.fio
[global]
directory=/mnt/nfs-share/fio-test
ioengine=libaio
time_based
runtime=60
group_reporting

[seq-read-1M]
rw=read
bs=1M
size=512M
numjobs=1

[rand-read-4k]
rw=randread
bs=4k
size=256M
numjobs=4
iodepth=16

[mixed-rw]
rw=randrw
rwmixread=70
bs=8k
size=256M
numjobs=4
iodepth=8
EOF

# Run all tests
mkdir -p /mnt/nfs-share/fio-test
fio nfs-benchmark.fio
rm -rf /mnt/nfs-share/fio-test
```

## nconnect Benchmarking

The `nconnect` mount option (kernel 5.3+) creates multiple TCP connections per mount, improving throughput for parallel workloads.

### Testing Different nconnect Values

```bash
#!/bin/bash
# nconnect-benchmark.sh — Compare NFS performance with different nconnect values

SERVER="192.168.1.10"
EXPORT="/data"
MOUNT="/mnt/nfs-test"
TEST_FILE="$MOUNT/nconnect-test"

for NCONN in 1 4 8 16; do
    echo "=== Testing nconnect=$NCONN ==="

    # Mount with specific nconnect value
    sudo mkdir -p $MOUNT
    if [ "$NCONN" -eq 1 ]; then
        sudo mount -t nfs4 -o rsize=1048576,wsize=1048576 $SERVER:$EXPORT $MOUNT
    else
        sudo mount -t nfs4 -o rsize=1048576,wsize=1048576,nconnect=$NCONN $SERVER:$EXPORT $MOUNT
    fi

    # Verify connections
    echo "TCP connections: $(ss -tn | grep :2049 | wc -l)"

    # Sequential write
    echo "Write:"
    dd if=/dev/zero of=$TEST_FILE bs=1M count=1024 oflag=direct 2>&1 | tail -1

    # Sequential read
    echo "Read:"
    dd if=$TEST_FILE of=/dev/null bs=1M iflag=direct 2>&1 | tail -1

    # Parallel read (4 threads)
    echo "Parallel read (4 threads):"
    fio --name=parallel --filename=$TEST_FILE --ioengine=libaio \
        --rw=read --bs=1M --numjobs=4 --runtime=10 --time_based \
        --group_reporting 2>&1 | grep "READ:"

    # Cleanup and unmount
    rm -f $TEST_FILE
    sudo umount $MOUNT
    echo ""
done
```

### Expected nconnect Results

| nconnect | Sequential Read | Sequential Write | Parallel Read (4 jobs) |
|----------|----------------|------------------|------------------------|
| 1 (default) | ~110 MB/s | ~110 MB/s | ~110 MB/s |
| 4 | ~110 MB/s | ~110 MB/s | ~350 MB/s |
| 8 | ~110 MB/s | ~110 MB/s | ~600 MB/s |
| 16 | ~110 MB/s | ~110 MB/s | ~900 MB/s |

Note: Sequential single-thread performance stays the same. `nconnect` benefits parallel operations by distributing I/O across multiple connections. Actual results depend on network (1GbE vs 10GbE), server disk, and CPU.

### When to Use nconnect

- Multiple processes/containers accessing the same NFS mount simultaneously
- Large file transfers with multiple threads
- CI/CD workloads with heavy parallel builds
- NOT useful for single-threaded sequential access

## NFS Version Performance Comparison

### NFSv3 vs NFSv4 vs NFSv4.2

| Feature | NFSv3 | NFSv4 | NFSv4.2 |
|---------|--------|--------|----------|
| Protocol | Stateless | Stateful | Stateful |
| Ports required | Multiple (111, 2049, dynamic) | Single (2049) | Single (2049) |
| Compound operations | No | Yes (fewer round trips) | Yes |
| Delegation | No | Yes (client caching) | Yes |
| Server-side copy | No | No | Yes |
| Sparse files | No | No | Yes |
| Locking | Separate NLM protocol | Built-in | Built-in |

### Performance Characteristics

| Workload | NFSv3 | NFSv4 | NFSv4.2 |
|----------|--------|--------|----------|
| Small file metadata | Slower (multiple RPCs) | Faster (compound ops) | Fastest |
| Large sequential I/O | Similar | Similar | Similar |
| Random I/O | Similar | Slightly better (delegation) | Better (delegation) |
| File copy (server-to-server) | Client passes through | Client passes through | Server-side (no network transfer) |
| Firewall-friendly | No (many ports) | Yes (port 2049 only) | Yes (port 2049 only) |
| WAN performance | Poor (chatty protocol) | Better (fewer round trips) | Best |

### Benchmarking Version Differences

```bash
#!/bin/bash
# nfs-version-benchmark.sh — Compare NFS version performance

SERVER="192.168.1.10"
EXPORT="/data"
MOUNT="/mnt/nfs-version-test"

for VERS in 3 4 4.2; do
    echo "=== NFSv$VERS ==="

    sudo mkdir -p $MOUNT
    sudo mount -t nfs -o vers=$VERS,rsize=1048576,wsize=1048576 $SERVER:$EXPORT $MOUNT

    # Metadata test (small file operations)
    echo "Metadata (create 100 files):"
    time (for i in $(seq 1 100); do touch "$MOUNT/test-$i"; done) 2>&1

    echo "Metadata (stat 100 files):"
    time (for i in $(seq 1 100); do stat "$MOUNT/test-$i" > /dev/null; done) 2>&1

    echo "Metadata (delete 100 files):"
    time (for i in $(seq 1 100); do rm "$MOUNT/test-$i"; done) 2>&1

    # Sequential I/O
    echo "Sequential write (256MB):"
    dd if=/dev/zero of="$MOUNT/bigfile" bs=1M count=256 oflag=direct 2>&1 | tail -1

    echo "Sequential read (256MB):"
    dd if="$MOUNT/bigfile" of=/dev/null bs=1M iflag=direct 2>&1 | tail -1

    rm -f "$MOUNT/bigfile"
    sudo umount $MOUNT
    echo ""
done
```

### NFSv4.2 Server-Side Copy

```bash
# Server-side copy avoids transferring data through the client
# Requires NFSv4.2 on both server and client

# Mount with NFSv4.2
sudo mount -t nfs -o vers=4.2 server:/data /mnt/data

# Use copy_file_range (transparent with cp on modern kernels)
cp /mnt/data/source-file /mnt/data/destination-file

# Verify server-side copy is being used
nfsstat -c | grep copy
# Look for "copy" operation counts incrementing
```
