# free & top Memory Cheatsheet

Quick reference for checking memory usage on Linux using `free`, `top`, and related tools.

---

## free

`free` displays the total amount of free and used physical and swap memory in the system.

### Basic Usage

```bash
# Human-readable output (KB, MB, GB)
free -h

# Output in megabytes
free -m

# Output in gigabytes
free -g

# Output in bytes
free -b

# Output in kibibytes (default)
free -k

# SI units (1000-based instead of 1024-based)
free --kilo             # Kilobytes (1000 bytes)
free --mega             # Megabytes (1000 KB)
free --giga             # Gigabytes (1000 MB)
free --tera             # Terabytes (1000 GB)

# IEC units (1024-based, same as -k/-m/-g)
free --kibi             # Kibibytes (1024 bytes) — same as -k
free --mebi             # Mebibytes (1024 KiB) — same as -m
free --gibi             # Gibibytes (1024 MiB) — same as -g
free --tebi             # Tebibytes (1024 GiB)
free --peti             # Pebibytes (1024 TiB)

# Continuous output every 2 seconds
free -h -s 2

# Repeat 5 times, every 3 seconds
free -h -s 3 -c 5

# Wide output (separates buffers and cache into two columns)
free -hw

# Show low and high memory stats (32-bit systems)
free -l

# Show totals line (RAM + Swap combined)
free -t
```

### Understanding the Output

```
              total        used        free      shared  buff/cache   available
Mem:          15Gi        5.2Gi       1.8Gi       412Mi       8.4Gi       9.5Gi
Swap:         4.0Gi          0B       4.0Gi
```

| Column | Meaning |
|--------|---------|
| `total` | Total installed RAM |
| `used` | Memory actively used by processes (`total - free - buffers - cache`) |
| `free` | Completely unused memory |
| `shared` | Memory used by `tmpfs` and shared segments |
| `buff/cache` | Memory used for kernel buffers and page cache |
| `available` | Estimated memory available for starting new applications (without swapping) |

> **Key insight:** `available` is the number you should look at — not `free`. Linux aggressively uses free RAM for caching, so `free` appears low even when the system is healthy. `available` accounts for reclaimable cache.

### Wide Output (-w)

```bash
free -wh
```

```
              total        used        free      shared     buffers       cache   available
Mem:           15Gi       8.2Gi       2.1Gi       512Mi       256Mi       4.5Gi       6.5Gi
Swap:         2.0Gi          0B       2.0Gi
```

Separates `buff/cache` into two columns:
- `buffers` — block device I/O buffer cache
- `cache` — page cache and slabs

### Legacy Output (RHEL 6 / older systems)

On RHEL 6 and older kernels, `free` had no `available` column. Instead it showed a `-/+ buffers/cache` row:

```
             total       used       free     shared    buffers     cached
Mem:           94G        44G        49G       161M       993M       1.3G
-/+ buffers/cache:        42G        52G
Swap:          15G         0B        15G
```

The `-/+ buffers/cache` line showed:
- `used` = actual usage minus buffers and cache
- `free` = actual available memory (equivalent to modern `available`)

On RHEL 7+ / modern kernels, this was replaced by the `available` column which is computed more accurately using `/proc/meminfo`'s `MemAvailable` field.

### Why `free` Looks Low

Linux uses idle memory for disk cache (page cache). This is a feature, not a problem:

```
total = used + free + buff/cache
available ≈ free + reclaimable portion of buff/cache
```

If `available` is close to zero, the system is genuinely running low on memory. If only `free` is low but `available` is high, you're fine — the kernel will reclaim cache as needed.

### Useful One-Liners

```bash
# Just show available memory
free -h | awk '/^Mem:/ {print "Available: " $7}'

# Show memory usage percentage
free | awk '/^Mem:/ {printf "Memory used: %.1f%%\n", $3/$2 * 100}'

# Show swap usage percentage
free | awk '/^Swap:/ {if ($2 > 0) printf "Swap used: %.1f%%\n", $3/$2 * 100; else print "No swap"}'

# Monitor memory every second (like watch)
watch -n 1 free -h

# Alert if available memory drops below 500MB
free -m | awk '/^Mem:/ {if ($7 < 500) print "WARNING: Low memory - " $7 "MB available"}'

# Total memory (RAM + Swap combined)
free -ht

# Log memory usage with timestamp
while true; do
    echo "$(date): $(free -h | grep Mem:)" >> memory_log.txt
    sleep 300
done

# CSV format for trend analysis
echo "$(date +%Y-%m-%d\ %H:%M:%S),$(free | awk '/Mem:/ {print $3,$7}')" >> memory_history.csv
```

---

## Interpreting Results

### Healthy System

```
              total        used        free      shared  buff/cache   available
Mem:           16Gi       4.0Gi       8.0Gi       100Mi       4.0Gi      11.5Gi
Swap:         2.0Gi          0B       2.0Gi
```

- High `available` memory
- Low or no swap usage
- `buff/cache` being utilised (good — disk caching is working)

### Memory Under Pressure

```
              total        used        free      shared  buff/cache   available
Mem:           16Gi      14.5Gi       500Mi       100Mi       1.0Gi       1.2Gi
Swap:         2.0Gi       1.5Gi       500Mi
```

- Low `available` memory
- Significant swap usage
- Low `buff/cache` (kernel has already reclaimed it)
- Action needed: identify what's consuming memory, add RAM, or set limits

### Memory Used Efficiently (Normal, No Action Needed)

```
              total        used        free      shared  buff/cache   available
Mem:           16Gi       6.0Gi       1.0Gi       200Mi       9.0Gi      10.0Gi
Swap:         2.0Gi          0B       2.0Gi
```

- Low `free` but high `available` — this is fine
- Large `buff/cache` (this is good — improves I/O performance)
- No swap usage
- "Free memory is wasted memory" — Linux uses idle RAM for caching

### When to Worry

- `available` below 10% of total
- Consistent swap usage combined with active swapping (`si`/`so` in vmstat)
- OOM killer messages in `dmesg`

---

## Dropping Caches

Force the kernel to release cached memory. Useful for benchmarking or testing — not for production use.

```bash
# Sync dirty pages to disk first
sync

# Drop page cache only
echo 1 > /proc/sys/vm/drop_caches

# Drop dentries and inodes
echo 2 > /proc/sys/vm/drop_caches

# Drop page cache, dentries, and inodes (all)
echo 3 > /proc/sys/vm/drop_caches

# Then check
free -h
```

> **Warning:** Dropping caches temporarily hurts performance because subsequent file reads must go to disk instead of cache. The kernel will rebuild the cache over time. Never do this routinely in production — it exists for testing purposes.

---

## Monitoring Scripts

### Multi-level alert script

```bash
#!/bin/bash
while true; do
    AVAILABLE=$(free | awk '/Mem:/ {print $7}')
    TOTAL=$(free | awk '/Mem:/ {print $2}')
    PERCENT=$(awk "BEGIN {printf \"%.0f\", ($AVAILABLE/$TOTAL)*100}")

    echo "$(date): ${PERCENT}% memory available"

    if [ "$PERCENT" -lt 10 ]; then
        echo "CRITICAL: Low memory!"
    elif [ "$PERCENT" -lt 20 ]; then
        echo "WARNING: Memory getting low"
    fi

    sleep 60
done
```

### Memory usage report

```bash
#!/bin/bash
echo "=== Memory Usage Report ==="
echo "Date: $(date)"
echo ""

free -h | awk '
    /Mem:/ {
        printf "Total Memory:     %s\n", $2
        printf "Used Memory:      %s (%.1f%%)\n", $3, ($3/$2)*100
        printf "Available Memory: %s (%.1f%%)\n", $7, ($7/$2)*100
    }
    /Swap:/ {
        printf "\nTotal Swap:  %s\n", $2
        printf "Used Swap:   %s\n", $3
        printf "Free Swap:   %s\n", $4
    }
'
```

### Memory trend (collect samples)

```bash
# Collect available memory every minute for 10 minutes
for i in {1..10}; do
    free -m | awk '/Mem:/ {print $7}' >> samples.txt
    sleep 60
done

# Calculate average
awk '{sum+=$1} END {print "Average available (MB):", sum/NR}' samples.txt
```

---

## /proc/meminfo

For detailed memory breakdown beyond what `free` shows:

```bash
# Full memory info
cat /proc/meminfo

# Specific values
grep -E "MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapTotal|SwapFree" /proc/meminfo

# Dirty pages (data waiting to be written to disk)
grep -i dirty /proc/meminfo

# Slab cache (kernel data structures)
grep -i slab /proc/meminfo

# Hugepages
grep -i huge /proc/meminfo
```

Key fields:

| Field | Meaning |
|-------|---------|
| `MemTotal` | Total usable RAM |
| `MemFree` | Free RAM (not used at all) |
| `MemAvailable` | Available for applications (free + reclaimable) |
| `Buffers` | Memory used for block device I/O buffers |
| `Cached` | Page cache (file data cached in RAM) |
| `SwapCached` | Swap data also present in RAM (avoids re-reading from swap) |
| `Active` | Recently accessed memory (less likely to be reclaimed) |
| `Inactive` | Not recently accessed (candidate for reclaiming) |
| `Dirty` | Data modified in cache but not yet written to disk |
| `Slab` | Kernel slab allocator (dentries, inodes, etc.) |
| `SReclaimable` | Slab memory that can be reclaimed |
| `SUnreclaim` | Slab memory that cannot be reclaimed |
| `CommitLimit` | Total amount the system can allocate (based on overcommit settings) |
| `Committed_AS` | Total memory allocated by processes (may exceed physical RAM) |

---

## top / htop Memory Columns

### top

Press `M` in top to sort by memory usage.

```bash
# Start top sorted by memory
top -o %MEM

# Show specific user
top -u username

# Batch mode (for scripting)
top -b -n 1 | head -20
```

Memory fields in top's header:

```
MiB Mem :  15906.4 total,   1843.2 free,   5324.8 used,   8738.4 buff/cache
MiB Swap:   4096.0 total,   4096.0 free,      0.0 used.   9712.0 avail Mem
```

Per-process memory columns:

| Column | Meaning |
|--------|---------|
| `%MEM` | Percentage of physical RAM used by the process |
| `VIRT` | Total virtual memory (code + data + shared libraries + swap + mapped files) |
| `RES` | Resident set size — physical RAM actually used (non-swapped) |
| `SHR` | Shared memory — portion of RES shared with other processes |

> **Tip:** `RES - SHR` gives a rough estimate of private (unshared) memory for a process. `VIRT` is usually much larger than actual usage and is misleading on its own.

### htop

htop provides the same info with color-coded bars. Memory bar breakdown:

| Color | Meaning |
|-------|---------|
| Green | Used memory (processes) |
| Blue | Buffer pages |
| Orange/Yellow | Cache pages |
| Red | Used swap |

---

## vmstat (Memory Section)

```bash
# Single snapshot
vmstat -s

# Continuous (every 2 seconds)
vmstat 2

# With timestamps
vmstat -t 2
```

Memory columns in `vmstat` output:

| Column | Meaning |
|--------|---------|
| `swpd` | Swap used (KB) |
| `free` | Free memory (KB) |
| `buff` | Buffer memory (KB) |
| `cache` | Cache memory (KB) |
| `si` | Swap in (KB/s) — memory paged in from swap |
| `so` | Swap out (KB/s) — memory paged out to swap |

> **Red flag:** Non-zero `si`/`so` values sustained over time means active swapping, indicating memory pressure.

---

## Checking Memory for a Specific Process

```bash
# RSS and VSZ for a process
ps -p <PID> -o pid,rss,vsz,comm

# Detailed memory map
cat /proc/<PID>/status | grep -i "vm\|rss\|swap"

# Per-process memory breakdown
pmap -x <PID>

# Smaps (most detailed — per-mapping breakdown)
cat /proc/<PID>/smaps_rollup

# All processes sorted by RSS
ps aux --sort=-%mem | head -20

# Same with custom format
ps -eo pid,user,%mem,rss,vsz,comm --sort=-%mem | head -20
```

Key fields in `/proc/<PID>/status`:

| Field | Meaning |
|-------|---------|
| `VmSize` | Total virtual memory |
| `VmRSS` | Resident set size (physical RAM) |
| `VmSwap` | Amount swapped out |
| `VmData` | Private data segments |
| `VmStk` | Stack size |
| `VmLib` | Shared library size |
| `RssAnon` | Anonymous (private) resident memory |
| `RssFile` | File-backed resident memory |
| `RssShmem` | Shared memory resident |

---

## Quick Answers

### How much memory is available right now?

```bash
free -h | awk '/^Mem:/ {print $7}'
```

### Is the system swapping?

```bash
# Check swap usage
free -h | awk '/^Swap:/ {print "Used: " $3 " / Total: " $2}'

# Check if actively swapping (si/so columns)
vmstat 1 5 | tail -4
```

### What's using the most memory?

```bash
ps -eo pid,user,%mem,rss,comm --sort=-%mem | head -10
```

### Total memory used by a service (all its processes)?

```bash
# Example: all nginx processes
ps -C nginx -o rss= | awk '{sum+=$1} END {printf "%.0f MB\n", sum/1024}'

# Example: all java processes
ps -C java -o rss= | awk '{sum+=$1} END {printf "%.0f MB\n", sum/1024}'
```

### Is memory pressure causing OOM kills?

```bash
dmesg | grep -i "oom\|out of memory\|killed process"
journalctl -k | grep -i oom
```

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `free -h` | Human-readable memory summary |
| `free -hw` | Wide format (split buffers and cache) |
| `free -h -s 2` | Refresh every 2 seconds |
| `cat /proc/meminfo` | Detailed kernel memory stats |
| `top -o %MEM` | Top sorted by memory usage |
| `ps -eo pid,%mem,rss,comm --sort=-%mem` | Processes by memory |
| `vmstat 2` | Memory and swap activity |
| `pmap -x <PID>` | Memory map for a process |
| `cat /proc/<PID>/status` | Per-process memory details |
| `cat /proc/<PID>/smaps_rollup` | Summarized memory map |
| `dmesg \| grep -i oom` | Check for OOM kills |
