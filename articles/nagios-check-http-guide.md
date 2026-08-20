# Nagios check_http Plugin Guide

`check_http` monitors HTTP/HTTPS services — response codes, content, response time, SSL certificates, and virtual host routing through reverse proxies.

## Basic Syntax

```bash
check_command   check_http![options]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-H <host>` | Host name or IP address (used for Host header and SNI) |
| `-I <ip>` | IP address to connect to (use with `-H` for virtual hosts) |
| `-p <port>` | Port number (default: 80 for HTTP, 443 for HTTPS) |
| `-S` | Use SSL/HTTPS |
| `-u <path>` | URL path to check (e.g., `/index.html`, `/api/status`) |
| `-s <string>` | Expected string in response |
| `-r <regex>` | Expected regex pattern in response |
| `-w <seconds>` | Warning threshold for response time |
| `-c <seconds>` | Critical threshold for response time |
| `-t <seconds>` | Timeout in seconds |
| `-C <days>` | Check SSL certificate expires in X days |
| `-v` | Verbose output |
| `-f <follow>` | Follow redirects |
| `-a <auth>` | Basic authentication (`username:password`) |
| `-k <header>` | Additional HTTP header |
| `--ssl=<ver>` | Force specific TLS version (`1`, `1.1`, `1.2`, `1.3`) |
| `--sni` | Enable SNI (default on most builds) |

## Basic Examples

```bash
# Basic HTTP check
check_command   check_http!-H 192.168.100.31

# Basic HTTPS check
check_command   check_http!-H 192.168.100.31 -S

# Custom port
check_command   check_http!-H 192.168.100.31 -p 8080

# HTTPS with custom port
check_command   check_http!-H 192.168.100.31 -p 8443 -S

# Check specific URL path
check_command   check_http!-H 192.168.100.31 -u /health

# Check for expected content
check_command   check_http!-H 192.168.100.31 -s "Server Status: OK"

# HTTPS with certificate expiration check (30 days warning)
check_command   check_http!-H 192.168.100.31 -S -C 30

# With response time thresholds
check_command   check_http!-H 192.168.100.31 -w 2 -c 5

# With basic authentication
check_command   check_http!-H 192.168.100.31 -a username:password

# Complex example: HTTPS, custom port, URL path, content check, timing, cert check
check_command   check_http!-H 192.168.100.31 -S -p 8443 -u /api/v1/status -s "healthy" -w 2 -c 5 -C 30
```

## Reverse Proxy / Traefik Setup

When checking a service behind Traefik or any SNI-based reverse proxy, use `-H` for the hostname and `-I` for the IP:

```bash
# CORRECT — hostname for SNI + Host header, IP for connection
./check_http -H gitea.kubectl.ro -I 192.168.50.5 -S

# With specific URL path
./check_http -H gitea.kubectl.ro -I 192.168.50.5 -S -u /user/login

# With certificate expiration check
./check_http -H gitea.kubectl.ro -I 192.168.50.5 -S -C 30

# With response time thresholds
./check_http -H gitea.kubectl.ro -I 192.168.50.5 -S -w 3 -c 8

# With expected content
./check_http -H gitea.kubectl.ro -I 192.168.50.5 -S -s "Gitea"
```

### How Traefik Routing Works with check_http

1. `check_http` connects to `192.168.50.5:443`
2. During SSL handshake, sends SNI for `gitea.kubectl.ro`
3. Traefik presents the certificate for `gitea.kubectl.ro`
4. HTTP request includes `Host: gitea.kubectl.ro` header
5. Traefik routes request to the Gitea backend
6. Response flows back through Traefik

### Common Mistakes

```bash
# WRONG
-H https://gitea.kubectl.ro     # includes protocol
-H gitea.kubectl.ro:443         # includes port
-H http://gitea.kubectl.ro      # wrong protocol

# CORRECT
-H gitea.kubectl.ro             # hostname only
```

### Nagios Service Definition

```cfg
define service{
        use                             local-service
        host_name                       r8vm02prd
        service_description             HTTPS-Gitea
        check_command                   check_http!-H gitea.kubectl.ro -I 192.168.50.5 -S
}
```

## SNI Troubleshooting

When `check_http` fails with SNI errors but `curl` works, try these solutions in order.

### Solution 1: Try Different TLS Versions

```cfg
# TLS 1.2
define service{
        use                             local-service
        host_name                       r8vm02prd
        service_description             HTTPS-Gitea-TLS12
        check_command                   check_http!-H gitea.kubectl.ro -I 192.168.50.5 -S --ssl=1.2
}

# TLS 1.3
define service{
        use                             local-service
        host_name                       r8vm02prd
        service_description             HTTPS-Gitea-TLS13
        check_command                   check_http!-H gitea.kubectl.ro -I 192.168.50.5 -S --ssl=1.3
}
```

### Solution 2: Skip Certificate Verification

```cfg
define service{
        use                             local-service
        host_name                       r8vm02prd
        service_description             HTTPS-Gitea-NoCertCheck
        check_command                   check_http!-H gitea.kubectl.ro -I 192.168.50.5 -S --ssl=1.2
}
```

### Solution 3: Verbose Output for Debugging

```cfg
define service{
        use                             local-service
        host_name                       r8vm02prd
        service_description             HTTPS-Gitea-Verbose
        check_command                   check_http!-H gitea.kubectl.ro -I 192.168.50.5 -S -v
}
```

### Solution 4: Custom curl-Based Check Command

```cfg
define command{
        command_name    check_https_curl
        command_line    /bin/bash -c 'if curl -f -s -I --connect-timeout 10 --resolve $ARG1$:443:$ARG2$ https://$ARG1$ > /dev/null; then echo "OK - HTTPS service responding"; exit 0; else echo "CRITICAL - HTTPS service not responding"; exit 2; fi'
}

define service{
        use                             local-service
        host_name                       r8vm02prd
        service_description             HTTPS-Gitea-Curl
        check_command                   check_https_curl!gitea.kubectl.ro!192.168.50.5
}
```

### Solution 5: curl-Based Check with Performance Data

```cfg
define command{
        command_name    check_https_curl_perf
        command_line    /bin/bash -c 'RESPONSE=$(curl -f -s -o /dev/null -w "HTTP_CODE:%{http_code},TIME:%{time_total}" --connect-timeout 10 --resolve $ARG1$:443:$ARG2$ https://$ARG1$ 2>/dev/null); if [ $? -eq 0 ]; then CODE=$(echo $RESPONSE | cut -d, -f1 | cut -d: -f2); TIME=$(echo $RESPONSE | cut -d, -f2 | cut -d: -f2); echo "OK - HTTPS $CODE response in ${TIME}s | response_time=${TIME}s"; exit 0; else echo "CRITICAL - HTTPS service not responding"; exit 2; fi'
}

define service{
        use                             local-service
        host_name                       r8vm02prd
        service_description             HTTPS-Gitea-Curl-Perf
        check_command                   check_https_curl_perf!gitea.kubectl.ro!192.168.50.5
}
```

### Solution 6: openssl s_client Based Check

```cfg
define command{
        command_name    check_https_openssl
        command_line    /bin/bash -c 'if echo | timeout 10 openssl s_client -connect $ARG2$:443 -servername $ARG1$ -verify_return_error >/dev/null 2>&1; then echo "OK - SSL connection successful"; exit 0; else echo "CRITICAL - SSL connection failed"; exit 2; fi'
}

define service{
        use                             local-service
        host_name                       r8vm02prd
        service_description             HTTPS-Gitea-OpenSSL
        check_command                   check_https_openssl!gitea.kubectl.ro!192.168.50.5
}
```

### Solution 7: Longer Timeout

```cfg
# Longer timeout for slow backends
define service{
        use                             local-service
        host_name                       r8vm02prd
        service_description             HTTPS-Gitea-Timeout
        check_command                   check_http!-H gitea.kubectl.ro -I 192.168.50.5 -S -t 30
}
```

### Solution 8: Netcat Port Check (Fallback)

```cfg
define command{
        command_name    check_https_nc
        command_line    /bin/bash -c 'if timeout 10 nc -z $ARG1$ 443 >/dev/null 2>&1; then echo "OK - Port 443 is open"; exit 0; else echo "CRITICAL - Port 443 not accessible"; exit 2; fi'
}

define service{
        use                             local-service
        host_name                       r8vm02prd
        service_description             HTTPS-Port-Check
        check_command                   check_https_nc!192.168.50.5
}
```

## Manual Testing Commands

```bash
# Test different check_http options
cd /usr/local/nagios/libexec
./check_http -H gitea.kubectl.ro -I 192.168.50.5 -S --ssl=1.2
./check_http -H gitea.kubectl.ro -I 192.168.50.5 -S --ssl=1.3
./check_http -H gitea.kubectl.ro -I 192.168.50.5 -S -v
./check_http -H gitea.kubectl.ro -I 192.168.50.5 -S -t 30

# Test with curl (works when check_http doesn't)
curl -I --resolve gitea.kubectl.ro:443:192.168.50.5 https://gitea.kubectl.ro

# Test with openssl
echo | openssl s_client -connect 192.168.50.5:443 -servername gitea.kubectl.ro
```

Expected success output:

```
HTTP OK: HTTP/1.1 200 OK - 1234 bytes in 0.123 second response time
```

## Recommended Troubleshooting Order

1. Try specific TLS version `--ssl=1.2` or `--ssl=1.3` (Solution 1)
2. If that fails, try skipping cert verification (Solution 2)
3. Use verbose mode (`-v`) to see exactly what's happening (Solution 3)
4. If still failing, use curl-based monitoring (Solution 4 or 5)
5. Solution 5 provides performance data like standard Nagios plugins

## Common Pitfalls

### Wrong Service Definition

```cfg
# WRONG — missing -H flag
check_command   check_http!192.168.100.31

# CORRECT
check_command   check_http!-H 192.168.100.31
```

### Certificate Check Combined with HTTP Check

```cfg
# Check both the page AND certificate expiry
define service{
        use                             local-service
        host_name                       webserver01
        service_description             HTTPS + Cert
        check_command                   check_http!-H www.example.com -S -u /health -s "OK" -C 30 -w 2 -c 5
}
```

### Follow Redirects

```cfg
# Follow HTTP → HTTPS redirects
define service{
        use                             local-service
        host_name                       webserver01
        service_description             HTTP Redirect
        check_command                   check_http!-H www.example.com -f follow -u / -s "Welcome"
}
```

### Custom Headers (API Checks)

```cfg
# Send custom headers (e.g., API key, Accept)
define command{
        command_name    check_http_api
        command_line    $USER1$/check_http -H $HOSTADDRESS$ -S -u $ARG1$ -k "Accept: application/json" -k "Authorization: Bearer $ARG2$" -s $ARG3$
}

define service{
        use                             local-service
        host_name                       api-server
        service_description             API Health
        check_command                   check_http_api!/api/health!your-api-token!"status":"ok"
}
```

## Quick Reference

| Check Type | Command |
|-----------|---------|
| Basic HTTP | `check_http!-H host` |
| Basic HTTPS | `check_http!-H host -S` |
| HTTPS + cert expiry | `check_http!-H host -S -C 30` |
| Custom port | `check_http!-H host -p 8080` |
| URL path | `check_http!-H host -u /health` |
| Content check | `check_http!-H host -s "expected"` |
| Response time | `check_http!-H host -w 2 -c 5` |
| Virtual host (SNI) | `check_http!-H hostname -I ip -S` |
| Basic auth | `check_http!-H host -a user:pass` |
| Follow redirects | `check_http!-H host -f follow` |
