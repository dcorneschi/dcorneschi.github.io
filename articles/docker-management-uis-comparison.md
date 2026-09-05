# Docker Management UIs: Portainer vs Dockge vs Dockhand and Others

A field guide to the self-hosted web UIs for managing Docker — Portainer, Dockge, Dockhand, Arcane, Komodo, Sencho, and Yacht. What each one is good at, how they differ, and how to pick.

## Overview

| | Portainer | Dockge | Dockhand | Arcane | Komodo | Sencho | Yacht |
|---|---|---|---|---|---|---|---|
| **Primary focus** | Full Docker/Kubernetes management | Compose stack manager | Security-focused Docker UI | Modern Docker/Compose UI | Build + deploy across many servers | Compose fleet cockpit | Template-driven container UI |
| **Scope** | Very broad | Narrow (Compose only) | Broad | Broad | Very broad (CI/CD-ish) | Compose-focused | Narrow |
| **Language/stack** | Go + Angular | Node.js | Bun + SvelteKit | Go + SvelteKit | Rust + React | Web app + Pilot agent | Python + Vue |
| **Multi-host** | Yes (agent/Edge) | No | Yes | Yes (agent) | Yes (Periphery agents) | Yes (Pilot agent) | Limited/planned |
| **Compose files as YAML on disk** | No (stored internally) | Yes | Yes | Yes | Yes (Git-backed) | Yes | No |
| **Kubernetes** | Yes | No | No | No | No | No | No |
| **Vulnerability scanning** | Business edition | No | Built-in | Built-in (Trivy) | No | No | No |
| **Git / GitOps deploys** | Yes | No | Yes (webhooks) | Yes (repo sync) | Yes (core feature) | No | No |
| **SSO / OIDC** | Business edition | No | Free | Yes (RBAC) | Yes | Yes (OIDC/LDAP) | No |
| **Private registries** | Yes | Via Compose | Yes | Yes | Yes | Yes | No |
| **License** | Zlib (CE) / commercial (BE) | AGPL-3.0 | Source-available | BSD-3-Clause | GPL-3.0 | Source-available (pre-1.0) | MIT |
| **Best for** | Teams, mixed Docker + K8s | Compose-only homelabs | Security-conscious homelabs/prod | Portainer replacement | Multi-server build/deploy | Compose fleets without SSH | Simple, template-first setups |

Facts checked against project and vendor documentation as of early 2026; versions and feature tiers change often. Content was rephrased for compliance with licensing restrictions.

## Portainer

The long-standing default. Portainer wraps the full Docker surface — containers, images, volumes, networks, and stacks — plus Docker Swarm and Kubernetes, in one web UI. It manages multiple hosts through an agent (and Edge agents for remote/NAT'd machines), and adds RBAC, registries, and app templates.

- **Editions:** Community Edition (CE, free, Zlib license) and Business Edition (BE, commercial). Several features people want in a team setting — richer RBAC, OAuth/OIDC SSO, and registry/image features — live in BE.
- **Strengths:** breadth, maturity, huge community, Kubernetes support, and the only tool here that treats Swarm and K8s as first-class.
- **Trade-offs:** Compose stacks are stored inside Portainer's own database rather than as plain files on disk, so editing them outside the UI is awkward. The heaviest option to run, and the most valuable features are gated behind BE.

```bash
docker run -d -p 9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Reach for Portainer when you manage mixed Docker + Kubernetes environments, need multi-host from day one, or want a proven tool for a team.

## Dockge

Created by Louis Lam (the developer behind Uptime Kuma), Dockge does one thing well: managing Docker Compose stacks through a clean web UI. You create, edit, start, stop, and update stacks, and the files live on disk as standard `compose.yaml` you can drive from the CLI directly.

- **Strengths:** simple, fast, actively maintained, and honest about scope. The interactive editor and real-time output are genuinely nice. Because files stay as plain YAML, there is no lock-in.
- **Trade-offs:** single-host, Compose-only. No container-level management beyond stacks, no images/volumes browser to speak of, no Kubernetes, no built-in auth beyond a single admin account.

```yaml
services:
  dockge:
    image: louislam/dockge:1
    restart: unless-stopped
    ports:
      - 5001:5001
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/app/data
      - /opt/stacks:/opt/stacks
    environment:
      - DOCKGE_STACKS_DIR=/opt/stacks
```

Reach for Dockge when your entire workflow is Compose on a single host and you want the lightest, cleanest option.

## Dockhand

A newer, security-focused Docker management platform built on Bun and SvelteKit. Dockhand aims to fold several separate homelab tools into one: real-time container management, Compose stacks, Git deployments with webhooks, multi-host support, and — notably — built-in vulnerability scanning and free OIDC/SSO. It runs on SQLite by default and is light enough for a Raspberry Pi.

- **Strengths:** the security features (image CVE scanning, SSO) are in the free tier rather than gated. Compose files stay as real files, multi-host is supported, and it's fast in day-to-day use. Some users retire Watchtower, Dozzle, or Diun after adopting it.
- **Trade-offs:** young project, so the ecosystem and long-term track record are still forming. Source-available rather than a permissive OSI license. No Kubernetes.

Reach for Dockhand when you want an all-in-one Docker UI with security scanning and SSO without paying for an enterprise tier.

## Arcane

A modern Docker and Compose management GUI with a Go backend and a Svelte 5 frontend, plus an optional headless agent for driving other hosts. Arcane covers containers, images, volumes, networks, and Compose projects, and adds live log streaming, an in-browser exec shell, Trivy vulnerability scanning, image update tracking, RBAC, and a fleet view.

- **Strengths:** the mental model is close to Portainer, but Compose files are kept as real files on disk, the vulnerability scanner is built in (not a paid tier), and it's BSD-3-Clause with no feature gating. Single Go binary shipped as a container.
- **Trade-offs:** no Kubernetes, and like Dockhand it's a comparatively young project still building community and stability.

Reach for Arcane if you like Portainer's breadth but want an open license, on-disk Compose files, and free vulnerability scanning.

## Komodo

Different category. Komodo (Rust core, React UI) is a self-hosted platform for building and deploying software across many servers, closer to a lightweight GitOps/CI-CD system than a plain dashboard. A central Core service talks to remote **Periphery** agents on each server. It handles auto-versioned image builds from Git via webhooks, Compose deployments (from the UI, host, or Git), server stats, procedures/automation, resource-scoped permissions, and even Docker Swarm.

- **Strengths:** genuine multi-server build-and-deploy with Git as the source of truth, an API-first design, and strong automation. If you outgrow "just manage containers" and want repeatable deployments, this is the tool.
- **Trade-offs:** more moving parts (Core + agents + a database), a steeper learning curve, and overkill for a single host or a handful of stacks. No Kubernetes.

Reach for Komodo when you're deploying across several servers and want Git-driven builds and deployments rather than a point-and-click dashboard.

## Sencho

A self-hosted Docker Compose management platform from Studio Saelix, aimed at homelab operators, small DevOps teams, and platform engineers. Sencho gives you one browser workspace for the Compose work you'd otherwise do over SSH: deploying stacks, editing files, watching logs, restarting containers, browsing volumes, and recovering from failures. It runs as a single container, and your Compose files stay on the host filesystem as the source of truth — each subdirectory under a Compose directory (for example `/opt/compose`) becomes a stack.

- **Strengths:** file-on-disk Compose workflow with no lock-in, a clean point-and-click interface (right-click context menus grouped by inspect/organize/lifecycle/destructive actions), OIDC and LDAP SSO, private registry credential injection, and multi-host management via a **Pilot agent** — a container you run on the remote machine that the primary instance connects to over HTTPS. It also documents running behind a Docker socket proxy for a tighter security posture.
- **Trade-offs:** Compose-focused, so no Kubernetes and no container-from-image workflow outside stacks. It's a pre-1.0 project that evolves quickly, so validate against your own setup before trusting it with critical infrastructure. No built-in vulnerability scanning.

Reach for Sencho when your workflow is Compose across one or more hosts, you want the files to stay on disk, and you value SSO and a socket-proxy-friendly design without stepping up to a full build/deploy system like Komodo.

## Yacht

A lightweight container UI positioned as a simpler alternative to Portainer, built around app templates as its differentiator. It deploys containers from templates, manages running containers, and offers basic monitoring, with support for Docker and Podman.

- **Strengths:** simple and template-first, which makes one-click deploys of common apps easy.
- **Trade-offs:** development has been largely stalled since 2023, it still carries a `0.0.x` version, it lacks proper Compose support, and its resource use is high for its feature set. Most comparisons now favor Dockge or Portainer over it.

Reach for Yacht only if template-based single-click deploys are your main need and you accept the slow development pace. For most people, Dockge or Arcane is the better lightweight pick today.

## Decision Guide

- **You run Kubernetes (or Swarm) alongside Docker** → Portainer. It's the only option here with first-class K8s.
- **Your whole life is Compose on one host** → Dockge. Cleanest, lightest, files stay as YAML.
- **You want security scanning + SSO for free, all-in-one** → Dockhand.
- **You want Portainer's breadth with an open license and on-disk Compose** → Arcane.
- **You deploy across many servers and want Git-driven builds/deploys** → Komodo.
- **You run Compose across one or more hosts, want files on disk plus SSO, but not a full CI system** → Sencho.
- **You just want template-based one-click app deploys** → Yacht (but check Dockge first).

## A Note on the Docker Socket

Every one of these tools works by talking to the Docker Engine API, almost always by mounting `/var/run/docker.sock`. That socket is effectively root on the host: anything with access to it can start privileged containers and take over the machine. Treat any UI with socket access as a root-equivalent service.

- Put the UI behind a reverse proxy with TLS and authentication; don't expose it raw to the internet.
- Prefer tools with real RBAC/SSO if more than one person uses it.
- Consider a socket proxy (for example, a read-only or scoped proxy in front of the socket) to limit what the UI can do.
- For remote hosts, prefer the tool's agent (Portainer agent, Arcane agent, Komodo Periphery, Sencho Pilot) over exposing the Docker daemon on a TCP port.

## Summary

Portainer remains the safe, broad default, especially with Kubernetes in the mix, though its best features live in the paid Business Edition. Dockge is the standout for Compose-only single-host setups. Dockhand and Arcane are the strong modern challengers — both keep Compose as files on disk and ship vulnerability scanning and SSO without a paywall, with Arcane's permissive BSD license being a point in its favor. Komodo is a step up into multi-server, Git-driven build-and-deploy. Sencho sits between Dockge and Komodo — a Compose-first cockpit with on-disk files, SSO, and multi-host Pilot agents, but no Kubernetes or build pipeline. Yacht is the legacy lightweight option that most people have moved past.
