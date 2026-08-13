# apt vs apt-get: Differences and When to Use Each

`apt` and `apt-get` are both command-line package managers for Debian-based systems. They use the same backend libraries and repositories, but differ in interface design, output stability, and feature set. Understanding when to use each prevents subtle issues in scripts and daily work.

## Key Difference

- **`apt`** — designed for interactive terminal use. Progress bars, colors, cleaner output. Introduced in Ubuntu 14.04 / Debian 8 (2014).
- **`apt-get`** — designed for scripts and automation. Stable output format across releases. Available since Debian's early days (1998).

Both call the same `libapt` library underneath. Neither is "better" — they serve different purposes.

## Command Mapping

| Task | `apt` | `apt-get` / `apt-cache` |
|------|-------|-------------------------|
| Update package index | `apt update` | `apt-get update` |
| Install a package | `apt install pkg` | `apt-get install pkg` |
| Remove a package | `apt remove pkg` | `apt-get remove pkg` |
| Purge a package | `apt purge pkg` | `apt-get purge pkg` |
| Upgrade installed packages | `apt upgrade` | `apt-get upgrade --with-new-pkgs` |
| Full upgrade (may remove pkgs) | `apt full-upgrade` | `apt-get dist-upgrade` |
| Autoremove unused deps | `apt autoremove` | `apt-get autoremove` |
| Search for packages | `apt search keyword` | `apt-cache search keyword` |
| Show package info | `apt show pkg` | `apt-cache show pkg` |
| Show package policy | `apt policy pkg` | `apt-cache policy pkg` |
| List installed packages | `apt list --installed` | `dpkg -l` |
| List upgradeable packages | `apt list --upgradeable` | `apt-get -u upgrade --assume-no` |
| Download without install | `apt download pkg` | `apt-get download pkg` |
| Clean package cache | `apt clean` | `apt-get clean` |
| Autoclean old cache | `apt autoclean` | `apt-get autoclean` |
| Fix broken dependencies | `apt --fix-broken install` | `apt-get install -f` |
| Edit sources.list | `apt edit-sources` | *(no equivalent)* |
| Show package dependencies | `apt depends pkg` | `apt-cache depends pkg` |
| Show reverse dependencies | `apt rdepends pkg` | `apt-cache rdepends pkg` |
| Show changelog | `apt changelog pkg` | `apt-get changelog pkg` |
| Build dependencies | *(no equivalent)* | `apt-get build-dep pkg` |
| Source download | `apt source pkg` | `apt-get source pkg` |
| Simulate/dry-run | `apt install -s pkg` | `apt-get install -s pkg` |

## Behavioral Differences

### Upgrade vs Upgrade

This is the most important difference:

| Command | Behavior |
|---------|----------|
| `apt upgrade` | Upgrades packages, installs new dependencies if needed, but **never removes** packages |
| `apt-get upgrade` | Upgrades packages but **skips** any that require new dependencies (marks them "kept back") |
| `apt full-upgrade` | Upgrades everything, may **remove** packages to resolve conflicts |
| `apt-get dist-upgrade` | Same as `apt full-upgrade` |

```bash
# apt upgrade ≈ apt-get upgrade --with-new-pkgs
# (installs new deps but won't remove anything)

# apt full-upgrade ≈ apt-get dist-upgrade
# (will remove packages if necessary)
```

> **Practical impact:** After `apt-get upgrade`, you may see "The following packages have been kept back." With `apt upgrade`, those packages get upgraded (unless removal is needed).

### Dependency Resolution

Both tools handle dependency resolution, but `apt` is more aggressive about it:

| Behavior | `apt` | `apt-get` |
|----------|-------|-----------|
| Installs required dependencies | Yes | Yes |
| Installs new deps during upgrade | Yes | No (keeps packages back) |
| Recommends suggested packages | Yes (shows in output) | No |
| Determines complex dependency chains | More advanced ordering | Basic ordering |
| Removes old package versions during upgrade | Yes (auto-cleans obsolete versions) | No (leaves old versions on disk) |

```bash
# apt upgrade automatically removes obsolete package versions
# apt-get upgrade does NOT — old versions remain in the system

# To get similar cleanup behavior with apt-get, run:
sudo apt-get upgrade && sudo apt-get autoremove
```

### Search

`apt` unified the search functionality that previously required a separate tool (`apt-cache`):

```bash
# Modern (apt) — single command with highlighted results
apt search nginx

# Legacy (apt-get era) — required apt-cache, no highlighting
apt-cache search nginx
```

> `apt search` provides colorized output with the package name highlighted, making it easier to scan results interactively.

### Output Format

```bash
# apt — colored, progress bar, human-friendly
$ apt install nginx
Reading package lists... Done
Building dependency tree... Done
The following NEW packages will be installed:
  nginx nginx-common
0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
Need to get 580 kB of archives.
Progress: [################............] 58%

# apt-get — plain text, machine-parseable
$ apt-get install nginx
Reading package lists... Done
Building dependency tree... Done
The following NEW packages will be installed:
  nginx nginx-common
0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
Need to get 580 kB of archives.
Get:1 http://archive.ubuntu.com/ubuntu noble/main amd64 nginx-common all 1.24.0-2 [30.4 kB]
```

### Exclusive Features

Features only in `apt` (not in `apt-get`):

| Feature | Command |
|---------|---------|
| List installed packages | `apt list --installed` |
| List upgradeable packages | `apt list --upgradeable` |
| List all versions | `apt list --all-versions pkg` |
| Edit sources interactively | `apt edit-sources` |
| Progress bar | Automatic |
| Colored output | Automatic |
| Number of upgradeable after update | Shown at end of `apt update` |

Features only in `apt-get` / `apt-cache` (not in `apt`):

| Feature | Command |
|---------|---------|
| Build dependencies | `apt-get build-dep pkg` |
| Check for broken packages | `apt-get check` |
| dselect upgrade | `apt-get dselect-upgrade` |
| Show package graph info | `apt-cache showpkg pkg` |
| Show unmet dependencies | `apt-cache unmet` |
| Cache statistics | `apt-cache stats` |
| Package names list | `apt-cache pkgnames` |
| Dump entire cache | `apt-cache dump` |
| Show source package info | `apt-cache showsrc pkg` |
| Show package records | `apt-cache show --no-all-versions pkg` |

## Output Stability

`apt` explicitly warns in its man page:

> "The command line interface of apt is not stable and may change between versions. Do not use apt in scripts."

| Aspect | `apt` | `apt-get` |
|--------|-------|-----------|
| Output format | May change between releases | Stable across releases |
| Exit codes | Same as apt-get | Well-documented, stable |
| Progress display | Progress bar (can interfere with piping) | Plain text (pipe-safe) |
| Color codes | Yes (can be disabled with `-o APT::Color=false`) | No |
| Scripting safe | No | Yes |

```bash
# In scripts, always use apt-get:
#!/bin/bash
apt-get update -qq
apt-get install -y -qq package-name

# For interactive use, prefer apt:
apt update
apt install package-name
```

## Configuration Differences

Both read the same configuration files (`/etc/apt/apt.conf.d/`), but:

```bash
# Disable color in apt (when piping output)
apt -o APT::Color=false install pkg

# Or permanently
echo 'APT::Color "false";' | sudo tee /etc/apt/apt.conf.d/99nocolor

# apt-get never uses color — no config needed
```

## Performance

There is no performance difference. Both tools:
- Use the same `libapt-pkg` library
- Download from the same repositories
- Call `dpkg` for actual installation
- Read the same package cache

The only visible difference is rendering speed of the progress bar in `apt`, which is negligible.

## History

| Year | Event |
|------|-------|
| 1998 | `apt-get` introduced in Debian 2.0 |
| 2004 | `apt-cache` split out for query operations |
| 2014 | `apt` introduced as a unified, user-friendly frontend (Debian 8 / Ubuntu 14.04) |
| 2016+ | `apt` becomes the recommended tool for interactive use |

`apt` was created because having separate `apt-get` (install/remove) and `apt-cache` (search/show) commands was confusing for users. `apt` merges the most commonly used subcommands into one tool with better defaults.

## When to Use Which

### Use `apt` when:
- Working interactively in a terminal
- You want progress bars and colored output
- Running one-off installs or upgrades
- You want `apt list --installed` or `apt list --upgradeable`
- Learning the package system (simpler mental model)

### Use `apt-get` / `apt-cache` when:
- Writing shell scripts or automation
- Building Docker images (`RUN apt-get install -y`)
- In CI/CD pipelines
- When output needs to be parsed
- Using features not in `apt` (`build-dep`, `check`, `showpkg`)
- In cron jobs or systemd services
- When you need guaranteed stable behavior across OS upgrades

### Dockerfile Example

```dockerfile
# Always use apt-get in Dockerfiles
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      ca-certificates && \
    rm -rf /var/lib/apt/lists/*
```

### Ansible Example

```yaml
# Ansible uses apt module (calls apt-get internally)
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes
```

## Common Misconceptions

| Misconception | Reality |
|---------------|---------|
| "`apt` is newer so it's better" | They serve different purposes — neither replaces the other |
| "`apt-get` is deprecated" | No. `apt-get` is actively maintained and recommended for scripts |
| "`apt` is faster" | Same backend, same speed |
| "`apt upgrade` = `apt-get upgrade`" | No! `apt upgrade` installs new deps, `apt-get upgrade` doesn't |
| "`apt-get dist-upgrade` upgrades the distro" | No — it resolves conflicts aggressively. Use `do-release-upgrade` for distro upgrades |

## Quick Decision Table

| Situation | Use |
|-----------|-----|
| Terminal, one-off commands | `apt` |
| Shell scripts | `apt-get` |
| Dockerfiles | `apt-get` |
| CI/CD pipelines | `apt-get` |
| Cron jobs | `apt-get` |
| Ansible/Puppet/Chef | Module wraps `apt-get` |
| Checking what's upgradeable | `apt list --upgradeable` |
| Parsing output programmatically | `apt-get` / `apt-cache` / `dpkg-query` |
| Building from source | `apt-get build-dep` |
| Interactive search | `apt search` |
