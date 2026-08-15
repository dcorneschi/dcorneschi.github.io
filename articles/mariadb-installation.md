# Installing MariaDB on RHEL and Ubuntu

This guide covers installing and configuring MariaDB on RHEL 7–10 and Ubuntu 22.04/24.04 — service management, security hardening, user management, firewall rules, and common administration commands.

## Overview

| Component | Purpose |
|-----------|---------|
| `mariadb-server` | Database server daemon (`mysqld`) |
| `mariadb` | Command-line client |
| `mysql_secure_installation` | Post-install hardening script |
| `/etc/my.cnf` | Main configuration file (RHEL) |
| `/etc/mysql/mariadb.conf.d/` | Configuration directory (Ubuntu) |
| `/var/lib/mysql` | Default data directory |

---

## Installation

### RHEL 7

```bash
yum install -y mariadb mariadb-server
```

### RHEL 8 / 9 / 10

```bash
dnf install -y mariadb mariadb-server
```

### Ubuntu 22.04 / 24.04

```bash
apt install -y mariadb-server mariadb-client
```

---

## Service Management

### Start and Enable

```bash
systemctl start mariadb
systemctl enable mariadb
```

### Check Status

```bash
systemctl status mariadb
```

### Restart / Reload

```bash
systemctl restart mariadb
systemctl reload mariadb
```

---

## Initial Security Hardening

Run the interactive hardening script after installation:

```bash
mysql_secure_installation
```

This script will:
- Set the root password
- Remove anonymous users
- Disallow remote root login
- Remove the test database
- Reload privilege tables

---

## Configuration

### Configuration File Locations

| Distribution | Main Config | Additional Configs |
|--------------|-------------|-------------------|
| RHEL | `/etc/my.cnf` | `/etc/my.cnf.d/*.cnf` |
| Ubuntu | `/etc/mysql/my.cnf` | `/etc/mysql/mariadb.conf.d/*.cnf` |

### Sample Configuration Files

MariaDB ships with sample configurations in `/usr/share/mysql/`:

```bash
ls /usr/share/mysql/*.cnf
```

Copy one as a starting point:

```bash
cp /usr/share/mysql/my-medium.cnf /etc/my.cnf.d/custom.cnf
```

### Common Settings

Edit `/etc/my.cnf` or a file in the conf directory:

```ini
[mysqld]
bind-address = 0.0.0.0
port = 3306
datadir = /var/lib/mysql
socket = /var/lib/mysql/mysql.sock
max_connections = 100
innodb_buffer_pool_size = 256M
character-set-server = utf8mb4
collation-server = utf8mb4_general_ci

[client]
default-character-set = utf8mb4
```

### Listen on All Interfaces (Remote Access)

By default, MariaDB only listens on localhost. To allow remote connections:

```ini
[mysqld]
bind-address = 0.0.0.0
```

Then restart:

```bash
systemctl restart mariadb
```

---

## Firewall Configuration

### RHEL 7–10 (firewalld)

```bash
firewall-cmd --permanent --add-service=mysql
firewall-cmd --reload
```

### Ubuntu (ufw)

```bash
ufw allow 3306/tcp
```

### Verify Listening Port

```bash
ss -tlnp | grep 3306
```

---

## User Management

### Log In

```bash
# As root
mysql -u root -p

# As a specific user
mysql -u myuser -p

# Execute a query directly
mysql -u myuser -p -e 'SHOW DATABASES;'

# With cleartext plugin (for PAM/LDAP auth)
mysql -u myuser -p --enable-cleartext-plugin -e 'SHOW DATABASES;'
```

### Create Users

```sql
-- Local access only
CREATE USER 'myuser'@'localhost' IDENTIFIED BY 'strongpassword';

-- Access from specific host
CREATE USER 'myuser'@'192.168.1.%' IDENTIFIED BY 'strongpassword';

-- Access from any host
CREATE USER 'myuser'@'%' IDENTIFIED BY 'strongpassword';
```

### Grant Privileges

```sql
-- All privileges on all databases
GRANT ALL PRIVILEGES ON *.* TO 'myuser'@'localhost';

-- All privileges on a specific database
GRANT ALL PRIVILEGES ON mydb.* TO 'myuser'@'localhost';

-- Read-only access
GRANT SELECT ON mydb.* TO 'myuser'@'localhost';

-- Specific privileges
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'myuser'@'localhost';

-- Apply changes
FLUSH PRIVILEGES;
```

### View and Revoke Privileges

```sql
-- Show grants for a user
SHOW GRANTS FOR 'myuser'@'localhost';

-- Revoke privileges
REVOKE ALL PRIVILEGES ON mydb.* FROM 'myuser'@'localhost';

-- Drop user
DROP USER 'myuser'@'localhost';
```

### Change Password

```sql
ALTER USER 'myuser'@'localhost' IDENTIFIED BY 'newpassword';
-- or (older syntax)
SET PASSWORD FOR 'myuser'@'localhost' = PASSWORD('newpassword');
FLUSH PRIVILEGES;
```

---

## Database Management

### Create and Use Databases

```sql
CREATE DATABASE mydb;
CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
USE mydb;
SHOW DATABASES;
DROP DATABASE mydb;
```

### Backup and Restore

```bash
# Backup a single database
mysqldump -u root -p mydb > mydb_backup.sql

# Backup all databases
mysqldump -u root -p --all-databases > all_databases.sql

# Backup with compression
mysqldump -u root -p mydb | gzip > mydb_backup.sql.gz

# Restore
mysql -u root -p mydb < mydb_backup.sql

# Restore from compressed
gunzip < mydb_backup.sql.gz | mysql -u root -p mydb
```

### Table Operations

```sql
SHOW TABLES;
DESCRIBE tablename;
SHOW TABLE STATUS;
SHOW CREATE TABLE tablename;
```

---

## Useful Commands

### Server Information

```sql
SELECT VERSION();
SHOW VARIABLES LIKE 'datadir';
SHOW VARIABLES LIKE 'port';
SHOW VARIABLES LIKE 'bind_address';
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Threads_connected';
```

### Process and Connection Management

```sql
-- Show active connections
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;

-- Kill a query
KILL query_id;
```

### Check and Repair

```bash
# Check all tables in a database
mysqlcheck -u root -p --check mydb

# Repair
mysqlcheck -u root -p --repair mydb

# Optimize
mysqlcheck -u root -p --optimize mydb
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Can't connect from remote host | `bind-address = 127.0.0.1` | Set to `0.0.0.0` and restart |
| Can't connect from remote host | Firewall blocking port 3306 | `firewall-cmd --permanent --add-service=mysql && firewall-cmd --reload` |
| Can't connect from remote host | User limited to localhost | `CREATE USER 'user'@'%'` or specific host |
| Access denied for root | No password set or wrong password | `mysql_secure_installation` or reset root password |
| `ERROR 1698: Access denied for root` | Root uses unix_socket auth (Ubuntu) | `sudo mysql` or switch to `mysql_native_password` |
| Service won't start | Corrupted data directory | Check `/var/log/mariadb/mariadb.log` or `journalctl -u mariadb` |
| Service won't start | Port already in use | `ss -tlnp \| grep 3306` to find conflicting process |
| Slow queries | Missing indexes or buffer too small | Enable slow query log, increase `innodb_buffer_pool_size` |

### Log Locations

```bash
# RHEL
/var/log/mariadb/mariadb.log

# Ubuntu
/var/log/mysql/error.log

# systemd journal
journalctl -u mariadb --no-pager
```

### Reset Root Password

```bash
systemctl stop mariadb
mysqld_safe --skip-grant-tables &
mysql -u root
```

```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'newrootpassword';
```

```bash
kill $(pgrep mysqld)
systemctl start mariadb
```

---

## Version Differences

| Feature | RHEL 7 | RHEL 8 | RHEL 9 | RHEL 10 | Ubuntu 22.04 | Ubuntu 24.04 |
|---------|--------|--------|--------|---------|--------------|--------------|
| MariaDB version | 5.5 | 10.3 | 10.5 | 10.11+ | 10.6 | 10.11 |
| Package manager | yum | dnf | dnf | dnf | apt | apt |
| Config path | `/etc/my.cnf` | `/etc/my.cnf` | `/etc/my.cnf` | `/etc/my.cnf` | `/etc/mysql/` | `/etc/mysql/` |
| Service name | `mariadb` | `mariadb` | `mariadb` | `mariadb` | `mariadb` | `mariadb` |
| Root auth | password | password | unix_socket | unix_socket | unix_socket | unix_socket |
