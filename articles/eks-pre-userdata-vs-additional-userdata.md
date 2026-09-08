# EKS: pre_userdata vs additional_userdata

## Overview

When configuring EC2 instances in EKS node groups, you can inject custom user
data scripts that run at two different phases of node initialization — before and
after the EKS bootstrap script that joins the node to the cluster.

> These `pre_userdata` / `additional_userdata` hooks come from the Terraform
> `terraform-aws-eks` module's self-managed/launch-template node group, wrapping
> the classic Amazon Linux 2 `/etc/eks/bootstrap.sh` flow. On **AL2023 and
> Bottlerocket**, bootstrapping uses `nodeadm` / node config (a MIME multipart
> user data) instead, and these two variables don't apply the same way — see the
> note at the end.

## These Names Are Not Built In

`pre_userdata` and `additional_userdata` are **custom variable names** you define
in your own Terraform when wiring native resources (`aws_launch_template` +
`aws_eks_node_group`). They aren't provided by any module.

The official `terraform-aws-modules/eks/aws` module implements the same idea with
different names:

| Your custom variable | Official EKS module equivalent |
|----------------------|--------------------------------|
| `pre_userdata` | `pre_bootstrap_user_data` |
| `additional_userdata` | `post_bootstrap_user_data` |

So the pattern is standard even though the variable names are your choice — pick
whatever is consistent in your codebase.

## pre_userdata vs additional_userdata

### pre_userdata

- **Timing:** runs **before** the EKS bootstrap script.
- **State:** kubelet and other Kubernetes components aren't configured yet.
- **Purpose:** system-level setup that must be in place before Kubernetes starts.

Common uses: installing system packages, configuring containerd/Docker, setting
up logging agents, tweaking system config, or setting environment variables the
bootstrap process needs.

### additional_userdata

- **Timing:** runs **after** the EKS bootstrap script.
- **State:** kubelet is configured and the node has joined the cluster.
- **Purpose:** steps that depend on Kubernetes already being operational.

Common uses: post-bootstrap customizations, node-local agents, or configuration
that assumes the node is registered.

## Examples

### pre_userdata

```bash
#!/bin/bash
# Install the CloudWatch agent before the EKS bootstrap
yum update -y
yum install -y amazon-cloudwatch-agent

# Configure the container runtime (containerd example)
mkdir -p /etc/containerd
# ...drop in custom config.toml here...
systemctl restart containerd

# Environment picked up by the bootstrap process
export KUBELET_EXTRA_ARGS="--node-labels=node-type=custom"
```

### additional_userdata

```bash
#!/bin/bash
# Runs after the node has joined the cluster
echo "node bootstrap complete" >> /var/log/node-postboot.log

# Start or configure a node-local agent that assumes kubelet is up
systemctl enable --now my-node-agent
```

> Avoid running `kubectl apply` or `helm install` from `additional_userdata` to
> configure the *cluster*. Every node would run it, racing each other. Cluster-
> level resources belong in your GitOps/CD pipeline, not node user data — keep
> node user data to node-local concerns.

## Terraform Example

The two scripts are stitched into a single user-data template that also contains
the bootstrap call:

```hcl
resource "aws_launch_template" "example" {
  name_prefix   = "eks-node-group-"
  image_id      = data.aws_ami.eks_worker.id
  instance_type = "t3.medium"

  user_data = base64encode(templatefile("${path.module}/userdata.sh", {
    cluster_name         = aws_eks_cluster.example.name
    cluster_endpoint     = aws_eks_cluster.example.endpoint
    cluster_ca           = aws_eks_cluster.example.certificate_authority[0].data
    bootstrap_extra_args = ""
    pre_userdata         = file("${path.module}/pre_userdata.sh")
    additional_userdata  = file("${path.module}/additional_userdata.sh")
  }))
}

resource "aws_eks_node_group" "example" {
  cluster_name    = aws_eks_cluster.example.name
  node_group_name = "example"
  node_role_arn   = aws_iam_role.example.arn
  subnet_ids      = aws_subnet.example[*].id

  launch_template {
    name    = aws_launch_template.example.name
    version = aws_launch_template.example.latest_version
  }
}
```

The `userdata.sh` template runs `pre_userdata`, then `/etc/eks/bootstrap.sh`,
then `additional_userdata`. A minimal template looks like:

```bash
#!/bin/bash
set -o xtrace

# Pre-bootstrap
${pre_userdata}

# EKS bootstrap (joins the node to the cluster)
/etc/eks/bootstrap.sh ${cluster_name} ${bootstrap_extra_args} \
  --kubelet-extra-args '${kubelet_extra_args}'

# Post-bootstrap
${additional_userdata}
```

## Ways to Supply the Scripts

The template variables can be fed from several sources — pick based on how much
you need to vary them:

| Approach | How | Best for |
|----------|-----|----------|
| Input variables | `pre_userdata = var.pre_userdata` | Multiple environments; override via tfvars |
| External files | `pre_userdata = file("${path.module}/pre.sh")` | Longer scripts, version-controlled separately |
| Local values | `pre_userdata = local.pre_userdata` | Single environment, moderate complexity |
| Inline heredoc | `pre_userdata = <<-EOT ... EOT` | Simple setups, prototyping |

Example passing an environment-specific value via `terraform.tfvars`:

```hcl
pre_userdata = <<-EOT
  apt-get update -y
  apt-get install -y htop jq awscli
EOT

additional_userdata = <<-EOT
  # wait for kubelet before doing anything cluster-aware
  while ! systemctl is-active --quiet kubelet; do sleep 5; done
  apt-get install -y sysstat iftop
EOT
```

## Execution Order

```text
1. Instance boots
2. pre_userdata runs
3. EKS bootstrap script runs
   - configures kubelet
   - joins the node to the cluster
   - starts Kubernetes components
4. additional_userdata runs
5. Node is Ready
```

## Key Considerations

- **Timing decides the choice:** does the step need Kubernetes running or not?
  Before → `pre_userdata`; after → `additional_userdata`.
- **The bootstrap script sits between them** and handles cluster join.
- **Fail loudly:** a broken script can leave the node stuck NotReady — add error
  handling and `set -euo pipefail` where appropriate.
- **Log execution** for troubleshooting:

  ```bash
  exec > >(tee /var/log/user-data.log | logger -t user-data -s 2>/dev/console) 2>&1
  ```

## Best Practices

- Keep scripts **idempotent** (safe to re-run).
- Use **absolute paths** for commands and files.
- Test in a non-production node group first.
- Watch the node's `/var/log/user-data.log` (or cloud-init logs) and CloudWatch.
- For complex configuration, consider AWS Systems Manager instead of large
  user-data scripts.

## Note on AL2023, Bottlerocket, and Auto Mode

The `pre_userdata` / `additional_userdata` split is tied to the AL2
`bootstrap.sh` model. On newer setups:

- **AL2023** bootstraps with `nodeadm` and a MIME multipart user data
  (`NodeConfig` YAML); custom scripts are added as additional MIME parts rather
  than pre/additional strings.
- **Bottlerocket** uses TOML settings, not shell user data.
- **EKS Auto Mode** manages nodes for you — you generally don't supply node user
  data at all.

Confirm which AMI family your node group uses before copying these patterns.

## Related

- [EKS Node Bootstrap Deep Dive](articles/eks-node-bootstrap-deep-dive.md)
- [EKS Node Groups: Launch Templates](articles/eks-nodegroups-launch-templates.md)
- [EOF Escaping in Terraform user_data](articles/eof-escaping-userdata-terraform.md)

## Skills Practiced

- Choosing between pre- and post-bootstrap user data based on Kubernetes dependency
- Wiring both scripts into a launch-template user-data template in Terraform
- Understanding the boot → pre → bootstrap → additional → Ready order
- Recognizing when the AL2 model doesn't apply (AL2023 nodeadm, Bottlerocket, Auto Mode)
