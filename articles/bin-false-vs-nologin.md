# /bin/false vs /sbin/nologin

Both `/bin/false` and `/sbin/nologin` are used as login shells to prevent users from logging in interactively. They look similar but behave differently — the choice affects user experience, logging, and security auditing.

## The Difference

| Aspect | `/bin/false` | `/sbin/nologin` |
|--------|-------------|----------------|
| **Message** | None (silent) | "This account is currently not available." |
| **Exit code** | 1 (failure) | 1 (failure) |
| **Custom message** | No | Yes — via `/etc/nologin.txt` |
| **Logging** | Minimal | Better audit trail |
| **Purpose** | System accounts (never accessed by humans) | Service accounts, disabled users |
| **FTP access** | Denied (not a valid shell) | May be allowed depending on FTP config |

## /bin/false

Simply exits immediately with status code 1 (failure). No output, no message, no explanation.

```bash
# What /bin/false does internally:
exit 1
```

When someone tries to SSH:

```bash
$ ssh nobody@server
# Connection closes immediately — no message, no explanation
```

**Use for:** System accounts that should be completely silent and never accessed by humans.

```bash
# Typical accounts using /bin/false
nobody:x:65534:65534:nobody:/nonexistent:/bin/false
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
```

## /sbin/nologin

Displays an informative message before exiting with status 1. Tells the user their account is disabled.

```bash
# What /sbin/nologin does:
# 1. Prints "This account is currently not available." to stderr
# 2. Exits with status 1
```

When someone tries to SSH:

```bash
$ ssh apache@server
This account is currently not available.
Connection to server closed.
```

**Use for:** Service accounts (nginx, mysql, postgres) and temporarily disabled user accounts.

```bash
# Typical accounts using /sbin/nologin
apache:x:48:48:Apache:/usr/share/httpd:/sbin/nologin
nginx:x:995:991:Nginx web server:/var/lib/nginx:/sbin/nologin
mysql:x:27:27:MySQL Server:/var/lib/mysql:/sbin/nologin
postgres:x:26:26:PostgreSQL Server:/var/lib/pgsql:/sbin/nologin
```

## Custom nologin Message

You can customize the message displayed by `/sbin/nologin`:

```bash
# Create a custom message
echo "This account is disabled for maintenance. Contact admin@company.com" > /etc/nologin.txt

# Now any user with /sbin/nologin as their shell sees this message instead
$ ssh serviceuser@server
This account is disabled for maintenance. Contact admin@company.com
Connection to server closed.
```

> **Note:** `/etc/nologin.txt` affects ALL users with `/sbin/nologin` as their shell. There's no per-user customization.

## Testing the Difference

```bash
# Test /bin/false — silent, returns 1
/bin/false
echo $?    # Output: 1 (no message printed)

# Test /sbin/nologin — prints message, returns 1
/sbin/nologin
# Output: This account is currently not available.
echo $?    # Output: 1
```

## Setting the Shell

```bash
# Set shell for existing user
sudo usermod -s /sbin/nologin username
sudo usermod -s /bin/false username

# Create user with nologin shell
sudo useradd -s /sbin/nologin -r -M serviceuser

# Create user with false shell
sudo useradd -s /bin/false -r -M systemuser

# Verify
grep username /etc/passwd
```

## /etc/nologin (Different from /sbin/nologin!)

Don't confuse `/sbin/nologin` (a shell) with `/etc/nologin` (a file):

```bash
# /etc/nologin — if this FILE exists, ALL non-root users are blocked from login
# Used during maintenance windows

# Block all logins
sudo touch /etc/nologin
echo "System maintenance in progress. Try again later." | sudo tee /etc/nologin

# Allow logins again
sudo rm /etc/nologin
```

| Path | Type | Effect |
|------|------|--------|
| `/sbin/nologin` | Binary (shell) | Blocks login for users with this shell |
| `/etc/nologin` | File | If exists, blocks ALL non-root logins |
| `/etc/nologin.txt` | File | Custom message for `/sbin/nologin` |

## Path Variations

The binary location varies by distro and version:

| Location | Distro |
|----------|--------|
| `/sbin/nologin` | RHEL 6, RHEL 7, CentOS 6/7 |
| `/usr/sbin/nologin` | RHEL 8+, RHEL 9, RHEL 10, Ubuntu 20.04+, Debian 10+ |
| `/bin/false` | All Linux distros (all versions) |
| `/usr/bin/false` | Same (symlinked on newer systems with merged /usr) |

> **RHEL 6 note:** On RHEL 6, the path is `/sbin/nologin`. After the `/usr` merge in RHEL 7+, `/sbin` is a symlink to `/usr/sbin`, so both paths work. On RHEL 8+ and Ubuntu, always use `/usr/sbin/nologin` in new configurations.

```bash
# Find the actual path on your system
which nologin
which false

# Check if it's in /etc/shells (shouldn't be for security)
cat /etc/shells
grep -E "(false|nologin)" /etc/shells
```

> **Security:** Neither `/bin/false` nor `/sbin/nologin` should be listed in `/etc/shells`. Some FTP servers only allow login if the user's shell is listed in `/etc/shells`.

## View Accounts Using These Shells

```bash
# Show all accounts with nologin or false
grep -E "(false|nologin)" /etc/passwd

# Count them
grep -cE "(false|nologin)" /etc/passwd

# Show only nologin accounts
awk -F: '$7 ~ /nologin/ {print $1}' /etc/passwd

# Show only /bin/false accounts
awk -F: '$7 ~ /false/ {print $1}' /etc/passwd
```

## Ansible Examples

### Create Service Accounts

```yaml
- name: Create service accounts with nologin shell
  user:
    name: "{{ item }}"
    shell: /sbin/nologin
    system: yes
    create_home: no
  loop:
    - nginx
    - mysql
    - redis
    - apache
    - postgres
```

### Create System Accounts

```yaml
- name: Create system accounts with false shell
  user:
    name: "{{ item }}"
    shell: /bin/false
    system: yes
    create_home: no
  loop:
    - nobody
    - nfsnobody
```

### Disable a User Account

```yaml
- name: Disable user login (keep account for file ownership)
  user:
    name: former_employee
    shell: /sbin/nologin
```

## Security Auditing

```bash
# Monitor login attempts to disabled accounts
journalctl -u sshd | grep -i "nologin\|not allowed"

# Check auth log (RHEL)
grep -i "nologin" /var/log/secure

# Check auth log (Ubuntu)
grep -i "nologin" /var/log/auth.log

# Find accounts that probably shouldn't have a real shell
awk -F: '$7 !~ /(nologin|false|sync|shutdown|halt)/ && $3 >= 1000 {print $1, $7}' /etc/passwd
```

## SELinux Context

Both shells work with SELinux without special configuration:

```bash
# Check SELinux context of both binaries
ls -Z /bin/false /sbin/nologin

# Both should be in the shell_exec_t context
# No special booleans needed for standard usage
```

## When to Use Which

### Use `/sbin/nologin` when:
- Creating service accounts (web servers, databases, message queues)
- Temporarily disabling user accounts
- You want users to understand why they can't log in
- You need better security auditing (the message generates log entries)
- Compliance requires informative denial messages

### Use `/bin/false` when:
- Creating pure system accounts (nobody, daemon, bin, sys)
- Building minimal/embedded systems where no messages are wanted
- The account will never be accessed by a human
- You want absolute minimal overhead
- Scripting where you check exit codes and don't want stderr noise

### General Rule

> **If a human might ever try to log in as that account → `/sbin/nologin`**
> **If only the system cares about the account → `/bin/false`**

In modern practice, most sysadmins default to `/sbin/nologin` for everything unless there's a specific reason to use `/bin/false`.
