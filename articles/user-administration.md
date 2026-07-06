# User Administration on RHEL

<img src="/articles/images/rhel-logo.svg" alt="Red Hat Logo" width="200">

## UID ranges

Specific UID numbers and ranges of numbers are used for specific purposes by Red Hat Enterprise Linux.

* UID 0 is always assigned to the superuser account, root.
* UID 1-200 is a range of "system users" assigned statically to system processes by Red Hat.
* UID 201-999 is a range of "system users" used by system processes that do not own files on the file system. They are typically assigned dynamically from the available pool when the software that needs them is installed. Programs run as these "unprivileged" system users in order to limit their access to just the resources they need to function.
* UID 1000+ is the range available for assignment to regular users.

Prior to Red Hat Enterprise Linux 7, the convention was that UID 1-499 was used for system users and UID 500+ for regular users. Default ranges used by useradd and groupadd can be changed in the `/etc/login.defs` file.

Given the automatic creation of user private groups (GID 1000+), it is generally recommended to set aside a range of GID numbers to be used for supplementary groups. A higher range will avoid a collision with a system group (GID 0-999).

The **-r** option will create a system group using a GID from the range of valid system GID numbers listed in the `/etc/login.defs` file.

## /etc/login.defs

### Password Aging

| Parameter | Description | Typical hardened value |
|-----------|-------------|----------------------|
| PASS_MAX_DAYS | Maximum number of days a password can be used | 90 |
| PASS_MIN_DAYS | Minimum days allowed between password changes | 1 |
| PASS_MIN_LEN | Minimum acceptable password length (legacy, not enforced by PAM) | 8 |
| PASS_WARN_AGE | Days of warning before password expires | 14 |

> **Note:** On RHEL 8+, password length and complexity are enforced through `pam_pwquality` (`/etc/security/pwquality.conf`), not `PASS_MIN_LEN`.

### UID/GID Ranges

```
UID_MIN                  1000       # RHEL 7+ (was 500 in RHEL 6)
UID_MAX                 60000
SYS_UID_MIN               201
SYS_UID_MAX               999
GID_MIN                  1000
GID_MAX                 60000
SYS_GID_MIN               201
SYS_GID_MAX               999
```

### Subordinate IDs (RHEL 8+ — rootless containers)

These enable `/etc/subuid` and `/etc/subgid` for unprivileged (rootless) containers:

```
SUB_UID_MIN            100000
SUB_UID_MAX         600100000
SUB_UID_COUNT           65536
SUB_GID_MIN            100000
SUB_GID_MAX         600100000
SUB_GID_COUNT           65536
```

Each new user is allocated 65536 subordinate UIDs/GIDs for user namespace mapping (used by Podman, Docker rootless).

### UMASK and HOME_MODE

```
UMASK           022       # Used for home directory creation if HOME_MODE is not set
HOME_MODE       0700      # RHEL 9+ explicit home directory permissions (takes precedence over UMASK)
```

| UMASK | Files created as | Directories created as |
|-------|-----------------|----------------------|
| 022 | 644 (rw-r--r--) | 755 (rwxr-xr-x) |
| 027 | 640 (rw-r-----) | 750 (rwxr-x---) |
| 077 | 600 (rw-------) | 700 (rwx------) |

> **RHEL 9+:** `HOME_MODE` is the preferred way to set home directory permissions. When set, it overrides the UMASK calculation for home dirs.

### ENCRYPT_METHOD

| Version | Default | Notes |
|---------|---------|-------|
| RHEL 6–9 | SHA512 | `$6$` prefix in /etc/shadow |
| RHEL 10 | YESCRYPT | `$y$` prefix in /etc/shadow |

```
# RHEL 6–9
ENCRYPT_METHOD SHA512
SHA_CRYPT_MIN_ROUNDS 5000
SHA_CRYPT_MAX_ROUNDS 10000

# RHEL 10
ENCRYPT_METHOD YESCRYPT
YESCRYPT_COST_FACTOR 5
```

> On RHEL 8+, user password hashing is primarily handled through PAM. Keep `ENCRYPT_METHOD` consistent with the PAM configuration since shadow-utils tools (group passwords, batch account creation) still read it.

### Other Useful Settings

```
CREATE_HOME      yes       # Auto-create home directories for new users
USERGROUPS_ENAB  yes       # User Private Group (UPG) scheme — each user gets own group
LOG_OK_LOGINS    no        # Set to yes on production for audit trail
LOG_UNKFAIL_ENAB no        # Log unknown usernames on failed login (privacy concern)
LOGIN_RETRIES    5         # Max login attempts before failure
LOGIN_TIMEOUT    60        # Seconds before login times out
LASTLOG_UID_MAX  100000    # RHEL 9+ — skip lastlog for high UIDs (remote identity users)
```

### Viewing Active Configuration

```sh
# Show all active (non-comment) settings
awk 'NF && $1 !~ /^#/' /etc/login.defs

# Check a specific setting
grep -E '^ENCRYPT_METHOD' /etc/login.defs
```

---

## Password Hash Format

There are three pieces of information stored in a modern password hash.

**SHA-512 format** (`$6$`):

`$6$gCjLa2/Z$6Pu0EK0AzfCjxjv2hoLOB/...`

1. `$6$` — The hashing algorithm (6 = SHA-512)
2. `gCjLa2/Z` — The salt (randomly generated)
3. `6Pu0EK0AzfCjxjv2hoLOB/...` — The encrypted hash

**yescrypt format** (`$y$`) — RHEL 10 default:

`$y$j9T$salt$hash`

1. `$y$` — The algorithm identifier (yescrypt)
2. `j9T` — Cost parameters (CPU/memory factors encoded)
3. `salt` — The salt
4. `hash` — The encrypted hash

**All algorithm identifiers:**

| Prefix | Algorithm | Supported in |
|--------|-----------|-------------|
| `$1$` | MD5 | All (do not use) |
| `$5$` | SHA-256 | RHEL 6+ |
| `$6$` | SHA-512 | RHEL 6+ (default until RHEL 9) |
| `$y$` | yescrypt | RHEL 9+ (default in RHEL 10) |

When a user logs in, the system looks up their entry in `/etc/shadow`, combines the stored salt with the typed password, hashes using the algorithm specified by the prefix, and compares the result. This allows authentication without storing the password in a reversible form.

**Changing the algorithm:**

```sh
# RHEL 6/7 (deprecated in RHEL 8+)
authconfig --passalgo sha512 --update

# RHEL 8+ — edit /etc/login.defs and PAM config
# /etc/login.defs:
ENCRYPT_METHOD YESCRYPT
```

### /etc/shadow Format

Nine colon-separated fields:

`name:password:lastchange:minage:maxage:warning:inactive:expire:blank`

| Field | Description |
|-------|-------------|
| name | Login name |
| password | Encrypted password (or `!`/`!!` if locked, `*` if no login) |
| lastchange | Days since Jan 1 1970 password was last changed |
| minage | Minimum days between password changes |
| maxage | Maximum days before password must be changed |
| warning | Days before expiry user is warned |
| inactive | Days after expiry account is disabled |
| expire | Days since Jan 1 1970 account expires |
| blank | Reserved |

## Users

* Add a customized username: `useradd -u <uid> -g <gid/group> -c "Comment" -m -d </home/homedir> -s <shell> username`
* Add the username to users group (GID 100): `useradd -n <username>`
* Display the current default values: `useradd -D`
* Change default shell (or edit /etc/default/useradd): `useradd -D -s /bin/sh`
* Manually edit /etc/passwd: `vipw`
* Display passwd file: `cat /etc/passwd | column -t -s :`
* Create a system account:

```sh
groupadd -r <group>
useradd -r -g <group> -c <comment> -s /sbin/nologin -d / <username>
```

* List all users with a password set: `egrep -v '.*:\*|:\!' /etc/shadow | awk -F: '{print $1, $2}'`
* List all users > UID_MIN (RHEL 5-6): `awk -F: '($3>=500) && ($3!=65534)' /etc/passwd | cut -d ":" -f1`
* List all users > UID_MIN (RHEL 7+): `awk -F: '($3>=1000)' /etc/passwd | cut -d ":" -f1`
* Show UID, GID, and group memberships: `id <username>`
* Query user from NSS (works with LDAP/SSSD): `getent passwd <username>`
* List all user accounts with details (login times, password status): `lslogins`
* Delete a user and their home directory: `userdel -r <username>`
* Disable login for a user: `usermod -s /sbin/nologin <username>`
* Expire an account at a specific date: `usermod -e 2025-12-31 <username>`
* Remove account expiration: `usermod -e "" <username>`
* Find all files owned by a user: `find / -user <username>`
* Find orphaned files (no matching user): `find / -nouser -o -nogroup`
* Show last login for all users: `lastlog`
* Show login history: `last`
* Show failed login attempts: `lastb`
* Show currently logged-in users: `w` or `who`

## Groups

The use of the -a option makes usermod function in "append" mode. Without it, the user would be removed from all other supplementary groups.

* Add the user to the group: `usermod -aG <group> <username>` or `gpasswd -a <username> <group>`
* Remove the user from the group: `gpasswd -d <username> <group>`
* Remove all secondary groups for a user: `usermod -G "" <username>`
* Set a password for a group: `gpasswd <group>`
* Change the current group ID: `newgrp <group>`
* Log out of current group ID: `newgrp`
* Manually edit /etc/group: `vigr`
* Show groups for a user: `groups <username>`
* List members of a group: `lid -g <group>` or `groupmems -l -g <group>`
* Query group from NSS (works with LDAP/SSSD): `getent group <group>`
* Delete a group: `groupdel <group>`
* Verify integrity of group files (/etc/group & /etc/gshadow): `grpck`

## Passwords

* Display the status of the password for a given account: `passwd -S <username>`
* Lock a user's password: `usermod -L <username>` ("!" in front of the password) or `passwd -l <username>` ("!!" in front of the password)
* Unlock a user's password (removes the '!'): `usermod -U <username>` or `passwd -u <username>`
* Expire password (force change on next login): `passwd -e <username>` or `chage -d 0 <username>`
* Set min/max/warn days from CLI: `passwd -n <min> -x <max> -w <warn> <username>`
* Update passwords in batch mode (password in text clear): `echo "username:password" | chpasswd`
* Set password from standard input (password in text clear): `echo "password" | passwd --stdin <username>`
* Set an encrypted password: `usermod -p "<encrypted-password>" <username>`
* Delete a password for an account: `passwd -d <username>`
* Generate a random password: `</dev/urandom tr -dc A-Za-z0-9 | head -c16`
* Generate a SHA-512 password hash: `openssl passwd -6 -salt <salt> <password>`
* Generate a yescrypt password hash (RHEL 10): `mkpasswd --method=yescrypt <password>`
* View shadow entry for a user: `getent shadow <username>`
* Verify integrity of password files (/etc/passwd & /etc/shadow): `pwck -s`
* List aging information for all users > UID_MIN: `awk -F':' '{ if ( $3 >= 1000 ) print $0 }' /etc/passwd | cut -d: -f1 | xargs -I {} chage -l {}`

## Shell

* Print the list of shells listed in /etc/shells: `chsh -l`
* Change your finger information: `chfn <username>`
* Change your login shell: `chsh -s /bin/sh <username>`
* Set the nologin shell for an account: `usermod -s /sbin/nologin <username>`

## chage

Typically if the password is expired, users are forced to change it during their next login. You can also set an additional condition where, after the password is expired, if the user never tried to login for X days, the account is automatically locked using option `-I`.

Once an account is locked due to inactivity, only system administrators will be able to unlock it.

> **Note:** Password aging settings in `/etc/login.defs` only apply to accounts created **after** the change. Use `chage` to update existing accounts.

* Operate in interactive mode: `chage <username>`
* Show account aging information: `chage -l <username>`
* Force password change on next login: `chage -d 0 <username>` or `passwd -e <username>`
* Set the account expiry date: `chage -E 2025-12-31 <username>`
* Lock account after X days of inactivity: `chage -I 30 <username>`
* Disable password aging: `chage -m 0 -M 99999 -I -1 -E -1 <username>`
* Set password max age (days between password change): `chage -M 90 <username>`
* Apply hardened aging to an existing user: `chage -M 90 -m 1 -W 14 <username>`
* Increase account expiry by 90 days: `chage -E $(date -d "+90 days" +%Y-%m-%d) <username>`

Bulk update all regular users:

```sh
for user in $(awk -F: '$3 >= 1000 && $3 < 60000 {print $1}' /etc/passwd); do
    chage -M 90 -m 1 -W 14 "$user"
done
```

## /bin/false vs /sbin/nologin

Use of the nologin shell prevents interactive use of the system, but does not prevent all access. A user may still be able to authenticate and upload or retrieve files through applications such as web applications, file transfer programs, or mail readers.

* `/bin/false` — returns non-zero exit code, silent
* `/sbin/nologin` — politely refuses login with a message (customizable via `/etc/nologin.txt`), recommended for service accounts. Some FTP servers still allow access with this shell.

```
$ ssh serviceuser@host
This account is currently not available.
```

## Creating Bulk Users

The format is the same as `/etc/passwd`. Passwords will be encrypted automatically. Files are not copied from `/etc/skel`.

```sh
vi list_users
```

```
jsmith:password:1002:1002:John Smith:/home/jsmith:/bin/bash
jdoe:password:1003:1003:Jane Doe:/home/jdoe:/bin/bash
```

```sh
newusers list_users
```

## Changing UID/GID for an Account

```sh
# Change UID and GID
usermod -u 1006 jsmith
groupmod -g 1006 jsmith

# Find and fix ownership of old UID/GID files
find / -user 1005 -exec chown -h jsmith {} \;
find / -group 1005 -exec chgrp -h jsmith {} \;

# Verify (list files with old UID/GID)
find / -user 1005 -exec ls -l {} \;
find / -group 1005 -exec ls -l {} \;
```

## pam_tally2 (RHEL 6–7)

> **Deprecated:** `pam_tally2` is removed in RHEL 8. Use `faillock` instead (see below).

* Check maximum attempts for a user: `pam_tally2 -u <username>`
* Reset the fail login counter: `pam_tally2 -r -u <username>`
* Reset using pam_tally: `pam_tally --reset --user <username>`

## faillock (RHEL 8+)

* View all unsuccessful login attempts: `faillock`
* View failed login attempts for a particular user: `faillock --user <username>`
* Clear a user's authentication failure logs: `faillock --user <username> --reset`
* Reset all users: `faillock --reset`

### Configuration (`/etc/security/faillock.conf`)

```
# Number of failed attempts before lockout
deny = 5

# Time in seconds before the account is unlocked (0 = manual unlock only)
unlock_time = 900

# Time window in seconds for counting failures
fail_interval = 900

# Lock out root account as well (default: no)
even_deny_root = no

# Root unlock time (if even_deny_root = yes)
root_unlock_time = 60
```

## Changes Across RHEL Versions (6 → 10)

### UID Ranges

| Version | Regular user UID start | System user range |
|---------|----------------------|-------------------|
| RHEL 6 | 500 | 1–499 |
| RHEL 7+ | 1000 | 1–999 (static: 1–200, dynamic: 201–999) |

### Password Hashing Algorithms

| Prefix | Algorithm | Default in |
|--------|-----------|-----------|
| `$1$` | MD5 | Legacy (RHEL 5 and earlier) |
| `$5$` | SHA-256 | Optional (RHEL 6+) |
| `$6$` | SHA-512 | RHEL 6, 7, 8, 9 |
| `$y$` | yescrypt | RHEL 10 (Fedora 35+) |

**yescrypt** is based on scrypt and NIST-approved primitives. It is stronger against GPU/ASIC attacks than SHA-512. The hash format differs from the numeric prefix convention:

```
$y$j9T$salt$hash
```

Where `j9T` encodes the cost parameters (CPU and memory factors).

To change the default algorithm on RHEL 6/7:

```sh
# RHEL 6/7 (deprecated in RHEL 8+)
authconfig --passalgo sha512 --update
```

To change the default algorithm on RHEL 8+:

```sh
# Edit /etc/login.defs
ENCRYPT_METHOD yescrypt    # or SHA512, SHA256
```

### authconfig → authselect

| Version | Tool | Notes |
|---------|------|-------|
| RHEL 6–7 | `authconfig` | Manages PAM/NSS configuration |
| RHEL 8 | `authselect` | `authconfig` deprecated (compat package available) |
| RHEL 9 | `authselect` | `authconfig` compat package removed |
| RHEL 10 | `authselect` | Only option |

Common `authselect` usage:

```sh
# List available profiles
authselect list

# Select sssd profile with mkhomedir
authselect select sssd with-mkhomedir

# Show current configuration
authselect current

# Create a custom profile
authselect create-profile myprofile -b sssd
```

### Account Lockout: pam_tally2 → pam_faillock

| Version | Module | Config |
|---------|--------|--------|
| RHEL 6–7 | `pam_tally2` | `/etc/pam.d/` |
| RHEL 8+ | `pam_faillock` | `/etc/security/faillock.conf` |

`pam_tally2` is completely removed in RHEL 8. Use `pam_faillock` instead:

```sh
# View failed login attempts
faillock --user <username>

# Reset the failure counter
faillock --user <username> --reset

# View all users with failures
faillock
```

Configuration in `/etc/security/faillock.conf`:

```
deny = 5
unlock_time = 900
fail_interval = 900
```

### /usr Merge (RHEL 8+)

Starting with RHEL 8, `/bin`, `/sbin`, `/lib`, and `/lib64` are symlinks to their `/usr` counterparts:

* `/sbin/nologin` → `/usr/sbin/nologin`
* `/bin/bash` → `/usr/bin/bash`

Both paths still work (symlinked), but canonical paths are under `/usr`.

### SSH Key Authentication Changes (RHEL 9+)

RHEL 9 disables RSA/SHA-1 signatures by default due to the system-wide crypto policy. This affects user key-based authentication:

```sh
# Generate a modern key (preferred)
ssh-keygen -t ed25519 -C "user@host"

# RSA still works but requires SHA-2 (default on modern clients)
ssh-keygen -t rsa -b 4096 -C "user@host"
```

If old RSA/SHA-1 keys stop working after upgrading to RHEL 9+, either regenerate with ed25519 or temporarily allow SHA-1:

```sh
update-crypto-policies --set DEFAULT:SHA1
```

### Crypto Policies (RHEL 8+)

System-wide cryptographic policies affect user authentication, SSH, and TLS:

```sh
# View current policy
update-crypto-policies --show

# Available policies: LEGACY, DEFAULT, FUTURE, FIPS
update-crypto-policies --set FUTURE

# RHEL 10 stricter defaults — weaker algorithms disabled out of the box
```

### Summary Table

| Feature | RHEL 6 | RHEL 7 | RHEL 8 | RHEL 9 | RHEL 10 |
|---------|--------|--------|--------|--------|---------|
| UID_MIN | 500 | 1000 | 1000 | 1000 | 1000 |
| Hash algorithm | SHA-512 | SHA-512 | SHA-512 | SHA-512 | yescrypt |
| Auth config tool | authconfig | authconfig | authselect | authselect | authselect |
| Account lockout | pam_tally2 | pam_tally2 | pam_faillock | pam_faillock | pam_faillock |
| SSH default key | RSA | RSA | RSA | ed25519 | ed25519 |
| /usr Merge | No | No | Yes | Yes | Yes |
| Crypto policies | No | No | Yes | Yes | Yes (stricter) |
