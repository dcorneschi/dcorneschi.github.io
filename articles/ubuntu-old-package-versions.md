# Finding Old Package Versions on Ubuntu

Ubuntu repositories do not keep all versions. They only maintain the most recent versions for each pocket (release, updates, security). This guide covers where to find historical versions and how to install them.

## What Ubuntu Repositories Keep

### Active Repositories

| Pocket | What's Kept |
|--------|------------|
| Main repository | Only the original release version |
| Updates repository | Only the latest stable update version |
| Security repository | Only the latest security-patched version |

At any given time, you typically see only 2–3 versions in active repositories:

- Original release version
- Current update version
- Sometimes a security-specific version

### Example

For Ubuntu 20.04 (focal), `apt list -a snapd` might show:

```bash
snapd/focal-updates 2.70+20.04 amd64
snapd/focal 2.45.1+20.04 amd64
```

Only 2 versions — not the full history.

## Where Old Versions Are Kept

### Launchpad Archive (Complete History)

All historical versions are preserved on Launchpad:

- `https://launchpad.net/ubuntu/+source/<package>`

You can browse and download any old version here.

### Ubuntu Archive Pool

Browse specific release pools:

- `http://archive.ubuntu.com/ubuntu/pool/main/s/snapd/`

This directory contains old `.deb` files that may still be available (though not guaranteed to remain indefinitely).

### snapshot.debian.org (for Debian Packages)

For packages originating from Debian:

- `https://snapshot.debian.org/package/<package>/`

### packages.ubuntu.com

Search interface for current Ubuntu packages:

- `https://packages.ubuntu.com/`

## How to List Available Versions

### Check Active Repositories

```bash
# Show versions in configured repositories
apt-cache madison snapd

# Alternative method
apt list -a snapd

# Detailed version and origin info
apt-cache policy snapd
```

### Query Launchpad API

```bash
# Get all published versions for a release
curl -s "https://api.launchpad.net/1.0/ubuntu/+archive/primary?ws.op=getPublishedSources&source_name=snapd&distro_series=/ubuntu/focal" | \
    python3 -m json.tool | grep "source_package_version"
```

### Browse Archive Pool

```bash
# List all available .deb files in the pool
curl -s http://archive.ubuntu.com/ubuntu/pool/main/s/snapd/ | \
    grep -oP 'snapd_[0-9][^"]+\.deb' | sort -V
```

## How to Install Old Versions

### Download from Archive Pool

```bash
# Example: Download snapd 2.58
wget http://archive.ubuntu.com/ubuntu/pool/main/s/snapd/snapd_2.58+20.04_amd64.deb

# Install the package
sudo dpkg -i snapd_2.58+20.04_amd64.deb

# Fix any dependency issues
sudo apt --fix-broken install

# Hold the package to prevent updates
sudo apt-mark hold snapd
```

### Download from Launchpad

```bash
# Direct download from Launchpad
wget https://launchpad.net/ubuntu/+archive/primary/+files/snapd_2.58+20.04_amd64.deb

# Install and hold
sudo dpkg -i snapd_2.58+20.04_amd64.deb
sudo apt-mark hold snapd
```

### Downgrade Using apt

```bash
# If the older version is still in the repos
sudo apt install snapd=2.45.1+20.04

# Hold to prevent re-upgrade
sudo apt-mark hold snapd
```

## Why Repositories Don't Keep All Versions

1. **Mirror space** — hosting every version would require massive storage across all mirrors worldwide
2. **Bandwidth** — fewer packages to sync across the mirror network
3. **Security** — encourage users to run current, patched versions
4. **Support simplicity** — only support and test the latest versions
5. **Repository performance** — smaller package indices are faster to download and parse

## Managing Pinned/Held Packages

### Hold a Package

```bash
# Prevent upgrades
sudo apt-mark hold snapd

# Check held packages
apt-mark showhold
```

### Remove Hold

```bash
# Allow updates again
sudo apt-mark unhold snapd

# Update package
sudo apt update && sudo apt upgrade snapd
```

### Version Pinning with apt Preferences

```bash
# Pin a package to a specific version
cat << 'EOF' | sudo tee /etc/apt/preferences.d/snapd-pin
Package: snapd
Pin: version 2.58*
Pin-Priority: 1001
EOF

sudo apt update
```

## Verification Commands

```bash
# Check installed version
dpkg -l | grep snapd

# Check snap version (if applicable)
snap --version

# Check available versions in repos
apt-cache policy snapd

# Check if package is held
apt-mark showhold

# Show package changelog (see what changed between versions)
apt changelog snapd
```

## Alternative: Snapshot Repositories

Some services maintain historical package snapshots:

| Service | Scope |
|---------|-------|
| snapshot.debian.org | Debian packages (full history) |
| Launchpad | Ubuntu packages (full history) |
| Local apt-cacher-ng | Packages that passed through your proxy |
| Aptly/Reprepro | Self-hosted snapshot mirrors |

### Set Up a Local Cache

```bash
# Install apt-cacher-ng (caches all packages that pass through)
sudo apt install apt-cacher-ng

# Clients point to your cache
echo 'Acquire::http::Proxy "http://cache-server:3142";' | sudo tee /etc/apt/apt.conf.d/02proxy
```

## Best Practices

- Document which version you're running and why it's pinned
- Test downgrades in a non-production environment first
- Monitor for security updates — old versions miss patches
- Use version pinning only when absolutely necessary
- Have a plan to eventually move to supported versions
- Consider using Aptly or Reprepro for local mirrors if you need version control
