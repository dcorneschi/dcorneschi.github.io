<div align="center">

<img src="articles/images/homelab-wiki-logo.svg" alt="corneschi.ro" width="700" />

<p>Quick reference guides, command tables, and longer-form write-ups covering Kubernetes, container runtimes, networking, terminal tooling, and infrastructure as code — all tailored to a self-hosted homelab environment.</p>

<br/>

</div>

## Usage

This site is built with [docsify](https://docsify.js.org/) and served via GitHub Pages. Browse the sidebar or click a cheatsheet above to get started. Search is available in the sidebar to quickly find specific commands across all guides.

## What's Inside

Each cheatsheet is a self-contained reference guide with installation instructions, common commands, practical examples, and troubleshooting tips.

| Cheatsheet | Description |
|------------|--------------|
| [crictl](cheatsheets/crictl/) | CLI for inspecting and debugging container runtimes (containerd, CRI-O) at the CRI level. |
| [Helm](cheatsheets/helm/) | Package manager for Kubernetes — repositories, chart installs, upgrades, rollbacks, and releases. |
| [iotop](cheatsheets/iotop/) | Interactive I/O monitoring tool — per-process disk read/write usage, batch logging, and troubleshooting. |
| [k9s](cheatsheets/k9s/) | Terminal UI for navigating, observing, and managing Kubernetes clusters. |
| [Kitty](cheatsheets/kitty/) | GPU-accelerated terminal emulator — tabs, windows, layouts, and configuration. |
| [ps](cheatsheets/ps/) | Process status tool — listing, filtering, and inspecting running processes, resource usage, and process trees. |
| [ss](cheatsheets/ss/) | Socket statistics tool for inspecting TCP/UDP connections, states, and troubleshooting network issues. |
| [Terraform](cheatsheets/terraform/) | Infrastructure as Code tool — core commands, state management, workspaces, and variable handling. |
| [tmux](cheatsheets/tmux/) | Terminal multiplexer for sessions, windows, panes, and copy mode. |

## Articles

Longer-form write-ups on specific concepts, going deeper than a quick reference.

### Kubernetes

| Article | Description |
|---------|--------------|
| [Getting Started with Argo CD](articles/getting-started-argo.md) | Deploying your first application with ArgoCD — installing ArgoCD, understanding Application manifests, and deploying the Metrics Server via Helm. |
| [Kubernetes imagePullPolicy](articles/kubernetes-imagepullpolicy.md) | How `imagePullPolicy` controls image pulling behavior — policy values, defaults, digests vs tags, private registries, parallel pulls, and troubleshooting. |
| [Kubernetes emptyDir Volumes](articles/kubernetes-emptyDir-volumes.md) | How `emptyDir` volumes work, sharing files between containers in a Pod, and memory-backed volumes. |
| [Kubernetes PriorityClasses Guide](articles/kubernetes-priority-classes-guide.md) | How PriorityClasses control scheduling and preemption — recommended tiers, preemptionPolicy, globalDefault, and common pitfalls. |
| [Ingress for Kubernetes Dashboard on MicroK8s](articles/ingress-kubernetes-dashboard-microk8s.md) | Exposing the Kubernetes Dashboard through an Ingress resource on MicroK8s with NGINX annotations and token auth. |
| [Ingress with MetalLB on MicroK8s](articles/ingress-metallb-microk8s-guide.md) | Setting up NGINX Ingress with MetalLB for bare-metal load balancing on MicroK8s. |
| [NFS Storage for MicroK8s](articles/nfs-microk8s-installation.md) | Installing the NFS CSI driver on MicroK8s for persistent volumes backed by a network NFS server. |

### Terraform

| Article | Description |
|---------|--------------|
| [terraform.tfstate vs .terraform/ State](articles/terraform-tfstate-vs-terraform-directory-state.md) | The difference between `terraform.tfstate` and `.terraform/terraform.tfstate` — state file vs backend cache, `init -reconfigure`, state locking, recovery, and recommended `.gitignore`. |
| [terraform init -upgrade and Constraints](articles/terraform-init-upgrade-and-constraints.md) | How `terraform init -upgrade` works, version constraint syntax (`~>`, `>=`, `=`), when to tighten or loosen constraints, the lock file workflow, and common pitfalls. |
| [terraform get -update vs init -upgrade](articles/terraform-get-update-vs-init-upgrade.md) | The difference between `terraform get -update` and `terraform init -upgrade` — scope, provider handling, performance, and when to use each command. |
| [Terraform Lock File Checksums: zh and h1](articles/terraform-lock-file-checksums.md) | How `zh:` and `h1:` hashes in `.terraform.lock.hcl` work — calculating the dirhash, where `zh` hashes originate, multi-platform hashes, security model, and troubleshooting mismatches. |

### Misc

| Article | Description |
|---------|--------------|
| [Fix Gitea Runner Docker Hub Rate Limits](articles/gitea-runner-fix.md) | Mounting a Docker config file into the runner container to authenticate pulls and avoid rate limiting. |
| [dbash — Docker Shell Function](articles/dbash-function.md) | A simple bash function to quickly access shell environments in Docker containers with automatic shell detection. |
| [Bash Troubleshooting Guide](articles/bash-troubleshooting-guide.md) | Bash debugging and troubleshooting — `set -x`, `set -n`, `set -euo pipefail`, PS4 customization, tracing functions, and practical examples. |
| [macOS Bash Upgrade Guide](articles/macos-bash-upgrade-guide.md) | How to install and use a newer version of bash on macOS, which ships with the outdated 3.2.x due to licensing. |
| [Remove .DS_Store from Git](articles/remove-ds-store-guide.md) | How to remove .DS_Store files from your git repo and prevent them from being tracked in the future. |
| [sed Replace Line Guide](articles/sed-replace-line-guide.md) | How to use `sed` to replace entire lines based on a string match. |
| [Vim White Spaces](articles/vim-white-spaces.md) | Configuring vim to show white spaces with custom symbols. |
| [Linux File Permissions Guide](articles/linux-file-permissions.md) | Linux file permissions and ownership — standard permissions, umask, special permissions (SUID, SGID, sticky bit), ACLs, precedence, and practical examples. |
| [RHEL Releases Overview](articles/rhel-releases-overview.md) | Comprehensive overview of RHEL major releases (2.1–10) — release dates, codenames, kernels, key features, lifecycle/EOL dates, and in-place upgrade paths with Leapp. |
| [User Administration on RHEL](articles/user-administration.md) | Linux user and group management — UID ranges, password hashing, chage, PAM lockout, and changes across RHEL 6 to 10. |
| [Configuring sysstat on Ubuntu](articles/configuring-sysstat-ubuntu.md) | Installing and configuring sysstat on Ubuntu 22.04 and 24.04 — enabling collection, setting 1-minute intervals via systemd timers, and data retention. |
| [JetBrains Mono Font](articles/jetbrains-mono-font.md) | JetBrains Mono — a free, open-source monospaced font designed for terminals and code editors with improved readability. |

## About

This is a personal wiki/knowledge base for tools, commands, and configurations used across a self-hosted homelab environment — spanning Kubernetes, container runtimes, networking, terminal tooling, and infrastructure as code. A collection of all commands and knowledge gathered over the last 15+ years, written as standalone Markdown references aimed at fast lookup rather than deep tutorials.

- **Author:** Daniel Corneschi
- **Site:** Built with [docsify](https://docsify.js.org/), hosted on [GitHub Pages](https://pages.github.com/)
- **Format:** Every cheatsheet lives in `cheatsheets/<tool>/README.md` with an accompanying `images/` folder for logos and diagrams
- **Scope:** Personal use — commands and examples reflect this homelab's specific setup (namespaces, socket names, config paths) and may need adjusting for other environments
