# HP-UX nPartitions (nPars)

nPartitions are HP-UX's **hardware** partitioning technology on cell-based servers: a complex is divided at the cell/hardware level into electrically isolated partitions, each with its own cells (processors + memory), I/O chassis, firmware, and OS instance. This guide covers the genesis partition, creating and modifying nPars, adding/removing cells, and the reboot/shutdown-for-reconfig operations. For **software** partitioning within an nPar, see [HP-UX Virtual Partitions (vPars)](articles/hpux-vpars.md).

## Concepts

- An **nPartition** is a hardware partition made of one or more **cell boards** (each with processors and memory) plus an **I/O chassis** with a **core I/O card**.
- nPars are electrically isolated — a fault in one doesn't affect another — and each runs its own independent firmware and OS.
- Administered from the **Service Processor (MP/GSP)**, **Partition Manager** (GUI, `/opt/parmgr/bin/parmgr`), or the **`par*` command-line tools**.

### Complex, Cells, and Why Isolation Matters

The whole cell-based server is called a **complex**. Inside it, a cell board is the fundamental building block — it carries the processors and physical memory. A cell connects through the backplane to an **I/O chassis**, and the chassis holds the **core I/O card** that provides the console path, boot devices, and LAN needed for an independent OS. An nPartition is simply an *assignment* of one or more cells (and their I/O) to a partition definition held in the complex's stable (non-volatile) configuration.

The reason nPars are described as "hardware" partitions is that the isolation is enforced by the hardware itself, not by software scheduling. Each nPar gets its own firmware instance and boots its own OS, and a hardware fault — a failed CPU, a memory error, a panic — is contained within the cells of that partition. This is the key difference from vPars: an nPar boundary survives failures that would take down a shared software layer. The tradeoff is granularity: because you assign whole cells, you cannot split a single cell's CPUs between two nPars (that finer split is what vPars are for).

Every cell also has a **use-on-next-boot** attribute (`y`/`n`). This is central to nPar work: it decides whether a cell participates the next time the partition goes through a reconfiguration, and it's why many changes require a "reboot for reconfig" rather than taking effect instantly.

### The Genesis Partition

The **genesis partition** is the very first nPar created in a complex — a single-cell partition built via the MP (Management Processor). Once created you load HP-UX into it, and from there you create additional nPars.

```
MP > CM > CC > option "G"
```

Pick a cell that has processors and memory and is connected to an I/O chassis with a core I/O card. Then boot it and install HP-UX:

```
MP> bo                      # boot the nPar (insert the HP-UX install DVD)
```

The genesis partition exists to solve a bootstrapping problem: the `par*` command-line tools run *inside* HP-UX, but a brand-new complex has no OS anywhere. So the very first partition must be created at the firmware level from the MP, deliberately kept minimal (a single cell), just enough to install HP-UX and gain a working command environment. Once you're logged into that OS, all further partitions are created and modified with the `par*` tools rather than from firmware. Choose a genesis cell that is fully self-sufficient — it must have both processors and memory *and* be wired to an I/O chassis with a core I/O card, otherwise it cannot boot an OS.

After install, log in and check status:

```bash
parstatus -w                # which partition am I in?
parstatus -Vp0              # detailed view of partition 0 (the genesis nPar)
parstatus -C                # summary of all cells in the complex
parstatus -X                # complex-wide product/serial and global info
```

## Creating Additional nPars

From the genesis partition, find free hardware and create a new nPar.

```bash
# Identify available hardware
parstatus -AC               # available cell boards
parstatus -AI               # available (free) I/O chassis

# Create a single-cell nPar using cell 1
parcreate -P test_npar -c1:::
```

The `-c<cell>:::` colon-separated fields are cell **type** (`base`), **use-on-next-boot** (`y`), and **interleave reuse** (`ri`, reuse memory after a failure). These are the defaults, so they can be omitted — the explicit form is:

```bash
parcreate -P test_npar -c1:base:y:ri
# multi-cell example:
parcreate -c4:base:y:ri -c6:base:y:ri
```

Reading the `-c` fields left to right:

- **cell** — the cell number from `parstatus -AC`; it must currently be free (not assigned to another nPar).
- **type** — `base` is the normal type; base cells always participate in the partition.
- **use-on-next-boot** — `y` means the cell joins the partition at the next boot/reconfig, `n` stages it as assigned-but-inactive.
- **interleave/failure-usage** — `ri` ("reuse if interleave") controls how the cell's memory is treated across reboots and after a memory failure.

On success `parcreate` prints the new partition number, which you'll pass to `parmodify`/`parstatus` as `-p<n>`. A newly created nPar has *no boot paths yet*, so set them before you try to boot (below). Multi-cell nPars gain more CPU and memory but note that all base cells must be present and healthy for the partition to boot — if one is missing at boot, the nPar can fail to come up rather than booting degraded, so plan cell counts around your availability needs.

Define boot disks and inspect the new nPar:

```bash
# Primary (-b) and alternate (-t) boot disks for partition 1
parmodify -p1 -b 1/0/0/2/0.6.0 -t 1/0/0/3/0.6.0
parstatus -Vp1              # detailed view of partition 1
parstatus -P               # list all partitions
```

Then boot the new nPar (`MP> bo`) with the install DVD in the drive and install HP-UX.

## Renaming an nPar

```bash
parmodify -p0 -P prod_npar     # rename partition 0 (genesis) to prod_npar
parstatus -P                   # verify
```

## Adding and Removing Cells

The key rule: **active** cell changes on an **active** nPar require a **reboot for reconfig** (`shutdown -R`); inactive-cell changes do not.

The reasoning behind the rule is that a running OS has already claimed a fixed set of CPUs, memory, and I/O from its active cells. HP-UX nPars are not "hot-add" at the OS level, so changing which cells are *active* means the partition must be re-formed from firmware — that is exactly what a reboot for reconfig does: it drops back to firmware, re-reads the partition definition, and rebuilds the hardware view before the OS boots again. Changes that only touch *inactive* cells (staging a cell with `use-on-next-boot=n`, or removing a cell that was never active) don't alter what the running OS is using, so they apply without a reconfig cycle. The `-B` flag on `parmodify` is what promotes a cell change to "active now," which is why the `-B` examples below are paired with a reconfig reboot.

### Add a cell to an active nPar

```bash
# Add cell 2 to partition 1; -B activates the cell
parmodify -p1 -a2::y: -B
shutdown -Ry now               # reboot for reconfig to activate
parstatus -P
```

### Add a cell to an inactive nPar

Use the same `parmodify -a` command. Include `-B` to boot and activate the cell, or omit `-B` to leave it unbooted. Either way, **no reboot-for-reconfig** is needed.

### Remove an active cell from an active nPar

```bash
parmodify -p1 -d2 -B
shutdown -Ry now               # reboot for reconfig for the change to take effect
```

### Remove an inactive cell

```bash
parmodify -p1 -d2              # no -B and no reboot needed
```

## Removing an nPar

### Active nPar

Run `parremove` from the nPar you're deleting, with `-F`. This unassigns all its cells and destroys the partition definition; then shut it down and halt it:

```bash
parremove -Fp1
shutdown -RH now               # reboot-for-reconfig + halt
```

### Inactive nPar

Run `parremove` from an **active** nPar, specifying the target partition number — no `-F` and no shutdown needed (it's already down):

```bash
parremove -p1
```

## Reboot vs Shutdown for Reconfig

Cell-based servers provide two special operations for nPar changes:

| Operation | Command | Use |
|-----------|---------|-----|
| **Reboot for reconfig** | `shutdown -R` | Reboots the nPar and re-reads its configuration (activates/deactivates cells) |
| **Shutdown for reconfig (+halt)** | `shutdown -RH` | Reconfig then halt — used when removing an active nPar |

`-R` triggers the reconfiguration; `-H` additionally halts rather than rebooting.

A plain `shutdown -r` (lowercase, no reconfig) reboots the OS but leaves the current hardware configuration in place — the partition comes back with the same active cells it had. Use the uppercase `-R` only when a cell activation/deactivation needs to take effect. Reboot for reconfig takes longer than an ordinary reboot because firmware re-runs power-on self-test and rebuilds the partition, so schedule it as a maintenance action rather than a quick bounce.

### Locks and parunlock

nPar configuration lives in the complex's stable storage, protected by locks so that two administrators (or a crashed command) can't corrupt it with simultaneous writes. If a `par*` command aborts partway through, it can leave a **stale configuration lock** that blocks further changes. `parunlock` clears such a lock:

```bash
parunlock -p1              # clear the config lock for partition 1
parunlock -C              # clear the complex-profile (stable-storage) lock
```

Use `parunlock` only after confirming no legitimate operation is in progress — clearing a lock while another change is genuinely running risks corrupting the partition profile.

## Administration Tools

- **Partition Manager** — GUI: `/opt/parmgr/bin/parmgr`.
- **nPartition commands** — CLI: `parcreate`, `parmodify`, `parremove`, `parstatus`, `parunlock`.
- **Service Processor (MP/GSP) menus** — hardware-level management and console.
- **EFI Boot Manager / EFI Shell** — on Integrity, when an nPar is active but hasn't booted an OS.
- **BCH menu** — on cell-based PA-RISC servers, when an nPar is active but hasn't booted an OS.

## Supported Operating Systems

nPartitions can run more than HP-UX:

- HP-UX 11i v1 (B.11.11), v2 (B.11.23), v3 (B.11.31)
- HP OpenVMS I64 8.2-1 and 8.3
- Microsoft Windows Server 2003
- Red Hat Enterprise Linux
- SUSE Linux Enterprise Server 9 and 10

## nPars vs vPars

| | nPartition (nPar) | Virtual Partition (vPar) |
|---|-------------------|--------------------------|
| Level | Hardware (cells) | Software (`vpmon`) |
| Isolation | Electrical — fault-isolated | Software isolation within one nPar |
| Granularity | Whole cell boards | Individual processors/memory |
| Managed by | MP + `par*` commands | `vpar*` commands |
| Created | In the complex | Inside an nPar |

The two are complementary layers: split a complex into nPars, then optionally subdivide an nPar into vPars.

## Command Reference

| Task | Command |
|------|---------|
| Create genesis partition | MP > CM > CC > `G` |
| Which partition am I in | `parstatus -w` |
| List all partitions | `parstatus -P` |
| Detailed partition view | `parstatus -Vp<n>` |
| Available cells / I/O | `parstatus -AC` / `parstatus -AI` |
| Create an nPar | `parcreate -P <name> -c<cell>:base:y:ri` |
| Set boot disks | `parmodify -p<n> -b <primary> -t <alt>` |
| Rename an nPar | `parmodify -p<n> -P "<name>"` |
| Add / remove a cell | `parmodify -p<n> -a<cell>::y: -B` / `-d<cell> -B` |
| Reboot for reconfig | `shutdown -Ry now` |
| Remove active nPar | `parremove -Fp<n>` then `shutdown -RH now` |
| Remove inactive nPar | `parremove -p<n>` (from an active nPar) |
| Boot an nPar | `MP> bo` |
| Clear a config lock | `parunlock -p<n>` / `parunlock -C` |
| Cell / complex summary | `parstatus -C` / `parstatus -X` |

## Related Articles

- [HP-UX Virtual Partitions (vPars)](articles/hpux-vpars.md) — software partitioning that subdivides a single nPar
- [HP-UX Management Processor](articles/hpux-management-processor.md) — the MP/GSP console used to create the genesis partition and access nPar consoles
- [HP-UX Boot Process](articles/hpux-boot-process.md) — how an nPar boots through firmware into HP-UX after a reconfig
- [HP-UX Installation with Ignite-UX](articles/hpux-installation-ignite.md) — installing HP-UX into a newly created nPar
- [HP-UX System Information](articles/hpux-system-information.md) — identifying the running partition and its assigned resources
