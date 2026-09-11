# Apache HTTP Server Administration on RHEL

Install, tune, secure, benchmark, and monitor Apache HTTP Server (`httpd`) on Red Hat Enterprise Linux. The examples target Apache 2.4 on RHEL 7, 8, and 9, with notes for older Apache 2.2 terminology.

> **Note:** RHEL 5 and 6 have reached end of life. Keep legacy systems isolated and migrate them to a supported release.

## Install and Start Apache

On RHEL 8 and 9:

```bash
dnf install httpd httpd-tools
systemctl enable --now httpd
```

On RHEL 7:

```bash
yum install httpd httpd-tools
systemctl enable --now httpd
```

Open HTTP and HTTPS if `firewalld` is active:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

Verify the service and configuration:

```bash
systemctl status httpd
apachectl configtest
curl -I http://localhost/
```

The main RHEL paths are:

| Purpose | Path |
|---------|------|
| Main configuration | `/etc/httpd/conf/httpd.conf` |
| Additional configuration | `/etc/httpd/conf.d/*.conf` |
| Module configuration | `/etc/httpd/conf.modules.d/*.conf` |
| Document root | `/var/www/html` |
| Access log | `/var/log/httpd/access_log` |
| Error log | `/var/log/httpd/error_log` |

## Understand MPM Limits

Apache uses one Multi-Processing Module (MPM) to manage requests. Check the active MPM before changing worker limits:

```bash
httpd -V | grep -i 'Server MPM'
httpd -M | grep mpm
```

The important directive names differ by Apache generation:

| Apache 2.2 name | Apache 2.4 name | Purpose |
|-----------------|-----------------|---------|
| `MaxClients` | `MaxRequestWorkers` | Maximum simultaneous requests |
| `MaxRequestsPerChild` | `MaxConnectionsPerChild` | Requests handled before recycling a child |

`MaxClients` therefore appears in older RHEL documentation and log messages, while current Apache 2.4 configurations use `MaxRequestWorkers`.

### Prefork MPM

Prefork creates separate child processes, each of which handles one request at a time. `ServerLimit` is the upper bound for `MaxRequestWorkers`:

```apache
# /etc/httpd/conf.d/mpm-tuning.conf
<IfModule mpm_prefork_module>
    StartServers             5
    MinSpareServers          5
    MaxSpareServers         10
    ServerLimit            300
    MaxRequestWorkers      300
    MaxConnectionsPerChild 1000
</IfModule>
```

When raising the prefork limit above the default ceiling of 256, raise `ServerLimit` correspondingly. Do not increase it without first measuring the memory consumed by each child process.

A conservative sizing estimate is:

```text
MaxRequestWorkers = memory available to Apache / average Apache child RSS
```

For example, if Apache may use 3,000 MiB and a representative child consumes 50 MiB, begin with no more than approximately 60 workers. Leave headroom for the operating system and traffic spikes; swapping usually causes worse performance than briefly queueing requests.

### Worker and Event MPMs

Worker and event MPMs create multiple threads in each child process. Their capacity is determined by both process and thread limits:

```text
Maximum capacity = ServerLimit × ThreadsPerChild
```

`MaxRequestWorkers` must not exceed that capacity and should be a multiple of `ThreadsPerChild`:

```apache
# /etc/httpd/conf.d/mpm-tuning.conf
<IfModule mpm_event_module>
    StartServers             3
    ServerLimit             16
    ThreadsPerChild         25
    MinSpareThreads         75
    MaxSpareThreads        250
    MaxRequestWorkers      400
    MaxConnectionsPerChild 1000
</IfModule>
```

The same thread directives apply to `mpm_worker_module`. Event MPM is generally preferred for modern threaded deployments because idle keep-alive connections do not occupy worker threads.

> **Important:** Do not copy these values directly into production. Measure process memory, application behavior, CPU use, file-descriptor limits, and expected concurrency first.

## Tune Spare Capacity

Spare workers allow Apache to handle bursts without creating processes or threads during the request.

For prefork:

| Directive | Purpose | Traditional default |
|-----------|---------|---------------------|
| `StartServers` | Child processes created at startup | 5 |
| `MinSpareServers` | Minimum idle child processes | 5 |
| `MaxSpareServers` | Maximum idle child processes | 10 |

For worker and event:

| Directive | Purpose |
|-----------|---------|
| `StartServers` | Child processes created at startup |
| `MinSpareThreads` | Minimum idle threads across all children |
| `MaxSpareThreads` | Maximum idle threads across all children |
| `ThreadsPerChild` | Worker threads created in each child |

Too few spare workers can increase latency during sudden bursts. Too many consume memory without serving requests. Tune them from observed busy and idle worker counts rather than from request rate alone.

After changing MPM settings, validate and reload Apache:

```bash
apachectl configtest
systemctl reload httpd
journalctl -u httpd --since '-5 minutes'
```

If the worker limit is reached, the error log may contain messages similar to:

```text
server reached MaxRequestWorkers setting, consider raising the MaxRequestWorkers setting
```

Older Apache releases report `MaxClients` instead. Check both the journal and error log:

```bash
journalctl -u httpd | grep -E 'Max(RequestWorkers|Clients)'
grep -E 'Max(RequestWorkers|Clients)' /var/log/httpd/error_log
```

Do not automatically raise the limit. First determine whether memory, CPU, a slow application, long keep-alive timeouts, or abusive traffic is exhausting the workers.

## Benchmark with ApacheBench

The `httpd-tools` package provides ApacheBench (`ab`). Run it only against systems you own or are explicitly authorized to test.

```bash
# 1,000 requests, with up to 20 concurrent requests
ab -n 1000 -c 20 http://127.0.0.1/

# Benchmark a virtual host through the loopback interface
ab -n 1000 -c 20 -H 'Host: www.example.com' http://127.0.0.1/

# Test a keep-alive workload
ab -k -n 1000 -c 20 https://www.example.com/
```

Important output fields include:

| Field | Meaning |
|-------|---------|
| `Failed requests` | Requests that failed or returned unexpected content lengths |
| `Requests per second` | Overall completed-request rate |
| `Time per request` | Mean latency, reported in two forms |
| `Transfer rate` | Response throughput |
| `Percentage of the requests served within...` | Latency percentiles |

Increase concurrency gradually while watching CPU, memory, worker utilization, errors, and response-time percentiles. A benchmark from the same host does not include real network latency and competes with Apache for local resources.

## Enable HTTP Compression

`mod_deflate` compresses eligible response bodies before transmission. Confirm that the module is loaded:

```bash
httpd -M | grep deflate
```

Create `/etc/httpd/conf.d/deflate.conf`:

```apache
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/plain text/html text/xml text/css
    AddOutputFilterByType DEFLATE application/javascript application/json
    AddOutputFilterByType DEFLATE application/xml application/xhtml+xml

    # Work around problematic legacy clients and proxies.
    BrowserMatch ^Mozilla/4 gzip-only-text/html
    BrowserMatch ^Mozilla/4\.0[678] no-gzip
    BrowserMatch \bMSIE !no-gzip !gzip-only-text/html
    Header append Vary User-Agent env=!dont-vary
</IfModule>
```

Do not recompress formats that are already compressed, such as JPEG, PNG, ZIP, gzip, MP4, and most modern font formats. Validate and reload:

```bash
apachectl configtest
systemctl reload httpd
curl -sI -H 'Accept-Encoding: gzip' http://localhost/ | grep -iE 'content-encoding|vary'
```

A compressible response should normally include `Content-Encoding: gzip` and an appropriate `Vary` header.

## Configure TLS/SSL

Install the SSL module and OpenSSL:

```bash
dnf install mod_ssl openssl
```

Use `yum` instead of `dnf` on RHEL 7. Installing `mod_ssl` creates `/etc/httpd/conf.d/ssl.conf` and enables port 443.

### Create a private key and CSR

Create the private key with restrictive permissions:

```bash
install -d -m 700 /etc/pki/tls/private
openssl genrsa -out /etc/pki/tls/private/www.example.com.key 2048
chmod 600 /etc/pki/tls/private/www.example.com.key
```

Create a certificate signing request (CSR). Replace the subject values and ensure the certificate authority includes every required DNS name in the Subject Alternative Name (SAN) extension:

```bash
openssl req -new -sha256 \
  -key /etc/pki/tls/private/www.example.com.key \
  -out /etc/pki/tls/certs/www.example.com.csr \
  -subj '/C=RO/ST=Timis/L=Timisoara/O=Example/CN=www.example.com'
chmod 600 /etc/pki/tls/certs/www.example.com.csr
```

Inspect the CSR before submitting it:

```bash
openssl req -noout -text -in /etc/pki/tls/certs/www.example.com.csr
```

Place the issued certificate and intermediate chain in `/etc/pki/tls/certs/`. Do not delete or overwrite an existing production key or certificate without first backing it up and confirming a rollback path.

### Create a self-signed certificate for testing

On RHEL 8 and 9, create a one-year self-signed certificate with a SAN:

```bash
openssl req -x509 -newkey rsa:2048 -sha256 -nodes -days 365 \
  -keyout /etc/pki/tls/private/www.example.com.key \
  -out /etc/pki/tls/certs/www.example.com.crt \
  -subj '/C=RO/ST=Timis/L=Timisoara/O=Example/CN=www.example.com' \
  -addext 'subjectAltName=DNS:www.example.com,DNS:example.com'
chmod 600 /etc/pki/tls/private/www.example.com.key
chmod 644 /etc/pki/tls/certs/www.example.com.crt
```

> **Note:** Older OpenSSL versions may not support `-addext`. On RHEL 7, define `subjectAltName` in an OpenSSL configuration file and pass it with `-config`; a CN-only certificate is not sufficient for modern clients.

Self-signed certificates are suitable for testing but are not trusted by clients unless the issuing certificate is added to their trust stores.

### Configure the TLS virtual host

Create or update a virtual host under `/etc/httpd/conf.d/`, for example `/etc/httpd/conf.d/www.example.com-ssl.conf`:

```apache
<VirtualHost *:443>
    ServerName www.example.com
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/www.example.com.crt
    SSLCertificateKeyFile /etc/pki/tls/private/www.example.com.key
    # Use this when the CA supplies a separate intermediate chain:
    # SSLCertificateChainFile /etc/pki/tls/certs/www.example.com-chain.crt

    ErrorLog logs/www.example.com_ssl_error_log
    CustomLog logs/www.example.com_ssl_access_log combined
</VirtualHost>
```

Confirm that the key matches the certificate by comparing their public keys:

```bash
openssl x509 -in /etc/pki/tls/certs/www.example.com.crt -pubkey -noout | sha256sum
openssl pkey -in /etc/pki/tls/private/www.example.com.key -pubout | sha256sum
```

The hashes must match. Then validate, restart, and inspect the served certificate:

```bash
apachectl configtest
systemctl restart httpd
openssl s_client -connect www.example.com:443 -servername www.example.com -showcerts </dev/null
openssl x509 -in /etc/pki/tls/certs/www.example.com.crt -noout -subject -issuer -dates -ext subjectAltName
```

Enumerate enabled protocol versions and cipher suites from an authorized host:

```bash
nmap --script ssl-enum-ciphers -p 443 www.example.com
```

## Disable HTTP TRACE

Disable TRACE globally by adding the following directive to `/etc/httpd/conf.d/security.conf`:

```apache
TraceEnable Off
```

Validate, reload, and test:

```bash
apachectl configtest
systemctl reload httpd
curl -i -X TRACE http://localhost/
```

The TRACE request should be rejected, normally with `405 Method Not Allowed`. Apache does not implement the nonstandard `TRACK` method, but application frameworks or proxies in front of Apache should be checked separately.

A rewrite rule can reject TRACE inside a specific virtual host when a global change is not possible:

```apache
RewriteEngine On
RewriteCond %{REQUEST_METHOD} =TRACE
RewriteRule ^ - [F]
```

Prefer `TraceEnable Off` when you control the server-wide configuration.

## Control `.htaccess` Overrides

Apache ignores `.htaccess` files when `AllowOverride None` is configured. This default is faster and keeps configuration centralized. Enable only the override classes an application requires, and limit them to the smallest directory scope possible:

```apache
<Directory "/var/www/html/example-app">
    Options -Indexes +FollowSymLinks
    AllowOverride FileInfo AuthConfig Limit
    Require all granted
</Directory>
```

Use `AllowOverride All` only when the application genuinely requires all override classes:

```apache
<Directory "/var/www/html/example-app">
    Options -Indexes +FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

`Require all granted` is Apache 2.4 syntax. Legacy Apache 2.2 configurations used `Order allow,deny` and `Allow from all`.

Check which directives are allowed in `.htaccess`, then validate and reload:

```bash
apachectl configtest
systemctl reload httpd
```

If a rule does not work, confirm that its module is loaded and review `/var/log/httpd/error_log`. Avoid enabling directory indexes unless exposing file listings is intentional.

## Monitor Requests with apachetop

`apachetop` provides a live, top-like view of Apache access logs. Package availability depends on the enabled RHEL repositories; it is commonly available from EPEL.

```bash
# RHEL 8/9 after enabling an appropriate repository
sudo dnf install apachetop

# RHEL 7 after enabling an appropriate repository
sudo yum install apachetop
```

Monitor one or more logs:

```bash
apachetop -f /var/log/httpd/access_log
apachetop -f /var/log/httpd/access_log -f /var/log/httpd/another_access_log
```

The user running `apachetop` must have permission to read the logs. Use it to identify busy URLs, response codes, client addresses, and traffic changes, but use persistent monitoring for historical analysis and alerting.

## Verify and Troubleshoot

| Check | Command |
|-------|---------|
| Validate syntax | `apachectl configtest` |
| Show active virtual hosts | `httpd -S` |
| Show loaded modules | `httpd -M` |
| Show compile-time settings and MPM | `httpd -V` |
| Follow service logs | `journalctl -fu httpd` |
| Follow Apache errors | `tail -f /var/log/httpd/error_log` |
| Verify listening ports | `ss -lntp \| grep httpd` |
| Test HTTP locally | `curl -I http://localhost/` |
| Test HTTPS with SNI | `openssl s_client -connect localhost:443 -servername www.example.com </dev/null` |
| Review SELinux denials | `ausearch -m AVC -ts recent` |

For configuration changes, use this sequence:

```bash
apachectl configtest && systemctl reload httpd
```

Use a restart instead of a reload when changing loaded modules or when the new configuration does not take effect. If Apache fails to start, inspect both `journalctl -u httpd` and `/var/log/httpd/error_log` before changing additional settings.

## Quick Checklist

1. Install `httpd` and `httpd-tools`.
2. Enable and start `httpd`.
3. Open only the required firewall services.
4. Identify the active MPM before tuning it.
5. Size `MaxRequestWorkers` from measured memory and concurrency.
6. Keep `ServerLimit`, `ThreadsPerChild`, and `MaxRequestWorkers` consistent.
7. Validate every configuration change with `apachectl configtest`.
8. Benchmark only authorized systems and increase concurrency gradually.
9. Enable compression only for compressible content types.
10. Protect private keys and use SAN-enabled certificates.
11. Set `TraceEnable Off`.
12. Keep `AllowOverride None` unless an application requires scoped overrides.
13. Monitor error logs and worker saturation after each tuning change.

## See Also

- [Monitoring Apache Web Server Performance](articles/monitoring-apache-performance.md) — MPM internals, worker sizing, `mod_status`, metrics, and alerting
- [RHEL LAMP Stack Setup](articles/rhel-lamp-stack-setup.md) — Apache, MariaDB, PHP, SELinux, and service management
- [Installing DokuWiki on RHEL](articles/dokuwiki-installation-rhel.md) — application-specific `.htaccess` setup
- [SNI Certificates Guide](articles/sni-certificates-guide.md) — serving multiple TLS certificates from one address
- [SELinux Cheatsheet](articles/selinux-cheatsheet.md) — contexts, booleans, and denial troubleshooting
