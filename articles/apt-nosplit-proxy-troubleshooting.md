# Troubleshoot APT NOSPLIT and Excess Data Errors Behind a Proxy

Diagnose APT repository metadata failures caused by caching proxies, authentication gateways, content filters, or malformed HTTP responses. This guide focuses on errors such as `NOSPLIT`, invalid clear-signed files, unexpected data after a response, and zero-length repository metadata.

> **Security:** Do not solve these errors by permanently disabling TLS certificate verification, repository signature checks, or Release-file expiration checks. Those controls protect package integrity and repository freshness.

## Symptoms

An affected `apt-get update` may report errors similar to:

```text
Clearsigned file isn't valid, got 'NOSPLIT'
The repository is no longer signed
Writing more data than expected
excess found
zero-length body
Hash Sum mismatch
```

The exact message varies with the APT and HTTP method versions. These errors indicate that APT did not receive or parse the repository object it expected; they do not prove a single root cause.

## What NOSPLIT Usually Means

Ubuntu and Debian repositories publish signed metadata such as `InRelease`. An `InRelease` file is a clear-signed document that begins with an OpenPGP marker:

```text
-----BEGIN PGP SIGNED MESSAGE-----
```

APT verifies this content before trusting package indexes. A `NOSPLIT` or clear-signature parsing error can occur when the downloaded object is not the expected signed metadata, for example:

- A proxy login or access-denied page was returned as HTML.
- A caching proxy appended a banner, footer, or diagnostic payload.
- A proxy incorrectly attached a response body to `304 Not Modified`.
- A transparent filter modified, truncated, or recompressed the response.
- A captive portal intercepted the request.
- A stale or partially downloaded APT list remained on disk.
- A mirror or content-delivery endpoint returned an incomplete object.
- The system clock or repository metadata is invalid.

GPG errors are often the integrity check detecting upstream corruption rather than a problem with the repository key itself.

## Why a Successful curl Test Is Not Conclusive

A successful `curl` request proves that an HTTP client reached an endpoint. It does not prove that APT received identical content because the two clients may send different:

- Cache-control and conditional request headers
- User-Agent values
- Proxy authentication headers
- Compression headers
- HTTP versions or connection-reuse patterns
- URLs and repository paths

Test the exact URL that APT reports as failing, through the same proxy and from the same host.

## Establish a Baseline

Check time, disk space, configured repositories, and APT proxy settings:

```bash
date -Is
timedatectl status
df -h / /var
apt-config dump | grep -iE 'Acquire::(http|https).*Proxy|No-Cache|Pipeline|Timeout|Retries|ForceIPv'
grep -RhsE '^[^#].*(deb |URIs:)' /etc/apt/sources.list /etc/apt/sources.list.d/ 2>/dev/null
```

Check whether shell-level proxy variables differ between the current user and `sudo`:

```bash
env | grep -iE '^(http|https|no)_proxy='
sudo env | grep -iE '^(http|https|no)_proxy='
```

APT configuration under `/etc/apt/apt.conf.d/` normally takes precedence for system package operations.

## Capture APT HTTP Diagnostics

Run an update with acquire debugging enabled:

```bash
sudo apt-get update \
  -o Debug::Acquire::http=true \
  -o Debug::Acquire::https=true
```

For a protected log file:

```bash
sudo sh -c 'umask 077; apt-get update \
  -o Debug::Acquire::http=true \
  -o Debug::Acquire::https=true \
  2>&1 | tee /root/apt-acquire-debug.log'
```

Look for:

```bash
sudo grep -iE \
  'NOSPLIT|clearsigned|excess|zero-length|HTTP/[0-9.]+|407|403|content-type|content-length' \
  /root/apt-acquire-debug.log
```

> **Sensitive data:** Debug output can contain internal repository names, proxy addresses, request headers, and authentication details. Sanitize it before sharing and delete it according to local policy.

## Inspect the Failing Object

Download the exact failing `InRelease` object through the proxy. Replace the URL and proxy address:

```bash
curl --fail --show-error --location \
  --proxy http://proxy.example.com:3128 \
  --dump-header /tmp/inrelease.headers \
  --output /tmp/InRelease \
  http://archive.ubuntu.com/ubuntu/dists/noble/InRelease
```

Inspect the status, content type, size, and first lines:

```bash
sed -n '1,30p' /tmp/inrelease.headers
file /tmp/InRelease
wc -c /tmp/InRelease
head -n 5 /tmp/InRelease
```

Expected metadata should begin with the OpenPGP clear-signed marker. HTML, proxy branding, authentication forms, legal banners, or unrelated JSON indicate interception.

Do not include proxy credentials directly in commands retained in shell history. Use the organization's approved credential mechanism and protect any APT configuration containing secrets with mode `0600`.

## Interpret HTTP Evidence

| Evidence | Likely interpretation | Next action |
|----------|-----------------------|-------------|
| `407 Proxy Authentication Required` | Proxy credentials are missing or rejected | Correct authentication; do not change signature validation |
| `403 Forbidden` with HTML | Proxy or repository policy denied the request | Review allowlists and proxy policy |
| `200 OK` with OpenPGP metadata | Transport appears valid for that request | Compare APT headers, URL, and cached state |
| `200 OK` with HTML or a banner | Proxy or captive portal replaced the object | Fix proxy/filter policy |
| `304 Not Modified` with an unexpected body | Broken intermediary cache behavior | Test APT No-Cache and correct the proxy |
| Truncated or zero-length object | Proxy, mirror, storage, or network interruption | Retry, compare endpoints, and inspect proxy logs |
| Valid content but Release expired | Clock skew or stale repository metadata | Correct time or repository; keep expiration checks enabled |
| Hash mismatch after a mirror sync | Mirror or cache has inconsistent objects | Retry later or use a healthy mirror |

In one incident, a proxy appended 12,784 bytes to a cached response. That number is incident-specific; any unexpected payload length must be investigated rather than used as a universal signature.

## Test the Minimal No-Cache Workaround

First test No-Cache as a one-time option without changing persistent configuration:

```bash
sudo apt-get update \
  -o Acquire::http::No-Cache=true \
  -o Acquire::https::No-Cache=true
```

`No-Cache` asks APT's HTTP methods to avoid relying on cached responses. It can work around a proxy that mishandles cache revalidation or stale objects.

It does **not** guarantee that every response will be clean or that a proxy cannot modify a `200 OK` response. Repository signature verification remains the final integrity control.

If the one-time test consistently succeeds while the normal update fails, add a persistent drop-in:

```bash
sudo install -m 600 /dev/null \
  /etc/apt/apt.conf.d/90corporate-proxy-nocache
sudo vi /etc/apt/apt.conf.d/90corporate-proxy-nocache
```

```text
// Work around incorrect cache handling by the corporate proxy.
Acquire::http::No-Cache "true";
Acquire::https::No-Cache "true";
```

Confirm that APT loaded it:

```bash
apt-config dump | grep -i No-Cache
```

No-Cache can increase bandwidth usage and repository/proxy load. Treat it as a documented workaround while the proxy behavior is corrected.

## Configure an Explicit Proxy

If APT requires a proxy and one is not already configured, extend the protected drop-in:

```text
Acquire::http::Proxy "http://proxy.example.com:3128/";
Acquire::https::Proxy "http://proxy.example.com:3128/";

Acquire::http::No-Cache "true";
Acquire::https::No-Cache "true";
```

An `http://` proxy URL is common even for HTTPS repositories because the client establishes a CONNECT tunnel through the proxy. Follow the organization's proxy documentation.

Verify the effective values without printing credentials into shared logs:

```bash
sudo apt-config dump | grep -iE 'Acquire::(http|https)::(Proxy|No-Cache)'
```

### Bypass the proxy for an approved host

When policy permits a repository to be reached directly:

```text
Acquire::http::Proxy::security.ubuntu.com "DIRECT";
Acquire::https::Proxy::security.ubuntu.com "DIRECT";
```

Use host-specific bypasses only when direct routing and firewall policy allow them.

## Disable Pipelining Only When Needed

Some older or noncompliant proxies mishandle pipelined HTTP requests. Test without pipelining:

```bash
sudo apt-get update \
  -o Acquire::http::Pipeline-Depth=0 \
  -o Acquire::https::Pipeline-Depth=0
```

If this test is required in addition to No-Cache, persist only those settings:

```text
Acquire::http::Pipeline-Depth "0";
Acquire::https::Pipeline-Depth "0";
```

Disabling pipelining can reduce performance. Do not add it when No-Cache alone resolves the incident.

## Add Timeouts, Retries, or IPv4 Selectively

Timeouts and retries help with transient transport failures but do not repair modified content:

```text
Acquire::http::Timeout "120";
Acquire::https::Timeout "120";
Acquire::Retries "3";
```

Force IPv4 only when diagnostics show broken IPv6 routing or proxy resolution:

```text
Acquire::ForceIPv4 "true";
```

Do not combine unrelated workarounds into the first change. Apply one evidence-based adjustment at a time so the effective fix remains clear.

## Keep TLS Verification Enabled

Do not persist these unsafe settings:

```text
// Unsafe: do not use as a production workaround.
Acquire::https::Verify-Peer "false";
Acquire::https::Verify-Host "false";
```

If the corporate proxy performs authorized TLS inspection, install its issuing CA certificate instead:

```bash
sudo install -m 644 corporate-proxy-ca.crt \
  /usr/local/share/ca-certificates/corporate-proxy-ca.crt
sudo update-ca-certificates
```

Then verify HTTPS with certificate validation enabled:

```bash
curl --fail --show-error --head \
  --proxy http://proxy.example.com:3128 \
  https://security.ubuntu.com/
```

A CA certificate must be obtained through a trusted internal channel and its fingerprint verified before installation.

## Keep Repository Freshness and Signature Checks Enabled

Do not disable Release-file expiration as a proxy workaround:

```text
// Unsafe for a permanent fix:
Acquire::Check-Valid-Until "false";
```

An expired Release file can indicate clock skew, an abandoned repository, or stale mirrored metadata. Check `timedatectl`, proxy cache age, and repository health instead.

Do not attempt to force a single digest with an undocumented setting such as:

```text
APT::Hashes "SHA256";
```

APT negotiates supported hashes from signed repository metadata. Keep APT current and let signature and hash verification reject modified content.

## Reset Downloaded List State Safely

If APT may have retained incomplete lists, preserve the old directory for rollback and create a clean one:

```bash
sudo mv /var/lib/apt/lists \
  "/var/lib/apt/lists.corrupt.$(date +%Y%m%d%H%M%S)"
sudo install -d -m 755 /var/lib/apt/lists
sudo install -d -o _apt -g root -m 700 /var/lib/apt/lists/partial
sudo apt-get clean
sudo apt-get update
```

`apt-get clean` removes downloaded package archives; replacing `/var/lib/apt/lists` forces repository metadata to be downloaded again.

After a successful update and an appropriate retention period, remove the preserved directory according to local change procedures.

## Verify the Fix

Run a normal update first:

```bash
sudo apt-get update
```

Then capture HTTP status lines when needed:

```bash
sudo apt-get update -o Debug::Acquire::http=true 2>&1 \
  | grep -E 'HTTP/[0-9.]+ [0-9]{3}'
```

Successful verification means:

- APT completes without `NOSPLIT`, clear-signature, excess-data, or hash errors.
- Repository signatures remain enabled and valid.
- The expected repositories are used.
- Proxy authentication and TLS validation succeed.
- Repeated updates behave consistently.

A `200 OK` response can support the diagnosis, but status code alone does not establish content integrity.

## Roll Back the Workaround

After the proxy is fixed, test without the persistent workaround:

```bash
sudo mv /etc/apt/apt.conf.d/90corporate-proxy-nocache \
  /etc/apt/90corporate-proxy-nocache.disabled
sudo apt-get update
```

If updates succeed repeatedly, retain the disabled file only for the incident record or remove it through normal configuration management. If the failure returns, restore the file and continue remediation with the proxy team.

## Deploy to Ubuntu EKS Nodes

APT applies only to Ubuntu-based EKS nodes. Standard Amazon Linux and Bottlerocket EKS images use different package and operating-system management models.

EKS nodes are replaceable infrastructure. A manual change on one node does not survive instance replacement or a managed node-group update.

### Preferred rollout methods

Use these methods in order of preference:

1. Bake the tested APT configuration into a versioned Ubuntu EKS AMI.
2. Add the file through launch-template or pre-bootstrap user data.
3. Apply it with an approved configuration-management system.
4. Use a privileged DaemonSet only as a temporary, explicitly reviewed fallback.

Example cloud-init configuration for user data:

```yaml
#cloud-config
write_files:
  - path: /etc/apt/apt.conf.d/90corporate-proxy-nocache
    owner: root:root
    permissions: '0600'
    content: |
      Acquire::http::Proxy "http://proxy.example.com:3128/";
      Acquire::https::Proxy "http://proxy.example.com:3128/";
      Acquire::http::No-Cache "true";
      Acquire::https::No-Cache "true";
```

Do not embed reusable proxy credentials directly in AMIs, Git repositories, ConfigMaps, or unencrypted user data. Use an approved secret-delivery mechanism.

### DaemonSet risks

A DaemonSet that writes the host's `/etc/apt/apt.conf.d/` requires a host mount and elevated privileges. It can:

- Modify every matching node's operating system.
- Expose proxy credentials to Kubernetes resources or logs.
- Be blocked by Pod Security Admission or policy engines.
- Race with node bootstrap or package operations.
- Disappear from a replacement node until the pod schedules and runs.

If a DaemonSet is unavoidable, pin it to the intended Ubuntu node groups, use least privilege, avoid credentials in manifests, make changes idempotent, record the previous file, and provide a rollback action. Node bootstrap or an image pipeline is more reliable for persistent configuration.

## Troubleshooting Checklist

1. Confirm the exact failing repository URL.
2. Verify system time and available disk space.
3. Check APT and environment proxy configuration.
4. Capture APT acquire debugging securely.
5. Inspect the actual response body for HTML or injected data.
6. Test `No-Cache` as a one-time option.
7. Persist only the option that fixes the reproducible failure.
8. Add `Pipeline-Depth "0"` only if required.
9. Install the corporate CA instead of disabling TLS verification.
10. Keep signature and Release-expiration checks enabled.
11. Refresh local list state after correcting the transport.
12. Re-test repeatedly and monitor proxy logs.
13. For EKS, apply the fix through the node image or bootstrap path.
14. Remove the workaround after the proxy defect is resolved.

## See Also

- [apt Cheatsheet](articles/apt-cheatsheet.md) — proxy options, acquire debugging, package operations, and common errors
- [Fixing apt Lock Held Errors](articles/apt-lock-held-fix.md) — diagnose APT and `dpkg` lock contention safely
- [cloud-init Cheatsheet](articles/cloud-init-cheatsheet.md) — write persistent files during instance bootstrap
- [EKS: pre_userdata vs additional_userdata](articles/eks-pre-userdata-vs-additional-userdata.md) — choose the correct node bootstrap phase
- [EKS Node Lifecycle During Template Updates](articles/eks-node-lifecycle-during-updates.md) — understand why node-local changes disappear during replacement
