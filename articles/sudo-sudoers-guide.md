# sudoers Guide

`sudo` controls who can run what commands as which user. All of this is configured in `/etc/sudoers` and drop-in files under `/etc/sudoers.d/`. This guide covers the full range: basic syntax, users, groups, LDAP, command restrictions, aliases, and practical tips.

## Adding a User to sudo via Group Membership

The quickest way to grant sudo access without touching sudoers is to add the user to the `wheel` group (RHEL/CentOS/Rocky) or `sudo` group (Debian/Ubuntu):

```bash
# RHEL / CentOS / Rocky
usermod -aG wheel username

# Debian / Ubuntu
usermod -aG sudo username
```

The corresponding sudoers rule (already present by default on most distros) does the rest:

```
%wheel  ALL=(ALL)  ALL
# or without a password prompt:
%wheel  ALL=(ALL)  NOPASSWD: ALL
```

The group change takes effect at the user's next login.

## Editing sudoers Safely

Always use `visudo` — never edit `/etc/sudoers` directly with a text editor.

```bash
visudo
```

`visudo` locks the file and validates the syntax before saving. A syntax error in sudoers can lock every user out of `sudo` entirely.

To edit a drop-in file:

```bash
visudo -f /etc/sudoers.d/myconfig
```

Drop-in files in `/etc/sudoers.d/` are included automatically. Each file must not have a `.` or `~` in the name, and must have permissions `0440`:

```bash
chmod 0440 /etc/sudoers.d/myconfig
```

## sudoers Syntax

The basic rule format is:

```
WHO  WHERE=(AS_WHOM)  WHAT
```

| Field | Meaning |
|-------|---------|
| `WHO` | User or group who is granted the rule |
| `WHERE` | Hostname(s) the rule applies to (`ALL` = everywhere) |
| `AS_WHOM` | User (and optionally group) to run as — `(root)`, `(ALL)`, `(www-data:www-data)` |
| `WHAT` | Commands allowed, with full paths |

## Granting Access to a User

Allow a single user to run any command as root:

```
# Allow user to run all commands
username ALL=(ALL:ALL) ALL
alice    ALL=(ALL)     ALL
```

Allow only specific commands:

```
# Allow user to run specific commands
username ALL=(root) /bin/ls, /bin/cat, /usr/bin/vim
alice    ALL=(root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

Allow without a password prompt:

```
# Allow user to run all commands without password
username ALL=(ALL:ALL) NOPASSWD: ALL
alice    ALL=(root)    NOPASSWD: /usr/bin/systemctl restart nginx
```

Mix password and no-password rules:

```
alice  ALL=(root)  /usr/bin/apt-get update, NOPASSWD: /usr/bin/systemctl status *
```

Run commands as a specific non-root user:

```
# Allow user to run commands as specific user
username ALL=(webuser) /usr/bin/systemctl restart apache2
```

## Granting Access to a Group

Prefix the group name with `%`:

```
# Allow all users in admin group full access
%admin     ALL=(ALL:ALL)   ALL
%wheel     ALL=(ALL)       ALL
%sudo      ALL=(ALL:ALL)   ALL
%devops    ALL=(root)      /usr/bin/systemctl, /usr/bin/journalctl
%dbadmins  ALL=(postgres)  /usr/bin/psql, /usr/bin/pg_dump

# Allow wheel group full access without password
%wheel  ALL=(ALL:ALL)  NOPASSWD: ALL

# Allow sudo group to run specific commands
%sudo   ALL=(root)  /sbin/service, /sbin/systemctl
```

`(ALL:ALL)` means the group can run commands as any user and any group.

Allow a group without a password:

```
%operators  ALL=(root)  NOPASSWD: ALL
```

## Host-Specific Rules

By default, rules use `ALL` for the hostname field, meaning they apply on any machine. You can restrict a rule to specific hosts instead — useful when the same sudoers file is distributed across many servers via configuration management.

```
# Allow on a single named host only
username  webserver=(ALL:ALL)  ALL

# Allow on multiple hosts (comma-separated)
username  web1,web2,db1=(ALL:ALL)  ALL

# Allow only specific commands on a specific host
alice  db1=(postgres)  /usr/bin/psql, /usr/bin/pg_dump

# Use a Host_Alias for maintainability
Host_Alias  WEBSERVERS = web1, web2, web3
Host_Alias  DBSERVERS  = db1, db2

username  WEBSERVERS=(ALL:ALL)  ALL
username  DBSERVERS=(postgres)  /usr/bin/psql
```

> **Note:** sudo uses the hostname returned by `hostname -s` for matching. If the machine's hostname doesn't match the rule, the rule is silently skipped and access is denied. Always verify with `sudo -l` after deploying host-restricted rules.

## Running as a Specific User or Group

The `AS_WHOM` field can specify both user and group:

```
# Run as www-data user
alice  ALL=(www-data)  /usr/local/bin/deploy.sh

# Run as www-data user in www-data group
alice  ALL=(www-data:www-data)  /usr/local/bin/deploy.sh

# Run as any user
alice  ALL=(ALL)  /usr/bin/id

# Run as any user in any group
alice  ALL=(ALL:ALL)  /usr/bin/id
```

The caller then specifies the target at runtime:

```bash
sudo -u www-data /usr/local/bin/deploy.sh
sudo -u postgres -g dba /usr/bin/psql
```

## Aliases

Aliases let you define reusable sets of users, hosts, commands, or run-as targets. They make large sudoers files maintainable.

### User Alias

```
User_Alias  ADMINS    = alice, bob, carol
User_Alias  DBTEAM    = dave, eve
User_Alias  WEBTEAM   = frank, grace, henry
User_Alias  WEBADMINS = alice, charlie
```

Applied:

```
# Define user aliases
User_Alias ADMINS    = john, jane, bob
User_Alias WEBADMINS = alice, charlie

# Use in rules
ADMINS    ALL=(ALL:ALL) ALL
WEBADMINS ALL=(www-data) /usr/sbin/apache2ctl
```

### Host Alias

```
Host_Alias  WEBSERVERS = web1, web2, web3
Host_Alias  DBSERVERS  = db1, db2
Host_Alias  ALL_PROD   = WEBSERVERS, DBSERVERS
```

### Runas Alias

```
Runas_Alias  DB       = postgres, mysql
Runas_Alias  WEBUSER  = www-data, nginx
```

### Command Alias

```
Cmnd_Alias  SERVICES    = /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *
Cmnd_Alias  NETWORKING  = /usr/sbin/ip, /usr/sbin/ifconfig, /usr/sbin/netstat, /sbin/route, /bin/ping, /sbin/dhclient
Cmnd_Alias  PKGMGMT     = /usr/bin/apt-get, /usr/bin/apt, /usr/bin/dpkg, /bin/rpm, /usr/bin/yum, /usr/bin/dnf
Cmnd_Alias  SYSV        = /sbin/service, /sbin/chkconfig, /usr/bin/systemctl
Cmnd_Alias  REBOOT      = /usr/sbin/reboot, /usr/sbin/shutdown
```

Applied:

```
# Define command aliases
Cmnd_Alias NETWORKING = /sbin/route, /sbin/ifconfig, /bin/ping, /sbin/dhclient
Cmnd_Alias SOFTWARE   = /bin/rpm, /usr/bin/up2date, /usr/bin/yum
Cmnd_Alias SERVICES   = /sbin/service, /sbin/chkconfig, /usr/bin/systemctl

# Use aliases in rules
username ALL = NETWORKING, SOFTWARE, SERVICES
```

### Using Aliases Together

```
ADMINS    ALL=(ALL:ALL)   ALL
WEBADMINS ALL=(www-data)  /usr/sbin/apache2ctl
DBTEAM    DBSERVERS=(DB)  /usr/bin/psql, /usr/bin/pg_dump
WEBTEAM   WEBSERVERS=(WEBUSER)  SERVICES, NOPASSWD: /usr/bin/tail -f /var/log/nginx/*
```

## Wildcards and Command Arguments

Use `*` to match any argument:

```
alice  ALL=(root)  /usr/bin/systemctl * nginx
alice  ALL=(root)  /usr/bin/tail -f /var/log/*
```

Restrict to no arguments at all (the command must be called with nothing after it):

```
alice  ALL=(root)  /usr/sbin/reboot ""
```

> **Warning:** Wildcards in command paths or arguments can be abused. `/usr/bin/vim *` lets the user open any file as root — and from inside vim, launch a shell. Be as specific as possible with arguments.

## Denying Commands

Prefix with `!` to explicitly deny:

```
%developers  ALL=(root)  ALL, !/usr/sbin/reboot, !/usr/sbin/shutdown, !/usr/bin/su
```

> **Note:** Deny rules can be bypassed by calling the same binary through a different path, a shell, or a script. They are a hint, not a security guarantee. Do not rely on `!` as your only control.

## NOPASSWD and PASSWD

`NOPASSWD` skips the password prompt for the commands that follow it. `PASSWD` re-enables it (useful to override an earlier `NOPASSWD`):

```
# No password for status checks, password required for restarts
alice  ALL=(root)  NOPASSWD: /usr/bin/systemctl status *, PASSWD: /usr/bin/systemctl restart *
```

The tags apply to all commands to their right until overridden.

## Defaults

`Defaults` lines configure sudo's global or per-user behaviour.

### Common Global Defaults

```
# Require full tty (prevents sudo from cron/scripts without a terminal)
Defaults  requiretty

# Log all sudo commands to syslog
Defaults  log_host, log_year, logfile="/var/log/sudo.log"

# Set a secure PATH to prevent hijacking via $PATH manipulation
Defaults  secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# Password timestamp timeout in minutes (0 = always ask, -1 = never expire)
Defaults  timestamp_timeout=5

# Number of password attempts before failure
Defaults  passwd_tries=3

# Insult users who type wrong passwords (just for fun)
Defaults  insults

# Show asterisks while typing the password
Defaults  pwfeedback

# Keep HOME environment variable (be careful — can affect behaviour)
Defaults  always_set_home

# Set the editor used by visudo
Defaults  editor=/usr/bin/vim

# Send email on a bad password attempt
Defaults  mail_badpass
Defaults  mailto="ops@example.com"

# Send email on every sudo invocation (noisy — use with care)
Defaults  mail_always
```

### Per-User and Per-Group Defaults

```
# No password timeout for alice (always re-ask)
Defaults:alice  timestamp_timeout=0

# Give the ops group a longer timeout
Defaults:%ops  timestamp_timeout=60

# Allow bob to preserve his environment
Defaults:bob  !env_reset

# No requiretty for automation user
Defaults:ansible  !requiretty
```

### Preserving Environment Variables

```
# Keep specific variables for all users
Defaults  env_keep += "HTTP_PROXY HTTPS_PROXY NO_PROXY"
Defaults  env_keep += "EDITOR VISUAL PAGER"

# Keep variables for a specific user
Defaults:alice  env_keep += "MY_APP_ENV"
```

## LDAP Integration

When sudo is compiled with LDAP support (`sudo-ldap` package on Debian/Ubuntu), rules can be stored in the directory instead of flat files. This centralises access control across many hosts.

### Package

```bash
# Debian / Ubuntu
apt-get install sudo-ldap

# RHEL / CentOS / Rocky
yum install sudo  # LDAP support is included in the standard package
```

### /etc/sudo-ldap.conf (or /etc/ldap.conf)

```
uri      ldap://ldap.example.com
sudoers_base  ou=SUDOers,dc=example,dc=com
binddn   cn=sudo,ou=system,dc=example,dc=com
bindpw   secret
ssl      start_tls
tls_cacertfile /etc/ssl/certs/ca-certificates.crt
```

### /etc/nsswitch.conf

Tell sudo where to look for sudoers rules (LDAP first, then local file as fallback):

```
sudoers: ldap files
```

### LDAP Schema

Each `sudoRole` object holds a complete rule:

```ldif
dn: cn=webops,ou=SUDOers,dc=example,dc=com
objectClass: sudoRole
cn: webops
sudoUser: %webops
sudoHost: ALL
sudoCommand: /usr/bin/systemctl restart nginx
sudoCommand: /usr/bin/systemctl reload nginx
sudoOption: !authenticate
```

| LDAP Attribute | sudoers Equivalent |
|----------------|-------------------|
| `sudoUser` | `WHO` field — usernames or `%group` |
| `sudoHost` | `WHERE` field |
| `sudoRunAsUser` | Run-as user (`(user)`) |
| `sudoRunAsGroup` | Run-as group |
| `sudoCommand` | `WHAT` field |
| `sudoOption` | `Defaults` options, e.g. `!authenticate` = NOPASSWD |
| `sudoNotBefore` | Rule valid from this time (ISO 8601) |
| `sudoNotAfter` | Rule expires at this time |
| `sudoOrder` | Integer — lower numbers are evaluated first |

### Testing LDAP Rules

```bash
# Check effective rules for a user
sudo -l -U alice

# Verify which sudoers source is being used
sudo -l -U alice 2>&1 | head -5
```

## Viewing Effective Rules

```bash
# What can the current user run?
sudo -l

# What can a specific user run? (requires root or sudo)
sudo -l -U alice

# Parse and display sudoers in a structured way
sudo -ll

# Test if a specific command is allowed without running it
sudo -l | grep systemctl
```

## Sudoers Order of Evaluation

Rules are evaluated top to bottom. The **last matching rule wins**. This is the opposite of most firewall policies.

```
# This grants alice ALL first...
alice  ALL=(ALL)  ALL

# ...but this DENY is below it, so it takes effect
alice  ALL=(ALL)  !/usr/sbin/reboot
```

Switch the order and the deny rule would be overridden by the `ALL` below it. Place more specific rules below more general ones.

## drop-in Files in /etc/sudoers.d/

Rather than editing the main sudoers file, put separate configs in `/etc/sudoers.d/`. This is easier to manage with configuration management tools like Ansible, Puppet, or Chef.

```bash
# Create a rule for the deploy user
visudo -f /etc/sudoers.d/deploy

# Typical content
deploy  ALL=(root)  NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/systemctl reload myapp
```

Files are included in **alphabetical order**. Name them with a numeric prefix to control load order:

```
/etc/sudoers.d/10-base
/etc/sudoers.d/20-teams
/etc/sudoers.d/50-deploy
/etc/sudoers.d/90-overrides
```

For a quick one-off grant you can use `echo` instead of `visudo`, but only if you are confident in the syntax — there is no validation:

```bash
# Grant a single user full NOPASSWD sudo
echo -e "username\tALL=(ALL)\tNOPASSWD: ALL" > /etc/sudoers.d/020_sudo_for_me
chmod 0440 /etc/sudoers.d/020_sudo_for_me
```

> **Warning:** Always set `0440` permissions on drop-in files. sudo silently ignores files with world-writable or world-readable permissions as a security measure.

Files with names ending in `~` or containing `.` are silently ignored. Use underscores or hyphens instead.

### Include Directives

The main sudoers file pulls in drop-in files via an include directive. There are two equivalent forms:

```
# Traditional form (comment-style, older)
#includedir /etc/sudoers.d

# Modern form (supported since sudo 1.9.1)
@includedir /etc/sudoers.d

# Include a single specific file
@include /etc/sudoers.local
```

Most distributions ship with `#includedir /etc/sudoers.d` already in the default sudoers file.

## Sudoers Automation

Methods to safely append rules to `/etc/sudoers.d/` files from scripts and automation tools.

### Automated approach with validation

```bash
# Create temporary file with content
cat << 'EOF' > /tmp/sudoers_temp
username ALL=(ALL) NOPASSWD: /usr/bin/command1
username ALL=(ALL) NOPASSWD: /usr/bin/command2
EOF

# Validate syntax
sudo visudo -c -f /tmp/sudoers_temp

# If validation passes, append to target file
if [ $? -eq 0 ]; then
    sudo tee -a /etc/sudoers.d/file < /tmp/sudoers_temp
    sudo chmod 0440 /etc/sudoers.d/file
    echo "Successfully added to sudoers"
else
    echo "Syntax error - not applied"
fi

# Cleanup
rm /tmp/sudoers_temp
```

### One-liner with inline validation

```bash
echo "username ALL=(ALL) NOPASSWD: /usr/bin/command" | sudo EDITOR='tee -a' visudo -f /etc/sudoers.d/file
```

### Script-friendly function

```bash
add_sudoers_rule() {
    local file="$1"
    local rule="$2"
    
    # Create temp file
    local tmp=$(mktemp)
    echo "$rule" > "$tmp"
    
    # Validate
    if sudo visudo -c -f "$tmp" &>/dev/null; then
        sudo tee -a "/etc/sudoers.d/$file" < "$tmp" > /dev/null
        sudo chmod 0440 "/etc/sudoers.d/$file"
        echo "Rule added successfully"
    else
        echo "Invalid sudoers syntax" >&2
        rm "$tmp"
        return 1
    fi
    
    rm "$tmp"
}

# Usage:
add_sudoers_rule "myfile" "username ALL=(ALL) NOPASSWD: /usr/bin/command"
```

### Idempotent approach (Ansible/automation style)

Only adds the rule if it's not already present:

```bash
RULE="username ALL=(ALL) NOPASSWD: /usr/bin/command"
FILE="/etc/sudoers.d/myfile"

if ! sudo grep -qF "$RULE" "$FILE" 2>/dev/null; then
    echo "$RULE" | sudo tee -a "$FILE" > /dev/null
    sudo chmod 0440 "$FILE"
    sudo visudo -c -f "$FILE" || sudo rm "$FILE"
fi
```

### Direct append methods

#### Single line append

```bash
echo "username ALL=(ALL) NOPASSWD: /usr/bin/command" | sudo tee -a /etc/sudoers.d/file
```

#### Multiple lines append with heredoc

```bash
sudo tee -a /etc/sudoers.d/file << 'EOF'
username ALL=(ALL) NOPASSWD: /usr/bin/command1
username ALL=(ALL) NOPASSWD: /usr/bin/command2
EOF
```

#### Using cat

```bash
cat << 'EOF' | sudo tee -a /etc/sudoers.d/file
username ALL=(ALL) NOPASSWD: /usr/bin/command
EOF
```

### Validation and permissions

```bash
# Validate syntax
sudo visudo -c -f /etc/sudoers.d/file

# Set correct permissions (required — sudo ignores files without 0440)
sudo chmod 0440 /etc/sudoers.d/file
```

> **Important:** Files in `/etc/sudoers.d/` must have permissions `0440`. Files must not contain `.` or end with `~` (they'll be silently ignored). Always validate syntax after editing.

## Userdata Scripts (Cloud-Init)

For cloud-init (AWS EC2, Azure, GCP, etc.) userdata scripts run as root by default — no `sudo` needed.

### Simple single rule

```bash
#!/bin/bash

echo -e "\nusername ALL=(ALL) NOPASSWD: /usr/bin/command" | tee -a /etc/sudoers.d/myfile
chmod 0440 /etc/sudoers.d/myfile
```

### Multiple rules using heredoc

```bash
#!/bin/bash

cat > /etc/sudoers.d/myapp << 'EOF'
# Application service account
appuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp
appuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl status myapp
EOF

chmod 0440 /etc/sudoers.d/myapp
visudo -c -f /etc/sudoers.d/myapp
```

### With validation before applying

```bash
#!/bin/bash

cat > /tmp/sudoers_new << 'EOF'

# Service account rules
ubuntu ALL=(ALL) NOPASSWD: /usr/bin/docker
ubuntu ALL=(ALL) NOPASSWD: /usr/bin/systemctl
EOF

if visudo -c -f /tmp/sudoers_new; then
    cat /tmp/sudoers_new >> /etc/sudoers.d/custom
    chmod 0440 /etc/sudoers.d/custom
    echo "Sudoers rules applied successfully"
else
    echo "Validation failed - rules not applied"
fi

rm /tmp/sudoers_new
```

### Complete userdata with error handling

```bash
#!/bin/bash
set -e

add_sudoers_rule() {
    local filename="$1"
    local content="$2"
    local filepath="/etc/sudoers.d/${filename}"
    
    local tmp=$(mktemp)
    echo "$content" > "$tmp"
    
    if visudo -c -f "$tmp" 2>/dev/null; then
        cat "$tmp" > "$filepath"
        chmod 0440 "$filepath"
        echo "Successfully added sudoers file: $filename"
    else
        echo "ERROR: Invalid sudoers syntax in $filename" >&2
        rm "$tmp"
        return 1
    fi
    
    rm "$tmp"
}

add_sudoers_rule "appuser" "
# Application user permissions
appuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp
appuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl status myapp
"

add_sudoers_rule "ubuntu" "
# Ubuntu user docker access
ubuntu ALL=(ALL) NOPASSWD: /usr/bin/docker
"
```

### AWS EC2 userdata with ubuntu user

```bash
#!/bin/bash

cloud-init status --wait

cat > /etc/sudoers.d/docker << 'EOF'

# Docker access for ubuntu user
ubuntu ALL=(ALL) NOPASSWD: /usr/bin/docker
ubuntu ALL=(ALL) NOPASSWD: /usr/bin/docker-compose
EOF

chmod 0440 /etc/sudoers.d/docker

if ! visudo -c -f /etc/sudoers.d/docker; then
    echo "ERROR: Invalid sudoers configuration" >&2
    rm /etc/sudoers.d/docker
    exit 1
fi
```

### Using `printf` (more portable)

```bash
#!/bin/bash

printf "\nec2-user ALL=(ALL) NOPASSWD: /usr/bin/systemctl\n" | tee -a /etc/sudoers.d/ec2-user
chmod 0440 /etc/sudoers.d/ec2-user
```

### Multiple users and commands

```bash
#!/bin/bash

cat > /etc/sudoers.d/custom << 'EOF'

# Development team permissions
devuser1 ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart app
devuser2 ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart app

# Operations team permissions
opsuser1 ALL=(ALL) NOPASSWD: /usr/bin/systemctl *
opsuser1 ALL=(ALL) NOPASSWD: /usr/bin/journalctl

# Service accounts
serviceaccount ALL=(ALL) NOPASSWD: ALL
EOF

chmod 0440 /etc/sudoers.d/custom
visudo -c -f /etc/sudoers.d/custom
```

### Notes for userdata scripts

- Userdata runs as root — no `sudo` needed
- Always set permissions to `0440`
- Files must not contain `.` or end with `~`
- Always validate with `visudo -c -f /etc/sudoers.d/filename`
- Use `set -e` to exit on errors
- Consider logging: `exec > >(tee /var/log/userdata.log) 2>&1`
- For AWS EC2, logs are in `/var/log/cloud-init-output.log`

## Logging and Auditing

### Syslog (default)

By default, sudo logs to syslog under the `auth` facility:

```bash
grep sudo /var/log/auth.log
grep sudo /var/log/secure        # RHEL/CentOS
journalctl -u sudo               # systemd systems
```

### Dedicated Log File

```
Defaults  logfile="/var/log/sudo.log"
Defaults  log_input, log_output
Defaults  iolog_dir=/var/log/sudo-io/%{user}
```

`log_input` and `log_output` record full terminal I/O for every sudo session — useful for compliance auditing but generates significant disk usage.

### Viewing I/O Logs

```bash
sudoreplay /var/log/sudo-io/alice/00/00/01
```

## Tips and Tricks

### Running a Shell Function as sudo

`sudo` runs a new process and does not inherit shell functions defined in your current session. There are two ways around this.

**Option 1 — Inline the function with `declare -f`:**

```bash
# Define the function normally
dbash() { docker exec -it $(docker ps -aqf "name=$1") bash; }

# Export and run it under sudo in a single call
sudo bash -c "$(declare -f dbash); dbash apache_web"
```

`declare -f dbash` prints the function definition as a string. Wrapping it in `$(...)` and passing it to `bash -c` makes the elevated subshell define and then call the function.

**Option 2 — Wrapper script that sources your profile:**

```bash
#!/bin/bash
# /home/username/privileged-wrapper.sh
source /home/username/.bashrc
"$@"
```

```bash
chmod +x /home/username/privileged-wrapper.sh
sudo /home/username/privileged-wrapper.sh dbash apache_web
```

The wrapper sources `.bashrc` (which defines your functions) before executing whatever arguments are passed to it.

> **Note:** Option 2 sources user-controlled files as root. Only use it if you trust the contents of that `.bashrc` — a malicious or accidentally modified `.bashrc` would execute as root.

### Use `sudo -v` to Refresh the Timestamp

Extends the sudo credential cache without running a command. Useful in scripts that need sudo later but want to authenticate upfront:

```bash
sudo -v    # refresh (or establish) the cached credential
# ... long operation ...
sudo systemctl restart myapp
```

### Use `sudo -k` to Invalidate the Timestamp

Forces the next `sudo` call to re-authenticate immediately:

```bash
sudo -k
```

### `sudo -n` for Non-Interactive Scripts

`-n` (non-interactive) prevents sudo from prompting for a password. If a password would normally be required, the command fails immediately with an error instead of hanging:

```bash
sudo -n systemctl reload nginx
```

Useful in scripts where you want to fail fast rather than block waiting for input. Combine with `NOPASSWD` in sudoers for the commands the script needs.

### `sudo -b` to Background a Command

Runs the command in the background and returns the prompt immediately:

```bash
sudo -b /opt/scripts/long-running-job.sh
```

### `sudo !!` — Re-run Last Command with sudo

A shell shortcut that expands `!!` to the previous command:

```bash
systemctl restart nginx
# Permission denied
sudo !!
# Expands to: sudo systemctl restart nginx
```

> **Note:** This is a bash/zsh history expansion, not a sudo feature. It won't work in shells where history expansion is disabled.

### Wrap Aliases in sudoers

Create shell aliases for common privileged operations so users don't need to remember exact paths:

```bash
# In ~/.bashrc
alias sservice='sudo systemctl'
alias supdate='sudo -- sh -c "apt update && apt upgrade"'
```

### Test Rules Without Running Commands

```bash
# -l lists what's allowed; -U targets another user (needs root)
sudo -l -U deploy
```

### Check Whether a Specific Command Is Permitted

```bash
sudo -l | grep "systemctl"
```

### Verify sudoers Syntax Without Applying

```bash
visudo -c               # check main file
visudo -c -f /etc/sudoers.d/myconfig   # check a drop-in
```

### Avoid Editing sudoers on a Production Host Directly

Use configuration management (Ansible, Puppet, Chef) to distribute sudoers rules. Always test on a non-production host first and keep a root session open while applying changes — just in case.

### Keep a Root Session Open When Making Changes

Before editing sudoers, open a separate terminal with an active root shell (`sudo -i`). If you introduce a syntax error and get locked out of sudo, you still have a way back.

### Restrict sudo to a TTY

`requiretty` prevents sudo from being called from cron jobs, scripts, or SSH non-interactive sessions:

```
Defaults  requiretty
```

If you need sudo in automation, disable it selectively:

```
Defaults:ansible  !requiretty
Defaults:deploy   !requiretty
```

### Time-Limited Access (sudoNotAfter via LDAP)

With LDAP-based sudo, you can grant temporary access that expires automatically:

```ldif
sudoNotBefore: 20240801T000000Z
sudoNotAfter:  20240802T235959Z
```

Without LDAP, the closest alternative is to add and then remove a drop-in file manually, or use a tool like `sudo_pair` or PAM time restrictions.

### Prevent Privilege Escalation via Editors and Shells

Any command that can open a shell or write arbitrary files is effectively `ALL`. Common escape hatches to avoid in sudoers:

```
# These are dangerous — all can spawn a root shell
/usr/bin/vim
/usr/bin/nano
/usr/bin/less
/usr/bin/man
/usr/bin/find  # find . -exec /bin/sh \;
/usr/bin/awk
/usr/bin/perl
/usr/bin/python3
/bin/bash
/bin/sh
```

If you need to allow file editing, use `sudoedit` instead:

```
alice  ALL=(root)  sudoedit /etc/nginx/nginx.conf
```

`sudoedit` copies the file to a temp location, opens it as the user with their own editor, and writes it back as root — the editor itself never runs as root.

## Practical Examples

### Service Management

```bash
# Restart / reload services
sudo systemctl restart nginx
sudo systemctl reload apache2
sudo systemctl start postgresql
sudo systemctl stop mysql
sudo systemctl enable docker
sudo systemctl disable firewalld

# Check status
sudo systemctl status sshd
sudo journalctl -u nginx -f
```

### Package Management

```bash
# Debian / Ubuntu
sudo apt update
sudo apt install nginx
sudo apt remove nano
sudo apt autoremove
sudo apt upgrade
sudo dpkg -i package.deb

# RHEL / CentOS / Rocky
sudo yum install nginx
sudo dnf install httpd
sudo rpm -ivh package.rpm
```

### File and Directory Operations

```bash
# Create directories owned by root
sudo mkdir -p /opt/myapp/config

# Change ownership
sudo chown alice:developers /opt/myapp
sudo chown -R www-data:www-data /var/www/html

# Change permissions
sudo chmod 755 /opt/myapp
sudo chmod 640 /etc/app/secrets.conf

# Copy files to system directories
sudo cp app.conf /etc/app/app.conf

# Edit system files safely
sudo -e /etc/hosts
sudo -e /etc/nginx/nginx.conf

# Append to a root-owned file (use tee, not >>)
echo "192.168.1.50 myhost" | sudo tee -a /etc/hosts > /dev/null
```

### Filesystem and Mount

```bash
sudo mount /dev/sdb1 /mnt/backup
sudo umount /mnt/backup
sudo mount -o remount,rw /
sudo fdisk -l
sudo lsblk
```

### Network Configuration

```bash
# Interface management
sudo ip addr add 192.168.1.100/24 dev eth0
sudo ip link set eth0 up
sudo ip route add default via 192.168.1.1
sudo ifconfig eth0 192.168.1.100

# Firewall
sudo iptables -L -n -v
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo ufw enable
sudo ufw allow 22/tcp
sudo firewall-cmd --list-all
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
```

### Process Management

```bash
# Kill processes
sudo kill -9 1234
sudo killall nginx
sudo pkill -f "python script.py"

# Run a command as a service user
sudo -u www-data php /var/www/html/cron.php
sudo -u postgres psql -c "SELECT version();"
sudo -u git /usr/local/bin/gitea
```

### Log Inspection

```bash
sudo tail -f /var/log/syslog
sudo tail -f /var/log/auth.log
sudo tail -f /var/log/nginx/error.log
sudo grep "Failed password" /var/log/secure
sudo journalctl -xe
sudo journalctl --since "1 hour ago" -u sshd
```

## Troubleshooting

### User is not in the sudoers file

Check whether the user belongs to the right group:

```bash
groups username
id username
```

If the user is missing from `wheel` or `sudo`, add them:

```bash
usermod -aG wheel username   # RHEL
usermod -aG sudo username    # Debian/Ubuntu
```

The change takes effect at next login. To apply it in the current session without logging out:

```bash
newgrp wheel
```

### Syntax error locked me out

If sudo stops working after a sudoers edit, use `pkexec` (PolicyKit) to recover:

```bash
pkexec visudo
```

Or boot to single-user mode / recovery shell and fix the file directly.

### sudo: command not found

The `sudo` binary may not be installed or may not be in `PATH`:

```bash
which sudo
apt-get install sudo   # Debian/Ubuntu
yum install sudo       # RHEL/CentOS
```

### Test whether a specific command is allowed

```bash
sudo -l -U username | grep command
```

## Quick Reference

```bash
visudo                         # edit /etc/sudoers safely
visudo -f /etc/sudoers.d/foo   # edit a drop-in file
visudo -c                      # syntax check
sudo -l                        # list own permissions
sudo -l -U alice               # list alice's permissions
sudo -v                        # refresh credential cache
sudo -k                        # invalidate credential cache
sudo -i                        # full root login shell
sudo -s                        # root shell (keeps current env)
sudo su -                      # switch to root login shell via su
sudo -u bob command            # run as a different user
sudo -g devops command         # run as a different group
sudo -e /etc/hosts             # safe file editing (same as sudoedit)
sudo -n command                # non-interactive — fail if password required
sudo -b command                # run command in the background
sudo !!                        # re-run last command with sudo
groups username                # check group memberships
id username                    # show UID, GID, and all groups
```
