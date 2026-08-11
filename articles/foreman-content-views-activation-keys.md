# Foreman/Katello: Content Views and Activation Keys Strategy

## The Problem

When managing content in Foreman/Katello, you need to decide how to structure:

- **Content Views (CV)** — what content is grouped together
- **Composite Content Views (CCV)** — how CVs are combined
- **Activation Keys (AK)** — how hosts are subscribed to content

There is no single right answer — it depends on your organization, number of OS types, applications, and update cycles.

## Key Concepts

| Concept | Purpose |
|---------|---------|
| Content View (CV) | A curated set of repositories that belong together (same update cycle) |
| Composite Content View (CCV) | Combines multiple CVs into one consumable view |
| Activation Key (AK) | Configures what content a host can access (subscriptions + enabled repos) |
| Lifecycle Environment | Promotion path for content (Library → Dev → Production) |

## Content View Design Guidelines

### One CV Per Product

Group repositories that share the same update cycle into a single Content View:

```
CV: CentOS-8
  ├── CentOS 8 - BaseOS
  ├── CentOS 8 - AppStream
  └── CentOS 8 - Extras

CV: EPEL
  └── EPEL 8

CV: Docker
  └── Docker CE Stable

CV: Custom-RPMs
  └── Internal RPM repository
```

**Why?** You cannot easily update parts of a CV without updating all content. Each publish refreshes everything in that CV. If you combine repos with different update intervals (e.g., CentOS + EPEL), you're forced to publish both whenever either one changes.

### When to Use Separate CVs

Use separate CVs when:

- Products have different update cycles (OS monthly, EPEL weekly, custom RPMs ad-hoc)
- Different teams manage different content
- You need different filtering rules per product
- You want independent version control per product

### When to Combine Repos in One CV

Combine repos in one CV when:

- They are part of the same product (BaseOS + AppStream + Extras)
- They always update together
- They are tightly coupled (dependencies between them)

## Composite Content Views

CCVs combine individual CVs into a single view that hosts can subscribe to:

```
CCV: Production-Servers
  ├── CV: RHEL-8          (version 3.0)
  ├── CV: EPEL            (version 5.0)
  ├── CV: Custom-RPMs     (version 12.0)
  └── CV: Monitoring      (version 2.0)

CCV: Docker-Hosts
  ├── CV: RHEL-8          (version 3.0)
  ├── CV: Docker          (version 4.0)
  └── CV: Custom-RPMs     (version 12.0)
```

**Advantages:**

- Each CV can be published independently
- The CCV pins specific versions of each CV
- Promote the CCV through lifecycle environments as one unit
- Different host types get different combinations

**Trade-off:** More overhead when updating — you need to publish the CCV after updating any component CV.

## Activation Key Strategy

### Option 1: One AK per CCV (Simple)

```
AK: production-servers  → CCV: Production-Servers, Lifecycle: Production
AK: docker-hosts        → CCV: Docker-Hosts, Lifecycle: Production
AK: dev-servers         → CCV: Production-Servers, Lifecycle: Development
```

Register hosts with a single key:

```bash
subscription-manager register --activationkey="production-servers" --org="ACME"
```

### Option 2: Multiple AKs (Flexible)

Use one "base" AK to set the CCV and lifecycle environment, then additional AKs to enable/disable specific repos:

```
AK: rhel8-production     → Sets CCV + Lifecycle Environment
AK: enable-epel          → Enables EPEL repository
AK: enable-docker        → Enables Docker repository
AK: enable-monitoring    → Enables monitoring agent repo
```

Register hosts with multiple keys (comma-separated):

```bash
subscription-manager register --activationkey="rhel8-production,enable-epel,enable-monitoring" --org="ACME"
```

**Important:** Only one AK should set the Content View and Lifecycle Environment. Using multiple AKs with different CV/lifecycle settings on the same host will result in unexpected behavior.

**Advantages of multiple AKs:**

- Fewer total AKs needed (compose on the fly)
- Easy to add/remove repos per host type
- Base key is reusable across many configurations

**Disadvantage:**

- Can be confusing how they compose together
- Must be careful only one sets the CV/lifecycle

### Recommendation

| Scenario | Approach |
|----------|----------|
| Simple (few host types, one OS) | One CV + one AK |
| Medium (multiple apps, same OS) | CCVs + one AK per CCV |
| Complex (many host types, mix of repos) | CCVs + multiple AKs (base + repo-enabling) |

## Practical Example

### Setup

Your environment has:

- RHEL 8 (all servers)
- EPEL (most servers, but not all)
- Docker (only container hosts)
- Custom RPMs (all servers)
- Monitoring agent (all servers)

### Content Views

```bash
# Create individual CVs
hammer content-view create --name "RHEL8" --organization "ACME"
hammer content-view create --name "EPEL8" --organization "ACME"
hammer content-view create --name "Docker" --organization "ACME"
hammer content-view create --name "Custom-RPMs" --organization "ACME"
hammer content-view create --name "Monitoring" --organization "ACME"

# Add repos to each CV
hammer content-view add-repository --name "RHEL8" --organization "ACME" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --repository "Red Hat Enterprise Linux 8 for x86_64 - BaseOS RPMs 8"

hammer content-view add-repository --name "RHEL8" --organization "ACME" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --repository "Red Hat Enterprise Linux 8 for x86_64 - AppStream RPMs 8"

# Publish each CV
hammer content-view publish --name "RHEL8" --organization "ACME"
hammer content-view publish --name "EPEL8" --organization "ACME"
hammer content-view publish --name "Docker" --organization "ACME"
hammer content-view publish --name "Custom-RPMs" --organization "ACME"
hammer content-view publish --name "Monitoring" --organization "ACME"
```

### Composite Content Views

```bash
# CCV for standard servers (RHEL + EPEL + Custom + Monitoring)
hammer content-view create --name "Standard-Servers" --organization "ACME" --composite
hammer content-view component add --content-view "Standard-Servers" --organization "ACME" \
  --component-content-view "RHEL8" --latest
hammer content-view component add --content-view "Standard-Servers" --organization "ACME" \
  --component-content-view "EPEL8" --latest
hammer content-view component add --content-view "Standard-Servers" --organization "ACME" \
  --component-content-view "Custom-RPMs" --latest
hammer content-view component add --content-view "Standard-Servers" --organization "ACME" \
  --component-content-view "Monitoring" --latest
hammer content-view publish --name "Standard-Servers" --organization "ACME"

# CCV for docker hosts (RHEL + Docker + Custom + Monitoring)
hammer content-view create --name "Docker-Hosts" --organization "ACME" --composite
hammer content-view component add --content-view "Docker-Hosts" --organization "ACME" \
  --component-content-view "RHEL8" --latest
hammer content-view component add --content-view "Docker-Hosts" --organization "ACME" \
  --component-content-view "Docker" --latest
hammer content-view component add --content-view "Docker-Hosts" --organization "ACME" \
  --component-content-view "Custom-RPMs" --latest
hammer content-view component add --content-view "Docker-Hosts" --organization "ACME" \
  --component-content-view "Monitoring" --latest
hammer content-view publish --name "Docker-Hosts" --organization "ACME"
```

### Activation Keys

```bash
# Base key for standard servers
hammer activation-key create \
  --name "standard-production" \
  --organization "ACME" \
  --lifecycle-environment "Production" \
  --content-view "Standard-Servers" \
  --unlimited-hosts

# Base key for docker hosts
hammer activation-key create \
  --name "docker-production" \
  --organization "ACME" \
  --lifecycle-environment "Production" \
  --content-view "Docker-Hosts" \
  --unlimited-hosts
```

### Register Hosts

```bash
# Standard server
subscription-manager register --activationkey="standard-production" --org="ACME"

# Docker host
subscription-manager register --activationkey="docker-production" --org="ACME"
```

## Content View Filters

Use filters when you need to restrict specific packages within a CV (e.g., pin Docker to a specific version):

```bash
# Create a filter to include only specific Docker version
hammer content-view filter create \
  --name "Docker CE 24.x Only" \
  --content-view "Docker" \
  --organization "ACME" \
  --type rpm \
  --inclusion true

hammer content-view filter rule create \
  --content-view-filter "Docker CE 24.x Only" \
  --content-view "Docker" \
  --organization "ACME" \
  --name "docker-ce" \
  --version "24.*"
```

## Update Workflow

When new content is available:

```bash
# 1. Sync repositories
hammer product synchronize --name "Red Hat Enterprise Linux for x86_64" --organization "ACME" --async

# 2. Publish new CV version
hammer content-view publish --name "RHEL8" --organization "ACME"

# 3. Publish new CCV version (picks up latest CV versions)
hammer content-view publish --name "Standard-Servers" --organization "ACME"

# 4. Promote through lifecycle environments
VERSION=$(hammer --csv content-view version list \
  --content-view "Standard-Servers" --organization "ACME" \
  --order "version DESC" --per-page 1 | tail -1 | cut -d, -f3)

hammer content-view version promote \
  --content-view "Standard-Servers" \
  --organization "ACME" \
  --to-lifecycle-environment "Development" \
  --version "$VERSION"

# 5. After testing, promote to Production
hammer content-view version promote \
  --content-view "Standard-Servers" \
  --organization "ACME" \
  --to-lifecycle-environment "Production" \
  --version "$VERSION"
```

## Summary

| Guideline | Reason |
|-----------|--------|
| One CV per product/update-cycle | Avoids publishing content that hasn't changed |
| Use CCVs to combine CVs for host types | Independent versioning, flexible composition |
| One AK sets CV + lifecycle | Avoid conflicts from multiple AKs setting different views |
| Additional AKs enable/disable repos | Composable, fewer total keys needed |
| Use filters for package version pinning | Don't create separate CVs just for version differences |
| Promote CCVs through environments | Test in Dev before reaching Production |
