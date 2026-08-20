# SELinux Cheatsheet

Security-Enhanced Linux (SELinux) provides mandatory access control (MAC) on top of standard Linux permissions. This guide covers modes, contexts, booleans, troubleshooting, and policy management on RHEL-based systems.

## Modes

| Mode | Behavior |
|------|----------|
| Enforcing | Policies enforced, violations denied and logged |
| Permissive | Policies not enforced, violations logged only |
| Disabled | SELinux completely off (requires reboot to re-enable) |

### Check Current Mode

```bash
# Current mode
getenforce

# Detailed status
sestatus

# From config file
cat /etc/selinux/config
```

### Change Mode

```bash
# Switch to permissive (temporary, until reboot)
sudo setenforce 0

# Switch to enforcing (temporary)
sudo setenforce 1

# Change permanently (requires reboot)
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# Note: /etc/sysconfig/selinux is a symlink to /etc/selinux/config on RHEL
# Both paths work:
sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/sysconfig/selinux
```

> **Warning:** Switching from disabled to enforcing requires a full filesystem relabel. Add a `/.autorelabel` file before rebooting, or boot to permissive first and run `fixfiles relabel`.

## Contexts (Labels)

Every file, process, port, and user has an SELinux context in the format:

```
user:role:type:level
```

For most administration, only the **type** matters (e.g., `httpd_sys_content_t`).

### View Contexts

```bash
# Files
ls -Z /var/www/html/
ls -lZ /etc/httpd/

# Processes
ps -eZ | grep httpd
ps -efZ                     # Full format with context
ps auxZ | grep nginx

# Current user context
id -Z

# Ports
semanage port -l | grep http

# Login mappings
semanage login -l
```

### Change File Context

```bash
# Change context temporarily (lost on relabel)
chcon -t httpd_sys_content_t /var/www/html/index.html

# Change recursively
chcon -R -t httpd_sys_content_t /var/www/html/

# Set user only
chcon -u system_u /path/to/file

# Set role only
chcon -r object_r /path/to/file

# Set type only
chcon -t httpd_sys_content_t /path/to/file

# Change user, role, and type together
chcon -u system_u -r object_r -t httpd_sys_content_t /var/www/html/file

# Copy context from a reference file
chcon --reference=/var/www/html/index.html /var/www/html/newfile.html

# Restore to default context (from policy)
restorecon -v /var/www/html/index.html

# Restore recursively
restorecon -Rv /var/www/html/

# Restore entire filesystem
fixfiles relabel
# Or touch /.autorelabel and reboot

# Check for incorrect labels (dry run — no changes)
fixfiles check

# Restore only incorrectly labeled files (without full relabel)
fixfiles restore
```

### Set Default Context (Persistent)

```bash
# Add a file context rule (permanent)
semanage fcontext -a -t httpd_sys_content_t "/srv/www(/.*)?"

# Apply the new rule
restorecon -Rv /srv/www

# List all file context rules
semanage fcontext -l

# List custom (locally defined) rules only
semanage fcontext -l -C

# Delete a custom rule
semanage fcontext -d -t httpd_sys_content_t "/srv/www(/.*)?"
```

### Common File Types

| Type | Used For |
|------|----------|
| `httpd_sys_content_t` | Web server static content (read-only) |
| `httpd_sys_rw_content_t` | Web server writable content |
| `httpd_log_t` | Apache/Nginx log files |
| `samba_share_t` | Samba shared files |
| `nfs_t` | NFS exported files |
| `home_dir_t` | User home directories |
| `var_log_t` | Log files in /var/log |
| `tmp_t` | Temporary files |
| `container_file_t` | Container/Podman volumes |
| `svirt_sandbox_file_t` | Docker/container files |

## Booleans

Booleans are on/off switches that control optional SELinux policy features.

### List Booleans

```bash
# List all booleans
getsebool -a

# List with descriptions
semanage boolean -l

# Filter for specific service
getsebool -a | grep httpd
getsebool -a | grep samba
getsebool -a | grep nfs
```

### Set Booleans

```bash
# Set temporarily (lost on reboot)
setsebool httpd_can_network_connect on

# Set permanently
setsebool -P httpd_can_network_connect on

# Set multiple at once
setsebool -P httpd_can_network_connect=on httpd_can_sendmail=on
```

### Common Booleans

| Boolean | Purpose |
|---------|---------|
| `httpd_can_network_connect` | Allow Apache/Nginx to make outbound connections |
| `httpd_can_network_connect_db` | Allow web server to connect to databases |
| `httpd_can_sendmail` | Allow web server to send mail |
| `httpd_enable_homedirs` | Allow web server to serve user home directories |
| `httpd_use_nfs` | Allow web server to access NFS mounts |
| `httpd_execmem` | Allow web server to execute memory (PHP, etc.) |
| `samba_enable_home_dirs` | Allow Samba to share user home directories |
| `samba_export_all_rw` | Allow Samba to export any file read-write |
| `nfs_export_all_rw` | Allow NFS to export any file read-write |
| `allow_user_mysql_connect` | Allow users to connect to MySQL |
| `container_manage_cgroup` | Allow containers to manage cgroups |
| `virt_sandbox_use_all_caps` | Allow VMs to use all capabilities |
| `domain_can_mmap_files` | Allow all domains to mmap files |

## Port Labels

### View Port Labels

```bash
# List all port labels
semanage port -l

# Filter for a service
semanage port -l | grep http
semanage port -l | grep ssh
semanage port -l | grep mysql
```

### Add/Modify Port Labels

```bash
# Allow httpd to listen on port 8443
semanage port -a -t http_port_t -p tcp 8443

# Allow httpd to listen on a range
semanage port -a -t http_port_t -p tcp 8000-8100

# Modify existing port label
semanage port -m -t http_port_t -p tcp 9090

# Delete custom port label
semanage port -d -t http_port_t -p tcp 8443

# List custom port rules only
semanage port -l -C
```

### Common Port Types

| Type | Ports |
|------|-------|
| `http_port_t` | 80, 443, 8080, etc. |
| `ssh_port_t` | 22 |
| `mysqld_port_t` | 3306 |
| `postgresql_port_t` | 5432 |
| `smtp_port_t` | 25, 587 |
| `dns_port_t` | 53 |
| `nfs_port_t` | 2049 |

## Troubleshooting

### Finding Denials

```bash
# View recent denials in audit log
ausearch -m avc -ts recent

# View today's denials
ausearch -m avc -ts today

# View denials for a specific process
ausearch -m avc -c httpd

# Grep the audit log directly
grep "denied" /var/log/audit/audit.log

# Human-readable with sealert (requires setroubleshoot)
sealert -a /var/log/audit/audit.log

# Real-time monitoring
tail -f /var/log/audit/audit.log | grep denied
```

### Interpret AVC Messages

```
type=AVC msg=audit(1234567890.123:456): avc:  denied  { read } for  pid=1234 comm="httpd" name="index.html" dev="sda1" ino=789 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=0
```

| Field | Meaning |
|-------|---------|
| `denied { read }` | The action that was denied |
| `comm="httpd"` | The process name |
| `name="index.html"` | The target file |
| `scontext=...httpd_t...` | Source context (the process) |
| `tcontext=...default_t...` | Target context (the file) |
| `tclass=file` | The object class |

### Generate and Apply Policy Fixes

```bash
# Install troubleshooting tools
sudo dnf install -y setroubleshoot-server setools-console

# Generate a policy module from denials
ausearch -m avc -ts recent | audit2allow -M mypolicy

# Review the generated policy before applying
cat mypolicy.te

# Install the policy module
semodule -i mypolicy.pp

# One-liner: generate and install
ausearch -m avc -ts recent | audit2allow -M mypolicy && semodule -i mypolicy.pp
```

### audit2why — Explain Why Access Was Denied

```bash
# Explain recent denials
ausearch -m avc -ts recent | audit2why

# Common outputs:
# "seboolean" → a boolean needs to be enabled
# "allow" → a policy rule is missing
# "constraint" → a policy constraint blocks access
```

### Common Fixes

```bash
# Fix: wrong file context
restorecon -Rv /path/to/files

# Fix: boolean needs enabling
setsebool -P httpd_can_network_connect on

# Fix: non-standard port
semanage port -a -t http_port_t -p tcp 8080

# Fix: custom file location needs context
semanage fcontext -a -t httpd_sys_content_t "/custom/path(/.*)?"
restorecon -Rv /custom/path
```

## Policy Modules

### List Modules

```bash
# List all loaded modules
semodule -l

# List with priority
semodule -lfull

# Check if a specific module is loaded
semodule -l | grep mymodule
```

### Manage Modules

```bash
# Install a module
semodule -i mymodule.pp

# Remove a module
semodule -r mymodule

# Disable a module (keep installed)
semodule -d mymodule

# Enable a disabled module
semodule -e mymodule

# Export the current policy
semodule -E
```

### Create Custom Module

```bash
# 1. Set permissive, reproduce the action, collect denials
sudo setenforce 0
# ... do the thing that gets denied ...
sudo setenforce 1

# 2. Generate module from collected denials
ausearch -m avc -ts recent | audit2allow -M mycustom

# 3. Review the .te file
cat mycustom.te

# 4. Install
semodule -i mycustom.pp
```

## User Mappings

```bash
# List SELinux user mappings
semanage login -l

# Map a Linux user to an SELinux user
semanage login -a -s staff_u username

# Map all unlisted users to a restricted SELinux user
semanage login -m -s user_u __default__

# Delete a mapping
semanage login -d username

# List SELinux users
semanage user -l
```

## Network

```bash
# Allow a domain to connect to a network port
semanage port -a -t http_port_t -p tcp 8080

# Check if a boolean allows network access
getsebool httpd_can_network_connect

# Allow NFS home directories
setsebool -P use_nfs_home_dirs on

# Allow Samba home directories
setsebool -P samba_enable_home_dirs on
```

## Containers (Podman/Docker)

```bash
# Label a volume for container use
chcon -Rt svirt_sandbox_file_t /path/to/volume

# Or use the :Z/:z bind mount option (auto-relabel)
podman run -v /host/path:/container/path:Z myimage

# Allow containers to manage cgroups
setsebool -P container_manage_cgroup on

# Set context for container data
semanage fcontext -a -t container_file_t "/var/containers(/.*)?"
restorecon -Rv /var/containers
```

## Filesystem Relabel

```bash
# Relabel entire filesystem (takes time on large systems)
fixfiles relabel

# Or create autorelabel file and reboot
touch /.autorelabel
reboot

# Relabel on next boot (alternative)
fixfiles -F onboot

# Check relabel progress (during boot)
journalctl -b | grep relabel
```

## Useful Queries with seinfo/sesearch

```bash
# Install setools
sudo dnf install -y setools-console

# List all types
seinfo -t

# List all types for a domain
seinfo -t | grep httpd

# List all booleans
seinfo -b

# Search what a type can do
sesearch --allow -s httpd_t

# Search what can access a target type
sesearch --allow -t httpd_sys_content_t

# Search for specific permission
sesearch --allow -s httpd_t -t httpd_sys_content_t -p read
```

### Check Status (Verbose)

```bash
sestatus -v                 # Verbose with process and file contexts
```

### View Context with stat

```bash
stat -c "%n %C" /var/www/html/index.html
```

### Copy Context from Another File

```bash
chcon --reference=/var/www/html/index.html /var/www/html/newfile.html
```

### Modify or Delete File Context Rules

```bash
# Modify an existing rule
semanage fcontext -m -t httpd_sys_rw_content_t "/srv/uploads(/.*)?"

# Delete a custom rule
semanage fcontext -d "/srv/uploads(/.*)?"
```

## Role Management

```bash
# List roles (requires setools-console)
seinfo -r

# List SELinux users and their assigned roles
semanage user -l

# Assign roles to an SELinux user
semanage user -m -R "staff_r sysadm_r" staff_u
```

## Type Queries (seinfo / sesearch)

```bash
# Install setools if not present
sudo dnf install -y setools-console

# List all types
seinfo -t

# List types matching a pattern
seinfo -t | grep httpd

# What can a source type access?
sesearch --allow --source httpd_t

# What can access a target type?
sesearch --allow --target httpd_sys_content_t

# Show type transitions
sesearch --type_trans --source unconfined_t

# Show domain transitions
sesearch --type_trans --source init_t --target httpd_exec_t
```

## Create Policy from Scratch

```bash
# Write a type enforcement (.te) file
cat > my_app.te << 'EOF'
policy_module(my_app, 1.0.0)
type my_app_t;
type my_app_exec_t;
domain_type(my_app_t)
EOF

# Compile the module
checkmodule -M -m -o my_app.mod my_app.te

# Package it
semodule_package -o my_app.pp -m my_app.mod

# Install
semodule -i my_app.pp
```

## Find Files with Wrong Context

```bash
# Find files under /var/www that don't have the expected context
find /var/www -type f ! -context "*:httpd_sys_content_t:*"

# Find files with unlabeled_t (common after moving files)
find / -context "*:unlabeled_t:*" 2>/dev/null
```

## Important Files and Directories

| Path | Purpose |
|------|---------|
| `/etc/selinux/config` | Main SELinux configuration (mode, policy type) |
| `/var/log/audit/audit.log` | Audit log with AVC denials |
| `/etc/selinux/targeted/` | Targeted policy files |
| `/usr/share/selinux/` | SELinux policy packages |
| `/sys/fs/selinux/` | SELinux pseudo-filesystem (runtime) |

## Mode Transitions

```
disabled → permissive:  Edit /etc/selinux/config, reboot (relabel needed)
permissive → enforcing: setenforce 1 (or edit config + reboot)
enforcing → permissive: setenforce 0
enforcing → disabled:   Edit config + reboot
disabled → enforcing:   Edit config, touch /.autorelabel, reboot
```

> **Note:** Going from disabled to enforcing without relabeling will cause widespread denials. Always relabel first.

## Debugging Workflow

1. **Check status:** `sestatus`
2. **Find denials:** `ausearch -m avc -ts recent`
3. **Analyze:** `audit2why` or `audit2allow -a`
4. **Fix via:** boolean adjustment, context change, or custom policy
5. **Verify:** Run operation again and check audit log
6. **Make permanent:** `setsebool -P`, `semanage fcontext`, or `semodule -i`

## Quick Reference

| Action | Command |
|--------|---------|
| Check mode | `getenforce` |
| Detailed status | `sestatus` |
| Set permissive (temp) | `setenforce 0` |
| Set enforcing (temp) | `setenforce 1` |
| View file context | `ls -Z /path` |
| View process context | `ps -eZ \| grep name` |
| Restore context | `restorecon -Rv /path` |
| Set permanent context | `semanage fcontext -a -t type "/path(/.*)?"` |
| List booleans | `getsebool -a` |
| Set boolean | `setsebool -P name on` |
| Add port | `semanage port -a -t type -p tcp port` |
| View denials | `ausearch -m avc -ts recent` |
| Explain denial | `ausearch -m avc -ts recent \| audit2why` |
| Generate fix | `ausearch -m avc -ts recent \| audit2allow -M name` |
| Install module | `semodule -i name.pp` |
| Full relabel | `fixfiles relabel` |
