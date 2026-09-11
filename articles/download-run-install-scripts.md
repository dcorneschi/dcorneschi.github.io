# Safely Download and Run Installation Scripts

Compare common methods for downloading scripts, installers, release assets, configuration files, and archives with `curl` or `wget`. The examples progress from safer reviewable workflows to convenient pipe-to-shell one-liners, with guidance for checksums, version pinning, privileges, and endpoint testing.

> **Warning:** A remote installation script runs with your account's permissions—or full system permissions when invoked through `sudo`. Treat it as software installation, not as harmless text retrieval.

## Choose the Installation Method

Prefer the method with the strongest provenance, repeatability, and rollback support:

| Priority | Method | Typical use |
|----------|--------|-------------|
| 1 | Distribution package manager | Software available from a trusted OS repository |
| 2 | Vendor's signed package repository | Vendor-maintained packages and updates |
| 3 | Pinned release artifact with checksum or signature | Standalone binaries, packages, and archives |
| 4 | Download, inspect, verify, then execute | Vendor installation or repository-bootstrap scripts |
| 5 | Pipe a remote script directly into a shell | Disposable or interactive use after accepting the risk |

Package managers and signed release artifacts are usually easier to audit, upgrade, and remove than an unversioned remote script.

## Understand the Important Download Flags

### curl

| Flag | Meaning |
|------|---------|
| `-f`, `--fail` | Return nonzero for HTTP 400+ instead of saving an error page |
| `-sS` | Hide progress but retain error messages |
| `-L`, `--location` | Follow redirects |
| `-o FILE` | Write to a chosen filename |
| `-O` | Use the filename from the final URL |
| `-J` | Use a server-provided `Content-Disposition` filename with `-O` |
| `--proto '=https'` | Allow only HTTPS for this transfer |
| `--tlsv1.2` | Require TLS 1.2 or newer |
| `--retry N` | Retry selected transient failures |

A common scripted-download baseline is:

```bash
curl --proto '=https' --tlsv1.2 -fsSL \
  -o install.sh \
  https://downloads.example.com/install.sh
```

### wget

| Flag | Meaning |
|------|---------|
| `-q` | Quiet output |
| `-nv` | Reduced output while retaining useful errors |
| `-O FILE` | Write to a chosen file; use `-` for stdout |
| `--https-only` | Follow only HTTPS links |
| `--content-disposition` | Use a server-provided filename |

```bash
wget -nv --https-only \
  -O install.sh \
  https://downloads.example.com/install.sh
```

## Recommended: Download, Inspect, Verify, Execute

Downloading first separates retrieval from execution and prevents a truncated transfer from being executed as it arrives.

```bash
url='https://downloads.example.com/install.sh'
script=$(mktemp)
trap 'rm -f "$script"' EXIT

# Download as the current unprivileged user
curl --proto '=https' --tlsv1.2 -fsSL \
  -o "$script" \
  "$url"

# Confirm it is text and inspect its interpreter and contents
file "$script"
head -n 1 "$script"
less "$script"

# Validate shell syntax; choose bash -n for a Bash script
bash -n "$script"

# Execute only after review and verification
bash "$script"
```

Use `sh -n` and `sh` only when the script is POSIX shell-compatible. Running a Bash script with `sh` can fail or behave differently because arrays, `[[ ... ]]`, process substitution, and other Bash features are not portable POSIX syntax.

Optional static analysis with ShellCheck:

```bash
shellcheck "$script"
```

Static checks improve review but do not prove that a script is safe.

## Verify Checksums and Signatures

When the publisher provides a checksum:

```bash
curl -fsSLO https://downloads.example.com/tool-v1.2.3.tar.gz
curl -fsSLO https://downloads.example.com/tool-v1.2.3.sha256
sha256sum -c tool-v1.2.3.sha256
```

On macOS:

```bash
shasum -a 256 -c tool-v1.2.3.sha256
```

A checksum downloaded from the same compromised location as the artifact does not protect against a complete publisher compromise. A cryptographic signature verified with a key obtained through an independent trusted channel provides stronger provenance.

Verify a detached GPG signature when available:

```bash
gpg --verify tool-v1.2.3.tar.gz.asc tool-v1.2.3.tar.gz
```

Confirm the signing-key fingerprint against the publisher's official documentation before trusting it.

## Pin the Script or Release Version

Avoid mutable URLs such as `master`, `main`, `latest`, or an unversioned installer when repeatability matters:

```text
# Mutable
https://raw.githubusercontent.com/owner/repo/main/install.sh

# Pinned to a commit
https://raw.githubusercontent.com/owner/repo/0123456789abcdef/install.sh

# Pinned release asset
https://github.com/owner/repo/releases/download/v1.2.3/tool-linux-amd64.tar.gz
```

Record the version, URL, checksum, and signing identity in automation and change records.

## Install a Reviewed Script on PATH

For a script intended to become a reusable command:

```bash
curl -fsSLo tool \
  https://raw.githubusercontent.com/owner/repo/COMMIT_ID/tool

less tool
bash -n tool
sha256sum tool

sudo install -o root -g root -m 0755 tool /usr/local/bin/tool
tool --version
```

`install` sets destination ownership and permissions atomically. `/usr/local/bin` is appropriate for locally managed commands that are not owned by the distribution package manager.

A directly executable script should have an appropriate shebang, for example:

```bash
#!/usr/bin/env bash
```

## Pipe Directly to a Shell

The shortest pattern downloads and executes concurrently:

```bash
curl -fsSL https://downloads.example.com/install.sh | bash
wget -qO- https://downloads.example.com/install.sh | bash
```

This is convenient but has important limitations:

- The code is executed before it can be fully reviewed.
- A network failure can leave the shell with a partial script.
- A mutable URL can change between executions.
- The shell's exit code may hide a download failure unless pipeline handling is configured.
- Logs do not automatically retain the exact script that ran.

If the risk is accepted, enable pipeline failure detection and use a pinned URL:

```bash
set -o pipefail
curl -fsSL \
  https://raw.githubusercontent.com/owner/repo/COMMIT_ID/install.sh \
  | bash
```

`pipefail` exposes a curl failure as a failed pipeline, but it cannot stop Bash from executing content already received. Downloading to a file remains safer.

### Pass arguments to the remote script

```bash
curl -fsSL https://downloads.example.com/install.sh \
  | bash -s -- --version 1.2.3 --prefix /usr/local
```

`bash -s --` tells Bash to read the script from standard input and treats subsequent values as positional arguments.

### Run with elevated privileges

Avoid this pattern for unreviewed code:

```bash
# High risk: remote content is executed as root immediately.
curl -fsSL https://downloads.example.com/install.sh | sudo bash
```

Prefer downloading and reviewing as an unprivileged user, then elevate only the reviewed execution:

```bash
curl -fsSLo install.sh https://downloads.example.com/install.sh
less install.sh
bash -n install.sh
sudo bash install.sh
```

Better still, use a package manager or run only the specific installation step that requires elevation.

## Execute with Command Substitution

Some projects publish commands such as:

```bash
sh -c "$(curl -fsSL https://downloads.example.com/install.sh)"
bash -c "$(wget -qO- https://downloads.example.com/install.sh)"
```

Command substitution completes the download before `sh -c` or `bash -c` starts, unlike a pipeline. However, it still executes unreviewed mutable content, stores the script in memory, can obscure the downloader's exit status, and does not retain the downloaded script for auditing.

A checked variant separates the exit status:

```bash
remote_script=$(curl -fsSL https://downloads.example.com/install.sh) || exit 1
bash -c "$remote_script"
```

A temporary file is preferable for nontrivial installers because it can be inspected, hashed, and archived.

## Execute with Process Substitution

Bash and Zsh support process substitution:

```bash
bash <(curl -fsSL https://downloads.example.com/install.sh)
```

Arguments can be supplied normally:

```bash
bash <(curl -fsSL https://downloads.example.com/install.sh) \
  --version 1.2.3
```

Process substitution is not POSIX shell syntax. It also allows execution while the download is still in progress, so it does not provide the safety benefits of downloading to a regular file first.

## Understand Common Vendor One-Liners

Official projects often display commands such as:

```bash
# Oh My Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

These commands are concise, but `master` and `HEAD` are mutable references. Before running them:

1. Confirm the URL from the project's official documentation.
2. Download and review the current script.
3. Check whether the installer modifies shell startup files, repositories, users, services, or permissions.
4. Run it as the intended user, not root, unless the official installation model requires elevation.
5. Save the reviewed script or pin a commit when reproducibility is required.

## Download Files without Executing Them

### Choose an explicit filename

```bash
curl -fsSL \
  -o docker-compose.yml \
  https://raw.githubusercontent.com/owner/repo/COMMIT_ID/docker-compose.yml
```

Validate configuration files before deployment:

```bash
docker compose -f docker-compose.yml config
```

### Preserve the URL filename

```bash
curl -fLO \
  https://github.com/owner/repo/releases/download/v1.2.3/tool.tar.gz
```

### Use a server-provided filename

```bash
curl -fLJO https://github.com/owner/repo/tarball/v1.2.3
```

`-J` trusts the server's `Content-Disposition` filename. Prefer `-o` with an explicit name in automation so the destination is predictable and cannot collide with an unexpected server-selected name.

## Download the Latest GitHub Release Asset

The GitHub CLI is the cleanest option when available:

```bash
gh release download v1.2.3 \
  --repo owner/repo \
  --pattern '*linux-armhf.deb'
```

To query the API, parse JSON with `jq` instead of `grep` and `cut`:

```bash
asset_url=$(
  curl -fsSL https://api.github.com/repos/owner/repo/releases/latest \
    | jq -er '.assets[]
        | select(.name == "tool-bullseye-armhf.deb")
        | .browser_download_url'
) || exit 1

curl -fL -o tool-bullseye-armhf.deb "$asset_url"
```

Always quote the resulting URL and fail if no matching asset is found. For reproducible automation, request a specific release tag rather than `latest`.

GitHub API requests are rate-limited. Use `gh` or authenticated API requests where appropriate, but never print tokens into logs.

## Download and Extract Archives

### Recommended: download, list, then extract

```bash
archive=$(mktemp)
extract_dir=$(mktemp -d)
trap 'rm -f "$archive"; rm -rf "$extract_dir"' EXIT

curl -fsSL \
  -o "$archive" \
  https://github.com/owner/repo/archive/refs/tags/v1.2.3.tar.gz

# Inspect paths before extraction
tar -tzf "$archive" | less

# Extract as the current user
tar -xzf "$archive" \
  --no-same-owner \
  --no-same-permissions \
  --strip-components=1 \
  -C "$extract_dir"
```

Review archive paths for absolute names, `..` traversal, unexpected links, device files, and files outside the intended top-level directory. Avoid extracting untrusted archives as root.

### Stream a trusted archive

```bash
set -o pipefail
curl -fsSL \
  https://github.com/owner/repo/archive/refs/tags/v1.2.3.tar.gz \
  | tar -xzf - --strip-components=1
```

Streaming saves temporary disk space but prevents prior archive inspection and checksum verification. Use it only for a pinned, trusted artifact in a prepared empty directory.

## Bootstrap a Package Repository

Commands such as the following execute a vendor-provided repository setup script as root:

```bash
curl -fsSL https://packages.example.com/install/repositories/product/script.deb.sh \
  | sudo bash
```

A safer workflow is:

```bash
curl -fsSLo repository-setup.sh \
  https://packages.example.com/install/repositories/product/script.deb.sh

less repository-setup.sh
bash -n repository-setup.sh
sudo bash repository-setup.sh
```

Before execution, identify the files and trust stores the script will modify, such as:

- `/etc/apt/sources.list.d/`
- `/etc/apt/keyrings/`
- `/etc/yum.repos.d/`
- Package-manager pinning or preference files

Prefer documented manual repository setup when it provides explicit key fingerprints and `signed-by` configuration.

## Test a Download or Installation Endpoint

### Retrieve the full page

```bash
curl -fsSL https://service.example.com/
```

### Request headers only

```bash
curl -fsSI https://service.example.com/
```

`-I` sends a HEAD request. Some applications handle HEAD differently or do not support it, so it is not always a complete reachability test.

### Test a GET without printing the body

```bash
curl -sS -o /dev/null \
  -w 'HTTP %{http_code} in %{time_total}s\n' \
  https://service.example.com/
```

Add `--fail` or `--fail-with-body` when an HTTP error must produce a nonzero exit code.

### Test HTTP/2

```bash
curl --http2 -sS -o /dev/null \
  -w 'HTTP %{http_version} status %{http_code}\n' \
  https://service.example.com/
```

The installed curl build must include HTTP/2 support:

```bash
curl --version
```

### Test a specific address while preserving TLS identity

```bash
curl -fsSI \
  --resolve service.example.com:443:192.0.2.10 \
  https://service.example.com/
```

`--resolve` connects to the selected address while retaining the URL hostname for the HTTP Host header, TLS SNI, and certificate validation.

Do not combine it with `-k` merely because the target uses an internal address. Install or provide the correct CA instead:

```bash
curl --cacert internal-ca.crt \
  --resolve service.example.com:443:192.0.2.10 \
  https://service.example.com/
```

## Secure Automation Pattern

A reusable automation pattern should pin the source, protect temporary files, verify integrity, and install atomically:

```bash
#!/usr/bin/env bash
set -euo pipefail

version='1.2.3'
asset="tool-${version}-linux-amd64.tar.gz"
base_url="https://github.com/owner/repo/releases/download/v${version}"
workdir=$(mktemp -d)
trap 'rm -rf "$workdir"' EXIT

curl --proto '=https' --tlsv1.2 -fsSL \
  --retry 3 \
  -o "$workdir/$asset" \
  "$base_url/$asset"

curl --proto '=https' --tlsv1.2 -fsSL \
  -o "$workdir/checksums.txt" \
  "$base_url/checksums.txt"

(
  cd "$workdir"
  sha256sum -c checksums.txt --ignore-missing
  tar -xzf "$asset" --no-same-owner --no-same-permissions
)

sudo install -o root -g root -m 0755 \
  "$workdir/tool" /usr/local/bin/tool

/usr/local/bin/tool --version
```

Retries are suitable for idempotent downloads. Avoid automatic retries around state-changing remote API calls or installer execution unless replay is safe.

## Method Comparison

| Pattern | Reviewable | Pins content | Detects HTTP errors | Partial-download execution risk | Recommended use |
|---------|------------|--------------|---------------------|---------------------------------|-----------------|
| Package manager | Yes | Repository-controlled | Yes | No | Preferred |
| Download + checksum + execute | Yes | Yes, when versioned | Yes with `-f` | No | Recommended script workflow |
| `curl URL \| bash` | No | Only with immutable URL | Yes with `-f`, but pipeline handling matters | Yes | Accepted risk only |
| `bash -c "$(curl ...)"` | No | Only with immutable URL | Easy to obscure | Lower, but still unreviewed | Convenience only |
| Process substitution | No | Only with immutable URL | Download status can be obscure | Yes | Interactive convenience |
| Streamed archive extraction | Limited | Yes, when versioned | Yes with `-f` and `pipefail` | Extracts while downloading | Trusted archives only |

## Installation Checklist

1. Confirm the URL through official documentation.
2. Prefer a package manager or signed release artifact.
3. Pin a release tag, commit, or immutable version.
4. Download with HTTPS, redirects, and HTTP failure handling.
5. Inspect the script, configuration, or archive before use.
6. Verify a published checksum or signature.
7. Run syntax and static checks where applicable.
8. Execute first as an unprivileged user when possible.
9. Elevate only the operation that requires root access.
10. Avoid storing credentials in URLs, scripts, or shell history.
11. Record exactly what version and digest were installed.
12. Verify the installed command, service, or configuration.
13. Keep an uninstall or rollback procedure.

## See Also

- [curl Cheatsheet](articles/curl-cheatsheet.md) — HTTP requests, downloads, proxies, TLS, output, and diagnostics
- [Bash Essentials Guide](articles/bash-essentials-guide.md) — pipelines, substitutions, scripts, and shell execution
- [Bash Pipelines and Redirections](articles/bash-redirection-operators.md) — pipeline behavior and redirection operators
- [Downloading GitHub Release Assets from the Command Line](articles/download-github-release-assets.md) — curl, wget, GitHub CLI, checksums, and assets
- [Running a Script Directly from a GitHub Gist](articles/run-script-from-github-gist.md) — raw Gist URLs, revision pinning, and execution methods
- [Linux File Permissions Guide](articles/linux-file-permissions.md) — executable bits, ownership, and secure installation modes
