# Benchmarking Amazon EBS Volumes with FIO

Benchmarking is how you measure what your storage can actually do — versus what the spec
sheet promises — and understand the factors that shape it. For Amazon EBS the big levers
are the **volume type**, the **application I/O profile** (block size, sequential vs
random, read vs write, concurrency), and the **host-side logical volume layout** (single
volume vs a striped array).

This article walks an end-to-end workflow: attach and format volumes, run realistic
workloads with **FIO (Flexible I/O Tester)**, read the numbers, confirm them in
**CloudWatch**, and then combine volumes with **RAID 0 striping** to multiply IOPS. The
commands match a typical EC2 + EBS setup: a `gp3` boot volume, two `st1` volumes for logs,
and four `gp3` volumes for data.

> **What you're measuring.** Two headline numbers: **IOPS** (I/O operations per second —
> dominates for small, random I/O like databases) and **throughput / bandwidth** (MiB/s —
> dominates for large, sequential I/O like log writes or media). Which one matters depends
> entirely on your workload's block size and access pattern.

---

## EBS volume types at a glance

| Type | Class | Best for | Baseline behavior |
|------|-------|----------|-------------------|
| `gp3` | SSD | General purpose, most workloads | 3,000 IOPS + 125 MiB/s baseline, provisionable to 16,000 IOPS / 1,000 MiB/s |
| `io2` / `io2 Block Express` | SSD | Latency-sensitive, high-IOPS databases | Provisioned up to 64,000 (256,000 Block Express) IOPS |
| `st1` | HDD | Big, sequential throughput (logs, streaming) | ~40 MiB/s per TB, burst to 250 MiB/s |
| `sc1` | HDD | Cold, infrequent sequential access | Lower throughput, lowest cost |

Two things to internalize before benchmarking:

- **HDD (`st1`/`sc1`) throughput scales with size.** A minimum 125 GiB `st1` volume gets
  roughly **5 MiB/s baseline** with burst up to ~31 MiB/s — far below the 250 MiB/s a
  multi-TB `st1` can reach. Small HDD volumes are slow by design.
- **`gp3` gives a flat 3,000 IOPS baseline** regardless of size, so it's the natural
  starting point for random-I/O data volumes — and the thing you'll try to exceed later
  with striping.

## Prerequisites

- An EC2 instance with several EBS volumes attached (this walkthrough assumes two `st1`
  log volumes and four `gp3` data volumes, plus the `gp3` root).
- Connect with **Session Manager** (no SSH keys or open ports needed), or SSH if you
  prefer.
- Familiarity with `lsblk`, `mkfs`, and `mount`.

## Step 1: Install FIO

FIO generates configurable I/O workloads and reports IOPS, bandwidth, and latency. Install
it from the default repos:

```bash
# Amazon Linux 2 / RHEL family
sudo yum install -y fio

# Amazon Linux 2023 / newer
sudo dnf install -y fio

# Ubuntu / Debian
sudo apt install -y fio
```

## Step 2: Identify and format the volumes

List the block devices to see what's attached and how big each one is:

```bash
lsblk
```

```text
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda    202:0    0    8G  0 disk
└─xvda1 202:1    0    8G  0 part /
xvdb    202:16   0  125G  0 disk          # Logs1 (st1)
xvdc    202:32   0  125G  0 disk          # Logs2 (st1)
xvdd    202:48   0    8G  0 disk          # Data1 (gp3)
xvde    202:64   0    8G  0 disk          # Data2 (gp3)
xvdf    202:80   0    8G  0 disk          # Data3 (gp3)
xvdg    202:96   0    8G  0 disk          # Data4 (gp3)
```

The root volume (`xvda`) is first. The 125 GiB devices are the `st1` log volumes; the 8 GiB
devices are the `gp3` data volumes.

> **Do not create a file system on the root device (`xvda`).** Only format the non-root
> data/log volumes.

Formatting isn't strictly required to benchmark a raw device, but doing it the way
production will run gives more representative numbers. Create an XFS file system on the
first log volume and mount it:

```bash
sudo mkfs -t xfs /dev/xvdb
sudo mkdir /logs1
sudo mount /dev/xvdb /logs1
```

Repeat for the first data volume:

```bash
sudo mkfs -t xfs /dev/xvdd
sudo mkdir /data1
sudo mount /dev/xvdd /data1
```

Confirm both are mounted:

```bash
lsblk
```

```text
xvdb    202:16   0  125G  0 disk /logs1
xvdd    202:48   0    8G  0 disk /data1
```

## Step 3: Benchmark the log volume (sequential write throughput)

Log files are written **sequentially**, so the metric that matters is **throughput
(MiB/s)**, not IOPS. The question: does a minimum-size `st1` volume deliver enough
bandwidth for the app's log writes?

FIO parameters used here:

- `--filename=/dev/xvdb` — target device (the log volume).
- `--direct=1` — bypass the OS page cache so you measure the device, not RAM.
- `--rw=write` — sequential writes.
- `--bs=1024k` — 1 MB block size (large, matching sequential log writes).
- `--runtime=180` — run for 180 seconds.
- `--allow_mounted_write=1` — permit writing to a mounted device.
- `--name=...` — a label for the job.

```bash
sudo fio --filename=/dev/xvdb --direct=1 --rw=write --bs=1024k --runtime=180 --allow_mounted_write=1 --name=fio_logs_seq_write
```

> **Why 180 seconds?** 60s is enough for FIO to converge, but CloudWatch polls EBS at
> **1-minute intervals**. Running for 3 minutes gives several data points so the FIO result
> and the CloudWatch graph line up.

Read the **`bw`** (bandwidth) value on the line beginning with `write:`. On a 125 GiB
`st1` volume you'll typically see roughly **31 MiB/s** (~33,000 KB/s) — the burst rate.
That comfortably clears a requirement of, say, 5 MiB/s, so the cheap `st1` volume is fine
for log writes.

## Step 4: Benchmark the data volume (random write IOPS)

Primary application data is usually **small and random**, so now **IOPS** is the metric.
The app writes in ~16 KB chunks, so set the block size accordingly and switch to a random
write pattern:

```bash
sudo fio --filename=/dev/xvdd --direct=1 --rw=randwrite --bs=16k --runtime=120 --allow_mounted_write=1 --name=fio_data_rand_write
```

Read the **IOPS** value at the top of the summary. A single-threaded 16 KB random write
workload on a `gp3` volume typically lands around **700–1,100 IOPS** — well under the
3,000 IOPS `gp3` baseline, so one `gp3` volume handles the write path easily.

> **Experiment:** rerun with `--bs=8k` or `--bs=64k`. Smaller blocks push IOPS up (more,
> smaller ops); larger blocks push IOPS down but throughput up. This is the IOPS-vs-block-
> size tradeoff in action.

## Step 5: Benchmark the data volume under concurrent reads

Peak read load is often **many parallel small reads**. Simulate six concurrent jobs of
4 KB random reads. Two new parameters:

- `--numjobs=6` — six parallel workers (matching peak concurrent sessions).
- `--group_reporting` — aggregate the workers into one combined result.

```bash
sudo fio --filename=/dev/xvdd --direct=1 --rw=randread --bs=4k --numjobs=6 --runtime=90 --group_reporting --name=fio_data_rand_read
```

Read the **IOPS** on the `read:` line. This parallel 4 KB read workload will **hit the
~3,000 IOPS ceiling** of a stock `gp3` volume. That's the signal that the single volume is
now the bottleneck for peak reads — you need more IOPS.

Your options:

1. **Provision more IOPS on `gp3`** (up to 16,000, for extra cost).
2. **Switch to `io2`** (up to 64,000 IOPS).
3. **Stripe multiple volumes with RAID 0** to combine their IOPS — covered in Step 7.

## Step 6: Confirm the results in CloudWatch

EBS publishes metrics to CloudWatch automatically. In the EC2 console → **Elastic Block
Store → Volumes**, select a volume and open the **Monitoring** tab.

- **Throughput / bandwidth (MiB/s):** open the **Write bandwidth** widget, set the period
  to **1 minute**. For the log test it should climb toward ~33,000 KiB and plateau —
  matching FIO.
- **IOPS:** open the **Write throughput** and **Read throughput** widgets, again at a
  **1-minute** period (view as 1-minute sums to read ops/sec). These should roughly track
  your random write and read FIO runs.

> **Why the numbers won't match exactly.** CloudWatch samples at 1-minute intervals, which
> rarely align perfectly with a short test window. The longer you run FIO, the closer the
> CloudWatch graph converges to the FIO result.

## Step 7: Multiply IOPS with RAID 0 striping

**Disk striping (RAID 0)** presents several EBS volumes to the OS as one logical device and
spreads I/O across all of them, combining their IOPS and throughput. Build a RAID 0 array
from the three unused `gp3` data volumes with `mdadm`:

```bash
sudo mdadm --create --verbose /dev/md0 --level=0 --name=RAID_FROM_DATA --raid-devices=3 /dev/xvde /dev/xvdf /dev/xvdg
```

Watch it initialize and confirm it's active:

```bash
cat /proc/mdstat
sudo mdadm --detail /dev/md0
```

Create a file system and mount it:

```bash
sudo mkfs.ext4 /dev/md0
sudo mkdir -p /mnt/raid
sudo mount /dev/md0 /mnt/raid
sudo lsblk
```

```text
xvde    202:64   0    8G  0 disk
└─md0     9:0    0   24G  0 raid0 /mnt/raid
xvdf    202:80   0    8G  0 disk
└─md0     9:0    0   24G  0 raid0 /mnt/raid
xvdg    202:96   0    8G  0 disk
└─md0     9:0    0   24G  0 raid0 /mnt/raid
```

Re-run the parallel random read benchmark against the array:

```bash
sudo fio --filename=/dev/md0 --direct=1 --rw=randread --bs=4k --numjobs=6 --runtime=60 --group_reporting --name=fio_raid_rand_read
```

With three `gp3` volumes at 3,000 IOPS each, the striped device should deliver **well over
8,000 IOPS** — roughly 3× a single volume, blowing past the earlier 3,000 IOPS wall.

> **The tradeoffs of RAID 0.** There is **no redundancy**: if any one member volume fails,
> the whole logical volume is lost. Performance is also capped by the **slowest** member,
> so stripe volumes of the same type and size. For durability, rely on **EBS snapshots**
> (and note that snapshotting a striped array needs a consistent, application- or
> filesystem-level freeze across all members).

## The instance is also a bottleneck (volume/instance mismatch)

A volume's provisioned numbers are only reachable if the **instance** can drive them. Two
independent ceilings apply, and **the lower one wins**:

- **The volume's** provisioned IOPS/throughput (what you configured on the `gp3`/`io2`
  volume).
- **The instance's** dedicated EBS bandwidth and IOPS limit (a property of the instance
  type and size).

So a `gp3` volume provisioned for 6,000 IOPS attached to an instance capped at 4,000 EBS
IOPS delivers **4,000** — and your FIO run will show it. If a benchmark falls short of the
volume spec for no obvious reason, suspect the instance. Larger instances get more EBS
bandwidth, so sizing up the instance (not just the volume) is often the fix.

> **EBS-optimized** is what gives an instance dedicated bandwidth for EBS I/O (separate
> from general network traffic). It's on by default on essentially all current-generation
> instances, so this is rarely a toggle today — but the *ceiling* it implies is very real.

### Watch for instance-level burst

Smaller instances have a **baseline** EBS rate plus the ability to **burst** above it for a
limited window (historically ~30 minutes), then drop back to baseline. A short benchmark
can accidentally measure the burst rate and overstate sustained performance. To measure
**steady state**, run long enough to exhaust burst credits (several minutes at least), or
test on an instance whose baseline already meets your target. This is the instance-side
cousin of the `st1`/`gp2` volume burst behavior.

## Initialize volumes restored from a snapshot

New, empty EBS volumes deliver full performance immediately. But a volume **created from a
snapshot** lazy-loads its blocks from Amazon S3 on first access, so the *first* read of
each block is slow. If you benchmark such a volume cold, the numbers are artificially low.

Options:

- **Pre-warm / initialize** by reading every block once before benchmarking (e.g.
  `sudo fio --filename=/dev/xvdX --rw=read --bs=1M --iodepth=32 --ioengine=libaio --direct=1 --name=init`
  or `sudo dd if=/dev/xvdX of=/dev/null bs=1M`).
- **Fast Snapshot Restore (FSR)** — enable it on the snapshot so restored volumes are fully
  initialized on creation and hit full performance with no pre-warming. Use it when you
  need optimal performance immediately (e.g. DR restores).

## When RAID 0 is (and isn't) the right tool

Striping (Step 7) is worth it when a **single volume can't meet the requirement**. Rough
thresholds where RAID 0 across multiple EBS volumes makes sense:

- Storage requirement **> 16 TiB** (a single volume's max historically).
- Throughput requirement **> ~1,000 MiB/s**.
- IOPS requirement **> 64,000 @ 16K** (beyond a single high-end volume).

Below those, prefer a **single, larger/faster `gp3` or `io2`** volume — it's simpler and
has no striping failure-domain risk.

**Do not use RAID for redundancy on EBS:**

- EBS already **replicates data within the Availability Zone**, so mirroring/parity is
  redundant with what EBS gives you.
- **RAID 1** halves usable EBS bandwidth (every write goes to two volumes).
- **RAID 5/6** burns ~20–30% of I/O on parity.

For durability, use **EBS snapshots**, not host-side RAID.

## Persisting mounts across reboots

The `mount` commands above don't survive a reboot. For a real deployment, add entries to
`/etc/fstab` using stable identifiers (UUIDs), and for a RAID array also save the mdadm
config:

```bash
# Save the array definition so it re-assembles on boot
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf

# Find UUIDs
sudo blkid

# Example fstab entries (use your actual UUIDs)
# UUID=<data1-uuid>  /data1     xfs   defaults,nofail  0  2
# UUID=<raid-uuid>   /mnt/raid  ext4  defaults,nofail  0  2
```

> Use `nofail` so a missing EBS volume doesn't block instance boot. On some AMIs you'll
> also want to rebuild the initramfs after editing `/etc/mdadm.conf`.

## Cleanup

To tear down the RAID array (e.g., to rebuild it from the log volumes as a follow-up):

```bash
sudo umount /mnt/raid
sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock /dev/xvde /dev/xvdf /dev/xvdg
```

To unmount a plain volume before repurposing it:

```bash
sudo umount /dev/xvdb
```

## Key takeaways

- **Match the benchmark to the workload.** Sequential + large block = throughput test;
  random + small block = IOPS test. Use `--direct=1` so you measure the device, not cache.
- **Block size drives IOPS.** The same volume reports very different IOPS at 4k vs 64k.
  Benchmark with the block size your app actually uses.
- **Small `st1` volumes are throughput-limited by size**, and **stock `gp3` caps at 3,000
  IOPS** — know these baselines before you're surprised in production.
- **Concurrency matters.** A single-threaded test can look fine while the parallel peak
  saturates the volume. Use `--numjobs` to model real concurrency.
- **RAID 0 multiplies IOPS/throughput but sacrifices redundancy.** Provisioned IOPS
  (`gp3`/`io2`) is the safer path when durability matters.
- **Validate with CloudWatch** at a 1-minute period, and run tests long enough for the
  metrics to converge.

---

### Sources

- [Amazon EBS volume types](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)
- [Benchmark EBS volumes (I/O performance)](https://docs.aws.amazon.com/ebs/latest/userguide/benchmark_procedures.html)
- [Amazon EBS volume performance on Linux](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-performance.html)
- [RAID configuration on Linux (Amazon EBS)](https://docs.aws.amazon.com/ebs/latest/userguide/raid-config.html)
- [Make an EBS volume available for use on Linux](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html)
- [FIO documentation](https://fio.readthedocs.io/en/latest/fio_doc.html)
