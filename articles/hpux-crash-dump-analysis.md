# HP-UX Crash Dump Analysis with Q4

Analyzing an HP-UX system crash or hang. When the kernel panics (or you force a dump on a hung system), HP-UX writes a memory image that `savecrash` turns into a crash dump directory. The `q4` debugger then extracts a human-readable analysis of what happened. Related: [HP-UX Boot Process](articles/hpux-boot-process.md) and [HP-UX Kernel Configuration and Tuning](articles/hpux-kernel-configuration.md).

## How Crash Dumps Are Produced

When the kernel detects an unrecoverable condition — a bad memory reference in kernel context, a failed assertion, a hardware machine check, or a manually forced transfer-of-control — it **panics**. A panic prints a message on the console, stops normal scheduling, and enters the dump path: it writes the contents of physical memory (or a selected subset) to the configured **dump devices**. Because the machine is in an untrusted state at that point, the dump is written using the kernel's own low-level dump drivers rather than the normal filesystem/LVM stack — it writes raw blocks directly to the dump devices.

The reason a two-stage design exists (dump-to-device at panic time, then copy-to-filesystem at boot time) is that the filesystem cannot be trusted or safely mounted during a panic. So the flow is:

1. **At panic**: the kernel dumps memory to the raw dump device(s). By default the primary swap device doubles as a dump device, which is why "swap" and "dump" are entangled on HP-UX.
2. **At next boot**: early in startup, before swap is activated and would overwrite the image, `savecrash` reads the raw dump back off the device and writes it into a crash directory on a real filesystem.

- Crash dumps are saved under `/var/adm/crash/`, one numbered directory per crash: `crash.0`, `crash.1`, and so on. The next number to use is tracked in `/var/adm/crash/bounds`.
- Each directory holds the saved kernel image (`vmunix.gz`), the memory pages (`image.*.gz`), and an `INDEX` file describing the dump (panic string, time, dump type, page ranges).

For a **hung** (but not panicked) system — one that responds to nothing — there is no automatic panic, so you must force one from the console. A **TC (Transfer of Control)** issued from the [Management Processor](articles/hpux-management-processor.md) forces the CPU into the panic/dump path, producing the same kind of dump you'd get from a spontaneous panic. That's the only way to get a memory image out of a wedged system for later analysis.

### Dump Devices and Sizing

A dump device must be a block device dedicated (at dump time) to receiving the image: a swap/dump logical volume, a whole disk, or a dedicated dump-only LV. On PA-RISC these are configured in the kernel; on 11i they're managed at runtime with `crashconf`.

Sizing is the perennial tradeoff. A **full dump** captures all of physical memory, so the dump device(s) must total at least the size of RAM (plus a little overhead). On a machine with tens or hundreds of GB of RAM, that's a lot of disk — and a full dump also takes a long time to write and to `savecrash`, extending the outage. That's why HP-UX supports **selective dump** (also called *partial* or *class-based* dump): the kernel dumps only the memory classes likely to matter for diagnosis (kernel code and data, kernel stacks, etc.) and skips, for example, user process pages and the filesystem buffer cache. Selective dumps are dramatically smaller and faster, and for the vast majority of panics they contain everything an analyst needs.

Two related space concerns:

- The **dump device** must be big enough for the amount of memory the selected dump classes will produce. If the configured dump space is too small, the dump is truncated and may be unusable.
- The **crash directory filesystem** (`/var/adm/crash`) must have room for the copied-and-compressed image. `savecrash` compresses as it copies, but a full dump of a large machine can still be huge. If `/var` is too small, either point the crash directory elsewhere or rely on selective dump.

## Configuring Crash Dumps with crashconf

On 11i, `crashconf` displays and changes the runtime dump configuration: which memory classes are dumped and which devices receive the dump. Run with no arguments to see the current state:

```bash
crashconf                 # show current dump classes and dump devices
crashconf -v              # verbose: per-class page counts and device sizes
```

The output lists each **dump class** (for example `UNUSED`, `USERPAGE`, `BCACHE`, `KCODE`, `USTACK`, `FSDATA`, `KDDATA`, `KSDATA`) with a flag showing whether it will be dumped. Classes are marked as `INCLUDE` (always dump), `EXCLUDE` (never dump), or left to the default. A **full dump** includes everything; a **selective dump** typically excludes the large, rarely-needed classes such as user pages (`USERPAGE`) and the filesystem buffer cache (`BCACHE`) while keeping the kernel classes.

Change the classes at runtime (takes effect for the next panic):

```bash
crashconf KCODE KSDATA KDDATA USTACK       # include just these classes (selective)
crashconf -i USERPAGE                      # add (include) a class
crashconf -e BCACHE                        # exclude a class
```

To make the configuration **persist across reboots**, set it in `/etc/rc.config.d/`. The startup script reads these variables and calls `crashconf` at boot:

```bash
# /etc/rc.config.d/crashconf
CRASHCONF_ENABLED=1
# space-separated list of classes to dump, or the keyword to select full/selective
```

Add or remove a dump device:

```bash
crashconf -a /dev/vg00/lvol2               # add a device to the dump set
crashconf -d /dev/vg00/lvol2               # remove a device from the dump set
```

Because the primary swap LV is usually also the primary dump device, be careful when resizing swap: the dump device must remain large enough for the selected dump classes. See [HP-UX Swap Management](articles/hpux-swap-management.md) for how swap and dump devices interrelate, and [HP-UX Kernel Configuration and Tuning](articles/hpux-kernel-configuration.md) for the boot-time dump settings on PA-RISC kernels.

## savecrash at Boot

`savecrash` is what turns the raw on-device image into the analyzable `/var/adm/crash/crash.N` directory. It runs automatically early in the boot sequence, driven by `/etc/rc.config.d/savecrash`:

```bash
# /etc/rc.config.d/savecrash
SAVECRASH=1                          # 1 = run savecrash at boot (0 disables)
SAVECRASH_DIR=/var/adm/crash         # where crash directories are written
SAVECRASH_INPLACE=0                  # 1 = defer copy, reference dump on device
```

Key behaviors and why they matter:

- **Timing**: `savecrash` must run *before* swap is enabled, because once swap activates it will reuse the very device holding the dump. This is why it's early in the rc sequence and why a system that hangs during that early boot can lose its dump.
- **Compression**: it compresses pages as it writes them (`vmunix.gz`, `image.*.gz`), so the crash directory is smaller than raw memory.
- **`SAVECRASH_INPLACE=1`**: on systems where copying a huge dump would delay boot too long, in-place mode records the dump on the device and copies it later (or lets you copy it manually with `savecrash -r`), speeding the return to service. The risk is that the dump device isn't reclaimed for swap until the copy is done.
- **Manual run**: you can invoke it by hand against the current dump device, e.g. `savecrash -r -d /var/adm/crash` to recover a dump you deferred.
- **Housekeeping**: crash directories are not pruned automatically. Old `crash.N` directories accumulate under `/var/adm/crash` and can fill `/var`; delete analyzed dumps once support no longer needs them.

## Forcing a Dump on a Hung System

If a system is hung — no console response, no ping, no login — there is nothing to type a command into, so you force the panic/dump from the service processor. From the [Management Processor](articles/hpux-management-processor.md) (MP/GSP/iLO) console:

```
MP> CM
CM> TC
```

`TC` (Transfer of Control) drives the CPUs into the panic path, which then dumps memory to the configured dump devices exactly as a spontaneous panic would. On the next boot, `savecrash` saves it and you analyze it with Q4 as usual.

A few cautions:

- `TC` is a hard action — it stops the OS immediately. Only use it when the system is genuinely hung and you've decided a dump is worth more than any chance of the system recovering on its own.
- Make sure dump devices are large enough for at least a selective dump *before* you need this; a forced dump into an undersized device yields a truncated, less useful image.
- On some platforms a physical **TOC** button provides the same function when you don't have MP access; consult the hardware's operations guide.

## Analyzing a Dump with Q4

`q4` is the HP-UX crash-dump debugger, found at `/usr/contrib/Q4/bin/q4`. Point it at a crash directory and run its analysis macros.

```bash
cd /var/adm/crash/crash.0
/usr/contrib/Q4/bin/q4 -p .          # note the trailing "dot" — the current directory
```

At the `q4>` prompt, run the analysis macros and redirect their output to files:

```
q4> run Analyze AU > ana.out
q4> run WhatHappened -HANG > what.out
q4> exit
```

- `Analyze AU` produces a general analysis of the dump (`AU` = analyze/unwind), written to `ana.out`.
- `WhatHappened -HANG` focuses on why the system hung, written to `what.out`.
- Both macros can take **several minutes** to process; **Ctrl-C** interrupts them if needed.
- After exiting, read `ana.out` and `what.out` (and share them with HP support along with the dump if you have a support contract).

### What Q4 Actually Does

`q4` is a structured debugger, not just a text viewer. It reads the compressed dump directory, reconstructs the kernel's data structures from the saved memory image, and lets you walk them by name and type. When it starts it must first build (or load) a **layout** describing the kernel's symbols and structure offsets so it can interpret raw memory. On first run against a dump you'll often see it perform a `load` step; subsequent runs are faster because the layout is cached in the crash directory.

The `-p` flag points `q4` at the crash directory you give it (here `.`, the current crash directory), where it reads the saved kernel image and memory pages:

```bash
cd /var/adm/crash/crash.0
/usr/contrib/Q4/bin/q4 -p .
```

On its first run against a dump, `q4` builds the symbol/structure layout it needs (you may see a `load` step and a short delay); it caches that work in the crash directory so later runs start faster.

### A Fuller Q4 Workflow

Beyond the two canned macros, a typical hands-on investigation looks like this. Everything below is entered at the `q4>` prompt:

```
q4> load struct utsname from &utsname     # identify the OS: release, node, machine
q4> print utsname.release utsname.nodename

q4> ex &panicstr using s                  # print the panic string (why it panicked)
q4> trace                                  # stack trace of the panicking context

q4> run Analyze AU > ana.out               # broad automated analysis
q4> run WhatHappened -HANG > what.out      # hang-focused analysis
```

Interpreting the results:

- The **panic string** (`panicstr`) is the single most useful clue — it's the message the kernel printed on the console. Common examples point at a specific subsystem (e.g. a data page fault in kernel mode, a spinlock/deadlock condition, or a hardware machine-check).
- The **stack trace** (`trace`) of the failing thread shows the call chain that led to the panic. The top frames usually name the driver or kernel module responsible, which is what you'd hand to HP support or use to identify a bad patch/driver.
- `Analyze AU` cross-checks many structures at once (run queues, sleeping processes, locks) and flags anomalies — useful when the panic string alone is generic.
- `WhatHappened -HANG` is aimed at forced dumps (a `TC` on a hung box) where there is no organic panic string; it reasons about what the CPUs were doing and where they were stuck.

### Practical Notes and Gotchas

- **Match the tools to the dump.** Q4's ability to interpret the dump depends on having symbol/layout information consistent with the kernel that crashed. Analyze the dump on the same system (or a system running the same OS release and patch level) whenever possible.
- **Work on a copy.** The dump directory can be large; copy it off to a workstation or support-upload area before poking at it, and keep the original intact.
- **Redirect long output to files** (as shown) rather than scrolling it in the terminal — the macros produce a lot of text and you'll want to search it afterward.
- **Deep analysis is HP's job.** For most sites the goal is to capture a good dump, run `Analyze AU` / `WhatHappened`, and open a support case attaching `ana.out`, `what.out`, and the dump. Reading raw kernel structures is an expert task; the value you add is ensuring dumps are configured, complete, and preserved.

## Command Reference

| Task | Command |
|------|---------|
| Show current dump configuration | `crashconf -v` |
| Set selective dump classes | `crashconf KCODE KSDATA KDDATA USTACK` |
| Add / remove a dump device | `crashconf -a <dev>` / `crashconf -d <dev>` |
| Manually save a deferred dump | `savecrash -r -d /var/adm/crash` |
| Find the next crash number | `cat /var/adm/crash/bounds` |
| Force a dump on a hung system | `MP> CM` then `CM> TC` |
| Change to the crash directory | `cd /var/adm/crash/crash.0` |
| Launch Q4 against the dump | `/usr/contrib/Q4/bin/q4 -p .` |
| Print the panic string | `q4> ex &panicstr using s` |
| Stack trace of failing context | `q4> trace` |
| General dump analysis | `q4> run Analyze AU > ana.out` |
| Hang-focused analysis | `q4> run WhatHappened -HANG > what.out` |
| Interrupt a running macro | `Ctrl-C` |
| Quit Q4 | `q4> exit` |

## Related Articles

- [HP-UX Boot Process](articles/hpux-boot-process.md)
- [HP-UX Kernel Configuration and Tuning](articles/hpux-kernel-configuration.md)
- [HP-UX Management Processor (MP / GSP / iLO)](articles/hpux-management-processor.md)
- [HP-UX Swap Management](articles/hpux-swap-management.md)
- [HP-UX Performance Monitoring and Event Management](articles/hpux-performance-monitoring.md)
- [HP-UX Disaster Recovery (DRD and Ignite-UX)](articles/hpux-disaster-recovery.md)
- [HP-UX Administration Tips and Recipes](articles/hpux-admin-tips-recipes.md)


