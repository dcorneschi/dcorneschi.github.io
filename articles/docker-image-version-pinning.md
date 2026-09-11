# Pin Docker Image Versions Instead of latest

Using the `latest` tag on container images makes deployments unpredictable, hard to roll back, and difficult to audit. Pinning to explicit versions — and, for the strongest guarantee, to image digests — gives you reproducible, controlled, and traceable deployments.

This guide explains why `latest` is risky, how to pin versions in Docker Compose and Dockerfiles, and how to run a safe update workflow.

> **Key nuance:** A version tag such as `1.30.5` is far better than `latest`, but tags are still mutable — a registry can re-point them to a different build. For true reproducibility, pin by digest (`@sha256:...`) or enable tag immutability on your registry.

## Why latest Is a Problem

`latest` is just a conventional tag name with no special meaning to the registry. It is not guaranteed to be the newest release, and it can point to a different image over time.

### Security

- You cannot be certain which version is actually running.
- An image can change under you, silently introducing new code or vulnerabilities.
- Vulnerability tracking and patching are harder when the running version is ambiguous.
- Scanners cannot tie results to a fixed, known artifact.

### Operations

- A container recreate or a new node can pull a different image for the same tag.
- Rollback is difficult because there is no distinct previous version to return to.
- The same Compose file can deploy different software on different days or hosts.
- Debugging is harder when you cannot state the exact version in play.

### Compliance

- Audit trails of what ran in production are incomplete.
- Change-control and security-compliance requirements are harder to satisfy.
- Systematic vulnerability management is nearly impossible without fixed versions.

## Tags Versus Digests

| Reference | Mutable | Example | Reproducible |
|-----------|---------|---------|--------------|
| `latest` | Yes | `vaultwarden/server:latest` | No |
| Version tag | Yes | `vaultwarden/server:1.30.5` | Mostly, unless the tag is re-pushed |
| Digest | No | `vaultwarden/server@sha256:2b0c...` | Yes, always the same bytes |

A tag is a human-friendly label. A digest is a cryptographic hash of the exact image content, so it always resolves to the same image. For related Kubernetes-side detail, see [Finding the Real Image Version Behind a latest Tag](articles/kubernetes-resolve-running-image-version.md) and [Kubernetes imagePullPolicy Explained](articles/kubernetes-imagepullpolicy.md).

## Recommended Approach

### Use specific version tags

```yaml
# Bad — using latest
services:
  vaultwarden:
    image: vaultwarden/server:latest

# Good — using a specific version
services:
  vaultwarden:
    image: vaultwarden/server:1.30.5
```

### Pin by digest for maximum reproducibility

Keep the readable tag in a comment and pin the immutable digest:

```yaml
services:
  vaultwarden:
    # vaultwarden/server:1.30.5
    image: vaultwarden/server@sha256:REPLACE_WITH_REAL_DIGEST
```

Find the digest of an image you have pulled and trust:

```bash
# RepoDigests shows the registry digest for the pulled image
docker image inspect vaultwarden/server:1.30.5 \
  --format '{{ index .RepoDigests 0 }}'
```

Use the digest that command prints; do not copy the example placeholder above.

### Benefits of pinning

- Predictable deployments: you know exactly what runs.
- Controlled updates: versions change when you decide, not automatically.
- Easy rollbacks: revert to a known previous version or digest.
- Deliberate security: you choose when to absorb changes and rescan.
- Reproducible environments: the same reference yields the same result.

## Update Workflow

### 1. Inventory what is running

```bash
# Local images and their tags
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.CreatedAt}}"

# Running containers and the image each uses
docker ps --format "table {{.Names}}\t{{.Image}}"

# Exact resolved digests for running containers
docker inspect --format '{{ .Name }} {{ .Image }}' $(docker ps -q)
```

### 2. Research stable versions

For each service, check the official Docker Hub or registry page, the project's GitHub releases and changelog, and any security advisories. Prefer a documented stable release over a pre-release tag.

### 3. Stage and test

- Pin the new version in staging first.
- Test the application and its data/migrations.
- Confirm vulnerability scans pass on the new version.
- Record any breaking changes or migration steps.

### 4. Update production deliberately

Keep the current version active and the candidate version visible but inactive until it is approved:

```yaml
services:
  app:
    image: myapp:1.2.3      # current stable
    # image: myapp:1.2.4    # next version, enabled after testing
```

Apply the change and recreate only the affected service:

```bash
docker compose pull app
docker compose up -d app
```

### 5. Document and track

Keep a changelog of version updates, the reason for each choice, and the date each security patch was deployed. Maintain a written rollback procedure.

## Example: Pinned Compose Services

```yaml
services:
  traefik:
    image: traefik:v3.0.4

  vaultwarden:
    image: vaultwarden/server:1.30.5

  grafana:
    image: grafana/grafana:10.2.3

  portainer:
    image: portainer/portainer-ce:2.19.4
```

Treat these version numbers as examples. Check the current stable release for each project before adopting a value, and move the pin forward deliberately.

## Rollback

Because each version is a distinct reference, rollback is a matter of pointing back to the previous one.

```bash
# Edit the compose file back to the previous version or digest, then:
docker compose up -d app

# Confirm the running image
docker inspect --format '{{ .Config.Image }}' "$(docker compose ps -q app)"
```

Avoid `docker compose down` for a single-service rollback; it stops the whole stack. Recreating just the changed service is less disruptive.

## Monitoring and Maintenance

Establish a cadence rather than reacting only to incidents:

- Weekly vulnerability scans of running images.
- Monthly review of available version updates.
- Quarterly evaluation of major-version upgrades.
- Emergency patching when a critical advisory lands.

### Automation options

- Use [Renovate](https://docs.renovatebot.com/) or Dependabot to open pull requests when a pinned image has a newer version.
- Add image scanning to CI so unpinned or vulnerable images fail the pipeline.
- Alert on critical advisories for images you run.
- Apply updates to staging automatically, but keep production changes gated on review.

Renovate and Dependabot can also maintain digest pins, updating both the digest and the version comment together.

## Emergency Procedures

When a vulnerability is found:

1. Assess severity and exposure for your deployment.
2. Check whether a newer version fixes it.
3. Test the fix in staging immediately.
4. Deploy the patched version to production if the risk warrants it.
5. Record the incident, the version change, and the outcome.

## Scanning Tools

For finding vulnerabilities in the images you pin:

- Trivy
- Docker Scout
- Grype or Anchore
- Snyk

For guidance on remediating scan findings, see [Fixing Critical Vulnerabilities in Public Docker Images](articles/docker-fix-critical-vulnerabilities.md).

## Reference Guidelines

- CIS Docker Benchmark
- NIST SP 800-190, Application Container Security Guide
- OWASP container security guidance
- Docker's official security documentation

## Quick Reference

```bash
# Inventory local images with tags
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.CreatedAt}}"

# Show the registry digest for a pulled, trusted image
docker image inspect nginx:1.27.1 --format '{{ index .RepoDigests 0 }}'

# Pull an exact digest
docker pull nginx@sha256:REPLACE_WITH_REAL_DIGEST

# Update a single pinned service in Compose
docker compose pull app && docker compose up -d app

# Confirm the image a running container uses
docker inspect --format '{{ .Config.Image }}' "$(docker compose ps -q app)"
```

Prefer explicit version tags over `latest`, and pin by digest where reproducibility matters most. See the [Docker Cheatsheet](articles/docker-cheatsheet.md) and [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md) for related commands.
