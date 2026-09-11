<div align="center">

<img src="articles/images/homelab-wiki-logo.svg" alt="corneschi.ro" width="700" />

<p>Quick reference guides, command tables, and longer-form write-ups covering Kubernetes, container runtimes, networking, terminal tooling, and infrastructure as code — all tailored to a self-hosted homelab environment.</p>

<br/>

</div>

## Usage

This site is built with [docsify](https://docsify.js.org/) and served via GitHub Pages. Browse the sidebar or click an article to get started. Search is available in the sidebar to quickly find specific commands across all guides.

## What's Inside

### Kubernetes

| Article |
|---------|
| [Using jq with kubectl](articles/kubectl-jq-guide.md) |
| [kubectl JSONPath Guide](articles/kubectl-jsonpath-guide.md) |
| [kubectl + sed Combinations](articles/kubectl-sed-combinations.md) |
| [Getting Started with Argo CD](articles/getting-started-argo.md) |
| [Kubernetes imagePullPolicy](articles/kubernetes-imagepullpolicy.md) |
| [Kubernetes emptyDir Volumes](articles/kubernetes-emptyDir-volumes.md) |
| [Kubernetes PriorityClasses Guide](articles/kubernetes-priority-classes-guide.md) |
| [Kubernetes QoS Classes — Requests and Limits](articles/kubernetes-qos-requests-limits.md) |
| [Kubernetes Pod Evictions Cheatsheet](articles/kubernetes-evictions-cheatsheet.md) |
| [Evicting Pods from Nodes: A Practical Guide](articles/kubectl-evict-pods-guide.md) |
| [Kubernetes PodDisruptionBudgets Guide](articles/kubernetes-pdb-guide.md) |
| [kubectl run vs kubectl create](articles/kubectl-run-vs-create.md) |
| [Init Containers vs Regular Containers](articles/kubernetes-init-vs-regular-containers.md) |
| [Ingress for Kubernetes Dashboard on MicroK8s](articles/ingress-kubernetes-dashboard-microk8s.md) |
| [Ingress with MetalLB on MicroK8s](articles/ingress-metallb-microk8s-guide.md) |
| [NFS Storage for MicroK8s](articles/nfs-microk8s-installation.md) |
| [Dynamic PV/PVC Provisioning with a StorageClass](articles/kubernetes-dynamic-provisioning-storageclass.md) |
| [Krew: The kubectl Plugin Manager](articles/kubectl-krew-plugin-manager.md) |
| [crictl Cheatsheet](articles/crictl-cheatsheet.md) |
| [ctr Cheatsheet (containerd)](articles/ctr-cheatsheet.md) |
| [Kubernetes Schema Validation](articles/kubernetes-schema-validation.md) |
| [k9s Cheatsheet](articles/k9s-cheatsheet.md) |
| [Node Selectors in Kubernetes](articles/kubernetes-node-selectors.md) |
| [Node Affinity in Kubernetes](articles/kubernetes-node-affinity.md) |
| [Kubernetes Taints and Tolerations](articles/kubernetes-taints-tolerations.md) |
| [Taint vs Cordon vs Drain](articles/kubernetes-taint-cordon-drain.md) |
| [Kubernetes Scheduling Deep Dive](articles/kubernetes-scheduling-deep-dive.md) |
| [LimitRange and ResourceQuota](articles/kubernetes-limitrange-resourcequota.md) |
| [Kubernetes Pod Conditions Flow](articles/kubernetes-pod-conditions-flow.md) |
| [Pod Phases and the Succeeded Phase](articles/kubernetes-pod-phase-succeeded.md) |
| [Kubernetes Pod Commands](articles/kubernetes-pod-commands.md) |
| [Fix DaemonSet Scheduling on EKS](articles/eks-daemonset-scheduling-fix.md) |
| [Kubernetes Control Plane API Commands](articles/kubernetes-api-commands.md) |
| [kubectl Cheatsheet](articles/kubectl-cheatsheet.md) |
| [kubectl Client-Side Throttling Explained](articles/kubectl-client-side-throttling.md) |
| [Kubelet Image-Pull Throttling: "pull QPS exceeded"](articles/kubelet-pull-qps-exceeded.md) |
| [Kubernetes Field Selectors](articles/kubernetes-field-selectors.md) |
| [kubectl logs Guide](articles/kubectl-logs-guide.md) |
| [kubectl logs --previous](articles/kubectl-logs-previous.md) |
| [kubectl set env](articles/kubectl-set-env.md) |
| [Kubernetes Log Locations by Distribution](articles/kubernetes-log-locations.md) |
| [Kubernetes Jobs and CronJobs](articles/kubernetes-jobs-cronjobs.md) |
| [EKS Port Communication](articles/eks-port-communication.md) |
| [EKS Node Lifecycle During Updates](articles/eks-node-lifecycle-during-updates.md) |
| [Kubernetes Cluster Setup with kubeadm](articles/kubeadm-cluster-setup.md) |
| [How kubeadm Creates a Control Plane (Self-Managed)](articles/kubeadm-control-plane-creation.md) |
| [Kubernetes Distributions: K3s vs MicroK8s vs Minikube vs kubeadm and Others](articles/kubernetes-distributions-comparison.md) |
| [HPA with scaleDown Behavior](articles/kubernetes-hpa-scaledown-behavior.md) |
| [Ingress](articles/kubernetes-ingress-guide.md) |
| [NodePort Services](articles/kubernetes-nodeport-service.md) |
| [HAProxy Ingress Dashboard Metrics](articles/haproxy-ingress-dashboard-metrics.md) |
| [Cron vs CronJob in Kubernetes](articles/kubernetes-cron-vs-cronjob.md) |
| [Kubernetes CronJob Examples & Reference](articles/kubernetes-cronjob-examples.md) |
| [Kubernetes Vertical Pod Autoscaler (VPA)](articles/kubernetes-vpa-guide.md) |
| [In-Place Pod Resize with the VPA](articles/in-place-pod-resize-with-vpa.md) |
| [EKS Node NotReady with I/O and CPU Spikes](articles/eks-node-notready-io-cpu-spikes.md) |
| [Kubernetes Node Disk Pressure](articles/kubernetes-node-disk-pressure.md) |
| [Kubernetes Resource Scheduling & Node Capacity](articles/kubernetes-resource-scheduling-node-capacity.md) |
| [Troubleshooting CrashLoopBackOff with No Logs](articles/kubernetes-crashloopbackoff-no-logs.md) |
| [CrashLoopBackOff Explained](articles/kubernetes-crashloopbackoff-explained.md) |
| [Kubernetes Security Mechanisms](articles/kubernetes-security-mechanisms.md) |
| [Kubernetes Pod Security Standards (PSS)](articles/kubernetes-pod-security-standards.md) |
| [Kubernetes Scheduling](articles/kubernetes-scheduling-guide.md) |
| [Persistent Volumes on EKS with EBS CSI Driver](articles/eks-persistent-volumes-ebs-csi.md) |
| [EBS Volume Available but PV Still Bound](articles/eks-ebs-available-but-pv-bound.md) |
| [Check If Deployments Run the Latest Image](articles/kubernetes-check-latest-image-deployments.md) |
| [Finding the Real Image Version Behind a latest Tag](articles/kubernetes-resolve-running-image-version.md) |
| [Kubernetes Gateway API Guide](articles/kubernetes-gateway-api-guide.md) |
| [Kubelet Privilege and Capability Check](articles/kubelet-privilege-check.md) |
| [Kubernetes Variables Guide](articles/kubernetes-variables-guide.md) |
| [How to Pause a Pod in Kubernetes](articles/kubernetes-pause-pod.md) |
| [kubectl run & expose Guide](articles/kubectl-run-expose-guide.md) |
| [Sidecar Log Agent Pattern](articles/kubernetes-sidecar-log-agent-pattern.md) |
| [Kubernetes Deployment Strategies](articles/kubernetes-deployment-strategies.md) |
| [Kubernetes Production Readiness Checklist](articles/kubernetes-production-readiness-checklist.md) |
| [HAProxy Session Metrics: Frontend vs Backend](articles/haproxy-session-metrics-frontend-backend.md) |
| [EKS Load Balancers: ALB vs NLB](articles/eks-load-balancer-alb-vs-nlb.md) |
| [AWS EKS — CIDR Allocation Reference](articles/eks-cidr-allocation-reference.md) |
| [EKS ENI Allowance Counters (ENA Driver)](articles/eks-ena-allowance-counters.md) |
| [EC2 Network Burst Bandwidth — r7i.2xlarge](articles/ec2-network-burst-bandwidth.md) |
| [EKS Node Network Interfaces and Traffic Flow](articles/eks-node-network-interfaces-traffic-flow.md) |
| [Seeing Network Traffic on EKS Nodes](articles/eks-node-network-traffic-debugging.md) |
| [Kubernetes Cluster Autoscaler Tuning](articles/kubernetes-cluster-autoscaler-tuning.md) |
| [Cluster Autoscaler Scale-Up Troubleshooting](articles/kubernetes-cluster-autoscaler-scale-up-troubleshooting.md) |
| [Cluster Autoscaler Loop: DaemonSets Preempting Overprovisioning Pods](articles/cluster-autoscaler-overprovisioning-daemonset-loop.md) |
| [EKS Traffic Flow: ALB → HAProxy → Pods](articles/eks-traffic-flow-alb-haproxy-pods.md) |
| [Maximum Packets Per Second (PPS) Reference](articles/network-max-pps-reference.md) |
| [Kubernetes allowPrivilegeEscalation Explained](articles/kubernetes-allowprivilegeescalation.md) |
| [EKS aws-auth ConfigMap Guide](articles/eks-aws-auth-configmap-guide.md) |
| [Troubleshooting EKS Access: The 401 That Isn't Kubernetes](articles/eks-access-troubleshooting.md) |
| [ArgoCD Access Methods on EKS](articles/argocd-access-methods-eks.md) |
| [Cluster Autoscaler vs Karpenter for EKS](articles/eks-cluster-autoscaler-vs-karpenter.md) |
| [Cluster Autoscaler on EKS](articles/eks-cluster-autoscaler-setup.md) |
| [Cluster Autoscaler Scale-Down Failures Across EKS Clusters](articles/eks-cluster-autoscaler-scale-down-failures.md) |
| [Kustomize Cheatsheet](articles/kustomize-cheatsheet.md) |
| [Fix Cluster Autoscaler on Hetzner Cloud](articles/hetzner-cluster-autoscaler-fix.md) |
| [EKS Cluster IAM Roles Setup](articles/eks-cluster-iam-roles-setup.md) |
| [Kubernetes Pods vs Deployments](articles/kubernetes-pods-vs-deployments.md) |
| [Uncordon Disabled Nodes in Kubernetes](articles/kubernetes-uncordon-disabled-nodes.md) |
| [VPA and HPA Metrics Collection](articles/kubernetes-vpa-hpa-metrics-collection.md) |
| [kubectl run with Resource Requests & Limits](articles/kubectl-run-resource-requests-limits.md) |
| [Why Pod Shows 0/1 Ready Status](articles/kubernetes-pod-0-1-ready-status.md) |
| [Kubernetes Health Checks: Liveness, Readiness, Startup Probes](articles/kubernetes-health-checks-probes.md) |
| [Time Required for a Pod to Reach Running/Ready](articles/kubectl-pod-time-to-ready.md) |
| [Troubleshooting Workloads When No Pod Shows Up](articles/pod-not-showing-any-state.md) |
| [Troubleshooting a Pending Pod: Insufficient Resources](articles/pod-pending-insufficient-resources.md) |
| [Why Pods Get Throttled Even When Node Has Available CPU](articles/kubernetes-cpu-throttling-available-cpu.md) |
| [Burstable vs. Non-Burstable Instances for Kubernetes](articles/burstable-vs-nonburstable-kubernetes.md) |
| [EKS Node Troubleshooting Guide](articles/eks-node-troubleshooting-guide.md) |
| [Using systemctl to Debug EKS Nodes](articles/eks-node-systemctl-debugging.md) |
| [Cleaning Up Kubernetes Clusters from .kube/config](articles/kubeconfig-cleanup-guide.md) |
| [Fixing k3d TLS Certificate SAN Errors](articles/k3d-tls-san-certificate-fix.md) |
| [EKS Node Groups Explained](articles/eks-node-groups-explained.md) |
| [EKS Fargate](articles/eks-fargate-guide.md) |
| [runAsNonRoot: true](articles/kubernetes-runasnonroot.md) |
| [CoreDNS on EKS — Cheatsheet](articles/coredns-eks-cheatsheet.md) |
| [Troubleshooting DNS Resolution Issues on EKS](articles/eks-dns-troubleshooting.md) |
| [DNS Observability and Troubleshooting in Kubernetes](articles/kubernetes-dns-observability.md) |
| [Velero: Kubernetes Backup and Disaster Recovery](articles/velero-kubernetes-backup-dr.md) |
| [ImagePullBackOff Troubleshooting Guide](articles/kubernetes-imagepullbackoff-troubleshooting.md) |
| [Kubernetes Pod Troubleshooting Guide](articles/kubernetes-pod-troubleshooting-guide.md) |
| [HAProxy Ingress Setup on EKS](articles/haproxy-ingress-eks-setup.md) |
| [HAProxy 5xx Errors During EKS Node Drains](articles/haproxy-5xx-eks-node-drains.md) |
| [Deep Dive: system:masters Group on EKS](articles/eks-system-masters-group.md) |
| [What Happens When You Run kubectl apply](articles/kubectl-apply-internals.md) |
| [Kubernetes Objects vs Resources vs Custom Resources](articles/kubernetes-objects-resources-explained.md) |
| [What Happens When You Run kubectl delete pod](articles/kubectl-delete-pod-internals.md) |
| [CrashLoopBackOff Internals](articles/kubernetes-crashloopbackoff-internals.md) |
| [What Happens During a Rolling Update](articles/kubernetes-rolling-update-internals.md) |
| [What Happens When a Node Goes NotReady](articles/kubernetes-node-notready-internals.md) |
| [How Kubernetes Watches Work — Informers](articles/kubernetes-watches-informers-internals.md) |
| [How RBAC Evaluation Works](articles/kubernetes-rbac-evaluation-internals.md) |
| [How Admission Webhooks Fire](articles/kubernetes-admission-webhooks-internals.md) |
| [What Happens When You Create a Service](articles/kubernetes-service-creation-internals.md) |
| [What Happens When a Pod Gets an IP](articles/kubernetes-pod-ip-assignment-internals.md) |
| [What Happens When a Pod is Unschedulable](articles/kubernetes-unschedulable-pod-internals.md) |
| [What Happens When You Scale a Deployment](articles/kubernetes-scale-deployment-internals.md) |
| [How kubectl Discovers and Resolves API Resources](articles/kubectl-api-discovery-internals.md) |
| [How kubectl exec Works](articles/kubectl-exec-internals.md) |
| [Migrating Streaming from SPDY to WebSockets](articles/kubernetes-spdy-to-websockets.md) |
| [Checking Client vs Server API Support](articles/kubectl-client-vs-server-api-support.md) |
| [How kubectl port-forward Works](articles/kubectl-port-forward-internals.md) |
| [kubectl wait and Condition-Based Scripting](articles/kubectl-wait-condition-scripting.md) |
| [kubectl debug — Ephemeral Containers and Node Debugging](articles/kubectl-debug-ephemeral-containers.md) |
| [kubectl Server-Side vs Client-Side Operations](articles/kubectl-server-side-vs-client-side.md) |
| [How NetworkPolicies Are Enforced](articles/kubernetes-networkpolicy-enforcement-internals.md) |
| [How Ingress Controllers Work Internally](articles/kubernetes-ingress-controller-internals.md) |
| [What Happens When a PV Is Reclaimed](articles/kubernetes-pv-reclaim-internals.md) |
| [CSI Driver Architecture](articles/kubernetes-csi-driver-architecture.md) |
| [How ServiceAccount Tokens Work](articles/kubernetes-serviceaccount-tokens-internals.md) |
| [How Pod Security Admission Enforces Policies](articles/kubernetes-pod-security-admission-internals.md) |
| [How kubectl drain Works Internally](articles/kubectl-drain-internals.md) |
| [How etcd Stores and Retrieves Kubernetes Objects](articles/kubernetes-etcd-storage-internals.md) |
| [How the Garbage Collector Works](articles/kubernetes-garbage-collector-internals.md) |
| [How Leader Election Works in Controllers](articles/kubernetes-leader-election-internals.md) |
| [Separate Kubeconfig Files for EKS Clusters](articles/eks-separate-kubeconfig-files.md) |
| [EKS API Server Throttling and Priority & Fairness](articles/eks-api-priority-fairness.md) |
| [EKS Pod Identity vs IRSA — Deep Dive](articles/eks-pod-identity-vs-irsa-deep-dive.md) |
| [Cross-AZ Traffic in EKS — Patterns, Costs, and Optimization](articles/eks-cross-az-traffic-optimization.md) |
| [EKS Version Upgrades — Checklist and Process](articles/eks-version-upgrade-checklist.md) |
| [Upgrading an EKS Control Plane from 1.34 to 1.35](articles/eks-control-plane-upgrade-1.34-to-1.35.md) |
| [Amazon EKS Cluster Insights](articles/eks-cluster-insights.md) |
| [EKS Add-on Management Guide](articles/eks-addon-management-guide.md) |
| [EKS Node Bootstrap Deep Dive](articles/eks-node-bootstrap-deep-dive.md) |
| [EKS: pre_userdata vs additional_userdata](articles/eks-pre-userdata-vs-additional-userdata.md) |
| [Bottlerocket OS for EKS](articles/bottlerocket-os-eks-guide.md) |
| [TLS/SSL Certificates Explained](articles/tls-ssl-certificates-explained.md) |
| [OAuth2 and OIDC Flow Explained](articles/oauth2-oidc-flow-explained.md) |
| [How kube-proxy Works — iptables vs IPVS vs nftables](articles/kubernetes-kube-proxy-internals.md) |
| [Kubernetes DNS Deep Dive — CoreDNS Architecture](articles/kubernetes-coredns-deep-dive.md) |
| [DNS Policies for Pods in Kubernetes](articles/kubernetes-pod-dns-policies.md) |
| [GitHub Actions for Kubernetes Deployments](articles/github-actions-kubernetes-deployments.md) |
| [Pipeline to Get the Latest Ubuntu EKS AMI](articles/eks-ubuntu-ami-latest-pipeline.md) |
| [ConfigMaps and Secrets](articles/kubernetes-configmaps-secrets.md) |
| [Resource Quotas & LimitRanges](articles/kubernetes-resource-quotas-limitranges.md) |
| [Kubeadm Cluster Upgrade](articles/kubeadm-cluster-upgrade.md) |
| [HPA with scaleDown Behavior](articles/hpa-scaledown-behavior.md) |
| [HPA ScalingLimited (TooManyReplicas)](articles/hpa-scaling-limited-too-many-replicas.md) |
| [CPU Starvation Diagnostic Guide](articles/cpu-starvation-diagnostic-guide.md) |
| [Installing metrics-server](articles/metrics-server-install.md) |

### Helm

| Article |
|---------|
| [Helm Cheatsheet](articles/helm-cheatsheet.md) |
| [What's New in Helm 4](articles/helm-4-whats-new.md) |
| [Kustomize vs Helm](articles/kustomize-vs-helm.md) |
| [Installing Helm Charts Without helm repo add](articles/helm-install-without-repo-add.md) |
| [Replacing an Existing Deployment with a Helm Chart](articles/helm-overwrite-existing-deployment.md) |

### EKS Auto Mode

| Article |
|---------|
| [EKS Auto Mode](articles/eks-auto-mode.md) |
| [EKS Auto Mode Cheatsheet](articles/eks-auto-mode-cheatsheet.md) |
| [EKS Auto Mode Security Deep Dive](articles/eks-auto-mode-security.md) |
| [Deploy 2048 Game on EKS Auto Mode](articles/eks-auto-mode-2048-game.md) |
| [Troubleshoot DNS in EKS Auto Mode](articles/eks-auto-mode-dns-troubleshooting.md) |
| [Troubleshoot Custom NodePool and NodeClass in EKS Auto Mode](articles/eks-auto-mode-nodepool-nodeclass-troubleshooting.md) |

### Docker

| Article |
|---------|
| [Docker Cheatsheet](articles/docker-cheatsheet.md) |
| [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md) |
| [Docker Compose: ports vs expose](articles/docker-ports-vs-expose.md) |
| [Docker Compose: Running Containers Without Root](articles/docker-compose-non-root.md) |
| [Fix Gitea Runner Docker Hub Rate Limits](articles/docker-gitea-runner-fix.md) |
| [dbash — Docker Shell Function](articles/docker-dbash-function.md) |
| [Building Docker Images with Dockerfile](articles/docker-build-image-guide.md) |
| [Docker Buildx and Multi-Platform Builds](articles/docker-buildx-multi-platform-builds.md) |
| [Docker Restart Policies](articles/docker-restart-policies.md) |
| [Docker Compose Build Configuration and Build Context](articles/docker-compose-build-configuration.md) |
| [Configuring DNS in Docker Compose](articles/docker-compose-dns-configuration.md) |
| [Docker Compose depends_on: Startup Order and Readiness](articles/docker-compose-depends-on.md) |
| [Tagging Built Images in Docker Compose](articles/docker-compose-image-tagging.md) |
| [Updating Docker Compose Containers](articles/docker-compose-updating-containers.md) |
| [Docker Healthcheck Examples](articles/docker-healthcheck-examples.md) |
| [Docker Compose vs Docker Swarm](articles/docker-compose-vs-swarm.md) |
| [Rebuilding Docker Compose Images and Containers](articles/docker-compose-rebuild-images.md) |
| [SUID, SGID, and Capabilities in Docker](articles/docker-suid-sgid-capabilities.md) |
| [Docker Swarm Cheatsheet](articles/docker-swarm-cheatsheet.md) |
| [Docker Swarm Storage](articles/docker-swarm-storage.md) |
| [Shared Storage Options for Docker Swarm](articles/docker-swarm-storage-options.md) |
| [Docker Overlay2 Storage Driver](articles/docker-overlay2-storage.md) |
| [Move the Docker Data Directory (data-root)](articles/docker-move-data-root.md) |
| [Pin Docker Image Versions Instead of latest](articles/docker-image-version-pinning.md) |
| [Docker Compose: Bind Mounts vs Named Volumes](articles/docker-compose-volumes-vs-bind-mounts.md) |
| [Set the Container User (UID/GID) in Docker Compose](articles/docker-compose-set-container-user.md) |
| [Defining Variables in Docker Compose](articles/docker-compose-variables-env.md) |
| [Where to Store Docker Compose Bind-Mount Data on the Host](articles/docker-compose-host-data-layout.md) |
| [Fixing Docker Bind-Mount Permission Errors](articles/docker-bind-mount-permissions.md) |
| [Installing Podman on RHEL 7–10](articles/podman-installation-rhel.md) |
| [Fixing Critical Vulnerabilities in Public Docker Images](articles/docker-fix-critical-vulnerabilities.md) |
| [Docker Management UIs: Portainer vs Dockge vs Dockhand and Others](articles/docker-management-uis-comparison.md) |

### AWS

| Article |
|---------|
| [AWS CLI Installation](articles/aws-cli-install.md) |
| [How to Use the AWS Free Tier Effectively](articles/aws-free-tier-guide.md) |
| [AWS Login: Simplified Developer Access](articles/aws-login-command.md) |
| [AWS STS Assume Role with MFA](articles/aws-sts-assume-role.md) |
| [Assume an IAM Role via CLI (Step by Step)](articles/aws-assume-role-cli-walkthrough.md) |
| [AWS AssumeRole Concepts](articles/aws-assume-role-concepts.md) |
| [AWS IAM Concepts Guide](articles/aws-iam-concepts-guide.md) |
| [IAM: Access Keys vs Roles vs Instance Profiles](articles/aws-iam-access-keys-vs-roles-vs-instance-profiles.md) |
| [AWS IAM CLI Cheatsheet](articles/aws-iam-cheatsheet.md) |
| [AWS IAM Role Users Audit](articles/aws-iam-role-users-audit.md) |
| [Temporarily Disabling AWS Credentials Safely](articles/aws-credentials-disable-with-trap.md) |
| [ECS Cluster Architecture](articles/ecs-architecture-guide.md) |
| [AWS EFS Cheatsheet](articles/aws-efs-cheatsheet.md) |
| [AWS Lightsail Cheatsheet](articles/aws-lightsail-cheatsheet.md) |
| [AWS Load Balancer Cheatsheet](articles/aws-elb-cheatsheet.md) |
| [AWS VPC Cheatsheet](articles/aws-vpc-cheatsheet.md) |
| [AWS VPC Design Guide](articles/aws-vpc-design-guide.md) |
| [AWS CloudFormation Cheatsheet](articles/aws-cloudformation-cheatsheet.md) |
| [AWS API Throttling Guide](articles/aws-api-throttling-guide.md) |
| [AWS Well-Architected Framework](articles/aws-well-architected-framework.md) |
| [AWS Migration 7 R's](articles/aws-migration-7rs.md) |
| [AWS Route 53 Cheatsheet](articles/aws-route53-cheatsheet.md) |
| [AWS ECR Cheatsheet](articles/aws-ecr-cheatsheet.md) |
| [Pulling Images from ECR with ctr and Docker](articles/ecr-pull-with-ctr-docker.md) |
| [EC2 Cheatsheet](articles/aws-ec2-cheatsheet.md) |
| [EC2 Extend EBS Volume](articles/aws-ec2-extend-disk.md) |
| [EC2 Instance Metadata Service (IMDS)](articles/aws-ec2-metadata.md) |
| [EC2 User Data: Serving Instance Metadata on a Web Page](articles/ec2-userdata-instance-metadata-webpage.md) |
| [EC2 vs ELB Health Checks](articles/aws-ec2-vs-elb-health-checks.md) |
| [EBS Cheatsheet](articles/aws-ebs-cheatsheet.md) |
| [Benchmarking Amazon EBS Volumes with FIO](articles/aws-ebs-benchmarking-fio.md) |
| [S3 Cheatsheet](articles/aws-s3-cheatsheet.md) |
| [Unattached EBS Volumes: Detection and Monitoring](articles/aws-ebs-unattached-volumes.md) |
| [EC2 fstab: Why Device Names Change on Nitro](articles/aws-ec2-fstab-labels.md) |
| [Installing SSM Agent](articles/aws-ssm-agent-install.md) |
| [JMESPath Query Guide](articles/aws-jmespath-guide.md) |
| [AWS CLI Tag Filtering with Variables](articles/aws-cli-tag-filtering-variables.md) |
| [EKS Authentication Modes: ConfigMap vs Access Entries](articles/eks-authentication-modes.md) |
| [Securing Kubernetes Containers: Security Contexts](articles/eks-security-contexts.md) |
| [EKS Node Groups: With and Without Launch Templates](articles/eks-nodegroups-launch-templates.md) |
| [EKS AMI Comparison: Ubuntu vs Ubuntu Pro vs Amazon Linux](articles/eks-ubuntu-ami-comparison.md) |
| [EKS Architecture Deep Dive](articles/eks-architecture-deep-dive.md) |
| [How the EKS Control Plane Is Created and Reached](articles/eks-control-plane-architecture.md) |
| [EKS VPC CNI: IPAMD Guide](articles/eks-vpc-cni-ipamd-guide.md) |
| [EKS VPC CNI Proxy Configuration](articles/eks-vpc-cni-proxy-configuration.md) |
| [eksctl Cheatsheet](articles/eksctl-cheatsheet.md) |
| [Creating an EKS Cluster with eksctl](articles/eks-cluster-with-eksctl.md) |
| [AWS CLI EKS Commands](articles/aws-eks-cli-cheatsheet.md) |
| [EKS Node Not Joining Cluster](articles/eks-node-not-joining-troubleshooting.md) |
| [EKS Node Monitoring](articles/eks-node-monitoring.md) |
| [Why aws-auth Looks Different on Different Clusters](articles/eks-aws-auth-why-different.md) |
| [aws-auth ConfigMap Technical Details](articles/eks-aws-auth-technical-details.md) |
| [Validating cloud-init on EKS Nodes](articles/eks-cloud-init-validation.md) |
| [Cluster Autoscaler Restarts Troubleshooting](articles/eks-cluster-autoscaler-restarts.md) |
| [HAProxy on EKS with NLB](articles/eks-haproxy-nlb.md) |
| [EKS Node Health and Auto-Repair](articles/eks-node-health-auto-repair.md) |
| [EKS vs AKS vs GKE](articles/eks-vs-aks-vs-gke.md) |
| [EKS Node Group Rolling Updates](articles/eks-node-group-rolling-updates.md) |
| [EKS EC2 Tags vs Node Selectors](articles/eks-ec2-tags-vs-node-selectors.md) |
| [Karpenter Guide](articles/karpenter-guide.md) |

### Azure

| Article |
|---------|
| [Azure CLI Cheatsheet](articles/azure-cli-cheatsheet.md) |
| [Azure Resource Groups Cheatsheet](articles/azure-resource-groups-cheatsheet.md) |
| [Azure Networking Cheatsheet](articles/azure-networking-cheatsheet.md) |
| [Azure Storage Cheatsheet](articles/azure-storage-cheatsheet.md) |
| [Azure VM Management Cheatsheet](articles/azure-vm-cheatsheet.md) |
| [AKS Cheatsheet](articles/azure-aks-cheatsheet.md) |
| [Azure Key Vault, Monitoring, IAM, and App Services](articles/azure-keyvault-monitoring-iam-cheatsheet.md) |
| [Azure VM Instance Types and Free Tier](articles/azure-vm-instance-types-free-tier.md) |

### GCP

| Article |
|---------|
| [GCP Compute Engine with jq Cheatsheet](articles/gcloud-compute-jq-cheatsheet.md) |
| [gcloud CLI Cheatsheet](articles/gcloud-cheatsheet.md) |

### DigitalOcean

| Article |
|---------|
| [doctl Cheatsheet](articles/doctl-cheatsheet.md) |
| [HAProxy for Kubernetes on DigitalOcean](articles/haproxy-kubernetes-digitalocean.md) |
| [DOKS Node Pools](articles/doks-node-pools.md) |

### Hetzner

| Article |
|---------|
| [hcloud CLI Cheatsheet](articles/hcloud-cheatsheet.md) |
| [hetzner-k3s Cheatsheet](articles/hetzner-k3s-cheatsheet.md) |

### Virtualization

| Article |
|---------|
| [Vagrant Cheatsheet](articles/vagrant-cheatsheet.md) |
| [Increasing Vagrant Box Disk Space on Provisioning](articles/vagrant-increase-disk-size-provisioning.md) |
| [Adding an SSH Public Key to a Vagrant VM](articles/vagrant-add-ssh-public-key.md) |
| [Installing KVM](articles/kvm-installation.md) |
| [KVM / virsh Cheatsheet](articles/kvm-cheatsheet.md) |
| [Adding a New Disk in KVM](articles/kvm-add-disk.md) |
| [Enable virsh console](articles/kvm-virsh-console.md) |
| [Running virt-manager Remotely](articles/virt-manager-remote-display.md) |
| [Installing KVM Guests](articles/kvm-guest-installation.md) |
| [Using Cloud qcow2 Images with KVM](articles/kvm-qcow2-cloud-images.md) |
| [Enable SSH Password Auth in Ubuntu Cloud Images](articles/ubuntu-cloud-image-ssh-password.md) |
| [Converting VMware VMs to KVM](articles/kvm-convert-vmware-to-kvm.md) |
| [KVM libguestfs Tools](articles/kvm-libguestfs-tools.md) |
| [VirtualBox CLI Cheatsheet](articles/virtualbox-cheatsheet.md) |

### Proxmox

| Article |
|---------|
| [Proxmox Cheatsheet](articles/proxmox-cheatsheet.md) |
| [Importing OVA/qcow2 into Proxmox](articles/proxmox-import-ova-qcow2.md) |
| [Troubleshooting cloud-init on Proxmox](articles/proxmox-cloud-init-troubleshooting.md) |
| [Resize a Partition on Proxmox](articles/proxmox-resize-partition.md) |
| [QEMU Guest Agent on Proxmox](articles/proxmox-qemu-guest-agent.md) |
| [Changing the Root Password in Proxmox LXC Containers](articles/proxmox-lxc-change-root-password.md) |
| [Setting the LXC Root Password with cloud-init Userdata](articles/proxmox-lxc-cloud-init-root-password.md) |
| [Proxmox Two-Node Cluster Quorum](articles/proxmox-two-node-cluster-quorum.md) |
| [Why Proxmox Needs libguestfs-tools](articles/proxmox-libguestfs-tools.md) |
| [Migrating VMs and LXC Containers Between Proxmox Nodes](articles/proxmox-migrate-vms-containers.md) |
| [Proxmox xterm.js Serial Console](articles/proxmox-xtermjs-serial-console.md) |
| [Proxmox ACME SSL with Hetzner DNS](articles/proxmox-acme-hetzner-dns.md) |
| [Setting Up a Brand-New Proxmox Server: First Tools to Install](articles/proxmox-first-tools-to-install.md) |
| [ProxMenux Monitor: Guide, One-Liners, Tips & Tricks](articles/proxmenux-monitor-guide.md) |

### Terraform

| Article |
|---------|
| [Terraform Cheatsheet](articles/terraform-cheatsheet.md) |
| [Packer Cheatsheet](articles/packer-cheatsheet.md) |
| [terraform.tfstate vs .terraform/terraform.tfstate](articles/terraform-tfstate-vs-terraform-directory-state.md) |
| [terraform init -upgrade and Constraints](articles/terraform-init-upgrade-and-constraints.md) |
| [terraform get -update vs init -upgrade](articles/terraform-get-update-vs-init-upgrade.md) |
| [Terraform Lock File Checksums: zh and h1](articles/terraform-lock-file-checksums.md) |
| [Terraform Root Module vs Child Modules](articles/terraform-root-vs-child-modules.md) |
| [EOF Escaping in Userdata, Terraform, and Shell Scripts](articles/eof-escaping-userdata-terraform.md) |
| [Migrating State Off Terraform Cloud](articles/terraform-migrate-state-off-terraform-cloud.md) |
| [Where Terraform Can Store State: Backends Guide](articles/terraform-state-storage-backends.md) |
| [Terraform tfvars: Variable Definitions Reference](articles/terraform-tfvars-guide.md) |
| [Terraform Variables: Declaration, Validation, and Usage](articles/terraform-variables-guide.md) |
| [Importing Existing Infrastructure Into Terraform](articles/terraform-import-guide.md) |
| [Exporting Datadog Monitors to Terraform](articles/exporting-monitors-to-terraform.md) |
| [Terraform Backend Configuration Changed](articles/terraform-backend-configuration-changed.md) |
| [Terraform Conditional Expressions](articles/terraform-conditional-expressions.md) |
| [Terraform Provisioners Guide](articles/terraform-provisioners-guide.md) |
| [Terraform Outputs Guide](articles/terraform-outputs-guide.md) |
| [Managing DNS with Terraform and BIND](articles/terraform-bind-dns-management.md) |
| [Terraform JSON, One-Liners, and Tips](articles/terraform-json-tips-tricks.md) |
| [Terraform Troubleshooting](articles/terraform-troubleshooting.md) |
| [Terraform Debugging Guide](articles/terraform-debugging-guide.md) |
| [Terraform Heredoc: EOT vs EOF](articles/terraform-heredoc-eot-eof.md) |
| [Terraform toset() and for_each Guide](articles/terraform-toset-foreach-guide.md) |
| [Escaping $ in Terraform Userdata](articles/terraform-userdata-dollar-escaping.md) |
| [Terraform Lifecycle Guide](articles/terraform-lifecycle-guide.md) |
| [Terraform Config Drift Detection](articles/terraform-drift-detection.md) |
| [Understanding `<=`, `+`, and Other Signs in a Terraform Plan](articles/terraform-plan-symbols.md) |
| [Terraform UserData Base64 Encoding/Decoding](articles/terraform-userdata-base64.md) |

### Ansible

| Article |
|---------|
| [Ansible Ad-Hoc Commands](articles/ansible-adhoc-cheatsheet.md) |
| [Ansible Cheatsheet](articles/ansible-cheatsheet.md) |
| [Ansible Inventory Guide](articles/ansible-inventory-guide.md) |
| [Ansible Run Command Modules](articles/ansible-shell-command-modules.md) |
| [Ansible Sudo: Ubuntu vs RHEL](articles/ansible-sudo-ubuntu-vs-rhel.md) |
| [Ansible Vault: Storing Sudo Passwords](articles/ansible-vault-become-pass.md) |
| [Ansible Python Interpreter](articles/ansible-python-interpreter.md) |
| [Ansible Ad-Hoc Commands vs Playbooks](articles/ansible-adhoc-vs-playbooks.md) |
| [Ansible: Editing Files and Creating Scripts](articles/ansible-file-editing-creation.md) |
| [Ansible Configuration: ansible.cfg Guide](articles/ansible-cfg-guide.md) |
| [Ansible Roles Directory Structure](articles/ansible-roles-directory-structure.md) |
| [ansible-lint Guide](articles/ansible-lint-guide.md) |

### Git

| Article |
|---------|
| [Git Cheatsheet](articles/git-cheatsheet.md) |
| [GitHub CLI (gh) Cheatsheet](articles/gh-cli-cheatsheet.md) |
| [Deleting GitHub Deployments with the gh CLI](articles/github-delete-deployments.md) |
| [Staging Changes in Git: git add . vs -A vs -u](articles/git-add-dot-vs-all.md) |
| [Deleting Git Branches: Local, Remote, and Cleanup](articles/git-delete-branches.md) |
| [Getting the Latest Changes from Master into Your Feature Branch](articles/git-update-feature-branch-from-master.md) |
| [Fixing "Binary files differ" in git diff](articles/git-diff-binary-files-differ.md) |
| [Removing Untracked Files with git clean](articles/git-clean-untracked-files.md) |
| [Aborting and Investigating Merge Conflicts](articles/git-abort-investigate-merge-conflicts.md) |
| [Showing Commits in Your Branch That Aren't in Master](articles/git-commits-in-branch-not-master.md) |
| [Git Credential Helpers: Storing Passwords and Tokens](articles/git-credential-helpers.md) |
| [Detecting and Fixing Whitespace Errors in Git](articles/git-whitespace-errors-check-fix.md) |
| [Fixing "git apply" Whitespace Errors](articles/git-apply-whitespace-errors.md) |
| [Understanding HEAD in Git](articles/git-head-explained.md) |
| [Undoing a Pushed Commit: revert vs reset](articles/git-undo-pushed-commit.md) |
| [Creating and Applying Git Patch Files](articles/git-create-apply-patches.md) |
| [git push vs git push origin HEAD](articles/git-push-vs-push-origin-head.md) |
| [Viewing Unpushed Commits in Git](articles/git-view-unpushed-commits.md) |
| [Viewing a File's Change History and Blame](articles/git-file-history-blame.md) |
| [Downloading GitHub Release Assets from the CLI](articles/download-github-release-assets.md) |
| [Comparing Your Branch Against Master with git diff](articles/git-diff-branch-against-master.md) |
| [Resetting a Local Branch to Match the Remote](articles/git-reset-local-to-remote.md) |
| [Removing a Local Commit: reset, revert, and rebase](articles/git-remove-local-commit.md) |
| [Running a Script Directly from a GitHub Gist](articles/run-script-from-github-gist.md) |
| [Git Clone Methods and Options](articles/git-clone-methods.md) |
| [Fixing Shell Script Execute Permissions Across Windows and Linux](articles/git-shell-script-executable-permissions.md) |
| [Restoring Files with git restore](articles/git-restore-files.md) |
| [Restoring a Deleted File from a Repository](articles/git-restore-deleted-file.md) |
| [Listing Git Branches by Author](articles/git-list-branches-by-author.md) |
| [Gitea / Forgejo Actions vs GitHub Actions Compatibility](articles/gitea-forgejo-actions-github-compatibility.md) |
| [Setting Up a Gitea Actions Runner (act_runner)](articles/gitea-act-runner-setup.md) |
| [Editing a Commit Message](articles/git-edit-commit-message.md) |
| [Changing a Commit Message After Pushing (No One Has Pulled)](articles/git-change-commit-message-after-push.md) |
| [Blocking Commits with Trailing Whitespace](articles/git-block-trailing-whitespace-commits.md) |
| [Normalizing Line Endings with .gitattributes](articles/gitattributes-line-endings.md) |
| [Git Stash Guide](articles/git-stash-guide.md) |
| [Sharing a Branch: Push, Then Fetch vs Pull](articles/git-share-branch-push-fetch-pull.md) |
| [Undoing Changes in Git: reset vs checkout vs revert](articles/git-undo-reset-checkout-revert.md) |
| [git reset Guide: soft, mixed, and hard](articles/git-reset-guide.md) |
| [Git Hooks Guide](articles/git-hooks-guide.md) |
| [Git Merge vs Rebase](articles/git-merge-vs-rebase.md) |
| [Git Shell Functions and Aliases](articles/git-shell-functions.md) |
| [git rebase Guide](articles/git-rebase-guide.md) |
| [Restoring a Repository to a Past Commit's State](articles/git-restore-to-past-commit.md) |
| [Syncing a Branch Across Multiple Computers](articles/git-sync-branch-across-computers.md) |
| [Git SSH Keys and Credential Storage](articles/git-ssh-keys-credential-storage.md) |
| [Canceling a git revert](articles/git-cancel-revert.md) |

### GitLab and GitHub

| Article |
|---------|
| [GitLab vs GitHub Cheatsheet](articles/gitlab-vs-github-cheatsheet.md) |
| [Testing a GitLab Runner with a Standalone Pipeline](articles/gitlab-runner-test-pipeline.md) |
| [Switching a GitLab Runner to the Shell Executor](articles/gitlab-runner-shell-executor.md) |
| [Canceling GitLab Runner Jobs and Cleaning Up](articles/gitlab-runner-cancel-jobs-cleanup.md) |
| [Continuously Mirror a GitHub Repository into GitLab (CI/CD)](articles/gitlab-ci-mirror-from-github.md) |

### Bash and Shell

| Article |
|---------|
| [bash Cheatsheet](articles/bash-cheatsheet.md) |
| [Korn Shell (ksh) Cheatsheet](articles/ksh-cheatsheet.md) |
| [Bash Essentials Guide](articles/bash-essentials-guide.md) |
| [Safely Download and Run Installation Scripts](articles/download-run-install-scripts.md) |
| [Bash Pipelines and Redirections](articles/bash-redirection-operators.md) |
| [Bash History Guide](articles/bash-history-guide.md) |
| [Bash Test Conditions: \[ \] vs \[\[ \]\]](articles/bash-test-conditions-guide.md) |
| [Bash Single vs Double Brackets](articles/bash-single-vs-double-brackets.md) |
| [Bash Subshells](articles/bash-subshells-guide.md) |
| [Bash Troubleshooting Guide](articles/bash-troubleshooting-guide.md) |
| [sed Replace Line Guide](articles/sed-replace-line-guide.md) |
| [Running Multiple Commands with sudo](articles/sudo-multiple-commands.md) |
| [sudoers Guide](articles/sudo-sudoers-guide.md) |
| [Vim White Spaces](articles/vim-white-spaces.md) |
| [Cron Cheatsheet](articles/cron-cheatsheet.md) |
| [Bash Aliases and Functions](articles/bash-aliases-functions.md) |
| [Bash Read Builtin](articles/bash-read-builtin.md) |
| [Bash While Loop Examples](articles/bash-while-loops-examples.md) |
| [Bash Loops Guide: for, while, until, select](articles/bash-loops-guide.md) |
| [Ignoring Command Errors with \|\| true](articles/bash-ignore-command-errors.md) |
| [awk Cheatsheet](articles/awk-cheatsheet.md) |
| [Print Column Numbers for Any Command Output](articles/awk-print-column-numbers.md) |
| [sed Cheatsheet](articles/sed-cheatsheet.md) |
| [Vim Search and Replace](articles/vim-search-replace.md) |
| [Display Tabs and Whitespace in Files](articles/display-tabs-whitespace.md) |
| [ShellCheck Guide](articles/shellcheck-guide.md) |
| [Linux Job Control](articles/linux-job-control.md) |

### Linux System Administration

| Article |
|---------|
| [systemd Cheatsheet](articles/systemd-cheatsheet.md) |
| [journalctl Cheatsheet](articles/journalctl-cheatsheet.md) |
| [Enable Persistent systemd Journal Logging](articles/systemd-journal-persistent-logging.md) |
| [dpkg Cheatsheet](articles/dpkg-cheatsheet.md) |
| [apt Cheatsheet](articles/apt-cheatsheet.md) |
| [Configure Automatic Updates with unattended-upgrades](articles/unattended-upgrades-guide.md) |
| [apt vs apt-get](articles/apt-vs-apt-get.md) |
| [Fixing "Packages Have Been Kept Back" on Ubuntu](articles/apt-packages-kept-back.md) |
| [Aptitude Cheatsheet](articles/aptitude-cheatsheet.md) |
| [debsums Cheatsheet](articles/debsums-cheatsheet.md) |
| [Snap Cheatsheet](articles/snap-cheatsheet.md) |
| [Ubuntu Repositories Guide](articles/ubuntu-repositories-guide.md) |
| [Finding Old Package Versions on Ubuntu](articles/ubuntu-old-package-versions.md) |
| [Installing Node.js on Ubuntu 22.04 and 24.04](articles/install-nodejs-ubuntu.md) |
| [Fixing apt Lock Held Errors](articles/apt-lock-held-fix.md) |
| [Troubleshoot APT NOSPLIT and Excess Data Errors Behind a Proxy](articles/apt-nosplit-proxy-troubleshooting.md) |
| [DEBIAN_FRONTEND for Scripts](articles/debian-frontend-noninteractive.md) |
| [Linux File Permissions Guide](articles/linux-file-permissions.md) |
| [SELinux Cheatsheet](articles/selinux-cheatsheet.md) |
| [OpenSCAP Security Compliance Guide](articles/openscap-guide.md) |
| [Linux Audit (auditd) Cheatsheet](articles/auditd-cheatsheet.md) |
| [Postfix Gmail SMTP Relay Setup](articles/postfix-gmail-relay.md) |
| [psacct / acct Cheatsheet](articles/psacct-cheatsheet.md) |
| [/bin/false vs /sbin/nologin](articles/bin-false-vs-nologin.md) |
| [Linux User Quotas](articles/linux-user-quotas.md) |
| [User Administration on RHEL](articles/user-administration.md) |
| [LDAP Client Configuration](articles/ldap-client-configuration.md) |
| [NSCD and SSSD Guide](articles/nscd-sssd-guide.md) |
| [Configure Samba](articles/samba-configuration.md) |
| [MySQL LDAP Authentication](articles/mysql-ldap-authentication.md) |
| [Installing MediaWiki on RHEL](articles/mediawiki-installation-rhel.md) |
| [Reinstall and Restore MediaWiki on RHEL](articles/mediawiki-reinstallation-rhel.md) |
| [Installing DokuWiki on RHEL](articles/dokuwiki-installation-rhel.md) |
| [RHEL LAMP Stack Setup](articles/rhel-lamp-stack-setup.md) |
| [Apache HTTP Server Administration on RHEL](articles/apache-http-server-administration-rhel.md) |
| [Protect SSH with fail2ban](articles/fail2ban-ssh-protection.md) |
| [ReaR Backup Guide](articles/rear-backup-guide.md) |
| [Veeam Agent for Linux](articles/veeam-agent-linux.md) |
| [Installing MariaDB](articles/mariadb-installation.md) |
| [RHEL Releases Overview](articles/rhel-releases-overview.md) |
| [Timezone Configuration](articles/timezone-configuration.md) |
| [subscription-manager Cheatsheet](articles/subscription-manager-cheatsheet.md) |
| [dnf / yum Cheatsheet](articles/dnf-yum-cheatsheet.md) |
| [RPM Cheatsheet](articles/rpm-cheatsheet.md) |
| [RPM Building Guide](articles/rpm-building-guide.md) |
| [RHEL Boot Modes and Troubleshooting](articles/rhel-boot-troubleshooting.md) |
| [GRUB2 Cheatsheet](articles/grub-cheatsheet.md) |
| [initramfs vs initrd](articles/initramfs-vs-initrd.md) |
| [Rebuild initramfs in RHEL](articles/rebuild-initramfs-rhel.md) |
| [chroot Guide](articles/chroot-guide.md) |
| [LUKS Disk Encryption and NBDE (Tang/Clevis)](articles/luks-nbde-encryption.md) |
| [Chroot SFTP Setup](articles/chroot-sftp-setup.md) |
| [Linux Kernel Panics](articles/linux-kernel-panics.md) |
| [Why Processes in D State Can't Be Killed](articles/linux-processes-d-state.md) |
| [fuser Cheatsheet](articles/fuser-cheatsheet.md) |
| [Linux Capabilities](articles/linux-capabilities.md) |
| [Linux System Calls](articles/linux-syscalls.md) |
| [Linux Kernel Map](articles/linux-kernel-map.md) |
| [Linux SysRq Guide](articles/linux-sysrq-guide.md) |
| [Linux ulimit Guide](articles/linux-ulimit-guide.md) |
| [LD_LIBRARY_PATH and Shared Libraries](articles/linux-ld-library-path.md) |
| [sosreport Guide](articles/sosreport-guide.md) |
| [RHEL Post-Installation Steps](articles/rhel-post-installation.md) |
| [Linux cgroups and Kubernetes](articles/linux-cgroups-kubernetes.md) |

### Server Hardware and BMC

| Article |
|---------|
| [Dell racadm and OMSA Cheatsheet](articles/dell-racadm-omsa-cheatsheet.md) |
| [HP iLO CLI Cheatsheet (SSH / SMASH CLP)](articles/hp-ilo-cli-cheatsheet.md) |

### Satellite and Foreman

| Article |
|---------|
| [Hammer CLI Cheatsheet](articles/hammer-cheatsheet.md) |
| [Installing Foreman with Katello](articles/foreman-katello-installation.md) |
| [Installing Satellite from ISO](articles/satellite-installation-iso.md) |
| [Foreman Remote Execution Setup](articles/foreman-remote-execution.md) |
| [Content Views and Activation Keys Strategy](articles/foreman-content-views-activation-keys.md) |
| [Registering Hosts in Foreman](articles/foreman-host-registration.md) |
| [Red Hat Insights](articles/red-hat-insights.md) |
| [Satellite Remote Execution Setup](articles/satellite-remote-execution-setup.md) |

### Linux Performance and IO

| Article |
|---------|
| [Linux Load Average](articles/linux-load-average.md) |
| [Linux CPU Steal Time](articles/linux-cpu-steal.md) |
| [Linux I/O Schedulers](articles/linux-io-schedulers.md) |
| [Linux Disk I/O Internals](articles/linux-disk-io-internals.md) |
| [blktrace Guide](articles/blktrace-guide.md) |
| [Configuring sysstat on Ubuntu](articles/configuring-sysstat-ubuntu.md) |
| [sysstat / sar Cheatsheet](articles/sysstat-sar-cheatsheet.md) |
| [Understanding vmstat Output](articles/understanding-vmstat-output.md) |
| [iostat Cheatsheet](articles/iostat-cheatsheet.md) |
| [Understanding iostat -x Output](articles/understanding-iostat-x-output.md) |
| [iotop Cheatsheet](articles/iotop-cheatsheet.md) |
| [ps Cheatsheet](articles/ps-cheatsheet.md) |
| [top Cheatsheet](articles/top-cheatsheet.md) |
| [free Cheatsheet](articles/free-cheatsheet.md) |
| [Performance Co-Pilot (PCP) Cheatsheet](articles/pcp-cheatsheet.md) |
| [tuned-adm Cheatsheet](articles/tuned-adm-cheatsheet.md) |
| [NUMA Tuning Guide](articles/numa-tuning-guide.md) |
| [NUMA Performance Tuning on RHEL 7-10](articles/rhel-numa-performance-tuning.md) |
| [RHEL Performance Analysis and VM Tuning](articles/rhel-performance-analysis-vm-tuning.md) |
| [Installing Nagios Core from Source](articles/nagios-installation-source.md) |
| [Nagios check_by_ssh Guide](articles/nagios-check-by-ssh.md) |
| [Nagios check_http Guide](articles/nagios-check-http-guide.md) |
| [Installing Cacti from Source](articles/cacti-installation-source.md) |

### Linux Memory

| Article |
|---------|
| [Linux Memory: RSS, VSZ, and Why RSS Alone Is Misleading](articles/linux-memory-rss-vsz.md) |
| [Linux Swap Usage: When Processes Aren't the Culprit](articles/linux-swap-shm-segments.md) |
| [Linux Swap Management](articles/linux-swap-management.md) |

### Linux Storage and Filesystems

| Article |
|---------|
| [Linux /etc/fstab Guide](articles/linux-fstab-guide.md) |
| [iSCSI Cheatsheet](articles/iscsi-cheatsheet.md) |
| [Linux NFS Cheatsheet](articles/linux-nfs-cheatsheet.md) |
| [Linux NFS Troubleshooting](articles/linux-nfs-troubleshooting.md) |
| [NFS Performance Testing and Monitoring](articles/nfs-performance-testing.md) |
| [Multipath Cheatsheet](articles/multipath-cheatsheet.md) |
| [EMC PowerPath Cheatsheet](articles/emc-powerpath-cheatsheet.md) |
| [SAN Storage Commands](articles/san-storage-commands.md) |
| [Fibre Channel Error Statistics](articles/fc-statistics-guide.md) |
| [Linux Storage Stack](articles/linux-storage-stack.md) |
| [fdisk Cheatsheet](articles/fdisk-cheatsheet.md) |
| [LVM Cheatsheet](articles/lvm-cheatsheet.md) |
| [fsck Cheatsheet](articles/fsck-cheatsheet.md) |
| [Partition Alignment Guide](articles/partition-alignment-guide.md) |
| [XFS Internals: Superblock and Addressing](articles/xfs-internals-superblock.md) |
| [ext4 Journal Modes](articles/ext4-journal-modes.md) |
| [Extending Partitions with growpart](articles/growpart-extend-partitions.md) |
| [Extend a SAN LUN Online with Multipath and GFS2](articles/linux-extend-lun-multipath-gfs2.md) |
| [GFS2 & RHEL Cluster Cheatsheet](articles/gfs2-cluster-cheatsheet.md) |
| [Disk Health & Maintenance](articles/disk-health-maintenance.md) |

### Networking

| Article |
|---------|
| [DNS Cheatsheet](articles/dns-cheatsheet.md) |
| [curl Cheatsheet](articles/curl-cheatsheet.md) |
| [Setting Up a DNS Server on RHEL 9](articles/dns-server-rhel9.md) |
| [Cloud DNS Routing Policies Explained](articles/dns-routing-policies.md) |
| [Elastic Load Balancing Best Practices](articles/elb-best-practices.md) |
| [resolvectl Cheatsheet](articles/resolvectl-cheatsheet.md) |
| [SSH Cheatsheet](articles/ssh-cheatsheet.md) |
| [SSH ControlMaster](articles/ssh-controlmaster.md) |
| [SSH ProxyJump vs ProxyCommand](articles/ssh-proxyjump-vs-proxycommand.md) |
| [SSH Managing Multiple Keys](articles/ssh-managing-multiple-keys.md) |
| [SSH Generate Keys](articles/ssh-keygen-guide.md) |
| [SSH Convert Keys](articles/ssh-convert-keys.md) |
| [SSH Remote Script Execution](articles/ssh-remote-script-execution.md) |
| [SSH Remote Sudo Execution](articles/ssh-remote-sudo-execution.md) |
| [SSH Heredoc Variable Expansion](articles/ssh-heredoc-variables.md) |
| [ip Command Cheatsheet](articles/ip-command-cheatsheet.md) |
| [Configure a Static IP with Netplan on Ubuntu](articles/ubuntu-netplan-static-ip.md) |
| [ss Cheatsheet](articles/ss-cheatsheet.md) |
| [VNC Cheatsheet](articles/vnc-cheatsheet.md) |
| [tcpdump Cheatsheet](articles/tcpdump-cheatsheet.md) |
| [Diagnosing Packet Loss with mtr](articles/mtr-packet-loss-guide.md) |
| [iperf3 Cheatsheet](articles/iperf3-cheatsheet.md) |
| [Test Network Speed Between Two Hosts](articles/network-speed-testing-guide.md) |
| [netstat Cheatsheet](articles/netstat-cheatsheet.md) |
| [/proc/net Cheatsheet](articles/proc-net-cheatsheet.md) |
| [Ephemeral Ports vs Conntrack Max](articles/ephemeral-ports-vs-conntrack.md) |
| [/proc/net/sockstat Explained](articles/proc-net-sockstat-explained.md) |
| [Monitor Interface Traffic](articles/monitor-interface-traffic.md) |
| [NetHogs Cheatsheet](articles/nethogs-cheatsheet.md) |
| [nmcli Cheatsheet](articles/nmcli-cheatsheet.md) |
| [UFW Cheatsheet](articles/ufw-cheatsheet.md) |
| [FirewallD Cheatsheet](articles/firewalld-cheatsheet.md) |
| [iptables and FirewallD Rules Guide](articles/iptables-firewalld-rules-guide.md) |
| [SNI and TLS Certificates Guide](articles/sni-certificates-guide.md) |
| [Nmap Cheatsheet](articles/nmap-cheatsheet.md) |

### Cloud-Init

| Article |
|---------|
| [cloud-init Cheatsheet](articles/cloud-init-cheatsheet.md) |
| [cloud-init status: Errors and Failure Modes](articles/cloud-init-status-command.md) |
| [cloud-init: bootcmd vs runcmd](articles/cloud-init-bootcmd-vs-runcmd.md) |
| [cloud-init: Why tee Output Doesn't Appear in Logs](articles/cloud-init-tee-output-missing.md) |
| [cloud-init: User Management and the gecos Field](articles/cloud-init-users-gecos.md) |
| [cloud-init: Different Ways to Create Files on a Server](articles/cloud-init-write-files.md) |
| [Cloud-Init Run Modes and Frequencies](articles/cloud-init-run-modes.md) |
| [Cloud-Init Heredoc and Logging Guide](articles/cloud-init-heredoc-logging-guide.md) |

### Datadog

| Article |
|---------|
| [Datadog Agent Cheatsheet](articles/datadog-agent-cheatsheet.md) |
| [Datadog API Reference](articles/datadog-api-reference.md) |
| [Datadog Dashboards Guide](articles/datadog-dashboards-guide.md) |
| [Datadog Monitor Notification Variables](articles/datadog-monitor-notification-variables.md) |
| [Datadog Monitor Tagging Best Practices](articles/datadog-monitor-tagging-best-practices.md) |
| [Datadog Monitors Tips & Tricks](articles/datadog-monitors-tips-and-tricks.md) |
| [Monitoring Apache Web Server Performance](articles/monitoring-apache-performance.md) |

### Terminal and Tools

| Article |
|---------|
| [JSON Query Tools: JMESPath vs jq vs JSONPath](articles/json-query-tools.md) |
| [bat Cheatsheet](articles/bat-cheatsheet.md) |
| [cut Cheatsheet](articles/cut-cheatsheet.md) |
| [tmux Cheatsheet](articles/tmux-cheatsheet.md) |
| [Kitty Cheatsheet](articles/kitty-cheatsheet.md) |
| [JetBrains Mono Font](articles/jetbrains-mono-font.md) |
| [PuTTY Default Settings](articles/putty-default-settings.md) |
| [rclone Cheatsheet](articles/rclone-cheatsheet.md) |
| [lssh Cheatsheet](articles/lssh-cheatsheet.md) |
| [VS Code Git Actions and Git CLI Equivalents](articles/vscode-git-cli-equivalents.md) |
| [Kiro CLI Cheatsheet](articles/kiro-cli-cheatsheet.md) |
| [Understanding Context Usage in AI Assistants](articles/ai-context-usage-explained.md) |
| [Orca: The Agent Development Environment (ADE)](articles/orca-agent-development-environment.md) |
| [Test a Docsify Site Locally](articles/docsify-test-locally.md) |

### macOS

| Article |
|---------|
| [Homebrew Cheatsheet](articles/homebrew-cheatsheet.md) |
| [macOS Bash Upgrade Guide](articles/macos-bash-upgrade-guide.md) |
| [Making List View the Default in macOS Finder](articles/macos-finder-default-list-view.md) |
| [Finding and Managing Git Credentials in the macOS Keychain](articles/macos-keychain-git-credentials.md) |
| [Remove .DS_Store from Git](articles/remove-ds-store-guide.md) |

### Windows

| Article |
|---------|
| [Windows Battery Report](articles/windows-battery-report.md) |
| [Windows Tips and Commands](articles/windows-tips-commands.md) |
| [Outlook Instant Search Syntax](articles/outlook-search-syntax.md) |

### Solaris

| Article |
|---------|
| [Solaris Disk and Filesystem Management](articles/solaris-disk-management.md) |
| [Solaris Zones](articles/solaris-zones.md) |
| [Solaris Boot Management: OpenBoot, eeprom, and bootadm](articles/solaris-boot-openboot.md) |
| [Solaris SVR4 Package Management](articles/solaris-svr4-package-management.md) |
| [Solaris 10 Patch Management](articles/solaris-patch-management.md) |
| [Solaris SMF (Service Management Facility) and Cron](articles/solaris-smf-services.md) |
| [Solaris System Information and Inventory](articles/solaris-system-information.md) |
| [Solaris Tips and Tricks](articles/solaris-tips-and-tricks.md) |
| [Solaris 11 IPS: Local Package Repository and pkg Management](articles/solaris-ips-pkg-repository.md) |
| [Solaris Performance and Resource Monitoring](articles/solaris-performance-monitoring.md) |
| [Solaris Network Configuration Files](articles/solaris-network-configuration.md) |
| [Solaris 11: Configure a Static IP with ipadm and dladm](articles/solaris11-static-ip-ipadm.md) |
| [Oracle Solaris 11 Installation Methods](articles/solaris11-installation-methods.md) |
| [Solaris User and Password Administration](articles/solaris-user-password-administration.md) |

### HP-UX

| Article |
|---------|
| [HP-UX History, Versions, and Support Lifecycle](articles/hpux-history-and-versions.md) |
| [HP-UX Boot Process (PA-RISC and Integrity)](articles/hpux-boot-process.md) |
| [HP-UX User and Password Administration](articles/hpux-user-password-administration.md) |
| [HP-UX LVM (Logical Volume Manager)](articles/hpux-lvm.md) |
| [HP-UX Virtual Partitions (vPars)](articles/hpux-vpars.md) |
| [HP-UX nPartitions (nPars)](articles/hpux-npars.md) |
| [HP-UX Management Processor (MP / GSP / iLO)](articles/hpux-management-processor.md) |
| [HP-UX Installation and Ignite-UX](articles/hpux-installation-ignite.md) |
| [HP-UX Disaster Recovery (DRD and Ignite-UX)](articles/hpux-disaster-recovery.md) |
| [HP-UX Patch Management](articles/hpux-patch-management.md) |
| [HP-UX Kernel Configuration and Tuning](articles/hpux-kernel-configuration.md) |
| [HP-UX Administration Tips and Recipes](articles/hpux-admin-tips-recipes.md) |
| [HP-UX Crash Dump Analysis with Q4](articles/hpux-crash-dump-analysis.md) |
| [HP-UX Performance Monitoring and Event Management](articles/hpux-performance-monitoring.md) |
| [HP-UX System Information and Initial Configuration](articles/hpux-system-information.md) |
| [HP-UX Startup, Run Levels, and Network Services](articles/hpux-startup-and-services.md) |
| [HP-UX Device Management (ioscan, scsimgr, DSFs)](articles/hpux-device-management-ioscan.md) |
| [HP-UX Software Distribution (SD-UX): Depots and swinstall](articles/hpux-software-depots-swinstall.md) |
| [HP-UX Network Configuration](articles/hpux-network-configuration.md) |
| [HP-UX SD-UX Software Structure, IPD, and swlist](articles/hpux-swlist-software-structure.md) |
| [HP-UX Swap and Pseudo-Swap Management](articles/hpux-swap-management.md) |
| [HP-UX Filesystem Management (HFS, JFS/VxFS)](articles/hpux-filesystem-management.md) |
| [HP-UX NFS (Server and Client)](articles/hpux-nfs.md) |
| [HP-UX Fibre Channel and SAN Storage](articles/hpux-fibre-channel-san.md) |

### AIX

| Article |
|---------|
| [IBM AIX: An Overview](articles/aix-overview.md) |
| [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md) |
| [AIX Boot and Init Cheatsheet](articles/aix-boot-init-cheatsheet.md) |
| [AIX CDE and X Window System Cheatsheet](articles/aix-cde-x11-cheatsheet.md) |
| [AIX Filesystems Cheatsheet](articles/aix-filesystems-cheatsheet.md) |
| [AIX LVM Cheatsheet](articles/aix-lvm-cheatsheet.md) |
| [AIX NIM Cheatsheet](articles/aix-nim-cheatsheet.md) |
| [AIX SMIT Cheatsheet](articles/aix-smit-cheatsheet.md) |
| [AIX LDAP Cheatsheet](articles/aix-ldap-cheatsheet.md) |
| [AIX NFS Cheatsheet](articles/aix-nfs-cheatsheet.md) |
| [AIX VIOS Cheatsheet](articles/aix-vios-cheatsheet.md) |
| [AIX Package Management Cheatsheet](articles/aix-package-management-cheatsheet.md) |
| [AIX HMC Cheatsheet](articles/aix-hmc-cheatsheet.md) |
| [AIX PowerVM Virtualization Concepts](articles/aix-powervm-virtualization-concepts.md) |
| [AIX Users and Groups Cheatsheet](articles/aix-users-groups-cheatsheet.md) |
| [AIX ODM Cheatsheet](articles/aix-odm-cheatsheet.md) |
| [AIX Software Updates and Fixes Cheatsheet](articles/aix-software-updates-fixes-cheatsheet.md) |
| [AIX Devices and Hardware Cheatsheet](articles/aix-devices-hardware-cheatsheet.md) |
| [AIX Cron and Job Scheduling Cheatsheet](articles/aix-cron-cheatsheet.md) |
| [AIX System Dump and Core File Cheatsheet](articles/aix-system-dump-core-cheatsheet.md) |
| [AIX Error Logging and System Logs Cheatsheet](articles/aix-error-logging-cheatsheet.md) |
| [AIX Power Systems, LPAR, and Boot Concepts](articles/aix-power-lpar-boot-concepts.md) |
| [AIX Performance Monitoring Cheatsheet](articles/aix-performance-monitoring-cheatsheet.md) |
| [AIX Networking Cheatsheet](articles/aix-networking-cheatsheet.md) |
| [AIX Paging Space Cheatsheet](articles/aix-paging-space-cheatsheet.md) |
| [AIX Login Auditing and Session Tracking Cheatsheet](articles/aix-login-auditing-cheatsheet.md) |
| [AIX / Power Service Processor and ASMI](articles/aix-service-processor-asmi.md) |
| [AIX MPIO and Fibre Channel Cheatsheet](articles/aix-mpio-fibre-channel-cheatsheet.md) |
| [AIX System Resource Controller (SRC) Cheatsheet](articles/aix-src-services-cheatsheet.md) |
| [AIX Storage Provisioning Tasks](articles/aix-storage-provisioning-tasks.md) |
| [AIX System Administration Tips Cheatsheet](articles/aix-sysadmin-tips-cheatsheet.md) |

## About

This is a personal wiki/knowledge base for tools, commands, and configurations used across a self-hosted homelab environment — spanning Kubernetes, container runtimes, networking, terminal tooling, and infrastructure as code. A collection of all commands and knowledge gathered over the last 15+ years, written as standalone Markdown references aimed at fast lookup rather than deep tutorials.

- **Author:** Daniel Corneschi
- **Site:** Built with [docsify](https://docsify.js.org/), hosted on [GitHub Pages](https://pages.github.com/)
- **Format:** All content lives in `articles/` — cheatsheets and longer-form guides side by side, with shared images in `articles/images/`
- **Scope:** Personal use — commands and examples reflect this homelab's specific setup (namespaces, socket names, config paths) and may need adjusting for other environments
