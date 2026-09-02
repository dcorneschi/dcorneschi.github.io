# HP-UX Swap and Pseudo-Swap Management

Managing swap on HP-UX — the pseudo-swap concept that lets processes reserve more swap than is physically configured, the swap types (device, filesystem), enabling and prioritizing swap with `swapon`, reporting with `swapinfo`, `/etc/fstab` entries, and guidelines for choosing swap areas. Covers HP-UX 11i v1–v3.

## Pseudo-Swap

The crucial idea to grasp first is the difference between **reservation** and **use**. When a process starts (or calls `malloc`, `fork`, etc.), HP-UX *reserves* enough swap to guarantee it could page that memory out if it ever had to — even if it never actually does. Historically this meant you needed device swap at least as large as the memory you wanted to commit, which wasted disk on systems with lots of RAM. Pages are only *used* (written to a swap area) when the memory manager actually needs to reclaim physical memory under pressure.

To use disk and memory more efficiently, HP-UX relaxes swap **reservation** limits with **pseudo-swap**: it lets processes reserve more swap than has actually been configured, using a portion of physical memory as additional reservation space.

- HP-UX allocates up to **75% of physical memory** as pseudo-swap.
- Example: 4 GB RAM + 1 GB device swap → with pseudo-swap **disabled**, only ~1 GB of processes can run; **enabled**, processes can reserve up to ~4 GB (1 GB + 75% of 4 GB).
- Example: 8 GB RAM + 8 GB device swap → total reservable swap = **14 GB** (8 GB device + 6 GB pseudo-swap).

Key points:

- Pseudo-swap is **enabled by default** on all current HP-UX versions.
- On **11i v1/v2** it can be disabled by setting the `swapmem_on` kernel parameter to `0`. On **11i v3** it's always enabled and `swapmem_on` no longer exists.
- Pseudo-swap is used only to **reserve** swap for new processes — it is **not** used for demand paging. So you don't need device swap as large as (or larger than) physical memory.

Why this matters in practice: on a machine with a large amount of RAM, pseudo-swap lets you configure a modest amount of device swap and still run a memory footprint far larger than that device swap alone would allow, because up to 75% of RAM counts toward the reservation ceiling. The remaining ~25% of RAM is held back for the kernel and for memory that genuinely cannot be paged. The tradeoff is that if physical memory *does* fill and the system needs to page out, it can only write to the real (device/filesystem) swap areas — so undersizing device swap turns a memory shortage into hard failures (`ENOMEM`, processes killed) rather than graceful, if slow, paging. Size device swap for the workload you expect to actually page, not just to satisfy reservation.

## Swap Types

| Type | What it is | Notes |
|------|-----------|-------|
| **Device swap** | A whole disk or logical volume dedicated to swap | Fastest; preferred |
| **Filesystem swap** | Space borrowed from a mounted filesystem | Slower; use only when device swap can't be added |
| **Pseudo-swap** | Physical memory used for swap *reservation* | Reservation only, not paging |

Device swap is fastest because the kernel writes to raw block ranges with no filesystem layer in between — no directory lookups, no allocation maps, no journaling overhead. A dedicated logical volume also lets the swap area be *contiguous*, which keeps disk-head movement predictable. Filesystem swap, by contrast, competes with normal file I/O on the same filesystem and is constrained by the `-l` limit you give it; the kernel effectively borrows space through the filesystem, which adds overhead on every page in/out. Treat filesystem swap as a temporary measure — for example to survive until you can add a disk — rather than a permanent design.

There's also a distinction within device swap: a **primary** swap area (usually `/dev/vg00/lvol2`, configured at install and recorded via `lvlnboot`) is activated very early in boot, while **secondary** swap areas are enabled later by the startup script from `/etc/fstab`. The primary area is what the system falls back on before the rest of the configuration is in place, so it should live on a reliable local disk.

## Creating and Enabling Swap (swapon)

```bash
# Device swap on a logical volume
lvcreate -L 1024 -n swapvol -C y /dev/vg01     # -C y = contiguous LV (required for swap)
swapon /dev/vg01/swapvol

# Device swap on a whole disk
swapon /dev/disk/disk2

# Device swap in space reserved at the end of a filesystem disk
newfs -R 1024 /dev/rdisk/disk3                 # reserve 1024 MB at end of disk for swap
swapon -e /dev/disk/disk3                       # -e = use the reserved end space

# Filesystem swap
swapon -p 4 -l 1024m /data                      # priority 4, cap at 1024 MB from /data
swapon -l 100m /data2                           # cap at 100 MB
```

Key `swapon` options:

| Option | Meaning |
|--------|---------|
| `-p <0-10>` | Priority (0 = highest, default 1) |
| `-l <size>` | Limit — max space `vhand` may take from a filesystem for swap |
| `-e` | Use the space reserved at the end of a disk (via `newfs -R`) |
| `-a` | Enable all swap listed in `/etc/fstab` (used by the startup script) |

The `-C y` (contiguous) allocation policy is not optional for a swap LV — the kernel wants swap space laid out in a single contiguous run so that paging I/O is sequential rather than scattered across the disk. If you try to `swapon` a striped or non-contiguous LV, allocate it with the contiguous policy first. Also make sure the LV is *not* mounted as a filesystem; a device swap LV holds raw paging data, not a filesystem, so there is nothing to `mount`.

`swapon` changes take effect immediately but are **not** persistent by themselves — the area is active only until reboot unless you also add it to `/etc/fstab` (see below). Conversely, an `/etc/fstab` entry alone doesn't activate swap until `swapon -a` runs at boot or you run it by hand.

Check the primary swap logical volume:

```bash
lvlnboot -v                                     # shows the primary swap LV
```

`lvlnboot` maintains the boot-time pointers in the vg00 BDRA (boot data reserved area), including which LV is the primary swap and which is the dump device. If you relocate or resize primary swap you must update these pointers with `lvlnboot -s <lv>` (swap) so early boot finds the right area; otherwise the change may not survive a reboot. See [HP-UX LVM](articles/hpux-lvm.md) for the volume-group and logical-volume mechanics behind swap LVs.

## Reporting Swap (swapinfo)

```bash
swapinfo            # usage of all swap areas
swapinfo -f         # filesystem swap only
swapinfo -d         # device swap only
swapinfo -tm        # values in MB with a total line
swapinfo -at        # all areas plus a total (a = show all, including reserve/memory rows)
```

Reading `swapinfo` output is a skill in itself. The columns that matter most:

- **Kb / AVAIL** — total space in that area.
- **USED** — space actually written (paged out) to a device/filesystem area, *or* reservation held against the memory (pseudo-swap) row.
- **FREE** — what's left.
- **RESERVE** — reservation charged against this area but not yet used.
- **PCT USED** — the percentage that tends to alarm people. On a healthy system with plenty of RAM, PCT USED against *device* swap should be low or zero even when overall reservation is high, because reservations are landing on pseudo-swap first.

Row **types** you'll see: `dev` (device), `localfs` (filesystem swap), `reserve` (pending reservations not tied to a specific area), and `memory` (the pseudo-swap contribution from RAM). If the `dev` rows start filling, the system is genuinely paging to disk — investigate memory pressure (with `vmstat`) rather than just adding swap. The `total` line summarizes reservation across everything; when *that* approaches the ceiling, new process creation and `malloc` calls start failing.

## Enabling Swap via /etc/fstab

The `/sbin/init.d/swap_start` startup script runs `swapon -a`, which reads `/etc/fstab` and activates all listed swap areas.

```bash
# device            mount  type    options            dump pass
/dev/vg01/datavol   /data  vxfs    defaults           0    2
/dev/vg01/swapvol   .      swap    defaults           0    0
/dev/disk/disk2     .      swap    defaults           0    0
/dev/disk/disk3     .      swap    end                0    0     # use newfs -R reserved space
/dev/vg01/datavol   /data  swapfs  pri=4,lim=400m     0    0     # filesystem swap
```

- `type swap` = device swap; `end` in the options field uses the reserved end-of-disk space.
- `type swapfs` with `pri=`/`lim=` = filesystem swap with a priority and size cap.

## Swap Priority

Every swap area gets a priority **0–10** (set with `swapon -p`, default 1). Lower number = higher priority.

- Paging starts at the **highest-priority** (lowest-number) area — priority `0` is used before priority `1`.
- Areas with the **same** priority are used **round-robin** (interleaved), which spreads I/O and improves throughput.
- Recommendation: give most swap areas the **same** priority (to interleave and limit disk-head movement) — only lower a device's priority if it's significantly slower than the rest.

## Disabling Swap

- **11i v1/v2** — remove the entry from `/etc/fstab` and **reboot** (`shutdown -ry 0`); swap can't be removed on the fly.
- **11i v3** — disable dynamically with `swapoff`, and also remove the `/etc/fstab` entry so it stays off.

```bash
swapoff /dev/vg01/swapvol      # 11i v3 only
```

## Selecting Device Swap Areas

- Two swap areas on **different disks** are better than one large area.
- Configure only **one** swap area per disk.
- Keep device swap areas **similar in size**.
- Prefer **faster** disks.

## Selecting Filesystem Swap Areas

- Avoid filesystem swap **altogether** if you can.
- Avoid **busy** filesystems (like root) and ones **near capacity**.
- Avoid filesystems on disks that already have device swap.
- Set priorities sensibly — faster devices and less-busy filesystems get higher priority.

## Troubleshooting

- **`swapon` fails with "device busy" or "not enough space".** The LV may be mounted as a filesystem, may not be contiguous, or may overlap another swap area. Confirm it isn't in `/etc/fstab` as a `vxfs`/`hfs` mount, and that it was created with `-C y`.
- **`swapinfo` shows device swap filling up.** This is real paging, not just reservation. Correlate with `vmstat 5` — sustained non-zero `po` (page-outs) and a low `free` column mean a memory shortage. Adding swap only delays the problem; the fix is usually more RAM or a smaller working set.
- **Reservation exhausted but disk swap looks empty.** The `total`/`reserve` line has hit the ceiling (device swap + pseudo-swap). Processes fail to fork or `malloc` even though `dev` rows show free space. Add device swap, or on 11i v1/v2 confirm pseudo-swap wasn't disabled (`swapmem_on`).
- **A swap area didn't come back after reboot.** It was enabled at runtime with `swapon` but never added to `/etc/fstab`, so `swapon -a` didn't re-enable it. Add the `/etc/fstab` entry to make it persistent.

```bash
vmstat 5                       # watch po/pi (page out/in) and free memory
swapinfo -tm                   # confirm totals after a change
```

## Command Reference

| Task | Command |
|------|---------|
| Create a swap LV | `lvcreate -L <mb> -n swapvol -C y /dev/vg01` |
| Enable LV / disk swap | `swapon /dev/vg01/swapvol` / `swapon /dev/disk/disk2` |
| Reserve + enable end-of-disk swap | `newfs -R <mb> <rdisk>` + `swapon -e <dsk>` |
| Filesystem swap | `swapon -p <pri> -l <size> /mount` |
| Enable all from fstab | `swapon -a` |
| Show primary swap LV | `lvlnboot -v` |
| Report swap | `swapinfo [-f\|-d\|-tm]` |
| Disable swap (11i v3) | `swapoff <device>` |
| Disable pseudo-swap (v1/v2) | set `swapmem_on=0` (kernel param) |
| Watch paging activity | `vmstat 5` (`po`/`pi` columns) |

## Related Articles

- [HP-UX LVM](articles/hpux-lvm.md) — creating and managing the logical volumes used for device swap
- [HP-UX Filesystem Management](articles/hpux-filesystem-management.md) — filesystem swap and the `/etc/fstab` entries that activate swap at boot
- [HP-UX Kernel Configuration](articles/hpux-kernel-configuration.md) — the `swapmem_on` and related tunables that govern pseudo-swap
- [HP-UX Performance Monitoring](articles/hpux-performance-monitoring.md) — using `vmstat` and friends to tell reservation pressure from real paging
- [HP-UX Crash Dump Analysis](articles/hpux-crash-dump-analysis.md) — the dump device shares primary-swap concepts and `lvlnboot` pointers
