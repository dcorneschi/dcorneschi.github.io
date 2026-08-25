# tuned-adm Cheatsheet

`tuned` is a dynamic tuning daemon that adapts the operating system to the current workload using pre-defined profiles. It adjusts CPU governor, I/O scheduler, kernel parameters, disk readahead, and power settings — all without reboot.

On RHEL 7+, `tuned` is enabled by default and auto-selects a profile at install time based on detected hardware (physical vs virtual, server vs laptop).

## Quick Reference

```bash
# Show active profile
tuned-adm active

# Get recommended profile for this system
tuned-adm recommend

# List available profiles
tuned-adm list

# Switch to a profile
tuned-adm profile throughput-performance

# Disable tuning (remove all tuned settings)
tuned-adm off

# Verify current profile settings match what's applied
tuned-adm verify

# Show profile info
tuned-adm profile_info throughput-performance
```

## How It Works

```
┌──────────────────────────┐
│     tuned daemon         │
│                          │
│  Reads profile from:     │
│  /usr/lib/tuned/<name>/  │
│  /etc/tuned/<name>/      │
│                          │
│  Applies:                │
│  - sysctl values         │
│  - CPU governor          │
│  - I/O scheduler         │
│  - disk readahead        │
│  - THP settings          │
│  - power management      │
│  - network tunings       │
└──────────────────────────┘
```

- `tuned` runs as a systemd service
- Profiles define settings declaratively in `tuned.conf`
- Profiles can inherit from other profiles (child overrides parent)
- Changes are applied dynamically — no reboot needed
- When `tuned` stops or is disabled, settings revert to defaults

## Available Profiles

| Profile | Use Case |
|---------|----------|
| `balanced` | Compromise between performance and power saving |
| `desktop` | Desktop workloads, fast response |
| `latency-performance` | Low-latency, disables power saving |
| `network-latency` | Low network latency (extends latency-performance) |
| `network-throughput` | High network throughput |
| `throughput-performance` | High throughput, server workloads |
| `virtual-guest` | Optimized for VMs (default on virtual machines) |
| `virtual-host` | Optimized for KVM/libvirt hypervisors |
| `powersave` | Maximum power saving |
| `oracle` | Oracle RDBMS (extends throughput-performance) |
| `mssql` | Microsoft SQL Server on Linux |
| `sap-netweaver` | SAP NetWeaver |
| `sap-hana` | SAP HANA (disables THP, NUMA optimized) |
| `aws` | Amazon EC2 instances (available via `tuned-profiles-aws`) |

```bash
# List all profiles with descriptions
tuned-adm list

# See what a profile actually does
cat /usr/lib/tuned/throughput-performance/tuned.conf
```

## Profile Directories

| Path | Purpose |
|------|---------|
| `/usr/lib/tuned/` | Default profiles shipped by Red Hat (don't edit) |
| `/etc/tuned/` | Custom profiles and overrides (admin-created) |
| `/etc/tuned/active_profile` | Currently active profile name |
| `/etc/tuned/profile_mode` | Whether profile was set manually or auto |
| `/etc/tuned/tuned-main.conf` | Main tuned daemon configuration |

Profiles in `/etc/tuned/` take precedence over same-named profiles in `/usr/lib/tuned/`.

## Creating Custom Profiles

Custom profiles support inheritance — inherit an existing profile and override only what you need.

### Example: virtual-guest Without THP

Some workloads (SAP HANA, databases, HPC) perform worse with Transparent Huge Pages. Create a profile that inherits `virtual-guest` but disables THP:

```bash
mkdir /etc/tuned/virtual-guest-no-thp
cat > /etc/tuned/virtual-guest-no-thp/tuned.conf << 'EOF'
[main]
include=virtual-guest

[vm]
transparent_hugepages=never
EOF

# Activate
tuned-adm profile virtual-guest-no-thp

# Verify
tuned-adm active
cat /sys/kernel/mm/transparent_hugepage/enabled
# Should show: always madvise [never]
```

### Example: Custom Throughput Profile with sysctl

```bash
mkdir /etc/tuned/my-throughput
cat > /etc/tuned/my-throughput/tuned.conf << 'EOF'
[main]
include=throughput-performance

[sysctl]
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.ipv4.tcp_rmem=4096 87380 16777216
net.ipv4.tcp_wmem=4096 65536 16777216
vm.swappiness=10

[disk]
readahead=4096
EOF

tuned-adm profile my-throughput
tuned-adm verify
```

### Example: Custom Profile with Script

```bash
mkdir /etc/tuned/my-custom
cat > /etc/tuned/my-custom/tuned.conf << 'EOF'
[main]
include=throughput-performance

[script]
script=script.sh
EOF

cat > /etc/tuned/my-custom/script.sh << 'EOF'
#!/bin/bash
# Custom tuning script executed when profile is activated
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo 1 > /proc/sys/net/ipv4/tcp_low_latency
EOF

chmod +x /etc/tuned/my-custom/script.sh
tuned-adm profile my-custom
```

## Profile Configuration Sections

A `tuned.conf` file can contain these sections:

| Section | Controls |
|---------|----------|
| `[main]` | Inheritance (`include=parent_profile`) |
| `[cpu]` | CPU governor, energy_perf_bias, min/max frequency |
| `[disk]` | I/O scheduler, readahead, APM level |
| `[vm]` | Transparent huge pages |
| `[sysctl]` | Kernel sysctl parameters |
| `[net]` | Wake-on-LAN, coalesce settings |
| `[audio]` | Audio power saving |
| `[video]` | GPU power saving |
| `[script]` | Custom shell script to run |
| `[bootloader]` | Kernel command-line parameters (requires reboot) |
| `[scheduler]` | Process scheduler tuning |
| `[selinux]` | SELinux mode |

### Example Section Content

```ini
[cpu]
governor=performance
energy_perf_bias=performance
min_perf_pct=100

[disk]
elevator=deadline
readahead=4096

[vm]
transparent_hugepages=never

[sysctl]
kernel.sched_min_granularity_ns=10000000
kernel.sched_wakeup_granularity_ns=15000000
vm.dirty_ratio=40
vm.dirty_background_ratio=10
net.core.busy_poll=50
net.core.busy_read=50
```

## Managing the tuned Service

```bash
# Check service status
systemctl status tuned

# Start and enable (should already be enabled by default)
systemctl enable --now tuned

# Stop (reverts tuning settings)
systemctl stop tuned

# Disable permanently
tuned-adm off
systemctl disable tuned

# Restart after config changes
systemctl restart tuned
```

## Checking What's Applied

```bash
# Show active profile
tuned-adm active

# Verify settings match the profile
tuned-adm verify
# Returns "Verification succeeded" or lists mismatches

# Check specific settings
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/block/sda/queue/scheduler
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
sysctl vm.swappiness
```

## Common Tasks

### Check and Change THP (Transparent Huge Pages)

```bash
# Current THP status
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never — brackets show active setting

# Disable THP temporarily (lost on reboot or profile change)
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Disable THP permanently via tuned profile (survives reboot)
# Create custom profile as shown above
```

> Workloads where THP should be disabled: SAP HANA, Oracle DB, Redis, MongoDB, Hadoop, Splunk, many JVM-based applications.

### Switch Profile for Different Workloads

```bash
# Database server
tuned-adm profile throughput-performance

# Network appliance / proxy
tuned-adm profile network-latency

# KVM hypervisor
tuned-adm profile virtual-host

# VM guest (auto-selected if virt detected)
tuned-adm profile virtual-guest

# Power saving (laptops, idle servers)
tuned-adm profile powersave

# Back to auto-recommended
tuned-adm profile $(tuned-adm recommend)
```

### Use tuned with Ansible (RHEL System Roles)

```yaml
- name: Apply tuned profile
  hosts: databases
  become: yes
  tasks:
    - name: Ensure tuned is running
      service:
        name: tuned
        state: started
        enabled: yes

    - name: Set throughput-performance profile
      command: tuned-adm profile throughput-performance
      changed_when: true
```

Or using the `rhel-system-roles.tuned` role (RHEL 8+):

```yaml
- name: Configure tuned
  hosts: all
  become: yes
  vars:
    tuned_profile: "throughput-performance"
  roles:
    - rhel-system-roles.tuned
```

## Dynamic Tuning

tuned can monitor system activity in real time and adjust settings dynamically (disabled by default):

```bash
# Enable dynamic tuning in /etc/tuned/tuned-main.conf
# dynamic_tuning = 1
# update_interval = 10

# This enables tuned to:
# - Adjust CPU governor based on load
# - Change disk readahead based on I/O patterns
# - Modify power settings based on activity
```

> Dynamic tuning adds CPU overhead from monitoring. For most production servers with predictable workloads, static profiles are preferred.

## Troubleshooting

```bash
# Check tuned logs
journalctl -u tuned

# Verbose mode
tuned-adm --debug profile throughput-performance

# Verify profile is correctly applied
tuned-adm verify
# If mismatched, something else is overriding tuned (sysctl.conf, other tools)

# Check if another tool conflicts
# Common conflicts: sysctl.conf, /etc/rc.local, other daemons
grep -r "transparent_hugepage" /etc/sysctl.conf /etc/sysctl.d/

# Profile not appearing in list
ls /usr/lib/tuned/ /etc/tuned/
# Custom profiles must have a tuned.conf file inside their directory
```

| Symptom | Cause | Fix |
|---------|-------|-----|
| `No current active profile` | tuned stopped or `tuned-adm off` was run | `systemctl start tuned && tuned-adm profile <name>` |
| Profile applied but settings not matching | Another tool overriding (sysctl.conf) | Remove conflicting entries, let tuned manage them |
| Custom profile not in list | Missing `tuned.conf` in profile directory | Create the file with at least `[main]` section |
| `Verification failed` | Settings were manually changed after profile applied | `tuned-adm profile <name>` to re-apply |

## Important Notes

- `tuned` integrates with Cockpit — you can change profiles from the web UI
- On RHEL 8+, `tuned` profiles also support `[bootloader]` section for kernel cmdline changes (requires reboot)
- `tuned-adm recommend` logic: detects if running in a VM (→ virtual-guest), if it's a laptop (→ balanced), or bare metal server (→ throughput-performance)
- Use `sysctl.conf` for settings tuned doesn't manage; use tuned for everything it supports (avoids conflicts)
