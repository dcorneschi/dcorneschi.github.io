# Reinstall and Restore MediaWiki on RHEL

Rebuild an existing MediaWiki deployment on a clean RHEL server while preserving its pages, users, uploads, configuration, extensions, and database. This is a reinstallation and recovery workflow, not a first-time wiki installation.

> **Important:** Test the complete restore on a non-production host before rebuilding the production server. A backup is useful only after its files and database can be restored successfully.

## What Must Be Preserved

A MediaWiki instance consists of application code plus persistent site data:

| Item | Typical location | Required? |
|------|------------------|-----------|
| Database | MariaDB or MySQL database such as `mediawikidb` | Yes |
| Site configuration | `/var/www/html/wiki/LocalSettings.php` | Yes |
| Uploaded files | `/var/www/html/wiki/images/` | Yes, if uploads are enabled |
| Custom extensions | `/var/www/html/wiki/extensions/` | If installed |
| Custom skins | `/var/www/html/wiki/skins/` | If installed |
| Web server configuration | `/etc/httpd/conf.d/` | If customized |
| TLS certificates and keys | Site-specific certificate paths | If reused |
| Scheduled jobs | `/etc/cron.d/`, systemd timers, or crontabs | If configured |

Pages, users, permissions, revision history, and most wiki settings are stored in the database. Copying only `/var/www/html/wiki` does not preserve a working wiki.

## Reinstallation Strategy

Use this sequence:

1. Inventory the existing MediaWiki and platform versions.
2. Put the source wiki into read-only mode.
3. Back up the database and persistent files.
4. Verify and transfer the backups.
5. Provision Apache, PHP, MariaDB, and fresh MediaWiki code on the replacement host.
6. Recreate the database and restore the dump.
7. Restore `LocalSettings.php`, uploads, compatible extensions, and custom skins.
8. Correct ownership and SELinux contexts.
9. Run MediaWiki database updates.
10. Validate the restored wiki before DNS or load-balancer cutover.

Using fresh MediaWiki core files avoids carrying obsolete, corrupted, or compromised application files into the rebuilt server. Restore site-specific data rather than blindly replacing the new installation with the complete old code tree.

## Define the Paths and Names

The examples use these values:

```bash
WIKI_ROOT=/var/www/html/wiki
DB_NAME=mediawikidb
BACKUP_DIR=/root/mediawiki-backup
SOURCE_HOST=old-wiki.example.com
```

Adjust every path, database name, and hostname for the actual environment.

## Inventory the Existing Wiki

Record the MediaWiki, PHP, database, Apache, and operating-system versions before changing anything:

```bash
cd /var/www/html/wiki
php maintenance/run.php version
php --version
mariadb --version
httpd -V
cat /etc/redhat-release
```

For MediaWiki releases that predate the maintenance command runner, use:

```bash
php maintenance/version.php
```

Also record:

- The URL and value of `$wgServer`
- The path in `$wgScriptPath`
- Database settings from `LocalSettings.php`
- Enabled extensions and skins shown on `Special:Version`
- PHP extensions from `php -m`
- Apache virtual-host and TLS configuration
- Cron jobs, systemd timers, and external integrations

Do not print or copy database passwords into tickets, chat messages, or command output retained by monitoring systems.

## Quiesce the Source Wiki

Prevent writes while taking the final backup. Add this temporarily to `LocalSettings.php`:

```php
$wgReadOnly = 'Maintenance in progress. The wiki will return shortly.';
```

Confirm that edits and uploads are rejected. For the shortest possible outage, take a preliminary backup while the wiki remains online, then enable read-only mode and take the final database dump and file synchronization immediately before cutover.

## Back Up the Source Server

Create a protected backup directory:

```bash
install -d -m 700 /root/mediawiki-backup
```

### Back Up the Database

Use a transaction-consistent dump for MariaDB or MySQL:

```bash
mariadb-dump \
  --single-transaction \
  --quick \
  --default-character-set=binary \
  --routines \
  --triggers \
  -u root -p mediawikidb \
  | gzip > /root/mediawiki-backup/mediawikidb.sql.gz
```

On systems where the command is named `mysqldump`:

```bash
mysqldump \
  --single-transaction \
  --quick \
  --default-character-set=binary \
  --routines \
  --triggers \
  -u root -p mediawikidb \
  | gzip > /root/mediawiki-backup/mediawikidb.sql.gz
```

Check that the compressed dump is readable:

```bash
gzip -t /root/mediawiki-backup/mediawikidb.sql.gz
gzip -dc /root/mediawiki-backup/mediawikidb.sql.gz | head
```

`--single-transaction` provides a consistent snapshot when all relevant tables use transactional storage such as InnoDB. Stop writes or stop MariaDB before the dump if nontransactional tables are present.

### Back Up Persistent Files

Archive the configuration, uploads, extensions, and skins:

```bash
tar --acls --xattrs -C /var/www/html/wiki -czf \
  /root/mediawiki-backup/mediawiki-files.tar.gz \
  LocalSettings.php images extensions skins
```

Back up customized Apache configuration separately:

```bash
tar -C /etc/httpd -czf \
  /root/mediawiki-backup/httpd-conf.tar.gz \
  conf conf.d conf.modules.d
```

If TLS material must be preserved, archive only the certificate, chain, and private key used by this virtual host. Protect that archive as a secret.

### Create Checksums

```bash
cd /root/mediawiki-backup
sha256sum *.gz > SHA256SUMS
sha256sum -c SHA256SUMS
```

Store a second copy of the backup outside the server being rebuilt.

## Transfer the Backup Securely

From the replacement server, use a dedicated migration key:

```bash
install -d -m 700 /root/.ssh
rsync -aP \
  -e 'ssh -i /root/.ssh/mediawiki-migration' \
  old-wiki.example.com:/root/mediawiki-backup/ \
  /root/mediawiki-backup/
```

Verify the transferred artifacts:

```bash
cd /root/mediawiki-backup
sha256sum -c SHA256SUMS
```

> **Security:** Do not copy a long-lived root private key from the old host into `/root/.ssh/id_rsa`. Prefer a temporary, dedicated key with narrowly scoped access. Remove the key and its `authorized_keys` entry after the migration.

If direct root SSH is prohibited, copy the artifacts through an approved administrative account and use `sudo` locally to place them in the protected backup directory.

## Prepare the Replacement Server

Provision Apache, PHP, MariaDB, required PHP modules, firewall rules, and fresh MediaWiki code. Use the same MediaWiki release as the source for a pure recovery. For an upgrade, first restore the old release successfully, take another backup, and then upgrade according to the supported version path.

Do not run the web installer against an empty database when restoring an existing wiki. The database dump and original `LocalSettings.php` supply the existing site configuration.

The replacement host should contain a clean MediaWiki tree at:

```text
/var/www/html/wiki
```

Before restoring, confirm that the MediaWiki release supports the installed PHP and database versions.

## Recreate the Database

Start MariaDB and open an administrative session:

```bash
systemctl enable --now mariadb
mariadb -u root -p
```

Create the database and a dedicated local account. Replace the placeholder with a strong, unique password:

```sql
CREATE DATABASE mediawikidb
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'mediawiki'@'localhost'
  IDENTIFIED BY '<STRONG_DATABASE_PASSWORD>';

GRANT ALL PRIVILEGES ON mediawikidb.*
  TO 'mediawiki'@'localhost';

EXIT;
```

`WITH GRANT OPTION` is not required for the MediaWiki application account and should not be added.

Restore the database:

```bash
gzip -dc /root/mediawiki-backup/mediawikidb.sql.gz \
  | mariadb -u root -p mediawikidb
```

Confirm that tables were restored:

```bash
mariadb -u root -p -e \
  "SELECT COUNT(*) AS table_count FROM information_schema.tables WHERE table_schema='mediawikidb';"
```

## Restore Configuration and Uploaded Files

Extract the file archive into a temporary directory rather than over the fresh MediaWiki tree:

```bash
install -d -m 700 /root/mediawiki-restore

tar -xzf /root/mediawiki-backup/mediawiki-files.tar.gz \
  -C /root/mediawiki-restore
```

Restore the required site data:

```bash
install -m 640 -o root -g apache \
  /root/mediawiki-restore/LocalSettings.php \
  /var/www/html/wiki/LocalSettings.php

rsync -aHAX --delete \
  /root/mediawiki-restore/images/ \
  /var/www/html/wiki/images/
```

Review extensions and skins before copying them. Use versions compatible with the restored MediaWiki release. Copy only custom or separately installed components; do not overwrite extensions and skins bundled with the fresh release using older copies.

```bash
# Example for one reviewed custom extension
rsync -aHAX \
  /root/mediawiki-restore/extensions/ExampleExtension/ \
  /var/www/html/wiki/extensions/ExampleExtension/

# Example for one reviewed custom skin
rsync -aHAX \
  /root/mediawiki-restore/skins/ExampleSkin/ \
  /var/www/html/wiki/skins/ExampleSkin/
```

## Update LocalSettings.php

Review `/var/www/html/wiki/LocalSettings.php` and update values that changed on the replacement server:

```php
$wgServer = "https://wiki.example.com";
$wgScriptPath = "/wiki";

$wgDBserver = "localhost";
$wgDBname = "mediawikidb";
$wgDBuser = "mediawiki";
$wgDBpassword = "<STRONG_DATABASE_PASSWORD>";
```

Also verify:

- `$wgArticlePath`
- `$wgUploadDirectory` and `$wgUploadPath`
- Cache, job queue, mail, LDAP, SSO, and proxy settings
- Secret keys and upgrade keys
- Extension and skin loading statements
- Any absolute filesystem paths or old hostnames

Keep the temporary `$wgReadOnly` setting until restore validation is complete.

## Restore Extensions

Check every extension against the target MediaWiki release before enabling it. Common examples include:

- [SyntaxHighlight](https://www.mediawiki.org/wiki/Extension:SyntaxHighlight)
- [RecentActivity](https://www.mediawiki.org/wiki/Extension:RecentActivity)

Enable only installed, compatible extensions in `LocalSettings.php`. The exact extension directory and load name can differ from the public page title; follow the extension's version-specific instructions.

For example, current SyntaxHighlight packages commonly use:

```php
wfLoadExtension( 'SyntaxHighlight_GeSHi' );
```

After restoring an extension, install its PHP or Composer dependencies if required and run the MediaWiki database updater. If the wiki fails after an extension is enabled, comment out that extension's loading statement and inspect Apache and PHP logs.

## Set Ownership and SELinux Contexts

Keep application code and configuration non-writable by the web server. Allow Apache to write only to directories that require it, such as `images/`:

```bash
chown -R root:root /var/www/html/wiki
chown root:apache /var/www/html/wiki/LocalSettings.php
chmod 640 /var/www/html/wiki/LocalSettings.php
chown -R apache:apache /var/www/html/wiki/images
find /var/www/html/wiki -type d -exec chmod 755 {} +
find /var/www/html/wiki -type f -exec chmod 644 {} +
chmod 640 /var/www/html/wiki/LocalSettings.php
```

Define a persistent writable SELinux context for uploads, then restore labels:

```bash
semanage fcontext -a -t httpd_sys_rw_content_t \
  '/var/www/html/wiki/images(/.*)?'
restorecon -RFv /var/www/html/wiki
```

If the rule already exists, use `semanage fcontext -m` instead of `-a`.

When the database is remote, allow Apache to make database connections:

```bash
setsebool -P httpd_can_network_connect_db on
```

Do not disable SELinux to work around ownership or labeling problems.

## Run MediaWiki Maintenance Updates

Run the schema updater from the restored MediaWiki directory. On newer MediaWiki releases:

```bash
cd /var/www/html/wiki
sudo -u apache php maintenance/run.php update --quick
```

On older releases without `maintenance/run.php`:

```bash
cd /var/www/html/wiki
sudo -u apache php maintenance/update.php --quick
```

Review all output and resolve extension or schema errors before serving traffic. Then process a limited batch of queued jobs:

```bash
sudo -u apache php maintenance/run.php runJobs --maxjobs 100
```

For older releases:

```bash
sudo -u apache php maintenance/runJobs.php --maxjobs 100
```

## Reset a MediaWiki Password

On newer MediaWiki releases, change a local user's password with the maintenance command runner:

```bash
cd /var/www/html/wiki
sudo -u apache php maintenance/run.php changePassword \
  --user=example \
  --password='<NEW_PASSWORD>'
```

On older releases:

```bash
cd /var/www/html/wiki
sudo -u apache php maintenance/changePassword.php \
  --user=example \
  --password='<NEW_PASSWORD>'
```

> **Security:** A password supplied as a command-line argument may be visible briefly in the process list and retained in shell history. Run the command from a protected administrative session, avoid real passwords in copied examples, and clear sensitive history according to local policy. Prefer the application's normal password-reset workflow when available.

Password changes affect local MediaWiki accounts. Accounts managed by LDAP, SSO, or another external identity provider must be changed through that provider.

## Configure and Start Apache

Restore or recreate only the required Apache virtual-host configuration. Do not unpack an old `/etc/httpd` archive directly over a newer RHEL installation without reviewing module names, paths, and syntax.

Validate Apache before starting it:

```bash
apachectl configtest
systemctl enable --now httpd
systemctl status httpd
```

If `firewalld` is active:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

## Validate the Restored Wiki

Test locally before changing public DNS or the load balancer. Send the production hostname to the replacement server with `curl --resolve`:

```bash
curl -Ik --resolve wiki.example.com:443:127.0.0.1 \
  https://wiki.example.com/wiki/Main_Page
```

Validate all of the following:

1. Main page and several historical revisions load.
2. Local users can sign in.
3. Expected groups and permissions remain intact.
4. Images and other uploaded files load.
5. A test edit and upload succeed after read-only mode is removed.
6. `Special:Version` lists the expected MediaWiki, PHP, extensions, and skins.
7. Search, email, scheduled jobs, APIs, SSO, and external integrations work.
8. Apache, PHP, MediaWiki, MariaDB, and SELinux logs show no new errors.
9. HTTPS presents the correct certificate and complete chain.
10. A new database and file backup can be created from the replacement host.

Useful checks:

```bash
journalctl -u httpd -u mariadb --since '-15 minutes'
tail -f /var/log/httpd/error_log
ausearch -m AVC -ts recent
```

## Complete the Cutover

After validation:

1. Take a final source backup if any writes occurred after the earlier dump.
2. Restore the final dump and synchronize final uploads.
3. Change DNS or the load-balancer target.
4. Remove or comment out `$wgReadOnly` on the replacement host.
5. Perform a test edit, upload, login, and logout through the public URL.
6. Monitor HTTP errors, PHP errors, database health, and background jobs.
7. Keep the old host isolated but available for rollback until the acceptance period ends.
8. Revoke and remove temporary migration SSH keys.

Do not allow both the old and replacement wiki to accept writes independently. Divergent databases and upload directories are difficult to reconcile safely.

## Rollback

Before cutover, document a rollback trigger and owner. A basic rollback is:

1. Put the replacement wiki into read-only mode.
2. Point DNS or the load balancer back to the source host.
3. Remove read-only mode from the source only after confirming it still has the authoritative database and uploads.
4. Preserve replacement-host logs and failed restore artifacts for analysis.

If users wrote data to the replacement wiki, do not simply switch back without deciding how those changes will be preserved.

## Quick Reinstallation Checklist

- [ ] Record MediaWiki, PHP, MariaDB, Apache, OS, extension, and skin versions.
- [ ] Enable MediaWiki read-only mode.
- [ ] Dump and test the database backup.
- [ ] Back up `LocalSettings.php`, `images/`, custom extensions, and custom skins.
- [ ] Create and verify SHA-256 checksums.
- [ ] Store a backup outside the source server.
- [ ] Provision a compatible replacement stack and fresh MediaWiki code.
- [ ] Restore the database and verify table counts.
- [ ] Restore configuration and uploads.
- [ ] Review every extension and skin for compatibility.
- [ ] Apply ownership and persistent SELinux contexts.
- [ ] Run the MediaWiki schema updater and queued jobs.
- [ ] Validate with the production hostname before cutover.
- [ ] Remove read-only mode only after the final synchronization.
- [ ] Monitor the replacement and retain a tested rollback path.
- [ ] Revoke temporary migration credentials.

## See Also

- [Installing MediaWiki on RHEL 8/9](articles/mediawiki-installation-rhel.md) — provision Apache, PHP, MariaDB, and MediaWiki prerequisites
- [Apache HTTP Server Administration on RHEL](articles/apache-http-server-administration-rhel.md) — Apache configuration, TLS, security, and troubleshooting
- [Installing MariaDB](articles/mariadb-installation.md) — MariaDB installation and administration
- [SELinux Cheatsheet](articles/selinux-cheatsheet.md) — contexts, booleans, and denial troubleshooting
