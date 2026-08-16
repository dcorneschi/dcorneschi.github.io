# Default Python Versions by OS

## Summary

| OS | Default Python | Path | Python 3.9+ Available? |
|----|---------------|------|----------------------|
| RHEL 7.9 | 2.7.5 | `/usr/bin/python2.7` | No (requires SCL) |
| RHEL 8.10 | 3.6.8 | `/usr/libexec/platform-python` | Yes (`dnf install python39`) |
| RHEL 9.8 | 3.9.x | `/usr/bin/python3.9` | Yes (default) |
| RHEL 10.2 | 3.12.x | `/usr/bin/python3.12` | Yes (default) |
| Ubuntu 20.04 | 3.8.10 | `/usr/bin/python3` | Yes (deadsnakes PPA) |
| Ubuntu 22.04 | 3.10.x | `/usr/bin/python3.10` | Yes (default) |
| Ubuntu 24.04 | 3.12.x | `/usr/bin/python3.12` | Yes (default) |

## RHEL 7

- Ships with **Python 2.7.5** only
- No Python 3 in base repos
- `/usr/bin/python` → Python 2.7
- `/usr/bin/python3` → does not exist

### Installing Python 3.9 on RHEL 7

```bash
# Requires SCL (Software Collections)
yum install -y centos-release-scl
yum install -y rh-python39

# Path: /opt/rh/rh-python39/root/usr/bin/python3.9
# Must enable SCL to use: scl enable rh-python39 bash
```

## RHEL 8

- Ships with **Python 3.6.8** as `platform-python`
- `/usr/bin/python3` → may not exist without explicit install
- `/usr/libexec/platform-python` → Python 3.6 (used by system tools like dnf)
- Python 2.7 available via `python2` package

### Installing Python 3.9 on RHEL 8

```bash
dnf install -y python39

# Path: /usr/bin/python3.9
# Also available: python3.11 (dnf install python3.11)
```

## RHEL 9

- Ships with **Python 3.9.x** as default
- `/usr/bin/python3` → Python 3.9
- `/usr/bin/python3.9` → Python 3.9
- No Python 2

## RHEL 10

- Ships with **Python 3.12.x** as default
- `/usr/bin/python3` → Python 3.12
- `/usr/bin/python3.12` → Python 3.12
- No Python 2

## Ubuntu 20.04 (Focal)

- Ships with **Python 3.8.10**
- `/usr/bin/python3` → Python 3.8
- `/usr/bin/python` → does not exist (install `python-is-python3`)
- No Python 2 by default

### Installing Python 3.9 on Ubuntu 20.04

```bash
apt install -y software-properties-common
add-apt-repository -y ppa:deadsnakes/ppa
apt update
apt install -y python3.9 python3.9-distutils

# Path: /usr/bin/python3.9
# Note: python3-apt only works with system Python 3.8
# Symlink required for Ansible apt module:
ln -sf /usr/lib/python3/dist-packages/apt_pkg.cpython-38-x86_64-linux-gnu.so \
       /usr/lib/python3/dist-packages/apt_pkg.cpython-39-x86_64-linux-gnu.so
```

## Ubuntu 22.04 (Jammy)

- Ships with **Python 3.10.x**
- `/usr/bin/python3` → Python 3.10
- `/usr/bin/python3.10` → Python 3.10
- No Python 2 by default

## Ubuntu 24.04 (Noble)

- Ships with **Python 3.12.x**
- `/usr/bin/python3` → Python 3.12
- `/usr/bin/python3.12` → Python 3.12
- No Python 2 by default

## How to Check Python Version Remotely

```bash
# Check all hosts using raw module (works even without Python 3.9)
ansible all -i inventory.ini -m raw -a "python3 --version 2>/dev/null || python --version 2>/dev/null"

# Check specific Python paths
ansible all -i inventory.ini -m raw -a "ls /usr/bin/python3*"

# Check with gather_facts (requires compatible Python on target)
ansible all -i inventory.ini -m setup -a "filter=ansible_python_version"
```

## OS End of Life Dates

| OS | Standard Support | Extended/Pro Support | Action |
|----|-----------------|---------------------|--------|
| RHEL 7.9 | June 2024 (ended) | June 2028 (ELS) | Plan migration to RHEL 9+ |
| RHEL 8.10 | May 2029 | May 2032 (ELS) | Supported, install Python 3.9 |
| RHEL 9.8 | May 2032 | May 2035 (ELS) | Fully supported |
| RHEL 10.2 | ~2035 | ~2038 (ELS) | Fully supported |
| Ubuntu 20.04 | April 2025 (ended) | April 2030 (Pro/ESM) | Supported with Pro |
| Ubuntu 22.04 | April 2027 | April 2032 (Pro/ESM) | Fully supported |
| Ubuntu 24.04 | April 2029 | April 2034 (Pro/ESM) | Fully supported |

When a host reaches end of standard support, consider:
- Upgrading to the next LTS/major version
- Removing from automation if no longer needed
- Extended support subscriptions add time but not new features

## Ansible Controller Requirements

The machine you run Ansible **from** (your Mac, a CI server, etc.) also has Python requirements:

| Ansible Core | Controller Python Required |
|--------------|---------------------------|
| 2.14 | Python 3.9+ |
| 2.16 | Python 3.10+ |
| 2.17 | Python 3.10+ |
| 2.21 | Python 3.11+ |

### Check your controller Python

```bash
python3 --version
ansible --version | grep "python version"
```

### macOS note

macOS ships with no Python by default (removed in Monterey+). Install via:

```bash
# Homebrew (recommended)
brew install python@3.12

# Or use the Python.org installer
# https://www.python.org/downloads/
```

### Installing Ansible on the controller

```bash
# Latest
pip install ansible-core

# Specific version (for older target compatibility)
pip install ansible-core==2.16.14

# With pipx (isolated)
pipx install ansible-core
pipx install ansible-core==2.16.14
```

## Ansible Compatibility

| Ansible Core | Minimum Target Python | Compatible OS (out of box) |
|--------------|----------------------|---------------------------|
| 2.14 | 2.7 / 3.5+ | All above |
| 2.16 | 2.7 / 3.6+ | All above |
| 2.17 | 3.7+ | RHEL 8+, Ubuntu 20.04+ |
| 2.21 | 3.9+ | RHEL 9+, Ubuntu 22.04+ |

## What This Means for Each Ansible Version

### Ansible 2.14 / 2.16 (supports Python 2.7+)

All hosts work out of the box without any Python upgrades:

| OS | Works? | Interpreter |
|----|--------|-------------|
| RHEL 7 | Yes | `/usr/bin/python2.7` |
| RHEL 8 | Yes | `/usr/libexec/platform-python` |
| RHEL 9 | Yes | `/usr/bin/python3.9` |
| RHEL 10 | Yes | `/usr/bin/python3.12` |
| Ubuntu 20.04 | Yes | `/usr/bin/python3` |
| Ubuntu 22.04 | Yes | `/usr/bin/python3.10` |
| Ubuntu 24.04 | Yes | `/usr/bin/python3.12` |

Best choice if you need to manage older infrastructure without modifications.

### Ansible 2.17 (requires Python 3.7+)

| OS | Works? | Action needed |
|----|--------|---------------|
| RHEL 7 | No | Install Python 3.9 via SCL |
| RHEL 8 | Yes | Use `/usr/libexec/platform-python` (3.6 is technically below 3.7, but `platform-python` often works) |
| RHEL 9 | Yes | None |
| RHEL 10 | Yes | None |
| Ubuntu 20.04 | Yes | None (3.8 >= 3.7) |
| Ubuntu 22.04 | Yes | None |
| Ubuntu 24.04 | Yes | None |

Only RHEL 7 is a problem. Everything else works.

### Ansible 2.21 (requires Python 3.9+)

| OS | Works? | Action needed |
|----|--------|---------------|
| RHEL 7 | No | Install Python 3.9 via SCL (complex, may not work) |
| RHEL 8 | No (default) | `dnf install python39`, set interpreter to `/usr/bin/python3.9` |
| RHEL 9 | Yes | None |
| RHEL 10 | Yes | None |
| Ubuntu 20.04 | No (default) | Install Python 3.9 from deadsnakes PPA + symlink `apt_pkg` |
| Ubuntu 22.04 | Yes | None |
| Ubuntu 24.04 | Yes | None |

Most restrictive. Half the hosts need manual Python upgrades. The `apt` module on Ubuntu 20.04 additionally requires a symlink hack because `python3-apt` is compiled against the system Python 3.8.

### Practical Recommendation

| Scenario | Recommended Ansible |
|----------|-------------------|
| Mixed fleet (RHEL 7–10 + Ubuntu 20–24) | Ansible 2.16 |
| Modern only (RHEL 9+, Ubuntu 22.04+) | Ansible 2.21 |
| RHEL 8+ and Ubuntu 20.04+ | Ansible 2.17 |
| Need latest features, willing to prep hosts | Ansible 2.21 + install Python 3.9 on old hosts |

### Downgrading Ansible (if needed)

```bash
# Install a specific version
pip install ansible-core==2.16.14

# Or use pipx for isolation
pipx install ansible-core==2.16.14
```

## Key Takeaways

- **RHEL 7** is the hardest — Python 3.9 requires SCL which may not be available without CentOS repos
- **RHEL 8** is straightforward — `dnf install python39` from AppStream
- **Ubuntu 20.04** can install Python 3.9 from deadsnakes PPA, but `python3-apt` needs a symlink hack
- **RHEL 9+, Ubuntu 22.04+** — no action needed, Python 3.9+ is the default
- **Controller (your Mac)** also needs Python 3.11+ for Ansible 2.21
- **Plan for EOL** — RHEL 7 and Ubuntu 20.04 standard support has ended, consider upgrading
