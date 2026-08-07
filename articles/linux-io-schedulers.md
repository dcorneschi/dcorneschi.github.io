# Linux I/O Schedulers

The I/O scheduler (also called the elevator) determines the order in which block I/O requests are submitted to storage devices. Choosing the right scheduler affects latency, throughput, and fairness for disk-intensive workloads.

## Single-Queue vs Multi-Queue

Linux historically used single-queue (sq) block layer schedulers. Starting with kernel 4.12+, the multi-queue (mq) block layer became default. RHEL 7 uses the old sq schedulers; RHEL 8+ and Ubuntu 18.04+ use mq schedulers exclusively.

| Block Layer | Kernel | Schedulers | Used by |
|-------------|--------|------------|---------|
| Single-queue (sq) | < 4.12 | `noop`, `deadline`, `cfq` | RHEL 6, RHEL 7, Ubuntu 14.04–16.04 |
| Multi-queue (mq) | ≥ 4.12 | `none`, `mq-deadline`, `bfq`, `kyber` | RHEL 8+, Ubuntu 18.04+ |

> **Note:** The mq schedulers are not renamed versions of the old ones — they are completely different implementations designed for modern NVMe/SSD hardware with multiple hardware queues.

---

## Scheduler Comparison

### Single-Queue Schedulers (Legacy)

| Scheduler | Description | Best for |
|-----------|-------------|----------|
| `anticipatory` | Delays I/O briefly anticipating sequential follow-up requests. Removed in kernel 2.6.33. | RHEL 4-5 default. Sequential workloads on spinning disks. |
| `noop` | No reordering — FIFO. Minimal CPU overhead. | SSDs, NVMe, virtualised disks (host does scheduling) |
| `deadline` | Guarantees a maximum latency for reads/writes. Prevents starvation. | Databases, latency-sensitive workloads, mixed read/write |
| `cfq` | Completely Fair Queuing — allocates time slices per process. Default on RHEL 6/7. | Desktop, general-purpose, multi-user systems |

### Multi-Queue Schedulers (Modern)

| Scheduler | Description | Best for |
|-----------|-------------|----------|
| `none` | No reordering — passes I/O directly to the device. Equivalent to old `noop`. | NVMe, fast SSDs, virtual disks |
| `mq-deadline` | Multi-queue version of deadline. Ensures requests are served within a deadline. | Databases, latency-sensitive, SATA SSDs |
| `bfq` | Budget Fair Queuing — provides fairness and low latency for interactive tasks. Higher CPU usage. | Desktop, interactive workloads, USB storage |
| `kyber` | Lightweight latency-targeted scheduler. Self-tuning with minimal configuration. | Fast SSDs, cloud workloads with latency targets |

---

## Default Schedulers by Distribution

| Distribution | Kernel | Default Scheduler (HDD) | Default Scheduler (SSD/NVMe) | Block Layer |
|-------------|--------|------------------------|------------------------------|-------------|
| RHEL 4 | 2.6.9 | `anticipatory` | N/A | Single-queue |
| RHEL 5 | 2.6.18 | `anticipatory` | N/A | Single-queue |
| RHEL 6 | 2.6.32 | `cfq` | `cfq` | Single-queue |
| RHEL 7 | 3.10 | `deadline` | `deadline` | Single-queue |
| RHEL 8 | 4.18 | `mq-deadline` | `none` | Multi-queue |
| RHEL 9 | 5.14 | `mq-deadline` | `none` | Multi-queue |
| RHEL 10 | 6.x | `mq-deadline` | `none` | Multi-queue |
| Ubuntu 16.04 | 4.4 | `deadline` | `deadline` | Single-queue |
| Ubuntu 18.04 | 4.15/5.4 | `mq-deadline` | `none` | Multi-queue |
| Ubuntu 20.04 | 5.4 | `mq-deadline` | `none` | Multi-queue |
| Ubuntu 22.04 | 5.15 | `mq-deadline` | `none` | Multi-queue |
| Ubuntu 24.04 | 6.8 | `mq-deadline` | `none` | Multi-queue |

> **Note:** On RHEL 8+/Ubuntu 18.04+, the kernel automatically selects `none` for NVMe/fast SSDs and `mq-deadline` for rotational drives. This auto-detection is handled by udev rules.

---

## Checking the Current Scheduler

```bash
# Show current scheduler for a device (active one is in brackets)
cat /sys/block/sda/queue/scheduler

# Example output (single-queue):
# noop [deadline] cfq

# Example output (multi-queue):
# [mq-deadline] none bfq kyber

# Check all block devices at once
for dev in /sys/block/sd* /sys/block/nvme*; do
    echo "$(basename $dev): $(cat $dev/queue/scheduler 2>/dev/null)"
done

# Check if device is rotational
cat /sys/block/sda/queue/rotational
# 1 = HDD, 0 = SSD/NVMe

# Check number of hardware queues (mq)
cat /sys/block/sda/queue/nr_requests
ls /sys/block/nvme0n1/queue/
```

---

## Changing the Scheduler

### Temporary (Until Reboot)

```bash
# Change scheduler for sda
echo "mq-deadline" > /sys/block/sda/queue/scheduler

# Change to none for NVMe
echo "none" > /sys/block/nvme0n1/queue/scheduler

# Change to bfq
echo "bfq" > /sys/block/sda/queue/scheduler

# Verify
cat /sys/block/sda/queue/scheduler
```

### Persistent — udev Rules (RHEL 8+, Ubuntu 18.04+)

Create a udev rule that sets the scheduler based on device type:

```bash
# /etc/udev/rules.d/60-io-scheduler.rules

# Set mq-deadline for rotational drives (HDD)
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"

# Set none for non-rotational drives (SSD)
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"

# Set none for NVMe
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/scheduler}="none"
```

Reload udev rules:

```bash
udevadm control --reload-rules
udevadm trigger
```

### Persistent — Kernel Command Line (All Versions)

Add to GRUB kernel parameters:

```bash
# RHEL 6/7 (single-queue)
elevator=deadline

# RHEL 8+ (multi-queue) — sets default for all devices
# Note: elevator= is ignored on mq; use udev rules instead
```

On RHEL 6/7:

```bash
# Edit /etc/default/grub
GRUB_CMDLINE_LINUX="... elevator=deadline"

# Regenerate GRUB config
grub2-mkconfig -o /boot/grub2/grub.cfg        # BIOS
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg  # UEFI
```

> **Important:** The `elevator=` kernel parameter only works with the single-queue block layer (RHEL 6/7). On RHEL 8+ with the multi-queue layer, it is ignored. Use udev rules instead.

### Persistent — tuned Profiles (RHEL)

`tuned` can set the I/O scheduler as part of a performance profile:

```bash
# Install tuned (if not already present)
yum install tuned          # RHEL/CentOS
apt install tuned          # Ubuntu/Debian
systemctl enable --now tuned

# Check current profile
tuned-adm active

# List available profiles
tuned-adm list

# Apply a profile (sets scheduler among other tunings)
tuned-adm profile throughput-performance   # sets mq-deadline
tuned-adm profile latency-performance      # sets mq-deadline
tuned-adm profile virtual-guest            # sets none

# Check what scheduler tuned set
tuned-adm verify
cat /sys/block/sda/queue/scheduler
```

Common tuned profiles and their I/O scheduler:

| Profile | Scheduler | Use case |
|---------|-----------|----------|
| `throughput-performance` | `mq-deadline` | High-throughput servers |
| `latency-performance` | `mq-deadline` | Low-latency workloads |
| `virtual-guest` | `none` | VMs (host handles scheduling) |
| `virtual-host` | `mq-deadline` | Hypervisors |
| `desktop` | `bfq` | Interactive desktop |
| `balanced` | `mq-deadline` | General purpose |

---

## Scheduler Tuning Parameters

Each scheduler exposes tunables in `/sys/block/<dev>/queue/iosched/`:

### mq-deadline

```bash
ls /sys/block/sda/queue/iosched/
# fifo_batch  front_merges  read_expire  write_expire  writes_starved

# Read deadline in ms (default: 500)
cat /sys/block/sda/queue/iosched/read_expire

# Write deadline in ms (default: 5000)
cat /sys/block/sda/queue/iosched/write_expire

# Number of reads to serve before switching to writes (default: 2)
cat /sys/block/sda/queue/iosched/writes_starved

# Batch size for FIFO dispatch (default: 16)
cat /sys/block/sda/queue/iosched/fifo_batch

# Tune: lower read deadline for database workloads
echo 100 > /sys/block/sda/queue/iosched/read_expire
echo 1000 > /sys/block/sda/queue/iosched/write_expire
```

### bfq

```bash
ls /sys/block/sda/queue/iosched/
# back_seek_max  back_seek_penalty  fifo_expire_async  fifo_expire_sync
# low_latency  max_budget  slice_idle  strict_guarantees  timeout_sync

# Enable low latency mode (default: 1)
cat /sys/block/sda/queue/iosched/low_latency

# Disable for throughput workloads
echo 0 > /sys/block/sda/queue/iosched/low_latency
```

### kyber

```bash
ls /sys/block/sda/queue/iosched/
# read_lat_nsec  write_lat_nsec

# Target read latency in nanoseconds (default: 2000000 = 2ms)
cat /sys/block/sda/queue/iosched/read_lat_nsec

# Target write latency in nanoseconds (default: 10000000 = 10ms)
cat /sys/block/sda/queue/iosched/write_lat_nsec

# Tune for tighter latency
echo 1000000 > /sys/block/sda/queue/iosched/read_lat_nsec
echo 5000000 > /sys/block/sda/queue/iosched/write_lat_nsec
```

### Queue Depth

```bash
# Max number of requests in the queue (applies to all schedulers)
cat /sys/block/sda/queue/nr_requests

# Increase for throughput (default varies: 64–256)
echo 256 > /sys/block/sda/queue/nr_requests

# Decrease for latency
echo 32 > /sys/block/sda/queue/nr_requests
```

---

## Recommendations by Workload

| Workload | Device | Recommended Scheduler | Reasoning |
|----------|--------|----------------------|-----------|
| Database (PostgreSQL, MySQL) | SSD | `mq-deadline` | Bounded latency, prevents write starvation |
| Database (PostgreSQL, MySQL) | NVMe | `none` or `mq-deadline` | NVMe has internal scheduling; `none` adds least overhead |
| Web server (nginx, Apache) | SSD | `mq-deadline` | Predictable latency under mixed read/write |
| File server (Samba, NFS) | HDD | `mq-deadline` | Prevents read starvation from bulk writes |
| Desktop / Interactive | SSD/HDD | `bfq` | Fair scheduling, responsive UI under load |
| Virtual machine (guest) | Virtual disk | `none` | Host hypervisor handles scheduling |
| Hypervisor (host) | SSD/HDD | `mq-deadline` | Fair I/O across multiple VMs |
| Streaming / sequential I/O | HDD | `mq-deadline` | Minimal reordering for sequential access |
| NVMe storage | NVMe | `none` | Device has many internal queues; kernel scheduling adds overhead |
| Elasticsearch / Kafka | SSD | `none` or `kyber` | High throughput, low latency |
| USB / SD card storage | USB | `bfq` | Fair queuing for slow devices |

---

## RHEL 6/7 Specific Configuration

### Setting the Scheduler on RHEL 4/5

```bash
# Temporary
echo deadline > /sys/block/sda/queue/scheduler

# Persistent via GRUB
# Edit /etc/grub.conf and add elevator=deadline to the end of the kernel line:
vi /etc/grub.conf
# kernel /vmlinuz-2.6.18-xxx ... elevator=deadline
```

Available schedulers on RHEL 4/5: `anticipatory`, `noop`, `deadline`, `cfq`

### Setting the Scheduler on RHEL 6

```bash
# Temporary
echo deadline > /sys/block/sda/queue/scheduler

# Persistent via GRUB
# Edit /boot/grub/grub.conf and add to the kernel line:
# elevator=deadline

# Or per-device via rc.local
echo 'echo deadline > /sys/block/sda/queue/scheduler' >> /etc/rc.local
```

### Setting the Scheduler on RHEL 7

```bash
# Temporary
echo deadline > /sys/block/sda/queue/scheduler

# Persistent via GRUB
# /etc/default/grub:
GRUB_CMDLINE_LINUX="... elevator=deadline"
grub2-mkconfig -o /boot/grub2/grub.cfg

# Or use tuned
tuned-adm profile throughput-performance
```

### cfq Tunables (RHEL 6/7 Only)

```bash
ls /sys/block/sda/queue/iosched/
# back_seek_max  back_seek_penalty  fifo_expire_async  fifo_expire_sync
# group_idle  group_isolation  low_latency  quantum  slice_async
# slice_async_rq  slice_idle  slice_sync  target_latency

# Time slice for sync I/O (ms, default: 100)
cat /sys/block/sda/queue/iosched/slice_sync

# Time slice for async I/O (ms, default: 40)
cat /sys/block/sda/queue/iosched/slice_async

# Idle time waiting for next request (ms, default: 8)
cat /sys/block/sda/queue/iosched/slice_idle

# Disable slice_idle for SSDs (improves throughput)
echo 0 > /sys/block/sda/queue/iosched/slice_idle
echo 1 > /sys/block/sda/queue/iosched/quantum
```

### deadline Tunables (RHEL 6/7)

```bash
# Read expire in ms (default: 500)
cat /sys/block/sda/queue/iosched/read_expire

# Write expire in ms (default: 5000)
cat /sys/block/sda/queue/iosched/write_expire

# Writes starved (reads before writes, default: 2)
cat /sys/block/sda/queue/iosched/writes_starved

# FIFO batch size (default: 16)
cat /sys/block/sda/queue/iosched/fifo_batch
```

---

## Monitoring I/O Scheduler Performance

```bash
# I/O queue depth (requests in flight)
cat /sys/block/sda/queue/nr_requests
cat /sys/block/sda/inflight

# I/O stats per device
cat /proc/diskstats | grep sda
iostat -x 1

# Average queue size and wait times
iostat -x -d sda 2

# Per-cgroup I/O stats (RHEL 8+, cgroups v2)
cat /sys/fs/cgroup/io.stat

# blktrace for detailed I/O tracing
blktrace -d /dev/sda -o trace &
blkparse -i trace.blktrace.0

# Check scheduler-related stats
cat /sys/block/sda/stat
# Fields: reads_completed reads_merged read_sectors read_time
#         writes_completed writes_merged write_sectors write_time
#         io_in_progress io_time weighted_io_time
```

---

## Troubleshooting

### Scheduler shows `none` when I expect mq-deadline

On NVMe devices, the kernel defaults to `none` because NVMe controllers have their own internal scheduling with multiple hardware queues. This is correct behaviour — adding a kernel scheduler on top adds CPU overhead without benefit.

### "elevator=deadline" has no effect on RHEL 8+

The `elevator=` kernel parameter only works with the single-queue block layer (RHEL 6/7). On RHEL 8+ (multi-queue), it is silently ignored. Use udev rules or tuned profiles instead.

### High I/O wait despite correct scheduler

The scheduler only reorders requests — it cannot fix:
- Undersized disks (IOPS saturated)
- Filesystem fragmentation
- Application-level contention (too many writers)
- Incorrect `nr_requests` (too low = queue starvation, too high = latency spikes)

Check with:

```bash
iostat -x 2
# Look at: %util (>90% = saturated), await (high = slow device), avgqu-sz (queue depth)
```

### bfq not available

`bfq` is compiled as a module on some distributions. Load it:

```bash
modprobe bfq
echo bfq > /sys/block/sda/queue/scheduler
```

To load automatically at boot:

```bash
echo "bfq" > /etc/modules-load.d/bfq.conf
```

---

## Quick Reference

```bash
# Check current scheduler
cat /sys/block/sda/queue/scheduler

# Change scheduler (temporary)
echo mq-deadline > /sys/block/sda/queue/scheduler

# Check if SSD or HDD
cat /sys/block/sda/queue/rotational

# View scheduler tunables
ls /sys/block/sda/queue/iosched/

# Set via udev rule (persistent)
# /etc/udev/rules.d/60-io-scheduler.rules
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"

# RHEL 6/7: set via GRUB
# elevator=deadline in GRUB_CMDLINE_LINUX

# RHEL 8+: use tuned
tuned-adm profile throughput-performance

# Check all devices
for d in /sys/block/sd* /sys/block/nvme*; do echo "$(basename $d): $(cat $d/queue/scheduler 2>/dev/null)"; done
```
