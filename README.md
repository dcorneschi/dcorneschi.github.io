<div align="center">

<img src="articles/images/homelab-wiki-logo.svg" alt="corneschi.ro" width="700" />

<p>Quick reference guides, command tables, and longer-form write-ups covering Kubernetes, container runtimes, networking, terminal tooling, and infrastructure as code — all tailored to a self-hosted homelab environment.</p>

<br/>

</div>

## Usage

This site is built with [docsify](https://docsify.js.org/) and served via GitHub Pages. Browse the sidebar or click an article to get started. Search is available in the sidebar to quickly find specific commands across all guides.

## What's Inside

### Kubernetes

| Article | Description |
|---------|--------------|
| [Using jq with kubectl](articles/kubectl-jq-guide.md) | Composing jq commands for kubectl JSON output — filtering, aggregation, formatting, and scripts. |
| [kubectl JSONPath Guide](articles/kubectl-jsonpath-guide.md) | Built-in JSONPath expressions — pods, nodes, services, storage, events, and formatting. |
| [Getting Started with Argo CD](articles/getting-started-argo.md) | Installing ArgoCD and deploying your first application. |
| [Kubernetes imagePullPolicy](articles/kubernetes-imagepullpolicy.md) | How `imagePullPolicy` controls image pulling behavior. |
| [Kubernetes emptyDir Volumes](articles/kubernetes-emptyDir-volumes.md) | How `emptyDir` volumes work and sharing files between containers. |
| [Kubernetes PriorityClasses Guide](articles/kubernetes-priority-classes-guide.md) | How PriorityClasses control scheduling and preemption. |
| [Kubernetes QoS Classes — Requests and Limits](articles/kubernetes-qos-requests-limits.md) | QoS classes, eviction order, OOM kill priority, and the CPU limits debate. |
| [Kubernetes Pod Evictions Cheatsheet](articles/kubernetes-evictions-cheatsheet.md) | Eviction methods, PDB respect, graceful termination, and eviction actors. |
| [Kubernetes PodDisruptionBudgets Guide](articles/kubernetes-pdb-guide.md) | How PDBs protect availability during voluntary disruptions. |
| [kubectl run vs kubectl create](articles/kubectl-run-vs-create.md) | When to use `kubectl run` (bare Pods) vs `kubectl create` (other resources). |
| [Init Containers vs Regular Containers](articles/kubernetes-init-vs-regular-containers.md) | Lifecycle differences, kubelet orchestration, cgroup allocation, and use cases. |
| [Ingress for Kubernetes Dashboard on MicroK8s](articles/ingress-kubernetes-dashboard-microk8s.md) | Exposing the Kubernetes Dashboard through Ingress on MicroK8s. |
| [Ingress with MetalLB on MicroK8s](articles/ingress-metallb-microk8s-guide.md) | NGINX Ingress with MetalLB for bare-metal load balancing. |
| [NFS Storage for MicroK8s](articles/nfs-microk8s-installation.md) | NFS CSI driver on MicroK8s for persistent volumes. |
| [Helm Cheatsheet](articles/helm-cheatsheet.md) | Package manager for Kubernetes — repos, installs, upgrades, and rollbacks. |
| [crictl Cheatsheet](articles/crictl-cheatsheet.md) | CLI for inspecting and debugging container runtimes at the CRI level. |
| [ctr Cheatsheet (containerd)](articles/ctr-cheatsheet.md) | containerd CLI — images, containers, tasks, namespaces, snapshots, and Kubernetes debugging. |
| [Kubernetes Schema Validation](articles/kubernetes-schema-validation.md) | Validating K8s manifests — yamllint, kubeconform, kubectl dry-run (client vs server), pluto, and CI/CD strategies. |
| [k9s Cheatsheet](articles/k9s-cheatsheet.md) | Terminal UI for navigating and managing Kubernetes clusters. |
| [Node Selectors in Kubernetes](articles/kubernetes-node-selectors.md) | nodeSelector, node affinity, built-in labels, taints, scheduling constraints, and combining strategies. |
| [Node Affinity in Kubernetes](articles/kubernetes-node-affinity.md) | Required and preferred rules, operators (In, NotIn, Exists, Gt, Lt), weight scoring, combining with taints, and troubleshooting. |
| [Kubernetes Taints and Tolerations](articles/kubernetes-taints-tolerations.md) | Taint effects, keys, values, toleration operators, real-world patterns, removal, monitoring, and validation rules. |
| [Kubernetes Scheduling Deep Dive](articles/kubernetes-scheduling-deep-dive.md) | Full scheduling pipeline — filtering, scoring, preemption, PriorityClasses, topology spread, QoS, NUMA, PDBs, and debugging. |
| [LimitRange and ResourceQuota](articles/kubernetes-limitrange-resourcequota.md) | Namespace resource controls — default limits, per-pod constraints, namespace quotas, scoping by priority class, and enforcement. |
| [Kubernetes Pod Conditions Flow](articles/kubernetes-pod-conditions-flow.md) | Pod lifecycle conditions — PodScheduled, Initialized, ContainersReady, Ready, readiness gates, and troubleshooting stuck pods. |
| [Kubernetes Pod Commands](articles/kubernetes-pod-commands.md) | command vs args — Docker ENTRYPOINT/CMD mapping, YAML syntax styles, shell vs exec form, and signal handling. |
| [Fix DaemonSet Scheduling on EKS](articles/eks-daemonset-scheduling-fix.md) | DaemonSet pods stuck Pending — diagnosis, PriorityClass fix, resource reservation, eviction, and prevention strategies. |
| [Kubernetes Control Plane API Commands](articles/kubernetes-api-commands.md) | Direct API server access — kubectl raw, curl with tokens, API discovery, health endpoints, metrics, and debugging. |
| [kubectl Cheatsheet](articles/kubectl-cheatsheet.md) | kubectl commands — nodes, pods, deployments, services, events, logs, performance, RBAC, custom-columns, and one-liners. |
| [Kubernetes Field Selectors](articles/kubernetes-field-selectors.md) | --field-selector — supported fields per resource type, operators, practical examples, and limitations. |
| [kubectl logs Guide](articles/kubectl-logs-guide.md) | Pod log retrieval — follow, tail, timestamps, previous, multi-container, labels, debug bundles, and troubleshooting. |
| [Kubernetes Log Locations by Distribution](articles/kubernetes-log-locations.md) | Log file paths across MicroK8s, kubeadm, K3s, kind, k3d, minikube, EKS, GKE, AKS — containers, kubelet, control plane, and runtime. |
| [Kubernetes Jobs and CronJobs](articles/kubernetes-jobs-cronjobs.md) | Jobs and CronJobs — commands, cron syntax, concurrency policies, CrashLoopBackOff fix, and auto-cleanup. |
| [EKS Port Communication](articles/eks-port-communication.md) | Control plane to worker node ports — kubelet API, kube-proxy, security group rules, and communication flows. |
| [EKS Node Lifecycle During Updates](articles/eks-node-lifecycle-during-updates.md) | Node state progression during rolling updates — cordon, drain, eviction API, PDBs, ASG integration, and timing. |
| [Kubernetes Cluster Setup with kubeadm](articles/kubeadm-cluster-setup.md) | Step-by-step kubeadm cluster setup — CRI-O, Calico CNI, metrics server, node joins, validation, upgrades, and troubleshooting. |

### Docker

| Article | Description |
|---------|--------------|
| [Docker Cheatsheet](articles/docker-cheatsheet.md) | Docker CLI — containers, images, volumes, networks, Dockerfile, and troubleshooting. |
| [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md) | Docker Compose — services, builds, networks, volumes, profiles, overrides, and patterns. |
| [Docker Compose: ports vs expose](articles/docker-ports-vs-expose.md) | Differences between `ports` and `expose` — when to publish vs keep internal. |
| [Docker Compose: Running Containers Without Root](articles/docker-compose-non-root.md) | UID/GID mapping, bind mount permissions, image inspection, privileged ports, and security hardening. |
| [Fix Gitea Runner Docker Hub Rate Limits](articles/docker-gitea-runner-fix.md) | Mounting Docker config into the runner to avoid rate limiting. |
| [dbash — Docker Shell Function](articles/docker-dbash-function.md) | Bash function to quickly shell into Docker containers. |
| [Building Docker Images with Dockerfile](articles/docker-build-image-guide.md) | Dockerfile instructions, build commands, tagging, multi-stage builds, heredoc syntax, CMD vs ENTRYPOINT, and best practices. |
| [Docker Swarm Cheatsheet](articles/docker-swarm-cheatsheet.md) | Swarm clustering — init, nodes, services, scaling, rolling updates, networks, secrets, configs, stacks, HA, and backups. |
| [Docker Swarm Storage](articles/docker-swarm-storage.md) | Swarm storage strategies — NFS server/client, GlusterFS, Ceph, Docker NFS volumes, stack examples, backups, and monitoring. |

### AWS

| Article | Description |
|---------|--------------|
| [AWS CLI Installation](articles/aws-cli-install.md) | Install AWS CLI v2 on RHEL, Ubuntu, macOS — configuration, profiles, auto-completion, and Docker. |
| [AWS STS Assume Role with MFA](articles/aws-sts-assume-role.md) | Temporary credentials via AssumeRole — MFA enforcement, session scripts, named profiles, duration, and role chaining. |
| [Assume an IAM Role via CLI (Step by Step)](articles/aws-assume-role-cli-walkthrough.md) | Full walkthrough — create user, policy, trust policy, role, assume it, export credentials, and named profile alternative. |
| [AWS IAM Concepts Guide](articles/aws-iam-concepts-guide.md) | IAM fundamentals — roles, policy types, evaluation logic, Identity Center (SSO), federation, permission boundaries, and root vs admin. |
| [AWS IAM CLI Cheatsheet](articles/aws-iam-cheatsheet.md) | All `aws iam` and `aws sts` commands — users, groups, roles, policies, access keys, MFA, simulation, and audit one-liners. |
| [ECS Cluster Architecture](articles/ecs-architecture-guide.md) | ECS internals — Fargate vs EC2, task definitions, services, networking, auto scaling, capacity providers, ECS Exec, and CLI commands. |
| [AWS EFS Cheatsheet](articles/aws-efs-cheatsheet.md) | Elastic File System — create, mount, access points, security, performance modes, lifecycle, ECS/EKS/Lambda integration, and monitoring. |
| [AWS Lightsail Cheatsheet](articles/aws-lightsail-cheatsheet.md) | Simplified compute — instances, static IPs, firewall, snapshots, disks, databases, load balancers, containers, DNS, and export to EC2. |
| [AWS Load Balancer Cheatsheet](articles/aws-elb-cheatsheet.md) | ALB, NLB, CLB — create, target groups, listeners, routing rules, health checks, SSL, access logs, WAF, and common patterns. |
| [AWS VPC Cheatsheet](articles/aws-vpc-cheatsheet.md) | VPC — subnets, route tables, IGW, NAT, security groups, NACLs, endpoints, peering, Transit Gateway, and flow logs. |
| [AWS Route 53 Cheatsheet](articles/aws-route53-cheatsheet.md) | DNS — hosted zones, record types, routing policies, health checks, failover, DNSSEC, resolver, and domain registration. |
| [AWS ECR Cheatsheet](articles/aws-ecr-cheatsheet.md) | Container registry — push/pull, scanning, lifecycle policies, replication, pull-through cache, and repository policies. |
| [EC2 Cheatsheet](articles/aws-ec2-cheatsheet.md) | AWS EC2 — instances, AMIs, security groups, EBS, Elastic IPs, metadata, and CLI patterns. |
| [EC2 Extend EBS Volume](articles/aws-ec2-extend-disk.md) | Resize EBS volumes and grow filesystems (XFS, ext4, LVM) — no downtime required. |
| [EC2 Instance Metadata Service (IMDS)](articles/aws-ec2-metadata.md) | IMDS endpoint, IMDSv1 vs IMDSv2, ec2-metadata script, IAM credentials, tags, spot notices, and configuration. |
| [EC2 vs ELB Health Checks](articles/aws-ec2-vs-elb-health-checks.md) | EC2 status checks vs ELB health checks — what each monitors, how they interact with Auto Scaling, and self-healing patterns. |
| [EBS Cheatsheet](articles/aws-ebs-cheatsheet.md) | EBS volume types, create/attach/resize, snapshots, DLM lifecycle policies, encryption, multi-attach, performance tuning, and monitoring. |
| [S3 Cheatsheet](articles/aws-s3-cheatsheet.md) | S3 CLI — buckets, objects, sync, presigned URLs, lifecycle, storage classes, versioning, encryption, and performance tuning. |
| [Unattached EBS Volumes: Detection and Monitoring](articles/aws-ebs-unattached-volumes.md) | Finding orphaned volumes, cost estimation, safe cleanup workflow, CloudWatch idle detection, AWS Config rules, Lambda automation, and prevention. |
| [EC2 fstab: Why Device Names Change on Nitro](articles/aws-ec2-fstab-labels.md) | NVMe device reordering, using LABEL/UUID in fstab, cloud-init provisioning, and instance store volumes. |
| [Installing SSM Agent](articles/aws-ssm-agent-install.md) | Install and configure SSM Agent on RHEL, Ubuntu — Session Manager, VPC endpoints, and SSH over SSM. |
| [JMESPath Query Guide](articles/aws-jmespath-guide.md) | JMESPath query language for AWS CLI — filtering, sorting, functions, and real-world examples. |
| [EKS Authentication Modes: ConfigMap vs Access Entries](articles/eks-authentication-modes.md) | EKS authentication methods — aws-auth ConfigMap vs API access entries, migration, and best practices. |
| [Securing Kubernetes Containers: Security Contexts](articles/eks-security-contexts.md) | Linux kernel primitives, security contexts, capabilities, seccomp, AppArmor, and production hardening. |
| [EKS Node Groups: With and Without Launch Templates](articles/eks-nodegroups-launch-templates.md) | Managed node groups — when you need a launch template, what EKS manages automatically, custom AMIs, and Terraform examples. |
| [EKS Architecture Deep Dive](articles/eks-architecture-deep-dive.md) | EKS internals — control plane, etcd, cross-account ENIs, VPC CNI, authentication flow, API endpoint access, add-ons, upgrade process, and limits. |
| [eksctl Cheatsheet](articles/eksctl-cheatsheet.md) | eksctl CLI — cluster lifecycle, node groups, scaling, labels, IAM service accounts, OIDC, add-ons, Fargate, and identity mappings. |
| [Creating an EKS Cluster with eksctl](articles/eks-cluster-with-eksctl.md) | Step-by-step guide — prerequisites, ClusterConfig YAML, managed node groups, IRSA, add-ons, private clusters, and production-ready examples. |
| [EKS Auto Mode](articles/eks-auto-mode.md) | Fully managed data plane — what AWS handles, Auto vs Standard comparison, node pools, security model, networking, storage, and migration. |
| [AWS CLI EKS Commands](articles/aws-eks-cli-cheatsheet.md) | All `aws eks` commands — clusters, node groups, add-ons, access entries, Pod Identity, Fargate, updates, tokens, and waiters. |
| [EKS Node Not Joining Cluster](articles/eks-node-not-joining-troubleshooting.md) | Troubleshooting nodes that fail to join — IAM, security groups, networking, kubelet, bootstrap, and step-by-step diagnosis. |
| [EKS Node Monitoring](articles/eks-node-monitoring.md) | Node health monitoring — CloudWatch metrics, node conditions, node-problem-detector, Prometheus, alerts, and capacity planning. |
| [Why aws-auth Looks Different on Different Clusters](articles/eks-aws-auth-why-different.md) | How aws-auth ConfigMap is created, why fields vary, cluster creator access, Access Entries migration, and common mistakes. |
| [aws-auth ConfigMap Technical Details](articles/eks-aws-auth-technical-details.md) | How EKS authenticates IAM identities — token format, aws-iam-authenticator webhook, STS flow, edge cases, and debugging. |
| [Validating cloud-init on EKS Nodes](articles/eks-cloud-init-validation.md) | Verifying cloud-init completion — status checks, log files, bootstrap validation, common failures, and fleet monitoring. |
| [Cluster Autoscaler Restarts Troubleshooting](articles/eks-cluster-autoscaler-restarts.md) | Diagnosing "Service Unavailable" restarts — API server overload, AWS throttling, probe tuning, and monitoring. |
| [HAProxy on EKS with NLB](articles/eks-haproxy-nlb.md) | Deploying HAProxy with NLB — architecture, AWS LB Controller, ip target type, TLS options, health checks, and annotations reference. |
| [EKS Node Health and Auto-Repair](articles/eks-node-health-auto-repair.md) | Node health monitoring, auto-repair mechanisms, Node Problem Detector, NTH, Karpenter disruption, and self-healing patterns. |
| [EKS vs AKS vs GKE](articles/eks-vs-aks-vs-gke.md) | Managed Kubernetes compared — control plane, networking, IAM, node management, upgrades, security, and when to choose which. |
| [Karpenter Guide](articles/karpenter-guide.md) | Intelligent node scaling for EKS — cost savings over CA, NodePool/EC2NodeClass config, Spot management, consolidation, GPU, migration, and monitoring. |

### Azure

| Article | Description |
|---------|--------------|
| [Azure CLI Cheatsheet](articles/azure-cli-cheatsheet.md) | Azure CLI — installation, authentication, configuration, output formats, JMESPath queries, resource groups, and extensions. |
| [Azure Resource Groups Cheatsheet](articles/azure-resource-groups-cheatsheet.md) | Resource groups — create, list, tags, locks, RBAC, move resources, export templates, and deployment history. |
| [Azure Networking Cheatsheet](articles/azure-networking-cheatsheet.md) | Azure networking — VNets, subnets, NSGs, public IPs, NICs, peering, DNS, load balancers, and troubleshooting. |
| [Azure Storage Cheatsheet](articles/azure-storage-cheatsheet.md) | Azure Storage — blob containers, upload/download, SAS tokens, access tiers, file shares, managed disks, lifecycle policies, and AzCopy. |
| [Azure VM Management Cheatsheet](articles/azure-vm-cheatsheet.md) | Azure VMs — create, power management, disks, networking, resize, images, run commands, extensions, snapshots, and tags. |
| [AKS Cheatsheet](articles/azure-aks-cheatsheet.md) | Azure Kubernetes Service — cluster lifecycle, node pools, scaling, autoscaler, ACR integration, addons, RBAC, and troubleshooting. |
| [Azure Key Vault, Monitoring, IAM, and App Services](articles/azure-keyvault-monitoring-iam-cheatsheet.md) | Key Vault secrets/keys, monitoring metrics/alerts, Log Analytics, cost optimization, RBAC, managed identities, and App Service management. |
| [Azure VM Instance Types and Free Tier](articles/azure-vm-instance-types-free-tier.md) | Azure VM sizes, free tier eligibility, and instance type selection. |

### Virtualization

| Article | Description |
|---------|--------------|
| [Vagrant Cheatsheet](articles/vagrant-cheatsheet.md) | VM lifecycle, Vagrantfile, providers, provisioning, networking, multi-machine, plugins, and tips. |
| [Installing KVM](articles/kvm-installation.md) | KVM installation on RHEL 7–10 and Ubuntu 22.04/24.04 — packages, networking, storage, and verification. |
| [KVM / virsh Cheatsheet](articles/kvm-cheatsheet.md) | virsh commands — VM lifecycle, disks, snapshots, networks, pools, migration, and monitoring. |
| [Adding a New Disk in KVM](articles/kvm-add-disk.md) | Create, attach, partition, format, mount, resize, and detach disks in KVM guests. |
| [Enable virsh console](articles/kvm-virsh-console.md) | Configure serial console access for KVM VMs — GRUB, systemd getty, and troubleshooting. |
| [Running virt-manager Remotely](articles/virt-manager-remote-display.md) | X11 forwarding for virt-manager — PuTTY/Xming, XQuartz, SSH flags, and troubleshooting. |
| [Installing KVM Guests](articles/kvm-guest-installation.md) | Creating VMs with virt-install — ISO, kickstart, network install, cloud images, PXE, and automation. |
| [Using Cloud qcow2 Images with KVM](articles/kvm-qcow2-cloud-images.md) | Deploy pre-built cloud images — virt-customize, cloud-init, password changes, SSH keys, and deployment scripts. |
| [Enable SSH Password Auth in Ubuntu Cloud Images](articles/ubuntu-cloud-image-ssh-password.md) | virt-edit and virt-customize to enable password authentication — Ubuntu 20.04 vs 22.04/24.04 config paths, scripts, and verification. |
| [Converting VMware VMs to KVM](articles/kvm-convert-vmware-to-kvm.md) | virt-v2v — convert from vCenter, OVA, VMDK, Xen, and Hyper-V to KVM with libvirt. |
| [KVM libguestfs Tools](articles/kvm-libguestfs-tools.md) | virt-edit, virt-cat, virt-customize, virt-sysprep, guestfish — accessing and modifying VM disk images offline. |
| [Proxmox Cheatsheet](articles/proxmox-cheatsheet.md) | Proxmox VE — VM/CT management, storage, networking, clusters, and backups. |
| [Importing OVA/qcow2 into Proxmox](articles/proxmox-import-ova-qcow2.md) | Import VMware OVA, qcow2, and VMDK into Proxmox — qm importdisk, cloud-init, templates, and UEFI. |
| [Troubleshooting cloud-init on Proxmox](articles/proxmox-cloud-init-troubleshooting.md) | cloud-init debugging — datasource issues, network config, SSH keys, custom snippets, and template preparation. |
| [Resize a Partition on Proxmox](articles/proxmox-resize-partition.md) | Growing a VM disk — qm resize, growpart, LVM extend, ext4/XFS expansion, and online resize. |
| [QEMU Guest Agent on Proxmox](articles/proxmox-qemu-guest-agent.md) | Why the guest agent matters — consistent backups, IP display, remote commands, file transfer, and fstrim. |

### Terraform

| Article | Description |
|---------|--------------|
| [Terraform Cheatsheet](articles/terraform-cheatsheet.md) | IaC tool — core commands, state management, and workspaces. |
| [Packer Cheatsheet](articles/packer-cheatsheet.md) | Machine image builder — install, HCL2 templates, builders (AWS, Proxmox, QEMU, Docker), provisioners, post-processors, data sources, and CI/CD patterns. |
| [terraform.tfstate vs .terraform/terraform.tfstate](articles/terraform-tfstate-vs-terraform-directory-state.md) | Difference between `terraform.tfstate` and `.terraform/terraform.tfstate`. |
| [terraform init -upgrade and Constraints](articles/terraform-init-upgrade-and-constraints.md) | How `terraform init -upgrade` works with version constraints. |
| [terraform get -update vs init -upgrade](articles/terraform-get-update-vs-init-upgrade.md) | Difference between `terraform get -update` and `terraform init -upgrade`. |
| [Terraform Lock File Checksums: zh and h1](articles/terraform-lock-file-checksums.md) | How `zh:` and `h1:` hashes in `.terraform.lock.hcl` work. |
| [Terraform Root Module vs Child Modules](articles/terraform-root-vs-child-modules.md) | Module hierarchy — root vs child, calling modules, inputs/outputs, source types, and best practices. |
| [EOF Escaping in Userdata, Terraform, and Shell Scripts](articles/eof-escaping-userdata-terraform.md) | Heredoc quoting, `$${` escaping, templatefile(), and multi-layer variable expansion. |
| [Migrating State Off Terraform Cloud](articles/terraform-migrate-state-off-terraform-cloud.md) | Manual state migration from TFC to local, S3, AzureRM, or GCS backends. |
| [Terraform tfvars: Variable Definitions Reference](articles/terraform-tfvars-guide.md) | Complete reference for all variable types in `.tfvars` files — strings, lists, maps, objects, and nested structures. |
| [Terraform Variables: Declaration, Validation, and Usage](articles/terraform-variables-guide.md) | Variable blocks, type constraints, validation rules, usage patterns, dynamic blocks, and complex examples. |
| [Importing Existing Infrastructure Into Terraform](articles/terraform-import-guide.md) | Step-by-step import workflow, import blocks (1.5+), config generation, modules, for_each, and bulk import. |
| [Exporting Datadog Monitors to Terraform](articles/exporting-monitors-to-terraform.md) | Export monitors via console or API, convert to HCL with jq, and import into state. |
| [Terraform Backend Configuration Changed](articles/terraform-backend-configuration-changed.md) | Understanding -migrate-state vs -reconfigure — when to use each, .terraform/terraform.tfstate explained. |

### Ansible

| Article | Description |
|---------|--------------|
| [Ansible Ad-Hoc Commands](articles/ansible-adhoc-cheatsheet.md) | One-off remote execution — modules, patterns, packages, services, files, users, facts, and recipes. |
| [Ansible Cheatsheet](articles/ansible-cheatsheet.md) | Playbooks, inventory, variables, roles, vault, templates, handlers, loops, and project layout. |
| [Ansible Inventory Guide](articles/ansible-inventory-guide.md) | INI/YAML formats, groups, ranges, become, SSH, cloud, Windows, environments, and vault integration. |
| [Ansible Run Command Modules](articles/ansible-shell-command-modules.md) | command vs shell vs raw vs script — when to use each, error handling, and idempotency patterns. |
| [Ansible Sudo: Ubuntu vs RHEL](articles/ansible-sudo-ubuntu-vs-rhel.md) | Why become fails on RHEL — sudo/wheel groups, timestamp_timeout, passwordless setup, and mixed environments. |
| [Ansible Vault: Storing Sudo Passwords](articles/ansible-vault-become-pass.md) | Step-by-step vault setup for become_pass — create, reference, directory layout, vault IDs, and troubleshooting. |
| [Ansible Python Interpreter](articles/ansible-python-interpreter.md) | Fixing Python interpreter discovery warnings and errors — `ansible_python_interpreter`, auto-discovery, and per-host/group configuration. |
| [Ansible Ad-Hoc Commands vs Playbooks](articles/ansible-adhoc-vs-playbooks.md) | Why ad-hoc is faster, when playbooks win, side-by-side comparison, performance tips, and decision flowchart. |
| [Ansible: Editing Files and Creating Scripts](articles/ansible-file-editing-creation.md) | copy, template, lineinfile, blockinfile, replace — creating, editing, and deploying files on remote servers. |
| [Ansible Configuration: ansible.cfg Guide](articles/ansible-cfg-guide.md) | Config file precedence, global vs local, all sections and options, SSH tuning, fact caching, and production examples. |

### Bash and Shell

| Article | Description |
|---------|--------------|
| [bash Cheatsheet](articles/bash-cheatsheet.md) | GNU Bourne Again SHell — quoting, escaping, variables, loops, and built-ins. |
| [Korn Shell (ksh) Cheatsheet](articles/ksh-cheatsheet.md) | ksh88/ksh93 — history (fc/r), editing modes, variables, arrays, functions, and differences from bash. |
| [Bash Essentials Guide](articles/bash-essentials-guide.md) | Shell sessions, environment variables, quoting, history, prompt customization, and shortcuts. |
| [Bash Pipelines and Redirections](articles/bash-redirection-operators.md) | File descriptors, redirection operators, pipes, here-documents, process substitution, custom file descriptors, and advanced techniques. |
| [Bash History Guide](articles/bash-history-guide.md) | Command history — event designators, word designators, modifiers, Ctrl+R search, fc, configuration, and sharing across sessions. |
| [Bash Test Conditions: \[ \] vs \[\[ \]\]](articles/bash-test-conditions-guide.md) | Differences between `test`, `[ ]`, and `[[ ]]` — pattern matching, regex, and file tests. |
| [Bash Subshells](articles/bash-subshells-guide.md) | How subshells work, the pipeline variable problem, isolation patterns, and performance tips. |
| [Bash Troubleshooting Guide](articles/bash-troubleshooting-guide.md) | Bash debugging — `set -x`, `set -euo pipefail`, PS4, and tracing. |
| [macOS Bash Upgrade Guide](articles/macos-bash-upgrade-guide.md) | Installing a newer bash on macOS (ships with outdated 3.2.x). |
| [sed Replace Line Guide](articles/sed-replace-line-guide.md) | Using `sed` to replace entire lines based on a string match. |
| [Running Multiple Commands with sudo](articles/sudo-multiple-commands.md) | Subshells, heredocs, logical operators, pipes, and running as a specific user. |
| [sudoers Guide](articles/sudo-sudoers-guide.md) | Granting access to users, groups, and LDAP; aliases, Defaults, logging, and tips. |
| [Vim White Spaces](articles/vim-white-spaces.md) | Configuring vim to show white spaces with custom symbols. |
| [Cron Cheatsheet](articles/cron-cheatsheet.md) | Cron jobs — scheduling syntax, crontab management, environment, logging, locking, email, and scripting patterns. |
| [Bash Aliases and Functions](articles/bash-aliases-functions.md) | Productivity aliases for git, Docker, Kubernetes, systemd, networking, and utility shell functions (extract, mkcd, backup). |
| [awk Cheatsheet](articles/awk-cheatsheet.md) | Pattern scanning and text processing — fields, separators, regex, arithmetic, BEGIN/END, and one-liners. |
| [Print Column Numbers for Any Command Output](articles/awk-print-column-numbers.md) | Generic awk one-liner to identify column positions — examples with iotop, ps, df, ss, free, and top. |
| [sed Cheatsheet](articles/sed-cheatsheet.md) | Stream editor — substitution, deletion, insertion, addressing, capture groups, hold space, and one-liners. |
| [Vim Search and Replace](articles/vim-search-replace.md) | Vim substitution — ranges, flags, regex, capture groups, expression replacements, magic modes, and multi-file operations. |
| [Display Tabs and Whitespace in Files](articles/display-tabs-whitespace.md) | Revealing invisible characters — cat -A, grep, sed, vim :set list, hexdump, expand/unexpand, and conversion one-liners. |

### Linux System Administration

| Article | Description |
|---------|--------------|
| [systemd Cheatsheet](articles/systemd-cheatsheet.md) | systemctl commands, unit files, service types, timers, targets, boot analysis, and recipes. |
| [journalctl Cheatsheet](articles/journalctl-cheatsheet.md) | systemd journal — viewing, filtering, grep, output formats, disk management, and troubleshooting recipes. |
| [Enable Persistent systemd Journal Logging](articles/systemd-journal-persistent-logging.md) | Persistent journal on RHEL 7–10 — mkdir vs Storage=persistent, journalctl --flush, retention, and rotation. |
| [dpkg Cheatsheet](articles/dpkg-cheatsheet.md) | Debian package manager — install, remove, query, verify, hold, diversions, alternatives, and .deb creation. |
| [apt Cheatsheet](articles/apt-cheatsheet.md) | APT package management — install, upgrade, repositories, pinning, cache, proxy, offline installs, and automation. |
| [apt vs apt-get](articles/apt-vs-apt-get.md) | When to use each — command mapping, behavioral differences, output stability, and scripting guidelines. |
| [DEBIAN_FRONTEND for Scripts](articles/debian-frontend-noninteractive.md) | Non-interactive installs — debconf frontends, dpkg config options, preseeding, needrestart, Docker, and CI/CD patterns. |
| [Linux File Permissions Guide](articles/linux-file-permissions.md) | Permissions, ownership, umask, SUID/SGID, sticky bit, and ACLs. |
| [/bin/false vs /sbin/nologin](articles/bin-false-vs-nologin.md) | Login shell differences — behavior, custom messages, /etc/nologin, path variations, and when to use each. |
| [Linux User Quotas](articles/linux-user-quotas.md) | Disk quotas — ext4/XFS setup, setquota, edquota, grace periods, project quotas, warnquota, and troubleshooting. |
| [User Administration on RHEL](articles/user-administration.md) | User and group management — UIDs, password hashing, PAM, and chage. |
| [LDAP Client Configuration](articles/ldap-client-configuration.md) | Configuring LDAP authentication on RHEL 5–10 and Ubuntu 22.04/24.04 — SSSD, authconfig, authselect, TLS, and Active Directory. |
| [Configure Samba](articles/samba-configuration.md) | Samba file server on RHEL 6–10 and Ubuntu — anonymous/authenticated shares, SELinux, AD membership, client access, and troubleshooting. |
| [MySQL LDAP Authentication](articles/mysql-ldap-authentication.md) | MySQL PAM authentication with SSSD and LDAP — tarball install, PAM plugin, SSSD config, TLS certificates, and troubleshooting. |
| [Installing MediaWiki on RHEL](articles/mediawiki-installation-rhel.md) | MediaWiki with Apache, PHP 7.4, and MariaDB on RHEL 8/9 — packages, database, SELinux, firewall, and deployment. |
| [Installing DokuWiki on RHEL](articles/dokuwiki-installation-rhel.md) | DokuWiki with Apache and PHP on RHEL 8/9 — packages, mod_rewrite, SELinux, .htaccess, and web installer. |
| [Protect SSH with fail2ban](articles/fail2ban-ssh-protection.md) | fail2ban on RHEL and Ubuntu — installation, SSH jail, whitelisting, incremental bans, email alerts, and troubleshooting. |
| [ReaR Backup Guide](articles/rear-backup-guide.md) | Relax-and-Recover — disaster recovery ISOs, NFS/CIFS/USB/rsync targets, incremental backups, recovery process, and scheduling. |
| [Veeam Agent for Linux](articles/veeam-agent-linux.md) | Veeam backup agent — installation, jobs, volume/file-level restore, NFS server setup, dd+ssh imaging, and command reference. |
| [Installing MariaDB](articles/mariadb-installation.md) | MariaDB on RHEL 7–10 and Ubuntu — installation, hardening, user management, backup/restore, and troubleshooting. |
| [RHEL Releases Overview](articles/rhel-releases-overview.md) | RHEL major releases (2.1–10) — features, lifecycle, and upgrade paths. |
| [Timezone Configuration](articles/timezone-configuration.md) | timedatectl, NTP, hardware clock, per-process TZ, cloud-init, Docker, and troubleshooting. |
| [subscription-manager Cheatsheet](articles/subscription-manager-cheatsheet.md) | RHEL subscriptions — register, attach, repos, release lock, Satellite, SCA, and troubleshooting. |
| [dnf / yum Cheatsheet](articles/dnf-yum-cheatsheet.md) | Package management on RHEL 7–10 — install, update, repos, modules, groups, history, versionlock, and troubleshooting. |
| [RPM Cheatsheet](articles/rpm-cheatsheet.md) | RPM commands — install, query, verify, signatures, custom query formats, database management, and rpm2cpio. |
| [RHEL Boot Modes and Troubleshooting](articles/rhel-boot-troubleshooting.md) | Boot modes, rescue/emergency targets, and recovery techniques. |
| [GRUB2 Cheatsheet](articles/grub-cheatsheet.md) | grubby, grub2-mkconfig, kernel args, default entry, password, BLS, serial console, and rescue. |
| [initramfs vs initrd](articles/initramfs-vs-initrd.md) | Early boot filesystem — differences, dracut, update-initramfs, inspection, rebuild, and troubleshooting. |
| [Rebuild initramfs in RHEL](articles/rebuild-initramfs-rhel.md) | Step-by-step rebuild across RHEL 3–10 — dracut, mkinitrd, rescue mode, backups, and verification. |
| [chroot Guide](articles/chroot-guide.md) | Change root environment — system recovery, virtual mounts, LVM/LUKS rescue, package builds, and escape limitations. |
| [Linux Kernel Panics](articles/linux-kernel-panics.md) | Hard panics (Aieee!) and soft panics (Oops) — causes, interpretation, kdump, crash analysis, and prevention. |
| [Why Processes in D State Can't Be Killed](articles/linux-processes-d-state.md) | Uninterruptible sleep explained — why kill -9 fails, diagnosing stuck processes, NFS hangs, and TASK_KILLABLE. |
| [Linux Capabilities](articles/linux-capabilities.md) | Breaking root privilege into fine-grained units — getcap, setcap, capability sets, systemd integration, and Docker. |
| [Linux System Calls](articles/linux-syscalls.md) | Syscall interface — how user programs talk to the kernel, tracing with strace, categories, seccomp, and patterns. |
| [Linux Kernel Map](articles/linux-kernel-map.md) | Kernel subsystems overview — process, memory, VFS, network, drivers, security, source tree, boot process, and tuning. |
| [Linux SysRq Guide](articles/linux-sysrq-guide.md) | Magic SysRq Key — REISUB safe reboot, emergency commands, debugging a hung system, and serial console usage. |
| [Linux ulimit Guide](articles/linux-ulimit-guide.md) | Per-process resource limits, limits.conf, systemd directives, sysctl, and troubleshooting. |
| [Remove .DS_Store from Git](articles/remove-ds-store-guide.md) | Remove and prevent .DS_Store files from being tracked in git. |
| [LD_LIBRARY_PATH and Shared Libraries](articles/linux-ld-library-path.md) | Dynamic linker search order, LD_LIBRARY_PATH usage and risks, ldconfig, compiling in $HOME, and best practices. |

### Satellite and Foreman

| Article | Description |
|---------|--------------|
| [Hammer CLI Cheatsheet](articles/hammer-cheatsheet.md) | Hammer CLI for Red Hat Satellite/Foreman — hosts, content views, repos, errata, provisioning, and remote execution. |
| [Installing Foreman with Katello](articles/foreman-katello-installation.md) | Foreman 3.13 + Katello 4.15 on RHEL 9 — prerequisites, installation, content setup, and troubleshooting. |
| [Installing Satellite from ISO](articles/satellite-installation-iso.md) | Red Hat Satellite 6.16 disconnected install on RHEL 9 — ISO mount, local repos, installer, and content import. |
| [Foreman Remote Execution Setup](articles/foreman-remote-execution.md) | Setting up REX — SSH key distribution, running jobs, custom templates, Ansible integration, and troubleshooting. |
| [Content Views and Activation Keys Strategy](articles/foreman-content-views-activation-keys.md) | Designing CVs, CCVs, and activation keys — one CV per product, composing with multiple AKs, and update workflows. |
| [Registering Hosts in Foreman](articles/foreman-host-registration.md) | Registering RHEL and Ubuntu hosts — global registration, subscription-manager, activation keys, capsules, and troubleshooting. |

### Linux Performance and IO

| Article | Description |
|---------|--------------|
| [Linux Load Average](articles/linux-load-average.md) | What load average actually measures on Linux, how to interpret it, and common misconceptions. |
| [Linux CPU Steal Time](articles/linux-cpu-steal.md) | %steal in VMs — what it means, detection one-liners, RHEL/Ubuntu specifics, causes, cloud/on-prem remediation, and alerting. |
| [Linux I/O Schedulers](articles/linux-io-schedulers.md) | I/O schedulers (noop, deadline, cfq, mq-deadline, bfq, kyber), tuning, and per-distro defaults. |
| [Linux Disk I/O Internals](articles/linux-disk-io-internals.md) | Page cache, standard I/O, direct I/O, mmap, block alignment, and write durability. |
| [blktrace Guide](articles/blktrace-guide.md) | Block layer I/O tracing — capturing events, parsing output, latency analysis with btt, and diagnosing disk performance. |
| [Configuring sysstat on Ubuntu](articles/configuring-sysstat-ubuntu.md) | Installing and configuring sysstat on Ubuntu with systemd timers. |
| [sysstat / sar Cheatsheet](articles/sysstat-sar-cheatsheet.md) | System Activity Reporter — CPU, memory, disk, network monitoring, historical analysis, sadf output formats, and alerting. |
| [Understanding vmstat Output](articles/understanding-vmstat-output.md) | Understanding vmstat output — CPU, memory, I/O, and process scheduling diagnostics. |
| [iostat Cheatsheet](articles/iostat-cheatsheet.md) | Block device I/O statistics — IOPS, throughput, latency, queue depth, and saturation patterns. |
| [Understanding iostat -x Output](articles/understanding-iostat-x-output.md) | Extended block device I/O statistics — granular disk performance monitoring. |
| [iotop Cheatsheet](articles/iotop-cheatsheet.md) | Interactive I/O monitoring — per-process disk read/write usage. |
| [du Cheatsheet](articles/du-cheatsheet.md) | Disk usage — estimating file and directory space, sorting, filtering, and practical patterns. |
| [ps Cheatsheet](articles/ps-cheatsheet.md) | Process status — listing, filtering, and inspecting running processes. |
| [top Cheatsheet](articles/top-cheatsheet.md) | Interactive process viewer — CPU, memory, sorting, filtering, and batch mode. |
| [free Cheatsheet](articles/free-cheatsheet.md) | Memory usage — free, top, /proc/meminfo, vmstat, and per-process memory. |

### Linux Memory

| Article | Description |
|---------|--------------|
| [Linux Memory: RSS, VSZ, and Why RSS Alone Is Misleading](articles/linux-memory-rss-vsz.md) | Virtual memory, RSS vs VSZ, shared pages, and accurate memory measurement. |
| [Linux Swap Usage: When Processes Aren't the Culprit](articles/linux-swap-shm-segments.md) | Why per-process swap doesn't add up — SHM segments, `/proc/sysvipc/shm`, and Oracle SGA. |

### Linux Storage and Filesystems

| Article | Description |
|---------|--------------|
| [Linux /etc/fstab Guide](articles/linux-fstab-guide.md) | Syntax, device identification, filesystem types, mount options (ext4, XFS, NFS, CIFS, tmpfs, swap), and security hardening. |
| [iSCSI Cheatsheet](articles/iscsi-cheatsheet.md) | iSCSI initiator and target — discovery, login, CHAP, multipath, targetcli, performance tuning, and troubleshooting. |
| [Multipath Cheatsheet](articles/multipath-cheatsheet.md) | DM-Multipath — setup, configuration, failover, LVM, troubleshooting, and path management. |
| [EMC PowerPath Cheatsheet](articles/emc-powerpath-cheatsheet.md) | EMC PowerPath — powermt commands, policies, HBA management, and array-specific configuration. |
| [SAN Storage Commands](articles/san-storage-commands.md) | SCSI scanning, HBA info, Fibre Channel diagnostics, disk mapping, I/O scheduler, and SAR monitoring. |
| [Fibre Channel Error Statistics](articles/fc-statistics-guide.md) | FC HBA error counters in sysfs — CRC, tx word, link failure, loss of sync/signal, monitoring scripts, and troubleshooting. |
| [Linux Storage Stack](articles/linux-storage-stack.md) | VFS, filesystems, page cache, block layer, device mapper, SCSI/NVMe — layers, tools, and tuning. |
| [fdisk Cheatsheet](articles/fdisk-cheatsheet.md) | Disk partitioning — fdisk, gdisk, parted, sgdisk, sfdisk, mkfs, and LVM setup. |
| [LVM Cheatsheet](articles/lvm-cheatsheet.md) | Logical Volume Manager — PVs, VGs, LVs, snapshots, thin provisioning, RAID, cache, and troubleshooting. |
| [fsck Cheatsheet](articles/fsck-cheatsheet.md) | Filesystem check and repair — e2fsck, xfs_repair, badblocks, and SMART. |
| [Partition Alignment Guide](articles/partition-alignment-guide.md) | Why 1 MiB alignment matters for SSDs, 4Kn HDDs, RAID, LVM, and virtual machines. |
| [XFS Internals: Superblock and Addressing](articles/xfs-internals-superblock.md) | XFS superblock structure, allocation groups, and block/inode addressing schemes. |
| [ext4 Journal Modes](articles/ext4-journal-modes.md) | Journal modes (ordered, writeback, journal), configuration, commit intervals, and barriers. |
| [Extending Partitions with growpart](articles/growpart-extend-partitions.md) | Extend partitions online on cloud/VM instances — AWS, Azure, GCP, LVM, and troubleshooting. |
| [Extend a SAN LUN Online with Multipath and GFS2](articles/linux-extend-lun-multipath-gfs2.md) | Expanding a SAN-attached LUN on a Linux cluster — SCSI rescan, multipath resize, LVM extend, and GFS2 grow without downtime. |
| [GFS2 & RHEL Cluster Cheatsheet](articles/gfs2-cluster-cheatsheet.md) | GFS2 and RHEL cluster management — cman, ccs, fencing, DLM, service management, filesystem operations, and troubleshooting. |
| [Disk Health & Maintenance](articles/disk-health-maintenance.md) | SMART monitoring, bad sectors, performance testing, secure wiping, disk cloning, SSD tuning, and emergency commands. |

### Networking

| Article | Description |
|---------|--------------|
| [DNS Cheatsheet](articles/dns-cheatsheet.md) | DNS lookup tools — dig, host, nslookup, getent, resolvectl, record types, and troubleshooting. |
| [Setting Up a DNS Server on RHEL 9](articles/dns-server-rhel9.md) | BIND DNS server — installation, forward/reverse zones, caching, split-horizon, secondary servers, and SELinux. |
| [resolvectl Cheatsheet](articles/resolvectl-cheatsheet.md) | systemd-resolved CLI — DNS config, caching, DoT, DNSSEC, search domains, routing domains, and resolv.conf modes. |
| [SSH Cheatsheet](articles/ssh-cheatsheet.md) | OpenSSH client and server — connections, keys, forwarding, tunnels, and troubleshooting. |
| [SSH ControlMaster](articles/ssh-controlmaster.md) | SSH connection multiplexing — reuse a single TCP connection for multiple sessions. |
| [SSH ProxyJump vs ProxyCommand](articles/ssh-proxyjump-vs-proxycommand.md) | Differences between ProxyJump and ProxyCommand for reaching hosts behind bastions. |
| [SSH Managing Multiple Keys](articles/ssh-managing-multiple-keys.md) | Per-service SSH keys for AWS, Proxmox, GitHub, and homelab — config, agent, and rotation. |
| [SSH Generate Keys](articles/ssh-keygen-guide.md) | Generate Ed25519, RSA, and ECDSA keys — passphrases, options, FIDO2, and security hardening. |
| [SSH Convert Keys](articles/ssh-convert-keys.md) | Convert between OpenSSH, PuTTY PPK, PEM, PKCS#8, DER, and SSH.com key formats. |
| [SSH Remote Script Execution](articles/ssh-remote-script-execution.md) | Tools for running scripts on remote hosts — SSH, pssh, pdsh, Ansible, Fabric, and more. |
| [SSH Remote Sudo Execution](articles/ssh-remote-sudo-execution.md) | Running privileged commands remotely — `sudo -S`, password automation, NOPASSWD, expect, and security practices. |
| [SSH Heredoc Variable Expansion](articles/ssh-heredoc-variables.md) | Heredocs over SSH — local vs remote expansion, quoted/unquoted EOF, nested heredocs, envsubst, and CI/CD patterns. |
| [ip Command Cheatsheet](articles/ip-command-cheatsheet.md) | iproute2 ip command — addresses, links, routes, neighbours, multicast, and net-tools migration. |
| [ss Cheatsheet](articles/ss-cheatsheet.md) | Socket statistics — inspecting TCP/UDP connections and states. |
| [VNC Cheatsheet](articles/vnc-cheatsheet.md) | TigerVNC on RHEL 6–10 and Ubuntu — installation, configuration, service management, SSH tunnels, one-liners, and troubleshooting. |

### Cloud-Init

| Article | Description |
|---------|--------------|
| [cloud-init Cheatsheet](articles/cloud-init-cheatsheet.md) | Cross-platform cloud instance initialization — user-data, modules, networking, and debugging. |
| [cloud-init status: Errors and Failure Modes](articles/cloud-init-status-command.md) | Status command, exit codes, critical vs recoverable errors, per-stage diagnostics, and scripting patterns. |
| [cloud-init: bootcmd vs runcmd](articles/cloud-init-bootcmd-vs-runcmd.md) | Boot stages, execution timing, frequency differences, and common mistakes. |
| [cloud-init: Why tee Output Doesn't Appear in Logs](articles/cloud-init-tee-output-missing.md) | How cloud-init's output directive interacts with tee, buffering, and pipes. |
| [cloud-init: User Management and the gecos Field](articles/cloud-init-users-gecos.md) | Users module, gecos history, all user keys, default user, and common patterns. |
| [cloud-init: Different Ways to Create Files on a Server](articles/cloud-init-write-files.md) | write_files module, runcmd redirects, bootcmd, shell scripts, Jinja templates, and encoding options. |

### Datadog

| Article | Description |
|---------|--------------|
| [Datadog Agent Cheatsheet](articles/datadog-agent-cheatsheet.md) | Agent installation, service management, configuration, checks, logs, DogStatsD, APM, and troubleshooting. |
| [Datadog API Reference](articles/datadog-api-reference.md) | Authentication, client libraries, common endpoints (monitors, dashboards, metrics, events), rate limits, and scripting patterns. |
| [Datadog Dashboards Guide](articles/datadog-dashboards-guide.md) | Dashboard types, widget selection, query patterns, template variables, layout strategies, and best practices. |
| [Datadog Monitor Notification Variables](articles/datadog-monitor-notification-variables.md) | Conditional variables, attribute/tag variables, template variables, dynamic handles, and advanced notification formatting. |
| [Datadog Monitor Tagging Best Practices](articles/datadog-monitor-tagging-best-practices.md) | Tagging strategies for monitors — filtering, downtime scheduling, dashboard widgets, SLO organization, and API usage. |
| [Datadog Monitors Tips & Tricks](articles/datadog-monitors-tips-and-tricks.md) | Hidden settings, anti-flapping, noise reduction, formulas, composite monitors, notification tricks, and API-only options. |
| [Monitoring Apache Web Server Performance](articles/monitoring-apache-performance.md) | Apache metrics, MPM internals, mod_status, Datadog integration, log collection, and dashboard setup. |

### Terminal and Tools

| Article | Description |
|---------|--------------|
| [JSON Query Tools: JMESPath vs jq vs JSONPath](articles/json-query-tools.md) | Comparing JMESPath, jq, and JSONPath — syntax, filtering, transformation, and when to use each. |
| [bat Cheatsheet](articles/bat-cheatsheet.md) | A cat clone with syntax highlighting, git integration, themes, and paging. |
| [cut Cheatsheet](articles/cut-cheatsheet.md) | Extract fields, characters, or bytes from text — delimiters, ranges, and practical patterns. |
| [tmux Cheatsheet](articles/tmux-cheatsheet.md) | Terminal multiplexer — sessions, windows, panes, and copy mode. |
| [Kitty Cheatsheet](articles/kitty-cheatsheet.md) | GPU-accelerated terminal emulator — tabs, windows, and layouts. |
| [JetBrains Mono Font](articles/jetbrains-mono-font.md) | Free monospaced font designed for terminals and code editors. |
| [PuTTY Default Settings](articles/putty-default-settings.md) | Font, bell, colors, window size, and scrollback — settings to apply after a fresh Windows install. |
| [rclone Cheatsheet](articles/rclone-cheatsheet.md) | Cloud storage CLI — copy, sync, mount, encrypt, serve, filtering, backup patterns, and 70+ backends. |
| [Homebrew Cheatsheet](articles/homebrew-cheatsheet.md) | macOS package manager — install, update, services, taps, Brewfile, casks, versions, and cleanup. |

### Windows

| Article | Description |
|---------|--------------|
| [Windows Battery Report](articles/windows-battery-report.md) | Using the built-in `powercfg` tool to check battery health, usage, and degradation. |
| [Windows Tips and Commands](articles/windows-tips-commands.md) | Useful commands — perfmon, winsat, diskpart, Test-NetConnection, hosts file, resmon, gpupdate, and Java config. |
| [Outlook Instant Search Syntax](articles/outlook-search-syntax.md) | Search query syntax — operators, from/to/cc, attachments, dates, size filters, flags, calendar, and contacts. |

## About

This is a personal wiki/knowledge base for tools, commands, and configurations used across a self-hosted homelab environment — spanning Kubernetes, container runtimes, networking, terminal tooling, and infrastructure as code. A collection of all commands and knowledge gathered over the last 15+ years, written as standalone Markdown references aimed at fast lookup rather than deep tutorials.

- **Author:** Daniel Corneschi
- **Site:** Built with [docsify](https://docsify.js.org/), hosted on [GitHub Pages](https://pages.github.com/)
- **Format:** All content lives in `articles/` — cheatsheets and longer-form guides side by side, with shared images in `articles/images/`
- **Scope:** Personal use — commands and examples reflect this homelab's specific setup (namespaces, socket names, config paths) and may need adjusting for other environments
