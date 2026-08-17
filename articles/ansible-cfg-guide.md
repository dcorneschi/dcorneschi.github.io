# Ansible Configuration: ansible.cfg Guide

How Ansible finds its configuration, what sections and options are available, and how to use global vs project-level config files to control behavior across different environments.

## Configuration File Precedence

Ansible checks for configuration in this order (first match wins):

| Priority | Location | Scope |
|----------|----------|-------|
| 1 (highest) | `ANSIBLE_CONFIG` environment variable | Any path you specify |
| 2 | `./ansible.cfg` | Current working directory (project-level) |
| 3 | `~/.ansible.cfg` | User home directory (user-level) |
| 4 (lowest) | `/etc/ansible/ansible.cfg` | System-wide (global) |

```bash
# Check which config file Ansible is currently using
ansible --version
# Look for "config file" in the output

# Or explicitly
ansible-config dump --only-changed
```

> **Security note:** Ansible ignores `./ansible.cfg` if the current directory is world-writable. This prevents privilege escalation from shared `/tmp`-style directories.

## Global vs Local: When to Use Which

### Global — `/etc/ansible/ansible.cfg`

Use for system-wide defaults that apply to all users and all projects on a machine. Typical for shared jump hosts or centralized automation servers.

```ini
# /etc/ansible/ansible.cfg
[defaults]
remote_user = ansible
host_key_checking = False
log_path = /var/log/ansible.log
```

### User-Level — `~/.ansible.cfg`

Use for personal preferences that apply across all your projects but don't affect other users.

```ini
# ~/.ansible.cfg
[defaults]
vault_password_file = ~/.vault_pass
stdout_callback = yaml
```

### Project-Level — `./ansible.cfg` (Recommended)

Use for project-specific settings. This is the most common approach — check it into git with the rest of your Ansible code.

```ini
# ./ansible.cfg (in your project root)
[defaults]
inventory = inventory/
roles_path = roles/
collections_paths = collections/
vault_password_file = .vault_pass
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
```

### Environment Variable Override

Use for CI/CD pipelines, containers, or temporary overrides.

```bash
export ANSIBLE_CONFIG=/opt/automation/production.cfg
ansible-playbook deploy.yml
```

## File Structure

`ansible.cfg` uses INI format with sections in brackets:

```ini
[section_name]
key = value
key2 = value2
```

Boolean values accept: `true`/`false`, `yes`/`no`, `1`/`0`.

## [defaults] Section

The most commonly used section — controls general Ansible behavior.

### Inventory and Paths

```ini
[defaults]
# Inventory file or directory
inventory = ./inventory/hosts.ini

# Colon-separated list of roles paths
roles_path = ./roles:/usr/share/ansible/roles:/etc/ansible/roles

# Colon-separated list of collections paths
collections_paths = ./collections:~/.ansible/collections

# Module library path (custom modules)
library = ./library

# Plugins paths
filter_plugins = ./plugins/filter
callback_plugins = ./plugins/callback
lookup_plugins = ./plugins/lookup
```

### Connection Settings

```ini
[defaults]
# Default remote user
remote_user = deploy

# Default module to use for ad-hoc commands
module_name = command

# Disable host key checking (useful for ephemeral cloud instances)
host_key_checking = False

# Number of parallel processes
forks = 20

# Connection timeout (seconds)
timeout = 30

# Default transport (ssh, paramiko, local, docker, etc.)
transport = ssh
```

### Output and Logging

```ini
[defaults]
# Stdout callback plugin (yaml is more readable than default)
stdout_callback = yaml

# Additional callback plugins to enable
callbacks_enabled = timer, profile_tasks

# Log all output to a file
log_path = /var/log/ansible/ansible.log

# Show custom stats at the end of a play
show_custom_stats = True

# Display skipped hosts
display_skipped_hosts = False

# Display ok hosts
display_ok_hosts = True

# Reduce output noise
deprecation_warnings = False
system_warnings = True
```

### Fact Gathering and Caching

```ini
[defaults]
# Fact gathering behavior: implicit (default), explicit, smart
gathering = smart

# Fact caching plugin
fact_caching = jsonfile

# Path for jsonfile cache
fact_caching_connection = /tmp/ansible_facts_cache

# Cache timeout in seconds (3600 = 1 hour)
fact_caching_timeout = 3600

# Gather subset (limits what facts are collected)
gather_subset = !hardware,network,virtual
```

`gathering` values:
- `implicit` — gather facts at the start of every play (default)
- `explicit` — only gather when `gather_facts: true` is set in the play
- `smart` — gather only if facts are not already cached

### Vault

```ini
[defaults]
# Path to a file containing the vault password
vault_password_file = ~/.vault_pass

# Vault identity list (multiple vaults)
vault_identity_list = dev@~/.vault_pass_dev, prod@~/.vault_pass_prod
```

### Retry and Error Handling

```ini
[defaults]
# Disable .retry files on failed playbook runs
retry_files_enabled = False

# Where to save retry files (if enabled)
retry_files_save_path = ~/.ansible-retry

# Any errors from a module set this host as unreachable for remaining tasks
any_errors_fatal = False
```

### Miscellaneous

```ini
[defaults]
# Hash behavior when combining variables: replace (default) or merge
hash_behaviour = replace

# Allow Jinja2 extensions
jinja2_extensions = jinja2.ext.do,jinja2.ext.i18n

# Interpreter discovery (auto, auto_legacy, auto_silent, or explicit path)
interpreter_python = auto_silent

# Enable/disable cowsay (yes, no)
nocows = True

# Executable for ad-hoc commands
executable = /bin/bash
```

## [privilege_escalation] Section

Controls sudo/become behavior.

```ini
[privilege_escalation]
# Enable privilege escalation by default
become = True

# Escalation method: sudo, su, pbrun, pfexec, doas, dzdo, ksu, runas
become_method = sudo

# Target user for escalation
become_user = root

# Whether to prompt for the become password
become_ask_pass = False

# Flags passed to the become method
become_flags = -H -S -n
```

## [ssh_connection] Section

Tuning SSH connections for performance and reliability.

```ini
[ssh_connection]
# Enable SSH pipelining (major speed improvement)
# Requires requiretty to be disabled in sudoers on target
pipelining = True

# SSH arguments
ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s

# Path for SSH control sockets
control_path_dir = ~/.ansible/cp

# Control path format
control_path = %(directory)s/%%h-%%r

# Enable SCP for file transfer instead of SFTP
# Options: smart (default), sftp, scp, piped
transfer_method = smart

# Number of SSH retries
retries = 3

# Use SSH for SFTP operations
sftp_batch_mode = True

# Timeout for SSH persistent connections
timeout = 30
```

### SSH Pipelining Explained

Pipelining reduces the number of SSH connections Ansible makes. Without it, Ansible opens a new connection for each operation (copy module, execute, fetch result). With pipelining, it reuses a single connection.

```ini
[ssh_connection]
pipelining = True
ssh_args = -C -o ControlMaster=auto -o ControlPersist=300s
```

> **Requirement:** The target host's `/etc/sudoers` must NOT have `requiretty` enabled. Check with: `grep requiretty /etc/sudoers`. If present, remove it or add `Defaults:ansible !requiretty`.

## [persistent_connection] Section

For network devices and long-running connections.

```ini
[persistent_connection]
# Connection timeout (seconds)
connect_timeout = 30

# Idle timeout before disconnecting
command_timeout = 30

# Retry count
connect_retry_timeout = 15
```

## [colors] Section

Customize output colors in the terminal.

```ini
[colors]
highlight = white
verbose = blue
warn = bright purple
error = red
debug = dark gray
deprecate = purple
skip = cyan
unreachable = red
ok = green
changed = yellow
diff_add = green
diff_remove = red
diff_lines = cyan
```

## [galaxy] Section

Configure Ansible Galaxy behavior.

```ini
[galaxy]
# Galaxy server URL
server_list = release_galaxy

[galaxy_server.release_galaxy]
url = https://galaxy.ansible.com/
token_path = /api/v3/
```

## [inventory] Section

Controls inventory plugin behavior.

```ini
[inventory]
# Enable specific inventory plugins
enable_plugins = host_list, script, auto, yaml, ini, toml

# Ignore extensions when parsing inventory directories
ignore_extensions = .pyc, .pyo, .swp, .bak, ~, .rpm, .md, .txt

# Treat unparsed inventory sources as fatal
unparsed_is_failed = True

# Cache inventory
cache = True
cache_plugin = jsonfile
cache_connection = /tmp/ansible_inventory_cache
cache_timeout = 3600
```

## Full Production Example

A well-tuned `ansible.cfg` for a production environment:

```ini
[defaults]
inventory = inventory/
roles_path = roles/
collections_paths = collections/
remote_user = deploy
forks = 30
timeout = 30
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml
callbacks_enabled = timer, profile_tasks
interpreter_python = auto_silent
gathering = smart
fact_caching = jsonfile
fact_caching_connection = .facts_cache/
fact_caching_timeout = 7200
vault_password_file = .vault_pass
deprecation_warnings = False
display_skipped_hosts = False
any_errors_fatal = False
nocows = True

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True
ssh_args = -C -o ControlMaster=auto -o ControlPersist=300s -o ServerAliveInterval=30
retries = 3

[inventory]
enable_plugins = yaml, ini, auto
unparsed_is_failed = True
```

## Minimal Project Example

A simple starting point for small projects:

```ini
[defaults]
inventory = hosts.ini
remote_user = admin
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
```

## Environment Variable Overrides

Every `ansible.cfg` option can be overridden with an environment variable. The naming pattern is `ANSIBLE_<SECTION>_<KEY>` (uppercase, with underscores):

```bash
# Override forks
export ANSIBLE_FORKS=50

# Override remote user
export ANSIBLE_REMOTE_USER=deploy

# Disable host key checking
export ANSIBLE_HOST_KEY_CHECKING=False

# Enable pipelining
export ANSIBLE_PIPELINING=True

# Set inventory
export ANSIBLE_INVENTORY=/path/to/inventory

# Set vault password file
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass

# Set stdout callback
export ANSIBLE_STDOUT_CALLBACK=yaml
```

> **Note:** Environment variables always override config file values, regardless of which config file is being used.

## Useful Commands

```bash
# Show current config file location and all settings
ansible-config view

# List all configuration options with descriptions
ansible-config list

# Show only settings that differ from defaults
ansible-config dump --only-changed

# Show config for a specific setting
ansible-config dump | grep FORKS

# Validate your config (check for deprecated options)
ansible-config validate
```

## Tips and Gotchas

- **Don't put secrets in ansible.cfg** — Use `vault_password_file` pointing to a file outside the repo, or use environment variables in CI/CD.
- **gitignore sensitive files** — Add `.vault_pass`, `.facts_cache/`, and `*.retry` to `.gitignore`.
- **Pipelining + sudo** — If pipelining breaks privilege escalation, check for `requiretty` in `/etc/sudoers` on the target.
- **hash_behaviour = merge is deprecated** — Avoid it. Use `combine` filter in playbooks instead.
- **forks vs serial** — `forks` controls parallelism for all plays. `serial` in a playbook limits how many hosts run a specific play at once. They work together: `forks=30` + `serial=5` means 5 hosts at a time, but up to 30 SSH connections can be reused.
- **ControlPersist timeout** — Set it longer than your longest task. If it expires mid-play, Ansible reconnects (slower but not fatal).
- **ansible-config validate** — Run this after editing to catch typos and deprecated options.
- **Per-project ansible.cfg in git** — This is the recommended pattern. It makes your automation portable and self-contained.
