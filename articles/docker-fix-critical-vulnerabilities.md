# Fixing Critical Vulnerabilities in Public Docker Images

A practical guide to scanning public Docker images, understanding which CVEs you can actually fix, and rebuilding clean images. Not every vulnerability is fixable — this walks through how to tell the difference and what to do in each case.

## The Reality of Public Image CVEs

When you scan a popular base image (`node`, `python`, `nginx`, `postgres`), you will find vulnerabilities. This is normal. The important questions are:

- Is there a **fixed version** available for the vulnerable package?
- Is the vulnerable package actually **used** at runtime, or just present?
- Does the CVE apply to **your usage** (config, exposed surface)?

A CVE is **fixable** when the OS/language package maintainer has published a patched version. It is **not fixable by you** when:

- No upstream fix exists yet (status: `affected`, `will_not_fix`, or `pending`)
- The vulnerability is in the base OS and the maintainer hasn't released an update
- The fix requires a major version bump that breaks your app

## Step 1: Scan the Image

### Trivy (most common)

```bash
# Install (macOS)
brew install trivy

# Scan an image, only show fixable CRITICAL issues
trivy image --severity CRITICAL --ignore-unfixed node:18

# Include HIGH, show everything
trivy image --severity CRITICAL,HIGH node:18

# Machine-readable output
trivy image --severity CRITICAL --format json -o report.json node:18
```

The `--ignore-unfixed` flag is the single most useful option — it hides CVEs that have no available fix, leaving only the ones you can actually act on.

### Grype (Anchore)

```bash
# Install
brew install grype

# Scan, only fixable
grype node:18 --only-fixed

# Filter severity
grype node:18 --fail-on critical
```

### Docker Scout (built into Docker)

```bash
# Quick overview
docker scout quickview node:18

# Detailed CVE list
docker scout cves node:18

# Show only fixable, with recommended base image
docker scout cves --only-fixed node:18
docker scout recommendations node:18
```

## Step 2: Understand What You're Looking At

Trivy output columns tell you everything:

```
┌────────────┬────────────────┬──────────┬────────┬───────────────────┬───────────────┐
│  Library   │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version │
├────────────┼────────────────┼──────────┼────────┼───────────────────┼───────────────┤
│ openssl    │ CVE-2024-xxxxx │ CRITICAL │ fixed  │ 3.0.11-1          │ 3.0.13-1      │  ← fixable
│ zlib1g     │ CVE-2023-xxxxx │ CRITICAL │ affected│ 1:1.2.13         │               │  ← NOT fixable
└────────────┴────────────────┴──────────┴────────┴───────────────────┴───────────────┘
```

| Status | Meaning | Action |
|--------|---------|--------|
| `fixed` | Patched version exists | Upgrade the package or base image |
| `affected` | Vulnerable, no fix yet | Wait, mitigate, or accept risk |
| `will_not_fix` | Maintainer won't patch | Switch base image or accept risk |
| `fix_deferred` | Fix planned, not released | Track upstream, mitigate |
| `end_of_life` | Distro version unsupported | Move to a supported base |

## Step 3: The Fix Strategies (in order of preference)

### Strategy 1: Update the Base Image Tag

Most CVEs get fixed just by pulling the latest patch of your base image. `node:18` today is not the same as `node:18` last month.

```bash
# Pull fresh and re-scan
docker pull node:18
trivy image --severity CRITICAL --ignore-unfixed node:18
```

Pin to a specific patched digest for reproducibility once clean:

```dockerfile
FROM node:18.20.4-bookworm-slim@sha256:abc123...
```

### Strategy 2: Switch to a Slimmer / Distroless Base

Fewer packages means fewer CVEs. The vulnerable package often isn't even needed.

| Base | Approx. package count | Typical CVE count |
|------|:---------------------:|:-----------------:|
| `node:18` (Debian full) | ~400+ | High |
| `node:18-slim` | ~120 | Medium |
| `node:18-alpine` | ~40 | Low |
| `gcr.io/distroless/nodejs18` | ~15 | Very low |

```dockerfile
# Before — full Debian, many CVEs
FROM node:18

# After — distroless, minimal attack surface
FROM node:18-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .

FROM gcr.io/distroless/nodejs18-debian12
COPY --from=build /app /app
WORKDIR /app
CMD ["server.js"]
```

### Strategy 3: Patch OS Packages During Build

If the base image is stale but the distro has published fixes, update packages in your own layer.

```dockerfile
# Debian / Ubuntu
FROM python:3.12-slim
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends openssl && \
    rm -rf /var/lib/apt/lists/*
```

```dockerfile
# Alpine
FROM node:18-alpine
RUN apk update && apk upgrade --no-cache
```

```dockerfile
# RHEL / UBI
FROM registry.access.redhat.com/ubi9/ubi-minimal
RUN microdnf update -y && microdnf clean all
```

This forces the latest patched versions of every OS package at build time — often clearing a batch of CRITICALs in one shot.

### Strategy 4: Upgrade the Vulnerable Application Dependency

For CVEs in language packages (npm, pip, gem, go modules), fix the lockfile, not the OS.

```bash
# Node — find and fix
npm audit
npm audit fix
# Force a specific transitive dep
npm install lodash@4.17.21

# Python
pip install --upgrade urllib3
# or pin in requirements.txt / constraints.txt

# Go
go get -u vulnerable/module@v1.2.3
go mod tidy
```

Then rebuild and re-scan.

### Strategy 5: Remove the Vulnerable Package Entirely

If a package is present but unused, drop it.

```dockerfile
FROM debian:12-slim
# Remove build tools and package managers after use
RUN apt-get update && apt-get install -y build-essential \
    && make \
    && apt-get purge -y build-essential \
    && apt-get autoremove -y \
    && rm -rf /var/lib/apt/lists/*
```

Multi-stage builds do this naturally — build tooling stays in the build stage and never ships in the final image.

## Step 4: When a CVE Can't Be Fixed

Sometimes there is genuinely no upstream fix. Options:

### Assess actual exploitability

A CRITICAL CVE in a library your code never calls is low real-world risk. Check:

- Is the vulnerable function reachable from your code?
- Is the affected service exposed to the network?
- Does exploitation require conditions you don't meet?

### Mitigate at runtime

```yaml
# Reduce blast radius even with an unfixable CVE present
services:
  app:
    read_only: true
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    user: "1000:1000"
```

### Accept and document (suppress the noise)

Trivy `.trivyignore`:

```
# CVE-2023-xxxxx — zlib, no upstream fix, not reachable from our code
# Reviewed 2026-08-29, revisit in 90 days
CVE-2023-xxxxx
```

Grype `.grype.yaml`:

```yaml
ignore:
  - vulnerability: CVE-2023-xxxxx
    reason: "No fix available; package not used at runtime"
```

Always document why and set a review date. Suppression is a decision, not a fix.

## Step 5: Verify and Prevent Regression

### Confirm the fix

```bash
docker build -t myapp:fixed .
trivy image --severity CRITICAL --ignore-unfixed myapp:fixed
# Expect: no fixable CRITICAL vulnerabilities
```

### Fail CI on new fixable CRITICALs

```bash
# Non-zero exit if any fixable CRITICAL is found
trivy image \
  --severity CRITICAL \
  --ignore-unfixed \
  --exit-code 1 \
  myapp:fixed
```

### GitHub Actions example

```yaml
- name: Scan image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    severity: CRITICAL
    ignore-unfixed: true
    exit-code: 1
```

### Keep base images fresh automatically

- Enable Dependabot / Renovate for `FROM` lines and lockfiles
- Rebuild on a schedule (weekly) even without code changes — CVE fixes land in base images continuously
- Rescan images already in your registry, not just at build time

## Decision Flow

```
Scan image  ──►  Any fixable CRITICALs?
                        │
            ┌───────────┴───────────┐
           Yes                      No
            │                        │
   OS package?  App dep?      Any unfixable CRITICALs?
     │            │                  │
  apt/apk      npm/pip/go       ┌────┴────┐
  upgrade      upgrade         Yes        No
     │            │             │          │
     └─────┬──────┘        Assess +     ✅ Done
           │               mitigate /
    Still present?         suppress +
           │               document
    Switch base image
    (slim / distroless)
```

## Quick Reference

| Goal | Command |
|------|---------|
| Show only fixable CRITICALs | `trivy image --severity CRITICAL --ignore-unfixed IMAGE` |
| Recommend a better base image | `docker scout recommendations IMAGE` |
| Fail CI on fixable CRITICALs | `trivy image --severity CRITICAL --ignore-unfixed --exit-code 1 IMAGE` |
| Patch all OS packages (Debian) | `apt-get update && apt-get upgrade -y` |
| Patch all OS packages (Alpine) | `apk update && apk upgrade --no-cache` |
| Fix npm CVEs | `npm audit fix` |
| Suppress an unfixable CVE | Add to `.trivyignore` with a reason + review date |

## Key Takeaways

- `--ignore-unfixed` separates the actionable CVEs from the noise. Start there.
- Most CRITICALs disappear by pulling a fresh base image or switching to a slim/distroless variant.
- OS CVEs are fixed with package managers; app CVEs are fixed in your lockfile.
- Not every CVE is fixable by you — when it isn't, assess exploitability, mitigate at runtime, and document the suppression with a review date.
- Automate rescanning and base image updates so fixes stay fixed.
