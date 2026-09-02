# HP-UX SD-UX Software Structure, IPD, and swlist

The Software Distributor (SD-UX) object model on HP-UX — how software is organized into filesets, subproducts, products, and bundles; the Installed Product Database (IPD); the `swagentd` daemon; and using `swlist`/`swinstall`/`swremove` to list, install, and remove software. For building and serving depots, see [SD-UX Depots and swinstall](articles/hpux-software-depots-swinstall.md).

## SD-UX Software Object Model

SD-UX organizes software in a hierarchy — from smallest to largest:

| Object | Definition |
|--------|-----------|
| **Fileset** | A group of related files and control scripts installed/managed as a unit — the **smallest** manageable SD-UX object |
| **Subproduct** | A group of related filesets within a product (the same fileset can belong to more than one subproduct) |
| **Product** | A collection of subproducts and filesets; SD-UX commands keep a product focus but let you target subproducts/filesets |
| **Bundle** | A collection of filesets, possibly from several products, managed as a single entity |

Most administrators install and manage software at the **bundle** level. Software in a **directory depot** is stored under a normal filesystem directory (default `/var/spool/sw`).

### Why the Hierarchy Matters

The object model isn't just cataloging — it determines *what you can select* and *how much you get*. When you name a bundle to `swinstall`, SD-UX expands it into every fileset the bundle references, pulls in each fileset's control scripts, and honors the dependencies declared between filesets. This is why bundles are the normal unit of work: HP ships a patch bundle or an application as a single named object, and installing that one name delivers a tested, self-consistent set of filesets rather than leaving you to hand-pick pieces.

A few properties of the model are worth internalizing:

- A **fileset** is the atomic unit of *state*. The IPD tracks configured/installed status per fileset, and dependencies (`prerequisite`, `corequisite`) are declared between filesets. When something is "half installed," it is almost always at the fileset level.
- **Subproducts** are just convenient groupings; the same fileset can appear in several subproducts, so removing a subproduct does not necessarily remove a fileset another subproduct still needs.
- A **bundle** is a *reference container*. Removing a bundle definition doesn't automatically remove the underlying filesets unless they're not referenced elsewhere — a frequent source of "I removed it but it's still there" confusion.

### Software Specification and Versioning

SD-UX identifies software with a **software specification** of the form `product[.subproduct][.fileset],r=<revision>,a=<arch>`. You can target any level precisely, for example `OS-Core.CORE-SHLIBS` to act on a single fileset within a product. Every fileset also carries a *revision* and *architecture*, which is how multiple versions can be tracked and how `swinstall` decides whether an install is an upgrade, a downgrade, or a no-op. When a command seems to "do nothing," it's usually because the requested revision already matches what the IPD records.

## The Installed Product Database (IPD)

The **IPD** records everything installed via SD-UX (under `/var/adm/sw/products`):

- `swinstall` **adds** a record to the IPD.
- `swremove` **removes** a record from the IPD.
- `swlist` **displays** the contents of the IPD.

The IPD is the *authoritative source of truth* for what SD-UX believes is installed — it is metadata, not the files themselves. This distinction matters: if someone deletes files by hand (`rm`) instead of using `swremove`, the IPD still lists the software as installed even though the files are gone, and a later `swinstall` of the same version may skip it because the database says it's already there. Conversely, `swverify` compares the IPD's expectations against the actual filesystem to catch exactly this kind of drift.

```bash
swverify OS-Core            # verify a product's files, scripts, and dependencies
swverify \*                 # verify everything recorded in the IPD
```

Because the IPD is critical and rebuilt only with difficulty, treat `/var/adm/sw` as something to back up. Do **not** hand-edit files under `/var/adm/sw/products`; let the SD-UX commands maintain it. If SD-UX operations start failing with lock or consistency errors, that usually points to a crashed agent or a stale lock rather than a corrupt database (see the daemon and troubleshooting notes below).

## The swagentd Daemon

SD-UX operations are serviced by `swagentd`, which starts automatically at **run level 2** (so SD-UX is not available in single-user mode):

```bash
/sbin/init.d/swagentd start
ps -ef | grep swagentd
/usr/sbin/swagentd -r          # restart (re-read config)
```

`swagentd` is a *registration and control* daemon: when you run a command like `swinstall`, the command talks to `swagentd`, which spawns a short-lived `swagent` process to do the actual work — locally or on a remote target. For remote operations (installing from or to another host), `swagentd` must be running and reachable on **both** ends. Its behavior is governed by policy defaults in `/var/adm/sw/defaults` (system) and `~/.sw/defaults` (per-user); the same options you pass with `-x` on the command line can be set there as persistent defaults.

Because SD-UX serializes access with a lock, only one modifying operation can run at a time. If a command aborts uncleanly you may see errors about an existing session or an unavailable agent; restarting the daemon usually clears a stale state:

```bash
/sbin/init.d/swagentd stop
/sbin/init.d/swagentd start
```

## Listing Software (swlist)

```bash
# By object level, on the local host
swlist -l bundle
swlist -l product
swlist -l fileset

# On another host
swlist -l bundle @ otherhost
swlist -l product @ otherhost

# Depots
swlist -l depot                       # depots on the local host
swlist -l depot @ server              # depots on a depot server
swlist -l product -d @ server:/depot  # products in a specific depot
swlist -d @ /dvd                      # software on mounted media

# Interactive / GUI
swlist -i                             # interactively list installed software
swlist -i -d                          # GUI mode for a software depot
```

Patches and categories:

```bash
swlist -l patch                       # all applied patches
swlist -l category                    # available patch categories
swlist -s /dvdrom | grep Ignite-UX    # search a source for a product
```

Useful attribute and file queries:

```bash
swlist -a readme OS-Core                        # a product's README
swlist -d -a is_protected @ /SD_CDROM           # does CD software need a codeword?
swlist -l file | grep /bin/ls                   # which product owns a file
swlist -d @ /var/spool/sw                       # software in a local depot
swlist -d @ server1:/var/spool/sw               # software in a remote depot
```

### Reading swlist Output and Verbosity

`swlist` is the primary *inspection* tool, and its two most important switches are `-l` (which object level to display) and `-a` (which attribute to show). Combined, they let you extract almost any fact SD-UX knows:

```bash
swlist -l fileset -a state OS-Core              # per-fileset installed/configured state
swlist -l product -a revision                   # revision of each installed product
swlist -v OS-Core                               # verbose: all attributes for a product
swlist -l bundle -a title                       # human-readable bundle titles
```

The `state` attribute is especially useful for spotting trouble: a fileset should be `configured`. A fileset stuck in `installed` (but not configured) or `transient` usually means a control script failed partway through — rerun `swconfig`, or verify with `swverify` to see what SD-UX expects versus what's present.

A common gotcha: `swlist` with no `-d` inspects the **IPD** (what's installed on the host), while adding `-d @ <path-or-host:path>` inspects a **depot** (what's *available* to install). Mixing these up is why "I can't find the software" happens — you may be listing the wrong catalog. Point `-d` at the source when you want to know what a depot contains, and drop `-d` when you want to know what the running system has.

## Installing and Updating Software (swinstall)

After you specify a source, `swinstall` lists the available software — by default all bundles plus products not already in those bundles. It **won't reinstall** a fileset if the same version is already present.

`swinstall` runs in phases: it *selects* the requested software, *analyzes* (checks disk space, dependencies, and conflicts against the IPD), *loads* the files, and finally *configures* them by running each fileset's configure script. The analysis phase is where most installs stop safely — if a prerequisite is missing or disk space is short, SD-UX reports it and makes no changes. Reading that analysis output before confirming is the single best habit for avoiding half-finished installs.

Two options come up constantly:

- `-x autoreboot=true` lets SD-UX reboot automatically when a selected fileset is marked as requiring a reboot (typically kernel filesets). Without it, such filesets are loaded but the system waits for you to reboot to finish configuration.
- `-p` (preview) performs selection and analysis **without** loading anything — the safest way to see exactly what an install would do:

```bash
swinstall -p -s svr:/depot mybundle             # preview only, no changes
swinstall -x logfile=/tmp/swi.log -s svr:/depot mybundle
```

If an install is interrupted, rerunning the same command is generally safe — SD-UX resumes based on the IPD and skips filesets already at the target revision.

```bash
swinstall -s /dev/rmt/0m                         # local tape depot
swinstall -s /var/spool/sw                       # local directory depot
swinstall -s /cdrom                              # CD mounted at /cdrom
swinstall -s depothost:/depotpath                # network depot
swinstall -s /tmp/myapp.depot                    # a depot file

# Install a specific product/bundle, auto-reboot if it has a kernel fileset
swinstall -s /labs/depots/echoapp.depot -x autoreboot=true EchoApp
swinstall -s svr:/depot -x autoreboot=true mybundle

# Update everything already installed to the depot's newer versions
swinstall -s svr:/depot -x autoreboot=true -x match_target=true
```

Mounting media and installing from it:

```bash
ioscan -fnkC disk | more                 # find the CD/DVD device
mount /dev/dsk/c1t2d0 /dvdrom            # mount it
swinstall -s /dvdrom Ignite-UX          # install a product from it
```

## Removing Software (swremove)

```bash
swremove EchoApp                          # remove a product/bundle
swremove -x autoreboot=true mybundle      # reboot after (kernel filesets)

# Remove applied patches
swremove -d PHCO_45879
swremove -d PHKL_45879 @ server1:/depots/     # from a remote depot
```

## Copying Patches to a Depot

`swcopy` takes the source distribution with `-s` and copies the named selection into the target depot after `@`:

```bash
swcopy -s /tmp/PHKL_14235.depot PHKL_14235 @ /var/spool/sw
```

## Troubleshooting

Start with the logs — every SD-UX command writes a detailed session log, and the last "ERROR"/"WARNING" lines usually name the exact fileset and reason:

```bash
tail -50 /var/adm/sw/swinstall.log       # last install session
tail -50 /var/adm/sw/swagent.log         # agent-side detail
tail -50 /var/adm/sw/swremove.log        # last remove session
```

Common situations and how to approach them:

- **"Another agent/session is active" or the command hangs at start.** A previous operation left a lock or a stuck `swagent`. Confirm nothing legitimate is running, then bounce the daemon (`/sbin/init.d/swagentd stop; start`).
- **A fileset shows a non-`configured` state.** Configuration didn't complete. Re-run configuration explicitly, or verify to see the discrepancy:

```bash
swconfig OS-Core                         # (re)run configure scripts
swverify -x check_scripts=true OS-Core   # verify files, attributes, and scripts
```

- **Install "does nothing."** The requested revision already matches the IPD. Use `swlist -l fileset -a revision` on both the target and the depot to compare, and add `-x reinstall=true` only when you truly need to force it.
- **Dependency failures during analysis.** Read which fileset is required; install the prerequisite (often from the same depot) first, or select both together so SD-UX can order them.

## Key Files

| File | Purpose |
|------|---------|
| `/var/spool/sw` | Default directory-depot location |
| `/var/adm/sw/products` | Installed Product Database (IPD) |
| `/var/adm/sw/swinstall.log` | swinstall log |
| `/var/adm/sw/swagent.log` | swagent (agent-side) log |
| `/var/adm/sw/swremove.log` | swremove log |
| `/var/adm/sw/defaults` | System-wide SD-UX default options |
| `/var/adm/sw/.codewords` | Stored codewords for protected software |

## Command Reference

| Task | Command |
|------|---------|
| Start/restart the agent | `/sbin/init.d/swagentd start` / `swagentd -r` |
| List installed (by level) | `swlist -l bundle\|product\|fileset` |
| List on another host | `swlist -l product @ host` |
| List a depot's contents | `swlist -l product -d @ server:/depot` |
| List applied patches | `swlist -l patch` |
| Which product owns a file | `swlist -l file \| grep <path>` |
| View a README | `swlist -a readme <product>` |
| Install from a source | `swinstall -s <source> [<bundle>]` |
| Update to depot versions | `swinstall -s <depot> -x match_target=true` |
| Remove software | `swremove [<product>]` (`-x autoreboot=true`) |
| Remove a patch | `swremove -d <patch>` |
| Verify installed software | `swverify <product>` / `swverify \*` |
| (Re)configure a fileset | `swconfig <product>` |
| Preview an install | `swinstall -p -s <source> <bundle>` |
| IPD location | `/var/adm/sw/products` |

## Related Articles

- [HP-UX Software Depots and swinstall](articles/hpux-software-depots-swinstall.md) — building and serving the depots that `swlist`/`swinstall` read from
- [HP-UX Patch Management](articles/hpux-patch-management.md) — patches and patch bundles are SD-UX objects managed with these same tools
- [HP-UX Installation with Ignite-UX](articles/hpux-installation-ignite.md) — Ignite uses SD-UX depots to lay down the OS and applications
- [HP-UX System Information](articles/hpux-system-information.md) — cross-referencing installed software with system and OS details
