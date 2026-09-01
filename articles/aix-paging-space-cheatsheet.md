# AIX Paging Space Cheatsheet

Command reference for managing paging (swap) space on IBM AIX — listing paging space and activity (`lsps`, `swap -l`), creating and resizing paging spaces (`mkps`, `chps`), activating/deactivating (`swapon`/`swapoff`, `swap -a`/`-d`, `chps -a`), removing paging spaces (`rmps`), and finding the top paging-space consumers with `svmon`.

> Most of these commands require `root`. The default paging space is `hd6` in `rootvg` and should not be removed. Deactivating or removing paging space while it is in use can hang or crash the system — reduce demand first and confirm with `lsps -a`.

## Listing Paging Space

```sh
# List size, %used, type, and paging activity per space
lsps -a

# Summary of all paging space (total size and % used)
lsps -s

# Display swap information (BSD-style, similar to lsps -a)
swap -l

# Total physical RAM in the system
lsattr -El sys0 -a realmem
```

## Creating and Resizing

```sh
# Create a paging space: 80 LPs in rootvg on hdisk2,
# add to /etc/swapspaces (-a) and activate now (-n)
mkps -a -n -s 80 rootvg hdisk2

# Extend an existing paging space by 8 physical partitions
chps -s 8 hd6

# Reduce a paging space by 5 logical partitions
chps -d 5 paging01
```

| Option | Command | Purpose |
|--------|---------|---------|
| `-s <n>` | `mkps` | Size in logical partitions |
| `-a` | `mkps` | Add the space to `/etc/swapspaces` (activate at boot) |
| `-n` | `mkps` | Activate the space immediately |
| `-s <n>` | `chps` | Grow by N partitions |
| `-d <n>` | `chps` | Shrink by N partitions |

## Activating and Deactivating

```sh
# Activate a paging space at boot via chps
chps -a y hd6

# Do not activate at boot
chps -a n paging00

# Activate now (System V style)
swapon paging00

# Activate all configured paging spaces
swapon -a

# Deactivate now
swapoff paging00

# Activate a paging space (BSD style)
swap -a /dev/paging01

# Deactivate a paging space (BSD style)
swap -d /dev/paging01
```

> `chps -a y|n` controls whether a space is activated automatically at boot (via `/etc/swapspaces`), while `swapon`/`swapoff` change the current runtime state.

## Removing Paging Space

A paging space must be deactivated before it can be removed:

```sh
swapoff /dev/paging00
rmps paging00
```

## Finding Top Paging-Space Consumers

```sh
# Top 10 processes using the most paging space
svmon -Pg -t 1 | grep Pid ; svmon -Pg -t 10 | grep "N"
```

## Sizing Guidance

Paging-space sizing depends heavily on RAM and workload; modern systems with large memory rarely need the old "2× RAM" rule. A few practical points:

- **Spread paging across multiple physical disks.** AIX uses a round-robin algorithm across active paging spaces, so several equally sized spaces on separate disks give better throughput than one large space.
- **Keep paging spaces close to the same size.** Uneven sizes defeat the round-robin balancing.
- **Avoid putting a second paging space on the same disk as `hd6`** — it adds no I/O parallelism.
- **Watch `%Used` with `lsps -a`.** Consistently high usage means either more RAM or more paging space is needed; heavy sustained paging is a memory-shortage symptom, not something extra paging space fixes on its own.
- **`hd6` should stay in `rootvg`** and generally be the largest/primary space.

## Important Files

| File | Purpose |
|------|---------|
| `/etc/swapspaces` | Lists paging spaces activated at boot (managed by `mkps`/`chps -a`) |

## Quick Reference

| Task | Command |
|------|---------|
| List paging spaces + activity | `lsps -a` |
| Summary of paging space | `lsps -s` |
| Total physical RAM | `lsattr -El sys0 -a realmem` |
| Create paging space | `mkps -a -n -s 80 rootvg hdisk2` |
| Extend by 8 PPs | `chps -s 8 hd6` |
| Reduce by 5 LPs | `chps -d 5 paging01` |
| Activate at boot | `chps -a y hd6` |
| Activate now | `swapon paging00` |
| Activate all | `swapon -a` |
| Deactivate now | `swapoff paging00` |
| Remove paging space | `rmps paging00` |
| Top paging consumers | `svmon -Pg -t 10` |

## Related

- [AIX Performance Monitoring Cheatsheet](articles/aix-performance-monitoring-cheatsheet.md) — `svmon`/`vmstat` for spotting paging pressure and top paging consumers.
- [AIX Storage Provisioning Tasks](articles/aix-storage-provisioning-tasks.md) — extending paging space when adding disks to `rootvg`.
