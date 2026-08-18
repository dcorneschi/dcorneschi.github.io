# Installing DokuWiki on RHEL

DokuWiki is a lightweight, file-based wiki that requires no database. It runs on PHP and stores pages as plain text files, making backups and version control straightforward. This guide covers installation on RHEL 8/9 with Apache.

## Prerequisites

- RHEL 8 or 9 (or compatible: CentOS Stream, Rocky, AlmaLinux)
- Root or sudo access
- Internet access for package downloads

## Install Required Packages

Install Apache, PHP, and the PHP extensions DokuWiki needs:

```bash
dnf install wget httpd php php-gd php-xml php-json
```

> **Note:** On RHEL 9 with PHP 8.0+, `php-json` is bundled into PHP core. The install will simply skip it if not needed.

Set SELinux to permissive for the httpd domain (allows Apache to function while still logging denials):

```bash
semanage permissive -a httpd_t
```

Enable and start Apache:

```bash
systemctl enable httpd.service && systemctl start httpd.service
```

## Enable mod_rewrite

DokuWiki uses clean URLs via Apache's `mod_rewrite`. Create a config file to load the module:

```bash
echo LoadModule rewrite_module modules/mod_rewrite.so > /etc/httpd/conf.d/addModule-mod_rewrite.conf
systemctl restart httpd.service
```

## Download and Extract DokuWiki

Download the latest stable release and extract it to the web root:

```bash
cd /tmp/
wget https://download.dokuwiki.org/src/dokuwiki/dokuwiki-stable.tgz
tar zxvf dokuwiki-stable.tgz -C /var/www/html
```

Rename the extracted directory and set ownership:

```bash
cd /var/www/html
mv dokuwiki-2023-04-04a dokuwiki
chown apache:apache -R /var/www/html/dokuwiki
```

> **Note:** The directory name (e.g., `dokuwiki-2023-04-04a`) changes with each release. Adjust accordingly.

## Configure .htaccess

Activate the distributed `.htaccess` file:

```bash
cd /var/www/html/dokuwiki
mv .htaccess.dist .htaccess
```

Edit `.htaccess` if you need to customize rewrite rules:

```bash
vi .htaccess
```

## Configure Apache to Allow .htaccess Overrides

Edit the main Apache configuration:

```bash
vi /etc/httpd/conf/httpd.conf
```

Find the `<Directory "/var/www/html">` block and set `AllowOverride All`:

```apache
<Directory "/var/www/html">
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

Restart Apache to apply:

```bash
systemctl restart httpd.service
```

## Firewall

If `firewalld` is running, open HTTP (and optionally HTTPS):

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

## Run the Installer

Open a browser and navigate to:

```
http://<server-ip>/dokuwiki/install.php
```

Complete the web installer:

1. Set the wiki name
2. Choose ACL (access control) policy
3. Create the admin user and password
4. Select license

After installation, **remove the installer** for security:

```bash
rm /var/www/html/dokuwiki/install.php
```

## Verify

Browse to `http://<server-ip>/dokuwiki/` and confirm the wiki loads with the configured admin account.

## Troubleshooting

| Issue | Fix |
|-------|-----|
| 403 Forbidden | Check ownership: `chown apache:apache -R /var/www/html/dokuwiki` |
| SELinux denials | Review with `ausearch -m avc -ts recent` or set permissive: `semanage permissive -a httpd_t` |
| mod_rewrite not working | Confirm the module config exists and `AllowOverride All` is set |
| PHP errors | Ensure `php-gd`, `php-xml`, and `php-json` are installed |
| Install page not loading | Verify Apache is running: `systemctl status httpd` |
