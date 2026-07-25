# Understanding **iostat -x** Output

## Overview

The `iostat` command is one of the most important tools for monitoring block device I/O performance on Linux systems. Adding the `-x` flag outputs **extended block device I/O statistics**, providing much more granular insight into disk performance than the basic output.

This article explains what `iostat -x` does, how to read its output, what each field means, and how to identify common disk performance issues like stalled storage.


## What Does iostat -x Do?

Adding the `-x` flag to `iostat` outputs extended block device I/O statistics. Instead of just showing `tps` (I/O per second), the extended statistics break this into separate `r/s` (reads per second) and `w/s` (writes per second) columns, along with many other metrics. The extended statistics are generally more useful for understanding and monitoring the I/O load on a system.

### How iostat Works Internally

The `iostat` utility reads `/proc/diskstats` and computes the difference between two samples taken at a specified interval (e.g., every 5 seconds). Since many columns are expressed as a quantity per second, calculated differences between samples are divided by the sample interval time.

**Example:** If the number of completed reads was 150 in the last sample and 200 in the current sample with a 2-second interval, then `r/s` = (200-150)/2 = 25.00 reads per second.

### Important: The First Sample

When `/proc/diskstats` is first read, there is no previous sample to compare against. In this case, `iostat` compares against an all-zeros line and uses the system's uptime as the interval. This means **the first output from `iostat` is an average from boot time until now**.

For this reason, always use at minimum something like:

```bash
iostat -x 5 1 -y
```

The `-y` flag skips the boot-time average, and then outputs one sample averaged over the last 5 seconds.


## Installation

The `iostat` utility is provided by the **sysstat** package:

```bash
yum install sysstat
```

Version history across RHEL releases:
- RHEL 10: sysstat 12.7.6
- RHEL 9: sysstat 12.5.4
- RHEL 8: sysstat 11.7.3
- RHEL 7: sysstat 10.1.5
- RHEL 6: sysstat 9.0.4
- RHEL 5: sysstat 7.0.2


## Data Source: /proc/diskstats

The common data source for `iostat`, `sar`, `collectl`, PCP (Performance Co-Pilot), and similar tools is `/proc/diskstats`. The fields are:

```
Maj Minor Dev  #1   #2   #3   #4   #5   #6   #7   #8   #9  #10  #11  #12  #13  #14  #15  #16  #17
              |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |
              |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    +-- #17: total ms spent flushing
              |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    +------- #16: total flush requests completed
              |    |    |    |    |    |    |    |    |    |    |    |    |    |    +------------ #15: total ms spent doing discards
              |    |    |    |    |    |    |    |    |    |    |    |    |    +----------------- #14: total sectors discarded
              |    |    |    |    |    |    |    |    |    |    |    |    +---------------------- #13: merged discards
              |    |    |    |    |    |    |    |    |    |    |    +--------------------------- #12: total successful discards
              |    |    |    |    |    |    |    |    |    |    +-------------------------------- #11: weighted total count (avgqu-sz)
              |    |    |    |    |    |    |    |    |    +------------------------------------- #10: total ms doing I/O (%util)
              |    |    |    |    |    |    |    |    +------------------------------------------ #9:  in-progress counter
              |    |    |    |    |    |    |    +----------------------------------------------- #8:  total ms spent writing (w_await)
              |    |    |    |    |    |    +---------------------------------------------------- #7:  total sectors written (wkB/s)
              |    |    |    |    |    +--------------------------------------------------------- #6:  total write merges (wrqm/s)
              |    |    |    |    +-------------------------------------------------------------- #5:  total writes completed (w/s)
              |    |    |    +------------------------------------------------------------------- #4:  total ms spent reading (r_await)
              |    |    +------------------------------------------------------------------------ #3:  total sectors read (rkB/s)
              |    +----------------------------------------------------------------------------- #2:  total read merges (rrqm/s)
              +---------------------------------------------------------------------------------- #1:  total reads completed (r/s)
```

**Notes on fields:**
- All fields are cumulative, monotonic counters, except field #9 which resets to zero as I/Os complete.
- Fields #12-#15 (discard statistics) were added in Linux 4.18.
- Fields #16-#17 (flush statistics) were added in Linux 5.5. Flush requests are not tracked for partitions — only whole disks. The block layer combines flush requests and executes at most one at a time.
- Since kernel 4.19, request times are measured with nanosecond precision and truncated to milliseconds in this interface.
- Since kernel 5.0, field #10 (`io_ticks`) uses a sampling-based approach rather than precise calculation (see the %util Known Issues section below).


## Output Format by RHEL Version

The column order differs between RHEL versions:

| RHEL Version | Columns |
|---|---|
| RHEL 9/10 | `r/s w/s rkB/s wkB/s rrqm/s wrqm/s %rrqm %wrqm r_await w_await aqu-sz rareq-sz wareq-sz d/s dkB/s drqm/s %drqm d_await dareq-sz f/s f_await svctm %util` |
| RHEL 8 | `r/s w/s rkB/s wkB/s rrqm/s wrqm/s %rrqm %wrqm r_await w_await aqu-sz rareq-sz wareq-sz svctm %util` |
| RHEL 7 | `rrqm/s wrqm/s r/s w/s rkB/s wkB/s avgrq-sz avgqu-sz await r_await w_await svctm %util` |
| RHEL 5/6 | `rrqm/s wrqm/s r/s w/s rsec/s wsec/s avgrq-sz avgqu-sz await svctm %util` |

### Notable Changes in RHEL 9/10

- Added discard statistics: `d/s`, `dkB/s`, `drqm/s`, `%drqm`, `d_await`, `dareq-sz`
- Added flush statistics: `f/s`, `f_await`
- The combined `await` field is back (in addition to separate `r_await` and `w_await`)

### Notable Changes in RHEL 8

- Columns appear in a different order
- Added `%rrqm`, `%wrqm`, `rareq-sz`, `wareq-sz`
- Renamed `avgqu-sz` to `aqu-sz`
- Dropped the combined `await` column (separate `r_await` and `w_await` only)
- Default data unit is kB/s instead of sectors
- Average request size is now in kB instead of sectors


## Field Definitions

### rrqm/s - Read Request Merges Per Second

The number of read requests merged per second that were queued to the I/O scheduler.

- **Formula:** `read-merges / interval-in-seconds`
- **Note:** `rrqm/s + r/s` = total number of reads submitted to the I/O scheduler
- Merges are counted at I/O submit time, while `r/s` is measured at I/O completion
- No or few read merges often indicates random I/O or direct/synchronous I/O, especially if average request size is small (<16kB)

### wrqm/s - Write Request Merges Per Second

The number of write requests merged per second that were queued to the I/O scheduler.

- **Formula:** `write-merges / interval-in-seconds`
- **Note:** `wrqm/s + w/s` = total number of writes submitted to the I/O scheduler
- Same interpretation as `rrqm/s` but for writes

### %rrqm - Percentage of Read Merges (RHEL 8+)

The percentage of read requests merged together before being sent to the device.

- **Formula:** `(read-merges / (read-merges + reads)) * 100`

### %wrqm - Percentage of Write Merges (RHEL 8+)

The percentage of write requests merged together before being sent to the device.

- **Formula:** `(write-merges / (write-merges + writes)) * 100`

### r/s - Reads Per Second

The number of read requests, **after merges**, that were completed by the device per second.

- **Formula:** `reads-completed / interval-in-seconds`
- Only incremented when an I/O actually completes
- **IOPS** can be calculated as `r/s + w/s`

### w/s - Writes Per Second

The number of write requests, **after merges**, that were completed by the device per second.

- **Formula:** `writes-completed / interval-in-seconds`
- Only incremented when an I/O actually completes

### rsec/s / rkB/s - Read Throughput

The number of sectors (or kilobytes with `-k`) read from the device per second.

- **Formula:** `(read-sectors / interval-in-seconds) / conversion-factor`
- Only updated when I/O actually completes
- **Throughput** = `rsec/s + wsec/s` (or `rkB/s + wkB/s`)
- KiB output is preferred over MiB to avoid hiding small I/O rates behind `0.00`

### wsec/s / wkB/s - Write Throughput

The number of sectors (or kilobytes with `-k`) written to the device per second.

- **Formula:** `(written-sectors / interval-in-seconds) / conversion-factor`
- Same notes as read throughput

### avgrq-sz - Average Request Size (RHEL 5-7)

The average size, in sectors, of the requests completed by the device.

- **Formula:** `(read-sectors + written-sectors) / (total-reads + total-writes)`
- Always displayed in sectors; `-k` and `-m` do not affect this column
- Split into `rareq-sz` and `wareq-sz` in RHEL 8

### rareq-sz - Average Read Request Size (RHEL 8+)

The average size, in kilobytes (KiB), of read requests issued to the device.

- **Formula:** `(sectors-read / reads-completed) / 2`
- Always displayed in KiB

### wareq-sz - Average Write Request Size (RHEL 8+)

The average size, in kilobytes (KiB), of write requests issued to the device.

- **Formula:** `(sectors-written / writes-completed) / 2`
- Always displayed in KiB

### avgqu-sz / aqu-sz - Average Queue Length

The average queue length of requests issued to the device.

- **Formula:** `(weighted-time-in-queue / interval-in-seconds) / 1000`
- Includes I/O in the scheduler's sort queue AND in-progress I/O within storage
- Updated at I/O start, during progress, and at completion

**Maximum value:** The nominal maximum is `(nr_requests * 2) + max-lun-queue-depth`. For example, with a default `nr_requests` of 128 and a fibre channel lun queue depth of 32, the maximum would be 288.

**Interpretation:**
- Queue depth increasing with steady await time = high I/O submission rate (normal load)
- Queue depth increasing with climbing await time = storage latency issue

### await - Average I/O Wait Time (RHEL 5-7)

The average time (in milliseconds) for I/O requests to be served, including time in the I/O scheduler queue and time in storage.

- **Formula:** `(time-spent-reads + time-spent-writes) / (reads + writes)`
- Combined read+write await time
- Not output in RHEL 8 (replaced by separate `r_await` and `w_await`)

**What the await code path covers:**

The latency is measured from the front of the I/O scheduler until I/O completion time. This full code path includes:
1. I/O scheduler queue time (sort queue, plug/unplug events for merging)
2. HBA driver time
3. Transport (bus) time
4. Switch routing time (in the case of FC or FCoE)
5. Storage controller queueing and processing time
6. Actual storage device latency

For virtual machines, the path also includes hypervisor scheduler time and possibly multipathing and/or filesystem layers on the hypervisor host.

**The two queues within the code path:**

1. **Scheduler queue** — depth controlled by `nr_requests` (applied separately to reads and writes). If a process attempts to schedule an I/O when the scheduler queue is full and that I/O cannot be merged, the thread blocks.
2. **LUN/hardware queue** — within the storage controller (e.g., backplane RAID controllers like HP Smart Array, or SAN controllers like EMC Symmetrix). The `queue_depth` parameter controls the maximum number of I/O submitted to storage at any given time.

**Key points:**
- Measured per-I/O from scheduler submission until I/O completion
- Tends to spike once `avgqu-sz` nears or exceeds the device's lun queue depth
- When queue depth is low, await approximates actual storage latency
- Await is directly measured for each I/O and does NOT take into account parallel/simultaneous I/O (unlike `svctm`)

**Interpreting high await using `avgqu-sz`:**

- **`avgqu-sz` much less than `queue_depth`:** Little time is spent in the scheduler queue. The scheduler continues passing I/O to the driver while outstanding I/O remains under the limit. In this case, await time is dominated by storage servicing time alone — high await here points to a storage-side issue.
- **`avgqu-sz` near or exceeding `queue_depth`:** Some I/O is being retained in the scheduler sort queue until in-progress I/O drops back under the limit. It becomes difficult to determine if await time is due to scheduler queueing or storage service time. You can test by keeping simultaneous I/O under the queue_depth, or by increasing the queue_depth limit.
- **Sustained `avgqu-sz` above `queue_depth`:** The system is generating more I/O than storage can service in a sustained manner. This will impact perceived system performance.

**Common causes of increased await:**
1. Increased I/O load from applications
2. Storage latency issues (often from virtualization or shared storage infrastructure)
3. Accumulated dirty pages in large-memory configurations — e.g., a 64GB+ system with 40% dirty pages trying to flush 20+ GB to a disk capable of 100 MB/s can cause spikey write loads and sustained high await
4. Contention on shared storage (FC SAN) or shared hypervisor resources

### r_await - Read Await Time (RHEL 7+)

The average time (in milliseconds) for read requests to be served.

- **Formula:** `time-spent-performing-reads / reads-performed`
- Helps identify if latency issues are read-specific

### w_await - Write Await Time (RHEL 7+)

The average time (in milliseconds) for write requests to be served.

- **Formula:** `time-spent-performing-writes / writes-performed`
- Helps identify if latency issues are write-specific

### svctm - Service Time (Deprecated/Unreliable)

The effective average service time in milliseconds for completed I/O.

- **Formula:** `(%util * 1000ms) / (r/s + w/s)`
- This is a **pure calculation**, NOT a direct measurement
- It accounts for parallel I/O (unlike await)
- Has no relationship to published disk service time specifications
- The `svctm` column has been removed from the latest upstream sysstat package (since 2018)
- `await` values are typically much more useful to focus on

**Example showing the difference:**
- 1 I/O outstanding for 10ms: `await = 10ms`, `svctm = 10ms`
- 100 parallel I/O all completing in 10ms: `await = 10ms`, `svctm = 0.1ms`

**Why svctm is unreliable:** It yields an "effective I/O rate" for storage — not a representation of actual disk latency. It is unrelated to any representation of per-I/O service time and its use for any given specific purpose is suspect.

### %util - Device Utilization (Busy Time)

Percentage of elapsed wall clock time during which I/O requests were issued to the device.

- **Formula:** `(clock-time-with-io-present / interval) / 10`
- Simply measures what percentage of the sample interval had **any** I/O present

#### Does 100% `%util` Mean Saturation?

**Generally NO.** `%util` is just a measure of device busy time. It means at least one I/O was always outstanding during the sample, but says nothing about how much capacity the device has left.

**Example:** A 50+ disk RAID6 logical device showing `%util=100%` with `avgqu-sz=1` means only 1 of 50+ physical disks is working at any time -- the device is far from saturated despite showing 100% utilization. The same applies to SSDs and NVMe devices with multiple internal channels.

The interpretation of how much capacity is being used depends on the I/O load profile, storage type, and configuration -- information often unknown from the host's perspective.

### d/s - Discards Per Second (RHEL 9+)

The number of discard requests (after merges) completed per second for the device.

- **Formula:** `discards-completed / interval-in-seconds`
- Discard operations (TRIM/UNMAP) inform the storage device that blocks of data are no longer in use
- Common with SSDs, thin-provisioned storage, and filesystems that issue TRIM

### dkB/s - Discard Throughput (RHEL 9+)

The amount of data discarded for the device per second, in kilobytes.

### drqm/s - Discard Request Merges Per Second (RHEL 9+)

The number of discard requests merged per second that were queued to the device.

### %drqm - Percentage of Discard Merges (RHEL 9+)

The percentage of discard requests merged together before being sent to the device.

### d_await - Discard Await Time (RHEL 9+)

The average time (in milliseconds) for discard requests issued to the device to be served, including queue time and service time.

### dareq-sz - Average Discard Request Size (RHEL 9+)

The average size (in kibibytes) of the discard requests that were issued to the device.

### f/s - Flush Requests Per Second (RHEL 9+)

The number of flush requests completed per second for the device.

- Flush requests are not tracked for partitions — only whole disks
- Before being merged, flush operations are counted as writes
- The block layer combines flush requests and executes at most one at a time

### f_await - Flush Await Time (RHEL 9+)

The average time (in milliseconds) for flush requests issued to the device to be served.

- Because the block layer combines and serializes flush requests, flush operations can appear to take twice as long: wait for the current flush to finish, then execute the new one.


## Calculating IOPS

For an individual device:

```
Total IOPS = r/s + w/s
```

Without the `-x` option, this is shown as the `tps` (transactions per second) column.

**Note:** `rsec/s` and `wsec/s` are **throughput** (data per second), NOT IOPS. Each I/O command reads or writes a chunk of data, so these are different measurements.

### System-wide IOPS (RHEL 8+)

Use the `-g` option to group devices:

```bash
iostat -g System /dev/sd*[a-z] | egrep "Device|System"
```

This sums all matching devices into a single "System" group in the output.


## Identifying Stalled Disks

A stalled disk is one of the most critical patterns to identify. The signature is:

- `r/s = 0` and `w/s = 0` (no I/O completing)
- `%util` near 100% (device has outstanding I/O)
- `avgqu-sz` non-zero (I/O is queued)
- When the stall resolves, `await` jumps to a very large value

### Partial Stall Pattern

```
sda   0.00  0.00  1.00   0.00   2.00   0.00  4.00  1.00  543.00   ...  100.00  <- slow I/O
sda   0.00  0.00  66.00  0.00   132.00 0.00  4.00  1.00  17.23    ...  99.60   <- burst completes
sda   0.00  0.00  0.00   0.00   0.00   0.00  0.00  1.00  0.00     ...  100.00  <- STALL: no I/O completing
sda   0.00  0.00  0.00   0.00   0.00   0.00  0.00  1.00  0.00     ...  100.00  <- still stalled
sda   0.00  0.00  1.00   0.00   2.00   0.00  4.00  1.00  4431.00  ...  100.00  <- stall resolves, huge await
sda   0.00  0.00  306.00 0.00   612.00 0.00  4.00  0.86  2.82     ...  86.20   <- back to normal
```

**Key indicators:**
- `svctm` approaching sample time (e.g., 999ms in a 1s sample) during partial stalls
- Large `r_await` or `w_await` when stall resolves reflects time I/O was outstanding
- Pattern is often cyclical and repeatable

### Full Hardware Stall Pattern

```
sdf  ...  0.00  130.00  0.00  52944.00  ...  151.84  547.58   ...  100.00   <- I/O submitted
sdf  ...  0.00  0.00    0.00  0.00      ...  163.00  0.00     ...  100.00   <- FULL STALL
sdf  ...  0.00  0.00    0.00  0.00      ...  163.00  0.00     ...  100.00   <- still stalled
sdf  ...  0.00  0.00    0.00  0.00      ...  163.00  0.00     ...  100.00   <- still stalled
sdf  ...  0.00  70.00   0.00  32300.00  ...  161.52  6893.67  ...  100.10   <- resolves, huge await
```

**Characteristics:**
- `%util = 100%` indicating I/O was outstanding the entire sample
- Zero I/O completions for multiple consecutive samples
- Large `avgqu-sz` remaining constant (I/O stuck in queue)
- When hardware becomes "unstuck," await reflects the total time I/O was stalled (e.g., 4-7 seconds)

**Common causes:**
- Firmware bugs in storage controllers
- RAID rebuild with priority configured over host I/O
- Shared storage contention


## Await Discrepancies in Stacked Device-Mapper Environments (LVM over dm-multipath)

A common source of confusion is observing significantly higher `await` times on an LVM logical volume compared to the underlying dm-multipath device and individual paths. This is expected behavior, not a bug.

### The Symptom

In a typical storage stack like:

```
Block device (/dev/sda) --> dm-multipath (mpatha) --> LVM PV --> LVM logical volume
```

You may see output like:

```
Device:    ...  r/s     w/s     avgrq-sz  avgqu-sz  await   ...
dm-2       ...  569.00  424.00  80.56     3.09      3.11    ...   <- multipath
dm-3       ...  569.00  1380.00 41.05     34.22     17.56   ...   <- LVM (5x higher await!)
```

### Root Cause

Starting in RHEL 6, device-mapper-multipath uses "request-based device-mapper" where I/O requests are merged at the dm-multipath layer before being sent to individual paths. This differs from RHEL 5 and earlier where BIOs were merged by individual paths after allocation.

The consequences:

1. **I/O merging inflates LVM await:** The `await` is calculated as `(total time to complete all I/O) / (total number of completed I/O)`. At the LVM layer, many small I/Os are counted individually. At the dm-multipath layer, those same I/Os have been merged into fewer, larger requests. The per-I/O average time is naturally lower when dividing by fewer (merged) I/Os.

2. **Scheduler queue time:** I/Os are held in the scheduler queue at the LVM layer for additional time to see if subsequent I/Os can be merged before sending them down. This extra wait improves throughput but inflates `await`.

3. **Write caching effects:** VFS/VM layer write caching can affect await times of the LVM device differently than the multipath layer below it.

4. **Multiple LV contention:** When the same LVM PV is used for multiple LVs/filesystems, multiple workloads queue to the same device queue, increasing wait times.

### How to Confirm This Is the Cause

Compare `avgrq-sz` and `avgqu-sz` between layers:
- LVM layer: many requests (high `avgqu-sz`) with small size (low `avgrq-sz`)
- Multipath layer: few requests (low `avgqu-sz`) with large size (high `avgrq-sz`)
- High `wrqm/s` at the multipath layer confirms merging is happening

### Validation with Direct I/O

When using direct I/O (bypassing caching and merging), the await discrepancy disappears:

```bash
dd if=/dev/zero of=/dev/vgtest/lvtest bs=1M oflag=direct
```

With `oflag=direct`, the LVM and multipath layers show virtually identical await times because no merging or caching occurs. However, throughput is typically lower (e.g., 111 MB/s with direct I/O vs. 242 MB/s with caching/merging).

### Mitigation

- **If the higher LVM await causes application issues:** Dedicate a PV to each LV to prevent contention between multiple workloads on the same queue.
- **If you need uniform await for monitoring:** Use direct I/O from your application, but test thoroughly as throughput may be impacted.
- **For accurate storage latency measurement:** Look at the multipath or physical device layer await times rather than the LVM layer, as those more closely reflect actual storage response time.
- **For deeper analysis:** Capture `blktrace` across all layers simultaneously:

```bash
mount -t debugfs none /sys/kernel/debug
blktrace -w 120 /dev/dm-3 /dev/dm-2 /dev/sda /dev/sdb
```


## Diagnosing High Await with blktrace

When you need to understand where time is being spent within the I/O stack, `blktrace` provides timestamps at key points in the lower I/O path.

### blktrace Event Points

```
Maj,min  cpu  seq#  Time         PID    Act  RWBS  Sector + Len  Description
 8,0     18   3     0.000001721  30916  Q    WS    173237576 + 8 [mysqld]   ; __make_request()
 8,0     18   6     0.000007879  30916  I    WS    173237576 + 8 [mysqld]   ; insert into sorted/merged queue
 8,0     18   8     0.000031626  30916  D    WS    173237576 + 8 [mysqld]   ; dispatch to driver
 8,0     6    1     0.000115301  0      C    WS    173237576 + 8 [0]        ; io done
```

### Await Time Breakdown (Q->C)

The await time measures **Q->C time** (from `__make_request()` to I/O done). It has 3 main sub-components:

| Segment | Meaning |
|---|---|
| **Q->I** | Time from queue insertion to being placed in the sorted/merged queue |
| **I->D** | Time in scheduler awaiting dispatch (includes plug/unplug events for merging) |
| **D->C** | Driver time + adapter time + transport time + storage service time (and back) |

Q->I and I->D are usually lumped together as "scheduler" related time. For most diagnostic purposes, the ratio of **D->C time to total Q->C time** is the key metric:

- **High D->C ratio (e.g., 74% of total Q->C):** Most time is spent in driver/controller/transport/storage. Examine switch counters, storage controller maintenance interfaces, or underlying SAN infrastructure.
- **Low D->C ratio:** Most time is in the scheduler queue, indicating the device is overwhelmed with I/O requests — look at reducing I/O load or tuning `nr_requests`/`queue_depth`.

### Other Diagnostic Tools

- **blktrace** — microscopic view of I/O flow; adds overhead so cannot directly compare numbers to non-blktrace runs. Run blktrace and iostat simultaneously to correlate.
- **iotop** (`iotop -tbo -d 1`) — shows which processes are contributing to the I/O load
- **ps/top** — check for processes in D state (uninterruptible sleep, usually waiting for I/O)

### Quick Decision Tree for High Await

```
High await observed
    |
    +-- Is avgqu-sz much less than queue_depth?
    |       |
    |       YES --> Storage-side issue (latency in controller/transport/disk)
    |
    +-- Is avgqu-sz near or exceeding queue_depth?
    |       |
    |       YES --> Ambiguous: could be scheduler queueing OR storage latency
    |               Try: keep simultaneous I/O under queue_depth, or increase queue_depth
    |
    +-- Is avgqu-sz sustained well above queue_depth?
            |
            YES --> System is generating more I/O than storage can handle
                    Consider: dirty page tuning, I/O load reduction, faster storage
```

### Dirty Page Tuning for Large Memory Systems

One common cause of spikey high-await writes: accumulated dirty pages in large physical memory configurations. For example, a 64GB+ system with 40% dirty ratio can accumulate 20+ GB of dirty data. When the kernel decides to flush, it can overwhelm a disk capable of only 100 MB/s write rates — taking 200+ seconds to drain. Tuning `vm.dirty_ratio` and `vm.dirty_background_ratio` to reduce dirty page accumulation prevents these spikes.


## Best Practices

1. **Always skip the first sample** using `-y` flag or ignoring the first output line
2. **Use at least a 5-second interval** for meaningful averages: `iostat -x 5`
3. **Prefer KiB output** over MiB to avoid hiding small I/O rates behind `0.00`
4. **Focus on `await`/`r_await`/`w_await`** for understanding storage latency
5. **Don't trust `%util` on modern kernels (5.0+)** — it uses inaccurate jiffy-based sampling. Use `aqu-sz` as the primary load indicator instead.
6. **Correlate `avgqu-sz` with await** to distinguish between high load vs. storage latency
7. **Look at the pattern over time**, not just individual samples
8. **Monitor discard and flush latency** on SSD/NVMe workloads — high `d_await` or `f_await` can indicate storage firmware issues or journal contention
9. **Baseline your system** — compare against what your system looks like when functioning normally. The importance of getting a baseline cannot be overstressed.


## Quick Reference

| What You Want to Know | How to Calculate |
|---|---|
| Total IOPS | `r/s + w/s` (add `d/s` for discards) |
| Total throughput | `rkB/s + wkB/s` |
| Storage latency | `r_await` and `w_await` (when queue depth is low) |
| Device busy time | `%util` (unreliable on kernel 5.0+; prefer `aqu-sz`) |
| Queue depth | `avgqu-sz` / `aqu-sz` |
| Total reads submitted to scheduler | `rrqm/s + r/s` |
| Total writes submitted to scheduler | `wrqm/s + w/s` |
| Is disk stalled? | `r/s=0, w/s=0` but `%util~100%` and `avgqu-sz>0` |
| Discard activity | `d/s`, `dkB/s`, `d_await` (RHEL 9+) |
| Flush latency | `f_await` (RHEL 9+) |
