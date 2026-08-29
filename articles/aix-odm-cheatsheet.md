# AIX ODM Cheatsheet

Command reference for the IBM AIX **Object Data Manager (ODM)** — the object-oriented configuration database that stores device definitions, attributes, software (SWVPD), SMIT menus, and error-log templates. Many AIX commands (`lsdev`, `lsattr`, `cfgmgr`, `installp`) read and write the ODM behind the scenes; the `odm*` tools let you query and, carefully, repair it directly.

> ODM edits are powerful and risky — a corrupted ODM can make devices or the system unbootable. Always `odmget` a class to a file (a backup) before `odmdelete`/`odmchange`, and prefer the high-level tools (`chdev`, `rmdev`, `chgrp`) over raw ODM edits where possible.

## What Is the ODM?

The ODM is a database for storing system information — physical and logical device data is held as **objects** with associated **descriptors** (characteristics). For security, ODM data is stored in **binary** format: you cannot edit ODM files with a text editor, only through the ODM command-line interface.

Structurally, the ODM is made of **object classes** (files) that contain individual **objects** (records), whose layout is described by **descriptors**. Device information lives in the customized and predefined databases — the `Cu*` and `Pd*` classes.

When AIX boots, the **Configuration Manager (`cfgmgr`)** configures devices. It uses the **`Config_Rules`** object class to determine the correct sequence for configuring devices; `Config_Rules` also references the methods files used for device management.

## ODM Repositories

The ODM lives in three locations, selected by the `ODMDIR` environment variable (default `/etc/objrepos`).

| Path | Contents |
|------|----------|
| `/etc/objrepos` | Customized (system-specific) config — devices, attributes, SWVPD (`$ODMDIR` default) |
| `/usr/lib/objrepos` | Predefined device/software classes and SMIT menus (shared) |
| `/usr/share/lib/objrepos` | Architecture-independent shared data |

```sh
echo $ODMDIR                         # which repository the odm* tools use
export ODMDIR=/etc/objrepos          # point at a specific repository
```

## Key Object Classes

### Device configuration

| Class | Meaning |
|-------|---------|
| `PdDv` | **Predefined Devices** — supported device types |
| `PdAt` | **Predefined Attributes** — default attribute values per device type |
| `CuDv` | **Customized Devices** — devices actually defined on this system (state, parent, location) |
| `CuAt` | **Customized Attributes** — non-default attribute values for configured devices |
| `CuDep` | Customized dependencies between devices |
| `CuVPD` | Customized Vital Product Data |
| `CuPath` | MPIO path information |

### Software (SWVPD)

| Class | Meaning |
|-------|---------|
| `product` | Installed software products |
| `lpp` | Installed licensed program products (filesets) |
| `history` | Install/update history |
| `inventory` | Files that filesets installed |

## Querying the ODM (odmget)

`odmget` retrieves objects from a class as editable stanzas.

```sh
odmget CuDv                          # all customized devices
odmget -q "name=hdisk0" CuDv         # one device
odmget -q "name=hdisk0" CuAt         # a device's customized attributes
odmget -q "attribute=queue_depth AND name=hdisk0" CuAt
odmget -q "PdDvLn LIKE disk/*" PdDv  # predefined disk types (wildcard)
odmget PdAt | more                   # predefined attributes

# Save a class to a file (your backup / edit source)
odmget CuAt > /tmp/CuAt.bak
```

The `-q` qualifier tests descriptor values with these operators (combine with `AND`/`OR`, and `LIKE` for wildcards):

| Operator | Test |
|----------|------|
| `=` | equal |
| `!=` | not equal |
| `>` | greater than |
| `>=` | greater than or equal |
| `<` | less than |
| `<=` | less than or equal |
| `LIKE` | wildcard match (`?` single char, `*` any) |

```sh
odmget -q "name like hdisk?" CuDv    # LIKE with a single-char wildcard
```

> **SWVPD note:** `lslpp -l` reports each fileset once **per location** it's recorded in (`/`, `/usr/lib`, `/usr/share/lib`), which can be noisy. `lslpp -L` reports each fileset just **once**, without the root/usr/share distinction.

## Viewing a Class Definition (odmshow)

```sh
odmshow CuDv                         # show the C-structure definition of a class
odmshow CuAt
```

## Adding, Changing, Deleting Objects

```sh
# Add objects from a stanza file
odmadd /tmp/newobjects.add

# Change objects matching a qualifier
odmchange -o CuAt -q "name=hdisk0 AND attribute=queue_depth" /tmp/CuAt.new

# Delete objects matching a qualifier (be careful!)
odmdelete -o CuAt -q "name=hdisk0 AND attribute=pvid"
odmdelete -o CuAt -q "name=lp1"           # e.g. remove a printer's attributes
odmdelete -o CuDv -q "name=hdisk5"
```

Export/import a class as text (edit between the two steps):

```sh
odmget -q "name=lp1" CuAt > lp1.CuAt      # export matching objects to a file
odmadd < lp1.CuAt                         # re-import from the file (stdin form)
odmadd /tmp/newobjects.add                # or pass the file as an argument
```

A stanza file for `odmadd`/`odmchange` looks like:

```
CuAt:
    name = "hdisk0"
    attribute = "queue_depth"
    value = "256"
    type = "R"
    generic = "DU"
    rep = "nl"
    nls_index = 0
```

## Creating / Dropping Classes

For custom applications that use the ODM (rarely needed for admins):

```sh
odmcreate myclass.cre                # create object classes from a .cre definition file
odmdrop -o MyClass                   # drop (delete) an object class entirely
```

## How the High-Level Tools Map to the ODM

You usually manipulate the ODM indirectly:

| Command | ODM effect |
|---------|-----------|
| `lsdev -C` | Reads `CuDv` |
| `lsattr -El <dev>` | Reads `CuAt` (falls back to `PdAt` defaults) |
| `chdev` | Writes `CuAt` (and may update `CuDv`) |
| `mkdev` / `cfgmgr` | Adds `CuDv`/`CuAt` from `PdDv`/`PdAt` |
| `rmdev` | Removes/marks `CuDv` (`-d` deletes the definition) |
| `lslpp` | Reads the SWVPD classes (`lpp`, `product`, `history`) |

Prefer these over raw `odm*` edits — they keep related classes consistent.

## Common Repair Scenarios

> Take a `mksysb`/`savevg` first. See the [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md).

### Remove a stale/ghost disk definition

```sh
odmget -q "name=hdisk9" CuDv > /tmp/hdisk9.bak     # back up first
rmdev -dl hdisk9                                   # preferred (uses ODM safely)
# only if rmdev fails and the device is truly gone:
odmdelete -o CuDv -q "name=hdisk9"
odmdelete -o CuAt -q "name=hdisk9"
```

### Fix a duplicated/incorrect PVID in the ODM

```sh
odmget -q "name=hdisk3 AND attribute=pvid" CuAt    # inspect
chdev -l hdisk3 -a pv=clear                        # clear via the supported tool
# or, as a last resort:
odmdelete -o CuAt -q "name=hdisk3 AND attribute=pvid"
```

### Fix ODM problems for a non-rootvg volume group

Re-importing a VG rebuilds its ODM entries from the VGDA on disk — the simplest fix for VG-related ODM inconsistencies (data is preserved; the disk keeps its content).

```sh
varyoffvg homevg
exportvg homevg              # removes the (bad) ODM definition
importvg -y homevg hdiskX    # re-reads the VGDA and rebuilds the ODM
```

### Repair a broken ODM database

```sh
# Verify software VPD consistency (reads SWVPD ODM classes)
lppchk -v

# Rebuild the ODM device configuration by re-running config manager
cfgmgr

# Recover the ODM for a volume group (VIOS/AIX LVM)
synclvodm <vg>            # sync VGDA -> ODM
redefinevg -d <disk> <vg># sync ODM <- VGDA
rvgrecover               # (VIOS) rebuild the ODM for a VG
```

## Safety Checklist

1. Back up the class first: `odmget <Class> > /tmp/<Class>.bak`.
2. Use `-q` qualifiers precise enough to match only the intended objects.
3. Prefer `chdev`/`rmdev`/`cfgmgr` over `odmchange`/`odmdelete`.
4. After edits, verify with `lsdev`/`lsattr` and, for boot-critical changes, `savebase -d /dev/hdiskX` so the change survives into the boot image.
5. Keep a `mksysb` so you can recover if the ODM becomes inconsistent.

## Quick Reference

| Task | Command |
|------|---------|
| Which repository | `echo $ODMDIR` |
| List customized devices | `odmget CuDv` |
| One device's attributes | `odmget -q "name=hdisk0" CuAt` |
| Wildcard query | `odmget -q "PdDvLn LIKE disk/*" PdDv` |
| Back up a class | `odmget <Class> > /tmp/<Class>.bak` |
| Add objects | `odmadd <file>` |
| Change objects | `odmchange -o <Class> -q "<qualifier>" <file>` |
| Delete objects | `odmdelete -o <Class> -q "<qualifier>"` |
| Show class structure | `odmshow <Class>` |
| Save ODM to boot image | `savebase -d /dev/hdisk0` |

## Related

- [AIX LVM Cheatsheet](articles/aix-lvm-cheatsheet.md) — `synclvodm`/`redefinevg` sync the ODM with the VGDA
- [AIX Boot and Init Cheatsheet](articles/aix-boot-init-cheatsheet.md) — `savebase`/`restbase` and the ODM in the boot logical volume
- [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md) — mksysb before risky ODM edits
- [AIX Package Management Cheatsheet](articles/aix-package-management-cheatsheet.md) — the SWVPD classes behind `lslpp`
