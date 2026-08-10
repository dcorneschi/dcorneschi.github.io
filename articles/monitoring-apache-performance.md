<img src="/articles/images/datadog-logo.svg" alt="Datadog" width="150"/>

# Monitoring Apache Web Server Performance

Key metrics and concepts for monitoring Apache HTTP Server health and performance. Covers throughput, latency, resource utilization, MPM internals, and error tracking.

## Metric Categories

When monitoring Apache, focus on four areas:

| Category | What it tells you |
|----------|-------------------|
| Throughput & latency | How fast requests are being served |
| Resource utilization | Whether workers/threads are over- or underutilized |
| Host-level resources | Memory, CPU, file descriptors on the server |
| Errors | Client (4xx) and server (5xx) error rates |

## Throughput & Latency Metrics

| Metric | Description | Source |
|--------|-------------|--------|
| Request processing time | Microseconds to process a client request | Access log |
| Requests per second | Average rate of client requests | mod_status |
| Bytes served | Total bytes sent to clients | mod_status |

### Request processing time

Slow request processing may be caused by:
- Long-running database queries (MySQL, PostgreSQL)
- Slow application code (PHP, Python)
- Apache itself (worker thread blocking, keep-alive timeouts)

Tuning tips:
- Reduce `KeepAliveTimeout` to free up worker threads faster
- Switch to the **event MPM** for more efficient keep-alive handling
- Use NGINX as a reverse proxy for static content
- Enable Apache's caching modules or place Varnish in front
- Verify `HostnameLookups Off` in config (DNS lookups on every request are expensive)
- Use IP addresses instead of hostnames in access control directives

### Requests per second

The `requests/sec` value from mod_status is an **average since server start** — not a real-time rate. For meaningful alerting, calculate the rate of change over a short window (e.g., last 60 seconds) using a monitoring tool.

Alert on:
- **Sudden spike** — may indicate a DDoS attack or traffic burst; verify capacity
- **Sudden drop** — may indicate upstream failures (servers swapping to disk, database crashes)

## Apache Multi-Processing Modules (MPMs)

Apache can only run one MPM at a time. The MPM determines how the server handles connections.

### Prefork MPM

```
Parent Process
├── Child Process 1 (handles 1 request at a time)
├── Child Process 2 (handles 1 request at a time)
├── Child Process 3 (handles 1 request at a time)
└── ...
```

- One process per connection
- Most memory-hungry option
- Only needed for non-thread-safe libraries (e.g., mod_php)
- Uses accept mutex to assign incoming connections to idle processes

### Worker MPM

```
Parent Process
├── Child Process 1
│   ├── Listener Thread
│   ├── Worker Thread 1
│   ├── Worker Thread 2
│   └── ...
├── Child Process 2
│   ├── Listener Thread
│   ├── Worker Thread 1
│   └── ...
└── ...
```

- Multiple threads per child process
- More memory-efficient than prefork (one thread per connection, not one process)
- Listener thread per child accepts connections, passes to idle workers

### Event MPM (recommended for modern systems)

```
Parent Process
├── Child Process 1
│   ├── Listener Thread (manages keep-alive sockets via epoll/kqueue)
│   ├── Worker Thread 1
│   ├── Worker Thread 2
│   └── ...
└── ...
```

- Default MPM on modern Unix-like systems (Apache 2.4+)
- Key advantage: worker threads are **not blocked** by keep-alive connections
- After serving a request, the worker passes the socket back to the listener thread
- Listener thread monitors sockets using kernel I/O methods (epoll on Linux, kqueue on BSD)
- Workers are free to handle new requests while connections idle

## Key MPM Configuration Directives

| Directive | Description |
|-----------|-------------|
| `StartServers` | Number of child processes created at startup |
| `MaxRequestWorkers` | Max simultaneous connections (was `MaxClients` before 2.4) |
| `MinSpareServers` / `MinSpareThreads` | Minimum idle processes/threads to maintain |
| `MaxSpareServers` / `MaxSpareThreads` | Maximum idle processes/threads before killing extras |
| `ThreadsPerChild` | Threads per child process (worker/event MPMs) |
| `MaxConnectionsPerChild` | Connections per child before restart (guards against memory leaks) |
| `ListenBacklog` | TCP queue size for connections waiting when all workers are busy (default 511) |

## Resource Utilization Metrics

| Metric | Description | Source |
|--------|-------------|--------|
| Busy workers | Worker threads/processes currently handling requests | mod_status |
| Idle workers | Worker threads/processes waiting for requests | mod_status |
| Async connections: writing | Connections in async write state (event MPM only) | mod_status |
| Async connections: closing | Connections in async close state (event MPM only) | mod_status |

### Interpreting worker counts

**Too many idle workers:**
- Lower `MinSpareServers`/`MinSpareThreads`
- You're wasting memory on processes/threads that aren't needed

**Too few idle workers (consistently near zero):**
- Server constantly spawning new threads/processes to handle requests
- Increases latency
- If busy + idle approaches `MaxRequestWorkers`, new requests queue up

**Approaching MaxRequestWorkers:**
- Additional requests go into the TCP `ListenBacklog` queue
- Increase `MaxRequestWorkers` cautiously (each worker needs memory)
- Consider scaling horizontally instead

**Many keep-alive connections (K state in scoreboard):**
- Clients holding connections open without sending requests
- Switch to event MPM if not already using it
- Lower `KeepAliveTimeout`

## Host-Level Metrics

### Memory

Memory is the most critical host resource for Apache. When the server runs out of RAM, it swaps to disk and performance degrades severely.

**Sizing MaxRequestWorkers based on memory:**

```
MaxRequestWorkers = (Total_RAM - Other_Memory_Needed) / Memory_Per_Apache_Process
```

Example: 4 GB RAM, each Apache process uses ~50 MB, other services need 1 GB:

```
MaxRequestWorkers = (4000 - 1000) / 50 = 60
```

Use `htop` to measure actual per-process memory usage. Be conservative — swapping is far worse than queueing a few requests.

**Reducing memory usage:**
- Switch from prefork to worker/event MPM
- Disable unused modules (`httpd -D DUMP_MODULES` or `apache2ctl -M`)
- Lower `StartServers` and `MaxSpareThreads`
- Set `MaxConnectionsPerChild` to a reasonable value (e.g., 1000) to guard against memory leaks

### CPU

Rising CPU usage indicates the server is struggling to keep up with request volume.

Actions:
- Move database/application servers to separate hosts
- Scale horizontally (add more Apache instances behind a load balancer)
- Reduce `MaxRequestWorkers` if context switching is excessive

### Open file descriptors

Apache opens a file descriptor for each connection and each log file. With many virtual hosts (each generating separate logs), you can hit system limits.

Solutions:
- Raise the system file descriptor limit (`ulimit -n`)
- Write all virtual host logs to a single file, then split downstream with `split-logfile`

## Error Metrics

| Metric | Description | Source |
|--------|-------------|--------|
| Client error rate (4xx) | 403 Forbidden, 404 Not Found, etc. | Access log |
| Server error rate (5xx) | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable | Access log |

### Server errors (5xx)

Calculate: `5xx_count / total_requests` over a time window.

**503 Service Unavailable** typically means Apache is overloaded:
- Not enough workers to handle the request load
- A config directive is limiting connections to a resource

**Troubleshooting steps:**
1. Check the error log for details (`/var/log/apache2/error.log` or `/var/log/httpd/error_log`)
2. Check if workers are saturated (`mod_status` scoreboard)
3. Verify `MaxRequestWorkers` isn't set too low
4. Look for application-level errors causing 500s

### Client errors (4xx)

Not always actionable, but useful for detecting:
- Vulnerability scanning attempts (repeated 403s from one IP)
- Broken links on your site (404s for removed pages — set up redirects)
- Authentication issues (401s)

## mod_status Quick Setup

Enable the status module to expose live metrics:

```apache
# Load the module (if not already loaded)
LoadModule status_module modules/mod_status.so

# Enable extended status for additional metrics
ExtendedStatus On

<Location "/server-status">
    SetHandler server-status
    Require ip 127.0.0.1
    Require ip 10.0.0.0/8
</Location>
```

Access at `http://localhost/server-status?auto` for machine-readable output.

**Extended status adds:**
- Requests per second (average since start)
- Bytes per second / bytes per request
- CPU load
- Worker scoreboard (visual map of thread states)

### Scoreboard symbols

| Symbol | State |
|--------|-------|
| `_` | Waiting for connection (idle) |
| `S` | Starting up |
| `R` | Reading request |
| `W` | Sending reply (writing) |
| `K` | Keep-alive (reading) |
| `D` | DNS lookup |
| `C` | Closing connection |
| `L` | Logging |
| `G` | Gracefully finishing |
| `I` | Idle cleanup |
| `.` | Open slot (no process) |

## Summary: What to Alert On

| Metric | Alert condition | Likely cause |
|--------|----------------|--------------|
| Request processing time | Sustained increase above baseline | Slow backend, resource exhaustion, or config issue |
| Requests per second | Sudden spike or drop | DDoS/traffic burst, or upstream failure |
| Busy workers | Approaching `MaxRequestWorkers` | Insufficient capacity, need to scale |
| Memory usage | Approaching system RAM | Too many workers, memory leak, need to tune `MaxRequestWorkers` |
| CPU utilization | Sustained high usage | Too many connections, move services to separate hosts |
| Server error rate (5xx) | Increasing over time | Overloaded server, misconfiguration, application bugs |

## Datadog Agent Integration for Apache

### Prerequisites

1. Apache mod_status enabled (see the mod_status section above)
2. Datadog Agent installed on the Apache host
3. The `auto` query parameter accessible: `http://localhost/server-status?auto`

### Configure the integration

```yaml
# /etc/datadog-agent/conf.d/apache.d/conf.yaml

init_config:

instances:
  - apache_status_url: http://localhost/server-status?auto
    tags:
      - env:production
      - service:web
      - team:platform

    # Optional: if mod_status requires auth
    # username: datadog
    # password: <PASSWORD>

    # Optional: TLS settings
    # tls_verify: true
    # tls_cert: /path/to/client.pem
    # tls_private_key: /path/to/client.key
    # tls_ca_cert: /path/to/ca.pem
```

### Verify the check

```bash
sudo datadog-agent check apache
```

Expected output shows metrics like:
- `apache.net.hits` — total requests
- `apache.net.bytes` — total bytes served
- `apache.performance.busy_workers` — currently active workers
- `apache.performance.idle_workers` — idle workers
- `apache.performance.uptime` — server uptime in seconds
- `apache.performance.cpu_load` — CPU load (requires ExtendedStatus)

### Metrics collected by the integration

| Metric | Type | Description |
|--------|------|-------------|
| `apache.net.hits` | rate | Requests per second |
| `apache.net.bytes` | rate | Bytes served per second |
| `apache.performance.busy_workers` | gauge | Number of busy workers |
| `apache.performance.idle_workers` | gauge | Number of idle workers |
| `apache.performance.cpu_load` | gauge | CPU load percentage |
| `apache.performance.uptime` | gauge | Seconds since last restart |
| `apache.conns_total` | gauge | Total current connections (event MPM) |
| `apache.conns_async_writing` | gauge | Async connections in writing state |
| `apache.conns_async_keep_alive` | gauge | Async connections in keep-alive state |
| `apache.conns_async_closing` | gauge | Async connections closing |

### Multiple Apache instances

If you run multiple virtual hosts or instances on different ports:

```yaml
# /etc/datadog-agent/conf.d/apache.d/conf.yaml

init_config:

instances:
  - apache_status_url: http://localhost:80/server-status?auto
    tags:
      - instance:main-site
      - port:80

  - apache_status_url: http://localhost:8080/server-status?auto
    tags:
      - instance:api
      - port:8080
```

## Apache Log Collection with Datadog

### Enable log collection in the agent

```yaml
# /etc/datadog-agent/datadog.yaml
logs_enabled: true
```

### Configure Apache log collection

```yaml
# /etc/datadog-agent/conf.d/apache.d/conf.yaml

logs:
  - type: file
    path: /var/log/apache2/access.log
    source: apache
    service: apache
    tags:
      - env:production
      - log_type:access

  - type: file
    path: /var/log/apache2/error.log
    source: apache
    service: apache
    tags:
      - env:production
      - log_type:error
    log_processing_rules:
      - type: multi_line
        name: multiline_error_log
        pattern: \[\w+ \w+ \d+ \d+:\d+:\d+
```

> **Note:** On RHEL/CentOS, logs are typically at `/var/log/httpd/access_log` and `/var/log/httpd/error_log`.

### Grant read permissions

```bash
# The dd-agent user needs read access to log files
sudo usermod -aG adm dd-agent          # Debian/Ubuntu
sudo usermod -aG apache dd-agent       # RHEL/CentOS

sudo systemctl restart datadog-agent
```

### What Datadog parses automatically

With `source: apache`, Datadog's log pipeline automatically extracts:

| Field | Description | Example |
|-------|-------------|---------|
| `network.client.ip` | Client IP address | `192.168.1.100` |
| `http.method` | Request method | `GET`, `POST` |
| `http.url` | Requested URL path | `/api/users` |
| `http.status_code` | Response status code | `200`, `404`, `503` |
| `network.bytes_written` | Response size in bytes | `4523` |
| `http.useragent` | Client user agent string | `Mozilla/5.0...` |
| `http.referer` | Referrer URL | `https://example.com` |
| `duration` | Request processing time (if logged) | `1250` (microseconds) |

### Adding request duration to access logs

By default, Apache doesn't log request duration. Add `%D` (microseconds) or `%T` (seconds) to your LogFormat:

```apache
# In apache2.conf or httpd.conf
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %D" combined_with_time
CustomLog /var/log/apache2/access.log combined_with_time
```

Datadog will automatically parse the `%D` field as request duration when using `source: apache`.

## Common Apache Log Format Patterns

### Standard formats

```apache
# Common Log Format (CLF)
LogFormat "%h %l %u %t \"%r\" %>s %b" common
# Output: 10.0.0.1 - frank [10/Oct/2023:13:55:36 -0700] "GET /index.html HTTP/1.1" 200 2326

# Combined Log Format (most common)
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
# Output: 10.0.0.1 - frank [10/Oct/2023:13:55:36 -0700] "GET /index.html HTTP/1.1" 200 2326 "https://example.com" "Mozilla/5.0..."
```

### Enhanced formats with timing and connection info

```apache
# Combined + request duration (microseconds) + connection status
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %D %X" combined_extra

# JSON format (easy to parse, great for log pipelines)
LogFormat "{\"time\":\"%t\",\"remote_ip\":\"%a\",\"method\":\"%m\",\"url\":\"%U%q\",\"status\":%>s,\"bytes\":%B,\"duration_us\":%D,\"referer\":\"%{Referer}i\",\"user_agent\":\"%{User-Agent}i\"}" json
CustomLog /var/log/apache2/access.json json
```

### Format string reference

| Token | Description | Example |
|-------|-------------|---------|
| `%h` | Remote hostname (or IP) | `10.0.0.1` |
| `%a` | Remote IP address | `10.0.0.1` |
| `%l` | Remote logname (usually `-`) | `-` |
| `%u` | Remote user (HTTP auth) | `frank` |
| `%t` | Timestamp | `[10/Oct/2023:13:55:36 -0700]` |
| `%r` | First line of request | `GET /index.html HTTP/1.1` |
| `%m` | Request method | `GET` |
| `%U` | URL path (without query string) | `/index.html` |
| `%q` | Query string (with leading `?`) | `?page=2` |
| `%>s` | Final response status code | `200` |
| `%b` | Response size in bytes (or `-` if 0) | `2326` |
| `%B` | Response size in bytes (0 if empty) | `2326` |
| `%D` | Request processing time (microseconds) | `1250` |
| `%T` | Request processing time (seconds) | `0` |
| `%{ms}T` | Request processing time (milliseconds) | `1` |
| `%X` | Connection status: `X`=aborted, `+`=keep-alive, `-`=closed | `-` |
| `%{Header}i` | Request header value | `%{Referer}i` |
| `%{Header}o` | Response header value | `%{Content-Type}o` |
| `%v` | Server name (virtual host) | `www.example.com` |
| `%p` | Server port | `443` |

### Conditional logging

```apache
# Don't log health check requests
SetEnvIf Request_URI "^/health$" dontlog
SetEnvIf Request_URI "^/ready$" dontlog
SetEnvIf Request_URI "^/server-status" dontlog
CustomLog /var/log/apache2/access.log combined env=!dontlog

# Log slow requests (>1 second) to a separate file
# Requires mod_filter — alternatively, filter in your log pipeline
```

### Error log format

The error log format is not configurable the same way (prior to Apache 2.4). In 2.4+:

```apache
# Default error log format
ErrorLogFormat "[%{u}t] [%-m:%l] [pid %P:tid %T] %7F: %E: [client %a] %M% ,\\ referer\\ %{Referer}i"

# Simpler format
ErrorLogFormat "[%t] [%l] [pid %P] %F: %E: [client %a] %M"
```

## Recommended Datadog Dashboard Widgets

### Traffic overview

| Widget | Query | Visualization |
|--------|-------|---------------|
| Request rate | `avg:apache.net.hits{*}` | Timeseries (line) |
| Bytes served/sec | `avg:apache.net.bytes{*}` | Timeseries (line) |
| Requests by host | `avg:apache.net.hits{*} by {host}` | Timeseries (stacked) |

### Worker utilization

| Widget | Query | Visualization |
|--------|-------|---------------|
| Busy vs idle workers | `avg:apache.performance.busy_workers{*}`, `avg:apache.performance.idle_workers{*}` | Timeseries (stacked area) |
| Worker saturation % | `(avg:apache.performance.busy_workers{*} / (avg:apache.performance.busy_workers{*} + avg:apache.performance.idle_workers{*})) * 100` | Timeseries + horizontal marker at 80% |
| Async keep-alive connections | `avg:apache.conns_async_keep_alive{*}` | Timeseries |

### Errors and latency

| Widget | Query | Visualization |
|--------|-------|---------------|
| 5xx error rate | `count:apache.access.log{status:5*}.as_rate()` | Timeseries (bars) |
| 4xx error rate | `count:apache.access.log{status:4*}.as_rate()` | Timeseries (bars) |
| Request latency (p50/p95/p99) | `percentile:apache.access.log.duration{*}` | Timeseries (line, multiple percentiles) |
| Top URLs by error count | `count:apache.access.log{status:5*} by {http.url}.as_count()` | Top list |

### Host resources

| Widget | Query | Visualization |
|--------|-------|---------------|
| CPU by Apache host | `avg:system.cpu.user{service:apache} by {host}` | Timeseries |
| Memory usage | `avg:system.mem.used{service:apache} by {host}` | Timeseries |
| Disk I/O (log writes) | `avg:system.io.w_s{service:apache}` | Timeseries |

### Suggested monitors

| Monitor | Condition | Severity |
|---------|-----------|----------|
| High worker saturation | `busy_workers / (busy + idle) > 0.85` for 5 min | Warning |
| Approaching MaxRequestWorkers | `busy_workers > MaxRequestWorkers * 0.9` for 5 min | Critical |
| 5xx error spike | 5xx rate > 5% of total requests for 3 min | Critical |
| Request latency degradation | p95 latency > 2x baseline for 5 min | Warning |
| Apache process down | No `apache.performance.uptime` reported for 2 min | Critical |

## References

- [Monitoring Apache web server performance](https://www.datadoghq.com/blog/monitoring-apache-web-server-performance/) — key metrics and MPM internals
- [Collecting Apache performance metrics](https://www.datadoghq.com/blog/collect-apache-performance-metrics/) — how to collect metrics from mod_status and access logs
- [Monitor Apache web server with Datadog](https://www.datadoghq.com/blog/monitor-apache-web-server-datadog/) — integration setup, dashboards, and log analytics
