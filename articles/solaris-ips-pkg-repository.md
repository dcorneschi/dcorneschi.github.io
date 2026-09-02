# Solaris 11 IPS: Local Package Repository and pkg Management

The Image Packaging System (IPS) is Oracle Solaris 11's network-based package manager, replacing the older SVR4 `pkg*` tooling. This guide covers hosting a **local IPS repository** (from the downloaded repo image), pointing clients at it via publishers, applying monthly **Support Repository Updates (SRUs)**, updating the OS with or without a network repo, and the everyday `pkg` commands.

> IPS is the Solaris 11 successor to SVR4 packaging (`pkgadd`/`pkgrm`). Packages come from **publishers** (repositories reached over HTTP or a local file path), and the whole OS image is updated atomically with `pkg update`, which creates a new boot environment.

## Set Up the Repository Server

Put the repository on its own ZFS dataset and disable `atime` (repos are read-heavy; access-time writes are wasted I/O):

```bash
zfs create rpool/export/IPS
zfs set atime=off rpool/export/IPS
```

Populate the repo from the downloaded repository image, then configure and enable the IPS depot server (SMF service `application/pkg/server`):

```bash
# Unpack the downloaded repo image into the dataset
./install-repo.ksh -d /export/IPS/repo -v -c

# Point the pkg depot server at the repo and make it read-only
svccfg -s application/pkg/server setprop pkg/inst_root=/export/IPS/repo
svccfg -s application/pkg/server setprop pkg/readonly=true

# Refresh SMF config, then enable the service
svcadm refresh application/pkg/server
svcadm enable application/pkg/server

# Confirm it's online
svcs application/pkg/server

# Refresh the repository catalog/search data
pkgrepo refresh -s /export/IPS/repo
```

- `pkg/inst_root` — filesystem path to the repository the depot serves.
- `pkg/readonly=true` — clients can fetch but not modify the repo.
- The depot listens on port **80** here (default configurable via `pkg/port`).

## Configure the Client

Point the client's `solaris` publisher at your local server, replacing the default origins:

```bash
# -G '*' removes all existing origins; -g adds the new one
pkg set-publisher -G '*' -g http://192.168.1.40/ solaris

# Verify the publisher/origin
pkg publisher
```

- `-g <uri>` adds a package origin (repository URL).
- `-G '*'` removes existing origins first, so the client uses only your local repo.

Sample `pkg publisher` output:

```
PUBLISHER   TYPE     STATUS P LOCATION
solaris     origin   online F http://192.168.1.40/
```

## Everyday Package Operations

Once a publisher is set, install/remove/query individual packages:

```bash
# Search the repo for a package
pkg search screen

# Show details (installed or available)
pkg info screen
pkg info -r screen            # -r = from the remote publisher

# Install / uninstall
pkg install screen
pkg uninstall screen

# List installed packages / check for available updates
pkg list
pkg list -u                  # only packages with updates available

# Show which package owns a file
pkg search -l -o pkg.name '/usr/bin/screen'

# Verify and repair installed packages
pkg verify
pkg fix <pkg>
```

IPS package names are FMRIs, e.g. `pkg://solaris/text/screen@4.0.3`. You can usually refer to a package by its short name (`screen`) unless it's ambiguous.

## Populate the Repository with a Monthly SRU

Oracle ships monthly **Support Repository Updates**. Load the new SRU image into the repo, restart the depot so it serves the updated catalog, then update clients:

```bash
# Add the new SRU content to the existing repo
./install-repo.ksh -c -v -d /export/IPS/repo

# Restart the depot to pick up the new packages
svcadm restart svc:/application/pkg/server

# On the client: apply the update (auto-accept licenses)
pkg update --accept
```

`pkg update` computes the new image, downloads changed packages, and (for kernel/OS updates) creates a new **boot environment** you reboot into — the running system stays intact until you reboot.

### Boot Environments (beadm)

Because `pkg update` clones a new ZFS boot environment (BE), you get a built-in rollback: if the update misbehaves, boot the previous BE. Manage them with `beadm`:

```bash
beadm list                       # show boot environments and which is active
beadm create solaris-backup      # snapshot the current BE before risky work
pkg update --accept              # update (usually creates a new BE automatically)
init 6                           # reboot into the new BE

# If the new BE is bad, activate the old one and reboot
beadm activate solaris-backup
init 6
```

Sample `beadm list`:

```
BE            Active Mountpoint Space Policy Created
solaris       N      /          9.2G  static 2024-01-10 09:12
solaris-1     R      -          4.1G  static 2024-06-01 14:33
```

`Active` column: `N` = active now, `R` = active on reboot, `NR` = both.

## Update the OS Without a Remote Repository

If there's no network depot, point the client directly at a repo on local storage (or mounted media) using a `file://` publisher:

```bash
# Refresh the local repo image
./install-repo.ksh -c -v -d /export/IPS/repo

# Use the on-disk repo directly (no HTTP server needed)
pkg set-publisher -g file:///export/IPS/repo solaris

# Apply the update
pkg update --accept
```

The `file://` origin avoids running the depot service entirely — handy for air-gapped or single-host updates.

## Common pkg Commands

```bash
# Check a repository's status/details over HTTP
pkgrepo info -s http://192.168.1.40

# Show configured publishers and their origins
pkg publisher

# Remove the default publisher (e.g. before repointing)
pkg unset-publisher solaris

# List all packages available in a repository
pkg search -p -s http://192.168.1.40 '*'

# Find which package delivers a given file
pkg search -o path,pkg.name -l /usr/bin/screen

# Rebuild the local search index (speeds up searches)
pkg rebuild-index
```

- `pkg search -l` searches the **local** (installed) image; `-s <uri>` searches a **remote** repo.
- `-o path,pkg.name` limits output columns; `-p` matches by package.
- `pkgrepo info` reports package counts, status, and last-update time for a repo.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `pkg install` fails "no matching publisher" | Publisher not set/reachable | `pkg publisher`; re-add with `pkg set-publisher -g <uri> solaris` |
| Depot service won't come online | Bad `pkg/inst_root` or perms | `svcs -xv application/pkg/server`; check the repo path and logs |
| Clients see stale catalog after SRU | Depot not restarted / catalog not refreshed | `svcadm restart application/pkg/server`; `pkgrepo refresh` |
| `pkg update` says "no updates available" | Publisher points at old repo, or phased | Confirm origin; `pkg refresh --full` |
| Update broke the system | Bad package set | Boot the previous BE: `beadm activate <old>` then `init 6` |
| Slow searches | Search index stale | `pkg rebuild-index` |

```bash
# Diagnose the depot service
svcs -xv application/pkg/server
```

## Command Reference

| Task | Command |
|------|---------|
| Create repo dataset | `zfs create rpool/export/IPS` + `zfs set atime=off ...` |
| Load/refresh repo image | `./install-repo.ksh -c -v -d /export/IPS/repo` |
| Point depot at repo | `svccfg -s application/pkg/server setprop pkg/inst_root=...` |
| Enable depot | `svcadm enable application/pkg/server` |
| Refresh repo catalog | `pkgrepo refresh -s /export/IPS/repo` |
| Set client publisher (HTTP) | `pkg set-publisher -G '*' -g http://host/ solaris` |
| Set client publisher (file) | `pkg set-publisher -g file:///export/IPS/repo solaris` |
| Show publishers | `pkg publisher` |
| Remove publisher | `pkg unset-publisher solaris` |
| Update the OS | `pkg update --accept` |
| List all repo packages | `pkg search -p -s http://host '*'` |
| Find file's package | `pkg search -o path,pkg.name -l /path` |
| Rebuild search index | `pkg rebuild-index` |
| Repo status | `pkgrepo info -s http://host` |

## IPS vs SVR4 Packaging

| Concept | IPS (Solaris 11) | SVR4 (Solaris 10 and earlier) |
|---------|------------------|-------------------------------|
| Primary command | `pkg` | `pkgadd` / `pkgrm` |
| Source | Publisher (network/file repo) | Media, spool, stream file |
| Update model | Whole-image `pkg update`, new boot environment | Per-package `pkgadd` |
| Query | `pkg search` / `pkg info` | `pkginfo` / `pkgchk` |
| Repo tooling | `pkgrepo`, `application/pkg/server` | `/var/spool/pkg` |

For the SVR4 tooling, see the [Solaris SVR4 Package Management](articles/solaris-svr4-package-management.md) article.

## References

- [Adding and Updating Software in Oracle Solaris 11 (IPS)](https://docs.oracle.com/cd/E23824_01/html/E21802/index.html) — official Oracle docs
- [Copying and Creating Oracle Solaris Package Repositories](https://docs.oracle.com/cd/E23824_01/html/E21803/index.html) — official Oracle docs
- [pkg(1) man page](https://docs.oracle.com/cd/E23824_01/html/821-1451/pkg-1.html) — official Oracle docs
