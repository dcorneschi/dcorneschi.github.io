# curl Cheatsheet

Transfer data to or from a server using URLs. Supports HTTP, HTTPS, FTP, and many other protocols. Essential for testing APIs, downloading files, and debugging web services.

## Basic Requests

| Command | What It Does |
|---------|--------------|
| `curl http://example.com` | Fetch a URL and print to stdout |
| `curl -o file.html http://example.com` | Save output to a file |
| `curl -O http://example.com/file.tar.gz` | Save with the remote filename |
| `curl -s http://example.com` | Silent mode (no progress bar) |
| `curl -S http://example.com` | Show errors even in silent mode |
| `curl -sS http://example.com` | Silent but show errors |
| `curl -L http://example.com` | Follow redirects (3xx responses) |
| `curl -I http://example.com` | Fetch headers only (HEAD request) |

## HTTP Methods

| Command | What It Does |
|---------|--------------|
| `curl http://localhost/api` | GET request (default) |
| `curl -X POST http://localhost/api` | Explicit POST (empty body) |
| `curl -X PUT http://localhost/api` | PUT request |
| `curl -X PATCH http://localhost/api` | PATCH request |
| `curl -X DELETE http://localhost/api` | DELETE request |

## Sending Data

### Form Data (application/x-www-form-urlencoded)

```bash
# POST with form fields (implies -X POST)
curl -d "user=admin&pass=secret" http://localhost/login

# Same thing with separate -d flags
curl -d "user=admin" -d "pass=secret" http://localhost/login

# Read data from a file
curl -d @data.txt http://localhost/api
```

### Multipart Form Data (multipart/form-data)

```bash
# POST with multipart form fields
curl -F "arg1=foo" -F "arg2=bar" http://localhost/api

# Upload a file
curl -F "file=@/path/to/photo.jpg" http://localhost/upload

# Upload with a custom filename
curl -F "file=@photo.jpg;filename=avatar.png" http://localhost/upload

# Mix files and fields
curl -F "name=John" -F "avatar=@photo.jpg" http://localhost/profile
```

### JSON Data

```bash
# POST JSON payload
curl -H "Content-Type: application/json" \
     -d '{"name":"Alice","role":"admin"}' \
     http://localhost/api/users

# Read JSON from a file
curl -H "Content-Type: application/json" \
     -d @payload.json \
     http://localhost/api/users

# Shorthand with --json (curl 7.82+)
curl --json '{"name":"Alice"}' http://localhost/api/users
```

## Verbose and Debugging

| Command | What It Does |
|---------|--------------|
| `curl -v http://localhost/api` | Show request/response headers and connection details |
| `curl -vvv http://localhost/api` | Even more verbose output |
| `curl --trace - http://localhost/api` | Full hex dump of all data |
| `curl --trace-ascii - http://localhost/api` | Full ASCII trace |
| `curl -w "%{http_code}\n" -o /dev/null -s http://localhost` | Print only the HTTP status code |
| `curl -w "%{time_total}\n" -o /dev/null -s http://localhost` | Print total request time |

### Verbose Output Example

```bash
# See sent and received HTTP GET status and API response
curl http://localhost:80/someapp/api -v

# See sent and received HTTPS POST status and response
curl https://localhost:443/someapp/api -v -F "arg1=foo" -F "arg2=bar"
```

## Headers

```bash
# Set a custom header
curl -H "Authorization: Bearer TOKEN123" http://localhost/api

# Multiple headers
curl -H "Accept: application/json" \
     -H "X-Request-ID: abc123" \
     http://localhost/api

# Remove a default header (send empty)
curl -H "User-Agent:" http://localhost/api
```

## Authentication

| Command | What It Does |
|---------|--------------|
| `curl -u user:pass http://localhost/api` | Basic authentication |
| `curl -u user http://localhost/api` | Basic auth (prompts for password) |
| `curl -H "Authorization: Bearer TOKEN" http://localhost/api` | Bearer token auth |
| `curl --negotiate -u : http://localhost/api` | Kerberos/SPNEGO auth |
| `curl -n http://localhost/api` | Use credentials from `~/.netrc` |

## SSL/TLS

| Command | What It Does |
|---------|--------------|
| `curl https://example.com` | HTTPS with certificate verification |
| `curl -k https://example.com` | Skip certificate verification (insecure) |
| `curl --cacert ca.pem https://example.com` | Use a specific CA certificate |
| `curl --cert client.pem --key client-key.pem https://example.com` | Client certificate auth |
| `curl --tlsv1.2 https://example.com` | Force minimum TLS 1.2 |

## Download and Upload

```bash
# Download a file
curl -O https://example.com/release-v1.0.tar.gz

# Download with a custom name
curl -o myfile.tar.gz https://example.com/release-v1.0.tar.gz

# Resume a broken download
curl -C - -O https://example.com/largefile.iso

# Download multiple files
curl -O https://example.com/file1.txt -O https://example.com/file2.txt

# Upload a file via PUT
curl -T localfile.txt https://example.com/upload/

# Upload via FTP
curl -T localfile.txt ftp://ftp.example.com/ -u user:pass

# Limit download speed
curl --limit-rate 1M -O https://example.com/largefile.iso
```

## Timeouts and Retries

| Command | What It Does |
|---------|--------------|
| `curl --connect-timeout 5 http://example.com` | Timeout for connection phase (seconds) |
| `curl -m 30 http://example.com` | Max time for entire operation (seconds) |
| `curl --retry 3 http://example.com` | Retry on transient failures |
| `curl --retry 3 --retry-delay 2 http://example.com` | Retry with 2s delay between attempts |
| `curl --retry 3 --retry-all-errors http://example.com` | Retry on all errors (not just transient) |

## Cookies

```bash
# Send a cookie
curl -b "session=abc123" http://localhost/api

# Save cookies to a file
curl -c cookies.txt http://localhost/login -d "user=admin&pass=secret"

# Load cookies from a file
curl -b cookies.txt http://localhost/dashboard

# Save and send cookies (cookie jar)
curl -b cookies.txt -c cookies.txt http://localhost/api
```

## Proxies

```bash
# Use an HTTP proxy
curl -x http://proxy:8080 http://example.com

# Use a SOCKS5 proxy
curl --socks5 localhost:1080 http://example.com

# Proxy with authentication
curl -x http://user:pass@proxy:8080 http://example.com

# Bypass proxy for specific hosts
curl --noproxy "localhost,127.0.0.1" -x http://proxy:8080 http://localhost/api
```

## DNS and Name Resolution

```bash
# Test DNS name resolution (use 'host' command)
host www.someapp.org

# Override DNS resolution in curl
curl --resolve example.com:443:127.0.0.1 https://example.com

# Use a specific DNS server (curl 7.62+)
curl --doh-url https://dns.google/dns-query http://example.com
```

> **Note:** The `host` command requires the `bind-utils` package (`yum -y install bind-utils`).

## Output Formatting

### Write-Out Variables

```bash
# HTTP response code only
curl -w "%{http_code}" -o /dev/null -s http://localhost/api

# Timing breakdown
curl -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nTotal: %{time_total}s\n" \
     -o /dev/null -s https://example.com

# Response size
curl -w "Downloaded: %{size_download} bytes\n" -o /dev/null -s http://example.com

# All variables in JSON (curl 7.70+)
curl -w "%{json}" -o /dev/null -s http://example.com | jq .
```

### Useful Write-Out Variables

| Variable | What It Shows |
|----------|---------------|
| `%{http_code}` | HTTP status code |
| `%{time_total}` | Total time in seconds |
| `%{time_namelookup}` | DNS resolution time |
| `%{time_connect}` | TCP connection time |
| `%{time_appconnect}` | TLS handshake time |
| `%{time_starttransfer}` | Time to first byte (TTFB) |
| `%{size_download}` | Bytes downloaded |
| `%{size_upload}` | Bytes uploaded |
| `%{url_effective}` | Final URL (after redirects) |
| `%{redirect_url}` | Redirect target URL |
| `%{num_redirects}` | Number of redirects followed |
| `%{content_type}` | Content-Type header value |
| `%{remote_ip}` | Server IP address |

## Recipes

### Test an API endpoint

```bash
# Quick health check
curl -sf http://localhost:8080/health && echo "OK" || echo "FAIL"

# POST to an API and pretty-print JSON response
curl -sS -H "Content-Type: application/json" \
     -d '{"query":"test"}' \
     http://localhost/api/search | jq .

# Check response headers and body together
curl -i http://localhost/api/users
```

### Download with progress

```bash
# Show progress bar (default when output is not a terminal)
curl -# -O https://example.com/largefile.iso

# Progress bar to stderr while piping body
curl -# http://example.com/data.json | jq .
```

### Test multiple URLs

```bash
# Sequential requests
curl -s http://localhost/api/{users,posts,comments} | jq .

# With URL globbing
curl -s "http://localhost/page[1-5].html" -o "page_#1.html"
```

### Simulate slow connections

```bash
# Limit upload and download speed
curl --limit-rate 100K http://example.com/largefile
```

### Send request from a specific interface/IP

```bash
# Bind to a specific local address
curl --interface eth1 http://example.com
curl --interface 192.168.1.100 http://example.com
```

## Gotchas

- **Quotes around URLs with special characters** — URLs containing `&`, `?`, or `{}` must be quoted in the shell: `curl "http://api.com/search?q=hello&page=2"`
- **`-d` implies POST** — You don't need `-X POST` when using `-d`
- **`-F` implies multipart** — Don't combine `-d` and `-F`; they use different encodings
- **No redirect following by default** — Always add `-L` if you expect 3xx redirects
- **Certificate verification** — Don't use `-k` in production scripts; provide the CA cert with `--cacert` instead
- **Binary output to terminal** — curl will warn about binary output; pipe to a file or use `-o`
- **Exit codes** — curl returns 0 on success even if the HTTP status is 4xx/5xx. Use `-f` to make curl return non-zero on HTTP errors

## See Also

- [RHEL LAMP Stack Setup](articles/rhel-lamp-stack-setup.md) — setting up a web server to test with curl
- [Bash Essentials Guide](articles/bash-essentials-guide.md) — shell scripting fundamentals
