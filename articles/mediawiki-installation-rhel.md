# Installing MediaWiki on RHEL 8/9

This guide covers installing MediaWiki with Apache, PHP, and MariaDB on RHEL 8/9. It includes package installation, database setup, SELinux contexts, and firewall configuration.

## Overview

| Component | Purpose |
|-----------|---------|
| Apache (`httpd`) | Web server serving MediaWiki |
| PHP 7.4+ | Runtime for MediaWiki application |
| MariaDB | Database backend |
| `mod_ssl` | HTTPS support |
| SELinux | File context for web content |

---

## Install Packages

Enable the PHP 7.4 module stream and install all required packages:

```bash
dnf module reset php
dnf module enable php:7.4
dnf install httpd php php-mysqlnd php-gd php-xml mariadb-server mariadb php-mbstring php-json mod_ssl php-intl php-apcu
```

---

## Configure MariaDB

### Start and Secure

```bash
systemctl start mariadb
mysql_secure_installation
```

### Create Database and User

```bash
mysql -u root -p
```

```sql
CREATE USER 'wiki'@'localhost' IDENTIFIED BY 'password';
CREATE DATABASE mediawiki;
GRANT ALL PRIVILEGES ON mediawiki.* TO 'wiki'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Enable Services

```bash
systemctl enable mariadb
systemctl enable httpd
```

---

## Download and Install MediaWiki

```bash
cd /var/www/html/
wget https://releases.wikimedia.org/mediawiki/1.39/mediawiki-1.39.3.tar.gz
tar -zxf mediawiki-1.39.3.tar.gz
mv mediawiki-1.39.3 mediawiki
rm mediawiki-1.39.3.tar.gz
chown -R apache:apache /var/www/html/mediawiki
```

Restore SELinux context and restart Apache:

```bash
restorecon -FR /var/www/html/mediawiki
systemctl restart httpd
```

---

## Run the Web Installer

1. Open `http://your-server/mediawiki` in a browser
2. Follow the installation wizard — provide the database name (`mediawiki`), user (`wiki`), and password
3. Download the generated `LocalSettings.php` file at the end

---

## Deploy LocalSettings.php

Move the generated configuration file into the MediaWiki directory:

```bash
mv /home/ec2-user/LocalSettings.php /var/www/html/mediawiki/
chown apache:apache /var/www/html/mediawiki/LocalSettings.php
restorecon -FR /var/www/html/mediawiki
```

---

## Configure Firewall

```bash
firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --permanent --zone=public --add-service=https
firewall-cmd --reload
```

---

## Verify

Access the wiki at:

```
http://your-server/mediawiki
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| 403 Forbidden | SELinux context wrong | `restorecon -FR /var/www/html/mediawiki` |
| 403 Forbidden | File ownership wrong | `chown -R apache:apache /var/www/html/mediawiki` |
| Database connection failed | MariaDB not running | `systemctl start mariadb` |
| Database connection failed | Wrong credentials | Verify user/password in `LocalSettings.php` matches `GRANT` statement |
| Blank page or PHP errors | Missing PHP module | Check with `php -m` and install missing module |
| Can't reach site externally | Firewall blocking | `firewall-cmd --add-service=http --permanent && firewall-cmd --reload` |
| Installer won't start | `LocalSettings.php` already present | Remove or rename it to re-run the installer |
