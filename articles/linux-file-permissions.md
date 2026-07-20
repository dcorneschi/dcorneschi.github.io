# Linux File Permissions Guide

## Who Can Change Permissions and Ownership

Only the **owner** and **root** (super user) are allowed to change the permissions of a file or directory. This means that the owner and the super user can set the read (`r`), write (`w`) and execute (`x`) permissions.

However, changing the ownership (user/group) of files and directories with `chown`/`chgrp` is only allowed to **root**. The owner of a file may change the group of the file to any group of which that owner is a member.

**Summary for normal operation:**

- Only root and the owner can `chmod`
- Only root can `chown`
- Only root and the owner can `chgrp` — the owner can change the group as long as they are a member of the target group

**Security consideration:** any user with write permission to the directory containing the file can replace the file with a copy, and thus become the owner, gaining the ability to modify the permissions and contents.

> **Note:** The user needs to be part of the group in order to set SGID on the directory.

**Additional considerations:**

- **Linux capabilities** — root isn't the only way to bypass permission checks. A process with `CAP_FOWNER` can `chmod` any file, and `CAP_CHOWN` can `chown`/`chgrp` any file, even without being UID 0.
- **Immutable attribute** — even root cannot modify a file if the immutable flag is set (`chattr +i`). The flag must be removed first (`chattr -i`) before any permission or ownership change.
- **SELinux / AppArmor** — traditional DAC permissions may allow an action, but a MAC (Mandatory Access Control) policy can still deny it. A file might be `chmod 777` but still inaccessible due to its SELinux context.
- **Mount options** — filesystems mounted with `nosuid` ignore setuid/setgid bits. Mounting with `ro` prevents any changes regardless of file permissions.
- **Owners cannot give away files** — on Linux, unlike some Unix systems, an unprivileged user cannot `chown` a file they own to another user. This prevents users from evading disk quotas.

---

## Understanding Permissions

Every file and directory has three permission sets:

| Set | Applies to |
|-----|-----------|
| User (u) | The file owner |
| Group (g) | Members of the file's group |
| Other (o) | Everyone else |

Each set can have:

| Permission | Effect on Regular File | Effect on Directory |
|------------|----------------------|---------------------|
| `r` (read) | Contents of file can be read | Contents of directory (file names) can be listed |
| `w` (write) | Contents of file can be changed | Any file in directory can be created or deleted |
| `x` (executable) | Contents of file can be executed as a command | Contents of directory can be accessed (dependent on file's own permissions) |

> **Note:** Read without execute on a directory lets you list filenames but not access anything. Execute without read lets you access files (if you know the name) but not list them. Write on a directory without the sticky bit means any user with that permission can delete any file in it, regardless of the file's own permissions.

### Reading Permissions

```
-rwxr-xr-- 1 alice developers 4096 Jan 15 10:30 script.sh
│├──┤├──┤├──┤
│ u    g    o
│
└── file type (- = file, d = directory, l = symlink)
```

### Numeric (Octal) Notation

| Permission | Value |
|------------|-------|
| Read | 4 |
| Write | 2 |
| Execute | 1 |

Common patterns:

| Octal | Symbolic | Typical use |
|-------|----------|-------------|
| `755` | `rwxr-xr-x` | Directories, executable scripts |
| `644` | `rw-r--r--` | Regular files |
| `700` | `rwx------` | Private home directories |
| `750` | `rwxr-x---` | Group-shared directories |
| `600` | `rw-------` | Private files (SSH keys, configs with secrets) |
| `440` | `r--r-----` | sudoers files |

---

## umask

The `umask` determines the default permissions for newly created files and directories by masking out (removing) bits from the maximum permissions.

- Maximum permissions for files: `666` (no execute by default)
- Maximum permissions for directories: `777`

**Calculation:** `default permissions = max permissions - umask`

| umask | Files created as | Directories created as |
|-------|-----------------|----------------------|
| `022` | `644` (rw-r--r--) | `755` (rwxr-xr-x) |
| `027` | `640` (rw-r-----) | `750` (rwxr-x---) |
| `077` | `600` (rw-------) | `700` (rwx------) |
| `002` | `664` (rw-rw-r--) | `775` (rwxrwxr-x) |

```sh
# Display current umask
umask

# Display in symbolic form
umask -S

# Set umask for the current session
umask 027

# Persistent: add to ~/.bashrc or /etc/profile
echo "umask 027" >> ~/.bashrc
```

---

## Special Permissions

### setuid (`u+s`)

When set on an executable file, the process runs with the **file owner's** privileges instead of the user who launched it.

```sh
# Example: passwd runs as root regardless of who calls it
ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 68208 ... /usr/bin/passwd
```

> **Security:** setuid on shell scripts is ignored by the kernel on Linux. It only applies to compiled binaries.

### setgid (`g+s`)

- **On files:** the process runs with the file's group privileges.
- **On directories:** new files and subdirectories inherit the directory's group (instead of the creator's primary group).

```sh
# Set SGID on a directory
chmod g+s /shared/project

# New files inherit the group
touch /shared/project/newfile
ls -l /shared/project/newfile
-rw-r--r-- 1 alice developers ...   # group is 'developers', not alice's primary
```

### Sticky bit (`o+t`)

When set on a directory, only the file owner, directory owner, or root can delete/rename files within it. Prevents users from deleting each other's files in shared directories.

```sh
# /tmp is the classic example
ls -ld /tmp
drwxrwxrwt 15 root root 4096 ... /tmp

# Set sticky bit
chmod +t /shared/directory
```

### Displaying Special Permissions

The `ls -l` command shows special permissions in the execute position:

- Lowercase `s` or `t` — the special permission **and** the underlying execute bit are both set
- Uppercase `S` or `T` — the special permission is set but the underlying execute bit is **not** set (unusual, draws attention)

```
drwsrwsr-t.  1 alice developers 4096 May 11 16:47 shared   # all execute bits set
drwxrwx--T.  1 alice developers 4096 May 11 16:47 dropbox  # sticky set, but other has no execute
```

### Special Permissions Summary

| Special Permission | Effect on Regular File | Effect on Directory |
|--------------------|----------------------|---------------------|
| SUID (`chmod u+s`) | File executes as user that owns the file, not the user that ran the file | No effect |
| SGID (`chmod g+s`) | File executes as group that owns the file | Files newly created in the directory have group owner set to match group that owns the directory |
| Sticky bit (`chmod o+t`) | No effect | Users with **write** on the directory can only remove files they own, they cannot remove files owned by other users |

### Symbolic and Numeric Notation

Symbolically:

| Permission | Symbol |
|------------|--------|
| setuid | `u+s` |
| setgid | `g+s` |
| sticky | `o+t` |

Numerically (fourth preceding digit):

| Permission | Value |
|------------|-------|
| setuid | 4 |
| setgid | 2 |
| sticky | 1 |

```sh
# setuid + 755
chmod 4755 /usr/local/bin/myapp

# setgid + 775
chmod 2775 /shared/project

# sticky + 777
chmod 1777 /tmp

# setgid + sticky + 775
chmod 3775 /shared/dropbox
```

---

## Immutable File Attributes (chattr/lsattr)

Beyond standard permissions and ACLs, Linux ext2/ext3/ext4 and xfs filesystems support extended file attributes set with `chattr`. These override normal permissions — even root is restricted until the attribute is removed.

### Common Attributes

| Attribute | Flag | Effect |
|-----------|------|--------|
| Immutable | `i` | File cannot be modified, deleted, renamed, or linked to. No data can be written. Even root cannot change it without first removing the flag. |
| Append only | `a` | File can only be opened in append mode for writing. Cannot be deleted or renamed. Useful for log files. |
| No dump | `d` | File is excluded from `dump` backups. |
| Secure deletion | `s` | When deleted, blocks are zeroed (filesystem support varies). |
| Undeletable | `u` | When deleted, contents are saved for potential recovery. |

### Commands

```sh
# Set immutable attribute (requires root)
chattr +i /etc/resolv.conf

# Remove immutable attribute
chattr -i /etc/resolv.conf

# Set append-only
chattr +a /var/log/audit.log

# Remove append-only
chattr -a /var/log/audit.log

# View file attributes
lsattr /etc/resolv.conf
----i--------e-- /etc/resolv.conf

# View attributes recursively
lsattr -R /etc/
```

### Practical Use Cases

```sh
# Prevent accidental modification of critical config files
chattr +i /etc/passwd
chattr +i /etc/shadow
chattr +i /etc/group
chattr +i /etc/sudoers

# Protect log files from tampering (append only)
chattr +a /var/log/messages
chattr +a /var/log/secure

# Find all immutable files on the system
find / -xdev -exec lsattr {} + 2>/dev/null | grep -e "----i"
```

> **Note:** `chattr` only works on ext2/ext3/ext4 and xfs filesystems. The `CAP_LINUX_IMMUTABLE` capability is required to set or remove the immutable flag.

---

## Access Control Lists (ACLs)

ACLs extend the traditional user/group/other model by allowing fine-grained permissions for specific users or groups on a file or directory.

### Prerequisites

The filesystem must be mounted with ACL support (enabled by default on ext4, xfs):

> **Note:** File systems created by the installer have ACLs turned on internally by default even if you do not specify it. However, file systems created by hand do **not** automatically enable the `acl` mount option.

```sh
# Check if ACLs are supported
tune2fs -l /dev/sda1 | grep "Default mount options"
# Should show: acl

# Mount with ACL support explicitly
mount -o remount,acl /mountpoint

# Enable ACL internally on the filesystem (like the installer does)
tune2fs -o acl /dev/sda1

# Remove ACL mount option from defaults
tune2fs -o ^acl /dev/sda1
```

### Setting ACLs

```sh
# Grant a specific user read/write access
setfacl -m u:bob:rw /path/to/file

# Grant a specific group read/execute access
setfacl -m g:devops:rx /path/to/directory

# Remove a specific ACL entry
setfacl -x u:bob /path/to/file

# Remove all ACLs
setfacl -b /path/to/file

# Apply recursively
setfacl -R -m u:bob:rwx /path/to/directory
```

### ACL Precedence

When a process accesses a file, permissions are resolved in 4 steps (first match wins):

| Step | Question | Permission applied |
|------|----------|-------------------|
| 1 | Is the process running as the user that owns the file? | **User** permissions apply |
| 2 | Does the file have an ACL entry for the process's user? | **User's ACL entry** applies |
| 3 | Is the process running as the group that owns the file, or a group with a group ACL entry? | Any matching **Group or Group ACL entry** granting access applies |
| 4 | None of the above match? | Permissions for **other** applies |

### Default ACLs (Inheritance)

Default ACLs on directories define the ACL that new files/subdirectories will inherit:

```sh
# Set default ACL — new files will grant bob rw access
setfacl -d -m u:bob:rw /shared/project

# Set default ACL for a group
setfacl -d -m g:devops:rwx /shared/project

# View defaults
getfacl /shared/project
```

> **Important:**
> - A new directory created inside will inherit the default ACL as expected.
> - A new regular file will have its mask set to `rw-` for security reasons (new regular files never get execute permission automatically).
> - A default ACL entry on a directory does **not** imply that the user/group has any access on the directory itself — you must also set a normal ACL or permission allowing it.

### Viewing ACLs

```sh
# A '+' at the end of permissions indicates ACLs are set
ls -l /path/to/file
-rw-rw-r--+ 1 alice developers 4096 ... file.txt

# View full ACL
getfacl /path/to/file
# file: path/to/file
# owner: alice
# group: developers
user::rw-
user:bob:rw-
group::r--
group:devops:r-x
mask::rwx
other::r--
```

### The ACL Mask

The mask defines the maximum effective permissions for named users, named groups, and the owning group. It acts as an upper bound:

```sh
# Set the mask (limits effective permissions)
setfacl -m m::rx /path/to/file

# If bob has rw but mask is rx, effective permission is r only
```

**Important behavior:**

- On a file with ACLs, `chmod g-w` changes the **mask**, not the owning group's actual permissions.
- The group permissions shown by `ls -l` represent the **mask**, not the actual owning group's permissions.
- The mask gets recalculated with every new ACL entry set unless the `-n` switch is used, so mask restrictions have to be reapplied.
- Effective rights are shown by the `getfacl` command.

```sh
# Set mask to rx — limits all named users/groups/owning group to rx max
setfacl -m m::rx script.sh

# Set group permissions without recalculating the mask
setfacl -n -m group::rwx script.sh
```

### Backup and Restore ACLs

```sh
# Backup
getfacl -R /shared/project > acl_backup.txt

# Restore
setfacl --restore=acl_backup.txt
```

---

## Common chmod/chown/chgrp Commands

```sh
# Change permissions symbolically
chmod u+x script.sh            # Add execute for owner
chmod go-w file.txt            # Remove write for group and others
chmod a+r file.txt             # Add read for all (a = u+g+o)
chmod u=rwx,g=rx,o= file.txt  # Set exact permissions

# Change permissions numerically
chmod 755 script.sh
chmod 644 config.yml

# Recursive changes
chmod -R 750 /project/dir

# Copy permissions from another file
chmod --reference=source.txt target.txt

# Change owner
chown alice file.txt
chown alice:developers file.txt   # owner and group at once
chown :developers file.txt        # group only (same as chgrp)
chown -R alice:developers /dir    # recursive

# Change group
chgrp developers file.txt
chgrp -R developers /dir

# Useful find + chmod patterns
find /var/www -type d -exec chmod 755 {} \;   # directories
find /var/www -type f -exec chmod 644 {} \;   # files
find /home/user -type f -name "*.sh" -exec chmod 750 {} \;  # scripts
```

---

## Give Write Permissions to Multiple Users on a Folder

### Solution 1 — Dedicated Group

```sh
groupadd users
usermod -a -G users user1
usermod -a -G users user2
chgrp -R users /path/to/the/directory
chmod -R 770 /path/to/the/directory
```

> **Note:** Group assignment changes won't take effect until the users log out and back in.

### Solution 2 — SGID with Shared Group

```sh
usermod -a -G www-data <some_user>
chgrp -R www-data /var/www
chmod -R g+w /var/www
find /var/www -type d -exec chmod 2775 {} \;
find /var/www -type f -exec chmod ug+rw {} \;
```

Setting the SGID bit (`2775`) on directories ensures that new files and subdirectories inherit the group ownership of the parent directory.

---

## About the `users` Group

The `users` group exists just to be assigned to users which don't need to belong in any other group, as far as permissions are concerned. It basically exists just because every user must be at least part of a primary group (which you can find in `/etc/passwd`). Think of `users` like a "fallback" — if no group is assigned to a user, the `useradd` utility uses it as a default when homonym groups are disabled.
