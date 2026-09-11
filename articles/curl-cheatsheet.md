# curl Cheatsheet

Transfer data to or from a server using URLs. `curl` supports HTTP, HTTPS, FTP, proxies, authentication, file transfers, API testing, and detailed connection diagnostics.

## Command Syntax

```bash
curl [options] URL...
```

Multiple URLs can be supplied in one invocation. Some options apply globally, while others apply only to the next transfer; use `--next` to separate transfers that need different options.

## Basic Requests

| Command | What It Does |
|---------|--------------|
| `curl https://example.com` | Fetch a URL and print the body to stdout |
| `curl -o file.html https://example.com` | Save output with a chosen filename |
| `curl -O https://example.com/file.tar.gz` | Save with the remote filename |
| `curl -s https://example.com` | Silent mode, including errors |
| `curl -sS https://example.com` | Silent progress but still show errors |
| `curl -L https://example.com` | Follow redirects |
| `curl -I https://example.com` | Send a HEAD request and show headers |
| `curl -i https://example.com` | Include response headers with the body |
| `curl -f https://example.com` | Fail with exit code 22 on HTTP 400+ and suppress error body |
| `curl --fail-with-body https://example.com` | Fail on HTTP 400+ but retain the response body |

`--fail-with-body` is useful for API diagnostics because scripts receive a nonzero exit code while the server's error details remain available. It requires curl 7.76 or newer.

## Quick Option Reference

| Short | Long form | Purpose |
|-------|-----------|---------|
| `-X` | `--request` | Set the request method token |
| `-H` | `--header` | Add, replace, remove, or empty a header |
| `-A` | `--user-agent` | Set the User-Agent value |
| `-d` | `--data` | Send URL-encoded or textual request data |
| — | `--data-urlencode` | URL-encode a name or value |
| — | `--data-binary` | Send data without newline conversion |
| — | `--json` | Send JSON with JSON request headers |
| `-F` | `--form` | Send multipart form data or files |
| `-T` | `--upload-file` | Upload a file with PUT or another protocol |
| `-o` | `--output` | Write output to a selected file |
| `-O` | `--remote-name` | Use the remote filename |
| `-C -` | `--continue-at -` | Resume a download automatically |
| `-u` | `--user` | Provide or prompt for credentials |
| `-L` | `--location` | Follow redirects |
| `-f` | `--fail` | Return nonzero for HTTP 400+ |
| — | `--fail-with-body` | Fail on HTTP 400+ while preserving the body |
| `-sS` | `--silent --show-error` | Hide progress but retain errors |
| `-i` | `--include` | Include response headers with the body |
| `-I` | `--head` | Request headers only |
| `-w` | `--write-out` | Print selected transfer metadata |
| `-b` | `--cookie` | Send cookies or read a cookie file |
| `-c` | `--cookie-jar` | Save cookies to a file |
| `-x` | `--proxy` | Route the request through a proxy |
| `-k` | `--insecure` | Disable TLS verification; testing only |

## HTTP Methods

| Command | What It Does |
|---------|--------------|
| `curl https://api.example.com/users` | GET request, the default |
| `curl -X POST https://api.example.com/users` | Explicit POST with an empty body |
| `curl -X PUT https://api.example.com/users/1` | Explicit PUT with an empty body |
| `curl -X PATCH https://api.example.com/users/1` | Explicit PATCH with an empty body |
| `curl -X DELETE https://api.example.com/users/1` | Explicit DELETE |

`-X` changes the method token only. It does not create a request body or select an encoding. Options such as `-d`, `--json`, `-F`, and `-T` select body behavior and often imply an appropriate method.

### POST JSON

```bash
curl --json '{"name":"John","age":30}' \
  https://api.example.com/users
```

### PUT JSON

```bash
curl -X PUT \
  -H "Content-Type: application/json" \
  --data '{"name":"John","age":31}' \
  https://api.example.com/users/1
```

### PATCH JSON

```bash
curl -X PATCH \
  -H "Content-Type: application/json" \
  --data '{"age":32}' \
  https://api.example.com/users/1
```

## Sending Data

### URL-Encoded Form Data

```bash
# POST with form fields; -d implies POST
curl -d "user=example" -d "role=admin" \
  https://api.example.com/users

# Encode spaces, ampersands, Unicode, and other reserved characters safely
curl --data-urlencode "user=${USER_NAME}" \
  --data-urlencode "message=${MESSAGE}" \
  https://api.example.com/messages

# Read textual form data from a file
curl -d @form-data.txt https://api.example.com/submit
```

Use `--data-urlencode` when values are not already encoded. Plain `-d "name=value"` assumes reserved characters have been handled correctly.

### Multipart Form Data

```bash
# POST multipart fields
curl -F "name=John" -F "role=admin" \
  https://api.example.com/users

# Upload a file
curl -F "file=@/path/to/photo.jpg" \
  https://api.example.com/upload

# Set the uploaded filename and media type
curl -F "file=@document.pdf;filename=report.pdf;type=application/pdf" \
  https://api.example.com/upload

# Mix fields and files
curl -F "description=Profile image" \
  -F "avatar=@photo.jpg" \
  https://api.example.com/profile
```

### JSON Data

```bash
# Explicit JSON headers and inline body
curl -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"name":"Alice","role":"admin"}' \
  https://api.example.com/users

# Preserve a file body exactly
curl -H "Content-Type: application/json" \
  --data-binary @payload.json \
  https://api.example.com/users

# JSON shorthand: sets Content-Type and Accept headers (curl 7.82+)
curl --json @payload.json https://api.example.com/users
```

`--data` removes carriage returns and newlines when reading a file. Use `--data-binary` for byte-preserving bodies, signed payloads, and formats where line endings matter.

## Headers and User-Agent

```bash
# Set a custom header
curl -H "Accept: application/json" https://api.example.com/users

# Set multiple headers
curl -H "Accept: application/json" \
  -H "X-Request-ID: abc123" \
  https://api.example.com/users

# Set the User-Agent
curl -A "MyClient/1.0" https://api.example.com/users

# Remove curl's generated User-Agent header
curl -H "User-Agent:" https://api.example.com/users

# Force an empty User-Agent header
curl -H "User-Agent;" https://api.example.com/users
```

An empty value after a colon removes an internally generated header. A semicolon uses curl's special syntax to send the header with an empty value.

## Authentication

| Command | What It Does |
|---------|--------------|
| `curl -u user https://api.example.com/private` | Basic auth and prompt for the password |
| `curl -H "Authorization: Bearer ${TOKEN}" https://api.example.com/private` | Bearer-token authentication |
| `curl -H "X-API-Key: ${API_KEY}" https://api.example.com/data` | API key in a service-specific header |
| `curl --negotiate -u : https://service.example.com/` | Kerberos/SPNEGO authentication |
| `curl -n https://api.example.com/private` | Read matching credentials from `~/.netrc` |

API-key header names vary by service. Avoid query-string API keys because URLs are commonly retained in access logs, browser history, monitoring systems, and proxy logs.

Do not put real passwords or reusable tokens directly in scripts or copied command lines. Command arguments can remain in shell history and may be visible to local process inspection. Prefer prompts, protected credential files, environment injection from a secret manager, or service-specific short-lived credentials.

Protect a `.netrc` file:

```bash
chmod 600 ~/.netrc
```

## Verbose Output and Tracing

| Command | What It Does |
|---------|--------------|
| `curl -v https://api.example.com` | Show connection details and request/response headers |
| `curl --trace trace.bin https://api.example.com` | Write a full binary/hex trace |
| `curl --trace-ascii trace.txt https://api.example.com` | Write a readable trace |
| `curl -w "%{http_code}\n" -o /dev/null -sS https://example.com` | Print only the HTTP status code |
| `curl -w "%{time_total}\n" -o /dev/null -sS https://example.com` | Print total request time |

Repeating `-v` does not create additional verbosity levels. Use `--trace` or `--trace-ascii` for deeper diagnostics.

> **Sensitive data:** Verbose output and traces can contain `Authorization`, `Cookie`, proxy-authentication, and request-body data. Store traces with restrictive permissions, sanitize them before sharing, and never commit them to source control.

```bash
umask 077
curl --trace-ascii curl-trace.txt https://api.example.com
```

## SSL/TLS

| Command | What It Does |
|---------|--------------|
| `curl https://example.com` | Use HTTPS with certificate and hostname verification |
| `curl -k https://example.com` | Skip certificate verification; insecure |
| `curl --cacert ca-bundle.crt https://example.com` | Trust a specific CA bundle |
| `curl --cert client.pem --key client-key.pem https://api.example.com` | Present a client certificate and key |
| `curl --cert-type P12 --cert client.p12:password https://api.example.com` | Present a PKCS#12 client certificate |
| `curl --tlsv1.2 https://example.com` | Require TLS 1.2 or newer |

Do not use `-k` in production scripts. Install the correct CA or provide it with `--cacert` instead.

## Download and Upload

```bash
# Download with the remote filename
curl -O https://example.com/release-v1.0.tar.gz

# Download with a custom filename
curl -o release.tar.gz https://example.com/release-v1.0.tar.gz

# Follow redirects while preserving the remote filename
curl -LO https://example.com/latest-release.tar.gz

# Resume a partial download
curl -C - -O https://example.com/largefile.iso

# Download multiple files
curl -O https://example.com/file1.txt \
  -O https://example.com/file2.txt

# Upload a file using HTTP PUT
curl -T localfile.txt https://example.com/upload/

# Upload through FTP; prompt for the password
curl -T localfile.txt -u user ftp://ftp.example.com/remote/

# Limit transfer bandwidth to 1 MiB per second
curl --limit-rate 1M -O https://example.com/largefile.iso
```

`--limit-rate` limits bytes transferred per second. It does not limit API requests per second.

## Timeouts and Retries

| Command | What It Does |
|---------|--------------|
| `curl --connect-timeout 5 https://example.com` | Limit the connection phase to five seconds |
| `curl --max-time 30 https://example.com` | Limit the entire operation to 30 seconds |
| `curl --retry 3 https://example.com` | Retry selected transient failures |
| `curl --retry 3 --retry-delay 2 https://example.com` | Wait two seconds between retries |
| `curl --retry 3 --retry-max-time 60 https://example.com` | Limit the total retry window |
| `curl --retry 3 --retry-all-errors https://example.com` | Retry all transfer errors |

Use `--retry-all-errors` only when replay is safe. Retrying a POST, PATCH, payment, or other state-changing operation after an ambiguous failure can duplicate side effects. Use idempotency keys when the API supports them.

## Cookies

```bash
# Send a cookie
curl -b "session=abc123" https://example.com/dashboard

# Save cookies to a jar
curl -c cookies.txt https://example.com/login

# Load cookies from a jar
curl -b cookies.txt https://example.com/dashboard

# Read and update the same cookie jar
curl -b cookies.txt -c cookies.txt https://example.com/dashboard
```

Cookie jars can contain authenticated session tokens. Protect and exclude them from source control:

```bash
chmod 600 cookies.txt
```

## Proxies

```bash
# Use an HTTP proxy
curl -x http://proxy.example.com:8080 https://example.com

# HTTP proxy authentication; prompt for password
curl -x http://proxy.example.com:8080 \
  --proxy-user proxyuser \
  https://example.com

# SOCKS5 with local DNS resolution
curl --socks5 127.0.0.1:1080 https://example.com

# SOCKS5 with proxy-side DNS resolution
curl --socks5-hostname 127.0.0.1:1080 https://example.com

# Bypass the proxy for selected destinations
curl --noproxy "localhost,127.0.0.1,.internal.example.com" \
  -x http://proxy.example.com:8080 \
  https://service.internal.example.com
```

`--socks5` resolves the destination hostname locally. Use `--socks5-hostname` or a `socks5h://` proxy URL when DNS should also traverse the proxy.

## Cache and Proxy Diagnostics

curl does not maintain an HTTP response cache itself, but gateways, reverse proxies, CDNs, and corporate proxies may cache or modify responses. Request cache directives can help isolate intermediary behavior:

| Request header | Meaning |
|----------------|---------|
| `Cache-Control: no-cache` | Require a cache to revalidate a stored response before reusing it |
| `Cache-Control: max-age=0` | Accept only a response with no current age, normally causing revalidation |
| `Cache-Control: no-store` | Ask intermediaries not to store this request or response; does not purge existing entries |
| `Pragma: no-cache` | Legacy request directive for HTTP/1.0-compatible caches |

`no-cache` does not guarantee a direct origin fetch or a `200 OK`. A cache may revalidate an object successfully and then serve the validated representation. Always inspect the returned body and headers instead of relying only on the status code.

### Compare normal and revalidated responses

Use GET requests so injected, truncated, or replaced response bodies can be detected. This example uses an Ubuntu `InRelease` metadata file:

```bash
url='https://archive.ubuntu.com/ubuntu/dists/noble/InRelease'
workdir=$(mktemp -d)
chmod 700 "$workdir"

# Normal request
curl --fail --show-error --location \
  --dump-header "$workdir/normal.headers" \
  --output "$workdir/normal.body" \
  "$url"

# Ask intermediaries to revalidate cached content
curl --fail --show-error --location \
  -H 'Cache-Control: no-cache' \
  -H 'Pragma: no-cache' \
  --dump-header "$workdir/nocache.headers" \
  --output "$workdir/nocache.body" \
  "$url"
```

Compare status lines, cache metadata, body sizes, hashes, and content:

```bash
grep -iE 'HTTP/|age:|cache-control:|etag:|last-modified:|via:|x-cache:|content-length:' \
  "$workdir"/*.headers

wc -c "$workdir"/*.body
sha256sum "$workdir"/*.body
cmp -s "$workdir/normal.body" "$workdir/nocache.body"
printf 'cmp exit code: %s\n' "$?"
head -n 5 "$workdir"/*.body
```

For an `InRelease` file, the body should begin with:

```text
-----BEGIN PGP SIGNED MESSAGE-----
```

Different sizes or hashes justify investigation but do not alone prove injection; the origin object may have changed between requests. HTML login pages, proxy banners, impossible framing, or data appended after signed content are stronger evidence.

### Compare through an explicit proxy

```bash
proxy='http://proxy.example.com:8080'

curl --fail --show-error --location \
  --proxy "$proxy" \
  -H 'Cache-Control: no-cache' \
  --dump-header "$workdir/proxy.headers" \
  --output "$workdir/proxy.body" \
  "$url"

head -n 30 "$workdir/proxy.headers"
file "$workdir/proxy.body"
head -n 5 "$workdir/proxy.body"
```

Do not embed proxy passwords in the URL. Use `--proxy-user proxyuser` to prompt, or an approved protected credential mechanism.

### Compare HTTP protocol behavior

Protocol selection is useful for diagnosing a broken intermediary, not as a guaranteed cache bypass:

```bash
curl --http1.1 -H 'Cache-Control: no-cache' -o /dev/null -sS \
  -w 'HTTP/1.1: %{http_code} %{size_download} bytes\n' "$url"

curl --http2 -H 'Cache-Control: no-cache' -o /dev/null -sS \
  -w 'HTTP/2: %{http_code} %{size_download} bytes\n' "$url"
```

Use `--http1.0` only as a diagnostic for a known legacy proxy. It does not inherently disable caching.

### Important cache-testing caveats

- A `304 Not Modified` is meaningful only for a conditional request containing a validator such as `If-None-Match` or `If-Modified-Since`. curl does not add those headers by default.
- `-H 'If-None-Match:'` removes that header if another source added it; it has no effect when the header was absent already.
- `--no-keepalive` disables TCP keepalive probes. It does **not** disable HTTP connection reuse. `Connection: close` requests closure for HTTP/1.1, but separate curl processes already use separate connection pools.
- A HEAD request (`-I`) cannot reveal content appended to a GET response body. Capture a GET body when diagnosing content injection.
- Query-string timestamps or random values create a different cache key, but they also change the requested URL. Some repositories and signed-resource endpoints reject or handle those URLs differently.
- `must-revalidate` and `Expires` primarily describe response-cache behavior and are not reliable request-side bypass controls.
- A fake User-Agent such as `APT/1.0` does not reproduce APT accurately. Capture the real APT request with APT acquire debugging when matching its behavior matters.
- Verbose output, proxy logs, and captured bodies may expose credentials or internal infrastructure. Protect and sanitize diagnostic artifacts.

Remove the temporary data when it is no longer needed:

```bash
rm -rf "$workdir"
```

## DNS and Name Resolution

```bash
# Inspect DNS outside curl
host www.example.com

# Connect to a chosen address while preserving hostname and TLS SNI
curl --resolve example.com:443:192.0.2.10 https://example.com

# Use DNS over HTTPS (curl build must support it)
curl --doh-url https://dns.google/dns-query https://example.com

# Force IPv4 or IPv6
curl -4 https://example.com
curl -6 https://example.com
```

On RHEL, `host` is provided by `bind-utils`; on Ubuntu and Debian, it is provided by `dnsutils`.

## Redirects and HTTP Failures

```bash
# Follow redirects, stopping after five
curl -L --max-redirs 5 https://example.com

# Return nonzero on HTTP errors and retain the response body
curl -sS --fail-with-body https://api.example.com/resource

# Capture the body, status, and curl exit code separately
body_file=$(mktemp)
status=$(curl -sS -o "$body_file" -w '%{http_code}' \
  https://api.example.com/resource)
curl_exit=$?
printf 'curl_exit=%s http_status=%s\n' "$curl_exit" "$status"
cat "$body_file"
rm -f "$body_file"
```

When following `301`, `302`, or `303`, curl may change a POST to GET according to HTTP redirect behavior. Use `--post301`, `--post302`, or `--post303` only when the API explicitly requires preserving POST. Credentials are not forwarded to a different hostname by default; avoid `--location-trusted` unless that trust is intentional.

## Output Formatting

### Inline Write-Out Values

```bash
# HTTP status code only
curl -sS -o /dev/null -w '%{http_code}\n' https://example.com

# Timing breakdown
curl -sS -o /dev/null \
  -w 'DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n' \
  https://example.com

# Response size
curl -sS -o /dev/null \
  -w 'Downloaded: %{size_download} bytes\n' \
  https://example.com

# All available values as JSON (curl 7.70+)
curl -sS -o /dev/null -w '%{json}\n' https://example.com | jq .
```

### File-Based Write-Out Template

Create `curl-format.txt`:

```text
DNS:        %{time_namelookup}s\n
Connect:    %{time_connect}s\n
TLS:        %{time_appconnect}s\n
Pretransfer:%{time_pretransfer}s\n
Redirect:   %{time_redirect}s\n
TTFB:       %{time_starttransfer}s\n
Total:      %{time_total}s\n
Status:     %{http_code}\n
```

Use the template:

```bash
curl -sS -o /dev/null -w @curl-format.txt https://example.com
```

### Useful Write-Out Variables

| Variable | What It Shows |
|----------|---------------|
| `%{http_code}` | Final HTTP status code |
| `%{exitcode}` | curl process result represented in write-out output |
| `%{time_total}` | Total transfer time |
| `%{time_namelookup}` | DNS resolution time |
| `%{time_connect}` | TCP connection time |
| `%{time_appconnect}` | TLS handshake completion time |
| `%{time_starttransfer}` | Time to first byte |
| `%{size_download}` | Downloaded bytes |
| `%{size_upload}` | Uploaded bytes |
| `%{url_effective}` | Final URL after redirects |
| `%{redirect_url}` | Redirect target when not followed |
| `%{num_redirects}` | Number of redirects followed |
| `%{content_type}` | Response Content-Type |
| `%{remote_ip}` | Connected server address |

## Recipes

### Test an API Endpoint

```bash
# Health check that fails on HTTP errors
curl -fsS https://api.example.com/health >/dev/null && \
  echo "OK" || echo "FAILED"

# POST JSON and format the response
curl -sS --fail-with-body \
  --json '{"query":"test"}' \
  https://api.example.com/search | jq .

# Show response headers and body together
curl -i https://api.example.com/users
```

### Download with Progress

```bash
# Use curl's simple progress-bar display while saving a file
curl -# -O https://example.com/largefile.iso

# Progress remains on stderr while the body is piped
curl -# https://example.com/data.json | jq .
```

### Test Multiple URLs

```bash
# Shell brace expansion creates multiple URL arguments
curl -sS https://api.example.com/{users,posts,comments}

# Quoted curl URL globbing with numbered output files
curl -sS "https://example.com/page[1-5].html" \
  -o "page_#1.html"
```

### Limit Request Start Rate

For curl 7.84 or newer, `--rate` limits how frequently transfers begin in parallel mode:

```bash
curl --parallel --rate 5/s \
  "https://api.example.com/items/[1-20]" \
  -o "item-#1.json"
```

This is different from `--limit-rate`, which limits transfer bandwidth rather than request frequency. Client-side rate limiting does not override an API's server-side quotas.

### Bind to a Local Interface or Address

```bash
curl --interface eth1 https://example.com
curl --interface 192.168.1.100 https://example.com
```

## Common Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Transfer completed successfully |
| `3` | URL malformed |
| `5` | Could not resolve proxy |
| `6` | Could not resolve host |
| `7` | Failed to connect |
| `18` | Partial file or transfer |
| `22` | HTTP error when `--fail` or `--fail-with-body` is active |
| `23` | Failed to write received data locally |
| `26` | Failed to read local upload data |
| `28` | Operation timed out |
| `35` | TLS/SSL connection error |
| `47` | Too many redirects |
| `52` | Server returned an empty reply |
| `56` | Failure while receiving network data |
| `60` | Peer certificate could not be authenticated |

HTTP status and curl exit status are separate values. Without `--fail` or `--fail-with-body`, a completed HTTP `404` or `500` transfer can still produce curl exit code `0`.

Check the process exit status in a shell:

```bash
curl -fsS https://api.example.com/health
curl_exit=$?
printf 'curl exit code: %s\n' "$curl_exit"
```

Consult `curl --manual` or `man curl` for the complete, version-specific exit-code list.

## Gotchas

- **Quote URLs containing shell characters** — Quote URLs containing `&`, `?`, `[]`, or `{}` when they should reach curl unchanged.
- **`-d` implies POST** — Explicit `-X POST` is normally unnecessary with `-d`, `--data-binary`, or `--json`.
- **`-F` implies multipart** — Do not combine `-d` and `-F`; they select different body formats.
- **`-X` does not create a body** — Specify the correct data option separately.
- **Redirects are not followed by default** — Add `-L` when redirects are expected and review method changes across them.
- **Certificate verification matters** — Replace `-k` with the correct CA trust configuration.
- **Binary output should not go to the terminal** — Save it with `-o` or pipe it to a suitable consumer.
- **Retries can duplicate side effects** — Retry non-idempotent requests only with application-level protection.
- **Traces can expose secrets** — Sanitize verbose output, traces, cookie jars, and copied commands.
- **Status and exit code differ** — Use `--fail` or `--fail-with-body` when HTTP errors must fail a script.

## See Also

- [Troubleshoot APT NOSPLIT and Excess Data Errors Behind a Proxy](articles/apt-nosplit-proxy-troubleshooting.md) — diagnose HTTP proxy and content-integrity problems
- [RHEL LAMP Stack Setup](articles/rhel-lamp-stack-setup.md) — set up a web server for curl testing
- [Bash Essentials Guide](articles/bash-essentials-guide.md) — shell scripting, variables, quoting, and exit handling
