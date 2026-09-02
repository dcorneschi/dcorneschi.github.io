# Fixing "The following packages have been kept back" on Ubuntu

Run `sudo apt upgrade` on Ubuntu and you'll eventually see this:

```
The following packages have been kept back:
  linux-generic linux-headers-generic linux-image-generic
0 upgraded, 0 newly installed, 0 to remove and 3 not upgraded.
```

The packages have updates available, but `apt` refused to install them. This isn't an error or a broken system — it's `apt` being deliberately cautious. This article explains why it happens and the right way to resolve it.

On **Ubuntu 24.04**, the same situation is often reported with clearer wording:

```
The following upgrades have been deferred due to phasing:
  ...
```

Both messages mean the same thing — updates exist but apt is holding them back for now.

## Why Packages Get Kept Back

`apt upgrade` has one hard rule: it will **never install new packages or remove existing ones** to satisfy an upgrade. It only upgrades packages that can be updated in place without changing what else is installed.

A package is held back when upgrading it would require one of those forbidden actions. The common triggers:

- **New dependencies** — the newer version needs a package that isn't installed yet. Installing it means adding a new package, which `apt upgrade` won't do.
- **Removing a package** — the new version conflicts with or obsoletes something currently installed, so the upgrade would need a removal.
- **Changed dependencies (very common with kernels)** — meta-packages like `linux-generic` point at a new versioned package (e.g. `linux-image-6.8.0-40-generic`) that must be *installed* alongside/replacing the old one. That's a new-package install, so it's kept back.
- **Phased updates** — Ubuntu rolls some updates out gradually to a percentage of machines. If your machine isn't in the current phase yet, the update is intentionally held back for a while (you'll often see it resolve itself after a few days).

In short: kept-back packages are ones whose upgrade changes the set of installed packages, not just their versions.

### Why You Suddenly Started Seeing This on the CLI

Phased updates aren't new — Ubuntu introduced them back in 2013 — but for years they only applied to the graphical **Update Manager**. The `apt` and `apt-get` command-line tools were exempt, on the assumption that anyone running them manually intended to pull every available update. That's why the CLI historically showed more updates than the GUI.

That changed in **Ubuntu 21.04**, when a new APT version began honoring phased updates on the command line too. So if you only recently started seeing "packages have been kept back" during `apt upgrade`, this is why — the behavior moved into `apt` itself, not something that broke on your system.

## Diagnosing Before You Act

Before forcing anything, see *why* a package is held back. Simulate the upgrade with `-s` (no changes are made):

```bash
# Dry run — shows what WOULD be installed/removed, without doing it
sudo apt upgrade -s

# Inspect a single kept-back package
apt-cache policy linux-generic
apt list --upgradable
```

Look at the simulated output: if it wants to **install** new packages or **remove** existing ones, that's exactly why `apt upgrade` skipped it.

### Check for Phased Updates

If a package is held back purely due to phasing, `apt policy` shows a phasing percentage, and the reason appears in the simulation:

```bash
apt-cache policy <package>
# Look for a line like:  Phased: 40%
```

If it's a phased update, the cleanest action is simply to **wait** — it'll be offered automatically once your machine enters the rollout phase. Forcing it works too, but phasing exists to catch bad updates before they hit everyone.

## The Recommended Fix: `full-upgrade`

The correct tool for kept-back packages is `apt full-upgrade` (formerly `dist-upgrade`). Unlike `upgrade`, it's allowed to install new dependencies and remove packages when needed to complete upgrades:

```bash
sudo apt update
sudo apt full-upgrade
```

Because `full-upgrade` **can remove packages**, always read the summary before confirming — check the "will be REMOVED" list to be sure nothing important is going away. For routine kept-back kernel meta-packages, the removals are usually just superseded packages and it's safe.

## Upgrading Specific Kept-Back Packages

If you'd rather not run a full system upgrade, upgrade just the held-back packages by name. Prefer `--only-upgrade`, which permits `apt` to pull in new dependencies **without** flipping the packages' auto/manual mark:

```bash
# Upgrade only these packages; don't mark them manually installed
sudo apt install --only-upgrade linux-generic linux-headers-generic linux-image-generic
```

Why `--only-upgrade` matters: a plain `sudo apt install <pkg>` also upgrades the package, but it marks it as **manually installed**. That changes autoremove behavior — the package (and its now-orphaned deps) won't be cleaned up automatically later even when nothing needs them. `--only-upgrade` avoids that side effect, so it's the safer choice for packages that were originally pulled in automatically (like kernel meta-packages).

To grab the exact list programmatically and upgrade it in one line:

```bash
sudo apt install --only-upgrade $(apt list --upgradable 2>/dev/null | awk -F/ 'NR>1 {print $1}')
```

This is a good middle ground when you want control over what changes but don't want to type each package name.

## Method Comparison

| Command | Installs new deps? | Removes packages? | Use when |
|---------|:--:|:--:|----------|
| `apt upgrade` | No | No | Routine updates; leaves complex ones kept back |
| `apt full-upgrade` | Yes | Yes | Clearing kept-back packages system-wide |
| `apt install --only-upgrade <pkgs>` | Yes | Only if required | Upgrading specific kept-back packages without changing their auto/manual mark |
| (wait) | — | — | Held back only by phased rollout |

## Are These Packages Actually "Held"?

"Kept back" is not the same as a package being **on hold**. A held package is one you (or a tool) explicitly froze with `apt-mark hold`. Verify nothing is intentionally held:

```bash
# List packages explicitly marked hold
apt-mark showhold

# Remove a hold if you find one you didn't intend
sudo apt-mark unhold <package>
```

If a package shows up under `showhold`, that's a different mechanism — `full-upgrade` won't touch it until you `unhold` it. Kept-back packages, by contrast, have no hold mark; they're skipped only because of the install/remove rule.

## Verify Afterward

```bash
# Should report 0 packages kept back / not upgraded
sudo apt upgrade -s

# Confirm nothing is left upgradable
apt list --upgradable

# If a new kernel was installed, reboot to use it
[ -f /var/run/reboot-required ] && cat /var/run/reboot-required
```

A freshly installed kernel only takes effect after a reboot, so check for `/var/run/reboot-required`.

## Quick Decision Guide

- Kept-back packages are **kernels or meta-packages**, and you want them updated → `sudo apt full-upgrade` (review removals).
- You want to update **only specific** kept-back packages → `sudo apt install <package names>`.
- `apt policy` shows a **Phased:** percentage → wait a few days, or force with `install`/`full-upgrade` if urgent.
- Package appears in `apt-mark showhold` → it's a deliberate hold; `sudo apt-mark unhold <package>` first.

## References

- [Ubuntu apt documentation](https://help.ubuntu.com/community/AptGet/Howto) — official community docs
- [apt(8) manual page](https://manpages.ubuntu.com/manpages/noble/en/man8/apt.8.html) — official Ubuntu manpage
- [Ubuntu phased updates](https://wiki.ubuntu.com/PhasedUpdates) — official wiki
