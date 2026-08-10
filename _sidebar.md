- [Home](README.md)

- Kubernetes
  - [Using jq with kubectl](articles/kubectl-jq-guide.md)
  - [kubectl JSONPath Guide](articles/kubectl-jsonpath-guide.md)
  - [Getting Started with Argo CD](articles/getting-started-argo.md)
  - [Kubernetes imagePullPolicy](articles/kubernetes-imagepullpolicy.md)
  - [Kubernetes emptyDir Volumes](articles/kubernetes-emptyDir-volumes.md)
  - [Kubernetes PriorityClasses Guide](articles/kubernetes-priority-classes-guide.md)
  - [Kubernetes QoS Classes — Requests and Limits](articles/kubernetes-qos-requests-limits.md)
  - [Kubernetes Pod Evictions Cheatsheet](articles/kubernetes-evictions-cheatsheet.md)
  - [Kubernetes PodDisruptionBudgets Guide](articles/kubernetes-pdb-guide.md)
  - [kubectl run vs kubectl create](articles/kubectl-run-vs-create.md)
  - [Init Containers vs Regular Containers](articles/kubernetes-init-vs-regular-containers.md)
  - [Ingress for Kubernetes Dashboard on MicroK8s](articles/ingress-kubernetes-dashboard-microk8s.md)
  - [Ingress with MetalLB on MicroK8s](articles/ingress-metallb-microk8s-guide.md)
  - [NFS Storage for MicroK8s](articles/nfs-microk8s-installation.md)
  - [Helm Cheatsheet](articles/helm-cheatsheet.md)
  - [crictl Cheatsheet](articles/crictl-cheatsheet.md)
  - [k9s Cheatsheet](articles/k9s-cheatsheet.md)

- Docker
  - [Docker Cheatsheet](articles/docker-cheatsheet.md)
  - [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md)
  - [Docker Compose: ports vs expose](articles/docker-ports-vs-expose.md)
  - [Fix Gitea Runner Docker Hub Rate Limits](articles/docker-gitea-runner-fix.md)
  - [dbash — Docker Shell Function](articles/docker-dbash-function.md)

- AWS
  - [AWS CLI Installation](articles/aws-cli-install.md)
  - [EC2 Cheatsheet](articles/aws-ec2-cheatsheet.md)
  - [EC2 Extend EBS Volume](articles/aws-ec2-extend-disk.md)
  - [Installing SSM Agent](articles/aws-ssm-agent-install.md)
  - [JMESPath Query Guide](articles/aws-jmespath-guide.md)
  - [EKS Authentication Modes: ConfigMap vs Access Entries](articles/eks-authentication-modes.md)

- Virtualization
  - [Installing KVM](articles/kvm-installation.md)
  - [KVM / virsh Cheatsheet](articles/kvm-cheatsheet.md)
  - [Adding a New Disk in KVM](articles/kvm-add-disk.md)
  - [Enable virsh console](articles/kvm-virsh-console.md)
  - [Proxmox Cheatsheet](articles/proxmox-cheatsheet.md)

- Terraform
  - [Terraform Cheatsheet](articles/terraform-cheatsheet.md)
  - [terraform.tfstate vs .terraform/terraform.tfstate](articles/terraform-tfstate-vs-terraform-directory-state.md)
  - [terraform init -upgrade and Constraints](articles/terraform-init-upgrade-and-constraints.md)
  - [terraform get -update vs init -upgrade](articles/terraform-get-update-vs-init-upgrade.md)
  - [Terraform Lock File Checksums: zh and h1](articles/terraform-lock-file-checksums.md)
  - [EOF Escaping in Userdata, Terraform, and Shell Scripts](articles/eof-escaping-userdata-terraform.md)
  - [Migrating State Off Terraform Cloud](articles/terraform-migrate-state-off-terraform-cloud.md)
  - [Terraform tfvars: Variable Definitions Reference](articles/terraform-tfvars-guide.md)
  - [Terraform Variables: Declaration, Validation, and Usage](articles/terraform-variables-guide.md)
  - [Importing Existing Infrastructure Into Terraform](articles/terraform-import-guide.md)

- Bash & Shell
  - [bash Cheatsheet](articles/bash-cheatsheet.md)
  - [Bash Essentials Guide](articles/bash-essentials-guide.md)
  - [Bash Test Conditions: \[ \] vs \[\[ \]\]](articles/bash-test-conditions-guide.md)
  - [Bash Subshells](articles/bash-subshells-guide.md)
  - [Bash Troubleshooting Guide](articles/bash-troubleshooting-guide.md)
  - [macOS Bash Upgrade Guide](articles/macos-bash-upgrade-guide.md)
  - [sed Replace Line Guide](articles/sed-replace-line-guide.md)
  - [Running Multiple Commands with sudo](articles/sudo-multiple-commands.md)
  - [sudoers Guide](articles/sudo-sudoers-guide.md)
  - [Vim White Spaces](articles/vim-white-spaces.md)

- Linux System Administration
  - [Linux File Permissions Guide](articles/linux-file-permissions.md)
  - [User Administration on RHEL](articles/user-administration.md)
  - [RHEL Releases Overview](articles/rhel-releases-overview.md)
  - [RHEL Boot Modes and Troubleshooting](articles/rhel-boot-troubleshooting.md)
  - [Linux ulimit Guide](articles/linux-ulimit-guide.md)
  - [Remove .DS_Store from Git](articles/remove-ds-store-guide.md)

- Linux Performance & I/O
  - [Linux Load Average](articles/linux-load-average.md)
  - [Linux I/O Schedulers](articles/linux-io-schedulers.md)
  - [Linux Disk I/O Internals](articles/linux-disk-io-internals.md)
  - [Configuring sysstat on Ubuntu](articles/configuring-sysstat-ubuntu.md)
  - [Understanding vmstat Output](articles/understanding-vmstat-output.md)
  - [Understanding iostat -x Output](articles/understanding-iostat-x-output.md)
  - [iotop Cheatsheet](articles/iotop-cheatsheet.md)
  - [ps Cheatsheet](articles/ps-cheatsheet.md)
  - [top Cheatsheet](articles/top-cheatsheet.md)
  - [free Cheatsheet](articles/free-cheatsheet.md)

- Linux Memory
  - [Linux Memory: RSS, VSZ, and Why RSS Alone Is Misleading](articles/linux-memory-rss-vsz.md)
  - [Linux Swap Usage: When Processes Aren't the Culprit](articles/linux-swap-shm-segments.md)

- Linux Storage & Filesystems
  - [Multipath Cheatsheet](articles/multipath-cheatsheet.md)
  - [EMC PowerPath Cheatsheet](articles/emc-powerpath-cheatsheet.md)
  - [SAN Storage Commands](articles/san-storage-commands.md)
  - [Linux Storage Stack](articles/linux-storage-stack.md)
  - [fdisk Cheatsheet](articles/fdisk-cheatsheet.md)
  - [LVM Cheatsheet](articles/lvm-cheatsheet.md)
  - [fsck Cheatsheet](articles/fsck-cheatsheet.md)
  - [Partition Alignment Guide](articles/partition-alignment-guide.md)
  - [XFS Internals: Superblock and Addressing](articles/xfs-internals-superblock.md)
  - [ext4 Journal Modes](articles/ext4-journal-modes.md)
  - [Extending Partitions with growpart](articles/growpart-extend-partitions.md)

- Networking
  - [SSH Cheatsheet](articles/ssh-cheatsheet.md)
  - [SSH ControlMaster](articles/ssh-controlmaster.md)
  - [SSH ProxyJump vs ProxyCommand](articles/ssh-proxyjump-vs-proxycommand.md)
  - [SSH Managing Multiple Keys](articles/ssh-managing-multiple-keys.md)
  - [SSH Generate Keys](articles/ssh-keygen-guide.md)
  - [SSH Convert Keys](articles/ssh-convert-keys.md)
  - [SSH Remote Script Execution](articles/ssh-remote-script-execution.md)
  - [ss Cheatsheet](articles/ss-cheatsheet.md)

- Cloud-Init
  - [cloud-init Cheatsheet](articles/cloud-init-cheatsheet.md)
  - [cloud-init: bootcmd vs runcmd](articles/cloud-init-bootcmd-vs-runcmd.md)
  - [cloud-init: Why tee Output Doesn't Appear in Logs](articles/cloud-init-tee-output-missing.md)
  - [cloud-init: User Management and the gecos Field](articles/cloud-init-users-gecos.md)

- Terminal & Tools
  - [JSON Query Tools: JMESPath vs jq vs JSONPath](articles/json-query-tools.md)
  - [bat Cheatsheet](articles/bat-cheatsheet.md)
  - [cut Cheatsheet](articles/cut-cheatsheet.md)
  - [tmux Cheatsheet](articles/tmux-cheatsheet.md)
  - [Kitty Cheatsheet](articles/kitty-cheatsheet.md)
  - [JetBrains Mono Font](articles/jetbrains-mono-font.md)
  - [PuTTY Default Settings](articles/putty-default-settings.md)

- Windows
  - [Windows Battery Report](articles/windows-battery-report.md)
