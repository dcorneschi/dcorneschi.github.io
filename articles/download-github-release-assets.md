# Downloading GitHub Release Assets from the Command Line

Grabbing a binary, manifest, or tarball from a GitHub release is a common setup step. There are three reliable ways to do it from a terminal: `curl`, `wget`, and the GitHub CLI (`gh`). This guide covers each, the "latest" download URL trick, verifying checksums, and a full download-extract-install example.

## The latest Download URL Pattern

GitHub exposes a predictable redirect for the newest release, so you don't have to hardcode a version:

```text
https://github.com/<owner>/<repo>/releases/latest/download/<asset>
```

Swap `latest/download` for `download/<tag>` to pin a specific version:

```text
https://github.com/<owner>/<repo>/releases/download/<tag>/<asset>
```

Both are redirects, which is why `curl`/`wget` need to follow redirects (see below).

## Method 1: curl (most common)

```bash
# Latest release, keep the original filename
curl -LO https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# A specific version
curl -LO https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.7.0/components.yaml

# Save under a custom name
curl -L https://github.com/owner/repo/releases/latest/download/file.tar.gz -o custom-name.tar.gz

# Fail loudly on HTTP errors (good in scripts) and show progress
curl -fL --progress-bar -O https://github.com/owner/repo/releases/latest/download/file.tar.gz
```

`-L` is essential — without it, curl saves the redirect HTML instead of the asset.

## Method 2: wget

```bash
# Latest release (wget follows redirects by default)
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Custom output name
wget -O custom-name.yaml https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## Method 3: GitHub CLI (gh)

`gh` is the cleanest option — it resolves assets for you and handles private repos via your auth token.

```bash
# All assets of the latest release
gh release download --repo kubernetes-sigs/metrics-server

# A specific version
gh release download v0.7.0 --repo kubernetes-sigs/metrics-server

# Only assets matching a pattern
gh release download --repo kubernetes-sigs/metrics-server --pattern "*.yaml"

# Into a target directory
gh release download v0.7.0 --repo owner/repo --dir ./downloads

# The source archive instead of built assets
gh release download v0.7.0 --repo owner/repo --archive tar.gz
```

Run from inside a cloned repo and you can omit `--repo`. `gh` is also the only method here that downloads assets from **private** releases without extra token juggling.

## Picking the Right Asset for Your Platform

Release assets are usually per-OS/architecture. Match your machine:

```bash
# Identify your platform
uname -s    # Linux / Darwin
uname -m    # x86_64 / aarch64 / arm64

# Example: fetch the linux amd64 build
curl -LO https://github.com/owner/repo/releases/download/v1.0.0/tool-linux-amd64.tar.gz
```

## Verifying the Download

Many projects publish checksums (and sometimes signatures) alongside assets. Verify before trusting a binary:

```bash
# Download the asset and its checksums file
curl -LO https://github.com/owner/repo/releases/download/v1.0.0/tool-linux-amd64.tar.gz
curl -LO https://github.com/owner/repo/releases/download/v1.0.0/checksums.txt

# Verify (Linux)
sha256sum -c checksums.txt --ignore-missing

# Verify (macOS)
shasum -a 256 -c checksums.txt --ignore-missing
```

## Full Example: Download, Extract, Install

```bash
# 1. Download the tarball for your platform
curl -LO https://github.com/owner/repo/releases/download/v1.0.0/binary-linux-amd64.tar.gz

# 2. Extract
tar xzf binary-linux-amd64.tar.gz

# 3. Make it executable and move onto your PATH
chmod +x binary
sudo mv binary /usr/local/bin/

# 4. Confirm it runs
binary --version
```

## Scripting: Fetch the Latest Version Tag

When you need the version string itself (for naming or logging), query the API:

```bash
# Latest tag via the GitHub API + jq
curl -fsSL https://api.github.com/repos/owner/repo/releases/latest | jq -r '.tag_name'

# Or with gh
gh release view --repo owner/repo --json tagName --jq '.tagName'
```

Note the API is rate-limited for unauthenticated requests (60/hour). Authenticate with a token — or just use `gh`, which uses your credentials — to raise the limit.

## Flags Reference

| Flag | Tool | Meaning |
|------|------|---------|
| `-L` | curl | Follow redirects (required for release URLs) |
| `-O` | curl | Save with the original remote filename |
| `-o <name>` | curl | Save under a custom filename |
| `-f` | curl | Fail on HTTP errors instead of saving error pages |
| `-O <name>` | wget | Save under a custom filename |
| `--pattern` | gh | Download only matching assets |
| `--dir` | gh | Output directory |
| `--archive` | gh | Download the source archive (`tar.gz`/`zip`) |

## Which to Use

- **curl** — universally available; the go-to for a known asset URL, especially in Dockerfiles and CI.
- **wget** — equivalent for simple downloads; follows redirects without extra flags.
- **gh** — best for private repos, pattern matching, or when you'd rather not build URLs by hand.
