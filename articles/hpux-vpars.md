# HP-UX Virtual Partitions (vPars)

Virtual Partitions (vPars) are HP-UX's **software** partitioning technology: they divide a server (or an nPar) into several independent virtual machines, each running its own HP-UX instance with dedicated processor(s), memory, and I/O (LBAs). This guide covers the concepts, bound vs unbound processors, and the full workflow to create, boot, and manage vPars.

## Why vPars

Physical servers are often far larger than any single workload needs, and HP's partitionable systems can be split at the hardware level into **nPars** — but nPars split only at cell (physical) boundaries, so their granularity is coarse and you cannot make one smaller than a cell. vPars fill the gap by subdividing a single server or nPar **in software**, down to the level of an individual processor. Each vPar boots and runs a fully independent HP-UX image: it has its own kernel, its own patch level, its own root disk, and its own network identity, and a crash or reboot in one vPar does not touch the others. This lets you consolidate several small servers onto one box, run a test instance beside production on the same hardware, or stage an OS upgrade in one vPar while the rest keep running — all while sharing the underlying iron. The cost of that flexibility is that vPars share the physical machine, so they are *software*-isolated, not electrically isolated the way nPars are.

## What vPars Are

- A **vPar** is a virtual, standalone server running its own HP-UX instance with its own processor(s), memory, and Local Bus Adapter(s) (LBAs).
- Each vPar needs **at least one processor** — the max number of vPars in a server/nPar equals the number of processors it has.
- A complex supports a minimum of **2** vPars and a maximum of **128** in the largest complex.
- vPars in the same server/nPar can run **different patch levels** and different applications, isolated from each other.

### vpmon and the vPar Database

When vPars are created, a software monitor called **`vpmon`** is enabled between the server firmware and the HP-UX instances. At vPar boot, `vpmon`:

1. Reads the vPar database **`/stand/vpdb`**,
2. Loads HP-UX into the vPar,
3. Uses the boot disks defined for that vPar and assigns its configured hardware.

You can run vPar commands from **any vPar** in the complex, or use the Virtual Partition Manager GUI.

Conceptually, `vpmon` sits in the same slot the HP-UX kernel would normally occupy: firmware loads `vpmon` first, and `vpmon` then loads a separate `vmunix` into each vPar and arbitrates their access to shared hardware. This is why making a system "vPar-aware" is a matter of changing the boot string in the AUTO file to load `/stand/vpmon` instead of `/stand/vmunix` directly — see [HP-UX Boot Process](articles/hpux-boot-process.md) for how that boot string is read. Because `vpmon` is the authority on the configuration, all `vpar*` commands ultimately read or write the same shared database (`/stand/vpdb`), which is why you can administer the whole complex from inside any one vPar.

### vPars Versions

There are two major generations. **vPars A.03.xx** runs on HP-UX 11i v2 and v3 and is the classic model described throughout this guide (processor-, memory-, and LBA-granular). **vPars A.05.xx / vPars and Integrity VM (HP-UX 11i v3)** integrated vPars with the Integrity VM hypervisor, adding finer memory granularity and online migration features. Command names (`vparcreate`, `vparstatus`, etc.) are consistent across generations, but available attributes and limits differ, so confirm the installed version before relying on a specific feature:

```bash
vparstatus -V        # show the vPars monitor/version
swlist -l product | grep -i vpar
```

### Prerequisite

Creating the first vPar in an nPar requires the vPars software (**T1335CC**) installed:

```bash
swlist | grep T1335CC
```

## Bound vs Unbound Processors

| Type | Handles I/O interrupts? | Online add? | Online remove? | Use for |
|------|:---:|:---:|:---:|---------|
| **Bound** | Yes | Yes | No | vPars needing both CPU and I/O (every vPar needs ≥1 bound CPU) |
| **Unbound** | No | Yes | Yes | CPU-intensive but not I/O-intensive workloads |

- Every vPar **must** have at least one bound processor (it services that vPar's I/O interrupts).
- An unbound processor is either unassigned, or assigned but not handling I/O — and can be added/removed online.

The reason for the distinction is interrupt handling. Every vPar needs *some* CPU that is guaranteed to be there to field its device interrupts, so at least one processor is **bound** to it — pinned, and therefore not eligible for online removal (you can't yank the CPU that is handling the disk interrupts out from under a running vPar). **Unbound** processors are the flexible pool: they do computation but not I/O interrupt servicing, so `vpmon` can hand them to whichever vPar needs cycles and reclaim them later without disrupting I/O. This is what makes vPars useful for bursty workloads — bind the minimum you need for I/O, then float unbound CPUs to whoever is busy.

### Moving Processors Between vPars Online

Unbound CPUs can be reassigned while vPars are running, which is the everyday knob for rebalancing load:

```bash
# Add 2 CPUs to vpar01 (drawn from the free pool, up to its max)
vparmodify -p vpar01 -a cpu::2

# Remove 1 (unbound) CPU from vpar01, returning it to the pool
vparmodify -p vpar01 -d cpu::1

# Add a specific processor by hardware path, bound
vparmodify -p vpar01 -a cpu:41

# Change the min/max CPU bounds for a vpar
vparmodify -p vpar01 -a cpu:::1:4
```

You can only remove **unbound** CPUs online, and you can never take a vPar below its configured minimum or its single required bound CPU. Memory on the classic A.03 model is not dynamically migratable the way unbound CPUs are — plan memory allocation at creation and change it while the target vPar is down.

## Creating the First vPar

Example complex (rp8400 nPar): 2 cells, 8 processors, 16 GB RAM, 2 I/O chassis. Processor hardware addresses: 41, 45, 101, 109, 141, 145, 201, 209.

Create `test_vpar0` with 1 bound + 1 unbound processor (min 1, max 2), 8 GB RAM, two LBAs, and a primary boot disk:

```bash
vparcreate -p test_vpar0 \
  -a cpu::2 \
  -a cpu:::1:2 \
  -a cpu:41 \
  -a mem::8192 \
  -a io:1/0/0/2 \
  -a io:1/0/0/3 \
  -a io:1/0/0/2/0.6.0:BOOT \
  -B search -B auto
```

| Option | Meaning |
|--------|---------|
| `-p` | Name of the vPar |
| `-a cpu::2` | Number of processors to allocate |
| `-a cpu:::1:2` | Min and max processor limits (1 and 2) |
| `-a cpu:41` | Hardware address of the processor to **bind** (omit → unbound) |
| `-a mem::8192` | Memory in MB |
| `-a io:1/0/0/2` | Allocate all devices on that LBA path to this vPar |
| `-a io:.../0.6.0:BOOT` | Primary boot path |
| `-B search` | Auto-search for the boot device at vPar boot |
| `-B auto` | Auto-boot from the boot device |

The configuration is stored in `/stand/vpdb` (created if absent).

### Make the nPar vPar-Aware

Modify the AUTO file on the primary boot path so the system loads `vpmon` before the HP-UX kernel at next reboot:

```bash
# HP 9000 (PA-RISC) servers
mkboot -a "hpux /stand/vpmon -a" /dev/rdsk/c0t6d0

# Integrity servers
mkboot -a "boot vpmon -a" /dev/rdsk/c0t6d0
```

After this, each reboot of the nPar brings it up as a vPar.

### Boot the First vPar

From the nPar console, press **Ctrl-A** to reach the virtual console monitor `MON>` prompt:

```
MON> vparload -p test_vpar0
```

Or, on an HP 9000 server, from the ISL prompt:

```
ISL> hpux /stand/vpmon vparload -p test_vpar0
```

Check status once it's up:

```bash
vparstatus                       # summary of all vPars
vparstatus -vp test_vpar0        # detailed view of one vPar
```

In `vparstatus` output, the **Attributes** column shows `Dyn` (dynamic hardware changes allowed) and `Auto` (auto-bootable).

## Creating Another vPar

From within `test_vpar0`, create `test_vpar1` with a bound processor, 8 GB, and primary + alternate boot paths:

```bash
vparcreate -p test_vpar1 \
  -a cpu::2 -a cpu:::1:2 -a cpu:101 \
  -a mem::8192 \
  -a io:2/0/0/2 -a io:2/0/0/3 \
  -a io:2/0/0/2/0.6.0:BOOT \
  -a io:2/0/0/3/0.6.0:ALTBOOT \
  -B search -B auto
```

Then:

1. **Install HP-UX** into `test_vpar1` (include the vPars software in the OE). Shut down `test_vpar0`, go to BCH, and install onto one of `test_vpar1`'s boot disks using the nPar's DVD drive.
2. Boot `test_vpar0` back up from the monitor: `MON> vparload -p test_vpar0`.
3. From `test_vpar0`, set the AUTO file on `test_vpar1`'s boot disks so they load `vpmon`:

   ```bash
   mkboot -a "hpux /stand/vpmon -a" /dev/rdsk/c2t6d0
   mkboot -a "hpux /stand/vpmon -a" /dev/rdsk/c3t6d0
   ```

4. Bring `test_vpar1` up from `test_vpar0`:

   ```bash
   vparboot -p test_vpar1
   ```

## Booting vPars (`vparload` at the MON> prompt)

```
MON> vparload -all                         # boot all vPars
MON> vparload -p vpar01                     # boot one vPar
MON> vparload -auto                         # boot all vPars with autoboot=AUTO
MON> vparload -p vpar01 -o "is"             # boot to single-user mode
MON> vparload -p vpar01 -b /stand/vmunix.prev   # boot an alternate kernel
MON> vparload -p vpar01 -B 0/3/0/0.2.0      # boot from a specific disk device
```

Boot a vPar without entering the vpmon prompt (from ISL):

```
ISL> hpux /stand/vpmon -p vpar01
```

## Managing vPars from a Running vPar

Commands issued from inside one vPar can control others in the complex:

```bash
# Boot another vPar
vparboot  -p vpar02

# Delete a vPar
vparremove -p vpar02

# Halt/shut down another vPar
vparreset -p vpar02 -h

# Status (detailed, one vPar)
vparstatus -v -p vpar01
```

> Note: the halt/reset command is `vparreset` (the `-h` option halts). Use it carefully — it stops the target vPar.

## Command Reference

| Task | Command |
|------|---------|
| Check vPars software | `swlist \| grep T1335CC` |
| Create a vPar | `vparcreate -p <name> -a cpu:: -a mem:: -a io:... -B search -B auto` |
| Make nPar vPar-aware | `mkboot -a "hpux /stand/vpmon -a" <disk>` (9000) / `"boot vpmon -a"` (Integrity) |
| Boot a vPar (monitor) | `MON> vparload -p <name>` / `-all` / `-auto` |
| Boot a vPar (from a vPar) | `vparboot -p <name>` |
| Single-user boot | `MON> vparload -p <name> -o "is"` |
| Status | `vparstatus` / `vparstatus -v -p <name>` |
| Halt a vPar | `vparreset -p <name> -h` |
| Remove a vPar | `vparremove -p <name>` |
| vPar database | `/stand/vpdb` |
| Monitor prompt | `Ctrl-A` → `MON>` |

## Troubleshooting

| Symptom | Likely cause | Action |
|---------|--------------|--------|
| `vparcreate` fails: software not found | vPars fileset (T1335CC) not installed | `swlist \| grep T1335CC`, install from the OE |
| System boots straight to normal HP-UX, no `MON>` | AUTO file still loads `vmunix`, not `vpmon` | `mkboot -a "hpux /stand/vpmon -a" <disk>` (9000) / `"boot vpmon -a"` (Integrity) |
| vPar won't boot: boot device not found | Wrong/renumbered boot LBA path in the vPar config | `vparstatus -v -p <name>` to check the BOOT path; fix with `vparmodify` |
| Can't remove a CPU online | It's the bound CPU, or would drop below min | Only unbound CPUs above the minimum are removable |
| `vparload` at `MON>` hangs waiting for device | `-B search` finding nothing, or disk offline | Boot with an explicit `-B <path>`, verify the disk is present |
| Changes to a vPar don't take effect | vPar wasn't `Dyn`, or change needs a reboot | Check the Attributes column in `vparstatus`; reboot the vPar for static changes |

> **Watch the AUTO file.** The single most common "my vPars aren't working" cause is a boot disk whose AUTO file still points at `/stand/vmunix` — the nPar boots as a plain HP-UX system and `vpmon` never loads, so there is no `MON>` prompt and no vPars. Re-check every boot disk with `mkboot`/`lifcp` after installs and OS updates, which sometimes rewrite the boot string.

## vPars vs nPars

- **nPars** (hardware/node partitions) split a complex at the **cell/hardware** level — electrically isolated, each with its own firmware.
- **vPars** subdivide a single nPar (or server) in **software** via `vpmon`, sharing the underlying hardware but each running an independent HP-UX instance.
- You create vPars *inside* an nPar; the two are complementary partitioning layers.

Think of it as a hierarchy of isolation strength versus flexibility. nPars give the strongest isolation — a hardware or firmware fault in one cell doesn't cross into another — but you can only carve at cell boundaries and reconfiguration means moving physical resources. vPars give fine-grained, software-defined partitions you can resize online (at least for unbound CPUs), at the cost of sharing the physical machine and its `vpmon` layer. Many large HP complexes use both: nPars to create a handful of electrically isolated hardware partitions, then vPars *within* each nPar to subdivide further and float CPU capacity between workloads. The layers stack rather than compete.

| | nPars | vPars |
|---|-------|-------|
| Partition boundary | Cell / hardware | Processor / software |
| Isolation | Electrical, independent firmware | Software (`vpmon`) |
| Granularity | Whole cells | Individual CPUs, memory, LBAs |
| Online resize | Limited (cell-level) | Yes for unbound CPUs |
| Fault containment | Strongest | Per-vPar OS crash isolation |

## Related Articles

- [HP-UX Boot Process (PA-RISC and Integrity)](articles/hpux-boot-process.md) — how `vpmon` is loaded ahead of the kernel via the AUTO file
- [HP-UX LVM (Logical Volume Manager)](articles/hpux-lvm.md) — managing each vPar's boot and data disks
