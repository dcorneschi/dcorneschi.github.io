# RHEL LAMP Stack Setup Guide

Install and configure a LAMP (Linux, Apache, MariaDB, PHP) stack on Red Hat Enterprise Linux 7+ for local web development.

## Install Packages

```bash
# Install Apache, MariaDB, and PHP with MySQL bindings
yum -y install httpd mariadb-server php-mysql php
```

This installs:
- `httpd` — the Apache web server
- `mariadb-server` — the MariaDB database (drop-in replacement for MySQL shipped with RHEL 7)
- `php-mysql` — PHP MySQL/MariaDB bindings
- `php` — the PHP interpreter

## Manage Services with systemctl

| Command | What It Does |
|---------|--------------|
| `systemctl status httpd` | Show service info: PID, child processes, uptime, man pages, recent logs |
| `systemctl start httpd mariadb` | Start the httpd and mariadb services |
| `systemctl stop httpd mariadb` | Stop the services |
| `systemctl restart httpd mariadb` | Restart the services |
| `systemctl enable httpd mariadb` | Enable services to start at boot |
| `systemctl disable httpd mariadb` | Disable services from starting at boot |
| `systemctl mask httpd` | Completely prevent a service from being started (even manually) |
| `systemctl unmask httpd` | Reverse a mask, allowing the service to start again |

## Check Logs with journalctl

You must be a member of the `adm` group or run with `sudo` to see all log messages.

| Command | What It Does |
|---------|--------------|
| `journalctl -f -l` | Follow the system log in real time, without truncating long lines |
| `journalctl -f -l -u httpd -u mariadb` | Follow logs for httpd and mariadb only |
| `journalctl -f -l -u httpd -u mariadb --since -300` | Same, but only messages less than 300 seconds old |

## Get the Server IP

| Command | What It Does |
|---------|--------------|
| `nmcli d` | Show status of all network interfaces |
| `nmcli d show eth0` | Show details of eth0 (or use `ip a s eth0`) |
| `nmcli d connect eth0` | Bring up the eth0 interface |
| `nmcli d disconnect eth0` | Bring down the eth0 interface |

## Drop a Test PHP File

```bash
cat << EOF > /var/www/html/test.php
<?php
    phpinfo();
?>
EOF
```

All text between the first line and `EOF` is written to `/var/www/html/test.php`. Any existing content is overwritten. This syntax is called a heredoc.

## Verify It Works

```bash
# From the server itself
curl http://localhost/test.php

# From another machine (replace with your server's IP)
curl http://10.0.0.10/test.php
```

## File Ownership for Development

Files in `/var/www/html` are owned by `apache` by default. For a dev environment, share ownership with a developer group:

```bash
# Change ownership of a file to apache user and developers group
chown apache:developers /var/www/html/test.php

# Recursively change group ownership of the web root
chown -R :developers /var/www/html

# Allow group members to read and write
chmod g+rw /var/www/html/test.php

# Set SGID so new files inherit the directory's group
chmod g+s /var/www/html
```

## SELinux Considerations

RHEL ships with SELinux enabled. Apache can only read files with the correct SELinux labels.

| Command | What It Does |
|---------|--------------|
| `ls -lZ test.php` | Show the SELinux label of a file |
| `restorecon -FvR /var/www/html` | Restore default labels on all files under `/var/www/html` |
| `ausearch -sv no --comm httpd` | Search audit log for denied events triggered by Apache |
| `getenforce` | Show current SELinux mode (Disabled, Permissive, Enforcing) |
| `setenforce 1` | Switch to enforcing mode |

Required labels for web content:
- `httpd_sys_content_t` — content readable by Apache
- `httpd_sys_rw_content_t` — content readable and writable by Apache

### Allow Apache to Connect to Remote Databases

If your database is on a separate server, SELinux blocks outbound connections by default:

```bash
# Allow httpd to make database connections
setsebool httpd_can_network_connect_db 1

# Make it permanent (survives reboot)
setsebool -P httpd_can_network_connect_db 1

# View all SELinux booleans
getsebool -a

# View SELinux file context rules for /var/www
semanage fcontext -l | grep '/var/www'
```

> **Note:** Install `policycoreutils-python` with yum to get the `semanage` command.

## Java Alternative

If you prefer Java over PHP:

```bash
# Enable Software Collections repo (required for Maven)
subscription-manager repos --enable rhel-server-rhscl-7-rpms

# Install Java compiler, Tomcat, Maven, and Git
yum -y install java-1.8.0-openjdk-devel tomcat maven30 git
```

## Quick Start Checklist

1. Install packages: `yum -y install httpd mariadb-server php-mysql php`
2. Start services: `systemctl start httpd mariadb`
3. Enable at boot: `systemctl enable httpd mariadb`
4. Check logs: `journalctl -f -l -u httpd -u mariadb`
5. Get IP: `nmcli d show eth0`
6. Drop test file: `cat << EOF > /var/www/html/test.php`
7. Verify: `curl http://localhost/test.php`
8. Fix SELinux if needed: `restorecon -FvR /var/www/html`

## See Also

- [SELinux Cheatsheet](articles/selinux-cheatsheet.md) — full SELinux reference
- [journalctl Cheatsheet](articles/journalctl-cheatsheet.md) — log querying and filtering
- [nmcli Cheatsheet](articles/nmcli-cheatsheet.md) — network management
- [Linux File Permissions Guide](articles/linux-file-permissions.md) — chmod, chown, ACLs
