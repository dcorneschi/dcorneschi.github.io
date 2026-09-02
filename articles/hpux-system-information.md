# HP-UX System Information and Initial Configuration

Identifying an HP-UX system and setting its initial identity — the `set_parms` post-install configuration tool, and the commands for OS version, hardware model, CPU/memory inventory, run level, serial number, and quick resource checks. Covers HP-UX 11i v1–v3.

## Concepts: How HP-UX Identifies Itself

Before running any of the commands below it helps to understand where HP-UX keeps identity and inventory data, because different commands read from different places and can disagree with each other if the system was recently reconfigured.

- **Kernel-reported values** come from the running `/stand/vmunix` kernel and its in-memory I/O tree. `uname`, `who -r`, and `ioscan -k` all read this live state, so they reflect what the machine is doing *right now*.
- **Firmware/manifest values** — model, serial number, firmware revision, and cell/board layout — originate in the system's Processor Dependent Code (PDC) or, on Integrity servers, the Management Processor (MP) and EFI. Tools such as `machinfo`, `print_manifest`, and `getconf MACHINE_SERIAL` surface these.
- **Persisted configuration** (hostname, timezone, network) lives in text files under `/etc` (`/etc/hosts`, `/etc/rc.config.d/netconf`, `/etc/TIMEZONE`) that are written by `set_parms` and read at boot.

Because these three layers are separate, a value you just changed with `set_parms` may not match what a running daemon reports until the relevant service or the system is restarted. Keep that distinction in mind when a command's output surprises you.

## Initial Configuration with set_parms

`set_parms` runs automatically after OS installation to set up the system's identity; you can also re-run individual sections later. It is an interactive menu-driven front end that ultimately just writes the persisted configuration files described above, so anything it does can also be done by editing those files by hand — `set_parms` simply spares you from remembering every file and variable name.

```bash
set_parms initial        # full first-boot setup (runs after install)
set_parms hostname       # set the hostname
set_parms timezone       # set / display the timezone
set_parms date_time      # set date and time
set_parms locale         # set locales
set_parms ip_address     # set the primary IP address
set_parms addl_netwrk    # default gateway, DNS/NIS server details
```

Each subcommand launches the relevant part of the setup interview, so you can reconfigure one aspect (e.g. just the hostname or network) without redoing everything.

## OS Version and Release

```bash
uname -r                 # OS release, e.g. B.11.31 (11i v3)
uname -a                 # full system line
who -r                   # current run level
swlist 'HPUX*OE*'        # Operating Environment and OS version update
```

HP-UX release strings follow the form `B.11.<vv>`:

| Release string | Marketing name | Notes |
|----------------|----------------|-------|
| `B.11.00` | HP-UX 11.0 | Pre-11i; PA-RISC only |
| `B.11.11` | 11i v1 | PA-RISC; long-lived, widely deployed |
| `B.11.23` | 11i v2 | First release supporting Integrity (Itanium) alongside PA-RISC |
| `B.11.31` | 11i v3 | Introduces the agile device view and native multipathing; PA-RISC + Integrity |

The leading `B.` is the release *format* identifier and is effectively constant; the meaningful part is the numeric tail. Note that `uname -r` tells you the base OS version but **not** the Operating Environment (OE) update level — for that you need `swlist`, described below.

### OE Version and Update Level

The base OS and the bundled applications are packaged together as an **Operating Environment** (OE) — for example the Base OE (BOE), Data Center OE (DCOE), or Virtual Server OE (VSE-OE). The OE has its own version stamp that advances with each quarterly update release, independent of the `B.11.31` base string:

```bash
swlist -l bundle | grep -i OE     # which OE bundle(s) are installed
swlist 'HPUX*OE*'                 # OE and its update/version
```

Two systems can both report `B.11.31` from `uname -r` yet be months apart in patches; the OE bundle version (e.g. `B.11.31.1103` = the March 2011 update) is what actually pins the patch baseline. See [HP-UX swlist and Software Structure](articles/hpux-swlist-software-structure.md) for how bundles, products, and filesets nest.

## Hardware Model and Inventory

```bash
uname -m                 # machine model / hardware type
print_manifest | more    # full hardware manifest (CPU, memory, I/O, firmware)
getconf MACHINE_SERIAL   # system serial number
getconf KERNEL_BITS      # running kernel bit-width (32/64)
getconf HW_CPU_SUPP_BITS # bit-width the CPU hardware supports (32/64)
```

`print_manifest` (from Ignite-UX) is the most complete single inventory — model, CPUs, memory, disks, LAN, firmware, and installed software. It is generated from the same data Ignite-UX collects when making a recovery image, so it doubles as a "known-good baseline" you can archive and diff against after hardware changes.

`uname -m` prints a numeric model code on PA-RISC (for example `9000/800`) and a descriptive string on Integrity (for example `ia64`). To get the full marketing model name and a lot more, prefer `model` and `machinfo`:

```bash
model                    # e.g. "ia64 hp server rx2660" or "9000/800/rp3440"
machinfo                 # Integrity: model, CPU type/speed, memory, firmware
```

> **Gotcha:** `machinfo` exists only on Integrity (Itanium) servers. On PA-RISC systems it is absent; use `print_manifest`, `model`, and `ioscan` there instead.

## CPU Information

```bash
# Count the processors
ioscan -kfnC processor | grep processor | wc -l

# Machine model (also indicates architecture family)
uname -m
```

`ioscan -kfnC processor` lists processor hardware; counting the `processor` lines gives the CPU count. Note that on multi-core and Hyper-Threaded Integrity systems this counts *logical* processors (each core/thread appears as a `processor` entry), which may exceed the number of physical sockets. On Integrity, `machinfo` breaks this down explicitly:

```bash
machinfo | grep -Ei 'cpu|core|logical'   # sockets, cores, and logical CPUs
```

The `-k` flag reads the kernel's cached I/O tree instead of physically probing the hardware, so it returns instantly and is safe to run on a busy production box. Drop `-k` only when you genuinely need to force a fresh probe (for example right after hot-adding hardware), because a live probe briefly quiesces the bus. See [HP-UX Device Management (ioscan)](articles/hpux-device-management-ioscan.md) for the full `ioscan` treatment.

## Memory

```bash
# Total physical memory (several equivalent methods)
dmesg | grep -i Physical
/opt/ignite/bin/print_manifest | grep Memory
machinfo | grep Memory                    # Integrity servers

# Free memory (via the kernel debugger)
echo 'freemem/D' | adb /stand/vmunix /dev/kmem
```

- `machinfo` is the go-to hardware/memory summary on **Integrity** servers.
- `freemem/D` read through `adb` reports free memory pages from the running kernel. The value is in **pages**, not bytes — multiply by the page size (typically 4 KB) to get an approximate free-memory figure. This is a point-in-time snapshot and, because HP-UX aggressively uses free memory for buffer/file cache, a "low free memory" reading is usually normal rather than a problem.

For a live view of memory pressure and paging activity, `vmstat` is more useful than a single free-memory reading:

```bash
vmstat 5 5              # 5 samples, 5s apart: watch 'free', 'po' (page-out), 'w'
swapinfo -tam           # device + pseudo swap usage in MB, with totals
```

Sustained non-zero **page-out (`po`)** in `vmstat` is the real signal of a memory shortage; a low `free` column on its own is not. `swapinfo -tam` shows how much backing store (swap) is committed — HP-UX uses both device swap and *pseudo-swap* (a portion of physical RAM), so watch the total line.

## Quick Process / Resource Checks

```bash
# Top memory-consuming process (SZ = size in pages).
# UNIX95= enables the XPG4 ps that supports -o output columns.
UNIX95= ps -ef -o pid,ppid,pcpu,sz,args | sort -nbk 4 | tail -1

# (drop the tail -1 to see the full sorted list; use tail -10 for top 10)
UNIX95= ps -ef -o pid,ppid,pcpu,sz,args | sort -nbk 4 | tail -10
```

> The `UNIX95=` prefix enables XPG4 behavior for that command, which is what lets HP-UX `ps` accept the `-o` column syntax.

## Diagnostics

```bash
stm                      # Support Tools Manager (hardware diagnostics; cstm/xstm variants)
```

`stm` (and its `cstm` command-line and `xstm` X11 variants) runs hardware diagnostics and reports device status.

## Reading Man Pages

HP-UX man output can contain overstrike formatting; filter it for clean paging:

```bash
export PAGER="col -b | less"     # set a clean pager for man
man <topic> | col -b | more      # or filter inline
```

## Command Reference

| Task | Command |
|------|---------|
| First-boot setup | `set_parms initial` |
| Set hostname / IP / network | `set_parms hostname` / `ip_address` / `addl_netwrk` |
| Set timezone / date / locale | `set_parms timezone` / `date_time` / `locale` |
| OS release | `uname -r` |
| Machine model | `uname -m` |
| Run level | `who -r` |
| OE / OS update level | `swlist 'HPUX*OE*'` |
| Full hardware manifest | `print_manifest \| more` |
| Serial number | `getconf MACHINE_SERIAL` |
| CPU bit-width | `getconf KERNEL_BITS` (kernel) / `getconf HW_CPU_SUPP_BITS` (CPU) |
| CPU count | `ioscan -kfnC processor \| grep processor \| wc -l` |
| Physical memory | `dmesg \| grep -i Physical`, `machinfo \| grep Memory` |
| Free memory | `echo 'freemem/D' \| adb /stand/vmunix /dev/kmem` |
| Top memory process | `UNIX95= ps -ef -o pid,ppid,pcpu,sz,args \| sort -nbk 4 \| tail` |
| Hardware diagnostics | `stm` / `cstm` / `xstm` |
| Marketing model name | `model` |
| Integrity hardware summary | `machinfo` |
| Paging / memory pressure | `vmstat 5 5` |
| Swap usage | `swapinfo -tam` |

## Troubleshooting

**`uname -r` and `swlist` disagree about the version.** They are answering different questions: `uname -r` reports the base OS release compiled into the running kernel, while `swlist 'HPUX*OE*'` reports the installed Operating Environment update level. A newly patched system can show the same `B.11.31` from `uname` while its OE bundle version has advanced. Trust the OE bundle version for patch-baseline decisions.

**`machinfo: not found`.** You are on a PA-RISC system. `machinfo` is Integrity-only. Use `print_manifest`, `model`, and `ioscan -kfC processor` instead.

**`ps -o` reports "illegal option".** The default HP-UX `ps` does not accept `-o` column formatting; you must prefix the command with `UNIX95=` to switch it into XPG4 mode for that invocation, as shown in the resource-check section above.

**`getconf MACHINE_SERIAL` is blank or wrong.** The serial number is read from firmware. On a VM or a partition (vPar/nPar) it may reflect the container rather than physical hardware, and on some older/whiteboxed units the firmware field may be unset. Cross-check against `print_manifest` and the physical label.

**Man pages show `^H` overstrike garbage.** HP-UX `nroff` output embeds backspace overstrike sequences for bold/underline. Filter with `col -b` as shown in the Reading Man Pages section, or set the `PAGER` environment variable once in your profile.

## Related Articles

- [HP-UX Startup, Run Levels, and Network Services](articles/hpux-startup-and-services.md)
- [HP-UX Device Management (ioscan)](articles/hpux-device-management-ioscan.md)
- [HP-UX swlist and Software Structure](articles/hpux-swlist-software-structure.md)
- [HP-UX Software Distribution (SD-UX): Depots and swinstall](articles/hpux-software-depots-swinstall.md)
- [HP-UX Kernel Configuration](articles/hpux-kernel-configuration.md)
- [HP-UX Installation with Ignite-UX](articles/hpux-installation-ignite.md)
