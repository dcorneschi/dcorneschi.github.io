# HP-UX Kernel Configuration and Tuning

Building and tuning the HP-UX kernel. Two eras: the older **11i v1** flow that rebuilds a kernel under `/stand/build` and swaps it in manually, and the **11i v2/v3** `kc*` toolset (`kconfig`, `kctune`, `kcmodule`, `kcusage`, `kcalarm`, `kclog`) that manages named configurations, dynamic tunables, and module states. Also covers booting a specific kernel from ISL/BCH/EFI.

## Concepts: What the Kernel Configuration Actually Is

The HP-UX kernel is not a single monolithic file you edit — it is an *assembled* object built from a set of **modules** (drivers and subsystems, each in one of several states) plus a set of **tunable parameters** (numeric or flag values that size and shape kernel behaviour). A **kernel configuration** is the complete, named description of that assembly: which modules are static/dynamic/unused and what every tunable is set to. On 11i v2/v3 you can keep several named configurations side by side and boot whichever one you want, which is what makes safe experimentation possible.

The historical split matters because the tooling changed fundamentally:

- On **11i v1**, changing the kernel means editing a configuration description and *rebuilding* a new kernel binary under `/stand/build`, then swapping it in and rebooting. It is a compile-and-replace model.
- On **11i v2/v3**, the `kc*` toolset manages configurations declaratively. Many tunables and module states can change *dynamically* on the running kernel, and named configurations are first-class objects you can snapshot, compare, and boot. There is no manual rebuild step.

Understanding this shift explains why the two halves of this article look so different: v1 is about building a kernel, v2/v3 is about managing configurations.

## Tunable Parameter Types

- **Dynamic** tunables can be applied **without** a reboot.
- **Static** tunables require a **reboot** to take effect.

The distinction is not cosmetic. A **dynamic** tunable is one the kernel can re-read and act on while running — changing it takes effect immediately and is instantly reversible, so these are low-risk to adjust on a live system. A **static** tunable sizes a data structure or makes a decision that is fixed at boot; changing it only records the new value for the *next* boot, and the running kernel keeps the old value until you reboot. Some tunables are dynamic within a range but static beyond it. Always confirm a tunable's type before planning a change — `kctune` reports it — so you know whether the change needs a maintenance window.

A related trap: setting a static tunable does not fail or warn loudly, it simply queues the change. If you set a static tunable and expect immediate effect without rebooting, the system will behave as though nothing happened. Use `kconfig -D` / `kctune -D` to see pending next-boot changes so a queued change does not surprise you later.

## 11i v1: Rebuilding the Kernel

`/stand/system` is the kernel configuration file and can serve as a template for other systems. A newly built kernel has three components under `/stand/build`:

| File | Contents |
|------|----------|
| `/stand/build/vmunix_test` | Static kernel executable |
| `/stand/build/dlkm.vmunix_test` | Compiled DLKM modules for the new kernel |
| `/stand/build/system.SAM` | Text file listing the static drivers, subsystems, and parameters in the new kernel |

Backup kernel files use a `.prev` extension. This convention is your safety net on 11i v1: before a rebuild swaps in a new kernel, the working `/stand/system` and `/stand/vmunix` are preserved as `.prev` copies so you can boot the previous kernel if the new one panics or fails to boot. Never delete the `.prev` files until you have confirmed the new kernel boots and behaves correctly.

The build works by taking the text configuration in `/stand/system`, compiling a new static kernel and its DLKM (Dynamically Loadable Kernel Modules) alongside it under `/stand/build`, and then — via `kmupdate` — arranging for those freshly built files to become the running `/stand/vmunix` at the next boot. Until you reboot, the current kernel keeps running unchanged; the build is entirely staged.

### Applying a kernel manually

```bash
cd /stand
cp /stand/system /stand/system.prev
cp /stand/build/system.SAM /stand/system
kmupdate /stand/build/vmunix_test
shutdown -ry 0
```

### Booting a backup kernel (ISL prompt)

```
boot pri
hpux ls /stand
hpux -is /stand/vmunix.prev
```

### Moving the original kernel back into place

```bash
cd /stand
mv system.prev system
kmupdate /stand/vmunix.prev
shutdown -ry 0
```

## 11i v2 / v3: The kc* Toolset

Three interfaces manage the kernel configuration:

- **`kc*` CLI** — eight utilities that manage any kernel configuration (not just the current one)
- **`kcweb` TUI** — current configuration only (`kcweb -t`)
- **`kcweb` GUI** — current configuration only

The eight CLI utilities:

| Command | Purpose |
|---------|---------|
| `kconfig` | Manage whole kernel configurations |
| `kcpath` | Show the current kernel path |
| `kcmodule` | Manage kernel modules |
| `kctune` | Manage tunable parameters (e.g. `kctune nproc=4096`) |
| `kcusage` | Monitor resource usage |
| `kcalarm` | Set alarms on kernel tunables |
| `kclog` | View the kernel configuration log |

The design intent behind the `kc*` family is separation of concerns: `kconfig` operates on whole named configurations (save, load, list, compare), `kctune` operates on individual tunables, and `kcmodule` operates on individual modules. Because each tool can target *any* configuration — not just the running one — you can prepare and inspect an alternate configuration without ever touching the live kernel, then boot into it when ready. The `kcweb` TUI/GUI, by contrast, only ever acts on the *current* configuration, which is why the CLI is preferred for anything involving a staged or alternate config.

A recommended safety habit before any change: snapshot the current configuration under a name (`kconfig -s beforechanges`). If a tuning change destabilizes the system you can load or boot the snapshot to get back to a known-good state, and `kclog` records exactly what changed and when.

Logs live in `/var/adm/kc.log` and `/var/sam/log/samlog` (view with `/usr/sam/bin/samlog_viewer`).

### Modifying the current configuration

```bash
kconfig -s beforechanges     # snapshot the current config under a name first
kcmodule -h                  # help / hold module changes
kctune -h
kconfig -a                   # review the configuration
kconfig -D                   # check for pending (next-boot) changes
```

### Loading a named configuration

```bash
kconfig                      # list available configurations
kconfig -l myconfig          # load a named configuration
kconfig -D                   # check for pending changes
```

### Booting a named configuration

From **BCH** (PA-RISC), via the ISL prompt:

```
boot pri isl
hpux ls
hpux /stand/myconfiguration/vmunix
```

From **EFI** (Integrity):

```
ll
boot myconfiguration
```

## Kernel Modules

A module can be in one of five states:

| State | Meaning |
|-------|---------|
| `unused` | Not part of the kernel |
| `static` | Bound into the static kernel |
| `auto` | Loaded automatically when needed |
| `loaded` | Currently loaded (dynamic) |
| `best` | The recommended state for the module |

```bash
kcmodule                             # list all modules
kcmodule -v modulename               # view a module's state
kcmodule -h modulename               # hold the change for next boot
kcmodule -D                          # list pending module changes for next-boot kernel
```

## Tunables

```bash
kctune                               # list tunable parameters
kctune nproc=4096                    # set a tunable
kctune nproc                         # show one tunable, its value, and whether it is dynamic/static
kctune -v nproc                      # verbose: current value, default, and constraints
kctune -h nproc=4096                 # hold the change until next boot instead of applying now
kctune 'maxfiles+=1024'              # relative adjustment (add to current value)
kctune -D                            # show pending (next-boot) tunable changes
```

Tunables often have dependencies on one another — for example, process- and file-table sizes are related — so `kctune` may adjust or refuse a value that would violate a constraint. Read the message it prints rather than forcing a value. When a change is dynamic, `kctune` applies it immediately and logs it; when it is static, `kctune` tells you a reboot is required and stages the value. Setting a value equal to `default` restores the shipped default. Every change is recorded in `/var/adm/kc.log`, which `kclog` reads.

Common tuning gotchas:

- **Assuming a change took effect.** If a tunable is static, nothing changes until you reboot. Verify with `kctune -D` (pending) versus `kctune <name>` (current).
- **Tuning the running config when you meant to stage it.** Use `kctune -h` or target a saved configuration when preparing changes for a later reboot rather than altering the live kernel.
- **Ignoring dependent tunables.** Raising one limit without the related ones can leave the intended effect capped by a second parameter.

## Viewing the Kernel Log (kclog)

```bash
kclog 5                              # last 5 entries for the current kernel
kclog -c weekendConfig 5             # last 5 entries for a named configuration
kclog -f "Oracle" 5                  # last 5 entries containing a string
kclog -n "maxdsiz" 5                 # last 5 entries for a specific tunable or module
```

`kclog` is the audit trail for the kernel configuration: every tunable change, module state change, and configuration operation is recorded with a timestamp. When a system starts behaving differently after "someone changed something," `kclog` is the fastest way to find out what and when — filter by tunable name (`-n`), by free text (`-f`), or by configuration (`-c`).

## Monitoring Usage and Setting Alarms

Tuning is only meaningful if you can see how close a resource is to its limit. `kcusage` reports current consumption of tunable-governed resources against their configured maximums, so you can tell whether, say, the process table or semaphore pool is near exhaustion before it causes failures. `kcalarm` lets you register an alarm that fires when a resource crosses a threshold, turning reactive firefighting into proactive tuning.

```bash
kcusage                              # snapshot of resource usage vs. limits
kcusage nproc                        # usage for a specific tunable-governed resource
kcalarm                              # list configured kernel tunable alarms
kcalarm nproc                        # show alarms configured for a tunable
```

## Troubleshooting

- **A tunable change "did not work."** It was almost certainly a static tunable that only applies at the next boot. Confirm with `kctune -D` (pending changes) and reboot to apply.
- **`kctune` refuses a value or auto-adjusts it.** The value violated a constraint or a dependency on another tunable. Read the message, raise the dependent tunables too, and re-apply.
- **New kernel will not boot (11i v1) or a named config panics (v2/v3).** Boot the previous kernel or a known-good named configuration from the firmware prompt, then investigate. On v1 boot `/stand/vmunix.prev`; on v2/v3 boot a saved configuration. Restore with `kconfig -l <goodconfig>` once you are back up.
- **Not sure what changed or when.** Use `kclog` (kernel change log) and `samlog_viewer` to review the history before making further changes.
- **Which kernel/config is actually running?** `kcpath` shows the path to the running kernel and its configuration name; `kconfig` lists all configurations and marks the current one.

```bash
kcpath                               # path and name of the running kernel/config
kconfig                              # list configs; the running one is marked
kclog 20                             # recent kernel configuration history
```

## Command Reference

| Task | Command |
|------|---------|
| Set a tunable | `kctune <tunable>=<value>` |
| List modules | `kcmodule` |
| View module state | `kcmodule -v <module>` |
| Hold change for next boot | `kcmodule -h <module>` / `kctune -h` |
| Check pending changes | `kconfig -D` / `kcmodule -D` |
| Snapshot current config | `kconfig -s <name>` |
| Load a named config | `kconfig -l <name>` |
| Show kernel path | `kcpath` |
| Monitor resource usage | `kcusage` |
| Set a tunable alarm | `kcalarm` |
| View kernel log | `kclog <n>` |
| Rebuild kernel (11i v1) | `kmupdate /stand/build/vmunix_test` |

## Related Articles

- [HP-UX Boot Process](articles/hpux-boot-process.md)
- [HP-UX Disaster Recovery (DRD and Ignite-UX)](articles/hpux-disaster-recovery.md)
- [HP-UX Patch Management](articles/hpux-patch-management.md)
- [HP-UX Performance Monitoring](articles/hpux-performance-monitoring.md)
- [HP-UX Crash Dump Analysis](articles/hpux-crash-dump-analysis.md)
- [HP-UX Swap Management](articles/hpux-swap-management.md)
- [HP-UX System Information](articles/hpux-system-information.md)


