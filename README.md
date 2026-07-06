<div align="center">

<h1>📚 Cheatsheets & Articles</h1>

<p>Quick reference guides, command tables, and longer-form write-ups covering Kubernetes, container runtimes, networking, terminal tooling, and infrastructure as code — all tailored to a self-hosted homelab environment.</p>

<br/>

</div>

## About

This is a personal wiki/knowledge base for tools, commands, and configurations used across a self-hosted homelab environment — spanning Kubernetes, container runtimes, networking, terminal tooling, and infrastructure as code. Each guide is written as a standalone Markdown reference, aimed at fast lookup rather than deep tutorials.

- **Author:** Daniel Corneschi
- **Site:** Built with [docsify](https://docsify.js.org/), hosted on [GitHub Pages](https://pages.github.com/)
- **Format:** Every cheatsheet lives in `cheatsheets/<tool>/README.md` with an accompanying `images/` folder for logos and diagrams
- **Scope:** Personal use — commands and examples reflect this homelab's specific setup (namespaces, socket names, config paths) and may need adjusting for other environments

## What's Inside

Each cheatsheet is a self-contained reference guide with installation instructions, common commands, practical examples, and troubleshooting tips.

| Cheatsheet | Description |
|------------|--------------|
| [crictl](cheatsheets/crictl/) | CLI for inspecting and debugging container runtimes (containerd, CRI-O) at the CRI level. |
| [Helm](cheatsheets/helm/) | Package manager for Kubernetes — repositories, chart installs, upgrades, rollbacks, and releases. |
| [k9s](cheatsheets/k9s/) | Terminal UI for navigating, observing, and managing Kubernetes clusters. |
| [Kitty](cheatsheets/kitty/) | GPU-accelerated terminal emulator — tabs, windows, layouts, and configuration. |
| [ss](cheatsheets/ss/) | Socket statistics tool for inspecting TCP/UDP connections, states, and troubleshooting network issues. |
| [Terraform](cheatsheets/terraform/) | Infrastructure as Code tool — core commands, state management, workspaces, and variable handling. |
| [tmux](cheatsheets/tmux/) | Terminal multiplexer for sessions, windows, panes, and copy mode. |

## Articles

Longer-form write-ups on specific concepts, going deeper than a quick reference.

| Article | Description |
|---------|--------------|
| [Getting Started with Argo CD](articles/getting-started-argo.md) | Deploying your first application with ArgoCD — installing ArgoCD, understanding Application manifests, and deploying the Metrics Server via Helm. |
| [Kubernetes emptyDir Volumes](articles/kubernetes-emptyDir-volumes.md) | How `emptyDir` volumes work, sharing files between containers in a Pod, and memory-backed volumes. |
| [Fix Gitea Runner Docker Hub Rate Limits](articles/gitea-runner-fix.md) | Mounting a Docker config file into the runner container to authenticate pulls and avoid rate limiting. |
| [dbash — Docker Shell Function](articles/dbash-function.md) | A simple bash function to quickly access shell environments in Docker containers with automatic shell detection. |
| [macOS Bash Upgrade Guide](articles/macos-bash-upgrade-guide.md) | How to install and use a newer version of bash on macOS, which ships with the outdated 3.2.x due to licensing. |
| [Remove .DS_Store from Git](articles/remove-ds-store-guide.md) | How to remove .DS_Store files from your git repo and prevent them from being tracked in the future. |
| [sed Replace Line Guide](articles/sed-replace-line-guide.md) | How to use `sed` to replace entire lines based on a string match. |
| [Vim White Spaces](articles/vim-white-spaces.md) | Configuring vim to show white spaces with custom symbols. |
| [Ingress for Kubernetes Dashboard on MicroK8s](articles/ingress-kubernetes-dashboard-microk8s.md) | Exposing the Kubernetes Dashboard through an Ingress resource on MicroK8s with NGINX annotations and token auth. |
| [Ingress with MetalLB on MicroK8s](articles/ingress-metallb-microk8s-guide.md) | Setting up NGINX Ingress with MetalLB for bare-metal load balancing on MicroK8s. |
| [NFS Storage for MicroK8s](articles/nfs-microk8s-installation.md) | Installing the NFS CSI driver on MicroK8s for persistent volumes backed by a network NFS server. |
| [User Administration on RHEL](articles/user-administration.md) | Linux user and group management — UID ranges, password hashing, chage, PAM lockout, and changes across RHEL 6 to 10. |

## Usage

This site is built with [docsify](https://docsify.js.org/) and served via GitHub Pages. Browse the sidebar or click a cheatsheet above to get started. Search is available in the sidebar to quickly find specific commands across all guides.
