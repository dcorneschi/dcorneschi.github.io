# HP-UX Software Distribution (SD-UX): Depots and swinstall

Managing software and patches on HP-UX with Software Distributor (SD-UX) — building and registering depot servers, and the `sw*` command set (`swcopy`, `swinstall`, `swremove`, `swlist`, `swreg`, `swverify`) for pulling and pushing software. Covers HP-UX 11i v1–v3.

## Concepts: How SD-UX Sees Software

Software Distributor organizes everything as a hierarchy of **objects**, and every `sw*` command operates on that hierarchy:

- A **fileset** is the smallest installable unit — a group of files delivered and patched together.
- A **product** is a collection of related filesets (for example a database engine and its libraries).
- A **bundle** is a convenience grouping that references products/filesets (an Operating Environment is a bundle).
- A **depot** is a container of software objects, either a **directory depot** (a filesystem tree, the usual case) or a **media depot** (a mounted CD/DVD or a tape).

Two more pieces of state make SD-UX behave predictably:

- The **Installed Products Database (IPD)** under `/var/adm/sw/` records what is installed on a system. `swlist` without `-s`/`-d` reads the IPD; `swinstall`/`swremove` update it. If the IPD and the real files diverge, `swverify` is how you detect it.
- Every command accepts a **software selection** (`product`, `product.fileset`, or `*`) and, after `@`, a **target** (`host:/path` for a depot, or a hostname for a running system). Reading `swcopy -s /dvd FooProd @ /mydepot` as "copy selection *from* source *to* target" makes the whole command set consistent.

The `-x option=value` syntax sets **options** that modify behavior (dependency enforcement, autoreboot, patch matching). These are the same across commands, which is why the same `-x enforce_dependencies=false` works for `swcopy`, `swinstall`, and `swremove`.

## Why a Depot Server

Instead of juggling DVDs, CDs, and tapes, a central **SD-UX depot server** lets clients `swinstall` software and patches over the network:

- One point of administration for software and patch updates; installs systems with no DVD/tape drive.
- Ensures a consistent software/patch image across all hosts.
- Enables remote install/management: clients **pull**, and (11i+) the server can **push** to targets.
- When you select a product, `swinstall` **auto-selects** its required products/dependencies from the depot.
- If the depot contains patches for a selected product, `swinstall` installs those patches at the same time — cutting downtime.

Depots are commonly structured by release and content type:

```
/depots/Rel_B.11.31.1009/core_dcoe
/depots/Rel_B.11.31.1009/appl
/depots/Rel_B.11.31.1103/core_dcoe
/depots/Rel_B.11.31.1103/appl
/depots/Rel_B.11.31/other
```

## Adding Software to a Depot (swcopy)

`swcopy` copies software from a source depot (DVD, tape, or another directory depot) into a directory depot; it auto-registers the target depot.

```bash
mkdir /mydepot

# Copy one product from a DVD depot to a directory depot
swcopy -s /dvd FooProd @ /mydepot

# Copy ALL software from a DVD depot
swcopy -s /dvd '*' @ /mydepot
swcopy -s /dvdrom '*' @ /var/depot2

# Copy all software from a local DVD to a depot on another host
swcopy -s /dvdrom '*' @ hp02:/var/depot1

# From a tape depot
swcopy -s /dev/rtape/tape0_BEST '*' @ /mydepot

# From one directory depot to another
swcopy -s /myolddepot '*' @ /mydepot
```

`swcopy` reproduces the software's catalog/metadata in the target depot as well as the file payload, which is what lets the target act as a fully functional source for later `swinstall` operations. Because it registers the target automatically, a directory depot becomes network-visible the moment the first product lands in it.

Install a product and auto-select its matching patches:

```bash
swinstall -s svr:/mydepot -x autoselect_patches=true -x autoreboot=true FooProd
```

If the auto-selected patches have dependencies, `swinstall` selects those dependents too.

> **Gotcha:** When copying "everything" from media, quote the wildcard — `'*'` — so the shell does not expand it against your current directory before `swcopy` ever sees it. This applies to every `sw*` command that takes a selection.

## Adding Patches to a Depot

When building a **patch depot** from many single-patch `.depot` files, disable dependency checking so `swcopy` doesn't refuse patches whose dependencies live in other files:

```bash
# Add a patch to a depot (skip dependency enforcement)
swcopy -s /tmp/PHCO_1000.depot -x enforce_dependencies=false '*' @ /mydepot

# Copy all patches from an 11i v1 Support Plus (GOLDBASE) bundle
swcopy -s /cd/GOLDBASE11i -x enforce_dependencies=false '*' @ /mydepot

# Patch a whole system from the depot (only patches matching installed products)
swinstall -s svr:/mydepot -x patch_match_target=true -x autoreboot=true
```

By default `swcopy` won't copy a patch unless its dependencies (documented in the patch's `.text` file) can be resolved in the depot — hence `-x enforce_dependencies=false` when staging patches from separate `.depot` files.

The reasoning: dependency *enforcement* is appropriate at **install** time (you don't want to install a patch whose prerequisites are missing on the target) but counterproductive at **staging** time, when you are still assembling the collection and the prerequisites simply haven't been copied yet. So the common pattern is: disable enforcement while building the patch depot, then let `swinstall` enforce dependencies normally when clients pull from it.

HP-UX patch names encode their type: `PHCO_` = command/library patch, `PHKL_` = kernel patch, `PHNE_` = networking patch, `PHSS_` = subsystem/other. A `PHKL_` kernel patch relinks the kernel and therefore needs a reboot — which is why `autoreboot=true` is common when patching. `patch_match_target=true` tells `swinstall` to select only the patches in the depot that correspond to products already installed on the target, so you can point a system at a large patch depot and pull just what applies. For a deeper treatment see [HP-UX Patch Management](articles/hpux-patch-management.md).

## Removing Software from a Depot (swremove)

```bash
# Remove one product from a depot
swremove -d FooProd @ /mydepot

# Remove all products from a depot (and the depot itself)
swremove -d '*' @ /mydepot
rm -rf /mydepot

# Remove all software in a remote depot and unregister it
swremove -d '*' @ hp02:/var/depot
```

When other products/patches depend on what you're removing, two options control behavior:

| Option | Default | Effect |
|--------|---------|--------|
| `-x enforce_dependencies=true\|false` | `true` | Whether removal is allowed when the item is required by others |
| `-x autoselect_dependents=true\|false` | `false` | Whether to also select the item's dependents for removal |

## Listing Software (swlist)

```bash
# Depots registered on the local system
swlist -l depot

# Depots available on a remote host
swlist -l depot @ sanfran

# Software in a specific depot
swlist -s sanfran:/mydepot

# Products in a local directory depot
swlist -l product -d @ /var/depot

# Products in a remote depot
swlist -l product -s sanfran:/mydepot
swlist -l product -d @ hp02:/var/depot
```

## Registering / Unregistering a Depot (swreg)

A depot must be **registered** to be visible to network clients. `swcopy` auto-registers a directory depot; use `swreg` to register a mounted CD/DVD depot manually.

```bash
# Register (e.g. a mounted CDROM depot)
swreg -l depot @ /cdrom
swlist -l depot

# Unregister
swreg -u -l depot @ /cdrom
swlist -l depot
```

- Copying software to a directory depot auto-registers it; removing the last product auto-unregisters it.
- You can always install from a **local** depot even if it isn't registered.

## Verifying a Depot (swverify)

```bash
swverify -d '*' @ /var/depot        # verify all software in a local depot
swverify '*'                        # verify installed software against the IPD
swverify FooProd                    # verify one installed product
```

`swverify` cross-checks files, permissions, and dependencies against the catalog. Run it against a **depot** (`-d`) to confirm the depot is complete and consistent before you rely on it, and against **installed** software (no `-d`) to detect files that were modified, removed, or whose dependencies are unsatisfied. It is the standard first diagnostic after a failed or interrupted install.

### Where SD-UX Logs Go

Every `sw*` command writes a detailed log, and reading it is usually faster than guessing why something failed:

| Log | Contents |
|-----|----------|
| `/var/adm/sw/swinstall.log` | swinstall/swcopy sessions: selections, analysis, errors |
| `/var/adm/sw/swremove.log` | swremove sessions |
| `/var/adm/sw/swagent.log` | The per-host agent's view (both local and remote/push operations) |

The agent log on a **target** is the place to look when a push or a remote pull fails, since the client-side agent does the actual work.

## Pulling Software (client → depot)

Clients install just like from a CD — point `-s` at `server:/depotpath`:

```bash
swinstall -s svr:/mydepot -x autoreboot=true FooProd
```

`swinstall` runs in two phases: an **analysis** phase that checks disk space, dependencies, and compatibility without changing anything, followed by the actual **install/configure** phase. Because analysis is non-destructive, you can preview an install with `-p` (preview) to surface problems before committing:

```bash
swinstall -p -s svr:/mydepot FooProd            # analysis only, no changes
swinstall -s svr:/mydepot -x autoreboot=true FooProd
```

After analyzing requirements and auto-selecting dependencies/patches, `swinstall` installs and configures the software. If the product contains a **kernel fileset**, it reboots automatically (hence `autoreboot=true`). Without `autoreboot=true`, `swinstall` will refuse to proceed with software that requires a reboot rather than leaving the system in a half-configured state — so for kernel-affecting products either set the option or schedule the install for a maintenance window.

## Pushing Software (depot → targets, 11i+)

The 11i "push" feature installs/updates software from the depot server out to one or more remote targets at once. It needs setup on both sides.

On the depot server (only if using the swinstall GUI):

```bash
svr# touch /var/adm/sw/.sdkey
```

On each target client — explicitly allow the server to push:

```bash
tgt# /usr/lbin/sw/setaccess svrname
tgt# swacl -l root
```

Then push install / list / remove:

```bash
svr# swinstall -s svr:/mydepot FooProd @ tgt1 tgt2
svr# swlist @ tgt1 tgt2
svr# swremove FooProd @ tgt1 tgt2
```

**Push limitations:**

- You cannot push an HP-UX **OS update** to remote systems.
- `-x patch_match_target` works with push, but only to **one** target at a time.
- Not supported by push: `update-ux`, `install-sd`, `swpackage`, `swmodify`.
- Push only from an **11i** depot server, though targets can be 11.00, 11i v1, or 11i v2.

## Making a Depot from a DVD (make_depots)

```bash
make_depots -r B.11.31 -s /dev/dsk/c0t0d0
ls -l /var/opt/ignite/depots/Rel_B.11.31/core
```

## Checking Installed Patches

```bash
what /stand/vmunix        # show patch/version strings compiled into the kernel
swlist -l patch           # list installed patches
```

## Command Reference

| Task | Command |
|------|---------|
| Copy software into a depot | `swcopy -s <src> '*' @ <depot>` |
| Stage patches (no dep check) | `swcopy -s <patch> -x enforce_dependencies=false '*' @ <depot>` |
| Install (pull) | `swinstall -s <svr>:/<depot> [-x autoreboot=true] <product>` |
| Install + matching patches | `swinstall -s ... -x autoselect_patches=true` |
| Patch a system | `swinstall -s ... -x patch_match_target=true` |
| Remove from depot | `swremove -d <product> @ <depot>` |
| List local depots | `swlist -l depot` |
| List software in a depot | `swlist -s <svr>:/<depot>` |
| Register / unregister | `swreg -l depot @ <path>` / `swreg -u -l depot @ <path>` |
| Verify a depot | `swverify -d '*' @ <depot>` |
| Push to targets | `swinstall -s <svr>:/<depot> <product> @ tgt1 tgt2` |
| Make depot from DVD | `make_depots -r <rel> -s <device>` |
| Installed patches | `swlist -l patch`, `what /stand/vmunix` |
| Preview an install (analysis only) | `swinstall -p -s <svr>:/<depot> <product>` |
| Verify installed software | `swverify '*'` |
| Verify a depot | `swverify -d '*' @ <depot>` |

## Troubleshooting

**`swcopy` refuses a patch citing unresolved dependencies.** You are staging patches from separate `.depot` files whose prerequisites aren't in the depot yet. Add `-x enforce_dependencies=false` while building the depot; let `swinstall` enforce dependencies at install time.

**`swinstall` fails analysis with a dependency or space error.** Analysis is non-destructive, so nothing was changed. Read `/var/adm/sw/swinstall.log` for the specific selection that failed, resolve it (add the missing product/patch to the depot, or free disk space), and re-run.

**A client can't see a network depot.** The depot must be registered (`swlist -l depot @ server` from the client). `swcopy` auto-registers directory depots, but a mounted CD/DVD depot needs a manual `swreg -l depot @ /path`. Also confirm the SD daemon/agent is reachable and the target granted access for push operations.

**A push to a target is rejected.** The target must explicitly allow the server (`/usr/lbin/sw/setaccess svrname` and the appropriate `swacl` entry). Check `/var/adm/sw/swagent.log` on the *target*, not the server. Remember push cannot deliver an OS update and `patch_match_target` works only one target at a time.

**Installed software behaves oddly or files seem missing.** Run `swverify` against the installed selection to compare files/permissions/dependencies with the catalog, then reinstall the affected fileset if it reports discrepancies.

## Related Articles

- [HP-UX swlist and Software Structure](articles/hpux-swlist-software-structure.md)
- [HP-UX Patch Management](articles/hpux-patch-management.md)
- [HP-UX Installation with Ignite-UX](articles/hpux-installation-ignite.md)
- [HP-UX System Information and Initial Configuration](articles/hpux-system-information.md)
- [HP-UX Startup, Run Levels, and Network Services](articles/hpux-startup-and-services.md)
- [HP-UX Kernel Configuration](articles/hpux-kernel-configuration.md)
