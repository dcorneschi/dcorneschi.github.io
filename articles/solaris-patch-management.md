# Solaris 10 Patch Management

Managing patches on Oracle Solaris 10 (and earlier) with the `patchadd`, `patchrm`, and `showrev` tools — checking what's installed, applying and removing individual patches, reading patch logs, and installing a Recommended Patch Cluster.

> This is the Solaris 10 SVR4-style patching model. On **Solaris 11**, patches are delivered as IPS packages and applied with `pkg update` — see [Solaris 11 IPS](articles/solaris-ips-pkg-repository.md). For the underlying package tooling, see [Solaris SVR4 Package Management](articles/solaris-svr4-package-management.md).

## Checking Installed Patches

```bash
# Is a specific patch installed? (search by patch number)
showrev -p | grep 148363

# List all installed patches
showrev -p
```

`showrev -p` lists every applied patch with the packages it touches. Grep for the patch ID to confirm presence.

Sample `showrev -p` output:

```
Patch: 119254-88 Obsoletes: ... Requires: 119042-02 Incompatibles:  Packages: SUNWcsu SUNWcsr
Patch: 120900-04 Obsoletes:     Requires:            Incompatibles:  Packages: SUNWcakr
```

Each line shows the patch ID, patches it **obsoletes** (supersedes), patches it **requires** (must be applied first), any **incompatibles**, and the packages it modifies.

### Patch ID and Types

- A patch ID is `NNNNNN-RR` — a base number plus a two-digit **revision**. A higher revision supersedes lower ones (e.g. `119254-88` obsoletes `119254-30`).
- **IDR** (`IDR<id>`) — Interim Diagnostic/Relief patch, a temporary fix from support; usually removed once a formal patch ships.
- **T-patch** — a test/pre-release patch (not for production).

## Installing Patches (`patchadd`)

```bash
# Apply a patch
patchadd 101010-01

# Apply and delete the previously saved (old) version copy to reclaim space
patchadd -d 101010-01

# Apply many patches from a directory, in the order listed in a file
patchadd -M /var/tmp/patches patch_order

# Apply a patch to an alternate boot environment / mounted root
patchadd -R /a 101010-01
```

- A patch ID has the form `NNNNNN-RR` (base number + revision), e.g. `101010-01`.
- By default `patchadd` saves the files it replaces so the patch can be backed out later.
- The `-d` option tells `patchadd` **not** to save those backout copies (saves disk, but the patch can no longer be removed with `patchrm`).

## Removing Patches (`patchrm`)

```bash
# Remove a patch and restore the files it replaced
patchrm 101010-01

# Remove an IDR (Interim Diagnostic/Relief) patch
patchrm IDR148363-26
```

`patchrm` restores the files that were modified or replaced by the patch — provided the backout copies were saved (i.e. the patch was **not** applied with `patchadd -d`).

## Patch Log Files

```bash
# Installation data / logs directory
ls -l /var/sadm/install_data

# Per-patch log files
ls -l /var/sadm/patch/<PatchID>/log
```

- `/var/sadm/install_data` — holds cluster/patch install logs (e.g. from `installcluster`).
- `/var/sadm/patch/<PatchID>/log` — the individual log for a specific patch; check here if a patch fails or behaves unexpectedly.

## Installing a Recommended Patch Cluster (Solaris 10 x86)

Oracle ships a **Recommended Patch Cluster** that bundles current patches. The workflow: download, unzip, optionally drop to single-user mode, read the README for the passcode, run `installcluster`, then reboot.

```bash
# 1. Download and extract the cluster
wget https://updates.oracle.com/patch_cluster/10_x86_Recommended.zip
unzip -q 10_x86_Recommended.zip

# 2. (Optional) drop to single-user mode for a clean patch run
shutdown -g0 -y -i s

# 3. Find the passcode the installer requires
cd 10_x86_Recommended
grep PASSCODE README

# 4. Apply prerequisite patches first, then the cluster
./installcluster --apply-prereq --s10cluster

# 5. Reboot to activate
init 6
# or: shutdown -y -g0 -i6
```

- `--apply-prereq` installs the prerequisite patches the cluster needs before the main set.
- `--s10cluster` is the passcode/token for the Solaris 10 cluster (the exact value is shown by `grep PASSCODE README`).
- Single-user mode (`-i s`) is **not mandatory** but gives the cleanest patch application, especially for kernel patches.
- Always **reboot** afterward so patched kernel/driver components take effect.

## Patching an Inactive Boot Environment (Live Upgrade)

To minimize downtime and keep a guaranteed fallback, patch a copy of the OS (an inactive boot environment) while the system keeps running, then reboot into it:

```bash
# Create a new boot environment (a copy of the current one)
lucreate -n s10-patched

# Mount the inactive BE so you can patch it
lumount s10-patched /mnt

# Apply patches to the mounted BE (use its mount point as the alternate root)
patchadd -R /mnt <id>
# or a cluster: ./installcluster --apply-prereq --s10cluster -R /mnt

# Unmount, then activate the patched BE and reboot into it
luumount s10-patched
luactivate s10-patched
init 6

# If it misbehaves, fall back to the original BE
luactivate <original-BE>
init 6
```

This keeps the running BE untouched, so a bad patch is a reboot away from recovery. On Solaris 11 this concept is handled automatically by IPS boot environments (see the IPS article).

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `patchadd` fails "requires patch" | Prerequisite not installed | Apply the required patch first (see `Requires:` in `showrev -p`) |
| Patch won't apply, already superseded | A newer revision/obsoleting patch is on | Check `showrev -p`; no action needed |
| Cannot `patchrm` | Applied with `-d` (no backout) | Restore from backup/BE; you can't back it out |
| Cluster install stops midway | A single patch failed | Read `/var/sadm/install_data/*log`; re-run cluster (it skips applied patches) |
| System won't boot after patching | Bad kernel patch | Boot the pre-patch BE (`luactivate`) and reboot |

```bash
# Review the most recent cluster install log
ls -lt /var/sadm/install_data/ | head
```

## Command Reference

| Task | Command |
|------|---------|
| Check if a patch is installed | `showrev -p \| grep <id>` |
| List all patches | `showrev -p` |
| Apply a patch | `patchadd <id>` |
| Apply without backout copy | `patchadd -d <id>` |
| Remove a patch | `patchrm <id>` |
| Remove an IDR patch | `patchrm IDR<id>` |
| Cluster/patch logs | `/var/sadm/install_data` |
| Per-patch log | `/var/sadm/patch/<id>/log` |
| Apply patch cluster | `./installcluster --apply-prereq --s10cluster` |

## Notes

- If you patched with `patchadd -d`, you cannot back the patch out with `patchrm` — there are no saved originals. Only use `-d` when you're confident you won't need to revert and want to reclaim space.
- Back up (or take a ZFS/UFS snapshot of) the boot environment before large patch clusters so you have a clean rollback path.
- On Solaris 10 with Live Upgrade / multiple boot environments, patch an inactive BE and activate it, to minimize downtime and keep a fallback.

## References

- [patchadd(1M) man page](https://docs.oracle.com/cd/E23823_01/html/816-5166/patchadd-1m.html) — official Oracle docs
- [patchrm(1M) man page](https://docs.oracle.com/cd/E23823_01/html/816-5166/patchrm-1m.html) — official Oracle docs
- [Oracle Solaris 10 Patch Management](https://docs.oracle.com/cd/E26505_01/html/E29492/index.html) — official Oracle docs
