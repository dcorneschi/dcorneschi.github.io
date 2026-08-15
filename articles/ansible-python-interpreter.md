# Ansible Python Interpreter Discovery — Fixing Warnings and Errors

## The Problem

Ansible 2.21+ requires Python 3.9 or newer on target hosts. When running playbooks, you may see:

**Warning (auto-discovery):**
```
[WARNING]: Host 'server.example.com' is using the discovered Python interpreter at '/usr/bin/python3.9',
but future installation of another Python interpreter could cause a different interpreter to be discovered.
```

**Error (Python too old):**
```
Ansible requires Python 3.9 or newer on the target.
Current version: 3.6.8
```

**Error (no Python 3):**
```
The module interpreter '/usr/bin/python3' was not found.
```

## The Fix

Set `ansible_python_interpreter` explicitly for each host in your inventory:

```ini
[servers]
rhel8.example.com   ansible_host=10.0.0.1  ansible_python_interpreter=/usr/bin/python3.9
rhel9.example.com   ansible_host=10.0.0.2  ansible_python_interpreter=/usr/bin/python3.9
rhel10.example.com  ansible_host=10.0.0.3  ansible_python_interpreter=/usr/bin/python3.12
ubuntu20.example.com ansible_host=10.0.0.4  ansible_python_interpreter=/usr/bin/python3.9
ubuntu22.example.com ansible_host=10.0.0.5  ansible_python_interpreter=/usr/bin/python3.10
ubuntu24.example.com ansible_host=10.0.0.6  ansible_python_interpreter=/usr/bin/python3.12
```

This tells Ansible exactly which Python to use — no auto-discovery, no warnings.

## Python Paths by OS

| OS | Default Python | Path |
|----|---------------|------|
| RHEL 7 | 2.7 | `/usr/bin/python2.7` |
| RHEL 8 | 3.6 | `/usr/libexec/platform-python` |
| RHEL 9 | 3.9 | `/usr/bin/python3.9` |
| RHEL 10 | 3.12 | `/usr/bin/python3.12` |
| Ubuntu 20.04 | 3.8 | `/usr/bin/python3` |
| Ubuntu 22.04 | 3.10 | `/usr/bin/python3.10` |
| Ubuntu 24.04 | 3.12 | `/usr/bin/python3.12` |

## Ansible Version Compatibility

| Ansible Core | Minimum Target Python |
|--------------|----------------------|
| 2.14 | Python 2.7 / 3.5+ |
| 2.16 | Python 2.7 / 3.6+ |
| 2.17 | Python 3.7+ |
| 2.21 | Python 3.9+ |

## Installing Python 3.9+ on Older Hosts

### RHEL 7

```bash
yum install -y centos-release-scl
yum install -y rh-python39
```
Path: `/opt/rh/rh-python39/root/usr/bin/python3.9`

### RHEL 8

```bash
dnf install -y python39
```
Path: `/usr/bin/python3.9`

### Ubuntu 20.04

```bash
apt install -y software-properties-common
add-apt-repository -y ppa:deadsnakes/ppa
apt update
apt install -y python3.9
```
Path: `/usr/bin/python3.9`

## Finding the Python Path on a Host

```bash
# Check which python3 versions are available
ls /usr/bin/python3*

# Or find the exact path
which python3.9
which python3.12
```

## Alternative: Set Interpreter Per Group

Instead of per-host, set it per group if all hosts in a group share the same Python version:

```ini
[rhel9]
server1.example.com  ansible_host=10.0.0.1
server2.example.com  ansible_host=10.0.0.2

[rhel9:vars]
ansible_python_interpreter=/usr/bin/python3.9
```

## Alternative: ansible.cfg

Disable the warning globally (not recommended — hides real issues):

```ini
[defaults]
interpreter_python=auto_silent
```

This keeps auto-discovery but suppresses the warning. The risk is that a newly installed Python could change behavior unexpectedly.
