# HP-UX Patch Management

Managing HP-UX patches with SD-UX and Software Assistant (SWA) — patch naming and categories, supersession chains, ratings, where patches come from, installing from the HP Support Center / DVD / depot server, listing, removing and committing patches, patch states, category tags and ancestry, cleanup, and verifying the standard Quality Pack bundles. Complements [HP-UX Software Depots and swinstall](articles/hpux-software-depots-swinstall.md).

## Concepts: How HP-UX Patching Works

HP-UX patch management is built entirely on top of **SD-UX** (Software Distribution for UNIX), the same packaging system used for products and depots. A patch is not a special file format — it is an ordinary SD-UX fileset that *replaces* files belonging to a base product, with extra metadata (its ancestry, category, and supersession relationships) that SD-UX uses to reason about install order and removability. Understanding that a patch is "just software that knows what it modifies" explains most of the behaviour below: why patches can be rolled back, why superseded patches linger, and why removing a base product also removes its patches.

Three ideas underpin everything else:

- **Ancestry** — every patch records the original component (its *ancestor*) that it modifies. SD-UX uses ancestry to decide whether a patch even applies to what you have installed.
- **Supersession** — patches are cumulative, so a newer patch declares the older ones it replaces. Only the newest in a chain is active; the rest stay on disk but inactive.
- **State** — a patch is `applied`, `superseded`, `committed`, or `committed/superseded`. State determines whether it can be rolled back and whether its saved original files still exist.

The single most important operational consequence is the rollback mechanism: when `swinstall` applies a patch it copies the files it is about to overwrite into `/var/adm/sw/save`. That saved copy is what lets you `swremove` the patch later and return to the previous behaviour. Committing a patch throws those saved files away to reclaim space — which is why a committed patch can no longer be removed.

## Patch Naming

Patches use the form `PHxx_yyyyy`:

- `P` — Patch
- `H` — HP-UX
- `xx` — the area patched (see below)
- `yyyyy` — a unique four- or five-digit number; higher numbers are generally more recent

| Code | Area | Reboot? |
|------|------|---------|
| `CO` | General commands, libraries, man pages | Usually no |
| `KL` | Kernel and related structures | Almost always |
| `NE` | Network components | Maybe |
| `SS` | Other subsystems/applications (X11, CDE, OpenView) | Maybe |

The two-letter area code is a quick reality check before you schedule an install. A `PHKL` (kernel) patch almost always requires a reboot because it changes the static kernel or a driver bound into it, so plan a maintenance window. `PHCO` command patches usually replace user-space binaries and libraries and rarely need a reboot. `PHNE` and `PHSS` land in the "maybe" category — a networking patch that touches a kernel networking module needs a reboot, while one that only replaces a daemon does not. Do not rely on the code alone: the patch's own readme (its `.text` file) states definitively whether a reboot is required, and `swinstall` will tell you when it stages a change that needs one.

Each patch is packaged as a `shar` (shell archive) — a self-extracting shell script. Running it with `sh` unpacks it into a `.depot` file (the actual SD-UX software) and a `.text` file (the readme). Always read the `.text` file before installing: it lists the reboot requirement, dependencies, defects fixed, and any special instructions.

## SD-UX Locations

| Path | Purpose |
|------|---------|
| `/var/adm/sw/` | Default source and target software depot |
| `/var/adm/sw/.codewords` | Stored software codewords |
| `/var/adm/sw/products` | Installed Product Database (IPD) — the SD-UX database |
| `/var/adm/swinstall.log` | `swinstall` log |
| `/var/adm/swremove.log` | `swremove` log |
| `/var/adm/sw/save` | Original pre-patch files saved for rollback |

## Patch Supersession

HP patches are usually **cumulative**. A newer patch may **supersede** older ones, and must include all functionality from its predecessors. A series of patches, each replacing the previous one, forms a **supersession chain** (patch numbers generally increase along the chain).

- Only the final patch in the chain is in the **active/applied** state; superseded patches remain on the system but inactive.
- You don't need to install every patch in a chain — installing the superseding patch is enough.
- The presence of a superseding patch **prevents** installation of any predecessor.

Why do superseded patches stay on the system instead of being deleted? Because they anchor the rollback path. Each patch in the chain saved the files *it* replaced, so to fully unwind to the original component you may need to remove the patches in reverse order. SD-UX enforces this: a superseded patch cannot be rolled back on its own — you must remove the superseding patch first. Over months of patching this leaves a build-up of inactive, superseded patches and their saved rollback files under `/var/adm/sw/save`, which is exactly what the `cleanup` command exists to prune.

A practical implication of cumulativeness: when HP tells you a defect is fixed in `PHKL_43210`, you install *that* patch and it brings along every fix from the patches it superseded. You never chase and install a long chain by hand — the newest patch is the whole chain.

## Patch Ratings

| Rating | Meaning |
|--------|---------|
| `*` | HP tested it; no unwanted effects found |
| `**` | Installed in a reasonable number of customer environments with no problems reported |
| `***` | Stress- and performance-tested by HP |

## Patch Sources

- **HP Support Center (HPSC) patch database** — online database of all available HP-UX patches
- **HP-UX media kit patch bundles** — critical, tested bundles on the DVDs
- **Software Pack patch bundles** — bundles of enhancement patches
- **HP Enterprise Response Center patch media** — custom media for some support-contract customers
- **Local SD-UX depot server** — locally managed depot of patches approved for your environment

## Patch Tools

- **SD-UX utilities** — `swinstall`, `swlist`, `swremove`, `swcopy`
- **HPSC patch database search engine** — web utility to search and download patches
- **Software Assistant (SWA)** — CLI tool that analyzes a system and recommends/downloads security and Quality Pack patches
- **HP Patch Assessment Tool** — web utility that analyzes a system and recommends/downloads a variety of patches and bundles

## Installing Patches

### A single patch from the HPSC

```bash
gzip -d /tmp/patches.tgz
tar -xvf /tmp/patches.tar
sh /tmp/PHCO_1000                 # unpacks into a .text and a .depot file
more /tmp/PHCO_1000.text          # read the patch notes
swinstall -s /tmp/PHCO_1000.depot -x autoreboot=true -x patch_match_target=true
```

### Multiple patches from the HPSC

```bash
gzip -d /tmp/patches.tgz
tar -xvf /tmp/patches.tar
cd /tmp
./create_depot_hp-ux_11           # bundle the patches into a single depot
swlist -a readme -s /tmp/depot | more
swinstall -s /tmp/depot -x autoreboot=true -x patch_match_target=true
```

`patch_match_target=true` tells `swinstall` to select only the patches that apply to software already installed on the target. This is the option that makes bulk patch installs safe and painless: point `swinstall` at a large depot and let it work out which patches are relevant to *this* system rather than trying to install everything. Combined with `autoreboot=true`, it lets a mixed depot (some patches needing a reboot, some not) install and reboot in a single unattended pass.

A few options worth knowing when installing patches:

```bash
swinstall -p -s /tmp/depot -x patch_match_target=true   # -p = preview: analyze only, install nothing
swinstall -s /tmp/depot -x autoreboot=true -x patch_match_target=true -x mount_all_filesystems=false
swinstall -s /tmp/depot -x patch_match_target=true -x reinstall=false   # skip already-installed patches
```

Always run a `-p` (preview) pass first on production systems. Preview reports dependency problems, disk-space shortfalls, and exactly which patches would be selected, without changing anything — a cheap way to avoid a failed install mid-window.

### From DVD

```bash
ioscan -funC disk
mkdir /dvd
mount /dev/disk/diskx /dvd
ls /dvd
swlist -a readme -s /dvd/GOLDQPK | more    # 11i v1
swlist -a readme -s /dvd | more            # 11i v2/v3
swinstall -s /dvd -x autoreboot=true -x patch_match_target=true
swinstall -s /dvd -x autoreboot=true PHCO_1000 PHCO_2000   # specific patches
```

### From a depot server

```bash
swlist -a readme -s svrname:/depotpath
swinstall -s svrname:/depotpath -x autoreboot=true -x patch_match_target=true
swinstall -s svrname:/depotpath -x autoreboot=true PHCO_1000 PHCO_2000
```

If a depot contains both products and patches, SD-UX auto-selects the patches too. To install a **product without** its patches:

```bash
swinstall -s svrname:/depotpath -x autoreboot=true -x autoselect_patches=false FooProd
```

## Listing Patches

```bash
swlist -l patch                   # all installed individual patches
swlist -l patch LVM               # patches and their associated filesets
swlist -l product "PH*"           # patches only
```

## Removing and Committing Patches

`swinstall` saves the original pre-patch files under `/var/adm/sw/save`, which is what makes rollback possible.

```bash
swremove -x autoreboot=true PHCO_1000
```

If many patches accumulate, `/var/adm/sw/save/` can grow and fill `/var`. A patch you're sure you'll never remove can be **committed** to reclaim that space:

```bash
swmodify -x patch_commit=true PHCO_1000                                    # commit an installed patch
swinstall -x autoreboot=true -x patch_match_target=true -x patch_save_files=false   # commit during install
```

Once committed, a patch **cannot** be removed unless you remove its associated product. HP does not recommend committing patches.

### Rollback / removal rules

- Removing a base product fileset with `swremove` removes **all** patches applied to it.
- Rollback files are removed when their base product fileset is removed.
- A **superseded** patch cannot be rolled back unless the superseding patch is rolled back too.
- **Committed** patches cannot be removed unless the base product is also removed.

## Patch States

An installed patch is in one of four states:

| State | Meaning |
|-------|---------|
| **applied** | Currently active on the system |
| **committed** | Can't be removed — the files it replaced are no longer saved |
| **superseded** | Replaced by another installed patch |
| **committed/superseded** | Both committed and superseded |

```bash
show_patches                                       # formatted patch listing
check_patches                                      # report issues -> /tmp/check_patches.report
swlist -l fileset -a patch_state *,c=patch         # current state of installed patches
```

## Category Tags and Ancestry

A **category tag** classifies a patch — e.g. `hardware enablement`, `enhancement`, `special release`, `critical`, `firmware`. It's shown on the patch detail page / readme.

```bash
swlist -l fileset -a category_tag *,c=patch
```

A patch's **ancestor** is the original software component it modified; ancestry affects install and removal.

```bash
swlist -l fileset -a ancestor *,c=patch
```

## Cleanup

`cleanup` commits superseded patches and removes their saved rollback files, logging to `/var/adm/cleanup.log`. The `-c` value is how many times a patch must have been superseded to qualify.

```bash
cleanup -c 1        # commit patches superseded at least once
cleanup -p          # preview only, don't actually clean up
```

Or commit everything saved:

```bash
cd /var/adm/sw/save
swmodify -x patch_commit=true *
```

Other cleanup tasks:

```bash
cleanup -t                    # trim /var/adm/sw*.log to the most recent 5 entries
cleanup -i                    # remove overwritten patch entries from the IPD
cleanup -d /depot/patch_depot # remove superseded patches from a patch depot
```

## Verifying Standard Patch Bundles

```bash
# HP-UX 11i v1
swlist -l bundle BUNDLE11i HWEnable11i GOLDBASE11i GOLDAPPS11i
# HP-UX 11i v2
swlist -l bundle BUNDLE11i HWEnable11i FEATURE11i QPKBASE QPKAPPS
# HP-UX 11i v3
swlist -l bundle BUNDLE11i HWEnable11i FEATURE11i QPKBASE
```

## Software Assistant (SWA)

SWA analyzes the system and recommends/downloads security and Quality Pack patches. `swa` was the successor to the older ITRC-based patch analysis tools.

| File | Purpose |
|------|---------|
| `$HOME/.swa.conf` | Per-user SWA config |
| `/etc/opt/swa/swa.conf` | System-wide SWA config |
| `/var/opt/swa/swa.log` | Default log for root |

SWA commands:

- `swa report` — generate a report of issues and recommended actions
- `swa get` — download software and create (or add to an existing) depot
- `swa step` — run an individual step of a `report`/`get` (both are multi-step operations)
- `swa clean` — remove software and files cached by SWA

The value of SWA is that it turns "which of the thousands of HP-UX patches do I actually need?" into a concrete, machine-specific answer. It inventories the installed software, compares it against HP's catalog of security bulletins and recommended (Quality Pack) patches, and produces a report plus, optionally, a ready-to-install depot. A common workflow is `swa report` to see what is recommended, review the findings, then `swa get` to build a depot and `swinstall` from it with `patch_match_target=true`.

## Reducing Patch Downtime with DRD

Because kernel and many subsystem patches require a reboot, patching a production system usually means a maintenance window. Dynamic Root Disk turns that risk down considerably: clone `vg00`, install the patches onto the *inactive* clone with `drd runcmd swinstall`, then boot the clone during the window. If the patched clone misbehaves, reboot back onto the original disk — no restore, no lengthy rollback. See [HP-UX Disaster Recovery (DRD and Ignite-UX)](articles/hpux-disaster-recovery.md) for the full DRD workflow.

```bash
drd clone -t /dev/disk/disk3 -x overwrite=true
drd runcmd swinstall -s server:/mydepot -x patch_match_target=true
drd activate -x reboot=false        # boot the patched clone at the next reboot
```

## Troubleshooting

- **`swinstall` reports dependency errors and installs nothing.** SD-UX is telling you a selected patch needs a base fileset or another patch that is missing. Run with `-p` to see the full dependency analysis, and add the missing pieces to the depot. Do not force past dependencies.
- **A patch will not install because a predecessor is "superseded."** The presence of a superseding patch blocks its predecessors by design — you are trying to install something already covered by a newer patch. Install the newest patch in the chain instead.
- **`/var` filling up.** The `/var/adm/sw/save` rollback area grows with every patch. Prune it with `cleanup -c 1` (commit patches superseded at least once) after previewing with `cleanup -p`. Reserve committing for patches you are certain you will never remove.
- **A patch made the system misbehave and you need to back it out.** `swremove PHxx_yyyyy` restores the saved originals — but only if the patch is still `applied` (not `committed`) and not itself superseded by another installed patch. Check state first with `swlist -l fileset -a patch_state *,c=patch`.
- **Post-install verification.** After any patch batch, run `swverify \*` to confirm filesets are consistent and `check_patches` to flag patch-specific issues; review `/var/adm/sw/swinstall.log` and `/var/adm/sw/swagent.log` for warnings.

```bash
swverify \*                                         # verify all installed software after patching
swlist -l fileset -a patch_state *,c=patch          # confirm a patch is still removable
more /var/adm/sw/swinstall.log                      # review the install log for WARNINGs/ERRORs
```

## Command Reference

| Task | Command |
|------|---------|
| Install single patch | `swinstall -s <patch>.depot -x autoreboot=true -x patch_match_target=true` |
| Install product w/o patches | `swinstall -s <depot> -x autoselect_patches=false <product>` |
| List installed patches | `swlist -l patch` |
| Remove a patch | `swremove -x autoreboot=true PHCO_1000` |
| Commit a patch | `swmodify -x patch_commit=true PHCO_1000` |
| Patch states | `swlist -l fileset -a patch_state *,c=patch` |
| Category tags | `swlist -l fileset -a category_tag *,c=patch` |
| Ancestry | `swlist -l fileset -a ancestor *,c=patch` |
| Cleanup superseded | `cleanup -c 1` (`-p` to preview) |
| SWA report / get | `swa report` / `swa get` |

## Related Articles

- [HP-UX Software Depots and swinstall](articles/hpux-software-depots-swinstall.md)
- [HP-UX Installation and Ignite-UX](articles/hpux-installation-ignite.md)
- [HP-UX swlist and Software Structure](articles/hpux-swlist-software-structure.md)
- [HP-UX Disaster Recovery (DRD and Ignite-UX)](articles/hpux-disaster-recovery.md)
- [HP-UX Kernel Configuration and Tuning](articles/hpux-kernel-configuration.md)
- [HP-UX Swap Management](articles/hpux-swap-management.md)
